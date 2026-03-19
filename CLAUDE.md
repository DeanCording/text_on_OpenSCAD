# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

An OpenSCAD library (`text_on.scad`) for placing 3D text on geometric surfaces. The entire library is a single `.scad` file. There is no build system, package manager, or test runner.

## Development Commands

**Render example PNGs** (requires OpenSCAD binary at the path in the script):
```bash
cd examples && bash make_example_pngs.sh
```

**Render a single example** (adjust camera/path as needed):
```bash
openscad -o examples/text_on_cylinder.scad.png --enable=text --camera=0,0,0,75,0,25,150 examples/text_on_cylinder.scad
```

**Create a release zip:**
```bash
cd development && bash make_release_zip.sh
```

**Usage in a project** — copy `text_on.scad` next to the user's file and add:
```scad
use <text_on.scad>
```

## Architecture

All public modules eventually call `text_extrude()`, which wraps OpenSCAD's `linear_extrude()`+`text()`. The call hierarchy is:

- `text_on_sphere()` → `__internal_text_on_sphere_helper()` → `text_extrude()`
- `text_on_cylinder()` → `text_on_circle()` (top/bottom face) or `__internal_text_on_cylinder_side()` → `text_extrude()`
- `text_on_circle()` → `text_extrude()` (one call per character)
- `text_on_cube()` → `text_extrude()` (single call with full string)

Modules prefixed with `__internal_` are private helpers; do not call them directly.

### Textmetrics

The library detects OpenSCAD version at runtime:
```scad
internal_tm_avail = version()[0] > 2021;
```

When `internal_tm_avail` is true, OpenSCAD's `textmetrics()` built-in is used for precise proportional-font character positioning. When false, the legacy `internal_space_fudge = 0.80` approximation is used. Every character-positioning calculation has both branches (`tm_*` functions for textmetrics, plain functions for legacy).

### Key Global State

- `debug = true` — left enabled; `echo()` calls throughout are commented out in most places
- `default_font = "Liberation Mono"` — monospace default; proportional fonts work via textmetrics
- `default_buffer_width = 0` — letter thickening via `offset()` before extrusion; exposed as `buffer_width` param

### Coordinate Conventions

- All faux-objects are treated as `center=true`; the `cylinder_center` param handles the exception
- `eastwest` rotates around Z-axis; `northsouth` tilts on sphere; `updown` shifts vertically on cylinder/cube
- `rotate` parameter spirals text helically on curved surfaces (not simple 2D rotation)
- `halign`/`valign` are accepted but **not supported** on sphere/cylinder/circle — a warning is echoed

### Character-by-Character Rendering

`text_on_sphere`, `text_on_circle`, and `__internal_text_on_cylinder_side` render one character at a time in a `for` loop, computing each character's angular position on the surface. `text_on_cube` passes the full string to `text_extrude()` directly.
