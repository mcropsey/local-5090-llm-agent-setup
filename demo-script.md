# Local Agentic Stack — Demo Notes (v3)

Repo: https://github.com/mcropsey/local-5090-llm-agent-setup
Target: **~24 min.** Deck slide numbers in brackets.

---

## Take 1 fixes — read first

| Problem | Fix |
|---|---|
| 4 min hunting the LM Studio server toggle | Path is in Act 4. Know it cold |
| Old chats visible | Delete before recording |
| Called the Claude Code question "dumb" | It's the best question in the video. Script in Act 3 |
| Forgot "harness" in the recap | Four words on screen [3]. Read them |
| Shrugged at the context number | One line of VRAM math, Act 4 |
| npx/node ramble (90 s) | One sentence, Act 7 |
| Name mumbled | Slow. On screen |
| SearXNG pronounced three ways | Say once: "SearXNG — S-E-A-R-X-N-G" |
| Unsure if httpd was already on .98 | Pre-flight below |
| "my local Claude as a CLI…" | "Claude Code is the CLI. opencode is the open-source equivalent" |

**Pre-flight (5 min before):**
```bash
curl -s http://192.168.1.194:1234/v1/models                              # LM Studio up, ONE model
curl -s "http://192.168.1.101:8080/search?q=test&format=json" | head -c 100   # JSON back
ssh mcropsey@192.168.1.98 'rpm -q httpd'                                  # expect: not installed
ssh mcropsey@192.168.1.101 'sudo -n podman ps'                            # no password prompt
```
Also: `noname` and `crapi` set to `false` in config.json (reference notes noname was left on).

---

# 1 — WHO / WHAT / WHY (2:00) [2]

- Name, slowly.
- Pitch: a local agent that does what Claude Code does, on hardware I own. Less capable. Very capable for lab work.
- What I use it for: configure servers, install/remove packages, re-IP, read logs. Runs on each Linux box, or from the laptop over SSH — I do both.
- Why not just Claude Code: $20 plan, hit the limit, wait. Claude Code for the hard stuff (big Terraform, lots of moving parts). Local for basic AWS — spin up an EC2, a couple of resources. Slower, free. That's the trade.
- **Hardware b-roll.** Keep the wife joke. Don't cut it.

# 2 — WHAT'S WHERE (3:00) [4]

- Laptop (macOS): opencode + mcp-searxng subprocess. MCP doesn't have to live here — convenience.
- 5090 (.194): LM Studio. Doesn't have to be LM Studio.
- Rocky 9 (.101): SearXNG in podman. Rocky 9 not 10 — Akamai sensors. Ten seconds, move on.
- Why search exists: a downloaded model is frozen at its cutoff. SearXNG is its way out. MCP is how it gets there.
- Land it: one HTTP call and one SSH session hold the whole thing together.

# 3 — FOUR PARTS + CLAUDE CODE (3:00) [3] → [7]

Read off the screen:

| Brain | model | Qwen3.8 27B on the 5090 |
|---|---|---|
| **Hands** | harness | opencode, laptop |
| **Senses** | tools | SearXNG via MCP, bash, SSH |
| **Personality** | instructions | `AGENTS.md` |

config.json isn't a fifth part — it's the wiring.

Then [7]:
> "The obvious question: isn't this just Claude Code? Fair question. Claude Code is the better agent — frontier model. What's different is the brain and the hands. Claude Code is Anthropic's CLI; opencode is the open-source equivalent that points at any OpenAI-compatible endpoint. MCP is identical on both sides. `AGENTS.md` instead of `CLAUDE.md`."

- I run both — Claude Code with MCP against product management APIs and a deliberately vulnerable API app.
- Land it: "Different jobs. I use both."

# 4 — LM STUDIO (4:00)

Chats deleted. Model selected.

**4.1 Load** — tried a few, settled on this one. Skip the model browser.

**4.2 Context — the one setting that matters**
- Default is **8k**. Set **96k (98304)**. Flash Attention ON. KV cache Q8.
- Why (one line): "32 GB card. Weights at Q4 take roughly 17. What's left is the KV cache — the agent's short-term memory. At Q8 with Flash Attention, that buys ~96k."
- Failure symptom: leave it at 8k and it dies on anything non-trivial — every tool result gets pasted back into context.
- GPU offload: spill to CPU and it crawls.

**4.3 One model, JIT off** (new — biggest "it's hanging" cause)
- Settings → Developer → **JIT model loading OFF**, idle TTL off.
- Why: opencode fires a title-gen call alongside your first message. JIT answers it by loading a second copy. Two copies spill to CPU: 24 s for 8 tokens instead of 65 tok/s.

**4.4 Start the server — say it in this order:**
1. Settings → Developer → **Developer mode ON**. Nothing appears until you do this.
2. Developer tab → Local Server → **Status: Running** (port 1234).
3. Gear → **Serve on Local Network ON**. Without it, localhost only.
4. Require Authentication OFF — lab. "Turn it on if you leave your LAN."

**Act break, from the laptop:**
```bash
curl http://192.168.1.194:1234/v1/models
```
> "That's the entire contract between these two machines. One URL."

- Reboot tax: server doesn't come back after a Windows reboot. Manual every time.

# 5 — OPENCODE + SSH (2:00)

- Install: one curl. Add to PATH. Done.
- SSH: passwordless keys. **Keep the trick:** once opencode runs, tell it *"log into these servers with this ID and set up key-based auth."* It does keygen and copy.
- If stuck on passwords: `SSH_ASKPASS` from a keychain, or `sshpass -f` with a 600 file. Never `-p` — lands in `ps` and history. One sentence each.

