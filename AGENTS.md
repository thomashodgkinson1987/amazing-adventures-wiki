# LLM Wiki — Schema & Operating Rules

This directory is a **persistent, agent-maintained knowledge base** for the
*Amazing Adventures Around the World* (Steam AppID **3530**) reverse-engineering and
patching project, following Andrej Karpathy's "LLM Wiki" pattern.

The wiki is a **compiled** artifact: it is derived from immutable **raw sources** plus
verification work. An agent reads this file first, then `wiki/index.md`, before doing
anything.

> **Audience:** any AI agent or human. `CLAUDE.md` is a symlink to this file so that
> agent harnesses which look for `CLAUDE.md` pick up the rules automatically.

---

## 1. Folder layout

```
llm-wiki/
├── AGENTS.md              # this file — the schema/rulebook
├── CLAUDE.md              # symlink -> AGENTS.md (harness discovery)
├── README.md              # human-oriented orientation + how to prompt the agent
├── raw/                   # IMMUTABLE source of truth (never edit in place)
│   ├── FINDINGS.md             # prior session's performance-fix notes
│   ├── SPLASHSCREEN-PATCHING.md# prior session's splash analysis/patch notes
│   └── snapshots/              # third-party config only (game-derived files are not bundled)
│       └── ddraw.ini
└── wiki/                  # COMPILED, interlinked, editable knowledge
    ├── index.md           # master categorized catalog (entry point)
    ├── log.md             # append-only operation log
    ├── overview.md        # high-level project summary
    ├── architecture/      # how the game & binaries are built
    ├── fixes/             # modifications, patches and fixes
    ├── tooling/           # external tools that automate the fixes
    ├── testing/           # test plans, checklists, open questions
    └── queries/           # filed answers to substantial questions
```

The game's own files (`AmazingAdventures2.exe`, `Resource.dll`, `ddraw.dll`,
`bass.dll`, `config.txt`, `ddraw.ini`, `properties/`, `drm/`,
`readme.html`, `license.txt`, images, etc.) live in the game folder — written
**`<GAME_DIR>`** throughout (see [README.md](README.md) for its usual locations) — and are
**never** modified, moved, or deleted by wiki work, and **never copied into this
repository** (it holds text only). Only `FINDINGS.md` and `SPLASHSCREEN-PATCHING.md` were
moved into `raw/` (untouched).

### Naming conventions

- Wiki files use **`kebab-case.md`** (e.g. `splash-patch-option-a.md`).
- Sub-directories group by kind: `architecture/`, `fixes/`, `tooling/`, `testing/`, `queries/`.
- One concept per page. Prefer several focused pages over one giant page.
- Raw files keep their original names; snapshots keep the original basename.
- Directory names are lowercase; the top-level structural files (`AGENTS.md`,
  `CLAUDE.md`, `README.md`, `index.md`, `log.md`) are the only fixed names.

### Linking convention

**Use standard relative Markdown links everywhere** — e.g.
`[cnc-ddraw performance fix](../fixes/cnc-ddraw-performance-fix.md)`. Do **not** use
Obsidian `[[wikilinks]]`. Rationale: relative links render on GitHub, in editors, and
in Obsidian, and are checkable by the Lint pass. Links to raw sources use paths relative
to the wiki page, e.g. `[FINDINGS.md](../../raw/FINDINGS.md)`.

---

## 2. Standard frontmatter

Every page under `wiki/` **must** start with YAML frontmatter:

```yaml
---
title: Human readable title
type: architecture | fix | tool | testing | overview | query | index | log
status: confirmed-runtime | confirmed-static | inferred | untested
updated: YYYY-MM-DD
sources:
  - ../../raw/FINDINGS.md
  - ../../raw/snapshots/ddraw.ini
---
```

- **`title`** — short, human-readable.
- **`type`** — the page category; matches its folder (`overview`/`index`/`log` for the
  top-level files).
- **`status`** — the page's *strongest* claim. See §3.
- **`updated`** — ISO date of the last meaningful edit.
- **`sources`** — list of raw sources (paths relative to the page) and/or external
  references that back the page. May be empty (`[]`) only for pure index/log pages.

Pages may add optional keys (e.g. `related`, `applies_to`) but must not drop the five
above.

---

## 3. Verification discipline

The single most important rule: **never blur the line between what is proven and what is
guessed.** Every non-obvious claim should be attributable to one of four statuses, and
pages should mark individual claims inline when a page mixes levels.

| Status | Meaning | Evidence required |
|---|---|---|
| `confirmed-runtime` | Observed working on a real, running game. | A run on a real display/GPU + Proton, with what was observed. |
| `confirmed-static` | Proven from the files themselves, without running the game. | Reproducible static analysis: offsets, strings, hashes, disassembly. |
| `inferred` | A reasoned conclusion from confirmed facts; not directly observed. | Explicit "inferred from …" reasoning. |
| `untested` | Hypothesis or open question; no evidence yet. | Listed as a test to run. |

Rules:

1. **A page's `status` is its weakest load-bearing claim.** If a page contains an
   unsupported hypothesis that matters, set `status: untested` (or `inferred`) and mark
   the hypothesis as such.
2. Static facts are **not** runtime facts. `confirmed-static` proves the bytes/behaviour
   *in the binary*, not that the game runs that path on this machine. Say so.
3. Runtime claims require actually running the game. This host **cannot** run the game
   (no display/GPU; DRM needs Steam), so nearly everything here is at best
   `confirmed-static` or `inferred`. Protected values (see §5) are the exception: the
   performance fix was reported `confirmed-runtime` by the user.
