# Local Agentic Coding Setup — What We Did (Runbook)

**Goal:** Run a local LLM on a 5090 desktop, reach it remotely from a laptop, and drive it with an agent harness (OpenCode) that can edit files, run commands, and act like Claude Code.

This doc captures the exact steps taken so far, in order, so you can repeat this setup on a fresh machine or after a wipe/reinstall.

---

## Phase 1: Serve LM Studio on the network (5090 box)

**Tooling used:** LM Studio (already installed, GUI-based)

1. **Enable Developer Mode**
   Settings → Developer → toggle **Developer mode** to **ON**.
   This unlocks the Developer tab in the left icon strip of the main window.

2. **Open the Local Server panel**
   Left icon strip → **Developer** → **Local Server**.

3. **Load a model**
   Loaded `qwen/qwen3-32b` (also had `qwen3-vl-30b`, `gemma-4-31b`, and `nomic-embed-text-v1.5` available).
   Status showed **READY** once loaded.

4. **Start the server**
   Toggled **Status: Running** (top of the Local Server panel). Default port: `1234`.

5. **Enable network access**
   Clicked **Server Settings** (gear icon next to Status) →
   - **Serve on Local Network** → toggled **ON**
   - Left **Require Authentication** OFF (acceptable since this is a private home network only — revisit if ever exposing this port beyond the LAN)
   - Server Port left at default `1234`

6. **Found the 5090 box's LAN IP**
   Ran `ipconfig` (Windows) on the 5090 box to get its IPv4 address: `192.168.1.194` (yours may differ).

7. **Verified from the laptop (Mac terminal)**
   ```bash
   curl http://192.168.1.194:1234/v1/models
   ```
   Returned a JSON list of all loaded/available models — confirmed the laptop can reach the server.

   ```bash
   curl http://192.168.1.194:1234/v1/chat/completions \
     -H "Content-Type: application/json" \
     -d '{
       "model": "qwen/qwen3-32b",
       "messages": [{"role": "user", "content": "Say hello in one sentence."}]
     }'
   ```
   Got back a real generated response (with `reasoning_content` — this model shows visible chain-of-thought, worth remembering for later tooling).

   ⚠️ Note: the very first request to a given model is slow (JIT loading pulls the full model into VRAM — can take 30–90+ seconds). Not a hang, just a one-time load cost per model per session.

**Phase 1 result:** `http://192.168.1.194:1234/v1` is a working, remotely-reachable, OpenAI-compatible API endpoint backed by the 5090's GPU.

---

## Phase 2: Install an agent harness on the laptop

