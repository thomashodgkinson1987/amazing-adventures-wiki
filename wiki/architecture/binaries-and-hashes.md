---
title: Reference — Binaries, Files & SHA-256 Hashes
type: architecture
status: confirmed-static
updated: 2026-11-21
sources:
  - ../../raw/FINDINGS.md
  - ../../raw/SPLASHSCREEN-PATCHING.md
---

# Binaries, Files & SHA-256 Hashes

All files live in the game folder — written **`<GAME_DIR>`** (see [README](../../README.md)).
**Neither binaries nor game-derived files are copied into this repository** — everything is
referenced here by path and hash. Hashes were re-verified during wiki setup and match both
raw documents.

> **This repo is text-only.** No binaries or game-derived files are bundled. The two
> extracted-game captures (`copy.exe`, `copy-after-splash-screen-patch.exe`) were
> **verified, documented here, and then deleted** (2026-11-21) — both are byte-for-byte
> derivable from the surviving EXEs (see *Verified captures*).

> **Current state (2026-11-20):** `AmazingAdventures2.exe` has since been **modified** by
> the approved [Option A splash patch](../fixes/splash-patch-option-a.md). Its table value
> below is the **pristine** hash; the on-disk file now differs — see *Modified state*.

## Pristine (stock) hashes

| File (`<GAME_DIR>/...`) | SHA-256 | Role |
|---|---|---|
| `AmazingAdventures2.exe` | `275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3` | Stock Steam EXE = PopCap DRM/Steam wrapper; contains the real game as an overlay. |
| `Resource.dll` | `581c31ccae1d403cf89304aa530ea1a5d6d4d9c768563aa54deba11457464db8` | 40 MB data-only asset PE (textures, audio, XUI). |
| `publisher.jpg` | `220e28784b4dd65bf248d90ad9af0efc2650cb1072c55e7a04721da50a1772e5` | Loose 350×350 JPEG; likely source of the `std:publisher.jpg` (SpinTop) texture. |
| `ddraw.dll` | `7fa0770e930f21640241c0f375bf8f422cb5303d2e06a30e9162ef0dad955d77` | cnc-ddraw 32-bit DirectDraw reimplementation (the performance fix). |
| `bass.dll` | *(not hashed in sources)* | BASS SDK 2.x audio library. |

Other shipped files (not hashed in the raw sources): `ddraw.ini`, `config.txt`,
`drm/` (DRM UI + images + `.sig` files), `properties/partner.xml` (+ `.sig`),
`readme.html`, `license.txt`, `drmss.jpg`.

## Modified state (2026-11-20)

`AmazingAdventures2.exe` was patched with the approved
[Option A splash patch](../fixes/splash-patch-option-a.md): three float constants in the
embedded overlay zeroed.

| | SHA-256 |
|---|---|
| Pristine (backup) | `275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3` |
| Current on disk (patched) | `d920cde4b531d61b377b5feead659ac13e8cb11a88cd9622761c2ed4ea4eb833` |

- Bytes changed: exactly 6 (offsets `0x2a1d8c`, `0x2a1d8d`, `0x2a8464`, `0x2a8465`,
  `0x2a84ec`, `0x2a84ed`); file size unchanged (3,024,144).
- Restore the pristine EXE with Steam → *Properties → Installed Files → Verify integrity of
  game files* (re-downloads `275b8b27…c00e3`), or with the
  [toolkit](../tooling/amazing-adventures-toolkit.md)'s *remove splash patch* action.

> ⚠️ **Steam "Verify integrity of game files" reverts the patch.** `AmazingAdventures2.exe`
> is part of the Steam manifest, so verify/reinstall restores the pristine
> `275b8b27…c00e3` copy and drops the splash fix. Re-apply the three bytes afterwards (the
> extracted payload is re-generated each launch, so patching it once does not persist).

### Verified captures (removed 2026-11-21)

Two user-made captures of the extracted `popcapgame1.exe` were made during analysis. Both
were **verified byte-for-byte against the surviving EXEs**, recorded here, and then
**deleted** to keep the wiki and game folder binary-free. They are fully reproducible:

