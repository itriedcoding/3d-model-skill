---
name: 3d-model-generator
description: Create, modify, inspect, validate, and export real 3D models and assets from natural-language requests. Use when an AI agent is asked to build, design, sculpt, generate, repair, resize, recolor, or prepare a 3D object, mesh, product mockup, game prop, architectural object, or 3D-printable part, including GLB, glTF, OBJ, STL, and PLY deliverables. Also use when a user asks for a model preview, turntable, or multi-angle render.
---

# 3D Model Generator

Create useful, inspectable, downloadable 3D assets—not merely pictures of objects. A successful task produces geometry that can be opened in a 3D application, a preview that makes the result easy to review, and a concise explanation of what was delivered.

The central production loop is:

**Understand → plan → build → validate → render → inspect → revise → export → report.**

Never claim that a model is complete merely because a script ran. Confirm that the expected files exist, inspect the preview when rendering is available, and report limitations honestly.

---

## 1. When to use this skill

Use this skill when the user asks to:

- Create, model, build, sculpt, or generate a 3D object.
- Make a product mockup, prop, furniture item, vehicle, character, building, environment, or mechanical part.
- Produce a `.glb`, `.gltf`, `.obj`, `.stl`, or `.ply` file.
- Modify an existing mesh: resize, recolor, add details, repair, simplify, or split it.
- Prepare a model for a game engine, 3D viewer, AR workflow, or 3D printer.
- Create a multi-view render, product-style preview, or turntable animation.

Do not use this skill for:

- A 2D illustration, logo, poster, or ordinary image-generation request.
- A CAD drawing that must be delivered as DXF/DWG unless a suitable CAD workflow is explicitly available.
- An interactive Three.js/WebGL website. That is a separate web-development task.
- A request that only asks for an explanation of 3D modeling concepts.

If the user asks for a rendered image only, do not silently substitute a downloadable mesh. If they ask for a mesh, do not deliver only a rendered image.

---

