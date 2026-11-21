# Amazing Adventures Around the World — Startup Splash Screens: Analysis & Patching Notes

Handoff notes for the **second** modification to this game (after the performance fix in
[`FINDINGS.md`](./FINDINGS.md)).

- **Goal:** remove / instant-skip the three startup splash screens (PopCap → SpinTop →
  "Amazing Adventures") so the game jumps straight to the title/main menu.
- **Status:** analysis complete; **no patch applied yet**. All findings below come from
  static reverse-engineering of the stock binaries (confirmed against two independent
  copies — see *Verification*).
- **Confidence:** the state-machine layout, the XUI contents and the timing constant
  addresses are **confirmed**. The exact *visual* order of the three splashes and the
  effect of each individual patch are **inferred and still need a runtime test** (this
  host cannot run the game — no display/GPU and the DRM needs Steam).

> If you are a new agent/session: read *TL;DR* and *Patch options* first, then the
> *Address quick reference* at the bottom. Everything is reproducible with the tools
> listed in *Appendix A*.

---

## TL;DR

The "splash screens" are **not** videos or a separate intro. They are the first few
states of the game's own front-end screen state machine, implemented in the extracted
game executable (`popcapgame1.exe`, captured here as `copy.exe`). Each screen is an
`<image>` node from `splashscreens.xui` (inside `Resource.dll`), and the state machine
dwell-transitions between them on a timer built from a handful of **float constants in
`.rdata`**.

- There is **no supported config / registry / command-line switch** to skip them
  (checked `config.txt`, `properties/partner.xml`, CLI parsing in both EXEs, XUI
  attributes).
- The cheap, faithful fix is to **zero the dwell/transition durations** (Option A) so the
  state machine still runs (it must — the title menu is built during state 3) but the
  splashes flash past.
- The durable place to patch is the **game payload embedded in `AmazingAdventures2.exe`**
  (offset `0x17b1a2`), because the DRM wrapper re-extracts `popcapgame1.exe` from that
  overlay at launch. Patching the loose `copy.exe` alone will not stick.
- If the PopCap DRM rejects a modified overlay, fall back to a **runtime memory patch**
  from a tiny proxy DLL (Option A′).

---

## 1. How the startup sequence works

### 1.1 The front-end state machine

The front-end lives in `.\src\aaApplication.cpp` inside the extracted game. Key routines
(all VAs are in the extracted game, image base `0x400000`):

| VA | Role |
|---|---|
| `0x405770` | `Application::Init()` — runs the loader, creates surfaces, then `setState(getState())` → starts on **state 1**. |
| `0x40c000` | Splash/XUI loader — loads `splashscreens.xui`, resolves the 5 named nodes into globals. |
| `0x406a30` | Per-frame front-end **update**, dispatches on `[app+0x84]` (state) via jump table `0x407104` (states 1..18). |
| `0x407150` | `setState(state)` — jump table `0x40772c`. |
| `0x407780` | `getState()` — returns the first populated screen field (see 1.3). |
| `0x407850` | `transition(state, …)` — records old state `[+0x88]`, sets new `[+0x84]`, maps state→screen field, calls `0x4065f0`. |
| `0x4079b0` | reverse transition (go back). |
| `0x4065f0` | builds the fade **tween**: duration (seconds) × `1000.0` → ms, registers a per-frame callback. |
| `0x406730` / `0x406910` | the two tween callbacks; they call the per-frame update `0x406a30`. |
| `0x4090f0` | "select panel" helper (sets menu labels); called when leaving state 3 → **the title menu is prepared here**. |

`0x406a30(frame)` is invoked from the tween callbacks as
`call 0x406a30` with `cl = (tween[+0x28] == 1)`. The `cl` flag selects the
"advance" vs "reverse" branch inside each state handler.

### 1.2 State → screen field, and `getState` order

`0x407850` (and the reverse `0x4079b0`) map each state number to a screen-object pointer
field on the application singleton (`0x555730`):

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

### 1.3 The five startup screens

`Application::Init` creates **5 screen surfaces** (each **800×600**: width `0x320`,
height `0x258`), at globals `0x555b04`, `0x555e14`, `0x555e84`, `0x555c98`, `0x555d08`
(created via `0x4814f0`). These are consistent with the 5 front-end screen fields used by
states 1–5:

