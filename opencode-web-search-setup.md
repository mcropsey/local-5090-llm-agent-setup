# Adding Web Search to opencode (Local LLM) — Step by Step

Goal: when the local model's training data is stale, have it look up current info on the
internet instead of confidently handing you outdated versions, flags, or config.

**How it works:** opencode gets a web-search tool via a **Tavily MCP server**. A rule in
`AGENTS.md` tells the model *when* to reach for it. The MCP provides the capability; the
rule provides the trigger. You need both.

---

## Step 1 — Get a Tavily API key

1. Sign up at **tavily.com**.
2. Free tier = **1,000 searches/month** (plenty for personal use).
3. Copy your API key.

> Heads up: the free tier can run out fast on research-heavy days. If you hit the limit,
> see "Alternative: self-hosted Searxng" at the bottom.

---

## Step 2 — Put the key in your shell environment

Don't hardcode the key in a config file. Add to `~/.bashrc` / `~/.zshrc`:

```bash
export TAVILY_API_KEY="tvly-xxxxxxxxxxxxxxxx"
```

Then reload: `source ~/.bashrc` (or open a new terminal).

---

## Step 3 — Register the MCP server in opencode

Edit `~/.config/opencode/opencode.json` and add the `mcp` block. If the file already has
other config (e.g. your LM Studio provider), just add the `mcp` key alongside it — don't
overwrite what's there.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "tavily": {
      "type": "local",
      "command": ["npx", "-y", "tavily-mcp"],
      "environment": {
        "TAVILY_API_KEY": "{env:TAVILY_API_KEY}"
      }
    }
  }
}
```

**Config-format gotcha:** most blog/Claude-Code examples show a `mcpServers` block with
`command` + `args` as separate fields. opencode uses a *different* schema — `mcp`, with
`type: "local"` and `command` as a single array. If the server won't load, this mismatch is
almost always why. Check opencode's own MCP docs for the current exact keys — this part of
the format drifts between versions.

Provides two tools once loaded: `tavily_search` (web search) and `tavily_extract`
(scrape a specific URL).

---

## Step 4 — Restart opencode

The MCP is read at startup. Fully restart opencode (not just a new session in the same
process). `npx -y` will auto-download `tavily-mcp` the first time — first launch may pause
a few seconds while it fetches.

---

## Step 5 — Add the trigger rule to AGENTS.md

This is the part that actually makes a local model *use* the tool. Without it, the model
has web search available but keeps answering from stale memory anyway.

Add to `~/.config/opencode/AGENTS.md` (global) or your project `AGENTS.md`:

```markdown
## When to search the web
Your training data has a cutoff and is often out of date. Before answering anything about
current versions, package/CLI flags, library APIs, or how a public project is configured
today, assume your knowledge MAY be stale and use the tavily search tool to check.

Search — do not guess — whenever:
- You're about to state a version number, port, flag, or install command.
- The user mentions something released or changed recently.
- You're unsure whether what you "know" is still current.

Quote the search results. If results contradict your training data, trust the results and
say what changed.
```

---

## Step 6 — Test it

In an opencode session:

```
What's the current recommended way to run OWASP crAPI? Search first — my notes may be stale.
```

**Pass:** it fires a Tavily search and quotes real, current results.
**Fail:** it recites an answer from memory (e.g. the fake single-image crAPI answer) without
searching — revisit Step 5, and try naming the tool explicitly: "Use tavily to search for…".

---

## Important reality check

**A local model can't reliably tell when its own info is outdated.** Stale facts feel
exactly as confident to it as current ones — the same over-claiming weakness behind the
"container is running" hallucination. So:

- The AGENTS.md rule deliberately **over-triggers** — it tells the model to search
  proactively on anything version/flag/config-shaped, rather than trying to self-detect
  staleness (which it's bad at). Expect some unnecessary searches; that's the right trade.
- The **100%-reliable** version is to force it per-prompt when *you* know something's
  time-sensitive: *"Search the web for the current X before you answer."* You know the info
  is likely stale; the model often doesn't.

With a 30B local model the *retrieving* is reliable; the *deciding-to-retrieve* leans on
your AGENTS.md nudge plus your per-prompt hints. A frontier model self-triggers better, but
this gets you most of the way.

---

## Web search vs. RAG — don't confuse them

These solve different problems; you may want both:

| Need | Tool | Fixes |
|---|---|---|
| Current *public* facts (real compose files, latest flags, library changes) | **Tavily web search** (this doc) | Stale training data |
| Your *own* facts (your IPs, your ports, crAPI is a stack not an image) | **Local RAG** over a `~/lab-kb` folder | Inventing *your* infra details |

Web search won't fix hallucinations about *your* lab specifics — those aren't on the
internet. For those, a local docs folder (keyword `rg` search or a local-RAG MCP) is the
fix. Running both together is the combo that actually kills the hallucinations.

---

## Alternative: self-hosted Searxng (no API key, no rate limit)

If you outgrow Tavily's free tier or want privacy/no external API:

- **Searxng** is a self-hosted metasearch engine aggregating 70+ engines. No API key.
- Setup is two parts: deploy Searxng (a container), then point a Searxng MCP server at it.
- Natural fit for your 5090 box later — same spirit as self-hosting embeddings/Qdrant.

Trade-off: more moving parts than Tavily's 5-minute setup. Start with Tavily; switch to
Searxng only if rate limits or privacy push you there.

---

## Quick reference

```bash
# 1. Key in shell
export TAVILY_API_KEY="tvly-..."

# 2. opencode.json  →  mcp.tavily  (type: local, command array, env var)
# 3. AGENTS.md      →  "When to search the web" rule
# 4. Restart opencode

# Force a search any time you know info is time-sensitive:
#   "Search the web for the current <thing> before you answer."
```
