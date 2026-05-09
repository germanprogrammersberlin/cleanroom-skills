---
name: google-drive
description: Access the company's Google Shared Drive via rclone. Pull data, manuscripts and library files; ALWAYS push your work outputs back into the project's output/ subtree so the user (who lives in Google Drive, not on the server) can see and review them.
---

# Google Drive

Access the "Cleanroom Limited" Shared Drive. **The user lives in Google Drive on their Mac and reviews work there. They do not log into the server.** Anything you produce that the user is meant to see MUST be uploaded to Drive in a discoverable location — local-only output is invisible to them.

## When to Use

- **Pulling** raw data, manuscripts, library files, or briefings from the board
- **Pushing** your outputs back so the user can see them in Google Drive
- Reading or referencing PDFs, figures, or earlier draft versions
- Coordinating handoffs between agents (one agent's output is another's input)

## Setup

rclone is pre-configured with a Google service account (no token expiry, no interactive auth). The remote is called `cleanroom-drive` and points to the "Cleanroom Limited" Shared Drive root.

### Inside the agent container

The default rclone config is bind-mounted read-only-ish at:

```
/paperclip/.config/rclone/rclone.conf      # the default $HOME/.config location, picked up automatically
```

Plain `rclone <subcommand>` works for **all read operations** (`lsf`, `lsd`, `ls`, `cat`, `copy <remote>:src local/`). No `--config` flag needed.

**Caveat — write-config workaround.** Because the file is a single-file bind mount, rclone occasionally tries to atomically rewrite it (refresh state) and fails with `Failed to save config after 10 tries: ... rename ... device or resource busy`. This does not affect the operation that triggered it (data transfer succeeds), but the warning is noisy. To suppress it cleanly, copy the config once at the top of your work and point at the writable copy:

```bash
RCLONE_CFG=/tmp/rclone.conf
[ -f "$RCLONE_CFG" ] || cp /paperclip/.config/rclone/rclone.conf "$RCLONE_CFG"
export RCLONE_CONFIG="$RCLONE_CFG"
# now: rclone <whatever> works without warnings
```

### From the host (operator perspective)

The full Drive root is also mounted as a FUSE filesystem at:

```
/srv/paperclip/data/cleanroom-limited/vault    # on the host
```

This is for SSH/host-side inspection only — agents inside the container do not currently see this path (a future propagation update will expose it as `/srv/data/cleanroom-limited/vault`). Until then, agents must use `rclone` for all Drive interaction.

## Commands

### List files

```bash
rclone ls cleanroom-drive:                    # All files (recursive)
rclone lsd cleanroom-drive:                   # Top-level folders only
rclone ls cleanroom-drive:hCuI/drafts/        # Files in a specific folder
```

### Download files

```bash
# Single file to current directory
rclone copy cleanroom-drive:hCuI/briefing.pdf .

# Entire project folder
rclone copy cleanroom-drive:hCuI/ ./intake/ --progress

# Read a text file without downloading
rclone cat cleanroom-drive:hCuI/notes.txt
```

### Upload files

```bash
# Single file
rclone copy ./paper_v3.pdf cleanroom-drive:hCuI/drafts/

# Entire folder
rclone copy ./figures/ cleanroom-drive:hCuI/figures/ --progress
```

### Create folders

```bash
rclone mkdir cleanroom-drive:new-project/drafts
```

### Delete files

```bash
rclone delete cleanroom-drive:hCuI/drafts/old-version.pdf
```

## Folder convention — CBZ paper

The CBZ paper project on the Shared Drive is laid out like this:

```
cleanroom-drive:CBZpaper/
├── manuscript/
│   ├── main_manuscript.docx          ← canonical paper (Word native citations)
│   ├── main_manuscript.md            ← legacy markdown source (read-only reference)
│   ├── storyline.docx / storyline.md
│   ├── results_discussion.md
│   └── backups/                      ← timestamped DOCX backups (do not overwrite)
├── library.bib                       ← BibTeX library (single source of truth for citations)
├── code/                             ← analysis & figure-generation scripts
├── data/                             ← experimental raw data (CSV, .dat, .npz, …)
├── figures/                          ← PRODUCTION figures + .blend / .pptx sources
│   └── SI/                           ← supplementary figures
├── literature/                       ← reference PDFs + cbz paper library.enl
├── output/                           ← AGENT OUTPUTS land here (see below)
│   ├── drafts/                       ← prose drafts, analysis writeups, proposals
│   ├── figures/                      ← agent-generated figure variants and renders
│   └── reviews/                      ← review notes, devils-advocate reports, scoring
└── presentations/                    ← project meeting decks
```

## Read-only zones vs. agent workspace

The Drive is split into a **canonical (read-only) zone** and an **agent workspace**:

| Zone | Path | Agents can read | Agents can write |
|---|---|---|---|
| Canonical files | `manuscript/`, `code/`, `data/`, `figures/`, `literature/`, `presentations/`, `library.bib`, `cbz paper library.enl` | ✅ | ❌ |
| Agent workspace | `output/` (and all subfolders) | ✅ | ✅ |

