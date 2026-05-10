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

## Always emit a base64-embedded `*_preview.md` alongside the canonical

The user reviews markdown in a previewer (VS Code, Typora, Obsidian, browser) on a Drive-synced folder. Relative `../figures/...` paths are fragile in that setup: if Drive-Stream hasn't cached the PNG locally, or the previewer refuses parent-dir traversal, every figure shows as a broken icon.

So whenever you publish a markdown source file with image references, also publish a self-contained preview that has every image base64-inlined. Two files, one source of truth:

| File | Image syntax | Used by |
|---|---|---|
| `supporting_information.md` | `![caption](../figures/SI/Figure_S<N>.png)` | DOCX builder, machine-readable, version-control-friendly |
| `supporting_information_preview.md` | `![caption](data:image/png;base64,…)` | Human reviewer in any markdown previewer |

Same applies to `main_manuscript.md` → `main_manuscript_preview.md`.

### Generator pattern

```python
import base64, re
from pathlib import Path

IMG_RE = re.compile(r'!\[([^\]]*)\]\(([^)\s]+)\)')

def embed(md_path: Path, out_path: Path, image_root: Path):
    text = md_path.read_text(encoding='utf-8')
    def sub(m):
        alt, ref = m.group(1), m.group(2)
        # resolve ref against image_root (e.g. project root), not the md's folder
        clean = ref.lstrip('./').lstrip('/')
        candidates = [image_root / clean, (image_root / ref).resolve(),
                      image_root / Path(ref).name]
        f = next((c for c in candidates if c.is_file()), None)
        if f is None:
            return m.group(0)
        b64 = base64.b64encode(f.read_bytes()).decode('ascii')
        return f'![{alt}](data:image/png;base64,{b64})'
    out_path.write_text(IMG_RE.sub(sub, text), encoding='utf-8')
```

If the resulting preview file exceeds **8 MB**, downscale the source PNGs to 1600 px long edge (`Pillow.thumbnail((1600, 1600))`) before embedding — large embedded MD files freeze some previewers.

### When to regenerate the preview

Every time the canonical markdown changes — i.e. the preview is a build artefact, not a manually-edited file. After every edit cycle that produces a new `supporting_information.md`, immediately regenerate `supporting_information_preview.md` and push both to Drive together.

## What NOT to Do

- Do not change main text content beyond inserting SI references
- Do not decide what belongs in SI on your own — follow the finding or ask
- Do not use `*File:* path.png` pointer syntax (see above)
- Do not skip the `*_preview.md` companion file — the user can't review otherwise
