# Free Web Search for opencode — Self-Hosted SearXNG + MCP

Goal: give the local model live web search with **no API key, no rate limits, no data leaving
your network**. SearXNG runs as a container on a Linux server in your env; opencode on the
laptop talks to it over the LAN via the `mcp-searxng` MCP server.

---

## How all the pieces fit together

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAPTOP                                                             │
│                                                                     │
│  ┌─────────────┐   MCP protocol   ┌──────────────────────────────┐ │
│  │   opencode  │ ───────────────► │  mcp-searxng                 │ │
│  │  (the CLI)  │ ◄─────────────── │  (Node.js process)           │ │
│  └─────────────┘   tool results   │                              │ │
│                                   │  started by opencode via:    │ │
│                                   │  npx -y mcp-searxng          │ │
│                                   │                              │ │
│                                   │  fetched from npmjs.com      │ │
│                                   │  on first launch, cached in  │ │
│                                   │  ~/.npm/_npx/                │ │
│                                   └──────────────┬───────────────┘ │
│                                                  │ HTTP            │
└──────────────────────────────────────────────────┼─────────────────┘
                                                   │
                                    LAN (e.g. 192.168.1.x)
                                                   │
┌──────────────────────────────────────────────────┼─────────────────┐
│  SERVER                                          │                 │
│                                   ┌──────────────▼───────────────┐ │
│                                   │  SearXNG container           │ │
│                                   │  port 8080                   │ │
│                                   │                              │ │
│                                   │  GET /search?q=...&format=json│ │
│                                   └──────────────┬───────────────┘ │
│                                                  │                 │
└──────────────────────────────────────────────────┼─────────────────┘
                                                   │ HTTPS
                                            public search
                                            engines (Google,
                                            Bing, DDG, etc.)
```

**What each piece does:**

| Component | Where it runs | What it is |
|---|---|---|
| opencode | Laptop | The AI coding CLI — orchestrates the model and MCP tools |
| mcp-searxng | Laptop (subprocess) | Node.js MCP server; translates opencode tool calls into SearXNG HTTP requests |
| npx | Laptop | Node.js package runner — downloads and runs mcp-searxng from npm automatically |
| SearXNG | Remote server | Self-hosted meta-search engine; fans out queries to real search engines and returns JSON |

---

## Prerequisite: Node.js must be installed on the laptop

`npx` is how opencode launches `mcp-searxng`. It's bundled with Node.js — if Node is
installed, you have `npx`. opencode itself often pulls Node in as a dependency, so it may
already be there.

Verify:

```bash
node --version    # e.g. v20.x.x
npx --version     # should just work if node does
which node        # shows where it came from
```

If `node` is missing, install it. Common ways:

```bash
# macOS
brew install node

# Debian/Ubuntu
sudo apt install nodejs npm

# Fedora/Rocky
sudo dnf install nodejs

# Or install a version manager (nvm, fnm) for more control
```

### How mcp-searxng gets onto your machine

You don't install it manually. The `npx -y mcp-searxng` in your config does it automatically:

1. On first opencode launch, opencode runs `npx -y mcp-searxng`.
2. `npx` checks whether the package is already cached.
3. If not, it fetches it from **npmjs.com** and caches it in `~/.npm/_npx/`.
4. It runs the cached copy. No `npm install`, no `git clone` needed.

The `-y` flag means "don't prompt, just download" — so it pulls silently.

**Supply-chain note:** `npx -y <package>` runs code from the internet that you didn't audit.
For this specific package:

- Confirm the npm listing points at `ihor-sokoliuk/mcp-searxng` on GitHub (not a typosquat):
  ```bash
  npm view mcp-searxng
  ```
- The config uses no version pin, so `npx` can silently pull a newer version on re-fetch.
  To lock it, pin explicitly:
  ```json
  "command": ["npx", "-y", "mcp-searxng@1.0.0"]
  ```
  Find the current version with `npm view mcp-searxng version`, then decide when to upgrade
  deliberately instead of automatically.

---

---

## Step 1 — Run SearXNG as a container on your server (rootful)

Run this **as root** (with `sudo`). We use **rootful** podman deliberately — it keeps all
storage paths predictable (`/var/lib/containers/...`), matches the system-level
`podman-restart.service` in Step 1b, and avoids the rootless linger/user-service complexity.
Run every `podman` command in this guide with `sudo`.

```bash
sudo podman run -d \
  --name searxng \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v searxng-config:/etc/searxng:z \
  docker.io/searxng/searxng:latest
