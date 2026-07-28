# Syncing a mxinden-bot fork's main with upstream, in a web sandbox

How to fast-forward the `mxinden-bot` fork's `main` to match an upstream repo
(e.g. `mozilla/neqo`, `quinn-rs/quinn`, `mozilla-firefox/firefox`) from a
Claude Code web session, per CLAUDE.md hard rule 4. Verified 2026-07 syncing
neqo, quinn, and firefox.

## The proxy problem

`origin` is rewritten through a local git proxy scoped to `mxinden-bot/*`
(`insteadOf = https://github.com/` in the global gitconfig), so a genuine
`upstream` remote added the normal way cannot reach `github.com/mozilla/...`
or any other non-`mxinden-bot` path. Reach the real upstream directly,
per-command, bypassing the global config:

```sh
GIT_CONFIG_GLOBAL=/dev/null git fetch <upstream-url> main
```

## The shallow-clone problem

The session repos are shallow clones. A plain `--depth 1` fetch of upstream's
tip has no shared history with the fork's current `origin/main`, so the later
push is rejected as non-fast-forward, even when the fork is genuinely just
behind with no real divergence.

`--shallow-exclude=<fork-tip-sha>` looks like the fix but fails against this
proxy (`fatal: expected 'acknowledgments'`).

## What works

Fetch shallow, then deepen in a loop with a growing depth until the fork's
tip is an ancestor of the fetched history:

```sh
GIT_CONFIG_GLOBAL=/dev/null git fetch <upstream-url> main --depth 1
# repeat with a growing N until this succeeds:
GIT_CONFIG_GLOBAL=/dev/null git fetch <upstream-url> main --deepen=<N>
git merge-base --is-ancestor origin/main FETCH_HEAD && echo "caught up"
```

Once it succeeds, `origin/main` is a true ancestor of the fetched upstream
tip, so the push is a plain fast-forward, no force needed:

```sh
git push origin FETCH_HEAD:refs/heads/main
```

Verified 2026-07: mozilla-central was about 1300 commits behind, and the three
deepen steps needed only grew `firefox/.git` from 1.1G to 1.2G, so this stays
cheap even for a large repo.
