---
name: german-letter-ocr
description: Local, air-gapped two-phase OCR + classification pipeline for scanned German correspondence (creditor letters, court orders, official mail). Phase 1 extracts text per page (PDF embedded text first via PyMuPDF, Tesseract `lang=deu` fallback with OpenCV preprocessing). Phase 2 sends all page texts in one batch to a local Ollama model (Gemma 3) with a fixed JSON schema to group multi-page documents and extract sender/recipient/letter-type/amount/case-number/deadline. Use ONLY when the operator requires air-gap (no cloud) or for mass ingestion where per-PDF Claude calls would be cost-prohibitive (hundreds of letters at once). For everything else use the `document-extract` skill (Claude Document API), which is the project default per architecture rule. Do NOT use for handwritten notes, foreign-language letters, or single-document workflows where Claude would be more accurate.
---

# German Letter OCR (Air-Gapped Variant)

Local, two-phase OCR + classification pipeline. Built for the BriefButler
desktop product — a Tauri app that runs **fully offline on the user's
machine** and never calls a cloud API. Reusable for any stack of structured
German correspondence in air-gap or mass-ingestion contexts.

> **This skill is the exception, not the rule.** The project's binding
> architecture rule is: PDFs go through Claude's Document API via
> `document-extract`. This skill exists for two specific situations:
>
> 1. **Air-gap requirement** — operator policy or product design forbids
>    cloud calls (BriefButler desktop, regulated environments).
> 2. **Mass ingestion** — hundreds of routine letters at once where
>    per-PDF Claude calls would be cost-prohibitive. The recommended pattern
>    is: this skill as a cheap pre-filter, then `document-extract` (Claude)
>    on the uncertain or high-stakes subset.
>
> If neither applies → use `document-extract`. It's faster to set up and
> more accurate per document.

## When to Use

- Operator requires no data ever leaves the local machine (BriefButler
  desktop product, debtor-data privacy policy)
- Mass ingestion of hundreds of routine letters where Claude per-PDF cost
  is unacceptable — pre-filter with this skill, then escalate uncertain
  cases to `document-extract`
- Splitting a multi-letter "Sammelmappe" PDF into individual documents
  based on letterhead detection, locally
- Mixed input where some pages are digital PDFs (cheap embedded-text
  extraction) and others are photographed scans (need OCR)

## When NOT to Use

- **Default case for any single PDF in an Akte** → use `document-extract`
  instead (Claude Document API is more accurate and produces a richer
  `extract.md`)
- Handwritten notes (Tesseract is poor at handwriting; Claude vision via
  `document-extract` does better)
- Non-German letters — prompts and regex patterns are tuned to German
- Image-only scans without recoverable text (low-res faxes — pre-process
  upstream)
- Latency-critical paths (Phase 2 LLM call takes 10–60 s per batch)

## Pipeline overview

```
PDF/Image input
     │
     ▼
┌──────────────────────────────────────────────────────┐
│ PHASE 1 — Per-page text extraction                   │
│                                                      │
│   PDF? ──► PyMuPDF page.get_text()                   │
│             │                                        │
│             ├─ length > 80 chars → use this text     │
│             └─ length ≤ 80 chars → render to PNG     │
│                                  → Tesseract OCR     │
│                                                      │
│   Image? ─► Tesseract OCR (with OpenCV preprocessing)│
└──────────────────────────────────────────────────────┘
     │
     ▼  page_texts: dict[int, str]
┌──────────────────────────────────────────────────────┐
│ PHASE 2 — Batch classification                       │
│                                                      │
│   Concatenate all pages with [SEITE N] markers       │
│   Run regex extractor → produce [REGEX-ERGEBNISSE]   │
│   Send to Ollama gemma3:12b (one call, JSON schema)  │
│   Parse JSON array → one entry per logical document  │
│                                                      │
│   Ollama unreachable? → regex-only fallback          │
└──────────────────────────────────────────────────────┘
     │
     ▼  list[Document] with seiten/absender/brieftyp/...
```

