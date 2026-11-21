# Amazing Adventures Around the World — Splash Patch Runtime Test Result

Raw, immutable capture of the **first runtime test** of the Option A splash patch
(see `SPLASHSCREEN-PATCHING.md`, `FINDINGS.md`). Recorded 2026-11-20.

## Setup under test

- `AmazingAdventures2.exe` patched with the three main Option A constants
  (see `SPLASHSCREEN-PATCHING.md` §3):
  - `0x2a1d8a` splash dwell 1.25 s (`0x5279e8`): `00 00 a0 3f` → `00 00 00 00`
  - `0x2a84ea` state4→5 1.75 s (`0x52e148`): `00 00 e0 3f` → `00 00 00 00`
  - `0x2a8462` state5 2.5 s (`0x52e0c0`): `00 00 20 40` → `00 00 00 00`
- Patched file SHA-256: `d920cde4b531d61b377b5feead659ac13e8cb11a88cd9622761c2ed4ea4eb833`
- Pristine backup: `llm-wiki/backups/AmazingAdventures2.exe.orig`
  (`275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3`)
- Launched via **Steam** with the working cnc-ddraw setup
  (`WINEDLLOVERRIDES="ddraw=n,b" %command%`).

## Observed result (reported by user)

- **Launched cleanly — no DRM error.**
- Flashed through **PopCap → SpinTop → Amazing Adventures in ~1 frame each**.
- Landed on a **working main menu**.
- **Repeated** the launch a second time: **same result** (~1 frame per splash, straight
  to the main menu).

This confirms at runtime:

1. The three splash states still **execute** (menu is built) but are **effectively
   instantaneous** — Option A works as designed.
2. The wrapper **does not reject** the modified overlay (no DRM/integrity failure).
3. The **visual order** is PopCap → SpinTop → Amazing Adventures (state 1 = distributor =
   PopCap; state 2 = publisher = SpinTop; state 3 = spintopsplash).

## Corroborating artifact: `copy-after-splash-screen-patch.exe`

The user captured a fresh copy of the extracted `popcapgame1.exe` **while the patched game
was running** (`/workspace/copy-after-splash-screen-patch.exe`).

- Size: `0x165e00` = 1,465,856 bytes (same as `copy.exe`).
- SHA-256: `42edd1c477bf05c530d6eeee9754c27e3cb0e0a57df184e833b336defe978bee`.
- **Byte-for-byte identical to the game overlay inside the patched
  `AmazingAdventures2.exe`** (`[0x17b1a2 : 0x17b1a2+0x165e00]`).
- Differs from the pre-patch `copy.exe`
  (`18e40b96c95cfd1c4f7ae9df5bc522b76fcb8cfc9bd20586a417604ac7a4569e`) in **exactly
  6 bytes**, all inside the three patched floats:

  ```
  copy.exe offset 0x126bea/0x126beb : a0 3f -> 00 00  (VA 0x5279e8 splash dwell)
  copy.exe offset 0x12d2c2/0x12d2c3 : 20 40 -> 00 00  (VA 0x52e0c0 state5)
  copy.exe offset 0x12d34a/0x12d34b : e0 3f -> 00 00  (VA 0x52e148 state4->5)
  ```

**Interpretation:** the DRM wrapper re-extracts the game payload **verbatim** from its
overlay at launch and does **not** repair, re-encrypt, or reject the modified payload.
This resolves the previously-open questions about (a) DRM overlay verification blocking
the patch and (b) runtime behaviour of zero-length fades.

## Unverified at this time

- Whether the game remains stable over a long session (only startup was observed).
- Loose-file override precedence (Option C) — not tested, and no longer needed.
- Where the wrapper writes the extracted `popcapgameN.exe` (a copy was captured in the
  game folder, suggesting the game-folder working dir).
