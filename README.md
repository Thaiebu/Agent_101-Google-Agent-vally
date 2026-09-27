# Agent 101 — The Archive (Agent Valley)

> Lab & Learning Notes for **Agent 101 Live: Remember (Session State, Memory Bank, RAG & GraphRAG)**.

The full week 4 implementation, scripts, and interactive Next.js + FastAPI application live in [`week4/`](file:///Users/mohamedthaiebu/Documents/Agent%20101/week4).

---

## Quick Navigation

- **Detailed Week 4 Guide & Code**: [week4/README.md](file:///Users/mohamedthaiebu/Documents/Agent%20101/week4/README.md)
- **Detailed Learning Notes**: [week4/Thing_I_learned_from_seesion.md](file:///Users/mohamedthaiebu/Documents/Agent%20101/week4/Thing_I_learned_from_seesion.md)
- **Agent Workflow Graph**: [week4/archive/agent.py](file:///Users/mohamedthaiebu/Documents/Agent%20101/week4/archive/agent.py)
- **State Definitions**: [week4/archive/state.py](file:///Users/mohamedthaiebu/Documents/Agent%20101/week4/archive/state.py)
- **Backend Service**: [week4/archive/service.py](file:///Users/mohamedthaiebu/Documents/Agent%20101/week4/archive/service.py)
- **Floor 4 (BigQuery RAG & GraphRAG)**: [week4/archive/season/__init__.py](file:///Users/mohamedthaiebu/Documents/Agent%20101/week4/archive/season/__init__.py)

---

## Quick Start (Week 4)

```bash
cd week4
uv sync
cp .env.example .env
uv run python scripts/preflight.py
bash valley.sh
```

- **Next.js Web UI**: [http://localhost:3440](http://localhost:3440)
- **FastAPI Agent Service**: [http://127.0.0.1:8440](http://127.0.0.1:8440)

---

## Executive Summary: How AI Agents Remember

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

### The 5 Memory Tiers

1. 🪑 **The Desk (Session State)**: In-memory session state (`session.state[CASE]`) for volatile facts within the current conversation.
2. 📚 **Floor 1 (Persisted Sessions)**: `SqliteSessionService` storing conversations in `archive.db` so interactions survive server restarts.
3. 🗄️ **Floor 2 (User State)**: Prefixed keys (`user:case`) extending lifetime to follow a visitor across independent sessions.
4. 🏛️ **Floor 3 (Memory Bank)**: `VertexAiMemoryBankService` for semantic extraction and recall based on meaning rather than verbatim keywords.
5. 🌄 **Floor 4 (RAG + GraphRAG)**: BigQuery `VECTOR_SEARCH` (similarity lookup) paired with Property Graph `GRAPH_TABLE` GQL MATCH (relationship walks across items, marks, and fixes).
