# Local Agentic Stack — Architecture Reference

Read this when something breaks or you want to understand why a decision was made.
For step-by-step build instructions, see `runbook.md`.
For turning MCP servers on and off, see `mcp-servers.md`.

Start at **Latency triage** if the symptom is "opencode is hanging" — that's the most
common report and it's almost always the 5090, not the config.

---

## Physical topology

```
┌──────────────────────────────────────────────────────────────────────┐
│  LAPTOP (macOS)                                                      │
│  opencode CLI  ←→  config.json  ←→  AGENTS.md                       │
│  mcp-searxng subprocess (npx -y mcp-searxng)                         │
│  aws CLI  |  ssh client (key auth as mcropsey)                       │
└────────────┬───────────────────────────────────────┬─────────────────┘
             │ HTTP :1234                             │ SSH :22
             ▼                                        ▼
┌────────────────────────────┐      ┌────────────────────────────────┐
│  5090 BOX (Windows)        │      │  hv-rocky-linux-4 (Rocky)      │
│  192.168.1.194             │      │  192.168.1.101                 │
│                            │      │                                │
│  LM Studio :1234           │      │  SearXNG container :8080       │
│  OpenAI-compatible /v1     │      │  rootful podman                │
│                            │      │                                │
│  Qwen3-Coder-30B-A3B       │      │  SearXNG → Google/Bing/DDG     │
│  Q4_K_M · 64k ctx          │      │  over HTTPS (public)           │
│  FlashAttn · Q8 KV         │      │                                │
└────────────────────────────┘      └────────────────────────────────┘

             │ HTTP :8009 / :8013  (MCP — disabled by default)
             ▼
┌──────────────────────────────────────┐
│  MCP HOST — 192.168.1.102            │
│  crapi   MCP :8009/mcp/   29 tools   │
│  noname  MCP :8013/mcp    39 tools   │
│  both "enabled": false               │
└──────────────────────────────────────┘
```

### Component roles

| Component | Host | Role |
|---|---|---|
| opencode | Laptop | Harness: owns tools, runs the think→act→observe loop |
| LM Studio | 5090 | Brain: serves the model over an OpenAI-compatible API |
| mcp-searxng | Laptop (subprocess) | Translates MCP tool calls → SearXNG HTTP requests |
| SearXNG | Rocky .101 | Self-hosted metasearch; fans out to real engines, returns JSON |
| AGENTS.md | Laptop | Behavioral control plane — injected into every system prompt |

> Most "opencode isn't working" problems are **model** problems, not harness problems.

---

## One turn, end to end

```
You ──► opencode ──► builds system prompt (base + tool defs + AGENTS.md)
                          │
                          ▼
                     POST /v1/chat/completions → LM Studio (.194)
                          │
                          ▼
                     tool_call: bash("ssh mcropsey@... podman ps")
                          │
                          ▼
                     permission gate (config.json → "bash": "ask")
                     └── every command → confirmation dialog → you approve
                         (nothing pre-approved, nothing blocked)
                          │
                          ▼
                     execute on target (.101 / AWS / local)
                          │
                          ▼
                     tool result appended to context
                          │
                          ▼
                     LM Studio → final answer (or next tool call → loop)
                          │
                          ▼
                     You ◄── response
```

---

## Configuration control plane

Two files do different jobs:

| File | Controls | When read |
|---|---|---|
| `config.json` | What the harness **can** do — provider endpoint, MCP servers, permission rules | opencode start |
| `AGENTS.md` | What the model **will** do — triggers, guardrails, lab context | Session start (injected into system prompt) |

**Capability and trigger are separate — you need both.** The MCP block in config.json gives the model a search tool. The AGENTS.md rule makes it reach for one. Same pattern for sudo: the permission gate lets it through once you approve, and the AGENTS.md line makes the model call it instead of explaining it.

**Only one half of config.json costs tokens.** The two blocks behave completely differently:

| Block | Sent to the model? | Cost per request |
|---|---|---|
| `permission` | **No** — evaluated locally by opencode before it executes a tool | 0 tokens |
| `mcp` | **Yes** — every enabled server's full tool schema goes in the system prompt | ~2,000 tokens per small server, ~8,000 for noname + crapi |

This is why `permission` collapsed to a single `"bash": "ask"` line with no loss: the
pattern list was local policy the model never saw, so trimming it bought clarity, not
speed. The `mcp` block is the opposite — it's the one that actually shows up in latency,
so it's the one worth keeping short. See `mcp-servers.md` for the toggle and measured
costs.

