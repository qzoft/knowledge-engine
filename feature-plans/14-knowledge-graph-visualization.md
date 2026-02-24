# Feature Plan: Knowledge Graph Visualization

**Priority**: Medium  
**Depends On**: #1 Wikilinks & Backlinks (link data), #6 Explicit Relationship Types (typed edges)  
**Enhanced By**: #5 Usage Analytics (node sizing by popularity), #3 Staleness Metrics (node coloring by freshness)  
**Theory**: Spatial cognition — visual graph exploration reveals clusters, orphans, and structural gaps that text-based search cannot surface

---

## Overview

Add an interactive knowledge graph visualization to the web UI. Documents are nodes, relationships and wikilinks are edges. The graph supports zooming, panning, filtering by category/relationship type, and clicking nodes to navigate to documents. This turns the knowledge base into a visual, explorable map — making it easy to spot orphan documents, dense clusters, knowledge gaps, and superseded chains.

---

## 1. Data API — `web/app.py`

### New Endpoint: `GET /api/graph`

Returns the full graph structure as JSON for the frontend to render.

```python
async def api_graph(request: Request) -> JSONResponse:
    """Return nodes and edges for the knowledge graph."""
    category = request.query_params.get("category", None)
    
    nodes = []
    edges = []
    
    # Get all documents (optionally filtered by category)
    docs = _fts.get_latest_documents(category=category)
    
    for doc in docs:
        nodes.append({
            "id": doc["file_path"],
            "title": doc["title"],
            "category": doc["categories"][0] if doc.get("categories") else doc.get("category", "_uncategorized"),
            "status": doc.get("status", "draft"),
            "revision": doc.get("revision", 1),
        })
    
    # Get wikilinks (feature #1)
    for doc in docs:
        links = _fts.get_outbound_links(doc["file_path"])
        for link in links:
            edges.append({
                "source": doc["file_path"],
                "target": link["target"],
                "type": "wikilink",
            })
    
    # Get explicit relationships (feature #6)
    for doc in docs:
        rels = _fts.get_outgoing_relationships(doc["file_path"])
        for rel in rels:
            edges.append({
                "source": doc["file_path"],
                "target": rel["target"],
                "type": rel["relationship_type"],
                "note": rel.get("note", ""),
            })
    
    return JSONResponse({"nodes": nodes, "edges": edges})
```

### Optional Enrichments (when features are available)

If **#5 Usage Analytics** is implemented, add `access_count` to each node for popularity-based sizing.

If **#3 Staleness Metrics** is implemented, add `confidence` to each node for freshness-based coloring.

---

## 2. Frontend — `web/static/graph.js` + `web/templates/graph.html`

### Library Choice: D3.js force-directed graph

D3.js is already a standard choice for data visualization and requires no build step — just a `<script>` tag. The force-directed layout naturally clusters related documents and pushes orphans to the periphery.

**Alternative considered:** Cytoscape.js (more graph-specific, heavier). D3 is lighter and fits the existing static-asset approach.

### Graph Features

#### Core Interactions
- **Pan & zoom** — mouse wheel to zoom, drag background to pan
- **Node drag** — drag individual nodes to rearrange
- **Click node** — navigate to document page (`/documents/{file_path}`)
- **Hover node** — tooltip showing title, category, status, revision
- **Hover edge** — tooltip showing relationship type and note

