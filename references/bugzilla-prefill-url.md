# Bugzilla: pre-fill a new bug via URL

When Max asks to draft a Bugzilla bug, do not hand him prose to copy. Build a URL
that opens `enter_bug.cgi` with the fields already filled in, so he clicks it,
glances over it, and presses Save/Submit. Bugzilla reads these fields straight
from the query string.

## Endpoint

    <BUGZILLA_BASE_URL>/enter_bug.cgi?product=...&component=...&short_desc=...&comment=...

Use whichever Bugzilla instance the bug is for. Do not hard-code an internal or
private hostname in this public repo; take the base URL from Max per request.

## Parameters (query string to form field)

| Field               | Parameter           | Notes                                   |
|---------------------|---------------------|-----------------------------------------|
| Product             | `product`           | Required, or you land on the chooser.   |
| Component           | `component`         | Include so the form is fully ready.     |
| Version             | `version`           |                                         |
| Summary             | `short_desc`        |                                         |
| Description         | `comment`           | First comment; encode newlines as %0A.  |
| Severity            | `bug_severity`      |                                         |
| Priority            | `priority`          |                                         |
| Hardware / platform | `rep_platform`      |                                         |
| Operating system    | `op_sys`            |                                         |
| Initial status      | `bug_status`        | e.g. NEW, CONFIRMED.                    |
| Assignee            | `assigned_to`       | Email.                                  |
| QA contact          | `qa_contact`        | Email.                                  |
| CC                  | `cc`                | Repeat the param or comma-separate.     |
| URL                 | `bug_file_loc`      |                                         |
| Keywords            | `keywords`          | Comma-separated.                        |
| Depends on          | `dependson`         | Bug IDs.                                |
| Blocks              | `blocked`           | Bug IDs.                                |
| Alias               | `alias`             |                                         |
| Target milestone    | `target_milestone`  |                                         |
| Whiteboard          | `status_whiteboard` | If enabled.                             |
| Classification      | `classification`    | If the instance uses classifications.   |
| See also            | `see_also`          | URLs to related bugs.                   |
| Custom field        | `cf_<name>`         | Only if the field has the enter_bug flag.|
| Entry template      | `format`            | Selects which entry template loads.     |

## Gotchas

- Always include `product` (and `classification` if the instance uses them), or
  Bugzilla shows the product chooser instead of the filled-in form.
- Product, component, version, and milestone values must match the instance's
  configured names exactly (case-sensitive). A wrong value errors or is ignored.
- URL-encode everything: space as %20, newline as %0A, `&` as %26, `#` as %23,
  `+` as %2B.
- Max still reviews and clicks the submit button; nothing is filed until he does.
- The link needs an authenticated session if the instance requires login.
- A long `comment` makes a long URL. Usually fine, but very large ones can hit
  browser or server limits.

## Example

    https://bugzilla.example.org/enter_bug.cgi?product=Core&component=Networking&version=unspecified&short_desc=DNS%20resolver%20times%20out%20on%20IPv6&comment=Steps:%0A1.%20Disable%20IPv4%0A2.%20Load%20a%20page%0AExpected%20resolves,%20actual%20times%20out.&bug_severity=normal&op_sys=Linux
