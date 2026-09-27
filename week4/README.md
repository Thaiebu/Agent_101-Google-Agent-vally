# The Archive — Agent Valley, week four

> A tower that writes everything down, and an archivist who is pleased to meet you every single time.

This is the lab repository for **Agent 101 Live, chapter four: Remember**. You climb four rungs of agent memory — a visit, a visitor, what was said, the whole valley — and every rung up happens because something fell out of the one below, on screen, where you can watch it fall.

```
floor 4   the season        the whole valley   BigQuery, embedded in place chapter 5
floor 3   the tower         what was said      Vertex AI Memory Bank       chapters 3-4
floor 2   your drawer       this visitor       the `user:` prefix          chapter 2
floor 1   the books         every visit        SqliteSessionService        chapter 1
the desk  the slip          this visit         session.state               chapter 1
```

Three edits, each one line; one tower you build and connect; one warehouse you load. The codelab is the guide; this is what it drives.

---

## Quick Start: Run It

```bash
uv sync
cp .env.example .env
uv run python scripts/preflight.py     # is this machine ready, and where am I?
bash valley.sh                         # the Archive: agent 8440 + app 3440
```

Open **http://localhost:3440**. Press **▶ Start**, then **The Archive**, and pick whoever has come to ask.

The workbench is the other window onto the same agent:

```bash
uv run adk web --session_service_uri=sqlite:///archive.db . --allow_origins="*"
```

Same file, same graph, two surfaces. Nothing in `archive/` knows which one is looking at it.

---

# Summary: How AI Agents Remember
### Session State, Memory Bank, RAG & GraphRAG Explained

*An agent doesn't need one memory system — it needs different layers for different problems.*

### The Question That Started This

**How does an AI agent actually remember things?**

A common assumption is that the answer is simple: save the conversation to a database, retrieve it later, done.

In building **The Archive** — where an archivist deer named Vesper runs a memory tower with visitors bringing in odd items to be "filed" — it becomes clear that memory isn't just one thing. It's several different problems wearing the same name.

### A Story, Not a Definition

Picture a visitor walking into Vesper's archive:

```
Visitor talks to Vesper
        ↓
"Remember my case number, please"
        ↓
That's Session State (The Desk)

Visitor returns next week
        ↓
"Do you remember me?"
        ↓
That's User State / Persistent Memory (Floor 2)

The archive holds 100,000 old cases
        ↓
"Have you seen something like this before?"
        ↓
That's RAG / Semantic Vector Search (Floor 4)

"Which other visitors had the same mark, and what fixed it?"
        ↓
That's GraphRAG / Graph Traversal (Floor 4)
```

Each question is reasonable, but each one requires a completely different kind of memory to answer.

### The Mental Model

```
                 AI AGENT
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      STATE       MEMORY       RAG
        │           │           │
     Current       User      Organization
   conversation   history     knowledge
                                │
                         ┌──────┴──────┐
                         ↓             ↓
                       Vector        Graph
                       Search       Traversal
                                       │
                                   GraphRAG
```

> **State remembers what's happening. Memory remembers what matters. RAG retrieves knowledge. GraphRAG finds relationships.**

---

## The 5-Floor Memory Architecture

In this project, the architecture is physically represented as five floors of an interactive tower:

| Floor | Memory Tier | Scope | Purpose | Implementation Detail |
| :--- | :--- | :--- | :--- | :--- |
| 🪑 **The Desk** | **Session State** | Current visit only | Prevents repeating data within one conversation | `write_down` tool fills `session.state[CASE]` (`no`, `item`, `mark`, `symptom`) |
| 📚 **Floor 1** | **Persisted Sessions** | Across app restarts | Conversation survives crashes/restarts | `SqliteSessionService` saves to `archive.db` on disk |
| 🗄️ **Floor 2** | **User State** | Specific visitor across visits | Recognizes returning visitors in fresh sessions | `user:` prefix convention (e.g. `user:case`) extends lifetime |
| 🏛️ **Floor 3** | **Memory Bank** | What was said (semantic) | Extracts distilled facts by meaning, not exact wording | `VertexAiMemoryBankService`, `search_memory(query)`, topic-filtered `file` |
| 🌄 **Floor 4** | **RAG + GraphRAG** | The whole valley (all users) | Answers cross-user similarity and multi-hop relationships | BigQuery `VECTOR_SEARCH` (RAG) + Property Graph `GRAPH_TABLE` GQL (GraphRAG) |

### 1. 🪑 The Desk — Session State
- **What:** Temporary info tied strictly to the *current* conversation.
- **Why:** So Vesper doesn't ask you to repeat your case number five times in one visit.
- **Implementation:** The `write_down` tool in `archive/agent.py` populates fields directly into `session.state[CASE]` (`no`, `item`, `mark`, `symptom`), mirroring a physical paper intake slip.

### 2. 📚 Floor 1 — Persisted Sessions
- **What:** Conversation turns persist on disk, not just in volatile RAM.
- **Why:** Closing the app or restarting the server does not wipe out conversation progress.
- **Implementation:** `SqliteSessionService` persists sessions and events into `archive.db`.

