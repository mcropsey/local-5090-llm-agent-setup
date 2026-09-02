# Local Agentic Stack — Demo Notes (v2, rebuilt from Take 1)

Notes format. Follows the order you actually recorded.
Repo: https://github.com/mcropsey/local-5090-llm-agent-setup

---

## FIX LIST FROM TAKE 1 — read this first

| # | What happened | Fix for Take 2 |
|---|---|---|
| 1 | **~4 min lost hunting the LM Studio server toggle** (14:00–17:20). Ended up asking Claude on camera. | Pre-stage it. Exact path in Act 4 below. If you still want to show it, show it *knowing* the path — 20 seconds, not 4 minutes. |
| 2 | Old chats visible in LM Studio, deleted them live (11:00) | Delete before recording |
| 3 | "Isn't this just Claude Code? That's kind of a dumb question" (7:50) | Don't call your own framing question dumb. It's the best question in the video. Reframe — script in Act 3 |
| 4 | Forgot the word "harness" mid-recap (9:16) | Recap is 4 words: brain, harness, senses, personality. Put it on screen so you can read it |
| 5 | "I don't know where they came up with that number" re: 65k context (11:41) | You do now — VRAM math, one line, in Act 4 |
| 6 | npx/node explanation rambled ~90s (21:59–22:35) | One line: "Node.js. mcp-searxng is a JavaScript package; npx runs it." Move on |
| 7 | Name said as "Mike Crops" (0:01) | Say it slowly, and put it on screen |
| 8 | SearXNG said as "Siri XE" / "CRX NG" / "SEER XNG" | Pick one, say it once: "SearXNG — S-E-A-R-X-N-G" |
| 9 | Wasn't sure if a web server was already on .98 (26:34–27:12) | Pre-check. `ssh mcropsey@192.168.1.98 'rpm -q httpd'` before rolling |
| 10 | "my local Claude as a CLI for Claude code" (8:00) | Muddled. Just: "Claude Code is the CLI. opencode is the open-source equivalent" |

**Pre-flight, 5 min before recording:**
```bash
curl -s http://192.168.1.194:1234/v1/models          # LM Studio server IS running
curl -s "http://192.168.1.101:8080/search?q=test&format=json" | head -c 100
ssh mcropsey@192.168.1.98 'rpm -q httpd'             # expect: not installed
ssh mcropsey@192.168.1.101 'sudo -n podman ps'
```

---

# ACT 1 — WHO / WHAT / WHY (~2 min)

**0:00 — Open**
- Name, slowly. On screen.
- One line: local agentic LLM stack that does what Claude Code does, on hardware I own.
- Be upfront: not as capable as Claude Code. Very capable for lab work.

**What I use it for** — say these concretely, they land:
- Configure servers, install/uninstall packages, re-IP, troubleshoot logs
- Two ways to run it: opencode installed on each Linux box, or one install on the laptop that SSHes out. You do both.

**Why not just Claude Code**
- $20 plan, hit the limit, wait for reset. This eliminates that.
- Claude Code for the hard stuff — your example was good: a big Terraform file with a lot of moving objects in AWS, where you need real troubleshooting depth.
- Local for basic AWS: spin up an EC2 from the CLI, create a couple of resources. Works fine.
- It thinks slower. It's free. That's the trade.

**2:00 — The hardware**
- B-roll of the 5090 box. It's also the gaming rig / driving simulator.
- **Keep the wife joke.** "I justified the card as work hardware for local AI — which is true. Let's be honest about the ratio though."
- This humanizes the whole video. Don't cut it in the edit.

---

# ACT 2 — WHAT'S INSTALLED WHERE (~3 min)

**Show `01-topology.svg`.**

- **Laptop (macOS)** — opencode, plus the MCP server running locally as a subprocess.
  - Say clearly: *MCP doesn't have to live here.* You have MCP servers elsewhere on the network for other work. It's local here for convenience.
- **5090 (.194)** — LM Studio serving the model. Say: "doesn't have to be LM Studio, that's just what I run."
- **Rocky Linux (.101)** — SearXNG in podman.

**Why the search piece exists** — this was one of your clearest explanations in Take 1, keep it:
- A downloaded local model is frozen at its training cutoff. Ask it about recent news, it has nothing.
- So it needs a way out to the live web. That's SearXNG. MCP is what lets the model talk to it.

**Rocky 9 not 10** — the Akamai API security sensors weren't supported on RHEL 10 for a while. Ten seconds, then move.

---

# ACT 3 — THE FOUR PARTS + "ISN'T THIS JUST CLAUDE CODE?" (~3 min)

**Put the four words on screen. Read them off. Don't improvise this part.**

| | | |
|---|---|---|
| **Brain** | the model | Qwen3-Coder 30B on the 5090 |
| **Harness** | the hands | opencode, on the laptop |
| **Senses** | the tools | SearXNG via MCP, plus SSH and bash |
| **Personality** | the instructions | `AGENTS.md` |

