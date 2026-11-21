---
title: Tool — amazing-adventures-toolkit (automates both fixes)
type: tool
status: confirmed-static
updated: 2026-11-21
sources:
  - https://github.com/thomashodgkinson1987/amazing-adventures-toolkit
  - ../../raw/SPLASHSCREEN-PATCHING.md
  - ../../raw/SPLASH-PATCH-RUNTIME-TEST.md
---

# Tool — amazing-adventures-toolkit

**Status: `confirmed-static`.** An external, cross-platform terminal program that
automates the two fixes this wiki documents — installing/removing **cnc-ddraw** and
applying/removing the **Option A splash patch** — with byte-level hash checks. Its patch
logic was verified against this wiki's offsets and hashes; it has not been run on this host,
so the page is `confirmed-static` (the fixes it applies remain `confirmed-runtime`).

Repository: <https://github.com/thomashodgkinson1987/amazing-adventures-toolkit>

## What it does

A single Rust (edition 2024) program for the Steam title *Amazing Adventures Around the
World* (**AppID 3530**, see [overview.md](../overview.md)), offering an interactive menu
**and** non-interactive CLI flags. Each action is fully reversible:

| Action | Wiki equivalent | What it does |
|---|---|---|
| Install / remove cnc-ddraw | [cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md) | Downloads the latest [cnc-ddraw](https://github.com/FunkyFr3sh/cnc-ddraw) release and installs (or deletes) `ddraw.dll` + `ddraw.ini` in the game folder. |
| Apply / remove splash skip | [splash-patch-option-a.md](../fixes/splash-patch-option-a.md) | Writes the same **three Option A constants** (`0x2a1d8a`, `0x2a84ea`, `0x2a8462` → `00 00 00 00`) into the embedded overlay inside `AmazingAdventures2.exe`, and can restore the stock bytes. |

It does **not** replace the analysis here — the offsets, state machine and rationale live
in the pages above; the tool is the packaged automation of them. Notable behaviours it
mirrors from this wiki:

- It patches the **overlay inside `AmazingAdventures2.exe`** (offset `0x17b1a2`), not a
  loose extracted `popcapgameN.exe`, for the reasons in
  [drm-wrapper.md](../architecture/drm-wrapper.md).
- It verifies the expected stock bytes before writing, and refuses to patch if they do not
  match.

## Verification

- The splash patch reproduces this wiki's Option A exactly: stock `AmazingAdventures2.exe`
  `275b8b27…c00e3` → patched `d920cde4…eb833` (the same 6 bytes at the same three offsets).
  `confirmed-static` (hash round-trip against the pristine stock EXE, `275b8b27…c00e3`).
- Covered by unit tests plus a gated end-to-end round-trip against the pristine backup.
- The patches it applies are themselves `confirmed-runtime` on a real Steam launch — see
  [splash-patch-option-a.md](../fixes/splash-patch-option-a.md) and
  [`raw/SPLASH-PATCH-RUNTIME-TEST.md`](../../raw/SPLASH-PATCH-RUNTIME-TEST.md).

## Caveats

- The same **Steam "Verify integrity of game files"** caveat applies: it reverts the EXE
  splash patch (the cnc-ddraw files survive). See
  [steam-verify-integrity-and-reapply.md](../queries/steam-verify-integrity-and-reapply.md).
- The **`WINEDLLOVERRIDES="ddraw=n,b"`** launch option is still required for the
  performance fix and is **not** set by the tool — see
  [proton-launch-options.md](../fixes/proton-launch-options.md).
- Prebuilt binaries are published on the repo's Releases (Windows, Linux, macOS) alongside
  a `SHA256SUMS` file and build-provenance attestations. The binaries are **unsigned**, so
  macOS may quarantine them and Windows may show a SmartScreen prompt; building from source
  is the fallback.

## Related

- [cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md)
- [splash-patch-option-a.md](../fixes/splash-patch-option-a.md) ·
  [splash-screen-state-machine.md](../fixes/splash-screen-state-machine.md)
- [proton-launch-options.md](../fixes/proton-launch-options.md)
- [steam-verify-integrity-and-reapply.md](../queries/steam-verify-integrity-and-reapply.md)
