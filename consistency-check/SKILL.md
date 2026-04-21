---
name: consistency-check
description: Check consistency across all sections — values, units, sample names, figure references, symbols. Cross-check data between text, figures, tables. Report contradictions.
---

# Consistency Check

Ensure consistency across the entire paper.

## What to Check

- **Values**: Does Section 3 say "5.2 nm" and Section 5 say "5.3 nm"?
- **Units**: Not "nm" and "nanometer" mixed in the same paper
- **Sample names**: Not "Sample A" and "Probe A" mixed
- **Figure references**: \ref{fig:3} — does Figure 3 exist?
- **Table values vs text**: Do numbers in tables match those cited in text?
- **Symbols**: Not "σ" and "sigma" and "stress" used interchangeably

## How to Check

1. Read all sections the finding refers to
2. Extract all numerical values with units
3. Compare across sections, figures, and tables
4. On contradiction: report both values and their locations

## What NOT to Do

- Do not decide which value is correct — only report the contradiction
- Do not rewrite text — only correct values, units, and names
