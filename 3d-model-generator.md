# SKILL.md

\---

name: 3d-model-generator
description: Generate complete, real 3D models (GLB, OBJ+MTL, STL) from a text description using only Python and numpy - no three.js, Blender, or web viewer needed - then render a multi-angle preview image to check the result. Use this skill whenever the user asks Claude,opencode,cursor,antigravity, any ai agents to make, build, model, design, sculpt, or generate a 3D model, 3D object, 3D asset, mesh, 3D-printable part, game prop, product mockup in 3D, or asks for a .glb/.obj/.stl/.gltf file, even if they just say "make me a 3D X", "model a X", or "I want to 3D print a X". Also use it to modify, recolor, resize, or fix an existing mesh this skill produced. Do NOT use it for 2D images, CAD drawings (DXF/DWG), or for building interactive three.js/WebGL pages.
---

# 3D Model Generator

Build real, downloadable 3D models by writing a short Python script that assembles watertight
triangle meshes from primitives, then exporting them. The loop is: **plan -> script -> validate ->
look at the render -> fix -> export**. Looking at the render is the step that separates a decent
model from a broken one, so never skip it.

Everything lives in `scripts/` next to this file (numpy + Pillow only, both preinstalled):

* `scripts/mesh3d.py` - primitives, transforms, validation, GLB/OBJ/STL export
* `scripts/render.py` - software renderer (contact sheet, single view, turntable GIF)
* `references/recipes.md` - how to model common object families (furniture, vehicles, characters, architecture, organic shapes, hollow parts, printable parts). Read it when the object isn't a trivial combination of primitives.
* `references/examples/` - three complete, working scripts (`mug.py`, `cottage.py`, `robot.py`). Skim the one closest to the request and copy its header.

## Workflow

### 1\. Plan before coding (2 minutes of thought saves 3 rewrites)

* Pick **real-world dimensions** and a unit (mm is the default; say it in the final message since STL has no unit field). Proportions drawn from real sizes look right; guessed ones look toy-like.
* **Decompose** the object into parts that map to primitives: round things -> `lathe`/`cylinder`/`sphere`; blocky things -> `box`/`rounded\_box`; outlines -> `extrude`; bent rods/handles/pipes -> `tube`; terrain -> `heightfield`. Write the part list as a comment at the top of the script.
* Decide colors/materials per part (a palette of 3-5 colors reads as designed; see recipes).
* If the request is vague ("a cool spaceship"), pick a distinctive concept and go; mention your assumptions afterwards. Only ask a question if the answer changes the whole approach (e.g. 3D-print vs. game asset).

### 2\. Write the build script

Start from the header of an example so imports work wherever the skill is installed:

```python
import os, sys
for \_p in ("/mnt/skills/user/3d-model-generator/scripts", "/mnt/skills/examples/3d-model-generator/scripts"):
    if os.path.isdir(\_p): sys.path.insert(0, \_p); break
from mesh3d import \*

S = Scene("my\_object")
S.add(lathe(\[...]), "body", color="#c0392b", metallic=0.1, roughness=0.4)
S.ground()                         # centre XY, rest on z=0
print(S.validate\_report())
print(S.export("/mnt/user-data/outputs/my\_object"))   # .glb .obj .mtl .stl + \_preview.png
```

If the skill sits somewhere else, `find / -name mesh3d.py 2>/dev/null` and use that directory.

**Conventions (get these right and parts line up on the first try):** Z is up, right-handed, the
object's front faces **-Y**, units are your choice. Transforms return *new* meshes, so
`leg = cylinder(...)` can be reused with different `.translate()` calls. Build one side and loop over
`for sx in (-1, 1)` to mirror.

**API cheat sheet**

