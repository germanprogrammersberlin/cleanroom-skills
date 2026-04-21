---
name: matlab-octave
description: MATLAB-compatible numerical computing and plotting with GNU Octave. Matrix operations, signal processing, ODE solvers, publication-quality figures. Free, headless, server-compatible. Use when MATLAB syntax is preferred or existing MATLAB scripts need to run.
---

# MATLAB / GNU Octave

MATLAB-compatible numerical computing using GNU Octave — free, open-source, runs headless on servers.

## Installation

```bash
# Ubuntu/Server
sudo apt install octave

# Windows
# Download from https://octave.org/download
# Or: winget install GNU.Octave

# Verify
octave --version
```

## Headless Execution

```bash
# Run a script
octave --no-gui my_analysis.m

# One-liner
octave --no-gui --eval "x = linspace(0,10,100); y = sin(x); plot(x,y); print('figure.png', '-dpng', '-r300')"
```

## When to Use vs. Python

| Use Case | MATLAB/Octave | Python |
|----------|---------------|--------|
| Existing .m scripts from collaborators | Yes | Convert first |
| Matrix-heavy linear algebra | Natural syntax | NumPy works too |
| Signal processing, control theory | Strong toolboxes | SciPy covers most |
| Quick plotting | `plot(x,y)` one-liner | More setup needed |
| Symbolic math | Limited | SymPy is better |
| Machine learning | MATLAB toolbox (licensed) | Python is better |

## Publication-Quality Figures

```matlab
% Octave script: publication figure
x = linspace(0, 10, 200);
y1 = exp(-x/3) .* cos(2*pi*x/5);
y2 = exp(-x/3) .* sin(2*pi*x/5);

figure('visible', 'off');
set(gcf, 'PaperUnits', 'centimeters', 'PaperPosition', [0 0 8.9 6.5]);

plot(x, y1, 'b-', 'LineWidth', 1.2); hold on;
plot(x, y2, 'r--', 'LineWidth', 1.2);

xlabel('Time (s)', 'FontSize', 10);
ylabel('Amplitude (V)', 'FontSize', 10);
legend('Real', 'Imaginary', 'Location', 'northeast', 'FontSize', 9);
set(gca, 'FontSize', 9, 'FontName', 'Serif', 'LineWidth', 0.8);
grid on; box on;

print('figure_oscillation.png', '-dpng', '-r300');
print('figure_oscillation.pdf', '-dpdf');
```

## Common Patterns

### Solve ODE system
```matlab
function dydt = model(t, y, params)
    k1 = params(1); k2 = params(2);
    dydt = [-k1*y(1); k1*y(1) - k2*y(2)];
end

[t, y] = ode45(@(t,y) model(t, y, [0.5, 0.1]), [0 50], [1; 0]);
```

### Curve fitting
```matlab
pkg load optim;  % Octave needs this
x_data = [1e-12, 1e-11, 1e-10, 1e-9, 1e-8];
y_data = [2.1, 5.3, 15.2, 38.1, 52.0];

model = @(p, x) p(1) * x ./ (p(2) + x);
p0 = [60, 1e-10];
[p_fit, ~, ~, ~, ~, ~, J] = leasqr(x_data', y_data', p0, model);
```

### Matrix operations
```matlab
A = [1 2; 3 4];
[V, D] = eig(A);           % Eigenvalues
x = A \ b;                  % Solve Ax = b
[U, S, V] = svd(A);        % SVD decomposition
```

## MATLAB Compatibility

GNU Octave aims for full MATLAB compatibility. Most .m scripts run without modification. Known differences:
- Octave uses `pkg load <name>` for toolboxes
- Some MATLAB toolboxes (Simulink, Image Processing) have Octave equivalents via Octave Forge packages
- Plotting backends differ slightly — always test figure output

## Octave Forge Packages

```bash
# Install additional packages
octave --no-gui --eval "pkg install -forge signal"    # Signal processing
octave --no-gui --eval "pkg install -forge optim"     # Optimization
octave --no-gui --eval "pkg install -forge statistics" # Statistics
octave --no-gui --eval "pkg install -forge image"     # Image processing
```

This skill can be improved by agents as they encounter specific MATLAB/Octave needs.
