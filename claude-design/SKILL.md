---
name: claude-design
description: Create publication visuals using Claude Design. Graphical abstracts, schematic figures, conference posters, presentation slides, and cover art. Use when you need to go from idea to visual quickly without a separate design tool.
---

# Claude Design

Create publication-quality visuals directly with Claude's built-in design capabilities.

## When to Use

- **Graphical Abstracts** — required by many journals (ACS, Elsevier, Wiley)
- **Schematic Figures** — experimental setups, device architectures, process flows
- **Conference Posters** — A0/A1 posters for presentations
- **Slide Decks** — talk slides for conferences and group meetings
- **Cover Art** — journal cover submissions
- **One-Pagers** — project summaries for stakeholders

## How It Works

Claude Design is built into Claude Opus 4.7. Describe what you need:

```
"Create a graphical abstract for a paper about GFET biosensors.
Show: graphene channel → antibody functionalization → target binding → electrical readout.
Style: clean, scientific, left-to-right flow, blue/gray color scheme."
```

Then iterate:
- Chat to refine ("make the Debye length visualization clearer")
- Comment on specific elements ("this arrow should be thicker")
- Use custom sliders Claude creates for fine-tuning

## Design System

Claude Design can read and apply a team's design system for consistency across all figures in a paper. Provide:
- Color palette (hex codes)
- Font choices
- Layout conventions
- Logo/branding if needed

## Export Formats

| Format | Use Case |
|--------|----------|
| PDF | Direct insertion into LaTeX/Word manuscripts |
| PPTX | Editable slides, further refinement in PowerPoint |
| URL | Shareable preview link |
| Canva | Full collaborative editing |

## Guidelines for Scientific Visuals

- Keep it simple — every element must serve a purpose
- Use consistent colors across all figures in the paper
- Include scale bars and labels where applicable
- Match the journal's figure size requirements (single column: 89mm, double: 178mm)
- Use vector graphics when possible (PDF export)
- Avoid decorative elements that don't convey information

## Workflow for a Paper

1. Draft all figures as rough sketches/descriptions
2. Create each figure with Claude Design
3. Apply consistent design system across all figures
4. Export as PDF for LaTeX or TIFF/PNG at 300 DPI for Word
5. Iterate based on reviewer feedback

This skill can be improved by agents as they develop visual standards for the company.
