# Reading network logs from a Firefox profile

How to capture Necko and neqo logs into a Firefox profile and read them with
`profiler-cli`. For a profile's stacks and markers, Firefox's in-tree
`profiler-analysis` skill drives the same CLI: see
`references/firefox-agent-skills.md`.

## Install `profiler-cli` in a web sandbox

The web sandbox has no `~/.mozbuild`, `./mach bootstrap` is slow and heavy, and
`profiler-cli` is a toolchain artifact. This installs it to `~/.mozbuild/bin` in
seconds:

```sh
cd /home/user/firefox && ./mach python -c "
import sys; sys.path.insert(0,'python/mozboot')
from pathlib import Path
from mozboot.base import BaseBootstrapper
b=BaseBootstrapper(no_interactive=True)
b.state_dir=Path.home()/'.mozbuild'
b.ensure_npm_tools()"
export PATH=~/.mozbuild/bin:$PATH
```

Without the `state_dir` line it dies with a `NoneType / str` TypeError.
Verified 2026-11 (profiler-cli 0.15.0).

## neqo logs reach shipped builds

neqo's debug and trace logs DO reach shipped builds (verified 2026-11 on
Nightly 163: `V/neqo_transport` lines in a profile). neqo logs through
`tracing`, which the `log` crate's `release_max_level_info` cap in
[`toolkit/library/rust/shared/Cargo.toml`](https://github.com/mozilla-firefox/firefox/blob/main/toolkit/library/rust/shared/Cargo.toml)
does not touch. Module syntax: `neqo_transport::*:5`.

qlog still has no HTTP/3 SETTINGS. For a server's SETTINGS, use
`neqo_http3::*:5` or a Chrome `chrome://net-export` log.

## Congestion control from a profile

Use this `about:logging` string with output to the profiler:

```
timestamp,sync,neqo_transport::*:5,neqo_glue::*:5,neqo_http3::*:4,happy_eyeballs::*:5,nsHttp:5,nsHostResolver:5
```

Drop `nsSocketTransport`: it is pure noise. Happy Eyeballs C++ logs only at
`nsHttp:5`.

For a download, the client cwnd is meaningless (the client only sends ACKs).
Read the receive side instead:

- `[Recvd-ap] received N`: holes in the packet numbers (never received vs.
  late);
- `RTT latest=`: RTT samples;
- `PTO: marking packet N lost`.

Verified 2026-11.
