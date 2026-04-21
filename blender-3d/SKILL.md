---
name: blender-3d
description: Create scientific 3D visualizations using Blender headless mode. Crystal structures, molecular models, material microstructures, schematic diagrams. Runs without GUI — write a Python script, execute with blender --background. Works on servers.
---

# Blender 3D Visualization

Create publication-quality 3D visualizations using Blender in headless mode. No GUI, no addon, no MCP server needed — just `blender --background --python script.py`.

## When to Use

- Crystal structure visualizations
- Molecular models and protein structures
- Material microstructures and cross-sections
- Schematic diagrams with 3D perspective
- Any figure where 2D plotting is insufficient

## How It Works

Write a Python script using Blender's `bpy` API, then execute it headless:

```bash
blender --background --python my_figure.py
```

Blender starts without GUI, runs the script, renders to file, and exits. Works on any server with Blender installed.

## Version Compatibility

Blender renamed its render engines between versions. Always use this pattern:

```python
import bpy

# Compatible with Blender 4.x and 5.x
try:
    bpy.context.scene.render.engine = 'BLENDER_EEVEE'
except TypeError:
    bpy.context.scene.render.engine = 'BLENDER_EEVEE_NEXT'
```

## Template: Scientific Figure Script

```python
"""Scientific 3D figure — run with: blender --background --python this_script.py"""
import bpy

# --- CLEAR SCENE ---
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()

# --- BUILD YOUR SCENE HERE ---
# Create objects, materials, etc.
bpy.ops.mesh.primitive_uv_sphere_add(radius=0.2, location=(0, 0, 0))
obj = bpy.context.active_object
mat = bpy.data.materials.new(name="SciMat")
mat.use_nodes = True
bsdf = mat.node_tree.nodes["Principled BSDF"]
bsdf.inputs["Base Color"].default_value = (0.0, 0.45, 0.7, 1.0)
obj.data.materials.append(mat)

# --- CAMERA ---
bpy.ops.object.camera_add(location=(3, -3, 2))
cam = bpy.context.active_object
bpy.ops.object.empty_add(location=(0, 0, 0))
target = bpy.context.active_object
constraint = cam.constraints.new('TRACK_TO')
constraint.target = target
constraint.track_axis = 'TRACK_NEGATIVE_Z'
constraint.up_axis = 'UP_Y'
bpy.context.scene.camera = cam

# --- LIGHTING ---
bpy.ops.object.light_add(type='SUN', location=(3, -2, 5))
bpy.context.active_object.data.energy = 3.0

# --- WHITE BACKGROUND ---
world = bpy.context.scene.world
if not world:
    world = bpy.data.worlds.new("World")
    bpy.context.scene.world = world
world.use_nodes = True
bg = world.node_tree.nodes["Background"]
bg.inputs["Color"].default_value = (1.0, 1.0, 1.0, 1.0)

# --- RENDER SETTINGS ---
scene = bpy.context.scene
try:
    scene.render.engine = 'BLENDER_EEVEE'
except TypeError:
    scene.render.engine = 'BLENDER_EEVEE_NEXT'
scene.render.resolution_x = 1200
scene.render.resolution_y = 900
scene.render.filepath = "/path/to/output.png"
scene.render.image_settings.file_format = 'PNG'

# --- RENDER ---
bpy.ops.render.render(write_still=True)
print("Done")
```

## Style Guidelines for Scientific Figures

- Clean white or neutral background
- Consistent lighting (Sun or HDRI)
- Colorblind-friendly palette: blue #0072B2, orange #D55E00, green #009E73
- No unnecessary visual effects (no lens flare, minimal shadows)
- Export at 1200x900 minimum for single-column, 2400x1800 for double-column
- Use wireframe modifier (thickness 0.03+) for unit cell edges — thin wires disappear in renders

## Wireframe Tip

Blender wireframes are often invisible in renders. Use a WIREFRAME modifier:

```python
mod = obj.modifiers.new("Wire", "WIREFRAME")
mod.thickness = 0.04  # Make it thick enough to see
```

Or use cylinder meshes for edges if you need precise control.

This skill can be improved by agents as they learn what visualizations are needed.
