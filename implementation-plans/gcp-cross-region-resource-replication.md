# Cross-Region Resource Replication

## Overview

This document specifies the generic cross-region resource replication mechanism for Gecko. It defines the event model, publisher/receiver architecture, conflict resolution, ownership transfer, and deployment topology.

This implements the architecture decided in [cross-region-resource-replication](../design-decisions/infrastructure/cross-region-resource-replication.md).

**Repository**: [gecko](https://github.com/openshift-online/gecko)

---

## Design Decisions

| Decision | Choice |
|---|---|
| Transport | Google Cloud Pub/Sub (shared topic, per-region subscriptions) |
| Resource type selection | Startup flags (`--replicate`) — configurable per deployment |
| Echo prevention | Annotation `replication.gcp.managed.openshift.io/replicated-from` + publisher predicate filter |
| Conflict resolution | Last-writer-wins via `UpdatedAt` timestamp |
| Ownership transfer | Editing a replicated object strips `replicated-from`, transfers ownership to the editing region |
| Namespace handling | Receiver auto-creates target namespace if missing |
| Error handling | Permanent errors Ack'd (prevent poison pill), logged at ERROR, counted via metrics; transient errors Nack'd (Pub/Sub retries) |
| Periodic resync | Each Publisher re-publishes all locally-owned resources on a configurable interval (`--resync-interval`) |
| New region bootstrap | On startup with empty database, publish `RESYNC_REQUEST` — existing regions immediately re-publish their resources |
| Replicated object expiry | Objects carry a `refresh-deadline` annotation; GC deletes expired objects only when origin region is confirmed alive |
| Observability | Structured audit logs + Prometheus metrics for all Publisher, Receiver, and GC operations |
| Initial use case | Authorization Roles and RoleBindings |

---

## Annotations

Two annotations control replication behavior. Both use the `replication.gcp.managed.openshift.io` prefix. All objects of configured resource types are globally replicated — there is no per-object opt-out.

### Replicated-From Annotation

```
replication.gcp.managed.openshift.io/replicated-from: <origin-region>
```

Set by the Receiver on objects it creates/updates from replication events. This annotation serves two purposes:

1. **Echo prevention**: The Publisher's predicate filter excludes objects with this annotation, preventing infinite replication loops (A publishes → B receives and writes → B's publisher sees the write → filtered out because annotation is present).
2. **Provenance tracking**: Indicates which region originally created the object.

### Refresh-Deadline Annotation

```
replication.gcp.managed.openshift.io/refresh-deadline: <RFC3339 timestamp>
```

