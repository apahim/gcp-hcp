---
name: Risk Tracking Process
description: How the GCP HCP team identifies, assesses, tracks, and mitigates project risks using Jira Risk issue types.
---

# Risk Tracking Process

***Scope***: GCP-HCP

**Date**: 2026-07-28

This document defines how the GCP HCP team identifies, assesses, tracks, and mitigates project risks.

## Tooling

Use the **Risk issue type** in the GCP Jira project.

### Fields

**Required fields:**

| Field | Jira Field ID | Type | Purpose |
|-------|---------------|------|---------|
| Risk Probability | customfield_10642 | Dropdown | How likely is this risk to occur (Rare → Very Likely) |
| Risk Impact | customfield_10842 | Dropdown | Severity if the risk materializes (Annoyance → High) |

**Auto-calculated fields** (populated by ScriptRunner when Probability and Impact are set):

| Field | Jira Field ID | Type | Purpose |
|-------|---------------|------|---------|
| Risk Score | customfield_10976 | Number | Probability weight × Impact weight (range: 1–250) |
| Risk Score Assessment | customfield_10974 | Text | Overall assessment label (Low, Low Med, Medium, Med Hi, High) |

**Optional fields** (available on the screen, use when they add value):

| Field | Jira Field ID | Type | Purpose |
|-------|---------------|------|---------|
| Risk Proximity | customfield_10645 | Dropdown | When the risk could materialize (Imminent, Short term, Medium term, Long term) |
| Risk Response | customfield_10846 | Dropdown | Response strategy (Accept, Avoid, Enhance, Exploit, Mitigate, Share, Transfer) |
| Risk Category | customfield_10679 | Dropdown | Classification (Competitive, Customer Impact, Infrastructure, Legal, Opportunity, Quality, Reputation, Resource, Schedule, Scope) |

Standard Jira fields are also used:

- **Summary** -- one-line risk statement
- **Description** -- detailed risk context, background, and mitigation/contingency plan
- **Assignee** -- risk owner responsible for monitoring and response
- **Reporter** -- person who identified the risk
- **Components** -- GCP component area the risk relates to

### Workflow

The Risk issue type uses the following workflow statuses:

```text
New  -->  Refinement  -->  To Do  -->  In Progress  -->  Review  -->  Closed
```

- **New** -- risk has been raised but not yet assessed
- **Refinement** -- risk is being evaluated for probability, impact, and response strategy
- **To Do** -- risk has been assessed and is ready for mitigation work to begin
- **In Progress** -- active mitigation or response plan is underway
- **Review** -- mitigation actions are complete; risk is being validated as resolved
- **Closed** -- risk has been resolved, accepted, or is no longer relevant

### Board & Dashboard

