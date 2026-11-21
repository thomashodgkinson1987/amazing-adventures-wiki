---
title: Fix — Startup Splash Screens: State Machine
type: fix
status: confirmed-runtime
updated: 2026-11-21
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
  - ../../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Startup Splash Screens: The State Machine

**Goal:** remove / instant-skip the three startup splash screens (PopCap → SpinTop →
"Amazing Adventures") so the game jumps straight to the title/main menu.

**Status:** analysis complete and **verified at runtime on 2026-11-20** (Option A applied;
splashes flash past in ~1 frame, main menu works). The state-machine layout, the XUI
contents and the timing-constant addresses are **confirmed-static**; the **visual order**
and the **effect of the patch** are now **confirmed-runtime**.

The "splash screens" are **not videos or a separate intro**. They are the first few
states of the game's own **front-end screen state machine**, implemented in the extracted
game (`popcapgame1.exe`, extracted from the EXE overlay). Each screen is an `<image>` node
from `splashscreens.xui` (inside `Resource.dll`); the machine dwell-transitions between
them on a timer built from a handful of **float constants in `.rdata`**.

There is **no supported config / registry / command-line switch** to skip them (checked
`config.txt`, `properties/partner.xml`, CLI parsing in both EXEs, XUI attributes).

## Front-end routines (VA base `0x400000`)

| VA | Role |
|---|---|
| `0x405770` | `Application::Init()` — loader, creates surfaces, then `setState(getState())` → starts on **state 1**. |
| `0x40c000` | Splash/XUI loader — loads `splashscreens.xui`, resolves the 5 named nodes into globals. |
| `0x406a30` | Per-frame front-end **update**; dispatches on `[app+0x84]` (state) via jump table `0x407104` (states 1..18). |
| `0x407150` | `setState(state)` — jump table `0x40772c`. |
| `0x407780` | `getState()` — returns the first populated screen field. |
| `0x407850` | `transition(state, …)` — records old state `[+0x88]`, sets new `[+0x84]`, maps state→screen field, calls `0x4065f0`. |
| `0x4079b0` | reverse transition (go back). |
| `0x4065f0` | builds the fade **tween**: duration (s) × `1000.0` → ms, registers a per-frame callback. |
| `0x406730` / `0x406910` | the two tween callbacks; they call the per-frame update `0x406a30`. |
| `0x4090f0` | "select panel" helper (sets menu labels); called when leaving state 3 → **the title menu is prepared here**. |

`0x406a30(frame)` is invoked from the tween callbacks as `call 0x406a30` with
`cl = (tween[+0x28] == 1)`. The `cl` flag selects the "advance" vs "reverse" branch
inside each state handler.

## State → screen field, and `getState` order

`0x407850` (and reverse `0x4079b0`) map state numbers to screen-object pointer fields on
the application singleton (`0x555730`):

| state | field | state | field | state | field |
|---|---|---|---|---|---|
| 1 | `+0x1d8` | 7 | `+0x1a4` | 13 | `+0x174` |
| 2 | `+0x1d4` | 8 | `+0x1a8` | 14 | `+0x184` |
| 3 | `+0x1d0` | 9 | `+0x134` | 15 | `+0x194` |
| 4 | `+0x1e4` | 10 | `+0x144` | 16 | `+0x294` |
| 5 | `+0x1e0` | 11 | `+0x154` | 17 | `+0x19c` |
| 6 | `+0x1b8` | 12 | `+0x164` |  |  |

`getState()` (`0x407780`) checks, in order, `+0x1d8, +0x1d4, +0x1d0, +0x1e4, +0x1e0,
+0x1b8` and returns the first whose screen is loaded → **1..6**. At cold start this
returns **1**, so the game always begins on the first splash.

## The five startup screens

`Application::Init` creates **5 screen surfaces** (each **800×600**: width `0x320`,
height `0x258`) at globals `0x555b04`, `0x555e14`, `0x555e84`, `0x555c98`, `0x555d08`
(created via `0x4814f0`), consistent with the 5 front-end screen fields used by states 1–5:

| state | XUI node | texture | visible content | dwell/fade const |
|---|---|---|---|---|
| 1 | `distributor` | `std:distributor.jpg` | PopCap logo | `0x5279e8` = 1.25 s |
| 2 | `publisher` | `std:publisher.jpg` | SpinTop logo | `0x5279e8` = 1.25 s |
| 3 | `spintopsplash` | `spintopsplash.jpg` | Amazing Adventures splash | `0x5279e8` = 1.25 s |
| 4 | `titleback` | `title_background.jpg` | title background | `0x52e148` = 1.75 s |
| 5 | `title` | `title_background.jpg` | title / main menu | `0x52e0c0` = 2.5 s |

> **Confirmed-runtime (2026-11-20):** the state↔logo mapping in rows 1–3 was observed on a
> real run: **state 1 = PopCap (`distributor`)**, **state 2 = SpinTop (`publisher`)**,
> **state 3 = Amazing Adventures (`spintopsplash`)**. The partner attribution is also
> corroborated statically: SpinTop is the studio, PopCap the distributor — see the
> attribution note below.