| state | XUI node | texture | likely visible content | dwell/fade const |
|---|---|---|---|---|
| 1 | `distributor` | `std:distributor.jpg` | **PopCap** logo | `0x5279e8` = 1.25 s |
| 2 | `publisher` | `std:publisher.jpg` | **SpinTop** logo | `0x5279e8` = 1.25 s |
| 3 | `spintopsplash` | `spintopsplash.jpg` | **Amazing Adventures** splash | `0x5279e8` = 1.25 s |
| 4 | `titleback` | `title_background.jpg` | title background | `0x52e148` = 1.75 s |
| 5 | `title` | `title_background.jpg` | **title / main menu** | `0x52e0c0` = 2.5 s |

> The state↔logo mapping in rows 1–3 is **inferred**. It follows the node order and the
> user's own description (PopCap, SpinTop, Amazing Adventures). Confirm visually on the
> first test run. (`properties/partner.xml` confirms `Publisher = SpinTop`,
> `Distributor = PopCap`, i.e. `publisher` = SpinTop, `distributor` = PopCap.)

### 1.4 The `splashscreens.xui` definition

Stored **verbatim as text** inside `Resource.dll` at file offset `0x25ee040` (ends at
`0x25ee303`), and listed in the resource name table as `SPLASHSCREENS.XUI`
(`Resource.dll` file offset `0x4e53e`):

```xml
<xui>
    <texture id="spintopsplash_tex" uri="spintopsplash.jpg"/>
    <texture id="distributor_tex"  uri="std:distributor.jpg"/>
    <texture id="publisher_tex"    uri="std:publisher.jpg"/>
    <texture id="title_tex"        uri="title_background.jpg"/>

    <image id="spintopsplash" tex="spintopsplash_tex" x="0" y="0" flags="traverse"/>
    <image id="distributor"   tex="distributor_tex"   x="0" y="0" flags="traverse"/>
    <image id="publisher"     tex="publisher_tex"     x="0" y="0" flags="traverse"/>
    <image id="titleback"     tex="title_tex"         x="0" y="0" flags="traverse"/>

    <container id="title" flags="traverse">
        <image tex="title_tex" x="0" y="0"/>
    </container>
</xui>
```

The loader `0x40c000` looks these nodes up by name (UTF-16 names live in `.rdata`):
`publisher` @ VA `0x522a1c`, `distributor` @ `0x522a30`, `spintopsplash` @ `0x522a48`,
`titleback` @ `0x522a64`, `title` @ `0x522a78`; the xui filename `splashscreens.xui` is
at VA `0x5229d4`. Resolved pointers are stored in globals
`0x555900`(spintopsplash), `0x555904`(publisher), `0x555908`(distributor),
`0x555910`(title), `0x555914`(titleback).

### 1.5 The timing constants (the thing to patch)

All are **32-bit little-endian floats in `.rdata`** (seconds; the engine multiplies by
the double `1000.0` at VA `0x52dfe8` to get milliseconds):

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

### 1.6 Per-state handler behaviour (from `0x406a30`, jump table `0x407104`)

| state | handler VA | `cl==0` (advance path) | `cl!=0` (reverse path) |
|---|---|---|---|
| 1 | `0x406a74` | `getState()` then `0x407850` (dur 1.25) | reverse, pre-fade 0.5, dur 1.25 |
| 2 | `0x406ae4` | `0x407850(state 3, dur 1.25)` | reverse (shared with state 1) |
| 3 | `0x406b1a` | `0x4090f0(1)` then `0x407850(state 4, dur 1.25)` | `0x405870` then reverse |
| 4 | `0x406b96` | *(nothing — waits)* | fade=1.0 then `0x407850(state 5, dur 1.75)` |
| 5 | `0x406bd9` | `0x40c2a0` + animate (`0.25`) | reverse, pre-fade 2.5, dur 1.25 |

**Read this carefully:** state 3's advance branch calls `0x4090f0(1)`, which is where the
title menu items get configured. That is why **jumping straight to state 5 is unsafe**
(you'd reach the title with an un-built menu) and why **Option A (run the states, zero the
durations) is the right approach.**

