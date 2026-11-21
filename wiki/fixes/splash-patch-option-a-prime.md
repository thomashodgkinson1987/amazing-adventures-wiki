---
title: Fix — Splash Patch Option A′ (runtime proxy DLL)
type: fix
status: untested
updated: 2026-11-20
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
---

# Splash Patch — Option A′ (runtime memory patch via a proxy DLL)

**Status: `untested` fallback — NOT NEEDED (2026-11-20).** Option A was verified at runtime
with no DRM rejection, so this runtime-proxy approach is retained for reference only.
Leave `AmazingAdventures2.exe` untouched and patch the **loaded process's memory** instead.

Because the game image is **non-relocatable** with `ImageBase = 0x400000`, the timing
constants are always at the same fixed addresses (`0x5279e8`, `0x52e148`, `0x52e0c0`;
see [splash-screen-state-machine.md](splash-screen-state-machine.md)).

## Mechanism

Drop a 32-bit **`version.dll` shim** in the game folder and add the launch option
`WINEDLLOVERRIDES="version=n,b" %command%`. The wrapper launches the real game as a child
whose working dir is the game folder, so the local `version.dll` is picked up — the same
mechanism that makes `ddraw.dll` work (see
[cnc-ddraw-performance-fix.md](cnc-ddraw-performance-fix.md) and
[drm-wrapper.md](../architecture/drm-wrapper.md)).

## Sketch

```c
// DllMain: after the real module is loaded, overwrite the float constants.
// Guard with a check that the module at 0x400000 is the game (MZ/PE + expected bytes).
static void patch(void) {
    if (*(uint16_t*)0x400000 != 0x5A4D) return;              // 'MZ'
    *(volatile float*)0x5279e8 = 0.0f;                        // splash dwell
    *(volatile float*)0x52e148 = 0.0f;                        // state4->5
    *(volatile float*)0x52e0c0 = 0.0f;                        // state5
}
```

- **Fail safe:** verify the bytes at `0x5279e8` equal `00 00 a0 3f` before writing.
- The child is launched with `-changedir`, and the DLL is loaded **by the child**.

## When to use

If modifying `AmazingAdventures2.exe` trips the wrapper's integrity check
(see [drm-integrity-open-questions.md](../testing/drm-integrity-open-questions.md)), fall
back to this. It is **safer against integrity checks** than Option A but less persistent
(needs the DLL + launch option every time).

## Related

- [splash-patch-option-a.md](splash-patch-option-a.md) · [option C](splash-patch-option-c.md)
- [splash-patch-test-plan.md](../testing/splash-patch-test-plan.md)
