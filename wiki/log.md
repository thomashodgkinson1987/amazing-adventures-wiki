---
title: Wiki Operation Log
type: log
status: confirmed-static
updated: 2026-11-21
sources: []
---

# Wiki Operation Log

Append-only. Format: `## [YYYY-MM-DD] <ingest|query|lint|init> | <Title>` (see
[AGENTS.md §5](../AGENTS.md)).

---

## [2026-11-20] init | Create llm-wiki structure and schema

- Created `/workspace/llm-wiki/` with `raw/`, `raw/snapshots/`, and `wiki/`
  (`architecture/`, `fixes/`, `testing/`, `queries/`).
- Wrote schema [`AGENTS.md`](../AGENTS.md); created [`CLAUDE.md`](../CLAUDE.md) as a
  symlink to it.
- Wrote [`README.md`](../README.md).
- Naming convention `kebab-case.md`; linking convention = standard relative Markdown
  links (no wikilinks). Four statuses defined: `confirmed-runtime`, `confirmed-static`,
  `inferred`, `untested`.
- Verified all referenced stock hashes in `/workspace` read-only; all matched.

## [2026-11-20] ingest | FINDINGS.md (performance/cursor fix)

- Moved `/workspace/FINDINGS.md` → [`raw/FINDINGS.md`](../raw/FINDINGS.md) (untouched).
- Compiled pages: [overview.md](../wiki/overview.md),
  [architecture/drm-wrapper.md](../wiki/architecture/drm-wrapper.md),
  [architecture/inner-game-payload.md](../wiki/architecture/inner-game-payload.md),
  [architecture/rendering-pipeline.md](../wiki/architecture/rendering-pipeline.md),
  [architecture/resource-dll-xui.md](../wiki/architecture/resource-dll-xui.md),
  [architecture/va-file-offset-mapping.md](../wiki/architecture/va-file-offset-mapping.md),
  [architecture/binaries-and-hashes.md](../wiki/architecture/binaries-and-hashes.md),
  [fixes/cnc-ddraw-performance-fix.md](../wiki/fixes/cnc-ddraw-performance-fix.md),
  [fixes/proton-launch-options.md](../wiki/fixes/proton-launch-options.md).
- The cnc-ddraw fix is recorded as `confirmed-runtime` (user-reported working); all
  static architecture is `confirmed-static`.

## [2026-11-20] ingest | SPLASHSCREEN-PATCHING.md (splash analysis)

- Moved `/workspace/SPLASHSCREEN-PATCHING.md` → [`raw/SPLASHSCREEN-PATCHING.md`](../raw/SPLASHSCREEN-PATCHING.md) (untouched).
- Compiled pages: [fixes/splash-screen-state-machine.md](../wiki/fixes/splash-screen-state-machine.md),
  [fixes/splash-patch-option-a.md](../wiki/fixes/splash-patch-option-a.md),
  [fixes/splash-patch-option-a-prime.md](../wiki/fixes/splash-patch-option-a-prime.md),
  [fixes/splash-patch-option-c.md](../wiki/fixes/splash-patch-option-c.md),
  [testing/splash-patch-test-plan.md](../wiki/testing/splash-patch-test-plan.md),
  [testing/splash-visual-order-checklist.md](../wiki/testing/splash-visual-order-checklist.md),
  [testing/drm-integrity-open-questions.md](../wiki/testing/drm-integrity-open-questions.md).
- Re-verified the embedded overlay (`0x17b1a2`, size `0x165e00`) is byte-for-byte
  identical to `copy.exe` (SHA-256 `18e40b96…569e`) and confirmed the stock timing bytes
  (`0x126be8=00 00 a0 3f`, `0x12d348=00 00 e0 3f`, `0x12d2c0=00 00 20 40`) inside it.
- **Flagged contradiction #1:** the doc cites `properties/partner.xml` for
  `Publisher = SpinTop` / `Distributor = PopCap`, but that file contains no such fields
  (see [snapshots/partner.xml](../raw/snapshots/partner.xml)). Corrected evidence
  recorded as `readme.html` + `drm/custom/drm.xml`.

## [2026-11-20] ingest | Lightweight snapshots (config/XUI)

- Copied read-only snapshots into [`raw/snapshots/`](../raw/snapshots/): `config.txt`,
  `ddraw.ini`, `partner.xml` (originals untouched in `/workspace`).
- Extracted the `splashscreens.xui` UTF-8 text from `Resource.dll` and saved it as
  [`raw/snapshots/splashscreens.xui`](../raw/snapshots/splashscreens.xui). No binaries
  copied; binaries referenced by `/workspace` path + SHA-256 only.