```

- Pick a host port that's free on that box (8080 shown; adjust if taken).
- The named volume `searxng-config` persists config so your Step 2 edits survive restarts.
- `--restart=unless-stopped` sets the restart policy (needed for Step 1b below).
- `docker run` is identical if you're on Docker instead of podman.

> Consistency matters: because you created this container with `sudo`, you must **always**
> use `sudo` for `podman ps`, `podman exec`, `podman restart`, etc. A plain `podman ps`
> (rootless) will show an empty list and make it look like the container vanished — it
> didn't, you're just looking at a different storage namespace.

### Step 1b — Make the container survive a host reboot

**Gotcha:** unlike Docker, podman's `--restart` policy does NOT automatically start
containers after a *host reboot* on its own. The flag only handles the container crashing
or being stopped while the host stays up. To replay restart policies on boot, enable the
podman-restart systemd service:

```bash
sudo systemctl enable --now podman-restart.service
```

This **system** service starts, on boot, every rootful container with a `--restart` policy
of `always` or `unless-stopped`. Since we're running rootful (Step 1), this is exactly the
right service — no user-scoped service or `enable-linger` needed. Without it, SearXNG stays
down after the server reboots until you manually start it.

---

## Step 2 — Enable the JSON API (THE critical gotcha)

**SearXNG ships with only HTML output enabled. The MCP needs JSON. If you skip this,
every search returns HTTP 403 Forbidden** — not a silent failure, but also not an obvious
"enable JSON" error message. This is the #1 thing people miss.

### 2a — Find where the config actually lives

On first launch, SearXNG writes a default `settings.yml` into the `searxng-config` volume.
Because it's a *named volume* (not a bind mount to a path you chose), it lives in podman's
storage, not at an obvious location. Find the real host path:

```bash
sudo podman volume inspect searxng-config
```

Look at the **`Mountpoint`** field. For **rootful** podman it will be under
`/var/lib/containers/storage/volumes/`. The exact subdirectory name matches whatever you
passed to `-v` at `podman run` time — it's not always `searxng-config`. For example, if
you ran `-v searxng:/etc/searxng`, the path is:

```
/var/lib/containers/storage/volumes/searxng/_data/settings.yml
```

Always use `sudo podman volume inspect <volume-name>` to get the exact path rather than
guessing — the volume name varies by how the container was originally launched.

> If `settings.yml` isn't there yet, the container hasn't finished first-run init. Give it a
> few seconds after `podman run`, or check `sudo podman logs searxng`.

### 2b — Edit settings.yml

Two equally valid ways — pick one.

**Option A — edit on the host directly (simplest for rootful):**

```bash
sudo nano /var/lib/containers/storage/volumes/searxng-config/_data/settings.yml
```

**Option B — edit inside the container** (if you prefer not to touch storage paths):

```bash
sudo podman exec -it searxng sh
# then inside: vi /etc/searxng/settings.yml   (nano may not be installed)
```

Both point at the same file — `/etc/searxng/settings.yml` *inside* the container is the same
bytes as the `_data/settings.yml` path *on the host*, because the volume is mounted there.

### 2c — Make these two changes

Find the `search:` section and add `json` to `formats` (this is the fix that matters):

```yaml
search:
  formats:
    - html
    - json
```

And set a unique `secret_key` under `server:` (SearXNG warns loudly if left default):

```yaml
server:
  secret_key: "change-me-to-something-random"
