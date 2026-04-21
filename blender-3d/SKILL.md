---
name: blender-3d
description: Create scientific 3D visualizations using Blender. Crystal structures, molecular models, material microstructures, schematic diagrams. Use when a paper needs high-quality 3D figures beyond standard 2D plots.
---

# Blender 3D Visualization

Create publication-quality 3D visualizations for scientific papers using Blender's Python API.

## When to Use

- Crystal structure visualizations
- Molecular models and protein structures
- Material microstructures and cross-sections
- Schematic diagrams with 3D perspective
- Any figure where 2D plotting is insufficient

## Available MCP Tools

The Blender MCP server provides these tools:

- `execute_blender_code` — Run Python code in Blender (bpy API)
- `get_scene_info` — Inspect current scene
- `get_object_info` — Get details about objects
- `get_viewport_screenshot` — Capture current viewport
- `set_texture` — Apply textures to objects
- `search_polyhaven_assets` — Find free PBR materials and HDRIs
- `download_polyhaven_asset` — Download assets from PolyHaven

## Workflow

1. Plan the visualization (what structure, what perspective, what style)
2. Use `execute_blender_code` to build the scene programmatically
3. Use `get_viewport_screenshot` to preview
4. Iterate until the figure matches journal standards
5. Render final image at publication resolution (300 DPI, TIFF or PNG)

## Style Guidelines for Scientific Figures

- Clean white or neutral background
- Consistent lighting (3-point or HDRI)
- Color scheme matching other figures in the paper
- Scale bars where applicable
- No unnecessary visual effects (no lens flare, minimal shadows)
- Export at minimum 300 DPI for print

## Example: Simple Crystal Unit Cell

```python
import bpy

# Clear scene
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()

# Create atoms as spheres
positions = [(0,0,0), (0.5,0.5,0), (0.5,0,0.5), (0,0.5,0.5)]
for pos in positions:
    bpy.ops.mesh.primitive_uv_sphere_add(radius=0.15, location=pos)

# Create unit cell wireframe
bpy.ops.mesh.primitive_cube_add(size=1, location=(0.25, 0.25, 0.25))
bpy.context.object.display_type = 'WIRE'
```

This skill can be improved by agents as they learn what visualizations are needed.
