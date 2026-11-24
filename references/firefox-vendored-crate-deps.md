# Dependency changes in crates Firefox vendors

Applies to any crate Firefox vendors: neqo, happy-eyeballs,
application-services, and the like. Every dependency change must match
Firefox's
[`Cargo.lock`](https://github.com/mozilla-firefox/firefox/blob/main/Cargo.lock).
Max insists on this and has had to repeat it.

## Before pushing

1. Diff the PR's `Cargo.lock` against the base branch and against Firefox's,
   for every added or changed package.
2. Pin each mismatch to the version Firefox ships:
   ```sh
   cargo update -p <crate> --precise <firefox-version>
   ```
3. Avoid crates and features Firefox does not already vendor. Example:
   `tracing-subscriber`'s `env-filter` feature is not vendored; use
   `filter::Targets` instead.

## What reaches Firefox

- Dev-dependencies never do.
- Normal dependencies of anything Firefox uses do, including `neqo-bin` (pulled
  in through the `http3server` test binary).
- Lockfile-only entries do too: `cargo vendor` copies every package in the
  lockfile, even one that is never built, such as `valuable` (pulled in by
  `tracing-log`, or by `tracing-core` with default features).

To check, build a scratch crate that depends on the changed crates the way
Firefox does, then read its fresh `Cargo.lock`.
