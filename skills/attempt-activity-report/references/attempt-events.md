# `attempt_events` reference

What the raw event stream looks like and the rules to apply to it. Read this
before changing any timing logic; each rule exists because the naive reading of
the stream produces a wrong timeline.

## Row shape

```json
{
  "id": 9900000001,
  "attempt_id": 900000001,
  "event": 3,
  "data": { "qid": 5550001, "qno": 1, "question_version": 7 },
  "inserttime": "2025-10-25T16:03:00+0000"
}
```

`data` is `{}` for events that are not question-scoped. Timestamps are UTC with
a `+0000` offset and second granularity.

## Event codes

| Code | Meaning | Used for |
| --- | --- | --- |
| 1 | Logged in | Attempt start marker |
| 2 | Viewed question list | Closes a question segment; Main Listing tick |
| 3 | Viewed question | Opens a question segment |
| 4 | Submitted answer | Tick only - **never** closes a segment |
| 5 | Code execution | Tick, per-question run count |
| 6 | Ping to server | Dropped entirely (high-volume heartbeat) |
| 7 | Went invisible / tab-switched away | Opens a tab-switch window |
| 8 | Came back visible | Closes a tab-switch window |
| 9 | Pasted external text | Tick, per-question paste count |
| 10 | Rendered code | Listed only |

**Codes 1 to 10 above are the entire Attempt Activity vocabulary.** The
`attempt_events` field is shared: an attempt can also return codes outside that
range, which belong to other parts of the platform rather than to attempt
activity. Those are dropped from the report entirely - not rendered in the event
table, never used to derive a boundary or a duration, and never shown in the
timeline.

Match on the 1-10 allow-list, and never on a list of codes you have seen before.
The set outside the range is open and additions are not announced, so a code must
never reach the report on the strength of being unrecognised. Do not attempt to
interpret one: an out-of-range code carries no meaning for this view.

## Ordering

**Never assume the array is in time order.** Sort by `inserttime` before
anything else, and do not treat `id` order as time order. Reading the array as
given can produce a nonsense timeline, and a single misplaced event moves every
segment boundary after it, so this is the first step and not an optional one.

Break ties deterministically. Where two events share a timestamp, order code 7
before code 8, then fall back to `id`: sort on `(inserttime, tiebreak, id)` with
7 first, 8 last and everything else between. Taken the other way round, a
same-second pair reads as a return with nothing open followed by a tab-switch
that never ends, which inflates away-time to the end of the attempt. A stable,
stated tie-break is also what makes two runs of the same attempt produce the
same report.

## Question segments

A segment opens on code 3 for a qid and closes on:

- the next code 3 with a **different** qid,
- the next code 2, or
- the attempt end.

Code 4 must not close a segment - candidates submit an answer and keep working
on the same question.

A repeated code 3 for the **same** qid is a page refresh, not a new visit; it is
counted as a refresh and does not split the segment. Re-entering a qid after
leaving it *is* a new visit, and that is what makes a question a revisit.

Questions appear in the report in first-visit order, not qid order.

## Question metadata

The event stream carries only `qid` and `qno`. Everything the report shows about
a question - its name, score and max score - comes from the `questions`
container on the candidate, which is why `expand=questions` is required.

Do not assume a single shape for `questions`. Handle both an object keyed by
question ID:

```json
"questions": { "5550001": { "name": "...", "score": 5, "max_score": 5 } }
```

and a plain array of question objects carrying `id` or `question_id`. Key it by
ID either way, then:

| Report value | Source |
| --- | --- |
| question name | `questions[qid].name` |
| question number shown | the question's position in first-visit order, **not** `qno` |
| per-question score / max | `questions[qid].score`, `.max_score` |
| score denominator | the sum of `max_score` across every question |

Match on the **string** form of the ID. `data.qid` is a number in the event and
the container's keys are strings, so a strict comparison fails.

`data.qno` is the question's position in the test, which is not the order the
candidate visited them in. It is used only as a name fallback, never as the
number the report prints - see `report-layout.md`.

A `qid` with no matching entry keeps its segment and timings. Fall back to
`Question {qno}` for its name, then to `Unidentified question (qid {qid})` -
never drop the segment and never invent a name.

## Tab-switch windows

A window opens on code 7 and closes on whichever of these comes first:

1. the next code 8,
2. a **presence event** — code 2, 3, 4, 5 or 9 — because those cannot happen
   unless the candidate is back at the keyboard, or
3. the attempt end.

An unterminated 7 closes at the attempt end. A trailing 7 can be stamped
*after* the attempt end, so every window is clamped into the attempt boundary
and its length floored at zero - a negative away-time is always a parsing bug,
never real data.

### Why a presence event closes a window

