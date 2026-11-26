# Human Evaluation Rubric · Juno

## What graders score

- **What you’re scoring:** Juno's daily experiment recommendation. That's the single action the Recommendation skill proposes for each running experiment (Ship, Keep running, Scale, or Kill a variant), with its cited reason. When Juno can't support a call, the "Paused" result it shows instead counts too.
- **Value proposition:** Before standup, tell me what to do with each running experiment. Give me one action backed by numbers I can check in the snapshot, or tell me plainly that the data can't support a call yet.

## Dimensions

| Dimension | Question | Scale | 1 or No | 3 | 5 or Yes |
|---|---|---|---|---|---|
| Evidence accuracy (deal-breaker) | Does every number in the reason match the snapshot field and value it cites? | Yes / No | At least one number is wrong, made up, or doesn't appear in the snapshot. | n/a | Every number traces to a named snapshot field with the exact same value. |
| Right call | Given this snapshot, is the proposed action the one you would have made? | 1 to 5 | Wrong direction, like Ship when a guardrail is near harm, or Kill when the variant is still gray. | Defensible, but you'd lean toward a different action or want it sent for review first. | Exactly the action you'd pick, at the right confidence level (shown on its own, or sent to you for review). |
| Knows when to hold back (deal-breaker) | Did Juno withhold or limit the recommendation whenever the data couldn't support it? | Yes / No | It recommended something despite a data flag, a stale source, or a guardrail P(harm) of 95% or more. Or it proposed anything other than Keep running on a sample that's too small. | n/a | Every unsafe case was paused or sent to the owner PM, labeled with the reason. Every under-minimum sample got Keep running only. |
| Ready to act | Could you make the decision without opening Hex or Optimizely to check? | 1 to 5 | The reason is vague or missing key metrics. You have to dig through the dashboards to understand the call. | Clear enough to act on, but you'd still check one or two numbers yourself. | You could act on it before standup and forward it to your team as is. |
| Stays in bounds | Does the reason use probabilities, avoid banned words, and stick to this experiment's snapshot? | Yes / No | It says "winning," "losing," or "significant." Or it states outcomes instead of P(beat control) and P(harm). Or it uses other experiments, past learnings, or business reasons to justify the call. | n/a | Probabilities only, none of the banned words, nothing from outside this experiment's snapshot. |

## Calibration

- **How many answers you’ll score, and how often:** 15 a week, pulled every Friday: 10 recommendations Juno showed (always including every Ship, Kill, and Scale, plus the ones I felt least sure about) and 5 that Juno paused or routed to a PM.
- **Who scores them with you:** Me, plus another growth PM scoring the same 15 independently. Weekly for the first month, then once a month after that.
- **What happens when two of you disagree:** If our scores differ by 2 or more on a 1–5 question, or we split on any yes/no question, we talk it through and rewrite that description so it's clearer next time. If we split on a deal-breaker, the answer counts as a fail until we agree, and it goes into the eval test set.

## Pass bar

An average of 4 or higher on Right call and on Ready to act, no single Right call score below 3, and Stays in bounds passes.

**Fails outright, whatever the average:**

- Evidence accuracy: any "no"
- Knows when to hold back: any "no"
- A Ship, Kill, or Scale shown without going to the owner PM for review first
- Any wording that says or implies Juno made the change itself (e.g., "shipped," "killed the variant")
- A paused result with no reason shown
