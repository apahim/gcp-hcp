---
name: onboard-konflux
description: Onboard a new container image to the gcp-hcp-tenant Konflux tenant — creates Application, Component, ImageRepository, IntegrationTestScenario, ReleasePlan, and auto-generated manifests.
argument-hint: "<app-name>"
effort: medium
---

# Onboard App to Konflux (gcp-hcp-tenant)

Register a new container image in the `gcp-hcp-tenant` Konflux tenant on the `kflux-prd-rh02` cluster. Creates all required Kubernetes resources and runs manifest generation.

**Tenant:** `gcp-hcp-tenant`
**Cluster:** `kflux-prd-rh02`
**Repo:** `gitlab.cee.redhat.com:releng/konflux-release-data`

---

## Phase 0: Locate repos and read current patterns

1. Find the konflux-release-data repo. Run:
   ```bash
   find "$HOME/git" -maxdepth 3 -type d -name "konflux-release-data" 2>/dev/null | head -5
   ```
   If multiple results, prefer a path under the user's own username directory.
   If no results, ask the user for the path.

2. Set `TENANT_DIR` = `<repo>/tenants-config/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant`

3. Read the current state:
   - `$TENANT_DIR/applications/kustomization.yaml` — list of existing apps
   - `$TENANT_DIR/gar-release-sa.yaml` — current GAR service account secrets
   - Pick ONE existing app directory (prefer `gcp-hcp-common-tools`) and read all its files to confirm the current templates haven't drifted from what this skill expects.

4. Verify `kustomize` is available: `which kustomize`. If not, tell the user to install it.

---

## Phase 1: Gather information

If `$ARGUMENTS` was provided, use it as the app name. Otherwise, ask the user.

Use `AskUserQuestion` to collect these details (skip questions where the user already provided answers):

### Required
- **App name**: The Konflux application/component name. Convention: prefix with the project name if the binary name is generic (e.g., `gecko-platform-api-server`, not just `platform-api-server`).
- **Source git repo URL**: The GitHub/GitLab repo URL (e.g., `https://github.com/openshift-online/gecko.git`)
- **Git revision**: Branch to track (default: `main`)
- **Containerfile path**: Path to the Containerfile/Dockerfile within the source repo, relative to repo root (e.g., `deploy/platform-api/Containerfile`)
- **Build context**: Directory context for the container build, relative to repo root (default: `.`)

### Release configuration
- **Release target**: GAR only, Quay only, or both?
  - **GAR** pushes to `us-docker.pkg.dev/gcp-hcp-commons/gcp-hcp-images/<app-name>` via tenant pipeline
  - **Quay** pushes to `quay.io/redhat-services-prod/gcp-hcp-tenant/<app-name>` via managed pipeline
- **Image visibility**: Private (default) or public — controls the Quay ImageRepository visibility

### Derived values (do not ask — compute from app name)
- Component name = app name
- ImageRepository name = `imagerepository-for-<app-name>`
- Image pull secret name = `imagerepository-for-<app-name>-image-pull`
- GAR image URL = `us-docker.pkg.dev/gcp-hcp-commons/gcp-hcp-images/<app-name>`
- ReleasePlan name = `<app-name>-releaseplan` (GAR) / `<app-name>-releaseplan-quay` (Quay)
- IntegrationTestScenario name = `<app-name>-enterprise-contract`

---

## Phase 2: Preview and confirm

Present the user with a summary of everything that will be created:

```
App name:           <app-name>
Source repo:        <url>
Containerfile:      <path>
Build context:      <context>
Release target:     GAR / Quay / Both
Image visibility:   private / public

Files to create:
  applications/<app-name>/application.yaml
  applications/<app-name>/components/<app-name>.yaml
  applications/<app-name>/components/image-repository.yaml
  applications/<app-name>/components/kustomization.yaml
  applications/<app-name>/integration-test-enterprise-contract.yaml
  applications/<app-name>/release-plan.yaml          (if GAR)
  applications/<app-name>/release-plan-quay.yaml     (if Quay)
  applications/<app-name>/kustomization.yaml

Files to update:
  applications/kustomization.yaml
  gar-release-sa.yaml                                (if GAR)
```