Take this shape of stream:

```
7 @ 15:40:55   went invisible
2 @ 15:41:10   viewed question list      <- interaction: they were back
7 @ 15:41:19   went invisible again      (no code 8 was ever recorded)
8 @ 15:41:30   came back
```

Pairing the first 7 with the next 8 counts 40:55 -> 41:30 as 35 seconds away,
but the interaction at 41:10 *proves* the candidate was present. Over-counting
away-time in an integrity report is the worst direction to be wrong, so the
window closes at the interaction.

Code 6 (ping) must never count as presence: it is a server heartbeat that
fires while the tab is hidden. It is dropped before the window builder runs,
which is part of why the noise filter is not optional.

A second code 7 with no intervening 8 *and* no intervening presence event is
left to extend the open window. There is no evidence of a return, and
inventing one would be a guess. The irregularity is recorded instead.

### Irregularities

Recognise these shapes. They are the usual reason a rebuilt total differs from
the reported one, so naming the shape is what lets the note explain a
disagreement instead of leaving it unexplained:

| Kind | Meaning |
| --- | --- |
| `closed_by_interaction` | No code 8; a presence event ended the window |
| `repeated_invisible` | A second code 7 with no return in between |
| `unmatched_visible` | A code 8 with no window open |
| `unterminated` | No code 8 at all; closed at the attempt end |

### What reconciles and what does not

A well-formed stream - every code 7 paired with a code 8 inside the attempt -
reconciles exactly against the reported figure, including when the payload
arrives out of order or grouped by code, since sorting fixes that first.

Two shapes can disagree, and both are visible in the irregularity kinds above: a
malformed pairing, and an attempt that ended with a tab-switch still open. Treat
a disagreement as a finding to report, never as something to tune away.

### Which away-time figure the report shows

**The headline total is HackerRank's `invisible_duration` whenever it is
present**, so a reviewer reads the same number here as anywhere else in the
product. The rebuilt figure is still computed, still reconciled, and still
shown in the Reconciliation table -- it is not discarded, just not the headline.

The **per-question Away column and the timeline bands stay rebuilt**, because a
single reported total carries no timestamps: there is no way to know from it
which question the time belongs to. So on a malformed stream the split can sum
to a different number than the headline. The report states this in a note
rather than leaving a reviewer to discover that the columns do not add up, and
**neither figure is ever scaled to force agreement** -- apportioning a reported
total across questions would be inventing per-question data.

When no window can be located at all but a total is reported, the report says
that explicitly, because a reviewer would otherwise see a non-zero total with
no bands on the timeline and no explanation.

### When the reported total is longer than the attempt

This happens, and it is not an error. An attempt that ends with the candidate
still tabbed away has an unterminated code 7, and the reported figure can keep
counting past the end of the attempt.

Quoted bare, a total larger than the attempt reads as a candidate absent for
longer than the test lasted. So whenever the reported total exceeds the attempt
duration the report:

- labels the tile "exceeds the attempt duration",
- states in a note how long the attempt actually was, that it ended with a
  tab-switch still open, and what the rebuilt figure inside the attempt window
  is.

It does **not** point at any out-of-range event as the cause, even where the
arithmetic would fit: such events are not rendered anywhere in the report, so
naming one sends the reader looking for a row that does not exist.

Give the caveat in the same breath as the figure. This is the one place where
quoting HackerRank's number without context would actively mislead.

Where the rebuilt figure and the reported figure disagree, the report leads with
the reported total and says so. Reproducing how the reported figure was arrived
at is out of scope: the report's job is to surface the disagreement, not to
explain or resolve it.

## Gross vs active time

Each question reports both:

- **gross** ("on screen") - wall-clock time inside its segments.
- **active** - gross minus the tab-switch windows that overlap those segments.

The gap between them is the integrity signal, which is why both are shown rather
than one blended number. Away-time that falls outside any question segment is
charged to the attempt total and to the Main Listing lane, never to a question.

## Reconciliation

The rebuilt figures are checked against the values the candidate record reports:

| Rebuilt | Reported field |
| --- | --- |
| Sum of tab-switch windows | `invisible_duration` |
| Count of code 9 events | `editor_paste_count` |

Durations agree within ±2s (timestamps are second-granular); counts must match
exactly. A disagreement is rendered as a `mismatch` pill with the delta, and the
report says that measure cannot be treated as settled. Do not widen the
tolerance to make a mismatch disappear - the disagreement is the finding.

## Boundaries

`attempt_starttime` and `attempt_endtime` from the candidate record define the
timeline. If the end time is missing, the last timing-relevant event is used
instead and the report says the window ends at that event rather than at a
submission. If timing-relevant events continue past the reported end, the
timeline extends to include them and says so.
