---
name: german-sbp-builder
description: Generate a German "Schuldenbereinigungsplan" (out-of-court debt-settlement plan per § 305 InsO) — deterministic calculation of the garnishable amount per § 850c ZPO and pixel-fixed PDF in the Schewtschenko (RA Schuldnerberatung) layout. Use when an agent needs to draft a settlement offer to all creditors of a debtor, distributing the garnishable monthly income proportionally. Do NOT use for individual creditor negotiations (those are normal letters), tax-debt-only cases (different procedure), or non-German jurisdictions.
---

# German Schuldenbereinigungsplan (SBP) Builder

> **Stability:** This skill is a frozen specification. The garnishment table,
> calculation formulas and PDF layout are deterministic and **must not** be
> changed by an agent. When the law changes (e.g. new garnishment table from
> 01.07.2026), this skill file is updated — not the agent's behavior.
>
> Layout reference: "RA Schuldnerberatung Schewtschenko, 11.09.2025".
> Code reference: `app/backend/db/sbp.py`, `app/backend/sbp_pdf.py`.

## Purpose

A Schuldenbereinigungsplan is the standardized offer a debtor makes to **all
of their creditors at once** in the out-of-court phase of German consumer
insolvency proceedings (§ 305 InsO). It lists, per creditor, the claim
amount, the proportional share, the settlement quota and the payment
schedule.

The plan is built around the **garnishable income** under § 850c ZPO and
distributes that amount **proportionally** across all non-excluded creditors.

## When to Use

- Drafting an out-of-court settlement offer covering **all** creditors of a
  given debtor in one document
- Re-running the calculation when the debtor's net income or maintenance
  obligations change (forces recalculation of all rates)
- Generating the formal PDF the debtor's counsel files with the insolvency
  court if creditors reject the plan

## When NOT to Use

- Individual creditor negotiations (use a normal `Vergleichsvorschlag` letter
  per creditor instead)
- Cases where the only creditor is the tax office (different procedure —
  Stundung/Erlass, not an SBP)
- Non-German jurisdictions
- "Erlass-Antrag" (debt waiver application) — that requires a quota < 100 %
  with explicit waiver requests, which is **not** what this skill produces

## Inputs

| Field | Type | Required | Note |
|-------|------|----------|------|
| `schuldner_name` | str | ✅ | Full name, e.g. "Dr. David Kaiser" |
| `mdnr` | str | ✅ | Counsel/case number ("Mandantennummer") |
| `nettoeinkommen` | float | ✅ | EUR/month, after tax + social contributions |
| `unterhaltspflichtige` | int | ✅ | Number of dependants (0–5+) |
| `laufzeit_monate` | int | ✅ | 72 by default. `0` = open-ended until 100 % paid |
| `start_datum` | str | ✅ | Format `DD.MM.YYYY`, first installment |
| `vergleichsquote_prozent` | float | ✅ | **always 100.00** (full repayment over duration) |
| `glaeubiger[]` | list | ✅ | see below |

### Per creditor (`glaeubiger[]`)

| Field | Type | Required | Note |
|-------|------|----------|------|
| `nr` | int | ✅ | Sequential creditor number (1, 2, 3, …) |
| `name` | str | ✅ | Company name |
| `adresse` | list[str] | ✅ | Multi-line (street, ZIP city) |
| `vertreter` | str \| null | ⛔ optional | Collection agency / lawyer |
| `vt_adr` | list[str] \| null | ⛔ | Representative's address |
| `az_gl` | str | ⚠️ if known | Creditor's case number |
| `az_vt` | str | ⚠️ if known | Representative's case number |
| `hf` | float | ✅ | Principal claim ("Hauptforderung") |
| `zi` | float | ✅ | Interest ("Zinsen") — may be 0 |
| `ko` | float | ✅ | Costs ("Kosten") — may be 0 |
| `gf` | float | ✅ | Total claim = hf + zi + ko |
| `zw` | str | ✅ | "monatlich" \| "vierteljährlich" \| "halbjährlich" \| "jährlich" |

Creditors flagged `sbp_ausgeschlossen = 1` (e.g. surety claims, tort claims,
family debts) are **not** included in the plan.

## Step 1 — Garnishable amount per § 850c ZPO

**Table valid from 01.07.2025 to 30.06.2026.** Values are EUR.
Format: `dependants → (table_start, first_step, increment_per_10_EUR)`.

