# Feature Implementation Plan

The implementation order is determined by the dependency graph between features. Features are grouped into tiers — each tier can only be started once all its dependencies from previous tiers are complete. Within a tier, features are independent and can be implemented in any order.

---

## Dependency Graph

```
Tier 1 (no dependencies — foundational)
  ├── #1  Wikilinks & Backlinks
  ├── #2  Progressive Summarization
  ├── #3  Staleness Metrics
  ├── #4  Review Tracking
  └── #12 Authentication & Authorization

Tier 2 (depends on Tier 1)
  ├── #5  Usage Analytics          ← depends on #4 (reuses usage_events table)
  ├── #6  Explicit Relationships   ← depends on #1 (extends wikilink concept)
  ├── #7  Contextual Retrieval     ← standalone, but complements #5
  └── #13 LLM Enrichment           ← standalone, enhances #2, #6, #8, #11

Tier 3 (depends on Tier 2)
  ├── #8  Smart Capture            ← depends on #6 (reuses document_relationships table)
  │                                   enhanced by #13 (relationship type inference)
  ├── #9  Import Pipeline          ← standalone (uses existing indexing pipeline)
  └── #14 Knowledge Graph Viz     ← depends on #1 + #6 (nodes from docs, edges from links/relationships)
                                      enhanced by #3 (node opacity), #5 (node sizing)

Tier 4 (depends on Tier 3)
  └── #11 Discussion Harvesting    ← depends on #8 (reuses auto-relate and category guessing)
                                      integrates with #1, #3, #6, #9
                                      uses #12 for user identity in insights
                                      uses #13 for synthesis (with fallback)

Tier 5 (depends on Tier 1)
  └── #10 Multi-Reviewer Collab    ← extends existing status workflow
                                      uses #12 for reviewer identity
```

---

## Implementation Checklist

Implement one feature at a time, top to bottom. Check off each feature once it's fully implemented and tested.

### Tier 1 — Foundation

- [ ] **#1 Wikilinks & Backlinks** — [feature-plans/01-wikilinks-and-backlinks.md](feature-plans/01-wikilinks-and-backlinks.md)
  - Parser: `[[wikilink]]` regex extraction, `links` field on `ParsedDocument`
  - DB: `document_links` table with source/target/validity tracking
  - FTS Store: `upsert_links`, `get_backlinks`, `get_outbound_links`, `get_link_validation_report`
  - Indexer: wikilink target normalization, link upsert during indexing
  - MCP: `get_backlinks` tool, `get_link_validation_report` tool, enhanced `get_related`
  - Web: backlinks sidebar, `/api/documents/{path}/backlinks` endpoint

- [ ] **#2 Progressive Summarization** — [feature-plans/02-progressive-summarization.md](feature-plans/02-progressive-summarization.md)
  - Parser: `summary_short`, `summary_points`, `summary` frontmatter fields
  - FTS Store: summary columns on documents, summary-aware search
  - MCP: `get_document_summary(file_path, tier)` tool, `search(..., summary_depth)` parameter
  - Web: tiered summary display on document page

- [ ] **#3 Staleness Metrics** — [feature-plans/03-staleness-metrics.md](feature-plans/03-staleness-metrics.md)
  - Parser: `decay_class` frontmatter field (fast/normal/slow)
  - DB: `decay_class` column on documents and revisions
  - FTS Store: `_compute_confidence()` method, `confidence_min` search filter, `get_all_documents_with_confidence()`
  - MCP: confidence badges in search results, `confidence_min` parameter on `search`
  - Web: confidence badges on document page, staleness dashboard in admin

- [ ] **#4 Review Tracking** — [feature-plans/04-review-tracking.md](feature-plans/04-review-tracking.md)
  - DB: `usage_events` table for tracking views, searches, edits
  - FTS Store: `log_event()`, `get_review_queue(days_threshold)`, freshness scoring
  - MCP: `get_review_queue` tool
  - Web: "Stale Knowledge" section in admin, access timestamps on documents

- [ ] **#12 Authentication & Authorization** — [feature-plans/12-authentication.md](feature-plans/12-authentication.md)
  - Config: `auth` section in `rag-config.yaml` (enabled, mode: local/sso, allow_signup, SSO settings)
  - DB: `users` table (local mode), user CRUD, argon2 password hashing
  - Middleware: session management (signed cookies), auth middleware, route protection
  - Roles: `viewer` → `editor` → `reviewer` → `admin` hierarchy
  - Routes: login, signup (local), SSO callback, logout, user management API
  - Identity integration: username auto-fills `author`, `approved_by`, `in_review_by`
  - Web: nav bar auth state, read-only mode for unauthenticated users, login/signup pages, admin user management
  - First-user admin: first sign-up in local mode gets `admin` role automatically

### Tier 2 — Building on Foundation

- [ ] **#5 Usage Analytics** — [feature-plans/05-usage-analytics.md](feature-plans/05-usage-analytics.md)
  - Requires: #4 Review Tracking (reuses `usage_events` table)
  - FTS Store: query logging, popularity metrics, search-to-click ratios
  - MCP: analytics in admin, popularity signal in search ranking
  - Web: analytics dashboard, top queries, orphan documents report

- [ ] **#6 Explicit Relationship Types** — [feature-plans/06-explicit-relationship-types.md](feature-plans/06-explicit-relationship-types.md)
  - Requires: #1 Wikilinks & Backlinks
  - Parser: `DocumentRelationship` dataclass, `relationships` frontmatter list
  - DB: `document_relationships` table with 7 typed relationships
  - FTS Store: `upsert_relationships`, `get_outgoing/incoming_relationships`, `find_knowledge_path` (BFS)
  - Indexer: relationship validation, upsert during indexing
  - MCP: `get_knowledge_path` tool, enhanced `get_related` with explicit relationships, deprecation notices for superseded docs
  - Web: relationship sidebar, `/api/documents/{path}/relationships` endpoint

