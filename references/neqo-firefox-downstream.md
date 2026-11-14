# Bumping neqo in Firefox

Recipe for landing a new neqo release (or rev) in Firefox. Verified 2026-08 on
the neqo v0.31.0 downstream bump.

## Steps

1. Bump the five neqo crate `tag`/`rev`s in
   [`netwerk/socket/neqo_glue/Cargo.toml`](https://searchfox.org/mozilla-central/source/netwerk/socket/neqo_glue/Cargo.toml)
   and
   [`netwerk/test/http3server/Cargo.toml`](https://searchfox.org/mozilla-central/source/netwerk/test/http3server/Cargo.toml).
2. Run `./mach vendor rust`. It updates `Cargo.lock`, re-vendors
   `third_party/rust/neqo-*`, and regenerates the cbindgen FFI header.
3. Add or extend the cargo-vet audit (below).
4. Fix the glue if the API moved:
   [`netwerk/socket/neqo_glue/src/lib.rs`](https://searchfox.org/mozilla-central/source/netwerk/socket/neqo_glue/src/lib.rs)
   is the client side (cbindgen feeds
   [`Http3Session.cpp`](https://searchfox.org/mozilla-central/source/netwerk/protocol/http/Http3Session.cpp));
   the `http3server` test binary uses neqo's server-side API.
5. Verify with `./mach build`.

Any dependency change must also match Firefox's
[`Cargo.lock`](https://github.com/mozilla-firefox/firefox/blob/main/Cargo.lock): see the
"Crates Firefox vendors" note in CLAUDE.md.

## cargo-vet audits for git-vendored crates

`mtu` is a neqo workspace crate and needs a fresh git audit in
[`supply-chain/audits.toml`](https://searchfox.org/mozilla-central/source/supply-chain/audits.toml)
on each bump.

Git audits must be **version-to-rev**, never rev-to-rev
(`@git:old -> @git:new`): afranchuk flagged this on
[D321289](https://phabricator.services.mozilla.com/D321289#inline-1719755).

- Crate version unchanged: extend the existing `<ver> -> <ver>@git:<rev>`
  audit's endpoint to the new rev:
  ```sh
  ./mach cargo vet certify <crate> <ver> <ver>@git:<newrev> --criteria safe-to-deploy --accept-all
  ```
- Published version also moved (e.g. `mls-rs-provider-sqlite` 0.22.0 -> 0.23.0
  during an mls-platform-api bump): split it into an importable `<old> -> <new>`
  version delta plus a `<new> -> <new>@git:<rev>` version-to-rev git delta
  (`importable = false`).

## Out of disk at the libxul link

In the web sandbox, `./mach build` runs out of disk at the `libxul.so` link,
and the symptom misleads: a bare `clang++: error: linker command failed with
exit code 1` that looks like a missing symbol. The tell is a `No space left on
device` on a nearby target (e.g. the geckodriver symlink) in the same failure.

Fix: delete `obj-*/x86_64-unknown-linux-gnu/release/incremental` (~4.7G in
2026-08). The link consumes the `.rlib`/`.o` products in `deps/`, not the
incremental cache, so this forces no Rust recompile; `./mach build binaries`
then goes straight to the link. Also worth clearing when tight:
`/root/.cargo/git` and `/root/.cargo/registry/cache` (safe once
`./mach vendor rust` has run).