```
0 dependants  → (1560, 3.50, 7.00)
1 dependant   → (2150, 2.50, 5.00)
2 dependants  → (2470, 2.00, 4.00)
3 dependants  → (2800, 1.50, 3.00)
4 dependants  → (3120, 1.00, 2.00)
5+ dependants → (3450, 0.50, 1.00)
```

**Upper bound:** 4,767.00 EUR. Anything above is fully garnishable.

### Calculation

```
key            = min(unterhaltspflichtige, 5)
(start, s1, i) = TABLE[key]

if einkommen < start:
    pfaendbar = 0

else:
    bracket   = (einkommen // 10) * 10        # round down to 10 EUR
    steps     = (bracket - start) // 10
    pfaendbar = s1 + steps * i

    if einkommen > 4767.00:
        # close out the table first
        end_bracket = (4767 // 10) * 10        # = 4760
        end_steps   = (end_bracket - start) // 10
        pfaendbar_table = s1 + end_steps * i
        pfaendbar       = pfaendbar_table + (einkommen - 4767.00)

pfaendbar = round(pfaendbar, 2)
```

### Verification examples

| Net income | Dependants | Garnishable |
|-----------|-----------|-------------|
| 1,500 € | 0 | **0.00 €** (below table start) |
| 2,000 € | 0 | **311.50 €** |
| 3,707.72 € | 0 | **1,501.50 €** ← reference value from code-doc |
| 5,000 € | 0 | **3,972.90 €** (table cap + overflow) |
| 2,500 € | 2 | **12.00 €** |

If your implementation produces different numbers, the implementation is
wrong — the table is fixed.

## Step 2 — Proportional distribution per creditor

```
gesamtschuld = SUM(g.gf for g in glaeubiger if not g.sbp_ausgeschlossen)

for each creditor g:
    anteil_prozent  = round((g.gf / gesamtschuld) * 100, 2)
    rate_monatlich  = round((g.gf / gesamtschuld) * pfaendbar, 2)
    vergleichsbetrag= round(rate_monatlich * laufzeit_monate, 2)
                       (if laufzeit=0 → vergleichsbetrag = g.gf, quota stays 100 %)
    vergleichsquote = 100.00                     # ALWAYS 100 %
```

### Payment-frequency mapping (display)

| `zw` | Number of payments | Per-payment amount |
|------|-------------------|-------------------|
| monatlich | `laufzeit_monate` | `rate_monatlich` |
| vierteljährlich | `laufzeit_monate / 3` | `vergleichsbetrag / count` |
| halbjährlich | `laufzeit_monate / 6` | `vergleichsbetrag / count` |
| jährlich | `laufzeit_monate / 12` | `vergleichsbetrag / count` |

## Step 3 — Generate the PDF

### Format

- **DIN A4 landscape** — 297 × 210 mm
- **Margin:** 12 mm all sides
- **Font:** Helvetica (Helvetica, Helvetica-Bold, Helvetica-BoldOblique)
- **Per-creditor block height:** 52 mm
- **Per page:** ~2 blocks on page 1 (with header), 3 blocks on follow-up pages

### Page 1 header (once)

```
[centered, 14pt bold] GLÄUBIGER-, FORDERUNGSAUFSTELLUNG und SCHULDENBEREINIGUNGSPLAN

[6pt small, left]    Hinweise: 1. Fehlende Gläubigernummern betreffen
                     ehemalige Gläubiger, die keine Forderungen gegen den
                     Schuldner mehr haben.
                     2. Die jeweils letzte Rate kann vom Zahlungsplan
                     abweichen, da eventuell nur noch eine Ausgleichszahlung
                     zum Vergleichsbetrag getätigt wird.

[7pt right]          MDNR: <mdnr>
```

### Running header (every page)

```
[9pt bold]  Für: <schuldner_name>          (BoldOblique italic)
[8pt]       Summe der Verbindlichkeiten:  <gesamtschuld €>     [10pt bold right]
[8pt bold]  Gläubigeranzahl: <n>                               [11pt bold right]
```

