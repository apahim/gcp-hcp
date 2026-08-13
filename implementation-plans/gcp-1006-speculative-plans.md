# TFC Speculative Plans on PRs

## Overview

Add a Prow presubmit that runs `terraform plan -refresh=false` on PRs, restoring the plan visibility Atlantis provided. This runs against the existing integration TFC workspaces on `main` and has no dependency on the progressive delivery workspace migration tracked separately under GCP-985.

| | |
|---|---|
| **Epic** | [GCP-532](https://redhat.atlassian.net/browse/GCP-532) -- Terraform Cloud Evaluation & Plan |
| **Story** | [GCP-1006](https://redhat.atlassian.net/browse/GCP-1006) -- Prow presubmit for speculative Terraform plans on PRs |
| **Related** | [GCP-985](https://redhat.atlassian.net/browse/GCP-985) -- Progressive delivery (tracked separately; no dependency between the two) |

### Target Flow

```text
Developer -> PR to main -----> Prow: speculative terraform plan (refresh=false)
                |                    posts check status + link to Prow logs
                v
             merge to main
```

### Workspace-to-Branch Mapping (Integration)

| TFC Workspace | Working Directory |
|---|---|
| `gcp-hcp-global-integration` | `terraform/config/global/integration/main/us-central1` |
| `gcp-hcp-region-int-main-us-central1` | `terraform/config/region/integration/main/us-central1` |
| `gcp-hcp-mc-int-main-us-central1-yjiv` | `terraform/config/management-cluster/integration/main/us-central1-yjiv` |

---

## Phase 1: Add Prow Presubmit for Speculative Plans

**Summary**: Create a Prow presubmit job that triggers speculative `terraform plan` runs on PR commits using CLI remote execution against VCS-connected TFC workspaces.

**Repos**: `openshift/release` (ci-operator config), `gcp-hcp-infra` (plan script)
**Depends on**: None -- works against existing integration workspaces on `main`

### Execution Mode: CLI Remote Execution

TFC supports CLI-driven speculative plans on VCS-connected workspaces. When `terraform plan` is run from the CLI against a workspace with a `cloud` backend block, TFC creates a configuration version from the local files and executes a speculative (plan-only) run remotely. This is the simplest approach -- no configuration version tarball upload or TFC Runs API orchestration required.

Key characteristics:
- Local terraform files are uploaded to TFC automatically
- TFC executes the plan remotely using the workspace's WIF credentials
- The plan is speculative (read-only) -- it cannot be promoted to apply
- `-refresh=false` is passed through to the remote execution
- The plan output streams back to the CLI stdout

**Tasks**:

### Plan Script (`gcp-hcp-infra`)

- [ ] Create `hack/tfc-speculative-plan.sh`:
  1. Read TFC API token from `/etc/terraform-cloud/token`
  2. Determine affected integration workspaces from the PR's changed files:
     - `terraform/config/global/integration/` -> `gcp-hcp-global-integration`
     - `terraform/config/region/integration/` -> `gcp-hcp-region-int-main-us-central1`
     - `terraform/config/management-cluster/integration/` -> `gcp-hcp-mc-int-main-us-central1-yjiv`
     - `terraform/modules/global/` -> `gcp-hcp-global-integration`
     - `terraform/modules/region/` -> all region integration workspaces
     - `terraform/modules/management-cluster/` -> all MC integration workspaces
     - `terraform/modules/<any other subdirectory>` -> all integration workspaces (catch-all -- shared modules like `commons/`, `service/`, etc. could affect any workspace)
     - `terraform/metadata/`, `terraform/workflows/`, `terraform/dashboards/` -> all integration workspaces
     - Changes only to non-integration paths (e.g., `terraform/config/global/stage/`) -> skip with a message "No integration workspaces affected"
  3. For each affected workspace:
     ```bash
     cd <working_directory>
     terraform init -input=false
     terraform plan -refresh=false -input=false -detailed-exitcode || rc=$?
     ```
     **Implementation note**: The script must capture the exit code explicitly (e.g., `|| rc=$?` or `set +e` around the plan command) rather than letting `set -e` terminate the loop on exit code 2. Continue processing all remaining workspaces before aggregating.
  4. Map exit codes per workspace: 0 = no changes, 1 = error, 2 = changes detected (success). Aggregate: return 1 if any workspace errored, otherwise return 0 (both "no changes" and "changes detected" are successful outcomes for a presubmit)

  **Validation required**: Confirm that `-detailed-exitcode` propagates correctly through CLI remote execution (cloud backend). The Terraform docs do not explicitly address whether exit code 2 returns from a remote speculative plan to the local CLI. Test this against a real TFC workspace before relying on it -- if it doesn't propagate, fall back to parsing the plan output or using the TFC Runs API to check for changes.
- [ ] Add a `Makefile` target `terraform-plan-speculative` that wraps the script
- [ ] Test locally with a sample PR diff

### Presubmit Result Contract

- **Reporter**: Prow ci-operator posts the result as a GitHub check run named `ci/prow/terraform-plan` on the PR's head SHA against `main`
- **Success (exit 0)**: Plan shows no changes -- check run is green
- **Changes detected (exit 2)**: Plan shows infrastructure diff -- check run is green (changes are expected on PRs). Plan output is available in the Prow job logs.
- **Error (exit 1)**: Plan failed (syntax error, init failure, TFC auth failure) -- check run is red
- **No workspace affected**: If the PR only changes non-integration terraform paths (stage, production) or non-terraform files that passed `run_if_changed`, the script exits 0 with a message "No integration workspaces affected by this PR"
- **PR and SHA association**: Prow automatically associates the check run with the PR's head commit SHA on `main`. No custom GitHub API calls needed.

### CI Operator Config (`openshift/release`)

- [ ] Add a new test step `terraform-plan` in `ci-operator/config/openshift-online/gcp-hcp-infra/openshift-online-gcp-hcp-infra-main.yaml`:
  ```yaml
  - as: terraform-plan
    steps:
      test:
      - as: plan
        commands: |
          git config --global url."https://github.com/".insteadOf "git@github.com:"
          git config --global credential.helper '!f() {
            if [ "$1" = "get" ]; then
              host="" proto=""
              while IFS= read -r line; do
                [ -z "$line" ] && break
                case "$line" in
                  host=*) host="${line#host=}" ;;
                  protocol=*) proto="${line#protocol=}" ;;
                esac
              done
              if [ "$proto" = "https" ] && [ "$host" = "github.com" ]; then
                echo "username=x-access-token"
                echo "password=$(cat /etc/github-private/oauth)"
              else
                exit 1
              fi
            fi
          }; f'
          umask 077
          cat > "$HOME/.terraformrc" <<TFRC
          credentials "app.terraform.io" {
            token = "$(cat /etc/terraform-cloud/token)"
          }
          TFRC
          make terraform-plan-speculative
        credentials:
        - mount_path: /etc/github-private
          name: github-credentials-openshift-ci-robot-private-git-cloner
          namespace: ci
        - mount_path: /etc/terraform-cloud
          name: tfcloud-ci-secret
          namespace: ci
        from: src
        resources:
          requests:
            cpu: "8"
            memory: 4Gi
  ```
- [ ] Set `run_if_changed` to trigger only on integration and shared terraform paths:
  ```yaml
  run_if_changed: '^terraform/(config/(global|region|management-cluster)/integration/|modules/|metadata/|workflows/|dashboards/)'
  ```
- [ ] **Do not bypass Prow's trust gating for this job.** The job executes with `tfcloud-ci-secret`, which has access to workspace variables and state during the remote plan -- this is not a low-risk job to auto-run for untrusted authors. Confirm the presubmit uses Prow's default trust behavior: auto-runs for trusted authors (org members / previously-merged contributors) and requires an org member to comment `/ok-to-test` for first-time or external contributors before it runs on their PR. Do not set config (e.g. `skip_report` combined with permissive trust settings) that would let an untrusted fork PR trigger the job without that gate.

**Acceptance Criteria**:
- [ ] PR touching `terraform/config/global/integration/` triggers a speculative plan against `gcp-hcp-global-integration`
- [ ] PR touching `terraform/modules/region/` triggers plans for all region integration workspaces
- [ ] PR touching only `terraform/config/global/stage/` does not trigger the job (filtered by `run_if_changed`)
- [ ] Plan output is visible in the Prow job logs, accessible from the `ci/prow/terraform-plan` check run on the PR
- [ ] Plan does not perform state refresh (`-refresh=false`)
- [ ] Plan does not block or interfere with in-flight promotion applies (speculative plans are read-only)
- [ ] Job exits 0 with a message when no integration workspace is affected
- [ ] A PR from a first-time/external contributor does not trigger the speculative plan until an org member (OWNERS) comments `/ok-to-test`

**Design Notes**:
- **Workspace discovery**: The script uses a static mapping from changed file paths to workspace names. The mapping is scoped to **integration workspaces only** since those are the only workspaces currently on TFC. Any `terraform/modules/` subdirectory not explicitly mapped to a specific workspace (e.g., `commons/`, `service/`, `pagerduty/`) falls through to the catch-all: plan against all integration workspaces. This is safe since speculative plans are read-only, and conservative since shared modules could affect any workspace. When stage/production workspaces are added to TFC, the mapping must be expanded to include those workspaces and their corresponding path scopes. Dynamic discovery via the TFC API (list workspaces, match by `working_directory` and `trigger_prefixes`) is a future enhancement to eliminate static-mapping drift entirely -- see Deferred section.
- **Fork PR trust boundary**: Speculative plans execute remotely with access to workspace variables and state -- `Plan runs` permission is security-equivalent to `Write`, not a confidentiality boundary. Rely on Prow's standard org-member/`/ok-to-test` gating rather than any custom allow-list logic in the script itself.
- **Which workspaces**: Only integration workspaces -- stage/production plans require separate credentials and are a future enhancement.
- **Credential helper scoping**: The Git credential helper is scoped to `github.com` only. It rejects credential requests for other hosts to prevent a malicious Terraform module source from exfiltrating the GitHub token.
- **Credential pattern lineage**: The `.terraformrc` + `tfcloud-ci-secret` pattern is already in production for the `terraform-validate` and `terraform-test` Prow jobs ([openshift/release@90ada45](https://github.com/openshift/release/commit/90ada459a3402a26bdda228c1ea001239c63da0d)). Both jobs run `terraform init` (which requires TFC authentication for the cloud backend and private module resolution) using this same `.terraformrc` approach. The speculative-plan job reuses the identical credential setup. The only difference is the credential helper: the existing jobs return the GitHub token unconditionally, while our plan adds `protocol=https` and `host=github.com` checks before returning it.
- **Why CLI over TFC Runs API**: CLI remote execution is simpler (no tarball upload, no polling for run completion, no plan output parsing). TFC handles configuration version creation and execution automatically. The API approach (`POST /api/v2/runs` with `plan-only: true`, `refresh: false`) is documented as a fallback if CLI limitations are hit.

## Phase 2: Update Branch Protection for Speculative Plans

**Summary**: Add `ci/prow/terraform-plan` to `main` branch status checks.

**Repo**: `openshift/release`
**File**: `core-services/prow/02_config/openshift-online/gcp-hcp-infra/_prowconfig.yaml`
**Depends on**: Phase 1

**Tasks**:
- [ ] Remove remaining Atlantis status checks from `main` branch required status checks (PR #83051 removes `atlantis-int/plan` and `atlantis-int/apply`; stage checks should also be removed when stage moves to TFC)
- [ ] Do **not** add `ci/prow/terraform-plan` as an unconditional required context -- the job uses `run_if_changed` and only runs when terraform paths are modified. Prow handles the "skip" case automatically by not reporting the context for unmatched PRs. Adding it as required would block non-terraform PRs.
- [ ] Add `ci/prow/terraform-plan` to Tide's `required-if-present-contexts` for `main` so it is enforced when the job runs but does not block non-terraform PRs

**Acceptance Criteria**:
- [ ] Speculative plan check runs appear on PRs to `main` (when terraform files are changed)
- [ ] PRs that do not modify terraform paths merge without waiting for `ci/prow/terraform-plan`
- [ ] PRs that modify terraform paths require a passing `ci/prow/terraform-plan` before merge

---

## PR Sequence

| # | Phase | PR | Repo | Applied By | Depends On |
|---|-------|-----|------|------------|------------|
| 1a | Speculative plan script | gcp-hcp-infra PR | gcp-hcp-infra | Prow merge | -- |
| 1b | Speculative plan CI config | openshift/release PR | openshift/release | Prow config merge | Phase 1a |
| 2 | Branch protection (main) | openshift/release PR | openshift/release | Prow config merge | Phase 1 |

---

## Key Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Speculative plans show unexpected drift | Confusing PR feedback | Medium | Expected with `refresh: false` -- document that PR plans are previews, not exact state comparisons |
| Prow job cannot reach TFC API | Plan job fails | Low | `build06` has outbound internet -- verify with test curl to `app.terraform.io` |
| `tfcloud-ci-secret` token lacks speculative plan permissions | Plan job auth failure | Low | Token already used for `terraform validate` which requires TFC API access -- verify `Plan runs` permission before rollout |

---

## Deferred

* **Expand scope to stage/production workspaces**: The static workspace mapping currently covers integration workspaces only. When stage and production workspaces are added to TFC, expand the mapping to include those environments, their `run_if_changed` paths, and any additional credentials required (stage/production may use separate TFC tokens or WIF SAs)
* **Plan output as PR comment**: Start with TFC's native GitHub check integration; add custom PR comment formatting later if needed
* **Automatic workspace discovery from metadata**: Use `terraform/metadata/environments.yaml` and `terraform/metadata/infra_ids.yaml` to dynamically generate workspace-to-branch mappings instead of the static path mapping in Phase 1. This would also eliminate the need to manually keep the static mapping in sync with TFC workspace `trigger_prefixes` defined in `hcp-terraform/gcp-hcp-integration/main.tf`
