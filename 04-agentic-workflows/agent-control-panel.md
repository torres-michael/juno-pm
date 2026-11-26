# Agent Control Panel · Juno

## What Juno Is Doing

Juno's Recommendation skill reads the verified daily status snapshot and proposes one action per running experiment (Ship, Keep running, Scale or Kill a variant), with a reason that cites the numbers behind it, so each growth PM starts the day with a ready decision queue.

## Four Levers

**Stop Conditions** (Loop):

max_steps: 4 per experiment per run (read snapshot → check thresholds → draft → verify). On step 4 with no verified output, stop and label "Incomplete: turn limit reached"; no recommendation is shown and the experiment is routed to the owner PM.

Abort early (no recommendation, route_to_human with the reason) if any of these happens:
- The snapshot has a data flag on this experiment (Failed to update, SRM, status conflict, Missing field)
- read_status_snapshot or read_thresholds fails 2x in a row
- The verifier fails the draft once (no retry)
- Spend on this experiment reaches $0.15

Hard timeout: 60s wall clock per experiment. Whole scheduled run: 15 min (7:00–7:15 a.m. PT); any experiment not finished by then shows "Paused: run timed out."

Done = every running experiment ends with either a verified recommendation or a labeled pause. Hitting a limit above means something went wrong, not that the job finished.

**Structured Tool Outputs** (Tools):

```
read_status_snapshot(experiment_id)
  OK:   { status: "ok", snapshot_date, as_of_by_source: {jira, optimizely, hex}, data_flags: [], metrics: [{name, role, p_beat, p_harm, sample_current, sample_required}] }
  FAIL: { status: "error" | "stale" | "not_found", reason }
  Rule: metrics count = 0, or any source older than 24h, counts as FAIL, not "nothing to report." A non-empty data_flags list means stop with no recommendation.

read_thresholds()
  OK:   { status: "ok", version: "v1.3", rules: {...} }
  FAIL: { status: "error", reason }
  Rule: on FAIL, Juno fails closed: no recommendations for any experiment this run.

draft_recommendation(experiment_id, action, reason, evidence)
  OK:   { status: "drafted", draft_id, action ∈ [ship, keep_running, scale, kill_variant], evidence: [{field, value}], threshold_version }
  FAIL: { status: "rejected", reason: "invalid_action" | "missing_evidence" }
  Rule: evidence must have ≥1 item, and every number in the reason must appear in evidence. Otherwise it's rejected.

verify(draft_id)   (verifier pass)
  OK:   { result: "pass", checks: [{rule, result: "pass", quoted_value}] }
  FAIL: { result: "fail", failed_rule, quoted_value }
  Rule: a "pass" missing a quoted_value counts as a fail.

route_to_human(experiment_id, reason)
  OK:   { status: "queued", queue_item_id }
  FAIL: { status: "error" } → show "Couldn't route; check queue" on the dashboard. Never fail silently.
```

**Confidence Thresholds** (Verification):

HIGH — Juno shows the recommendation on its own (tagged "Needs human decision"):
- Snapshot has no data flags and all sources are < 24h old
- Keep running: any metric still gray (P(beat) < 95%) and every guardrail P(harm) < 80%
- Keep running: sample < 100% of required, or runtime under the minimum in thresholds.json. Only "Keep running" is allowed (no Ship, Scale or Kill).
- Verifier passes every rule with a quoted value

MEDIUM — owner PM checks it first (route_to_human; draft shown as "Review before deciding"):
- Any Ship recommendation (requires sample ≥ 100% of required, input and primary P(beat control) ≥ 95%, every guardrail P(harm) < 80%). It's the riskiest call, so the owner PM reviews it first.
- Primary looks strong (P(beat) ≥ 95%) but a guardrail is borderline (P(harm) 80–94%)
- Input and primary disagree (one ≥ 95%, the other gray)
- Any Kill variant or Scale recommendation. These change what customers see, so a person always looks first.
- Past planned end date with no decision

TOO UNSURE — no recommendation at all; route_to_human with the reason:
- Any data flag (Failed to update, SRM, status conflict, Missing field)
- Any guardrail P(harm) ≥ 95% (routed the same morning as a guardrail alert)
- Verifier fails, or the turn limit/timeout is hit
- thresholds.json can't be read (fail closed for the whole run)

**North Star** (Context):

North Star: Give each growth PM one verified, number-backed recommendation per running experiment before standup. Always cite the snapshot values behind every claim, and never recommend an action on data Juno can't vouch for. When in doubt, pause and route to the owner PM.

Always:
- Cite the exact snapshot field and value for every number in the reason
- Report probabilities (P(beat control), P(harm)), not outcomes

Never:
- Recommend anything but "Keep running" on an under-minimum sample, or recommend at all on an experiment with a data flag or stale sources
- Use the words "winning," "losing," or "significant"
- Use anything outside this experiment's snapshot (no other experiments, no past learnings, no business reasons) to justify a call

## Rules of Engagement

**Agency Permission:**

Decides alone (auto):
- Read the status snapshot and thresholds.json
- Draft a recommendation and show it on the dashboard, tagged "Needs human decision"
- Route an experiment to the owner PM's decision queue (route_to_human)
- Pause itself: withhold a recommendation and label why

Proposes and waits:
- Every action (Ship, Keep running, Scale, Kill variant) is only a proposal. The PM makes the change in Optimizely.

Cannot do at all (no tool exists):
- Ship, kill, conclude or change traffic on any experiment
- Edit Jira status or thresholds.json
- Post to Slack or notify anyone outside the Juno dashboard

**Access Control:**

Reads only:
- The daily status snapshot for experiments on the self-serve growth team
- thresholds.json (read-only, versioned)

Cannot reach:
- Jira, Optimizely or Hex directly (only the Status skill can), so it can't route around a data flag
- The Learnings Log, other teams' experiments, raw warehouse data, Slack, customer-level data, roadmap or sales docs

Visibility: every recommendation is visible to the whole growth team, not just the owner PM. Juno keeps no per-PM memory.

**Fallback Protocols:**

- Data flag on the snapshot → no recommendation; label the flag; route_to_human
- Same tool fails 2x in a row → stop; "Failed to update"; route_to_human. No third attempt.
- Verifier fails → replace with "Paused: Juno couldn't verify this recommendation" + failed rule; route_to_human. No retry.
- Turn limit (4) or 60s timeout hit → "Incomplete: turn limit reached" or "Paused: run timed out"; no recommendation; route_to_human
- thresholds.json unreadable → fail closed: no colors, no recommendations for anyone this run
- Spend hits $0.15 per experiment or the $15 daily cap → stop; refreshes disabled until the next scheduled run, with the reason shown
- Every fallback is visible on the dashboard and added to the eval test set. Nothing is dropped silently.

**Checkpoints:**

1. Before any recommendation is shown: the verifier pass (automatic, every time)
2. Before any Medium-confidence item is shown (any Ship, borderline guardrail, input/primary disagreement, any Kill or Scale, past end date): owner PM reviews in the decision queue
3. Before anything changes for customers: a PM acts in Optimizely. Juno never does.
4. Weekly: human reviewers score a sample of verified recommendations against the eval rubric. If the verifier agrees with reviewers on <95% of checks, or one bad recommendation reaches a PM, swap in a code-based check. If >10% of recommendations fail verification in a week, pause the skill until a PM reviews the prompt.
