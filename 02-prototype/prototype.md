# Prototype · Juno

> Module 2 · Prototype. The working build that tests the idea, made in Lovable.

## Prototype link

https://juno-experiment-dashboard.lovable.app/

## What it demonstrates
**Testable assumption:** If the Experiment Status skill's output (Module 1) is shown as one view, a growth PM can tell in **under 2 minutes** which experiments are live, which have data they can't trust, and which need a decision. Today that takes about 20 minutes of cross-checking Jira, Optimizely and Hex.

**The one flow it proves:** morning triage. Open the dashboard → the alert banner shows which experiments have untrustworthy data → the summary cards show what needs a decision → click a row to see the evidence, with each value labeled by source and refresh time.

Design choices it tests:
- **Three metric layers.** Input (the controllable metric), primary (revenue) and guardrail (lead quality as the early warning). GRO-104 shows why this matters: form completion up 6.4pp, but revenue P(beat) 4% and lead-quality P(harm) 97% → Conclude to control.
- **Colors come from code, not AI.** Green, yellow, red and gray "Too early" are fixed thresholds from `thresholds.json`, and anyone can audit them. The AI only proposes the recommendation, and a human decides.
- **No recommendations on bad data.** When a source fails to update or data conflicts, the recommendation is paused and the experiment appears in a "failed to update" alert.

## Debrief
- **What worked:**
  - The build matched the spec on the first pass. All 18 experiments load, the banner shows 6 data issues, and the summary cards are correct (4 ready for a decision, 5 too early, 2 past due).
  - Every edge case rendered as designed: the Jira/Optimizely conflict (GRO-116), the missing planned split (GRO-118), the experiment with no Jira ticket (GRO-120), the missing guardrail (GRO-128), the 4-variant test with one arm to cut (GRO-109) and the killed variant (GRO-107).
  - The GRO-104 row tells the input → output → guardrail story at a glance. It's the strongest demo moment.
  - Pausing recommendations on bad data felt trustworthy, not broken. The banner plus "Learn more" turns a failure into a task.
- **What broke / felt like a toy:**
  - **Two sources of truth for derived numbers.** The UI says GRO-112 is on "Day 10 of 22", while the AI recommendation says "day 9 of 14 minimum". The app counts days inclusively and the recommendation counts them exclusively. If the UI and the AI can disagree about a fact, PMs will stop trusting both.
  - **Stale rows still show confident colors.** GRO-113's Hex data is 2 days old, but its input still shows a green "P(beat) 98%". The recommendation is paused, but the metric chips aren't.
  - **Every row says "Needs human decision"**, including "Keep running", so the tag stops meaning anything.
  - **The most important columns are hardest to see.** Primary, Guardrail and Recommendation sit at the far right and scroll off-screen on a laptop-width window.
  - **It's a snapshot, not a morning tool.** It uses static JSON, mocked recommendations, no "what changed since yesterday" view, and no links into the real tools. It shows today's state but not what's new.
- **What I'd change next pass:**
  1. Compute all derived values (days running, sample progress, past due) once in the status skill's output, and have both the UI and the recommendation skill read them, never recalculate them.
  2. Dim or gray out the metric chips for any experiment with a data issue, not only the recommendation.
  3. Show "Needs human decision" only on actionable calls (Ship, Scale, Cut variant, Conclude to control).
  4. Reorder the columns to Experiment → Recommendation → Primary → Guardrail → Input, and move dates and traffic into the drawer.
