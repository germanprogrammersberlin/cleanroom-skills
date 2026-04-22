---
name: image-editing
description: Edit existing images using Pillow and ImageMagick. Crop, composite, replace regions, overlay renders onto originals.
---

# Image Editing

Edit existing images — replace regions, composite layers, overlay renders onto
originals, crop, resize, adjust colors. Use this when you have a base image and
need to modify specific parts while keeping the rest intact.

**Core principle: preserve the original as much as possible.** Don't recreate
from scratch. Extract what works, fix what's broken, composite the result.

## Tools

- **Pillow (PIL)** — Python, pre-installed
- **ImageMagick** — CLI, pre-installed (`convert`, `composite`, `identify`)

## Inspect an Image

Always inspect before editing:

```python
from PIL import Image

img = Image.open('original.jpg')
print(f"Size: {img.size}")      # (width, height) in pixels
print(f"Mode: {img.mode}")      # RGB, RGBA, L, etc.
print(f"Format: {img.format}")  # JPEG, PNG, etc.
print(f"DPI: {img.info.get('dpi', 'unknown')}")
```

```bash
identify -verbose original.jpg | head -20
```

## Crop a Region

```python
from PIL import Image

img = Image.open('original.jpg')
w, h = img.size

# Crop left third
left_panel = img.crop((0, 0, w // 3, h))
left_panel.save('panel_left.png')

# Crop middle third
mid_panel = img.crop((w // 3, 0, 2 * w // 3, h))
mid_panel.save('panel_middle.png')

# Crop right third
right_panel = img.crop((2 * w // 3, 0, w, h))
right_panel.save('panel_right.png')
```

## Replace a Region (Paste)

Keep the original, replace only a specific area:

```python
from PIL import Image

original = Image.open('original.jpg').convert('RGBA')
replacement = Image.open('new_crystal_structure.png').convert('RGBA')

# Resize replacement to fit the target region
target_w, target_h = 400, 350
replacement = replacement.resize((target_w, target_h), Image.LANCZOS)

# Paste at position (x, y) — top-left corner of target region
x, y = 850, 50
original.paste(replacement, (x, y), replacement)  # 3rd arg = alpha mask

original.save('result.png')
```

## Composite with Alpha (Overlay)

Overlay a render with transparent background onto the original:

```python
from PIL import Image

base = Image.open('original.jpg').convert('RGBA')
overlay = Image.open('render_transparent.png').convert('RGBA')

# Resize overlay to match target area
overlay = overlay.resize((300, 300), Image.LANCZOS)

# Composite at position
result = base.copy()
result.paste(overlay, (100, 50), overlay)
result.save('composited.png')
```

## Replace a Region by Color Matching

Find and replace a specific colored region:

```python
from PIL import Image
import numpy as np

img = Image.open('original.jpg')
data = np.array(img)

# Find all pixels that are close to a target color (e.g. wrong blue)
target = np.array([100, 100, 200])  # RGB
tolerance = 40
mask = np.all(np.abs(data[:,:,:3].astype(int) - target) < tolerance, axis=2)

# Replace with new color
data[mask] = [72, 45, 120]  # new violet
Image.fromarray(data).save('recolored.png')
```

## Seamless Blending (Feathered Edges)

```python
from PIL import Image, ImageFilter

base = Image.open('original.jpg').convert('RGBA')
patch = Image.open('new_section.png').convert('RGBA')

# Create feathered mask
mask = Image.new('L', patch.size, 255)
feather = 20
for i in range(feather):
    alpha = int(255 * i / feather)
    # Fade edges
    mask.paste(alpha, (i, i, patch.width - i, patch.height - i))

patch.putalpha(mask)
base.paste(patch, (x, y), patch)
base.save('blended.png')
```

## ImageMagick CLI Operations

```bash
# Crop a region (WxH+X+Y)
convert original.jpg -crop 400x350+850+50 +repage cropped.png

# Resize
convert input.png -resize 800x600 output.png

# Composite (overlay with transparency)
composite -gravity center overlay.png base.jpg result.png

# Composite at specific position
composite -geometry +100+50 overlay.png base.jpg result.png

# Replace a color
convert input.png -fuzz 15% -fill '#482D78' -opaque '#6464C8' recolored.png

# Add text
convert input.png -pointsize 24 -font Arial -fill black \
  -annotate +50+400 'r-oxo-G' output.png

# Adjust brightness/contrast
convert input.png -brightness-contrast 10x20 output.png

# Make background transparent
convert input.png -fuzz 10% -transparent white output.png

# Combine images side by side
convert left.png middle.png right.png +append combined.png

# Combine images vertically
convert top.png bottom.png -append combined.png
```

## Workflow: Fix a Scientific Figure

When the client provides an existing figure that needs corrections:

1. **Inspect** the original — dimensions, DPI, color space
2. **Crop** the panels you need to fix into separate files
3. **Render** the corrected element (e.g. crystal structure) with transparent background, matching the size and style of the original
4. **Composite** the render back onto the original at the exact position
5. **Verify** the result — zoom in, check edges, compare with original
6. **Keep panels you don't need to fix** — don't redraw what already works

The goal is surgical replacement: change only what's wrong, preserve everything else.
