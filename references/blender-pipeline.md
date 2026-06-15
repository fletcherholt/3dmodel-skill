# Blender headless modelling and GLB export

Use Blender from the command line so the whole loop is scriptable: write a Python script, render a PNG, Read it, iterate. No GUI needed.

## Going pro: the standard pipeline, API surface, and key facts (verified against Blender docs)

- **The professional pipeline** (codified by Blender Studio's official Fundamentals course): model → texture → shade → rig → animate → light → render + composite to an image sequence. Character animation is taught "by example" from a bouncing ball up to full acting. That order is the learning path too.
- **Two render engines:** EEVEE is the real-time rasterizer (fast, interactive, renders PBR, good for drafts and even finals); Cycles is the path tracer (GPU rendering, render baking, Open Shading Language) for physically-accurate hero shots. They share the same shader nodes, so EEVEE is a live preview of Cycles materials. (Drafts in EEVEE, hero in Cycles.)
- **UV unwrap** (`Unwrap` operator cuts along seams, flattens, overwrites existing UVs) offers three methods: Angle-Based (ABF), Conformal (LSCM, better on simple objects), Minimum Stretch (SLIM, minimises area + angle distortion, added in 4.3).
- **glTF export** carries meshes, Principled-BSDF + Unlit materials, textures, cameras, punctual lights, extensions, custom properties, and animation (keyframe/shape-key/skinning); materials are metal/rough PBR (Base Color, Metallic, Roughness, baked AO, +Y tangent normal, Emissive). FBX/OBJ/STL are separate exporters.
- **The `bpy` API surface:** an embedded Python interpreter exposes `bpy.context` (current state), `bpy.data` (the .blend datablocks), `bpy.ops` (operators), `bpy.types`, `bpy.props`, `bpy.utils`, `bpy.app`, `bpy.path`, `bpy.msgbus`, plus standalone modules `bmesh` (mesh editing/retopo automation), `mathutils` (vectors/matrices/quaternions), `gpu` (drawing), `blf` (fonts), `aud` (audio). Almost everything (animation, rendering, import/export, object creation, repetitive tasks) is scriptable; prefer `bpy.data`/`bmesh` over `bpy.ops` for reliability in headless scripts.

## Setup

- Install: `brew install --cask blender` (macOS) gives a `blender` CLI. Python 3 with `Pillow`/`numpy` is handy for generating textures.
- Run a script headless: `blender -b --python myscript.py`
- Verify the API works: `blender -b --python-expr "import bpy; print(bpy.app.version_string)"`
- Optional live control: the community `blender-mcp` addon exposes a running Blender to an MCP client; the headless script loop needs none of it.

## Reusable render harness

Build this once and import it from every lesson/build script: factory reset, a gradient world, three-point or studio lights, a tracked camera, PBR helper, and a render function. Keep drafts fast.

- Use **EEVEE** for fast drafts (seconds), **Cycles** only for final hero shots (slow on CPU). In Blender 5.x the EEVEE engine enum is `BLENDER_EEVEE`.
- Render at a low sample count while iterating; raise it only for the final.
- `view_settings.view_transform = "AgX"` (or "Filmic"/"Standard") affects colour a lot; AgX desaturates, so push material saturation if colours look washed.
- Support a non-square aspect (`resolution_x`, `resolution_y`) so wide objects frame properly.

## Geometry tips

- Real corners are never perfectly sharp: add a small **Bevel** modifier (limit by angle) so edges catch light.
- For a rounded-rectangle faceplate, build the rounded-rect outline from corner arcs with `bmesh`, then extrude to depth; bevel only the front/back perimeter for a soft pillow edge.
- Subdivision surface + shade-smooth for organic smoothness.
- For an enclosure you will open up, model it as a real shell (delete the back cap, add a Solidify modifier for wall thickness) so internals are visible, or model parts separately for an exploded view.

## Materials that survive glTF export

glTF carries a metal/rough PBR model read from the **Principled BSDF**: Base Color, Metallic, Roughness, Normal (+Y, tangent space), Emissive (+ emissive strength), plus unlit via KHR_materials_unlit. Extra realism rides specific `KHR_materials_*` extensions (emissive strength, transmission, IOR export from the BSDF directly; volume/sheen/specular need dedicated nodes).

Procedural shader node networks do NOT export. If you used them, **bake to image textures** first (bake Base Color, Roughness, Normal, Emission), then plug the baked images into the BSDF.

## Export

```python
bpy.ops.export_scene.gltf(filepath="out.glb", export_format="GLB",
    use_selection=True, export_apply=True, export_animations=True)
```
- Prefer single-file `.glb` for sharing (embedded base64 is the least efficient form).
- Compress: Draco for static geometry; **Meshopt** if there is animation or morph targets (Draco drops those). KTX2/Basis for textures. The `gltf-transform` CLI does all of this after export.

## Export gotchas (these cost real time)

- **Apply scale to 1,1,1** (Ctrl+A in UI, or `bpy.ops.object.transform_apply(scale=True)`) before export or the model comes in at the wrong size. Exception: applying transforms on a skinned/armatured mesh can break skinning; turn the exporter's Skinning option ON for armatures.
- `export_apply=True` applies modifiers, and can bake an object's translation into its mesh so the exported node has no translation; do not rely on node transforms for placing labels afterwards.
- **UVs**: `smart_project` on a single quad gives unpredictable orientation; set UVs explicitly via `bmesh` loops for a clean planar map.
- **Animation**: glTF animates node transforms, morph weights and skin, NOT material/texture values. For a per-frame texture change, bake separate textures and swap in the viewer. The Blender exporter often writes one glTF clip per object Action; `<model-viewer>` plays only one clip, so either merge clips post-export or drive multi-node animation in the web layer.
- **API drift**: `mesh.use_auto_smooth` was removed in 4.x; the Action `fcurves` API changed with slotted actions in 4.4/5.x. Set new-keyframe interpolation via `bpy.context.preferences.edit.keyframe_new_interpolation_type` instead of poking fcurves.

## Generating textures programmatically

Use Pillow to draw silkscreen, dot-matrix faces, labels, decals as PNGs, then load them as image textures (Base Color / Emission). This is fast, controllable, and exports cleanly. For an emissive glow look, draw the lit shape and screen-composite a Gaussian-blurred copy over it.
