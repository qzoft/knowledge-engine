# Feature Plan: Multi-Reviewer Collaboration

**Priority**: Lower  
**Depends On**: None (extends existing status workflow)  
**Foundation For**: Team scaling, expertise discovery  
**Theory**: Knowledge management is collaborative — single-approver workflows create bottlenecks

---

## Overview

The current workflow supports a single `approved_by` and `in_review_by` field. For team knowledge bases, this is insufficient. This feature evolves the review workflow to support multiple reviewers, contribution tracking, and expertise discovery.

---

## 1. Database Schema Changes — `fts_store.py`

### New Table: `document_reviewers`

```sql
CREATE TABLE IF NOT EXISTS document_reviewers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_path TEXT NOT NULL,
    reviewer TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    assigned_at TEXT NOT NULL DEFAULT (datetime('now')),
    reviewed_at TEXT,
    comment TEXT DEFAULT '',
    FOREIGN KEY (file_path) REFERENCES documents(file_path) ON DELETE CASCADE,
    UNIQUE(file_path, reviewer)
);

CREATE INDEX IF NOT EXISTS idx_reviewers_file ON document_reviewers(file_path);
CREATE INDEX IF NOT EXISTS idx_reviewers_reviewer ON document_reviewers(reviewer);
CREATE INDEX IF NOT EXISTS idx_reviewers_status ON document_reviewers(status);
```

### New Table: `contributions`

```sql
CREATE TABLE IF NOT EXISTS contributions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_path TEXT NOT NULL,
    contributor TEXT NOT NULL,
    action TEXT NOT NULL,
    timestamp TEXT NOT NULL DEFAULT (datetime('now')),
    revision INTEGER,
    FOREIGN KEY (file_path) REFERENCES documents(file_path) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_contributions_contributor ON contributions(contributor);
CREATE INDEX IF NOT EXISTS idx_contributions_action ON contributions(action);
```

### Action Types

| Action | Description |
|--------|-------------|
| `created` | Initial document creation |
| `edited` | Content modification |
| `reviewed` | Review completed (approve or request changes) |
| `approved` | Final approval given |
| `status_change` | Status transition |

---

## 2. Enhanced Status Workflow

### Current
```
draft → in_review → approved
```

### New
```
draft → in_review (assigned to N reviewers) → all approve → approved
                                            → any request changes → draft
```

### Review States Per Reviewer

| State | Meaning |
|-------|---------|
| `pending` | Assigned, not yet reviewed |
| `approved` | Reviewer approves |
| `changes_requested` | Reviewer requests changes |

### Document Transitions

- `in_review`: At least one reviewer assigned and pending
- `approved`: All assigned reviewers have status `approved`
- Back to `draft`: Any reviewer sets `changes_requested`, or content is edited

---

## 3. FTS Store Methods — `fts_store.py`

```python
def assign_reviewers(self, file_path: str, reviewers: list[str]) -> bool:
    """Assign reviewers to a document. Sets document status to in_review."""

def submit_review(self, file_path: str, reviewer: str, status: str, comment: str = "") -> dict:
    """Submit a review decision. Returns updated document status."""

def get_review_status(self, file_path: str) -> dict:
    """Get all reviewer statuses for a document."""

def get_reviewer_queue(self, reviewer: str) -> list[dict]:
    """Get all documents pending review by a specific reviewer."""

def log_contribution(self, file_path: str, contributor: str, action: str, revision: int | None = None):
    """Log a contribution event."""

def get_contributions(self, file_path: str) -> list[dict]:
    """Get contribution history for a document."""

def get_contributor_profile(self, contributor: str) -> dict:
    """Get a contributor's activity: documents created, reviewed, approved, expertise areas."""
```

---

## 4. MCP Tools — `server.py`

### `assign_review(file_path, reviewers)`

```python
@mcp.tool()
async def assign_review(
    file_path: str,
    reviewers: list[str],
) -> str:
    """Assign reviewers to a document and set it to in_review status."""
```

### `submit_review(file_path, reviewer, decision, comment?)`

```python
@mcp.tool()
async def submit_review(
    file_path: str,
    reviewer: str,
    decision: str,
    comment: str = "",
) -> str:
    """Submit a review decision (approve or request_changes) for a document."""
```

### `get_review_queue(reviewer)`

```python
@mcp.tool()
async def get_review_queue(
    reviewer: str,
) -> str:
    """Get all documents pending review by a specific person."""
```

---

## 5. Web UI

### Review Queue Page

**Route**: `GET /reviews/{reviewer}`

Shows all documents assigned to a reviewer with pending status. Each document shows title, category, assigned date, and action buttons (approve / request changes).

### Document Page Enhancement

When a document is `in_review`, show the reviewer panel:
- List of assigned reviewers with their status (pending / approved / changes requested)
- Review comment display
- Approve / Request Changes buttons

### Contributor Profile Page

**Route**: `GET /contributor/{name}`

Shows:
- Documents created, edited, reviewed, approved
- Category distribution (expertise heatmap)
- Activity timeline

---

## Implementation Order

| Phase | Scope |
|-------|-------|
| **Phase 1** | DB schema + reviewer assignment + review submission |
| **Phase 2** | MCP tools for review workflow |
| **Phase 3** | Contribution tracking |
| **Phase 4** | Web UI review queue and document review panel |
| **Phase 5** | Contributor profiles and expertise discovery |
