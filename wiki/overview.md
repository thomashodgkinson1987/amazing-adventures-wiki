---
title: Project Overview — Amazing Adventures Around the World on Linux
type: overview
status: confirmed-runtime
updated: 2026-11-21
sources:
  - ../raw/FINDINGS.md
  - ../raw/SPLASHSCREEN-PATCHING.md
  - ../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Project Overview

Goal: make the Steam title **Amazing Adventures Around the World** (a 2008 PopCap/SpinTop
hidden-object game) play well on **Linux**, and optionally patch out its startup splash
screens. This wiki records what is known, how it was verified, and what is still
unproven.

## Environment

| Item | Value | Status |
|---|---|---|
| Host OS | **CachyOS Linux** | confirmed-runtime |
| Platform | Steam | confirmed-runtime |
| Compatibility tool | `proton-cachyos-11.0-20261015-slr-x86_64` | confirmed-runtime |
| Steam AppID | **3530** | confirmed-static (`properties/partner.xml`, `SteamId`) |
| Game folder | `.../steamapps/common/Amazing Adventures Around the World/` | confirmed-runtime |
| Proton prefix | `.../steamapps/compatdata/3530/pfx/` | confirmed-runtime |

See [proton-launch-options.md](fixes/proton-launch-options.md) for the launch
setup, and [binaries-and-hashes.md](architecture/binaries-and-hashes.md) for every file's
role and SHA-256. Throughout this wiki, **`<GAME_DIR>`** denotes the game folder above.

> Note on scope: the analysis was done on a host that **cannot run the game** (no
> display/GPU, and the PopCap DRM requires Steam), so the static findings are
> `confirmed-static`. The runtime results come from the user's own Steam launches on the
> gaming host: the **cnc-ddraw performance fix** and the **Option A splash patch** are both
> `confirmed-runtime`, as are the **visual splash order** (PopCap → SpinTop → Amazing
> Adventures) and the fact that the **DRM does not reject the modified overlay**.

## The game in one paragraph

The shipped Steam executable `AmazingAdventures2.exe` is **not the game** — it is a
PopCap **DRM/Steam wrapper**. At launch it extracts the real game (`popcapgame1.exe`)
from a PE overlay embedded in itself and runs it as a child process. The real game is a
32-bit PE built on the **"SDA Game Framework"** with **SDL 1.2** statically linked,
rendering through **DirectDraw** (no Direct3D at all), audio via `bass.dll`, and all
assets packed in a 40 MB `Resource.dll`. See [drm-wrapper.md](architecture/drm-wrapper.md),
[inner-game-payload.md](architecture/inner-game-payload.md) and
[rendering-pipeline.md](architecture/rendering-pipeline.md).

## Status of the two modifications

### 1. Performance / cursor fix — ✅ RESOLVED (confirmed-runtime)

Symptom: the game felt like ~30 fps with a flickering mouse cursor under Proton. Cause:
the inner game hard-codes `SDL_VIDEODRIVER=directx`, forcing the slow
`SDL → DirectDraw → wined3d → OpenGL` path. Fix: drop **cnc-ddraw** (`ddraw.dll` +
`ddraw.ini`) into the game folder and add a `WINEDLLOVERRIDES="ddraw=n,b"` launch option.

→ [cnc-ddraw-performance-fix.md](fixes/cnc-ddraw-performance-fix.md)

### 2. Splash-screen skip — ✅ RESOLVED (confirmed-runtime)

Goal: skip the three startup splashes (PopCap → SpinTop → "Amazing Adventures"). The
"splashes" are states 1–3 of the game's own front-end state machine, driven by float dwell
constants in `.rdata`. **Option A** zeroes those constants in the embedded overlay.
**Applied 2026-11-20 and verified at runtime:** the wrapper launched with **no DRM
error**, flashed **PopCap → SpinTop → Amazing Adventures in ~1 frame each**, and landed on
a working main menu (reproduced twice). The DRM re-extracts the patched payload verbatim.
⚠️ Steam **"Verify integrity of game files"** restores the pristine EXE
(`275b8b27…c00e3`) and undoes the patch (the [toolkit](tooling/amazing-adventures-toolkit.md)
can also revert it); see [splash-patch-option-a.md](fixes/splash-patch-option-a.md).

→ [splash-screen-state-machine.md](fixes/splash-screen-state-machine.md) ·
[option-a](fixes/splash-patch-option-a.md) · [option-a-prime](fixes/splash-patch-option-a-prime.md) · [option-c](fixes/splash-patch-option-c.md) ·
[test plan](testing/splash-patch-test-plan.md)

## Applying the fixes with a tool

Both fixes can also be applied by a small cross-platform tool —
[`amazing-adventures-toolkit`](tooling/amazing-adventures-toolkit.md) — which
installs/removes cnc-ddraw and applies/removes the Option A splash patch, hash-verified
and reversible. It automates the manual procedures above; the analysis on the fix pages
remains the source of truth.

## Unverified / open

- **Resolved at runtime (2026-11-20):** the splash patch works, no DRM rejection, and the
  visual order is PopCap → SpinTop → Amazing Adventures. See
  [drm-integrity-open-questions.md](testing/drm-integrity-open-questions.md) and
  [splash-visual-order-checklist.md](testing/splash-visual-order-checklist.md).
- Still open: long-session stability; loose-file override precedence (Option C); where the
  wrapper writes the extracted `popcapgameN.exe`.

## Navigation

- Full catalog: [index.md](index.md)
- Change history: [log.md](log.md)
