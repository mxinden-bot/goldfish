# Index

Catalog of this memory repository, one line per file. Keep it current: when you
add, move, or remove a file, update this index in the same commit.

## Memory and rules

- `CLAUDE.md`: entry point. Hard rules plus the working memory; auto-loaded at session start.
- `README.md`: what this repo is, for humans browsing it.
- `portable.md`: distilled, paste-ready core for chats that do not auto-load CLAUDE.md.

## Routines

- `routines/memory-upkeep.md`: scheduled lint pass that opens a PR with proposed memory fixes.

## References

- `references/bugzilla-prefill-url.md`: build a pre-filled `bugzilla.mozilla.org` enter_bug.cgi URL.
- `references/necko-triage.md`: run a Necko triage pass (untriaged queue query, criteria, mis-filings, regressor lookup).
- `references/neqo-cargo-test-in-web-sandbox.md`: build and `cargo test` mozilla/neqo in a web sandbox (git blocked, NSS needed).
