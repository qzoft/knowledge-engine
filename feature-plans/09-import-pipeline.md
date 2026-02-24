# Feature Plan: Import Pipeline (External Knowledge Sources)

**Priority**: Medium  
**Depends On**: None (standalone, uses existing indexing pipeline)  
**Foundation For**: Knowledge coverage, reduced capture friction  
**Theory**: Knowledge capture should meet content where it already lives

---

## Overview

Currently, all knowledge must be manually written as markdown files. This creates high friction for capturing knowledge that already exists in other formats — blog posts, research papers, PDFs, existing documentation, and code comments. This feature adds importers that convert external content into standard markdown with frontmatter and feed it into the existing indexing pipeline.

---

## 1. Importer Architecture

### Common Interface

All importers share a common pattern:

```python
@dataclass
class ImportResult:
    title: str
    content: str           # Markdown body
    categories: list[str]  # Auto-detected or user-specified
    author: str
    source_url: str | None
    source_type: str       # "url", "pdf", "bulk", "docstring"
    original_date: str | None
```

Each importer:
1. Accepts a source (URL, file path, directory)
2. Extracts text and metadata
3. Converts to clean markdown
4. Returns an `ImportResult`
5. The pipeline then calls `indexer.add_document()` with the result

### Source Attribution

Every imported document includes provenance in the frontmatter:

```yaml
---
title: Extracted Title
categories: [detected-category]
author: Original Author
date: 2025-07-20
status: draft
imported_from: "https://example.com/article"
import_type: url
import_date: 2025-07-20
---
```

---

## 2. URL → Markdown Importer

### Dependencies

- `httpx` for fetching (already async-compatible)
- `markdownify` or `html2text` for HTML → Markdown conversion
- `beautifulsoup4` for metadata extraction

### Process

1. Fetch the URL content
2. Extract metadata from HTML: `<title>`, `<meta name="author">`, `<meta name="description">`, `<time>`
3. Extract main content (skip nav, footer, ads) using readability heuristics or `<article>` tag
4. Convert HTML to clean markdown
5. Auto-detect categories via semantic search against existing documents
6. Return `ImportResult`

### MCP Tool

```python
@mcp.tool()
async def import_from_url(
    url: str,
    categories: list[str] | None = None,
    author: str = "",
) -> str:
    """Import a web article into the knowledge base as a markdown document."""
```

---

## 3. PDF → Markdown Importer

### Dependencies

- `pypdf` or `pdfplumber` for text extraction
- Optional: `pdfplumber` for table extraction

### Process

1. Read the PDF file
2. Extract text page by page, preserving heading structure where possible
3. Extract metadata from PDF properties (title, author, creation date)
4. Convert to markdown with heading hierarchy
5. Auto-detect categories
6. Return `ImportResult`

### MCP Tool

```python
@mcp.tool()
async def import_from_pdf(
    file_path: str,
    categories: list[str] | None = None,
    author: str = "",
) -> str:
    """Import a PDF document into the knowledge base as markdown."""
```

---

## 4. Bulk Import

### Process

1. Scan a directory recursively for `.md` files
2. For each file:
   - Parse existing frontmatter (preserve if present)
   - Add missing defaults (status=draft, date=now, author=Local)
   - Copy to knowledge directory
   - Index via normal pipeline
3. Report: N imported, N skipped (already exist), N failed

### MCP Tool

```python
@mcp.tool()
async def bulk_import(
    directory: str,
    categories: list[str] | None = None,
    overwrite: bool = False,
) -> str:
    """Import all markdown files from a directory into the knowledge base."""
```

---

## 5. Code Docstring Importer

### Dependencies

- `ast` (stdlib) for Python
- Optional: TypeScript parser for TS/JS

### Process

1. Parse source files via AST
2. Extract module, class, and function docstrings
3. Organize into a markdown document per module/class
4. Include function signatures as code blocks
5. Auto-categorize as the language name (e.g., "python")

### MCP Tool

```python
@mcp.tool()
async def import_from_code(
    file_path: str,
    language: str = "python",
    categories: list[str] | None = None,
) -> str:
    """Extract documentation from source code into the knowledge base."""
```

---

## 6. Web UI

### Admin Panel Enhancement

Add an "Import" section to the admin panel with:
- URL import form
- PDF upload
- Bulk import directory selector
- Import history table

### REST API

| Route | Method | Description |
|-------|--------|-------------|
| `/api/admin/import/url` | POST | Import from URL |
| `/api/admin/import/pdf` | POST | Import from uploaded PDF |
| `/api/admin/import/bulk` | POST | Bulk import from directory |
| `/api/admin/import/history` | GET | List past imports |

---

## Implementation Order

| Phase | Scope |
|-------|-------|
| **Phase 1** | URL importer + MCP tool |
| **Phase 2** | Bulk markdown import + MCP tool |
| **Phase 3** | PDF importer + MCP tool |
| **Phase 4** | Code docstring importer |
| **Phase 5** | Web UI import forms |
