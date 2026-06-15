# Autodesk Fusion (formerly Fusion 360): parametric CAD, its Python API, and driving it from an agent

Note on the name: Autodesk rebranded "Fusion 360" to **"Autodesk Fusion"**. It is the same application with the same Python API (`adsk.core`, `adsk.fusion`, `adsk.cam`) and the same workflow; only the brand changed. Docs, community guides, and the MCP bridges still mostly say "fusion360", so treat the two names as interchangeable.

Fusion is a history-based parametric modeller for precise, manufacturable parts (the opposite end from Blender's mesh/organic/render). Use it for engineered parts, enclosures with tolerances, anything for CNC/3D-print/sheet-metal, and assemblies with real joints. Verified against Autodesk and Blender primary docs plus the Fusion API reference.

## The parametric workflow (do it in this order)

1. **Sketch** a 2D profile (lines/arcs/circles/splines) on a plane or face.
2. **Constrain it fully.** Geometric constraints (coincident, parallel, perpendicular, tangent, equal, horizontal/vertical, concentric, midpoint, symmetry) define relationships; dimensions define sizes. Under-constrained geometry shows blue; fully constrained shows black with the sketch locked. Prefer one dimension + an equal constraint over repeating the same number.
3. **Features** turn sketches into solids: extrude, revolve, loft, sweep, then modify with fillet, chamfer, shell, hole, pattern (rectangular/circular), mirror, combine. Each is recorded in the **timeline** and auto-updates when upstream geometry or parameters change.
4. **Components, not bodies.** A body is raw geometry; a component is a container with its own origin/sketches/timeline and is what assemblies, joints, BOM and exploded views need. Start designs with components.
5. **Joints** describe motion (Rigid, Revolute, Slider, Cylindrical, Pin-Slot, Planar, Ball); constraints describe position. Add joints early, before fillets. Use as-built joints for parts already positioned (top-down).
6. **User parameters** drive everything: define `wall`, `width`, etc. and reference them in dimensions; change one number and the whole model updates.

Assembly approaches: bottom-up (assemble premade parts), middle-out, or top-down (skeleton/master layout first). Fusion can analyse assembly motion.

## Design-for-change best practices

Minimise dependencies (every reference is a future failure point). Name parameters meaningfully (`wall`, not `D1`) and drive shared dimensions from a single parameter. Rename sketches/features in the timeline. Do not over-constrain. Decide which dimensions must flex before modelling. Models break after edits from constraint conflicts or broken references; debug from the timeline's error markers and fix the upstream feature.

## Python API (`adsk.core`, `adsk.fusion`, `adsk.cam`)

Entry point: a script defines `def run(context):`; an add-in has `run` + `stop`. Object model: Application → Documents → Design → rootComponent → sketches/features/geometry.

```python
import adsk.core, adsk.fusion, traceback
def run(context):
    app = adsk.core.Application.get()
    ui  = app.userInterface
    design = adsk.fusion.Design.cast(app.activeProduct)
    root   = design.rootComponent

    # sketch a circle and extrude it
    sk = root.sketches.add(root.xYConstructionPlane)
    sk.sketchCurves.sketchCircles.addByCenterRadius(adsk.core.Point3D.create(0,0,0), 2.0)  # 2.0 = 2 cm!
    prof = sk.profiles.item(0)
    exts = root.features.extrudeFeatures
    inp  = exts.createInput(prof, adsk.fusion.FeatureOperations.NewBodyFeatureOperation)
    inp.setDistanceExtent(False, adsk.core.ValueInput.createByString("10 mm"))  # real units
    exts.add(inp)

    # a user parameter that drives a feature
    design.userParameters.add("wall", adsk.core.ValueInput.createByString("2 mm"), "mm", "wall thickness")
```

- **UNITS GOTCHA (the one that bites everyone):** internal database units are **centimetres for length and radians for angles**. `ValueInput.createByReal(5)` means 5 cm / 5 rad, no conversion. For real-world units use `ValueInput.createByString("5 mm")` (also accepts expressions like `"wall*2"` or parameter names). `UnitsManager.convert(...)` for explicit conversion.
- **Feature pattern:** simple features have `addSimple(...)`; complex ones use the input pattern `inp = feats.createInput(...); inp.set...(); feats.add(inp)` (the dialog-equivalent). `ObjectCollection.create()` builds transient groups to pass in.
- **Sketch constraints in code:** `c = sk.geometricConstraints` then `c.addCoincident/addParallel/addPerpendicular/addTangent/addEqual/...`; dimensions `sk.sketchDimensions.addDistanceDimension(p1, p2, orientation, textPoint)` and set/rename the returned dim's `.parameter`.
- **Joints in code:** `geo = adsk.fusion.JointGeometry.createByPlanarFace(...)`; `inp = root.joints.createInput(geo1, geo2)`; `inp.setAsRevoluteJointMotion(adsk.fusion.JointDirections.ZAxisJointDirection)` (or Slider/Rigid/...); `root.joints.add(inp)`.
- **Export:** `em = design.exportManager`; `em.execute(em.createSTEPExportOptions(path, root))` (also STL/IGES/F3D/3MF/SAT).
- **CAM:** `adsk.cam`; `cam.generateAllToolpaths(skipValid)` is asynchronous (returns a GenerateToolpathFuture); operation parameters are accessed by name; toolpaths can be posted. API is limited but growing.
- **Configurations** manage many variants from one parametric model. Community wrappers `fscad` and `EasyFusionAPI` cut boilerplate.
- **No headless/CLI batch.** Fusion only runs with the GUI; scripts execute inside the running app (Shift+S → Scripts and Add-Ins). Plan around that.

## Driving Fusion from a coding agent (MCP)

Like blender-mcp, community MCP servers bridge an AI client to a *running* Fusion via an add-in:
- **`ndoo/fusion360-mcp-bridge`** (best for Claude Code): tools `fusion_execute` (run arbitrary Fusion API Python inside Fusion, full API access) + `fusion_screenshot` (capture the viewport as PNG). That pair reproduces the exact generate→run→screenshot→critique loop used for Blender.
- Others: `faust-machines/fusion360-mcp-server` (toolbar-level tools, works with Claude Code/Codex/Cursor), `Misterbra/fusion360-claude-ultimate` (84+ NL CAD tools), `rahayesj/ClaudeFusion360MCP` (ships skill files).
- Setup mirrors blender-mcp: clone → copy the add-in into Fusion's AddIns → run it (Shift+S → Add-Ins) → register the MCP server (`claude mcp add`) → restart the session. Needs Fusion installed + open + Python 3.10+. Architecture is usually a JSON-file or socket poll between the MCP server and the add-in.

## Sourcing real existing CAD (so it matches reality)

Before modelling a real part, get the actual CAD: Fusion Insert → Insert McMaster-Carr Component (real STEP by part number), Insert Manufacturer Part, the SnapEDA Fusion app (PCB parts). Libraries: McMaster-Carr (free STEP for most parts), GrabCAD (millions of files), 3D ContentCentral, TraceParts, Digi-Key/SnapEDA. Import STEP/IGES via the Data Panel, then insert/derive. Recreate from references only when no usable model exists.

## Interop with Blender

CAD → render handoff: export STEP/IGES from Fusion for engineering exchange; export STL/OBJ (or import the STEP into Blender, which tessellates it to mesh) for rendering/look-dev in Blender, then glTF/FBX onward. Watch scale/units (Fusion cm vs Blender m), and that STL has no UVs and faceted normals, so re-shade/retopo in Blender for hero renders.
