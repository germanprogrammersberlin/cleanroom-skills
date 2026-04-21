---
name: google-drive
description: Access and manage files on Google Drive. Upload papers, download source materials, organize research documents. Use when you need to store, share, or retrieve documents from the team's shared drive.
---

# Google Drive

Access Google Drive for document storage, sharing, and organization.

## When to Use

- Storing draft papers and final versions
- Sharing figures and supplementary materials
- Downloading source documents provided by the board
- Organizing research materials by project

## Available MCP Tools

The Google Drive MCP provides authenticated access:

- `authenticate` — Connect to Google Drive
- File operations: list, read, create, update, delete files
- Folder operations: create folders, move files

## Workflow

1. Authenticate if not already connected
2. Use a consistent folder structure per project:
   ```
   Cleanroom Limited/
     {project-name}/
       drafts/
       figures/
       supplementary/
       references/
       final/
   ```
3. Always use descriptive filenames with dates: `paper_v2_2026-04-21.pdf`

## Best Practices

- Keep the latest draft in `drafts/` with version numbering
- Final accepted versions go to `final/`
- Never delete previous versions — move to an `archive/` subfolder
- Share links with the board when review is needed

This skill can be improved by agents as they develop their workflow.