---

## 2. Where the code and assets live

| File (this folder) | What it is | Notes |
|---|---|---|
| `AmazingAdventures2.exe` | The **stock Steam executable** = PopCap DRM/Steam wrapper. | Contains the real game as a **PE overlay at file offset `0x17b1a2`** (size `0x165e00` = 1,465,856). |
| `copy.exe` | A **user-made copy** of the extracted `popcapgame1.exe` (the real game). | Byte-identical to the overlay payload (`sha256 18e40b96…569e`). Byte-identical `code`; the `copy.exe` *file* is not shipped by Steam. |
| `Resource.dll` | 40 MB data-only asset PE. | Contains `splashscreens.xui`, splash textures (embedded JPEGs), strings, etc. |
| `ddraw.dll` + `ddraw.ini` | cnc-ddraw performance fix. | Keep. See `FINDINGS.md`. |
| `publisher.jpg` | Loose 350×350 JPEG. | Likely source for the `std:publisher.jpg` publisher (SpinTop) texture. |
| `properties/partner.xml` (+ `.sig`) | Partner metadata + **signature**. | Proof the DRM does *some* integrity checking — but of this file, not necessarily the overlay. |
| `config.txt` | Runtime config. | `distributorname=SteamPopCapv1005`, `fullscreen=true`, volumes. No skip option. |

**Key consequence:** the DRM re-extracts `popcapgame1.exe` from the `AmazingAdventures2.exe`
overlay at launch. So patching the extracted copy does not persist — patch the **overlay**,
or patch **memory at runtime**.

### 2.1 VA ↔ file-offset mapping (don't repeat the trap)

In `copy.exe` the section headers list **RVAs**; the image base is `0x400000`. Correct
mapping (this tripped up an earlier pass):

```
copy.exe:  .text  VA 0x401000  file 0x000400   → fo = 0x400    + (va - 0x401000)
           .rdata VA 0x517000  file 0x116200   → fo = 0x116200 + (va - 0x517000)
           .data  VA 0x540000  file 0x13E800   → fo = 0x13E800 + (va - 0x540000)
           .rsrc  VA 0x558000  file 0x14A200   → fo = 0x14A200 + (va - 0x558000)

AmazingAdventures2.exe (outer): every section has file == VA - 0x400000.
AmazingAdventures2.exe (overlay):  amz_offset = 0x17B1A2 + copy_offset
```

`copy.exe` has **no ASLR** (`DllCharacteristics = 0x8000`, `DYNAMIC_BASE` clear) and
`ImageBase = 0x400000`, so a runtime patch can use fixed addresses.

---

## 3. Patch options

### Option A — zero the splash dwell/fade durations (recommended)

Make the transitions instant so the states run through in a few frames and the title
appears immediately. Write `0x00000000` (float `0.0`) over each constant.

| constant | VA | `copy.exe` offset | `AmazingAdventures2.exe` offset | stock bytes → patched |
|---|---|---|---|---|
| splash dwell 1.25 s | `0x5279e8` | `0x126be8` | **`0x2a1d8a`** | `00 00 a0 3f` → `00 00 00 00` |
| state4→5 1.75 s | `0x52e148` | `0x12d348` | **`0x2a84ea`** | `00 00 e0 3f` → `00 00 00 00` |
| state5 2.5 s | `0x52e0c0` | `0x12d2c0` | **`0x2a8462`** | `00 00 20 40` → `00 00 00 00` |
| pre-fade 0.5 s | `0x52dff8` | `0x12d1f8` | `0x2a839a` | `00 00 00 3f` → `00 00 00 00` *(optional)* |
| 0.25 s | `0x52e0c4` | `0x12d2c4` | `0x2a8466` | `00 00 80 3e` → `00 00 00 00` *(optional)* |

- **Minimal first experiment:** patch only `0x5279e8` (the shared 1.25 s constant that
  all three splash states use). If that alone gives an instant splash, done.
- Optionally also patch `0x52e148` and `0x52e0c0` for a snappier title fade.
- Do **not** touch `0x52e03c` (0.2 s, general UI animation).
- Expected result: the three splashes and the title fade each last ~1 frame → effectively
  "straight to the menu". The state machine still executes, so the menu is built normally.