### 3. 🗄️ Floor 2 — User State
- **What:** Information scoped to *this visitor*, across multiple sessions.
- **Why:** A returning visitor is greeted with their past records rather than treated as a complete stranger.
- **Implementation:** Changing `CASE` in `archive/state.py` to use a `user:` prefix (`user:case`) instructs the session service to merge this dictionary across every session belonging to this `user_id`.

### 4. 🏛️ Floor 3 — Memory Bank
- **What:** Long-term semantic facts extracted from raw conversation history.
- **Why:** Full transcripts are verbose and noisy; semantic memory distills concise facts queryable by meaning.
- **Implementation:** Powered by `VertexAiMemoryBankService`. The `recall` node runs `await ctx.search_memory(query)` before Vesper speaks, injecting relevant cards into context. At closing time, `file` runs `await ctx.add_events_to_memory(...)` with strict topic filtering defined in `archive/topics.py`.

### 5. 🌄 Floor 4 — RAG + GraphRAG
- **What:** Cross-organization knowledge spanning everyone's past visits.
- **Why:** Solves questions requiring historical similarity ("has anyone had a lantern that dims at dusk?") and causal entity chains ("which other items carried mark X, who brought them, and what fix solved it?").
- **Implementation:**
  - `season_search(text)`: Runs BigQuery `VECTOR_SEARCH` over embedded descriptions (**RAG**).
  - `season_known_issue(mark)`: Traverses connections via BigQuery Property Graph query `GRAPH_TABLE(...) MATCH (m:Mark)<-[:stamped]-(i:Item)<-[a:asked]-(v:Visitor)` with a fallback to 3-way SQL joins if graph reservations are unavailable (**GraphRAG**).

---

## Vector Search vs Graph Traversal

> **Vector search finds things that are semantically similar.**  
> **Graph traversal finds things that are connected.**

In `archive/season/__init__.py`, both approaches sit side by side as complementary tools:
- Vector search identifies which past issue descriptions *sound* like the current complaint to discover candidate marks.
- Graph traversal walks the relationships from the discovered mark to verified fixes and historical root causes.

---

## System Architecture

```
Next.js Frontend (Port 3440: Animated Tower UI & Chat)
        ↓  Server-Sent Events (SSE)
FastAPI Backend (Port 8440: archive/service.py)
        ↓
ADK Workflow: route ──┬──▶ recall ──▶ vesper ──▶ write_down
                      ├──▶ file   ──▶ goodnight
                      └──▶ look   ──▶ vesper (Multimodal artifacts)
        ↓                                    ↓
  Vertex AI Memory Bank (Floor 3)    Session & User State (Desk, Floor 1–2)
        ↓
  BigQuery Vector Search & Property Graph GQL (Floor 4)
```

### Key Engineering Patterns
1. **Deterministic routing before the LLM:** Commands like `[close]` and `[show]` route via code before invoking the model, keeping execution fast, cheap, and deterministic.
2. **Recall before generation:** Memory retrieval happens as a graph node *before* the agent generates a response, eliminating hallucinated or missed lookups.
3. **Write policies & topic filtering:** Long-term memory extraction is batched at closing time and scoped strictly by topic whitelist (`archive/topics.py`).
4. **Graceful degradation:** Floor 4's graph query gracefully falls back to SQL joins if enterprise graph reservations are not enabled.

---

## What is Where

```
archive/
  agent.py      the graph: route → recall → vesper | file → goodnight | look
  state.py      every key the Archive remembers, and how long each one lives
  service.py    the app's back end — one agent, re-imported on every message
  progress.py   how the app knows which edits you have made (it reads your code)
  tower.py      listing and forgetting: the two things ADK's memory API cannot do
  topics.py     what the tower is allowed to keep
  season/       chapter 5 — two fixed queries: VECTOR_SEARCH and GQL MATCH
scripts/
  preflight.py      am I ready, and which floors are lit
  make_tower.py     build the Agent Engine that Memory Bank lives on, topics and all
  season_csv.py     the valley's past as five CSVs
  season.sh         GCS → BigQuery → embeddings in place → property graph over tables
  shelf.py          floor three from the terminal — list the cards, or burn one
  walk.py           walk the codelab chapter by chapter against the real agent
site/             the Archive itself — Next.js, talks to the service over SSE
```

---

## Verification: The One That Keeps Everyone Honest

```bash
uv run python scripts/walk.py        # all six chapters, plus the tower when .env names one
uv run python scripts/walk.py 3      # just one chapter
```

`walk.py` is not just a unit test. Every assertion in it is a **sentence the codelab says to the learner**, applied to the real agent with edits applied and undone.

---

## Core Takeaway

> **Don't start by choosing a memory technology. Start by understanding the memory problem:**
> 1. *What* does the agent need to remember?
> 2. *For how long* (turn, session, user, or organization-wide)?
> 3. *How* will it need to retrieve that information (by key, by meaning, or by relationship)?
