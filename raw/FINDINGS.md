# Amazing Adventures Around the World — Performance / Rendering Findings

Working notes from a prior session. Goal: fix the game feeling like ~30 fps and the
flickering mouse cursor when run on Linux via Proton.

**Status: RESOLVED.** The fix that worked is **cnc-ddraw** (a native DirectDraw
reimplementation) dropped into the game folder + a `WINEDLLOVERRIDES` launch option.
See [Solution that worked](#solution-that-worked-cnc-ddraw).

---

## Environment

- Host: Linux (CachyOS). Steam.
- Compatibility tool: `proton-cachyos-11.0-20261015-slr-x86_64`.
- Steam AppID: **3530** ("Amazing Adventures Around the World").
- Game folder (this directory, mounted from the host):
  `.../steamapps/common/Amazing Adventures Around the World/`
- Proton prefix: `.../steamapps/compatdata/3530/pfx/`

## Game architecture (confirmed by static analysis)

The game is a PopCap/SpinTop title with a two-layer DRM/packaging structure:

- **`AmazingAdventures2.exe`** (outer, ~3.0 MB) = **the original Steam executable**
  (this is the file Steam launches; it ships with the game and is stock/untouched). It is
  the PopCap DRM / Steam wrapper.
  - Contains a Lua engine, the trial/DRM screens (see `drm/`), and Steam glue
    (`Unable to load Steam.dll`, `SteamSetUser Failed`).
  - It does **not** do any SDL/DirectDraw rendering.
  - At runtime it **extracts the real game** as `popcapgame1.exe` / `popcapgame2.exe` /
    `popcapgame3.exe` and **launches it as a child process** with:
    `"<path-to-popcapgameN.exe>" -changedir="<dir>"`
    (confirmed: `CreateProcessA` call sites at file offsets `0x7bdb` and `0x10458f`;
    strings `popcapgame1/2/3.exe` at `0x133e23`; `-changedir=` at `0x533e98`).
  - Because it runs a **child process**, environment variables set in Steam launch
    options (e.g. `WINEDLLOVERRIDES`, `WINE_D3D_CONFIG`) propagate to the real game,
    and the child's working dir is set via `-changedir=` (the game folder itself —
    that's why a `ddraw.dll` placed here gets picked up).
- **The real game** = a second PE appended as an overlay inside `AmazingAdventures2.exe`
  (overlay starts at file offset `0x17b1a2`; embedded PE `ImageBase 0x400000`).
  - `copy.exe` in this folder is a **captured copy of `popcapgame1.exe`**, the file the DRM
    extracts and runs at runtime. It is **byte-for-byte identical** to the embedded overlay
    payload (`[0x17b1a2 : 0x17b1a2+1465856]`, size `0x165E00` = 1,465,856; sha256
    `18e40b96c95cfd1c4f7ae9df5bc522b76fcb8cfc9bd20586a417604ac7a4569e` for both). This
    confirms the overlay is written out **verbatim** (not decrypted/obfuscated) — useful if
    anyone wants to disassemble or patch the real game directly. The code inside `copy.exe`
    is 100% stock (it is the unmodified extracted game); the `copy.exe` file itself was
    created by the user for inspection. It is not a game binary that ships with Steam.
  - Built on the **"SDA Game Framework"** (`gfSDL`, `gfvfs.cpp`, © SDA Software
    Associates Inc.) with **SDL 1.2 statically linked**.
  - Renders through **DirectDraw** (`DirectDrawCreate`, `IDirectDrawSurface3`,
    hardware cursor, etc.). **There is NO Direct3D anywhere in it.**
  - Audio via **`bass.dll`** (BASS SDK 2.x).
  - Game assets live in **`Resource.dll`** (40 MB data-only PE; exports are texture/
    audio/UI resource names). Options UI (checkboxes etc.) is stored as XML/Lua inside it.

### The key code detail (why it's slow)

The inner game's SDL video init **hard-codes the SDL video driver** (inner VA
`0x00474ac0`):

```c
SDL_putenv("SDL_VIDEODRIVER=directx");   // DirectDraw  (string at inner VA 0x52CA74)
if (SDL_VideoInit(...) < 0)
    SDL_putenv("SDL_VIDEODRIVER=windib"); // GDI fallback (string at inner VA 0x52CA8C)
```

Consequences:

- Any `SDL_VIDEODRIVER=...` you put in Steam launch options is **overwritten** by the
  game, so you cannot switch drivers that way.
- By default everything goes through:
  `game → SDL 1.2 "directx" driver → DirectDraw → Wine wined3d → OpenGL → GPU/compositor`.
  That path is well known to be poor for old 2D SDL games: an extra GPU round-trip,
  vsync-locked (the "~30 fps" feel) and prone to hardware-cursor flicker.
- The only built-in fallback is to `windib` (GDI), and only if DirectDraw init fails.

### Other useful facts

- Product/registry name: `Amazing Adventures Around The World`
  → config is under `HKCU\Software\PopCap\Amazing Adventures Around The World`
  (framework keys seen: `ScreenMode`, `CustomCursors`, `Test3D\FailureReason`,
  `Test3D\Warning`).
- In-game Options screen has checkboxes `checkboxfullscreen`, `checkboxacceleration`
  (`ID_OPTIONS_HACC` = "Use Hardware Acceleration"), `checkboxrelaxed`, `checkboxhints`.
