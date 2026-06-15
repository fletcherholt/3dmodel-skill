# AI generation: the fast lane (generate a base, then refine)

What separates a fast, powerful 3D agent from a slow one is this: for organic, complex or one-off assets (props, characters, creatures, sculptural shapes, "a model of X"), do NOT hand-model from a blank scene. **Generate a base mesh in seconds with a text/image-to-3D model, then refine it in Blender.** Generation outputs GLB, which drops straight into the existing pipeline. The closed loop still rules: Read the generated result, regenerate with a better prompt/image or pick best-of-N, then clean it up.

When NOT to generate: precise, dimensioned, manufacturable parts (enclosures, brackets, anything for CNC/print-to-spec) — those go to Fusion 360 parametric CAD. Generation is for look, not tolerances.

## Pick the generator

- **Tripo** — fastest (~20-30s) and cheapest (API ~$0.01/credit, free credits on signup, up to ~2M polys), text or image to GLB. Default for speed/cost and best-of-N exploration.
- **Meshy** (Meshy-6) — ~40-60s, strong print-readiness, text/image/multi-image, separate texture-refine step, GLB.
- **Rodin / Hyper3D (Gen-2)** — highest fidelity/photorealism, slower (~60-180s), premium price. Use for a hero asset.
- **Open-source, self-hostable (free, offline, no per-asset cost):** **TRELLIS-2** (Microsoft, ~4B params, image/text to mesh + Gaussians + radiance fields, PBR, runs as low as ~6GB VRAM) is the best free starting point; **Hunyuan3D-2.1** (Tencent, shape DiT + texture Paint, production PBR, ~6GB shape / ~16GB shape+texture, full training code open) is top open quality; InstantMesh / Stable Fast 3D for single-image speed (lower geometric fidelity).
- Aggregators (3D AI Studio etc.) let you fire one prompt at several engines and pick the best — that "best-of-N across engines" is the quality trick.

These are REST APIs: POST a prompt or reference image, poll the job until done, download the GLB. An agent can do this end to end, then verify by rendering the result (close the loop) and re-prompting if it's wrong. Image-to-3D beats text-to-3D when you have a reference image or need exact styling; text-to-3D has largely caught up otherwise.

## Refine what you generated (this is where quality comes from)

Generated meshes are dense, triangulated and have messy topology and baked-on textures. Professional output = generate then clean:
1. **Import the GLB to Blender**, check scale/orientation.
2. **Retopologise** if it needs to deform or be edited — Quad Remesher / Blender's Remesh + shrinkwrap, or sculpt-then-retopo. For a static prop you can often skip and just decimate.
3. **UV unwrap** the clean mesh and **bake** the generated detail (normal, AO, base colour) from the dense original onto it.
4. **Re-shade** with a proper Principled BSDF (the auto textures are a starting point).
5. **Optimise + export** through the normal pipeline (Draco/Meshopt, KTX2, LOD) — see blender-pipeline.md / realtime-3d-web.md.

AI texturing also works on an existing mesh: text-to-texture (Meshy/Tripo texture mode, or Stable-Diffusion-based projection) paints PBR onto a model you already have.

## Real-world capture (when it must match a real object)

To reproduce a physical object you can photograph:
- **Photogrammetry** (COLMAP / RealityCapture / Metashape): Structure-from-Motion estimates camera poses → dense point cloud → mesh + textures. Gives a normal, editable, measurable mesh that fits straight into the retopo/texture pipeline. Best when you need explicit geometry.
- **Gaussian Splatting**: optimised splats give photoreal, fast real-time viewing but are NOT a normal mesh; convert to mesh if you need editable geometry. Best for photoreal scene viewing/capture speed.
Pair this with the "source an existing CAD/3D model" rule (fusion360.md / the skill) — if a real model already exists, download it; if it's a physical object, capture it; only model from scratch as a last resort.

## Why this makes the skill fast and worth using

A base asset in 20-60 seconds (or self-hosted for free) instead of an hour of blocking-out, then refined with the same closed-loop discipline, beats hand-modelling on both speed and quality for everything organic. Reserve hand-modelling for precise CAD (Fusion) and for surgical edits to generated/captured bases.
