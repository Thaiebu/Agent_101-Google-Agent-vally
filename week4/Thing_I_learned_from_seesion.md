Solid feedback — this is exactly the kind of edit that turns a technically-correct writeup into something people actually finish reading. Here's the rewritten version following that flow: story first, terminology second, implementation details tucked into clearly-marked subsections, and a more personal takeaway.

---

# How AI Agents Remember: Session State, Memory Bank, RAG & GraphRAG Explained

*I learned that an agent doesn't need one memory system — it needs different layers for different problems.*

---

## The Question That Started This

**How does an AI agent actually remember things?**

I used to assume the answer was simple: save the conversation to a database, retrieve it later, done.

Then I built a project called **The Archive** — part of Google's Agent Valley learning series — where an archivist deer named Vesper runs a memory tower with visitors bringing in odd items to be "filed." Working through it, I realized my original answer was wrong. Or at least, incomplete.

Memory isn't one thing. It's several different problems wearing the same name.

---

## A Story, Not a Definition

Picture a visitor walking into Vesper's archive.

```
Visitor talks to Vesper
        ↓
"Remember my case number, please"
        ↓
That's Session State

Visitor returns next week
        ↓
"Do you remember me?"
        ↓
That's User State / Memory

The archive holds 100,000 old cases
        ↓
"Have you seen something like this before?"
        ↓
That's RAG

"Which other visitors had the same mark,
and what fixed it?"
        ↓
That's GraphRAG
```

Notice something: each question is *reasonable*, but each one needs a completely different kind of memory to answer. That's the whole insight this article is built around.

Now let's put names to each layer.

---

## The Aha Diagram

Before going floor by floor, here's the mental model that ties everything together:

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

Keep that sentence in your head — every section below is just an expansion of it.

---

## The 5-Floor Architecture

In the project, this isn't just a metaphor — it's literally represented as five floors of a tower, each one lighting up as you implement it.

🪑 **The Desk** — this visit
📚 **Floor 1** — every visit
🗄️ **Floor 2** — this visitor
🏛️ **Floor 3** — what was said
🌄 **Floor 4** — the whole valley

Let's go floor by floor.

---

### 🪑 The Desk — Session State

**What?** Temporary info tied to the *current* conversation only.

**Why?** So Vesper doesn't ask you to repeat your case number five times in one visit.

**Example:** You mention your item once — for the rest of *this* conversation, it's already noted on the slip.

<details>
<summary>Behind the implementation</summary>

A `write_down` tool fills fields directly into `session.state[CASE]` — fields like `no`, `item`, `mark`, `symptom` — mirroring a physical intake slip.
</details>

---

### 📚 Floor 1 — Persisted Sessions

**What?** The conversation itself survives, not just while it's held in memory.

**Why?** So closing the app — or the server restarting — doesn't wipe out the conversation.

**Example:** You close mid-conversation, come back later, and Vesper picks up exactly where you left off.

<details>
<summary>Behind the implementation</summary>

`SqliteSessionService` writes sessions into a real database file (`archive.db`) instead of keeping them only in RAM.
</details>

---

### 🗄️ Floor 2 — User State

**What?** Info that belongs to *you specifically*, across every visit — not just today's.

**Why?** So a returning visitor is recognized, instead of treated as a stranger every time.

**Example:** You visited a month ago; Vesper still remembers you on your next visit, even though it's a brand-new session.

<details>
<summary>Behind the implementation</summary>

Same state mechanism as the Desk — but the key gets a `user:` prefix (e.g. `user:case`). That prefix alone is what changes the *lifetime* of the data from "this chat" to "this person."
</details>

---

### 🏛️ Floor 3 — Memory Bank

**What?** Distilled facts from past conversations, retrieved by *meaning*, not exact wording.

**Why?** Raw transcripts are messy. Semantic memory keeps only the useful nuggets.

**Example:** You mentioned a lantern issue weeks ago. Even phrased totally differently today, Vesper still recalls it — because it's searching by meaning.

<details>
<summary>Behind the implementation</summary>

`VertexAiMemoryBankService` handles storage and extraction. A `recall` step runs `search_memory(query)` *before* the agent responds, so retrieved memories are already available. A `file` step commits new memories at the end of the day, filtered by topic so noise doesn't pile up.
</details>

---

### 🌄 Floor 4 — RAG + GraphRAG

**What?** Knowledge across *everyone's* history — not just one person's memory.

**Why?** Some questions can only be answered by looking across all visitors: "who else had this, and what fixed it?"

**Example:**
> "Which visitors reported this same mark, and what was the outcome?"

Vector search alone finds *similar complaints*. Graph traversal finds the *actual chain* connecting mark → item → visitor.

