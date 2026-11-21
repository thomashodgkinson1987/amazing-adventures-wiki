---
title: Fix — cnc-ddraw Performance & Cursor Fix
type: fix
status: confirmed-runtime
updated: 2026-11-21
sources:
  - ../../raw/FINDINGS.md
  - ../../raw/snapshots/ddraw.ini
---

# Fix — cnc-ddraw Performance & Cursor Fix

**Status: RESOLVED (confirmed-runtime — reported working by the user).**

Symptom: the game felt like **~30 fps** with a **flickering mouse cursor**, plus
unreliable fullscreen/windowed behaviour, under Proton.

## The fix

[cnc-ddraw](https://github.com/FunkyFr3sh/cnc-ddraw) is a native reimplementation of the
DirectDraw API with GDI / OpenGL / Direct3D9 renderers. It **replaces Wine's slow
`ddraw → wined3d` path entirely**, which fixed all three complaints.

### What is installed

- `ddraw.dll` in the game folder (32-bit, 409,600 bytes,
  SHA-256 `7fa0770e930f21640241c0f375bf8f422cb5303d2e06a30e9162ef0dad955d77`).
- `ddraw.ini` (cnc-ddraw config; the default `[ddraw]` section is used — no game-specific
  section required). Snapshot: [`raw/snapshots/ddraw.ini`](../../raw/snapshots/ddraw.ini).
- Steam → game → Properties → **Launch Options**:

  ```
  WINEDLLOVERRIDES="ddraw=n,b" %command%
  ```

  (`n,b` = try **n**ative `ddraw.dll` first, fall back to **b**uiltin.)

### Why it works here

The DRM wrapper launches the real game as a **child process whose working directory is
the game folder** (see [drm-wrapper.md](../architecture/drm-wrapper.md)), so Windows/Wine's
DLL search finds the native `ddraw.dll` here and the override forces it to be used.

### Current `ddraw.ini` values (non-comment)

```
[ddraw]
width=800            height=600
fullscreen=false     windowed=true
maintas=false        maxfps=-1
vsync=false          adjmouse=false
shader=Shaders\interpolation\catmull-rom-bilinear.glsl
renderer=auto        border=true
resizable=true       d3d9_filter=2
center_window=1      maxgameticks=0
limiter_type=0       minfps=0
nonexclusive=true    singlecpu=true
fixchilds=2          hook=4
... (plus cnc-ddraw's large built-in per-game compatibility database)
```

Notes (confirmed-static):

- `renderer=auto` resolves to **Direct3D9 → DXVK → Vulkan** (falls back to OpenGL/GDI).
  Full render-path discussion: [rendering-pipeline.md](../architecture/rendering-pipeline.md).
- The referenced `Shaders\...glsl` file is **not present** (no `Shaders/` dir); it only
  matters for the OpenGL renderer, so it is harmless with `renderer=auto`.
- `windowed=true` / `fullscreen=false` starts windowed/borderless; **Alt+Enter** toggles
  fullscreen (`keytogglefullscreen=0x0D`).
- Useful tuning knobs: `renderer=gdi|opengl|direct3d9`, `maxfps`, `maxgameticks`,
  `vsync`, `width`/`height`, `windowed`.

### To revert

Remove the launch option and delete `ddraw.dll` + `ddraw.ini` from the game folder.

## Related

- [rendering-pipeline.md](../architecture/rendering-pipeline.md) — why the default path
  is slow.
- [proton-launch-options.md](proton-launch-options.md) — the launch options in full.
- [inner-game-payload.md](../architecture/inner-game-payload.md) — the hard-coded driver.
- [Tool — amazing-adventures-toolkit](../tooling/amazing-adventures-toolkit.md) —
  installs/removes this fix (and the splash patch) automatically.
