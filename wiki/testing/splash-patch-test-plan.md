---
title: Testing — Splash Patch Test Plan
type: testing
status: confirmed-runtime
updated: 2026-11-21
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
  - ../../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Splash Patch Test Plan

**Status: ✅ PASSED (confirmed-runtime, 2026-11-20).** Option A was applied and verified on
real Steam launches; all steps below completed successfully.

## Preconditions

- A machine that can run the game via Steam/Proton **with the working cnc-ddraw setup**
  (see [cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md)).
- Pristine hashes recorded (see
  [binaries-and-hashes.md](../architecture/binaries-and-hashes.md)) so you can restore.

## Steps

1. **Record the pristine hash** — ✅ **done**: stock `AmazingAdventures2.exe` is
   `275b8b27…c00e3` (Steam *Verify integrity of game files* restores it any time).
   Back up `Resource.dll` only if you later touch it.
2. **Apply the Option A patch** — ✅ **done (2026-11-20)**: three constants zeroed at
   `AmazingAdventures2.exe` offsets `0x2a1d8a`, `0x2a84ea`, `0x2a8462`
   (`275b8b27…c00e3` → `d920cde4…eb833`). See
   [splash-patch-option-a.md](../fixes/splash-patch-option-a.md) for exact bytes.
   *(The minimal single-constant variant — `0x2a1d8a` only — was the doc's suggested first
   experiment; all three main constants were applied instead for a single decisive run.)*
3. **Launch via Steam** with the same Proton / launch options as the working cnc-ddraw
   setup (`WINEDLLOVERRIDES="ddraw=n,b" %command%`). — ✅ **done**: launched cleanly.
4. **Observe: do the three splashes flash past and land on the title menu?** — ✅ **yes**:
   PopCap → SpinTop → Amazing Adventures in ~1 frame each, landing on a working menu;
   reproduced on a second launch.
   - The optional `0x2a84ea` / `0x2a8462` (state4→5 and state5 fades) were already applied
     in step 2, so the whole startup is snappy.
   - No DRM error, no re-extraction of a pristine exe → **Option A′ not needed**.
   - Zero-length fades behaved correctly → **no `0.01` fallback needed**.
5. **Record the confirmed state↔logo order** — ✅ done:
   [splash-visual-order-checklist.md](splash-visual-order-checklist.md).

## Decision order

Test **A → A′ → C** (see
[drm-integrity-open-questions.md](drm-integrity-open-questions.md)).

## On success

- Update the affected pages' `status` to `confirmed-runtime`, bump `updated`, add a
  status-change note, update [index.md](../index.md), and append to [log.md](../log.md).

## Related

- [splash-screen-state-machine.md](../fixes/splash-screen-state-machine.md)
- [splash-patch-option-a.md](../fixes/splash-patch-option-a.md)
