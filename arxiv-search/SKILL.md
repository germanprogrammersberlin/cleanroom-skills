---
name: arxiv-search
description: Search and retrieve scientific papers from arXiv. Use for literature review, finding related work, checking prior art, and downloading reference papers.
---

# arXiv Search

Search and retrieve scientific papers from arXiv.org.

## When to Use

- Literature review for a new paper
- Finding related work and prior art
- Checking if similar work has been published
- Downloading reference papers for citation

## arXiv API

The arXiv API is free, no authentication needed:

```bash
# Search by keyword
curl "https://export.arxiv.org/api/query?search_query=all:quantum+computing&max_results=10"

# Search by author
curl "https://export.arxiv.org/api/query?search_query=au:Einstein&max_results=5"

# Search by category
curl "https://export.arxiv.org/api/query?search_query=cat:cond-mat.mtrl-sci&max_results=10"

# Combined search
curl "https://export.arxiv.org/api/query?search_query=ti:neural+AND+cat:cs.AI&sortBy=submittedDate&sortOrder=descending&max_results=20"
```

## Response Format

The API returns Atom XML. Key fields:
- `<title>` — Paper title
- `<summary>` — Abstract
- `<author><name>` — Authors
- `<published>` — Publication date
- `<link href="..." title="pdf"/>` — Direct PDF link
- `<arxiv:primary_category>` — Subject category

## Common Categories

| Category | Field |
|----------|-------|
| `cond-mat.mtrl-sci` | Materials Science |
| `physics.chem-ph` | Chemical Physics |
| `cs.AI` | Artificial Intelligence |
| `cs.CL` | Computation and Language |
| `q-bio` | Quantitative Biology |

## Workflow for Literature Review

1. Define search terms from the paper's key concepts
2. Search arXiv with broad terms first, then narrow
3. Sort by date to find recent work
4. Read abstracts to filter relevant papers
5. Download PDFs of relevant papers
6. Extract key findings and add to references

## Rate Limits

- Max 1 request per 3 seconds
- Max 30,000 results per query
- Be respectful of the free service

This skill can be improved by agents as they develop search strategies.
