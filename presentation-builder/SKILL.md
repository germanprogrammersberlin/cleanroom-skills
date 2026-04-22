---
name: presentation-builder
description: Create and edit PowerPoint presentations with python-pptx. Inspect existing slides, modify in place, add figures. Iterative workflow — never recreate from scratch.
---

# Presentation Builder

Create, inspect, and iteratively edit PowerPoint (.pptx) presentations using python-pptx.

**Core principle: always edit in place.** Open the existing file, modify what needs changing,
save back. Never recreate a presentation from scratch unless starting a new one.

## Installation

```bash
pip install python-pptx Pillow
```

## Inspect: See What's on Each Slide

Before editing, always inspect the current state. This script dumps every slide's
content so you know exactly what's there:

```python
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.enum.text import PP_ALIGN

def inspect(path):
    """Print a structured summary of every slide."""
    prs = Presentation(path)
    w = prs.slide_width
    h = prs.slide_height
    print(f"Slide size: {w/914400:.1f}\" x {h/914400:.1f}\"")
    print(f"Slides: {len(prs.slides)}\n")
    for i, slide in enumerate(prs.slides, 1):
        layout = slide.slide_layout.name if slide.slide_layout else "none"
        print(f"--- Slide {i} (layout: {layout}) ---")
        for shape in slide.shapes:
            print(f"  [{shape.shape_type}] name={shape.name!r} "
                  f"pos=({shape.left},{shape.top}) size=({shape.width},{shape.height})")
            if shape.has_text_frame:
                for p in shape.text_frame.paragraphs:
                    txt = p.text.strip()
                    if txt:
                        print(f"    text: {txt[:120]}")
            if shape.shape_type == 13:  # Picture
                print(f"    image: {shape.image.content_type} "
                      f"({shape.image.size[0]}x{shape.image.size[1]} px)")
        print()

inspect("presentation.pptx")
```

Run this BEFORE and AFTER every edit to verify your changes.

## Edit In Place

Always open the existing file, modify, and save back to the same path:

```python
prs = Presentation("presentation.pptx")

# Edit text on slide 1
slide = prs.slides[0]
for shape in slide.shapes:
    if shape.has_text_frame and "Title" in shape.name:
        shape.text_frame.paragraphs[0].text = "New Title"

prs.save("presentation.pptx")  # Save back to same file
```

### Find a shape by name or text content

```python
def find_shape(slide, name=None, containing=None):
    """Find a shape by name or by text content."""
    for shape in slide.shapes:
        if name and shape.name == name:
            return shape
        if containing and shape.has_text_frame:
            if containing.lower() in shape.text_frame.text.lower():
                return shape
    return None

title = find_shape(prs.slides[0], containing="Introduction")
```

### Replace text preserving formatting

```python
def replace_text(shape, old, new):
    """Replace text in a shape, keeping the original formatting."""
    for paragraph in shape.text_frame.paragraphs:
        for run in paragraph.runs:
            if old in run.text:
                run.text = run.text.replace(old, new)

replace_text(title, "Draft", "Final")
```

### Modify a specific paragraph

```python
slide = prs.slides[2]
shape = find_shape(slide, containing="Results")
para = shape.text_frame.paragraphs[0]
para.runs[0].text = "Updated results text"
para.runs[0].font.bold = True
```

## Create New Slides (When Needed)

### Add a title slide

```python
layout = prs.slide_layouts[0]  # Title Slide layout
slide = prs.slides.add_slide(layout)
slide.shapes.title.text = "Presentation Title"
slide.placeholders[1].text = "Author Name — Date"
```

### Add a content slide with bullet points

```python
layout = prs.slide_layouts[1]  # Title and Content
slide = prs.slides.add_slide(layout)
slide.shapes.title.text = "Key Findings"

body = slide.placeholders[1].text_frame
body.text = "First point"
for point in ["Second point", "Third point"]:
    p = body.add_paragraph()
    p.text = point
    p.level = 0
```

### Add a figure slide

```python
layout = prs.slide_layouts[5]  # Blank layout
slide = prs.slides.add_slide(layout)

# Add image centered
img_path = "figures/correlation_plot.png"
slide.shapes.add_picture(img_path, Inches(1), Inches(1.5), Inches(8), Inches(5))

# Add caption below
txBox = slide.shapes.add_textbox(Inches(1), Inches(6.6), Inches(8), Inches(0.5))
tf = txBox.text_frame
p = tf.paragraphs[0]
p.text = "Figure 1: Correlation length as a function of temperature."
p.font.size = Pt(11)
p.font.italic = True
p.alignment = PP_ALIGN.CENTER
```

## Delete / Reorder Slides

### Delete a slide

```python
def delete_slide(prs, index):
    """Delete slide at index (0-based)."""
    rId = prs.slides._sldIdLst[index].rId
    prs.part.drop_rel(rId)
    del prs.slides._sldIdLst[index]

delete_slide(prs, 3)  # Remove 4th slide
```

### Move a slide

```python
def move_slide(prs, old_index, new_index):
    """Move slide from old_index to new_index."""
    el = prs.slides._sldIdLst[old_index]
    prs.slides._sldIdLst.remove(el)
    prs.slides._sldIdLst.insert(new_index, el)

move_slide(prs, 5, 2)  # Move slide 6 to position 3
```

## Formatting Standards

### Scientific presentations

```python
# Standard slide size (widescreen)
prs.slide_width = Inches(13.333)
prs.slide_height = Inches(7.5)

# Font sizes
TITLE_SIZE = Pt(28)
BODY_SIZE = Pt(18)
CAPTION_SIZE = Pt(14)
AXIS_LABEL_SIZE = Pt(12)

# Colors
from pptx.util import Pt
from pptx.dml.color import RGBColor

DARK_TEXT = RGBColor(0x33, 0x33, 0x33)
ACCENT_BLUE = RGBColor(0x00, 0x70, 0xC0)
```

### Typography (same as DOCX)

```python
EN = '\u2013'  # en-dash for ranges: 5–10
MI = '\u2212'  # minus sign (not hyphen)
TS = '\u2009'  # thin space around operators
PM = '\u00b1'  # plus-minus: 5.2 ± 0.3
NB = '\u00a0'  # non-breaking space: Figure 4
```

## Workflow

1. **Inspect** the current file: run `inspect()` to see all slides and shapes
2. **Plan** what to change — identify shapes by name or content
3. **Edit** in place — open, modify, save to same path
4. **Inspect** again to verify changes
5. **Upload** to Google Drive when ready for review

Never guess what's on a slide. Always inspect first.
