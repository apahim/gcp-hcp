# Jira Risk Template

Use this template when creating a Risk issue in the [GCP project](https://redhat.atlassian.net/jira/software/projects/GCP/boards). See [Risk Tracking Process](risk-tracking-process.md) for field definitions, qualifying criteria, probability/impact scales, and the full lifecycle.

## Is This a Risk?

Before creating, confirm the issue qualifies as a risk:

- Is there genuine uncertainty about whether/how it will happen?
- Does it describe something that *might* happen, not something already known or in progress?
- Is the trigger outside the team's full control?
- Is there a credible scenario it materializes before the team can address it?

If not — create a Story, Task, or Epic instead.

---

## Summary (Risk Statement)

Use this one-line pattern for the Jira summary field:

```
<event> could <consequence> due to <root cause>
```

**Examples:**
- "Cincinnati API outage could block new cluster creation due to version resolution dependency"
- "Vendor contract renewal delay could pause GKE upgrade path due to dependency on partner SLA"

---

## Description

```
_What could go wrong:_ <Describe the risk event in detail>

_What triggers it:_ <Conditions or events that would cause the risk to materialize>

_What would be affected:_ <Teams, services, milestones, or customers impacted>

_Mitigation/contingency plan:_ <Actions to reduce probability or impact; how to respond if it materializes. Leave blank if not yet assessed.>

_Originally raised: YYYY-MM-DD. Raised by: <your name>._
```

---

## Required Fields

| Field | Where to Set | Notes |
|---|---|---|
| Risk Probability | Right-hand panel | See probability scale below |
| Risk Impact | Right-hand panel | See impact scale below |
| Component | Right-hand panel | GCP component area this risk relates to |

**Risk Score** and **Risk Score Assessment** are auto-calculated by ScriptRunner when Probability and Impact are saved — do not fill these manually.

### Probability Scale (1–5)

| Score | Level | Criteria |
|---|---|---|
| 1 | Rare | Theoretical; no precedent in this or similar projects |
| 2 | Unlikely | Has happened elsewhere but conditions aren't present here |
| 3 | Moderate | Has happened before or some contributing factors exist today |
| 4 | Likely | Contributing factors are active; expected without changes |
| 5 | Very Likely | Already showing early signs; a matter of when, not if |

### Impact Scale (1–5)

| Score | Level | Criteria |
|---|---|---|
| 1 | Annoyance | Cosmetic or documentation issue, OR no effect on delivery, service, or customers, OR absorbed within normal workflow without re-planning |
| 2 | Low | Small delay or workaround required, OR limited to a single team or component, OR no customer-visible effect, OR minimal rework (days, not weeks) |
| 3 | Moderate | Noticeable delay to a milestone, OR partial service degradation, OR affects multiple components or teams, OR requires engineering intervention or re-planning, OR SLO breach possible |
| 4 | Medium | Significant schedule slip (weeks), OR service outage or data integrity issue, OR blocks dependent work streams, OR affects customers directly, OR reputational or compliance risk |
| 5 | High | Project delivery blocked, OR complete service unavailability or data loss, OR security or compliance breach, OR affects all customers or the entire project timeline, OR regulatory or contractual consequences |

## Optional Fields

| Field | Notes |
|---|---|
| Risk Proximity | When could it materialize? |
| Risk Response | Avoid / Mitigate / Transfer / Accept |
| Risk Category | Technical / Schedule / Resource / External / etc. |

---

## Workflow

New risks start in **New** status. Transition to **Refinement** → **To Do** once probability, impact, and a mitigation plan are assessed. See [risk-tracking-process.md](risk-tracking-process.md) for the full workflow, escalation criteria, and closure protocol.
