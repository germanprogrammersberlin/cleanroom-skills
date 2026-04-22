---
name: crystal-structure
description: Build crystal structures from space groups, CIF files, or coordinates using ASE and pymatgen. For rendering, use the ovito-rendering skill (ray-traced, publication quality). This skill is for structure BUILDING, ovito-rendering is for RENDERING.
---

# Crystal Structure Visualization

Build scientifically correct crystal structures and render publication-quality images
using ASE (Atomic Simulation Environment) and pymatgen.

## Installation

Pre-installed: `ase`, `pymatgen`, `matplotlib`, `numpy`.

## Build Structures from Space Group

```python
from ase import Atoms
from ase.spacegroup import crystal

# Example: h-CuI (hexagonal CuI, P6₃mc, #186)
# Cu at (1/3, 2/3, 0), I at (1/3, 2/3, 0.375)
hCuI = crystal(
    symbols=['Cu', 'I'],
    basis=[(1/3, 2/3, 0.0), (1/3, 2/3, 0.375)],
    spacegroup=186,
    cellpar=[4.31, 4.31, 7.09, 90, 90, 120]
)

# Example: gamma-CuI (zincblende, F-43m, #216)
gamma_CuI = crystal(
    symbols=['Cu', 'I'],
    basis=[(0, 0, 0), (0.25, 0.25, 0.25)],
    spacegroup=216,
    cellpar=[6.05, 6.05, 6.05, 90, 90, 90]
)
```

## Build Structures from CIF

```python
from ase.io import read

atoms = read("structure.cif")
```

## Build from pymatgen (Materials Project)

```python
from pymatgen.core import Structure, Lattice

# h-CuI from scratch
lattice = Lattice.hexagonal(a=4.31, c=7.09)
structure = Structure(
    lattice,
    ['Cu', 'I'],
    [[1/3, 2/3, 0.0], [1/3, 2/3, 0.375]]
)

# Convert to ASE for rendering
from pymatgen.io.ase import AseAtomsAdaptor
atoms = AseAtomsAdaptor.get_atoms(structure)
```

## Build Molecules (Clusters, Trimers)

```python
from ase import Atoms
import numpy as np

# Cu3I3 trimer (planar hexagonal ring, alternating Cu-I)
# Approximate: 6-membered ring, radius ~2.5 Å
r = 2.5
angles = [0, 60, 120, 180, 240, 300]
positions = []
symbols = []
for i, ang in enumerate(angles):
    rad = np.radians(ang)
    x, y = r * np.cos(rad), r * np.sin(rad)
    positions.append([x, y, 0])
    symbols.append('Cu' if i % 2 == 0 else 'I')

cu3i3 = Atoms(symbols=symbols, positions=positions)
```

## Render with ASE (matplotlib backend)

```python
from ase.visualize.plot import plot_atoms
import matplotlib.pyplot as plt

fig, ax = plt.subplots(1, 1, figsize=(8, 8))

# Top-down view (along z-axis)
plot_atoms(atoms, ax, rotation='0x,0y,0z', radii=0.8, show_unit_cell=2)

ax.set_axis_off()
plt.tight_layout()
plt.savefig('structure.png', dpi=300, bbox_inches='tight', transparent=True)
```

## Render Different Views

```python
# Top view (default, looking down z)
plot_atoms(atoms, ax, rotation='0x,0y,0z')

# Side view
plot_atoms(atoms, ax, rotation='90x,0y,0z')

# Perspective view
plot_atoms(atoms, ax, rotation='45x,30y,0z')
```

## Supercells (Larger Regions)

```python
# 3x3x1 supercell for a 2D layer
supercell = atoms.repeat((3, 3, 1))
```

## Custom Colors and Sizes

```python
from ase.data.colors import jmol_colors
from ase.data import atomic_numbers

# Custom color map
my_colors = {
    'Cu': (0.72, 0.45, 0.20),   # copper/gold
    'I':  (0.40, 0.20, 0.60),   # dark violet
    'C':  (0.30, 0.30, 0.30),   # dark grey
    'O':  (0.80, 0.20, 0.20),   # red for oxygen groups
}

# Apply to plot_atoms via colors array
colors = [my_colors.get(s, (0.5, 0.5, 0.5)) for s in atoms.get_chemical_symbols()]
plot_atoms(atoms, ax, rotation='0x,0y,0z', colors=colors, radii=0.8)
```

## Graphene and Oxo-Graphene

```python
from ase.build import graphene_nanoribbon, graphene

# Pristine graphene sheet
gr = graphene(size=(6, 6), vacuum=10.0)

# Oxo-graphene: add oxygen groups to some carbon positions
import random
oxo = gr.copy()
carbons = [i for i, s in enumerate(oxo.get_chemical_symbols()) if s == 'C']
# Add epoxy O above ~15% of C atoms
for idx in random.sample(carbons, k=int(0.15 * len(carbons))):
    pos = oxo.positions[idx].copy()
    pos[2] += 1.5  # above the plane
    oxo.append(Atoms('O', positions=[pos]))
```

## Heterostructures (Layer on Substrate)

```python
# Stack h-CuI layer on graphene
from ase.build import stack

interface = stack(graphene_slab, hcui_slab, distance=3.3, axis=2)
```

## Export for Blender (advanced 3D rendering)

```python
from ase.io import write

# Export as POV-Ray input (can convert to Blender)
write('structure.pov', atoms, rotation='45x,30y,0z')

# Or export as XYZ for import into any 3D tool
write('structure.xyz', atoms)
```

## Workflow

1. Identify the crystal structure from the paper (space group, lattice parameters, basis)
2. Build with ASE spacegroup or pymatgen
3. Create supercell if needed for visual clarity
4. Set colors matching the paper's color scheme
5. Render multiple views, pick the best
6. Export as transparent PNG for composition
