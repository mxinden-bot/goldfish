# Routine: Chrome networking watch

A scheduled report on what Chrome and QUICHE changed in networking since the
last run, filtered to Max's Firefox topics. The logic lives here in git; the
scheduler's prompt only points at this file (see "How this is wired up").

Cadence: every other day. Sources, query recipes and the topic filter:
`references/chrome-net-watch.md`. Read it first.

## Before anything: obey the hard rules

Read CLAUDE.md and follow every hard rule in it, above all: link every CL,
commit, bug and feature you mention (rule 6), and never use an em dash.

## Steps

1. Window: the last 3 days (one day of overlap with the previous run, so
   nothing falls between runs).
2. Chromium, through the Gerrit REST API:
   - changes under `dir:net` merged in the window;
   - changes under `dir:net` created or updated in the window that are still
     open (WIP included).
   Read the commit message of each candidate before reporting it. Do not rely
   on a local Chromium checkout; fetch files through Gitiles instead, e.g.
   `https://chromium.googlesource.com/chromium/src/+/main/net/base/features.cc?format=TEXT`
   (base64).
3. QUICHE: clone with `--shallow-since` the window start, list new commits,
   and diff `quiche/common/quiche_feature_flags_list.h` against the start of
   the window. A flipped default (internal or Chromium) is always worth
   reporting.
4. chromestatus: features about QUIC, HTTP/3, DNS, WebTransport, ECH or
   connection racing updated in the window.
5. Filter hard to the topic list in the reference. Drop autorolls, refactors,
   test-only and enterprise-proxy changes unless they change behavior Max cares
   about.

## Output

- The final reply is the report: it becomes the push and email summary. Lead
  with the two or three items most relevant to Firefox, one line each with the
  link and why it matters for Firefox. Then group the rest by topic. Keep it
  short.
- If nothing relevant happened, say so in one line.
- Write nothing to this repo on a normal run. If a fact in the baseline
  section of `references/chrome-net-watch.md` changed (a feature or flag
  default flipped, a version enabled), open one small PR updating that
  baseline, authored as Max, per the PR rules in `routines/memory-upkeep.md`.

## How this is wired up

The schedule lives in the web UI (every other day, fresh session, push and
email notifications). Its prompt is:

    Read routines/chrome-net-watch.md in this repository and carry out the
    routine exactly as described there.
