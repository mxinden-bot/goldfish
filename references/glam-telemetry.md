# Reading Firefox telemetry from GLAM

How to pull real Glean metric data (distributions, percentiles, label shares)
from GLAM's public JSON API. Used to source the networking talk decks. Verified
2026-07.

## The API

Undocumented but public, the same endpoint GLAM's own frontend calls. Fetch it
server-side (no CORS):

```
POST https://glam.telemetry.mozilla.org/api/v1/data/
Content-Type: application/json

{"query":{"product":"fog","app_id":"nightly","os":"*","ping_type":"*",
          "probe":"<probe>","aggregationLevel":"version","versions":6}}
```

Response is `{ "response": [ row, ... ] }`, one row per version (or build_id).
Take the last row for the latest version.

## Parameters, and the traps

- `product`: `"fog"` for desktop Firefox. Fenix (Android) is a different product;
  `app_id:"fenix"` under `product:"fog"` returns **HTTP 400**. So this API in
  practice gives you **desktop only** (`app_id` = `"nightly"` or `"release"`).
- `app_id`: `"nightly"` or `"release"`. Release has vastly more samples; nightly
  shows in-development features enabled more often.
- `os`: **the one that bites you.** `"*"`, or `"Windows"` / `"Linux"` / `"Mac"`.
  For any metric gated on an OS feature you MUST filter, because `"*"` is
  dominated by whatever OS has the most users (Windows). Example: GSO/GRO UDP
  offload is off on Windows, so receive-coalescing on `os:"*"` reads
  1/1/3/14/18 (P50..P99.9), while `os:"Linux"` shows the real 1/2/10/27/53.
  Always ask "is this metric OS-gated?" before trusting `"*"`. `"Mac"` often
  404s (too little data).
- `probe`: the Glean metric name, dots to underscores, category prefixed. Metric
  `http_3.udp_datagram_segments_received` in category `networking` becomes
  `networking_http_3_udp_datagram_segments_received`. Happy Eyeballs metrics are
  `netwerk_happy_eyeballs_*`. Confirm the prefix returns 200; it varies by
  category.
- `aggregationLevel`: `"version"` (one row per version) or `"build_id"` (finer,
  for time series). `versions:N` picks how many recent versions.

## Reading a row

- `.percentiles`: `{"1":..,"5":..,"25":..,"50":..,"75":..,"95":..,"99":..,"99.9":..}`.
  This is what you want for timing / size / count distributions.
- `.non_norm_histogram`: `{bucket: client_count}`. For **label shares** of a
  labeled_counter (ECN path capability, h3_discovery), sum `bucket * count`
  across buckets to estimate a per-event count, then divide. `sample_count` is
  per-client reach and is misleading for event counts, hence the histogram.
- `.version`, `.metric_key` (non-null only for labeled metrics).

## Know the unit before you plot

Metric definitions live in the Firefox tree:
- `netwerk/metrics.yaml` (category `networking`, e.g. `http_3`)
- `netwerk/protocol/http/metrics.yaml` (happy_eyeballs)

Read the YAML for `type` (memory_distribution = bytes, custom_distribution =
integer), `unit`, and description before charting.

## On-slide source link

`https://glam.telemetry.mozilla.org/fog/probe/<probe>/explore` (the reader can
re-select OS and channel there).

## Minimal fetch (Node 18+, no deps)

```js
const API = 'https://glam.telemetry.mozilla.org/api/v1/data/';
async function pct(probe, { app_id = 'nightly', os = '*' } = {}) {
  const res = await fetch(API, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: {
      product: 'fog', app_id, os, ping_type: '*', probe,
      aggregationLevel: 'version', versions: 6 } }),
  });
  if (!res.ok) throw new Error(`${probe} [${os}]: HTTP ${res.status}`);
  const rows = (await res.json()).response || [];
  const last = rows[rows.length - 1];
  return { version: last.version, percentiles: last.percentiles };
}
// OS-gated metric: filter, do not use "*"
await pct('networking_http_3_udp_datagram_segments_received', { os: 'Linux' });
```

On Node 22 in the web sandbox, `HTTPS_PROXY` is not honored for `fetch`, but
GLAM is reachable directly anyway.

## Second source: performance.mozilla.org for version-over-time

For page-load metrics over time (HTTP-version share, time-to-request-start, DNS
lookup times), `performance.mozilla.org/networking.html` embeds Redash chart
configs with per-query `api_key`s in its inline HTML. Pull the CSV directly:

```
curl "https://sql.telemetry.mozilla.org/api/queries/<ID>/results.csv?api_key=<KEY>"
```

Query IDs: 113403 HTTP-version share (H3 %), 114600 desktop time-to-request-start
by version, 115316 Fenix TTRS, 121688 DNS lookup P75/P95 DoH vs OS. Keys rotate:
on a 403, re-scrape the page HTML for the current key.