|Need|Call|
|-|-|
|Block / bevelled block|`box((x,y,z))`, `rounded\_box((x,y,z), radius, segments=6)`|
|Round solids|`cylinder(r, h, seg, radius\_top=)`, `cone(r,h)`, `sphere(r)`, `capsule(r, straight\_len)`, `torus(R, r)`|
|Anything round with a profile|`lathe(\[(r,z),...])` - solid on the LEFT of the path: up the outside, down the inside|
|Flat outline pushed up / side profile across a width|`extrude(poly, height, z0)` (for side profiles chain `.rotate('x',90).rotate('z',90)`, see recipes); `extrude\_ring(outer, inner, height)` for one hole (equal point counts)|
|Pipe / handle / cable along a curve|`tube(path3d, radius\_or\_taper\_list)`; make the path with `catmull\_rom(pts)` or `bezier(ctrl)`|
|Terrain / relief|`heightfield(Z\_array, size, base\_z)`|
|2D outlines|`circle\_poly`, `ngon\_poly`, `rect\_poly`, `rounded\_rect\_poly`, `arc\_pts`|
|Place|`.translate(x,y,z)` `.rotate('x'|
|Combine|`merge(\[a,b])` or `a + b` (one part, one material); separate `S.add` calls for different materials|
|Scene helpers|`S.ground()`, `S.scale\_to\_height(h)`, `S.validate\_report()`, `S.export(path)`|

**Materials:** `color` hex, `metallic` 0-1, `roughness` 0-1, `alpha` <1 for glass/clear plastic,
`emissive` hex for glowing parts. Colors you pass are sRGB; export converts correctly.

**No boolean/CSG.** Make holes and cavities by construction instead: `lathe` with an inner wall
(hollow vessels), `extrude\_ring` (washers, frames), assembling slabs around an opening (windows and
doors in walls), or a slightly-inset transparent/dark part that reads as a recess. Overlapping closed
parts are fine for viewing and for slicers; just bury joints (let a handle end inside the wall) so no
gaps show.

### 3\. Validate

`S.validate\_report()` prints per part: `OK` means watertight with outward normals. A `!!` line
means a construction bug: usually a `lathe` profile walked the wrong way (solid on the right ->
flip the point order), an `extrude` polygon that self-intersects, or a negative `scale`
you applied by hand to `v` instead of using `.scale()`. Fix the cause, don't ignore it - STL
slicers and many renderers choke on open/inverted meshes.

### 4\. Look at it (mandatory)

`S.export(...)` writes `<name>\_preview.png`: front / right / top / 3-4 views with dimensions. Open it with
`view` and critique it like a reviewer: Are proportions believable? Anything floating, intersecting
wrongly, missing, or facing backwards? Is it recognisably the thing? Is the silhouette interesting
from the front view? Fix and re-run until it passes. Budget 1-3 iterations. For a close-up use
`render\_view(S, path, azim=-40, elev=20, size=900)`; for a spin use `render\_turntable(S, "spin.gif")`
(both in `render.py`). Remember the preview is an orthographic software render; the GLB will look
more polished in a real viewer (proper PBR lighting).

### 5\. Deliver

Save outputs in `/mnt/user-data/outputs/` and call `present\_files` with the **.glb first** (opens in
Windows 3D Viewer, macOS Quick Look/Preview, Blender, Godot, Unity, online viewers), then the preview
PNG, then `.obj`+`.mtl` and `.stl`. Keep the closing message short: what it is, real-world size and unit,
parts count, which file to use for what (GLB = view/game/AR, STL = 3D printing, OBJ = Blender/legacy),
and any assumption you made. If the user wants to print it, mention parts overlap/are separate bodies and
that wall thickness should be checked against their printer (>=1.2 mm is typical for FDM).

## Quality bar: what makes a model look professional rather than like a box stack

* **Bevel everything visible.** Real objects have no razor edges. Prefer `rounded\_box` over `box` for anything a hand touches or the eye lingers on; use small lathe fillets.
* **Silhouette first.** Vary the profile (taper, flare, waist, overhang). Symmetric boring cylinders are the #1 tell.
* **Secondary detail earns realism:** seams, buttons, vents, feet, rims, screws, handles, panel insets, a base or stand. Add 3-6 details, not 60.
* **Segment budget:** 48-72 segments for big round parts, 16-24 for small ones; `rounded\_box` segments 3-4 for small trim. Aim for < 100k triangles total (the renderer takes seconds, viewers take milliseconds).
* **Consistent scale and a grounded pose:** call `S.ground()`; give things stands/feet that touch z=0.
* **Material variety:** metallic accents against matte bodies, glass with `alpha`, emissive lights/eyes/screens.
* **Name parts meaningfully** (`"wheel\_fl"`, `"lampshade"`); users open the file in editors and will thank you.

## Fallback when there is no code sandbox

If code execution is unavailable, write the geometry directly as OBJ text for simple shapes (vertex
lines `v x y z`, faces `f a b c` counter-clockwise seen from outside) inside a fenced code block or a file,
keeping it to boxes, prisms and low-poly forms - hand-writing curved meshes is error-prone, so say so.

## Troubleshooting

* *Model looks inside-out or black in a viewer*: validate says `outward=False` -> flip with `.flipped()` or fix the lathe direction.
* *Rendering shows faint seams or dark lines on curved parts*: raise `segments`; the preview uses flat per-face colour with smoothed normals.
* *Parts don't meet*: print `part.mesh.bounds()` for the two parts and compare; remember lathe/cylinder are centred at the origin along Z (cylinder spans `-h/2..+h/2`, lathe spans exactly the z values you gave).
* *Extrusion points sideways*: `extrude` builds along +Z from the 2D XY polygon; rotate afterwards (`.rotate('x', 90)` sends +Z to -Y and +Y to +Z).
* *A tall thin part renders tiny*: the sheet frames the whole scene; use `render\_view` with the part alone for detail checks.



\---

# recipes.md

# Modeling recipes

Each snippet assumes `from mesh3d import \*` and `S = Scene("x")`. Units mm, Z up, front = -Y.
Table of contents: [Round things](#round-things-lathe) - [Glass and transparency](#glass-and-transparency) -
[Furniture](#furniture) - [Vehicles](#vehicles) - [Gears and mechanical parts](#gears-and-mechanical-parts) -
[Architecture](#architecture) - [Organic: trees, terrain, creatures](#organic-trees-terrain-creatures) -
[Printable parts](#printable-parts) - [Smooth profiles](#smooth-profiles)

## Round things (lathe)

Anything with rotational symmetry is one `lathe` call. Walk the (r, z) profile with the solid on your
left: up the outside, over the lip, down the inside, finish on the axis (r=0).

```python
# Vase: smooth silhouette through control points (bottom -> lip), then walk back down the inside
outer = catmull\_rom(\[(30,0),(46,40),(52,110),(34,200),(22,250),(28,275)], 8)
inner = \[(max(r-4, 1), z) for r, z in reversed(outer) if z >= 12]       # wall thickness 4, floor at z=12
vase = lathe(\[(0,0)] + outer + inner + \[(0,12)], segments=72)
S.add(vase, "vase", color="#2f6f8f", roughness=0.15)
```

Chess pawn / bottle / lamp base / column: same idea, more points. Wheels: a tyre is a lathe of a closed cross-section loop,
`lathe(\[(R + t\*math.cos(a), t\*math.sin(a)) for a in np.linspace(0, 2\*math.pi, 24, endpoint=False)], 48, closed=True)`
(R = distance from axle to tyre centre line, t = half the tyre section), then `.rotate('x', 90)` to stand it up.

## Glass and transparency

`alpha<1` parts are drawn blended and exported as glTF BLEND materials. Model glass as a *thin shell*
(outer + inner wall) so it looks like glass, not a solid block.

```python
glass = lathe(\[(0,0),(28,0),(30,4),(34,60),(52,110),(60,160),(58,175),(55,175),(56,160),(48,112),(30,64),(26,8),(0,6)], 72)
S.add(glass, "glass", color="#dff3ff", alpha=0.35, roughness=0.03)
S.add(cylinder(52,1,72).translate(0,0,120), "wine", color="#6d0f1f", alpha=0.85, roughness=0.1)
```

## Furniture

Boxes + `rounded\_box` + cylinders. Real dimensions: dining table 750 high, seat height 450, desk 740, sofa seat 430.

```python
def chair(w=450, d=450, seat\_h=450, back\_h=450):
    c = Scene("chair")
    c.add(rounded\_box((w, d, 40), 10).translate(0, 0, seat\_h), "seat", color="#c58b4f", roughness=0.6)
    for sx in (-1, 1):
        for sy in (-1, 1):
            c.add(cylinder(18, seat\_h, 24, radius\_top=14).translate(sx\*(w/2-30), sy\*(d/2-30), seat\_h/2), "leg", color="#4a3322")
    c.add(rounded\_box((w, 30, back\_h), 10).translate(0, d/2-15, seat\_h+back\_h/2+20), "back", color="#c58b4f")
    return c
