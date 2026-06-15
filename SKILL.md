---
name: 3dmodel
description: Create 3D models and interactive 3D viewers from a coding agent. Use when asked to model an object/product/part in 3D, recreate something in 3D, build a web 3D viewer or product display, make an exploded or x-ray/cutaway view, export a GLB, or improve how a real-time 3D scene looks. Covers the closed-loop method (generate, render, look, critique, iterate), Blender headless modelling, Three.js web delivery, realism, and interactive-viewer UI patterns.
---

# 3D modelling and interactive 3D viewers

You cannot "see" a 3D scene by intuition. The only way to model well is a **closed feedback loop**: generate geometry, render it to a PNG (or screenshot the live viewer), **Read the image**, critique it like a picky art director, fix the worst defect, repeat. Never ship 3D you have not looked at. One defect per pass; re-render after each.

## Decide the delivery target first

- **Blender headless GLB** when the user wants an asset (a `.glb`/`.gltf` to reuse, hand off, or drop into a game/AR/another app), or photoreal stills. See `references/blender-pipeline.md`.
- **Procedural Three.js** when the user wants an interactive web viewer (orbit, animate, click, exploded/x-ray, configurator). You build geometry in JS, so you control everything and avoid export limits. See `references/realtime-3d-web.md`.
- **Hybrid**: model detailed parts in Blender, export GLB, load it in a Three.js viewer for interaction.

If the user wants a model on a website that is coloured, animated and user-movable, the simplest path is Google `<model-viewer>` (one HTML tag: `camera-controls`, `autoplay`, `ar`); the powerful path is Three.js / React-Three-Fiber. model-viewer plays only ONE animation clip at a time and is awkward to script, so for multi-part animation or click-to-reveal use Three.js directly.

## The loop, concretely

1. Set up a reusable render harness once (camera framing, 3-point or studio lights, neutral environment, sensible tone mapping). Reuse it for every iteration.
2. Build the model. Start simple, add detail in passes.
3. Render at low samples (fast). For Blender use EEVEE for drafts and Cycles only for hero shots. For web, screenshot the running page.
4. **Read the PNG.** Name the single worst problem (framing, proportion, material, lighting, a missing feature) and fix only that.
5. Repeat until it matches the reference. Then do a final high-quality render/export.

Always work to a reference. If recreating a real object, get reference images (or ask the user for them) and match proportion, colour, layout and the small identifying details. Looking "like a mockup" almost always means: flat untextured surfaces, no bevels, no shadows between parts, and wrong proportions.

## Making real-time 3D look real (not a mockup)

Biggest wins, roughly in order:
- **Soft shadow maps** (`renderer.shadowMap.enabled=true; type=PCFSoftShadowMap`, a key `DirectionalLight` with `castShadow`, large `shadow.mapSize`, `shadow.radius`/`blurSamples` for softness). Component-to-component shadows give crevice depth; a `ShadowMaterial` ground plane gives a soft contact shadow that grounds the object.
- **Bevelled edges** on everything (`RoundedBoxGeometry`, small radius). Sharp 90 degree edges read as fake because real edges catch a highlight.
- **Environment lighting** (`RoomEnvironment` + `PMREMGenerator`, or an HDRI) so metals and glossy plastics reflect something. Bump `envMapIntensity`.
- **Tone mapping** `ACESFilmicToneMapping` with tuned `toneMappingExposure`; modern Three.js lights need higher intensity than you expect.
- **PBR done right**: metalness is effectively binary (0 dielectric, 1 metal); never 0.2-0.8. Combine roughness + metalness + normal maps. For PCBs/painted plastic, a `MeshPhysicalMaterial` with a little `clearcoat` gives the lacquer/solder-mask sheen.
- **Texture detail beats geometry detail** at distance: bake silkscreen/labels/trace patterns into a canvas texture rather than modelling every part. Procedural shader nodes do NOT survive glTF export, so bake to image textures if exporting.
- Optional postprocessing (`EffectComposer`): subtle bloom on emissive parts, SSAO/N8AO for contact occlusion. Add only if needed; it adds dependencies and breakage risk.