- `config.txt` (this folder) only holds `distributorname`, `fullscreen`,
  `numberseparator`, `musicvol`, `sfxvol`.

---

## Solution that worked: cnc-ddraw

[cnc-ddraw](https://github.com/FunkyFr3sh/cnc-ddraw) is a native reimplementation of the
DirectDraw API with GDI / OpenGL / Direct3D9 renderers. It replaces Wine's slow
ddraw→wined3d path entirely, which fixed all three complaints (smoothness, cursor,
reliable fullscreen/windowed).

### What is installed now

- `ddraw.dll` in this folder (32-bit, 409,600 bytes,
  sha256 `7fa0770e930f21640241c0f375bf8f422cb5303d2e06a30e9162ef0dad955d77`).
- `ddraw.ini` in this folder (cnc-ddraw config; default `[ddraw]` section — no
  game-specific section is required).
- Steam → game → Properties → **Launch Options**:
  ```
  WINEDLLOVERRIDES="ddraw=n,b" %command%
  ```
  (`n,b` = try native `ddraw.dll` first, fall back to builtin.)

### Why it works here

The DRM wrapper launches the real game as a child whose working dir is this folder, so
Windows/Wine's DLL search finds the native `ddraw.dll` here and the override forces it
to be used.

### Current `ddraw.ini` values (non-comment lines)

```
[ddraw]
width=800
height=600
fullscreen=false
windowed=true
maintas=false
maxfps=-1
vsync=false
adjmouse=false
shader=Shaders\interpolation\catmull-rom-bilinear.glsl
renderer=auto
border=true
resizable=true
d3d9_filter=2
center_window=1
maxgameticks=0
limiter_type=0
minfps=0
nonexclusive=true
singlecpu=true
fixchilds=2
hook=4
... (plus cnc-ddraw's large built-in per-game compatibility database)
```

Notes:
- `renderer=auto` resolves to Direct3D9 → DXVK → Vulkan (falls back to OpenGL/GDI).
- The referenced `Shaders\...glsl` file is **not present** (no `Shaders/` dir here); it
  only matters for the OpenGL renderer, so it is harmless with `renderer=auto`.
- `windowed=true` / `fullscreen=false` starts windowed/borderless; **Alt+Enter**
  toggles fullscreen (`keytogglefullscreen=0x0D`).
- Useful knobs if tuning further: `renderer=gdi|opengl|direct3d9`, `maxfps`,
  `maxgameticks`, `vsync`, `width`/`height`, `windowed`.

### To revert

Remove the launch option and delete `ddraw.dll` + `ddraw.ini` from this folder.

---

## Alternatives considered (not needed now, but documented)

1. **`WINE_D3D_CONFIG=renderer=gdi`** (launch option). Modern replacement for the old
   `DirectDrawRenderer=gdi` (removed in Wine 6). Confirmed valid: wined3d reads
   `renderer` from `HKCU\Software\Wine\Direct3D` (and per-app
   `...\AppDefaults\<exe>\Direct3D`) or the `WINE_D3D_CONFIG` env var; values
   `gl` / `vulkan` / `gdi` (== `no3d`). Keeps SDL on DirectDraw but blits via
   software/GDI, removing the GL round-trip. Good fallback if cnc-ddraw is ever removed.
2. **`WINEDLLOVERRIDES="ddraw=d"`** — disable ddraw so the `directx` SDL driver fails,
   triggering the game's own fallback to `windib` (GDI). Achieves the same idea as
   patching the exe, with no patching. Risk: requests fullscreen; windib may reject the
   mode on some setups.
3. **In-game Options**: uncheck "Use Hardware Acceleration" (readme's own advice), and/or
   turn off Full Screen (`fullscreen=false` in `config.txt`).
4. **Wine virtual desktop**: `protontricks 3530 winecfg` → Graphics → Emulate a virtual
   desktop (classic fix for old fullscreen 2D games / cursor issues).
5. **Direct EXE patch (not recommended)**: the inner game's plaintext string
   `SDL_VIDEODRIVER=directx` could be edited to `windib`, but the PopCap DRM overlay may
   integrity-check the executable, and option (2) gives the same result with no risk.

---

## How the analysis was done (for reproducing)

- `file`, `objdump -x/-p`, `strings` on the two PEs.
- Python (`pefile`, `capstone`, installed via `pip install --break-system-packages`) to
  map VAs, locate strings in sections/overlay, and disassemble xrefs.
- Confirmed the outer→child `CreateProcessA` launch and the inner SDL driver forcing.
- Verified current Wine behavior against `wine-mirror/wine` source
  (`dlls/wined3d/wined3d_main.c` for the `renderer` / `WINE_D3D_CONFIG` handling).

## Caveats / unverified

- No GPU/display or Steam-DRM run was possible during analysis, so conclusions are from
  static analysis plus known Wine behavior; the cnc-ddraw result is reported by the user
  as working.
- `copy.exe` (~1.4 MB, PE32) is a **user-made copy** of `popcapgame1.exe`, captured while
  the game was running. The code inside it is byte-identical to the game payload embedded
  in `AmazingAdventures2.exe` (see the architecture section), so it is stock code and safe
  to inspect, disassemble or patch for experimentation. Note: `copy.exe` is the only
  non-stock file here — it was added by the user. The shipped Steam executable is
  `AmazingAdventures2.exe`, which is original and untouched.