<details>
<summary>Behind the implementation</summary>

- `season_search(text)` runs `VECTOR_SEARCH` in BigQuery — this is the RAG half.
- `season_known_issue(mark)` runs a BigQuery Property Graph query (`GRAPH_TABLE(...) MATCH (m:Mark)<-[:stamped]-(i:Item)<-[a:asked]-(v:Visitor)`) — this is the GraphRAG half, with an automatic SQL fallback if graph reservations aren't available.
</details>

---

## Vector Search vs Graph Search

This is probably the single most useful sentence in this whole article:

**Vector search finds things that are semantically similar.**
**Graph traversal finds things that are connected.**

In the project, these sit side by side as two functions in the same file — a clean real-world example of the distinction.

---

## The Complete Architecture

```
Next.js Frontend (chat + animated tower UI)
        ↓  SSE stream
FastAPI Backend
        ↓
ADK Workflow: route → recall → vesper → write_down → file/look
        ↓                              ↓
  Memory Bank (Floor 3)        Session/User State (Desk, Floor 1–2)
        ↓
  BigQuery Vector Search + Property Graph (Floor 4)
```

A few design choices worth calling out, because they're good engineering lessons on their own:

- **Deterministic routing before the LLM.** Commands like `[close]` or `[show]` are caught *before* reaching Vesper — cheap, predictable logic stays cheap and predictable; only ambiguous language goes to the model.
- **Recall happens before generation**, so the agent isn't guessing whether to look something up mid-response.
- **Memory writes are batched and filtered**, not saved continuously — keeping Memory Bank clean instead of noisy.
- **Graceful degradation.** The GraphRAG query falls back to plain SQL if graph reservations aren't available — a reminder that "works reliably" beats "uses the fanciest feature."

---

## Why Not Just One System for Everything?

It's tempting to think: *"Why not just dump everything into a vector database?"*

But each question in the story above needs a different tool:

- Remember the current case → **The Desk**
- Survive a restart → **Floor 1**
- Recognize a returning visitor → **Floor 2**
- Recall facts by meaning → **Floor 3**
- Answer questions across everyone's history, including relationships → **Floor 4**

---

## What This Changed for Me

Before this project, I mostly thought of agent memory as:

**"Store previous conversations and retrieve them later."**

Now I think about it differently. The first question isn't "which memory technology should I use?" It's:

**What does the agent need to remember?**

Then:

**For how long?**

And finally:

**How will it need to retrieve that information?**

A case number belongs in session state. A user's preference belongs in long-term memory. Organization-wide documentation needs RAG. Questions about relationships between historical entities need GraphRAG.

That's the biggest takeaway from this project for me:

> **Don't start by choosing a memory technology. Start by understanding the memory problem.**

---

## How I'd Apply This in a Real Agent

- **Session State** → the current conversation
- **Persisted Sessions** → survive restarts, no data loss
- **User State** → personalize across visits via a simple prefix convention
- **Memory Bank** → long-term semantic memory, filtered and batched
- **RAG + GraphRAG** → organization-wide knowledge, both by similarity and by relationship
- **Deterministic routing + callbacks** → keep cheap logic cheap, automate memory instead of hardcoding it

Each layer only gets added once the problem actually requires it — that's the pattern worth remembering more than any single API.

---

## GitHub

The full project — **The Archive (Agent Valley, Week 4: "Remember")** — is available here:

🔗 **[GitHub Repository — link to be added]**

*(Drop your repo URL in here once it's pushed.)*

---

## References

- Google Agent Valley Archive — [InMemoryMemoryService](https://codelabs.developers.google.com/codelabs/agent-valley-archive/instructions#4)
- Google Agent Valley Archive — [Memory Bank](https://codelabs.developers.google.com/codelabs/agent-valley-archive/instructions#5)
- Google Agent Valley Archive — [GraphRAG](https://codelabs.developers.google.com/codelabs/agent-valley-archive/instructions#6)
- Agent 101 series — session on Agent Memory and GraphRAG with Annie and Abun
- Personal project build: **The Archive** (Agent Valley, Week 4)

*This article is based on my learning while working through Google's Agent Valley Archive codelab, the Agent 101 session on agent memory, and my own implementation/edits of The Archive project. The underlying concepts and original codelab belong to their respective authors/projects; the project build and analysis reflect my own work extending it.*

#GenerativeAI #AgenticAI #RAG #GraphRAG #AIEngineering #GoogleCloud #LLM #GenAI #ADK

---

Want me to also rewrite the shorter LinkedIn post version to match this new story-first hook, or save this as a Markdown/Word file now so it's ready to publish once your repo link is in?