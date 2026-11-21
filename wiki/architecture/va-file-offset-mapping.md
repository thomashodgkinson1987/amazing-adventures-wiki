---
title: Reference — VA ↔ File-Offset Mapping
type: architecture
status: confirmed-static
updated: 2026-11-21
sources:
  - ../../raw/SPLASHSCREEN-PATCHING.md
---

# VA ↔ File-Offset Mapping (don't repeat the trap)

Reversing this game requires converting virtual addresses (VAs) to file offsets, and the
two PEs use **different** conventions. An earlier pass got this wrong; this page records
the correct mapping.

## The extracted real game (`popcapgame1.exe`)

*(The user capture of this file, `copy.exe`, was removed 2026-11-21; the offsets below are
inner-game offsets and remain valid — extract a fresh copy from the EXE overlay if needed,
see [binaries-and-hashes.md](binaries-and-hashes.md#verified-captures-removed-2026-11-21).)*

Section headers list **RVAs**; image base is `0x400000`. Use:

```
inner game: .text  VA 0x401000  file 0x000400   → fo = 0x000400 + (va - 0x401000)
            .rdata VA 0x517000  file 0x116200   → fo = 0x116200 + (va - 0x517000)
            .data  VA 0x540000  file 0x13E800   → fo = 0x13E800 + (va - 0x540000)
            .rsrc  VA 0x558000  file 0x14A200   → fo = 0x14A200 + (va - 0x558000)
```

The inner game has **no ASLR** (`DllCharacteristics = 0x8000`, `DYNAMIC_BASE` clear) and
`ImageBase = 0x400000`, so a runtime patch can use **fixed addresses**.

## `AmazingAdventures2.exe` (outer wrapper)

Every section has **file == VA − 0x400000**.

## `AmazingAdventures2.exe` (embedded overlay = the real game)

The overlay is the real game at a fixed delta:

```
amz_offset = 0x17B1A2 + inner_offset
```

So an inner-game offset `X` corresponds to `AmazingAdventures2.exe` offset
`0x17B1A2 + X`. This is how the splash patch offsets in
[splash-patch-option-a.md](../fixes/splash-patch-option-a.md) are derived
(e.g. inner `0x126be8` → `AmazingAdventures2.exe` `0x2a1d8a`).

> **Consistency note (2026-11-20):** [`FINDINGS.md`](../../raw/FINDINGS.md) refers to
> `-changedir=` "at `0x533e98`" — that value is a **VA**, while the same list labels its
> neighbours as file offsets. The correct outer file offset is `0x133e98` (VA `0x533e98`).
> The `popcapgame*.exe` strings are at file offset `0x133e18` (VA `0x533e18`). Mixed
> labelling only; the values are consistent once the convention is applied. Logged as a
> lint note.

## Tooling

- Disassemble with the mapping above. To find direct callers, scan for `0xE8` relative
  calls and compute `src + 5 + rel` (linear sweeps from arbitrary points desync — always
  start at a function prologue).
- To find a constant's users, search the whole EXE for its 4-byte little-endian value, or
  disassemble and match `[abs32]` operands.
- Tooling: `python3` with `pefile` + `capstone`
  (`pip install --break-system-packages pefile capstone`), plus `rizin`, `objdump`,
  `strings`, `file`.

## Related

- [inner-game-payload.md](inner-game-payload.md) — the overlay these offsets address.
- [splash-patch-option-a.md](../fixes/splash-patch-option-a.md) — uses this mapping.
- [binaries-and-hashes.md](binaries-and-hashes.md).