### Attribution of the SplashScreen logos (confirmed-static sources)

> **Discrepancy note (2026-11-20):** [`SPLASHSCREEN-PATCHING.md`](../../raw/SPLASHSCREEN-PATCHING.md)
> §1.3 says `properties/partner.xml` "confirms `Publisher = SpinTop`,
> `Distributor = PopCap`". That file does **not** contain those fields — it has
> `PartnerName = Valve Corporation`, `ProdName = AmazingAdventures2`,
> `RegistryKey = SteamPopCap\AmazingAdventures2`, `DisplayName`, `Height = 480`,
> `Width = 640`, `SteamId = 3530`). The **conclusion is correct** but the **citation is
> wrong**. Correct evidence: `readme.html` ("Created by SpinTop Games / Published by
> PopCap Games, Inc.") and `drm/custom/drm.xml` (`©2008 SpinTop Media, Inc. Published by
> PopCap Games, Inc.`). See [log.md](../log.md).

## The `splashscreens.xui` definition

Stored verbatim as text inside `Resource.dll` (name-table entry `SPLASHSCREENS.XUI`).
Full text and exact offsets: [resource-dll-xui.md](../architecture/resource-dll-xui.md).

The loader `0x40c000` resolves the nodes by name (UTF-16 in `.rdata`): `publisher`
@ `0x522a1c`, `distributor` @ `0x522a30`, `spintopsplash` @ `0x522a48`, `titleback`
@ `0x522a64`, `title` @ `0x522a78`; filename at `0x5229d4`. Globals: `0x555900`
(spintopsplash), `0x555904` (publisher), `0x555908` (distributor), `0x555910` (title),
`0x555914` (titleback).

## Timing constants (the thing to patch)

All are **32-bit little-endian floats in `.rdata`** (seconds; the engine multiplies by
the double `1000.0` at VA `0x52dfe8` to get ms):

| VA | value | used by |
|---|---|---|
| `0x5279e8` | **1.25 s** | front-end transitions in states 1–5 (8 refs: `0x406a8f, 0x406ab3, 0x406aef, 0x406b25, 0x406b64, 0x406bee, 0x406c69, 0x40719a`) |
| `0x52e148` | **1.75 s** | state 4 → 5 transition (`0x406bab`) |
| `0x52e0c0` | **2.5 s** | state 5 (`0x406bdd`) |
| `0x52dff8` | 0.5 s | pre-fade (10 refs, front-end + `0x4090f0`) |
| `0x52e0c4` | 0.25 s | `0x406c29, 0x4071db, 0x407b30, 0x4084a9` |
| `0x52e0c8` | 0.1 s | `0x405fec, 0x406e4a` |
| `0x52e03c` | 0.2 s | 13 refs — **general UI animation, not splash** (leave alone) |
| `0x52dfe8` | double `1000.0` | seconds→ms conversion (leave alone) |

## Per-state handler behaviour (from `0x406a30`, jump table `0x407104`)

| state | handler VA | `cl==0` (advance path) | `cl!=0` (reverse path) |
|---|---|---|---|
| 1 | `0x406a74` | `getState()` then `0x407850` (dur 1.25) | reverse, pre-fade 0.5, dur 1.25 |
| 2 | `0x406ae4` | `0x407850(state 3, dur 1.25)` | reverse (shared with state 1) |
| 3 | `0x406b1a` | `0x4090f0(1)` then `0x407850(state 4, dur 1.25)` | `0x405870` then reverse |
| 4 | `0x406b96` | *(nothing — waits)* | fade=1.0 then `0x407850(state 5, dur 1.75)` |
| 5 | `0x406bd9` | `0x40c2a0` + animate (`0.25`) | reverse, pre-fade 2.5, dur 1.25 |

**Critical:** state 3's advance branch calls `0x4090f0(1)`, which configures the title
menu items. That is why **jumping straight to state 5 is unsafe** (you'd reach the title
with an un-built menu) and why **Option A (run the states, zero the durations) is the
right approach.**

## Patch strategies

- ✅ **Option A** — zero the dwell/fade durations so the states flash past (recommended):
  [splash-patch-option-a.md](splash-patch-option-a.md).
- **Option A′** — runtime memory patch via a proxy DLL (if DRM rejects the edit):
  [splash-patch-option-a-prime.md](splash-patch-option-a-prime.md).
- **Option B** — force the start state (not recommended; unsafe menu build).
- **Option C** — blank/override the splash textures (visual only):
  [splash-patch-option-c.md](splash-patch-option-c.md).
- **Option D** — leave as-is (~4 s total).

## Related

- [resource-dll-xui.md](../architecture/resource-dll-xui.md) — the XUI it loads.
- [test plan](../testing/splash-patch-test-plan.md) ·
  [visual order checklist](../testing/splash-visual-order-checklist.md) ·
  [DRM integrity unknowns](../testing/drm-integrity-open-questions.md).
