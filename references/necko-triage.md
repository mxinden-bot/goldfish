# Necko bug triage

How to run a Necko (Core :: Networking*) triage pass. The team's dashboard and
Bugzilla do the real work; this file is the durable recipe so a future session
does not re-derive it. Present every bug as a clickable link, never a bare
number.

## Dashboard and the "untriaged" queue

- Triage helper: https://mozilla-necko.github.io/necko-triage/ (source:
  https://github.com/mozilla-necko/necko-triage, default branch `master`; the
  query is built in `necko-triage.js` / `app-settings.js`).
- The dashboard renders its table via JS, so a plain fetch of the page shows no
  bugs. Reconstruct the query against the Bugzilla REST / buglist API instead.

"Untriaged" means: product **Core**, across the ten networking components
(Networking, Networking: Cache, Networking: Cookies, Networking: DNS,
Networking: File, Networking: HTTP, Networking: JAR, Networking: Proxy,
Networking: WebSockets, DOM: Networking); **unresolved**; **minus** any bug that
already carries a necko whiteboard tag (`[necko-triaged]`,
`[necko-priority-queue]`, `[necko-priority-review]`, `[necko-would-take]`,
`[necko-backlog]`, `[necko-active]`, `[necko-next]`); **excluding** the bot
reporters `wptsync@mozilla.bugs` and `intermittent-bug-filer@mozilla.bugs`; and
**excluding** needinfo'd bugs.

Ready-to-click live list (buglist.cgi):
https://bugzilla.mozilla.org/buglist.cgi?query_format=advanced&product=Core&component=Networking&component=Networking%3A%20Cache&component=Networking%3A%20Cookies&component=Networking%3A%20DNS&component=Networking%3A%20File&component=Networking%3A%20HTTP&component=Networking%3A%20JAR&component=Networking%3A%20Proxy&component=Networking%3A%20WebSockets&component=DOM%3A%20Networking&resolution=---&f1=status_whiteboard&o1=notsubstring&v1=%5Bnecko-triaged%5D&f2=reporter&o2=notequals&v2=wptsync%40mozilla.bugs&f3=reporter&o3=notequals&v3=intermittent-bug-filer%40mozilla.bugs&f4=flagtypes.name&o4=notsubstring&v4=needinfo

Same query as JSON: swap `buglist.cgi` for `rest/bug` and append
`&include_fields=id,summary,component,severity,priority,type,keywords,whiteboard,creator,creation_time&limit=0`.
(Fetching the raw JSON via curl keeps the full list; WebFetch summarizes and can
drop rows.)

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
