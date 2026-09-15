# Report layout

The exact document to produce. Copy it character for character and replace only `{{...}}`
tokens. Do not restyle, reorder, rename sections, or add sections. Every class name is
referenced by the stylesheet at the end of this file.

## Formatting conventions

These are not optional. Getting any of them wrong makes the report differ from every other
report.

- **Durations.** Round to whole seconds, then take the first case that applies:
  zero or less is `0 sec`; an hour or more is `H hr M min`, dropping to `H hr` when the
  minutes are zero (`1 hr 45 min`, `2 hr`); a minute or more is `M min S sec`, dropping to
  `M min` when the seconds are zero (`1 min 51 sec`, `10 min`); otherwise `N sec`
  (`42 sec`). Never print `0 min 42 sec`, `10 min 0 sec`, or `90 min 0 sec` - a duration
  past an hour never stays in minutes. A duration that is absent entirely is
  `not recorded`, which is not the same as `0 sec`.
- **Elapsed, in the event table only.** `M:SS` from attempt start (`0:00`, `0:05`, `1:51`).
  Once it reaches an hour, `H:MM:SS` (`1:45:34`) - assessments routinely run longer than an
  hour, so this case is normal, not exceptional.
- **Percentages.** `PCT(n)` is `n / duration_seconds * 100`, clamped to 0-100, printed to
  exactly four decimals (`11.7117`, `0.0000`, `100.0000`). A bar's width is
  `PCT(exit) - PCT(entry)` computed from **unrounded** percentages and only then rounded to
  four decimals, otherwise the last digit drifts.
- **Bar and band geometry.** `SPAN(a, b)` prints `left:L%;width:W%`, where `L = PCT(a)`
  and `W = max(MINWIDTH, PCT(b) - PCT(a))` computed from **unrounded** percentages.
  `MINWIDTH` is `0.35` unless a block says otherwise, so a zero-length mark stays visible.
  If `L + W` would exceed 100, `L` becomes `max(0, 100 - W)` so nothing overflows the lane.
  Both numbers print to exactly four decimals.
- **Question naming.** `{{lane number}}` is the question's position in **first-visit
  order** - 1 for the first question opened - and is *not* the `qno` carried by the event.
  `{{question name}}` is `questions[qid].name`; when it is missing, fall back in order to
  `Question {qno}`, then `Unidentified question (qid {qid})`, then `Unidentified question`.
- **Deltas.** Signed, `+0s` / `-3s` for durations and `+0` / `-1` for counts.
- **Escaping.** HTML-escape every substituted value. `&middot;` and `&rarr;` below are
  literal markup, not data.

REPEAT blocks are emitted once per array entry in order, and omitted entirely when the
array is empty. ONLY-IF blocks are omitted unless the condition holds.

## Skeleton

