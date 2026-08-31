# metap-docs

Design docs, roadmap, feature briefs, and audits for the `metap` project — split out of `../metap`
2026-08-31 (`docs/roadmap/54-docs-repo-split.md`), same underlying reason as the 3 splits before it
(frontend library — Phase 47, demo apps — Phase 51, low-code control-plane — Phase 52): `../metap`
should be a pure Rust library workspace, nothing that isn't code.

## What's here

Everything that was `../metap/docs/`, unchanged in structure — see `docs/architectures/index.md`
for the arc42/C4 architecture doc set, `docs/roadmap.md` for phased history, `docs/features/` for
feature briefs, `docs/audits/` for point-in-time reviews. Internal links between these files are
unaffected by the move (they were already relative within `docs/`).

## Scope — only `metap`'s own docs

Each other repo in this project keeps its **own** `docs/` — `../metap-lowcode/docs/`,
`../metap-demo-crm/docs/`, `../metap-demo-jira/docs/` all stay where they are, same as each repo
keeps its own `README.md`/`CLAUDE.md`. This repo is only what used to be `../metap/docs/`.

## A real gap from this split — code comments were not rewritten

Unlike the 3 code splits before this one, nothing here has a compiler to catch a broken reference.
`../metap`'s `crates/*.rs` doc comments (150 occurrences across 84 files), `CLAUDE.md` (24), and
`README.md` (10) all cite paths like `docs/roadmap.md` — a deliberate decision (confirmed with the
project owner before the split) left every one of those as-is rather than rewriting them to
`../metap-docs/docs/...`. Same "don't rewrite history" instinct already applied to
`docs/roadmap/*.md` phase entries, extended to code comments: rewriting ~184 references by script
risked touching unrelated comments, for a benefit (an old comment pointing at the right repo)
that's minor next to the risk. If you're reading a `docs/...` reference anywhere in `../metap`'s
code or its `CLAUDE.md`/`README.md`, resolve it against this repo instead
(`../metap-docs/docs/...`) — `../metap`'s own `README.md`/`CLAUDE.md` says this once, at the top,
rather than at every call site.
