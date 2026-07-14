---
name: claude-mem
description: Claude-mem knowledge and long-term memory workflow skills.
---

# Claude-mem


## Claude-mem / knowledge-agent

Source: https://github.com/thedotmack/claude-mem/tree/main/plugin/skills/knowledge-agent/SKILL.md

---
name: knowledge-agent
description: Build and query AI-powered knowledge bases from claude-mem observations. Use when users want to create focused "brains" from their observation history, ask questions about past work patterns, or compile expertise on specific topics.
---

# Knowledge Agent

Build and query AI-powered knowledge bases from claude-mem observations.

## What Are Knowledge Agents?

Knowledge agents are filtered corpora of observations compiled into a conversational AI session. Build a corpus from your observation history, prime it (loads the knowledge into an AI session), then ask it questions conversationally.

Think of them as custom "brains": "everything about hooks", "all decisions from the last month", "all bugfixes for the worker service".

## Workflow

### Step 1: Build a corpus

```text
build_corpus name="hooks-expertise" description="Everything about the hooks lifecycle" project="claude-mem" concepts="hooks" limit=500
```

Filter options:
- `project` — filter by project name
- `types` — comma-separated: decision, bugfix, feature, refactor, discovery, change
- `concepts` — comma-separated concept tags
- `files` — comma-separated file paths (prefix match)
- `query` — semantic search query
- `dateStart` / `dateEnd` — ISO date range
- `limit` — max observations (default 500)

### Step 2: Prime the corpus

```text
prime_corpus name="hooks-expertise"
```

This creates an AI session loaded with all the corpus knowledge. Takes a moment for large corpora.

### Step 3: Query

```text
query_corpus name="hooks-expertise" question="What are the 5 lifecycle hooks and when does each fire?"
```

The knowledge agent answers from its corpus. Follow-up questions maintain context.

### Step 4: List corpora

```text
list_corpora
```

Shows all corpora with stats and priming status.

## Tips

- **Focused corpora work best** — "hooks architecture" beats "everything ever"
- **Prime once, query many times** — the session persists across queries
- **Reprime for fresh context** — if the conversation drifts, reprime to reset
- **Rebuild to update** — when new observations are added, rebuild then reprime

## Maintenance

### Rebuild a corpus (refresh with new observations)

```text
rebuild_corpus name="hooks-expertise"
```

After rebuilding, reprime to load the updated knowledge:

### Reprime (fresh session)

```text
reprime_corpus name="hooks-expertise"
```

Clears prior Q&A context and reloads the corpus into a new session.



## Claude-mem / mem-search

Source: https://github.com/thedotmack/claude-mem/tree/main/plugin/skills/mem-search/SKILL.md

---
name: mem-search
description: Search claude-mem's persistent cross-session memory database. Use when user asks "did we already solve this?", "how did we do X last time?", or needs work from previous sessions.
---

# Memory Search

Search past work across all sessions. Simple workflow: search -> filter -> fetch.

## When to Use

Use when users ask about PREVIOUS sessions (not current conversation):

- "Did we already fix this?"
- "How did we solve X last time?"
- "What happened last week?"

## 3-Layer Workflow (ALWAYS Follow)

**NEVER fetch full details without filtering first. 10x token savings.**

### Step 1: Search - Get Index with IDs

Use the `search` MCP tool:

```
search(query="authentication", limit=20, project="my-project")
```

**Returns:** Table with IDs, timestamps, types, titles (~50-100 tokens/result)

```
| ID | Time | T | Title | Read |
|----|------|---|-------|------|
| #11131 | 3:48 PM | 🟣 | Added JWT authentication | ~75 |
| #10942 | 2:15 PM | 🔴 | Fixed auth token expiration | ~50 |
```

**Parameters:**

