# Amazing Adventures Around the World — LLM Wiki

A persistent, agent-maintained knowledge base for reverse-engineering and patching
*Amazing Adventures Around the World* (Steam **AppID 3530**) on Linux (CachyOS) via
Proton. It follows Andrej Karpathy's "LLM Wiki" pattern: a small set of immutable
**raw sources** compiled into an interlinked **wiki**.

## Layout

```
.
├── AGENTS.md   # the schema/rulebook every agent must read first (CLAUDE.md -> it)
├── CONTRIBUTING.md   # how to contribute (workflow, lint checklist)
├── README.md   # this file
├── LICENSE     # MIT
├── .gitignore  # Obsidian UI state + editor / OS cruft
├── raw/        # immutable source of truth (notes + third-party config)
│   ├── FINDINGS.md
│   ├── SPLASHSCREEN-PATCHING.md
│   ├── SPLASH-PATCH-RUNTIME-TEST.md
│   └── snapshots/   ddraw.ini
└── wiki/       # compiled, interlinked knowledge
    ├── index.md        # START HERE — catalog of every page
    ├── log.md          # append-only operation log
    ├── overview.md     # high-level project summary
    ├── architecture/   # how the game & binaries work
    ├── fixes/          # performance fix, splash patch options
    ├── tooling/        # external tools that automate the fixes
    ├── testing/        # test plans, checklists, open questions
    └── queries/        # filed answers to substantial questions
```

The game's own files stay untouched in the Steam game folder — written **`<GAME_DIR>`**
throughout, e.g. `~/.steam/steam/steamapps/common/Amazing Adventures Around the World` on
Linux or `C:\Program Files (x86)\Steam\steamapps\common\Amazing Adventures Around the World`
on Windows. Files are referenced by `<GAME_DIR>` path + SHA-256 hash and **never copied
here** — this repo is text-only. Everything in `raw/` is immutable.

## Quick start

Read `wiki/index.md` for the full catalog, or `wiki/overview.md` for the summary.

## How to prompt an agent

The agent reads `AGENTS.md` (auto-discovered via `CLAUDE.md`) for the rules, then
`wiki/index.md`. You can ask for one of three operations:

**Ingest** — add new knowledge:

> "Ingest `/path/to/new-notes.md` into the llm-wiki: put the raw file in `raw/`, then
> update the affected pages, `index.md`, and `log.md`."

**Query** — answer a question from the wiki:

> "Query the llm-wiki: what exactly do I have to patch to skip the splash screens, and
> how risky is it?" *(Agent reads `index.md` first, answers with citations, and offers
> to file the answer under `wiki/queries/`.)*

**Lint** — health-check the wiki:

> "Lint the llm-wiki: contradictions, stale `untested` claims, orphan pages, broken
> links, missing pages, and `index.md` drift."

Every operation appends an entry to `wiki/log.md`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add or update knowledge, the ingest /
query workflows, and the pre-submit lint checklist. Schema and rules live in
[AGENTS.md](AGENTS.md).

## License

Released under the [MIT License](LICENSE).