- **Flagged contradiction #2 (minor):** the doc gives the XUI range as
  `0x25ee040..0x25ee303`; the exact `<xui>`…`</xui>` text actually spans
  `0x25ee068..0x25ee309` (doc range includes a ~40-byte binary prefix and ends 6 bytes
  short). Content identical.

## [2026-11-20] lint | Initial audit

- Scanned all `wiki/**/*.md` for the required frontmatter keys; all pages have
  `title`, `type`, `status`, `updated`, `sources`, and every `status` is one of the four
  allowed values.
- Checked every relative Markdown link resolves to an existing file; links to
  `/workspace` game files were verified as referenced by path/hash only (no copies).
- Orphans: none — every page is reachable from [index.md](../wiki/index.md), and each
  page has at least one incoming link.
- `index.md` coverage: complete — all 16 wiki pages listed with status.
- Findings recorded on the relevant pages
  ([splash-screen-state-machine.md](../wiki/fixes/splash-screen-state-machine.md),
  [resource-dll-xui.md](../wiki/architecture/resource-dll-xui.md),
  [va-file-offset-mapping.md](../wiki/architecture/va-file-offset-mapping.md)).
  These are documentation discrepancies in the raw sources, not wiki errors; raw sources
  are immutable so corrections live in the wiki.

## [2026-11-20] lint | Remediation — add frontmatter to log.md

- The initial audit found `wiki/log.md` missing the required frontmatter; added
  `title`/`type: log`/`status`/`updated`/`sources: []`.
- Re-ran the full lint: **0 broken links, 0 orphans, 0 frontmatter issues, complete
  `index.md` coverage**. 16 content pages (+ `index.md`, `log.md`), `queries/` empty.

## [2026-11-20] patch | Apply Option A splash patch to AmazingAdventures2.exe

- User explicitly approved modifying the game binary; this overrides the default
  "never touch game files" rule for this one change (documented in AGENTS.md §7).
- **Backup first:** copied pristine `/workspace/AmazingAdventures2.exe` →
  [`backups/AmazingAdventures2.exe.orig`](../backups/AmazingAdventures2.exe.orig)
  (`275b8b278b3ec38657635bca3cab02ac411e85e63d24f06139eb5f9d5d1c00e3`) and verified the
  hash matched before patching.
- **Patch applied** (three main Option A constants, each verified before writing):
  - `0x2a1d8a` splash dwell 1.25 s (`0x5279e8`): `00 00 a0 3f` → `00 00 00 00`
  - `0x2a84ea` state4→5 1.75 s (`0x52e148`): `00 00 e0 3f` → `00 00 00 00`
  - `0x2a8462` state5 2.5 s (`0x52e0c0`): `00 00 20 40` → `00 00 00 00`
- Result: exactly **6 bytes** changed; size unchanged (3,024,144); all three constants
  read back as `0.0f`. The two optional constants (`0x2a839a`, `0x2a8466`) were left
  alone.
- Hashes: before `275b8b27…c00e3`; after `d920cde4b531d61b377b5feead659ac13e8cb11a88cd9622761c2ed4ea4eb833`.
- Updated pages: [splash-patch-option-a.md](../wiki/fixes/splash-patch-option-a.md),
  [overview.md](../wiki/overview.md),
  [binaries-and-hashes.md](../wiki/architecture/binaries-and-hashes.md),
  [splash-patch-test-plan.md](../wiki/testing/splash-patch-test-plan.md),
  [drm-integrity-open-questions.md](../wiki/testing/drm-integrity-open-questions.md),
  [index.md](../wiki/index.md). Added `backups/` to AGENTS.md and the `patch` op.
- **Pending:** runtime verification — user will launch via Steam and report. On success,
  set Option A `status: confirmed-runtime` and close open question #1.

## [2026-11-20] ingest | Runtime test result — Option A splash patch CONFIRMED-RUNTIME

