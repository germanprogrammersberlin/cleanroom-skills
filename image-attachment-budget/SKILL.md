---
name: image-attachment-budget
description: Enforce per-message image budgets so agent runs never crash on `adapter_failed: Image format image/png not supported`. Applies whenever an agent emits a tool-result that contains image content (figure review, multi-panel composition, comment with embedded screenshots, etc.).
---

# Image Attachment Budget

Anthropic's API has hard limits on image content per request. Hitting them aborts the whole agent run with `adapter_failed`, and the auto-retry will repeat the same failure forever — which is exactly the failure mode that has historically blocked figure-composition issues.

## Hard budgets — never exceed

| Limit | Value | Why |
|---|---|---|
| Max bytes per image | **4 MB** | Anthropic caps at 5 MB; 4 MB is the safety margin for base64 overhead |
| Max images per tool-result | **6** | Avoids hitting the 100-image-per-request and 32-MB total caps |
| Max total bytes per tool-result | **20 MB** | Total request payload incl. base64 expansion (~33 % overhead) |
| Allowed declared formats | **`image/png`, `image/jpeg`, `image/webp`, `image/gif`** | Must match actual magic bytes — see below |

## Decision tree before embedding any image

1. **Is the file a real image of the declared format?** Check magic bytes.
   - PNG: `89 50 4E 47 0D 0A 1A 0A`
   - JPEG: `FF D8 FF`
   - WebP: `52 49 46 46 .. .. .. .. 57 45 42 50`
   - GIF: `47 49 46 38`
2. **Is the file ≤ 4 MB?** If not → downscale to a `.review.<ext>` copy at ≤ 1600 px long edge. Never modify the original artifact on Drive.
3. **How many images is this tool-result already carrying?** If ≥ 6 → don't embed; output a text list of paths instead.
4. **Will the cumulative payload exceed 20 MB?** Same rule: text list of paths.

If any check fails, **do not embed**. Output a text reference (`figures/SI/_regen/Figure_S15.png — 7.3 MB, exceeds 4 MB budget`) and let the next step re-fetch one at a time.

## Helper

```python
from pathlib import Path
from PIL import Image

MAGIC = {
    b"\x89PNG\r\n\x1a\n": "image/png",
    b"\xff\xd8\xff":      "image/jpeg",
    b"GIF87a":            "image/gif",
    b"GIF89a":            "image/gif",
}
MAX_BYTES = 4_000_000
MAX_LONG_EDGE = 1600

def detect_mime(path: Path) -> str | None:
    head = path.read_bytes()[:16]
    for prefix, mime in MAGIC.items():
        if head.startswith(prefix):
            return mime
    if head[:4] == b"RIFF" and head[8:12] == b"WEBP":
        return "image/webp"
    return None

def prepare_for_embed(path: Path) -> Path | None:
    """Returns a path safe to embed, or None if the file should be referenced as text."""
    mime = detect_mime(path)
    if mime is None:
        return None  # not an image — never embed
    if path.stat().st_size <= MAX_BYTES:
        return path
    img = Image.open(path)
    img.thumbnail((MAX_LONG_EDGE, MAX_LONG_EDGE))
    out = path.with_suffix(".review" + path.suffix)
    img.save(out, optimize=True)
    if out.stat().st_size > MAX_BYTES:
        return None  # still too big — let the caller fall back to text reference
    return out
```

## Multi-panel composition — special case

When composing N source panels into one figure, **do not** open all N source PNGs in a single tool-result. Use this pattern instead:

1. Read panel paths and sizes (text only, no images embedded).
2. Open at most 6 panels per inspection batch — using the helper above.
3. After composition, save the composite at REVIEW_DPI (150). Validate ≤ 4 MB. Only then emit the composite back as an embedded image.
4. Source PNGs at 300 dpi stay on Drive untouched. The composite uploaded to Drive can be re-rendered at 300 dpi in a separate, headless render step (no images in tool-result).

## Why this skill exists

Past failure: CLE-60 (CBZ SI multi-panel composition) sat blocked because a single tool-result contained 31 base64-PNG image blocks, one of which exceeded the 5 MB cap. The Anthropic API returned `messages.9.content.30.image.source.base64.data: Image format image/png not supported`. Auto-retry replayed the same payload. Following this skill prevents that failure mode entirely.