## Dependencies

| Package | Version | Purpose | Required? |
|---------|---------|---------|-----------|
| `pytesseract` | any | Tesseract Python wrapper | ✅ |
| `Pillow` | any | Image loading | ✅ |
| `PyMuPDF` (`fitz`) | any | PDF → text + PDF → images | ✅ |
| `requests` | any | Ollama HTTP client | ✅ |
| `opencv-python`, `numpy` | any | Preprocessing (deskew, threshold) | ⛔ optional but recommended |
| Tesseract binary | ≥ 5.0 | OCR engine — install `tesseract-ocr` + `tesseract-ocr-deu` system-wide | ✅ |
| Ollama daemon | ≥ 0.3 | local LLM server on `http://localhost:11434` | ⛔ optional (regex fallback if absent) |
| Model `gemma3:12b` | ~8 GB | pulled via `ollama pull gemma3:12b` | ⛔ optional |

The pipeline degrades gracefully:

- No OpenCV → skip preprocessing, raw Tesseract still works
- No Ollama → fall back to regex-only extraction (still produces structured output, lower recall)
- No Tesseract → only PDFs with embedded text survive; image scans return empty

## Constants (do not change without justification)

```python
_OLLAMA_URL    = "http://localhost:11434/api/chat"
_OLLAMA_BASE   = "http://localhost:11434"
_MODEL         = "gemma3:12b"
_MIN_TEXT_LEN  = 80           # pages below this fall through to OCR
_TESS_LANG     = "deu"
_TESS_CONFIG   = "--psm 6 --dpi 300"   # single uniform block of text
_PDF_RENDER_DPI= 200          # PyMuPDF rendering for OCR fallback
```

`PSM 6` (assume a single uniform block of text) is correct for letters; do
not switch to `PSM 3` or `PSM 1` — they segment too aggressively and break
on multi-column letterheads.

`DPI 200` for rendering + Tesseract internal `--dpi 300` is the sweet spot
on consumer hardware. Lower DPI loses Aktenzeichen digits; higher DPI
quadruples Tesseract runtime without recall improvement.

## Phase 1 — Text extraction

### PDF embedded text first

```python
import fitz  # PyMuPDF

def extract_pdf_texts(pdf_path: Path) -> list[str]:
    doc = fitz.open(str(pdf_path))
    texts = [page.get_text().strip() for page in doc]
    doc.close()
    return texts
```

### PDF → PNG fallback for OCR

```python
def pdf_to_images(pdf_path: Path, output_dir: Path) -> list[Path]:
    doc = fitz.open(str(pdf_path))
    images = []
    for i, page in enumerate(doc):
        pix = page.get_pixmap(dpi=200)
        out = output_dir / f"page_{i+1:03d}.png"
        pix.save(str(out))
        images.append(out)
    doc.close()
    return images
```

### Tesseract with OpenCV preprocessing

```python
import pytesseract
from PIL import Image

def ocr_image(image_path: Path) -> str:
    img = Image.open(str(image_path))

    # Optional preprocessing — graceful fallback if OpenCV missing
    try:
        import cv2, numpy as np
        img_cv = cv2.imread(str(image_path))
        gray = cv2.cvtColor(img_cv, cv2.COLOR_BGR2GRAY)
        denoised = cv2.medianBlur(gray, 3)
        binary = cv2.adaptiveThreshold(
            denoised, 255,
            cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv2.THRESH_BINARY,
            11, 2,
        )
        # Deskew only if noticeably skewed
        coords = np.column_stack(np.where(binary > 0))
        angle = cv2.minAreaRect(coords)[-1]
        if angle < -45:
            angle = 90 + angle
        if abs(angle) > 0.5:
            h, w = binary.shape[:2]
            M = cv2.getRotationMatrix2D((w//2, h//2), angle, 1.0)
            binary = cv2.warpAffine(
                binary, M, (w, h),
                flags=cv2.INTER_CUBIC,
                borderMode=cv2.BORDER_REPLICATE,
            )
        img = Image.fromarray(binary)
    except ImportError:
        pass

    return pytesseract.image_to_string(
        img, lang="deu", config="--psm 6 --dpi 300"
    ).strip()
```

