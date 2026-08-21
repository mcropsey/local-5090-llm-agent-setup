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
               "context": 65536,
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
         }
       }
     },
     "permission": {
       "bash": {
         "*": "ask",
         "ssh *": "allow",
         "scp *": "allow",
         "systemctl *": "allow",
         "journalctl *": "allow",
         "docker *": "allow",
         "podman *": "allow",
         "kubectl *": "ask",
         "aws *": "ask"
       }
     }
   }
   ```

   Config notes:
   - File is **`config.json`**, not `opencode.json` — many guides are wrong.
   - MCP schema is `mcp` / `type: "local"` / `command` as an **array**. The generic `mcpServers` + `command`/`args` shape won't load.
   - `"*": "ask"` — opencode shows a confirmation dialog before every bash command. You see the exact call, approve or deny.
   - `"allow"` entries — SSH, podman, docker run without prompting. Keep `aws` and `kubectl` on `"ask"` (destructive/costly).
   - The `mcp.searxng` block is already included. SearXNG setup is Phase 3.

5. **Verify opencode connects**
   ```bash
   mkdir ~/opencode-test && cd ~/opencode-test
   git init
   opencode
   ```
   Ask it: `"List the files in this directory."` — it should **run the tool** and return actual output, not hallucinate a file list.

   If you get `exceeds the available context size (8192 tokens)` → go back to LM Studio and set context to 64k, then reload the model.

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
  -v searxng-config:/etc/searxng:z \
  docker.io/searxng/searxng:latest
```

### 3b — Survive reboots

Unlike Docker, podman's `--restart` flag alone does **not** replay containers after a host reboot. Enable the system service that does:

```bash
sudo systemctl enable --now podman-restart.service
```

### 3c — Enable JSON output (the critical gotcha)

SearXNG ships with only HTML output. The MCP needs JSON. Skip this and every search returns HTTP 403 — no obvious error message.

Find the config path:
```bash
sudo podman volume inspect searxng-config
# look at the Mountpoint field
```

Edit settings.yml at that path (typically `/var/lib/containers/storage/volumes/searxng-config/_data/settings.yml`):
```yaml
search:
  formats:
    - html
    - json

server:
  secret_key: "change-me-to-something-random"
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
sudo podman ps                   # verify it's Up
sudo podman restart searxng
sudo podman volume inspect searxng-config   # find settings.yml path
```

| File | Path | Purpose |
|---|---|---|
| `config.json` | `~/.config/opencode/config.json` | Provider, MCP, permissions |
| `AGENTS.md` (global) | `~/.config/opencode/AGENTS.md` | Behavior rules, machine-wide |
| `AGENTS.md` (project) | launch directory | Per-project overrides |
| LM Studio API | `http://192.168.1.194:1234/v1` | OpenAI-compatible endpoint |
| SearXNG | `http://192.168.1.101:8080` | Self-hosted metasearch |