Wait for user confirmation before proceeding.

---

## Phase 3: Create application directory and files

All paths below are relative to `$TENANT_DIR`.

### 3a. `applications/<app-name>/application.yaml`

```yaml
---
apiVersion: appstudio.redhat.com/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: gcp-hcp-tenant
spec:
  displayName: <app-name>
```

### 3b. `applications/<app-name>/components/<app-name>.yaml`

```yaml
---
apiVersion: appstudio.redhat.com/v1alpha1
kind: Component
metadata:
  name: <app-name>
  namespace: gcp-hcp-tenant
  annotations:
    build.appstudio.openshift.io/request: configure-pac
    build.appstudio.openshift.io/pipeline: '{"name":"docker-build-oci-ta","bundle":"latest"}'
    git-provider: github
    git-provider-url: https://github.com
spec:
  application: <app-name>
  componentName: <app-name>
  source:
    git:
      url: <source-git-url>
      revision: <revision>
      context: <build-context>
      dockerfileUrl: <containerfile-path>
```

**Note:** If the source repo is on GitLab instead of GitHub, change `git-provider` and `git-provider-url` accordingly.

### 3c. `applications/<app-name>/components/image-repository.yaml`

```yaml
---
apiVersion: appstudio.redhat.com/v1alpha1
kind: ImageRepository
metadata:
  annotations:
    image-controller.appstudio.redhat.com/update-component-image: "true"
  name: imagerepository-for-<app-name>
  namespace: gcp-hcp-tenant
  labels:
    appstudio.redhat.com/application: <app-name>
    appstudio.redhat.com/component: <app-name>
spec:
  image:
    name: gcp-hcp-tenant/<app-name>
    visibility: <private|public>
  notifications:
    - config:
        url: https://bombino.api.redhat.com/v1/sbom/quay/push
      event: repo_push
      method: webhook
      title: SBOM-event-to-Bombino
```

### 3d. `applications/<app-name>/components/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: gcp-hcp-tenant
resources:
  - <app-name>.yaml
  - image-repository.yaml
```

### 3e. `applications/<app-name>/integration-test-enterprise-contract.yaml`

```yaml
---
apiVersion: appstudio.redhat.com/v1beta2
kind: IntegrationTestScenario
metadata:
  name: <app-name>-enterprise-contract
  namespace: gcp-hcp-tenant
spec:
  application: <app-name>
  contexts:
    - description: Application testing
      name: application
  params:
    - name: POLICY_CONFIGURATION
      value: rhtap-releng-tenant/app-interface-standard
  resolverRef:
    params:
      - name: url
        value: https://github.com/konflux-ci/build-definitions
      - name: revision
        value: main
      - name: pathInRepo
        value: pipelines/enterprise-contract.yaml
    resolver: git
```

### 3f. `applications/<app-name>/release-plan.yaml` (GAR — if selected)

```yaml
---
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  labels:
    release.appstudio.openshift.io/auto-release: 'true'
    release.appstudio.openshift.io/standing-attribution: 'true'
  name: <app-name>-releaseplan
spec:
  application: <app-name>
  data:
    mapping:
      components:
        - name: <app-name>
          repositories:
            - url: us-docker.pkg.dev/gcp-hcp-commons/gcp-hcp-images/<app-name>
              tags:
                - latest
                - "{{ git_sha }}"
                - "{{ git_short_sha }}"
  tenantPipeline:
    pipelineRef:
      params:
        - name: url
          value: https://github.com/konflux-ci/community-catalog.git
        - name: revision
          value: development
        - name: pathInRepo
          value: pipelines/push-snapshot-to-gar/push-snapshot-to-gar.yaml
      resolver: git
      useEmptyDir: true
    params:
      - name: wifAudience
        value: gcp-hcp-konflux
    serviceAccountName: gar-release-sa
    timeouts:
      pipeline: "1h0m0s"
