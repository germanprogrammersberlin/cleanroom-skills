---
name: scientific-illustration
description: Compose publication-quality scientific illustrations from multiple elements. Combine crystal structures, plots, schemata, and labels into multi-panel figures using Inkscape CLI and SVG. Final output as PNG/TIFF/PDF at journal resolution.
---

# Scientific Illustration

Compose publication-quality multi-panel scientific figures from individual elements
(renders, plots, diagrams) using Inkscape CLI and SVG.

Use this skill when you need to combine outputs from other skills (crystal-structure,
matplotlib-plotting, blender-3d) into a single coherent figure with labels, annotations,
and consistent styling.

## Tools

- **Inkscape** (CLI, headless) — SVG manipulation, PDF/PNG/TIFF export
- **Python + svgwrite** — Programmatic SVG creation
- **matplotlib** — Generate individual panels as SVG
- **PIL/Pillow** — Raster image compositing (fallback)

## Inkscape CLI Basics

```bash
# Convert SVG to PNG at 300 dpi
inkscape input.svg --export-type=png --export-dpi=300 --export-filename=output.png

# Convert SVG to PDF
inkscape input.svg --export-type=pdf --export-filename=output.pdf

# Convert SVG to TIFF (via PNG intermediate)
inkscape input.svg --export-type=png --export-dpi=600 --export-filename=temp.png
python3 -c "from PIL import Image; Image.open('temp.png').save('output.tiff', compression='tiff_lzw')"

# Get SVG dimensions
inkscape input.svg --query-width --query-height

# Export specific area
inkscape input.svg --export-area=0:0:800:400 --export-type=png --export-dpi=300 --export-filename=crop.png
```

## Build Multi-Panel Figures in SVG

```python
import svgwrite

# Journal-standard TOC image: ~8 cm wide, ~4 cm tall, 300 dpi
# At 300 dpi: 8cm = 945px, 4cm = 472px
W, H = 945, 472
PANEL_W = W // 3

dwg = svgwrite.Drawing('figure.svg', size=(f'{W}px', f'{H}px'),
                         viewBox=f'0 0 {W} {H}')

# Background
dwg.add(dwg.rect(insert=(0, 0), size=(W, H), fill='white'))

# Panel 1 (left) — embed a rendered image
dwg.add(dwg.image(href='panel_left.png', insert=(0, 0),
                    size=(f'{PANEL_W}px', f'{H}px')))

# Panel 2 (middle)
dwg.add(dwg.image(href='panel_middle.png', insert=(PANEL_W, 0),
                    size=(f'{PANEL_W}px', f'{H}px')))

# Panel 3 (right)
dwg.add(dwg.image(href='panel_right.png', insert=(2*PANEL_W, 0),
                    size=(f'{PANEL_W}px', f'{H}px')))

# Add text labels (editable, not baked into images)
dwg.add(dwg.text('(a)', insert=(10, 20),
         font_size='14px', font_family='Arial', fill='black'))
dwg.add(dwg.text('(b)', insert=(PANEL_W + 10, 20),
         font_size='14px', font_family='Arial', fill='black'))
dwg.add(dwg.text('(c)', insert=(2*PANEL_W + 10, 20),
         font_size='14px', font_family='Arial', fill='black'))

# Arrows, lines, annotations
dwg.add(dwg.line(start=(300, 200), end=(350, 200),
         stroke='black', stroke_width=2, marker_end='url(#arrow)'))

dwg.save()
```

## Add Arrows and Markers

```python
# Define arrowhead marker
marker = dwg.marker(insert=(10, 5), size=(10, 10), orient='auto')
marker.add(dwg.path(d='M0,0 L10,5 L0,10 Z', fill='black'))
dwg.defs.add(marker)

# Use in lines
line = dwg.line(start=(100, 100), end=(200, 100),
                stroke='black', stroke_width=1.5)
line['marker-end'] = marker.get_funciri()
dwg.add(line)
```

## Add Curved Arrows and Brackets

```python
# Curved arrow (quadratic Bezier)
path = dwg.path(d='M 100,200 Q 150,150 200,200',
                stroke='black', fill='none', stroke_width=1.5)
path['marker-end'] = marker.get_funciri()
dwg.add(path)

# Curly bracket
dwg.add(dwg.text('{', insert=(50, 150),
         font_size='60px', font_family='serif', fill='black'))
```

## Text Styling for Scientific Figures

```python
# Regular label
dwg.add(dwg.text('h-CuI', insert=(x, y),
         font_size='11px', font_family='Arial', fill='black'))

# Subscript (use tspan)
txt = dwg.text('Cu', insert=(x, y), font_size='11px', font_family='Arial')
txt.add(dwg.tspan('3', baseline_shift='sub', font_size='8px'))
txt.add(dwg.tspan('I', font_size='11px'))
txt.add(dwg.tspan('3', baseline_shift='sub', font_size='8px'))
dwg.add(txt)

# Italic
dwg.add(dwg.text('E', insert=(x, y),
         font_size='12px', font_family='Arial', font_style='italic'))

# Bold
dwg.add(dwg.text('Figure 1', insert=(x, y),
         font_size='12px', font_family='Arial', font_weight='bold'))
```

## Pillow Compositing (Fallback for Raster)

```python
from PIL import Image, ImageDraw, ImageFont

# Create canvas
W, H = 2835, 1417  # 8cm x 4cm at 900 dpi
canvas = Image.new('RGB', (W, H), 'white')

# Paste panels
left = Image.open('panel_left.png').resize((W//3, H))
canvas.paste(left, (0, 0))

middle = Image.open('panel_middle.png').resize((W//3, H))
canvas.paste(middle, (W//3, 0))

right = Image.open('panel_right.png').resize((W//3, H))
canvas.paste(right, (2*W//3, 0))

# Add text
draw = ImageDraw.Draw(canvas)
try:
    font = ImageFont.truetype('/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf', 36)
except:
    font = ImageFont.load_default()
draw.text((20, 20), '(a)', fill='black', font=font)

canvas.save('figure.png', dpi=(300, 300))
canvas.save('figure.tiff', dpi=(300, 300), compression='tiff_lzw')
```

## Journal Requirements

| Journal | TOC size | Format | DPI |
|---------|----------|--------|-----|
| ACS (most) | 8.25 cm × 4.76 cm | TIFF/PNG | 300-600 |
| Wiley | 5-8 cm wide | TIFF/EPS | 300 |
| Elsevier | max 531 × 1328 px | JPEG/TIFF | 96-300 |
| Nature | 180 mm wide max | TIFF/EPS | 300 |
| RSC | 8.3 cm × 4.0 cm | PNG/TIFF | 600 |

## Workflow

1. **Gather elements**: renders from crystal-structure, plots from matplotlib, icons
2. **Set canvas size** based on journal requirements
3. **Build SVG** with svgwrite — embed images, add text as real text (not baked in)
4. **Export** via Inkscape CLI to PNG/TIFF at required DPI
5. **Also deliver SVG** or PPTX so client can edit text
6. Verify: correct DPI, correct dimensions, text readable at print size
