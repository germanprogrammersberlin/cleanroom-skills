---
name: paper-docx-builder
description: Build journal-quality DOCX manuscripts using python-docx. Proper headings (italic, no horizontal rules), Times New Roman 12pt, double-spaced, justified, with correct subscripts, superscripts, and typographic characters. Use when producing manuscript submissions.
---

# Paper DOCX Builder

Build journal-quality DOCX manuscripts programmatically using python-docx. Produces clean formatting that journals expect — black italic headings (not Word's default blue headings with horizontal rules), proper typography, and correct academic formatting.

## Markdown is the source of truth — DOCX is rendered from it

The author and reviewers read `main_manuscript.md` / `supporting_information.md` directly in Markdown preview. The DOCX is **always** built from the current Markdown — never edit the DOCX as the canonical source.

Figure references in the Markdown use **standard Markdown image syntax** (so the preview renders the actual figure, not a file-path string):

```markdown
![Figure 1. Schematic of the SG-GFET architecture. (a) Synthesis, (b) DSA, (c) Detection.](figures/Figure_1.png)
```

When you build the DOCX, embed the referenced PNG with the alt text as the caption:

```python
from docx.shared import Inches
import re
import os

MD_IMG_RE = re.compile(r'!\[([^\]]*)\]\(([^)]+)\)')

def embed_md_image(doc, alt_text, png_path, width_in=6.0):
    """Embed a PNG referenced from Markdown with its alt text as caption."""
    p = doc.add_paragraph()
    p.alignment = WD_ALIGN_PARAGRAPH.CENTER
    p.add_run().add_picture(png_path, width=Inches(width_in))
    cap = doc.add_paragraph()
    cap.paragraph_format.alignment = WD_ALIGN_PARAGRAPH.CENTER
    R(cap, alt_text)
```

**Do not** parse the legacy `*File:* \`Figure_S1.png\`` pointer convention — convert any encountered legacy markdown to standard `![](path)` first, then build. See the `supplementary-info` skill for the migration rule.

### Build from the canonical markdown, never from the preview

The DOCX builder reads `main_manuscript.md` / `supporting_information.md` (the ones with relative `![caption](../figures/SI/Figure_S<N>.png)` paths). It does **not** read `*_preview.md` — those are base64-embedded artefacts for human review only. Embedded base64 inflates conversation context and breaks the figure-vs-text separation the docx builder needs.

After every successful DOCX build, also regenerate the matching `*_preview.md` (see the `supplementary-info` skill for the embed pattern) so the human reviewer's preview file stays in lockstep with the canonical markdown and the freshly built DOCX.

## Installation

```bash
pip install python-docx
```

## Core Functions

These helper functions produce journal-standard formatting:

```python
from docx import Document
from docx.shared import Pt, Emu
from docx.enum.text import WD_ALIGN_PARAGRAPH

# Typographic constants
TS = '\u2009'  # thin space
EN = '\u2013'  # en-dash
AP = '\u2248'  # approximately
MI = '\u2212'  # minus sign (not hyphen)
PM = '\u00b1'  # plus-minus
NB = '\u00a0'  # non-breaking space
TI = '\u223c'  # tilde operator


def setup_doc():
    """Create a document with journal-standard page setup."""
    doc = Document()
    s = doc.sections[0]
    # US Letter, 1-inch margins
    s.page_width = 7772400; s.page_height = 10058400
    s.left_margin = s.right_margin = s.top_margin = 899795
    s.bottom_margin = 720090
    # Base style: Times New Roman 12pt, double-spaced, justified
    sn = doc.styles['Normal']
    sn.font.name = 'Times New Roman'; sn.font.size = Pt(12)
    sn.paragraph_format.line_spacing = 2.0
    sn.paragraph_format.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
    sn.paragraph_format.space_after = Pt(0)
    sn.paragraph_format.space_before = Pt(0)
    return doc


def R(p, text, bold=False, italic=False, sub=False, sup=False):
    """Add a run with formatting. Use for inline text with mixed styles."""
    r = p.add_run(text)
    r.font.name = 'Times New Roman'; r.font.size = Pt(12)
    r.bold = bold; r.italic = italic
    if sub: r.font.subscript = True
    if sup: r.font.superscript = True
    return r


def body(doc):
    """Add a body paragraph with first-line indent."""
    p = doc.add_paragraph()
    pf = p.paragraph_format
    pf.line_spacing = 2.0; pf.space_after = Pt(0); pf.space_before = Pt(0)
    pf.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
    pf.first_line_indent = Emu(128270)
    return p


def heading(doc, text):
    """Add a section heading — italic, no horizontal rules, no blue color."""
    p = doc.add_paragraph()
    pf = p.paragraph_format
    pf.line_spacing = 2.0; pf.space_after = Pt(0); pf.space_before = Pt(12)
    pf.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
    R(p, text, italic=True)
    return p


def add_refs(doc, refs):
    """Add a references section with hanging indent.
    refs: list of (number, text) tuples."""
    p = doc.add_paragraph()
    p.paragraph_format.line_spacing = 2.0
    p.paragraph_format.space_after = Pt(0)
    p.paragraph_format.space_before = Pt(18)
    p.paragraph_format.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
    R(p, 'References', bold=True)
    for num, text in refs:
        p = doc.add_paragraph()
        pf = p.paragraph_format
        pf.line_spacing = 2.0; pf.space_after = Pt(0); pf.space_before = Pt(0)
        pf.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
        pf.left_indent = Emu(254000); pf.first_line_indent = Emu(-254000)
        R(p, f'[{num}]{TS}')
        R(p, text)
```

## Usage Example

```python
doc = setup_doc()

heading(doc, 'Introduction')

p = body(doc)
R(p, 'The measurement of ')
R(p, 'K', italic=True)
R(p, 'D', sub=True, italic=True)
R(p, f'{TS}={TS}0.022{TS}nM was performed using SPR.')
R(p, '31', sup=True)

heading(doc, 'Results and Discussion')

p = body(doc)
R(p, f'The limit of detection is {TI}5{TS}aM (S-protein) '
     f'and {TI}95{TS}aM (N-protein).')

add_refs(doc, [
    (31, 'Eshaghi, G.; Kaiser, D.; et al., ...'),
])

doc.save('manuscript.docx')
```

## Why Not Word Headings?

Word's built-in `Heading 1`, `Heading 2` styles produce blue text with horizontal rules, which journals reject. The `heading()` function creates proper italic headings that match ACS, Elsevier, Springer manuscript guidelines.

## Typography Rules

- Use `EN` (en-dash \u2013) for ranges: `5{EN}10`
- Use `MI` (minus \u2212) for negative numbers, not hyphen
- Use `TS` (thin space \u2009) around operators: `5{TS}={TS}10`
- Use `NB` (non-breaking space \u00a0) to prevent line breaks: `Figure{NB}4`
- Use `PM` (\u00b1) for uncertainties: `5.2{TS}{PM}{TS}0.3`
- Superscript references inline: `R(p, '31', sup=True)`
- Subscripts for chemical/physical notation: `R(p, 'D', sub=True, italic=True)`

This skill can be improved by agents as they encounter specific journal requirements.
