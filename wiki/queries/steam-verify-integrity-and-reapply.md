---
title: Query — Steam Verify-Integrity Survival & Re-Applying Both Fixes
type: query
status: inferred
updated: 2026-11-21
sources:
  - ../../raw/FINDINGS.md
  - ../../raw/SPLASHSCREEN-PATCHING.md
  - ../../raw/SPLASH-PATCH-RUNTIME-TEST.md
  - ../../raw/snapshots/ddraw.ini
---

# Query — What survives Steam "Verify integrity of game files", and how to re-apply

**Question:** After Steam's *Verify integrity of game files* (or a fresh install on a
second Linux machine such as a Steam Deck), which fixes survive, which break, why can't
`popcapgame1.exe` be patched once, and what are the exact commands / paths / SHA-256
checks / launch options to get both the performance fix and the splash-screen skip working
again?

> **Page status is `inferred`** because the load-bearing survival/re-apply claims below
> (verify reverts the EXE patch; cnc-ddraw files survive verify but aren't auto-installed
> fresh; launch options don't cross machines) are reasoned from manifest membership and
> have **not** been exercised as runtime tests. The patch mechanics they rest on are
> `confirmed-runtime` / `confirmed-static` and are marked inline.

## Summary

| Fix | Stored where | Steam "Verify integrity" / reinstall | Fresh 2nd machine |
|---|---|---|---|
| **Performance/cursor fix** — cnc-ddraw | `ddraw.dll` + `ddraw.ini` in the game folder + a launch option | **Survives** (user-added, not in the manifest) | **Absent** — copy in by hand; re-enter the launch option |
| **Splash-screen skip** — Option A | 6 bytes inside `AmazingAdventures2.exe` | **Breaks** (file is in the manifest → pristine restored) | **Absent** — re-apply the patch |

## 1. What survives, what breaks

**The performance fix survives.** `ddraw.dll` / `ddraw.ini` are user-added, **not** part of
the Steam manifest, so verify leaves them alone — see
[binaries-and-hashes.md](../architecture/binaries-and-hashes.md) (user-added list) and the
caveat in [splash-patch-option-a.md](../fixes/splash-patch-option-a.md) ("the cnc-ddraw fix
… is separate and is not part of the manifest, so it is normally left alone"). The fix
itself is `confirmed-runtime` ([cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md));
the "verify leaves it alone" behaviour is `inferred`.

**The splash fix breaks.** Option A writes 6 bytes into `AmazingAdventures2.exe`, a stock
manifest file, so verify/reinstall restores the pristine `275b8b27...c00e3` and drops the
patch ([splash-patch-option-a.md](../fixes/splash-patch-option-a.md),
[binaries-and-hashes.md](../architecture/binaries-and-hashes.md)). The patch's runtime
effect is `confirmed-runtime`; the verify-reverts-it caveat is `inferred`.

**On a fresh second machine (Steam Deck)** nothing installs for you except the stock game;
the Proton prefix (`.../steamapps/compatdata/3530/pfx/`) is per-machine and carries none of
the fixes. You must copy in `ddraw.dll` + `ddraw.ini`, re-add the launch option, and
re-apply the patch.

## 2. Why you can't patch `popcapgame1.exe` once

`popcapgame1.exe` is **not shipped** — it is extracted at launch from the overlay inside
`AmazingAdventures2.exe`:

- The wrapper is documented in [drm-wrapper.md](../architecture/drm-wrapper.md) /
  [inner-game-payload.md](../architecture/inner-game-payload.md): the real game is a PE
  appended at **file offset `0x17b1a2`, size `0x165e00`**, written out as `popcapgame1.exe`
  and launched as a child (`CreateProcessA`, `-changedir=`). `confirmed-static`.
- Extraction is **verbatim and un-rejected** — `confirmed-runtime`: a capture of the running
  `popcapgame1.exe` after patching was byte-identical to the patched overlay, differing from
  stock in exactly the 6 patched bytes
  ([SPLASH-PATCH-RUNTIME-TEST.md](../../raw/SPLASH-PATCH-RUNTIME-TEST.md),
  [drm-integrity-open-questions.md](../testing/drm-integrity-open-questions.md) item 1).

So the only durable place to patch is the overlay inside `AmazingAdventures2.exe`. **Open
question #5** (`untested`) still leaves the exact write location / whether the wrapper
overwrites a pre-existing loose copy each launch unproven — but it always launches the
freshly extracted payload.

## 3. Exact re-apply procedure

> Substitute your real game path for `<GAME_DIR>`; on a Steam Deck this is something like
> `~/.steam/steam/steamapps/common/Amazing Adventures Around the World` (or the SD-card
> equivalent).

### Step 0 — confirm state (read-only)

```bash
cd "/path/to/steamapps/common/Amazing Adventures Around the World"
sha256sum AmazingAdventures2.exe
#   stock / post-verify : 275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3
#   already patched     : d920cde4b531d61b377b5feead659ac13e8cb11a88cd9622761c2ed4ea4eb833
sha256sum ddraw.dll
#   expect              : 7fa0770e930f21640241c0f375bf8f422cb5303d2e06a30e9162ef0dad955d77
```

### Step 1 — re-apply the splash patch (Option A), on the stock EXE

```bash
python3 - <<'PY'
p = '/path/to/game/AmazingAdventures2.exe'   # <- your <GAME_DIR>
b = bytearray(open(p, 'rb').read())
patches = [
    (0x2a1d8a, '0000a03f', '00000000'),   # 1.25 s splash dwell   (inner VA 0x5279e8)
    (0x2a84ea, '0000e03f', '00000000'),   # 1.75 s state4->5 fade (inner VA 0x52e148)
    (0x2a8462, '00002040', '00000000'),   # 2.5 s state5          (inner VA 0x52e0c0)
]
for off, old, new in patches:
    o = bytes.fromhex(old)
    assert b[off:off+4] == o, f"unexpected bytes at {off:#x}: {b[off:off+4].hex()}"
    b[off:off+4] = bytes.fromhex(new)
open(p, 'wb').write(b)
print("patched 6 bytes")
PY
sha256sum AmazingAdventures2.exe   # run from <GAME_DIR>
#   expect: d920cde4b531d61b377b5feead659ac13e8cb11a88cd9622761c2ed4ea4eb833
```

Do **not** touch `0x2a839a` (0.5 s) / `0x2a8466` (0.25 s) (optional extras) or `0x52e03c`
(general UI animation). Offsets and bytes:
[splash-patch-option-a.md](../fixes/splash-patch-option-a.md) §Patch table and
[SPLASHSCREEN-PATCHING.md](../../raw/SPLASHSCREEN-PATCHING.md) §3; mapping
(`0x17B1A2 + inner_offset`) in [va-file-offset-mapping.md](../architecture/va-file-offset-mapping.md).
This exact 3-constant patch is **`confirmed-runtime`** (clean launch, no DRM error, splashes
~1 frame, working menu; see [SPLASH-PATCH-RUNTIME-TEST.md](../../raw/SPLASH-PATCH-RUNTIME-TEST.md)
and [splash-patch-test-plan.md](../testing/splash-patch-test-plan.md) ✅ PASSED).

Restore-to-stock (optional clean base, or to bring the splashes back):

```bash
# Steam → Properties → Installed Files → Verify integrity of game files
```

### Step 2 — restore the performance files

- `ddraw.dll` — 409,600 bytes, SHA-256 `7fa0770e930f21640241c0f375bf8f422cb5303d2e06a30e9162ef0dad955d77`.
- `ddraw.ini` — cnc-ddraw config (default `[ddraw]` section; snapshot at
  [raw/snapshots/ddraw.ini](../../raw/snapshots/ddraw.ini)). Key values: `renderer=auto`,
  `windowed=true`, `fullscreen=false`, `vsync=false`, `maxfps=-1`, `maxgameticks=0`,
  Alt+Enter toggles fullscreen. The referenced `Shaders\...glsl` is absent and harmless
  with `renderer=auto`.
- Both go in the game folder next to `AmazingAdventures2.exe`.

### Step 3 — Steam launch options

Steam → game → Properties → **Launch Options**:

```
WINEDLLOVERRIDES="ddraw=n,b" %command%
```

See [proton-launch-options.md](../fixes/proton-launch-options.md) and
[cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md). The local DLL is
found because the wrapper runs the game as a child whose working dir is the game folder
([drm-wrapper.md](../architecture/drm-wrapper.md)). **Do not** use an
`SDL_VIDEODRIVER=...` launch option — the inner game overwrites it in `SDL_VideoInit`
(`confirmed-static`, [inner-game-payload.md](../architecture/inner-game-payload.md)).
Alternatives if cnc-ddraw is ever removed: `WINE_D3D_CONFIG=renderer=gdi`,
`WINEDLLOVERRIDES="ddraw=d"`, or the in-game Options "Use Hardware Acceleration" toggle.

## 4. Verification statuses

- `confirmed-runtime`: cnc-ddraw perf/cursor fix; Option A splash patch (3 constants);
  verbatim payload extraction + no DRM rejection; zero-length fades OK; splash order
  PopCap → SpinTop → Amazing Adventures.
- `confirmed-static`: wrapper/overlay architecture, offsets `0x17b1a2`/`0x165e00`, the six
  timing floats and their patch offsets, all SHA-256s, hard-coded `SDL_VIDEODRIVER=directx`.
- `inferred`: verify reverts the EXE patch; cnc-ddraw files survive verify but aren't
  auto-installed fresh; launch options don't sync across machines.
- `untested`: open question #5 (write location of the extracted `popcapgameN.exe` /
  overwrite-per-launch).

## Related

- [fixes/splash-patch-option-a.md](../fixes/splash-patch-option-a.md) ·
  [fixes/cnc-ddraw-performance-fix.md](../fixes/cnc-ddraw-performance-fix.md) ·
  [fixes/proton-launch-options.md](../fixes/proton-launch-options.md)
- [Tool — amazing-adventures-toolkit](../tooling/amazing-adventures-toolkit.md) —
  automates re-applying both fixes (and reverting them).
- [architecture/drm-wrapper.md](../architecture/drm-wrapper.md) ·
  [architecture/binaries-and-hashes.md](../architecture/binaries-and-hashes.md)
- [testing/drm-integrity-open-questions.md](../testing/drm-integrity-open-questions.md)
