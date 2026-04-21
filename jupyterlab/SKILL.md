---
name: jupyterlab
description: Create and execute Jupyter notebooks for interactive scientific analysis. Combine code, equations, plots, and narrative. Use for exploratory data analysis, model development, and reproducible research workflows.
---

# JupyterLab

Create and execute Jupyter notebooks for interactive scientific computing. Notebooks combine executable code, equations, visualizations, and explanatory text — ideal for scientific analysis workflows.

## Installation

```bash
pip install jupyterlab ipykernel nbformat nbconvert
```

## Creating Notebooks Programmatically

Agents can create notebooks without a GUI using `nbformat`:

```python
import nbformat

nb = nbformat.v4.new_notebook()

# Add a markdown cell
nb.cells.append(nbformat.v4.new_markdown_cell("""
# Analysis: Binding Kinetics
## Model fitting for dose-response curves
"""))

# Add a code cell
nb.cells.append(nbformat.v4.new_code_cell("""
import numpy as np
import matplotlib.pyplot as plt
from scipy.optimize import curve_fit

# Load data
data = np.loadtxt('dose_response.csv', delimiter=',', skiprows=1)
concentration = data[:, 0]
signal = data[:, 1]

# Langmuir model
def langmuir(c, K_D, V_max):
    return V_max * c / (K_D + c)

popt, pcov = curve_fit(langmuir, concentration, signal)
print(f"K_D = {popt[0]:.3e} M")
print(f"V_max = {popt[1]:.1f} mV")
"""))

# Save
with open('analysis.ipynb', 'w') as f:
    nbformat.write(nb, f)
```

## Executing Notebooks Headless

Run a notebook without GUI and capture all outputs:

```bash
jupyter nbconvert --to notebook --execute analysis.ipynb --output analysis_executed.ipynb
```

Or in Python:

```python
import nbformat
from nbconvert.preprocessors import ExecutePreprocessor

with open('analysis.ipynb') as f:
    nb = nbformat.read(f, as_version=4)

ep = ExecutePreprocessor(timeout=600, kernel_name='python3')
ep.preprocess(nb)

with open('analysis_executed.ipynb', 'w') as f:
    nbformat.write(nb, f)
```

## Converting Notebooks

```bash
# To PDF (for sharing)
jupyter nbconvert --to pdf analysis.ipynb

# To HTML (for web)
jupyter nbconvert --to html analysis.ipynb

# To Python script (for production)
jupyter nbconvert --to script analysis.ipynb

# To LaTeX (for manuscript SI)
jupyter nbconvert --to latex analysis.ipynb
```

## Workflow for a Paper

1. **Exploratory notebook**: Load data, plot, explore, try different models
2. **Analysis notebook**: Clean version of the winning approach, all figures
3. **Execute headless**: `jupyter nbconvert --execute` to reproduce results
4. **Export figures**: Save publication-quality figures from notebook to files
5. **Convert to SI**: `nbconvert --to pdf` as Supplementary Information

## Best Practices

- One notebook per analysis task (not one giant notebook)
- First cell: imports and constants
- Last cell: summary of key results
- Save figures to separate files (`fig.savefig('fig01.pdf')`) for the manuscript
- Clear all outputs before committing to Git (`jupyter nbconvert --clear-output`)
- Include a requirements cell documenting package versions

This skill can be improved by agents as they develop analysis workflows.
