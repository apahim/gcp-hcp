# E2E Infrastructure Testing with HCP Terraform Ephemeral Workspaces

***Scope***: GCP-HCP

**Date**: 2026-08-10

## Decision

E2E infrastructure validation will use HCP Terraform (TFC) ephemeral workspaces with CLI-driven remote execution. Each test run renders a self-contained Terraform config from a template, creates an isolated TFC workspace, deploys the full region + management-cluster + customer-project stack, and destroys it after validation. The render script bundles all modules and metadata into a single directory so TFC can execute remotely without access to the source repository. A future improvement is to publish modules to the TFC private registry, which would eliminate the bundling step.

## Context

- **Problem Statement**: The GCP-HCP platform needs automated end-to-end infrastructure validation that can run in parallel across CI pipelines. The platform needs a full end-to-end validation pipeline that can build infrastructure, deploy applications, and test hosted cluster provisioning. Today Terraform infrastructure provisioning is validated via Tekton pipelines on the integration global cluster, but this approach is brittle — it cannot integrate with GitHub slash commands, only supports one concurrent run due to fixed state paths, and running on GKE Autopilot has proven difficult to maintain — the GCP projects, GKE clusters, networking, IAM, and service configuration that Terraform manages. Changes to Terraform modules (region, management-cluster, customer-project) need a way to validate that they can successfully provision and destroy a full environment before merging to main.

- **Constraints**:
  - Must support parallel test runs (multiple PRs testing simultaneously)
  - Must use the same TFC infrastructure used for managing the static environments (integration, stage)
  - GCP project IDs are globally unique and limited to 30 characters
  - `user_project_override = true` routes API quota through a billing project, which requires careful provider configuration for operations like folder creation
  - OPA policies block `roles/owner` grants and folder IAM bindings in the `gcp-hcp-ci` TFC project
  - Must not require manual cleanup — failed runs should be recoverable without admin intervention
  - All infrastructure must be IaC — no one-off gcloud commands

- **Assumptions**:
  - CI environments target full isolation from other environments. DNS zones and state bucket access still depend on integration infrastructure and are being decoupled as follow-up work
  - TFC WIF authentication is pre-configured via the `gcp-hcp-ci-tfc-access` workspace and project-scoped variable sets ([WIF design decision](hcp-terraform-workload-identity-federation.md)). Prow authenticates to TFC via a team-level API token (`TFE_TOKEN`) mounted from a cluster profile secret — this token enables workspace creation and CLI-driven runs
  - The e2e template deploys the same modules as production with feature flags to disable components that depend on infrastructure outside the ephemeral environment (e.g., DNS delegation to parent zones)
  - A workspace cleanup mechanism (script or scheduled job) will handle empty workspace removal separately from this design (follow-up work to be tracked)

## Alternatives Considered

1. **TFC ephemeral workspaces with CLI-driven execution (selected)**: A render script copies a template directory, replaces placeholders, bundles modules/metadata, and produces a self-contained config. `terraform init` auto-creates a workspace in the `gcp-hcp-ci` TFC project. `terraform apply` runs remotely via CLI-driven mode. Each run gets isolated state. Cleanup via `terraform destroy` or TFC `auto-destroy-at` API.

2. **VCS-driven TFC workspaces with trigger prefixes**: Connect TFC workspaces directly to the repository with `trigger_prefixes` pointing at module directories. Each PR creates a workspace via TFC API, TFC clones the repo and runs plan/apply. No render script needed.

3. **Tekton pipeline with local Terraform execution** (deprecated): Extend the Tekton pipeline infrastructure to run `terraform apply` inside a Tekton Task. State stored in GCS. Parallel runs via separate PipelineRuns with unique state prefixes. Tekton is being replaced by Prow for CI, making this a dead-end investment.

4. **Atlantis-based e2e** (planned for decommission): Use the existing Atlantis infrastructure to run e2e plans and applies from a dedicated e2e branch or PR. Atlantis already has the IAM permissions and provider configuration. Atlantis is being decommissioned as TFC becomes the standard for Terraform automation.