```

### 2d — Restart to apply

```bash
sudo podman restart searxng
```

Config changes are only read at startup, so this restart is required for them to take effect.

---

## Step 3 — Verify SearXNG works BEFORE touching opencode

From the **laptop** (proves both JSON output and LAN reachability in one shot):

```bash
curl "http://<SERVER-IP>:8080/search?q=test&format=json"
```

- **JSON back** → SearXNG is good, move on.
- **HTML or an error about format** → JSON isn't enabled; redo Step 2.
- **Connection refused / timeout** → firewall or binding. Make sure the server's firewall
  allows inbound on 8080, and the container is published to the host's LAN interface, not
  just loopback.

Don't proceed until this curl returns JSON. Every downstream problem traces back to here.

---

## Step 4 — Register the MCP in opencode

Edit `~/.config/opencode/config.json` on the laptop. **Note: the file is `config.json`, not
`opencode.json`** — many guides show the wrong name. Add the `mcp` block (keep any existing
config like your LM Studio provider — just add alongside).

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "searxng": {
      "type": "local",
      "command": ["npx", "-y", "mcp-searxng"],
      "environment": {
        "SEARXNG_URL": "http://<SERVER-IP>:8080"
      }
    }
  }
}
```

- `SEARXNG_URL` is the **only required** variable — point it at the server's LAN address
  (not localhost; the container is on a different box).
- **Package name:** use the actively-maintained `ihor-sokoliuk/mcp-searxng`. There's an older
  `tisDDM/searxng-mcp` that is now deprecated — avoid it. **Verify the exact npm package name
  against the mcp-searxng GitHub before running**, since package names drift.
- **Config-format gotcha:** opencode uses `mcp` / `type: "local"` / `command` as an array.
  Most blog examples show the generic `mcpServers` / `command` + `args` shape — that won't
  load in opencode. If the server doesn't appear, this mismatch is almost always why.

---

## Step 5 — Restart opencode

MCP servers are read at startup — fully restart opencode, not just a new session. First run
of `npx -y mcp-searxng` downloads the package; give it a few seconds.

---

## Step 6 — Add the trigger rule to AGENTS.md

The MCP makes search *available*; this rule makes the local model actually *use* it instead
of answering from stale memory. Add to `~/.config/opencode/AGENTS.md` (global) or your
project `AGENTS.md`:

```markdown
## When to search the web
Your training data has a cutoff and is often out of date. Before answering anything about
current versions, package/CLI flags, library APIs, or how a public project is configured
today, assume your knowledge MAY be stale and use the searxng search tool to check.

Search — do not guess — whenever:
- You're about to state a version number, port, flag, or install command.
- The user mentions something released or changed recently.
- You're unsure whether what you "know" is still current.

Quote the search results. If results contradict your training data, trust the results and
say what changed.
```

### Gotcha: adding search rules can break SSH/shell behavior in local models

If you had SSH or shell tool use working before adding the search section, it may stop
working after. **This is not a config regression** — your permissions and SSH instructions
are still there. What happens is that smaller local models (e.g. Qwen3-Coder, Llama, Mistral)
apply their own built-in safety training on top of your AGENTS.md instructions. As the
system prompt grows longer, earlier instructions get less "weight" and the model's own safety
filter can start winning.

**Fix:** Make any shell/SSH instructions come first in AGENTS.md and use emphatic language:

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
```

Put this **before** the search section so it reads first. Then append the search rules after.

**Per-prompt workaround** if the model still refuses: prefix requests with `"Use bash and
SSH to..."` — naming the tool explicitly bypasses the safety filter even when the system
prompt alone doesn't.

### Sudo commands: make the model run them, not explain them

By default, local models will explain how to run sudo commands rather than running them. The
fix is two parts that work together:

**1. Tell the model in AGENTS.md to run sudo via bash:**

```markdown
This includes commands that require sudo. When a task needs sudo, run it via the bash
tool with sudo — do not explain how to run it manually. opencode will automatically prompt
the user for approval before executing any command; you do not need to ask permission
yourself first. Just call the tool and let opencode handle the confirmation dialog.
```

**2. The `config.json` permission `"*": "ask"` is already the right setting.** It means
opencode intercepts every bash command and shows you a confirmation dialog before running it.
You see the exact command, approve or deny, and only then does it execute. So the model runs
the sudo command → opencode asks you → you say yes → it runs. You get the confirmation step
without the model having to narrate it.

```json
"permission": {
  "bash": {
    "*": "ask",
    "ssh *": "allow"
  }
}
```

If you want specific safe commands to skip the prompt entirely, add them to the allow list
(e.g. `"sudo podman ps": "allow"`). The `"*": "ask"` catch-all covers everything else.

