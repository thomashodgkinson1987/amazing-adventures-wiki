---
title: Architecture — The PopCap DRM / Steam Wrapper
type: architecture
status: confirmed-static
updated: 2026-11-21
sources:
  - ../../raw/FINDINGS.md
  - ../../raw/SPLASHSCREEN-PATCHING.md
---

# The PopCap DRM / Steam Wrapper

`AmazingAdventures2.exe` (≈3.0 MB) in the game folder is **the stock Steam executable**,
and it is **not the game**. It is a **PopCap DRM / Steam wrapper** (a "two-layer
DRM/packaging structure"). Steam launches this file; it then extracts and runs the real
game as a child process.

- The **stock, shipped** EXE has SHA-256
  `275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3` (restore it any time
  with Steam *Verify integrity of game files*).
- The **active on-disk** `AmazingAdventures2.exe` now carries the **6-byte Option A splash
  patch** (SHA-256 `d920cde4b531d61b377b5feead659ac13e8cb11a88cd9622761c2ed4ea4eb833`) — see
  [splash-patch-option-a.md](../fixes/splash-patch-option-a.md).

## What it contains

**Confirmed-static:**

- A **Lua engine** and the trial/DRM screens (the loose `drm/` folder holds the DRM UI:
  `drm/common/drm.xml`, `drm/custom/drm.xml`, images, fonts, scripts).
- **Steam glue** strings such as `Unable to load Steam.dll` and `SteamSetUser Failed`.
- **No SDL and no DirectDraw rendering** — it is a launcher/DRM shell only.

## How it launches the game

**Confirmed-static** from static analysis of the outer PE:

- It writes out `popcapgame1.exe` / `popcapgame2.exe` / `popcapgame3.exe` and runs one as
  a child via `CreateProcessA` with the command line:

  ```
  "<path-to-popcapgameN.exe>" -changedir="<dir>"
  ```

- `CreateProcessA` call sites at file offsets `0x7bdb` and `0x10458f`
  (i.e. VAs `0x407bdb`, `0x50458f`).
- The strings are **UTF-16LE** in `.rdata`:
  `popcapgame1/2/3.exe`, `-changedir="`, `Unable to extract game executable.`,
  `Unable to locate game content.`, plus `..\PopCap Games\`, `_DRMMutex`,
  `Fatal Error`, `LastTryFile`.

> **Confirmed-runtime (2026-11-20):** the extraction is **verbatim**. A copy of the
> extracted `popcapgame1.exe` captured while the patched game was running
> (`copy-after-splash-screen-patch.exe`, verified then deleted 2026-11-21) was
> **byte-identical to the (patched) overlay**, and the wrapper accepted the modified
> overlay with no DRM error. See
> [`raw/SPLASH-PATCH-RUNTIME-TEST.md`](../../raw/SPLASH-PATCH-RUNTIME-TEST.md).

**Why this matters for fixes:** because the real game is a **child process** whose
working directory is the game folder (`-changedir`), environment variables set in Steam
launch options (`WINEDLLOVERRIDES`, `WINE_D3D_CONFIG`, …) propagate to it, and a
`ddraw.dll` / `version.dll` placed in the game folder is found by the child's DLL search.
See [rendering-pipeline.md](rendering-pipeline.md) and
[cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md).

## DRM / integrity machinery

**Confirmed-static:**

- `properties/partner.xml` ships with a sibling `properties/partner.xml.sig`, and the
  `drm/` tree ships `drm.xml.sig` files — proof the DRM **verifies something**
  (signature files exist).
- No direct overlay checksum was identified. A `CRC32` polynomial (`0xEDB88320`) exists
  at outer VA `0x51fa08`, but this is the standard zlib table (the EXE also embeds image
  codecs), so it is likely **not** a DRM overlay check.

**Confirmed-runtime (2026-11-20):** the wrapper does **not** block a modified overlay — a
patch to the embedded payload launched cleanly. (It may still verify the `.xml`/`.sig`
files; that remains untested.) See
[drm-integrity-open-questions.md](../testing/drm-integrity-open-questions.md).

## The embedded game payload

The real game is appended as a **PE overlay inside this executable**; see the dedicated
page: [inner-game-payload.md](inner-game-payload.md).

## Related

- [inner-game-payload.md](inner-game-payload.md) — the extracted real game.
- [resource-dll-xui.md](resource-dll-xui.md) — the `Resource.dll` asset container.
- [binaries-and-hashes.md](binaries-and-hashes.md) — every file's role + hash.
- [proton-launch-options.md](../fixes/proton-launch-options.md) — launch configuration.