```

Reuse a Scene function by copying its parts: `for p in chair().parts: S.add(p\["mesh"].translate(x,y,0), p\["name"], color=p\["color"])`.

## Vehicles

Body = `extrude` of a side profile (gives a believable silhouette in one call; real car \~4200 x 1800 x 1400 mm), then window glass as slightly inset `alpha` slabs, wheels = tyre lathe + a smaller metallic cylinder for the rim.

```python
side = catmull\_rom(\[(0,350),(300,420),(1000,560),(1500,900),(2700,950),(3400,620),(4100,520),(4200,380)], 6)  # (x, z)
side += \[(4200,300),(0,300)]                         # close the underside
body = extrude(side, 1800).rotate('x', 90)           # profile XY -> XZ; extrusion runs along -Y
body = body.translate(0, -body.center()\[1], 0)
S.add(body, "body", color="#b3202a", metallic=0.6, roughness=0.3)
```

Wheels: `cylinder(330, 220, 48).rotate('x', 90)` for tyres, smaller lighter cylinder for the rim, four copies by a double loop; axle height = wheel radius.

## Gears and mechanical parts

```python
def gear\_outline(teeth=20, module=4):
    rp = module\*teeth/2; ro = rp + module; rr = rp - 1.25\*module
    pts, step = \[], 2\*math.pi/teeth
    for i in range(teeth):
        for f, r in ((0.0, rr), (0.25, ro), (0.5, ro), (0.75, rr)):
            pts.append((r\*math.cos((i+f)\*step), r\*math.sin((i+f)\*step)))
    return pts
