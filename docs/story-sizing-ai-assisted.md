# Proposal: Story Sizing for AI-Assisted Development

## Status

Draft — open for team feedback.

## Problem

Our current story sizing guide (in `docs/jira-story-template.md`) anchors on coding complexity: how many components, how much code, how tricky the logic. This made sense when engineers hand-wrote most implementation.

With AI agents handling increasing amounts of implementation — scaffolding controllers, generating boilerplate, applying patterns from specs — the bottleneck has shifted. We're observing:

- **Overestimation** of work that produces a lot of code but is straightforward to specify and verify (CRUD, boilerplate, repetitive patterns)
- **Underestimation** of work that produces little code but requires deep context assembly, security-critical verification, or iterative agent correction

Story points should reflect where the team actually spends effort, not where the code volume is.

## Proposal

Reframe story sizing around three effort dimensions. A story's size is driven by whichever dimension is highest — not an average.

### Dimension 1: Context Gathering

How much architectural knowledge must be assembled before the work can begin — whether by a human or as input to an agent?

| Score | Description |
|-------|-------------|
| Low | Self-contained change. Single file or component. No cross-cutting concerns. |
| Medium | Requires understanding 2-3 components or one design decision. Some reading of existing patterns. |
| High | Requires assembling knowledge across multiple repositories, services, or teams. Architectural trade-offs involved. May need a spike first. |

**Note:** Context gathering is front-loaded. The first story in a new area scores higher; follow-up stories in the same area benefit from accumulated context and score lower.

### Dimension 2: Verification Effort

How hard is it to validate that the output — whether human-written or agent-generated — is correct, secure, and complete?

| Score | Description |
|-------|-------------|
| Low | Standard unit tests. Output is deterministic and easy to inspect. Low blast radius. |
| Medium | Requires integration testing or manual validation. Multiple edge cases to check. Touches shared infrastructure. |
| High | Security-critical audit required. Cross-tenant isolation, IAM, encryption, or compliance implications. Failure modes are subtle and hard to detect. |

**This is typically the highest-weight dimension.** "The agent wrote it in 5 minutes, but we spent 3 days validating tenant isolation edge cases" is a real pattern.

### Dimension 3: Specification Clarity

Can the work be described as a clear, unambiguous spec — or does it require judgment calls that an agent (or a less experienced engineer) will likely get wrong?

| Score | Description |
|-------|-------------|
| Low | Well-defined inputs/outputs. Established patterns exist to follow. Clear acceptance criteria. |
| Medium | Some ambiguity in requirements. May need iteration to get right. Multiple valid approaches. |
| High | Requires novel design or domain judgment. No established pattern. High likelihood of rework or multiple agent iterations. |

## Revised Pointing Criteria

The Fibonacci scale (1, 2, 3, 5, 8, 13) remains unchanged. What changes is how we arrive at the number.

| Points | Traditional View | AI-Assisted View |
|--------|-----------------|------------------|
| **1** | One-line change, no risk | Clear spec, trivial verification, no context needed |
| **2** | Simple, well-understood | Agent can scaffold with minimal guidance, standard verification |
| **3** | Straightforward but time-consuming | Moderate context or verification, clear spec |
| **5** | Requires investigation and design | High in one dimension (deep context OR hard verification OR unclear spec) |
| **8** | Big and challenging — consider splitting | High in two or more dimensions — **must split** |
| **13** | Must be split | High in all three dimensions — split into spikes and implementation stories |

## Examples

### Example 1: CRUD Boilerplate for a New Resource

**Old sizing:** "500 lines of Go across controller, API types, and tests — 5 points"

**New sizing:**
- Context Gathering: Low — follows existing controller pattern
- Verification: Low — standard unit tests, deterministic output
- Specification Clarity: Low — well-defined CRUD operations

**Revised: 2 points.** Agent scaffolds from spec, engineer reviews and adjusts.

### Example 2: Add OAuth Middleware to 3 Services

**Old sizing:** "A lot of code, touches 3 services — 8 points"

**New sizing:**
- Context Gathering: Medium — need to understand auth flow across services
- Verification: High — security-critical, tenant isolation, token handling edge cases
- Specification Clarity: Medium — multiple valid approaches to token propagation

**Revised: 8 points, but for different reasons.** The effort is in verification and getting the spec right, not in writing the code.

### Example 3: Add a Prometheus Metric

**Old sizing:** "Simple, well-understood — 2 points"

**New sizing:**
- Context Gathering: Low — single component
- Verification: Low — metric shows up or it doesn't
- Specification Clarity: Low — clear what to measure

**Revised: 1-2 points.** No change — this was correctly sized already.

### Example 4: Migrate Operator from In-Cluster to Out-of-Cluster

**Old sizing:** "Complex, lots of code changes — 8 points, should split"

**New sizing:**
- Context Gathering: High — cross-component, RBAC, network policy, deployment topology
- Verification: High — must validate across environments, failure modes are subtle
- Specification Clarity: High — no established pattern, novel design required

**Revised: 13 — must split** into spike (context gathering) + implementation stories (scoped by component).

## Impact on Existing Process

- **Story template**: The existing pointing criteria table in `docs/jira-story-template.md` remains valid. This proposal adds the three dimensions as a lens for arriving at the number, not a replacement for the scale.
- **Velocity**: Re-baselining is expected. Stories that were historically 5 points may now be 2. This is correct — the team is faster at those tasks now. Velocity should reflect actual throughput, not maintain artificial continuity.
- **Grooming**: During refinement, ask: *"Where does the effort actually live — in spec, in code, or in verification?"* This single question captures the framework without requiring teams to formally score three dimensions per story.
- **Splitting criteria**: The existing splitting guidance still applies. Additionally, consider splitting when a story is High in two or more dimensions — separate the context gathering (spike) from the implementation.

## Open Questions

1. Should we formally score each dimension during grooming, or use them as informal guidance?
2. Do we need to update the story template examples to reflect AI-assisted workflows?
3. How do we handle stories where the agent cannot help at all (infrastructure provisioning, manual operations)?
