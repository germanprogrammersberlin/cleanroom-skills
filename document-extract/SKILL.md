---
name: document-extract
description: Extract structured data + searchable text from incoming PDFs (letters, court documents, invoices, mailings) using Claude's native document API. Original PDFs stay frozen as evidence; extract.md is a derivative for the agent. Use this whenever a PDF arrives in the inbox or in an Akte's eingehend/ folder. NEVER overwrite the original PDF or omit the SHA-256 of the source.
---

# Document Extract

Use Claude's document API to extract structured information and full text from PDF documents (German letters, official documents, invoices, court papers). No external OCR tools needed — Claude handles scanned PDFs and text PDFs equally.

## When to Use

- A new PDF arrives in `05_KORRESPONDENZ/eingehend/` and needs classification
- A PDF is moved into an Akte's `eingehend/` folder and needs an `extract.md` for the agent to reason about
- Re-extraction of an existing PDF (e.g., new prompt, better classification schema) — original PDF stays frozen, only the `extract.md` changes

## When NOT to Use

- For mass ingestion of thousands of unrelated PDFs (cost) — there use a cheaper local pipeline first and only call Claude on uncertain cases
- For purely text-based PDFs where structure doesn't matter (use `pdftotext` if available)
- For non-PDF formats (use the appropriate reader; .docx → docx-edit, images → image-editing)

## Hard rules

1. **The original PDF is the truth.** Never modify, re-encode, or delete it. Compute SHA-256 once at first import and pin it.
2. **Every extract.md cites its source.** YAML frontmatter MUST contain `source_pdf` (relative path) and `source_sha256`.
3. **Never invent facts.** If the PDF is unreadable, partially scanned, or ambiguous, write `[UNKLAR]` in the extract instead of guessing. Log the issue in the agent's notes.
4. **Per-PDF: one Claude call.** Don't loop "improve the extract" — burn cost. If the first extract is wrong, fix the prompt, not by repeated re-runs.
5. **PDFs go through Claude (document API), never through local OCR.** This is a binding architecture rule, not just a recommendation.
6. **Image attachments must be ≤ 2000 px in any dimension** before sending to Claude. Above that, the API downsamples (quality suffers) or rejects. Resize first with `sips -Z 1500` (macOS), `Pillow.thumbnail((1800, 1800))` (Python), or `convert -resize 1800x1800` (ImageMagick). PDF documents sent as `document` type are handled by the API directly and are exempt.

## Modellwahl

| Modell | Wofür | Wann eskalieren? |
|--------|-------|------------------|
| **Haiku 4.5** | Routine, Newsletter, Bestellbestätigungen, Werbung, einfache Klassifikation | bei niedriger Confidence → Sonnet |
| **Sonnet 4.6** (DEFAULT) | Standardbriefe: Mahnungen, Rechnungen, Verträge, Behörden | bei `dokumenttyp ∈ {gerichtsbescheid, vollstreckung}` → Opus |
| **Opus 4.7** | Gerichtsbescheide, Vollstreckungsbescheide, juristisch komplexe Akten | — |

Posteingangs-Bot startet typischerweise mit Sonnet, schaltet auf Opus
hoch wenn der erste Extract `kritisch: true` setzt. Werbung/Routine
darf direkt mit Haiku klassifiziert werden.

## Setup (every agent that does extraction)

```python
import os, base64, hashlib, json
from anthropic import Anthropic

client = Anthropic()  # picks up ANTHROPIC_API_KEY or PAPERCLIP_API_KEY
MODEL = "claude-sonnet-4-6"  # or claude-haiku-4-5 for low-priority routine
```

## Standard extraction call