- New raw source saved: [`raw/SPLASH-PATCH-RUNTIME-TEST.md`](../raw/SPLASH-PATCH-RUNTIME-TEST.md)
  (user's report + verification of the captured game payload).
- **Reported result:** game launched cleanly (no DRM error); PopCap → SpinTop → Amazing
  Adventures flashed past in ~1 frame each; landed on a working main menu; reproduced on a
  second launch.
- **Verified the artifact** `copy-after-splash-screen-patch.exe`
  (`42edd1c477bf05c530d6eeee9754c27e3cb0e0a57df184e833b336defe978bee`): byte-identical to
  the **patched overlay**, and differs from `copy.exe` in exactly the 6 patched bytes.
  ⇒ the DRM re-extracts the payload verbatim and does not reject/repair the modification.
- **Status changes (all → `confirmed-runtime`):**
  [splash-patch-option-a.md](../wiki/fixes/splash-patch-option-a.md),
  [splash-screen-state-machine.md](../wiki/fixes/splash-screen-state-machine.md),
  [splash-patch-test-plan.md](../wiki/testing/splash-patch-test-plan.md),
  [splash-visual-order-checklist.md](../wiki/testing/splash-visual-order-checklist.md),
  [overview.md](../wiki/overview.md).
- **Open questions resolved:** #1 DRM overlay verification (no blocking check), #2
  zero-length fades work, #3 visual order (PopCap → SpinTop → Amazing Adventures).
  [#4–6 remain open](../wiki/testing/drm-integrity-open-questions.md).
- **Also updated:** [binaries-and-hashes.md](../wiki/architecture/binaries-and-hashes.md)
  (new `copy-after-splash-screen-patch.exe` hash),
  [drm-wrapper.md](../wiki/architecture/drm-wrapper.md) (verbatim-extraction confirmed),
  [option-a-prime.md](../wiki/fixes/splash-patch-option-a-prime.md) and
  [option-c.md](../wiki/fixes/splash-patch-option-c.md) (marked "not needed"),
  [index.md](../wiki/index.md).

## [2026-11-21] ingest | Document Steam verify-integrity caveat & restore path

- No new raw source; documentation/operational update.
- Added a **persistence caveat**: Steam *Verify integrity of game files* (or reinstall)
  restores the pristine `AmazingAdventures2.exe` (`275b8b27…c00e3`) and **undoes the splash
  patch**; the extracted payload is regenerated each launch, so it cannot be patched once.
- Restore path: `cp -p llm-wiki/backups/AmazingAdventures2.exe.orig /workspace/AmazingAdventures2.exe`.
- Noted that the cnc-ddraw fix (`ddraw.dll` / `ddraw.ini`) is not in the manifest and is
  normally left alone by verify.
- Updated pages: [splash-patch-option-a.md](../wiki/fixes/splash-patch-option-a.md),
  [binaries-and-hashes.md](../wiki/architecture/binaries-and-hashes.md),
  [overview.md](../wiki/overview.md).

## [2026-11-21] patch | Verify & remove extracted-game captures

- User approved deleting the two user-made captures `copy.exe` and
  `copy-after-splash-screen-patch.exe` from `/workspace` (both are fully derivable, and the
  wiki/game folder should stay binary-free).
- **Safety check before deletion:** `copy.exe` == overlay(`backups/AmazingAdventures2.exe.orig`)
  and `copy-after-splash-screen-patch.exe` == overlay(current patched
  `AmazingAdventures2.exe`); both matched byte-for-byte, so removal is lossless.
- **Deleted:** `/workspace/copy.exe` (`18e40b96…569e`),
  `/workspace/copy-after-splash-screen-patch.exe` (`42edd1c4…ebee`). Hashes retained in the
  wiki; regenerate with `open(...,'wb').write(open('AmazingAdventures2.exe','rb').read()[0x17b1a2:0x17b1a2+0x165e00])`.
- **Records updated** (captures documented as "verified and removed"):
  [binaries-and-hashes.md](../wiki/architecture/binaries-and-hashes.md),
  [inner-game-payload.md](../wiki/architecture/inner-game-payload.md),
  [drm-wrapper.md](../wiki/architecture/drm-wrapper.md),
  [va-file-offset-mapping.md](../wiki/architecture/va-file-offset-mapping.md),
  [splash-screen-state-machine.md](../wiki/fixes/splash-screen-state-machine.md),
  [splash-patch-option-a.md](../wiki/fixes/splash-patch-option-a.md),
  [drm-integrity-open-questions.md](../wiki/testing/drm-integrity-open-questions.md),
  [index.md](../wiki/index.md), [AGENTS.md](../AGENTS.md).
- Note: `raw/` still names the captures (immutable history); the wiki notes their removal.
- The pristine restore copy `backups/AmazingAdventures2.exe.orig` is **kept** (it is the
  derivation source and the revert path).

## [2026-11-21] lint | Prose-consistency pass on wiki/ + README.md

- Scope: `wiki/**/*.md` and `README.md` only; `raw/` left untouched (immutable).
- **overview.md** — rewrote the "Note on scope" callout: no longer says the machine can't
  run the game in a way that implies unverified results; now states the cnc-ddraw fix and
  the Option A splash patch (plus visual splash order and DRM non-rejection) are all
  `confirmed-runtime`.
- **architecture/drm-wrapper.md** — split the top bullet into *stock shipped EXE*
  (`275b8b27…c00e3`, pristine copy in `backups/`) vs. *active on-disk EXE* (now carries the
  6-byte Option A patch, `d920cde4…eb83`).
- **fixes/splash-patch-option-a.md** — retitled the `0.01` fallback from "Caveat
  (untested)" to "Fallback (kept for reference)"; noted `0.0f` was confirmed working at
  runtime, so the fallback is retained only for reference/reversibility.
- **architecture/binaries-and-hashes.md** — removed the duplicated `license.txt` mention in
  the "Other shipped files" line (also dropped `publisher.jpg` / `bass.dll`, which are
  already hashed in the table above).
- **README.md** — added `backups/` to the directory tree, and added the previously-missing
  `raw/SPLASH-PATCH-RUNTIME-TEST.md` entry.
- Re-ran the structural lint: 0 broken links, 0 orphans, 0 frontmatter issues.

## [2026-11-21] query | Steam verify-integrity survival & re-apply procedure

- Question: after Steam *Verify integrity of game files* (or a fresh install on a second
  Linux machine / Steam Deck), which fixes survive, which break, why a one-time
  `popcapgame1.exe` patch cannot stick, and the exact re-apply commands/paths/hashes/launch
  options for the performance fix + splash skip.
- Answered from: [fixes/cnc-ddraw-performance-fix.md](../wiki/fixes/cnc-ddraw-performance-fix.md),
  [fixes/proton-launch-options.md](../wiki/fixes/proton-launch-options.md),
  [fixes/splash-patch-option-a.md](../wiki/fixes/splash-patch-option-a.md),
  [architecture/drm-wrapper.md](../wiki/architecture/drm-wrapper.md),
  [architecture/inner-game-payload.md](../wiki/architecture/inner-game-payload.md),
  [architecture/binaries-and-hashes.md](../wiki/architecture/binaries-and-hashes.md),
  [testing/drm-integrity-open-questions.md](../wiki/testing/drm-integrity-open-questions.md);
  raw [FINDINGS.md](../raw/FINDINGS.md), [SPLASHSCREEN-PATCHING.md](../raw/SPLASHSCREEN-PATCHING.md),
  [SPLASH-PATCH-RUNTIME-TEST.md](../raw/SPLASH-PATCH-RUNTIME-TEST.md).
- Re-verified read-only on disk (2026-11-21): patched `AmazingAdventures2.exe` =
  `d920cde4...eb833`; backup `AmazingAdventures2.exe.orig` = `275b8b27...c00e3` with stock
  bytes `0000a03f`/`0000e03f`/`00002040` at `0x2a1d8a`/`0x2a84ea`/`0x2a8462`; `ddraw.dll` =
  `7fa0770e...5d77`; `Resource.dll` = `581c31cc...4db8`; `publisher.jpg` = `220e2878...72e5`.
- **Flags:** the "verify reverts the patch" caveat and "launch options / cnc-ddraw files do
  not transfer to a second machine" are `inferred` (reasoned from manifest membership; not
  runtime-tested here). Open question #5 (where the wrapper writes `popcapgameN.exe`,
  whether it re-extracts every launch) remains `untested`.
