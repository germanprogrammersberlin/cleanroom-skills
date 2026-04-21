---
name: endnote-mcp
description: Search and cite from the team's EndNote reference library. Semantic search, BibTeX export, PDF reading, citation formatting, bibliography generation. Use when working with references, citations, or literature from the local library.
---

# EndNote MCP

Connect to the team's EndNote reference library. Search references, read PDFs, format citations, find related papers, and generate bibliographies — all locally, nothing uploaded to the cloud.

## Prerequisites

```bash
pip install endnote-mcp
endnote-mcp setup  # Wizard: auto-detects XML export, indexes library
```

The library must be exported from EndNote as XML first (File → Export → XML).

## Available MCP Tools (12)

The EndNote MCP server provides these tools once configured:

### Search
- **search_references** — Full-text keyword search across all fields
- **semantic_search** — Find papers by meaning, not just keywords (requires embeddings)
- **find_related** — Find papers related to a specific reference

### Read
- **get_reference** — Get full metadata for a reference by ID
- **read_pdf** — Read specific pages from a reference's PDF attachment

### Cite & Export
- **format_citation** — Format a reference in APA, MLA, Chicago, etc.
- **export_bibtex** — Export references as BibTeX entries for LaTeX
- **generate_bibliography** — Generate formatted bibliography from multiple references

### Browse
- **list_references** — Browse references with pagination
- **get_stats** — Library statistics (total refs, PDFs, by year/type)

## Usage Examples

```
"Search my library for graphene field-effect transistor biosensor"
"Find papers related to reference #27"
"Export references 26, 27, 31 as BibTeX"
"Read pages 5-7 from the Kaiser 2024 paper"
"Generate APA bibliography for references 26, 27, 31, 33, 35"
"Find papers about how ionic screening affects sensor sensitivity" (semantic)
```

## Workflow for Paper Writing

1. **Literature Review** — Use semantic_search to find relevant papers by topic
2. **Citation** — Use export_bibtex to get properly formatted BibTeX entries
3. **Verification** — Use read_pdf to check claims against source text
4. **Bibliography** — Use generate_bibliography for the references section
5. **Related Work** — Use find_related to discover papers you may have missed

## Anti-Hallucination

ALWAYS verify references against the EndNote library before citing:
- Search for the paper by author + year
- Confirm the title, journal, and year match
- Use read_pdf to verify specific claims if uncertain
- Never cite a paper that is not in the library without flagging it to the board

## Configuration

The MCP server reads from a local SQLite index. Key paths:
- XML export: set during `endnote-mcp setup`
- PDF directory: auto-detected from EndNote preferences
- Database: `~/.endnote-mcp/library.db`

Re-index after adding new references: `endnote-mcp index`

This skill can be improved by agents as they develop citation workflows.
