# E2E Infrastructure Testing with HCP Terraform Ephemeral Workspaces

***Scope***: GCP-HCP

**Date**: 2026-08-10

## Decision

E2E infrastructure validation will use HCP Terraform (TFC) ephemeral workspaces with CLI-driven remote execution. Each test run renders a self-contained Terraform config from a template, creates an isolated TFC workspace, deploys the full region + management-cluster + customer-project stack, and destroys it after validation. The render script bundles all modules and metadata into a single directory so TFC can execute remotely without access to the source repository.

## Context

- **Problem Statement**: The GCP-HCP platform needs automated end-to-end infrastructure validation that can run in parallel across CI pipelines. The existing e2e-smoke test uses a long-lived GCP project with Tekton pipelines for Hypershift testing, but cannot validate Terraform infrastructure provisioning itself — the GCP projects, GKE clusters, networking, IAM, and service configuration that Terraform manages. Changes to Terraform modules (region, management-cluster, customer-project) need a way to validate that they can successfully provision and destroy a full environment before merging to main. This is tracked under [GCP-848](https://redhat.atlassian.net/browse/GCP-848).

- **Constraints**:
  - Must support parallel test runs (multiple PRs testing simultaneously)
  - Must use the same TFC infrastructure already adopted for production ([GCP-536](https://redhat.atlassian.net/browse/GCP-536))
  - GCP project IDs are globally unique and limited to 30 characters
  - `user_project_override = true` is required for API quota routing but breaks folder-level Resource Manager API calls
  - OPA policies block `roles/owner` grants and folder IAM bindings in the `gcp-hcp-ci` TFC project
  - Must not require manual cleanup — failed runs should be recoverable without admin intervention
  - All infrastructure must be IaC — no one-off gcloud commands

- **Assumptions**:
  - CI environments bootstrap from integration's shared infrastructure (global project, commons, platform-ci folder) since CI does not have its own global or service project
  - TFC WIF authentication is pre-configured via the `gcp-hcp-ci-tfc-access` workspace and project-scoped variable sets ([WIF design decision](hcp-terraform-workload-identity-federation.md))
  - The e2e template deploys the same modules as production with feature flags to disable components that depend on infrastructure outside the ephemeral environment (DNS delegation, Atlantis IAM)
  - A workspace cleanup mechanism (script or scheduled job) will handle empty workspace removal separately from this design (follow-up work to be tracked)

## Alternatives Considered

1. **TFC ephemeral workspaces with CLI-driven execution (selected)**: A render script copies a template directory, replaces placeholders, bundles modules/metadata, and produces a self-contained config. `terraform init` auto-creates a workspace in the `gcp-hcp-ci` TFC project. `terraform apply` runs remotely via CLI-driven mode. Each run gets isolated state. Cleanup via `terraform destroy` or TFC `auto-destroy-at` API.

2. **VCS-driven TFC workspaces with trigger prefixes**: Connect TFC workspaces directly to the repository with `trigger_prefixes` pointing at module directories. Each PR creates a workspace via TFC API, TFC clones the repo and runs plan/apply. No render script needed.

3. **Tekton pipeline with local Terraform execution**: Extend the existing Tekton pipeline infrastructure to run `terraform apply` inside a Tekton Task. State stored in GCS. Parallel runs via separate PipelineRuns with unique state prefixes.

4. **Atlantis-based e2e**: Use the existing Atlantis infrastructure to run e2e plans and applies from a dedicated e2e branch or PR. Atlantis already has the IAM permissions and provider configuration.

5. **Dedicated long-lived e2e environment**: Maintain a permanent e2e GCP project (like the existing e2e-smoke setup) and run `terraform apply` against it. No ephemeral resources — test by updating in place.

## Decision Rationale

* **Justification**: Alternative 1 (TFC ephemeral workspaces) provides complete isolation between parallel test runs, uses the same TFC platform being adopted for production, and requires no additional infrastructure beyond what already exists. CLI-driven mode avoids the complexity of VCS-driven workspace management while keeping execution remote (consistent with production). The render script solves the key challenge: TFC remote execution needs all files uploaded, but the repo structure uses relative paths (`../../../modules/`) that only work locally. Bundling produces a self-contained directory that works in both local and remote contexts.

* **Evidence**: Validated end-to-end across multiple test runs:
  - test010: 437 resources applied → 0-error destroy (single-shot success)
  - test011: 452 resources applied → auto-destroy via TFC `auto-destroy-at` API → TFC triggered "infrastructure lifecycle" destroy → 0 resources remaining
  - Full stack deployed: 2 GKE Autopilot clusters, 3 GCP projects, 2 folders, Cloud DNS zones, Cloud Run services, Workflows, PubSub topics, Secret Manager secrets, IAM bindings, VPC networks, fleet registrations

* **Comparison**:
  - **Alternative 2 (VCS-driven)** requires the workspace to have repo access and know which branch to clone. For PR-based testing, this means creating workspaces that track feature branches — complex lifecycle management. CLI-driven mode is simpler: upload the config, run it, done.
  - **Alternative 3 (Tekton)** duplicates the provider configuration and IAM setup already managed by TFC. Running Terraform locally inside a Tekton Task means managing state buckets, credential injection, and concurrency separately from TFC — the platform the team is migrating to.
  - **Alternative 4 (Atlantis)** only supports PR-based workflows and is being deprecated in favor of TFC. Using it for e2e creates a dependency on infrastructure being removed.
  - **Alternative 5 (long-lived environment)** cannot validate create-from-scratch flows, masks state drift issues, and doesn't support parallel testing.

## Consequences

### Positive

* Complete isolation — each test run creates and destroys its own GCP projects, folders, clusters, and networking with no shared mutable state
* Parallel execution — multiple PRs can run e2e tests simultaneously since each workspace has independent state
* Same platform as production — e2e tests exercise the same TFC remote execution, WIF auth, and OPA policy enforcement used in production
* Create-from-scratch validation — every run proves the modules can provision a full environment from nothing, catching bootstrap ordering issues that update-in-place testing misses
* Auto-destroy safety net — TFC `auto-destroy-at` API provides a backstop against abandoned test infrastructure

### Negative

* Each test run takes ~15-20 minutes for apply and ~10 minutes for destroy — slower than running against existing infrastructure
* Render script adds a build step and must be kept in sync with module changes (new modules, path changes)
* Empty TFC workspaces accumulate after destroy and need periodic cleanup
* CI environment bootstraps from integration state — changes to integration's global or platform-ci outputs can break e2e runs
* `user_project_override` requires careful provider aliasing (`google.bootstrap` for folder creation, `google.project_creation` for folder IAM and project creation) — a pattern that must be maintained across module changes

## Cross-Cutting Concerns

### Security:

* TFC SAs authenticate via Workload Identity Federation — no static keys. Tokens are short-lived (~1h) and scoped to the TFC run phase (plan or apply)
* OPA policies enforce `block_folder_iam` and `block_owner_grants` on all e2e workspaces — same enforcement as production
* The e2e template passes `folder_iam_bindings = {}` to comply with `block_folder_iam` policy rather than requesting an exception
* TFC SA permissions are bootstrapped via project-creator SA impersonation — the TFC SA never holds persistent elevated permissions. The impersonation grant is scoped to all workspaces in the `gcp-hcp-ci` TFC project (via `apply_to_all_workspaces`), which is acceptable since all workspaces in this project are ephemeral e2e runs. OPA policies (`block_folder_iam`, `block_owner_grants`) provide guardrails against privilege escalation within those workspaces

### Reliability:

* **Destroy ordering**: GKE clusters depend on `null_resource.tfc_iam_ready` so IAM permissions are not revoked before cluster deletion completes. Without this, destroy fails with 403 errors on GKE API calls mid-deletion
* **Provider aliasing**: Folder creation uses `google.bootstrap` (impersonated SA, `billing_project` = global project) rather than the default provider (whose `billing_project` = `gcp-hcp-platform-ci` lacks folder permissions). `google.project_creation` has no `user_project_override` at all, making it safe for folder IAM operations. This is validated in e2e and will catch regressions if provider configuration changes
* **Resiliency**: Failed destroys should be retried with full state — never remove resources from Terraform state to work around permission errors, as this orphans GCP resources requiring admin cleanup

### Cost:

* Each e2e run creates 2 GKE Autopilot clusters, 3 GCP projects, and associated networking — approximately $2-5 per test run at current pricing
* Auto-destroy limits cost exposure from abandoned runs — the `auto-destroy-at` timer should be set via TFC API before `terraform apply` begins, so that even if apply or the CI pipeline crashes, the workspace has a destruction deadline. If the API call fails, the pipeline should abort before apply. The workspace cleanup process should also detect workspaces without auto-destroy timers as a secondary safety net
* No persistent infrastructure cost between test runs

### Operability:

* Render script validates inputs (RUN_ID format, REGION against `regions.yaml`) before producing output
* Rendered directories are gitignored and disposable — no need to commit test artifacts
* Workspace naming convention (`platform-e2e-{run_id}`) makes it easy to identify and filter test workspaces in the TFC UI
* All module feature flags (`enable_atlantis`, `enable_dns_delegation`, `skip_region_project_lookup`, `folder_iam_bindings`) are documented in the template with comments explaining why each is set

---

## Implementation Details

### Template Architecture

The e2e template lives at `terraform/config/platform-ci/@e2e/` and consists of:

- `cloud.tf` — TFC backend configuration with `@WORKSPACE_NAME@` placeholder
- `main.tf` — Module calls for region, management-cluster, and customer-project with e2e-specific settings

The render script (`scripts/e2e-render.sh`) produces a self-contained directory:

```
platform-ci/e2e-{run_id}/
├── cloud.tf          # TFC backend (workspace name substituted)
├── main.tf           # Module calls (placeholders substituted, paths rewritten)
├── modules/          # Bundled copies of region, management-cluster, customer-project, workflows, etc.
├── metadata/         # Bundled environments.yaml, regions.yaml, defaults.yaml, infra_ids.yaml
└── workflows/        # Bundled Cloud Workflows YAML definitions
```

### Provider Strategy

The e2e template configures three provider personas:

| Provider | Authentication | `user_project_override` | Used For |
|---|---|---|---|
| `google` (default) | TFC SA via WIF | `true` (billing: `gcp-hcp-platform-ci`) | Most resources |
| `google.project_creation` | Impersonated project-creator SA | not set | Project creation, folder IAM, TFC IAM bootstrap |
| `google.bootstrap` | Impersonated project-creator SA | `true` (billing: global project) | Folder creation (Resource Manager API) |

The `user_project_override` gotcha: enabling it routes ALL GCP API quota through the billing project. When the billing project lacks the necessary folder-level permissions, this causes `folders.get` permission denied errors even with `folderAdmin`. The solution is provider aliasing — `google.bootstrap` routes quota through the global project (which has folder permissions), and `google.project_creation` has no `user_project_override` at all (safe for folder IAM). Both impersonate the project-creator SA.

### Feature Flags

| Flag | Default | E2E Value | Purpose |
|---|---|---|---|
| `enable_atlantis` | `true` | `false` | Skip ~46 Atlantis IAM resources not needed in TFC-only runs |
| `enable_dns_delegation` | `true` | `false` | Skip NS record delegation to parent zones that don't exist for CI |
| `skip_region_project_lookup` | `false` | `true` | Skip `data.google_project.gkehub` that fails at plan time in single-state deployments |
| `folder_iam_bindings` | (map) | `{}` | Skip folder IAM to pass OPA `block_folder_iam` policy |
| `enable_tfc` | `false` | `true` | Enable TFC SA IAM bootstrap chain |

### Lifecycle

```
render → init (workspace auto-created) → apply → [validate] → destroy
                                                              ↑
                                              or: auto-destroy-at API
```

Workspaces persist after destroy with run history for debugging. Empty workspace cleanup is handled by a separate process.

### Related PRs

| PR | Repository | Description |
|---|---|---|
| [#1050](https://github.com/openshift-online/gcp-hcp-infra/pull/1050) | gcp-hcp-infra | E2E template + render script |
| [#1113](https://github.com/openshift-online/gcp-hcp-infra/pull/1113) | gcp-hcp-infra | `enable_atlantis` variable |
| [#1101](https://github.com/openshift-online/gcp-hcp-infra/pull/1101) | gcp-hcp-infra | Folder IAM grants for TFC SA |
| [#1088](https://github.com/openshift-online/gcp-hcp-infra/pull/1088) | gcp-hcp-infra | Billing APIs on platform-ci |
| [#1075](https://github.com/openshift-online/gcp-hcp-infra/pull/1075) | gcp-hcp-infra | TFC bootstrap in region/MC modules |