- `query` (string) - Search term
- `limit` (number) - Max results, default 20, max 100
- `project` (string) - Project name filter
- `type` (string, optional) - "observations", "sessions", or "prompts"
- `obs_type` (string, optional) - Comma-separated: bugfix, feature, decision, discovery, change
- `dateStart` (string, optional) - YYYY-MM-DD or epoch ms
- `dateEnd` (string, optional) - YYYY-MM-DD or epoch ms
- `offset` (number, optional) - Skip N results
- `orderBy` (string, optional) - "date_desc" (default), "date_asc", "relevance"

### Step 2: Timeline - Get Context Around Interesting Results

Use the `timeline` MCP tool:

```
timeline(anchor=11131, depth_before=3, depth_after=3, project="my-project")
```

Or find anchor automatically from query:

```
timeline(query="authentication", depth_before=3, depth_after=3, project="my-project")
```

**Returns:** `depth_before + 1 + depth_after` items in chronological order with observations, sessions, and prompts interleaved around the anchor.

**Parameters:**

- `anchor` (number, optional) - Observation ID to center around
- `query` (string, optional) - Find anchor automatically if anchor not provided
- `depth_before` (number, optional) - Items before anchor, default 5, max 20
- `depth_after` (number, optional) - Items after anchor, default 5, max 20
- `project` (string) - Project name filter

### Step 3: Fetch - Get Full Details ONLY for Filtered IDs

Review titles from Step 1 and context from Step 2. Pick relevant IDs. Discard the rest.

Use the `get_observations` MCP tool:

```
get_observations(ids=[11131, 10942])
```

**ALWAYS use `get_observations` for 2+ observations - single request vs N requests.**

**Parameters:**

- `ids` (array of numbers, required) - Observation IDs to fetch
- `orderBy` (string, optional) - "date_desc" (default), "date_asc"
- `limit` (number, optional) - Max observations to return
- `project` (string, optional) - Project name filter

**Returns:** Complete observation objects with title, subtitle, narrative, facts, concepts, files (~500-1000 tokens each)

## Examples

**Find recent bug fixes:**

```
search(query="bug", type="observations", obs_type="bugfix", limit=20, project="my-project")
```

**Find what happened last week:**

```
search(type="observations", dateStart="2025-11-11", limit=20, project="my-project")
```

**Understand context around a discovery:**

```
timeline(anchor=11131, depth_before=5, depth_after=5, project="my-project")
```

**Batch fetch details:**

```
get_observations(ids=[11131, 10942, 10855], orderBy="date_desc")
```

## Why This Workflow?

- **Search index:** ~50-100 tokens per result
- **Full observation:** ~500-1000 tokens each
- **Batch fetch:** 1 HTTP request vs N individual requests
- **10x token savings** by filtering before fetching

## Knowledge Agents

Want synthesized answers instead of raw records? Use `/knowledge-agent` to build a queryable corpus from your observation history. The knowledge agent reads all matching observations and answers questions conversationally.



## Claude-mem / how-it-works

Source: https://github.com/thedotmack/claude-mem/tree/main/plugin/skills/how-it-works/SKILL.md

---
name: how-it-works
description: Explain how claude-mem captures observations, when memory injection kicks in, and where data lives. Use when the user asks "how does claude-mem work?" or "what is this thing doing?".
---

# How claude-mem works

## What it does

Every Read, Edit, and Bash that Claude makes turns into a compressed observation. Observations get summarized at session end. Relevant ones get auto-injected into future prompts so the next session starts with context from the last one — no re-explaining the codebase, no re-discovering decisions.

## When it kicks in

Memory injection starts on your second session in a project.

The first session in a fresh project seeds memory; subsequent sessions receive auto-injected context for relevant past work. Run `/learn-codebase` if you want to front-load the entire repo into memory in a single pass (~5 minutes, optional).

## Where data lives

Everything stays in ~/.claude-mem on this machine.

Nothing leaves your machine except calls to whichever AI provider you configured for compression (Claude / OpenRouter / Gemini). The SQLite database, vector index, logs, and settings all live under that directory and are removed cleanly on `npx claude-mem uninstall`.
