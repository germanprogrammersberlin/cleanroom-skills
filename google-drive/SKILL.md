---
name: google-drive
description: Access the company's Google Shared Drive via rclone. Download source documents from the board, upload drafts and figures, organize research materials by project.
---

# Google Drive

Access the "Cleanroom Limited" Shared Drive for document storage and exchange with the Advisory Board.

## When to Use

- Downloading source documents, data files, or briefings provided by the board
- Uploading draft papers and final versions
- Sharing figures and supplementary materials
- Organizing research materials by project

## Setup

rclone is pre-configured with a service account. Config location:

```
/root/.config/rclone/rclone.conf
```

The remote is called `cleanroom-drive` and points to the Shared Drive.

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

## Folder Convention

Each project gets its own top-level folder on the Shared Drive:

```
(Shared Drive Root)/
  {project-name}/
    intake/           ← Raw data and briefings from the board
    drafts/           ← Paper versions (paper_v1.pdf, paper_v2.pdf, ...)
    figures/           ← Plots, schemata, images
    supplementary/     ← SI material
    references/        ← PDFs of cited papers
    final/             ← Accepted/submitted version
```

## Best Practices

- Use descriptive filenames with version or date: `paper_v2_2026-04-21.pdf`
- Never delete previous versions — move to an `archive/` subfolder
- Download intake materials at the start of a project, work locally, upload results
- The board drops files into `intake/` — check there first for new assignments
