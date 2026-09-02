---
name: hcp-feature-support-assessment
description: Assess GCP HCP feature support across HyperShift, Gecko, gcp-hcp-ctl, and HO dependencies; use when asked for supportability, implementation scope, or level of effort for a feature request.
---

# HCP Feature Support Assessment

Use this skill to assess whether a requested GCP HCP feature is already supported, what implementation scope remains, and the level of effort. The assessment must be based on repository evidence, not on assumptions.

## Repositories

Inspect these repositories as applicable. Prefer an available local clone for fast search and accurate line numbers. If a local clone is unavailable, use the GitHub URL to browse or fetch the relevant files.

| Repository | GitHub URL | Local Hint |
|------------|------------|------------|
| HyperShift and HyperShift Operator (HO) | `https://github.com/openshift/hypershift` | `../hypershift` |
| Gecko platform API and controllers | `https://github.com/openshift-online/gecko` | `../gecko` |
| `gcphcpctl` CLI | `https://github.com/openshift-online/gcp-hcp-ctl` | `../gcp-hcp-ctl` |
| GCP HCP design docs, plans, and project context | `https://github.com/openshift-online/gcp-hcp` | current repository |

Resolve local hints relative to the parent directory of the current repository. Do not hardcode user-specific absolute paths in the assessment or evidence.

Discover additional HO dependencies dynamically from HyperShift code, `go.mod`, and imports. CAPG is one possible dependency, not a fixed or exhaustive dependency list.

For reliable assessments, local clones of HyperShift, Gecko, and `gcp-hcp-ctl` are expected. Some repositories may be private or require GitHub authentication, so public web fetching may not work. If a required repository cannot be inspected locally or remotely, report the assessment as partial in `SUMMARY` and cite the missing repository in `EVIDENCE`.

## Repository Access Workflow

Before assessing supportability:

1. Determine the current repository root and its parent directory.
2. Check each local hint under the parent directory.
3. If a local clone exists, use that clone for `Glob`, `Grep`, and `Read` calls.
4. If a local clone is missing, try the GitHub URL only for public or authenticated access.
5. If neither local nor remote inspection is available, do not infer support. Mark the finding as unverified and explain which scope could not be inspected.

When using opencode tools from the `gcp-hcp` repository, remember that `Glob` and `Grep` default to the current repository. Pass the target repository path explicitly when searching sibling clones.

When using remote sources, prefer raw source files or fetched repository files that preserve line numbers. If line numbers cannot be verified remotely, cite the file path and state that line-level evidence was unavailable.

For broad cross-repo exploration, use the `Task` tool with `subagent_type="explore"` and point each task at one repository. Ask each subagent to return only evidence relevant to the requested feature, including file paths, line numbers, positive findings, and missing areas searched.

If the `Task` tool or `explore` subagent is unavailable, fall back to direct `Glob`, `Grep`, `Read`, and `webfetch` calls while preserving the same evidence standards.

Run cross-repo searches in parallel when possible:

- One search for HyperShift support.
- One search for Gecko API/controller exposure.
- One search for `gcp-hcp-ctl` CLI exposure.
- One search for GCP HCP design context.
- Additional searches for dependencies only after HyperShift evidence identifies them.

## Branch And Version Scope

Assess the default branch of each repository unless the user specifies a release, branch, commit, or product version. If the user specifies a version, inspect that branch or tag when available and state the inspected revision in `SUMMARY` or `EVIDENCE`.

Do not report `SUPPORTED: TRUE` solely because code exists on a branch that is not the requested branch or release line. If support exists only on a newer branch, report `SUPPORTED: FALSE` or partial support and explain the backport/release gap.

## Assessment Rules

Classify the request by the smallest scope that can correctly deliver the feature.

### Category 1: HyperShift Passthrough

Use when HyperShift already supports the feature through `HostedCluster`, `NodePool`, or another HO-managed API, and Gecko only needs to expose and pass through the same capability.

Typical scope:

- `GECKO-API`
- `GECKO-CONTROLLER`
- `CLI`

In this category, `GECKO-CONTROLLER` means adapter plumbing that copies the field from Gecko API resources into HO resources. It does not mean a new service workflow, asynchronous orchestration, external API lookup, durable state machine, or complex reconciliation.

### Category 2: Gecko Or CLI Transform

Use when HyperShift already supports the target configuration, but customer input must be transformed before Gecko writes the HO-facing resource. Examples include resolving a customer version to a release image, deriving a channel, normalizing a user-friendly option into a HyperShift enum, or expanding a short input into a full spec fragment.

Typical scope:

- `GECKO-API`
- `GECKO-CONTROLLER`
- `CLI`

If the transformation is purely client-side and does not need durable status, retries, credentials, or reconciliation, do not include `GECKO-CONTROLLER`.