5. **Dedicated long-lived e2e environment**: Maintain a permanent e2e GCP project and run `terraform apply` against it. No ephemeral resources — test by updating in place.

6. **Shell scripts with direct Terraform execution in Prow**: Run Terraform directly from Prow job steps via shell scripts. State stored in either GCS buckets or TFC workspaces. No render script — Prow clones the repo and runs Terraform with the correct working directory.

7. **Static e2e config folder with ephemeral state**: Create a permanent `terraform/config/e2e/` directory with a dedicated config that references the same modules but is parameterized for CI use. State isolation via per-run TFC workspaces or GCS prefixes. No render script — the config is committed and maintained alongside production configs.

## Decision Rationale

* **Justification**: Alternative 1 (TFC ephemeral workspaces) provides complete isolation between parallel test runs, uses the same TFC platform being adopted for production, and requires no additional Terraform execution platform beyond what already exists. CLI-driven mode avoids the complexity of VCS-driven workspace management while keeping execution remote (consistent with production). The render script solves the key challenge: TFC remote execution needs all files uploaded, but the repo structure uses relative paths (`../../../modules/`) that only work locally. Bundling produces a self-contained directory that works in both local and remote contexts.

* **Evidence**: Validated end-to-end across multiple test runs:
  - test010: 437 resources applied → 0-error destroy (single-shot success)
  - test011: 452 resources applied → auto-destroy via TFC `auto-destroy-at` API → TFC triggered "infrastructure lifecycle" destroy → 0 resources remaining
  - Full stack deployed: 2 GKE Autopilot clusters, 3 GCP projects, 2 folders, Cloud DNS zones, Cloud Run services, Workflows, PubSub topics, Secret Manager secrets, IAM bindings, VPC networks, fleet registrations

* **Comparison**:
  - **Alternative 2 (VCS-driven)** requires the workspace to have repo access and know which branch to clone. For PR-based testing, this means creating workspaces that track feature branches — complex lifecycle management. CLI-driven mode is simpler: upload the config, run it, done.
  - **Alternative 3 (Tekton)** is being replaced by Prow for CI, making it a dead-end investment. It also duplicates the provider configuration and IAM setup already managed by TFC.
  - **Alternative 4 (Atlantis)** is planned for decommission as TFC becomes the standard. Using it for e2e creates a dependency on infrastructure being removed.
  - **Alternative 5 (long-lived environment)** cannot validate create-from-scratch flows, masks state drift issues, and doesn't support parallel testing.
  - **Alternative 6 (shell scripts in Prow)** is viable but loses the benefits of TFC remote execution: consistent execution environment, run history in the TFC UI, OPA policy enforcement, and WIF credential management. With GCS state, parallel runs require careful state prefix management. With TFC state, this converges toward Alternative 1 but without the render script's path rewriting.
  - **Alternative 7 (static e2e folder)** reduces the moving parts (no render script) but creates a second copy of module call configuration that must be kept in sync with production templates. Config drift between the static e2e folder and production configs is the primary risk — changes to module interfaces require updates in two places.

## Consequences

### Positive

* Complete isolation — each test run creates and destroys its own GCP projects, folders, clusters, and networking with no shared mutable state
* Parallel execution — multiple PRs can run e2e tests simultaneously since each workspace has independent state
* Same platform as production — e2e tests exercise the same TFC remote execution, WIF auth, and OPA policy enforcement used in production
* Create-from-scratch validation — every run proves the modules can provision a full environment from nothing, catching bootstrap ordering issues that update-in-place testing misses
* Auto-destroy safety net — TFC `auto-destroy-at` API provides a backstop against abandoned test infrastructure

### Negative

* Each test run takes ~15-20 minutes for apply and ~10 minutes for destroy
* Render script adds a build step and must be kept in sync with module changes (new modules, path changes)
* Empty TFC workspaces accumulate after destroy and need periodic cleanup
* Remaining dependencies on integration infrastructure (DNS zones, state bucket) are being decoupled as follow-up work
* `user_project_override` requires careful provider aliasing (`google.bootstrap` for folder creation, `google.project_creation` for folder IAM and project creation) — a pattern that must be maintained across module changes