**The user owns everything outside `output/`.** Agents pull from canonical, work in `output/`, and never write back to canonical. The user can have any of the canonical files open in Word/Excel/etc. at any time without blocking agents — agent edits live entirely in `output/` clones.

If you need to modify a canonical file (e.g. propose a manuscript edit), the workflow is:

1. Pull the canonical file (read-only access).
2. Save your local working copy under a versioned filename in `output/<subfolder>/`.
3. Edit the workspace clone.
4. Push the workspace clone to `output/<subfolder>/`.
5. Tell the user where the clone is. They decide whether/how to integrate it into the canonical.

## Where to put YOUR outputs

| Output type | Drive destination | Filename pattern |
|---|---|---|
| Proposed edits to the canonical paper | `output/drafts/` | `main_manuscript__<agent>__<YYYY-MM-DD>__<topic>.docx` |
| New prose draft, analysis writeup, proposed section rewrite | `output/drafts/` | `<topic>__<agent>__<YYYY-MM-DD>.md` |
| Figure render or variant generated by an agent | `output/figures/` | `<figure-id>__<variant>__<YYYY-MM-DD>.png` |
| Review note, devils-advocate report, paper-scoring | `output/reviews/` | `<topic>__<reviewer>__<YYYY-MM-DD>.md` |
| Updated library entry proposal | comment / inline section inside an `output/drafts/` file (NEVER a direct write to `library.bib`) | — |

The `__` (double underscore) separator keeps filenames machine-parseable.

**Hard rule:** if your `rclone copy` destination path starts with anything other than `cleanroom-drive:CBZpaper/output/`, stop and ask in the issue thread for explicit permission before proceeding.

## Standard pull-clone-edit-push workflow

```bash
# 1. PULL the canonical (read-only access) into your local workspace
mkdir -p /tmp/cbz/clone
rclone copy cleanroom-drive:CBZpaper/manuscript/main_manuscript.docx /tmp/cbz/clone/

# 2. RENAME to a workspace clone immediately — so you can never accidentally
#    push it back to the canonical location with the original name.
CLONE="main_manuscript__AnalysisScientist__$(date -I)__binding-kinetics.docx"
mv /tmp/cbz/clone/main_manuscript.docx "/tmp/cbz/clone/$CLONE"

# 3. WORK on the clone locally (edit, render, analyze...)
# ... your tools / docx-mcp / matplotlib / whatever ...

# 4. PUSH the clone to output/ on Drive — NOT to manuscript/
rclone copy "/tmp/cbz/clone/$CLONE" cleanroom-drive:CBZpaper/output/drafts/
```

`rclone copy` is non-destructive: identical files are skipped, newer local versions overwrite older Drive versions. Use `rclone sync` only when you intend to mirror a whole folder including deletions (rare — usually `copy` is what you want).

## Common rclone recipes

```bash
# List the project layout
rclone lsf cleanroom-drive:CBZpaper/ --dirs-only
rclone lsf cleanroom-drive:CBZpaper/output/drafts/

# Read a small file inline without downloading
rclone cat cleanroom-drive:CBZpaper/library.bib | head -30

# Copy a single file into a subfolder
rclone copy ./figure_v2.png cleanroom-drive:CBZpaper/output/figures/

# Copy an entire folder (preserves structure)
rclone copy ./output/drafts/ cleanroom-drive:CBZpaper/output/drafts/ --progress

# Create a new folder
rclone mkdir cleanroom-drive:CBZpaper/output/drafts/2026-05/
```

## Best practices

- **Date-stamp filenames** so the user can sort by recency: `..._2026-05-02.md`
- **Include your agent name** in the filename so the user knows who produced it
- **Never delete files on Drive.** If a draft is superseded, the new file gets a new date — old one stays for history
- **Keep `manuscript/` clean.** That folder is for canonical files. Put work-in-progress in `output/drafts/`
- **Backups are sacred.** `manuscript/backups/` is for time-stamped snapshots of the main paper — do not overwrite
- **Local cache lives outside Drive.** Use `/tmp/cbz/` (or similar) for your local working copy. Push selected results back; do not push everything you touched

## Anti-patterns

- ❌ Pushing anything to `cleanroom-drive:CBZpaper/manuscript/`, `code/`, `data/`, `figures/`, `library.bib`, `literature/`, or `presentations/`. These are read-only for agents; if your `rclone copy` destination is not under `output/`, stop and ask first.
- ❌ Doing all the work locally and only summarizing in your reply — the user lives in Google Drive and can't review the artifacts otherwise
- ❌ Pushing scratch / temporary files to Drive and cluttering `output/`
- ❌ Building a "fresh" docx from `main_manuscript.md` — markdown is not a source of truth, and the rebuilt file would lose the user's accumulated edits and citations
- ❌ Inventing new top-level folders on Drive without coordination — stay within the existing layout
- ❌ Pulling the whole CBZpaper/ tree (large) when you only need one file