Include `GECKO-CONTROLLER` when the transformation requires external API calls, credentials, retries, cached state, status reporting, asynchronous completion, cross-resource coordination, or durable reconciliation.

### Category 3: Gecko Controller Or Service Logic

Use when the customer API requires non-trivial Gecko reconciliation or orchestration before writing `HostedCluster`, `NodePool`, or other HO-facing resources. Examples include upgrade services, asynchronous workflows, placement decisions, multi-resource orchestration, status feedback, retries, or policy-driven state machines.

Typical scope:

- `GECKO-API`
- `GECKO-CONTROLLER`
- `CLI`

### Category 4: HyperShift Operator Implementation

Use when HyperShift does not currently expose or implement the feature, or when HO must change its API, validation, controller logic, rendered manifests, ignition, NodePool behavior, CPO behavior, or feature gates before Gecko can consume it.

Typical scope:

- `HO`
- Plus `GECKO-API`, `GECKO-CONTROLLER`, or `CLI` depending on the customer-facing exposure path.

Before adding `HO`, verify that existing HyperShift fields and controllers cannot already express the requested behavior.

### Category 5: HyperShift Dependency Implementation

Use when the work requires implementation in an HO dependency before HO can support it. Examples include CAPI providers, CAPI, cloud provider operators, installer/image payload dependencies, or downstream forks that must be synced and released before HyperShift can consume them.

Typical scope:

- `HO-DEPENDENCY`
- `HO`
- Plus `GECKO-API`, `GECKO-CONTROLLER`, or `CLI` depending on the exposure path.

For dependency work, include upstream implementation, downstream/fork sync if applicable, dependency bump, HO integration, and Gecko/CLI exposure in the summary.

## Discovering HO Dependencies

When the feature appears to require provider, CAPI, cloud operator, or external component behavior that HyperShift itself does not implement:

1. Identify the infrastructure or platform layer the feature touches: compute, networking, storage, identity, ingress, DNS, machine lifecycle, release payload, credentials, observability, or another area.
2. Search HyperShift's `go.mod` for modules that provide relevant APIs or controllers.
3. Search HyperShift API and controller code for imports, type references, annotations, CRD fields, and comments related to the dependency.
4. Determine the dependency's upstream repository from the Go module path, import path, existing documentation, or generated code headers.
5. Check whether a local sibling clone exists using the repository name derived from the module or GitHub URL.
6. If a local clone exists, inspect it for the needed type, CRD field, controller behavior, tests, and release status.
7. If no local clone exists, use the GitHub URL to inspect relevant API types, CRDs, controller code, issues, or documentation when available.
8. Report the dependency module path or repository URL, what is missing or already present, and whether the work is upstream, downstream-only, a dependency bump, or already available but not consumed by HO.

## Required Investigation Workflow

Follow this order unless the request makes another order obviously better:

1. Restate the feature request in concrete terms: customer input, desired behavior, and likely affected resources.
2. Identify the requested branch, release, or product version. If none is specified, use the default branch and say so only when relevant.
3. Search HyperShift first for API fields, CRD schema, controller logic, validation, feature gates, docs, and tests related to the feature.
4. Verify that the support applies to GCP specifically, not only to AWS, Azure, IBM Cloud, KubeVirt, Agent, or another platform.
5. Search Gecko platform API for customer-facing types and generated public API fields.
6. Search Gecko controllers for adapters or reconcilers that map Gecko API state to HyperShift `HostedCluster`, `NodePool`, or related resources.
7. Search `gcp-hcp-ctl` for command flags, request payload construction, API client models, documentation, and tests.
8. Search GCP HCP design docs and implementation plans for known decisions, constraints, or intended behavior.
9. Discover and inspect dependency repositories only when HyperShift support appears blocked by provider, CAPI, cloud operator, or external component functionality.
10. Identify the minimum complete scope and level of effort.
11. Report concise evidence with file paths and line numbers where possible.

Do not implement code while using this skill unless the user explicitly asks for implementation. The primary output is an assessment.

## Search Guidance

Use `Glob` and `Grep` first. Prefer targeted searches over broad summaries.

Useful HyperShift locations and patterns:

- `api/**/*.go`
- `hypershift-operator/**/*.go`
- `control-plane-operator/**/*.go`
- `support/**/*.go`
- `docs/content/**/*.md`
- Search terms: feature name, related Kubernetes/OpenShift field names, `HostedClusterSpec`, `NodePoolSpec`, `HostedControlPlaneSpec`, `featuregate`, `configHash`, `hashStruct`, platform-specific structs, provider structs.

Useful Gecko locations and patterns:

