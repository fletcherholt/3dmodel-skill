# 3dmodel

A Claude Code skill for making 3D models and interactive 3D viewers from a coding agent.

It captures a repeatable way of working in 3D without being able to "see" a scene by intuition: generate geometry, render it, look at the image, critique it, fix the worst defect, repeat. Alongside the method it carries the practical knowledge that makes the output good rather than mocked up: Blender headless modelling, Three.js web delivery, realistic lighting and materials, and the UI patterns for interactive product, exploded and x-ray views.

## What is inside

- `SKILL.md` the skill itself: when to use it, the closed feedback loop, choosing a delivery target, realism techniques, interactive viewer patterns, and the pitfalls worth knowing before you hit them.
- `references/realtime-3d-web.md` reusable Three.js patterns with code: a scene skeleton with shadows and environment lighting, a spherical camera fly, an x-ray reveal, fixed-lane callouts with leader lines, colour-coded flow lines, the hide-the-overlay-while-moving rule, and click versus drag.
- `references/blender-pipeline.md` headless Blender modelling, materials that survive glTF export, GLB export, and the export gotchas that otherwise cost an afternoon.

## Install

Copy the folder into your Claude Code skills directory:

```
cp -r 3dmodel ~/.claude/skills/3dmodel
```

Then invoke it with `/3dmodel`, or just ask Claude to model or recreate something in 3D, build a web 3D viewer, make an exploded or x-ray view, export a GLB, or improve how a real-time scene looks.

## The short version

1. Decide the target first. A reusable asset or a photoreal still points to Blender and a GLB. An interactive web viewer points to Three.js where you control everything.
2. Always work to a reference and match proportion, colour, layout and the small identifying details. Looking like a mockup usually means flat surfaces, no bevels, no shadows between parts, and wrong proportions.
3. Close the loop every pass. Render or screenshot, read the image, fix one thing, repeat. Never ship 3D you have not looked at.
4. For realism in real time the biggest wins are soft shadow maps, bevelled edges, environment lighting, ACES tone mapping, correct PBR, and baked texture detail.
5. For interactive viewers keep labels in fixed lanes with leader lines, colour code the connections between parts, and hide the overlay while the view is moving so the lines never swing.

The skill is meant to grow. New techniques and pitfalls learned on later work get folded back in.