Set by the Receiver on every upsert. Value is `now + replication-ttl` (configurable via `--replication-ttl`, default `2h`). The garbage collector uses this annotation to identify expired replicated objects. An object whose `refresh-deadline` is in the past is a candidate for deletion — but only if the origin region is confirmed alive (see [Replicated Object Expiry](#replicated-object-expiry)).

---

## Replication Event Model

Events are serialized as JSON and published to Pub/Sub as message payloads.

```go
type ReplicationEvent struct {
    EventType    string          `json:"eventType"`    // "CREATE_OR_UPDATE" or "DELETE"
    ResourceKind string          `json:"resourceKind"` // e.g., "Role", "RoleBinding"
    OriginRegion string          `json:"originRegion"` // Region that originated the event
    Namespace    string          `json:"namespace"`    // Resource namespace (empty for cluster-scoped)
    Name         string          `json:"name"`         // Resource name
    UpdatedAt    time.Time       `json:"updatedAt"`    // Timestamp for last-writer-wins
    Object       json.RawMessage `json:"object"`       // Full serialized resource (CREATE_OR_UPDATE only)
}
```

| Field | Description |
|---|---|
| `EventType` | `CREATE_OR_UPDATE` for upserts, `DELETE` for removals, `RESYNC_REQUEST` to trigger immediate resync from all regions. |
| `ResourceKind` | The Kubernetes Kind of the resource (e.g., `"Role"`, `"RoleBinding"`). Used by the receiver to route deserialization. |
| `OriginRegion` | The region where the event originated. Used for echo prevention — receivers skip events from their own region. |
| `Namespace` | The Kubernetes namespace of the resource. Empty for cluster-scoped resources. |
| `Name` | The resource name. Combined with `Namespace` and `ResourceKind`, this uniquely identifies the resource. |
| `UpdatedAt` | Timestamp used for last-writer-wins conflict resolution. Derived from the resource's last-modified time or `time.Now()` for deletions. GCP NTP synchronization keeps clock skew under 1ms in practice, making timestamp comparison reliable for the expected write patterns. |
| `Object` | The full JSON-serialized resource. Present only for `CREATE_OR_UPDATE` events. Absent for `DELETE` events. |

---

## Publisher

The Publisher is a set of controller-runtime reconcilers — one per configured resource type. It watches local resource changes and publishes replication events to a shared Pub/Sub topic.

### Setup

```go
func (p *Publisher) SetupWithManager(mgr ctrl.Manager) error {
    // Register one controller per configured resource type
    for _, gvk := range p.replicatedTypes {
        ctrl.NewControllerManagedBy(mgr).
            For(resourceForGVK(gvk)).
            WithEventFilter(notReplicatedPredicate()).
            Complete(reconcilerFor(gvk, p))
    }
}
```

### Predicate Filter

The `notReplicatedPredicate()` filters out objects that have the `replicated-from` annotation. This prevents the publisher from re-publishing objects that were received from other regions, breaking the replication loop.

```go
func notReplicatedPredicate() predicate.Predicate {
    return predicate.NewPredicateFuncs(func(obj client.Object) bool {
        _, hasAnnotation := obj.GetAnnotations()["replication.gcp.managed.openshift.io/replicated-from"]
        return !hasAnnotation
    })
}
```

### Publishing Logic

On reconcile:

1. **Object not found** (deleted): Publish a `DELETE` event with the resource's namespace, name, and kind. This applies to both locally-owned and replicated objects — deletes propagate globally regardless of origin. (The predicate filter only applies to create/update events on existing objects; on deletion the object is gone, so the predicate does not fire.)
2. **Object has `replicated-from` annotation**: Skip (belt-and-suspenders check — the predicate should have already filtered it).
3. **Otherwise**: Serialize the object and publish a `CREATE_OR_UPDATE` event. This includes objects that were previously replicated but had their `replicated-from` annotation stripped via an edit (ownership transfer) — the Publisher picks them up as locally-owned and propagates the change.

Failed publishes are requeued after a 5-second delay (`replicationRetryDelay`).

### Periodic Resync

On a configurable interval (`--resync-interval`, default 30m), the Publisher re-lists all locally-owned (non-replicated) resources of each configured type and re-publishes each as a `CREATE_OR_UPDATE` event. This follows the controller-runtime resync pattern:

- **Self-healing**: If a Pub/Sub message was lost, the next resync corrects it.
- **Drift repair**: If a resource is in an inconsistent state across regions, the periodic resync converges it.
- **Harmless duplicates**: The Receiver's last-writer-wins check ensures that re-published events for already-up-to-date resources are silently skipped.

The resync iterates only over locally-owned resources (those without the `replicated-from` annotation). Resources received from other regions are not re-published — each region is responsible for re-publishing its own resources.

### Resync Request Handling

When the Publisher's companion Receiver receives a `RESYNC_REQUEST` event from another region, the Publisher triggers an immediate resync (re-list and re-publish all locally-owned resources), resetting the periodic resync timer.

---

## Receiver

The Receiver subscribes to the Pub/Sub topic via a region-specific subscription and processes incoming replication events.

### Message Processing

```
Message received
  ├─ Deserialize ReplicationEvent from JSON
  ├─ Echo prevention: skip if event.OriginRegion == receiver.region
  ├─ Route by EventType:
  │   ├─ CREATE_OR_UPDATE → upsert()
  │   ├─ DELETE → delete()
  │   └─ RESYNC_REQUEST → trigger immediate Publisher resync
  └─ Error handling:
      ├─ Permanent error (Invalid, Forbidden, MethodNotSupported) → Ack, log ERROR, increment replication_events_dropped_total
      └─ Transient error → Nack (Pub/Sub retries)
```

### Upsert Flow

1. Deserialize the incoming resource from `event.Object`
2. **Namespace auto-creation**: If the target namespace does not exist, create it. This is critical for non-primary regions where namespace-creating controllers (e.g., Marketplace controller) may not run.
3. Set the `replicated-from` annotation to `event.OriginRegion`
4. Set the `refresh-deadline` annotation to `now + replication-ttl`
5. Clear `ResourceVersion` (new write in local store)
6. Attempt `Get` on the existing object:
   - **Not found** → `Create` the object. Handle `AlreadyExists` gracefully (concurrent creation).
   - **Found** → **Last-writer-wins**: compare `event.UpdatedAt` with the existing object's last-modified timestamp. If the incoming event is newer, update the object (including refreshing the `refresh-deadline`). If the existing object is newer or equal, still refresh the `refresh-deadline` (the origin region is alive and still claims this object).

### Delete Flow

Delete the resource by namespace and name. NotFound is treated as success (idempotent). The replicated-from annotation on existing objects is not checked — if a DELETE event arrives, the object is removed regardless.

### Namespace Auto-Creation

```go
func (r *Receiver) ensureNamespace(ctx context.Context, namespace string) error {
    ns := &corev1.Namespace{
        ObjectMeta: metav1.ObjectMeta{Name: namespace},
    }
    err := r.client.Create(ctx, ns)
    if err != nil && !apierrors.IsAlreadyExists(err) {
        return err
    }
    return nil
}
```

This requires the replication controller's RBAC to include `get`, `list`, `watch`, `create` on `namespaces`.

---

## Pub/Sub Topology

```
Region A (Publisher)  ──publish──>  Pub/Sub Topic  <──publish──  Region B (Publisher)
                                       │
                          ┌─────────────┼─────────────┐
                          ▼                           ▼
                  Subscription A                Subscription B
                  (Region A Receiver)           (Region B Receiver)
                          │                           │
                   echo prevention:             echo prevention:
                   skip OriginRegion=A          skip OriginRegion=B
                          │                           │
                   process B's events           process A's events
```

Each region has:
- A **Publisher** that watches local resources and publishes to the shared topic
- A **Receiver** that subscribes to the topic via a region-specific subscription and processes events from other regions

The shared topic name and per-region subscription names are configured via startup flags.

---

## Configuration

The replication controller runs as a subcommand of the `gecko-controllers` binary.

### Startup Flags

| Flag | Env Var | Default | Description |
|---|---|---|---|
| `--region` | `REPL_REGION` | (required) | This region's identifier |
| `--pubsub-project` | `PUBSUB_PROJECT` | `gecko-local` | GCP project ID for Pub/Sub |
| `--pubsub-topic` | `REPL_PUBSUB_TOPIC` | `resource-replication` | Pub/Sub topic name |
| `--pubsub-subscription` | `REPL_PUBSUB_SUBSCRIPTION` | (required) | Region-specific subscription name |
| `--replicate` | `REPL_RESOURCE_TYPES` | (required) | Comma-separated list of resource types to replicate (e.g., `roles.gcp.managed.openshift.io,rolebindings.gcp.managed.openshift.io`) |
| `--resync-interval` | `REPL_RESYNC_INTERVAL` | `30m` | Interval between periodic resyncs (re-publish all locally-owned resources) |
| `--replication-ttl` | `REPL_TTL` | `2h` | TTL for replicated objects (refresh-deadline = now + TTL on each upsert). Must be > resync-interval. |
| `--gc-interval` | `REPL_GC_INTERVAL` | `1m` | How often the garbage collector scans for expired replicated objects |

The `PUBSUB_EMULATOR_HOST` environment variable is supported for local development with the Pub/Sub emulator.

Startup validation: the controller rejects `--replication-ttl` values less than or equal to `--resync-interval` to guarantee at least one resync opportunity before expiry.

---

## RBAC

The replication controller requires a ServiceAccount with a ClusterRole granting:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: replication-controller
rules:
  # Replicated resource types (example: roles + rolebindings)
  - apiGroups: ["gcp.managed.openshift.io"]
    resources: ["roles", "rolebindings"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  # Namespace auto-creation
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch", "create"]
  # Event recording
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```

The RBAC rules should match the configured `--replicate` resource types. The namespace and events rules are always required.

---

## Deployment

### Controller Binary

The replication controller is a subcommand of the existing `gecko-controllers` binary:

```
gecko-controllers replication \
  --region=us-east1 \
  --pubsub-subscription=repl-us-east1 \
  --replicate=roles.gcp.managed.openshift.io,rolebindings.gcp.managed.openshift.io
```

### Containerfile

Reuses the `gecko-controllers` binary image. The entrypoint specifies the `replication` subcommand:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:latest
COPY gecko-controllers /app/gecko-controllers
USER 65532:65532
ENTRYPOINT ["/app/gecko-controllers", "replication"]
```

### Kubernetes Deployment

Single replica per region in the `gecko-system` namespace. Environment variables configure region, Pub/Sub project, topic, subscription, and (for local dev) the emulator host.

---

## Local Development

### Pub/Sub Emulator

Local development uses the Google Cloud Pub/Sub emulator (`gcr.io/google.com/cloudsdktool/google-cloud-cli:emulators`). In a multi-cluster Kind setup, the emulator runs as a standalone container on the Docker/Podman network (shared by both clusters), not inside either cluster.

### Kind Multi-Cluster Setup

The `deploy/kind/setup-multi-region.sh` script:

1. Creates two Kind clusters (e.g., `gecko-us-east1`, `gecko-eu-west1`)
2. Starts a shared Pub/Sub emulator container
3. Creates topics and per-region subscriptions
4. Builds and loads controller images into both clusters
5. Deploys via Kustomize with region-specific overlays
6. Configures in-cluster headless Services pointing to the shared emulator's container IP

### Kustomize Overlays

Each region has a Kustomize overlay that patches the replication controller deployment with:
- Region-specific environment variables (`REPL_REGION`, `REPL_PUBSUB_SUBSCRIPTION`)
- Pub/Sub emulator host configuration

---

## E2E Test Coverage

The E2E test suite runs against two Kind clusters and validates:

| # | Test | What It Validates |
|---|---|---|
| 1 | Unidirectional replication (A → B) | Resource created in A appears in B with `replicated-from` annotation |
| 2 | RoleBinding replication | Namespace-scoped binding replicates correctly |
| 3 | Bidirectional replication (B → A) | Resource created in B appears in A |
| 4 | Echo prevention | Replicated object in B does not bounce back to A |
| 5 | Deletion replication | Deleting a resource in A removes it from B |
| 6 | Namespace auto-creation | Receiver creates the namespace in B if it does not exist |
| 7 | Resync request (new region) | New region publishes `RESYNC_REQUEST`, receives all existing resources from other regions |
| 8 | Periodic resync repairs drift | Resource manually deleted in B reappears after A's periodic resync |
| 9 | Object expiry (origin alive) | Replicated object whose origin region is alive but stopped refreshing it is garbage collected after TTL |
| 10 | Object preservation (origin offline) | Replicated object from an offline region (no recent refreshes on any of its objects) is preserved |
| 11 | Dropped DELETE recovery | DELETE event permanently dropped; object expires via GC after origin region's next resync does not include it |
| 12 | Ownership transfer via edit | Edit replicated object in B → `replicated-from` stripped → change propagates to A → A now has `replicated-from: B` |
| 13 | Delete replicated object propagates | Delete a replicated object in remote region; verify it is removed from all regions including origin |

Tests use polling with a 30-second timeout. A `--no-pause` flag supports CI execution (no interactive pauses).

---

## Replication Lifecycle

Resource replication operates at three levels:

### 1. Watch-Based Publishing (Real-Time)

The primary mechanism. The Publisher watches local resource changes via controller-runtime reconcilers and publishes events immediately as they happen. This provides sub-second replication latency for normal operations.

### 2. Periodic Resync (Drift Repair)

On a configurable interval (`--resync-interval`, default 30m), each region's Publisher re-lists all locally-owned (non-replicated) resources and re-publishes each as a `CREATE_OR_UPDATE` event. This follows the controller-runtime resync pattern and self-heals:

- **Missed events**: If a Pub/Sub message was lost or Ack'd but not processed due to a bug, the next resync corrects it.
- **Drift**: If a resource is in an inconsistent state across regions for any reason, the periodic resync converges it.
- **No special code path**: The re-published events flow through the normal Receiver upsert path. The last-writer-wins check ensures duplicates are harmless.

### 3. Resync Request (New Region Bootstrap)

When a new region starts with an empty database, it publishes a `RESYNC_REQUEST` event to the shared topic:

```
New region starts up
  → detects empty database (no replicated resources of any configured type)
  → publishes RESYNC_REQUEST to shared topic
  → all existing regions receive it (echo prevention skips the requesting region)
  → each existing region triggers an immediate resync:
      re-lists and re-publishes all locally-owned resources
  → new region's Receiver processes them through normal upsert flow
  → new region is populated
  → normal watch-based replication continues
```

**Design properties**:

- **No cross-region API access**: Everything flows through Pub/Sub. No special credentials, no direct API calls to other regions.
- **No one-time state tracking**: The resync request is fire-and-forget. If the new region restarts before it's fully populated, it detects the still-empty database and publishes another `RESYNC_REQUEST`.
- **Additive**: The upsert flow creates missing resources. It does not delete resources that exist locally but not in other regions.
- **Harmless to existing regions**: Existing regions also receive the re-published events but their last-writer-wins check skips already-up-to-date resources.

### Operational Procedure — Adding a New Region

Adding a new region (e.g., `europe-west4`):

1. Deploy the Gecko infrastructure to the new region (platform-api-server, Helm charts for PlatformRoles, etc.)
2. Create a Pub/Sub subscription for the new region on the shared replication topic
3. Deploy the replication controller with the normal flags (no special sync flags needed)
4. The controller detects an empty database, publishes `RESYNC_REQUEST`, and populates automatically
5. Verify via E2E tests or `kubectl` that replicated resources are present in the new region

---

## Replicated Object Expiry

Replicated objects are treated as **leases** — they must be periodically refreshed by the origin region or they expire. This ensures that dropped DELETE events do not leave stale resources granting unintended access.

### Refresh Mechanism

Every time the Receiver upserts a replicated object (whether from a watch event, periodic resync, or resync request response), it sets the `refresh-deadline` annotation to `now + replication-ttl`. This means:

- Objects that are still alive in the origin region get their deadline refreshed on every resync cycle
- Objects that were deleted in the origin region stop being re-published, so their deadline is never refreshed

### Garbage Collector

A background goroutine runs on `--gc-interval` (default `1m`) and scans all replicated objects (those with a `replicated-from` annotation):

```
for each replicated object where refresh-deadline < now:
    originRegion = object.annotations["replicated-from"]
    otherObjects = all replicated objects from originRegion where refresh-deadline >= now
    if len(otherObjects) > 0:
        // Origin region is alive (other objects were recently refreshed)
        // but this object was not refreshed → origin no longer has it
        delete(object)
        log INFO: "Expired replicated object {kind}/{ns}/{name} from {originRegion}"
        increment replication_gc_expired_total
    else:
        // No recent activity from origin region → region may be offline
        // Preserve the object as last-known-good state
        log INFO: "Preserving expired object {kind}/{ns}/{name} — origin {originRegion} appears offline"
        increment replication_gc_preserved_total
```

### Why This Is Safe

- **DELETE event succeeds** (normal case): Object removed immediately. No expiry needed.
- **DELETE event dropped** (permanent error): Origin region stops refreshing the object during periodic resync. On the next GC cycle after the `refresh-deadline` passes, the GC sees that other objects from the origin region are recently refreshed (origin is alive), so it deletes the stale object.
- **Origin region offline**: No objects from that region are refreshed. All their deadlines eventually expire, but the GC sees zero recently-refreshed objects from that region and preserves everything. This is the safe behavior — an offline region's authorization data should not be garbage collected.
- **Origin region comes back online**: Its next resync refreshes all objects it still has. Objects it deleted while offline are not refreshed and will expire on the next GC cycle.

### Known Limitation

If a region has exactly one replicated object in a remote region and that object is deleted, there are no "other objects" from the origin region to compare against. The GC treats the origin as offline and preserves the object. This is a minor edge case — for authorization data, the marketplace creates a RoleBinding alongside a Role, so there are always at least 2 objects per namespace.

---

## Ownership Transfer

Replicated objects can be edited or deleted from any region. Customers do not need to know which region created a resource.

### Edit in a Remote Region

When a customer edits a replicated object (one with a `replicated-from` annotation) via the public API:

1. The edit succeeds — the public API does not block edits to replicated objects
2. A public API validator (or admission webhook) strips the `replicated-from` annotation on update
3. The object is now locally owned in the editing region
4. The Publisher's watch detects the change (the object no longer has `replicated-from`, so the predicate allows it)
5. The Publisher publishes a `CREATE_OR_UPDATE` event with the updated object
6. All other regions (including the original origin) receive the event
7. The Receiver in each region upserts the object — the original origin's copy gets `replicated-from: <new-owner>`, completing the ownership transfer

```go
func stripReplicatedFromOnUpdate(ctx context.Context, obj client.Object) {
    annotations := obj.GetAnnotations()
    delete(annotations, "replication.gcp.managed.openshift.io/replicated-from")
    delete(annotations, "replication.gcp.managed.openshift.io/refresh-deadline")
    obj.SetAnnotations(annotations)
}
```

**Enforcement scope**: Public API only. The private API (kube-apiserver) is not affected — the Receiver uses the private API and must be able to set the `replicated-from` annotation.

### Delete in Any Region

When a replicated object is deleted (in any region, including remote regions):

1. The delete succeeds locally — the object is removed from the database
2. The Publisher's reconcile loop detects the deletion (object not found)
3. The Publisher publishes a `DELETE` event to the shared topic — the predicate filter does not apply because the object no longer exists to inspect
4. All other regions receive the DELETE event and remove their copies (including the origin region)

This means a customer can delete a replicated resource from any region without needing to know which region created it. The deletion takes effect everywhere.

### Convergence Properties

- Ownership transfer is atomic from a convergence standpoint — after one resync cycle, all regions agree on the new owner
- Concurrent edits in different regions are resolved by last-writer-wins (newer timestamp wins)
- The periodic resync ensures the current owner's version eventually propagates even if individual events are lost

---

## Observability

### Audit Logs

All replication operations emit structured log entries using the controller-runtime logger (`logger.Info`/`logger.Error` with key-value pairs).

**Publisher:**

| Level | Event | Fields |
|-------|-------|--------|
| INFO | Published event | `eventType`, `resourceKind`, `namespace`, `name` |
| INFO | Published after ownership transfer | `resourceKind`, `namespace`, `name` |
| INFO | Published `RESYNC_REQUEST` | `region` |
| INFO | Periodic resync completed | `resourcesPublished` (count) |
| INFO | Skipped replicated object | `resourceKind`, `namespace`, `name` |
| WARN | Publish failed, requeuing | `resourceKind`, `namespace`, `name`, `error` |

**Receiver:**

| Level | Event | Fields |
|-------|-------|--------|
| INFO | Upserted resource | `eventType`, `resourceKind`, `namespace`, `name`, `originRegion`, `outcome` (`created`/`updated`/`skipped_stale`) |
| INFO | Deleted resource | `resourceKind`, `namespace`, `name`, `originRegion` |
| INFO | Created namespace | `namespace` |
| INFO | Received `RESYNC_REQUEST` | `originRegion` |
| INFO | Refreshed deadline (stale event) | `resourceKind`, `namespace`, `name`, `originRegion` |
| ERROR | Permanent error, Ack'd | `eventType`, `resourceKind`, `namespace`, `name`, `originRegion`, `error` |

**Garbage Collector:**

| Level | Event | Fields |
|-------|-------|--------|
| INFO | Expired object deleted | `resourceKind`, `namespace`, `name`, `originRegion` |
| INFO | Expired object preserved | `resourceKind`, `namespace`, `name`, `originRegion`, `reason` (`origin_offline`) |
| INFO | GC cycle completed | `scanned`, `expired`, `preserved` |

### Prometheus Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `replication_events_published_total` | Counter | `event_type`, `resource_kind` | Events published by this region |
| `replication_events_received_total` | Counter | `event_type`, `resource_kind`, `origin_region` | Events processed by the Receiver |
| `replication_events_skipped_total` | Counter | `reason` (`echo`, `stale`) | Events skipped without processing |
| `replication_events_dropped_total` | Counter | `event_type`, `resource_kind`, `reason` | Permanent errors Ack'd |
| `replication_publish_errors_total` | Counter | `resource_kind` | Publish failures (requeued) |
| `replication_publish_duration_seconds` | Histogram | `event_type` | Time to publish a single event |
| `replication_receive_duration_seconds` | Histogram | `event_type` | Time to process a single received event |
| `replication_resync_duration_seconds` | Histogram | | Time for a full periodic resync cycle |
| `replication_gc_expired_total` | Counter | `resource_kind`, `origin_region` | Objects deleted by GC (origin alive) |
| `replication_gc_preserved_total` | Counter | `resource_kind`, `origin_region` | Expired objects preserved (origin offline) |
| `replication_replicated_objects` | Gauge | `resource_kind`, `origin_region` | Current count of replicated objects |

### Recommended Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| Replication events being dropped | `rate(replication_events_dropped_total[5m]) > 0` | Warning |
| Sustained publish failures | `rate(replication_publish_errors_total[5m]) > 0.1` for 10m | Warning |
| Replicated objects from a region dropping to zero | `replication_replicated_objects == 0` for a previously-nonzero origin region | Warning |
| GC expiring objects | `rate(replication_gc_expired_total[1h]) > 10` | Info (may indicate a region deleted many resources) |

---

## Integration: Authorization Use Case

The initial use case for cross-region replication is authorization Roles and RoleBindings. The integration points with the Cedar authorization system (see [Cedar authorization plan](gcp-cedar-public-api-authorization.md)):

- **Replicated resources**: `roles.gcp.managed.openshift.io` and `rolebindings.gcp.managed.openshift.io`
- **PlatformRoles are NOT replicated**: They are system-defined and deployed identically to all regions via Helm
- **Cedar hot-reload interaction**: When the receiver creates/updates/deletes a replicated Role or RoleBinding in the local database, the Cedar authorizer's watch mechanism detects the change and triggers a policy rebuild and cache invalidation — the same path as a local write.
- **Marketplace integration**: The Marketplace controller creates the initial `service-admin` RoleBinding in the primary region. The replication controller propagates it to all other regions, ensuring the customer is authorized everywhere.

---

## File Structure

```text
controllers/
  replication/
    publisher.go                        # Controller-runtime reconcilers for publishing
    publisher_test.go
    receiver.go                         # Pub/Sub message handler with upsert/delete
    receiver_test.go
    types.go                            # ReplicationEvent struct + constants
  cmd/replication/
    cmd.go                              # Cobra subcommand + wiring
deploy/kind/
  replication/
    Containerfile.controller            # Container image for replication controller
    controller-deployment.yaml          # Kubernetes deployment
    kustomization.yaml                  # Base kustomization
    rbac.yaml                           # ServiceAccount + ClusterRole + ClusterRoleBinding
    test/
      e2e-test.sh                       # 8-scenario E2E test suite
  setup-multi-region.sh                 # Multi-cluster Kind setup script
  teardown-multi-region.sh              # Cleanup script
deploy/multi-region/
  us-east1/kustomization.yaml           # Region overlay (primary)
  eu-west1/kustomization.yaml           # Region overlay (secondary)
```
