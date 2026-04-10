# Cross-Org Customer Notifications Implementation Plan

***Scope***: GCP-HCP

**Date**: 2026-03-26

## Overview

This document provides implementation guidance for sending operational alerts (cluster health, incidents, maintenance) from the GCP HCP service provider project to customer GCP projects using Pub/Sub.

**Selected approach:** Customer-Owned Topic with Cross-Project Publishing (Option B).

## Architecture

### Message Flow

```
Your Project                              Customer Project
+--------------------------+              +---------------------------+
| Cloud Monitoring Alert   |              |                           |
|   -> Pub/Sub (internal)  |              |  Customer's Pub/Sub Topic |
|   -> Eventarc            |   publish    |    -> Pull Subscription   |
|   -> Notification        | ----------> |    -> Push to Cloud Func  |
|      Publisher Service   |              |    -> Route to Workflows  |
+--------------------------+              +---------------------------+
```

### How It Works

1. Customer creates a Pub/Sub topic in **their** GCP project
2. Customer grants `roles/pubsub.publisher` to the service provider's service account
3. The Notification Publisher service publishes alert messages to the customer's topic
4. Customer manages their own subscriptions and downstream processing

### Integration with Existing Alerting Framework

Extends the existing alerting pipeline defined in `design-decisions/integrated-alerting-framework.md`:

```
Cloud Monitoring Alert
    -> Pub/Sub (internal topic: diagnose-workflow)
    -> Eventarc -> Cloud Run (diagnose-agent)
    -> [NEW] Notification Publisher Service
        -> for each affected customer:
            -> Pub/Sub Publish to customer's topic
```

### Regional Deployment Model

Per `design-decisions/regional-independence-architecture.md`, the Notification Publisher is deployed **per region project**, matching the `diagnose-agent` pattern in `gcp-hcp-infra/terraform/modules/region/agent.tf`:

