# kube-applier-gcp: Graduation to Dedicated Repository

**Scope**: GCP-HCP

**Date**: 2026-08-05

## Decision

Graduate the kube-applier-gcp experiment (currently at `gcp-hcp/experiments/kube-applier-gcp/`) to a dedicated repository at `github.com/openshift-online/kube-applier-gcp`, following the graduation process defined in the [Repository Organization Policy](repository-organization-policy.md).

## Context

The kube-applier-gcp is a per-management-cluster controller that runs on GKE and brokers between Google Cloud Firestore and the local Kubernetes API server. It reads `ApplyDesire`, `DeleteDesire`, and `ReadDesire` documents from Firestore and reconciles them against the cluster. The experiment has grown into a well-structured Go project with its own dependency management, controller framework, informers, integration tests, and container build tooling.

- **Problem Statement**: The experiment has outgrown its home in `gcp-hcp/experiments/`. With ~7,600 lines of Go code, its own `go.mod` dependency management, and a fundamentally different CI/CD pipeline (Go builds, Firestore emulator integration tests, container image publishing), it needs a dedicated repository to support independent releases, proper CI/CD, and clear ownership.
- **Assumptions**: The GCP HCP team has the Go expertise to maintain the controller. The PoC demonstrates readiness for continued development.

## Alternatives Considered

1. **Dedicated repository under `openshift-online`**: A standalone `openshift-online/kube-applier-gcp` repository with its own CI/CD, OWNERS file, and branch protections. Provides clear organizational ownership, appropriate pipeline configuration, and satisfies all four graduation criteria.

2. **Co-locate in `gcp-hcp`**: Keep the controller source inside the project hub repository. Avoids a new repository, but mixes Go source with documentation, imposes documentation-oriented CI/CD on a controller project, and contradicts the repository placement guide which directs graduated tooling/services to dedicated repositories.

3. **Co-locate in `openshift-online/gecko`**: Place kube-applier-gcp alongside the Platform API server in the gecko repository. Reduces the number of repositories but conflates two components that operate at different tiers of the GCP HCP architecture and serve fundamentally different roles:
   - **Different system tiers**: gecko runs on the regional cluster (once per region); kube-applier-gcp runs on every management cluster (N instances per region). Coupling them in a single repository binds components that operate at completely different levels of the system topology.
   - **API layer vs. execution layer**: gecko defines and manages the platform API surface; kube-applier-gcp executes desired state against the cluster. Co-locating them would conflate two distinct architectural layers that the Firestore transport boundary is specifically designed to separate.
   - **Independent release lifecycles**: a critical bug in the MC agent would force a gecko release and vice versa, coupling the blast radius of two independently operated components.
   - **Organizational flexibility**: the Firestore transport layer provides a clean interface boundary between the two components. That boundary means a sub-division of the GCP HCP team (e.g., one focused on cluster lifecycle or agent infrastructure) could own kube-applier-gcp independently. Co-locating in gecko would artificially constrain that flexibility by coupling OWNERS, CI/CD, and release responsibilities.

## Decision Rationale

* **Justification**: A dedicated repository satisfies all four graduation criteria and aligns with the repository organization policy's placement guide for "Graduated tooling/services." The controller has a distinct release lifecycle (versioned container images) and a fundamentally different CI/CD pipeline (Go builds, Firestore emulator integration tests, container image publishing) than documentation linting.

* **Comparison**: Co-locating in `gcp-hcp` (alternative 2) would impose documentation-oriented CI/CD on a controller project and make independent releases impractical. Co-locating in gecko (alternative 3) would couple two components that operate at different system tiers, conflate a clean architectural boundary, and constrain future ownership flexibility within the team.

## Graduation Criteria Assessment

This work meets all four graduation criteria defined in the [Repository Organization Policy](repository-organization-policy.md):

| Criterion | Assessment |
|---|---|
| **Independent release lifecycle** | The controller produces versioned container images released independently from documentation or infrastructure changes. |
| **Distinct CI/CD pipeline** | Go builds, Firestore emulator integration tests, and container image publishing are fundamentally different from documentation linting. |
| **Expected longevity > 6 months** | kube-applier-gcp is the long-term Firestore↔Kubernetes bridge for the GCP HCP platform, required for every management cluster. |
| **Clear single owner** | The GCP HCP team is the identified owner and will be listed in the OWNERS file. |

Supporting signals satisfied:

- Codebase exceeds 500 lines (~7,600 lines of Go)
- Own dependency management (`go.mod`)
- Dockerfile and Makefile for container builds already present

## Consequences

### Positive

* Organizational ownership, shared access controls, and branch protections from day one
* CI/CD pipeline tailored to Go builds, Firestore integration tests, and container image publishing
* Independent release lifecycle enables faster iteration without coupling to documentation changes
* Dedicated repository provides clear separation of concerns for the controller

### Negative

* Adds a new repository to the team's portfolio with ongoing maintenance overhead (OWNERS, branch protections, CI/CD)
* Developers must navigate multiple repositories when working across the controller (kube-applier-gcp) and infrastructure (gcp-hcp-infra)