### Per-creditor block (52 mm tall, full width)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Gläubiger: <nr>   ☐ Unerlaubte Handlung   ☐ Nahestehende Person              │
│                                                                              │
│ Hauptforderung:        Zinsen:        Kosten:           Zahlungsplan         │
│              <hf €>          <zi €>          <ko €>                          │
│              0,00 €          0,00 €          0,00 €                          │
│              0,00 €          0,00 €          0,00 €                          │
│                                                                              │
│ <Name>                  Gesamtforderung:        <gf €>   Betrag/Rate Anzahl  │
│ <Address line 1>        Anteil der Forderung:  <%>       Zahlweise           │
│ <Address line 2>        Vergleichsquote:    100,00 %     Zahlung am/vom bis  │
│ Vertreter:              Vergleichsbetrag:      <vb €>    <rate> <count> ...  │
│ <Representative>                                                             │
│ <Repr. address>                                                              │
│                                                                              │
│ Aktenz. des Gläubigers:           <az_gl>                                    │
│ Aktenz. des Gläubigervertreters:  <az_vt>                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Mandatory invariants:**

- `Hauptforderung/Zinsen/Kosten` are shown **three times** below each other
  (line 1 = current value, lines 2 & 3 = `0,00 €` placeholder for later
  amendments — Schewtschenko layout requirement, **do not change**).
- `Vergleichsquote` is **always** `100,00 %`.
- The two checkboxes top-left (Unerlaubte Handlung / Nahestehende Person)
  are **always empty** — counsel checks them by hand if applicable.

### Footer (every page)

```
[6.5pt left]   <Weekday, DD. Month YYYY>   (German)
[6.5pt center] Vorder- und Rückseite beachten
[6.5pt right]  SEITE <n> VON <total>
```

### Number format (mandatory)

EUR amounts: German format, period as thousands separator, comma as decimal
separator, suffix `" €"` with non-breaking space.

```
1234.5  →  "1.234,50 €"
0       →  "0,00 €"
80115.75→  "80.115,75 €"
```

US/UK format (`1,234.50`) is **rejected** by insolvency courts.

## Data sources (BriefButler database)

| Plan field | Source |
|------------|--------|
| `schuldner_name` | `schuldner.label` (fallback `name`) |
| `mdnr` | `schuldner.aktenzeichen` |
| `nettoeinkommen` | API parameter (UI-entered; persisted as `vergleichsvorschlag.einkommen`) |
| `unterhaltspflichtige` | API parameter (persisted as `vergleichsvorschlag.unterhaltspflichtige`) |
| `laufzeit_monate` | API parameter (default 72) |
| `start_datum` | API parameter (default today) |
| `glaeubiger[].name` | `glaeubieger.name` |
| `glaeubiger[].adresse` | `glaeubieger.kontakt` (parse multi-line) |
| `glaeubiger[].vertreter`, `vt_adr` | `glaeubieger.vertreter*` columns |
| `glaeubiger[].az_gl` | `glaeubieger.aktenzeichen` |
| `glaeubiger[].hf/zi/ko/gf` | `glaeubieger.hauptforderung/zinsen/kosten/forderung` |
| `glaeubiger[].zw` | `glaeubieger.zahlweise` (default `monatlich`) |

**Filter:** `WHERE schuldner_id = ? AND (sbp_ausgeschlossen IS NULL OR sbp_ausgeschlossen = 0)`

## API endpoints (BriefButler backend)

```
GET  /creditors/sbp                          List stored SBP entries
POST /creditors/sbp/{sbp_id}/link            Link SBP entry to creditor
POST /vergleich/sbp-pdf                      Generate + download PDF
     ?schuldner_id=<uuid>
     &einkommen=<float>
     &unterhaltspflichtige=<int>
     &laufzeit_monate=<int>
     &start_datum=<YYYY-MM-DD>
```

## Determinism guarantee

For **identical inputs** the PDF must be byte-identical (modulo the creation
date in the footer). A round-trip test:

```python
result1 = generate_sbp_pdf(schuldner="...", mdnr="...", glaeubiger=[...], einkommen=3000, ...)
result2 = generate_sbp_pdf(schuldner="...", mdnr="...", glaeubiger=[...], einkommen=3000, ...)
assert hash_without_timestamp(result1) == hash_without_timestamp(result2)
```

## Hard rules — what an agent **must not** do

- ❌ Compute its own garnishment table — always use the table above.
- ❌ Set `Vergleichsquote` below 100 % — that turns the document into a
  partial-waiver application ("Erlass-Antrag"), which is a different filing.
- ❌ Exclude creditors on its own — `sbp_ausgeschlossen` is a legal decision,
  not the agent's call.
- ❌ Simplify the layout — the three lines per claim component and the two
  empty checkboxes are formal requirements.
- ❌ Switch the EUR format — German format is mandatory for insolvency-court
  acceptance.
- ❌ Send the PDF without explicit human confirmation (same hard rule as
  `gmail-connector`: show draft → ask → wait → only then dispatch).
