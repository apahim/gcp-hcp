# Cross-Region Resource Replication

***Scope***: GCP-HCP

**Date**: 2026-08-25

## Decision

We will implement a generic cross-region resource replication mechanism using Google Cloud Pub/Sub with last-writer-wins conflict resolution based on timestamps. The replication controller is configured at startup with the set of resource types to replicate. All objects of configured resource types are globally replicated — there is no per-object opt-out. Replication operates at three levels: (1) watch-based real-time publishing on every resource change, (2) periodic resync where each region re-publishes all locally-owned resources on a configurable interval to self-heal missed events or drift, and (3) on-demand resync via a `RESYNC_REQUEST` message that triggers an immediate resync from all regions — used automatically by new regions to bootstrap from an empty database. Replicated objects are treated as **leases**: each carries a refresh deadline that is reset on every upsert. A garbage collector deletes expired replicated objects only when the origin region is confirmed alive (other objects from that region were recently refreshed), preserving objects from offline regions as last-known-good state. Any region can edit or delete a replicated object — edits strip the `replicated-from` annotation, transferring ownership to the editing region, and the change propagates globally. The initial use case is replicating authorization Roles and RoleBindings across regional Gecko instances, but the mechanism is designed to support any API resource type without code changes.

## Context

- **Problem Statement**: Gecko operates multiple independent regional instances, each with its own database. Certain resource types — starting with authorization Roles and RoleBindings — must be consistent across regions so that a user granted access in one region can also access their resources in other regions. Without replication, each region would require independent administrative action to maintain consistent state.
- **Constraints**: Each region must remain independently authoritative (no single point of failure). Replication must be asynchronous to avoid coupling regional availability. The mechanism must support both namespaced and cluster-scoped resources. Must work with the existing `ResourceStore` interface and controller-runtime framework. Must support local development via Pub/Sub emulator.
- **Assumptions**: Eventual consistency is acceptable for the target use cases (authorization data, not transactional data). Concurrent writes to the same resource across regions are rare. Pub/Sub provides at-least-once delivery with reasonable latency (sub-second in most cases). PlatformRoles (system-defined, cluster-scoped) are deployed identically to all regions via Helm and do not need replication.

## Alternatives Considered

1. **Pub/Sub fan-out/fan-in with periodic resync**: Each region runs a Publisher (watches local resource changes, publishes events to a shared Pub/Sub topic) and a Receiver (subscribes to the topic, replays events into the local database). Echo prevention via origin-region annotation. Last-writer-wins conflict resolution via timestamps. Periodic resync re-publishes all locally-owned resources on a configurable interval. New regions bootstrap by publishing a `RESYNC_REQUEST` that triggers an immediate resync from all existing regions.
2. **Spanner Change Streams with direct cross-region writes**: Use Spanner Change Streams as the event source. Each region reads the change stream and writes directly to other regions' Spanner instances.
3. **Shared global Spanner instance**: All regions share a single multi-region Spanner instance for replicated resource types.
4. **No replication (regional independence only)**: Each region manages its own resources independently. Cross-region consistency is the customer's responsibility or handled by an external orchestrator (e.g., Marketplace Handler fan-out).

## Decision Rationale

* **Justification**: Pub/Sub fan-out/fan-in (Alternative 1) preserves regional independence — each region remains authoritative for its own data and processes replication events asynchronously. The mechanism is generic: adding a new resource type to the replication set requires only a configuration change (startup flag), not code changes. The last-writer-wins conflict resolution via timestamps is simple, handles out-of-order Pub/Sub messages correctly, and is sufficient for the target use cases where concurrent cross-region writes to the same object are rare. GCP instances use NTP with tight synchronization (typically <1ms skew), making clock-based resolution practically reliable. The periodic resync pattern follows the controller-runtime resync model — even without watch events, each Publisher periodically re-processes all locally-owned objects. This self-heals missed events and makes new-region bootstrapping trivial: a new region publishes a `RESYNC_REQUEST`, all existing regions respond by re-publishing their resources, and the new region populates through the normal upsert flow. No cross-region API access, no special credentials, no one-time state tracking.
* **Evidence**: The proof-of-concept implementation (gecko `authz-mktplace-replication` branch) demonstrates working cross-region replication for Roles and RoleBindings across two Kind clusters with a shared Pub/Sub emulator. The PoC includes an E2E suite covering unidirectional replication, bidirectional replication, echo prevention, deletion propagation, and namespace auto-creation in non-primary regions.
* **Comparison**: Spanner Change Streams (Alternative 2) couples the replication mechanism to the Spanner storage backend, breaking the storage-backend abstraction. A shared Spanner instance (Alternative 3) introduces a cross-region dependency that violates the [regional independence architecture](regional-independence-architecture.md) decision and creates a single point of failure. No replication (Alternative 4) pushes consistency management to external systems (Marketplace Handler fan-out), which does not scale to user-defined resources that can be created from any region.

## Consequences

### Positive