# 6 — CONFIG.JSON (2:00) [5]

Three blocks, three questions:
- **provider** — which brain. `baseURL`, model ID, `limit.context`.
  - `limit.context` must equal what LM Studio *reports* (`98304`), not what you typed. 256 too high → `Internal Server Error` on long sessions.
- **mcp** — which senses. searxng on; noname/crapi off.
  - Every enabled server's tool schema rides on every request, even "what is 2×2". 4 tools → 2,155 tokens. 72 tools → 10,121, first token 3.4× slower.
- **permission** — what asks me. One line: `"bash": "ask"`. Every command stops for a yes. Evaluated locally — model never sees it, 0 tokens.

Two lines to say:
1. > "I decided what the agent can do ahead of time, in a file, while I was calm."
2. Lab caveat: don't run allow-everything in production. That's what Ansible and change control are for.

Optional 15 s: could add a Claude provider here as a fallback. Haven't. Say so.

# 7 — SEARXNG (3:00)

- Live web search. No API key. Self-hosted, nothing leaves the LAN. Lab setup — production wants auth.
- Steps:
  1. `sudo podman run` — podman because it's the RHEL default.
  2. Reboot survival ≠ Docker: `sudo systemctl enable --now podman-restart.service`
  3. **JSON output** — `settings.yml`, add `json` under `search: formats:`, set a real `secret_key`, restart. Skip it and the MCP gets a 403 with no useful error.
  4. Verify from the laptop **before** touching opencode:
     ```bash
     curl "http://192.168.1.101:8080/search?q=test&format=json"
     ```
- Prereq, one line: "Node.js — `mcp-searxng` is a JavaScript package, `npx` runs it."
- Rootful: created with sudo, so every podman command needs sudo. Rootless `podman ps` shows empty — looks deleted. It isn't.

# 8 — AGENTS.MD (3:00) [6]

> "`config.json` is what it *can* do. `AGENTS.md` is what it *will* do. You need both."

Walk the blocks:
- **CRITICAL / tool access** — "you have shell and SSH, including sudo. Run it, don't explain it." If you don't say this, it won't do it.
- **Truthfulness + verification** — only report output a tool returned; check state before claiming success. No config setting for this.
- **Loop discipline** — one command, read result, decide. No menus of six.
- **Grounding / search** — before stating a version, port, or flag: search, don't guess.
- **Cloud defaults** — never 0.0.0.0/0.
- Known servers.

**Search-order point (30 s, strongest bit):**
> "Where you put the rule matters. Put search at the end and the model may never reach for it. Perplexity searches first, always. ChatGPT sometimes does. Same problem — here you decide."

Optional 60 s: move the CRITICAL block to the bottom, restart, ask it to SSH. It refuses and hands you commands. Move it back. Only position changed.

# 9 — WHEN IT BREAKS (0:30) [8]

Set expectations before going live. It's always one of three:
- **Brain** — 8k context, or two models in VRAM. Looks exactly like hanging.
- **Personality** — CRITICAL block slid down the file.
- **Senses** — JSON off in SearXNG.
- Almost never the hands. Check the 5090 before you touch config.

# 10 — DEMOS (4:00) [9]

**1 — It can act**
```
Log in to 192.168.1.98 and install Apache.
```
- Leave the error in. "It hit an error and fixed itself. That's the loop. Claude Code does the same."
- Verify in a browser. Don't take its word for it.

**2 — It can see**
```
Search the web for the latest news about API security events.
```
- That's SearXNG doing the work.

**3 — It knows it's stale — best demo, don't rush**
```
What are the requirements for the most recent RHCSA exam?
```
- Raw model → RHEL 9 (training cutoff). Through opencode → current.
- Show both if you can. Side by side is the whole argument in 15 seconds.

# 11 — CLOSE (0:30) [10]

- Repo URL on screen 5+ s. Pin in description.
- Runbook to build it. Reference for when it breaks. mcp-servers for turning tools on and off.
- Thanks.

---

## Timing

| Act | Min |
|---|---|
| 1 Who/what/why | 2:00 |
| 2 What's where | 3:00 |
| 3 Four parts + Claude Code | 3:00 |
| 4 LM Studio | 4:00 |
| 5 opencode + SSH | 2:00 |
| 6 config.json | 2:00 |
| 7 SearXNG | 3:00 |
| 8 AGENTS.md | 3:00 |
| 9 When it breaks | 0:30 |
| 10 Demos | 4:00 |
| 11 Close | 0:30 |

## Cues

| Beat | Deck | Old SVG |
|---|---|---|
| Environment | [4] | `01-topology.svg` (keep dimmed in a corner) |
| Four parts | [3] | — |
| Claude Code | [7] | `06-claude-code-vs-local.svg` |
| VRAM / context | — | `07-vram-budget.svg` — **still says 64k; update to 96k or skip** |
| config.json | [5] | `02-one-turn.svg` (optional) |
| Rootful podman | — | `05-namespace-split.svg` |
| AGENTS.md | [6] | `04-capability-vs-trigger.svg` |
| When it breaks | [8] | — |
| Demos | [9] | — |

## Follow-up video ideas

- Broken systemd unit from journalctl
- Small Python stats page as a service + firewall port
- Cron health check ("works when I run it, fails from cron" = PATH)