- `platform-api/api/private/v1/*.go`
- `platform-api/api/public/v1/*.go`
- `controllers/**/*.go`
- `orlop/**/*.go`
- Search terms: feature name, public JSON field name, private type name, controller name, adapter name, `HostedCluster`, `NodePool`, `ManifestWork`, `VersionResolution`, `PlacementResult`.

Useful CLI locations and patterns:

- `pkg/cluster/**/*.go`
- `pkg/nodepool/**/*.go`
- `pkg/platformapi/**/*.go`
- `cmd/**/*.go`
- Search terms: feature name, flag name, JSON field name, request type, `cobra`, `Create`, `Update`, `Scale`, `version`, `channel`.

Useful GCP HCP context locations:

- `design-decisions/**/*.md`
- `implementation-plans/**/*.md`
- `studies/**/*.md`
- `docs/**/*.md`

Dependency indicators:

- Missing provider-side field or behavior in a CAPI provider, CAPI, cloud operator, or other HO dependency.
- HyperShift code cannot express the behavior without dependency API changes.
- HyperShift `go.mod` pins a dependency version that lacks the needed type or behavior.
- Existing TODOs, issues, or comments explicitly state dependency work is required.

## Support Determination

Set `SUPPORTED: TRUE` only when the complete customer-facing path already exists and appears usable without implementation in all required layers. This means HyperShift support alone is not enough if Gecko API or CLI exposure is required by the requested product surface.

Set `SUPPORTED: FALSE` when inspected evidence shows any required layer is missing, when only partial feature support exists, or when support depends on unimplemented upstream/provider work.

If HyperShift supports the low-level capability but Gecko does not expose or pass it through, or the CLI does not expose it to users, report `SUPPORTED: FALSE` with scope `GECKO-API, GECKO-CONTROLLER, CLI` or the minimal applicable set.

If HyperShift supports the feature for another platform but not GCP, report `SUPPORTED: FALSE` and include `HO` or `HO-DEPENDENCY` if GCP platform support needs implementation.

If one layer supports the feature but another required layer does not, report `SUPPORTED: FALSE` and list only the missing implementation scopes. Mention the existing support in `SUMMARY` or `EVIDENCE`.

If a required repository, branch, or dependency could not be inspected, avoid a definitive positive assessment. Use `SUPPORTED: FALSE` only when inspected evidence shows a required layer is missing. Otherwise, report the assessment as partial in `SUMMARY`, set `CONFIDENCE: LOW`, and cite the missing source in `EVIDENCE`.

If the request is ambiguous, make the narrowest reasonable interpretation and include the assumption in `SUMMARY`. Ask a short clarification question only when the ambiguity changes the scope materially.

## Scope Values

Use only these scope values:

- `NONE`: no implementation needed.
- `CLI`: `gcphcpctl` flags, commands, request payloads, output, docs, or tests.
- `GECKO-API`: Gecko customer-facing API types, generated public API, validation, OpenAPI/CRDs, or API tests.
- `GECKO-CONTROLLER`: Gecko reconciliation, adapters, orchestration, async state, status, retries, service integrations, or writes to HO resources.
- `HO`: HyperShift API, validation, controllers, CPO/HO rendering, feature gates, tests, docs, or release payload integration.
- `HO-DEPENDENCY`: CAPI provider, CAPI, cloud operator, upstream, downstream dependency, or release payload changes plus dependency bump/release work.

When multiple scopes are needed, list them comma-separated in dependency order, for example:

`SCOPE: HO-DEPENDENCY, HO, GECKO-API, GECKO-CONTROLLER, CLI`

## Level Of Effort

`SCOPE` describes where code must change. `LEVEL OF EFFORT` describes how hard the implementation is. Do not increase LOE only because several scope values are listed; increase it based on the hardest required behavior. Missing evidence lowers confidence, not necessarily effort.

Use this rubric:

- `NONE`: No implementation is needed because the complete customer-facing path is already supported and usable.
- `S`: Mechanical passthrough, CLI flag, simple API exposure, adapter field copy, docs, or local tests. Can span multiple layers if no new behavior, transform, reconciliation, dependency bump, or complex validation is required.
- `M`: Multiple-layer implementation with generated API/client updates, moderate validation, simple transformation, version/channel normalization, or non-trivial but local tests. No new durable controller workflow or upstream dependency work.
- `L`: New HyperShift behavior, new Gecko reconciliation/service logic, lifecycle semantics, rollout/upgrade/replacement behavior, status handling, retries, external API interaction, or significant integration/e2e test work.
- `XL`: Upstream/downstream dependency implementation, dependency release or bump chain, provider/CAPI/cloud-operator changes, release payload coordination, or broad cross-repo work blocked on dependency availability.

Examples:

