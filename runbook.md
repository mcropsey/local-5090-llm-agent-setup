# Local Agentic Stack — Ground-Up Runbook

Build a fully local coding agent from scratch: a GPU box serving an LLM, a laptop
running the [opencode](https://opencode.ai) harness, and a self-hosted SearXNG search
engine — no cloud model, no API key, nothing leaving your LAN.

Follow this top to bottom on fresh machines. Nothing here assumes you already have any
config; every file you need is written out in full, in the order you need it.

---

## 0 — Before you start

### 0.1 What you need

**Three machines** (they can be three physical boxes, or VMs — they just need to reach
each other on the same LAN):

| Machine | OS | Role | Must have |
|---|---|---|---|
| GPU box | Windows/Linux/macOS | Serves the model | A GPU with enough VRAM (see below) + [LM Studio](https://lmstudio.ai) |
| Laptop | macOS (Linux works too) | Runs the opencode harness | `curl`, `git`, and Node.js (for `npx`) |
| Search host | Linux (this guide uses Rocky) | Runs SearXNG | `podman` (or Docker) |

> **VRAM guide:** a 30B-class MoE model (e.g. Qwen3-Coder-30B-A3B) at Q4 is ~18–21 GB
> at 64k context, which fits a 32 GB card with room for the KV cache and nothing else.
> On 24 GB, drop to a ~20–24B model or a shorter context. On 16 GB, expect to run a
> smaller model at reduced context.

**Software to install first** (do this before Phase 1):

- **GPU box:** Install LM Studio and launch it once.
- **Laptop:** Confirm `curl` and `git` exist (`curl --version`, `git --version`). Install
  Node.js if `npx --version` fails — the search MCP is fetched via `npx`.
- **Search host:** Install podman (`sudo dnf install -y podman` on Rocky/RHEL, or
  `sudo apt install -y podman` on Debian/Ubuntu). Docker works too; substitute `docker`
  for `podman` throughout Phase 3.

### 0.2 Fill in your values

This guide is written with **reference values** (a specific set of IPs, a username, and a
model). If your setup matches them, every command below is copy-paste ready. If not,
decide your values now and either substitute as you go or find-and-replace them in this
file first.

| Placeholder | Reference value | What it is |
|---|---|---|
| `USER` | `mcropsey` | The Linux/SSH login on your search host (and any server the agent reaches) |
| `GPU_HOST` | `192.168.1.194` | LAN IP of the GPU box running LM Studio |
| `SEARX_HOST` | `192.168.1.101` | LAN IP of the SearXNG host |
| `LAB_HOST` | `192.168.1.102` | Optional extra host for lab MCP servers (skippable) |
| `MODEL_ID` | `qwen/qwen3.8-27b` | The exact model ID LM Studio serves (you confirm this in Phase 1) |

> **Don't invent the model ID.** You'll read the real one off the server in Phase 1
> step 5 and use it verbatim from then on. The reference `qwen/qwen3.8-27b` is only an
> example — whatever `GET /v1/models` returns on *your* box is the truth.

### 0.3 The machines at a glance

| Machine | IP (reference) | Role |
|---|---|---|
| Laptop (macOS) | — | opencode harness, SSH client, AWS CLI |
| GPU box (Windows) | `192.168.1.194` | LM Studio — serves the model |
| Search host (Rocky Linux) | `192.168.1.101` | SearXNG container (rootful podman) |

---

## Phase 1 — Serve the model (GPU box)

**Tooling:** LM Studio (GUI). Install it and open it once before starting.

1. **Enable Developer mode**
   Settings → Developer → toggle **Developer mode** ON. This unlocks the Developer tab.

2. **Download and load a tool-capable model**
   Discover tab → download a model that can make tool calls (this is non-negotiable —
   opencode is entirely tool-driven). Good starting choices: **Qwen3-Coder-30B-A3B**, a
   **Qwen3.x-27B**, or **Devstral Small 24B**. Avoid pure reasoning models like
   DeepSeek-R1 as the driver (see "Which models can drive the harness" below).

   When loading, set these on the **Load** tab (not Inference):
   - Context length (`n_ctx`): **65536** (64k) — start here
   - Flash Attention: **ON**
   - KV Cache Quantization: **Q8** (K and V)
   - Quant: **Q4_K_M or Q5** — below Q4, tool-calling reliability degrades

   > **VRAM math:** a 30B-A3B at Q4 ≈ 18 GB, leaving ~13 GB for KV cache on a 32 GB card.
   > Q8 cache quant lets 64k context fit comfortably. Start at 64k; drop to 32k if it's
   > tight. If you have headroom (e.g. a 27B VLM on 32 GB), you can load at 96k instead —
   > just make sure the number you actually load at matches what you put in `config.json`
   > later (Phase 2, and the `limit.context` note there).

   > **The whole stack is model-agnostic.** Any tool-capable GGUF works. Wherever this
   > guide names a specific model, substitute whichever ID `GET /v1/models` returns on
   > your box.

3. **Start the server**
   Developer → Local Server → toggle **Status: Running** (port `1234` by default).

4. **Enable LAN access**
   Gear icon → **Serve on Local Network: ON**. Leave *Require Authentication* OFF — this
   is a LAN-only setup. (Revisit before any external exposure.)

5. **Load exactly one model, and turn JIT loading OFF**
   Settings → Developer → turn off **Just-In-Time (JIT) model loading**, and set **idle
   TTL** to off (or cap **max loaded models** at 1). Then load your model once, manually,
   and leave it resident.

   Why this matters — both defaults cause the two weirdest failure modes on this stack:

   | Setting left ON | Symptom |
   |---|---|
   | **JIT model loading** | *Any* API request naming an unloaded model silently loads it. opencode fires a title-generation call concurrently with your first message, and JIT answers it by loading a **second full copy** of the model. Two copies spill VRAM to system RAM, dropping you from ~65 tok/s to ~1 tok/s — which looks exactly like opencode hanging. |
   | **Idle TTL auto-unload** | The model silently unloads after inactivity. Your next request pays a 30–90 s cold reload before the first token — indistinguishable from a hang. |

   With JIT off and one model pinned, load state is something you control instead of a
   side effect of whatever last hit the API.

6. **Verify from the laptop**
   ```bash
   curl http://192.168.1.194:1234/v1/models
   ```
   This should return a JSON list containing the model you loaded. **Note the exact model
   ID — publisher prefix and all** (e.g. `qwen/qwen3.8-27b`). You'll need it *verbatim* in
   Phase 2; this is your `MODEL_ID`.

   The first request to a freshly loaded model is slow (30–90 s of VRAM loading). That's
   not a hang.

---

## Phase 2 — Install and configure opencode (laptop)

1. **Install opencode**
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

3. **Give the agent key-based SSH + passwordless sudo on the search host**

   The agent runs commands non-interactively — it can't answer a password prompt. So it
   needs (a) SSH key auth and (b) passwordless sudo for podman on the search host.

   **SSH key auth:**
   ```bash
   ssh-keygen -t ed25519                     # skip if you already have a key
   ssh-copy-id mcropsey@192.168.1.101        # USER@SEARX_HOST
   ssh mcropsey@192.168.1.101 hostname       # must return with NO password prompt
   ```

   **Passwordless sudo for podman** — run these *on the search host, as root*:
   ```bash
   echo 'mcropsey ALL=(ALL) NOPASSWD: /usr/bin/podman' > /etc/sudoers.d/podman-agent
   chmod 440 /etc/sudoers.d/podman-agent
   visudo -c    # verify no syntax errors
   ```
   (Replace `mcropsey` with your `USER`.) Then verify from the laptop — must return with
   no prompt:
   ```bash
   ssh mcropsey@192.168.1.101 'sudo -n podman ps'
   ```

   > Without passwordless sudo, the agent's `sudo podman ps` fails silently and it falls
   > back to *rootless* `podman ps`, which returns an empty list — making SearXNG look
   > gone when it's actually running fine under rootful podman.

4. **Create the opencode config directory and `config.json`**

   You don't have a config yet — create the directory and the file now:
   ```bash
   mkdir -p ~/.config/opencode
   ```

   Then write `~/.config/opencode/config.json`. **Substitute your `MODEL_ID` (from
   Phase 1 step 6) everywhere the reference `qwen/qwen3.8-27b` appears, and your host IPs
   for the reference ones.**

   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "model": "local5090/qwen/qwen3.8-27b",
     "provider": {
       "local5090": {
         "npm": "@ai-sdk/openai-compatible",
         "name": "LM Studio GPU box",
         "options": {
           "baseURL": "http://192.168.1.194:1234/v1",
           "apiKey": "not-needed"
         },
         "models": {
           "qwen/qwen3.8-27b": {
             "name": "Qwen3.8 27B (default)",
             "tools": true,
             "limit": {
               "context": 98304,
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
           "Authorization": "Basic <base64-of-user:password>"
         },
         "enabled": false
       }
     },
     "permission": {
       "bash": "ask"
     }
   }
   ```

   **Config notes — read these, they're the difference between working and not:**

   - **The provider key `local5090` is just a label** — name it whatever you like, but
     it becomes the prefix in your model string (`local5090/<MODEL_ID>`) and in
     `opencode -m <prefix>/<MODEL_ID>`. Keep it consistent everywhere.
   - **`model` and the `models` block must use your real `MODEL_ID`.** The reference shows
     `qwen/qwen3.8-27b`; yours is whatever Phase 1 step 6 returned. A slash in the ID is
     fine — opencode splits `provider/model` on the *first* slash only, so
     `local5090/qwen/qwen3.8-27b` resolves correctly.
   - **`limit.context` must be LM Studio's reported `loaded_context_length`, not the
     number you typed into the GUI.** Ask for 64k and it reports `65280`, not `65536`; a
     27B loaded at 96k reports exactly `98304`. opencode uses this number to decide when
     to compact — set it even 256 tokens too high and requests near the top of the window
     get rejected with `Internal Server Error`. Read the real number off your box:
     ```bash
     curl -s http://192.168.1.194:1234/api/v0/models | python3 -c "
     import json,sys
     for m in json.load(sys.stdin)['data']:
         if m.get('state')=='loaded':
             print(f\"{m['id']:38s} loaded_ctx={m.get('loaded_context_length')}\")
     "
     ```
     Put that number in `limit.context`. Too low only costs you early compaction; too high
     is the `Internal Server Error`.
   - **`"bash": "ask"` is the entire permission block.** It takes a bare string, so you
     don't need per-command patterns. Every command prompts you for approval — including
     `sudo`, `rm`, anything — and nothing is pre-approved or blocked. Approve at the
     prompt.
   - **Only `searxng` is enabled; `noname` and `crapi` ship disabled.** They're optional
     lab MCP servers (they point at `LAB_HOST` and add ~68 tool definitions / ~8,000
     prompt tokens to *every* request). **You can delete both entries entirely** if you
     don't have those services — the stack works fine with just `searxng`. If you do use
     `crapi`, generate its auth header yourself rather than reusing anyone's:
     ```bash
     printf 'user:password' | base64        # → paste into the Authorization header
     ```
   - **The file is `config.json`.** opencode also accepts `opencode.json` /
     `opencode.jsonc` in the same directory and merges all of them; this guide uses
     `config.json` throughout, so if you follow other guides that say `opencode.json`,
     just pick one name and stay consistent.

5. **(Optional) Add keybinds for switching models — `tui.json`**

   If you want to change models from inside the TUI, create
   `~/.config/opencode/tui.json`:
   ```json
   {
     "$schema": "https://opencode.ai/tui.json",
     "keybinds": {
       "model_list": "ctrl+alt+m",
       "model_cycle_recent": "ctrl+alt+."
     }
   }
   ```
   `model_list` opens the model picker; `model_cycle_recent` cycles recently used models.
   (Recent opencode versions ship defaults for these — `<leader>m` and `f2` — but binding
   them explicitly here means you don't depend on the version.) You can skip this entirely
   and just set the default model via `config.json`'s top-level `"model"`.

6. **Verify opencode connects**
   ```bash
   mkdir ~/opencode-test && cd ~/opencode-test
   git init
   opencode
   ```
   Ask it: `"List the files in this directory."` — it should **run the tool** and return
   actual output, not hallucinate a file list.

   If you get `exceeds the available context size (8192 tokens)`, opencode is talking to a
   model loaded at the 8k default — go back to LM Studio, load your model at 64k (or your
   chosen size), and confirm `limit.context` matches.

---

## Switching models — and why loading one in LM Studio isn't enough

**Loading a model in LM Studio does not make opencode use it.** opencode only ever
requests the model IDs declared in its `provider.<name>.models` block, and it defaults to
`config.json`'s top-level `"model"`. Load a different model in the GUI, and opencode still
asks for the configured one — LM Studio then has to JIT-load *your configured* model
alongside the one you just loaded, which on a 32 GB card can fail outright:

```
HTTP 400  Failed to load model "<your-model-id>".
          Error: Engine protocol startup was aborted.
```

That 400 is what "opencode won't connect" actually looks like — and it's intermittent,
because it depends on what else is resident at that moment.

To actually switch models you must **declare the model in config, then select it.** Add
each tool-capable model you want to use as an entry in the `models` block. Example with
several declared:

```json
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
  "devstral-small-2-24b-instruct-2512": {
    "name": "Devstral Small 2 24B",
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

> `limit.context` is **per model, not global.** The `65280` above is what LM Studio
> reports for a 64k load; a model loaded at a different context needs its own number
> (e.g. `98304` for 96k). Read each one off `/api/v0/models` as shown in Phase 2 step 4
> — don't copy a neighbour's number.

Then pick a model three ways:

| How | Command / key | Scope |
|---|---|---|
| Default | top-level `"model": "local5090/<MODEL_ID>"` in config.json | every session |
| Per session | `opencode -m local5090/<MODEL_ID>` | that launch only |
| Mid-session | the keybinds you set in `tui.json` (Phase 2 step 5) | that session |

**Confirm opencode can see a model before launching the TUI:**
```bash
opencode models local5090
```
**If it isn't in that list, opencode cannot use it — full stop.** There's no
auto-discovery of LM Studio's loaded models; the `models` block in config.json is the
*complete* universe of what opencode will request. This is the single most common reason
for "I loaded it but opencode won't use it."

> **`opencode models` reads config.json, not the box.** It will happily list a model you
> deleted from the GPU box. Cross-check against what's actually loaded:
> ```bash
> curl -s http://192.168.1.194:1234/v1/models | python3 -c \
>   "import json,sys; [print(m['id']) for m in json.load(sys.stdin)['data']]"
> ```
> Selecting a declared-but-absent model triggers the JIT-load `HTTP 400` above (or, with
> JIT off, a plain model-not-found).

> **Always use the full model ID, publisher prefix included.** LM Studio will answer a
> request for a short name (e.g. `qwen3.8-27b` instead of `qwen/qwen3.8-27b`) with HTTP
> 200 — which makes the shortcut *look* safe — but it treats the short name as a separate
> model and JIT-loads a second copy at the **8192 default context**. You end up with two
> copies in VRAM, one of them crippled at 8k. Match the ID from `/api/v0/models` exactly.

### Which models can actually drive the harness

opencode needs `tool_use`. Ask LM Studio rather than guessing — it publishes capabilities
per model:

```bash
curl -s http://192.168.1.194:1234/api/v0/models | python3 -c "
import json,sys
for m in json.load(sys.stdin)['data']:
    if m.get('type')=='embeddings': continue
    caps=m.get('capabilities') or []
    print(f\"{m['id']:42s} {'tool_use OK' if 'tool_use' in caps else 'no tool_use'}\")
"
```

Rules of thumb:

- **Qwen3-Coder / Qwen3.x / Devstral / GPT-OSS** advertise `tool_use` and drive the
  harness well.
- **Reasoning-only models (e.g. DeepSeek-R1-8B) can't tool-call** — they *type* a
  markdown code fence as ordinary text instead of emitting a structured call. Same prompt,
  measured side by side: a Qwen coder returns a real `tool_calls` entry; DeepSeek-R1
  returns `finish_reason: stop`, zero tool calls, and burns its whole output budget
  "thinking" (it needed ~380 tokens just to answer "what is 2+2?"). Its text also lands in
  `reasoning_content`, which `@ai-sdk/openai-compatible` doesn't read, so opencode shows a
  blank reply. Keep such models for direct Q&A; don't use them as the driver.
- **Capability metadata is only populated once a model has been loaded at least once.** A
  never-loaded model may report nothing — treat "no capabilities" on a never-loaded model
  as *unknown*, not a definite no.

---

## Phase 3 — SearXNG on the search host

SearXNG gives the model live web search with no API key and no data leaving the LAN. Run
everything with `sudo` — this is **rootful** podman deliberately (predictable storage
paths, survives reboots via a system service).

### 3a — Run the container

SSH to the search host, then:
```bash
sudo podman run -d \
  --name searxng \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v searxng:/etc/searxng:z \
  docker.io/searxng/searxng:latest
```

> **The volume name is `searxng`**, not `searxng-config`. A wrong name creates a second
> orphan volume and leaves the config empty.

### 3b — Survive reboots

Unlike Docker, podman's `--restart` flag alone does **not** replay containers after a host
reboot. Enable the system service that does:
```bash
sudo systemctl enable --now podman-restart.service
```
It starts every container whose restart policy is `always` or `unless-stopped`. Verify:
```bash
systemctl is-enabled podman-restart.service   # should print: enabled
systemctl is-active  podman-restart.service   # should print: active
```

### 3c — Enable JSON output (the critical gotcha)

SearXNG ships with **only HTML** output. The MCP needs JSON. Skip this and every search
returns HTTP 403 with no obvious error.

Find the config path:
```bash
sudo podman volume inspect searxng --format '{{.Mountpoint}}'
# returns something like: /var/lib/containers/storage/volumes/searxng/_data
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

- **JSON back** → good, proceed.
- **HTML or 403** → JSON not enabled; redo 3c.
- **Connection refused** → firewall or binding issue; confirm port 8080 is open on the
  search host and the container published to the LAN interface.

Don't skip this check. Nearly every downstream search problem traces back to SearXNG not
returning JSON.

### 3e — The search MCP installs itself

`mcp-searxng` is already in the `config.json` you wrote in Phase 2. On first opencode
launch it runs `npx -y mcp-searxng`, which downloads the package from npm and caches it in
`~/.npm/_npx/`. No manual install.

Supply-chain note: `npx -y` runs unaudited npm code. Before relying on it, confirm the
package resolves to the real project (not a typosquat):
```bash
npm view mcp-searxng
# should point at ihor-sokoliuk/mcp-searxng on GitHub
```
To pin a version, change the command in config.json to
`["npx", "-y", "mcp-searxng@1.0.0"]`.

---

## Phase 4 — AGENTS.md (behavioral control plane)

Create `~/.config/opencode/AGENTS.md`. It's loaded into every system prompt and is what
makes a small local model actually use its tools instead of refusing.

**Order is load-bearing.** Local models weight earlier instructions more heavily. The
tool-access block (the CRITICAL section) **must come first**, or the model's built-in
safety training wins and it starts refusing SSH/sudo calls — especially as the file grows.

Paste this as your starting `AGENTS.md`, then edit the "Known servers" line and any
usernames to match your setup:

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
  (searxng search tool) instead of guessing image names, ports, or flags.
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
- 192.168.1.101 (search host): Linux podman host, lab network only.

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

> Replace `mcropsey` with your `USER` and the `192.168.1.101` line with your search
> host. Keep the CRITICAL block at the very top no matter what else you add later.

---

## Phase 5 — End-to-end verification

Run these in order — each isolates a different layer.

```bash
# 1. Model reachable
curl http://192.168.1.194:1234/v1/models

# 2. SearXNG reachable + JSON enabled
curl "http://192.168.1.101:8080/search?q=test&format=json"

# 3. SSH key auth working
ssh mcropsey@192.168.1.101 hostname   # no password prompt
```

Then, inside opencode:
```
# 4. Tool-calling works
"List the files in this directory."
→ should run ls, not hallucinate a list

# 5. SSH + rootful podman works
"SSH to 192.168.1.101 and show me what's running in rootful podman."
→ should run `sudo podman ps` and return SearXNG in the list
→ if it runs plain `podman ps` and gets an empty list, it reached rootless —
  the Phase 2 step 3 sudo setup is missing

# 6. Web search works
"Search the web for the current recommended way to run OWASP crAPI, and quote the results."
→ should call the searxng tool and quote real results

# 7. AWS works (only if you use AWS)
"Use your bash tool to run `aws sts get-caller-identity` and show me the actual output."
→ should call the tool; opencode prompts you to approve first
```

When all seven pass, the stack is up.

---

## Operational gotchas

### sudo fails with "a terminal is required" (laptop local sudo only)

opencode runs bash non-interactively (no TTY). macOS sudo refuses to prompt for a password
without one. This affects **local sudo on the laptop** — not remote sudo on the search
host, which is handled by the NOPASSWD rule in Phase 2 step 3.

**Fix:** before any session that needs local sudo, run once in your terminal:
```bash
sudo -v
```
Caches credentials for ~5 min; re-run if the session runs long. Your AGENTS.md already
tells the model to prompt you for this instead of giving up.

### SSH stops working after you add content to AGENTS.md

Adding new sections pushes the CRITICAL block further from the top, and the local model's
safety training starts winning.

**Fix:** keep the CRITICAL block at the very top. If SSH breaks right after an AGENTS.md
edit, this is almost always why. Per-prompt workaround: prefix with "Use bash and SSH
to…" — naming the tool explicitly bypasses the filter.

### Podman lock error

If you see `acquiring lock 0 … file exists`, the container is wedged — not just a warning:
```bash
sudo podman rm -f <container-id>
sudo podman stop --all
sudo podman system renumber    # rebuilds the lock table
```

### Model claims success it didn't verify

Classic small-model over-claiming. The AGENTS.md Verification block counters it. Also end
prompts with an explicit check: "then run `podman ps` and show me it's actually Up before
claiming success."

### Empty response after "thinking"

Tool-call formatting fumble — usually the wrong chat template for the loaded GGUF. Confirm
LM Studio is using the model's native tool-use chat template, not a generic one.

---

## LM Studio reboot note (GPU box)

LM Studio does **not** auto-start the local server after a reboot. After any reboot of the
GPU box:

1. Open LM Studio.
2. Load the model (My Models → your model → Load at the **same context** your
   `config.json` `limit.context` assumes — load it at a different size and you must edit
   that entry to match), FlashAttn ON, Q8 KV.
3. Developer → Local Server → toggle **Status: Running**.
4. Verify from the laptop: `curl http://192.168.1.194:1234/v1/models`.

> LM Studio has a **Launch at Login** option (Settings → General) but it only opens the
> GUI — it does not reload the model or start the server. On Windows you *can* build a Task
> Scheduler task that calls the LM Studio CLI with a saved preset on logon, but CLI support
> for this is limited. Simplest path: treat the GPU box as a manual step after any reboot.

---

## Optional — hybrid frontier escape hatch

For the occasional task too hard for the local model, keep a second provider in
config.json:

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

Set `ANTHROPIC_API_KEY` in your environment, then switch with
`--model frontier/claude-sonnet-5` for the hard 5%; local for everything else.

---

## Quick reference

```bash
# One-time per server the agent should reach:
ssh-copy-id USER@<ip>
ssh USER@<ip> hostname          # verify: no password prompt

# Before any session needing local sudo (laptop):
sudo -v

# Wedged podman containers (on the search host):
sudo podman rm -f <id>
sudo podman stop --all
sudo podman system renumber

# SearXNG container management (all with sudo, on the search host):
sudo podman ps                                                  # verify it's Up
sudo podman restart searxng
sudo podman volume inspect searxng --format '{{.Mountpoint}}'  # settings.yml lives here/_data/

# Read the real loaded context length off the GPU box:
curl -s http://GPU_HOST:1234/api/v0/models | python3 -c "
import json,sys
for m in json.load(sys.stdin)['data']:
    if m.get('state')=='loaded': print(m['id'], m.get('loaded_context_length'))
"
```

| File | Path | Purpose |
|---|---|---|
| `config.json` | `~/.config/opencode/config.json` | Provider, MCP, permissions |
| `tui.json` | `~/.config/opencode/tui.json` | TUI keybinds (optional) |
| `AGENTS.md` (global) | `~/.config/opencode/AGENTS.md` | Behavior rules, machine-wide |
| `AGENTS.md` (project) | launch directory | Per-project overrides |
| LM Studio API | `http://GPU_HOST:1234/v1` | OpenAI-compatible endpoint |
| SearXNG | `http://SEARX_HOST:8080` | Self-hosted metasearch |

**Reference values used in this guide** — substitute your own:
`USER=mcropsey`, `GPU_HOST=192.168.1.194`, `SEARX_HOST=192.168.1.101`,
`LAB_HOST=192.168.1.102`, `MODEL_ID=qwen/qwen3.8-27b`.
