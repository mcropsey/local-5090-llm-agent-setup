# Local Agentic Stack — Architecture Reference

Read this when something breaks or you want to understand why a decision was made.
For step-by-step build instructions, see `runbook.md`.

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
│                            │      │  + crAPI stack, Noname sensor  │
│  Qwen3-Coder-30B-A3B       │      │                                │
│  Q4_K_M · 64k ctx          │      │  SearXNG → Google/Bing/DDG     │
│  FlashAttn · Q8 KV         │      │  over HTTPS (public)           │
└────────────────────────────┘      └────────────────────────────────┘
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
                     permission gate (config.json permission block)
                     ├── matches "allow" pattern → run silently
                     └── matches "*": "ask"    → confirmation dialog → you approve
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

**Capability and trigger are separate — you need both.** The MCP block in config.json gives the model a search tool. The AGENTS.md rule makes it reach for one. Same pattern for sudo: the permission block allows it, the AGENTS.md line makes the model call it instead of explaining it.

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

**G1 is the recurring one** — it's architectural, not a config regression. Your instructions and the model's training compete for the same prompt, and prompt length shifts who wins.

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

## Open items

| Item | Notes |
|---|---|
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
