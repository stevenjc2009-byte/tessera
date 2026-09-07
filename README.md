# Tessera

A Nintendo 3DS homebrew app for making pixel art in 2D and building coloured 3D
models out of hand-drawn polygon faces, with a public gallery for sharing them.

A *tessera* is one of the small coloured tiles in a mosaic. Pixel tiles and flat
faces joined into a whole is the app.

## Status

**Design approved, no code yet.** The full design lives at
[`docs/superpowers/specs/2026-09-07-tessera-design.md`](docs/superpowers/specs/2026-09-07-tessera-design.md).

## What it does

- **Pixel art editor** — pencil, eraser, bucket, picker, line, rect, ellipse,
  undo. Constrained palettes (PICO-8, DawnBringer 16/32, Game Boy, NES), visible
  grid, one pixel per tap, no anti-aliasing.
- **Polygon face editor** — tap points on a grid to build a face of any shape:
  triangle, hexagon, or something completely irregular.
- **Face painting** — every face is its own small pixel canvas. The 2D editor
  *is* the face editor.
- **Model building** — join faces edge to edge, set the fold angle, and see the
  result in stereoscopic 3D on the top screen.
- **Colouring book** — pre-made line art, tap a region to fill it.
- **Export** — PNG to SD, and OBJ + MTL + texture for the 3D models.
- **Gallery** — upload, browse, download and rate creations from the console.

## Layout

The top screen previews; the bottom screen is where everything happens. The
3DS touch panel is single-contact resistive, so there are no gestures — the 3D
view is driven by the circle pad, never by dragging.

## Building

devkitPro / libctru / citro2d / citro3d. Ships as a CIA.

## Server

The gallery server lives in a separate repo:
[tessera-server](https://github.com/stevenjc2009-byte/tessera-server).

## Licence

MIT.