**Then the Claude Code question. Show `06-claude-code-vs-local.svg`.**

Reframe from Take 1 — say it like this:
> "The obvious question is: isn't this just Claude Code? It's a fair question, and the answer is
> that Claude Code is the better agent. It's a frontier model — Opus, Sonnet. What's different
> here is the brain and the hands. Claude Code is Anthropic's CLI; opencode is the open-source
> equivalent that lets me point at any OpenAI-compatible endpoint. MCP is identical on both
> sides. And the instruction file is `AGENTS.md` instead of `CLAUDE.md`."

- Mention you run both. You use Claude Code with MCP servers against management APIs for the products you support, and against a deliberately vulnerable API app for demos.
- Land it: *"Different jobs. I use both."*

**Recap, 20 seconds, reading off the screen:** brain on the 5090, harness on the laptop, MCP local, SearXNG for live search, AGENTS.md for behavior.

---

# ACT 4 — LM STUDIO (~4 min, was 7)

Chats already deleted. Model already selected.

**4.1 — Load the model**
- Point at the model, say why briefly: you tried a few, settled on this one.
- Skip the model browser entirely. You said "there are whole videos on that" — correct, so don't start one.

**4.2 — Context length: the one setting that matters**
- Default is **8k**. Change it to **65536**.
- **Say why, don't shrug at it** — show `07-vram-budget.svg`:
  > "The 5090 has 32 GB. Model weights at Q4 take about 18. That leaves roughly 13 for the KV cache — the agent's short-term memory. At Q8 with Flash Attention on, 13 GB buys you about 64k tokens. That's where the number comes from."
- Also turn on: **Flash Attention**, **KV cache Q8**.
- Say the failure symptom out loud: leave it at 8k and it hiccups and dies on anything but a trivial ask. Every tool result gets pasted back into context, so it fills fast.
- Mention GPU offload: if it spills to CPU it gets very slow.

**4.3 — Start the server — THE 4-MINUTE STUMBLE. Say it in this order:**