This same pattern extends to any CLI tool — AWS, kubectl, gcloud, etc. Add the tool to the
permission block and tell the model in AGENTS.md to run it directly:

```json
"permission": {
  "bash": {
    "*": "ask",
    "ssh *": "allow",
    "aws *": "ask",
    "kubectl *": "ask"
  }
}
```

```markdown
This also includes AWS CLI commands. When asked about AWS resources, run `aws` commands
directly via the bash tool. Do not explain how to run them manually. AWS credentials are
already configured on this machine. opencode will prompt for approval before executing.
```

Use `"ask"` for tools that can make destructive or costly changes (AWS, kubectl) so you
always see the exact command before it runs. Use `"allow"` only for read-only or
inherently safe commands you never need to review.

---

## Step 7 — Test end to end

In an opencode session:

```
Search the web for the current recommended way to run OWASP crAPI, and quote the results.
```

**Pass:** it calls the searxng tool and quotes real, current results.
**Fail:** it answers from memory without searching → revisit Step 6, or name the tool
explicitly ("Use searxng to search for…").

---

## Reality check (same as any local-model setup)

A local model **can't reliably tell when its own info is outdated** — stale facts feel as
confident as current ones (the same weakness behind the "container is running" hallucination).
So:

- The AGENTS.md rule deliberately **over-triggers** — search proactively on anything
  version/flag/config-shaped rather than self-detecting staleness (which it's bad at). Some
  unnecessary searches is the right trade.
- The **100%-reliable** move is to tell it per-prompt when *you* know something's
  time-sensitive: *"Search the web for the current X before you answer."*

---

## Optional: zero-setup version (for testing only)

If you just want to confirm this helps before hosting anything, `mcp-searxng` can auto-select
a **random public SearXNG instance** with no `SEARXNG_URL` at all. But public instances are
frequently rate-limited (HTTP 429) for programmatic use, so it's fine for a one-off test and
unreliable for daily work. Self-hosting (above) is the real answer.

---

## Troubleshooting quick map

| Symptom | Likely cause | Fix |
|---|---|---|
| curl returns HTTP 403 Forbidden | JSON format not enabled | Step 2 — add `json` to `search.formats`, restart |
| curl connection refused/timeout | firewall / container binding | Open 8080 inbound; publish to LAN interface |
| MCP tool never appears in opencode | wrong config schema | Use `mcp`/`type:local`/`command` array, not `mcpServers` |
| MCP loads but searches fail | `SEARXNG_URL` wrong or unreachable from laptop | Re-run the Step 3 curl from the laptop |
| Model has the tool but won't use it | no trigger rule | Step 6 AGENTS.md rule; or name the tool in-prompt |
| SSH/shell stops working after adding search | local model safety filter outweighing system prompt | Move SSH instructions to top of AGENTS.md with emphatic language (see Step 6 gotcha); or prefix prompts with "Use bash and SSH to..." |
| sudo commands fail with "terminal is required" | macOS sudo needs a TTY to prompt for password; opencode runs bash non-interactively | Run `sudo -v` in your terminal before the session to cache credentials (~5 min window) |
| Public instance 429s | rate-limited public instance | Self-host (don't rely on public) |

---

## Quick reference

```bash
# On the server (rootful — use sudo for ALL podman commands):
sudo podman run -d --name searxng --restart=unless-stopped -p 8080:8080 \
  -v searxng-config:/etc/searxng:z docker.io/searxng/searxng:latest
sudo systemctl enable --now podman-restart.service   # survive host reboots

# Find + edit config (path from: sudo podman volume inspect searxng-config):
sudo nano /var/lib/containers/storage/volumes/searxng-config/_data/settings.yml
#   search.formats: add  - json
#   server.secret_key: set something random
sudo podman restart searxng

# From the laptop (must return JSON):
curl "http://<SERVER-IP>:8080/search?q=test&format=json"

# ~/.config/opencode/config.json  -> mcp.searxng  (SEARXNG_URL = http://<SERVER-IP>:8080)
# AGENTS.md      -> "When to search the web" rule
# restart opencode, then: "Search the web for <current thing> and quote it."
```
