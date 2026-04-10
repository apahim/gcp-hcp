# Risk Scoring Guide

Risk Score = Impact x Probability

## Risk Impact (1-5)

Assign the score where **any one** of the criteria applies:

- **1 - Negligible**
  - Cosmetic or documentation issue, OR
  - No effect on delivery, service, or customers, OR
  - Absorbed within normal workflow without re-planning
- **2 - Minor**
  - Small delay or workaround required, OR
  - Limited to a single team or component, OR
  - No customer-visible effect, OR
  - Minimal rework (days, not weeks)
- **3 - Moderate**
  - Noticeable delay to a milestone, OR
  - Partial service degradation, OR
  - Affects multiple components or teams, OR
  - Requires engineering intervention or re-planning, OR
  - SLO breach possible
- **4 - Major**
  - Significant schedule slip (weeks), OR
  - Service outage or data integrity issue, OR
  - Blocks dependent work streams, OR
  - Affects customers directly, OR
  - Reputational or compliance risk
- **5 - Critical**
  - Project delivery blocked, OR
  - Complete service unavailability or data loss, OR
  - Security or compliance breach, OR
  - Affects all customers or the entire project timeline, OR
  - Regulatory or contractual consequences

## Risk Probability (1-5)

Assign the score where **any one** of the criteria applies:

- **1 - Rare**
  - No historical precedent in this project or similar systems, OR
  - Would require an unusual combination of factors, OR
  - Theoretical risk only
- **2 - Unlikely**
  - Has happened before in similar projects but not this one, OR
  - Known preconditions are mostly absent, OR
  - Would require a specific trigger that is not currently present
- **3 - Possible**
  - Has happened in this project or team before, OR
  - Some contributing factors are present today, OR
  - Depends on an external event that may or may not occur
- **4 - Likely**
  - Multiple precedents exist in this project, OR
  - Contributing factors are actively present, OR
  - Expected to occur given current trajectory without changes
- **5 - Almost Certain**
  - Currently happening or already manifesting as early warning signs, OR
  - Will occur without immediate intervention, OR
  - Only a matter of when, not if

## Risk Score Matrix

| | Impact 1 | Impact 2 | Impact 3 | Impact 4 | Impact 5 |
|---|---|---|---|---|---|
| **Probability 5** | 5 | 10 | 15 | 20 | 25 |
| **Probability 4** | 4 | 8 | 12 | 16 | 20 |
| **Probability 3** | 3 | 6 | 9 | 12 | 15 |
| **Probability 2** | 2 | 4 | 6 | 8 | 10 |
| **Probability 1** | 1 | 2 | 3 | 4 | 5 |

## Severity Bands

- **1-4 (Low)**: Accept or monitor. Review during regular planning cycles.
- **5-9 (Medium)**: Mitigate within current quarter. Assign owner and track.
- **10-16 (High)**: Mitigate urgently. Escalate to team lead. Requires active mitigation plan.
- **17-25 (Critical)**: Immediate action required. Escalate to management. Block related work until mitigated.

## Risk Categories

### Architecture

- Unvalidated design decisions that may not scale or meet requirements
- Gaps in architecture documentation leading to inconsistent implementation
- Technical debt accumulation blocking future feature delivery
- Missing or insufficient observability (monitoring, alerting, logging)
- Single points of failure in critical paths
- Cross-component coupling creating fragile change propagation

### External Dependencies

- Upstream API changes or deprecations (GCP APIs, GKE, Kubernetes)
- Upstream project delays or breaking changes (HyperShift, OpenShift)
- Third-party service availability or SLA gaps
- Vendor lock-in limiting migration or fallback options
- License changes affecting dependencies

### Team and Execution

- Key person dependencies (single owner of critical subsystems)
- Knowledge silos limiting team ability to respond to incidents
- Insufficient staffing for planned scope
- Competing priorities across teams causing delays
- Onboarding bottlenecks slowing new contributor ramp-up

### Security and Compliance

- Unpatched vulnerabilities in dependencies or infrastructure
- IAM misconfigurations or overly broad permissions
- Compliance requirements not yet met (FedRAMP, SOC2, data residency)
- Secret management gaps or credential exposure risk
- Missing audit trails for regulated operations

### Infrastructure and Operations

- Capacity limits (quotas, resource ceilings) blocking growth
- Disaster recovery gaps (untested backups, missing failover)
- CI/CD pipeline fragility causing delivery delays
- Environment drift between staging and production
- Insufficient incident response runbooks or automation

### Customer and Business

- Feature delivery delays affecting customer commitments
- SLO targets at risk due to known reliability gaps
- Migration path gaps for customers on legacy configurations
- Missing customer communication channels for incidents or changes
