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

Citations in the manuscript use Word's native citation system, not EndNote. Sources live as XML inside the .docx and are referenced by cite-key from `library.bib`.

To insert a citation:

1. **Find the cite-key.** Pull `library.bib`, search for the right entry by author + year + title:
   ```bash
   rclone cat cleanroom-drive:CBZpaper/library.bib | grep -A 8 "kaiser2024"
   ```
2. **Insert the citation field** at the target text position. Use docx-mcp's citation tools if available, or manipulate the OOXML directly:
   - The citation appears as `<w:fldSimple w:instr=" CITATION kaiser2024 \l 1031 ">` in `word/document.xml`
   - The source metadata goes into `customXml/itemX.xml` as a `<b:Source>` element with the same `<b:Tag>kaiser2024</b:Tag>`
3. **Mark it as a tracked insertion** so the user sees you added it.

Word renders the formatted citation (e.g., `(Kaiser, 2024)`) and updates the bibliography automatically when the user opens or refreshes the document. You do **not** insert formatted text or build a bibliography section — that is Word's job.

If you cannot find a matching cite-key in `library.bib`, do not invent one. Add a Word comment at the target location (`add_comment(para_id, "Citation needed: Kaiser 2024 — please add to library.bib")`) and let the user resolve it.

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
