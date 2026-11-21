---
title: Fix — Splash Patch Option A (zero dwell/fade durations)
type: fix
status: confirmed-runtime
updated: 2026-11-21
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
  - ../../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Splash Patch — Option A (recommended)

**Status: ✅ CONFIRMED-RUNTIME (2026-11-20).** The patch was written to disk and then
verified on a real Steam launch — see the result block below.

> **Runtime result (2026-11-20).** With the three constants zeroed, the game **launched
> cleanly with no DRM error**, flashed **PopCap → SpinTop → Amazing Adventures in ~1 frame
> each**, and landed on a **working main menu** (reproduced on a second launch). A fresh
> capture of the extracted `popcapgame1.exe` (`copy-after-splash-screen-patch.exe`;
> verified then deleted 2026-11-21) was **byte-identical to the patched overlay**, proving
> the wrapper re-extracts the modified payload verbatim. Full report:
> [`raw/SPLASH-PATCH-RUNTIME-TEST.md`](../../raw/SPLASH-PATCH-RUNTIME-TEST.md).

> **Patch applied (2026-11-20).** The three main constants were zeroed in
> `<GAME_DIR>/AmazingAdventures2.exe`:
>
> | offset | constant | before | after |
> |---|---|---|---|
> | `0x2a1d8a` | splash dwell 1.25 s (`0x5279e8`) | `00 00 a0 3f` | `00 00 00 00` |
> | `0x2a84ea` | state4→5 1.75 s (`0x52e148`) | `00 00 e0 3f` | `00 00 00 00` |
> | `0x2a8462` | state5 2.5 s (`0x52e0c0`) | `00 00 20 40` | `00 00 00 00` |
>
> Exactly **6 bytes** changed (still size 3,024,144); all three constants read back as
> `0.0f`. The two optional constants (`0x2a839a` pre-fade 0.5 s, `0x2a8466`
> `0.25 s`) were **not** touched.
>
> - `AmazingAdventures2.exe` before: `275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3`
> - `AmazingAdventures2.exe` after:  `d920cde4b531d61b377b5feead659ac13e8cb11a88cd9622761c2ed4ea4eb833`
>
> **Restore (revert to stock):** Steam → *Properties → Installed Files → Verify integrity
> of game files* re-downloads the pristine `275b8b27…c00e3`; the
> [toolkit](../tooling/amazing-adventures-toolkit.md) can also remove the patch.
>
> ✅ **Confirmed at runtime 2026-11-20** — `status` is now `confirmed-runtime`; result
> recorded in [log.md](../log.md) and [`raw/SPLASH-PATCH-RUNTIME-TEST.md`](../../raw/SPLASH-PATCH-RUNTIME-TEST.md).
>
> ⚠️ **Persistence caveat — Steam "Verify integrity of game files" will undo this patch.**
> The patch edits `AmazingAdventures2.exe`, which **is** part of the game's Steam manifest.
> Running *Properties → Installed Files → Verify integrity of game files* (or a reinstall)
> will re-download the **pristine** EXE and silently remove the patch. The extracted
> `popcapgame1.exe` is re-extracted from the overlay on **every** launch, so there is no
> "patch the loose exe once" shortcut — after any verify/reinstall, re-apply the three
> bytes. The cnc-ddraw fix (`ddraw.dll` / `ddraw.ini`) is separate and is **not** part of
> the manifest, so it is normally left alone.
>
> **Restore (revert to stock):** Steam → *Properties → Installed Files → Verify integrity of
> game files*, or the [toolkit](../tooling/amazing-adventures-toolkit.md)'s *remove splash
> patch*.
> **Re-apply after verify/reinstall:** re-run the three-byte patch (see below), or restore a
> saved patched copy.

Make the front-end transitions **instant** so the states run through in a few frames and
the title appears immediately. Write `00 00 00 00` (float `0.0`) over each timing
constant. The state machine still executes, so the title menu is built normally (see
[splash-screen-state-machine.md](splash-screen-state-machine.md)).

