# MCP Servers — Add, Remove, Enable, Disable

Every MCP server you enable prepends its full tool schema to **every single request**,
including "what is 2x2?". On a local model this is the single biggest lever you have over
response latency. Keep enabled only what the session actually needs.

For the build steps see `runbook.md`. For why things break see `reference.md`.

All edits happen in one file:

```
~/.config/opencode/config.json
```

Restart opencode after any change — the `mcp` block is read at startup, not per turn.

---

## Current state

| Server | Type | Endpoint | Tools | Default |
|---|---|---|---|---|
| `searxng` | local (stdio) | `http://192.168.1.101:8080` | 4 | **enabled** |
| `noname` | remote | `http://192.168.1.102:8013/mcp` | 39 | disabled |
| `crapi` | remote | `http://192.168.1.102:8009/mcp/` | 29 | disabled |

`searxng` stays on by default — it's the grounding layer that stops the model inventing
versions, ports, and image names. The other two are task-specific: turn them on for API
security work, then turn them back off.

---

## The toggle

Each server takes an `enabled` flag. One word, nothing else changes:

```json
"noname": {
  "type": "remote",
  "url": "http://192.168.1.102:8013/mcp",
  "enabled": false        // ← true to turn on, false to turn off
}
```

**Always write the flag explicitly, even when it's `true`.** A missing `enabled` key
defaults to on, so an unflagged server is silently active and its token cost is invisible
when you skim the file. Every server in this config carries the flag on purpose.

### Turn a server on for one task

```bash
# 1. flip the flag
vi ~/.config/opencode/config.json     # "enabled": false → true

# 2. restart opencode (mcp block is start-time only)
# 3. do the work
# 4. flip it back to false when done
```

There is no per-session override and no CLI flag for this — editing the file and
restarting is the whole mechanism.

---

## What it costs

Measured against the live stack, same prompt (`what is 2x2?`), same model
(Qwen3-Coder-30B-A3B on the 5090):

| Enabled servers | Tools | Prompt tokens | Time to first token | Total |
|---|---|---|---|---|
| none | 0 | 15 | 0.03s | 1.0s |
| **searxng only** ← default | 4 | 2,155 | 1.67s | 2.1s |
| searxng + noname + crapi | 72 | 10,121 | 5.68s | 6.8s |

Turning off `noname` and `crapi` cuts the prompt by **79%** (10,121 → 2,155 tokens) and
makes first-token latency **3.4x faster**.

The cost is per-request and unavoidable while a server is enabled — the schema goes in the
system prompt, so the model re-reads all 72 tool definitions before answering arithmetic.
Two other things follow from that:

- **Context burn.** 10k of your 64k window is gone before you type anything, so long
  sessions compact sooner and lose history faster.
- **Tool-selection accuracy.** A 30B model choosing among 72 tools picks wrong more often
  than one choosing among 4. Fewer enabled servers means better tool calls, not just
  faster ones.

---

## Add a remote server (HTTP / streamable)

```json
"mcp": {
  "my-server": {
    "type": "remote",
    "url": "http://192.168.1.102:9000/mcp",
    "headers": {
      "Authorization": "Basic <base64-of-user:pass>"
    },
    "enabled": true
  }
}
```

- `type` must be `"remote"`.
- `url` — trailing slash matters on some servers. `crapi` needs `/mcp/`, `noname` uses
  `/mcp`. If a server won't initialize, try the other form before debugging anything else.
- `headers` is optional; omit the block entirely for unauthenticated servers.

Generate a Basic auth value:

```bash
printf 'user@my.lab:password' | base64
```

> Credentials sit in plaintext in `config.json`. It's a LAN lab file — treat it as a
> secret, don't commit it to a public repo.

## Add a local server (stdio subprocess)

```json
"mcp": {
  "my-tool": {
    "type": "local",
    "command": ["npx", "-y", "some-mcp-package"],
    "environment": {
      "SOME_URL": "http://192.168.1.101:9999"
    },
    "enabled": true
  }
}
```

- `type` must be `"local"`.
- `command` is an **array**, not a string. opencode launches it and speaks MCP over
  stdin/stdout.
- `environment` (not `env`) passes variables to the subprocess.

**Pin the version.** `npx -y some-package` fetches and runs the latest unaudited code
from npm on every launch:

```json
"command": ["npx", "-y", "mcp-searxng@1.0.0"]
```

## Remove a server

Prefer `"enabled": false` over deleting — you keep the URL, headers, and version pin for
next time, which is the whole reason the disabled blocks are still in the file. Delete the
block outright only when you're done with the server for good.

---

## Verify what's actually loaded

Config is valid and flags are what you think:

```bash
python3 -c "
import json; c=json.load(open('$HOME/.config/opencode/config.json'))
for n,s in c['mcp'].items(): print(f'{n:10s} enabled={s.get(\"enabled\")}')
"
```

A remote server is reachable and its tool count (does `initialize` then `tools/list`):

```bash
URL=http://192.168.1.102:8013/mcp
SID=$(curl -s -D - -o /dev/null -X POST "$URL" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}' \
  | grep -i '^mcp-session-id' | tr -d '\r' | cut -d' ' -f2)

curl -s -X POST "$URL" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
  | sed 's/^data: //' | grep '{' \
  | python3 -c "import json,sys; print(len(json.load(sys.stdin)['result']['tools']),'tools')"
```

What the model is actually being charged, end to end — ask opencode `what is 2x2?` and
read `prompt_tokens`. Anything over ~2,500 with only searxng on means something else is
enabled.

---

## Gotchas

**A bare `GET` on a remote MCP URL looks like a hang.** It returns HTTP 200 and holds the
connection open — that's normal for streamable HTTP. Use the `POST` + `initialize` probe
above instead of `curl <url>`, or you'll chase a phantom outage.

**A disabled server is not a dead server.** `"enabled": false` only stops opencode from
loading it. The container on .102 keeps running; nothing on the server side changes.

**Restart is required.** Flipping a flag mid-session does nothing. The `mcp` block is read
once at startup.

**Slow ≠ hung.** If every request crawls after you enable a server, check the 5090's VRAM
before blaming the config — see the `G7` row in `reference.md`. An oversubscribed GPU turns
a 10k-token prefill into minutes of CPU work, which is what actually looks like a hang.

**Tool name collisions.** opencode namespaces tools by server (`searxng_web_search`), so
two servers exposing `search` won't clash. Server names themselves must be unique keys.