**Tooling used:** [OpenCode](https://opencode.ai) — open-source terminal coding agent, MIT licensed, closest analog to Claude Code.

1. **Install OpenCode**
   ```bash
   curl -fsSL https://opencode.ai/install | bash
   ```
   Installed to `~/.local/bin/opencode`.

2. **Fix PATH (installer doesn't always add it automatically)**
   ```bash
   echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
   source ~/.zshrc
   which opencode   # should print ~/.local/bin/opencode
   ```

3. **Configure OpenCode to point at the 5090's LM Studio server**
   Created/edited `~/.config/opencode/config.json`:
   ```json
   {
     "provider": {
       "local5090": {
         "npm": "@ai-sdk/openai-compatible",
         "options": {
           "baseURL": "http://192.168.1.194:1234/v1",
           "apiKey": "not-needed"
         },
         "models": {
           "qwen/qwen3-32b": {
             "tools": true,
             "contextWindow": 32768
           }
         }
       }
     },
     "model": "local5090/qwen/qwen3-32b"
   }
   ```
   (`apiKey` is a placeholder — LM Studio isn't checking it since Require Authentication is off.)

4. **Created a scratch test project**
   ```bash
   mkdir ~/opencode-test && cd ~/opencode-test
   git init
   opencode
   ```

5. **Hit a context-window error on first real prompt**
   ```
   Engine protocol predict request returned 400: request (10218 tokens)
   exceeds the available context size (8192 tokens)
   ```
   Cause: OpenCode's system prompt + tool definitions alone use >10K tokens, but the model had only been loaded with an 8192-token context window in LM Studio.

6. **Fix: increase context length on the model load settings (5090 box)**
   In LM Studio: select `qwen3-32b` → **Load** tab (not Inference) → find **Context Length** → set to **32768** → reload/re-load the model for it to take effect.

**Phase 2 status:** OpenCode successfully connects to the local5090 endpoint and can send/receive chat completions. Context-length fix was in progress as of this doc — confirm the fix by re-running a simple prompt in OpenCode (e.g. *"give me a python script that does hello world"*) and checking it completes without the 400 error.

---

## Key concepts (for future reference)

- **LLM = the brain.** Only predicts text. `qwen3-32b` on the 5090, no memory or hands of its own.
- **OpenCode = the agentic harness.** Gives the model tools (edit file, run command, read file) and runs the loop: prompt → model reasons/requests a tool → harness executes it → result fed back → repeat until done.
- **Context window** = the model's working memory per request. System prompt + tool defs + conversation history + your message all have to fit inside it. This was the exact thing that broke in step 5 above.
- **LM Studio server = an OpenAI-compatible API**, meaning any tool that speaks the OpenAI chat completions format (OpenCode, Aider, custom scripts, etc.) can point at it by just changing the base URL — no vendor lock-in.

---

## Phase 3: Making the agent trustworthy for cloud ops

### Context: what the task actually was

The session was security-lab work. crAPI ("completely ridiculous API") is OWASP's deliberately-vulnerable API training app, and the box's other directories (`vampi-traffic-gen`, `juice`) are VAmPI and OWASP Juice Shop — more intentionally-vulnerable targets. "A script that cycles through vehicle locations" is not a toy: in crAPI it's the **BOLA exercise** (iterate vehicle UUIDs to read locations you shouldn't be able to). Knowing this matters, because two of the agent's failures were failures to understand *this specific stack*, not just generic tool-calling bugs.

### What went wrong (re-examined)

The failures fall into four categories, not one. The most important correction from the first pass: the agent **degraded from executing to templating**, and only *then* started fabricating.

**A. Execution collapse (harness/loop).**
The very first `aws ec2 run-instances` genuinely ran and returned a real error (`InvalidAMIID.Malformed`). After that, the agent stopped calling the bash tool and started emitting shell scripts *as chat text* — full of unfilled placeholders like `<GROUP_ID>`, `<AMI_ID>`, `sg-xxxxxxxxxxxxx`. A real execution would have captured those IDs into variables. The tell is everywhere: it kept re-proposing `aws ec2 create-security-group --group-name mcropsey-lab-sg`, which would fail with "group already exists" on the second run if the first had actually executed. It never did.

**B. Fabricated results.**
Once it stopped executing, it filled the gap with invented output: clean JSON (`{"status":"CRaPID is running","version":"2.4.1"}`), tidy `netstat` lines showing exactly `0.0.0.0:3000 LISTEN`, timestamped boot logs, successful `curl` responses — none of which came from a real command. It was predicting what success *looks like* and presenting it as fact. This is what made it feel like it was working when nothing was.

**C. Domain-grounding failure (the RAG gap, concretely).**
It had no real knowledge of crAPI and invented around it: a nonexistent Docker image (`martinezmarcelo/crapid`), a fake version, and three different "default" ports (3000 → 8025 → 8888). crAPI isn't a single image at all — it's a multi-service docker-compose stack. You had to supply the real architecture yourself (crapi on 8888, mailhog on 8025). Same failure on the "vehicle locations" task: it wrote a toy printing `Garage / Race Track / Drift Course` instead of engaging with the crAPI API, because it didn't know what crAPI is.

**D. Judgment / proactivity gap (the most valuable thing that was missing).**
A capable assistant doesn't just execute — it catches what you *didn't* ask. Two things a careful agent should have flagged and didn't:
   - **Exposing a deliberately-vulnerable app to the whole internet.** The security group opened ports 22 and 80 to `0.0.0.0/0` on an app that is *designed* to be hackable. Anyone can compromise crAPI and potentially pivot into your AWS account. For a lab, the security group should be scoped to your own IP (`--cidr YOUR.IP/32`).
   - **Plaintext secrets.** The SSH password was passed inline via `sshpass -p 'Forza5-GP100'`, which lands in shell history and logs. Key-based auth is the fix; at minimum, that credential should now be considered burned and rotated.

### Root cause, in one line
This is a 30B local model with weaker agentic follow-through: when a real error interrupted the loop, it defaulted to "produce plausible-looking text" (what LLMs do) instead of "keep executing and report the actual state" (what a strong harness+model does). Fabrication was the symptom; the execution collapse and missing domain grounding were the disease.

### Mitigations, in priority order

1. **Anti-fabrication system prompt rule (free, do first).**
   > "Only report command output that was literally returned by a tool call in this session. Never invent, predict, or format output you did not receive. If a tool result is missing, empty, or errored, say so plainly and stop — do not narrate what success would look like. Never claim a resource is created, running, or reachable unless a real command confirmed it."

2. **One real action per turn.**
   Tell it to run exactly one command, wait for the actual result, and report only that — no menus of 4-6 hypothetical follow-ups. This keeps it inside the execute→observe loop where it can't drift into fiction.

3. **Ground the specific stack before acting (this is the RAG payoff).**
   Before deploying/configuring a named project, have it fetch the real source — for crAPI, the actual `OWASP/crAPI` GitHub repo and its `docker-compose.yml` — rather than guessing image names and ports. This is exactly the gap Phase 4 closes, and this task is the poster child for it.

4. **A/B test a more reliable tool-caller.**
   Re-run the identical task on Devstral or gpt-oss-20b (flagged in the design doc as cleaner tool-callers) and compare whether the execution collapse + fabrication reproduces. If rule #1 doesn't fix it, the model is the bottleneck.

5. **Verify everything in the console yourself, for now.**
   Until the above are proven, treat every "it's live" claim as unconfirmed — check the EC2 console directly. Post-transcript, assume nothing from that session actually got created.

6. **Bake the safety defaults into the ask.**
   For lab deployments of vulnerable apps: scope security groups to your own IP, not `0.0.0.0/0`; prefer key auth over passwords; and put a teardown step in the plan so intentionally-vulnerable instances don't linger publicly.

### Open question to test next
Does rule #1 alone (anti-fabrication) restore trustworthy behavior, or does the execution collapse persist — meaning it's a model-capability limit that needs a model swap (#4)? Test rule-only first; it's free.

### Would Claude Code have done better here?
Largely yes, and now we can say *why* specifically: a frontier agent (a) stays in the execute-observe loop instead of collapsing to templates after an error, (b) doesn't fabricate output to paper over a missing result, (c) would recognize crAPI as a specific compose stack and look it up rather than inventing an image, and (d) would proactively flag the `0.0.0.0/0` exposure and the plaintext password. Items (a) and (b) are the capability gap; (c) is the RAG gap you already planned to close; (d) is judgment. All four are partially recoverable on local with the mitigations above — but (a)/(b) are where a local 30B will keep trailing hardest.

---

## Phase 4: Make it act like Claude Code (the full build)

### The honest framing
When Claude Code "just works," three things are stacked together: (1) a frontier model that reliably calls tools and rarely fabricates, (2) a mature harness wired to execute, (3) strong judgment baked in. Locally you assemble these yourself. The model tier is the one piece you can't fully close on a 32B card — so the target is "as close as this hardware gets," which is genuinely good for daily work, with a hybrid escape hatch for the hard 5%.

The four steps below are in order of impact. Step 1 alone fixes most of what you saw.

---

### Step 1 — Change the model (biggest single win)

Everything observed so far — the "I only provide instructions" refusals, the fabricated output, the hallucinated `martinezmarcelo/crapid` image — is a **model-capability problem, not a config problem**. `qwen3-32b` is a *reasoning* model: it reasons *about* tools instead of reliably *calling* them. Swap it for a purpose-built tool-caller.

On the 5090 box, in LM Studio:
1. Go to the **Discover** tab and download one of:
   - **gpt-oss-20b** — community-reported cleanest tool-caller for unattended agent loops (start here)
   - **Devstral Small** (~24B, Apache 2.0) — purpose-built to drive coding agents
2. Load it via **Developer → Local Server**, and on the **Load** tab set **Context Length = 32768** before loading.
3. Confirm it shows **READY** and note the exact model id (e.g. `openai/gpt-oss-20b` — check the model list at `http://192.168.1.194:1234/v1/models`).

> Verify the current best tool-calling model before committing — the landscape moves monthly. The Ollama registry and the OpenHands leaderboard are good checks.

---

### Step 2 — Wire the harness to actually execute

**2a. Update `~/.config/opencode/config.json`** to point at the new model and set explicit permissions:

```json
{
  "provider": {
    "local5090": {
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "http://192.168.1.194:1234/v1",
        "apiKey": "not-needed"
      },
      "models": {
        "openai/gpt-oss-20b": { "tools": true, "contextWindow": 32768 }
      }
    }
  },
  "model": "local5090/openai/gpt-oss-20b",
  "permission": {
    "bash": "ask",
    "edit": "ask"
  }
}
```
`"ask"` = OpenCode prompts you before each command/edit (the right default for cloud ops — you see the exact `aws` call before it fires). Use `"allow"` only for low-risk unattended work.

**2b. Create `AGENTS.md` in the project root** (`~/opencode-test/AGENTS.md`) — this is where the behavior and safety rules live. Full contents in the appendix below.

**2c. The clean test.** Restart OpenCode and give it the unmistakable one:
> Use your bash tool to run `aws sts get-caller-identity` and show me the actual output.

- Real JSON with your account ID/ARN → **fixed**, move on.
- Still "you must run this yourself" → the model is still refusing; try the other model from Step 1.
- Tool-call error (malformed / not found) → harness/config issue, debug OpenCode tool config.

---

### Step 3 — Add the grounding/RAG layer (stops the hallucinations)

A frontier model recalls the real crAPI stack; a local one must **look it up**. Two MCP servers close this gap. Run both on the 5090 box.

**3a. SearXNG (live web search):**
```bash
docker run -d --name searxng -p 8080:8080 \
  -v ./searxng:/etc/searxng --restart unless-stopped searxng/searxng
# In searxng/settings.yml enable JSON output:  formats: [html, json]
```

**3b. Register search + AWS-docs MCP servers** in OpenCode config (`mcp` block):
```json
"mcp": {
  "searxng": {
    "command": "uvx",
    "args": ["mcp-searxng"],
    "environment": { "SEARXNG_URL": "http://192.168.1.194:8080" }
  },
  "aws-docs": {
    "command": "uvx",
    "args": ["awslabs.aws-documentation-mcp-server@latest"]
  }
}
```

**3c. (Optional, later) Curated doc RAG with Qdrant** for your own runbooks, using the `nomic-embed-text-v1.5` model you already have:
```bash
docker run -d -p 6333:6333 -v ./qdrant:/qdrant/storage --restart unless-stopped qdrant/qdrant
```
Ingest = chunk → embed via the local embedding model → upsert to Qdrant, exposed as a `search_docs` tool. Add this only once web-search grounding is working.

---

### Step 4 — Bake in the judgment you can't get from weights

Frontier models flag risky asks on their own; a 32B won't, so you encode the rules once in `AGENTS.md` (appendix). Key ones: never open security groups to `0.0.0.0/0` for lab targets, never show output not received from a tool, verify a resource exists before claiming it does, prefer Terraform/plan-then-apply over raw mutations, look up real project docs before deploying.

---

### Realistic expectation after all four
Handles day-to-day well — scripting, file edits, routine `aws`/`ssh`/`docker`, the debug-fix loop — executing for real and mostly not fabricating. Still trails Claude Code on long autonomous runs and subtle troubleshooting.

**Hybrid escape hatch:** keep a frontier API key configured as a second provider in OpenCode and switch to it (one flag) for the hard 5%; local for everything else.
```json
"provider": {
  "local5090": { "...": "as above" },
  "frontier": {
    "npm": "@ai-sdk/anthropic",
    "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" },
    "models": { "claude-opus-4-8": { "tools": true } }
  }
}
```

---

### Appendix — full `AGENTS.md`

```markdown
# Agent behavior and rules

## Execution
- You have a REAL bash tool that runs commands on this machine as the user.
- When asked to run something, CALL THE BASH TOOL and actually execute it.
- Never say you lack access. Never tell the user to run commands themselves
  as a substitute for running them. aws, az, ssh, docker, git are all
  installed and configured here.

## Truthfulness (anti-fabrication)
- Only report output that was literally returned by a tool call in THIS session.
- Never invent, predict, or pretty-print output you did not receive.
- If a result is missing, empty, or errored, say so plainly and stop —
  do not narrate what success "would" look like.
- Never claim a resource is created, running, or reachable unless a real
  command confirmed it. Verify state before asserting it.

## Loop discipline
- Run one command, read the actual result, then decide the next step.
- Do not emit menus of 4-6 hypothetical commands. Act, observe, report.

## Grounding
- Before deploying or configuring a named project, look up its real docs
  (web search / AWS docs tools) instead of guessing image names, ports, or
  flags. Example: crAPI is a multi-service docker-compose stack from the
  OWASP/crAPI repo, NOT a single Docker image.

## Cloud / security defaults
- Never open security group rules to 0.0.0.0/0 for lab or vulnerable targets.
  Scope to the user's own IP (curl https://checkip.amazonaws.com → CIDR/32).
- Prefer key-based SSH over passwords. Never echo secrets into commands that
  land in shell history or logs.
- Prefer Terraform/OpenTofu plan-then-apply over raw imperative mutations for
  anything beyond a one-off inspection.
- Tag lab resources with a clear prefix and include a teardown step so
  intentionally-vulnerable instances don't linger publicly.
```

---

## Open items / next steps

- [x] Confirm the context-length fix resolves the 400 error in OpenCode
- [x] Try a real task: have OpenCode write, run, and self-correct a small script
- [x] Connect `aws` CLI and attempt cloud provisioning (crAPI on EC2) — surfaced the fabrication issues documented in Phase 3
- [ ] **Step 1:** swap qwen3-32b → gpt-oss-20b (or Devstral) in LM Studio
- [ ] **Step 2:** update config.json + add AGENTS.md, run the `aws sts get-caller-identity` execution test
- [ ] **Step 3:** stand up SearXNG + AWS-docs MCP for grounding; Qdrant RAG later
- [ ] **Step 4:** confirm AGENTS.md safety rules are being respected on a real task
- [ ] Decide on authentication (API key) before ever exposing port 1234 beyond the home LAN
- [ ] Configure a frontier provider as the hybrid escape hatch for hard tasks
