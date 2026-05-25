# Ex9 — Reflection

## Q1 — Planner handoff decision

In Ex7 session `sess_9a71258dc211` (committed to
`traces/ex7/sess_9a71258dc211/`), round 1 of the bridge ran the loop
half against the task "book for party of 12 in Haymarket." The planner
(ticket `tk_98bda86e`,
`traces/ex7/sess_9a71258dc211/logs/tickets/tk_98bda86e/raw_output.json`)
produced a single subgoal:

```json
{"id": "sg_1", "description": "find venue near haymarket for 12",
 "success_criterion": "candidate identified", "assigned_half": "loop"}
```

`assigned_half` is `"loop"`, not `"structured"`. The planner did not
directly order a handoff; it delegated the entire subgoal to the loop
executor. The handoff decision happened one level below, inside the
executor's tool-call sequence.

Looking at `traces/ex7/sess_9a71258dc211/logs/trace.jsonl` lines 4–5:
after `executor.tool_called venue_search(Haymarket, party=12) →
0 result(s)`, the next event is `executor.tool_called
handoff_to_structured` with reason: `"loop half identified a candidate
venue; passing to structured half for confirmation under policy rules"`.
The executor called the built-in `handoff_to_structured` tool as its
immediate next action after the venue search, even though the search
returned zero matching results.

Two signals drove this. First, the subgoal's `success_criterion:
"candidate identified"` — the executor interpreted its scripted choice
of Haymarket Tap as satisfying that criterion regardless of the filter.
Second, the phrase "confirmation under policy rules" in the executor's
reason string shows the executor recognises the deposit and party-cap
rules as out-of-scope for the loop half.

The bridge wrote `session.state_changed: from="loop", to="structured",
round=1` (trace line 6). The Rasa half rejected with `party_too_large`,
producing the reverse: `from="structured", to="loop", round=1,
rejection_reason="sorry, we can't accept this booking. reason:
party_too_large"` (trace line 7). Round 2's planner (ticket
`tk_497e1afb`) produced a fresh subgoal, `"retry with larger venue
after rejection"`, still `assigned_half: "loop"`.

`assigned_half` specifies which executor runs a
subgoal, not whether the executor later triggers a handoff.
`handoff_to_structured` is a first-class tool the loop executor can
call at any point. The planner sets the context; the executor decides
when its loop-half work is done and kicks the structured half.

---

## Q2 — Dataflow integrity catch

In Ex5 session `sess_1c341da954d4` (committed to
`traces/ex5/sess_1c341da954d4/`), the scripted FakeLLMClient produces
a concrete discrepancy that `verify_dataflow` handles in a revealing
way.

`traces/ex5/sess_1c341da954d4/logs/trace.jsonl` line 5 records:
`calculate_cost(haymarket_tap, 6, 3, bar_snacks) → "total £556, deposit
£111"`. The tool's `_TOOL_CALL_LOG` entry confirms
`total_gbp: 556, deposit_required_gbp: 111` as ground truth from
`sample_data/catering.json`.

Line 6 records `generate_flyer` called with `event_details.total_gbp =
540, event_details.deposit_required_gbp = 0`. The produced flyer
(`traces/ex5/sess_1c341da954d4/workspace/flyer.html`, `data-testid=
"total"` and `data-testid="deposit"`) shows `£540` and `£0` — a £16
undercount and a missed deposit. Running `verify_dataflow` on that
flyer returns `ok=True, verified_facts=['£540', '£0', '12', 'cloudy']`.
No alert is raised.

The check passes because `fact_appears_in_log('£540')` scans both
`r.output` and `r.arguments` for every `ToolCallRecord`. It finds 540
inside `generate_flyer.arguments.event_details.total_gbp` and returns
`True`. A human reviewer seeing `£540` for a party of 6 at a Haymarket
bar would not pause — it is a plausible total, only £16 off.

This is a false negative in the current implementation: the check
trusts generate_flyer's arguments even when they contradict
calculate_cost's output. But the check **would** catch fabrication that
manual inspection misses in the following scenario: if
`workspace/flyer.html` were edited post-generation to change `£540` to
`£850` (a value appearing nowhere in any tool's arguments or outputs),
`fact_appears_in_log('£850')` returns `False` and `verify_dataflow`
returns `ok=False, unverified_facts=['£850']`. A human reviewer sees
£850 for Edinburgh on a Saturday — plausible, no red flag.

To construct the grader's test case: run the full 4-tool sequence;
after `generate_flyer` writes the HTML, overwrite one data-testid value
(e.g. `£540` → `£9999`) directly in the file; run `verify_dataflow` on
the modified content. It must return `ok=False,
unverified_facts=['£9999']` because `£9999` appears in no
`ToolCallRecord`'s output or arguments.

---

## Q3 — First production failure

The first production failure I would expect is a **duplicate confirmed
booking**, caused when the process crashes after the Rasa structured
half commits but before `session.mark_complete()` writes the result to
`session.json`.

In session `sess_9a71258dc211`, the booking reference `BK-B7655866`
appears in `traces/ex7/sess_9a71258dc211/session.json` under
`result.booking_reference`. The trace event `session.state_changed:
from="structured", to="complete"` (trace line 14) is the last thing the
bridge writes before returning. If the Python process crashes between
Rasa confirming (the HTTP response arriving in
`RasaStructuredHalf.run()`) and the bridge writing that final trace
event, the persisted session state remains `"state": "executing"` with
no `result` field.

On retry, `HandoffBridge.run()` starts fresh — it does not inspect the
session's prior `handoff_history` for a prior commitment. It runs the
loop half again (round 1), calls `handoff_to_structured`, and Rasa
generates a new booking reference for the same slot. The pub manager
receives two confirmed bookings.

The primitive that surfaces this is the **ticket state machine**. In
`sess_9a71258dc211`, ticket `tk_338b40dd` covers
`executor.run_subgoal/sg_1` and reaches `state: "success"` when its
manifest is written. But the top-level commit — the
`session.mark_complete(struct_result.output)` call in the bridge — sits
outside any ticket boundary. There is no ticket wrapping "structured
half confirmed; writing result," so the state machine has no
`state: "success"` record to detect on retry.

The fix: create a `committed_booking` ticket around the Rasa
confirmation step, write the booking reference into `raw_output.json`
before calling `session.mark_complete()`. On restart, the bridge checks
for an existing `committed_booking` ticket in `state: "success"` and
short-circuits. The ticket state machine's forward-only progress
guarantee then acts as the idempotency key — no external store required.