**Order inside AGENTS.md is load-bearing.** Local models weight earlier instructions more heavily, and their built-in safety training competes with your file. As the file grows, a tool-access block buried at the bottom loses to the safety filter. SSH stopped working after the web-search section was appended — that's the documented regression.

---

## Grounding layer — why it exists

```
Model needs a fact
        │
        ├── Public + current (versions, ports, flags, real repos)
        │       → SearXNG web search  [DEPLOYED]
        │
        ├── Your own infra (IPs, lab layout, crAPI stack details)
        │       → Local RAG / lab-kb + Qdrant  [PLANNED]
        │
        └── Neither
                → Model invents it — fabricated image names, wrong ports,
                  fake versions, presented confidently
```

Web search and local RAG fix **different** hallucinations. Web search will never cover your own infra details — they aren't on the internet. The original Tavily approach is superseded by self-hosted SearXNG (no API key, no rate limit, no data leaves the LAN).

---

## Failure map

```
LAPTOP                     NETWORK          5090 (.194)          ROCKY (.101)
─────────────────────────  ───────────────  ───────────────────  ─────────────────
G0  sudo needs a TTY                        G3  8k default ctx   G2  ssh password
    → sudo -v before                            → set 32–64k,        → ssh-copy-id
      session                                   FlashAttn+Q8KV,
G1  model dictates                              reload
    instead of running                      G6  empty response   G5  podman lock
    → CRITICAL block first                      after thinking       corruption
G4  claims success                              → wrong chat         → system
    without verifying                           template             renumber
    → verification rules
G9  MCP schema bloat                        G7  multi-model VRAM
    → disable unused                            thrash → unload
      servers                               G8  ctx limit > real
                                                → set 65280
```

| # | Symptom | Fix |
|---|---|---|
| G0 | `sudo: a terminal is required` | `sudo -v` in your terminal before the session (~5 min cache) |
| G1 | "I can't access external systems, here's what to run" | Tool-access block first in AGENTS.md; per-prompt: "Use bash and SSH to…" |
| G2 | SSH hangs waiting for input | `ssh-copy-id` first; must be password-free for you before the agent ever touches it |
| G3 | `request (N tokens) exceeds context size (8192)` | LM Studio: unload → Load tab → context 32–64k + FlashAttn + Q8 KV → reload |
| G4 | "The container is running" — it isn't | Verification block in AGENTS.md; end prompts with an explicit check command |
| G5 | `acquiring lock 0 … file exists` | `podman rm -f <id>` → `stop --all` → `system renumber` |
| G6 | Thinks, emits nothing | Wrong chat template in LM Studio for this GGUF; switch to model's native tool-use template |
| G7 | **Every request crawls; opencode looks hung, you hit ESC** | Too many models resident on the 5090. Unload all but the one in use — see "Latency triage" below |
| G8 | `AI_APICallError: Internal Server Error` on long sessions | `limit.context` set above LM Studio's real `loaded_context_length` (65280, not 65536) |
| G9 | Trivial prompts are slow even on a healthy GPU | MCP tool-schema bloat — disable unused servers (`mcp-servers.md`) |
| G10 | Loaded a different model in LM Studio, opencode won't connect (`400 Engine protocol startup was aborted`) | opencode still requests its configured model ID and JIT tries to load it alongside. Declare the new model and select it — see "Switching models" in `runbook.md` |
| G11 | Models load/unload in LM Studio on their own | JIT loading + idle TTL are both on by default. Turn both off; load one model manually |
| G12 | Model types <code>```bash ls ```</code> as text instead of running it | The model can't tool-call (DeepSeek-R1 declares no `tool_use` capability). Not a prompt problem — switch models |

**G1 is the recurring one** — it's architectural, not a config regression. Your instructions and the model's training compete for the same prompt, and prompt length shifts who wins.

---

## Latency triage — "opencode is hanging"

It is almost never hanging. It's waiting on the model, and there are only two real causes.
Check them in this order, because the first one dwarfs the second.

**1. Is the 5090 oversubscribed?** (G7 — this is the big one)

```bash
curl -s http://192.168.1.194:1234/api/v0/models | python3 -c "
import json,sys
for m in json.load(sys.stdin)['data']:
    if m.get('state')=='loaded':
        print(f\"{m['id']:38s} ctx={m.get('loaded_context_length')}\")
"
```

If more than one model is loaded, that's the problem. VRAM math on a 32 GB card:

| Loaded | Weights + KV | Fits? |
|---|---|---|
| Qwen3-Coder-30B-A3B alone | ~21 GB | ✅ |
| + a 27B VLM | ~41 GB | ❌ spills to system RAM |
| + a second 30B instance + an 8B | ~71 GB | ❌❌ hard CPU thrash |