> Caveat: if a zero-length tween misbehaves (e.g. a divide/clamp issue), use a very small
> non-zero value instead, e.g. `0.01` (`0x3C23D70A` → bytes `0a d7 23 3c`) — still
> imperceptible.

### Option A′ — runtime memory patch via a proxy DLL (fallback if DRM rejects the edit)

If modifying `AmazingAdventures2.exe` trips the wrapper's integrity check, leave the file
untouched and patch the loaded process's memory instead. Because the image is
non-relocatable and `ImageBase` is fixed `0x400000`, the constants are always at
`0x5279e8`, `0x52e148`, `0x52e0c0`.

Sketch (32-bit `version.dll` shim dropped in the game folder + launch option):

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

- Launch option: `WINEDLLOVERRIDES="version=n,b" %command%` (the wrapper launches the
  real game as a child whose working dir is the game folder, so a local `version.dll` is
  picked up — same mechanism that makes `ddraw.dll` work).
- Verify the module is actually the game before writing (check the bytes of `0x5279e8`
  equal `00 00 a0 3f`). This makes the patch fail safe.
- Note: the child is launched with `-changedir`; the DLL is loaded by the child.

### Option B — force the start state (not recommended)

Patch `0x405843`/`0x405848` (the `getState()` call in `Init`) to return `4` or `5`, or
patch `getState()` (`0x407780`). **Risky:** state 3 calls `0x4090f0(1)` to configure the
title menu, so starting at 4/5 may yield a blank/partial title. Only consider if Option A
fails.

### Option C — blank the splash textures (visual-only, no code patch)

Repoint/override the three splash textures so no logo is visible, leaving the ~1.25 s
delays. This does **not** skip the delay; it only hides the art.

- The textures are `spintopsplash.jpg`, `std:distributor.jpg`, `std:publisher.jpg`
  (`title_background.jpg` is the title art — keep it).
- `Resource.dll` embeds `SPINTOPSPLASH.JPG` (name table @ `0x4e51a`) and
  `TITLE_BACKGROUND.JPG` (@ `0x4eac4`); the folder also has a loose `publisher.jpg`.
- **Open question:** whether the engine prefers a loose file in the working dir over the
  embedded resource (the loose `publisher.jpg` suggests it might). If so, dropping
  blank/tiny `publisher.jpg` / `distributor.jpg` / `spintopsplash.jpg` overrides into the
  game folder could blank the splashes with no binary edits at all — worth testing
  before touching `Resource.dll`.

### Option D — leave as-is

The splashes are ~1.25 s each (~4 s total). Acceptable if patching proves fragile.

---

## 4. DRM / integrity considerations

- The wrapper extracts the real game from its **own overlay** and runs it as a child
  (`CreateProcessA` IAT `0x51606c`, sites `0x407bdb` and `0x50458f`; strings
  `popcapgame1/2/3.exe` @ `0x533e18`, `-changedir=` @ `0x533e98`, "Unable to extract game
  executable" nearby). `properties/partner.xml.sig` proves the DRM verifies *something*.
- **No direct overlay checksum was identified**, but this is **not** proof there is none.
  A `CRC32` polynomial (`0xEDB88320`) exists at outer VA `0x51fa08`; however that is the
  standard zlib table (the EXE also embeds image codecs), so it likely is *not* a DRM
  overlay check.
- **Risk ranking:** Option A (patch overlay) < Option A′ (runtime memory) in terms of
  persistence, but Option A′ is safer against integrity checks. Test in this order:
  A → A′ → C.
- Always keep **pristine backups** (hashes in §6) so you can restore.

---

## 5. Test plan (once a display is available)

1. Back up `AmazingAdventures2.exe` (and `Resource.dll` if you touch it).
2. Apply the **minimal** Option A patch (`0x2a1d8a` only): overwrite `00 00 a0 3f` with
   `00 00 00 00`.
3. Launch via Steam (same Proton/launch options as the working cnc-ddraw setup).
4. Observe: do the three splashes flash past and land on the title menu?
   - If the menu is intact and splashes are gone → optionally patch `0x2a84ea` /
     `0x2a8462` too, re-test.
   - If the game fails to launch / re-extracts a pristine exe / shows a DRM error →
     revert and do Option A′.
   - If a zero tween misbehaves → use `0.01` (`0a d7 23 3c`) instead.
5. Record the confirmed state↔logo order and update this file.

Quick in-place patcher (adjust `want`/`off` for each constant):

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

---

## 6. Verification & hashes (pristine stock)

```
AmazingAdventures2.exe  sha256 275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3
Resource.dll            sha256 581c31ccae1d403cf89304aa530ea1a5d6d4d9c768563aa54deba11457464db8
copy.exe (payload)      sha256 18e40b96c95cfd1c4f7ae9df5bc522b76fcb8cfc9bd20586a417604ac7a4569e
publisher.jpg           sha256 220e28784b4dd65bf248d90ad9af0efc2650cb1072c55e7a04721da50a1772e5
ddraw.dll (cnc-ddraw)   sha256 7fa0770e930f21640241c0f375bf8f422cb5303d2e06a30e9162ef0dad955d77
```

- Overlay payload location in `AmazingAdventures2.exe`: offset `0x17b1a2`, size `0x165e00`,
  and its sha256 equals `copy.exe`'s — i.e. the overlay is written out **verbatim** (not
  encrypted).