outline = gear\_outline(20)
hub\_hole = circle\_poly(8, len(outline))             # SAME point count as the outline
S.add(extrude\_ring(outline, hub\_hole, 12), "gear", color="#9aa4ae", metallic=0.9, roughness=0.35)
```

`extrude\_ring(outer, inner, h)` needs equal point counts (see `hub\_hole` above). Bolts: `ngon\_poly(r, 6)` extruded for the head + a cylinder shank. Springs/coils: `tube` along a helix `\[(R\*cos(t), R\*sin(t), p\*t/(2\*pi)) for t in np.linspace(0, 2\*pi\*turns, 20\*turns)]`.

## Architecture

Walls as boxes, roofs as extruded triangles, openings by *assembly*: to cut a door into a wall, build the wall from a lintel box above the door plus two side boxes, instead of subtracting. Floors/slabs 100-200 mm, walls 200-300 mm, door 900x2100, window sill 900, storey height 2700. Columns: `lathe` with a wider base/capital. Arches: `tube` is wrong; extrude a polygon made from `arc\_pts` (half-circle on top of a rectangle).

## Organic: trees, terrain, creatures

```python
# Tree: tapered trunk along a bent path + clustered leaf blobs
trunk = tube(catmull\_rom(\[(0,0,0),(15,10,400),(-10,0,800),(0,-20,1200)], 8), \[70,55,40,25], 16)
S.add(trunk, "trunk", color="#6b4a2f", roughness=1)
rng = np.random.default\_rng(3)
for \_ in range(9):
    c = rng.normal(0, 260, 3)\*\[1,1,0.6] + \[0, -20, 1450]
    S.add(sphere(rng.uniform(240, 340), 24, 12).translate(c), "leaves", color="#3f8f46", roughness=0.9)

# Terrain: smooth noise from summed sines, then a solid block
x = np.linspace(0, 6, 80); X, Y = np.meshgrid(x, x)
Z = 120\*np.sin(X\*1.3)\*np.cos(Y\*1.1) + 60\*np.sin(X\*3.1+Y\*2.3) + 200
S.add(heightfield(Z, (4000, 4000), base\_z=0), "terrain", color="#7a8f55", roughness=1)
```

Creatures/characters: ellipsoids (`sphere(1).scale(sx,sy,sz)`), capsules for limbs, spheres for joints, `tube` for tails/tentacles with a tapering radius list, symmetric loop for eyes/ears/legs. Keep the head 20-30% oversized for charm.

## Side-profile extrusion (stands, brackets, ramps, L-shapes, cross-sections)

Draw the cross-section as (depth, height) points, extrude it across the width, and turn it so depth runs along Y
and width along X. This chain is verified: `extrude(profile, width).rotate("x", 90).rotate("z", 90)`.

```python
T, DEPTH, WIDTH = 4, 90, 80
bracket = extrude(\[(0,0),(DEPTH,0),(DEPTH,T),(T,T),(T,60),(0,60)], WIDTH).rotate("x", 90).rotate("z", 90)
S.add(bracket, "bracket", color="#2b6cb0")
```

Orientation check after the chain: profile u -> +Y (front is y=0), profile v -> +Z, extrusion -> +X (centred only after `S.ground()`).
Walk the polygon in any order; `extrude` fixes the winding.

## Printable parts

* Make it one connected solid or a few flat-bottomed bodies; call `S.ground()` so it sits on z=0.
* Walls >= 1.2 mm (FDM) / 0.8 mm (resin); overhangs under 45 degrees; avoid features smaller than 0.4 mm.
* Overlap joints by 1-2 mm rather than merely touching. Export STL (Z-up, mm); `validate\_report` should show all `OK`.
* Hollow containers: `lathe` with an inner wall; add a lid as a separate body with a rim 0.3 mm smaller than the opening for fit.

## Smooth profiles

`catmull\_rom(points, n\_per\_seg)` passes through your points (good for vases, bottles, car sides);
`bezier(ctrl, n)` gives one smooth curve shaped by control points (good for a single swoosh). Chain them with
list addition. Keep r >= 0 in lathe profiles and end on the axis (r = 0) for closed solids.