```html
<!doctype html>
<html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Attempt Activity - {{candidate name}}</title>
<style>{{paste the STYLESHEET section of this file verbatim}}</style>
</head><body><div class="wrap">
<div class="banner">Replica view. Rebuilt from <code>attempt_events</code>. Not a HackerRank product page.</div>
<div class="card">
<div class="head"><div>
<h1>{{candidate name}}</h1>
<p class="email">{{candidate email}}</p>
</div>
<div class="ids">
Test <b>{{test id}}</b> &middot; Candidate <b>{{candidate id}}</b><br>
Attempt <b>{{attempt id}}</b><br>
Score <b>{{score}} / {{sum of every question max_score}} &middot; {{percentage_score}}%</b>
ONLY-IF integrity_status is present: <br>Integrity <b>{{integrity_status}}</b>
</div></div>
<div class="section-tab">Attempt Activity</div>
</div>
<div class="card">
<h2>Timeline <span class="dur">{{attempt duration}}</span></h2>
<p class="sub">{{attempt_starttime}} &rarr; {{attempt_endtime}} UTC</p>
<div class="legend">
The first three entries are always emitted, even when the chart carries no such mark:
<span><span class="sw" style="background:#4cc4a0"></span>On screen</span>
<span><span class="sw tick listing"></span>Question-list visit</span>
<span><span class="sw tick submit"></span>Answer submitted</span>
ONLY-IF any tab-switch window exists: <span><span class="sw" style="background:rgba(240,147,43,0.55)"></span>Tab-switched away</span>
ONLY-IF any code 9 event exists: <span><span class="sw tick paste"></span>Paste</span>
ONLY-IF any code 5 event exists: <span><span class="sw tick run"></span>Code execution</span>
</div>
<div class="scroll"><div class="gantt">
<div class="lane-label">Main Listing</div>
<div class="lane">
REPEAT per code 2 event:
<div class="tick listing" style="left:{{PCT(elapsed)}}%"></div>
REPEAT per tab-switch window whose start falls outside every question segment - a window
with no question open belongs to this lane:
<div class="away" style="{{SPAN(window start, window end)}}"></div>
</div>
REPEAT per question visited, in first-visit order. The FIRST question lane carries
class "lane alt", the second "lane", alternating from there:
<div class="lane-label">Question {{lane number}}<div class="qname">{{question name}}</div></div>
<div class="lane alt">
REPEAT per visit of this question:
<div class="bar" style="{{SPAN(entry, exit)}}"></div>
REPEAT per tab-switch window that overlaps one of this question's visits, at MINWIDTH
0.25, clamped to that visit and emitted once per window - stop at the first visit it
overlaps so a window spanning two visits is not drawn twice:
<div class="away" style="{{SPAN(window start, window end)}}"></div>
REPEAT per code 4, 5 and 9 event on this question in ONE pass over the events, so the
three kinds stay interleaved in chronological order rather than grouped by kind. The
class is "tick submit" for code 4, "tick run" for code 5, "tick paste" for code 9:
<div class="tick submit" style="left:{{PCT(elapsed)}}%"></div>
</div>
<div></div><div class="axis">
REPEAT for the fractions 0, 25, 50, 75 and 100 - the label is the wall-clock time at that
fraction of the attempt:
<span style="left:{{fraction to four decimals}}%">{{clock}}</span>
</div>
</div></div>
</div>
<div class="card">
<h2>Attempt summary</h2>
<div class="tiles">
<div class="tile"><div class="k">Timeline</div><div class="v">{{attempt duration}}</div><div class="n">Login to submission</div></div>
<div class="tile"><div class="k">Questions visited</div><div class="v">{{count}}</div><div class="n">{{n}} switch{{"" when n is 1, otherwise "es"}} &middot; {{n}} revisited</div></div>
<div class="tile"><div class="k">Tab-switched away</div><div class="v">{{reported invisible_duration when present, else the rebuilt total}}</div><div class="n">{{"as reported by HackerRank" when present, else "rebuilt from events"}}</div></div>
<div class="tile"><div class="k">Paste events</div><div class="v">{{count}}</div><div class="n">{{"none recorded" when 0, else "from an external source"}}</div></div>
</div>
</div>
<div class="card">
<h2>Time per question</h2>
ONLY-IF no question was visited, the whole section body is just this one line - omit the
explanatory note, the table and the wrapper entirely:
<p class="empty">No question was opened during this attempt.</p>
OTHERWISE:
<p class="sub">On screen is wall-clock time in the question. Active removes overlapping tab-switch windows; the gap between them is the integrity signal.</p>
<div class="scroll"><table><thead><tr>
<th>Question</th>
<th class="num">On screen</th>
<th class="num">Active</th>
<th class="num">Away</th>
<th class="num">Visits</th>
<th class="num">Submits</th>
<th class="num">Runs</th>
<th class="num">Pastes</th>
</tr></thead><tbody>
REPEAT per question visited:
<tr>
<td>Question {{lane number}} &middot; {{question name}}</td>
<td class="num">{{on-screen duration}}</td>
<td class="num">{{active duration}}</td>
<td class="num">{{away duration}}</td>
<td class="num">{{visits}}</td>
<td class="num">{{submits}}</td>
<td class="num">{{runs}}</td>
<td class="num">{{pastes}}</td>
</tr>
</tbody></table></div>
</div>
<div class="card">
<h2>Reconciliation</h2>
<p class="sub">Every figure above is rebuilt from raw events. These rows compare the rebuilt values against the values the candidate record reports (tolerance &plusmn;2s on durations, exact on counts).</p>
<div class="scroll"><table><thead><tr>
<th>Measure</th>
<th class="num">Rebuilt</th>
<th class="num">Reported</th>
<th class="num">Delta</th>
<th>Status</th>
<th>Note</th>
</tr></thead><tbody>
<tr>
<td>Tab-switched-away time</td>
<td class="num">{{rebuilt total}}</td>
<td class="num">{{reported invisible_duration}}</td>
<td class="num">{{signed delta}}</td>
<td><span class="pill {{"match" within 2s, else "mismatch"}}">{{"matches" or "mismatch"}}</span></td>
<td>{{"No tab-switch window was recorded. The reported figure is the one shown above." when no window exists; when the figures disagree, state that HackerRank's reported figure is the one shown above and the per-question split is rebuilt from the events; otherwise "Rebuilt from visibility events (codes 7 and 8)."}}</td>
</tr>
<tr>
<td>Paste events</td>
<td class="num">{{rebuilt count}}</td>
<td class="num">{{reported editor_paste_count}}</td>
<td class="num">{{signed delta}}</td>
<td><span class="pill {{"match" when equal, else "mismatch"}}">{{"matches" or "mismatch"}}</span></td>
<td>{{"Rebuilt from paste events (code 9)." when equal, otherwise say the paste count is uncertain}}</td>
</tr>
</tbody></table></div>
ONLY-IF there are warnings: <ul>REPEAT per warning: <li>{{warning}}</li></ul>
The warnings list carries only things that make a figure above less reliable: events
dropped for having no usable code or timestamp, a missing end time, or a rebuilt total
that disagrees with the reported one. An out-of-range event code is **not** a warning -
it is excluded by design rather than being an anomaly of this attempt, and warning about
it would put back on the page the thing that was removed.
</div>
<div class="card">
<h2>Attempt events</h2>
<div class="scroll"><table><thead><tr>
<th class="num">#</th>
<th>Time</th>
<th class="num">Elapsed</th>
<th class="num">Code</th>
<th>Label</th>
<th>Track</th>
</tr></thead><tbody>
REPEAT per event, in chronological order, **rendering only codes 1, 2, 3, 4, 5, 7, 8, 9
and 10**. That list is the whole set - it is an allow-list, not an exclusion list, so a
code it does not name is never rendered whatever its value: code 6 is the ping, anything
outside 1-10 is not attempt activity, and a zero, a negative or a missing code is
malformed.
Read the code as a number before you match it, so a `"3"` arriving as a string still
counts as code 3 rather than being dropped as unrecognised.
The row number counts the rows actually rendered, so it always runs 1, 2, 3 with no gaps:
<tr>
<td class="num">{{row number}}</td>
<td><span class="mono">{{clock}}</span></td>
<td class="num"><span class="mono">{{elapsed as M:SS}}</span></td>
<td class="num">{{code}}</td>
<td>{{short label - see the table below}}</td>
<td>{{the question NAME for a question-scoped event, else "Main Listing" for code 2, else "Attempt"}}</td>
</tr>
</tbody></table></div>
</div>
<p class="foot">Rebuilt locally by hackerrank-attempt-activity-report from read-only attempt_events. Contains candidate integrity data - handle accordingly.</p>
</div></body></html>
```