## Patch table

| constant | inner VA | inner (`popcapgame1.exe`) offset | `AmazingAdventures2.exe` offset | stock bytes → patched |
|---|---|---|---|---|
| splash dwell 1.25 s | `0x5279e8` | `0x126be8` | **`0x2a1d8a`** | `00 00 a0 3f` → `00 00 00 00` |
| state4→5 1.75 s | `0x52e148` | `0x12d348` | **`0x2a84ea`** | `00 00 e0 3f` → `00 00 00 00` |
| state5 2.5 s | `0x52e0c0` | `0x12d2c0` | **`0x2a8462`** | `00 00 20 40` → `00 00 00 00` |
| pre-fade 0.5 s | `0x52dff8` | `0x12d1f8` | `0x2a839a` | `00 00 00 3f` → `00 00 00 00` *(optional)* |
| 0.25 s | `0x52e0c4` | `0x12d2c4` | `0x2a8466` | `00 00 80 3e` → `00 00 00 00` *(optional)* |

The `AmazingAdventures2.exe` offsets come from `0x17B1A2 + inner_offset`; see
[va-file-offset-mapping.md](../architecture/va-file-offset-mapping.md). Stock bytes were
re-verified in the overlay during wiki setup.

## Procedure

- **Minimal first experiment:** patch **only** `0x5279e8` (the shared 1.25 s constant that
  all three splash states use). If that alone gives an instant splash, done.
- Optionally also patch `0x52e148` and `0x52e0c0` for a snappier title fade.
- **Do NOT touch** `0x52e03c` (0.2 s, general UI animation).
- Expected result: the three splashes and the title fade each last ~1 frame → effectively
  "straight to the menu". The state machine still executes, so the menu is built normally.

**Crucially:** patch the payload embedded in **`AmazingAdventures2.exe`** (offset
`0x17b1a2`), because the wrapper re-extracts `popcapgame1.exe` from that overlay at
launch. Patching a loose extracted copy alone will **not** stick (see
[drm-wrapper.md](../architecture/drm-wrapper.md)).

> **Fallback (kept for reference):** a zero-length tween was the original worry — if it
> misbehaved (e.g. a divide/clamp issue), a very small non-zero value could be used instead,
> e.g. `0.01` (`0x3C23D70A` → bytes `0a d7 23 3c`), still imperceptible. **Not needed:**
> `0.0f` was confirmed working at runtime (clean launch, menu built, splashes skipped), so
> this is retained only for reference and reversibility.

## Minimal in-place patcher

```python
import sys
path, off, old, new = sys.argv[1], int(sys.argv[2],0), bytes.fromhex(sys.argv[3]), bytes.fromhex(sys.argv[4])
b = bytearray(open(path,'rb').read())
assert b[off:off+len(old)] == old, "unexpected bytes at %#x" % off
b[off:off+len(new)] = new
open(path,'wb').write(b)
print("patched", path, hex(off))
# e.g. python patch.py AmazingAdventures2.exe 0x2a1d8a "0000a03f" "00000000"
```

> The patch was applied only on the user's explicit instruction; game files are otherwise
> left untouched. Keep a pristine copy for restore (Steam *Verify integrity* recreates it).

## Risk

Recommended test order: **A → A′ → C** (see
[drm-integrity-open-questions.md](../testing/drm-integrity-open-questions.md) and the
[test plan](../testing/splash-patch-test-plan.md)). Option A is the cheapest faithful
fix but is the most exposed to any overlay integrity check.

## Related

- [splash-screen-state-machine.md](splash-screen-state-machine.md)
- [splash-patch-option-a-prime.md](splash-patch-option-a-prime.md)
- [splash-patch-test-plan.md](../testing/splash-patch-test-plan.md)
- [Tool — amazing-adventures-toolkit](../tooling/amazing-adventures-toolkit.md) —
  applies/removes this patch (and cnc-ddraw) automatically.
