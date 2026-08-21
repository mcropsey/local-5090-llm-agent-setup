# opencode + Local LLM (LM Studio) — Setup & Gotchas

A working reference for driving a local coding agent: **opencode** on a laptop talking to a
model served by **LM Studio** on a PC with an RTX 5090. Includes the gotchas actually hit
during setup so you (or future-you) don't re-learn them the hard way.

---

## The mental model

An agent is just: **an LLM + tools + a loop**. In this setup the pieces map to:

- **LLM / "brain"** — the model loaded in LM Studio (e.g. Qwen3-Coder-30B-A3B).
- **Harness / tools + loop** — opencode. It owns the bash tool, file tools, and the
  think→act→observe loop.
- **Transport** — opencode (laptop) → LM Studio's OpenAI-compatible server (PC) over the LAN.

Most "opencode isn't working" problems are actually **model** problems, not harness problems.
The harness is fine; a weak or misconfigured model is usually the cause.

---

## Base configuration

### LM Studio (PC / 5090)

| Setting | Value | Why |
|---|---|---|
| Model | Qwen3-Coder-30B-A3B (or Devstral) | Tuned for agentic/tool-calling loops |
| Quant | Q4_K_M or Q5 | Below Q4, tool-calling reliability collapses |
| Context length (`n_ctx`) | 32k–64k | Agent prompts blow past the 8k default instantly |
| Flash Attention | On | Fits larger context in VRAM |
| KV Cache Quantization | Q8 (K and V) | Lets 64k context fit alongside the model in 32GB |
| Server | "Serve on Local Network" enabled | Binds to LAN IP, not just localhost |
| Port | 1234 (default) | Must be open in Windows Firewall (inbound) |

Rough VRAM math on a 5090 (32GB): 30B-A3B at Q4 ≈ 18GB, leaving ~13GB for KV cache.
With Q8 cache quant, 64k context fits comfortably; 128k gets tight — start at 64k.

### opencode (laptop)

- Register LM Studio as an **OpenAI-compatible provider** pointing at
  `http://<PC-IP>:1234/v1`.
- The **model ID must exactly match** what LM Studio exposes — check `GET /v1/models`.
- Confirm the **bash tool is allowed** (permission mode not silently blocking it).

### Quick end-to-end test

Ask the agent to list files in the project. If it **runs the tool** instead of hallucinating
a file list, transport + tool-calling are working.

---

## AGENTS.md — the highest-value file for a local model

opencode auto-loads `AGENTS.md` into the system prompt each session.

- **Per-project:** `AGENTS.md` in the directory you launch opencode from.
- **Global (per-machine):** `~/.config/opencode/AGENTS.md`.
- Global lives on each machine separately — it does **not** follow you between boxes.
  Sync it via a dotfiles repo or `scp` if you want it everywhere.
- New sessions only — the file is read at session start, not live.

### Recommended contents

Two jobs: (1) tell the model what infra exists, (2) force it to verify its own work.

```markdown
You have a working bash tool with full shell access.
You CAN reach machines on the local network via ssh.
SSH key auth is configured for user mcropsey on all local servers.
When asked about a remote machine, connect with:
  ssh mcropsey@<ip-or-hostname>
using whatever IP or hostname the user gives you.
Run commands directly and show the output instead of telling the user to run them.

## Verification (IMPORTANT)
When you run a command that changes state (start/stop/create a container, write a
file, restart a service), you MUST run a follow-up command that verifies the result
before reporting success. E.g. after `podman run`, run `podman ps` and confirm the
container shows "Up" in the output you actually received.

Never describe something as "running", "working", or "accessible" unless a command you
ran in THIS session shows it. If a command prints an error, treat it as a failure and
investigate — do NOT dismiss errors as "temporary" or "harmless". Quote the real output
when you claim a result.

## Known servers
- 192.168.1.101 (hv-rocky-linux-4): Rocky Linux podman host. Lab network only.
```

> The verification block matters **far more for a local model than for a hosted one**.
> Small models happily invent "it's running and accessible" without checking — the rules
> above directly counter that.

---

## Gotchas (things actually hit during setup)

### 1. The model dictates commands instead of running them
**Symptom:** You ask it to SSH somewhere; it replies "I can't access external devices,
here are the commands to run yourself."
**Cause:** Smaller models fall back on a training reflex and don't "believe" they have real
shell access — even though opencode gave them the bash tool.
**Fix:** Be explicit in the prompt ("Use the bash tool to run `...`"), and add the
capability statement to AGENTS.md. Once it succeeds once, it tends to keep using the tool.