## Event labels

Use these exact short labels in the event table. The nine codes below are the only ones
that ever produce a row.

**There is no fallback label, and you must never invent one.** If a code is not in this
table the row does not exist - do not write `Undocumented event code N`, do not write
`Unknown`, and do not pass the raw code through as its own label. A row whose label you
had to guess is a row that should have been dropped.

| Code | Label |
| --- | --- |
| 1 | Logged in |
| 2 | Viewed question list |
| 3 | Viewed question |
| 4 | Submitted answer |
| 5 | Ran code |
| 7 | Tab-switched away |
| 8 | Returned to the test |
| 9 | Pasted external text |
| 10 | Rendered code |

## Stylesheet

Paste verbatim into the `<style>` tag. The class names above depend on it.

```css
:root { color-scheme: light; }
* { box-sizing: border-box; }
body { margin: 0; padding: 28px 20px 56px; background: #f4f6f8;
  font: 14px/1.5 -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  color: #1f2933; -webkit-font-smoothing: antialiased; }
.wrap { max-width: 1080px; margin: 0 auto; }
.banner { background: #fff8e6; border: 1px solid #f2d99b; color: #7a5b1c;
  border-radius: 10px; padding: 14px 18px; margin-bottom: 22px; font-size: 14px; }
.card { background: #fff; border: 1px solid #e4e8ed; border-radius: 12px;
  padding: 26px 30px; margin-bottom: 22px; }
h1 { font-size: 30px; margin: 0 0 4px; letter-spacing: -0.3px; }
h2 { font-size: 20px; margin: 0 0 4px; }
.sub { color: #6b7785; margin: 2px 0 0; font-size: 13px; }
.dur { color: #6b7785; font-weight: 400; font-size: 17px; margin-left: 10px; }
.legend { display: flex; flex-wrap: wrap; gap: 20px; margin: 18px 0 22px; color: #4a5561; font-size: 13px; }
.legend span { display: inline-flex; align-items: center; gap: 7px; }
.sw { width: 26px; height: 11px; border-radius: 6px; display: inline-block; }
.sw.tick { width: 4px; height: 14px; border-radius: 1px; }
.tiles { display: grid; grid-template-columns: repeat(auto-fit, minmax(210px, 1fr)); gap: 16px; }
.tile { border: 1px solid #e4e8ed; border-radius: 10px; padding: 18px 20px; }
.tile .k { color: #6b7785; font-size: 12px; text-transform: uppercase; letter-spacing: 0.07em; }
.tile .v { font-size: 30px; font-weight: 600; margin: 8px 0 4px; letter-spacing: -0.5px; }
.tile .n { color: #6b7785; font-size: 13px; }
table { width: 100%; border-collapse: collapse; font-size: 13px; }
th, td { text-align: left; padding: 10px 12px; border-bottom: 1px solid #edf0f3; }
th { color: #6b7785; font-weight: 600; font-size: 12px; text-transform: uppercase; letter-spacing: 0.05em; }
td.num, th.num { text-align: right; font-variant-numeric: tabular-nums; }
tbody tr:last-child td { border-bottom: none; }
.mono { font-variant-numeric: tabular-nums; font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 12px; }
.pill { display: inline-block; padding: 3px 10px; border-radius: 999px;
  font-size: 12px; font-weight: 600; }
.pill.match { background: #e3f6ee; color: #1c7355; }
.pill.mismatch { background: #fdeaea; color: #a32b28; }
.pill.unavailable { background: #eef1f4; color: #5b6672; }
.empty { color: #4a5561; background: #f7f9fa; border: 1px solid #e4e8ed;
  border-radius: 8px; padding: 16px 18px; }
.note { color: #7a5b1c; background: #fff8e6; border: 1px solid #f2d99b;
  border-radius: 8px; padding: 12px 16px; margin-bottom: 12px; font-size: 13px; }
.foot { color: #8b95a1; font-size: 12px; text-align: center; }
.scroll { overflow-x: auto; }

.head { display: flex; flex-wrap: wrap; gap: 16px; justify-content: space-between; align-items: flex-start; }
.email { color: #6b7785; margin: 0; }
.ids { text-align: right; color: #6b7785; font-size: 13px; line-height: 1.8; }
.ids b { color: #1f2933; }
.section-tab { display: inline-block; margin-top: 22px; padding-bottom: 8px;
  border-bottom: 3px solid #2fae86; font-weight: 600; }

.gantt { display: grid; grid-template-columns: 190px 1fr; gap: 0 18px; align-items: center;
  min-width: 720px; }
.lane-label { text-align: right; padding: 10px 0; font-size: 13px; line-height: 1.35; }
.lane-label .qname { color: #6b7785; font-size: 12px; }
.lane { position: relative; height: 34px; border-radius: 8px; }
.lane.alt { background: #f7f9fa; }
.bar { position: absolute; top: 11px; height: 12px; border-radius: 6px;
  background: #4cc4a0; min-width: 3px; }
.away { position: absolute; top: 7px; height: 20px; border-radius: 3px;
  background: rgba(240, 147, 43, 0.55); border-left: 1px solid #e08a1e;
  border-right: 1px solid #e08a1e; min-width: 2px; }
.tick { position: absolute; top: 5px; height: 24px; width: 3px; border-radius: 1px; }
.tick.listing { background: #2fae86; }
.tick.submit { background: #3b6fd4; }
.tick.paste { background: #d9534f; }
.tick.run { background: #9aa5b1; }
.axis { position: relative; height: 20px; border-top: 1px solid #e4e8ed;
  margin-top: 4px; color: #6b7785; font-size: 12px; }
.axis span { position: absolute; top: 4px; transform: translateX(-50%); white-space: nowrap; }
.axis span:first-child { transform: none; }
.axis span:last-child { transform: translateX(-100%); }
```
