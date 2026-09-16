# Reading a GitHub PR's reviews from a web session

How to read a pull request's review feedback when the PR lives in a repo the
session is not scoped to, e.g. reviewing
[mozilla/neqo](https://github.com/mozilla/neqo) from a session pinned to the
`mxinden-bot` forks. Verified 2026-09 on
[mozilla/neqo#3982](https://github.com/mozilla/neqo/pull/3982).

## The one rule

**Start the session with the repo that owns the PR as a source.** Everything
below is what happens when you do not, and none of it recovers inline review
comments. If the goal is "review PR X in repo R", R has to be in the session
from the first turn.

## What the session scope actually blocks

A web session carries a fixed set of attached repos. The GitHub gateway
intercepts both `api.github.com` and `github.com` for shell tools, so `curl`
gets a 403 whose body is not from GitHub:

```json
{"message":"GitHub access to this repository is not enabled for this session.
Use add_repo to request access."}
```

`add_repo` does not fix it across owners:

- `access: "read"` on a public repo: answers that anonymous git reads already
  work and attaches nothing. Gives clone and fetch, **no API**.
- `access: "push"`, which is the only path to API access, fails outright:

```
add_repo: cross-tier adds are not supported in v1: requested "mozilla/neqo"
but session already has repos from owner(s) [mxinden-bot]
```

That is a backend limit on mixing owners in one session, not a permission
prompt, so no approval from Max changes it. The `mcp__github__*` tools stay
scoped to the attached repos and refuse the rest by name.

## What still works without the API

- **The code.** The git proxy serves anonymous reads of any public repo, PR
  refs included:

  ```sh
  git fetch --depth=50 https://github.com/mozilla/neqo \
      refs/pull/3982/head:pr3982
  ```

  So the diff, the head commit and the commit message are always reachable, and
  a full local review is possible with no GitHub access at all.

- **Review summary bodies, via `WebFetch` on the PR page.** `WebFetch` is not
  subject to the gateway's repo scoping and returns the server-rendered HTML of
  `https://github.com/<owner>/<repo>/pull/<n>`.

## The trap: bodies yes, inline threads no

GitHub server-renders each review's **summary body** into the PR page, but the
**inline thread comments** are fetched by the client afterwards. `WebFetch`
therefore returns a reviewer who wrote a summary in full, and returns nothing
at all for a reviewer whose feedback is entirely inline. The page still shows
the thread markers, so it looks like the comments are there:

```
## martinthomson Review
(no body)
```

That is not "Martin approved without comment". It means Martin left 13 inline
comments that `WebFetch` cannot see. Ask for the review bodies verbatim and
count them against the `Show resolved` markers before believing a reviewer said
nothing. `WebFetch` on `/pull/<n>/files` returns no comment bodies either.

Inline threads need `mcp__github__pull_request_read` with
`method: "get_review_comments"`, which needs the repo attached.

## Do not spawn a child session to fetch data

`add_repo`'s own error suggests starting a new session with the repo as its
initial source, and `create_session` will do that. It does not help the session
that needs the answer, because **there is no return channel from a cloud child
session**:

- `ListAgents` lists only sessions on the same machine, so a cloud sibling
  never appears, and `SendMessage` to its session ID answers
  `No agent named 'session_...' is reachable`.
- `get_session` returns status and a one-line `post_turn_summary`, never
  transcript content.
- The Claude Code Remote MCP server exposes no resources, so there is no read
  path into another session.
- The child cannot hand the data sideways either: `add_repo` from its side hits
  the same cross-tier refusal, so it has no shared repo to drop a file in.

The child does the work and the answer lands in its transcript, where only a
human can read it. Fine as a deliverable for Max, useless as a subroutine.

Do not quote the child's `post_turn_summary` as a finding either. It is a
generated one-liner, not data: the same session reported "13 review threads"
after its first pass and "75 review threads" after its second.

## Practical order

1. Reviewing a PR in an unscoped repo? Say so before starting, and open the
   session against that repo.
2. Already in the wrong session? Fetch the PR head over git and review the code
   locally: that half needs nothing from GitHub.
3. For the review threads, ask Max to paste them. It is faster than any
   workaround here and it is the only thing that gets inline comments into the
   session that is doing the review.
