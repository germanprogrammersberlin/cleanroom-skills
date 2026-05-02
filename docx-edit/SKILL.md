---
name: docx-edit
description: Edit the team's Word manuscript (.docx) in place with tracked changes. Use when modifying main_manuscript.docx or other shared Word documents. Pulls latest from cleanroom-drive, edits via docx-mcp with track-changes ON, pushes back. Treats DOCX as the canonical source of truth — never regenerates the manuscript from scratch. Citations go in as Word native fields referencing library.bib cite-keys.
---

# Editing the Team's Word Manuscript

The team's working manuscript is a Word document, not markdown. Both the user (in Microsoft Word) and agents edit the same `.docx` file. Edits are recorded as tracked changes so the user can review what each agent did before accepting.

## Source of truth

`cleanroom-drive:CBZpaper/manuscript/main_manuscript.docx` — this is the canonical file. Always pull the latest before editing; always push your edits back. Never regenerate the manuscript from scratch (that throws away the user's edits and any citations they've inserted in Word).

Markdown files (`*.md`) are agent scratch — drafts, notes, comparisons. They are not the manuscript. Do not write the manuscript as markdown and rebuild — edit the DOCX directly.

## Required prerequisites

This skill depends on:

- **docx-mcp** MCP server (must be running and registered with Claude Code on this adapter). Provides `open_document`, `search_text`, `delete_text`, `insert_text`, `add_comment`, `audit_document`, `save_document`, etc. See the auto-installed `docx-mcp` skill for the full tool list.
- **rclone** with the `cleanroom-drive:` remote configured (the `google-drive` skill).
- **library.bib** at `cleanroom-drive:CBZpaper/library.bib` (BibTeX export of the team's reference library) — needed when inserting citations.

## Workflow

### 1. Pull the latest manuscript

```bash
mkdir -p /tmp/manuscript
rclone copy cleanroom-drive:CBZpaper/manuscript/main_manuscript.docx /tmp/manuscript/
```

Always pull fresh. The user may have made edits in Word since your last run.

### 2. Open and orient

Use docx-mcp tools to understand the document:

```
open_document("/tmp/manuscript/main_manuscript.docx")
get_document_info()      # paragraph count, headings, document structure
get_headings()           # section outline
search_text("phrase")    # find target paragraphs
get_paragraph(para_id)   # read exact text before editing
```

Read before you write. Don't blindly insert at a paragraph index — verify the surrounding context.

### 3. Edit with tracked changes

Tracked changes are **on by default** when using docx-mcp's edit tools. Every deletion shows as red strikethrough, every insertion as green underline, in the user's Word view.

```
delete_text(para_id, "old wording")
insert_text(para_id, "new wording")
add_comment(para_id, "Reason for the change — be brief.")
```

Each edit you make accumulates as a separate revision. The user accepts/rejects each one in Word.

**Author attribution:** docx-mcp records you as the author on every revision. Use your agent name (e.g., `ManuscriptWriter`) so the user can filter by author in Word's review pane.

### 4. Citations — Word native fields, cite-keys from library.bib

Citations in the manuscript use Word's native citation system, not EndNote. Sources live as XML inside the .docx package (`customXml/item5.xml`) and are referenced by cite-key from `library.bib`.

#### How the in-text citation looks in the DOCX XML

A single citation:

```xml
<w:fldSimple w:instr=" CITATION Kaiser2021 \l 1031 ">
  <w:r><w:rPr><w:lang w:val="en-US"/></w:rPr>
    <w:t>(Kaiser, 2021)</w:t>
  </w:r>
</w:fldSimple>
```

A multi-source citation (separated by `\m`, NOT comma or space):

```xml
<w:fldSimple w:instr=" CITATION Chung2022 \l 1031  \m Zhou2020 \m Bahlmann2009 ">
  <w:r><w:rPr><w:lang w:val="en-US"/></w:rPr>
    <w:t>(Chung et al., 2022; Zhou et al., 2020; Bahlmann et al., 2009)</w:t>
  </w:r>
</w:fldSimple>
```

The displayed text inside `<w:t>` is what the user sees BEFORE Word re-renders the field. Word replaces it with the styled citation when the document is opened or the field is updated.

#### Workflow to insert a NEW citation

1. **Find the cite-key.** Pull `library.bib`, locate the entry by author + year + title:
   ```bash
   rclone cat cleanroom-drive:CBZpaper/library.bib | grep -A 8 "Kaiser2021"
   ```
   Confirm the exact `@article{KEY,` line — that's your cite-key.

2. **Verify the source is registered in the docx.** Open `customXml/item5.xml` and search for `<b:Tag>KEY</b:Tag>`. If absent, you must add a `<b:Source>` first (see "Adding a new source" below) — otherwise Word will render the citation as `(Source not found)` and skip it from the bibliography.

3. **Find the anchor text** in `word/document.xml` — the exact sentence/phrase where the citation should appear. Anchors are the SAFE way to locate insertion points; do not rely on paragraph indices alone.

4. **Splice the citation field** into the run. The pattern is: close the current `<w:t>` and `<w:r>`, insert the `<w:fldSimple>`, then re-open `<w:r>` (with the SAME `<w:rPr>`) and `<w:t xml:space="preserve">` for the rest of the text.

   Python helper template (run-properties preservation matters; copy the rPr from the surrounding run):
   ```python
   import re
   from html import escape

   def insert_citation_after(xml: str, anchor_text: str, key: str, displayed: str) -> str:
       pos = xml.find(anchor_text)
       if pos == -1:
           raise RuntimeError(f"anchor not found: {anchor_text!r}")
       insert_pos = pos + len(anchor_text)

       # Find the rPr of the run containing this position so we can preserve typography
       open_re = re.compile(r"<w:r(?:\s[^>]*)?>")
       opens = [(m.start(), m.end()) for m in open_re.finditer(xml, 0, insert_pos)]
       closes = [m.start() for m in re.finditer(r"</w:r>", xml, 0, insert_pos)]
       stack = []
       for p, kind, end in sorted(
           [(s, "o", e) for s, e in opens] + [(c, "c", None) for c in closes]
       ):
           stack.append((p, end)) if kind == "o" else stack and stack.pop()
       if not stack:
           raise RuntimeError("no enclosing <w:r> found")
       r_start, _ = stack[-1]
       r_close = xml.find("</w:r>", insert_pos)
       run_xml = xml[r_start:r_close]
       rpr_match = re.search(r"<w:rPr>.*?</w:rPr>", run_xml, re.S)
       rpr = rpr_match.group(0) if rpr_match else ""

       fld = (
           f'<w:fldSimple w:instr=" CITATION {key} \\l 1031 ">'
           f'<w:r>{rpr}<w:t>{escape(displayed)}</w:t></w:r>'
           f'</w:fldSimple>'
       )
       injection = "</w:t></w:r>" + fld + f'<w:r>{rpr}<w:t xml:space="preserve">'
       return xml[:insert_pos] + injection + xml[insert_pos:]
   ```

5. **For multi-cite (additional refs in same brackets):** find the existing `<w:fldSimple w:instr="...">`, append ` \m NEW_KEY` inside the instr attribute. Do not insert a new `<w:fldSimple>`.

6. **Mark as a tracked insertion** so the user can review (use docx-mcp's tracked-change tools to wrap your changes if you're working through that MCP server).

#### Adding a new source

If a cite-key isn't in `customXml/item5.xml` yet, you have two options:

(a) **The metadata is already in `library.bib`** — convert that BibTeX entry to OOXML Bibliography format and append before `</b:Sources>`:

```xml
<b:Source xmlns:b="http://schemas.openxmlformats.org/officeDocument/2006/bibliography">
  <b:Tag>Kaiser2024_chemokine</b:Tag>
  <b:SourceType>JournalArticle</b:SourceType>
  <b:Year>2024</b:Year>
  <b:Author><b:Author><b:NameList>
    <b:Person><b:Last>Kaiser</b:Last><b:First>David</b:First></b:Person>
    <!-- additional authors here, do NOT include "et al." as a Person -->
  </b:NameList></b:Author></b:Author>
  <b:Title>Ultrasensitive Detection of Chemokines …</b:Title>
  <b:JournalName>Adv. Mater.</b:JournalName>
  <b:Volume>—</b:Volume>
  <b:Issue>—</b:Issue>
  <b:Pages>—</b:Pages>
</b:Source>
```

Important field-level notes:
- `<b:Tag>` value MUST match the cite-key used in CITATION fields.
- `<b:SourceType>` is `JournalArticle` for papers, `Book` for books with editors.
- For books with editors, wrap in `<b:Author><b:Editor><b:NameList>…</b:NameList></b:Editor></b:Author>`.
- `<b:Person><b:Last>` is the surname, `<b:First>` is given names/initials. Word displays "Last, First" — getting these reversed makes the bibliography read wrong.
- **Do NOT include `"et al."` as a `<b:Person>`** — Word generates "et al." automatically based on author count.

(b) **The metadata is NOT in `library.bib`** — do not invent it. Add a Word comment at the target location:
```python
add_comment(para_id, "Citation needed: Kaiser 2024 — please add to library.bib")
```
…and let the user resolve it.

#### Don't try to render the bibliography yourself

You insert citation FIELDS in the body. Word generates the formatted bibliography automatically when the user inserts a `BIBLIOGRAPHY` field via Verweise → Literaturverzeichnis. Do not write a "References" section by hand — it will be a duplicate that gets out of sync.

### 5. Audit before saving

Always run `audit_document()` before save. It catches:
- Orphaned footnotes
- Duplicate paraIds
- Unpaired bookmarks
- Missing relationship targets
- Malformed tracked-change markers

Fix any reported issues before saving.

### 6. Save and push

```
save_document("/tmp/manuscript/main_manuscript.docx")
```

Then upload back to drive:

```bash
rclone copy /tmp/manuscript/main_manuscript.docx cleanroom-drive:CBZpaper/manuscript/
```

## What goes in a markdown scratch file vs the DOCX

| Belongs in `.md` (scratch) | Belongs in `.docx` (manuscript) |
|---------------------------|-----------------------------------|
| Your private analysis notes | Final prose for the paper |
| Draft alternatives, A/B versions | The chosen wording |
| Outlines, structure exploration | Section/heading edits |
| Code-driven figure plans | (figures stay separate) |
| Review summaries to other agents | Comments via `add_comment()` |

If you draft something in markdown first, that's fine — just make sure the final landing place for prose changes is the DOCX, not a new MD file.

## What NOT to do

- ❌ Do not regenerate `main_manuscript.docx` from `main_manuscript.md` or any markdown file. The DOCX has the user's edits and citations — overwriting destroys them.
- ❌ Do not call `paper-docx-builder` for the main manuscript. That skill is for new documents (drafts in `output/drafts/`, SI standalone docs, presentations). For the main manuscript, edit in place.
- ❌ Do not insert plain-text "(Author, Year)" citations. Use Word native citation fields with cite-keys so the bibliography stays accurate.
- ❌ Do not invent cite-keys that aren't in `library.bib`. Add a comment instead.
- ❌ Do not turn off tracked changes. The user must see what you changed.
- ❌ Do not edit the bibliography section directly. Word generates it from the cited sources.

## Coordination with the user

The user works in Word with tracked changes ON. They see your edits in green/red, can accept or reject each, and add their own edits (which then come back to you on the next pull as the new baseline).

Keep your edits **scoped and explainable**:
- One conceptual change = one block of related insertions/deletions + one comment explaining why
- Don't make sweeping changes the user has to wade through
- If you're considering a big restructure, write the proposal in a scratch `.md` first and tag the user via Word comment for approval before applying

## When to use this skill

- Editing main_manuscript.docx (always)
- Adding or revising a section in any shared Word document under cleanroom-drive
- Inserting citations into the manuscript
- Responding to user comments embedded in the docx
- Coordinated multi-agent edits where each agent's changes must be reviewable

## When NOT to use

- Creating a brand new document from scratch → use `paper-docx-builder` instead
- Building a SI document, presentation, or standalone draft that is not the main manuscript → `paper-docx-builder`
- Pure markdown writing (drafts, notes, internal docs) → use the `google-drive` skill to read/write `.md` files directly
