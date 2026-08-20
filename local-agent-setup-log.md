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

## Open items / next steps

- [ ] Confirm the context-length fix resolves the 400 error in OpenCode
- [ ] Try a real task: have OpenCode write, run, and self-correct a small script
- [ ] Decide on authentication (API key) before ever exposing port 1234 beyond the home LAN
- [ ] Phase 3 (future): connect `aws`/`az` CLI tools so the agent can provision cloud resources
- [ ] Phase 4 (future): add RAG (SearXNG + Qdrant, using the already-downloaded `nomic-embed-text-v1.5` model) so the agent can fill knowledge gaps from the web
