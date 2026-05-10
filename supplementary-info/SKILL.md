---
name: supplementary-info
description: Manage Supplementary Information. Move detailed derivations, extra data, and secondary figures from main text to SI. Insert SI references. Keep numbering consistent.
---

# Supplementary Information

Manage the Supplementary Information document.

## What to Do

- Move detailed derivations from main text to SI
- Move secondary figures and tables to SI
- Insert SI references in main text: "see Supporting Information, Fig. S1"
- Keep SI numbering consistent: Fig. S1, S2, ...; Table S1, S2, ...
- Ensure the main text remains understandable without SI

## Markdown must render standalone — no `*File:*` pointer syntax

Both `main_manuscript.md` and `supporting_information.md` are read by humans in Markdown preview (VS Code, Typora, GitHub, Obsidian). They MUST render correctly as Markdown — figures must show as actual images, not as text pointers.

**Use standard Markdown image syntax, always:**

```markdown
![Figure S1. CBZ adsorption isotherm on the MIP film at 295 K. (a) Kinetic trace, (b) equilibrium isotherm with Langmuir fit, (c) Freundlich linearization.](figures/SI/Figure_S1.png)
```

The DOCX build pipeline (`paper-docx-builder`) reads these `![alt](path)` references and embeds the actual PNG into the Word document with the alt text as the figure caption.

**Never write the legacy pointer convention:**

```markdown
### Figure S1 — CBZ adsorption isotherm
*File:* `Figure_S1.png` (panels a/b/c).
Caption text here…
```

That syntax does not render in Markdown previews — the file path appears as text and the image is invisible. It is also fragile for the docx builder (path is in code-spans, caption is split across multiple paragraphs).

If you encounter the legacy `*File:*` syntax in an existing file, convert it to `![caption](path)` as part of the next touch on that file. Keep the caption as the alt text so the human can read it directly in the preview.

## What NOT to Do

- Do not change main text content beyond inserting SI references
- Do not decide what belongs in SI on your own — follow the finding or ask
- Do not use `*File:* path.png` pointer syntax (see above)
