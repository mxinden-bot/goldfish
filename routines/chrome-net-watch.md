# Routine: Chrome networking watch

A scheduled report on what Chrome and QUICHE changed in networking since the
last run, filtered to Max's Firefox topics. The logic lives here in git; the
scheduler's prompt only points at this file (see "How this is wired up").

Cadence: weekly. Sources, query recipes and the topic filter:
`references/chrome-net-watch.md`. Read it first.

## Before anything: obey the hard rules

Read CLAUDE.md and follow every hard rule in it, above all: link every CL,
commit, bug and feature you mention, and never use an em dash.

## Steps

1. Window: the last 8 days (one day of overlap with the previous run).
2. Gather candidates:
   - Chromium, through the Gerrit REST API: changes under `dir:net` merged in
     the window, and open changes under `dir:net` updated in the window. Do
     not rely on a local Chromium checkout; fetch files through Gitiles, e.g.
     `https://chromium.googlesource.com/chromium/src/+/main/net/base/features.cc?format=TEXT`
     (base64).
   - QUICHE: clone with `--shallow-since` the window start, list new commits,
     and diff `quiche/common/quiche_feature_flags_list.h` across the window.
   - chromestatus: QUIC, HTTP/3, DNS, WebTransport, ECH or connection racing
     features updated in the window.
3. Read the commit message of every candidate, then keep only changes that
   change behavior, ship or unship something, or reveal field data or design
   intent in one of the projects below. Drop everything else, including:
   autorolls, refactors, renames, test-only, build, metrics plumbing without
   results, cherry-picks, reverts of trivial changes, enterprise and policy
   work, iOS-only or ChromeOS-only plumbing, and MoQ. When in doubt, drop it.
4. Group what survives by project. Merge a series of CLs on one change into a
   single item.

## Projects

Report under these headings, in this order, and only the ones with something
to say:

1. Happy Eyeballs v3 and connection racing (`HttpStreamPool`, QUIC
   `AsyncDnsJob`, TCP vs. QUIC racing).
2. DNS (HTTPS RR and address hints, optimistic or stale DNS, caching, DoH).
3. QUIC transport and UDP I/O (QUICHE transport, ACKs, congestion control,
   PMTUD, versions, recvmmsg/GRO/GSO, socket changes).
4. Reaching HTTP/3 (Alt-Svc, broken-QUIC handling, black holes, connection
   migration).
5. Pooling, preconnect and prioritization.
6. WebTransport.
7. Shipping and defaults: features enabled or disabled by default, Finch
   launches, QUICHE flag flips, chromestatus intents.

## Output

The final reply is the report; it becomes the push and email summary.

- **Highlights**: at most three bullets, the changes most relevant to
  Firefox. Each: what changed, its link, and one clause on what it means for
  Firefox.
- **Then one section per project** (headings above). Start each with one line
  on where the project stands, then at most three bullets. Each bullet: a
  short plain summary, the link(s), and the status (landed, in review, behind a
  flag, enabled by default).
- At most about 12 bullets in total. Prefer fewer, better items to complete
  coverage.
- If nothing relevant happened this week, reply with one line saying so.
- Write nothing to this repo on a normal run. If a fact in the baseline
  section of `references/chrome-net-watch.md` changed (a feature or flag
  default flipped, a version enabled), open one small PR updating that
  baseline, authored as Max, per the PR rules in `routines/memory-upkeep.md`.

## How this is wired up

The schedule lives in the web UI: weekly, Mondays at 04:37 UTC, a fresh session
each time, with push and email notifications. A routine session starts with no
repository and no `add_repo` tool, so its prompt clones this repo over HTTPS
(`git clone --depth 1 https://github.com/mxinden-bot/goldfish /home/user/goldfish`),
then points here. The prompt also carries a short fallback version of the recipe
in case the clone fails. Change behavior by editing this file, not the prompt.
