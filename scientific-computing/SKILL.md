---
name: scientific-computing
description: Numerical and symbolic scientific computing with Python. Model calculations, simulations, curve fitting, ODE/PDE solving, statistical analysis, formula derivation. Core tools for physics/chemistry/biology research papers.
---

# Scientific Computing

Python-based scientific computing for research papers. Covers numerical simulation, symbolic math, curve fitting, and data analysis.

## Core Libraries

```bash
pip install numpy scipy sympy matplotlib lmfit pandas
```

### NumPy + SciPy — Numerical Computing
- Linear algebra, FFT, integration, interpolation
- ODE/PDE solvers (`scipy.integrate.solve_ivp`)
- Optimization and curve fitting (`scipy.optimize.curve_fit`, `scipy.optimize.minimize`)
- Signal processing, statistics

### SymPy — Symbolic Math
- Derive analytical formulas
- Simplify expressions, solve equations symbolically
- LaTeX output for manuscripts (`sympy.latex()`)
- Series expansion, limits, integrals

### lmfit — Advanced Curve Fitting
- Parameter constraints, bounds, expressions
- Confidence intervals, error propagation
- Model comparison (AIC, BIC)

## Common Patterns

### Derive a formula and verify numerically
```python
import sympy as sp

x, a, b = sp.symbols('x a b', real=True)
expr = sp.integrate(a * sp.exp(-b * x), (x, 0, sp.oo))
print(sp.latex(expr))  # For the manuscript
print(float(expr.subs([(a, 1.0), (b, 0.5)])))  # Numerical check
```

### Fit a model to experimental data
```python
import numpy as np
from scipy.optimize import curve_fit

def model(x, K_D, V_max):
    return V_max * x / (K_D + x)

popt, pcov = curve_fit(model, x_data, y_data, p0=[1e-9, 0.1])
perr = np.sqrt(np.diag(pcov))
print(f"K_D = {popt[0]:.2e} ± {perr[0]:.2e}")
```

### Solve an ODE system
```python
from scipy.integrate import solve_ivp

def poisson_boltzmann(x, y, kappa):
    psi, dpsi = y
    return [dpsi, kappa**2 * np.sinh(psi)]

sol = solve_ivp(poisson_boltzmann, [0, 10], [psi_0, 0], args=(kappa,), dense_output=True)
```

## Domain-Specific Tools

Agents may install additional packages as needed:
- **ASE** — Atomic Simulation Environment (crystal structures, molecular dynamics)
- **Atomap** — TEM image analysis, strain mapping
- **MDAnalysis** — Molecular dynamics trajectory analysis
- **FEniCS** — Finite element PDE solving
- **COMSOL/MATLAB** — If licensed and available
- **GNU Octave** — Free MATLAB alternative

## Best Practices

- Always include units in variable names or comments
- Verify analytical results numerically (and vice versa)
- Report uncertainties with every fitted parameter
- Save intermediate results to files (don't recompute expensive simulations)
- Use reproducible random seeds for stochastic simulations

This skill can be improved by agents as they encounter specific computational needs.
