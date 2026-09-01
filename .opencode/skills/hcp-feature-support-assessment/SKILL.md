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

## Assessment Rules

Classify the request by the smallest scope that can correctly deliver the feature.

### Category 1: HyperShift Passthrough

Use when HyperShift already supports the feature through `HostedCluster`, `NodePool`, or another HO-managed API, and Gecko only needs to expose the same capability.

Typical scope:

- `GECKO-API`
- `CLI`

### Category 2: Gecko Or CLI Transform

Use when HyperShift already supports the target configuration, but customer input must be transformed before Gecko writes the HO-facing resource. Examples include resolving a customer version to a release image, deriving a channel, normalizing a user-friendly option into a HyperShift enum, or expanding a short input into a full spec fragment.

Typical scope:

- `GECKO-API`
- `GECKO-CONTROLLER`
- `CLI`

If the transformation is purely client-side and does not need durable status, retries, credentials, or reconciliation, do not include `GECKO-CONTROLLER`.

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
2. Search HyperShift first for API fields, CRD schema, controller logic, validation, feature gates, docs, and tests related to the feature.
3. Search Gecko platform API for customer-facing types and generated public API fields.
4. Search Gecko controllers for adapters or reconcilers that map Gecko API state to HyperShift `HostedCluster`, `NodePool`, or related resources.
5. Search `gcp-hcp-ctl` for command flags, request payload construction, API client models, documentation, and tests.
6. Search GCP HCP design docs and implementation plans for known decisions, constraints, or intended behavior.
7. Discover and inspect dependency repositories only when HyperShift support appears blocked by provider, CAPI, cloud operator, or external component functionality.
8. Identify the minimum complete scope and level of effort.
9. Report concise evidence with file paths and line numbers where possible.

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

Set `SUPPORTED: FALSE` when any required layer is missing, when only partial support exists, or when support depends on unimplemented upstream/provider work.

If HyperShift supports the low-level capability but Gecko/CLI do not expose it, report `SUPPORTED: FALSE` with scope `GECKO-API, CLI` or the minimal applicable set.

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

Use this rubric:

- `S`: Straightforward passthrough in one or two layers; no new controller logic; no new dependency; tests are local and obvious.
- `M`: Multiple layers need updates, or a simple transform/validation/generation path is required; no new long-running controller/service workflow; no upstream dependency work.
- `L`: New HO behavior, new Gecko controller/service logic, multi-resource orchestration, upgrade/rollout semantics, significant validation, or complex tests are required.
- `XL`: Any upstream/downstream dependency implementation and release chain, or broad cross-repo work spanning dependency, HO, Gecko controller, API, CLI, and e2e validation.

Choose the higher LOE when evidence points to uncertainty, rollout risk, API compatibility concerns, or missing dependency support.

## Required Output Format

Return exactly this structure:

```text
SUPPORTED: TRUE|FALSE
SCOPE: NONE|CLI|GECKO-API|GECKO-CONTROLLER|HO|HO-DEPENDENCY[, ...]
LEVEL OF EFFORT: S|M|L|XL
SUMMARY: One or two sentences explaining current support and what is needed.
EVIDENCE:
- repo-relative/path:line - concise finding
- repo-relative/path:line - concise finding
```

Keep the answer objective. Do not include a long implementation plan unless the user explicitly asks for one.

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
