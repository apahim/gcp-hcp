# Platform Secret Management: Layered Strategy with Workload Identity First

***Scope***: GCP-HCP

**Date**: 2026-07-31

## Decision

We adopt a layered secret management strategy that prioritizes eliminating secrets through Workload Identity, uses Bitwarden as the authoritative root of trust for mandatory secrets, syncs them to Google Cloud Secret Manager per project, and delivers them to GKE clusters via External Secrets Operator with namespaced SecretStores and granular permissions.

## Context

The GCP HCP platform requires a consistent, secure approach to managing secrets across multiple environments and GKE clusters. While Workload Identity eliminates most credential needs, some secrets remain unavoidable — GitHub app keys, external API tokens, SSH keys, and certificates for services that do not support federated identity.

- **Problem Statement**: No unified strategy exists for storing, distributing, and rotating mandatory secrets across the platform. Current practices are ad-hoc, with secrets scattered across Google Secret Manager projects and no single authoritative source. This decision covers platform-level secrets only — secrets used to operate the GCP HCP infrastructure itself. Customer-facing secrets (e.g., OIDC signing keys, customer-provided credentials) belong to a separate security domain and will be addressed in a dedicated decision document.
- **Constraints**: Must comply with Red Hat security policies; must support team-wide shared access with auditing; must integrate with GKE workloads without granting overly broad permissions; must support 2FA token management for shared service accounts.
- **Assumptions**: Workload Identity covers the majority of authentication needs (see [workload-identity-implementation](workload-identity-implementation.md)). The remaining mandatory secrets are relatively few and change infrequently. Red Hat IT-managed Bitwarden provides enterprise-grade security and access controls sufficient for root-of-trust storage.

## Alternatives Considered

1. **Bitwarden + Google Cloud Secret Manager + External Secrets Operator (chosen)**: Bitwarden as root of trust, Secret Manager as cloud-side store, ESO for K8s delivery. Layered approach where each component handles one concern.
2. **HashiCorp Vault (ACCE Vault)**: Enterprise-grade secret management already available at Red Hat. Full-featured but introduces significant operational overhead — requires dedicated infrastructure, expertise, and maintenance for a relatively small number of secrets.
3. **Google Cloud Secret Manager only**: Simpler architecture with secrets stored directly in Secret Manager. Lacks a team-accessible authoritative source, makes 2FA management difficult, and ties the root of trust to GCP project access.
4. **Sealed Secrets in Git**: Encrypt secrets into Git using Bitnami Sealed Secrets. Version-controlled but creates rotation friction, requires re-encryption for key rotation, and couples secret lifecycle to Git workflows.

## Decision Rationale

* **Justification**: The layered approach separates concerns cleanly. Bitwarden provides human-accessible shared storage with 2FA support, managed by Red Hat IT. Google Cloud Secret Manager provides GCP-native integration with IAM, audit logging, and rotation APIs. External Secrets Operator handles the last-mile delivery into Kubernetes with least-privilege access patterns.
* **Evidence**: Workload Identity already eliminates most static credentials (see [workload-identity-implementation](workload-identity-implementation.md)). The remaining mandatory secrets are few enough that Vault's operational overhead is not justified. Bitwarden is already provisioned and managed by Red Hat IT for the team.
* **Comparison**: Vault adds operational complexity disproportionate to our secret volume. Secret Manager alone lacks a team-accessible root of trust and 2FA support. Sealed Secrets create rotation friction and couple secrets to Git workflows. The chosen approach provides the right level of capability at each layer without over-engineering.

## Consequences

### Positive

* Clear separation of concerns: human access (Bitwarden), cloud storage (Secret Manager), K8s delivery (ESO)
* Team-wide shared access to secrets via Rover-managed group, with audit trail
* 2FA token management through Bitwarden for shared service accounts — Bitwarden stores TOTP seeds and generates time-based one-time passwords, so the team can share 2FA-protected service accounts (e.g., GitHub bot accounts) without passing codes through side channels
* Granular IAM permissions per secret per namespace, following least privilege
* Alignment with existing Workload Identity strategy — secrets are the exception, not the rule
* No additional infrastructure to operate (all components are managed services or lightweight operators)

### Negative

* Sync process between Bitwarden and Google Cloud Secret Manager is not yet automated — manual for now, automation TBD
* Three-layer architecture requires understanding of all components for troubleshooting
* Bitwarden dependency on Red Hat IT for access provisioning and collection management
* No single pane of glass for secret inventory across all layers

## Cross-Cutting Concerns

### Security:

* Bitwarden access governed by Rover group `gcp-hcp-eng` (Note `@BW@`), Bitwarden collection "Gcp Hcp Eng"
* Bitwarden provides encrypted storage, access auditing, and 2FA token management
* Google Cloud Secret Manager provides encryption at rest, IAM-based access control, and Cloud Audit Logs
* ESO SecretStores use namespaced service accounts with per-secret-key IAM bindings — no cluster-wide access
* ClusterSecretStores explicitly avoided to prevent cross-namespace secret leakage
* Secret rotation tracked per secret type in the platform secret inventory (maintained as a table in the team's internal documentation), recording rotation frequency, last rotation date, and responsible owner per secret; Workload Identity expansion reduces the tracked set over time

### Reliability:

* **Scalability**: Bitwarden and Secret Manager scale independently; ESO handles reconciliation per namespace
* **Observability**: Cloud Audit Logs for Secret Manager access; ESO metrics and events for sync status; Bitwarden access logs via Red Hat IT
* **Resiliency**: Secret Manager provides regional replication; ESO reconciles on failure; in emergency scenarios (GCP outage, Secret Manager unavailable), team members with Bitwarden desktop/mobile app retain cached read access to secrets for manual intervention — this is a break-glass fallback, not a standard operating procedure

### Cost:

* Bitwarden: managed by Red Hat IT, no direct cost to team
* Google Cloud Secret Manager: per-secret and per-access-operation pricing, negligible at our scale
* External Secrets Operator: open-source, runs as a lightweight controller on existing GKE clusters

### Operability:

* New secrets follow a documented flow: create in Bitwarden, add to Secret Manager in target project, configure ESO SecretStore and ExternalSecret manifests
* ESO SecretStore per namespace with dedicated service account reduces blast radius of misconfigurations
* Runbook needed for secret rotation procedures and emergency access scenarios
* Team onboarding requires Rover group membership for Bitwarden access

---

## Template Validation Checklist

### Structure Completeness
- [x] Title is descriptive and action-oriented
- [x] Scope is GCP-HCP
- [x] Date is present and in ISO format (YYYY-MM-DD)
- [x] All core sections are present: Decision, Context, Alternatives Considered, Decision Rationale, Consequences
- [x] Both positive and negative consequences are listed

### Content Quality
- [x] Decision statement is clear and unambiguous
- [x] Problem statement articulates the "why"
- [x] Constraints and assumptions are explicitly documented
- [x] Rationale includes justification, evidence, and comparison
- [x] Consequences are specific and actionable
- [x] Trade-offs are honestly assessed

### Cross-Cutting Concerns
- [x] Each included concern has concrete details (not just placeholders)
- [x] Irrelevant sections have been removed
- [x] Security implications are considered where applicable
- [x] Cost impact is evaluated where applicable

### Best Practices
- [x] Document is written in clear, accessible language
- [x] Technical terms are used appropriately
- [x] Document provides sufficient detail for future reference
- [x] All placeholder text has been replaced
- [x] Links to related documentation are included where relevant
