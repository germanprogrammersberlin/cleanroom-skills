---
name: matplotlib-plotting
description: Create publication-quality 2D figures using matplotlib. Use for data plots, charts, diagrams, and standard scientific figures. Covers styling for journal submission.
---

# Matplotlib Plotting

Create publication-quality 2D figures for scientific papers.

## When to Use

- Data plots (scatter, line, bar, histogram)
- Error bar plots and confidence intervals
- Phase diagrams and contour plots
- Multi-panel figures
- Any standard 2D scientific visualization

## Two-Stage DPI Workflow (review vs. submission)

**Never default to 300 dpi for working files.** Multi-panel composites at 300 dpi exceed 5 MB and break the 5 MB-per-image / 32 MB-per-request limits of agent vision pipelines. Use a two-stage workflow:

| Stage | DPI | Target size | Where |
|---|---|---|---|
| **Review / agent-readable** | 150 dpi | ≤ 2 MB per PNG | All intermediate iterations, all figures embedded in tool-results / comments |
| **Final submission** | 300 dpi | unconstrained | ONLY the final upload to journal/Drive after sign-off |

Always emit the review PNG first; only re-render at 300 dpi when the figure is approved and going to submission.

```python
import matplotlib.pyplot as plt
import matplotlib as mpl

# Default = REVIEW DPI. Override to 300 only for final submission.
REVIEW_DPI = 150
SUBMISSION_DPI = 300

mpl.rcParams.update({
    'font.family': 'serif',
    'font.size': 10,
    'axes.labelsize': 11,
    'axes.titlesize': 11,
    'xtick.labelsize': 9,
    'ytick.labelsize': 9,
    'legend.fontsize': 9,
    'figure.dpi': REVIEW_DPI,
    'savefig.dpi': REVIEW_DPI,
    'savefig.bbox': 'tight',
    'axes.linewidth': 0.8,
    'lines.linewidth': 1.2,
    'lines.markersize': 4,
})

# Common figure sizes (inches) for journals
SINGLE_COLUMN = (3.5, 2.8)    # ~89mm wide
DOUBLE_COLUMN = (7.0, 4.5)    # ~178mm wide
FULL_PAGE = (7.0, 9.0)        # full page
```

### Always size-check after savefig

```python
import os
fig.savefig(path, dpi=REVIEW_DPI)
size_mb = os.path.getsize(path) / 1e6
assert size_mb <= 4.0, f"PNG too large for agent pipeline: {size_mb:.1f} MB at {path}"
```

If the assert trips, lower DPI further (120/100), reduce the panel count per figure, or split into multiple figures. **Never** ship a > 4 MB PNG into a tool-result.

## Common Plot Types

### Scatter with Error Bars
```python
fig, ax = plt.subplots(figsize=SINGLE_COLUMN)
ax.errorbar(x, y, yerr=errors, fmt='o', capsize=3, color='#1f77b4')
ax.set_xlabel('Temperature (K)')
ax.set_ylabel('Conductivity (S/m)')
fig.savefig('figure_conductivity.pdf')
```

### Multi-Panel Figure
```python
fig, axes = plt.subplots(1, 3, figsize=DOUBLE_COLUMN)
for ax, label in zip(axes, ['(a)', '(b)', '(c)']):
    ax.text(0.02, 0.95, label, transform=ax.transAxes, fontweight='bold')
fig.savefig('figure_comparison.pdf')
```

## Export Guidelines

- **Format:** PDF or EPS for vector, TIFF for raster (300 DPI minimum)
- **Fonts:** Embed all fonts, use serif (matches LaTeX body text)
- **Colors:** Use colorblind-friendly palettes (e.g., `tab10`, `Set2`)
- **Labels:** All axes labeled with units in parentheses
- **Legend:** Inside plot or below, never overlapping data

## Colorblind-Friendly Palette

```python
colors = ['#0072B2', '#D55E00', '#009E73', '#CC79A7', '#F0E442', '#56B4E9']
```

This skill can be improved by agents as they learn project-specific plotting needs.
