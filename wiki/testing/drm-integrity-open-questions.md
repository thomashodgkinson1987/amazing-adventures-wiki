---
title: Testing — Open Questions (DRM Integrity & Runtime)
type: testing
status: untested
updated: 2026-11-21
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
  - ../../raw/FINDINGS.md
  - ../../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Open Questions (DRM Integrity & Runtime)

Items **1–3 are now RESOLVED** (confirmed-runtime, 2026-11-20). Items **4–6 remain
`untested`**. This host could not run the game itself; the runtime results come from the
user's Steam launches on the gaming host.

## 1. Does the DRM verify the embedded overlay? — ✅ RESOLVED (confirmed-runtime)

**Answer: No blocking integrity check.** Option A was applied and the game launched
cleanly with no DRM error. A fresh capture of the extracted `popcapgame1.exe`
(`copy-after-splash-screen-patch.exe`; verified then deleted 2026-11-21) was
**byte-identical to the patched overlay**,
proving the wrapper re-extracts the modified payload verbatim and does **not** repair or
reject it. Option A′ is therefore **not needed**.

Confirmed-static context (retained):

- `properties/partner.xml.sig` and the `drm.xml.sig` files prove the DRM verifies
  *something* (at least those `.xml` files).
- **No direct overlay checksum was identified.** A `CRC32` polynomial (`0xEDB88320`)
  exists at outer VA `0x51fa08`, but that is the standard **zlib** table (the EXE also
  embeds image codecs), so it is likely **not** a DRM overlay check.
- The overlay is stored **verbatim** (the pristine payload's SHA-256 was
  `18e40b96…569e`), so no decryption is needed to patch — but verbatim storage does not
  imply "no checksum".

**Resolution:** confirmed — no blocking check. Option A′ not needed.

## 2. Runtime behaviour of zero-length fades — ✅ RESOLVED (confirmed-runtime)

Zero-length (`0.0`) durations work: the state machine still built and displayed the title
menu. No tiny-value (`0.01`) fallback was needed.

## 3. Visual order of the three splashes — ✅ RESOLVED (confirmed-runtime)

Observed order: **PopCap (`distributor`, state 1) → SpinTop (`publisher`, state 2) →
Amazing Adventures (`spintopsplash`, state 3)**. See
[splash-visual-order-checklist.md](splash-visual-order-checklist.md).

## 4. Loose-file override precedence

Do `publisher.jpg` / `distributor.jpg` / `spintopsplash.jpg` dropped in the game folder
override the embedded resources? This decides whether **Option C** can be file-only (no
binary edits). See [splash-patch-option-c.md](../fixes/splash-patch-option-c.md).

## 5. Where the DRM writes the extracted `popcapgameN.exe`

Game folder vs. temp, and whether it re-extracts on **every** launch (assumed yes). If a
"patch the extracted file once" shortcut is desired, search the outer EXE for the
extraction writer. See [drm-wrapper.md](../architecture/drm-wrapper.md).

## 6. `0x4090f0`'s string table (menu labels)

The `push 0x521e..` targets read as single chars; likely `std::string`/UTF objects. **Not
needed** for the patch.

## Related

- [splash-patch-test-plan.md](splash-patch-test-plan.md)
- [drm-wrapper.md](../architecture/drm-wrapper.md)
- [splash-patch-option-a.md](../fixes/splash-patch-option-a.md) ·
  [option A′](../fixes/splash-patch-option-a-prime.md) ·
  [option C](../fixes/splash-patch-option-c.md)