- [ ] **#7 Contextual Retrieval** — [feature-plans/07-contextual-retrieval.md](feature-plans/07-contextual-retrieval.md)
  - Standalone (complements #5)
  - Session tracking via cookies, recent documents/searches
  - Search boost for recently engaged categories
  - Web: "Recent" sidebar
  - MCP: `context` parameter on search

- [ ] **#13 LLM Enrichment via MCP Sampling** — [feature-plans/13-llm-enrichment.md](feature-plans/13-llm-enrichment.md)
  - Standalone (enhances #2, #6, #8, #11)
  - `_llm_generate()` helper using `ctx.session.create_message()` with graceful fallback
  - Auto-summary generation (enhances #2 Progressive Summarization)
  - Relationship type inference (enhances #6 Explicit Relationships)
  - Category detection for new documents (enhances #8 Smart Capture)
  - Discussion synthesis for harvesting (enhances #11 Discussion Harvesting)
  - MCP: `enrich_document`, `suggest_relationships`, `batch_enrich` tools
  - Config: `llm_enrichment` section (enabled, max_input_tokens, temperature)

### Tier 3 — Intelligence Layer

- [ ] **#8 Smart Capture** — [feature-plans/08-smart-capture.md](feature-plans/08-smart-capture.md)
  - Requires: #6 Explicit Relationship Types
  - Enhanced by: #13 LLM Enrichment (relationship type inference, category detection)
  - Enhanced `save_knowledge`: auto-discover related docs, `parent_doc` parameter, orphan detection
  - DB: `relationship_discovery` table for tracking per-document connection metadata
  - FTS Store: `get_orphan_documents()`, relationship upsert with similarity scores
  - Vector Store: `search_similar` with `similarity_threshold` parameter
  - MCP: `find_orphan_documents` tool, enhanced `save_knowledge` response

- [ ] **#9 Import Pipeline** — [feature-plans/09-import-pipeline.md](feature-plans/09-import-pipeline.md)
  - Standalone
  - URL → Markdown importer (httpx + markdownify)
  - Bulk markdown import with frontmatter preservation
  - PDF → Markdown importer (pypdf/pdfplumber)
  - Code docstring importer (AST parsing)
  - MCP: `import_from_url`, `bulk_import`, `import_from_pdf`, `import_from_code` tools
  - Web: import forms in admin panel

- [ ] **#14 Knowledge Graph Visualization** — [feature-plans/14-knowledge-graph-visualization.md](feature-plans/14-knowledge-graph-visualization.md)
  - Requires: #1 Wikilinks & Backlinks, #6 Explicit Relationship Types
  - Enhanced by: #3 Staleness (node opacity), #5 Usage Analytics (node sizing), #2 Summaries (sidebar), #11 Discussions (sidebar action)
  - D3.js force-directed graph in web UI
  - Nodes: documents colored by category, bordered by status
  - Edges: wikilinks (dashed) and typed relationships (solid, colored, directed)
  - Three layouts: force-directed, category-grouped, radial
  - Filtering by category, relationship type, status; search highlighting; orphan highlighting
  - Sidebar detail panel with quick actions
  - Statistics panel: orphan count, hub documents, largest cluster, superseded chains
  - Route: `/graph` page + `/api/graph` JSON endpoint

### Tier 4 — Knowledge Creation Loop

- [ ] **#11 Discussion Harvesting** — [feature-plans/11-discussion-harvesting.md](feature-plans/11-discussion-harvesting.md)
  - Requires: #8 Smart Capture
  - Enhanced by: #13 LLM Enrichment (synthesis with fallback to mechanical assembly)
  - Integrates with: #1 Wikilinks, #3 Staleness, #6 Relationships, #9 Import Pipeline, #12 Auth (user identity in insights)
  - DB: `discussions` and `discussion_insights` tables
  - FTS Store: discussion CRUD, insight CRUD, cross-referencing queries
  - MCP: `start_discussion`, `add_insight`, `review_discussion`, `reclassify_insight`, `resolve_insight`, `harvest_discussion`, `list_discussions`, `suggest_discussions` tools
  - MCP Prompt: `discuss(topic)`
  - Web: discussion browser, discussion detail page, admin "Discussion Health" section
  - Agent: updated `technical-discussion.agent.md` uses MCP tools

### Tier 5 — Team Features

- [ ] **#10 Multi-Reviewer Collaboration** — [feature-plans/10-multi-reviewer-collaboration.md](feature-plans/10-multi-reviewer-collaboration.md)
  - Extends existing status workflow; uses #12 Auth for reviewer identity
  - DB: `document_reviewers` table, `contributions` table
  - FTS Store: `assign_reviewers`, `submit_review`, `get_review_status`, `get_contributor_profile`
  - MCP: `assign_review`, `submit_review`, `get_review_queue` tools
  - Web: review queue page, reviewer panel on documents, contributor profiles

---

## Notes

- Each feature has a detailed plan in its `feature-plans/` file with DB schemas, method signatures, and UI specs.
- Test each feature before moving to the next — features build on each other.
- Tier 1 features are independent — implement in whichever order feels most impactful.
- #12 (Auth) is in Tier 1 because it has no dependencies, but it's most impactful once you have features that need user identity (reviews, discussions). Can be deferred if single-user.
- #10 (Multi-Reviewer) is in Tier 5 because it has no dependents and is most useful for team scenarios. It can be implemented earlier if team use is a priority.