* Each region remains independently authoritative — Pub/Sub unavailability does not affect local operations
* Generic mechanism: new resource types are added via startup flags without code changes
* Any region can edit or delete replicated resources — edits transfer ownership to the editing region and propagate globally, deletes propagate globally. Customers do not need to know which region created a resource.
* Namespace auto-creation in non-primary regions ensures replicated namespaced resources always have a valid target namespace
* Works with any storage backend (Spanner, PostgreSQL, in-memory) via the `ResourceStore` interface
* New regions bootstrap automatically — publish a `RESYNC_REQUEST` on startup, existing regions re-publish their resources, no cross-region API access or special credentials needed
* Periodic resync self-heals missed events and drift without operator intervention
* Replicated objects expire automatically when the origin region is alive but no longer claims them — dropped DELETE events are self-healing
* Objects from offline regions are preserved as last-known-good state (fail-safe)

### Negative

* Eventual consistency — there is a window (typically sub-second, but unbounded during Pub/Sub outages) where regions have different state
* Last-writer-wins relies on wall-clock timestamps for conflict resolution. GCP NTP synchronization keeps clock skew under 1ms in practice, but in theory a clock-skewed write could win over a genuinely later write. Concurrent edits to the same object in different regions are resolved by last-writer-wins — one edit may be silently lost. This is acceptable for the current use cases (concurrent cross-region writes to the same object are rare) but may need vector clocks or CRDTs for higher-contention resources in the future.
* Pub/Sub does not guarantee message ordering — the receiver must handle out-of-order events correctly (the timestamp-based last-writer-wins resolution provides this)
* At-least-once delivery requires idempotent processing in the receiver
* No dead-letter queue — permanent errors are Ack'd to prevent poison-pill redelivery. Dropped events are logged at ERROR level and counted via Prometheus metrics. For CREATE/UPDATE drops, periodic resync self-heals. For DELETE drops, object expiry provides the safety net (the origin region stops refreshing the object, and it expires once the GC confirms the origin is alive).
* A `RESYNC_REQUEST` causes all regions to re-publish all their locally-owned resources simultaneously, creating a burst of Pub/Sub traffic. For the current use cases (small number of authorization resources) this is negligible, but could be a concern with larger replicated datasets.
* Periodic resync adds a baseline of Pub/Sub traffic proportional to the total number of replicated resources multiplied by the number of regions, on each resync interval. The receiver's last-writer-wins check ensures this is harmless but not free.

## Cross-Cutting Concerns

### Reliability:

* **Scalability**: Each additional resource type adds one controller-runtime reconciler to the Publisher. The Receiver processes all resource types from a single Pub/Sub subscription. Throughput is bounded by Pub/Sub subscription throughput and the receiver's processing rate.
* **Observability**: All replication operations emit structured audit logs (resource kind, namespace, name, origin region, event type, outcome). Publisher, Receiver, and GC events are logged at INFO level; failures at WARN/ERROR. Prometheus metrics cover events published/received/skipped/dropped, publish and receive durations, resync durations, GC expirations vs. preservations, and a gauge of current replicated objects by origin region. Dropped events (`replication_events_dropped_total`) are explicitly tracked to surface the concern raised about silently discarded permanent errors.
* **Resiliency**: Pub/Sub unavailability does not affect local operations — the Publisher requeues failed publishes after a delay. The Receiver uses Pub/Sub's built-in retry (Nack for transient errors). Permanent errors are Ack'd to prevent poison-pill redelivery, logged at ERROR, and counted via metrics. Periodic resync automatically repairs CREATE/UPDATE events lost during an outage. Object expiry handles dropped DELETE events — the origin region stops refreshing the deleted object, and the GC cleans it up once it confirms the origin is alive. Objects from offline regions are preserved (fail-safe). New regions bootstrap via `RESYNC_REQUEST` — no dependency on replaying historical Pub/Sub messages.

### Security:

* The `replication.gcp.managed.openshift.io/replicated-from` annotation marks objects received from other regions. The Publisher's predicate filter excludes annotated objects, preventing infinite replication loops (A publishes → B receives → B would publish → filtered out because annotation is present). Editing a replicated object strips the annotation, transferring ownership to the editing region — the Publisher then picks up the change and propagates it globally.
* Pub/Sub authentication uses GCP Workload Identity (production) or the emulator (local development).
* The replication controller's RBAC is scoped to only the configured resource types plus namespace read/create permissions.

### Performance:

* Replication is fully asynchronous — no impact on the request path
* The Publisher uses controller-runtime's reconcile loop (not a polling loop), so events are processed immediately on resource changes
* The Receiver processes events sequentially per subscription, which is sufficient for the expected write volume

### Cost:

* Pub/Sub cost scales with message volume — negligible for authorization data (low write frequency)
* No additional database cost — replicated resources are stored in the same regional database as other resources
* One additional controller deployment per region (lightweight Go binary)

### Operability:

* Local development uses the Google Cloud Pub/Sub emulator, either in-cluster or as a standalone container on the Docker/Podman network
* The replication controller runs as a subcommand of the existing `gecko-controllers` binary — no separate image needed
* Kind multi-cluster setup scripts automate the creation of two clusters with a shared Pub/Sub emulator for testing
* The E2E test suite validates all replication scenarios including edge cases (echo prevention, ownership transfer, namespace auto-creation, object expiry)
