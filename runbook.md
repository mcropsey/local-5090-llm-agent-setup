# Local Agentic Stack — Ground-Up Runbook

Follow this top to bottom on a fresh machine. Three machines involved:

| Machine | IP | Role |
|---|---|---|
| Laptop (macOS) | — | opencode harness, SSH client, AWS CLI |
| 5090 box (Windows) | 192.168.1.194 | LM Studio — serves the model |
| hv-rocky-linux-4 (Rocky) | 192.168.1.101 | SearXNG container (rootful podman) |

---

## Phase 1 — Serve the model (5090 box)

**Tooling:** LM Studio (GUI, already installed on the 5090)

1. **Enable Developer mode**
   Settings → Developer → toggle **Developer mode** ON. This unlocks the Developer tab.

2. **Load a model**
   Discover tab → download **Qwen3-Coder-30B-A3B**. When loading, set on the **Load** tab (not Inference):
   - Context length (`n_ctx`): **65536** (64k)
   - Flash Attention: **ON**
   - KV Cache Quantization: **Q8** (K and V)
   - Quant: Q4_K_M or Q5 — below Q4 tool-calling reliability degrades

   > VRAM math: 30B-A3B at Q4 ≈ 18 GB, leaving ~13 GB for KV cache on a 32 GB 5090.
   > Q8 cache quant lets 64k context fit comfortably. Start here; go to 32k if it's tight.

3. **Start the server**
   Developer → Local Server → toggle **Status: Running** (port 1234 default).

4. **Enable LAN access**
   Gear icon → **Serve on Local Network: ON**. Leave Require Authentication OFF (LAN only — revisit before any external exposure).

   **Load exactly one model, and turn JIT loading OFF** (or cap max loaded models at 1).
   A 32 GB 5090 fits one 30B-A3B at 64k context (~21 GB) and nothing else. A second
   resident model spills the whole thing to system RAM and drops you from ~65 tok/s to
   ~1 tok/s, which presents as opencode hanging. JIT is also what silently loads a
   *duplicate* of the same model: opencode issues its title-generation call concurrently
   with the main call at session start, and JIT answers the second one by loading another
   full copy. See `G7` in `reference.md`.

5. **Verify from the laptop**
   ```bash
   curl http://192.168.1.194:1234/v1/models
   ```
   Should return a JSON list with `qwen3-coder-30b-a3b-instruct` (or similar). Note the exact model ID — you'll need it in Phase 2.

   First request to a model is slow (30–90 s of VRAM loading). Not a hang.

---

## Phase 2 — Install opencode (laptop)

1. **Install**
   ```bash
   curl -fsSL https://opencode.ai/install | bash
   ```
   Installs to `~/.local/bin/opencode`.

