# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Tile Grid — a browser tool: load a photo, split it into a grid of cells, hide cells by clicking, and perspective-warp the whole grid by dragging four corner handles, then export a PNG. UI text and `PLAN.md` are in Russian; keep new UI strings in Russian.

- `tiles.html` — the main tool. Single self-contained file, plain WebGL1 + Canvas 2D, no libraries.
- `tiles-three-prototype.html` — earlier, separate prototype on three.js r128 (CDN) with camera/video input, 3D rotation, per-column depth/jitter and video recording. Not the main tool; don't merge its approach into `tiles.html` unless asked.
- `PLAN.md` — the spec and decisions for `tiles.html` (features, technical design, possible future work such as undo, JSON project save, per-column shift, transparent export, animation/video).

## Running

No build, no server, no tests, no package manager. Open the file directly in a browser (`open tiles.html`). Verify changes manually in a browser: photo load (button and drag & drop), clicking/painting cells after a perspective warp, PNG export, and the narrow-screen layout (`@media (max-width:700px)`).

## Architecture of `tiles.html`

Everything lives in one `<script>`; global state is the object `S` (photo, `cols`/`rows`, `hidden` as a `Set` of `"c,r"` keys, `corners` `[TL, TR, BR, BL]` in frame px, `H`/`Hinv`, settings).

**Coordinate spaces.** The grid is defined in (u, v) ∈ [0,1]². *Frame px* is the export resolution (`S.FW`×`S.FH`, long side `LONG_SIDE` = 2400), set by `setupFrame()` from the chosen format. The on-screen size is a CSS scale of the frame computed in `layout()`; `toFrame()` converts pointer events back to frame px.

**Homography is the core.** `squareToQuad(corners)` builds a 3×3 row-major `H` mapping the unit square to the corner quad; `updateH()` recomputes `H` and `Hinv` on every corner move.
- Rendering: the vertex shader multiplies `(u, v, 1)` by `H` and outputs the result with `w = p.z`, so the GPU does perspective-correct texture interpolation (no diagonal seams). `H` is transposed to column-major when uploaded as `u_H`.
- Hit-testing: `cellAt()` projects the pointer through `Hinv` to (u, v), then `floor(u·cols), floor(v·rows)`.
- Because straight lines stay straight under a homography, the 2D overlay draws grid lines/hover as segments between `project(H, …)` nodes.

**Two stacked canvases.**
1. `#gl` (WebGL, `preserveDrawingBuffer`) draws background, visible cells and outlines. PNG export reads this canvas directly, so export == preview. Anything that must appear in the export goes here.
2. `#ov` (2D) draws helper UI only — dashed grid, hover highlight, corner handles. Never exported. It also receives all pointer events.

**Mesh vs. uniforms.** `buildMesh()` rebuilds the cell and stroke vertex buffers (in uv space) and must be called whenever hidden cells, gap, outlines or grid size change. Corner drags only change `u_H` — no rebuild. Gap and stroke widths are specified in frame px and converted to uv via `S.gridW`/`S.gridH` (the size of the unwarped grid from `resetCorners()`). Strokes are bands centred on cell edges so shared edges at zero gap aren't doubled.

**Rendering is batched** through `render()` (one `requestAnimationFrame`); call it after state changes instead of drawing directly.

Other invariants: changing cols/rows clears `S.hidden`; changing the format calls `setupFrame()`, which resets corners; photos larger than `MAX_TEX` (or the GPU's `MAX_TEXTURE_SIZE`) are downscaled via an offscreen canvas before upload; corner handles are only draggable while the helper is shown.
