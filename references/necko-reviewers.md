# Necko reviewers for Firefox patches

Who to name as reviewer on a Firefox patch that touches `netwerk/`, and why not
to tag the group.

## Do not tag `#necko-reviewers`

Never write `r?#necko-reviewers` in a commit message: `moz-phab submit` rejects
it. Ignore firefox
[`AGENTS.md`](https://github.com/mozilla-firefox/firefox/blob/main/AGENTS.md)'s
generic `r?#group-name` advice for necko.

The tag is not needed anyway: Herald adds `#necko-reviewers` to any revision
touching `netwerk/` paths, so name an individual and the group attaches itself.
Confirmed 2026-08 on https://phabricator.services.mozilla.com/D315875: the
commit message said only `r?valentin`, yet the revision came up with reviewers
`valentin` AND `necko-reviewers`.

## Who to name

The authoritative roster is
[`mots.yaml`](https://searchfox.org/mozilla-central/source/mots.yaml), module
`core_necko`. Re-checked 2026-11:

- owner: `valentin` (Valentin Gosu)
- peers: `kershaw` (Kershaw Chang), `jesup` (Randell Jesup), `sunil` (Sunil Mayya)

Max's usual picks: `kershaw`, `valentin`, and `leggert` (Lars Eggert,
`[:lars]`, the neqo maintainer). Note the spelling: `leggeRt`, not `leggett`.

Inside a firefox checkout, the in-tree `find-reviewer` skill looks reviewers up
live via `mach`; see `references/firefox-agent-skills.md`. It cannot know Max's
personal picks, which is why this file exists.