- **Filed:** [queries/steam-verify-integrity-and-reapply.md](../wiki/queries/steam-verify-integrity-and-reapply.md)
  (`type: query`, `status: inferred`); added to `index.md` Queries section.

## [2026-11-21] ingest | Add `tooling/` category & document amazing-adventures-toolkit

- **Schema extension:** added a new page kind **`tooling/`** — new directory, plus `tool`
  added to the `type` enum in the folder layout, naming conventions and frontmatter rules
  of [AGENTS.md](../AGENTS.md) and the tree in [README.md](../README.md).
- New page [tooling/amazing-adventures-toolkit.md](../wiki/tooling/amazing-adventures-toolkit.md)
  (`type: tool`, `status: confirmed-static`): documents the external cross-platform tool
  <https://github.com/thomashodgkinson1987/amazing-adventures-toolkit> that automates both
  fixes — install/remove cnc-ddraw and apply/remove the Option A splash patch — hash-verified
  against this wiki's offsets/hashes and reversible.
- Cross-linked from [overview.md](../wiki/overview.md),
  [fixes/cnc-ddraw-performance-fix.md](../wiki/fixes/cnc-ddraw-performance-fix.md) (also bumped
  `updated`),
  [fixes/splash-patch-option-a.md](../wiki/fixes/splash-patch-option-a.md) and
  [queries/steam-verify-integrity-and-reapply.md](../wiki/queries/steam-verify-integrity-and-reapply.md);
  listed in [index.md](../wiki/index.md) under a new **Tooling** section.
