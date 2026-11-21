---
title: Wiki Index
type: index
status: confirmed-static
updated: 2026-11-21
sources:
  - ../raw/FINDINGS.md
  - ../raw/SPLASHSCREEN-PATCHING.md
  - ../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Amazing Adventures Around the World — Wiki Index

Master catalog. **Start here.** Every page below carries a verification status:
`confirmed-runtime` (verified running) · `confirmed-static` (proven from bytes) ·
`inferred` (reasoned, not observed) · `untested` (hypothesis / open).

Schema & rules: [AGENTS.md](../AGENTS.md) · Human orientation: [README.md](../README.md) ·
History: [log.md](log.md)

## Overview

| Page | Status | Summary |
|---|---|---|
| [overview.md](overview.md) | `confirmed-runtime` | Environment (CachyOS/Proton/AppID 3530), the game in one paragraph, status of the performance fix vs. the splash patch (both resolved). |

## Architecture & Binaries

| Page | Status | Summary |
|---|---|---|
| [architecture/drm-wrapper.md](architecture/drm-wrapper.md) | `confirmed-static` | `AmazingAdventures2.exe` is a PopCap DRM/Steam wrapper; extracts `popcapgameN.exe` from its own overlay and runs it as a child. |
| [architecture/inner-game-payload.md](architecture/inner-game-payload.md) | `confirmed-static` | The real game: SDA Game Framework, SDL 1.2, DirectDraw-only, `bass.dll`, hard-coded `SDL_VIDEODRIVER`. |
| [architecture/rendering-pipeline.md](architecture/rendering-pipeline.md) | `confirmed-static` | The slow `SDL→DirectDraw→wined3d→OpenGL` path and how cnc-ddraw replaces it. |
| [architecture/resource-dll-xui.md](architecture/resource-dll-xui.md) | `confirmed-static` | The 40 MB `Resource.dll` asset container and the XUI system (`splashscreens.xui`). |
| [architecture/va-file-offset-mapping.md](architecture/va-file-offset-mapping.md) | `confirmed-static` | VA↔file-offset math for both PEs and the overlay delta (`0x17B1A2 + copy_offset`). |
| [architecture/binaries-and-hashes.md](architecture/binaries-and-hashes.md) | `confirmed-static` | Every file's role and SHA-256; stock vs. user-added; patched EXE state; captures verified & removed. |

## Modifications & Fixes

| Page | Status | Summary |
|---|---|---|
| [fixes/cnc-ddraw-performance-fix.md](fixes/cnc-ddraw-performance-fix.md) | `confirmed-runtime` | The applied fix: cnc-ddraw + `WINEDLLOVERRIDES="ddraw=n,b"`; fixed ~30 fps feel & cursor flicker. |
| [fixes/proton-launch-options.md](fixes/proton-launch-options.md) | `confirmed-runtime` | Active launch options and documented Wine/Proton alternatives. |
| [fixes/splash-screen-state-machine.md](fixes/splash-screen-state-machine.md) | `confirmed-runtime` | The startup splash front-end state machine, timing constants, per-state handlers; visual order confirmed. |
| [fixes/splash-patch-option-a.md](fixes/splash-patch-option-a.md) | `confirmed-runtime` | **✅ Applied & verified 2026-11-20:** zeroed the three splash dwell/fade floats; splashes now flash past. |
| [fixes/splash-patch-option-a-prime.md](fixes/splash-patch-option-a-prime.md) | `untested` | Runtime memory patch via a `version.dll` proxy — **not needed** (Option A succeeded); kept for reference. |
| [fixes/splash-patch-option-c.md](fixes/splash-patch-option-c.md) | `untested` | Blank/override the splash textures — **not needed** (Option A succeeded); kept for reference. |

## Tooling

| Page | Status | Summary |
|---|---|---|
| [tooling/amazing-adventures-toolkit.md](tooling/amazing-adventures-toolkit.md) | `confirmed-static` | External cross-platform tool that automates both fixes (install/remove cnc-ddraw + apply/remove the Option A splash patch); hash-verified and reversible. |

## Testing & Open Questions

| Page | Status | Summary |
|---|---|---|
| [testing/splash-patch-test-plan.md](testing/splash-patch-test-plan.md) | `confirmed-runtime` | Splash-patch test plan — **✅ PASSED 2026-11-20** (launched clean, splashes gone, menu works). |
| [testing/splash-visual-order-checklist.md](testing/splash-visual-order-checklist.md) | `confirmed-runtime` | **✅ Confirmed:** PopCap → SpinTop → Amazing Adventures. |
| [testing/drm-integrity-open-questions.md](testing/drm-integrity-open-questions.md) | `untested` | Items 1–3 resolved at runtime; 4–6 (loose-file precedence, extraction path, etc.) still open. |

## Queries

| Page | Status | Summary |
|---|---|---|
| [queries/steam-verify-integrity-and-reapply.md](queries/steam-verify-integrity-and-reapply.md) | `inferred` | What survives Steam verify/reinstall (cnc-ddraw yes, EXE splash patch no), why `popcapgame1.exe` can't be patched once, and the exact re-apply commands/paths/hashes/launch options. |

Substantial answers can be filed under `queries/` (see
[AGENTS.md §4.2](../AGENTS.md)).

## Raw sources (immutable)

| Source | Summary |
|---|---|
| [raw/FINDINGS.md](../raw/FINDINGS.md) | Prior session: performance/cursor fix (cnc-ddraw), architecture, alternatives. |
| [raw/SPLASHSCREEN-PATCHING.md](../raw/SPLASHSCREEN-PATCHING.md) | Prior session: splash state machine, patch options, test plan, hashes. |
| [raw/SPLASH-PATCH-RUNTIME-TEST.md](../raw/SPLASH-PATCH-RUNTIME-TEST.md) | Runtime test result: Option A works, no DRM rejection, confirmed splash order. |
| [raw/snapshots/ddraw.ini](../raw/snapshots/ddraw.ini) | cnc-ddraw's own configuration snapshot (third-party, MIT). |

> Game-derived files (`config.txt`, `properties/partner.xml`, `splashscreens.xui`) are
> **not bundled**. See
> [binaries-and-hashes.md](architecture/binaries-and-hashes.md#re-creating-the-game-derived-snapshots)
> for how to obtain them from `<GAME_DIR>`.