| Capture (deleted) | SHA-256 | Reproduce with |
|---|---|---|
| `copy.exe` (pristine payload) | `18e40b96c95cfd1c4f7ae9df5bc522b76fcb8cfc9bd20586a417604ac7a4569e` | Overlay of the pristine `AmazingAdventures2.exe` (`275b8b27…c00e3`) at `0x17b1a2`, size `0x165e00`. |
| `copy-after-splash-screen-patch.exe` (patched payload) | `42edd1c477bf05c530d6eeee9754c27e3cb0e0a57df184e833b336defe978bee` | Overlay of the current (patched) `<GAME_DIR>/AmazingAdventures2.exe` at the same offset/size. |

- Verified 2026-11-21: `copy.exe` == overlay(pristine `AmazingAdventures2.exe`); `copy-after…`
  == overlay(patched `AmazingAdventures2.exe`). Both matched exactly.
- The patched capture differed from the pristine one in **exactly the 6 patched bytes**,
  proving the wrapper re-extracts the payload **verbatim** (see
  [`raw/SPLASH-PATCH-RUNTIME-TEST.md`](../../raw/SPLASH-PATCH-RUNTIME-TEST.md)).

Extract the payload any time with Python:

```python
b = open('AmazingAdventures2.exe','rb').read()   # run from <GAME_DIR>
open('popcapgame1.exe','wb').write(b[0x17b1a2:0x17b1a2+0x165e00])
```

## Overlay identity

- Embedded game payload in `AmazingAdventures2.exe`: offset `0x17b1a2`, size `0x165e00`
  (1,465,856 bytes).
- The pristine payload's SHA-256 is `18e40b96…569e` (was the deleted `copy.exe`);
  the patched payload's is `42edd1c4…ebee`. The overlay is written out **verbatim** (not
  encrypted) — re-verified byte-for-byte during wiki setup and at runtime.

## Which files are stock vs. user-added

- **Stock (shipped by Steam):** `AmazingAdventures2.exe`, `Resource.dll`, `bass.dll`,
  `config.txt`, `drm/`, `properties/`, images, `readme.html`, `license.txt`.
- **User-added (still present):** `ddraw.dll` + `ddraw.ini` (cnc-ddraw fix).
- **User-added (verified, then removed 2026-11-21):** `copy.exe`,
  `copy-after-splash-screen-patch.exe` (inspection captures; derivable from the EXEs).
- `AmazingAdventures2.exe` is **stock as shipped**, but the on-disk copy is now
  **patched** (see *Modified state*).

## Re-creating the game-derived snapshots

Three game-derived files are **not bundled**; obtain them from your own install:

| File | Source / command | SHA-256 (at capture) |
|---|---|---|
| `config.txt` | `cat "<GAME_DIR>/config.txt"` — runtime config (values change with in-game settings) | `45afabc8ebf91d6eb023203955a9afedb2853e82b9198dc62ae2ce88b2de4dcc` |
| `properties/partner.xml` | `cat "<GAME_DIR>/properties/partner.xml"` — ships with a `partner.xml.sig` | `00c47096b303cf32f4b22dbc35e5a990fc7232942080fc5884fdd5a1760350c5` |
| `splashscreens.xui` | extract from `<GAME_DIR>/Resource.dll`, bytes `0x25ee068`–`0x25ee309` (below) | `5f16671abd60f6f5d6d3a5ee87e0796790033181ad3318ff437b704269452586` |

```python
# splashscreens.xui — the <xui>…</xui> text embedded in Resource.dll (run from <GAME_DIR>)
d = open('Resource.dll', 'rb').read()
open('splashscreens.xui', 'wb').write(d[0x25ee068:0x25ee309])
```

Hashes are of the copies captured during analysis; the extracted 673-byte span is the same
text (the saved snapshot added a trailing newline). `raw/snapshots/ddraw.ini` **is**
bundled — it is cnc-ddraw's own configuration file (third-party, MIT), not a game asset.

## Related

- [drm-wrapper.md](drm-wrapper.md) · [inner-game-payload.md](inner-game-payload.md) ·
  [resource-dll-xui.md](resource-dll-xui.md) · [va-file-offset-mapping.md](va-file-offset-mapping.md)