1. **Settings → Developer → turn Developer mode ON.** Nothing appears until you do this. This is the step that isn't obvious.
2. **Developer tab → Server Settings.**
3. **Serve on Local Network** → ON. (Without this it binds to localhost and the laptop can't reach it.)
4. **Require Authentication** — off in the lab. Say: "turn this on if you're going past your own LAN."
5. Start the server.

**Prove it from the laptop** — this is your act break:
```bash
curl http://192.168.1.194:1234/v1/models
```
> "That's the entire contract between these two machines. One URL."

**Say the reboot tax:** the server does not come back on its own after a Windows reboot. Manual start every time.

---

# ACT 5 — INSTALL OPENCODE + SSH (~2 min)

**5.1 — Install**
- One curl command. Add to PATH. That's it.

**5.2 — SSH access** (keep this short, it's a side note in your flow)
- Passwordless keys is what you run, and it's the right answer.
- **Keep the trick you mentioned — it's a great line:** you don't have to set it up by hand. Once opencode is running, tell it: *"log into these servers with this ID and set up key-based auth."* It does the keygen and the copy for you.
- If you're stuck on passwords: you can hand it a user ID and password, but note the better options exist — `SSH_ASKPASS` with `SSH_ASKPASS_REQUIRE=force` pulling from a keychain, or `sshpass -f` with a 600 file. Never `-p` (lands in `ps` and history). One sentence each, don't stop the video for it.

---

# ACT 6 — CONFIG.JSON: THE PERMISSION BLOCK (~2 min)

**This is the part you flagged as "take a look at this" — give it the time.**

Walk the rules on screen:
- **`"allow"`** (runs silently): ssh, scp, systemctl, journalctl, kubectl, helm, minikube, kind, k3s, curl, wget — remote work and Kubernetes. Plus: dnf/apt read commands (list, search, info, repolist), `sudo dnf/apt install`, repo setup (`sudo tee /etc/yum.repos.d/*`, apt keyrings), binary drop-ins (`sudo install`, `sudo mv/chmod +x /usr/local/bin/*`), git clone/ls-remote, and gh CLI read operations.
- **`"ask"`** (confirmation dialog): docker, podman, npm, pip, brew, aws — package managers and mutation-heavy tools stay gated.
- `*` (catch-all) → **ask** — anything not explicitly listed also asks.
- You start conservative; loosen individual entries when friction outweighs the risk. Don't flip `"*"` to allow.

**Two things to say:**
1. > "This is where you decide what the agent can do on its own and what stops and asks you. I decided that ahead of time, in a file, while I was calm."
2. **The production caveat, verbatim from Take 1 — it was good:** this is a lab. Don't run allow-everything in production; that's what Ansible and change control are for.

Optional 15 seconds: mention you could configure a Claude provider here as a fallback for hard tasks. You said you haven't set it up — say that too, it's honest.

---

# ACT 7 — SEARXNG (~3 min)

**Show `05-namespace-split.svg` when you hit the sudo point.**

- What it is: gives the model live web search. **No API key.** Self-hosted, nothing leaves the LAN.
- Caveat you already make: lab setup. Production would want auth on everything.

**Steps:**
1. Run command — podman. Say why podman: it's the default runtime on RHEL variants.
2. **Reboot survival is different from Docker.** Podman's `--restart` flag alone doesn't replay after a host reboot:
   ```bash
   sudo systemctl enable --now podman-restart.service
   ```
3. **JSON output** — edit `settings.yml`, add `json` under `search: formats:`, set a real `secret_key`, restart. Without this the MCP server gets a 403 with no useful error.
4. Verify from the laptop before touching opencode:
   ```bash
   curl "http://192.168.1.101:8080/search?q=test&format=json"
   ```

**Prerequisite — one line, then move (fixes the 90-second ramble):**
> "You need Node.js installed, because `mcp-searxng` is a JavaScript package and `npx` is what runs it."

**Rootful note:** SearXNG was created with sudo, so every podman command for it needs sudo. Rootless `podman ps` shows an empty list and looks like you deleted it. That's the diagram.

---

# ACT 8 — AGENTS.MD (~3 min)

**Show `04-capability-vs-trigger.svg`.**

> "`config.json` is what it *can* do. `AGENTS.md` is what it *will* do. You need both."

Walk your actual blocks:
- **Tool access** — it can reach machines over SSH, including commands that need sudo. If you don't say this, it won't do it.
- **Loop discipline** — run one command, read the result, then decide. No menus of six options.
- **Grounding** — look up real docs before configuring something.
- **Web search** — when to reach for it. Your training data has a cutoff.
- **Cloud security defaults** — never open a security group to 0.0.0.0/0.
- Machine list, project overrides, testing/troubleshooting notes.

**The search-order point — this was one of your strongest bits, give it 30 seconds:**
> "Where you put the search instruction matters. Put it at the end of the file and the model may
> never reach for it. Perplexity searches first, always. ChatGPT sometimes does and sometimes
> doesn't. That's the same problem, and here you get to decide it."

**Optional but strong:** if you have 60 seconds, break it live. Move the tool-access block to the bottom, restart, ask it to SSH somewhere — it'll tell you it can't access external systems and hand you commands to run yourself. Move it back. Nothing changed but position.

---

# ACT 9 — DEMOS (~4 min)

**Demo 1 — Install Apache (26:30)**
```
Log in to 192.168.1.98 and install Apache.
```
- Pre-verify httpd is NOT installed before rolling.
- **Leave the error in.** It hit one and recovered on its own. Narrate it:
  > "It hit an error and fixed itself. That's the loop — it reads the result and adjusts. Claude Code does the same thing."
- Verify in a browser. Don't just take the agent's word for it — that's the habit worth modeling.

**Demo 2 — Live web search (28:30)**
```
Search the web for the latest news about API security events.
```
- Point out this is SearXNG doing the work.

**Demo 3 — Stale vs. current (28:50) — best demo in the video, don't rush it**
```
What are the requirements for the most recent RHCSA exam?
```
- Your point from Take 1: ask the raw model and it gives you RHEL 9, because that's where its training stopped. With search, you get current.
- **If you can, show both.** Ask once with search disabled or in plain LM Studio chat, then again through opencode. Side by side, that's the whole argument for the search layer in fifteen seconds.

---

# ACT 10 — CLOSE (~30 sec)

- GitHub URL on screen, hold 5+ seconds. Also pin it in the description.
- Say what's in it: runbook to build it, reference for when it breaks.
- Thanks for watching.

---

## Timing

Take 1 ran ~30 min with ~6 min of dead air. This should land at **~24 min** clean.

| Act | Target |
|---|---|
| 1 Who/what/why | 2:00 |
| 2 What's where | 3:00 |
| 3 Four parts + Claude Code | 3:00 |
| 4 LM Studio | 4:00 |
| 5 opencode + SSH | 2:00 |
| 6 config.json | 2:00 |
| 7 SearXNG | 3:00 |
| 8 AGENTS.md | 3:00 |
| 9 Demos | 4:00 |
| 10 Close | 0:30 |

---

## Diagram cues

| File | Where |
|---|---|
| `01-topology.svg` | Act 2 — and keep it dimmed in a corner throughout |
| `07-vram-budget.svg` | Act 4.2 — the 65k number |
| `06-claude-code-vs-local.svg` | Act 3 |
| `05-namespace-split.svg` | Act 7 — the sudo/podman point |
| `04-capability-vs-trigger.svg` | Act 8 |
| `02-one-turn.svg` | Optional, Act 6 — where the permission gate sits in the loop |

---

## If you want more demos later

You already have the strongest three. If you do a follow-up video, these were the ideas from the
earlier draft and they still hold — but they're a second video, not this one:

- Diagnose a deliberately broken systemd unit from journalctl
- Have it write a small Python stats page, run it as a service, open the firewall port
- Schedule a health check in cron — good teaching, because "works when I run it, fails from cron"
  is almost always the minimal PATH
