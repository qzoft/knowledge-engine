# Feature Plan: Discussion-Driven Knowledge Harvesting

**Priority**: Critical  
**Depends On**: Smart Capture (#8) — reuses auto-relate and category guessing  
**Integrates With**: Staleness Metrics (#3) — stale docs trigger discussion refresh; Wikilinks (#1) — insights reference existing knowledge; Explicit Relationships (#6) — harvested docs inherit discovered relationships; Import Pipeline (#9) — discussions can seed from imported external content  
**Theory**: Socratic Method + Progressive Distillation — structured dialogue refines raw thinking into verified knowledge

---

## Overview

The Knowledge Engine currently has a clean **capture → index → search → retrieve** pipeline. But knowledge doesn't start as clean markdown — it starts as messy conversations, debates, half-formed ideas, and exploratory reasoning. Today, the `technical-discussion.agent.md` agent handles this "dynamic phase" as a standalone VS Code agent mode that writes files to disk. This feature brings the discussion loop **inside the MCP server itself**, making it available to any LLM client and storing discussion state in the Knowledge Engine's database.

### The Problem

1. The discussion agent writes `.discussions/` files to disk — outside the Knowledge Engine's control, not searchable, not versioned, not relationship-aware
2. The "send to knowledge engine" step is manual and lossy — the structured discussion (Facts/Opinions/Assumptions/Questions) is discarded, only the free-form file survives
3. No client besides VS Code agent mode can use the discussion workflow
4. There's no feedback loop — the Knowledge Engine can't suggest "this topic needs discussion" based on staleness, contradictions, or gaps

### The Solution

New MCP tools that implement a **Discussion → Structured Insights → Knowledge Draft → Review → Approved Knowledge** pipeline, with state stored in SQLite and the structured discussion preserved as metadata alongside the final document.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Discussion Lifecycle                          │
│                                                                 │
│  ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐ │
│  │  Start    │───→│  Discuss  │───→│ Harvest  │───→│ Review   │ │
│  │ Discussion│    │  (iterate)│    │ to Draft │    │ & Approve│ │
│  └──────────┘    └───────────┘    └──────────┘    └──────────┘ │
│       ↑                                                 │       │
│       └──── stale doc / gap detected / contradiction ───┘       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Data Model — Discussion State

### Core Concepts

A **Discussion** is a structured reasoning session about a topic. It has:
- A **topic** (human-readable name)
- A **status** (`active`, `paused`, `harvested`, `archived`)
- An ordered list of **Insights** classified into types
- An optional link to a **source document** (for refresh discussions on existing knowledge)
- An optional link to a **harvested document** (the knowledge doc produced)

An **Insight** is an atomic piece of information surfaced during discussion:
- Classified as: `fact`, `opinion`, `assumption`, `open_question`, `key_concept`, `risk`
- Has a confidence level: `high`, `medium`, `low`
- May reference existing documents (via file_path)
- Tracks provenance: who contributed it, when, from what source

### Why Store in SQLite (Not Files)

- Discussions are **transient working state**, not canonical knowledge — they shouldn't clutter the knowledge directory
- SQLite gives us **queryable structure** — find all open questions across discussions, find discussions about a topic
- Discussion state is **client-agnostic** — works from VS Code, Claude Desktop, or any MCP client
- The Knowledge Engine can **cross-reference** discussions with existing documents (e.g., "3 discussions reference this stale doc")

---

## 2. Database Schema — `fts_store.py`

### New Table: `discussions`

```sql
CREATE TABLE IF NOT EXISTS discussions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    topic TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    source_file_path TEXT,
    harvested_file_path TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    summary TEXT DEFAULT '',
    FOREIGN KEY (source_file_path) REFERENCES documents(file_path) ON DELETE SET NULL
);

CREATE INDEX IF NOT EXISTS idx_discussions_status ON discussions(status);
CREATE INDEX IF NOT EXISTS idx_discussions_topic ON discussions(topic);
```

### New Table: `discussion_insights`

```sql
CREATE TABLE IF NOT EXISTS discussion_insights (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    discussion_id INTEGER NOT NULL,
    type TEXT NOT NULL,
    content TEXT NOT NULL,
    confidence TEXT NOT NULL DEFAULT 'medium',
    source_description TEXT DEFAULT '',
    referenced_file_path TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    resolved BOOLEAN NOT NULL DEFAULT 0,
    resolved_as TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    FOREIGN KEY (discussion_id) REFERENCES discussions(id) ON DELETE CASCADE,
    FOREIGN KEY (referenced_file_path) REFERENCES documents(file_path) ON DELETE SET NULL
);

CREATE INDEX IF NOT EXISTS idx_insights_discussion ON discussion_insights(discussion_id);
CREATE INDEX IF NOT EXISTS idx_insights_type ON discussion_insights(type);
CREATE INDEX IF NOT EXISTS idx_insights_resolved ON discussion_insights(resolved);
```

### Insight Type Vocabulary

| Type | Description | Example |
|------|-------------|---------|
| `fact` | Verifiable truth, confirmed information | "PostgreSQL uses MVCC for concurrency" |
| `opinion` | Subjective interpretation or preference | "Event sourcing is overkill for CRUD apps" |
| `assumption` | Unverified belief taken as given | "Our traffic won't exceed 10k RPS" |
| `open_question` | Unresolved question needing exploration | "How does this interact with connection pooling?" |
| `key_concept` | Important term, framework, or idea | "Circuit Breaker pattern — prevents cascade failures" |
| `risk` | Identified risk, gap, or uncertainty | "No rollback plan if migration fails" |

---

## 3. FTS Store Methods — `fts_store.py`

### Discussion CRUD

```python
def create_discussion(self, topic: str, source_file_path: str | None = None) -> dict:
    """Create a new discussion. Returns {id, topic, status, created_at}."""

def get_discussion(self, discussion_id: int) -> dict | None:
    """Get discussion with all insights grouped by type."""

def list_discussions(self, status: str | None = None) -> list[dict]:
    """List discussions, optionally filtered by status. Includes insight counts."""

def update_discussion_status(self, discussion_id: int, status: str) -> bool:
    """Update discussion status. Valid transitions: active→paused, active→harvested, paused→active, harvested→archived."""

def update_discussion_summary(self, discussion_id: int, summary: str) -> bool:
    """Update the running synthesis summary of a discussion."""

def link_harvested_document(self, discussion_id: int, file_path: str) -> bool:
    """Link a discussion to its harvested knowledge document."""
```

### Insight CRUD

```python
def add_insight(self, discussion_id: int, type: str, content: str,
                confidence: str = "medium", source_description: str = "",
                referenced_file_path: str | None = None) -> dict:
    """Add a classified insight to a discussion. Returns {id, type, content, ...}."""

def update_insight(self, insight_id: int, content: str | None = None,
                   type: str | None = None, confidence: str | None = None) -> bool:
    """Reclassify or update an insight (e.g., assumption → fact)."""

def resolve_insight(self, insight_id: int, resolved_as: str) -> bool:
    """Mark an insight as resolved (e.g., question answered, assumption verified/rejected)."""

def get_insights_by_type(self, discussion_id: int, type: str) -> list[dict]:
    """Get all insights of a specific type from a discussion."""
```

### Cross-Referencing

```python
def get_discussions_for_document(self, file_path: str) -> list[dict]:
    """Find all discussions that reference or were sourced from a document."""

def get_open_questions_across_discussions(self) -> list[dict]:
    """Get all unresolved open_question insights across active discussions."""

def get_unverified_assumptions(self) -> list[dict]:
    """Get all unresolved assumption insights across active discussions."""
```

---

## 4. MCP Tools — `server.py`

### `start_discussion(topic, source_file_path?, seed_content?)`

```python
@mcp.tool()
async def start_discussion(
    topic: str,
    source_file_path: str | None = None,
    seed_content: str | None = None,
) -> str:
    """Start a structured knowledge discussion about a topic.

    Starts a reasoning session where insights are classified as facts, opinions,
    assumptions, open questions, key concepts, or risks. Use this when exploring
    a technical topic, reviewing an existing document, or refining understanding.

    If source_file_path is provided, the discussion is linked to that existing
    document (useful for refreshing stale knowledge).

    If seed_content is provided (e.g., from a chat transcript, email, or brain dump),
    the system will do an initial classification of the content into insight types.
    """
```

**Workflow:**
1. Create discussion record in SQLite
2. If `source_file_path` provided: read the document, extract initial insights from its content
3. If `seed_content` provided: parse it into initial insight classifications (LLM-assisted or heuristic)
4. Search knowledge base for existing documents about the topic — note these as context
5. Return discussion ID, initial state, and any related existing knowledge found

### `add_insight(discussion_id, insight, type, confidence?, reference?)`

```python
@mcp.tool()
async def add_insight(
    discussion_id: int,
    insight: str,
    type: str,
    confidence: str = "medium",
    reference: str | None = None,
) -> str:
    """Add a classified insight to an active discussion.

    Type must be one of: fact, opinion, assumption, open_question, key_concept, risk.
    Confidence: high, medium, low.
    Reference: optional file_path to an existing knowledge document that supports this insight.

    The LLM should use this during conversation to capture important findings.
    Example flow:
    - User says "PostgreSQL handles this via MVCC" → add_insight(type="fact", confidence="high")
    - User says "I think Redis would be faster" → add_insight(type="opinion", confidence="medium")
    - User says "Assuming we stay under 100 connections" → add_insight(type="assumption")
    """
```

### `review_discussion(discussion_id)`

```python
@mcp.tool()
async def review_discussion(
    discussion_id: int,
) -> str:
    """Review the current state of a discussion.

    Returns all insights grouped by type, a running summary, open questions count,
    unverified assumptions count, and links to related knowledge base documents.
    Use this to check progress, identify gaps, and decide when to harvest.
    """
```

**Output format:**
```
## Discussion: Distributed Caching Strategies
Status: active | Started: 2025-07-20 | Insights: 12

### Facts (4)
1. [HIGH] Redis supports cluster mode with automatic sharding
2. [HIGH] Memcached is multi-threaded; Redis is single-threaded (per shard)
...

### Open Questions (3)
1. How does cache invalidation work across regions?
2. What's the memory overhead of Redis Cluster vs standalone?
...

### Assumptions (2) ⚠️
1. [MEDIUM] Our dataset fits in 64GB per node — needs verification
2. [LOW] Network latency between DCs is <10ms — needs measurement

### Key Concepts (2)
...

### Summary
[Running synthesis of the discussion state]

### Related Knowledge Base Documents
- Distributed Systems Patterns (92% similar)
- Redis Configuration Guide (87% similar)
```

### `reclassify_insight(insight_id, new_type, new_confidence?)`

```python
@mcp.tool()
async def reclassify_insight(
    insight_id: int,
    new_type: str,
    new_confidence: str | None = None,
) -> str:
    """Reclassify an insight when understanding changes.

    Common reclassifications:
    - assumption → fact (verified through research)
    - assumption → risk (found to be problematic)
    - open_question → fact (question answered)
    - opinion → key_concept (consensus reached)
    """
```

### `resolve_insight(insight_id, resolution)`

```python
@mcp.tool()
async def resolve_insight(
    insight_id: int,
    resolution: str,
) -> str:
    """Mark an insight as resolved with an explanation.

    Use for:
    - Answering an open question
    - Verifying or rejecting an assumption
    - Accepting or dismissing a risk

    The resolution text is preserved alongside the original insight for context.
    """
```

### `harvest_discussion(discussion_id, title?, categories?)`

```python
@mcp.tool()
async def harvest_discussion(
    discussion_id: int,
    title: str | None = None,
    categories: list[str] | None = None,
) -> str:
    """Harvest a discussion into a knowledge document draft.

    Synthesizes all insights from the discussion into a structured markdown document:
    - Facts and verified assumptions become the document body
    - Key concepts get their own sections
    - Unresolved questions are noted in a 'Open Questions' section
    - Active risks are noted in a 'Known Risks' section
    - The discussion's opinions inform tone but are not presented as facts

    The document is created via save_knowledge (draft status) and linked back
    to the discussion. Auto-detects categories via semantic search if not provided.

    Uses Smart Capture (feature #8) to find related documents and store relationships.
    The discussion is marked as 'harvested' after successful creation.
    """
```

**Synthesis workflow:**
1. Read all insights from the discussion, grouped by type
2. Build document structure:
   - Title and overview from discussion topic + summary
   - Body from facts + verified key concepts
   - "Open Questions" section from unresolved `open_question` insights
   - "Known Risks" section from unresolved `risk` insights
   - Frontmatter includes `discussion_id` for traceability
3. Call `save_knowledge` pipeline (auto-category, auto-relate)
4. Link the created document back to the discussion
5. Mark discussion as `harvested`
6. Return the created document details + relationship summary

### `list_discussions(status?)`

```python
@mcp.tool()
async def list_discussions(
    status: str | None = None,
) -> str:
    """List all discussions, optionally filtered by status.

    Status values: active, paused, harvested, archived.
    Returns topic, status, insight counts, and age for each discussion.
    """
```

### `suggest_discussions()`

```python
@mcp.tool()
async def suggest_discussions() -> str:
    """Suggest topics that would benefit from a discussion.

    Analyzes the knowledge base to find:
    - Stale documents (confidence < 70%) that need refresh
    - Documents with unresolved open questions from past discussions
    - Topics with contradicting documents
    - Categories with few documents (knowledge gaps)
    - Orphan documents that need relationship context

    Returns a prioritized list of suggested discussion topics with reasons.
    """
```

---

## 5. MCP Prompt — `server.py`

### New Prompt: `discuss`

```python
@mcp.prompt()
def discuss(topic: str) -> list[base.Message]:
    """Start a structured discussion about a topic to build knowledge."""
    return [
        base.UserMessage(
            f"I want to explore the topic: {topic}\n\n"
            "Use 'start_discussion' to initialize a discussion session.\n"
            "As we talk, use 'add_insight' to classify important findings as:\n"
            "- fact (verified truths)\n"
            "- opinion (subjective views)\n"
            "- assumption (unverified beliefs)\n"
            "- open_question (things to investigate)\n"
            "- key_concept (important ideas)\n"
            "- risk (uncertainties)\n\n"
            "Periodically use 'review_discussion' to check our progress.\n"
            "Challenge my assumptions and ask clarifying questions.\n"
            "When we've reached good understanding, use 'harvest_discussion' "
            "to create a knowledge document from our findings."
        ),
    ]
```

---

## 6. Server Instructions Update

Add to the `instructions` string in `FastMCP()`:

```
Discussion workflow for building knowledge through conversation:
1. Use `start_discussion(topic)` to begin a structured exploration.
2. During conversation, use `add_insight` to capture facts, opinions, assumptions,
   open questions, key concepts, and risks as they emerge.
3. Use `review_discussion` periodically to assess coverage and identify gaps.
4. Use `reclassify_insight` when understanding changes (e.g., assumption verified → fact).
5. When the discussion reaches good coverage, use `harvest_discussion` to synthesize
   findings into a knowledge document draft.
6. Use `suggest_discussions` to discover what topics need exploration based on
   staleness, gaps, and contradictions in the knowledge base.
```

---

## 7. Indexer Integration — `indexer.py`

### Harvested Document Metadata

When `harvest_discussion` creates a document, the frontmatter includes:

```yaml
---
title: Distributed Caching Strategies
categories: [architecture, databases]
author: AI-Agent
date: 2025-07-20
status: draft
source_discussion: 42
---
```

The `source_discussion` field links the document back to the discussion that produced it. This enables:
- Viewing the full discussion trail from a document
- Understanding *how* knowledge was derived, not just *what* it says
- Re-opening discussions when the harvested document becomes stale

### Post-Harvest Indexing

After `save_knowledge` creates the file:
1. The new document goes through the normal indexing pipeline
2. Smart Capture (#8) finds related documents and stores relationships
3. If the discussion had `referenced_file_path` insights, those become explicit relationships (`cites` type from Feature #6)
4. The discussion insight references are preserved in `discussion_insights.referenced_file_path`

---

## 8. Integration with Existing Features

### With Staleness Metrics (#3)

```python
# In suggest_discussions()
stale_docs = fts.get_all_documents_with_confidence()
for doc in stale_docs:
    if doc["confidence"] < 0.70:
        suggestions.append({
            "topic": f"Refresh: {doc['title']}",
            "reason": f"Confidence {doc['confidence']:.0%} — last updated {doc['days_ago']} days ago",
            "source_file_path": doc["file_path"],
            "priority": "high" if doc["confidence"] < 0.50 else "medium",
        })
```

### With Wikilinks (#1)

When insights reference existing documents, the harvested document includes wikilinks:

```markdown
As discussed, [[Redis Configuration Guide]] covers the basic setup,
but we identified gaps in the multi-region caching strategy.
```

### With Explicit Relationships (#6)

Discussion references become typed relationships on the harvested document:

- Insights with `referenced_file_path` → `cites` relationship
- Discussions sourced from an existing doc → `extends` relationship
- If a discussion identifies a superseding document → `supersedes` relationship

### With Smart Capture (#8)

`harvest_discussion` delegates document creation to the enhanced `save_knowledge` which already handles auto-category, auto-relate, and orphan detection.

---

## 9. Web UI

### New Page: Discussion Browser

**Route**: `GET /discussions`

List all discussions with filtering by status. Show topic, status badge, insight counts, creation date, linked documents (source and harvested).

### New Page: Discussion Detail

**Route**: `GET /discussion/{id}`

Full discussion view with:
- Insights grouped by type, color-coded (green=fact, yellow=opinion, orange=assumption, blue=question, purple=concept, red=risk)
- Resolution status for each insight
- Linked knowledge base documents (source and harvested)
- Timeline showing when insights were added

### Document Page Enhancement

On the document detail page, if the document was harvested from a discussion, show a "Discussion Trail" section with a link to the source discussion and key insights.

### Admin Panel Enhancement

Add "Discussion Health" section:
- Active discussions count and age
- Stale discussions (active for >30 days without new insights)
- Suggestion queue (from `suggest_discussions` logic)

---

## 10. REST API Endpoints — `app.py`

| Route | Method | Description |
|-------|--------|-------------|
| `/api/discussions` | GET | List discussions (optional `?status=` filter) |
| `/api/discussions` | POST | Create new discussion |
| `/api/discussions/{id}` | GET | Discussion detail with all insights |
| `/api/discussions/{id}/insights` | GET | List insights for a discussion |
| `/api/discussions/{id}/insights` | POST | Add insight to a discussion |
| `/api/discussions/{id}/harvest` | POST | Harvest discussion to knowledge document |
| `/api/discussions/{id}/status` | PATCH | Update discussion status |
| `/api/discussions/suggestions` | GET | Get suggested discussion topics |

---

## 11. Migration from `technical-discussion.agent.md`

The existing agent can be updated to use MCP tools instead of direct file manipulation:

### Before (direct file ops)
```
Agent creates .discussions/topic.md on disk
Agent creates topic.md in workspace root
User manually "sends to knowledge engine"
```

### After (MCP-native)
```
Agent calls start_discussion(topic) via MCP
Agent calls add_insight() during conversation via MCP
Agent calls harvest_discussion() to create knowledge draft via MCP
Knowledge Engine handles indexing, relationships, and review automatically
```

The agent becomes a **thin orchestration layer** that focuses purely on being a great thinking partner, while the Knowledge Engine handles all persistence, search, and quality control.

The `.discussions/` folder and two-file model are no longer needed — discussion state lives in SQLite, and the canonical output is the harvested knowledge document in the knowledge directory.

---

## Implementation Order

| Phase | Scope | Description |
|-------|-------|-------------|
| **Phase 1** | DB + CRUD | `discussions` and `discussion_insights` tables, FTS store methods |
| **Phase 2** | MCP Tools | `start_discussion`, `add_insight`, `review_discussion`, `list_discussions` |
| **Phase 3** | Harvest | `harvest_discussion` with document synthesis and Smart Capture integration |
| **Phase 4** | Intelligence | `suggest_discussions`, staleness integration, cross-referencing |
| **Phase 5** | Web UI | Discussion browser, detail page, admin enhancements |
| **Phase 6** | Agent Migration | Update `technical-discussion.agent.md` to use MCP tools |
