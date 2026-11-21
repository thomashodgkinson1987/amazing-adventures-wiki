---
title: Architecture — The Inner Game Payload (real game)
type: architecture
status: confirmed-static
updated: 2026-11-21
sources:
  - ../../raw/FINDINGS.md
  - ../../raw/SPLASHSCREEN-PATCHING.md
---

# The Inner Game Payload (real game)

The **real game** is a second PE appended as an **overlay** inside
[`AmazingAdventures2.exe`](drm-wrapper.md). The wrapper extracts it as
`popcapgame1.exe` and runs it as a child process.

## Overlay facts (confirmed-static)

| Property | Value |
|---|---|
| Overlay start (file offset in `AmazingAdventures2.exe`) | `0x17b1a2` |
| Overlay size | `0x165e00` = 1,465,856 bytes |
| Embedded PE `ImageBase` | `0x400000` |
| Overlay SHA-256 | `18e40b96c95cfd1c4f7ae9df5bc522b76fcb8cfc9bd20586a417604ac7a4569e` |

The overlay is written out **verbatim** (not encrypted/obfuscated): the overlay bytes were
**byte-for-byte identical** to the (now deleted) user capture `copy.exe` (same SHA-256
`18e40b96…569e`). This was re-verified during wiki setup and at runtime (the post-patch
capture matched the patched overlay). See
[binaries-and-hashes.md](binaries-and-hashes.md#verified-captures-removed-2026-11-21).

> `copy.exe` was a **user-made capture** of the runtime-extracted `popcapgame1.exe`,
> taken while the game ran; it was **verified and deleted 2026-11-21** (derivable from the
> EXE). The extracted payload is 100% stock; the shipped Steam binary is
> `AmazingAdventures2.exe` (now carrying the splash patch).

## What the real game is built on (confirmed-static)

- **"SDA Game Framework"** (`gfSDL`, `gfvfs.cpp`, © SDA Software Associates Inc.).
- **SDL 1.2, statically linked.**
- Renders through **DirectDraw** only (`DirectDrawCreate`, `IDirectDrawSurface3`,
  hardware cursor). **There is NO Direct3D anywhere in it.**
- Audio via **`bass.dll`** (BASS SDK 2.x).
- Assets live in **`Resource.dll`** (40 MB data-only PE); see
  [resource-dll-xui.md](resource-dll-xui.md).
- The front-end lives in `.\src\aaApplication.cpp` (path string inside the binary).
- Product/registry name: `Amazing Adventures Around The World`
  → config under `HKCU\Software\PopCap\Amazing Adventures Around The World`
  (framework keys seen: `ScreenMode`, `CustomCursors`, `Test3D\FailureReason`,
  `Test3D\Warning`).

## Why it is slow on Linux (the key code detail)

The inner game's SDL video init **hard-codes the SDL video driver** (inner VA
`0x00474ac0`):

```c
SDL_putenv("SDL_VIDEODRIVER=directx");   // DirectDraw  (string at inner VA 0x52CA74)
if (SDL_VideoInit(...) < 0)
    SDL_putenv("SDL_VIDEODRIVER=windib"); // GDI fallback (string at inner VA 0x52CA8C)
```

Consequences (confirmed-static):

- Any `SDL_VIDEODRIVER=…` in Steam launch options is **overwritten** by the game — you
  cannot switch drivers that way.
- The default path is `game → SDL 1.2 "directx" → DirectDraw → Wine wined3d → OpenGL →
  GPU`, which for old 2D games means an extra GPU round-trip, vsync-locked ("~30 fps"
  feel) and prone to cursor flicker.
- The only built-in fallback is `windib` (GDI), and only if DirectDraw init fails.

Details and the fix: [rendering-pipeline.md](rendering-pipeline.md) ·
[cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md).

## Front-end (splash) state machine

The startup splashes are implemented here (in the extracted game), not in the wrapper.
See [splash-screen-state-machine.md](../fixes/splash-screen-state-machine.md).

## Related

- [drm-wrapper.md](drm-wrapper.md) — the outer wrapper that extracts this payload.
- [va-file-offset-mapping.md](va-file-offset-mapping.md) — how to map its VAs to file
  offsets (the mapping that tripped up an earlier pass).
- [binaries-and-hashes.md](binaries-and-hashes.md).