### 2. SSH prompts for a password → agent hangs
**Cause:** The bash tool runs non-interactively; it can't answer an SSH password prompt.
**Fix:** Set up key auth *before* pointing the agent at a box:
```bash
ssh-keygen -t ed25519        # if you don't have a key
ssh-copy-id mcropsey@192.168.1.101
ssh mcropsey@192.168.1.101 hostname   # must return with NO password prompt
```
If it prompts for a password for *you*, it will hang for the agent.

### 3. `exceed_context_size_error` (400)
**Symptom:** `request (10491 tokens) exceeds the available context size (8192 tokens)`.
**Cause:** LM Studio loaded the model with the default 8k context. opencode's system prompt
+ tool definitions + AGENTS.md alone exceed that.
**Fix:** Unload the model → open load settings → set Context Length to 32k–64k → enable
Flash Attention + Q8 KV cache → reload. Verify the server tab shows the new context (LM
Studio sometimes keeps an old instance around).

### 4. The model claims success when it failed
**Symptom:** "The container is running and accessible" — but `podman ps` shows it isn't.
**Cause:** The model saw "container created with ID …" and pattern-completed the happy path
instead of verifying. Classic small-model over-claiming.
**Fix:** The verification rules in AGENTS.md. Also end prompts with an explicit check:
"then run `podman ps` and show me it's actually Up before claiming success."

### 5. Podman lock corruption dismissed as "temporary"
**Symptom:** `ERRO Refreshing container <id>: acquiring lock 0 …: file exists`, and the model
waves it off as harmless.
**Cause:** The container grabbed a lock slot already held by another container; podman can't
refresh its state. The container is wedged, not running. This is **not** temporary.
**Fix (run manually):**
```bash
podman rm -f <container-id>     # remove the wedged container
podman stop --all              # if renumber complains about running containers
podman system renumber         # rebuilds the lock table — the actual fix
```
Cleaning this up also gives the model cleaner tool output (fewer scary lines to misread).

### 6. Empty / no response after "thinking"
**Symptom:** The model thinks for a few seconds and emits nothing.
**Cause:** Tool-call formatting fumble — often a wrong chat template for the loaded GGUF.
**Fix:** Confirm LM Studio is using the model's native tool-use chat template, not a generic
one. Try a different quant/build if it persists.

---

## Getting more out of the local setup

In rough priority order:

1. **Model + quant + template.** Biggest lever. Q4_K_M/Q5, correct tool-use template. Try
   Devstral vs Qwen3-Coder head-to-head for reliability on your tasks.
2. **Context tuning.** Big enough to not truncate (32k–64k), but keep sessions focused —
   one task per session, start fresh often. Quality degrades as context fills.
3. **AGENTS.md verification rules.** Counters over-claiming; worth the most for local models.
4. **Scripts over judgment.** Move repeated workflows into deterministic shell scripts the
   agent just calls (e.g. a deploy script that greps `podman ps` and exits non-zero if the
   container isn't Up). The weak model can't over-claim if a script does the checking.
5. **opencode structural features.** Custom commands (reusable prompts) and MCP servers
   (structured podman/filesystem/git tools instead of raw shell parsing) make the harness
   pull more weight. *(Verify current syntax against opencode docs — it changes.)*

### Realistic expectations
A good 30B local model handles scoped tasks well ("fix this function", "deploy this
container and verify it") but won't match a frontier hosted model over long, 20+ tool-call
sequences in a big repo. Keep tasks small, specific, and imperative — you do the planning,
the model executes. That's where small models are strongest.

---

## Security notes

- Keep this SSH-enabled agent pointed **only at your lab network.** You're running crAPI (a
  deliberately vulnerable app) and juice-shop on the Rocky box — an agent that can SSH and
  occasionally over-claims success is not something you want reaching anything you care about.
- If you add opencode **plugins later, pin them to a specific version**
  (e.g. `plugin@1.8.0`). opencode auto-updates plugins on startup; an unpinned plugin is a
  supply-chain risk.

---

## Quick reference

```bash
# On each server the agent should reach (once):
ssh-copy-id mcropsey@<ip>
ssh mcropsey@<ip> hostname          # verify no password prompt

# Fix wedged podman containers / lock errors:
podman rm -f <id>
podman stop --all
podman system renumber

# Verify a deploy actually worked:
podman ps                            # container must show "Up"
```

| File | Location | Purpose |
|---|---|---|
| `AGENTS.md` (project) | launch dir | Per-project instructions |
| `AGENTS.md` (global) | `~/.config/opencode/AGENTS.md` | Machine-wide instructions |
| LM Studio server | `http://<PC-IP>:1234/v1` | OpenAI-compatible endpoint |
