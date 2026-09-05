# Running Firefox in a Claude Code web sandbox

Goal: actually launch Firefox and observe network-stack behavior (DNS, TRR/DoH,
HTTPS RR, connection handling) from a web session, without a local build.
Verified 2026-07 reproducing the HTTPS RR AliasMode bug
(https://bugzilla.mozilla.org/show_bug.cgi?id=1869075).

Do not `./mach build`: a full build is tens of minutes and fragile here. A
prebuilt Nightly has the same network stack and is enough for necko behavior.

## Get a binary

```sh
cd "$SCRATCH"   # your scratchpad dir
curl -sL -o ff.tar.xz 'https://download.mozilla.org/?product=firefox-nightly-latest-ssl&os=linux64&lang=en-US'
tar xf ff.tar.xz            # -> ./firefox/firefox
./firefox/firefox --version
```

Runtime libs: `libgtk-3`, `libXt`, `libasound` are present; `libdbus-glib` is
missing but headless Firefox launches fine without it.

## The five sandbox gotchas (all must be handled, or it silently hangs)

1. **Transparent TLS MITM.** Even direct `:443` egress is intercepted; only the
   system CA bundle (`/etc/ssl/certs/ca-certificates.crt`, which carries the
   proxy CA) trusts it. Firefox uses its own Mozilla roots and gets
   `SEC_ERROR_UNKNOWN_ISSUER` on every https site. For behavior testing, drive
   via Marionette with `acceptInsecureCerts: true` (below). A cert error page
   still means DNS + TCP + TLS all reached the server, so DNS success is
   observable as "insecure certificate" vs a real `dnsNotFound`. To get valid
   TLS instead, import `/root/.ccr/ca-bundle.crt` (needs `certutil`, not
   installed by default).

2. **The agent HTTP proxy hijacks Firefox.** Firefox's default
   `network.proxy.type=5` reads `https_proxy`/`http_proxy` from the env and
   routes everything (including the DoH connection) through `127.0.0.1:<port>`,
   which MITMs and stalls TRR. Fix both: set `network.proxy.type=0` AND launch
   with the proxy env vars unset:
   `env -u https_proxy -u HTTPS_PROXY -u http_proxy -u HTTP_PROXY -u all_proxy -u ALL_PROXY ...`.
   Direct egress works (HTTPS and even UDP/53), so direct is the right choice.

3. **Firefox thinks it is offline / behind a captive portal**, so it never
   dispatches external lookups (you see only periodic `127.0.0.1` resolves).
   Disable the checks:
   ```
   user_pref("network.connectivity-service.enabled", false);
   user_pref("network.captive-portal-service.enabled", false);
   user_pref("network.manage-offline-status", false);
   ```

4. **DNS runs in the socket process**, whose `MOZ_LOG` lands in a separate
   `.child-N.moz_log` file (often empty here). Force networking into the parent
   so `nsHostResolver`/`TRR` logs are in the main log file:
   ```
   user_pref("network.process.enabled", false);
   user_pref("browser.tabs.remote.autostart", false);
   ```

5. **`firefox -headless -screenshot URL` does NOT navigate reliably** (it idles,
   re-resolving localhost, then times out). Drive with `-marionette` plus a tiny
   socket client instead.

## Profile prefs (user.js)

Common:
```
user_pref("security.enterprise_roots.enabled", true);
user_pref("network.proxy.type", 0);
user_pref("network.process.enabled", false);
user_pref("browser.tabs.remote.autostart", false);
user_pref("network.connectivity-service.enabled", false);
user_pref("network.captive-portal-service.enabled", false);
user_pref("network.manage-offline-status", false);
user_pref("network.dns.upgrade_with_https_rr", true);
user_pref("network.dns.use_https_rr_as_altsvc", true);
user_pref("network.dns.echconfig.enabled", true);
```
DoH mode (TRR-first; `bootstrapAddr` avoids a chicken-and-egg on the resolver's
own name):
```
user_pref("network.trr.mode", 2);
user_pref("network.trr.uri", "https://mozilla.cloudflare-dns.com/dns-query");
user_pref("network.trr.bootstrapAddr", "104.16.249.249");
```
Native (non-DoH):
```
user_pref("network.trr.mode", 5);
user_pref("network.dns.native_https_query", true);
```

## Marionette driver (reliable navigation)

Launch with `-marionette` (listens on `127.0.0.1:2828`) and drive over the wire.
Framing is `"<len>:<json>"`; commands are `[0, id, "WebDriver:Navigate", {...}]`.
Set `acceptInsecureCerts` in `WebDriver:NewSession` so the MITM cert does not
block the load. A successful load ends in an `insecure certificate` error here;
a genuine resolution failure ends in `Reached error page: about:neterror?e=dnsNotFound`.
That contrast is the signal.

Launch pattern:
```sh
env -u https_proxy -u HTTPS_PROXY -u http_proxy -u HTTP_PROXY -u all_proxy -u ALL_PROXY \
  MOZ_LOG='timestamp,sync,nsHostResolver:5,TRR:5' MOZ_LOG_FILE="$SCRATCH/run.log" \
  ./firefox/firefox -headless -marionette -profile "$SCRATCH/prof" 'about:blank' &
# wait for port 2828, then send WebDriver:NewSession / SetTimeouts / Navigate
```
Useful `MOZ_LOG` modules: `nsHostResolver:5,TRR:5` (add `sync` so lines flush).
Grep the parent `*.moz_log` for `AliasForm host X => Y` (alias decoded),
`NameLookup ... effectiveTRRmode`, and the final `dnsNotFound`/`UNKNOWN_HOST`.

## What the HTTPS RR AliasMode test showed ([bug 1869075](https://bugzilla.mozilla.org/show_bug.cgi?id=1869075))

Same result in both DoH and native mode:

- `www.dotwtf.wtf` (control, resolves directly): reaches TLS -> DNS ok.
- `alias-mode.https.dotwtf.wtf` (AliasMode apex, NO A record): `dnsNotFound`.
  The log shows `AliasForm ... => www.dotwtf.wtf` was decoded, but Firefox never
  resolves the target's A/AAAA, so the name fails. This is the bug.
- `akamai.com` (AliasMode HTTPS at apex, but also has apex A/AAAA): loads via the
  apex A records; the alias to `www.akamai.com.edgekey.net` is decoded and then
  ignored for addressing.

Takeaway: Firefox parses AliasMode and follows it only to fetch the HTTPS
SvcParams; it does not move address resolution to the alias target. A pure-alias
apex (the [RFC 9460](https://www.rfc-editor.org/rfc/rfc9460) intent) therefore
fails unless the apex also serves A/AAAA.
