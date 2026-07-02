# Running `cargo test` for mozilla/neqo in a Claude Code web sandbox

As of 2026-07, the web sandbox blocks the raw git protocol through the proxy
(HTTP 403 on `git clone`/`git fetch`), but allows HTTPS to
`codeload.github.com`, `raw.githubusercontent.com`, and the crates.io sparse
index. neqo pulls its crypto from the `nss-rs` crate via a git dependency, and
needs a recent NSS (`>=` the version in `nss-rs`'s `min_version.txt`, which was
`3.121`). Debian's NSS is older than that and there is no system NSS, so
pkg-config cannot satisfy it. This recipe works around both.

Toolchain already present: `cargo`, `clang`/`libclang-18`, `ninja`, `cc`,
`python3`, `pip3`. Missing by default: `gyp`, `hg`, system NSS.

## Recipe

1. Get the neqo source without git (tarball over HTTPS). `<owner>` is e.g.
   `martinthomson` for a PR branch, `<ref>` is a branch name or commit SHA:
   ```sh
   curl -sSL "https://codeload.github.com/<owner>/neqo/tar.gz/<ref>" -o neqo.tgz
   mkdir neqo && tar xzf neqo.tgz -C neqo --strip-components=1
   ```

2. Vendor the `nss-rs` git dependency (the only git dep) at the exact revision
   pinned in `Cargo.lock`:
   ```sh
   rev=$(grep -m1 'nss-rs#' neqo/Cargo.lock | sed 's/.*nss-rs#//')
   curl -sSL "https://codeload.github.com/mozilla/nss-rs/tar.gz/$rev" -o nss-rs.tgz
   mkdir nss-rs && tar xzf nss-rs.tgz -C nss-rs --strip-components=1
   ```

3. Redirect the git dep to the vendored copy. Append to the workspace root
   `neqo/Cargo.toml` (absolute paths):
   ```toml
   [patch."https://github.com/mozilla/nss-rs"]
   nss-rs = { path = "/abs/path/nss-rs" }
   test-fixture = { path = "/abs/path/nss-rs/test-fixture" }
   ```

4. Build NSS. The firefox checkout at `/home/user/firefox` ships an NSS source
   tree new enough (was `3.126`, requirement was `3.121`). Copy it out (do not
   build in-tree: the build writes `out/` and a sibling `dist/`), with NSPR as a
   `../nspr` sibling, then build static libs:
   ```sh
   pip3 install gyp-next            # gyp is the only missing build tool
   cp -r /home/user/firefox/security/nss  build/nss
   cp -r /home/user/firefox/nsprpub       build/nspr
   ( cd build/nss && PATH=/usr/local/bin:$PATH ./build.sh \
       -Ddisable_tests=1 -Ddisable_dbm=1 -Ddisable_libpkix=1 \
       -Ddisable_ckbi=1 -Ddisable_fips=1 --opt --static )
   ```
   Fallback if the firefox NSS is absent or too old: `ftp.mozilla.org` is
   reachable over HTTPS, so download an NSS "with-nspr" release tarball from
   `https://ftp.mozilla.org/pub/security/nss/releases/` (it extracts to the same
   `nss/` + `nspr/` sibling layout).

5. Point `nss-rs`'s `build.rs` at the prebuilt NSS and run the tests:
   ```sh
   export NSS_DIR=/abs/build/nss          # must be absolute
   export NSS_PREBUILT=1                   # skip the rebuild
   export LIBCLANG_PATH=/usr/lib/llvm-18/lib  # for bindgen
   export PATH=/usr/local/bin:$PATH        # pip-installed gyp
   cargo test -p neqo-transport --lib [test_name]
   ```
   First compile is ~1 to 2 minutes (bindgen plus deps from crates.io, which is
   reachable). Verified end to end: `neqo-transport` unit tests build and run.

## Gotchas

- `--static` is required. `cargo test` uses the dev/debug profile, and `nss-rs`
  `build.rs` static-links NSS in debug builds; without `--static` the static
  `.a` libs are absent and linking fails.
- `NSS_DIR` must be absolute. The `NSS_DIR` code path skips the NSS version
  check (only the pkg-config path enforces `min_version.txt`), so a
  "Beta"/customized NSS from firefox is accepted.
- This is local-only scaffolding: the `[patch]` block and any test edits must
  not be committed to neqo.

## Running clippy (same NSS setup)

neqo's CI is `cargo clippy --locked --all-targets --no-default-features --
-D warnings`, run per crate. With the NSS env from step 5 exported
(`NSS_DIR`, `NSS_PREBUILT`, `LIBCLANG_PATH`, `PATH`), the exact CI check runs
locally, e.g.:

```sh
cargo clippy --locked --all-targets --manifest-path neqo-http3/Cargo.toml \
  --no-default-features -- -D warnings
```

Shortcut: `neqo-common` has no NSS dependency, so `cargo clippy -p neqo-common`
(and 32-bit `rustup target add i686-unknown-linux-gnu` +
`--target i686-unknown-linux-gnu`) needs none of the NSS scaffolding. It holds
the `usize`/`u64` conversion code and the pointer-width assumptions, so it is
the cheap first check for 32-bit / cast breakage.

Verified 2026-07: with NSS built (gyp via `pip install gyp-next`; if the
firefox checkout is absent, `nss-rs` will clone NSS+NSPR itself via `hg`, which
needs `pip install mercurial` and, in the web sandbox, a `~/.hgrc`
`[http_proxy] host = ${HTTPS_PROXY#http://}` pointing at the egress proxy whose
PORT changes every container restart), clippy runs clean per crate.
