# AI PRD · Juno

> **Fictional and privacy-safe.** RocketShip, its experiments, people and metrics are synthetic.

## Problem & user

RocketShip's growth PMs spend about 20 minutes every morning cross-checking Jira, Optimizely and Hex to work out which experiments are healthy, which have bad data, and which need a decision. Calls slip and data problems go unnoticed.

Juno handles that triage with three skills: **Status** (one trustworthy view of every running experiment), **Recommendation** (a proposed action with a number-backed reason) and **Learnings** (draft write-ups of concluded tests). It works inside limits a human set on purpose, and a PM makes every decision.

## The harness

| Surface | Specification |
|---|---|
| 01 Context | Jira, Optimizely and Hex for the Status skill only. The Recommendation and Learnings skills see only the status snapshot. No warehouse, Slack, customer or roadmap data. |
| 02 Tools | Read and draft verbs per skill, `publish_learning` and `route_to_human` as the only writes, no send verbs. |
| 03 Loop | Per experiment per run: Status 7 turns, Recommendation 4, Learnings 5. Escalate on repeated source failures, conflicts, SRM, guardrail harm ≥ 95% or past-due tests. |
| 04 Memory | Scoped per experiment and visible to the team. Nothing per PM. Human decisions always outrank model output. |
| 05 Permissions | read auto · draft auto · write confirm · send blocked |
| 06 Verification | A separate verifier pass checks every number and action against the snapshot and thresholds. Failures are labeled and paused, never retried or dropped. |

## 01 Context · Data Requirements

**Required sources, and exclusions**

| Source | What Juno reads | Used by |
|---|---|---|
| Jira (GRO project) | Experiment tickets: ID, owner PM, hypothesis, status, launch date, planned end date | Status skill |
| Optimizely | Flag state, variants, current and planned traffic split, sample size (current vs. minimum required), killed variants with date and reason | Status skill |
| Hex (experiment results dashboards) | Bayesian results per metric: role (input, primary or guardrail), control and variant values, lift, 95% credible interval, P(beat control), P(harm), expected loss | Status skill |
| thresholds.json (versioned config) | Color rules and minimum runtime or sample. PMs can audit it, and the AI never edits it | Dashboard, Recommendation skill, verifier |
| Status snapshot (JSON) | The normalized, flagged view of every experiment | Recommendation and Learnings skills (their only experiment data) |
| Notion: Experiment Learnings Log | Past entries, read so the skill doesn't duplicate them | Learnings skill |

The Recommendation and Learnings skills never touch Jira, Optimizely or Hex directly, so they can't route around a data flag.

Excluded:
- **Raw warehouse event data.** Hex is the single source of computed results. Juno never re-derives statistics and creates a second truth.
- **Slack threads and DMs.** These are opinions about experiments, not evidence, and they carry privacy risk.
- **Customer- or account-level data.** Decisions need only aggregate results.
- **Roadmap, sales pipeline and strategy docs.** They give the model a business reason to argue for a result the data doesn't support.
- **Experiments outside the self-serve growth team.** This keeps the scope to the users Juno is designed for.

**How fresh must it be**

- A full snapshot runs daily at **7:00 a.m. PT**, before morning triage. A PM can also trigger a manual refresh, which completes in about 5 minutes.
- **SLA of truth:** a change in any source shows up in Juno's answer by the next scheduled run (within 24 hours at most), or within about 5 minutes after a manual refresh.
- Every source shows its own last-updated time per experiment, so the PM can see how old each part of the answer is.

Unreachable or stale source (over 24 hours): the status view shows the last good snapshot labeled **"Failed to update,"** and recommendations for that experiment **pause**. A labeled, read-only status view harms no one. A recommendation built on stale results could lead someone to ship a losing variant, so that's where Juno stops. Missing values show "Missing," never 0 or an estimate. If thresholds.json can't be read, Juno **fails closed** and shows no colors and no recommendations, because the rules everything is judged by are missing.

## 02 Tools · System Capabilities

**The verb list**

Status skill
- `list_running_experiments(team)`: READ
- `read_jira_ticket(experiment_id)`: READ
- `read_optimizely_flag(flag_key)`: READ
- `read_hex_results(experiment_id)`: READ
- `read_thresholds()`: READ
- `build_status_snapshot(date)`: DRAFT

Recommendation skill
- `read_status_snapshot(date)`: READ
- `draft_recommendation(experiment_id, action, reason)`: DRAFT

Learnings skill
- `read_learnings_log(query)`: READ
- `draft_learning(experiment_id, summary, evidence)`: DRAFT
- `publish_learning(draft_id)`: WRITE

