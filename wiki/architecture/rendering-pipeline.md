---
title: Architecture — Rendering Pipeline (SDL 1.2 → DirectDraw → Wine)
type: architecture
status: confirmed-static
updated: 2026-11-20
sources:
  - ../../raw/FINDINGS.md
---

# Rendering Pipeline

The inner game ([inner-game-payload.md](inner-game-payload.md)) renders through
**DirectDraw only** — there is **no Direct3D** in it — using **SDL 1.2 statically
linked** and the SDA Game Framework.

## The default (slow) path

The game hard-codes its SDL video driver, so the default render path under Proton is:

```
game → SDL 1.2 "directx" driver → DirectDraw → Wine wined3d → OpenGL → GPU/compositor
```

Problems (confirmed-runtime symptom, confirmed-static cause):

- Extra GPU round-trip through wined3d/OpenGL for what is really a 2D blitter.
- **vsync-locked** — feels like ~30 fps.
- **hardware-cursor flicker.**

## The fix path (cnc-ddraw)

Dropping **cnc-ddraw** (`ddraw.dll` + `ddraw.ini`) into the game folder replaces Wine's
slow `ddraw → wined3d` path with a native DirectDraw reimplementation that can render
via GDI / OpenGL / **Direct3D9 → DXVK → Vulkan**. This fixed smoothness, cursor and
fullscreen behaviour. See [cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md).

## Relevant knobs

From the installed `ddraw.ini` (snapshot:
[ddraw.ini](../../raw/snapshots/ddraw.ini)):

| Key | Value | Meaning |
|---|---|---|
| `renderer` | `auto` | tries Direct3D9/OpenGL, falls back to GDI |
| `width` / `height` | `800` / `600` | requested game resolution |
| `fullscreen` | `false` | start windowed |
| `windowed` | `true` | windowed/borderless |
| `maxfps` | `-1` | real rendering rate = screen rate (does **not** cap game speed) |
| `maxgameticks` | `0` | game-speed limit (uncapped) |
| `vsync` | `false` | no vsync (can cause tearing; vsync adds input lag) |
| `hook` | `4` | hook mode |
| `fixchilds` | `2` | child-process handling |
| `nonexclusive` | `true` | non-exclusive fullscreen |
| `singlecpu` | `true` | affinity to one CPU |
| `shader` | `Shaders\interpolation\catmull-rom-bilinear.glsl` | only used by the OpenGL renderer |
| `keytogglefullscreen` | `0x0D` | **Alt+Enter** toggles fullscreen |

> **Note (confirmed-static):** the referenced `Shaders\...glsl` file is **not present**
> (there is no `Shaders/` directory in the game folder). It only matters for the OpenGL
> renderer, so it is harmless with `renderer=auto`.

Useful knobs if tuning further: `renderer=gdi|opengl|direct3d9`, `maxfps`,
`maxgameticks`, `vsync`, `width`/`height`, `windowed`.

## Alternatives (kept for reference)

1. **`WINE_D3D_CONFIG=renderer=gdi`** (launch option) — modern replacement for the
   removed `DirectDrawRenderer=gdi`. wined3d reads `renderer` from
   `HKCU\Software\Wine\Direct3D` (or per-app `...\AppDefaults\<exe>\Direct3D`) or the
   `WINE_D3D_CONFIG` env var; values `gl` / `vulkan` / `gdi` (== `no3d`). Keeps SDL on
   DirectDraw but blits via software/GDI, removing the GL round-trip.
2. **`WINEDLLOVERRIDES="ddraw=d"`** — disable ddraw so the `directx` SDL driver fails,
   triggering the game's own fallback to `windib` (GDI). No patching, but requests
   fullscreen and windib may reject the mode on some setups.
3. **In-game Options** — uncheck "Use Hardware Acceleration", and/or turn off Full
   Screen (`fullscreen=false` in `config.txt`). In-game checkboxes include
   `checkboxfullscreen`, `checkboxacceleration` (`ID_OPTIONS_HACC` = "Use Hardware
   Acceleration"), `checkboxrelaxed`, `checkboxhints`.
4. **Wine virtual desktop** — `protontricks 3530 winecfg` → Graphics → Emulate a virtual
   desktop.

See [proton-launch-options.md](../fixes/proton-launch-options.md) for the actual launch
options in use.

## Related

- [inner-game-payload.md](inner-game-payload.md) — where the driver is hard-coded.
- [cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md) — the applied fix.
- [drm-wrapper.md](drm-wrapper.md) — why the child process picks up a local `ddraw.dll`.