Anything past 32 GB runs on the CPU. Observed on this stack: **24 s to emit 8 tokens**
with four models loaded, versus **65 tok/s and 0.03 s to first token** with one. That
40× swing is the entire difference between "working" and "hung."

Unload the extras in LM Studio (Developer → eject), and **turn off JIT model loading**
(or cap max loaded models at 1). JIT is what silently created the duplicate
`qwen3-coder-30b-a3b-instruct:2` instance: opencode fires its title-generation request
concurrently with the main request at session start, and LM Studio answered the second
one by loading a whole second copy of the model.

**2. How big is the prompt before you've typed anything?** (G9)

Ask opencode `what is 2x2?` and read `prompt_tokens`. With only searxng enabled it should
be ~2,155. If it's ~10,000, `noname` and `crapi` are on — every request is carrying 68
extra tool definitions. `mcp-servers.md` has the toggle.

**What is *not* the cause:** the `permission` block and `AGENTS.md`. The permission rules
never leave your laptop, and AGENTS.md is ~500 tokens. Neither is a latency lever —
don't go trimming rules chasing speed.

**Reading the logs.** `~/.local/share/opencode/log/opencode.log`:

```bash
grep -E "ERROR" ~/.local/share/opencode/log/opencode.log | tail -20
```

`error=Aborted` means *you* pressed ESC — it's a symptom, not a cause. Look for the
`stream error` line above it for the real failure.

---

## Rootful vs. rootless — a namespace split worth knowing

SearXNG was created with `sudo`, which puts it in a different storage namespace than rootless podman:

```
sudo podman ps  →  /var/lib/containers/...       →  searxng visible  ✅
     podman ps  →  ~/.local/share/containers/... →  empty list       ❌  (looks deleted)
```

Consequences:
- Every `podman` command for SearXNG needs `sudo` — `ps`, `exec`, `restart`, `volume inspect`
- Reboot survival requires the **system** unit: `sudo systemctl enable --now podman-restart.service` — podman's `--restart` flag alone does not replay after a host reboot (unlike Docker)
- Volume paths are under `/var/lib/containers/storage/volumes/` — always get the real path from `sudo podman volume inspect`
- The agent needs passwordless sudo (or root SSH) to manage any rootful container

---

## Verified current state (2026-08-22)

| Component | Host | Status | Notes |
|---|---|---|---|
| SearXNG container | .101 | ✅ Up (2 days, RestartCount=0) | Volume: `searxng`, JSON enabled, secret_key set |
| podman-restart.service | .101 | ✅ enabled + active | Survives reboot — covers `unless-stopped` policy |
| mcp-heartbeat.service | .98 | ✅ enabled + active | Noname MCP session |
| mcp-sweep.service | .98 | ✅ enabled + active | Noname MCP session |
| LM Studio server | 5090 (.194) | ⚠ Manual start required | Not running at time of check — requires manual start after every Windows reboot |

---

## Open items

| Item | Notes |
|---|---|
| LM Studio auto-start | Server does not start automatically after Windows reboot — manual step every time. Windows Task Scheduler could automate this but LM Studio CLI support is limited. |
| aws-docs MCP | `awslabs.aws-documentation-mcp-server@latest` — add alongside searxng in config.json |
| Qdrant RAG | Local RAG over `~/lab-kb` using `nomic-embed-text-v1.5` (already in LM Studio). Closes the "your own infra" grounding gap. Add after web search is solid. |
| Frontier hybrid | Anthropic provider in config.json as escape hatch for hard tasks. Template in runbook. |
| Model A/B | Devstral vs Qwen3-Coder — re-run the same tool-calling task on both and compare reliability |
| Pin mcp-searxng | Lock `mcp-searxng@<version>` in config.json instead of `npx -y` (pulls latest silently) |
| LM Studio auth | Enable `Require Authentication` on port 1234 before any exposure beyond the home LAN |

---

## Security boundary

```
Trusted                          Semi-trusted
─────────────────────────────── ────────────────────────────────────
Laptop — SSH keys, AWS creds    LM Studio :1234
Rocky .101 — SearXNG host         (Require Authentication OFF — LAN only)
                                npx-fetched MCP code
                                  (unpinned = supply-chain risk)
                                SearXNG → public engines over HTTPS
```

Standing rules:
1. **Scope the SSH-enabled agent deliberately.** An agent that can SSH and sometimes over-claims success should only reach hosts you've decided it can reach.
2. **Pin MCP/plugin versions** (`mcp-searxng@1.0.0`). `npx -y` runs unaudited npm code and pulls newer versions silently.
3. **Don't expose :1234 past the LAN** while Require Authentication is off.