## Interactive viewer UI patterns

- **Orbit + zoom**: `OrbitControls` with damping. Constrain min/max distance and polar angle so users cannot lose the object.
- **Frame to fit**: pick camera distance from the object's size and the viewport aspect so it never clips off-screen; re-check on resize. Reserve screen margins if you draw side labels.
- **Camera moves**: interpolate camera position in SPHERICAL coordinates around the target (not linear) so it arcs around the object instead of flying through it. Disable `OrbitControls` while a scripted camera move is running, re-enable after, or it fights you.
- **Reveal internals two ways**: (a) **exploded view** parts translate apart along an axis; (b) **x-ray** the outer shell fades out (keep a faint `EdgesGeometry` wireframe so the silhouette stays) and the internals show. If the real product does not open, prefer x-ray and do not imply a hinge/removable cover.
- **Labels / callouts**: do NOT pin labels to the live projected 3D point of each part; they overlap and jitter. Put labels in **fixed screen lanes** (e.g. three down each side) and draw a thin **leader line + dot** from each label to its part's projected point. Stable, no overlap. Write what the part DOES, not just its name.
- **Flow / relationship lines**: draw colour-coded lines between parts to show what connects to what and what it carries (e.g. power vs data vs signal), with a small mid-line label and a legend. Makes an assembly understandable to a non-expert.
- **Hide the overlay while moving**: lines anchored to 3D points swing and look glitchy during a drag. Track whether the camera moved this frame; if it did (or inertia is still settling), fade the whole label/line overlay out, and fade it back in only once the view is still for ~200ms. Also hide any label/line whose part is behind the camera (projected z >= 1).
- **Click vs drag**: a tap (pointer moved < ~6px and < ~320ms) is a click action (e.g. toggle x-ray); anything longer is a drag/rotate. Distinguish on `pointerup`.
- **Match the user's design language** for the surrounding UI; keep chrome minimal and let the object lead.

## Pitfalls that waste time (check these first when something looks wrong)

- **Transparent material clipping**: a semi-transparent solid still writes depth and visibly cuts whatever is behind it. Set `depthWrite=false` while transparent; keep materials fully opaque when not fading, or sorting artifacts appear on other objects (e.g. glowing text).
- **Z-fighting**: never leave two faces coplanar. Offset by a hair, or show only one at a time.
- **Frame-swap "animation"** (e.g. a dot-matrix showing different frames): glTF cannot animate textures/materials; either bake separate emissive textures and toggle mesh visibility in JS, or stack frames and step their scale. Do this in the web layer, not the GLB.
- **Procedural shaders do not export** to glTF. Only Principled-BSDF channels + image textures survive. Bake first.
- **Exporter quirks** (Blender): apply object scale to 1,1,1 before export; `export_apply=True` can bake a node's translation into the mesh so `getWorldPosition` returns the origin (anchor labels to known coords instead); auto/`smart_project` UV on a single quad mangles a texture, so set UVs explicitly; the Action API changed in Blender 4.x/5.x (use the keyframe-interpolation preference instead of poking `action.fcurves`).
- **Finding real part models**: Sketchfab/GrabCAD models are mostly login-gated; STLs (Printables/Thingiverse/MakerWorld) have no materials; poly.pizza is free but low-poly. Often the most reliable route is to recreate the part from reference images with textured PCBs/bodies rather than chase a download.

## Verify before claiming done

Render or screenshot the FINAL state and Read it. For interactive viewers, drive the live page (a headless browser via Playwright works well), exercise each state (closed, opening, open, while-dragging) and screenshot each, because bugs hide in transitions.

## Keep this skill growing

After any 3D task, fold genuinely new techniques or pitfalls back into this skill and its reference files (see the linked memory). Prefer durable, reusable knowledge over project specifics; never put a specific client's or project's details here.