- Each region's Notification Publisher operates independently — no cross-region dependencies
- Customer Registration Store is scoped per region (Firestore collection within each region's project)
- Alerts remain regionally scoped: an alert in `us-central1` publishes only through the `us-central1` publisher
- Multi-region customers register the same topic from each region (or use separate topics per region at their discretion)
- No cross-region notification aggregation — this preserves regional independence and simplifies failure isolation

## Dependencies and Implementation Sequence

### Existing Infrastructure Dependencies

This plan extends the alerting pipeline already built in `gcp-hcp-infra`:

| Component | Location | Gate |
|-----------|----------|------|
| Pub/Sub topic `diagnose-workflow` | `terraform/modules/workflows/alerting.tf` | `var.enable_alerting` |
| Eventarc + Cloud Run APIs | `terraform/modules/workflows/alerting.tf` | `var.enable_alerting` |
| `alerting-pipeline` service account | `terraform/modules/workflows/alerting.tf` | `var.enable_alerting` |
| Cloud Run `diagnose-agent` service | `terraform/modules/region/agent.tf` | `var.enable_agent` |
| Eventarc trigger `diagnose-alert` | `terraform/modules/region/agent.tf` | `var.enable_agent` |

The Notification Publisher is a **separate Cloud Run service** (not integrated into `diagnose-agent`). It subscribes to the same `diagnose-workflow` Pub/Sub topic via its own Eventarc trigger, forming a fan-out from the single internal topic.

### Implementation Order

1. **Customer Registration Store** — Firestore collection + data model + registration API
2. **Notification Publisher Cloud Run service** — Container, Eventarc trigger, dedicated SA
3. **DLQ topic + monitoring** — Dead-letter topic, alerting on DLQ depth, replay endpoint
4. **Customer onboarding documentation** — `gcloud` instructions, registration API docs
5. **Canary subscription + integration tests** — Provider-owned canary topic, automated test suite

## Implementation Components

### 1. Notification Publisher Service

- **Runtime:** Cloud Run service (aligns with existing `diagnose-agent` pattern in `region/agent.tf`)
- **Trigger:** Eventarc from internal Pub/Sub topic `diagnose-workflow`, via its own trigger (parallel to `diagnose-alert`)
- **Logic:** Receive internal alert event -> look up affected customer(s) in Firestore -> publish to each customer's registered topic
- **Auth:** Dedicated service account `notification-publisher@{PROJECT}.iam.gserviceaccount.com` with no project-level Pub/Sub roles — publisher access is granted per-topic by customers
- **Concurrency:** Fan-out uses parallel goroutines bounded at 20 concurrent publishes per request
- **Scaling:** Min 0, Max 5 instances (higher than `diagnose-agent` to handle fan-out during incident storms)
- **Batching:** Pub/Sub client library batching enabled for efficient multi-topic publishing

### 2. Customer Registration Store

- Map of `cluster_id` / `customer_id` -> `projects/{customer-project}/topics/{topic-name}`
- Storage options: Firestore, Cloud SQL, or ConfigMap/Secret if customer count is small
- Required fields: topic reference, customer SA identity, registration timestamp, last successful publish

### 3. Notification Payload Schema

Stable, versioned contract for alert payloads:

```json
{
  "version": "1.0",
  "event_type": "operational_alert",
  "severity": "WARNING|ERROR|CRITICAL",
  "cluster_id": "abc-123",
  "summary": "etcd leader election timeout",
  "description": "...",
  "started_at": "2026-03-26T10:00:00Z",
  "resolved_at": null,
  "documentation_url": "https://..."
}
```

Consider Protocol Buffers for stronger typing in future versions. JSON is sufficient for v1.

#### Field Mapping from Cloud Monitoring Payload

The internal Cloud Monitoring v1.2 payload (documented in `design-decisions/integrated-alerting-framework.md`) is transformed to the customer-facing v1.0 schema:

| Customer Field | Source (Cloud Monitoring v1.2) |
|---|---|
| `cluster_id` | `incident.metric.labels.cluster_id` |
| `severity` | `incident.severity` (mapped: Error->ERROR, Critical->CRITICAL, Warning->WARNING) |
| `summary` | `incident.documentation.subject` |
| `description` | `incident.documentation.content` |
| `started_at` | `incident.started_at` (epoch seconds -> ISO 8601) |
| `resolved_at` | `incident.ended_at` (epoch seconds -> ISO 8601, `null` if open) |
| `documentation_url` | `incident.url` |
| `event_type` | Derived from `incident.state` (`open` -> `operational_alert`, `closed` -> `resolution`) |

The customer schema intentionally omits internal details: incident IDs, resource labels (namespace, project), policy references, and raw metric values.

### Schema Versioning Policy

- **Version 1.0 is stable** — no breaking changes to existing fields
- **Additive changes** (new fields): minor version bump (1.0 -> 1.1), no customer action required
- **Breaking changes**: new major version (2.0) with 90-day dual-delivery period where both versions are published
- Customers should parse the `version` field and ignore unknown fields for forward compatibility
- No version negotiation in v1 — all customers receive the same version
- Schema changes communicated via release notes and customer onboarding channel

### 4. Error Handling

- **Publish failures** (permission denied, topic not found): log, increment error metric, alert SRE team after 10 consecutive failures per customer topic
- **Retry policy:** Exponential backoff for transient errors (deadline exceeded, unavailable), max 3 retries per publish attempt
- **Dead-letter queue:**
  - Topic: `notification-dlq` in the service provider project
  - Message retention: 7 days
  - DLQ message format: original payload + target topic + error message + failure count + timestamp
  - Monitoring: alert when DLQ depth exceeds 100 messages in 1 hour
  - Replay: manual replay via Cloud Run endpoint (`POST /replay-dlq`) with optional customer filter

### Observability

SLIs and monitoring for the notification service itself:

- **SLI: Publish success rate** — target 99.9% of publishes succeed (excluding customer-side permission errors, which are tracked separately)
- **SLI: End-to-end latency** — alert fired to customer topic published < 30 seconds at p99
- **Metrics** (exported to Google Managed Prometheus):
  - `notification_publish_total{status, customer_id, region}` — publish attempt counter
  - `notification_publish_latency_seconds{customer_id}` — publish latency histogram
  - `notification_dlq_depth` — current DLQ message count
  - `notification_registration_count{status}` — active/inactive registrations
- **Alert-on-alert:** If publish success rate drops below 95% over 5 minutes, page SRE via PagerDuty
- **Dashboard:** Cloud Monitoring dashboard showing per-customer publish health, latency percentiles, DLQ depth
- **Canary:** Provider-owned canary subscription on a dedicated test topic; alert if canary stops receiving messages within expected interval

## Infrastructure

### Terraform Module Structure

Infrastructure follows the existing patterns in `gcp-hcp-infra`:

**`terraform/modules/workflows/notifications.tf`** (same pattern as `alerting.tf`):
- `google_service_account.notification_publisher` — dedicated SA for cross-project publishing
- `google_pubsub_topic.notification_dlq` — dead-letter topic for failed publishes
- IAM bindings: `roles/logging.logWriter`, `roles/datastore.user` (Firestore), SA-level token creator for Pub/Sub agent
- Gated by `var.enable_customer_notifications` (default: `false`)

**`terraform/modules/region/notification-publisher.tf`** (same pattern as `agent.tf`):
- `google_cloud_run_v2_service.notification_publisher` — Cloud Run service
  - Min instances: 0, Max instances: 5 (higher than `diagnose-agent` due to fan-out)
  - Health checks: `/health` endpoint on port 8080
  - Startup and liveness probes
  - Env vars: `GCP_PROJECT_ID`, `GCP_REGION`, `FIRESTORE_COLLECTION`
- `google_eventarc_trigger.customer_notifications` — Pub/Sub → Cloud Run `/notify`
  - Subscribes to the same `diagnose-workflow` topic as `diagnose-alert`
- `google_cloud_run_v2_service_iam_member` — invoker binding for `alerting-pipeline` SA
- Audit logging for Cloud Run API
- Gated by `var.enable_customer_notifications`

**`terraform/modules/region/firestore.tf`**:
- `google_firestore_database` — Native mode Firestore instance (or reuse existing if present)
- `google_firestore_index` — Composite index on `cluster_id` for efficient lookup

### 5. No-Subscriber Behavior

When publishing to a customer topic with zero subscriptions:

- The `Publish()` API call **succeeds** — returns a valid `message_id`
- The message is **immediately dropped** — Pub/Sub only retains messages for existing subscriptions
- **No error, no warning, no backpressure** to the publisher

**Decision:** Accept this as a customer responsibility. Document clearly in onboarding materials that customers must create at least one subscription to receive notifications. This keeps the architecture simple and avoids needing elevated permissions (viewer/admin) on customer topics.

## Customer Onboarding Flow

Documentation for customers:

1. Create a Pub/Sub topic:
   ```bash
   gcloud pubsub topics create hcp-notifications --project={customer-project}
   ```
2. Grant publisher access:
   ```bash
   gcloud pubsub topics add-iam-policy-binding hcp-notifications \
     --member=serviceAccount:notification-publisher@{region-project}.iam.gserviceaccount.com \
     --role=roles/pubsub.publisher \
     --project={customer-project}
   ```
3. Create at least one subscription (customer responsibility):
   ```bash
   gcloud pubsub subscriptions create hcp-alerts-sub \
     --topic=hcp-notifications \
     --project={customer-project}
   ```
4. Register topic reference with the platform (API call or config)
5. Receive a test notification to confirm connectivity

## Customer Lifecycle Management

### Topic Deletion

When a customer deletes their Pub/Sub topic, the publisher receives `NOT_FOUND` errors:

- After 10 consecutive failures, mark the registration as **inactive** in the registration store
- Alert SRE team via existing PagerDuty channel
- Do not auto-delete the registration — the customer may recreate the topic
- Inactive registrations are skipped during fan-out but remain queryable for audit

### Cluster Decommissioning

When a customer's hosted control plane is decommissioned:

- The cluster teardown workflow triggers cleanup of notification registration entries for that cluster
- If the customer has no remaining clusters, the registration remains but receives no further notifications (no orphan cleanup needed — alerts are cluster-scoped)

### Topic Changes

Customers can update their topic reference via the registration API:

- Publisher validates the new topic with a test publish before switching
- No downtime — the old topic continues receiving notifications until the new topic is confirmed
- Registration store records both `active_topic` and `pending_topic` during transition

### Multi-Cluster Deduplication

When multiple clusters owned by the same customer map to the same notification topic:

- Publisher deduplicates per-topic per alert event to avoid sending duplicate messages
- Deduplication key: `{alert_event_id}:{topic_resource_name}`
- If a platform-wide incident affects multiple clusters, the customer receives one notification per cluster (not duplicated per-topic)

## Security Considerations

- Service account only gets `roles/pubsub.publisher` — cannot read messages, list subscriptions, or modify the topic
- Use a dedicated SA for notification publishing, not the same SA used for cluster management
- Customer can revoke access at any time — publisher will receive `PERMISSION_DENIED` errors
- No cross-org IAM admin access required
- Message payloads must not contain customer secrets or internal infrastructure details
- Shared publisher SA is acceptable under "Least Common Mechanism" because IAM grants are scoped per-topic by each customer, publish failures are isolated (no cross-tenant impact), and the SA cannot enumerate customer topics

### IAM Requirements

| Service Agent / Account | Role | Scope | Purpose |
|------------------------|------|-------|---------|
| `notification-publisher@{PROJECT}` | (none at project level) | — | Dedicated SA for cross-project publishing |
| `notification-publisher@{PROJECT}` | `roles/pubsub.publisher` | Customer topic (granted by customer) | Publish notifications to customer Pub/Sub topics |
| `notification-publisher@{PROJECT}` | `roles/datastore.user` | Project | Read/write customer registration store (Firestore) |
| `notification-publisher@{PROJECT}` | `roles/logging.logWriter` | Project | Write notification and error logs |
| `service-{NUM}@gcp-sa-pubsub` | `roles/iam.serviceAccountTokenCreator` | On `notification-publisher` SA | Mint OIDC tokens for Eventarc push delivery |
| `alerting-pipeline@{PROJECT}` | `roles/run.invoker` | `notification-publisher` Cloud Run service | Invoke publisher service (if chained) |

### Incident Fallback

If the Notification Publisher is down during a platform incident — exactly when customers most need notifications:

- **Manual fallback:** SRE publishes directly via `gcloud pubsub topics publish` to registered customer topics, using registration data from Firestore
- **Status page:** Platform status page is updated independently of Pub/Sub notifications; this is the primary customer-facing incident channel
- **Customer onboarding docs** recommend subscribing to the status page as a secondary notification channel alongside Pub/Sub

## Alternatives Considered

### Option A: Provider-Owned Topic with Cross-Project Subscriptions

Provider creates a topic; customers create subscriptions to it. Simpler fan-out but exposes topic metadata to customers and requires managing per-customer IAM bindings on the provider side.

### Option C: Pub/Sub with Schema Registry + Eventarc

Extends Option A or B with schema validation and Eventarc routing. More complex setup on both sides. Could be adopted later as the schema matures.

### Option D: Pub/Sub Lite

Lower cost but fewer features, requires capacity planning, and is being deprecated. Not recommended.

## Cost Analysis

Cross-project Pub/Sub notification costs are negligible at expected scale:

| Component | Unit Cost | Estimated Usage (100 customers, 10 alerts/day) | Monthly Cost |
|-----------|-----------|-----------------------------------------------|-------------|
| Pub/Sub publishing | $0.40/million publishes | 30K publishes/month | ~$0.01 |
| Cloud Run | ~$0.00002/request | 300 requests/day | Free tier |
| Firestore reads | $0.06/100K reads | ~30K reads/month | ~$0.02 |
| DLQ Pub/Sub retention | $0.27/GiB-month | Negligible | ~$0.00 |
| Eventarc | No additional cost | (included in Pub/Sub) | $0.00 |

**Total estimated cost: < $5/month at 100 customers.** Cost scales linearly with customer count and alert volume. During incident storms, Cloud Run scales up but costs remain low (~$0.10 per 1000 additional fan-out requests).

## Verification

### Manual Validation

- Test cross-project publishing with a sandbox customer project
- Verify publish succeeds with zero subscriptions (confirm message is silently dropped)
- Test failure scenarios: topic deleted, permissions revoked, quota exceeded
- Validate notification schema with sample alert payloads from Cloud Monitoring
- Confirm dedicated SA has no excess permissions

### Automated Testing

- **Unit tests:** Schema mapping (Cloud Monitoring v1.2 -> customer v1.0), deduplication logic, retry/backoff logic
- **Integration tests:** Deploy publisher to sandbox project, register a test topic in a separate project, fire a synthetic alert via Pub/Sub, assert message delivery and schema compliance
- **Failure tests:** Revoke permissions mid-publish, delete topic during fan-out, exceed quota — verify DLQ behavior and SRE alerting fires correctly
- **Load test:** Fan-out to 100 concurrent topics, measure p50/p95/p99 publish latency, verify Cloud Run autoscaling behavior
- **Canary test:** Continuous canary subscription on a provider-owned test topic validates end-to-end delivery; alert if canary stops receiving messages within expected interval