2. **Fix PATH** (the installer doesn't always add it)
   ```bash
   echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
   source ~/.zshrc
   which opencode   # must print ~/.local/bin/opencode
   ```

3. **Set up key auth and passwordless sudo on the Rocky box**

   SSH key auth (agent can't answer password prompts):
   ```bash
   ssh-keygen -t ed25519            # skip if you already have a key
   ssh-copy-id mcropsey@192.168.1.101
   ssh mcropsey@192.168.1.101 hostname   # must return with NO password prompt
   ```

   Passwordless sudo for podman (agent needs rootful podman; no TTY for a password prompt):
   ```bash
   # on 192.168.1.101, as root:
   echo 'mcropsey ALL=(ALL) NOPASSWD: /usr/bin/podman' > /etc/sudoers.d/podman-agent
   chmod 440 /etc/sudoers.d/podman-agent
   visudo -c    # verify no syntax errors

   # verify from the laptop — must return with no prompt:
   ssh mcropsey@192.168.1.101 'sudo -n podman ps'
   ```

   Without passwordless sudo, the agent's `sudo podman ps` fails silently — rootless `podman ps` returns an empty list, making SearXNG look gone when it's not.

4. **Write `~/.config/opencode/config.json`**
   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "model": "local5090/qwen3-coder-30b-a3b-instruct",
     "provider": {
       "local5090": {
         "npm": "@ai-sdk/openai-compatible",
         "name": "LM Studio 5090",
         "options": {
           "baseURL": "http://192.168.1.194:1234/v1",
           "apiKey": "not-needed"
         },
         "models": {
           "qwen3-coder-30b-a3b-instruct": {
             "name": "Qwen3-Coder 30B-A3B",
             "tools": true,
             "limit": {
               "context": 65280,
               "output": 8192
             }
           }
         }
       }
     },
     "mcp": {
       "searxng": {
         "type": "local",
         "command": ["npx", "-y", "mcp-searxng"],
         "environment": {
           "SEARXNG_URL": "http://192.168.1.101:8080"
         },
         "enabled": true
       },
       "noname": {
         "type": "remote",
         "url": "http://192.168.1.102:8013/mcp",
         "enabled": false
       },
       "crapi": {
         "type": "remote",
         "url": "http://192.168.1.102:8009/mcp/",
         "headers": {
           "Authorization": "Basic bWlrZTFAbXkubGFiOk15bGFiMTIzIQ=="
         },
         "enabled": false
       }
     },
     "permission": {
       "bash": "ask"
     }
   }
   ```

   Config notes:
   - File is **`config.json`**, not `opencode.json` — many guides are wrong.
   - MCP schema is `mcp` / `type: "local"` / `command` as an **array**. The generic
     `mcpServers` + `command`/`args` shape won't load.
   - **`"bash": "ask"` is the whole permission block.** It accepts a bare string, so you
     don't need per-command patterns. Every command prompts — including `sudo`, `rm`,
     anything — and nothing is pre-approved or blocked. This replaced a 57-line pattern
     list; the patterns were never sent to the model and cost nothing in tokens, but they
     were easy to get subtly wrong and hard to audit. Approve at the prompt instead.
   - `"context": 65280`, not 65536. LM Studio reports `loaded_context_length: 65280` when
     you ask for 64k, and opencode uses this number to decide when to compact. Setting it
     256 tokens too high means requests at the top of the window get rejected with
     `Internal Server Error` — see `G8` in `reference.md`.
   - **`searxng` is enabled; `noname` and `crapi` ship disabled.** Those two add 68 tool
     definitions and ~8,000 prompt tokens to *every* request. Flip `enabled` to `true`
     when doing API security work, then back to `false`. Full instructions and measured
     costs: `mcp-servers.md`.

5. **Verify opencode connects**
   ```bash
   mkdir ~/opencode-test && cd ~/opencode-test
   git init
   opencode
   ```
   Ask it: `"List the files in this directory."` — it should **run the tool** and return actual output, not hallucinate a file list.

   If you get `exceeds the available context size (8192 tokens)` → go back to LM Studio and set context to 64k, then reload the model.

---

## Switching models — and why loading one in LM Studio isn't enough

**Loading a model in LM Studio does not make opencode use it.** opencode only ever requests
the model IDs declared in its `provider.local5090.models` block, and it defaults to
`config.json`'s top-level `"model"`. Load DeepSeek in the GUI, and opencode still asks for
`qwen3-coder-30b-a3b-instruct` — LM Studio then has to JIT-load Qwen *alongside* DeepSeek,
which on a 32 GB card can fail outright:

```
HTTP 400  Failed to load model "qwen3-coder-30b-a3b-instruct".
          Error: Engine protocol startup was aborted.
```

That 400 is what "opencode won't connect" actually looks like. Observed on this stack —
and it's intermittent, because it depends on what else is resident at that moment.

To actually switch models you must declare the model, then select it:

Every tool-capable model on the box, as currently configured:

```json
"model": "local5090/qwen/qwen3.8-27b",
...
"models": {
  "qwen/qwen3.8-27b": {
    "name": "Qwen3.8 27B (default)",
    "tools": true,
    "limit": { "context": 98304, "output": 8192 }
  },
  "qwen/qwen3.6-27b": {
    "name": "Qwen3.6 27B",
    "tools": true,
    "limit": { "context": 65280, "output": 8192 }
  },
  "qwen3-coder-30b-a3b-instruct": {
    "name": "Qwen3-Coder 30B-A3B",
    "tools": true,
    "limit": { "context": 65280, "output": 8192 }
  },
  "qwen/qwen3-coder-next": {
    "name": "Qwen3-Coder Next",
    "tools": true,
    "limit": { "context": 65280, "output": 8192 }
  },
  "devstral-small-2-24b-instruct-2512": {
    "name": "Devstral Small 2 24B",
    "tools": true,
    "limit": { "context": 65280, "output": 8192 }
  },
  "qwen3-30b-a3b-thinking-2507": {
    "name": "Qwen3 30B-A3B Thinking",
    "tools": true,
    "limit": { "context": 65280, "output": 8192 }
  },
  "openai/gpt-oss-20b": {
    "name": "GPT-OSS 20B",
    "tools": true,
    "limit": { "context": 65280, "output": 8192 }
  }
}
```

`limit.context` is `65280` for the 64k-loaded entries because that's what LM Studio reports
when you ask for 64k. **It is a per-model claim, not a global one** — if you load one of
these at a different context, fix its entry or you get the `G8` overrun. Qwen3.8 is the
live example: it is loaded at 96k, so its entry says `98304`. Read the number off the box
rather than copying a neighbour's:

```bash
curl -s http://192.168.1.194:1234/api/v0/models | python3 -c "
import json,sys
for m in json.load(sys.stdin)['data']:
    if m.get('state')=='loaded':
        print(f\"{m['id']:38s} loaded_ctx={m.get('loaded_context_length')}\")
"
# qwen/qwen3.8-27b    loaded_ctx=98304
```

Too low only costs you early compaction; too high is the `G8` `Internal Server Error`.

Then pick it, three ways:

| How | Command / key | Scope |
|---|---|---|
| Default | top-level `"model": "local5090/<id>"` in config.json | every session |
| Per session | `opencode -m local5090/<id>` | that launch only |
| Mid-session | `ctrl+alt+m` (model list), `ctrl+alt+.` (cycle recent) | that session |

The two keybinds are bound in `~/.config/opencode/tui.json` — opencode ships the
`model_list` / `model_cycle_recent` actions with **no default key**, so they do nothing
until you bind them.

Confirm opencode can actually see a model before launching the TUI:

```bash
opencode models local5090
```

**If it isn't in that list, opencode cannot use it — full stop.** There is no
auto-discovery of LM Studio's loaded models; the `models` block in config.json is the
complete universe of what opencode will request. This is the single most common reason
"I loaded it but opencode won't use it."

> **Use the model's full ID, publisher prefix included.** A slash in the ID is fine —
> opencode splits `provider/model` on the *first* slash only, so
> `local5090/qwen/qwen3.8-27b` resolves correctly (verified with `opencode models`).
>
> **Do not "simplify" it by dropping the prefix.** LM Studio will answer a request for
> the short `qwen3.8-27b` with HTTP 200, which makes the shortcut look safe — but it
> treats the short name as a *separate model* and JIT-loads a second copy at the **8192
> default context**, not your 96k. Both then sit in VRAM:
>
> ```
> qwen/qwen3.8-27b    loaded_ctx=98304   ← the one you configured
> qwen3.8-27b         loaded_ctx=8192    ← duplicate from the short name
> ```
>
> That's a silent `G3` (8k context) stacked on a `G7` (VRAM oversubscription), from a
> config that looks like it works. Match the ID from `/api/v0/models` exactly.

### Which models on the 5090 can actually drive the harness

opencode is tool-driven, so `tool_use` is non-negotiable. Ask LM Studio rather than
guessing — it publishes capabilities per model:

```bash
curl -s http://192.168.1.194:1234/api/v0/models | python3 -c "
import json,sys
for m in json.load(sys.stdin)['data']:
    if m.get('type')=='embeddings': continue
    caps=m.get('capabilities') or []
    print(f\"{m['id']:42s} {'tool_use OK' if 'tool_use' in caps else 'no tool_use'}\")
"
```

Current inventory (checked 2026-09-08):

**The box now holds only the two Qwen3.x-27B VLMs.** The older set was cleaned off it, but
their entries are still declared in config.json — declaring a model that isn't on the box
is harmless until you *select* it, at which point you get the JIT-load `HTTP 400` above
(or, with JIT off, a plain model-not-found). Treat the "On box" column as the one that
decides what you can actually run.

| Model | Tool use | On box | In config | Notes |
|---|---|---|---|---|
| `qwen/qwen3.8-27b` | ✅ | ✅ | ✅ | **Current default.** VLM, Q4_K_M, loaded at 96k (`98304`) |
| `qwen/qwen3.6-27b` | ✅ | ✅ | ✅ | Previous default. VLM, Q4_K_M, ~19 GB at 64k |
| `qwen3-coder-30b-a3b-instruct` | ✅ | — | ✅ | Default before 3.6; ~21 GB at 64k. Removed from box |
| `qwen/qwen3-coder-next` | ✅ | — | ✅ | Never tested here. Removed from box |
| `devstral-small-2-24b-instruct-2512` | ✅ | — | ✅ | The A/B candidate in Open items. Removed from box |
| `qwen3-30b-a3b-thinking-2507` | ✅ | — | ✅ | Reasoning variant — expect the token burn described below. Removed from box |
| `openai/gpt-oss-20b` | ✅ | — | ✅ | Smallest; left real VRAM headroom. Removed from box |
| `deepseek/deepseek-r1-0528-qwen3-8b` | ❌ | — | — | Can't tool-call — see below |
| `ibm/granite-3.2-8b` | ❌ | — | — | No capabilities declared |
| `qwen/qwen2.5-coder-32b` | ❌ | — | — | No capabilities declared |

Capability metadata is only populated once a model has been loaded at least once, so a
never-loaded model may report nothing until you load it. Treat "no capabilities" on a
never-loaded model as unknown, not as a definite ❌.

Verify the set resolves through opencode, not just LM Studio:

```bash
opencode models local5090     # lists the 7 declared entries, box contents notwithstanding
```

`opencode models` reads config.json, **not** the box — it will happily list a model that
was deleted from the 5090. Cross-check against `/v1/models` before blaming opencode:

```bash
curl -s http://192.168.1.194:1234/v1/models | python3 -c \
  "import json,sys; [print(m['id']) for m in json.load(sys.stdin)['data']]"
```

### Don't use DeepSeek-R1-8B as the driver model

It cannot make tool calls, which is the one thing the harness needs. Same prompt, same
tool definition, measured side by side:

| Model | `finish_reason` | `tool_calls` | Reasoning tokens |
|---|---|---|---|
| qwen3-coder-30b-a3b | `tool_calls` | **1** — `bash({"command":"ls -la"})` | 0 |
| deepseek-r1-0528-qwen3-8b | `stop` | **0** | 1,139 |

DeepSeek answered by *typing* a markdown fence — <code>```bash ls ```</code> — as ordinary
text instead of emitting a structured call. LM Studio confirms this in its model metadata:
Qwen advertises `capabilities: ["tool_use"]`, DeepSeek declares no capabilities at all.

Two more things make it a poor fit:

- **It burns the output budget thinking.** At `max_tokens: 200` it spent 198 tokens
  reasoning and returned `content: ""` with `finish_reason: "length"` — a blank reply. It
  needed ~380 tokens just to answer "what is 2+2?".
- **Its answer lands in the wrong field.** Reasoning models put text in
  `reasoning_content`, which is a DeepSeek/LM Studio extension that
  `@ai-sdk/openai-compatible` doesn't read. opencode sees `content: ""` and displays
  nothing — the `G6` "thinks, emits nothing" symptom, from the model rather than the
  chat template.

Keep DeepSeek for one-off reasoning questions you ask it directly. Leave Qwen3-Coder as
the harness driver.

### Turn off JIT loading and idle auto-unload

LM Studio ships with both on, and together they cause the two weirdest symptoms on this
stack:

| Setting | Symptom when left on |
|---|---|
| **JIT model loading** | Models load that you never asked for — *any* API request naming an unloaded model triggers a load. This is what creates the duplicate `qwen3-coder-30b-a3b-instruct:2` instance and what oversubscribes VRAM (`G7`). |
| **Idle TTL auto-unload** | Models silently unload after inactivity. The next opencode request pays a 30–90 s cold load from disk before the first token — indistinguishable from a hang. |

Settings → Developer → turn off **Just-In-Time model loading** and set the **idle TTL** to
off (or cap **max loaded models** at 1). Then load Qwen3-Coder once, manually, and leave it
resident. Load state becomes something you control instead of a side effect of whatever
last hit the API.

---

## Phase 3 — SearXNG on hv-rocky-linux-4 (.101)

SearXNG gives the model live web search with no API key and no data leaving the LAN.
Run everything with `sudo` — this is **rootful** podman deliberately (predictable storage paths, reboot survival via system service).

### 3a — Run the container

SSH to .101, then:

```bash
sudo podman run -d \
  --name searxng \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v searxng:/etc/searxng:z \
  docker.io/searxng/searxng:latest
```

> **Volume name is `searxng`**, not `searxng-config`. Using the wrong name creates a second orphan volume and leaves the config empty.

### 3b — Survive reboots

Unlike Docker, podman's `--restart` flag alone does **not** replay containers after a host reboot. Enable the system service that does:

```bash
sudo systemctl enable --now podman-restart.service
```

`podman-restart.service` starts all containers whose restart policy is `always` or `unless-stopped` (it filters on `should-start-on-boot=true`, which podman sets automatically for both of those policies).

**Verify it is enabled:**
```bash
systemctl is-enabled podman-restart.service   # should print: enabled
systemctl is-active  podman-restart.service   # should print: active
```

### 3c — Enable JSON output (the critical gotcha)

SearXNG ships with only HTML output. The MCP needs JSON. Skip this and every search returns HTTP 403 — no obvious error message.

Find the config path:
```bash
sudo podman volume inspect searxng --format '{{.Mountpoint}}'
# returns: /var/lib/containers/storage/volumes/searxng/_data
```

Edit `settings.yml` at that path:
```bash
sudo vi /var/lib/containers/storage/volumes/searxng/_data/settings.yml
```

Ensure these blocks are present:
```yaml
search:
  formats:
    - html
    - json

server:
  secret_key: "replace-with-something-random"   # change from the default
  image_proxy: true
```

Then restart:
```bash
sudo podman restart searxng
```

### 3d — Verify from the laptop before touching opencode

```bash
curl "http://192.168.1.101:8080/search?q=test&format=json"
```

- JSON back → good, proceed.
- HTML or 403 → JSON not enabled, redo 3c.
- Connection refused → firewall or binding issue; confirm port 8080 is open on .101 and the container published to the LAN interface.

Don't skip this check. Every downstream problem traces back to SearXNG not returning JSON.

### 3e — mcp-searxng install (automatic)

`mcp-searxng` is already in your config.json. On first opencode launch, it runs `npx -y mcp-searxng`, which downloads the package from npm and caches it in `~/.npm/_npx/`. No manual install needed.

Supply-chain note: `npx -y` runs unaudited npm code. Before relying on it, confirm the package:
```bash
npm view mcp-searxng
```
Should point at `ihor-sokoliuk/mcp-searxng` on GitHub (not a typosquat). To lock a version:
```json
"command": ["npx", "-y", "mcp-searxng@1.0.0"]
```

---

## LM Studio reboot note (5090 box — Windows)

LM Studio does not auto-start the local server after a Windows reboot. After any reboot of the 5090 box:

1. Open LM Studio
2. Load the model (Discover → **Qwen3-Coder-30B-A3B** → Load with 64k context, FlashAttn ON, Q8 KV)
3. Developer → Local Server → toggle **Status: Running**
4. Verify from the laptop: `curl http://192.168.1.194:1234/v1/models`

> LM Studio has a **Launch at Login** option (Settings → General) but it only opens the GUI — it does not automatically reload the model or start the server. The server start is always manual after a reboot.

**If you want the server to come up automatically**, create a Windows Task Scheduler task that runs on logon and calls the LM Studio CLI with your saved preset — but LM Studio's CLI support for this is limited. Simplest path for now: treat the 5090 as a manual step after any reboot.

---

## Phase 4 — AGENTS.md (behavioral control plane)

Create `~/.config/opencode/AGENTS.md`. This is loaded into every system prompt.

**Order is load-bearing.** Local models weight earlier instructions more heavily. The tool-access block (CRITICAL section) must come first or the model's built-in safety training will win, causing it to refuse SSH/sudo calls — especially as the file grows longer.

```markdown
## CRITICAL: You have full shell and SSH access — use it

You have a working bash tool with full shell access. You MUST use it.
Never refuse to run shell commands or SSH into servers — that is your job here.
Never tell the user to run commands themselves. Run them and show the output.

You CAN reach machines on the local network via SSH.
SSH key auth is already configured for user mcropsey on all local servers.
When asked about a remote machine, connect with:
  ssh mcropsey@<ip-or-hostname>
using whatever IP or hostname the user gives you.

This includes commands that require sudo. When a task needs sudo, run it via the bash
tool with sudo — do not explain how to run it manually. opencode will automatically
prompt the user for approval before executing; you do not need to ask permission first.

This also includes AWS CLI commands. When asked about AWS resources, run `aws` commands
directly via the bash tool. AWS credentials are configured on this machine. opencode
will prompt for approval before executing.

## Truthfulness (anti-fabrication)
- Only report output that was literally returned by a tool call in THIS session.
- Never invent, predict, or pretty-print output you did not receive.
- If a result is missing, empty, or errored, say so plainly and stop —
  do not narrate what success "would" look like.
- Never claim a resource is created, running, or reachable unless a real
  command confirmed it. Verify state before asserting it.

## Verification
When you run a command that changes state (start/stop a container, write a file,
restart a service), run a follow-up command that verifies the result before reporting
success. E.g. after `podman run`, run `podman ps` and confirm the container shows "Up"
in the output you actually received.

## Loop discipline
- Run one command, read the actual result, then decide the next step.
- Do not emit menus of 4–6 hypothetical commands. Act, observe, report.

## Grounding
- Before deploying or configuring a named project, look up its real docs
  (web search tool) instead of guessing image names, ports, or flags.
  Example: crAPI is a multi-service docker-compose stack from the OWASP/crAPI repo,
  NOT a single Docker image.

## Cloud / security defaults
- Never open security group rules to 0.0.0.0/0 for lab or vulnerable targets.
  Scope to the user's own IP (curl https://checkip.amazonaws.com → CIDR /32).
- Prefer key-based SSH over passwords. Never echo secrets into commands that
  land in shell history or logs.
- Prefer Terraform/OpenTofu plan-then-apply over raw imperative mutations for
  anything beyond a one-off inspection.
- Tag lab resources with a clear prefix and include a teardown step so
  intentionally-vulnerable instances don't linger publicly.

## Known servers
- 192.168.1.101 (hv-rocky-linux-4): Rocky Linux podman host, lab network only.

## When to search the web
Your training data has a cutoff and is often out of date. Before answering anything
about current versions, package/CLI flags, library APIs, or how a public project is
configured today, assume your knowledge MAY be stale and use the searxng search tool.

Search — do not guess — whenever:
- You're about to state a version number, port, flag, or install command.
- The user mentions something released or changed recently.
- You're unsure whether what you "know" is still current.

Quote the search results. If results contradict your training data, trust the results.

If a sudo command fails with "a terminal is required to read the password", tell the
user to run `sudo -v` in their terminal to cache credentials, then retry the command.
Do NOT fall back to explaining how to run the command manually.
```

---

## Phase 5 — End-to-end verification

Run these in order. Each one isolates a different layer.

```bash
# 1. Model reachable
curl http://192.168.1.194:1234/v1/models

# 2. SearXNG reachable + JSON enabled
curl "http://192.168.1.101:8080/search?q=test&format=json"

# 3. SSH key auth working
ssh mcropsey@192.168.1.101 hostname   # no password prompt
```

In opencode:
```
# 4. Tool-calling works
"List the files in this directory."
→ should run ls, not hallucinate a list

# 5. SSH + rootful podman works
"SSH to 192.168.1.101 and show me what's running in rootful podman."
→ should run `sudo podman ps` and return SearXNG in the list
→ if it runs plain `podman ps` and gets an empty list, it reached rootless — Phase 2 step 3 sudo setup is missing

# 6. Web search works
"Search the web for the current recommended way to run OWASP crAPI, and quote the results."
→ should call the searxng tool and quote real results

# 7. AWS works (if needed)
"Use your bash tool to run `aws sts get-caller-identity` and show me the actual output."
→ should call the tool; opencode prompts you to approve first
```

---

## Operational gotchas

### sudo fails with "a terminal is required" (macOS local only)

opencode runs bash non-interactively (no TTY). macOS sudo refuses to prompt for a password without one. This affects **local sudo on the laptop** — not remote sudo on .101, which is handled by the NOPASSWD sudoers rule in Phase 2 step 3.

**Fix:** Before any session where local sudo is needed, run once in your terminal:
```bash
sudo -v
```
Caches credentials for ~5 min. Re-run if the session runs long. AGENTS.md already tells the model to prompt you for this instead of giving up.

### SSH stops working after adding content to AGENTS.md

Adding new sections (like the search rules) pushes the CRITICAL block further from the top. Local model safety training starts winning.

**Fix:** The CRITICAL block must stay at the very top. If SSH breaks after an AGENTS.md edit, this is almost always why. Per-prompt workaround: prefix with "Use bash and SSH to…" — naming the tool explicitly bypasses the filter.

### Podman lock error

If you see `acquiring lock 0 … file exists`, the container is wedged — not just a warning:
```bash
sudo podman rm -f <container-id>
sudo podman stop --all
sudo podman system renumber    # rebuilds the lock table
```

### Model claims success it didn't verify

Classic small-model over-claiming. The AGENTS.md Verification block counters it. Also end prompts with explicit checks: "then run `podman ps` and show me it's actually Up before claiming success."

### Empty response after "thinking"

Tool-call formatting fumble — usually a wrong chat template for the loaded GGUF. Confirm LM Studio is using the model's native tool-use chat template, not a generic one.

---

## Optional: hybrid frontier escape hatch

For tasks too hard for the local model, keep a second provider in config.json:

```json
"provider": {
  "local5090": { "...": "as above" },
  "frontier": {
    "npm": "@ai-sdk/anthropic",
    "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" },
    "models": {
      "claude-sonnet-5": { "tools": true }
    }
  }
}
```

Switch with `--model frontier/claude-sonnet-5` for the hard 5%; local for everything else.

---

## Quick reference

```bash
# One-time per server the agent should reach:
ssh-copy-id mcropsey@<ip>
ssh mcropsey@<ip> hostname    # verify: no password prompt

# Before any session needing sudo:
sudo -v

# Wedged podman containers:
sudo podman rm -f <id>
sudo podman stop --all
sudo podman system renumber

# SearXNG container management (all with sudo):
sudo podman ps                                                  # verify it's Up
sudo podman restart searxng
sudo podman volume inspect searxng --format '{{.Mountpoint}}'  # settings.yml lives here/_data/
```

| File | Path | Purpose |
|---|---|---|
| `config.json` | `~/.config/opencode/config.json` | Provider, MCP, permissions |
| `AGENTS.md` (global) | `~/.config/opencode/AGENTS.md` | Behavior rules, machine-wide |
| `AGENTS.md` (project) | launch directory | Per-project overrides |
| LM Studio API | `http://192.168.1.194:1234/v1` | OpenAI-compatible endpoint |
| SearXNG | `http://192.168.1.101:8080` | Self-hosted metasearch |