- All splash/timing conclusions were reproduced identically from **both** `copy.exe` and
  the overlay inside `AmazingAdventures2.exe`.

---

## 7. Open questions / not yet verified

1. **Runtime behaviour** of zero-length fades (does the state machine still build the menu
   and advance? Is a tiny value needed?).
2. **Visual order** of `distributor`/`publisher`/`spintopsplash` vs states 1/2/3.
3. **Does the DRM verify the overlay?** If yes, Option A fails and we use Option A′.
4. **Loose-file override precedence** for `publisher.jpg` / `distributor.jpg` /
   `spintopsplash.jpg` (decides whether Option C can be file-only).
5. **Where exactly the DRM writes the extracted `popcapgameN.exe`** (game folder vs temp)
   and whether it re-extracts on every launch (assumed yes). Search the outer EXE for the
   extraction writer if this matters for a "patch the extracted file once" shortcut.
6. `0x4090f0`'s string table (menu labels) — the `push 0x521e..` targets read as
   single chars; likely `std::string`/UTF objects. Not needed for the patch.

---

## Appendix A — Reproduction / tooling

- `python3` with `pefile` + `capstone`
  (`pip install --break-system-packages pefile capstone`), plus `rizin`, `objdump`,
  `strings`, `file`.
- Disassemble with the correct VA→offset mapping from §2.1. To find direct callers,
  scan for `0xE8` relative calls and compute `src+5+rel` (linear sweeps from arbitrary
  points desync — always start at a function prologue).
- To find a constant's users, search the whole EXE for its 4-byte little-endian value, or
  disassemble and match `[abs32]` operands.

### Address quick reference (extracted game, base `0x400000`)

```
Init                0x405770      getState            0x407780
XUI loader          0x40c000      setState            0x407150
per-frame update    0x406a30      transition          0x407850
update jump table   0x407104      reverse transition  0x4079b0
tween builder       0x4065f0      select panel        0x4090f0
tween callbacks     0x406730 / 0x406910
xui nodes (VA):     splashscreens.xui 0x5229d4 ; publisher 0x522a1c ; distributor 0x522a30
                    spintopsplash 0x522a48 ; titleback 0x522a64 ; title 0x522a78
timing floats:      0x5279e8=1.25  0x52e148=1.75  0x52e0c0=2.5  0x52dff8=0.5  0x52e0c4=0.25
Resource.dll XUI:   0x25ee040..0x25ee303 (UTF-8 text)
```

### Patch offsets (quick copy)

```
AmazingAdventures2.exe  0x2a1d8a : 00 00 a0 3f  (1.25s dwell)  -> 00 00 00 00
AmazingAdventures2.exe  0x2a84ea : 00 00 e0 3f  (1.75s)        -> 00 00 00 00
AmazingAdventures2.exe  0x2a8462 : 00 00 20 40  (2.5s)         -> 00 00 00 00
```
