---
name: docx-edit
description: Propose edits to the team's Word manuscript by working on a clone in output/drafts/, NOT in place on manuscript/main_manuscript.docx. Pulls the latest manuscript, copies it into output/drafts/ with a versioned filename, edits the clone with tracked changes via docx-mcp, pushes the edited clone back to output/drafts/. The user reviews and decides whether to integrate the proposed changes into the canonical file themselves.
---

# Proposing Edits to the Team's Word Manuscript

## ⚡ The one rule

**COPY the canonical `.docx` into `output/drafts/`, EDIT the clone there, PUSH only the clone back. NEVER overwrite `manuscript/main_manuscript.docx` directly.**

The user owns the canonical files. The user has the `.docx` open in Word, accumulates edits, accepts/rejects tracked changes, inserts citations, etc. If you write to `manuscript/main_manuscript.docx`:
- you may collide with their open Word session (file locking, lost edits)
- you take on responsibility for not destroying their accumulated work
- merging becomes your problem, not theirs

Instead, you work in `output/drafts/`. Each agent run produces a versioned clone there. The user reviews the clone in Word at their own pace, then manually integrates whatever they like back into the canonical file (Word's *Compare* / *Combine* feature handles that).

## Source of truth, by zone

| Drive path | Who writes | Who reads |
|---|---|---|
| `manuscript/main_manuscript.docx` (and other canonical files) | **user only** | user + agents (read-only) |
| `output/drafts/<versioned-clone>.docx` | **agent (this skill)** | user + agents |
| `library.bib` | **user only** (agents propose via comment in their draft) | user + agents |

Rule of thumb: `output/` is the agent workspace. Anything outside is read-only.

## What to do, what NOT to do

- ✅ Pull `manuscript/main_manuscript.docx` → save with a versioned name into `output/drafts/` → edit the clone → push the clone back to `output/drafts/`.
- ❌ Push your edited clone back to `manuscript/main_manuscript.docx` (overwrites canonical).
- ❌ Build a new docx from `main_manuscript.md` and put it anywhere (loses the user's prior work and citations entirely — markdown is not a source of truth).
- ❌ Save under an ambiguous name like `main_manuscript_v2.docx` at the root of `manuscript/` — forks the canonical timeline; users won't know which to trust.

Markdown files (`*.md`) are agent scratch — drafts, notes, comparisons. They are not the manuscript. Do not treat them as the source of truth.

## Required prerequisites

This skill depends on:

- **docx-mcp** MCP server (must be running and registered with Claude Code on this adapter). Provides `open_document`, `search_text`, `delete_text`, `insert_text`, `add_comment`, `audit_document`, `save_document`, etc. See the auto-installed `docx-mcp` skill for the full tool list.
- **rclone** with the `cleanroom-drive:` remote configured (the `google-drive` skill).
- **library.bib** at `cleanroom-drive:CBZpaper/library.bib` (BibTeX export of the team's reference library) — needed when inserting citations.

## Workflow

### 1. Build the clone path

Compose a versioned filename so the user can tell at a glance who edited what, when:

```
output/drafts/main_manuscript__<AgentName>__<YYYY-MM-DD>__<short-topic>.docx
```

Example: `output/drafts/main_manuscript__ManuscriptWriter__2026-05-02__abstract-polish.docx`.

Use double-underscore (`__`) between fields so the filename remains parseable.

### 2. Pull canonical → stage clone

```bash
mkdir -p /tmp/cbz/clone
CLONE_NAME="main_manuscript__ManuscriptWriter__$(date -I)__abstract-polish.docx"

# Pull the canonical (read-only) copy to local
rclone copy cleanroom-drive:CBZpaper/manuscript/main_manuscript.docx /tmp/cbz/clone/

# Rename the local copy to the clone name (you'll edit this one and push it back)
mv "/tmp/cbz/clone/main_manuscript.docx" "/tmp/cbz/clone/$CLONE_NAME"
```

Always pull fresh. The user may have edited the canonical in Word since your last run.

### 3. Open the clone and orient

Use docx-mcp tools on the clone (NOT on any cached canonical):

```
open_document("/tmp/cbz/clone/<CLONE_NAME>")
get_document_info()      # paragraph count, headings, document structure
get_headings()           # section outline
search_text("phrase")    # find target paragraphs
get_paragraph(para_id)   # read exact text before editing
```

Read before you write. Don't blindly insert at a paragraph index — verify the surrounding context.

### 4. Edit with tracked changes

Tracked changes are **on by default** when using docx-mcp's edit tools. Every deletion shows as red strikethrough, every insertion as green underline, in the user's Word view.

```
delete_text(para_id, "old wording")
insert_text(para_id, "new wording")
add_comment(para_id, "Reason for the change — be brief.")
```

Each edit you make accumulates as a separate revision in the clone. The user reviews them in Word against your clone, then *Compare*s the clone to the canonical to decide what to integrate.

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

### 6. Save and push the clone (NOT the canonical)

Save the clone under the same versioned name you created in step 1:

```
save_document("/tmp/cbz/clone/<CLONE_NAME>")
```

Then upload to **`output/drafts/`** on Drive — never to `manuscript/`:

```bash
rclone copy "/tmp/cbz/clone/$CLONE_NAME" cleanroom-drive:CBZpaper/output/drafts/
```

### 7. Tell the user what you produced

In your reply to the issue, include:
- Path to the clone on Drive: `output/drafts/<CLONE_NAME>`
- One-line summary of the changes you made
- How many tracked-change revisions are in the clone
- Any uncertainty / questions for the user (e.g. "decide whether to merge")

The user opens the clone in Word, reviews the tracked changes, then decides whether to integrate them into the canonical `main_manuscript.docx` themselves (typically via Word's *Compare* or by manually accepting/rejecting in a copy of the canonical).

## What goes in a markdown scratch file vs the DOCX clone

| Belongs in `.md` scratch (under `/tmp/`) | Belongs in `output/drafts/<CLONE_NAME>.docx` |
|---------------------------|-----------------------------------|
| Private deliberation, A/B explorations | The proposed final prose |
| Outlines, structure exploration | Section/heading edits as tracked changes |
| Code-driven figure plans | (figures stay separate) |
| Review summaries to other agents | Comments via `add_comment()` |

Markdown scratch is local-only; only the docx clone lands on Drive.

## What NOT to do

- ❌ Do not push your clone back to `cleanroom-drive:CBZpaper/manuscript/` — that overwrites the user's canonical.
- ❌ Do not pull from `manuscript/` and immediately edit it locally without first renaming to a clone — easy to forget which is which.
- ❌ Do not regenerate the docx from `main_manuscript.md` or any markdown file — markdown is not a source of truth.
- ❌ Do not call `paper-docx-builder` for cloning the main manuscript (that builds NEW docs; here we want a faithful clone of the existing file).
- ❌ Do not insert plain-text "(Author, Year)" citations. Use Word native citation fields with cite-keys.
- ❌ Do not invent cite-keys that aren't in `library.bib`. Add a comment instead.
- ❌ Do not turn off tracked changes. The user must see what you changed.
- ❌ Do not edit the bibliography section directly. Word generates it from the cited sources.

## Coordination with the user

The user opens **the canonical** `manuscript/main_manuscript.docx` in Word with tracked changes ON. You write to **a separate clone** in `output/drafts/`. So:

- The user's open Word session never blocks you (you're touching a different file).
- The user's manual edits in the canonical never collide with yours.
- The user reviews your clone separately, then decides what to merge — they own the merge step.

Keep your edits **scoped and explainable**:
- One conceptual change per clone = one focused topic in the clone's filename suffix
- One block of related insertions/deletions + one `add_comment()` explaining why
- Don't make sweeping changes the user has to wade through
- If you're considering a big restructure, write the proposal in a scratch `.md` and tag the user via Word comment in the clone for approval before applying

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