4. Prefix uncertain claims in prose with a marker: **Confirmed-static:**,
   **Inferred:**, **Untested:**, **Confirmed-runtime:**.
5. When new evidence changes a status, update the page, bump `updated`, and add a
   `log.md` entry noting the change.

---

## 4. Core workflows

Every operation that changes the wiki ends by appending to `wiki/log.md` (format in §5).
Read-only work that changes nothing — notably a query that is **not** filed under
`wiki/queries/` (see §4.2) — does not modify `log.md`.

### 4.1 Ingest

Add a new raw source (a file dropped into `raw/`, a new analysis, a runtime report).

1. **Place & freeze the source.** Put it in `raw/` (or `raw/snapshots/`). Never edit a
   raw source after placement; treat `raw/` as immutable. If a raw source is wrong,
   record the correction in the wiki, don't rewrite history.
2. **Read it fully** and extract claims (facts, addresses, hashes, procedures).
3. **Create or update a source summary.** For a substantial source, create/refresh a
   summary page (under `testing/` for notes, or reference it from the pages it feeds)
   that lists what the source asserts and at what verification level.
4. **Update every affected page.** For each entity/system/fix/testing page the source
   touches, integrate the claims, update frontmatter `sources` and `updated`, and add
   cross-links both ways.
5. **Flag contradictions.** If the new source conflicts with existing pages, do not
   silently overwrite. Note both, mark which is stronger, and add a
   `contradiction` note (see §6).
6. **Update `wiki/index.md`** so every new/changed page is listed with a one-line
   summary and its status.
7. **Append to `wiki/log.md`.**

### 4.2 Query

Answer a question from the wiki.

1. **Read `wiki/index.md` first** to locate relevant pages (do not scan blindly).
2. Read the relevant pages and, where precision matters, the raw sources they cite.
3. **Synthesize an answer** that cites the specific wiki pages and raw sources used.
   Distinguish confirmed vs. inferred vs. untested in the answer.
4. If the answer is substantial or reusable, **offer to file it** under
   `wiki/queries/<kebab-case>.md` (with frontmatter, `type: query`, and a
   `status` reflecting the evidence). If filed, add it to `index.md`.
5. **Log only when filed.** Only append a `query` entry to `wiki/log.md` (op `query`) when
   a query is **filed as a new page under `wiki/queries/`** (step 4), or when the user
   **explicitly asks** to log it. Quick read-only lookups performed while working on other
   tasks must **not** modify `log.md`.

### 4.3 Lint

Audit the wiki for health. Report findings; fix safe issues directly.

1. **Contradictions** — pages that assert incompatible facts.
2. **Stale `untested` claims** — hypotheses that newer evidence has settled (or that are
   old and should be re-flagged).
3. **Orphan pages** — wiki pages with **no incoming links** from any other wiki page.
4. **Broken links** — relative links whose target file does not exist.
5. **Missing pages** — recurring concepts mentioned but lacking their own page.
6. **`index.md` drift** — pages present in `wiki/` but absent from `index.md` (or
   listed with a stale status).
7. **Frontmatter validity** — every page has the five required keys; `status` is one of
   the four allowed values; `updated` is a valid date.

Resolve what is clearly safe (add missing index entries, fix obvious link typos, add
back-links); list anything ambiguous for a human. **Append to `wiki/log.md`** (op
`lint`) with the findings.

---

## 5. `log.md` entry format

Append-only, newest at the bottom. Each entry:

```
## [YYYY-MM-DD] <ingest|query|lint|init> | <Title>

- bullet notes: what changed, which pages, which sources, any flags.
```

- Operations: `init`, `ingest`, `query`, `lint`, `patch` (extend only if truly necessary,
  and document the new op here). `patch` is used for an approved change to a file —
  patching a game binary or removing an approved user-added artifact; the entry must record
  the exact byte offsets or files changed, and the before/after SHA-256, plus a restore
  command. (No binary backups are kept in this repository; see §7.)

```
## [YYYY-MM-DD] patch | <what was changed and why>
```
- Dates are the real current date (ISO `YYYY-MM-DD`).

---

## 6. Contradiction & status-change notes

When sources disagree, add a short block to the affected page:

```
> **Contradiction (YYYY-MM-DD):** <source A> says X; <source B> says Y.
> Stronger: B (`confirmed-static`). Resolution: … / needs runtime test.
```

When a claim's status changes, note it inline and in the log entry:

```
> **Status change (YYYY-MM-DD):** was `inferred`, now `confirmed-static` because …
```

---

## 7. Safety & scope rules

1. **Never modify game files or binaries** in `<GAME_DIR>` (or anywhere outside this
   repository) unless the user has **explicitly approved a specific patch**. When an
   approved patch is applied: (a) keep a pristine copy for restore **outside** the repo
   (Steam *Verify integrity of game files* also restores it), (b) change only the
   documented bytes, (c) record the old and new SHA-256 plus the exact offsets in the
   affected pages and in `log.md`, and (d) provide a restore command.
2. **Never copy binaries or game-derived files** (`.exe`, `.dll`, large images,
   `config.txt`, `properties/partner.xml`, `splashscreens.xui`, …) into this repository —
   it holds **text only**. Reference them by their `<GAME_DIR>` path plus **SHA-256** hash.
   See [binaries-and-hashes.md](wiki/architecture/binaries-and-hashes.md) for how to
   re-create the game-derived snapshots locally.
3. **All new files live inside this repository.**
4. `raw/` is immutable once populated — corrections go in the wiki, not in raw.
   Game-derived files are never placed here at all (see rule 2).
5. Prefer primary evidence (bytes, offsets, hashes) over recollection.
6. Keep `index.md` and `log.md` current after every operation.
