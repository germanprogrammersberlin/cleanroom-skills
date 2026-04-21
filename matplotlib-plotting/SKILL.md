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

## Publication Style Setup

```python
import matplotlib.pyplot as plt
import matplotlib as mpl

# Journal-quality defaults
mpl.rcParams.update({
    'font.family': 'serif',
    'font.size': 10,
    'axes.labelsize': 11,
    'axes.titlesize': 11,
    'xtick.labelsize': 9,
    'ytick.labelsize': 9,
    'legend.fontsize': 9,
    'figure.dpi': 300,
    'savefig.dpi': 300,
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