#### Visual Encoding
| Element | Visual Property | Meaning |
|---------|----------------|---------|
| Node color | Category-based palette | Which knowledge domain |
| Node border | Status ring | `draft` = dashed, `in_review` = orange, `approved` = solid green |
| Node size | Base size (or access_count if #5 available) | Importance / popularity |
| Node opacity | Confidence score (if #3 available) | Freshness — stale docs fade |
| Edge color | Relationship type palette | `explains` = blue, `contradicts` = red, `supersedes` = gray, etc. |
| Edge style | Solid for explicit relationships, dashed for wikilinks | Formality of connection |
| Edge arrow | Directed edges for typed relationships | Direction of relationship |

#### Filtering Controls
- **Category filter** — dropdown/multi-select to show/hide categories
- **Relationship type filter** — checkboxes per relationship type
- **Status filter** — toggle draft/in_review/approved visibility
- **Search highlight** — text input that highlights matching nodes and dims others
- **Orphan highlight** — button to highlight nodes with zero edges

#### Layout Options
- **Force-directed** (default) — physics simulation, clusters emerge naturally
- **Category-grouped** — nodes pulled toward category centers
- **Radial** — selected node at center, related nodes in concentric rings by distance

### Graph Page Template

```html
<!-- graph.html -->
{% extends "_layout.html" %}
{% block content %}
<div class="graph-container">
  <div class="graph-controls">
    <!-- Filter dropdowns, search box, layout toggle -->
  </div>
  <div id="knowledge-graph" class="graph-canvas">
    <!-- D3 renders here -->
  </div>
  <div class="graph-sidebar" id="graph-detail">
    <!-- Shows selected node details without navigating away -->
  </div>
</div>
{% endblock %}
```

### Sidebar Detail Panel

When a node is selected (single click), the sidebar shows:
- Document title, category, status
- Summary (if #2 Progressive Summarization is available)
- Incoming and outgoing relationships
- Quick actions: "Open document", "View related", "Start discussion" (if #11 available)

---

## 3. Graph Statistics Panel

A collapsible stats panel at the top of the graph page showing:

- **Total nodes / edges** — overall graph size
- **Orphan count** — documents with zero connections (action: click to list them)
- **Largest cluster** — most interconnected group of documents
- **Most connected node** — document with the most relationships (hub document)
- **Superseded chains** — documents that have been superseded (action: click to review)
- **Category distribution** — mini bar chart of documents per category

---

## 4. Route Registration

```python
# In create_app()
Route("/graph", endpoint=graph_page),
Route("/api/graph", endpoint=api_graph),
```

Add "Graph" link to the main navigation bar in `_layout.html`.

---

## 5. Graceful Degradation

The graph visualization depends on features #1 and #6 for edge data. Before those features are implemented:

- **No #1 (wikilinks):** Graph shows nodes only, no wikilink edges. Still useful for seeing category distribution and orphans.
- **No #6 (relationships):** Graph shows wikilink edges only (once #1 is done). No typed edge coloring.
- **No #1 and no #6:** Graph shows only nodes grouped by category. Limited utility but still shows category balance and document count.
- **No #3 (staleness):** All nodes at full opacity. No freshness encoding.
- **No #5 (analytics):** All nodes same size. No popularity encoding.

The graph page should detect which features are available and adjust the legend and controls accordingly.

---

## 6. Performance Considerations

### Client-Side Rendering
All rendering happens in the browser via D3.js. The server only provides the JSON data.

### Large Knowledge Bases
- **< 200 nodes:** Full force simulation, no issues
- **200–1000 nodes:** Use `d3.forceSimulation` with `alphaDecay(0.03)` for faster convergence
- **1000+ nodes:** Switch to canvas renderer instead of SVG, limit initial render to selected category, add "Load all" button

### Caching
- Cache `/api/graph` response with `Cache-Control: max-age=60` (knowledge doesn't change every second)
- Add `?since=timestamp` parameter for incremental updates (future optimization)

---

## 7. Implementation Phases

### Phase 1 — Basic Graph (MVP)
- [ ] Add `/api/graph` endpoint returning nodes and edges JSON
- [ ] Create `graph.html` template with D3.js force-directed layout
- [ ] Node coloring by category, basic click-to-navigate
- [ ] Add "Graph" to navigation bar
- [ ] Register route

### Phase 2 — Interactions
- [ ] Hover tooltips for nodes and edges
- [ ] Sidebar detail panel on node selection
- [ ] Category filter dropdown
- [ ] Relationship type filter checkboxes
- [ ] Search highlight

### Phase 3 — Visual Richness
- [ ] Edge styling (solid/dashed, arrows, type-based colors)
- [ ] Node border encoding for document status
- [ ] Orphan highlight button
- [ ] Layout toggle (force / category-grouped / radial)
- [ ] Graph statistics panel

### Phase 4 — Optional Enrichments
- [ ] Node sizing by access_count (requires #5)
- [ ] Node opacity by confidence score (requires #3)
- [ ] Summary in sidebar (requires #2)
- [ ] "Start discussion" action in sidebar (requires #11)

---

## 8. Design Consistency

The graph page follows the existing web UI aesthetic:
- Dark terminal theme (matches existing CSS)
- JetBrains Mono for text in tooltips and sidebar
- Category color palette consistent with existing category badges
- Controls use the same form styling as the admin panel
