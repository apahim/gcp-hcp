# Cedar-Based Authorization for Gecko Public API

## Overview

This document specifies the Cedar-based authorization system for Gecko's public (customer-facing) API. It defines the permission model, API resource types, Cedar entity hierarchy, policy generation strategy, authorization flow, and caching/hot-reload mechanisms.

This implements the architecture decided in [cedar-public-api-authorization](../design-decisions/identity/cedar-public-api-authorization.md). Work is tracked under [GCP-339](https://redhat.atlassian.net/browse/GCP-339) (Epic).

**Repository**: [gecko](https://github.com/openshift-online/gecko)

---

## Design Decisions

| Decision | Choice |
|---|---|
| Authorization engine | Cedar (`cedar-go`), in-process evaluation |
| User identity source | `X-Endpoint-API-UserInfo` header (base64 JWT claims injected by ESPv2 sidecar) |
| Principal key | Email claim (e.g., `User::"alice@example.com"`) |
| API types | `PlatformRole` (cluster-scoped), `Role` (namespaced), `RoleBinding` (namespaced) |
| System role seeding | Helm chart templates for PlatformRole CRDs, deployed via ArgoCD |
| PlatformRole public API | None — no `platformrole.*` permissions exist; CRUD is via private API only |
| Cedar conditions | On `RoleBinding.spec.condition` (not on Role), enabling per-user ABAC |
| Policy generation | Per-binding with explicit namespace pins |
| Grant constraints | Relaxed: no self-grant prevention, no infrastructure-role restriction |
| Validation | Referenced role existence, Cedar condition syntax, `Namespace::` traversal rejection |
| Entity model | 3 types: User, NamespaceRole, Namespace |
| Entity cache | 1000-entry LRU, per-user invalidation on binding writes, full invalidation on role changes |
| Cross-namespace list | Namespace-filter via `ListOptions.Namespaces` in storage layer (database-level filtering) + per-item `ItemFilter` for Cedar conditions |
| Multi-resource conditions | `context.resourcePlural` guard required when condition targets a specific resource type |
| Auth disable flag | `--disable-auth` covers both private and public APIs for local development |
| Cross-region consistency | Via separate [cross-region resource replication](gcp-cross-region-resource-replication.md) mechanism |

---

## Granular Permissions

Every API operation maps to a single granular permission. Permissions follow the pattern `{resource}.{verb}` and each maps to a PascalCase Cedar action. All permissions are namespace-scoped.

| Permission | Cedar Action |
|---|---|
| `cluster.create` | `CreateCluster` |
| `cluster.list` | `ListClusters` |
| `cluster.get` | `GetCluster` |
| `cluster.update` | `UpdateCluster` |
| `cluster.delete` | `DeleteCluster` |
| `nodepool.create` | `CreateNodepool` |
| `nodepool.list` | `ListNodepools` |
| `nodepool.get` | `GetNodepool` |
| `nodepool.update` | `UpdateNodepool` |
| `nodepool.delete` | `DeleteNodepool` |
| `rolebinding.create` | `CreateRoleBinding` |
| `rolebinding.list` | `ListRoleBindings` |
| `rolebinding.get` | `GetRoleBinding` |
| `rolebinding.update` | `UpdateRoleBinding` |
| `rolebinding.delete` | `DeleteRoleBinding` |
| `role.create` | `CreateRole` |
| `role.list` | `ListRoles` |
| `role.get` | `GetRole` |
| `role.update` | `UpdateRole` |
| `role.delete` | `DeleteRole` |

Unknown permission names are rejected at validation time (Role and PlatformRole creation). Unknown HTTP methods or URL patterns are rejected before Cedar evaluation (fail-closed with 403).

---

## System Roles

System roles are `PlatformRole` resources (cluster-scoped, `+kubebuilder:resource:scope=Cluster`). They are deployed as Helm chart templates via ArgoCD, identical across regions. PlatformRoles have no public API endpoint — there are no `platformrole.*` permissions, so they are naturally immutable from the customer's perspective.

| Role | Permissions |
|---|---|
| `cluster-admin` | `cluster.create`, `cluster.list`, `cluster.get`, `cluster.update`, `cluster.delete`, `nodepool.create`, `nodepool.list`, `nodepool.get`, `nodepool.update`, `nodepool.delete` |
| `cluster-viewer` | `cluster.list`, `cluster.get`, `nodepool.list`, `nodepool.get` |
| `service-admin` | `rolebinding.create`, `rolebinding.list`, `rolebinding.get`, `rolebinding.update`, `rolebinding.delete`, `role.create`, `role.list`, `role.get`, `role.update`, `role.delete` |

**Separation of concerns**: Access management (`service-admin`) is fully separated from infrastructure management (`cluster-admin`, `cluster-viewer`). No single role conflates both. A user who needs both access-management and infrastructure permissions requires multiple bindings.

### Helm Template Format

Each system PlatformRole is a Helm template:

```yaml
{{- if .Values.platformRoles.enabled }}
apiVersion: gcp.managed.openshift.io/v1
kind: PlatformRole
metadata:
  name: cluster-viewer
spec:
  permissions:
    - cluster.list
    - cluster.get
    - nodepool.list
    - nodepool.get
{{- end }}
```

PlatformRoles are gated by `.Values.platformRoles.enabled` so they can be disabled in test environments.

---

## API Resources

### PlatformRole

Cluster-scoped. Defines a set of permissions. Managed exclusively via the private API (kube-apiserver + kube RBAC) and Helm.

```go
type PlatformRole struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec PlatformRoleSpec `json:"spec"`
}

type PlatformRoleSpec struct {
    Permissions []string `json:"permissions"`
}
```

### Role

Namespace-scoped. User-defined roles created via the public API by principals with `role.*` permissions. Defines a set of permissions drawn from the valid permission set.

```go
type Role struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec RoleSpec `json:"spec"`
}

type RoleSpec struct {
    Permissions []string `json:"permissions"`
}
```

### RoleBinding

Namespace-scoped. Binds a user email to a PlatformRole or a Role within a namespace. Optionally carries a Cedar condition for ABAC. All RoleBindings of configured resource types are globally replicated across regions (see [cross-region replication plan](gcp-cross-region-resource-replication.md)).

```go
type RoleBinding struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec RoleBindingSpec `json:"spec"`
}

type RoleBindingSpec struct {
    Subject   string  `json:"subject"`             // User email
    RoleRef   RoleRef `json:"roleRef"`              // References PlatformRole or Role
    Condition string  `json:"condition,omitempty"`   // Cedar expression body (optional)
}

type RoleRef struct {
    Kind     string `json:"kind"`     // "PlatformRole" or "Role"
    Name     string `json:"name"`
    APIGroup string `json:"apiGroup"` // "gcp.managed.openshift.io"
}
```

**RoleRef.Kind** distinguishes the binding target:
- `"PlatformRole"` — references a cluster-scoped PlatformRole (e.g., `cluster-admin`, `service-admin`)
- `"Role"` — references a namespace-scoped Role (user-defined)

---

## Authentication Middleware

### Production Mode

1. Read the `X-Endpoint-API-UserInfo` header (set by ESPv2 sidecar after JWT validation)
2. Base64-decode (try raw URL encoding first, fall back to padded)
3. Parse JSON and extract the `email` claim
4. Normalize email: NFC unicode normalization, lowercase the domain part (after `@`), preserve local part case
5. Store in request context via `authn.WithUser(ctx, email)`
6. Missing, malformed, or empty-email cases return 401 Unauthorized

### Development Mode

1. Read the `X-Dev-User` header directly (no JWT validation)
2. Missing header returns 401
3. Normalize and store in context

The `--disable-auth` flag activates development mode.

---

## Cedar Entity Model

The entity model uses three types. The Cedar schema below is documentation only — it is not enforced at runtime. The Go code constructs correctly-typed entities.

```cedarschema
entity User;
entity Namespace;
entity NamespaceRole in Namespace;
```

### Entity Construction

For a given user, the `EntityGetter` builds:

1. **`User::"alice@example.com"`** — parents: list of all `NamespaceRole` entities from the user's bindings
2. **`NamespaceRole::"project-a/cluster-admin/marketplace-service-admin"`** — parent: `Namespace::"project-a"`
3. **`Namespace::"project-a"`** — leaf entity, no parents

**Entity key format**: `NamespaceRole` entities use a three-part key: `{namespace}/{roleName}/{bindingName}`. This is critical for per-binding policy isolation.

### Per-Binding Isolation Rationale

Entity keys include the binding name (not just namespace/role) to prevent unconditional bindings from leaking into conditioned policies.

**Example**: User A has an unconditional binding to `cluster-viewer` in `project-a`. User B has a conditioned binding to `cluster-viewer` in `project-a` (only US regions). If the entity key were `project-a/cluster-viewer` (without binding name), User A's entity would match User B's conditioned policy — both would resolve to the same `NamespaceRole` entity. With the binding-name suffix, each binding produces a distinct `NamespaceRole` entity, and policies are generated per-binding with distinct entity references.

---

## Cedar Policy Generation

Each RoleBinding in the database produces a Cedar `permit` policy. The policy set is built at startup from all PlatformRoles, Roles, and RoleBindings, and rebuilt on changes (hot-reload).

### PlatformRole Bindings

For a RoleBinding referencing a PlatformRole:

```cedar
// platformrole:cluster-viewer:binding:project-a/viewer-binding
permit (
    principal,
    action in [Action::"ListClusters", Action::"GetCluster",
               Action::"ListNodepools", Action::"GetNodepool"],
    resource
)
when {
    principal in NamespaceRole::"project-a/cluster-viewer/viewer-binding" &&
    resource in Namespace::"project-a"
};
```

### Namespace-Scoped Role Bindings

For a RoleBinding referencing a user-defined Role:

```cedar
// role:us-east-viewer:binding:project-a/region-binding
permit (
    principal,
    action in [Action::"ListClusters", Action::"GetCluster"],
    resource
)
when {
    principal in NamespaceRole::"project-a/us-east-viewer/region-binding" &&
    resource in Namespace::"project-a" &&
    (!(context has resourceName) || context.spec.platform.gcp.region == "us-east1")
};
```

### Condition Wrapping

When a RoleBinding has a `condition` field, the policy generator wraps it:

```
<base_condition> && (!(context has resourceName) || <user_condition>)
```

The `!(context has resourceName)` guard ensures that **list operations** (which have no `resourceName` in the Cedar context) are authorized at the namespace level. Per-item filtering then applies the condition to each individual result via the `ItemFilter` mechanism.

### Multi-Resource Condition Guard

When a RoleBinding's condition targets a specific resource type (e.g., cluster attributes) but the referenced role grants permissions on multiple resource types (e.g., `cluster.get` and `nodepool.get`), the condition **must** use `context.resourcePlural` to scope itself:

```
context.resourcePlural != "clusters" || context.spec.platform.gcp.region == "us-east1"
```

This ensures the condition is only evaluated when accessing clusters, and does not spuriously deny nodepool access where the `spec.platform.gcp.region` path may not exist.

### Policy IDs

Policy IDs are deterministic:
- PlatformRole bindings: `platformrole:<roleName>:binding:<namespace>/<bindingName>`
- Role bindings: `role:<roleName>:binding:<namespace>/<bindingName>`

### Forbid Policies

The policy set supports platform-level `forbid` policies for actions that must be universally denied regardless of role bindings. A matching `forbid` overrides any matching `permit` — Cedar evaluates all applicable policies and a single matching `forbid` produces Deny.

### Hot-Reload

`platform-api-server` watches PlatformRole, Role, and RoleBinding resources via the `ResourceStore` watch mechanism. On create, update, or delete of any watched resource:

1. Load all PlatformRoles, Roles, and RoleBindings from the stores
2. Generate the new `cedar.PolicySet` via `GeneratePolicies()`
3. Swap atomically via `atomic.Pointer[cedar.PolicySet]` — the old set continues serving requests until the swap completes
4. Invalidate the entity cache:
   - PlatformRole or Role change → invalidate all cached entries
   - RoleBinding change → invalidate only the affected user's cache entry (and the previous subject's entry if the subject changed during an update)

---

## Authorization Middleware

### URL Parsing

The middleware parses Kubernetes API-style URL paths to extract:
- **Plural resource name** (e.g., `clusters`, `nodepools`, `rolebindings`, `roles`)
- **Namespace** (if present)
- **Resource name** (if present)

Path canonicalization is applied to prevent traversal attacks (e.g., `/../` sequences).

### Action Derivation Map

| HTTP Method | URL Pattern | Cedar Action | Strategy |
|---|---|---|---|
| GET | `/namespaces/{ns}/clusters` | `ListClusters` | Namespace authz |
| POST | `/namespaces/{ns}/clusters` | `CreateCluster` | Namespace authz |
| GET | `/namespaces/{ns}/clusters/{name}` | `GetCluster` | Single-resource authz |
| PUT/PATCH | `/namespaces/{ns}/clusters/{name}` | `UpdateCluster` | Single-resource authz |
| DELETE | `/namespaces/{ns}/clusters/{name}` | `DeleteCluster` | Single-resource authz |
| GET | `/clusters` | `ListClusters` | **Cross-namespace** |
| GET | `/namespaces/{ns}/nodepools` | `ListNodepools` | Namespace authz |
| POST | `/namespaces/{ns}/nodepools` | `CreateNodepool` | Namespace authz |
| GET | `/namespaces/{ns}/nodepools/{name}` | `GetNodepool` | Single-resource authz |
| PUT/PATCH | `/namespaces/{ns}/nodepools/{name}` | `UpdateNodepool` | Single-resource authz |
| DELETE | `/namespaces/{ns}/nodepools/{name}` | `DeleteNodepool` | Single-resource authz |
| GET | `/nodepools` | `ListNodepools` | **Cross-namespace** |
| GET | `/namespaces/{ns}/rolebindings` | `ListRoleBindings` | Namespace authz |
| POST | `/namespaces/{ns}/rolebindings` | `CreateRoleBinding` | Namespace authz |
| GET | `/namespaces/{ns}/rolebindings/{name}` | `GetRoleBinding` | Single-resource authz |
| PUT/PATCH | `/namespaces/{ns}/rolebindings/{name}` | `UpdateRoleBinding` | Single-resource authz |
| DELETE | `/namespaces/{ns}/rolebindings/{name}` | `DeleteRoleBinding` | Single-resource authz |
| GET | `/rolebindings` | `ListRoleBindings` | **Cross-namespace** |
| GET | `/namespaces/{ns}/roles` | `ListRoles` | Namespace authz |
| POST | `/namespaces/{ns}/roles` | `CreateRole` | Namespace authz |
| GET | `/namespaces/{ns}/roles/{name}` | `GetRole` | Single-resource authz |
| PUT/PATCH | `/namespaces/{ns}/roles/{name}` | `UpdateRole` | Single-resource authz |
| DELETE | `/namespaces/{ns}/roles/{name}` | `DeleteRole` | Single-resource authz |
| GET | `/roles` | `ListRoles` | **Cross-namespace** |

Health probes (`/healthz`, `/readyz`) are registered outside the middleware chain and bypass both authentication and authorization.

### Authorization Strategies

**Single-resource operations** (GET/POST/PUT/DELETE with resource name):

1. Extract user from `authn.UserFromContext()`
2. Derive Cedar action from HTTP method + URL structure
3. Build Cedar context from request body (for POST/PUT/PATCH): `resourceName`, `resourcePlural`, `method`, and `spec` (full resource spec converted to Cedar Record)
4. Read and restore the request body (the body must remain available for the handler)
5. Call `authorizer.AuthorizeWithContext(ctx, user, action, namespace, cedarCtx)`
6. Deny → 403 Forbidden

**Namespaced list** (GET with namespace, no resource name):

1. Authorize at the namespace level without Cedar context (no `resourceName`)
2. Inject an `ItemFilter` function into the context for per-item condition evaluation
3. The handler calls the filter for each list item, excluding items where the filter returns false

**Cross-namespace list** (GET without namespace, e.g., `GET /clusters`):

1. Detect cross-namespace list (namespaced resource, no namespace param, GET method)
2. Pre-compute the authorized namespace set from the user's RoleBindings whose referenced role grants the requested action's permission
3. Inject the namespace set into the request context
4. The handler reads authorized namespaces from context and sets `ListOptions.Namespaces`
5. The storage layer filters at the database level
6. Inject an `ItemFilter` for per-item Cedar condition evaluation
7. A cross-namespace list **never returns 403** — an empty authorized namespace set produces an empty result list

### Cedar Context

The Cedar context (`cedar.Record`) passed to evaluation contains:

| Key | Type | Description |
|---|---|---|
| `resourceName` | String | Name of the resource being accessed (absent for list operations) |
| `resourcePlural` | String | Plural resource type (e.g., `"clusters"`, `"nodepools"`) |
| `method` | String | HTTP method |
| `spec` | Record | Full `spec` object from the request body (for write operations) or from the stored object (for per-item filtering), recursively converted to Cedar types |

The `spec` record is built by recursively converting Go `map[string]interface{}` values to Cedar types: maps → `Record`, slices → `Set`, strings → `String`, booleans → `Boolean`, numbers → `Long`.

---

## Entity Cache

The entity cache is a thread-safe LRU cache storing `cedar.EntityMap` objects keyed by user email.

| Property | Value |
|---|---|
| Max size | 1000 entries |
| Cache key | User email (normalized) |
| Eviction | LRU (least recently used evicted on overflow) |
| Thread safety | `sync.Mutex` + `container/list` |

### Population

On the first authorization check for a user, the `EntityGetter`:

1. Queries RoleBindings where `spec.subject` matches the user email (via `FieldFilters` in `ListOptions`)
2. For each binding, resolves the referenced PlatformRole or Role
3. Builds the entity graph (User → NamespaceRole → Namespace)
4. Caches the entity map

### Invalidation

| Trigger | Scope |
|---|---|
| RoleBinding created/deleted | Affected user's cache entry evicted |
| RoleBinding updated (same subject) | Affected user's cache entry evicted |
| RoleBinding updated (subject changed) | Both old and new subjects' cache entries evicted |
| PlatformRole or Role created/updated/deleted | **All** cache entries evicted (policy set also rebuilt) |

Subject change detection uses the `PreviousObject` field on `ResourceEvent` (populated for `MODIFIED` events by the storage layer).

---

## Validation

### RoleBinding Validation

1. **`subject`** is required (non-empty)
2. **`roleRef.name`** is required; `roleRef.kind` must be `"PlatformRole"` or `"Role"`; `roleRef.apiGroup` must be `"gcp.managed.openshift.io"`
3. **Referenced role existence**: the validator verifies that the referenced role exists in the database (PlatformRole or Role, depending on `roleRef.kind`). Uses injected `ValidatorDeps` functions to avoid circular imports between the `v1` types package and the authz/storage packages.
4. **Cedar condition validation** (if `condition` is non-empty): wraps the condition in a Cedar policy template and parses via `cedar.Policy.UnmarshalCedar()`. Syntactically invalid conditions are rejected. References to `Namespace::` entities within conditions are rejected to prevent namespace traversal.

No self-grant prevention — a service-admin can grant any role (including `cluster-admin`) to themselves. This is intentional: it simplifies the bootstrap flow and trusts service-admins to manage their own namespace.

### Role Validation

1. At least one permission is required
2. All permissions must be in the valid permission set (20 permissions)

No infrastructure permission restriction — user-defined Roles may include any valid permission.

### PlatformRole Validation

Same as Role validation. PlatformRoles are only created via the private API (Helm/kubectl), so the validator enforces the same schema constraints.

### Ownership Transfer

When a Role or RoleBinding with the `replication.gcp.managed.openshift.io/replicated-from` annotation is updated via the public API, the annotation is stripped automatically (along with `refresh-deadline`), transferring ownership to the editing region. The edit then propagates globally via the replication Publisher. See the [cross-region replication plan](gcp-cross-region-resource-replication.md#ownership-transfer) for details.

### ValidatorDeps

A global singleton injected at server startup to break the circular import between `api/private/v1` (types package) and `pkg/authz` (storage/stores). Contains function pointers:

```go
type ValidatorDeps struct {
    RoleExists         func(ctx context.Context, namespace, name string) (bool, error)
    PlatformRoleExists func(ctx context.Context, name string) (bool, error)
}
```

---

## User-Defined Roles

Service-admins can create namespace-scoped Roles via the public API. These roles define custom permission sets and can be bound to users with optional Cedar conditions on the RoleBinding.

### Creating a User-Defined Role

```yaml
apiVersion: gcp.managed.openshift.io/v1
kind: Role
metadata:
  name: us-east-cluster-viewer
  namespace: project-a
spec:
  permissions:
    - cluster.list
    - cluster.get
```

### Binding with a Condition

The `condition` field on the RoleBinding is a **Cedar expression body** (not a full `when` clause). The policy generator wraps it inside a `when { ... }` block alongside the mandatory namespace constraints.

```yaml
apiVersion: gcp.managed.openshift.io/v1
kind: RoleBinding
metadata:
  name: alice-us-east-viewer
  namespace: project-a
spec:
  subject: alice@example.com
  roleRef:
    kind: Role
    name: us-east-cluster-viewer
    apiGroup: gcp.managed.openshift.io
  condition: 'context.resourcePlural != "clusters" || context.spec.platform.gcp.region == "us-east1"'
```

The `context.resourcePlural` guard ensures the region condition is only evaluated when accessing clusters — not when accessing nodepools (where the `spec.platform.gcp.region` path may not exist).

### Condition Validation

Conditions are validated at RoleBinding creation/update time:
1. The condition is wrapped in a Cedar policy and parsed via `cedar.Policy.UnmarshalCedar()`
2. Syntactically invalid Cedar expressions are rejected
3. References to `Namespace::` entities are rejected (prevents namespace traversal)

---

## Server Wiring

### Startup Sequence

1. **Storage factory selection**: Spanner (if `SPANNER_DATABASE` set) > PostgreSQL (if `DB_HOST` set) > in-memory (default)
2. **Shared memoized factory**: A `sharedFactory` wraps the storage factory with a `sync.Mutex`-protected map, ensuring the same store instance is returned for the same resource type. This is critical — both the Cedar authorizer and the API handlers must share the same stores.
3. **Authz store construction**: Creates `ResourceStore` instances for PlatformRole, Role, and RoleBinding using the shared factory
4. **Authorizer creation**: `authz.NewAuthorizer(ctx, authzStores)` loads the initial PolicySet from all roles/bindings in the stores at startup
5. **Validator deps injection**: Wires the role/binding existence check functions using the authz stores
6. **Middleware chain**: `authnMW → authzMW` is set as the public API middleware. The private API (kube-apiserver aggregated API) does **not** run Cedar middleware.
7. **Hot-reload start**: `authorizer.StartWatching(ctx)` is called synchronously before serving. If it fails, the server crashes (fail-fast rather than serving with stale policies).

### Server Ports

| Port | API | Auth |
|---|---|---|
| 8080 | Private (aggregated K8s API) | kube-apiserver authn/authz delegation |
| 8081 | Public (customer-facing) | ESPv2 + Cedar authn/authz middleware |

---

## File Structure

```text
platform-api/
  api/private/v1/
    platformrole_types.go              # PlatformRole (cluster-scoped)
    platformrole_validator.go
    role_types.go                      # Role (namespaced)
    role_validator.go
    rolebinding_types.go               # RoleBinding (namespaced)
    rolebinding_validator.go
    rolebinding_validator_test.go
    validation.go                      # ValidatorDeps + valid permissions
  api/public/v1/
    zz_generated.platformrole_types.go # (generated by orlop-gen)
    zz_generated.role_types.go
    zz_generated.rolebinding_types.go
    zz_generated.conversion.go
    zz_generated.schemas.go
  pkg/
    authn/
      context.go                       # WithUser / UserFromContext
      middleware.go                    # ESPv2 header extraction + dev mode
      middleware_test.go
    authz/
      authorizer.go                    # Cedar PolicySet + Authorize() + AuthorizedNamespaces()
      authorizer_test.go
      cache.go                         # LRU entity cache (1000 entries)
      cache_test.go
      entities.go                      # EntityGetter + AuthzStores
      entities_test.go
      middleware.go                    # HTTP authz middleware (URL parsing, action derivation)
      middleware_test.go
      permissions.go                   # Permission-to-Action mapping (20 permissions)
      policygen.go                     # Cedar policy generation from roles/bindings
      policygen_test.go
      reload.go                        # Watch-based hot-reload
      reload_test.go
  cmd/platform-api-server/
    main.go                            # Wire authn/authz, load roles, start watching
    resources.go                       # Resource config + authz store construction
helm/charts/platform-api-server/
  templates/platformroles/
    platformrole-cluster-admin.yaml
    platformrole-cluster-viewer.yaml
    platformrole-service-admin.yaml
  values.yaml                         # platformRoles.enabled flag
orlop/pkg/apiserver/
  handlers/context.go                  # AuthorizedNamespaces + ItemFilter context keys
  storage/interface.go                 # ListOptions.Namespaces, FieldFilters
  storage/types.go                     # ResourceEvent.PreviousObject
```

---

## Verification Checklist

- [ ] Unauthenticated request → 401
- [ ] Authenticated user with no bindings → 403
- [ ] cluster-admin can CRUD clusters and nodepools within their namespace
- [ ] cluster-admin cannot manage rolebindings or roles
- [ ] cluster-viewer can list/get clusters and nodepools, cannot create/update/delete
- [ ] service-admin can manage rolebindings and roles within their namespace
- [ ] service-admin cannot create/update/delete clusters or nodepools
- [ ] service-admin can bind cluster-admin to themselves (no self-grant prevention)
- [ ] Cross-namespace list returns only resources from authorized namespaces
- [ ] Cross-namespace list with no authorized namespaces returns empty list (not 403)
- [ ] User-defined Role with Cedar condition on RoleBinding filters results correctly
- [ ] User-defined Role with `context.resourcePlural` guard only filters the targeted resource type
- [ ] PlatformRole mutations via public API → no endpoint (no `platformrole.*` permissions)
- [ ] Cedar condition with invalid syntax → 400 at RoleBinding creation
- [ ] Cedar condition containing `Namespace::` → rejected at RoleBinding creation
- [ ] RoleBinding referencing non-existent role → rejected
- [ ] Hot-reload: creating a new RoleBinding takes effect without server restart
- [ ] Hot-reload: deleting a Role invalidates policies and cache immediately
- [ ] Per-binding policy isolation: unconditional binding does not satisfy conditioned policy for same role
- [ ] Health probes bypass authn/authz
