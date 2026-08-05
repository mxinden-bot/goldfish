# Firefox's in-tree agent skills

Firefox ships a catalog of agent skills in the tree itself:
https://github.com/mozilla-firefox/firefox/tree/main/.agents/skills . Read
2026-08 via `raw.githubusercontent.com` (works from a goldfish-only GitHub
scope, no MCP access to the firefox repo needed: these are public files,
fetch the `raw.githubusercontent.com/mozilla-firefox/firefox/main/...` URL
with WebFetch). 19 skills plus a README as of that date; content is a moving
target, treat this as a snapshot and re-check the directory for additions.

Per the directory's own README, in-tree skills are reserved for "broadly
useful, tree-wide workflows"; component-specific ones belong in marketplace
plugins instead (`firefox-aidev-plugins`, `aidev-plugins`, both by mozilla).
Skill descriptions load into every session's context, so that boundary is
deliberate: do not propose adding a narrow one to `.agents/skills/` upstream.

`bug-filing`'s `allowed-tools` lists both `.agents/skills/bug-filing/...` and
`.claude/skills/bug-filing/...` paths, implying a `.claude/skills` mirror so
Claude Code auto-discovers these in a firefox checkout session. Not yet
confirmed by inspecting an actual checkout (none was available when this was
written); confirm next time one is, and fix this note if it needs adjusting.

## Prefer these over goldfish's own recipes, when a firefox checkout is present

- `bug-filing` / `file-good-first-bug`: prefilled `enter_bug.cgi` via
  `./mach file-info bugzilla-component`, same idea as
  `references/bugzilla-prefill-url.md` but mach-driven and always current on
  component names. Use the goldfish recipe only outside a firefox checkout
  (e.g. drafting a bug against neqo or another repo with no `mach`).
  https://github.com/mozilla-firefox/firefox/blob/main/.agents/skills/bug-filing/SKILL.md
  https://github.com/mozilla-firefox/firefox/blob/main/.agents/skills/file-good-first-bug/SKILL.md
- `find-reviewer`: fills `r=` from `./mach file-info reviewers` (Herald group
  data first, VCS history fallback). Complements, does not replace, the necko
  reviewer roster in CLAUDE.md's "Lessons learned": that note explains *why*
  (`mots.yaml`, module `core_necko`) and carries Max's personal picks, which
  the skill has no way to know.
  https://github.com/mozilla-firefox/firefox/blob/main/.agents/skills/find-reviewer/SKILL.md
- `profiler-analysis`: drives `profiler-cli` against a loaded
  profiler.firefox.com/share.firefox.dev profile with structured queries over
  stacks, markers, threads, and samples; explicitly never hand-parses profile
  JSON or fetches the profiler UI with WebFetch. Different surface from
  `references/glam-telemetry.md` (that one is GLAM's aggregate percentile
  API; this is a single profile's raw samples).
  https://github.com/mozilla-firefox/firefox/blob/main/.agents/skills/profiler-analysis/SKILL.md

## The rest

- `webspec-index` / `specmap` / `gecko-web-standard-implementation`: spec ->
  code (-> WPT) mapping. `webspec-index` is the underlying CLI (WHATWG, W3C,
  IETF, TC39 spec queries); `specmap` builds the full spec/code/test gap
  table; `gecko-web-standard-implementation` uses it to review WebIDL/C++
  against spec algorithm steps line by line.
- `jj-split` / `reorganize-patches-for-review`: Jujutsu-based commit
  splitting and pre-review patch-stack cleanup (squash, reorder, extract
  land-early chunks).
- `accessibility-frontend-review`: parallel accessibility-checklist and
  High-Contrast-Mode subagents, output as a Phabricator-ready comment.
- `media-patch-review`: fans `dom/media` review checklists out one per
  subagent, added lines only, advisory.
- `mozlint`: `./mach lint` usage and how to add a new linter (Python, or
  Rust via mozcheck).
- `fluent-migration`: writes `python/l10n/fluent_migrations/` recipes; the
  hard rule is no partial migrations and no hardcoded strings.
- `documentation`: the Sphinx/MyST `./mach doc` workflow.
- `js-perf-investigation`: SpiderMonkey perf work, hypothesis first,
  `samply`, Mann-Whitney evaluation over knob-tuning.
- `stmo`: `stmo-cli` for sql.telemetry.mozilla.org, `REDASH_API_KEY`-gated,
  files live under `artifacts/stmo/`.
- `android` / `android-new-module`: use `./mach gradle`, not `gradlew`;
  scaffolding steps for a new android-components module.
- `firefox-desktop-frontend`: HTML/JS/CSS conventions (design-system
  tokens, Fluent for new strings, logical properties, `./mach build faster`).

Full list with links:
https://github.com/mozilla-firefox/firefox/tree/main/.agents/skills