```python
def extract(pdf_path: str, classification_hint: str = "auto") -> dict:
    pdf_bytes = open(pdf_path, "rb").read()
    sha = hashlib.sha256(pdf_bytes).hexdigest()
    pdf_b64 = base64.b64encode(pdf_bytes).decode()

    prompt = (
        "Extrahiere aus diesem Dokument strukturiert die folgenden Felder "
        "als JSON. Wenn ein Feld nicht im Dokument steht, schreibe null. "
        "Erfinde nichts.\n\n"
        "Felder:\n"
        "- absender (Name + Anschrift)\n"
        "- empfaenger\n"
        "- datum (ISO YYYY-MM-DD)\n"
        "- aktenzeichen (string oder null)\n"
        "- dokumenttyp (mahnung | inkasso | gerichtsbescheid | rechnung | "
        "  vollstreckung | brief_avis | werbung | routine | sonstiges)\n"
        "- hauptforderung (Zahl in EUR oder null)\n"
        "- nebenforderungen (Liste oder [])\n"
        "- frist (ISO-Datum oder null)\n"
        "- frist_typ (zahlung | widerspruch | einspruch | sonstiges | null)\n"
        "- volltext (kompletter Text als Markdown — Briefkopf, Body, "
        "  Unterschrift, alles)\n"
        "- besonderheiten (Liste auffälliger Beobachtungen, z.B. "
        "  'rote Markierung auf Seite 1', 'unleserliche Stelle Zeile 23')\n\n"
        "Antworte AUSSCHLIESSLICH mit dem JSON-Objekt, kein Kommentar davor "
        "oder danach."
    )

    msg = client.messages.create(
        model=MODEL,
        max_tokens=8000,
        messages=[{
            "role": "user",
            "content": [
                {"type": "document", "source": {
                    "type": "base64",
                    "media_type": "application/pdf",
                    "data": pdf_b64,
                }},
                {"type": "text", "text": prompt},
            ],
        }],
    )
    raw = msg.content[0].text.strip()
    # robust JSON parse — Claude sometimes wraps in ```json
    if raw.startswith("```"):
        raw = raw.split("```")[1].lstrip("json").strip()
    data = json.loads(raw)
    data["_source_pdf"] = os.path.basename(pdf_path)
    data["_source_sha256"] = sha
    data["_model"] = MODEL
    return data
```

## Output: extract.md format

For a file `eingehend/20241012_riverty_mahnstufe1.pdf` write
`eingehend/20241012_riverty_mahnstufe1_extract.md`:

```markdown
---
source_pdf: 20241012_riverty_mahnstufe1.pdf
source_sha256: abc123…
extracted_at: 2026-05-09T17:42:00Z
extracted_by: claude-sonnet-4-6 via document-extract skill v1
absender: "Riverty GmbH, Postfach 123, 80339 München"
datum: 2024-10-12
aktenzeichen: "RV-2024-04567"
dokumenttyp: mahnung
hauptforderung: 1234.56
frist: 2024-10-26
frist_typ: zahlung
---

# Mahnung Riverty — 12.10.2024

## Volltext

[der gesamte Brieftext, formatiert als Markdown]

## Besonderheiten

- Zahlart: SEPA-Lastschrift mit IBAN DE…
- Druckdatum scheint manuell überstempelt
```

## Klassifikation → Vault-Pfad

After extraction, the bot routes the PDF based on `dokumenttyp`:

| dokumenttyp | Pfad | frozen? |
|-------------|------|---------|
| werbung | `99_ARCHIV/spam/YYYY/MM/` | nein, 30 Tage TTL |
| routine | `99_ARCHIV/routine/YYYY/MM/` | ja |
| brief_avis | `05_KORRESPONDENZ/eingehend/YYYY/MM/` (mit Hinweis: physischer Brief folgt) | ja |
| mahnung / inkasso / rechnung | `03_AKTEN/<owner>/<akte-id>/eingehend/` | ja |
| gerichtsbescheid / vollstreckung | wie oben + Marker `kritisch: true` + Eintrag in `04_KALENDER/deadlines/` | ja |
| sonstiges | `05_KORRESPONDENZ/eingehend/YYYY/MM/` mit Issue an Sachbearbeiter zur Sichtung | ja |

## Failure modes

- **Verschlüsseltes PDF**: Claude meldet das. → Schreibe Issue an Principal mit Bitte um Pass.
- **Leeres / korruptes PDF**: extract liefert null/leeren Volltext. → `extract.md` mit `[KORRUPT]`-Marker, Issue an Principal.
- **Mehrere Briefe in einem PDF (Sammelmappe)**: Claude erkennt das im Volltext. → Vor dem Speichern: Trennung per Heuristik (Seitenwechsel + neuer Briefkopf), pro Brief eigener Akten-Eintrag.
- **Sehr große PDFs (>50 MB / 100 Seiten)**: Claude API hat Limits. → Vorher mit `pdfinfo` checken, ggf. nur erste 30 Seiten extrahieren + Hinweis.

## Cost awareness

- Sonnet: ~3000–8000 Input-Tokens pro PDF (je nach Seitenzahl), ~500–2000 Output
- Haiku: ~10× billiger, aber schwächere Extraktion bei handschriftlichem Text
- **Faustregel:** Routine-Mails (Bestellbestätigungen, Newsletter) → Haiku. Forderungen/Behörden/Gericht → Sonnet. Kritisches → Opus.

## Sammelmappen — mehrere Briefe in einem PDF

In der Praxis kommen oft "Sammelmappen" rein — eingescannte
Stapel mit mehreren unabhängigen Briefen in einem PDF, in beliebiger
Reihenfolge. Der Skill MUSS diese auseinandersortieren.

### Splitting via Claude

```python
def split_sammelmappe(pdf_path: str) -> list[dict]:
    """Erkennt Brief-Grenzen in einer Sammelmappe und liefert Index zurück."""
    pdf_b64 = base64.b64encode(open(pdf_path, "rb").read()).decode()

    prompt = (
        "Dieses PDF enthält mehrere unabhängige Briefe. Identifiziere die "
        "Brief-Grenzen anhand von neuem Briefkopf, Absender-Wechsel, "
        "Datum-Wechsel oder neuem Aktenzeichen.\n\n"
        "Liefere ein JSON-Array zurück, ein Element pro Brief:\n"
        "  {start_page: int (1-indexiert), end_page: int, "
        "   absender_kurz: str, datum: ISO-Datum, dokumenttyp: str}\n"
        "Kein Kommentar drumherum, nur das Array."
    )

    msg = client.messages.create(
        model="claude-sonnet-4-6",  # Sonnet ist robust für Splitting
        max_tokens=4000,
        messages=[{
            "role": "user",
            "content": [
                {"type": "document", "source": {
                    "type": "base64",
                    "media_type": "application/pdf",
                    "data": pdf_b64,
                }},
                {"type": "text", "text": prompt},
            ],
        }],
    )
    return json.loads(strip_codeblock(msg.content[0].text))
