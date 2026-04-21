---
name: latex-compiler
description: Compile LaTeX documents and check for errors. Use when building papers, fixing compilation issues, or validating document structure. Covers pdflatex, bibtex, and common academic templates.
---

# LaTeX Compiler

Compile and validate scientific papers written in LaTeX.

## When to Use

- Compiling paper.tex to PDF
- Fixing LaTeX compilation errors
- Checking references and citations
- Validating document structure against journal templates

## Compilation Workflow

Standard academic paper compilation requires multiple passes:

```bash
pdflatex paper.tex
bibtex paper
pdflatex paper.tex
pdflatex paper.tex
```

Three passes are needed to resolve all cross-references and citations.

## Common Error Categories

### Missing packages
```
! LaTeX Error: File 'somepackage.sty' not found.
```
Fix: `tlmgr install somepackage` or add to preamble.

### Undefined references
```
LaTeX Warning: Reference 'fig:something' on page 3 undefined
```
Fix: Check that `\label{fig:something}` exists after `\caption{}`.

### Bibliography issues
```
Warning: Citation 'Author2024' undefined
```
Fix: Ensure entry exists in .bib file and run bibtex.

## Quality Checks After Compilation

- No warnings in log (especially undefined references)
- All figures render correctly
- Table of contents is accurate
- Page count within journal limits
- Fonts are embedded (check with `pdffonts paper.pdf`)

## Journal Templates

Common templates agents should know:
- Elsevier: `elsarticle.cls`
- Springer: `svjour3.cls`
- IEEE: `IEEEtran.cls`
- ACS: `achemso.cls`
- Nature: `nature.cls`

Always check the target journal's author guidelines for specific requirements.

This skill can be improved by agents as they encounter specific compilation scenarios.