- **[Risk Kanban Board](https://redhat.atlassian.net/jira/software/c/projects/GCP/boards/13256)** -- all GCP risks by workflow status
- **[Risk Dashboard](https://redhat.atlassian.net/jira/dashboards/26267)** -- aggregated risk metrics and trends

Useful JQL queries for risk management:

- **All open risks**: `issuetype = Risk AND project = GCP AND status != Closed`
- **High-severity risks (Med Hi+)**: `issuetype = Risk AND project = GCP AND "Risk Score" >= 96`
- **Risks needing owners**: `issuetype = Risk AND project = GCP AND assignee = EMPTY AND status != Closed`

## Qualifying a Risk

Before creating a Risk issue, confirm the concern actually qualifies as a risk. A risk is an **uncertain future event** that could negatively affect the project if it occurs. Many concerns that feel like risks are actually planned work, known tech debt, or open design questions — these belong as stories, epics, or tasks instead.

**A concern qualifies as a risk when:**

- There is **genuine uncertainty** about whether or how it will happen. If you already know the problem exists and what the fix is, it's a backlog item — not a risk.
- It describes something that **might happen**, not something that has already happened or is a known state. "Config Connector uses `roles/editor`" is a fact (file a story). "A compromised credential could grant cross-region access before cluster separation is complete" is a risk.
- The **trigger is outside the team's full control** — external dependencies, upstream projects, vendor decisions, or timing uncertainties.
- There is a **credible scenario** where the concern materializes before the team can address it. Proximity to a milestone matters.

**A concern does NOT qualify as a risk when:**

- **The condition has already been resolved.** If the architecture or implementation has evolved past the concern, there is nothing to track.
- **It's too vague to assess.** A risk needs enough specificity to evaluate probability, impact, and a response plan. If it can't clear that bar, it isn't ready to be a Risk issue. Refine it first.
- **It's planned work with no time pressure.** "We haven't built quota monitoring yet" is not a risk when production is months away — it's a backlog item. It only becomes a risk if there's a credible scenario of reaching a milestone without the gap being closed.
- **It's an open design question with active engagement.** If there's a structured process underway to resolve the question (vendor engagement, spike, design review), track it there. A risk only emerges if the resolution process itself is at risk of failing or not completing in time.
- **It's work that is already underway with a known path.** If active work covers the concern (e.g., a feature in progress, a PAM entitlement being implemented), the work stream itself is the tracking mechanism.

## Process

### 1. Identify

Anyone on the team can raise a risk at any time by creating a Risk issue in the GCP project. Include:

- A clear, specific summary (e.g., "Cincinnati API outage could block new cluster creation due to version resolution dependency")
- A description covering: what could go wrong, what triggers it, and what would be affected
- Set status to **New**

Good times to identify risks:
- Sprint planning
- Design reviews and architecture discussions
- Retrospectives
- Incident postmortems
- External dependency changes

### 2. Assess

The risk owner (or the team during grooming) evaluates the risk:

1. Set **Risk Probability** and **Risk Impact** using the scoring criteria below (Risk Score and Risk Score Assessment are calculated automatically)
2. Write a mitigation/contingency plan in the **Description** field
3. Optionally set **Risk Response**, **Risk Proximity**, and **Risk Category** if they add clarity
4. Link the risk to any related epics, stories, or features using issue links
5. Transition status to **Refinement** or directly to **In Progress** if mitigation is already underway

#### Probability

Assign the level where any one of the criteria applies:

| Weight | Level | Criteria |
|-------|-------|----------|
| 1 | Rare | Theoretical; no precedent in this or similar projects |
| 2 | Unlikely | Has happened elsewhere but conditions aren't present here |
| 3 | Moderate | Has happened before or some contributing factors exist today |
| 4 | Likely | Contributing factors are active; expected without changes |
| 5 | Very Likely | Already showing early signs; a matter of when, not if |

#### Impact

Impact levels carry non-linear weights that reflect exponentially increasing severity. Assign the level where any one of the criteria applies:

| Weight | Level | Criteria |
|--------|-------|----------|
| 1 | Annoyance | Cosmetic or documentation issue, OR no effect on delivery, service, or customers, OR absorbed within normal workflow without re-planning |
| 3 | Low | Small delay or workaround required, OR limited to a single team or component, OR no customer-visible effect, OR minimal rework (days, not weeks) |
| 9 | Moderate | Noticeable delay to a milestone, OR partial service degradation, OR affects multiple components or teams, OR requires engineering intervention or re-planning, OR SLO breach possible |
| 32 | Medium | Significant schedule slip (weeks), OR service outage or data integrity issue, OR blocks dependent work streams, OR affects customers directly, OR reputational or compliance risk |
| 50 | High | Project delivery blocked, OR complete service unavailability or data loss, OR security or compliance breach, OR affects all customers or the entire project timeline, OR regulatory or contractual consequences |

#### Risk Score Matrix

Risk Score = Probability weight × Impact weight. Scores are auto-calculated by Jira.

| | Annoyance (1) | Low (3) | Moderate (9) | Medium (32) | High (50) |
|---|---|---|---|---|---|
| **Rare (1)** | 1 | 3 | 9 | 32 | 50 |
| **Unlikely (2)** | 2 | 6 | 18 | 64 | 100 |
| **Moderate (3)** | 3 | 9 | 27 | 96 | 150 |
| **Likely (4)** | 4 | 12 | 36 | 128 | 200 |
| **Very Likely (5)** | 5 | 15 | 45 | 160 | 250 |

Scores of **96+** (Med Hi assessment) should be escalated. See [Escalation](#escalation).

### 3. Track

- **Grooming**: Triage newly identified risks (status = New). Assign owners. Assess probability and impact.
- **Sprint reviews**: Review the Risk Board. Update status, probability, and impact for any risks that have changed.
- **Monthly**: Review all open risks with the full team. Close risks that are no longer relevant. Identify new risks.

### 4. Respond

For risks in **In Progress** status:

- Create linked stories or tasks for specific mitigation actions
- Track mitigation progress through those linked issues
- Update the mitigation/contingency section in the **Description** with progress
- Reassess probability and impact as mitigation actions complete
- When mitigation is complete, transition to **Review**

### 5. Close

Transition a risk to **Closed** when:

- The mitigation plan has been fully executed and the risk is resolved
- The risk is accepted with documented rationale (note the decision in the description)
- The risk is no longer applicable (document why)
- The risk materialized and has been handled through incident response

Add a comment explaining the closure rationale.

## Escalation

Escalate a risk to leadership when:

- Risk Score Assessment is **Med Hi** or **High** (score >= 96)
- The risk affects a milestone commitment
- The risk requires cross-team coordination or external dependencies
- The risk has been open for more than two sprints without progress on mitigation

## Risks Review Meeting

**Cadence:** Monthly
**Purpose:** Monthly sync to discuss project risks, assess probability/impact, and determine mitigation plans.
**Goal:** Updated risks, clear mitigation strategies, risk escalation where needed.

### Recurring Agenda

1. Triage existing risks — review scores, status, and mitigation progress
2. Identify any additional risks to track

### Resources

- [GCP Risks Kanban Board](https://redhat.atlassian.net/jira/software/c/projects/GCP/boards/13256)
- [GCP HCP Risks Dashboard](https://redhat.atlassian.net/jira/dashboards/26267)

## Related Documents

- [Definition of Done](definition-of-done.md)
- [Jira Story Template](jira-story-template.md)