```

Danach pro Brief: PDF mit `pypdf` o.ä. zerlegen, jedes Teil-PDF separat
durch `extract()` schicken, an den richtigen Akten-Pfad routen.

## _eingang/ — Drop-Off-Poll

Der Posteingangs-Bot pollt `butler-vault/_eingang/` alle 5 Minuten:

```python
new_files = list_new_pdfs("butler-vault/_eingang/")
for pdf in new_files:
    if is_sammelmappe(pdf):
        for brief in split_sammelmappe(pdf):
            process_brief(pdf, page_range=(brief["start_page"], brief["end_page"]))
    else:
        process_brief(pdf)

    # Original behalten, NICHT löschen
    move(pdf, f"butler-vault/_eingang/_processed/{today}/{pdf.name}")
```

## Aufbewahrungsregeln — UNVERHANDELBAR für kritische Dokumente

**Gerichtsdokumente und Vollstreckungsbescheide werden NIEMALS
gelöscht.** Auch nicht aus `_processed/`. Auch nicht nach 30 Jahren.

| Klasse | _processed/ Cleanup | Akten-Original |
|--------|---------------------|----------------|
| Werbung | nach 7 Tagen weg | Header-only, dann weg |
| Routine | nach 30 Tagen weg | 1 Jahr, dann Archiv |
| Mahnung / Inkasso / Rechnung | nach 30 Tagen weg | dauerhaft (Akten-Original frozen) |
| **Gerichtsbescheid / Vollstreckung** | **NIEMALS aus _processed/ löschen** | dauerhaft + redundant gesichert |

Bei Klassifikation `gerichtsbescheid` oder `vollstreckung`:

1. Original in Akte (`03_AKTEN/<owner>/<akte>/eingehend/`) — frozen, mit SHA
2. Kopie bleibt in `_eingang/_processed/_kritisch/` — niemals löschen
3. **Sofort** Mail an David UND Ronny mit:
   - Quelle (Gericht / Aktenzeichen)
   - Frist (mit Datum)
   - Akten-Link
   - "Bitte bestätigen Sie den Empfang"
4. **Sofort** Issue mit Label `kritisch:true` + `typ:auftrag`, Assignee Jurist
5. **Sofort** Eintrag in `04_KALENDER/deadlines/` mit Frist-Datum

## Integration mit Akten

Nach Extraktion und Klassifikation:

1. PDF kopieren (nicht verschieben!) zum richtigen Akten-Pfad
2. extract.md daneben schreiben
3. `_PROVENANCE.md` der Akte aktualisieren (neuer Eintrag mit Datum + SHA)
4. Wenn neue Akte nötig: Sachbearbeiter erstellt `_AKTE.md` mit Stammdaten aus extract
5. Wenn Frist im extract: Eintrag in `04_KALENDER/deadlines/` mit Akten-Verweis
6. Bei Gerichtsdokument: zusätzlich Schritt 3-5 aus „Aufbewahrungsregeln"
