---
name: ovito-rendering
description: Publication-quality crystal structure rendering with OVITO Python API. Ray-traced images with ambient occlusion, shadows, depth of field. Reads CIF files natively. Headless, no GPU needed. The standard for top-journal figures.
---

# OVITO Rendering

Render publication-quality crystal structures using OVITO's Python API.
Ray-traced output with ambient occlusion and shadows — the same quality
you see in Nature, ACS Nano, Advanced Materials.

Runs fully headless. No GPU, no display needed.

## Quick Start

```python
from ovito.io import import_file
from ovito.vis import Viewport, TachyonRenderer

pipeline = import_file("structure.cif")
pipeline.add_to_scene()

vp = Viewport(type=Viewport.Type.Perspective, camera_dir=(2, 1, -1))
vp.zoom_all(size=(1600, 1200))

renderer = TachyonRenderer(
    shadows=True,
    ambient_occlusion=True,
    ambient_occlusion_brightness=0.8,
    antialiasing_samples=12
)

vp.render_image(
    filename="crystal.png",
    size=(1600, 1200),
    renderer=renderer,
    background=(1, 1, 1),  # white
    alpha=True              # transparent background
)
```

## Load Structures

```python
from ovito.io import import_file

# From CIF file
pipeline = import_file("h-CuI.cif")

# From XYZ
pipeline = import_file("atoms.xyz")

# From POSCAR/VASP
pipeline = import_file("POSCAR")

# From ASE/pymatgen (save to temp file first)
from ase.io import write
write("/tmp/struct.cif", atoms)
pipeline = import_file("/tmp/struct.cif")
```

## Download CIF from Materials Project

```python
from mp_api.client import MPRester

# Free API key from materialsproject.org
# Set env: export MP_API_KEY="your_key"
with MPRester() as mpr:
    # Search for CuI structures
    docs = mpr.materials.summary.search(
        formula="CuI",
        fields=["material_id", "structure", "symmetry"]
    )
    for d in docs:
        print(f"{d.material_id}: {d.symmetry.symbol}")
        d.structure.to(filename=f"{d.material_id}.cif")
```

## Download CIF from Crystallography Open Database (no login)

```bash
# Search and download directly
curl -s "https://www.crystallography.net/cod/result?formula=Cu1%20I1&format=urls" | head -5
# Download specific entry
curl -s "https://www.crystallography.net/cod/1234567.cif" -o structure.cif
```

## Custom Colors and Sizes

```python
from ovito.vis import ParticlesVis
from ovito.data import ParticleType

pipeline = import_file("structure.cif")

# Access particle types
data = pipeline.compute()
for ptype in data.particles.particle_types.types:
    if ptype.name == 'Cu':
        ptype.color = (0.72, 0.45, 0.20)   # copper/gold
        ptype.radius = 1.28                   # covalent radius in Å
    elif ptype.name == 'I':
        ptype.color = (0.40, 0.20, 0.60)    # dark violet
        ptype.radius = 1.40
    elif ptype.name == 'C':
        ptype.color = (0.30, 0.30, 0.30)    # dark grey
        ptype.radius = 0.77
    elif ptype.name == 'O':
        ptype.color = (0.80, 0.20, 0.20)    # red
        ptype.radius = 0.66

# Adjust visual style
vis = pipeline.source.data.particles.vis
vis.shape = ParticlesVis.Shape.Sphere
vis.radius = 0.4  # scale factor
```

## Camera Angles

```python
from ovito.vis import Viewport
import numpy as np

# Top-down (plan view) — good for 2D layers
vp = Viewport(type=Viewport.Type.Ortho, camera_dir=(0, 0, -1))

# Perspective view — good for 3D structures
vp = Viewport(type=Viewport.Type.Perspective, camera_dir=(2, 1, -1))

# Side view
vp = Viewport(type=Viewport.Type.Ortho, camera_dir=(1, 0, 0))

# Custom angle (azimuth, elevation in radians)
az, el = np.radians(30), np.radians(45)
vp = Viewport(
    type=Viewport.Type.Perspective,
    camera_dir=(np.cos(el)*np.sin(az), np.cos(el)*np.cos(az), np.sin(el))
)

# Zoom to fit
vp.zoom_all(size=(1600, 1200))
```

## Renderers

```python
from ovito.vis import TachyonRenderer, OSPRayRenderer

# TachyonRenderer — fast CPU ray-tracer, good quality
tachyon = TachyonRenderer(
    shadows=True,
    ambient_occlusion=True,
    ambient_occlusion_brightness=0.8,
    antialiasing_samples=12,
    direct_light_intensity=0.9
)

# OSPRayRenderer — highest quality, path-tracing
ospray = OSPRayRenderer(
    shadows=True,
    ambient_occlusion_samples=64,
    samples_per_pixel=32,
    max_ray_recursion=10,
    direct_light_intensity=1.0
)
```

## Supercells and Slabs

```python
from ovito.modifiers import ReplicateModifier, SliceModifier

# Create 4x4x1 supercell (for 2D layer visualization)
pipeline.modifiers.append(ReplicateModifier(
    num_x=4, num_y=4, num_z=1,
    adjust_box=True
))

# Slice to show a specific plane
pipeline.modifiers.append(SliceModifier(
    normal=(0, 0, 1),
    distance=0.0
))
```

## Show Bonds

```python
from ovito.modifiers import CreateBondsModifier
from ovito.vis import BondsVis

pipeline.modifiers.append(CreateBondsModifier(
    cutoff=3.0,  # max bond length in Å
    lower_cutoff=0.5
))

# Style bonds
bonds_vis = pipeline.source.data.bonds.vis if hasattr(pipeline.source.data, 'bonds') else None
# After compute:
data = pipeline.compute()
if data.bonds:
    data.bonds.vis.width = 0.2
    data.bonds.vis.shading = BondsVis.Shading.Normal
```

## Transparent Background (for Compositing)

```python
vp.render_image(
    filename="crystal_transparent.png",
    size=(1600, 1200),
    renderer=tachyon,
    background=(1, 1, 1),
    alpha=True  # PNG with alpha channel
)
```

## Workflow for Graphical Abstracts

1. **Get CIF** from Materials Project, COD, or build with pymatgen/ASE
2. **Load** in OVITO, set colors to match paper's color scheme
3. **Create supercell** if needed (ReplicateModifier)
4. **Set camera** — perspective for 3D effect, ortho for plan view
5. **Render** with TachyonRenderer, transparent background, high resolution
6. **Composite** onto the original figure using image-editing skill
7. **Verify** structure is crystallographically correct before delivery
