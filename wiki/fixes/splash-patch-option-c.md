---
title: Fix — Splash Patch Option C (blank splash textures)
type: fix
status: untested
updated: 2026-11-21
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
---

# Splash Patch — Option C (blank the splash textures)

**Status: `untested` (visual-only, no code patch) — NOT NEEDED (2026-11-20).** Option A was
verified at runtime, so this texture-only approach is retained for reference only.

Repoint or override the three splash textures so **no logo is visible**, leaving the
~1.25 s delays in place. This **does not skip the delay** — it only hides the art. It is
the least invasive option if a full skip proves fragile.

## Textures involved

- `spintopsplash.jpg`
- `std:distributor.jpg`
- `std:publisher.jpg`
- (`title_background.jpg` is the title art — **keep it**.)

`Resource.dll` embeds `SPINTOPSPLASH.JPG` (name table @ `0x4e51a`) and
`TITLE_BACKGROUND.JPG` (@ `0x4eac4`). The game folder also has a loose `publisher.jpg`.

## Two sub-approaches

1. **Resource edit** — repoint/replace the resource entries in `Resource.dll`. Requires
   editing a 40 MB binary; higher risk.
2. **Loose-file override** — drop blank/tiny `publisher.jpg` / `distributor.jpg` /
   `spintopsplash.jpg` into the game folder and hope the engine prefers loose files over
   embedded resources.

> **Untested / open question:** whether the engine prefers a **loose file in the working
> dir over the embedded resource**. The presence of a loose `publisher.jpg` *suggests* it
> might. If so, approach 2 blanks the splashes with **no binary edits at all** — worth
> testing before touching `Resource.dll`. See
> [drm-integrity-open-questions.md](../testing/drm-integrity-open-questions.md) (item 4).

## Trade-offs

| | Delay skipped? | Art hidden? | Risk |
|---|---|---|---|
| Option A (zero durations) | ✅ yes | ✅ yes | binary edit / DRM |
| Option C | ❌ no | ✅ yes | low (loose files) / medium (Resource.dll) |

## Related

- [splash-patch-option-a.md](splash-patch-option-a.md) ·
  [splash-screen-state-machine.md](splash-screen-state-machine.md)
- [resource-dll-xui.md](../architecture/resource-dll-xui.md) — where the textures live.
- [splash-patch-test-plan.md](../testing/splash-patch-test-plan.md)