### Per-page decision

```python
def get_page_text(image_path: Path, embedded_text: str | None) -> str:
    if embedded_text and len(embedded_text) > 80:
        return embedded_text             # cheaper, error-free
    return ocr_image(image_path)         # fallback
```

## Phase 2 — Batch classification with Gemma 3

### Structured output schema (Ollama format parameter)

```python
STRUCTURED_OUTPUT_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "seiten":                {"type": "array", "items": {"type": "integer"}},
            "absender":              {"type": "string"},
            "empfaenger":            {"type": ["string", "null"]},
            "brieftyp":              {"type": "string"},
            "forderung_eur":         {"type": ["number", "null"]},
            "iban":                  {"type": ["string", "null"]},
            "aktenzeichen":          {"type": ["string", "null"]},
            "frist":                 {"type": ["string", "null"]},
            "zusammenfassung":       {"type": ["string", "null"]},
            "vertreter":             {"type": ["string", "null"]},
            "vertreter_aktenzeichen":{"type": ["string", "null"]},
        },
        "required": ["seiten", "absender", "empfaenger", "brieftyp", "zusammenfassung"],
    },
}
```

`brieftyp` enum: `Mahnung | Mahnbescheid | Vollstreckungsbescheid | Rechnung | Bussgeld | Ratenzahlung | Vergleichsangebot | Kuendigung | Sonstiges`.

### Prompt (verbatim — do not paraphrase)

```text
Du analysierst gescannte deutsche Briefe. Jede Seite ist mit [SEITE X] markiert.
Zu jeder Seite gibt es [REGEX-ERGEBNISSE] — uebernimm diese wenn sie plausibel sind.

REGELN:
- Anderer Absender = NEUES Dokument (auch bei OCR-Muell am Seitenanfang)
- Anderer Empfaenger = NEUES Dokument
- "Seite 2" oder fehlender Briefkopf + gleicher Absender = Fortsetzung
- zusammenfassung beschreibt NUR den Inhalt der eigenen Seiten
- Verwende KEINE Daten anderer Dokumente (kein Betrag, kein AZ von Nachbarseiten)
- Bei Inkasso: absender = Inkasso-Firma, vertreter = Originalglaeubieger

Antwort NUR als JSON-Array:
[{{"seiten":[1,2],"absender":"Firmenname","empfaenger":"Name","brieftyp":"Mahnung","forderung_eur":1234.56,"aktenzeichen":"AZ","iban":"DE...","vertreter":"Inkasso GmbH","vertreter_aktenzeichen":"VT-AZ","frist":"2026-01-15","zusammenfassung":"Ein Satz"}}]

brieftyp: Mahnung|Mahnbescheid|Vollstreckungsbescheid|Rechnung|Bussgeld|Ratenzahlung|Sonstiges
WICHTIG Mahnbescheid vs. Vollstreckungsbescheid:
- Mahnbescheid: Ueberschrift "MAHNBESCHEID", Widerspruchsfrist 2 Wochen, "Gericht hat nicht geprueft"
- Vollstreckungsbescheid: Ueberschrift "VOLLSTRECKUNGSBESCHEID", Einspruchsfrist 2 Wochen
- Wenn ein Mahnbescheid den Vollstreckungsbescheid als ZUKUNFT erwaehnt ("...kann einen Vollstreckungsbescheid erwirken"), ist es TROTZDEM ein Mahnbescheid!
frist: Immer das konkrete Datum (YYYY-MM-DD) angeben. Bei "innerhalb von zwei Wochen" den Typ trotzdem korrekt setzen, frist=null.
Felder sind null wenn nicht im Text erkennbar.
JEDE Seite muss in genau einem Dokument vorkommen!

{pages_text}
```