## Cross-Cutting Concerns

### Security:

* TFC SAs authenticate via Workload Identity Federation — no static keys. Tokens are short-lived (~1h) and scoped to the TFC run phase (plan or apply)
* OPA policies enforce `block_folder_iam` and `block_owner_grants` on all e2e workspaces — same enforcement as production
* The e2e template passes `folder_iam_bindings = {}` to comply with `block_folder_iam` policy rather than requesting an exception
* TFC SA permissions are bootstrapped via project-creator SA impersonation — the TFC SA never holds persistent elevated permissions. The impersonation grant is scoped to all workspaces in the `gcp-hcp-ci` TFC project (via `apply_to_all_workspaces`), which is acceptable since workspaces in this project are either ephemeral e2e runs or the permanent WIF access configuration workspace (`gcp-hcp-ci-tfc-access`). OPA policies (`block_folder_iam`, `block_owner_grants`) provide guardrails against privilege escalation within those workspaces

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
* Prow integration will pin e2e runs to a specific git commit, ensuring Terraform and ArgoCD deploy the same version of the codebase
* Boskos provides concurrency control (quota slots) to prevent exceeding GCP project or API quota limits during parallel runs

---

## Implementation Details

### Template Architecture

The e2e template lives at `terraform/config/platform-ci/@e2e/` and consists of:

- `cloud.tf` — TFC backend configuration with `@WORKSPACE_NAME@` placeholder
- `main.tf` — Module calls for region, management-cluster, and customer-project with e2e-specific settings

The render script (`scripts/e2e-render.sh`) produces a self-contained directory:

```text
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

```text
render → init (workspace auto-created) → apply → [validate] → destroy
```

The pipeline runs `terraform destroy` as the final step to tear down all infrastructure. A 24-hour `auto-destroy-at` timer is set on the workspace before apply as a safety net — if the pipeline's destroy fails or a human keeps the environment online for diagnosis, the timer triggers a secondary destroy to prevent resource leaks. Workspaces persist after destroy with run history for debugging. Empty workspace cleanup is handled by a separate process.


### Parallel Run Safety Audit

All Terraform resource names in the region, management-cluster, and customer-project modules are parameterized via `infra_id`, which is unique per e2e run. No hardcoded resource names were found that would conflict between simultaneous runs:

- **GCP projects**: `{abbreviation}-{type}-{region_code}-{infra_id}` — unique per run
- **Folders**: Named after project ID — unique per run
- **VPC networks**: `{env}-{type}-{region_code}-vpc` — scoped to the per-run project, no cross-project conflicts
- **DNS zones**: `hc-{region}-{infra_id}-{N}` — unique per run
- **GKE clusters**: `{project_id}-gke` — unique per run (project ID contains infra_id)
- **Service accounts**: Created within per-run projects — no cross-project conflicts
- **IAM bindings**: Scoped to per-run projects — no conflicts
- **Cloud Run services, Workflows, PubSub topics**: All within per-run projects

The `random_id` resource in customer-project generates a unique 4-character suffix per workspace state, providing additional isolation. Boskos will provide concurrency control (quota slots) in the Prow integration to prevent exceeding GCP quota limits.
### Related PRs

| PR | Repository | Description |
|---|---|---|
| [#1050](https://github.com/openshift-online/gcp-hcp-infra/pull/1050) | gcp-hcp-infra | E2E template + render script |
| [#1113](https://github.com/openshift-online/gcp-hcp-infra/pull/1113) | gcp-hcp-infra | `enable_atlantis` variable |
| [#1101](https://github.com/openshift-online/gcp-hcp-infra/pull/1101) | gcp-hcp-infra | Folder IAM grants for TFC SA |
| [#1088](https://github.com/openshift-online/gcp-hcp-infra/pull/1088) | gcp-hcp-infra | Billing APIs on platform-ci |
| [#1075](https://github.com/openshift-online/gcp-hcp-infra/pull/1075) | gcp-hcp-infra | TFC bootstrap in region/MC modules |
