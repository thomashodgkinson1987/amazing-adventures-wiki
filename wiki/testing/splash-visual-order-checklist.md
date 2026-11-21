---
title: Testing — Splash Visual-Order Checklist
type: testing
status: confirmed-runtime
updated: 2026-11-21
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
  - ../../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Splash Visual-Order Checklist

**Status: ✅ CONFIRMED-RUNTIME (2026-11-20).** Observed on a real run: the splashes appear
**PopCap → SpinTop → Amazing Adventures**, landing on the title menu.

## What to confirm

The three splash images come from `splashscreens.xui` (inside `Resource.dll`) and map to
front-end states 1–3.
Watch the startup and tick off what you actually see, in order:

- [x] State 1 = `distributor` node → `std:distributor.jpg` → **PopCap** logo
- [x] State 2 = `publisher` node → `std:publisher.jpg` → **SpinTop** logo
- [x] State 3 = `spintopsplash` node → `spintopsplash.jpg` → **Amazing Adventures** art
- [x] State 4 = `titleback` → `title_background.jpg` (title background)
- [x] State 5 = `title` → `title_background.jpg` (**title / main menu**)

Confirmed 2026-11-20; the table in
[splash-screen-state-machine.md](../fixes/splash-screen-state-machine.md) is updated to
`confirmed-runtime`.

## Supporting evidence (confirmed-static)

- `properties/partner.xml` does **not** name SpinTop/PopCap as publisher/distributor —
  the correct evidence is `readme.html` and `drm/custom/drm.xml`
  ("Created by SpinTop Games / Published by PopCap Games, Inc."). See the discrepancy note
  in [splash-screen-state-machine.md](../fixes/splash-screen-state-machine.md).
- Node order in `splashscreens.xui`: `spintopsplash`, `distributor`, `publisher`,
  `titleback`, then `title` container — so **node order alone does not settle the visual
  order**; a runtime observation is required.

## How to capture it

- The splash timings are ~1.25 s each; use the cnc-ddraw screenshot key or an external
  screen recorder to capture frames if the transition is too fast to read.
- After a successful skip, the check becomes "did we land directly on the built title
  menu?" rather than "which logo first".

## Related

- [splash-patch-test-plan.md](splash-patch-test-plan.md)
- [splash-screen-state-machine.md](../fixes/splash-screen-state-machine.md)