- No raw source added (the tool is an external project, referenced by URL only). The page's
  status is `confirmed-static`: the tool has not been run on this host, and the fixes it
  applies remain `confirmed-runtime`.

## [2026-11-21] ingest | Make the wiki standalone (own repo) — drop game files, genericize paths

- Preparing to publish this wiki as its **own public GitHub repository** (MIT). Removed all
  content the maintainer has no right to publish and all host-specific paths.
- **Deleted game-derived files (not bundled):** `raw/snapshots/config.txt`,
  `raw/snapshots/partner.xml`, `raw/snapshots/splashscreens.xui`, and the pristine
  `backups/AmazingAdventures2.exe.orig`. `raw/snapshots/ddraw.ini` is **kept** (cnc-ddraw's
  own file, MIT). Re-creation recipes added to
  [binaries-and-hashes.md](../wiki/architecture/binaries-and-hashes.md#re-creating-the-game-derived-snapshots);
  the `backups/` directory is gone and the protocol in `AGENTS.md` §7 updated (no binaries or
  game files in the repo; restore via Steam *Verify integrity*).
- **Paths genericized:** `/workspace` → `<GAME_DIR>` (defined in `README.md`), and
  `llm-wiki/backups/…` references replaced. Historical `raw/` notes and earlier `log.md`
  entries keep the original working path (immutable / append-only).
- **Pages updated:** `AGENTS.md`, `README.md`, `index.md`, `overview.md`,
  `architecture/binaries-and-hashes.md`, `architecture/drm-wrapper.md`,
  `architecture/resource-dll-xui.md`, `fixes/splash-patch-option-a.md`,
  `fixes/splash-screen-state-machine.md`, `fixes/splash-patch-option-c.md`,
  `testing/splash-patch-test-plan.md`, `testing/splash-visual-order-checklist.md`,
  `queries/steam-verify-integrity-and-reapply.md`, `tooling/amazing-adventures-toolkit.md`.
- Added `LICENSE` (MIT) and `.gitignore` (Obsidian UI state + the non-bundled game files).
- Re-ran the structural lint: 0 broken links, 0 orphans, full `index.md` coverage.

## [2026-11-21] lint | Full post-cleanup lint pass (links, paths, tooling, layout)

- **Links:** all relative Markdown links in `README.md`, `AGENTS.md`, and `wiki/**/*.md`
  resolve to existing files. No references to `backups/` or the removed
  `raw/snapshots/{config.txt,partner.xml,splashscreens.xui}` remain in any editable page.
- **Paths:** no `/workspace` or `llm-wiki/` paths remain outside the append-only history in
  `log.md` (and the immutable `raw/` notes); all game-folder references use `<GAME_DIR>`
  (defined in `README.md`).
- **tooling/:** `amazing-adventures-toolkit.md` has valid frontmatter (`type: tool`,
  `status: confirmed-static`, ISO date, sources incl. the external repo URL + `raw/` notes),
  is listed in `wiki/index.md` (§Tooling), and is cross-linked from `overview.md`,
  `index.md`, `architecture/binaries-and-hashes.md`, `fixes/cnc-ddraw-performance-fix.md`,
  `fixes/splash-patch-option-a.md`, and `queries/steam-verify-integrity-and-reapply.md`.
- **Layout parity:** `AGENTS.md` §1 and `README.md` now both list `LICENSE` and `.gitignore`;
  both trees match the actual repo contents. Added the missing `.gitignore` line to `AGENTS.md`.
- **Result:** 0 broken links (non-log), 0 frontmatter problems, 0 orphan pages, full
  `index.md` coverage. The only unresolved link targets are intentional history references in
  `log.md` to files deleted during the standalone-repo cleanup.

## [2026-11-21] init | Add CONTRIBUTING.md

- Added `CONTRIBUTING.md` (practical contributor guide): principles (evidence over
  recollection, text-only, immutable `raw/`, append-only `log.md`), getting started,
  making a change, ingest/query workflows, a pre-submit lint checklist mirroring
  [AGENTS.md](../AGENTS.md) §4.3, and commit-message conventions (Conventional Commits).
- Listed in the layout trees of both `README.md` and `AGENTS.md`; linked from
  `README.md` (new **Contributing** section).
- Link check: all 13 links resolve; no game files or `/workspace` paths added.