```

### 3g. `applications/<app-name>/release-plan-quay.yaml` (Quay — if selected)

```yaml
---
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  labels:
    release.appstudio.openshift.io/auto-release: 'true'
    release.appstudio.openshift.io/standing-attribution: 'true'
    release.appstudio.openshift.io/releasePlanAdmission: <app-name>
  name: <app-name>-releaseplan-quay
spec:
  application: <app-name>
  target: rhtap-releng-tenant
```

**Note:** Quay releases also require a ReleasePlanAdmission in `config/kflux-prd-rh02.0fk9.p1/service/ReleasePlanAdmission/gcp-hcp/`. Use the `/create-rpa` command in the konflux-release-data repo for that. Remind the user.

### 3h. `applications/<app-name>/kustomization.yaml`

List all resources created above:

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: gcp-hcp-tenant
resources:
  - application.yaml
  - components
  - integration-test-enterprise-contract.yaml
  - release-plan.yaml            # include if GAR
  - release-plan-quay.yaml       # include if Quay
```

---

## Phase 4: Update existing tenant files

### 4a. `applications/kustomization.yaml`

Add `<app-name>` to the `resources` list. Maintain alphabetical order if the existing list is sorted, otherwise append.

### 4b. `gar-release-sa.yaml` (only if GAR release selected)

Add `imagerepository-for-<app-name>-image-pull` to the ServiceAccount's `secrets` list.

---

## Phase 5: Generate auto-generated manifests

This is **critical** — CI will fail without this step.

Run from the `tenants-config/` directory:

```bash
./build-manifests.sh kustomize
```

This regenerates all files under `tenants-config/auto-generated/`. The new/modified files for our tenant will appear as untracked or modified in git.

Verify the output:

```bash
git status tenants-config/auto-generated/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant/ --short
```

You should see new files matching the resources you created (Application, Component, ImageRepository, ReleasePlan, IntegrationTestScenario) and a modified `gar-release-sa` file (if GAR).

---

## Phase 6: Validate

Run kustomize build and check for errors:

```bash
kustomize build tenants-config/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant/ 2>&1 | grep -i "error\|warning" || echo "Clean build"
```

Verify the new app appears in the rendered output:

```bash
kustomize build tenants-config/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant/ 2>&1 | grep "<app-name>"
```

If there are errors, fix them before proceeding.

---

## Phase 7: Commit and push

1. Create a new branch (if not already on one):
   ```bash
   git checkout -b onboard-<app-name> origin/main
   ```

2. Stage all changes (source files + auto-generated):
   ```bash
   git add tenants-config/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant/applications/<app-name>/
   git add tenants-config/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant/applications/kustomization.yaml
   git add tenants-config/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant/gar-release-sa.yaml  # if GAR
   git add tenants-config/auto-generated/cluster/kflux-prd-rh02/tenants/gcp-hcp-tenant/
   ```

3. Commit with a descriptive message:
   ```
   Onboard <app-name> to gcp-hcp-tenant

   Add Konflux Application, Component, ImageRepository, IntegrationTestScenario,
   and <GAR/Quay/GAR+Quay> ReleasePlan for <app-name>.
   ```

4. Push and provide the MR creation URL.

---

## Phase 8: Summary

Present a summary:

- **Files created**: List all new files
- **Files modified**: List all updated files
- **Branch**: The branch name
- **MR URL**: The GitLab merge request creation URL
- **Next steps**:
  - Create the MR and get it reviewed/merged
  - If Quay release was selected, remind user to also create a ReleasePlanAdmission using `/create-rpa`
  - Once merged, Konflux will auto-configure PipelineAsCode on the source repo — the first build triggers on the next push to the tracked branch
