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

## Hard-won gotchas (verified building a real parametric enclosure in code)

These are the failures that actually stop a build script; each one threw `RuntimeError: 3` or an `AttributeError` in practice.

- **`startExtent` is a property, not a setter.** Use `extInput.startExtent = adsk.fusion.OffsetStartDefinition.create(ValueInput.createByString("29 mm"))`. There is no `setStartExtent(...)` method — calling it raises `AttributeError: ... has no attribute 'setStartExtent'`. (The distance side does use methods: `setDistanceExtent`, `setSymmetricExtent`, `setTwoSidesExtent`.)
- **Root component name is read-only.** `root.name = '...'` throws `RuntimeError: 3 : root component name cannot be changed`. Rename the *document* (`doc.name`) or a sub-component instead, never the root.
- **Cut order matters — never recess deeper than the wall before cutting the through-hole.** A front recess (e.g. 3 mm acrylic seat) cut into a 2.5 mm wall removes the wall entirely in that footprint; a later window cut started on that face then fails with `RuntimeError: 3 : No target body found to cut or intersect!` because its profile sits in empty space. Cut the **through-window first**, then the **shallow rebate** (keep rebate depth < wall thickness so a seating lip survives).
- **Offset construction planes flip sign with the base plane's normal.** `xYConstructionPlane` normal is +Z, `yZConstructionPlane` is +X, but **`xZConstructionPlane` normal is -Y** (X×Z = -Y). So `setByOffset(xZ, +d)` moves toward -Y. Mismatched sign puts the sketch plane on the wrong face and the cut hits nothing. For side/bottom port slots it is far more robust to **skip offset planes entirely** and do a world-space tool cut: sketch the slot profile straddling the wall on a known plane (e.g. xY), then `startExtent = OffsetStartDefinition(...)` + `setDistanceExtent` to sweep the cut through the wall at the right depth. Deterministic regardless of face normals.
- **A cut profile only needs to overlap *some* body to succeed.** Partly over a void is fine; entirely over a void is the "No target body" error. Use this: a slot profile can extend past the part edge into empty space and still cut cleanly.
- **Verify a GUI-only script with a log file, not just a screenshot.** Have the script `open(path,'w')` then append a line after each feature and wrap the whole `run()` in `try/except` that writes `traceback.format_exc()`. Read that file back to see exactly which feature failed and why — the closed-loop equivalent of a render critique when you cannot orbit the viewport reliably. Each Run re-reads the script from disk, so edit-and-rerun needs no reload.
- **Each `documents.add(...)` makes a new file.** Re-running a build script that calls it leaves a pile of `Untitled(n)` docs (Personal tier caps editable docs at 10). Either reuse `app.activeProduct` when it is an empty design, or close the failed-run docs (tab × → Don't Save) between attempts.
- **Shell needs a planar face to open.** Pick the open-back face by scanning `body.faces` for the planar face with the extreme centroid along the open axis, add it to an `ObjectCollection`, then `shellFeatures.createInput(faces); sin.insideThickness = ...`.

## Driving Fusion from a coding agent (MCP)

Like blender-mcp, community MCP servers bridge an AI client to a *running* Fusion via an add-in:
- **`ndoo/fusion360-mcp-bridge`** (best for Claude Code): tools `fusion_execute` (run arbitrary Fusion API Python inside Fusion, full API access) + `fusion_screenshot` (capture the viewport as PNG). That pair reproduces the exact generate→run→screenshot→critique loop used for Blender.
- Others: `faust-machines/fusion360-mcp-server` (toolbar-level tools, works with Claude Code/Codex/Cursor), `Misterbra/fusion360-claude-ultimate` (84+ NL CAD tools), `rahayesj/ClaudeFusion360MCP` (ships skill files).
- Setup mirrors blender-mcp: clone → copy the add-in into Fusion's AddIns → run it (Shift+S → Add-Ins) → register the MCP server (`claude mcp add`) → restart the session. Needs Fusion installed + open + Python 3.10+. Architecture is usually a JSON-file or socket poll between the MCP server and the add-in.

### Wiring up ndoo/fusion360-mcp-bridge (verified end-to-end)

This is the fast path — once running you drive Fusion entirely through `fusion_execute` (run any `adsk.*` Python, `print()` output comes back) and `fusion_screenshot` (clean viewport PNG, any named direction), with zero mouse/keyboard and no modal error dialogs (exceptions return as text). Setup that worked:

1. `git clone https://github.com/ndoo/fusion360-mcp-bridge` then `bash scripts/quickstart-mac.sh` — it pip-installs `mcp`+`httpx` (prefers a `~/venv` if present, else `--user`; make a venv first to dodge PEP 668), generates `~/.fusion-mcp-secret` (mode 600), copies the add-in to `~/Library/Application Support/Autodesk/Autodesk Fusion 360/API/AddIns/FusionMCPBridge`, and merges an `mcpServers.fusion360` entry into `~/.claude/settings.json`.
2. In Fusion: **Shift+S → Add-Ins tab → select FusionMCPBridge → toggle Run** (its manifest sets `runOnStartup:true` so it auto-starts thereafter). A dialog confirms "started on port 7654 (token auth enabled)".
3. **Restart Claude Code** so the `fusion_execute`/`fusion_screenshot` MCP tools load.

- **Before the restart you can already drive it via curl** (handy to verify, or to keep working in the same session): all requests need `-H "Authorization: Bearer $(cat ~/.fusion-mcp-secret)"`. `GET /health`; `POST /execute -d '{"script":"...python..."}'` — the body key is **`script`** (not `code`); `print()` output returns in `result`. `POST /screenshot -d '{"direction":"iso-top-right","width":1024,"height":768}'` (POST, not GET) returns base64 in `screenshot`. Directions: front/back/left/right/top/bottom/iso-*.
- **Version bug to expect:** the add-in's screenshot handler calls `viewport.saveAsImageFileWithOptions(name,w,h,True)`, which on Fusion 2703.x raises "takes 2 positional arguments but 5 were given". Fix is `viewport.saveAsImageFile(name, w, h)` (the stable 3-arg method). Edit the add-in `.py`, then reload it without restarting Fusion by toggling its Run off/on in Shift+S (Fusion re-imports the module).
- **Reloading add-in code = toggle Run off then on** in Shift+S; no Fusion restart needed.

## Sourcing real existing CAD (so it matches reality)

Before modelling a real part, get the actual CAD: Fusion Insert → Insert McMaster-Carr Component (real STEP by part number), Insert Manufacturer Part, the SnapEDA Fusion app (PCB parts). Libraries: McMaster-Carr (free STEP for most parts), GrabCAD (millions of files), 3D ContentCentral, TraceParts, Digi-Key/SnapEDA. Import STEP/IGES via the Data Panel, then insert/derive. Recreate from references only when no usable model exists.

## Interop with Blender

CAD → render handoff: export STEP/IGES from Fusion for engineering exchange; export STL/OBJ (or import the STEP into Blender, which tessellates it to mesh) for rendering/look-dev in Blender, then glTF/FBX onward. Watch scale/units (Fusion cm vs Blender m), and that STL has no UVs and faceted normals, so re-shade/retopo in Blender for hero renders.