Shared
- `route_to_human(experiment_id, reason)`: WRITE (adds the experiment to the owner PM's decision queue with a reason)

**Deliberate omissions**

- **No `ship_variant()`, `kill_variant()`, `set_traffic_split()` or `conclude_experiment()`.** Changing a live experiment is a SEND: it changes what real customers see and what revenue we book. Rolling back a flag doesn't undo a week of lost signups. A PM makes that change in Optimizely in V1, full stop. Juno only recommends.
- **No `update_jira_status()`.** Jira is the team's record of truth. If Juno could edit it, the source it reads would start reflecting the model's opinion. A status conflict is surfaced as a flag, never "fixed" by Juno.
- **No `post_to_slack()`.** A morning digest broadcast to the team is a SEND with a wide reach, so an early wrong call spreads before anyone checks the dashboard. That's a V2 decision, once the evals show recommendation accuracy.
- **No `query_warehouse()`.** Raw data access would let Juno re-run statistics and produce a second, unaudited answer.

## 03 Loop · AI Costs & Latency

**Turn ceiling and escalation**

Ceiling, per experiment per run, including the verifier pass:
- **Status skill: 7 turns.** That covers Jira, Optimizely, Hex, thresholds, a retry, the snapshot and verification.
- **Recommendation skill: 4 turns.**
- **Learnings skill: 5 turns**, and it only runs on concluded experiments.

If a skill hits its ceiling, it returns what it has, labeled **"Incomplete: turn limit reached,"** and recommendations for that experiment pause. It never truncates silently, and it never guesses the missing piece.

Escalate (`route_to_human` to the owner PM) when:
- **2 consecutive failed calls to the same source.** The experiment is marked "Failed to update" and its recommendation pauses.
- **Sources contradict each other.** For example, Jira says "Concluded" while the Optimizely flag is still running. Which one is true is a human's call, not a tiebreak the model should invent.
- **Sample ratio mismatch is detected.** The randomization may be broken, so any result is suspect.
- **A guardrail's P(harm) is ≥ 95%.** The experiment goes to the owner the same morning.
- **The experiment is past its planned end date with no decision.**

Every recommendation, escalated or not, carries **"Needs human decision."** Escalation controls how fast it reaches the owner, not whether a human decides.

**Latency and cost target**

- **Latency:** the 7:00 a.m. run finishes by **7:15 a.m.** for about 20 experiments. The dashboard loads in **under 2 seconds** because it reads a precomputed snapshot. A manual refresh takes **under 60 seconds** for one experiment and **under 5 minutes** for all of them.
- **Cost:** a ceiling of **$0.15 per experiment per run** across all skills and verification. At about 20 experiments × 1 scheduled run plus up to 3 manual refreshes, that's **about $10 a day at worst, or about $220 a month** over 22 workdays. A **hard daily cap of $15** disables refreshes until the next scheduled run, and the dashboard says why.
- **Why it's worth it:** Juno replaces about 20 minutes per PM each morning. Even for one PM, that's roughly 7 hours a month saved for $220.

Turns multiply, they don't add: 20 experiments × up to 16 turns × 4 runs is about **1,300 model turns a day** at the worst case. That's why the ceilings are set per experiment, not per run.

## 04 Memory · Data Requirements

**What persists, at what scope**

- **Turn:** raw responses from Jira, Optimizely and Hex are discarded once the snapshot is built. Nothing raw is kept.
- **Experiment (team-visible):** the daily snapshot, the recommendation and its reason, the threshold version used, and every PM decision or override. This is the unit of memory, and each experiment is judged only on its own data.
- **Team:** published learnings in the Notion log, plus thresholds.json, which only a human changes, through review.

What never crosses:
- **No per-PM memory.** Juno doesn't track which PM overrides most, ships fastest or misses end dates. It judges experiments, not people. A PM scorecard would turn a triage tool into a surveillance tool and kill adoption.
- **No carryover between experiments.** The Recommendation skill doesn't read the learnings log. This stops "the last pricing test lost, so this one will too" from biasing a call the data should make.

**Expiry and write rules**

| Fact | Lives for |
|---|---|
| Raw source pulls | End of run |
| Daily snapshots and recommendations | Life of the experiment + 90 days, then deleted |
| PM decisions and overrides | Life of the experiment + 1 year, as the audit trail |
| PM acknowledgment of a data issue | Until the next run. If the issue is still there, it's raised again, so a dismissed "Failed to update" never stays hidden. |
| Published learnings | Permanent, owned by the team |

Who outranks whom:
- A PM's decision is written as the final state, with their reason. The model's recommendation is kept next to it for audit, marked **superseded**. Model output never overwrites a human decision.
- If new data later changes the model's view, it shows both instead of quietly re-recommending: *"PM chose Keep running on Nov 20; with 6 more days of data, Juno now recommends Ship."* The human decision stands until a human changes it.
- In the learnings log, human edits win. Juno can't republish over a page a person has edited.
- Every snapshot records the threshold version it used, so any past color can be explained.

## 05 Permissions · AI Risks & Mitigations

**The permission tier**

| Side-effect class | Tier |
|---|---|
| read | auto |
| draft | auto |
| write | confirm |
| send | blocked |

**Justification**

**Read: auto.** All reads are against Jira, Optimizely, Hex, thresholds and the learnings log. The worst case is wasted tokens or a stale pull, and stale data is already labeled and pauses recommendations. Nothing changes in any system.

**Draft: auto.** Snapshots, recommendations and learning drafts appear only on the dashboard, each tagged "Needs human decision." A wrong draft costs a PM a few seconds of reading, and nothing is acted on until a person decides. Making drafts confirm would mean approving 20 items every morning to see your own dashboard, which brings back the triage work Juno exists to remove.

**Write: confirm.** `publish_learning` puts a conclusion into the team's permanent record. A wrong learning ("annual-plan nudges don't work") can quietly kill future tests. Undoing the page is easy, but undoing the belief isn't. A PM approves every publish.
*Exception:* `route_to_human` runs auto. It only adds the experiment to the owner's decision queue with a reason, can be cleared in one click, and an escalation that waits for approval defeats its purpose.

**Send: blocked.** No send verbs exist in V1: no ship, kill, traffic change, conclude or Slack post. Changing a live experiment changes what customers see and what revenue we book, and rolling back a flag doesn't recover a week of lost signups. Mistakes would be both irreversible and widespread, so this stays blocked no matter how the evals look. Revisiting it is a V2 decision.

## 06 Verification · AI Testing & Measurement

**The check before output ships**

A separate **verifier pass** checks every output before it reaches the dashboard. It's a second model call with its own prompt, and it sees only the output and the snapshot, never the first skill's reasoning. It answers **pass or fail on each rule, quoting the snapshot value it checked against**. A pass without a quoted value counts as a fail.

Status snapshot:
- Every number matches a field in this run's Jira, Optimizely or Hex pull.
- Each source is under 24 hours old, sample ratio mismatch and status conflicts are flagged, and missing fields show "Missing," never 0 or an estimate.

Recommendation:
- Every number in the reason appears in this experiment's snapshot.
- The action is consistent with thresholds.json. "Ship" requires input and primary at P(beat) ≥ 95% and guardrail P(harm) under 80%. Only "Keep running" or "Scale" is allowed while metrics are gray, and there's no recommendation at all while the experiment has a data flag.
- It contains none of "winning," "losing" or "significant." Juno reports probabilities. It doesn't declare outcomes.

Learning draft:
- It cites only concluded experiments, and every claim references an experiment ID and a metric value from the final snapshot.

**Bar:** every rule must pass. Dashboard colors come straight from thresholds.json, and the verifier checks the AI's words against those same rules.

**Known limit:** an AI checking an AI can miss the same mistake. That's why human reviewers score a weekly sample of verified outputs against the eval rubric.

**Failure behaviour**

- **A status check fails:** the experiment still shows, with the specific flag, and its recommendation pauses.
- **A recommendation check fails:** it's replaced with **"Paused: Juno couldn't verify this recommendation,"** plus the failed rule, and routed to the owner PM. **No retry.** Regenerating until the output passes teaches the model to write reasons that pass the gate, not reasons that are true.
- **A learning check fails:** it stays a draft labeled "Unverified claim," and publishing is disabled.
- **Nothing is silently dropped.** Every failure is visible, labeled and added to the eval test set.

**When the verifier gets replaced by code.** A deterministic rule check replaces the AI verifier if either of these happens:
- In human review, the verifier agrees with reviewers on **less than 95%** of rule checks.
- **A recommendation that passed verification but broke a threshold reaches a PM.** One is enough.

If **more than 10% of recommendations fail verification in a week**, the Recommendation skill pauses until a PM reviews the prompt.

## Eval plan

_To be completed with the human eval rubric._ It will cover:
- **Golden set:** the 18 seeded experiment scenarios (clear ship, guardrail breach, SRM, stale source, status conflict, under-sampled, past due and others), each with its expected status, color and recommendation.
- **Pass bar:** 100% on the hard rules in Verification, plus a reviewer-scored bar for whether each recommendation's reason is correct and useful.
- **Cadence:** the full golden set reruns on every prompt or threshold change, and reviewers score a weekly sample of live outputs.

## Out of scope

Any capability not in the verb list above, and any action tiered `blocked`. Both are decisions on the record, not omissions. In V1 that specifically means:
- Shipping, killing, concluding or re-allocating traffic on any experiment
- Editing Jira tickets or experiment status
- Posting to Slack or sending any notification outside Juno's dashboard
- Querying raw warehouse data or recomputing statistics
- Reading experiments outside the self-serve growth team
