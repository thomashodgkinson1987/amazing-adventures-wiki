---
title: Architecture — Resource.dll & the XUI System
type: architecture
status: confirmed-static
updated: 2026-11-21
sources:
  - ../../raw/FINDINGS.md
  - ../../raw/SPLASHSCREEN-PATCHING.md
---

# Resource.dll & the XUI System

`Resource.dll` is a **40 MB data-only PE**. Its exports are
**texture / audio / UI resource names**; it contains no game code. All game assets live
here.

- SHA-256: `581c31ccae1d403cf89304aa530ea1a5d6d4d9c768563aa54deba11457464db8`

## What is inside (confirmed-static)

- Texture, audio and UI resources.
- Options UI (checkboxes etc.) stored as **XML/Lua text**.
- Splash screen definitions (`splashscreens.xui`) and splash textures (embedded JPEGs).
- A **resource name table** listing entries such as `SPLASHSCREENS.XUI`,
  `SPINTOPSPLASH.JPG`, `TITLE_BACKGROUND.JPG` (name-table offsets observed around
  file offset `0x4e53e`).

## The XUI system

The XUI system is the game's XML-based UI description format, interpreted by the SDA
Game Framework. UI screens are `.xui` files: `<texture>` declarations plus
`<image>` / `<container>` nodes referenced by name.

### `splashscreens.xui`

Stored **verbatim as UTF-8 text** inside `Resource.dll`, and listed in the resource name
table as `SPLASHSCREENS.XUI`.

- Exactly located during wiki setup at **`Resource.dll` file offsets `0x25ee068`–`0x25ee309`**
  (673 bytes, `<xui>` … `</xui>`), preceded by a ~40-byte binary header.
- The text is reproduced below; re-extract it from `Resource.dll` with the recipe in
  [binaries-and-hashes.md](binaries-and-hashes.md#re-creating-the-game-derived-snapshots).

Contents:

```xml
<xui>
	<texture id="spintopsplash_tex" uri="spintopsplash.jpg"/>
	<texture id="distributor_tex" uri="std:distributor.jpg"/>
	<texture id="publisher_tex" uri="std:publisher.jpg"/>
    <texture id="title_tex" uri="title_background.jpg"/>

	<image id="spintopsplash" tex="spintopsplash_tex" x="0" y="0" flags="traverse"/>
	<image id="distributor" tex="distributor_tex" x="0" y="0" flags="traverse"/>
	<image id="publisher" tex="publisher_tex" x="0" y="0" flags="traverse"/>
    <image id="titleback" tex="title_tex" x="0" y="0" flags="traverse"/>

    <container id="title" flags="traverse">
        <image tex="title_tex" x="0" y="0"/>
    </container>
</xui>
```

> **Discrepancy note (2026-11-20):** [`SPLASHSCREEN-PATCHING.md`](../../raw/SPLASHSCREEN-PATCHING.md)
> cites the XUI at `0x25ee040..0x25ee303`. The exact `<xui>`…`</xui>` text actually spans
> `0x25ee068..0x25ee309`; the document's range starts ~40 bytes earlier (including the
> binary length/header prefix) and ends 6 bytes short. The content is identical. See
> [log.md](../log.md).

### How the loader resolves nodes

The inner game's XUI loader (`0x40c000`) looks up nodes by name. UTF-16 names in
`.rdata` (VA): `publisher` @ `0x522a1c`, `distributor` @ `0x522a30`,
`spintopsplash` @ `0x522a48`, `titleback` @ `0x522a64`, `title` @ `0x522a78`; the xui
filename `splashscreens.xui` @ `0x5229d4`. Resolved pointers stored in globals:
`0x555900` (spintopsplash), `0x555904` (publisher), `0x555908` (distributor),
`0x555910` (title), `0x555914` (titleback).

These nodes feed the startup splash state machine; see
[splash-screen-state-machine.md](../fixes/splash-screen-state-machine.md).

### Option C — texture override

Because the textures are referenced by `uri` (`spintopsplash.jpg`,
`std:distributor.jpg`, `std:publisher.jpg`, `title_background.jpg`), blanking the three
splash images could be done either by repointing resources in `Resource.dll` **or** by
dropping loose override files in the working directory — **if** the engine prefers loose
files over embedded ones. That precedence is an open question; see
[splash-patch-option-c.md](../fixes/splash-patch-option-c.md).

## Related

- [inner-game-payload.md](inner-game-payload.md) — the game that consumes these assets.
- [va-file-offset-mapping.md](va-file-offset-mapping.md) — VA↔offset math.
- [binaries-and-hashes.md](binaries-and-hashes.md).