The Mahnbescheid-vs-Vollstreckungsbescheid hint is **load-bearing**: in real
data, Mahnbescheide reference the Vollstreckungsbescheid as a future
consequence ("...andernfalls kann ein Vollstreckungsbescheid ergehen") and
naive keyword matching mis-classifies them.

### Calling Ollama

```python
import requests

def call_llm_batch(pages_text: str) -> list[dict]:
    body = {
        "model": "gemma3:12b",
        "messages": [{"role": "user", "content": pages_text}],
        "stream": False,
        "format": STRUCTURED_OUTPUT_SCHEMA,
        "keep_alive": "24h",        # avoid model reload between calls
        "options": {"num_ctx": 4096, "temperature": 0},
    }
    resp = requests.post("http://localhost:11434/api/chat", json=body, timeout=300)
    resp.raise_for_status()
    return parse_llm_json(resp.json()["message"]["content"])
```

`temperature=0` is mandatory — non-zero produces visibly worse JSON
adherence and inconsistent grouping. `keep_alive: 24h` keeps the model
loaded between invocations so subsequent batches respond in 1–5 s instead
of 30–60 s.

### JSON post-processing

Gemma sometimes emits German decimal format (`"3.757,59"`) where the schema
expects a number. Repair before parsing:

```python
import re

def parse_llm_json(content: str) -> list[dict] | None:
    arr_match = re.search(r'\[\s*\{.*?\}\s*\]', content, re.DOTALL)
    if not arr_match:
        return None
    raw = arr_match.group(0)
    raw = re.sub(
        r'"forderung_eur"\s*:\s*"([\d.]+),([\d]+)"',
        lambda m: f'"forderung_eur": {m.group(1).replace(".", "")}.{m.group(2)}',
        raw,
    )
    return json.loads(raw)
```

## Regex extractor (Phase 2 fallback + LLM hint)

The regex layer serves two purposes: it runs **before** the LLM call so the
prompt can include `[REGEX-ERGEBNISSE SEITE N]: Absender: X | IBAN: Y | …`
hints, and it is the sole path when Ollama is offline.

### IBAN

```python
_IBAN_RE = re.compile(
    r'\b([A-Z]{2}\d{2})[\s]?([\dA-Z]{4}[\s]?){3,7}[\dA-Z]{1,4}\b'
)
def extract_iban(text: str) -> str | None:
    m = _IBAN_RE.search(text)
    return re.sub(r'\s+', '', m.group(0)) if m else None
```

### Sender (Absender)

Letterhead in the first ~20 lines, recipient block detected by salutation
(`Herrn`, `Frau`, `Dr.`) or PLZ-line preceded by a name.

Three passes (in order — first hit wins):

1. **Court-document Antragsteller** — `Antragsteller:\s*\n?\s*(.+)` regex
   over the first 1500 chars. The Antragsteller is the real creditor; the
   Mahngericht itself is never the relevant sender.
2. **Legal-form suffix line** — match `GmbH | AG | SE | KGaA | KG | OHG | GbR | PartmbB | e.V. | Versicherung | Bank | Sparkasse | Rechtsanwälte | Inkasso | Services | Sozietät | Kanzlei | Kredit | Finanz | Energie | Werke | Verband | Zweckverband | Versorgungs`.
3. **Known-sender keyword** — match a curated list (`Finanzamt`,
   `Berufsgenossenschaft`, `BG BAU`, `Sparkasse`, `Postbank`, `Deutsche Bank`,
   `Barclays`, `BAWAG`, `Volksbank`, `Commerzbank`, `Stadtwerke`,
   `Zweckverband`, `Feuersozietät`, `Hörnlein`, `Riverty`, `Creditreform`,
   `SCHUFA`, `Inkasso`, `Rechtsanwalt`, `Kanzlei`, `AOK`, `Krankenkasse`,
   `Investitionsbank`, `Wüstenrot`, `Bausparkasse`, `Amtsgericht`,
   `Medizinischer Dienst`).
