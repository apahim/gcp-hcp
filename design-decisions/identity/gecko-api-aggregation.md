# Gecko API Aggregation: Delegate AuthN/AuthZ to GKE kube-apiserver

***Scope***: GCP-HCP

**Date**: 2026-07-28

## Decision

We will replace gecko's private API HTTP layer with a Kubernetes aggregated API server (`GenericAPIServer` from `k8s.io/apiserver`). Internal consumers (controllers, SRE, monitoring) access gecko through the GKE kube-apiserver, which handles authentication, authorization, and audit logging. The external/public API remains unchanged as a standalone HTTP server.

## Context

The [Platform API decision](../governance/platform-api.md) established gecko as the single source of truth for the GCP HCP API definition. The initial implementation includes a custom authentication and authorization stack for the private (internal) API surface.

- **Problem Statement**: Gecko reimplements a significant portion of Kubernetes API server functionality: SA token authentication against a custom Secret store, a full RBAC engine (Role, ClusterRole, RoleBinding, ClusterRoleBinding stored in gecko's datastore), API discovery, watch protocol, and content negotiation. This custom auth stack carries operational burden: token provisioning, rotation, revocation, security patching, and audit trail — all for internal traffic that already runs on a GKE cluster with a fully functional kube-apiserver.
- **Constraints**: Must preserve the public API surface (customer-facing, chi.Router-based). Must reuse the existing `ResourceStore` interface and storage backends (PostgreSQL, Spanner). Must support local development without a kube-apiserver (`--disable-auth` flag). Must not introduce a hard dependency on kube-apiserver availability for the public API path.
- **Assumptions**: Internal consumers exclusively use client-go or controller-runtime to interact with gecko. GKE regional control plane SLA (99.95%) is acceptable for the internal traffic path. cert-manager is available on target clusters for TLS certificate lifecycle.

## Alternatives Considered

1. **API Aggregation via `GenericAPIServer`**: Register gecko as an aggregated API server with the GKE kube-apiserver via an `APIService` resource. Kube-apiserver proxies internal traffic, handling authn/authz/audit. Gecko implements `rest.Storage` adapters bridging `ResourceStore` to the `GenericAPIServer` registry.
2. **Keep custom auth stack, harden it**: Invest in production-hardening the existing SA token authenticator and RBAC engine — automated token rotation, audit logging, security reviews, on-call procedures.
3. **Webhook-based delegation**: Keep the chi.Router HTTP server but add TokenReview/SubjectAccessReview webhook calls to the kube-apiserver for authn/authz. Lighter integration than full `GenericAPIServer` adoption.

## Decision Rationale

* **Justification**: API aggregation eliminates the custom auth stack entirely rather than hardening it. The kube-apiserver is a battle-tested component that the team already operates and monitors. Controllers and SRE tooling interact with gecko using standard `kubectl` and client-go — no custom client libraries or token provisioning workflows. Structured audit logging, admission webhooks, impersonation, and API priority & fairness come for free.
* **Evidence**: The adapter layer (Phase 1) proved that `ResourceStore` maps cleanly to `rest.Storage` interfaces with minimal translation. The prototype was deployed on minikube with cert-manager TLS, verified that anonymous requests receive 403, and demonstrated RBAC enforcement for a controller service account. The implementation adds ~260 lines of real (non-generated) code. Standalone chi.Router mode for the private API was removed — `GenericAPIServer` with `--disable-auth` covers local development without added complexity.
* **Comparison**: Hardening the custom auth stack (Alternative 2) would require significant investment to reach the security and operational maturity that kube-apiserver already provides, and would remain a bespoke system the team must maintain indefinitely. Webhook delegation (Alternative 3) gets authn/authz benefits but misses content negotiation (protobuf/CBOR), structured audit events, OpenAPI aggregation, admission webhooks, and health endpoints — features that `GenericAPIServer` provides natively.

## Consequences

### Positive

* Eliminates custom authn/authz code and its operational burden (token lifecycle, RBAC policy management, security patching)
* Controllers and SRE use standard `kubectl` and client-go — no custom SDK or token provisioning
* Structured kube audit events (who, what, when, which resource) replace basic request logging
* Content negotiation adds protobuf (~60% smaller) and CBOR support alongside JSON
* Built-in `/healthz`, `/livez`, `/readyz` with pluggable health checks
* Admission webhooks (validating/mutating) available without additional plumbing
* Gecko API appears in cluster's unified OpenAPI schema and `kubectl api-resources`
* Projected SA tokens (auto-rotated, short-lived, audience-bound) replace manually provisioned secrets

### Negative

* Internal traffic depends on GKE kube-apiserver availability — if the control plane is down, controllers cannot reach gecko and their reconciliations are delayed; they naturally retry via their reconcile loops to catch up once the kube-apiserver comes back up (public API unaffected)
* Additional hop (kube-apiserver proxy) adds latency to internal API calls
* TLS required for the aggregated server — adds cert-manager dependency and certificate lifecycle management
* `rest.Storage` adapter layer adds a translation layer between `ResourceStore` and `GenericAPIServer`

## Cross-Cutting Concerns

### Reliability:

* **Scalability**: No change — gecko remains stateless and horizontally scalable. The kube-apiserver proxy layer scales with the GKE control plane.
* **Observability**: Standard apiserver Prometheus metrics (request latency, inflight requests, error rates) replace custom request logging. Structured audit events integrate with Cloud Audit Logs.
* **Resiliency**: GKE regional SLA is 99.95%. If the kube-apiserver is unavailable, gecko controller reconciliations are delayed — they naturally retry via their reconcile loops to catch up once the kube-apiserver comes back up. The public API path is completely independent.

### Security:

* Authentication delegated to GKE — supports Google identity, projected SA tokens, OIDC. No custom token store to secure.
* Authorization via native Kubernetes RBAC — policies auditable via standard `kubectl get clusterrolebindings`.
* TLS enforced between kube-apiserver and gecko via cert-manager certificates.
* `system:auth-delegator` ClusterRoleBinding and `extension-apiserver-authentication-reader` RoleBinding are the only cluster-level RBAC grants gecko requires.
* `--disable-auth` binds the private API to localhost only, so it cannot be reached off-host even if set by mistake in a deployed environment. Startup and deployment tests cover this binding behavior.

### Performance:

* Additional kube-apiserver proxy hop adds ~1-2ms latency per request (in-cluster communication).
* Protobuf content negotiation reduces wire size by ~60% for client-go consumers, partially offsetting the proxy overhead.

### Operability:

* Deployment requires: `APIService` resource, cert-manager `Certificate`, auth-delegator RBAC bindings, TLS-enabled Deployment.
* Minikube setup script automates local development: cert-manager install, image build, kustomize deploy.
* Private API always uses `GenericAPIServer` — no standalone mode to maintain. Local development uses `--disable-auth` to skip token/SAR delegation.
* RBAC for controllers is standard Kubernetes — `ClusterRole` + `ClusterRoleBinding` per service account.
