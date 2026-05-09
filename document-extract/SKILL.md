---
name: document-extract
description: Extract structured data + searchable text from incoming PDFs (letters, court documents, invoices, mailings). The agent uses its own Read tool — no external API call. Original PDFs stay frozen as evidence; extract.md is a derivative for downstream agents. Use whenever a PDF arrives in _eingang/ or in an Akte's eingehend/ folder. NEVER overwrite the original PDF or omit the SHA-256 of the source.
---

# Document Extract

Du (der Agent) liest die PDF **selbst** mit deinem Read-Tool. Du bist
Claude — du brauchst keine zusätzliche API. Lies, analysiere, schreibe
ein `extract.md`. Original-PDF bleibt unangetastet.

## When to Use

- Eine neue PDF liegt in `butler-vault/_eingang/` und braucht Klassifikation
- Eine PDF wandert in `03_AKTEN/<owner>/<akte>/eingehend/` und braucht
  ein `extract.md`
- Re-Extraktion einer bestehenden PDF (neues Schema) — Original bleibt
  frozen, nur `extract.md` wird neu geschrieben

## When NOT to Use

- Reine Text-PDFs ohne Struktur-Bedarf — `pdftotext` reicht
- Nicht-PDF-Formate (`.docx` → docx-edit, Bilder → image-editing)
- Beim Massen-Trash (Werbung) — Header-Klassifikation aus Mail-Subject
  reicht; PDF gar nicht öffnen

## Hard rules

1. **Original-PDF ist die Wahrheit.** Niemals modifizieren, neu kodieren
   oder löschen. SHA-256 einmal beim Import berechnen und festhalten.
2. **Jedes extract.md zitiert seine Quelle.** YAML-Frontmatter MUSS
   `source_pdf` (relativer Pfad) und `source_sha256` enthalten.
3. **Niemals halluzinieren.** Wenn das PDF unleserlich, teilweise
   gescannt oder mehrdeutig ist: `[UNKLAR]` schreiben, nicht raten.
   Issue an Sachbearbeiter mit Notiz.
4. **Pro PDF: ein Lesedurchgang.** Nicht in einer Schleife "verbessern"
   — wenn das erste Extract falsch ist, Prompt-Schema fixen, nicht
   nochmal lesen.
5. **Bilder ≤ 2000 px** wenn separat (nicht im PDF) — siehe
   ~/.claude/CLAUDE.md.
6. **Kein externer Anthropic-API-Call**, kein `from anthropic import …`,
   kein API-Key. Du bist Claude, du liest mit Read direkt.

## Modellwahl

Du bist Claude in einem Bot mit fester Modell-Zuordnung
(`adapter_config.model`). Das Modell wird PRO BOT konfiguriert, nicht
pro PDF. Routing-Hinweise:

| Bot-Modell | Wofür typisch |
|-----------|---------------|
| Haiku 4.5 | Posteingangs-Bot (Massentriage, schnelle Klassifikation) |
| Sonnet 4.6 | Sachbearbeiter (Briefe schreiben), Buchhalter |
| Opus 4.7 | Jurist (juristische Argumentation), Intelligence (Strategie) |

Wenn Posteingangs-Bot bei einem PDF unsicher wird (Klassifikation
`unsicher: true`), erstellt er ein Issue an Sachbearbeiter — der hat
Sonnet, kann tiefer gucken.

## Workflow (für jeden PDF in `_eingang/` oder `eingehend/`)

1. **Lies die PDF mit deinem Read-Tool.** Beispiel:
   `Read("/paperclip/data/butler/vault/_eingang/Sammelmappe50.pdf")`
   — das Tool liefert dir die Seiten als Text + Bilder.

2. **Berechne SHA-256:**
   `sha256sum <pdf_path>` via Bash, oder Python-Snippet.

3. **Klassifiziere + extrahiere die Felder** (siehe Schema unten).

4. **Schreibe `extract.md`** neben das Original (gleiches Verzeichnis,
   `<basename>_extract.md`).

5. **Route die PDF** entsprechend `dokumenttyp` (siehe Routing-Tabelle).

6. **Update `_PROVENANCE.md`** der Akte (neuer Eintrag mit Datum + SHA).

7. **Bei `kritisch: true`**: Issue mit Label `kritisch` an Jurist,
   Mail an David + Ronny, Eintrag in `04_KALENDER/deadlines/`.

## Extract-Schema

Du schreibst `extract.md` mit diesem YAML-Frontmatter (Pflichtfelder):

```yaml
---
source_pdf: <basename des Originals>
source_sha256: <SHA256-Hex>
extracted_at: <ISO-Timestamp UTC>
extracted_by: <dein Bot-Name>, model <dein Modell>
absender: "<Name + Anschrift, einzeilig oder als Block>"
empfaenger: "<wer ist Empfänger laut PDF>"
datum: <ISO YYYY-MM-DD oder null>
aktenzeichen: "<string oder null>"
dokumenttyp: <mahnung | inkasso | gerichtsbescheid | rechnung | vollstreckung | brief_avis | werbung | routine | sonstiges>
hauptforderung: <Zahl in EUR oder null>
nebenforderungen: [<Zahlen oder leer>]
frist: <ISO YYYY-MM-DD oder null>
frist_typ: <zahlung | widerspruch | einspruch | sonstiges | null>
kritisch: <true wenn dokumenttyp ∈ {gerichtsbescheid, vollstreckung}>
unsicher: <true wenn Klassifikation unklar — dann Issue an Sachbearbeiter>
---

# <kurzer Titel: Dokumenttyp + Absender + Datum>

## Volltext

[Kompletter Brieftext als Markdown — Briefkopf, Body, Unterschrift.
 Tabellen als Markdown-Tabellen. Erfundenes ist VERBOTEN. Bei
 unleserlichen Stellen: `[UNKLAR: Beschreibung was zu sehen ist]`.]

## Besonderheiten

- <z.B. "rote Markierung auf Seite 1">
- <z.B. "unleserliche Stelle Zeile 23">
- <leer wenn nichts auffällig>
```