- `NONE`: The requested feature is already exposed through HyperShift, Gecko API/controller plumbing, and `gcphcpctl` with no required implementation.
- `S`: Add a `gcphcpctl` flag and copy an existing Gecko API field into a HyperShift `NodePool`.
- `S`: Add Gecko API, controller passthrough, and CLI exposure for an existing HyperShift GCP field.
- `M`: Add Gecko API, generated clients, validation, CLI payload support, and a simple transform before writing HyperShift resources.
- `L`: Add Gecko reconciliation that calls an external API, persists status, retries, and coordinates cluster or node pool lifecycle.
- `XL`: Add missing CAPG/provider support, release it, bump HyperShift, implement HO integration, then expose it through Gecko and CLI.

## Risk

Use this rubric:

- `LOW`: Mechanical field passthrough, local validation, CLI/docs updates, no customer-visible lifecycle behavior change, no dependency or release-chain uncertainty.
- `MEDIUM`: Public API compatibility concerns, generated clients, moderate validation, uncertain rollout behavior, multiple repos needing coordinated changes, or integration/e2e coverage needed.
- `HIGH`: Upgrade or replacement semantics, long-running orchestration, credentials or external service calls, dependency release chain, production migration, provider limitations, or incomplete evidence for a critical required layer.

## Confidence

Use this rubric:

- `HIGH`: All required repositories and relevant branches were inspected locally or remotely, and evidence covers positive and negative findings.
- `MEDIUM`: Most required layers were inspected, but some evidence is indirect, generated, stale, or missing line-level confirmation.
- `LOW`: One or more required repositories, branches, dependencies, or generated artifacts could not be inspected, or the feature request is materially ambiguous.

## Required Output Format

Return exactly this structure:

```text
SUPPORTED: TRUE|FALSE
SCOPE: NONE|CLI|GECKO-API|GECKO-CONTROLLER|HO|HO-DEPENDENCY[, ...]
LEVEL OF EFFORT: NONE|S|M|L|XL
RISK: LOW|MEDIUM|HIGH
CONFIDENCE: HIGH|MEDIUM|LOW
SUMMARY: One or two sentences explaining current support, what is needed, and whether the assessment is complete or partial.
EVIDENCE:
- repo-relative/path:line - concise finding
- repo-relative/path:line - concise finding
```

Keep the answer objective. Do not include a long implementation plan unless the user explicitly asks for one.

If the assessment is partial because a repository, branch, or dependency could not be inspected, say so in `SUMMARY` and include an `EVIDENCE` bullet for the missing source.

## Evidence Quality

Evidence should include positive and negative findings. Prefer file paths with line numbers from code reads or grep results.

Good evidence:

- `hypershift/api/hypershift/v1beta1/nodepool_types.go:123 - NodePool exposes the requested field for GCP.`
- `gecko/platform-api/api/private/v1/nodepool_types.go:59 - Gecko exposes GCP node pool platform fields, but the requested field is absent.`
- `gcp-hcp-ctl/pkg/nodepool/create.go:88 - CLI create payload does not accept a flag for the requested field.`

Avoid vague evidence:

- `Searched the repo and did not find it.`
- `Probably supported by HyperShift.`

If no direct code evidence exists, cite the most relevant searched files and state the absence precisely.

## Worked Example

Example request:

`Assess support for setting GCP resource labels on node pools.`

Expected investigation:

1. Search HyperShift `NodePool` GCP platform API and controller code for resource label support.
2. Verify the field is under GCP-specific node pool platform configuration, not only another provider.
3. Search Gecko private and public `NodePool` API types for the same customer-facing field.
4. Search Gecko controllers or adapters to confirm the field is copied into HyperShift `NodePool` specs.
5. Search `gcp-hcp-ctl` nodepool create/update commands for flags and request payload fields.

Example output shape:

```text
SUPPORTED: FALSE
SCOPE: GECKO-CONTROLLER, CLI
LEVEL OF EFFORT: S
RISK: LOW
CONFIDENCE: HIGH
SUMMARY: HyperShift and Gecko expose GCP node pool resource labels, but the assessment did not find adapter plumbing or CLI payload wiring for customers to set them. The remaining work is passthrough wiring from Gecko to HO plus a CLI flag and unit coverage.
EVIDENCE:
- hypershift/api/hypershift/v1beta1/gcp.go:467 - GCP node pool platform spec includes resource labels.
- gecko/platform-api/api/private/v1/nodepool_types.go:79 - Gecko node pool GCP platform spec includes resource labels.
- gecko/controllers/nodepool/manifest/manifests.go:105 - Node pool manifest builder copies existing GCP platform fields, but not resource labels.
- gcp-hcp-ctl/pkg/nodepool/create.go:48 - Node pool create flags include machine type, disk, zone, and version options, but no resource labels flag.
```

This example is illustrative. Always inspect current code before returning an assessment.
