---
title: Fix — Proton / Wine Launch Options & Alternatives
type: fix
status: confirmed-runtime
updated: 2026-11-20
sources:
  - ../../raw/FINDINGS.md
---

# Proton / Wine Launch Options & Alternatives

This page collects the Steam launch options and Wine/Proton environment knobs relevant to
this game. The **currently active** option is the cnc-ddraw override.

## Active launch options

```
WINEDLLOVERRIDES="ddraw=n,b" %command%
```

- `n,b` = try native `ddraw.dll` first, fall back to builtin. See
  [cnc-ddraw-performance-fix.md](cnc-ddraw-performance-fix.md).
- Works because the DRM wrapper runs the real game as a **child** whose working directory
  is the game folder, so the local `ddraw.dll` is discovered. See
  [drm-wrapper.md](../architecture/drm-wrapper.md).

## Alternatives (documented, not currently used)

| Option | What it does | Notes |
|---|---|---|
| `WINE_D3D_CONFIG=renderer=gdi` | Keeps SDL on DirectDraw but blits via software/GDI, removing the GL round-trip. | Modern replacement for the removed `DirectDrawRenderer=gdi` (Wine 6). wined3d reads `renderer` from `HKCU\Software\Wine\Direct3D`, per-app `...\AppDefaults\<exe>\Direct3D`, or this env var; values `gl`/`vulkan`/`gdi` (== `no3d`). Good fallback if cnc-ddraw is ever removed. |
| `WINEDLLOVERRIDES="ddraw=d"` | Disable ddraw so SDL's `directx` driver fails → game falls back to `windib` (GDI). | Same idea as patching the EXE, with no patching. Risk: requests fullscreen; `windib` may reject the mode on some setups. |
| In-game Options | Uncheck "Use Hardware Acceleration"; turn off Full Screen (`fullscreen=false` in `config.txt`). | The readme's own advice. |
| Wine virtual desktop | `protontricks 3530 winecfg` → Graphics → Emulate a virtual desktop. | Classic fix for old fullscreen 2D games / cursor issues. |
| Direct EXE patch (not recommended) | Edit inner plaintext `SDL_VIDEODRIVER=directx` → `windib`. | The PopCap DRM overlay may integrity-check the executable, and the `WINEDLLOVERRIDES="ddraw=d"` route gives the same result with no risk. |

> **Note:** a `SDL_VIDEODRIVER=…` launch option would **not** work — the game overwrites
> it in `SDL_VideoInit` (confirmed-static). See
> [inner-game-payload.md](../architecture/inner-game-payload.md).

## Related

- [cnc-ddraw-performance-fix.md](cnc-ddraw-performance-fix.md)
- [rendering-pipeline.md](../architecture/rendering-pipeline.md)
- [overview.md](../overview.md)
