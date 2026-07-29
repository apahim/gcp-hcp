# Atlantis to HCP Terraform Cutover

## Overview

Migrate infrastructure automation from self-hosted Atlantis to HCP Terraform Cloud (TFC) for the GCP HCP platform. TFC workspaces authenticate to GCP via the infra-platform [`gcp-dynamic-creds`](https://app.terraform.io/app/hp-platform-engineering/registry/modules/private/hp-platform-engineering/gcp-dynamic-creds/tfe) module, which creates a WIF pool, OIDC provider, and plan/apply service accounts in a dedicated access GCP project per environment. `apply_to_all_workspaces = true` delivers WIF credentials to all workspaces in a TFC project automatically.

Integration environment first, then stage. Production does not have Atlantis today, so TFC will be the first automation there.

**Jira**: [GCP-536](https://redhat.atlassian.net/browse/GCP-536)

**Design decision**: [hcp-terraform-workload-identity-federation](../design-decisions/automation/hcp-terraform-workload-identity-federation.md)

**Experiment**: [hcp-terraform-wif-playground](../experiments/terraform-automation-tools/hcp-terraform-wif-playground.md) — validated module E2E, `apply_to_all_workspaces`, default audience behavior

**Bootstrap pattern**: [gcp-hcp-infra SETUP.md](https://github.com/openshift-online/gcp-hcp-infra/blob/main/terraform/config/global/SETUP.md) — same local-apply approach used for environment setup

**Architecture diagram**: [Miro board](https://miro.com/app/board/uXjVH9A0m0w=/?focusWidget=3458764677714315706)

### Authentication Flow

```text
TFC Workspace (e.g., gcp-hcp-global-integration)
    │
    ├─ 1. Inherits WIF variables from project-level variable set
    │      (module-managed, apply_to_all_workspaces=true)
    │
    ├─ 2. TFC generates OIDC token → GCP STS validates against
    │      WIF provider in gcp-hcp-{env_abbrev}-tfc-access
    │      attribute_condition scoped to TFC project
    │
    ├─ 3. STS returns federated token → exchanged for SA access token
    │      Plan phase: plan SA (view access on target projects)
    │      Apply phase: apply SA (write access on target projects)
    │
    └─ 4. SA token used for GCP API calls on target projects
```

---

## Spike Evaluation Summary

Findings from the [GCP-536](https://redhat.atlassian.net/browse/GCP-536) spike mapped to each evaluation area. The design decision and experiment docs cover authentication in depth; this section captures the remaining areas.

### 1. Workflow Fit

TFC uses a VCS-driven workflow: push to a branch triggers a speculative plan, merge to main triggers an apply. This replaces Atlantis's PR-comment-driven model (`atlantis plan`, `atlantis apply`). Key differences:

- **Plan triggers**: Automatic on PR push (no manual comment needed). Plans appear as GitHub check runs, not PR comments.
- **Apply triggers**: Configurable per workspace. Auto-apply on merge to main, or require manual confirmation in the TFC UI. We will start with manual confirmation and evaluate auto-apply after validation (Story 5).
- **Parallel operations**: TFC queues runs per workspace. Multiple PRs touching different workspaces run in parallel. PRs touching the same workspace queue sequentially (same as Atlantis per-project locking).
- **Developer experience**: Plan output is in the TFC UI (linked from the GitHub check), not inline in the PR. This is a visibility tradeoff: richer UI with history and logs, but one click away from the PR.

### 2. Authorization (RBAC)

TFC supports team-based access control at the organization, project, and workspace level. Evaluation is deferred to production rollout (Story 8) where stricter approval gates matter. For integration and stage:

- All team members (`gcp-hcp-eng`) have admin access on TFC projects (configured in infra-platform bootstrap)
- Apply approval is manual in the TFC UI (any team member can confirm)
- Sentinel/OPA policies enforce deletion protection (already configured via bootstrap module)
- Production will require a more restrictive model (e.g., separate plan-only and apply-allowed teams)

### 3. State Management

TFC manages workspace state internally. No migration from GCS is needed for new workspaces since TFC workspaces use the `cloud {}` backend from the start.

For existing infrastructure currently managed by Atlantis (global, region, MC), the workspaces will continue using GCS backends. TFC runs `terraform plan/apply` against the same GCS state buckets Atlantis uses today. No state migration is required for the cutover. The access workspace is the only workspace using TFC-managed state.

### 4. Integration Points

- **GitHub App**: TFC uses the org-level GitHub App configured in infra-platform bootstrap. No per-environment GitHub App needed (unlike Atlantis which requires one per environment).
- **Pre-apply validation**: Atlantis uses `hack/check-pr-labels.sh` to validate PR labels before apply. TFC equivalent is run tasks or Sentinel policies. For the initial migration, this validation will be handled by the existing Prow CI checks which remain unchanged. A TFC-native replacement can be added later if needed.
- **Drift detection**: TFC supports scheduled health assessments that detect configuration drift. Enabled per workspace via `assessments_enabled`. Will be evaluated during Story 5 validation but is not a migration blocker.
- **OPA/Sentinel policies**: Deletion protection policy already enforced via the bootstrap module (`gcp-hcp-deletion-protection` policy set). Additional policies can be added incrementally.

### 5. Operational Considerations

- **Availability**: TFC is a managed SaaS with [published SLA](https://www.hashicorp.com/cloud-operating-model). Removes the operational burden of running Atlantis on GKE (pod restarts, scaling, certificate renewal, Helm upgrades). If TFC is unavailable, no runs execute, but infrastructure is unaffected (same failure mode as Atlantis GKE downtime).
- **Auditability**: TFC provides a full audit log of all runs, state changes, and user actions per workspace. GCP Cloud Audit Logs record all API calls made by the plan/apply SAs. Combined, these provide a stronger audit trail than Atlantis (which relies on GKE pod logs that are subject to retention limits).
- **Cost**: WIF and STS token exchanges are free. TFC workspace costs are governed by the organization plan (managed by infra-platform team). No additional GCP charges beyond the access project (which has no running resources).
- **Fallback**: During the migration (Stories 1-5), Atlantis remains fully operational. TFC runs speculative plans only until validation is complete. Atlantis is not decommissioned until Story 9, after TFC is proven in all environments. If TFC does not work out, Atlantis continues as-is with no rollback needed.

### 6. Migration Strategy

- **Parallel running**: Atlantis and TFC coexist during migration. Atlantis handles all applies until TFC is validated. TFC runs speculative plans for comparison (Story 5). Risk of both systems triggering on the same PR is mitigated by disabling Atlantis autoplan for TFC-managed workspaces before enabling TFC applies.
- **Phased rollout**: Integration first (Stories 1-5), then stage (Story 7), then production (Story 8). Each environment is fully validated before the next begins.
- **Rollback**: At any point before Story 9 (decommission), Atlantis can resume full control by disabling TFC workspace auto-apply and re-enabling Atlantis autoplan. No state migration is involved since infrastructure workspaces use GCS backends throughout.
- **Order**: Lowest risk first. Integration has the fewest projects and the most tolerance for experimentation. Production is last and has no existing Atlantis to cut over from.

---

## Story 1: Bootstrap Access Project (Integration)

### Summary

Create the dedicated access GCP project for integration, containing the WIF pool, OIDC provider, plan/apply SAs, and TFC variable set. This is bootstrapped locally following the same pattern as environment setup.

**Repo**: `gcp-hcp-infra`
**Applied by**: Local `terraform apply` (operator with folder-admin permissions)

### Prerequisites

- Operator has folder-admin permissions on the integration environment folder
- Operator has a TFC API token (`TFE_TOKEN`) for variable set creation
- `gcp-hcp-integration` TFC project exists

### Tasks

- [ ] Create `terraform/config/tfc-access/integration/main.tf`:
  - `google_project` resource for `gcp-hcp-int-tfc-access` in the integration folder
  - `gcp-dynamic-creds` module call sourced from `app.terraform.io/hp-platform-engineering/gcp-dynamic-creds/tfe`
  - Module inputs: `project_id`, `tfc_organization`, `tfc_project_name = "gcp-hcp-integration"`, `apply_to_all_workspaces = true`
  - `plan_roles` — `roles/viewer` (read access on the access project only)
  - `apply_roles` — self-management roles for the access project only: `roles/viewer`, `roles/iam.workloadIdentityPoolAdmin`, `roles/iam.serviceAccountAdmin`, `roles/resourcemanager.projectIamAdmin`, `roles/serviceusage.serviceUsageAdmin`
  - Note: these roles apply to the access project, not target projects. The full Atlantis role set is granted on target projects in Story 2 (cross-project IAM).
  - `cloud {}` backend block (commented out for initial local apply)
- [ ] Create `terraform/config/tfc-access/integration/versions.tf` with provider version constraints
- [ ] Run `terraform init && terraform apply` locally
- [ ] Verify in TFC UI: variable set exists, `apply_to_all_workspaces` is enabled, 4 `TFC_GCP_*` variables present
- [ ] Register access workspace in meta workspace (`hcp-terraform/meta/main.tf` or equivalent) — workspace must exist before state migration
- [ ] Uncomment `cloud {}` backend block
- [ ] Run `terraform init -migrate-state` to migrate state to TFC

### Acceptance Criteria

- [ ] GCP project `gcp-hcp-int-tfc-access` exists in the integration folder
- [ ] WIF pool and OIDC provider exist in the access project
- [ ] Plan and apply SAs exist in the access project
- [ ] TFC variable set `*-gcp-dynamic-creds` attached to `gcp-hcp-integration` project with `apply_to_all_workspaces = true`
- [ ] Variable set contains the expected `TFC_GCP_*` variables (`TFC_GCP_PROVIDER_AUTH`, `TFC_GCP_PLAN_SERVICE_ACCOUNT_EMAIL`, `TFC_GCP_APPLY_SERVICE_ACCOUNT_EMAIL`, `TFC_GCP_WORKLOAD_PROVIDER_NAME`)
- [ ] State managed by TFC (not local)

---

## Story 2: Cross-Project IAM for Plan and Apply SAs (Integration)

### Summary

Grant the plan SA view access and the apply SA write access on each target project (global, region, MC). This mirrors the existing `atlantis.tf` IAM pattern but with two SAs instead of one.

**Repo**: `gcp-hcp-infra`
**Applied by**: Atlantis PR
**Depends on**: Story 1 (SAs must exist)

### Tasks

- [ ] Create `terraform/modules/global/tfc.tf`:
  - Variables: `enable_tfc` (default `false`), `tfc_plan_sa_email`, `tfc_apply_sa_email`
  - Apply SA — 19 project-level roles (same as Atlantis):
    `container.admin`, `gkehub.admin`, `compute.networkAdmin`, `storage.admin`, `compute.instanceAdmin.v1`, `iam.serviceAccountAdmin`, `resourcemanager.projectIamAdmin`, `serviceusage.serviceUsageAdmin`, `compute.securityAdmin`, `dns.admin`, `secretmanager.admin`, `certificatemanager.editor`, `iap.admin`, `artifactregistry.admin`, `cloudbuild.builds.editor`, `logging.admin`, `iam.workloadIdentityPoolAdmin`, `monitoring.metricsScopesAdmin`, `monitoring.editor`
  - Apply SA — 3 folder-level roles (on `folders/{parent_folder_id}`):
    `resourcemanager.projectCreator`, `resourcemanager.folderAdmin`, `logging.configWriter`
  - Plan SA — `roles/viewer` on the global project (minimum for speculative plans)
  - All resources gated by `var.enable_tfc`
- [ ] Create `terraform/modules/region/tfc.tf`:
  - Apply SA — 24 cross-project roles (same as `atlantis.tf`):
    `resourcemanager.projectIamAdmin`, `viewer`, `container.admin`, `compute.networkAdmin`, `storage.admin`, `compute.instanceAdmin.v1`, `iam.serviceAccountAdmin`, `iam.serviceAccountUser`, `serviceusage.serviceUsageAdmin`, `compute.securityAdmin`, `dns.admin`, `gkehub.admin`, `secretmanager.admin`, `workflows.admin`, `run.admin`, `pubsub.admin`, `eventarc.admin`, `resourcemanager.tagAdmin`, `resourcemanager.tagUser`, `privilegedaccessmanager.admin`, `storage.hmacKeyAdmin`, `bigquery.admin`, `artifactregistry.admin`, `monitoring.metricsScopesAdmin`
  - Plan SA — `roles/viewer` on the region project
  - Same bootstrap pattern as `atlantis.tf` (project-creator impersonation for initial `projectIamAdmin`)
- [ ] Create `terraform/modules/management-cluster/tfc.tf`:
  - Apply SA — 20 cross-project roles (region minus: `gkehub.admin`, `storage.hmacKeyAdmin`, `bigquery.admin`, `artifactregistry.admin`)
  - Plan SA — `roles/viewer` on the MC project
- [ ] Enable in integration configs:
  - `terraform/config/global/integration/main/us-central1/main.tf`: `enable_tfc = true`, `tfc_plan_sa_email = "..."`, `tfc_apply_sa_email = "..."`
  - `terraform/config/region/integration/main/us-central1/main.tf`: same
  - `terraform/config/management-cluster/integration/main/us-central1-yjiv/main.tf`: same
- [ ] Open PR, apply via Atlantis

### Acceptance Criteria

- [ ] `terraform plan` shows IAM bindings for both plan and apply SAs on all target projects
- [ ] After apply, plan SA can read resources in global/region/MC projects
- [ ] After apply, apply SA can create/modify resources in global/region/MC projects

### Notes

Split IAM creation from workspace creation — IAM propagation delay (~60s) means workspaces should not be created until IAM has propagated.

---

## Story 3: Commons Cross-Project Grants (Integration)

### Summary

Grant the apply SA access to commons resources: Terraform state bucket, GAR repo, and project-creator SA impersonation.

**Repo**: `gcp-hcp-infra`
**Applied by**: SRE manual apply (commons module is not managed by Atlantis)
**Depends on**: Story 1 (SAs must exist)

### Tasks

- [ ] Create `terraform/modules/commons/tfc-iam.tf` (parallel to `atlantis-iam.tf`):
  - `roles/storage.objectViewer` on commons TF state bucket for plan SA — needed for `terraform_remote_state` reads during speculative plans
  - `roles/storage.objectViewer` on commons TF state bucket for apply SA — same reads during apply phase
  - `roles/artifactregistry.admin` on commons GAR repo for apply SA
  - `roles/iam.serviceAccountTokenCreator` on `project-creator` SA for apply SA
  - Iterate over `var.environment_dns_zones` (excluding dev)
  - Members: `serviceAccount:{plan_sa}@gcp-hcp-{env_abbrev}-tfc-access.iam.gserviceaccount.com` and `serviceAccount:{apply_sa}@gcp-hcp-{env_abbrev}-tfc-access.iam.gserviceaccount.com`
  - Note: `objectViewer` is correct — matches Atlantis. Workspace state lives in TFC (cloud backend), not GCS. The commons bucket is only accessed via `terraform_remote_state` data sources for commons outputs.
- [ ] Coordinate with SRE for manual apply

### Acceptance Criteria

- [ ] Plan SA can read commons Terraform state (for `terraform_remote_state` in speculative plans)
- [ ] Apply SA can read commons Terraform state
- [ ] Apply SA can push to commons GAR
- [ ] Apply SA can impersonate `project-creator` SA

---

## Story 4: TFC Workspace Definitions (Integration)

### Summary

Create TFC workspaces mirroring the Atlantis projects in `atlantis-integration.yaml`. Workspaces inherit WIF credentials from the module-created variable set — no per-workspace WIF configuration needed.

**Repo**: `gcp-hcp-infra`
**Applied by**: TFC meta workspace (push to main triggers it)
**Depends on**: Story 1 (variable set), Story 2 (IAM), Story 3 (commons grants — state bucket access required for `terraform init`)

### Tasks

- [ ] Create `hcp-terraform/gcp-hcp-int/main.tf` with workspace definitions via `workspaces/tfe` module:

  | TFC Workspace | Working Directory | Trigger Prefixes |
  |---|---|---|
  | `gcp-hcp-global-integration` | `terraform/config/global/integration/main/us-central1` | `terraform/metadata/`, `terraform/dashboards/global/` |
  | `gcp-hcp-region-int-main-us-central1` | `terraform/config/region/integration/main/us-central1` | `terraform/workflows/`, `terraform/metadata/` |
  | `gcp-hcp-mc-int-main-us-central1-yjiv` | `terraform/config/management-cluster/integration/main/us-central1-yjiv` | `terraform/workflows/`, `terraform/metadata/` |

- [ ] Create `hcp-terraform/gcp-hcp-int/cloud.tf` pointing at meta workspace
- [ ] No `tfe_variable_set` or `tfe_variable` resources needed — module handles variable sets via `apply_to_all_workspaces`
- [ ] Open PR, merge to main (meta workspace applies)

### Acceptance Criteria

- [ ] Workspaces appear in `gcp-hcp-integration` TFC project
- [ ] Each workspace has WIF variables inherited from the project-level variable set
- [ ] Speculative plans run on PR pushes

---

## Story 5: Validation (Integration)

### Summary

Validate that TFC can manage the same infrastructure as Atlantis with the same outcomes.

**Depends on**: Stories 2, 3, 4

### Tasks

- [ ] **Plan comparison**: Run TFC plan on each workspace, compare with latest Atlantis plan — outputs should be identical (no-op or same diff)
- [ ] **Small change test**: Apply a minor change (e.g., add a resource label) via TFC on one workspace
- [ ] **State consistency**: Verify TFC reads the same GCS remote state as Atlantis
- [ ] **Cross-project operations**: Verify region workspace can create IAM bindings on the global project (via `modules/region/global-iam.tf`) — this is the key operation that validates the cross-project IAM grants
- [ ] **Plan SA isolation**: Verify speculative plans succeed with view-only plan SA (no write operations attempted during plan)

### Acceptance Criteria

- [ ] TFC plan output matches Atlantis for all integration workspaces
- [ ] At least one apply completes successfully via TFC
- [ ] Cross-project IAM operations work from region and MC workspaces

---

## Story 6: Scripts and Tooling

### Summary

Extend automation tooling for TFC workspace generation and update design docs.

**Repo**: `gcp-hcp-infra` (scripts), `gcp-hcp` (docs)

### Tasks

- [ ] Extend `scripts/infra.py` to generate TFC workspace entries:
  - Add workspace entries to `hcp-terraform/gcp-hcp-{env}/main.tf` when creating new regions/MCs
  - Add plan/apply SA cross-project IAM to `tfc.tf` files in region/MC modules
  - Same gate: `environment in ['integration', 'stage', 'production'] and sector != 'e2e'`
- [ ] Update design docs in `gcp-hcp`:
  - `design-decisions/automation/hcp-terraform-workload-identity-federation.md` — mark as implemented
  - Close experiment cleanup checklist items

### Acceptance Criteria

- [ ] `scripts/infra.py new region {env} {sector} {region}` generates TFC workspace entry alongside Atlantis project entry
- [ ] Generated IAM includes both plan and apply SA bindings

---

## Story 7: Repeat for Stage

Same as Stories 1–5 for the stage environment:

- Target access project: `gcp-hcp-stg-tfc-access`
- Target projects: `gcp-hcp-stg-global`, `stg-reg-*`, `stg-mgt-*`
- New workspace definitions: `hcp-terraform/gcp-hcp-stg/`
- TFC project: `gcp-hcp-stage`

---

## Story 8: Production Rollout

### Summary

Deploy TFC to production. Production does not have Atlantis today, so TFC will be the first automation. Same Stories 1–5 pattern, but no Atlantis cutover or parallel-running concern.

**Depends on**: Stories 5 and 7 (integration and stage validated)

- Target access project: `gcp-hcp-prd-tfc-access`
- Target projects: `gcp-hcp-prd-global`, `prd-reg-*`, `prd-mgt-*`
- New workspace definitions: `hcp-terraform/gcp-hcp-prd/`
- TFC project: `gcp-hcp-production`

### Notes

Production introduces additional considerations not present in integration/stage:
- Change approval gates may be stricter (RBAC, policy enforcement)
- First apply is also the first automated apply in production — manual validation of the initial plan is critical
- No Atlantis to compare against — validation relies on manual spot-checks and expected no-op plans

---

## Story 9: Cutover and Decommission Atlantis

### Summary

Disable Atlantis and remove its infrastructure after TFC is validated in integration, stage, and production.

### Tasks

- [ ] Disable Atlantis autoplan (ArgoCD sync disabled or webhook removed) for workspaces that TFC manages
- [ ] Validate TFC handles all PRs for ~1 week with no Atlantis fallback
- [ ] Remove Atlantis ArgoCD application (`argocd/config/global/atlantis/`)
- [ ] Remove Atlantis Helm chart (`helm/charts/atlantis-stack/`)
- [ ] Remove Atlantis SA and IAM bindings (`atlantis.tf` files in global/region/MC/commons modules)
- [ ] Remove `atlantis-{env}.yaml` files
- [ ] Update mintmaker agent (`agent/mintmaker/tools/atlantis.py` → TFC equivalent)
- [ ] Remove `enable_tfc` variable gates — TFC IAM becomes the only IAM

### Acceptance Criteria

- [ ] No Atlantis pods running in any environment
- [ ] All `atlantis.tf` and `atlantis-iam.tf` files removed
- [ ] All infrastructure changes flow through TFC

---

## PR Sequence (Integration)

| # | Story | PR | Applied By | Depends On |
|---|---|---|---|---|
| 1 | Access project bootstrap | Local apply (no PR) | Operator | — |
| 2 | Cross-project IAM | `gcp-hcp-infra` PR | Atlantis | Story 1 |
| 3 | Commons grants | `gcp-hcp-infra` PR | SRE manual | Story 1 |
| 4 | Workspace definitions | `gcp-hcp-infra` PR | TFC meta workspace | Stories 1, 2, 3 |
| 5 | Validation | Manual testing | — | Stories 2, 3, 4 |
| 6 | Scripts & tooling | `gcp-hcp-infra` + `gcp-hcp` PRs | N/A (tooling) | Story 4 |

Stories 2 and 3 can run in parallel (both depend only on Story 1).

## Key Risks and Mitigations

| Risk | Mitigation |
|---|---|
| IAM propagation delay on new SA roles | Split SA creation (Story 1) from workspace creation (Story 4); allow ~60s between apply and first workspace run |
| State file locking during parallel Atlantis + TFC | Only one system should apply at a time during validation; use TFC speculative plans |
| Commons module requires SRE manual apply | Coordinate with SRE; include in phase sequencing |
| Atlantis and TFC both triggering on same PR | Disable Atlantis autoplan for workspaces that TFC manages before enabling TFC |
| Module upstream breakage | Pin to specific module version in TFC private registry; test upgrades in integration first |
| Plan SA needs more than viewer for certain plans | If `terraform plan` fails with viewer-only, add specific read roles to plan SA — or use `use_apply_role_for_plan` ([infra-platform#119](https://github.com/openshift-online/infra-platform/pull/119)) to fall back to unified roles |

## CI Workspaces (Deferred)

`hypershift-ci` targets a single GCP project. `platform-ci` creates region and management-cluster projects, so it has the same cross-project IAM requirements as environment workspaces — the same access project pattern applies. Two options under evaluation:

- `gcp-dynamic-creds` module (same access project pattern)
- Extend existing Prow WIF pools with a TFC OIDC provider

CI workspace migration will be planned separately after environment workspaces are validated.

## PagerDuty

PagerDuty uses a PagerDuty API key — no GCP IAM needed. Migrated to TFC as a standalone workspace in `gcp-hcp-tooling` with no WIF configuration.
