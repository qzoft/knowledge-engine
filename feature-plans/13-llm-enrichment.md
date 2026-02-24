# Feature Plan: LLM Enrichment via MCP Sampling

**Priority**: High  
**Depends On**: None (uses MCP sampling, available in FastMCP already)  
**Foundation For**: Discussion Harvesting (#11) — synthesis step; Progressive Summarization (#2) — auto-summary generation; Smart Capture (#8) — relationship type inference  
**Theory**: The LLM is a service, not a component — the server borrows intelligence from the client without owning it

---

## Overview

The Knowledge Engine's ingestion pipeline is deliberately LLM-free: parsing, hashing, embedding, and indexing are deterministic and fast. But several features benefit from LLM intelligence — synthesizing discussion insights into prose, generating summaries when the author didn't provide them, suggesting relationship types between documents, and improving category detection.

Rather than embedding an LLM into the server (which would require API keys, model config, and external dependencies), this feature uses **MCP sampling** — the protocol's built-in mechanism for the server to ask the *client's* LLM to generate text. The server sends a self-contained prompt, receives the result in an isolated context window, and uses it. No API keys, no model dependency, no cost to the server.

### How MCP Sampling Works

```
Server tool (e.g., harvest_discussion)
  │
  ├── Gathers data from SQLite/LanceDB
  │
  ├── ctx.session.create_message(         ← Asks the client's LLM
  │     messages=[{role: "user",             in a FRESH, ISOLATED context
  │       content: "Synthesize these..."}],  (no conversation history,
  │     max_tokens=4096,                     no user system prompt)
  │     system_prompt="You are..."          
  │   )                                    
  │                                        
  │   Client shows approval dialog → User approves → LLM generates
  │                                        
  ├── Receives generated text              
  │                                        
  └── Uses result (save document, update metadata, etc.)
```

Key properties:
- **Isolated**: Fresh context window — no pollution of the user's conversation
- **Client-agnostic**: Works with whatever model the client has (Claude, GPT-4, Copilot)
- **Zero server config**: No API keys, no model selection, no cost on the server
- **Human-in-the-loop**: Client shows approval dialog before generation
- **Graceful fallback**: If the client doesn't support sampling, catch the error and fall back to non-LLM behavior

---

## 1. Sampling Helper — `server.py`

### Utility Function

```python
async def _llm_generate(
    ctx: Context,
    prompt: str,
    system_prompt: str = "",
    max_tokens: int = 4096,
) -> str | None:
    """Request LLM generation via MCP sampling. Returns None if unsupported."""
    try:
        result = await ctx.session.create_message(
            messages=[{
                "role": "user",
                "content": {"type": "text", "text": prompt},
            }],
            max_tokens=max_tokens,
            **({"system_prompt": system_prompt} if system_prompt else {}),
        )
        return result.content.text
    except Exception:
        return None
```

Every use of sampling follows the pattern:
```python
generated = await _llm_generate(ctx, prompt="...", system_prompt="...")
if generated is None:
    # Fallback: non-LLM behavior
```

---

## 2. Enrichment Use Cases

### 2a. Discussion Harvesting Synthesis (#11)

**Tool**: `harvest_discussion`

When harvesting a discussion, use sampling to synthesize insights into flowing prose instead of mechanical bullet-point assembly.

```python
@mcp.tool()
async def harvest_discussion(
    discussion_id: int,
    title: str | None = None,
    categories: list[str] | None = None,
    ctx: Context,
) -> str:
    discussion = fts.get_discussion(discussion_id)
    insights_text = _format_insights_for_prompt(discussion)
    
    synthesized = await _llm_generate(
        ctx,
        prompt=(
            f"Synthesize the following classified discussion insights into a "
            f"well-structured knowledge document about '{discussion['topic']}'.\n\n"
            f"Rules:\n"
            f"- Facts and verified key concepts form the main body\n"
            f"- Open questions go in an 'Open Questions' section\n"
            f"- Known risks go in a 'Known Risks' section\n"
            f"- Write in clear technical documentation style\n"
            f"- Do not invent information beyond what's in the insights\n\n"
            f"Insights:\n{insights_text}"
        ),
        system_prompt="You are a technical documentation writer. Produce clean markdown.",
        max_tokens=4096,
    )
    
    if synthesized is None:
        # Fallback: mechanical assembly
        synthesized = _assemble_document_from_insights(discussion)
    
    # Save via normal pipeline
    return indexer.add_document(title=title, content=synthesized, ...)
```

**Fallback**: Bullet-point assembly grouped by insight type.

### 2b. Auto-Summary Generation (#2)

**When**: During indexing, if a document has no `summary_short` in frontmatter.

**Approach**: Add an optional `enrich` step to the indexing pipeline. NOT on every index run — only when explicitly requested via an MCP tool or admin action.

```python
@mcp.tool()
async def enrich_document(
    file_path: str,
    ctx: Context,
) -> str:
    """Generate missing summaries and suggest relationships for a document using the client's LLM."""
    doc = fts.get_document(file_path)
    
    # Generate one-liner summary
    if not doc.get("summary_short"):
        summary = await _llm_generate(
            ctx,
            prompt=(
                f"Write a single-sentence summary (max 20 words) for this document:\n\n"
                f"Title: {doc['title']}\n"
                f"Content: {doc['content'][:2000]}"
            ),
            max_tokens=100,
        )
        if summary:
            fts.update_document_metadata(file_path, summary_short=summary.strip())
    
    # Generate key points
    if not doc.get("summary_points"):
        points = await _llm_generate(
            ctx,
            prompt=(
                f"Extract 3-5 key bullet points from this document:\n\n"
                f"Title: {doc['title']}\n"
                f"Content: {doc['content'][:3000]}"
            ),
            max_tokens=500,
        )
        if points:
            fts.update_document_metadata(file_path, summary_points=points.strip())
    
    return f"Enrichment complete for {doc['title']}"
```

**Fallback**: Skip enrichment — the document works fine without summaries.

### 2c. Relationship Type Inference (#6)

**When**: Smart Capture finds semantically similar documents, but doesn't know *how* they're related.

```python
async def _infer_relationship_type(
    ctx: Context,
    source_title: str,
    source_snippet: str,
    target_title: str,
    target_snippet: str,
) -> str | None:
    """Ask the LLM to classify the relationship between two documents."""
    result = await _llm_generate(
        ctx,
        prompt=(
            f"Given these two documents, classify their relationship as one of: "
            f"explains, extends, contradicts, cites, prerequisite, supersedes, related_to\n\n"
            f"Document A: {source_title}\n{source_snippet[:500]}\n\n"
            f"Document B: {target_title}\n{target_snippet[:500]}\n\n"
            f"Relationship (single word):"
        ),
        max_tokens=20,
    )
    if result and result.strip().lower() in VALID_RELATIONSHIP_TYPES:
        return result.strip().lower()
    return None
```

**Fallback**: Default to `related_to` (the generic type).

### 2d. Improved Category Detection

**When**: `save_knowledge` can't confidently guess a category from vector similarity alone.

```python
async def _llm_suggest_categories(
    ctx: Context,
    title: str,
    content: str,
    existing_categories: list[str],
) -> list[str] | None:
    """Ask the LLM to suggest categories from the existing set."""
    cats_str = ", ".join(existing_categories)
    result = await _llm_generate(
        ctx,
        prompt=(
            f"Given these existing categories: {cats_str}\n\n"
            f"Which categories best fit this document?\n"
            f"Title: {title}\n"
            f"Content: {content[:1000]}\n\n"
            f"Return only category names from the list, comma-separated:"
        ),
        max_tokens=100,
    )
    if result:
        suggested = [c.strip().lower() for c in result.split(",")]
        return [c for c in suggested if c in existing_categories] or None
    return None
```

**Fallback**: Current behavior — nearest embedding match.

### 2e. Batch Enrichment

**MCP Tool**: `enrich_category(category)` or `enrich_all()`

Run enrichment across multiple documents that are missing summaries:

```python
@mcp.tool()
async def enrich_category(
    category: str,
    ctx: Context,
) -> str:
    """Enrich all documents in a category with auto-generated summaries and relationship suggestions."""
    docs = fts.get_documents_by_category(category)
    enriched = 0
    for doc in docs:
        if not doc.get("summary_short"):
            # ... generate summary via _llm_generate ...
            enriched += 1
    return f"Enriched {enriched}/{len(docs)} documents in '{category}'"
```

---

## 3. New MCP Tools

| Tool | Description |
|------|-------------|
| `enrich_document(file_path)` | Generate missing summaries and suggest relationships for a single document |
| `enrich_category(category)` | Batch-enrich all documents in a category |
| `batch_enrich(max_docs?)` | Enrich up to N documents that are missing summaries across entire knowledge base |

---

## 4. Design Principles

### Always Optional, Always Graceful

- Every sampling call has a non-LLM fallback
- The engine works fully without sampling support
- Enrichment is an explicit action, never automatic during indexing
- Generated content is clearly marked (e.g., `summary_generated: true` flag)

### No Server-Side LLM Config

- Zero API keys, zero model selection, zero cost on the server
- The client's LLM does all the work
- Different users can use different models — quality varies, but the engine doesn't care

### Isolated Context

- Each `create_message()` call gets a fresh context window
- No conversation history, no user system prompt leaked
- The user's conversation context is not inflated
- The user only sees the final tool result, not the intermediate synthesis prompt

### Content Integrity

- LLM-generated summaries are flagged as auto-generated in metadata
- The original content is never modified by enrichment
- Human-reviewed summaries take precedence over auto-generated ones
- Enrichment can be re-run to update auto-generated content

---

## 5. Where Sampling Enhances Existing Features

| Feature | Without Sampling | With Sampling |
|---------|-----------------|---------------|
| **#2 Progressive Summarization** | Author writes summaries manually | `enrich_document` auto-generates missing summaries |
| **#6 Explicit Relationships** | Author writes relationships in frontmatter | `_infer_relationship_type` suggests types for auto-discovered similar docs |
| **#8 Smart Capture** | Finds similar docs by embedding distance | Also infers *why* docs are related (explains vs extends vs contradicts) |
| **#11 Discussion Harvesting** | Mechanical bullet-point assembly | Flowing prose synthesis from classified insights |
| **Category detection** | Nearest embedding match | LLM-assisted category suggestion from existing vocabulary |

---

## 6. Web UI

### Admin Panel Enhancement

Add "Enrichment" section:
- "Enrich Category" dropdown + button
- "Enrich All Unenriched" button with progress indicator
- Table showing documents with/without auto-generated summaries
- Re-enrich button per document

### Document Page Enhancement

When a document has auto-generated summaries, show a subtle badge: `🤖 auto-generated` next to the summary, with an "edit" button to replace with a human-written version.

---

## 7. REST API

| Route | Method | Description |
|-------|--------|-------------|
| `/api/documents/{path}/enrich` | POST | Trigger enrichment for a single document |
| `/api/admin/enrich/category/{name}` | POST | Batch-enrich a category |
| `/api/admin/enrich/status` | GET | Enrichment status (how many docs need enrichment) |

Note: Web API enrichment routes require auth (#12) with `admin` role. These routes internally call the MCP sampling helper, which requires an active MCP client session. If called from the web UI without an MCP session, they fall back to non-LLM behavior and return a message indicating enrichment requires an MCP client.

---

## Implementation Order

| Phase | Scope |
|-------|-------|
| **Phase 1** | `_llm_generate` helper, `enrich_document` tool with summary generation |
| **Phase 2** | Integration with `harvest_discussion` (synthesis with fallback) |
| **Phase 3** | Relationship type inference for Smart Capture |
| **Phase 4** | Batch enrichment tools (`enrich_category`, `batch_enrich`) |
| **Phase 5** | Web UI enrichment controls |
| **Phase 6** | Improved category detection with LLM fallback |
