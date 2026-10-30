# Watching Chrome networking and QUICHE

Recipe for "what is Chrome doing in networking?" updates, mapped to Max's
Firefox topics. All sources below are public and reachable from a web sandbox
(verified 2026-10).

## Max's topics to filter for

Happy Eyeballs v3 and connection racing, DNS (HTTPS RR and address hints,
stale/optimistic DNS, negative caching, DoH), QUIC transport (PMTUD, QUIC v2,
black-hole detection, ACK behavior, congestion control), fast UDP I/O
(GSO/GRO/recvmmsg), Alt-Svc and broken-QUIC handling, connection pooling,
RFC 9218 prioritization, WebTransport, ECH.

## Chromium: Gerrit REST API

```sh
curl -s "https://chromium-review.googlesource.com/changes/?q=project:chromium/src+dir:net+status:merged+after:2026-09-01&n=500&S=0&o=DETAILED_ACCOUNTS" | tail -c +5
```

- Strip the `)]}'` prefix (`tail -c +5`) before parsing JSON.
- Results cap at 500: page with `S=500`, `S=1000`, ...
- Open work: `status:open+-is:wip` and `status:open+is:wip`.
- Full commit message of one change:
  `changes/<n>?o=CURRENT_REVISION&o=CURRENT_COMMIT`.
- Drop noise by subject (`Roll`, `spanify`, `IWYU`, enterprise proxy,
  `disk_cache`, cookies), then grep for the topics above.
- Link a change as `https://chromium-review.googlesource.com/c/chromium/src/+/<n>`.
- Feature defaults (enabled or not) live in `net/base/features.cc`; Finch
  trials in `testing/variations/fieldtrial_testing_config.json`. The web
  session's `/home/user/chromium` checkout is shallow but current, so read
  those files locally.
- Shipping intents: `https://chromestatus.com/api/v0/features?q=<term>`
  (same `)]}'` prefix). Link as `https://chromestatus.com/feature/<id>`.

## QUICHE (Google's QUIC library used by Chrome, Google's servers, Envoy, Cronet)

The Chromium checkout does not populate the QUICHE submodule, so clone it:

```sh
git clone --shallow-since=<date> https://quiche.googlesource.com/quiche
```

- Chrome's pinned revision: `quiche_revision` in Chromium's `DEPS`.
- Rollout status: `quiche/common/quiche_feature_flags_list.h`, entries
  `QUICHE_FLAG(bool, name, internal_default, oss_default, doc)`. The
  `oss_default` is what Chromium gets; `internal_default` is Google's servers.
  "Off" can still be on via Finch or internal flags.
- Many commits say "No public description": internal-only work is invisible.
- Link a commit as `https://quiche.googlesource.com/quiche/+/<sha>` (short
  shas work).

## Baseline as of 2026-10 (re-check before relying on it)

- Chrome's HEv3 (`HappyEyeballsV3`) and QUIC `AsyncDnsJob` are off by default,
  the latter in a field trial. `bashi` (Takashi Toyoshima) owns most HEv3 CLs.
- Chrome enabled recvmmsg (`QuicUseReadMultiple`) by default; UDP GRO stayed
  off because it batched worse than recvmmsg in Finch.
- QUICHE: QUIC v2 (`quic_enable_version_rfcv2`) off everywhere and Chrome
  offers only RFCv1; no client-side PMTUD in Chrome (1250-byte default);
  `quic_priority_respect_incremental` off; ACK_FREQUENCY off; 5-RTO black-hole
  detection on; BBR default for Google's servers, not for Chromium.
