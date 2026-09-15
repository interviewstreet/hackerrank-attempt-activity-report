# HackerRank Attempt Activity

A skill for Claude Code and Codex that rebuilds HackerRank's **Attempt Activity** view
from raw event data and renders it as a self-contained HTML report.

If you run assessments in **Proctor Mode** or **Desktop App Mode**, you will not see the
Attempt Activity view on those attempts. This fills that gap, and works for any attempt
regardless of mode.

The report shows what a candidate did during their assessment: time per question, when they
switched questions or came back to one, tab-switch windows, and paste events.

## How it is split

| Responsibility | Where it lives |
| --- | --- |
| Fetching the attempt and its `attempt_events` | the `hackerrank-mcp` server, read-only |
| Parsing rules, report design, output | this skill |

The MCP obtains data and nothing else. It renders no HTML and holds no report design, so
the look of the report can change here without touching the server. Everything is
read-only; nothing is written back to HackerRank.

## Install

The skill is just `SKILL.md` plus its two reference files - there is no code to build and
no manifest is required to *use* it. Both clients discover a skill by finding `SKILL.md`
in a directory they already look in, so installing is a copy.

**Claude, from Settings.** No terminal needed. Open **Settings > Plugins**, click **Add**
then **Add marketplace**, and paste the repository link:

```
https://github.com/interviewstreet/hackerrank-attempt-activity-report.git
```

Add it, then turn on `hackerrank-attempt-activity-report` in the list that appears.

**Claude Code, in the terminal.** The same thing as two commands:

```bash
/plugin marketplace add interviewstreet/hackerrank-attempt-activity-report
/plugin install hackerrank-attempt-activity-report@hackerrank-attempt-activity-report
```

`/plugin update hackerrank-attempt-activity-report@hackerrank-attempt-activity-report` picks up a new
version. The `.claude-plugin/` manifests exist only for these two marketplace routes.

Or skip the marketplace and drop the skill straight in - `~/.claude/skills/` for every
project, `.claude/skills/` inside one repository:

```bash
mkdir -p ~/.claude/skills
cp -R skills/attempt-activity-report ~/.claude/skills/
```

**Codex.** Codex looks for skills in `.agents/skills/` - `$HOME/.agents/skills` for every
project, or `.agents/skills` at the root of a repository for that repository only. There is
no marketplace, so copy it in:

```bash
mkdir -p ~/.agents/skills
cp -R skills/attempt-activity-report ~/.agents/skills/
```

Start a new session after copying, in either client - skills are discovered at startup.

The `hackerrank-mcp` server must be configured in whichever client you use; the skill calls
its `list_candidates` and `get_candidate_result` tools.

## Use it

Ask in plain language, with a report URL or an ID:

- "Generate the attempt activity for https://www.hackerrank.com/work/tests/1234567/candidates?c=all&page=1&openReport=900000001"
- "Generate the attempt activity for attempt 900000001 and test 1234567"
- "Show me the attempt timeline for candidate 100000001 on test 1234567"
- "Time per question for this attempt - did they tab out or paste anything?"

The report is written to `output/<attempt_id>-attempt-activity.html`.

## Identifiers

The single most common failure. Three URL shapes appear and they do **not** carry the same
ID:

| URL shape | ID in the URL |
| --- | --- |
| `/x/tests/{test_id}/candidates/{id}/report` | **attempt** ID |
| `/work/tests/{test_id}/candidates?...&openReport={id}` | **attempt** ID |
| `/x/tests/{test_id}/candidates/{id}/pdf` | **candidate** ID |

`get_candidate_result` takes the **candidate** ID and returns "Resource not found" for an
attempt ID. An attempt ID is converted by paging `list_candidates` with
`fields=["id","attempt_id"]` at `limit=100` - the API rejects a larger limit - and matching
on `attempt_id`.

## What the report contains

- A timeline chart: one lane for the question list, one per question visited, with bars for
  time on screen, bands for tab-switched-away windows, and ticks for question-list visits,
  submissions, code runs and pastes.
- Per-question time on screen, active time (on screen minus tab-switched away), visits,
  revisits, submissions, code runs and pastes.
- A reconciliation table comparing the rebuilt figures against the `invisible_duration` and
  `editor_paste_count` HackerRank reports, so a disagreement is visible rather than hidden.
- The full event list, with undocumented event codes labelled as such.

## Files

```
skills/attempt-activity-report/   the skill - this is the whole thing
  SKILL.md                        the workflow: resolve, fetch, rebuild, render, report
  references/attempt-events.md    event vocabulary and every parsing rule, with evidence
  references/report-layout.md     the HTML skeleton and stylesheet to reproduce
.claude-plugin/                   only for Claude Code's marketplace install
  plugin.json                     plugin name, version, description
  marketplace.json                lets this repository be added as a marketplace
```

Copying `skills/attempt-activity-report/` on its own is enough for either client. The
manifests add nothing to how the report is generated.

## Reports contain candidate data

Generated reports include candidate names, emails and integrity signals. `output/` is
gitignored - keep it that way, and do not commit reports or paste their contents into
shared channels.
