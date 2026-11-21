# Contributing

This is an **agent-maintained knowledge base** ("LLM wiki") for reverse-engineering and
patching *Amazing Adventures Around the World* (Steam AppID 3530) on Linux/Proton. The
schema, verification rules, and workflows live in **[AGENTS.md](AGENTS.md)** — read it
first. This file covers the practical side of contributing.

## Principles

- **Evidence over recollection.** Every non-obvious claim must be attributable to one of
  four statuses: `confirmed-runtime`, `confirmed-static`, `inferred`, `untested`
  (see [AGENTS.md §3](AGENTS.md)). Mark claims inline when a page mixes levels.
- **Text only.** This repo holds analysis, offsets, hashes and procedures — **never game
  files or binaries** (`AmazingAdventures2.exe`, `Resource.dll`, `*.dll`, `config.txt`,
  `properties/partner.xml`, `splashscreens.xui`, …). Reference them by their `<GAME_DIR>`
  path plus SHA-256 hash. See
  [binaries-and-hashes.md](wiki/architecture/binaries-and-hashes.md) for how to
  re-create the game-derived snapshots locally.
- **`raw/` is immutable.** Once a source is in `raw/`, never edit it. Corrections go in
  the compiled wiki.
- **`wiki/log.md` is append-only.** Never rewrite history.

## Getting started

```sh
git clone git@github.com:thomashodgkinson1987/amazing-adventures-wiki.git
cd amazing-adventures-wiki
```

There is no build step. The wiki is plain Markdown with YAML frontmatter; any editor or
[GitHub](https://github.com) renders it. Start from:

- **[wiki/index.md](wiki/index.md)** — the master catalog and entry point.
- **[AGENTS.md](AGENTS.md)** — the schema and operating rules (frontmatter, statuses,
  lint checks, log format).

## Making a change

1. **Read `wiki/index.md`** to find the pages your change touches (don't scan blindly).
2. **Edit the affected `wiki/**/*.md` pages.** Keep one concept per page and use
   **kebab-case.md** filenames.
3. **Use standard relative Markdown links** (e.g. `` `[splash patch](../fixes/splash-patch-option-a.md)` ``
   from a `wiki/` sub-page), never Obsidian `[[wikilinks]]`. They must render on GitHub.
4. **Update the frontmatter** — bump `updated` to the ISO date and adjust `status` /
   `sources` if your change affects them.
5. **Update `wiki/index.md`** if you add or rename a page (one-line summary + status).
6. **Append a `wiki/log.md` entry** describing what changed, which pages, and why.

### Adding a raw source (ingest)

1. Put the new file in `raw/` (or `raw/snapshots/` if it is a third-party config such as
   `ddraw.ini`). Treat it as **frozen** from that point on.
2. Read it fully and extract its claims.
3. Integrate the claims into the relevant wiki pages (update frontmatter `sources` and
   `updated`, add cross-links both ways).
4. Flag any contradictions with existing pages — record both, mark which is stronger
   (see [AGENTS.md §6](AGENTS.md)).
5. Add the source to the **Raw sources** table in `wiki/index.md`.
6. Append a `log.md` entry (`op: ingest`).

> Game-derived files (e.g. `config.txt`, `properties/partner.xml`,
> `splashscreens.xui`) are **not** accepted — see *Text only* above. Add a re-creation
> recipe to
> [binaries-and-hashes.md](wiki/architecture/binaries-and-hashes.md) instead.

### Filing a query

If you answer a substantial, reusable question, file it under
`wiki/queries/<kebab-case>.md` (with frontmatter, `type: query`, and a `status`
reflecting the evidence), then add it to `wiki/index.md` and append a `log.md` entry
(`op: query`). See [AGENTS.md §4.2](AGENTS.md).

## Before opening a pull request

Run the lint checklist from [AGENTS.md §4.3](AGENTS.md):

- [ ] No broken relative links (every target file exists).
- [ ] Every page has valid frontmatter (`title`, `type`, `status`, `updated`, `sources`);
      `status` is one of the four allowed values; `updated` is a valid ISO date.
- [ ] No orphan pages (every wiki page has an incoming link from another wiki page).
- [ ] Every page is listed in `wiki/index.md`.
- [ ] Use `<GAME_DIR>` (defined in [README.md](README.md)) for game-folder references.
- [ ] No game files or binaries added.
- [ ] A `wiki/log.md` entry was appended.

## Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/) so history stays
readable, e.g.:

```
docs(fixes): clarify splash-patch restore via Steam Verify integrity
docs(architecture): add re-creation recipe for splashscreens.xui
```

Commits are **signed**; if you fork and submit a PR, plain commits are fine — the
maintainer signs the merge.

## License

By contributing, you agree your contributions are licensed under the
[MIT License](LICENSE).