## Klassifikation → Vault-Pfad

| dokumenttyp | Pfad | frozen? |
|-------------|------|---------|
| werbung | `99_ARCHIV/spam/YYYY/MM/` | nein, 30 Tage TTL |
| routine | `99_ARCHIV/routine/YYYY/MM/` | ja |
| brief_avis | `05_KORRESPONDENZ/eingehend/YYYY/MM/` (Hinweis: physischer Brief folgt) | ja |
| mahnung / inkasso / rechnung | `03_AKTEN/<owner>/<akte-id>/eingehend/` | ja |
| gerichtsbescheid / vollstreckung | wie oben + `kritisch: true` + Eintrag in `04_KALENDER/deadlines/` | ja |
| sonstiges | `05_KORRESPONDENZ/eingehend/YYYY/MM/` mit Issue an Sachbearbeiter | ja |

## Failure modes

- **Verschlüsseltes PDF**: Read-Tool meldet das. → Issue an Principal
  (David / Ronny) mit Bitte um Pass. `extract.md` mit `[VERSCHLÜSSELT]`.
- **Korruptes PDF**: Read-Tool wirft Fehler. → `extract.md` mit
  `[KORRUPT]`-Marker, Issue an Sachbearbeiter zur Sichtung.
- **Sammelmappe (mehrere Briefe in einem PDF)**: siehe nächster Abschnitt.
- **Sehr große PDFs (>50 MB / 100 Seiten)**: Read-Tool hat Limits.
  Vorab `pdfinfo <pdf>` (Bash) checken; bei >100 Seiten: erste 30
  Seiten extrahieren + Hinweis "Volltext gekürzt — siehe Original",
  Issue an Sachbearbeiter zur Vollsichtung.

## Sammelmappen — mehrere Briefe in einem PDF

In der Praxis kommen oft "Sammelmappen" rein — eingescannte Stapel mit
mehreren unabhängigen Briefen in einem PDF, in beliebiger Reihenfolge.

### Erkennen

Du liest das PDF einmal komplett. Wenn du mehrere Briefkopfe / mehrere
Absender / mehrere Datums findest, ist es eine Sammelmappe.

### Zerlegen

```bash
# Seitenbereich extrahieren mit pdftk (im Container vorhanden):
pdftk Sammelmappe50.pdf cat 1-3 output brief_01_seiten_1-3.pdf
pdftk Sammelmappe50.pdf cat 4-7 output brief_02_seiten_4-7.pdf
# usw.
```

Danach für jedes Teil-PDF einzelne `extract.md` schreiben + an den
richtigen Akten-Pfad kopieren.

**Original-Sammelmappe NICHT löschen** — sie bleibt in
`_eingang/_processed/<datum>/` als Provenance.

## _eingang/ — Drop-Off-Poll

Der Posteingangs-Bot pollt `butler-vault/_eingang/` jeden Heartbeat:

```python
# Pseudo-Code für deine Bash/Read-Logik:
for pdf in list_new_pdfs("butler-vault/_eingang/"):
    if is_sammelmappe(pdf):  # → Indikatoren: mehrere Briefkopfe beim ersten Lesen
        teile = split_sammelmappe(pdf)  # pdftk
        for teil in teile:
            process_brief(teil)
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
3. **Sofort** Mail-Draft an David UND Ronny mit:
   - Quelle (Gericht / Aktenzeichen)
   - Frist (mit Datum)
   - Akten-Link
   - "Bitte bestätigen Sie den Empfang"
4. **Sofort** Issue mit Label `kritisch:true` + `typ:auftrag`, Assignee Jurist
5. **Sofort** Eintrag in `04_KALENDER/deadlines/` mit Frist-Datum

## Integration mit Akten

Nach Extraktion und Klassifikation:

1. PDF kopieren (NICHT verschieben!) zum richtigen Akten-Pfad
2. `extract.md` daneben schreiben
3. `_PROVENANCE.md` der Akte aktualisieren (neuer Eintrag mit Datum + SHA)
4. Wenn neue Akte nötig: Sachbearbeiter erstellt `_AKTE.md` mit
   Stammdaten aus dem Extract
5. Wenn Frist im Extract: Eintrag in `04_KALENDER/deadlines/` mit
   Akten-Verweis
6. Bei Gerichtsdokument: zusätzlich Schritt 3–5 aus „Aufbewahrungsregeln"

## Migration aus alter Skill-Version

Frühere Version dieses Skills nutzte `from anthropic import Anthropic`
und brauchte einen `ANTHROPIC_API_KEY`. **Das ist überholt.** Bots in
Paperclip laufen als `claude_local`-Adapter — der Bot IST Claude und
liest PDFs direkt mit seinem eigenen Read-Tool. Kein API-Key, kein
SDK-Import, keine zusätzlichen Kosten neben dem Heartbeat selbst.
