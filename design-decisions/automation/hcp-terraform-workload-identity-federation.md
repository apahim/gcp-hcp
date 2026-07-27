# HCP Terraform Must Authenticate via infra-platform WIF Module with Dedicated Access Projects

***Scope***: GCP-HCP

**Date**: 2026-07-27

## Decision

HCP Terraform workspaces that manage GCP infrastructure must authenticate via Workload Identity Federation (WIF) using the infra-platform [`terraform-tfe-gcp-dynamic-creds`](https://app.terraform.io/app/hp-platform-engineering/registry/modules/private/hp-platform-engineering/gcp-dynamic-creds/tfe) module. Each environment gets a dedicated GCP project (e.g., `gcp-hcp-tfc-access-{env}`) that hosts the WIF pool, OIDC provider, and plan/apply service accounts. The module's `apply_to_all_workspaces` flag delivers WIF credentials to all workspaces in a TFC project automatically via a project-scoped variable set.

## Context

- **Problem Statement**: HCP Terraform is replacing Atlantis for infrastructure automation ([GCP-536](https://redhat.atlassian.net/browse/GCP-536)). TFC workspaces need to authenticate to GCP APIs to manage resources across multiple GCP projects per environment (global, region, management-cluster) without static service account keys. The infra-platform team published a reusable module ([infra-platform#90](https://github.com/openshift-online/infra-platform/pull/90)) that automates WIF pool, service account, and variable set lifecycle. However, the module creates resources within a single GCP project per call — it cannot natively handle the cross-project IAM pattern that Atlantis uses (one SA in the global project managing resources across global, region, and MC projects).
- **Constraints**:
  - No static GCP service account JSON keys (platform-wide WIF-first policy)
  - The module creates a WIF pool, OIDC provider, plan/apply SAs, and a TFC variable set — all scoped to a single GCP project per call
  - Cross-project IAM grants (SA in project A managing resources in projects B, C, D) are outside the module's scope
  - The module call must not live in the same TFC project as the workspaces it configures — partial apply creates a circular dependency where the variable set delivers credentials pointing at non-existent SAs (see [experiment issue #1](../experiments/terraform-automation-tools/hcp-terraform-wif-playground.md))
  - CI requires a TFC API token (`TFE_TOKEN`) for private registry module sourcing ([release#82376](https://github.com/openshift/release/pull/82376))
  - Atlantis currently manages 34 unique IAM roles across global (19 project-level + 3 folder-level), region (24), and management-cluster (20) modules
- **Assumptions**:
  - The module will continue to be published to the TFC private registry at `app.terraform.io/hp-platform-engineering/gcp-dynamic-creds/tfe`
  - Each environment follows the same access project pattern
  - The `apply_to_all_workspaces` module feature remains stable

## Alternatives Considered

1. **Module with dedicated access projects**: One `gcp-dynamic-creds` module call per environment, targeting a purpose-built GCP project that hosts only WIF resources (pool, provider, SAs). `apply_to_all_workspaces = true` delivers credentials to all workspaces in the TFC project. Cross-project IAM grants on target projects (global, region, MC) are managed separately via `tfc.tf` files in each infrastructure module.

2. **Direct WIF (no module)**: ~30 lines of Terraform per environment — one SA in the environment's global project, one WIF pool, one OIDC provider. Same cross-project IAM pattern as Atlantis. No module dependency, no additional GCP projects.

3. **Module targeting environment global projects directly**: One module call per target project (global, region, MC) within the same environment. Module creates per-project SAs with roles only on that project.

4. **Direct Workload Identity (no SAs)**: Use `TFC_GCP_PRINCIPAL_TYPE = workload_pool` with direct `principal://` IAM bindings. Eliminates service accounts entirely — the WIF federated identity accesses GCP resources directly.

5. **Static service account keys**: Export JSON keys for TFC service accounts and store as sensitive workspace variables.

## Decision Rationale

* **Justification**: Alternative 1 (module + dedicated access projects) resolves the module's single-project constraint while aligning with the infra-platform team's WIF tooling. The dedicated access project gives the module a clean target — one project, one module call, one variable set — while the SAs it creates receive cross-project IAM grants on global, region, and MC projects. This preserves Atlantis's proven cross-project access pattern. `apply_to_all_workspaces` eliminates per-workspace credential configuration, so adding new workspaces requires zero WIF setup.

* **Evidence**:
  - The module was validated end-to-end in the [Phase 2 experiment](../experiments/terraform-automation-tools/hcp-terraform-wif-playground.md): WIF pool, plan/apply SAs, and variable sets created successfully; a validation workspace authenticated with zero explicit WIF variables via `apply_to_all_workspaces` inheritance.
  - Default audience behavior works — no `TFC_GCP_WORKLOAD_IDENTITY_AUDIENCE` variable needed. The module does not set `allowed_audiences` on the OIDC provider, and HCP Terraform defaults to the provider resource name as the audience.
  - Workspace-scoped attribute conditions (`assertion.sub.startsWith(...)`) restrict authentication to the correct TFC project.
  - The circular dependency issue (module-created variable set poisoning the calling workspace) is resolved by isolating the module call from the workspaces it serves — either via a separate TFC project or workspace-level variable overrides (see [Open Items](#open-items)).

* **Comparison**:
  - **Alternative 2 (direct WIF)** is simpler (~30 lines of Terraform) and avoids a module dependency, but provides no alignment with the infra-platform team's tooling. The WIF plumbing is straightforward but error-prone — audience mismatch between `TFC_GCP_WORKLOAD_PROVIDER_AUDIENCE` (legacy) and `TFC_GCP_WORKLOAD_IDENTITY_AUDIENCE` (Dynamic Provider Credentials) caused debugging overhead during Phase 1. The module encapsulates these details.
  - **Alternative 3 (module per target project)** was explored and abandoned. Calling the module once per target project (global, region, MC) creates 6 SAs per environment (plan + apply x 3 projects), produces 3 variable sets for one TFC project with undefined precedence behavior, and requires supplemental cross-project IAM grants outside the module for every target project. The region module creates resources on both the region project AND the global project (`modules/region/global-iam.tf`) — a per-region-project SA cannot do this without additional grants. See [experiment issue #7](../experiments/terraform-automation-tools/hcp-terraform-wif-playground.md) for variable set precedence findings.
  - **Alternative 4 (direct WID)** eliminates SAs entirely but requires `principal://` IAM binding support across every GCP resource type the team manages (GKE, Compute, DNS, Secret Manager, Workflows, Pub/Sub, Eventarc, Cloud Run, Tags, PAM, BigQuery, Artifact Registry). This was never validated, and discovering an unsupported resource type during production rollout would require a mid-migration architecture change.
  - **Alternative 5 (static keys)** is prohibited by platform security policy and introduces key management burden.

## Consequences

### Positive

* Aligns with infra-platform team's WIF module — shared maintenance, upstream improvements, and organizational consistency across teams
* `apply_to_all_workspaces` eliminates per-workspace credential configuration — new workspaces inherit WIF credentials automatically
* Dedicated access project isolates WIF resources (pool, SAs) from infrastructure resources — clean separation of concerns
* Module handles the error-prone WIF plumbing (audience configuration, variable set wiring, attribute conditions) that caused debugging overhead during manual setup
* Workspace-scoped attribute conditions provide fine-grained authentication scoping per TFC project
* Default audience behavior simplifies configuration — no explicit `TFC_GCP_WORKLOAD_IDENTITY_AUDIENCE` or `allowed_audiences` needed

### Negative

* Introduces a new GCP project per environment (`gcp-hcp-tfc-access-{env}`) solely for WIF resources — additional project to create and manage
* Access workspace requires bootstrap with local or static credentials for initial apply — a manual step per environment
* Module dependency on infra-platform releases — upstream changes or breakages affect WIF infrastructure
* Cross-project IAM grants (the bulk of the implementation) are still managed separately outside the module — the module only handles WIF plumbing within the access project
* Plan/apply SA split creates two SA objects per environment regardless of whether roles differ
* IAM propagation delay (~60s) after module apply means dependent workspaces may fail on first run

## Cross-Cutting Concerns

### Security:

* Federated OIDC tokens are short-lived (~1h) and scoped to the specific HCP Terraform run phase (plan or apply)
* Workspace-scoped `attribute_condition` on the WIF provider restricts federation to workspaces within the configured TFC project — tighter than the org-wide condition used in Phase 1
* No secrets stored in HCP Terraform — the token exchange uses the workspace's OIDC identity
* Plan SA can be restricted to read-only roles on target projects, limiting blast radius during speculative plans
* Access project contains only WIF resources — compromising it does not expose infrastructure state or resources

### Operability:

* **Adding a new workspace**: Zero WIF configuration needed — workspace inherits credentials from the project-level variable set via `apply_to_all_workspaces`. Cross-project IAM for the apply SA must already be in place on the target project.
* **Adding a new environment**: Create access GCP project, create access workspace with bootstrap credentials, apply module, then create infrastructure workspaces. Grant cross-project IAM on each target project via `tfc.tf` files.
* **Adding a new region**: `scripts/infra.py` generates config and adds a workspace entry. WIF credentials are inherited — no per-workspace config needed. Cross-project IAM grants for the new region project must be added to `modules/region/tfc.tf`.
* **Debugging authentication failures**: Check the WIF provider attribute condition matches the TFC project name. Verify the module-created SAs have the required cross-project IAM bindings on target projects. Check for IAM propagation delay (~60s) on newly created bindings.
* **Module updates**: Pin to a specific module version in the TFC private registry. Test version upgrades in integration before promoting to stage/production.

### Reliability:

* **Observability**: WIF authentication failures surface as `invalid_grant` or `iam.serviceAccounts.getAccessToken` errors in the TFC run log. GCP Cloud Audit Logs record STS token exchanges and SA impersonation attempts, providing an audit trail. No additional monitoring infrastructure is needed — failures are visible in the TFC UI and GCP logs.
* **Resiliency**: WIF depends on GCP STS and the HCP Terraform OIDC token issuer. Both are managed services with high availability. If TFC is unavailable, no runs execute (same as Atlantis depending on its GKE cluster). If GCP STS is unavailable, all GCP API calls fail regardless of authentication method.

### Performance:

* Not materially impacted. The STS token exchange adds milliseconds to the `terraform init` phase — negligible compared to plan/apply execution time. No difference from Atlantis's GKE Workload Identity token exchange.

### Cost:

* WIF and STS token exchanges are free — no additional GCP charges
* Dedicated access projects have no running resources (no compute, no storage) — project-level costs are negligible
* HCP Terraform workspace costs are governed by the organization's plan, not by the authentication method

## Implementation Reference

### Access Project Architecture

```text
Per Environment:

┌──────────────────────────────────────────────────────────────┐
│ TFC Project: gcp-hcp-{env}                                   │
│                                                               │
│  ┌───────────────────────────────────┐                        │
│  │ Access Workspace                  │                        │
│  │ (bootstrap: local/static creds)   │                        │
│  │                                   │                        │
│  │  gcp-dynamic-creds module call    │                        │
│  │  → target: gcp-hcp-tfc-access-{env} GCP project           │
│  │  → creates: WIF pool, OIDC provider, plan/apply SAs       │
│  │  → creates: variable set (apply_to_all_workspaces=true)   │
│  └───────────────────────────────────┘                        │
│                                                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐    │
│  │ gcp-hcp-global-{env}    │  │ gcp-hcp-region-{env}-*  │    │
│  │ (inherits WIF creds)    │  │ (inherits WIF creds)    │    │
│  └─────────────────────────┘  └─────────────────────────┘    │
│                                                               │
│  ┌─────────────────────────┐                                  │
│  │ gcp-hcp-mc-{env}-*      │                                  │
│  │ (inherits WIF creds)    │                                  │
│  └─────────────────────────┘                                  │
└──────────────────────────────────────────────────────────────┘

GCP Projects:

┌──────────────────────────────┐     ┌──────────────────────┐
│ gcp-hcp-tfc-access-{env}    │     │ gcp-hcp-{env}-global │
│                              │     │                      │
│  • WIF pool                  │     │  • Infrastructure    │
│  • OIDC provider             │     │  • Cross-project IAM │
│  • Plan SA ─────── view ────────▶  │    for plan SA       │
│  • Apply SA ──── write ─────────▶  │    for apply SA      │
│                              │     │                      │
└──────────────────────────────┘     └──────────────────────┘
         │          │                          │
         │          │                 ┌────────┴────────┐
         │          │                 ▼                 ▼
         │          │     ┌──────────────────┐ ┌──────────────────┐
         │          └────▶│ {env}-reg-*      │ │ {env}-mgt-*      │
         └───── view ────▶│  • Cross-project │ │  • Cross-project │
                          │    IAM grants    │ │    IAM grants    │
                          └──────────────────┘ └──────────────────┘
```

### Authentication Flow

```text
HCP Terraform Workspace (e.g., gcp-hcp-global-integration)
    │
    ├─ 1. Workspace inherits WIF variables from project-level variable set
    │      (delivered by gcp-dynamic-creds module via apply_to_all_workspaces)
    │
    ├─ 2. TFC generates OIDC token:
    │      issuer:   https://app.terraform.io
    │      audience: (default — WIF provider resource name, auto-matched)
    │      subject:  organization:hp-platform-engineering:project:gcp-hcp-{env}:
    │                workspace:gcp-hcp-global-{env}:run_phase:apply
    │
    ├─ 3. Token sent to GCP STS
    │      → validated against WIF provider in gcp-hcp-tfc-access-{env}
    │      → attribute_condition checks sub starts with
    │        "...project:gcp-hcp-{env}:..."
    │
    ├─ 4. STS returns federated token
    │      → exchanged for apply SA access token
    │      SA: {prefix}-apply@gcp-hcp-tfc-access-{env}.iam.gserviceaccount.com
    │
    └─ 5. SA token used for GCP API calls on target projects
           (requires cross-project IAM grants on global/region/MC projects)
```

### Cross-Project IAM

The module does not manage cross-project IAM. The plan and apply SAs in the access project need IAM roles on each target project. These are managed via `tfc.tf` files in each infrastructure module, mirroring the existing `atlantis.tf` pattern:

| Module | File | Role Count | Pattern |
|---|---|---|---|
| Global | `modules/global/tfc.tf` | 19 project-level + 3 folder-level | Copy from `atlantis.tf`, change member to access project SA |
| Region | `modules/region/tfc.tf` | 24 cross-project | Copy from `atlantis.tf`, same bootstrap pattern |
| Management-Cluster | `modules/management-cluster/tfc.tf` | 20 cross-project | Copy from `atlantis.tf` |
| Commons | `modules/commons/tfc-iam.tf` | 3 cross-project | `storage.objectViewer` on state bucket, `artifactregistry.admin` on GAR, `iam.serviceAccountTokenCreator` on `project-creator` SA |

Member format: `serviceAccount:{sa_name}@gcp-hcp-tfc-access-{env}.iam.gserviceaccount.com`

Commons grants require SRE manual apply (commons module is not managed by Atlantis).

### CI Workspaces

CI workspaces (`hypershift-ci`, `platform-ci`) target single GCP projects with no cross-project IAM requirements. Two options are viable:

- **Module**: Same `gcp-dynamic-creds` pattern with a CI-scoped access project or one module call per CI project
- **Existing Prow WIF pools**: Extend the existing CI project WIF pools with a TFC OIDC provider

CI handling will be finalized during implementation.

### PagerDuty

The PagerDuty workspace uses a PagerDuty API key — no GCP IAM needed. It does not participate in the WIF topology.

## Open Items

### Resolved

- **Access project creation and bootstrap credentials**: The access project and WIF resources are bootstrapped locally following the same pattern as environment setup ([gcp-hcp-infra SETUP.md](https://github.com/openshift-online/gcp-hcp-infra/blob/main/terraform/config/global/SETUP.md)). An operator runs `terraform apply` locally with their own `gcloud` credentials and a `TFE_TOKEN` to create the access GCP project, WIF pool, SAs, and TFC variable set in a single apply. State is then migrated to TFC (the access workspace takes over ongoing management). No static service account keys are needed. The operator needs folder-admin permissions (to create the project), `iam.workloadIdentityPoolAdmin` + `iam.serviceAccountAdmin` (for WIF resources), and a TFC API token (for variable set creation).
- **Circular dependency**: The local bootstrap sidesteps the circular dependency entirely — the first apply runs locally, not in a TFC workspace, so there is no workspace for `apply_to_all_workspaces` to poison. After state is migrated to TFC, the access workspace inherits its own variable set, but at that point the SAs already exist and the credentials are valid. If a future `terraform apply` on the access workspace partially fails (e.g., SA recreation), the same manual detach-and-reapply fix from the [experiment](../experiments/terraform-automation-tools/hcp-terraform-wif-playground.md) applies.

- **Plan vs. apply role assignment**: The module creates separate plan and apply SAs with distinct roles. The plan SA gets view access on each target project (sufficient for `terraform plan`). The apply SA gets the full write role set (matching Atlantis). Cross-project IAM in each module's `tfc.tf` grants the appropriate roles to each SA separately.

### Future Considerations

- **CI workspace WIF approach**: Whether to use the `gcp-dynamic-creds` module or extend existing Prow WIF pools with a TFC OIDC provider for CI workspaces.
- **RBAC model**: Who can approve applies per TFC project/workspace — not yet evaluated.

## References

- [HCP Terraform WIF Playground Experiment](../experiments/terraform-automation-tools/hcp-terraform-wif-playground.md) — Phase 1 (SA impersonation) and Phase 2 (module) validation results
- [GCP-536](https://redhat.atlassian.net/browse/GCP-536) — Spike: Evaluate HCP Terraform for GCP-HCP Infrastructure
- [gcp-dynamic-creds module](https://app.terraform.io/app/hp-platform-engineering/registry/modules/private/hp-platform-engineering/gcp-dynamic-creds/tfe) — TFC private registry
- [infra-platform#90](https://github.com/openshift-online/infra-platform/pull/90) — Module implementation
- [Miro: TFC Workspace Architecture](https://miro.com/app/board/uXjVH9A0m0w=/?focusWidget=3458764677714315706) — Architecture diagram from Sr Eng call
- [Global Environment Setup Guide](https://github.com/openshift-online/gcp-hcp-infra/blob/main/terraform/config/global/SETUP.md) — Bootstrap pattern for new environments (same local-apply approach used for access projects)
