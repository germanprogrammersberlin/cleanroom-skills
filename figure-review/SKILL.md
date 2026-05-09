---
name: figure-review
description: Review figures visually (multimodal). Check axes, labels, colors, captions, consistency, data correctness. Read image files directly and compare with text.
---

# Figure Review

Review all figures visually. This requires multimodal capabilities (reading images).

## Check per Figure

1. **Consistent style** — Do all figures look like they belong together?
2. **Axis labels** — Readable? Units specified? Font size OK?
3. **Legends** — Order matches data order?
4. **Colors** — Consistent? Patterns in addition to colors (for B&W print)?
5. **Captions** — Detailed enough? All relevant info included?
6. **Placement** — Figure appears after first mention in text?
7. **Data correctness** — Do displayed values match values in text?

## How to Work

1. **Pre-flight every image before opening it** (see below)
2. Open each figure file (PNG, SVG, PDF)
3. Compare with the caption in the paper
4. Compare values in the figure with values in text
5. Check that figure is referenced in text

## Pre-flight (mandatory)

Image files reach the model through a base64 image block in the tool-result. The model's API rejects:

- single image > 5 MB
- non-PNG bytes labelled as `image/png`
- > ~100 images or > 32 MB total in one request

A single bad file fails the whole agent run with `adapter_failed: Image format image/png not supported` and the auto-retry will hit the same error. So **validate before reading**:

```python
from pathlib import Path
from PIL import Image

PNG_MAGIC = b"\x89PNG\r\n\x1a\n"
MAX_BYTES = 4_000_000  # 4 MB safety margin under 5 MB hard cap

def safe_for_review(path: Path) -> Path:
    raw = path.read_bytes()[:8]
    if not raw.startswith(PNG_MAGIC):
        raise ValueError(f"{path} is not a real PNG (magic bytes mismatch)")
    if path.stat().st_size <= MAX_BYTES:
        return path
    # Downscale for inspection — never modify the original
    small = path.with_suffix(".review.png")
    img = Image.open(path)
    img.thumbnail((1600, 1600))
    img.save(small, optimize=True)
    return small
```

Also cap the **number of images per single review pass**: at most 6 images per tool-result. If a figure-set is larger, list paths in text and review in batches.

## Output

Report: figure number, what exactly is wrong, severity.
