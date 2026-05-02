---
name: bibtex-library
description: Search and use the team's BibTeX reference library (library.bib on cleanroom-drive). Find cite-keys by author/year/topic, verify entries exist before citing, list all available references. Use whenever you need to insert a citation, fact-check a claim against the literature, or add new references to the library. The library is the single source of truth for citations in main_manuscript.docx.
---

# BibTeX Library Workflow

The team's reference library lives as a BibTeX file on cleanroom-drive:

```
cleanroom-drive:CBZpaper/library.bib
```

It is **the single source of truth** for citations. Every reference cited in `main_manuscript.docx` has a corresponding `@article{key, ...}` entry here. Adding a citation without a matching library entry breaks the bibliography.

The `library.bib` is paired with a Word-native source list embedded inside the docx (`customXml/item5.xml`). The two should stay in sync — every cite-key in `library.bib` should also appear as a `<b:Tag>` in the docx source list. See the `docx-edit` skill for how to add a new `<b:Source>` block when introducing a new ref.

## When to use this skill

- Looking up the cite-key for a known paper (author + year) before inserting a citation
- Searching the library by topic when you need to support a claim
- Listing all available references for an agent's planning step
- Verifying a cite-key actually exists before writing it into the manuscript
- Proposing new entries to add (output goes back to the user — do not auto-edit `library.bib` without explicit user approval)

## Pull the library

```bash
mkdir -p /tmp/lib
rclone copy cleanroom-drive:CBZpaper/library.bib /tmp/lib/
```

Always pull fresh. The user may have added or edited entries.

## BibTeX format reminder

```bibtex
@article{kaiser2024,
  author  = {Kaiser, David and others},
  title   = {Solution-gated graphene field-effect transistors with imprinting molecules},
  journal = {ACS Sensors},
  year    = {2024},
  volume  = {9},
  pages   = {123--135},
  doi     = {10.1021/acssensors.4b00123},
}
```

Key fields you care about: `cite-key` (the bare word after `@article{` and before the comma), `author`, `year`, `title`, `journal`, `doi`.

## Common operations

### Look up a cite-key by author and year

Quick grep:

```bash
grep -nE "^@" /tmp/lib/library.bib | grep -i "kaiser"
# or by year — slower, scans full entries:
awk -v RS='@' '/year\s*=\s*\{?2024/' /tmp/lib/library.bib | grep -E "^(article|book|inproceedings|misc)\{[^,]+,"
```

For more reliable lookup, use Python with `bibtexparser` (already available in the runtime via the python3 install layer) — see "Programmatic search" below.

### Verify a cite-key exists

```bash
grep -E "^@[a-z]+\{kaiser2024," /tmp/lib/library.bib && echo "FOUND" || echo "MISSING"
```

If MISSING: do not invent the key. Add a Word comment in the manuscript flagging the gap (`add_comment(para_id, "Citation needed: Kaiser 2024 — not in library.bib")`) and report back to the user.

### List all cite-keys

```bash
grep -oE "^@[a-z]+\{[^,]+," /tmp/lib/library.bib | sed -E 's/^@[a-z]+\{([^,]+),/\1/' | sort -u
```

Useful for planning ("which papers do I have available?") and for sanity checks.

### Programmatic search (Python)

```python
import bibtexparser
with open("/tmp/lib/library.bib") as f:
    db = bibtexparser.load(f)

# All entries
for e in db.entries:
    print(e["ID"], e.get("year"), e.get("author"))

# Find by author keyword (case-insensitive)
hits = [e for e in db.entries if "kaiser" in e.get("author", "").lower()]

# Find by topic in title/keywords
topic = "graphene field-effect transistor"
hits = [e for e in db.entries
        if topic.lower() in (e.get("title", "") + " " + e.get("keywords", "")).lower()]

# Get the cite-key for a specific paper
for e in hits:
    print(e["ID"])
```

If `bibtexparser` is not installed, fall back to grep/awk patterns or install on the fly: `python3 -m pip install --break-system-packages bibtexparser`.

## Once you have the cite-key

Hand it off to the `docx-edit` workflow. The Word native citation field gets the cite-key as `\l` argument:

```
<w:fldSimple w:instr=" CITATION kaiser2024 \l 1031 ">
```

Word renders the formatted citation when the user opens or refreshes the document. The library.bib is read by Word's citation manager (or by the docx-mcp tooling on the agent side that mirrors the BibTeX entries into the document's `customXml/itemX.xml` source list).

## Adding new references

If you encounter a paper that should be in the library but isn't:

1. Do **not** silently edit `library.bib` and push it. The user owns this file.
2. Format a proposed BibTeX entry using metadata you can verify (DOI lookup via `arxiv-search` skill, or read the PDF if available).
3. Surface the proposed entry in your output — the user will add it to the library and re-export.

Example proposal block in your agent output:

```
## Proposed library.bib additions

```bibtex
@article{ullah2024,
  author  = {Ullah, M. and others},
  title   = {Vacancy formation energy in copper iodide thin films},
  journal = {Phys. Rev. B},
  year    = {2024},
  volume  = {110},
  pages   = {044108},
  doi     = {10.1103/PhysRevB.110.044108},
}
```

Verified via DOI 10.1103/PhysRevB.110.044108. Please add to library.bib if you agree.
```

## Don't

- ❌ Do not invent cite-keys that aren't in the file. Word's bibliography will silently skip unknown citations and you'll ship a paper with broken refs.
- ❌ Do not hard-code citation strings like `(Kaiser et al., 2024)` as plain text in the manuscript. Use the citation field with the cite-key — Word renders the formatted text.
- ❌ Do not silently edit `library.bib`. The user is the gatekeeper for the library; you propose, they decide.
- ❌ Do not assume cite-keys follow a specific naming convention. Pull and inspect — they may be `kaiser2024`, `Kaiser_2024`, `Kai2024a`, etc., depending on how the user generates them.

## When NOT to use

- Drafting prose without yet needing a citation → just write, mark unsupported claims with a comment, look up cite-keys in a later pass
- Surveying recent literature not in your library yet → use `arxiv-search` first, then propose additions
- Formatting an already-known cite-key into a Word citation field → that's the `docx-edit` skill's job

## Pairs with

- **`docx-edit`** — once you have the cite-key, insert the citation field into the manuscript
- **`arxiv-search`** — find recent papers not yet in the library, then propose entries
- **`google-drive`** — pull library.bib (and PDFs for fact-checking)
