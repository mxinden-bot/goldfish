# Necko bug triage

How to run a Necko (Core :: Networking*) triage pass. The team's dashboard and
Bugzilla do the real work; this file is the durable recipe so a future session
does not re-derive it. Present every bug as a clickable link, never a bare
number.

## Dashboard and the "untriaged" queue

- Triage helper: https://mozilla-necko.github.io/necko-triage/ (source:
  https://github.com/mozilla-necko/necko-triage, default branch `master`; the
  query is built in
  [`necko-triage.js`](https://github.com/mozilla-necko/necko-triage/blob/master/necko-triage.js)
  /
  [`app-settings.js`](https://github.com/mozilla-necko/necko-triage/blob/master/app-settings.js)).
- The dashboard renders its table via JS, so a plain fetch of the page shows no
  bugs. Reconstruct the query against the Bugzilla REST / buglist API instead.

"Untriaged" means: product **Core**, in the nine components the dashboard
lists in `NeckoTriage.prototype.components` (Networking, Networking: Cache,
Networking: DNS, Networking: File, Networking: HTTP, Networking: JAR,
Networking: Proxy, Networking: WebSockets, DOM: Networking); **unresolved**;
**no needinfo flag**; reporter not `wptsync@mozilla.bugs` or
`intermittent-bug-filer@mozilla.bugs`; and EITHER no necko whiteboard tag at
all (`[necko-triaged]`, `[necko-priority-queue]`, `[necko-priority-review]`,
`[necko-would-take]`, `[necko-backlog]`, `[necko-active]`, `[necko-next]`) OR
`[necko-triaged]` with an empty priority or severity.

**Networking: Cookies is NOT in the list.** Cookies belong to the cookie team
(triage owner Tim Huang, cookie code mostly baku); do not tag cookie bugs
`[necko-triaged]` or count them as Necko queue. An earlier version of this file
included Cookies and sent a session chasing four cookie bugs (2026-10).

Ready-to-click live list (buglist.cgi, the dashboard's boolean chart verbatim):
https://bugzilla.mozilla.org/buglist.cgi?query_format=advanced&product=Core&resolution=---&component=Networking&component=Networking%3A%20Cache&component=Networking%3A%20DNS&component=Networking%3A%20File&component=Networking%3A%20HTTP&component=Networking%3A%20JAR&component=Networking%3A%20Proxy&component=Networking%3A%20WebSockets&component=DOM%3A%20Networking&f19=flagtypes.name&o19=substring&v19=needinfo&n19=1&email1=intermittent-bug-filer%40mozilla.bugs&emailreporter1=1&emailtype1=notequals&email2=wptsync%40mozilla.bugs&emailreporter2=1&emailtype2=notequals&j1=OR&f1=OP&f2=OP&n3=1&f3=status_whiteboard&o3=substring&v3=%5Bnecko-triaged%5D&n4=1&f4=status_whiteboard&o4=substring&v4=%5Bnecko-priority-queue%5D&n5=1&f5=status_whiteboard&o5=substring&v5=%5Bnecko-priority-review%5D&n6=1&f6=status_whiteboard&o6=substring&v6=%5Bnecko-would-take%5D&n7=1&f7=status_whiteboard&o7=substring&v7=%5Bnecko-backlog%5D&n8=1&f8=status_whiteboard&o8=substring&v8=%5Bnecko-active%5D&n9=1&f9=status_whiteboard&o9=substring&v9=%5Bnecko-next%5D&f10=CP&f11=OP&j12=OR&f12=OP&f13=priority&o13=equals&v13=--&f14=bug_severity&o14=equals&v14=--&f15=CP&f16=status_whiteboard&o16=substring&v16=%5Bnecko-triaged%5D&f17=CP&f18=CP

Same query as JSON: swap `buglist.cgi` for `rest/bug` and append
`&include_fields=id,summary,component,severity,priority,keywords,whiteboard,flags,creator,creation_time&limit=0`.
(Fetching the raw JSON via curl keeps the full list; WebFetch summarizes and can
drop rows.)

Other dashboard tables worth checking in a pass (queries in
[`necko-triage.js`](https://github.com/mozilla-necko/necko-triage/blob/master/necko-triage.js)
`availableTables`; transcribe them by hand, since evaluating the downloaded JS
is blocked in the web sandbox):

- **Webcompat needs diagnosis**: any product, unresolved, user story contains
  `diagnosis-team:networking`, keyword `webcompat:needs-diagnosis`. Live list:
  https://bugzilla.mozilla.org/buglist.cgi?resolution=---&f1=cf_user_story&o1=substring&v1=diagnosis-team%3Anetworking&f2=keywords&o2=anywordssubstr&v2=webcompat%3Aneeds-diagnosis
- **Untriaged bugs (awaiting ni?)**: same components, unresolved, has a
  `needinfo?`, and none of `[necko-triaged]`, `[necko-would-take]`,
  `[necko-backlog]`, `[necko-active]`, `[necko-next]`. Sorted by needinfo age:
  stale reporter needinfos are candidates for INCOMPLETE.

Checked against the dashboard source 2026-10.

## Max's own needinfo queue

Needinfos on public bugs are readable anonymously over REST (no API key):
`https://bugzilla.mozilla.org/rest/bug?f1=requestees.login_name&o1=equals&v1=mail@max-inden.de&f2=flagtypes.name&o2=equals&v2=needinfo%3F&include_fields=id,summary,status,flags,last_change_time`.
Needinfos on restricted (security) bugs are invisible without an API key, so
the count is lower than the profile page's; point Max at
https://bugzilla.mozilla.org/request.cgi?action=queue&requestee=mail%40max-inden.de&type=needinfo
for those. Many BugBot needinfos (uplift?, r+ not landed, backout) are already
moot by the time he looks: check the bug's later comments before ranking.

## When a bug counts as triaged

Docs: https://firefox-source-docs.mozilla.org/networking/submitting_networking_bugs.html
(and the lingo page: https://firefox-source-docs.mozilla.org/networking/necko_lingo.html).

A Necko bug is triaged once it has **a priority** and the **`[necko-triaged]`**
whiteboard tag. The triager either sets a priority (and severity), reassigns to
the right team when it is not a networking issue, or needinfos for missing info.
Reporters are asked not to set Priority/Severity themselves: doing so hides the
bug from the triage queue.

## Severity and priority shorthand

- Severity: S1 catastrophic (crash / data loss), S2 serious (major broken, full
  UI hang, no workaround), S3 normal, S4 minor / cosmetic / code cleanup.
- Priority: P1 current cycle, P2 soon or before a related feature ships, P3
  backlog, P5 will not actively work (patches welcome). Good-first-bugs are
  usually left **without** `[necko-triaged]` so they stay in the GFB queue.

## Applying changes

Claude Code on the web has **no Bugzilla write tool** (the `moz` MCP server is
often not attached). Do the analysis, then hand Max the call as clickable bug
URLs plus the exact field values to set, or offer to prep the code fix. Never
claim a bug was updated.

## Common mis-filings to watch

- wpt-sync auto-files fetch/XHR failures under **DOM: Networking**; many are
  `.tentative` tests (non-blocking) or belong elsewhere: HTTP/1 parsing ->
  Networking: HTTP, permissions-policy -> DOM: Security.
- The Bugbug ML classifier and reporters often dump unrelated bugs into Core ::
  Networking. Networking is only the transport, so check where the behavior
  actually lives: Widevine/GMP update traffic -> Core :: Audio/Video: GMP;
  address-bar / neterror UI -> a Firefox front-end component; a server-side WAF
  block or self-signed cert -> not a Firefox bug (INVALID / WORKSFORME +
  needinfo for a log).

## Finding the author / regressor

- The web-session `/home/user/firefox` checkout is a **shallow squashed
  snapshot** (~50 commits, one per file), so local `git blame` / `git log` are
  useless for authorship (they map every line to an unrelated squash commit).
- Use upstream history instead:
  - hg log / annotate:
    https://hg.mozilla.org/mozilla-central/log/tip/<path> and
    https://hg.mozilla.org/mozilla-central/annotate/tip/<path> . These 302 to
    `hg-edge.mozilla.org`; follow the redirect. Commit messages carry the Bug
    number and `r=` reviewers, and annotate attributes each line to its landing
    changeset.
  - searchfox: https://searchfox.org/mozilla-central/source/<path>
  - Bugzilla `regressed_by`, or run mozregression when it is unset.
- `searchfox-cli` is **not on PATH** in web sessions; read the checkout directly
  (narrow `rg` by directory) or use searchfox.org.