4. **Heuristic** — first non-noise multi-word line where digit ratio ≤ 30 %
   and special chars ≤ 2.

### Letter-type (brieftyp) — order matters

```python
_BRIEFTYP_PATTERNS = [
    (r"(?i)vergleichsangebot|vergleichsvorschlag",       "Vergleichsangebot"),
    (r"(?i)ratenzahlung|r[uü]ckf[uü]hrungsvereinbarung", "Ratenzahlung"),
    (r"(?i)rechnung",                                    "Rechnung"),
    (r"(?i)k[uü]ndigung",                                "Kuendigung"),
    (r"(?i)\bmahnung\b",                                 "Mahnung"),
    (r"(?i)bu[sß]geldbescheid",                          "Bussgeld"),
]
```

For Mahnbescheid vs. Vollstreckungsbescheid, look for the word as a
**header on its own line** (with optional letter-spacing, e.g.
`M A H N B E S C H E I D`), not embedded in body text.

### Continuation page detection

A page is a continuation (groups with the previous document) when:

- It contains `Seite [2-9]` or `– 2 –` markers (but not `Seite 1`)
- It contains court continuation phrases: `zugestellten Mahnbescheid`,
  `macht folgenden Anspruch geltend`, `Kosten des Mahnverfahrens`,
  `Anspruch des Antragstellers`
- It contains `Seite X von Y` where X > 1 AND no address block
- It has neither a standalone date (DD.MM.YYYY not preceded by
  `am | vom | zugestellt`) nor an address block (`Sehr geehrte`, `Herrn `,
  `Frau `, `<5-digit-PLZ> <word>`) in the top 500 chars

## Service health checks

```python
def check_tesseract() -> tuple[bool, str]:
    path = shutil.which("tesseract")
    return (True, f"found at {path}") if path else (False, "Tesseract not installed")

def check_ollama() -> tuple[bool, str]:
    try:
        r = requests.get("http://localhost:11434/api/tags", timeout=3)
        return (True, "reachable") if r.status_code == 200 else (False, f"status {r.status_code}")
    except Exception:
        return (False, "unreachable — falling back to regex")
```

Run both at process startup; surface the result in the agent's status
output so a human knows whether they got the full or degraded pipeline.

## Determinism

- Tesseract is deterministic for a given binary version + image bytes.
- PDF embedded text is deterministic.
- Gemma 3 with `temperature=0` is deterministic on a given Ollama version
  when the model file hash is unchanged. Ollama updates can shift outputs;
  pin the model in production with `ollama pull gemma3:12b` and record the
  digest from `ollama show gemma3:12b --modelfile | head -3`.

## Hard rules — what an agent must not do

- ❌ Skip Phase 1 PDF embedded-text and OCR every page anyway. Embedded
  text is faster, free of OCR artifacts and contains correct casing and
  punctuation. OCR every page only if you have evidence it was a scan.
- ❌ Send page images directly to a hosted vision model. The pipeline is
  local on purpose; cloud OCR for these documents is a privacy violation.
- ❌ Use `temperature > 0` on Gemma. Inconsistent JSON, inconsistent
  document grouping, breaks downstream tests.
- ❌ Trust regex extraction for `forderung_eur` when the LLM is available.
  Regex catches "EUR 1.234,56" but mis-parses "Saldo: 1.234,56 EUR zu Ihren
  Gunsten" as a debt — the LLM disambiguates.
- ❌ Drop the `[REGEX-ERGEBNISSE]` hint block from the prompt to "save
  tokens". The regex hints stabilize Gemma's output noticeably for
  ambiguous letterheads.
- ❌ Auto-act on extracted deadlines/amounts without a human-in-the-loop
  step. OCR errors on a single digit (`14.01.` vs `19.01.`) change a
  deadline by days; never schedule reminders or trigger filings without
  explicit confirmation.
