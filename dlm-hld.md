# Data Lifecycle Management (DLM) — High-Level Design Document v2

## 1 Terminology

| Term | Definition |
|---|---|
| Archival | Systematic process of collecting, storing, and preserving database records long-term in cost-effective storage. |
| Entity | A database table or group of related tables representing a business object (e.g., Order, Shipment). |
| Primary database | The database actively used for service read-write traffic. |
| Purging | Permanently deleting data from the primary database after successful archival or when retention is not required. |
| Archival Store | Target storage system (e.g., GCS, FDP) where archived data is persisted. |
| Plugin | A datastore-specific component implementing archival execution logic (Extract, Transfer, Purge). |
| Control Plane | Services managing archival configuration, scheduling, policies, and visibility. |
| Execution Plane | Components responsible for carrying out the archival workflow against a specific datastore. |
| Chunk | A bounded subset of eligible records processed as a single unit of work within an archival run. |
| Watcher | Component monitoring source database health and feeding signals to throttle or pause archival. |

---

## 2 Objective

This document presents a technological solution for data archival across diverse databases on PaaS. It captures functional and non-functional requirements, includes a case study of archival tools at Flipkart, provides a detailed design including architecture, data models, API contracts, security, failure handling, and observability, and outlines a phased rollout strategy.

---

## 3 Context and Background

### 3.1 Datastores at Flipkart
Various business entities (Order, Payment, Invoice, Shipment) use SQL (MySQL/TiDB) or NoSQL (HBase/Yak) databases depending on data model requirements.

### 3.2 Increasing Scale Requirements
Growing storage requirements create challenges in scaling databases and impact performance. Teams using MySQL are reaching vertical scalability limits.

### 3.3 Entity Lifecycle
Business entities have defined lifecycles. After reaching terminal state + a business-defined time T, entities can no longer be modified and can be safely removed.

### 3.4 Requirement to Store Data
- **Legal/Compliance:** Data retained for regulatory reasons.
- **Historical Analysis/Debugging:** Data required for past event analysis or production troubleshooting.

---

## 4 Problem Statement

### 4.1 High Hardware Cost
Teams do vertical scaling in the absence of archival, resulting in higher hardware costs.

### 4.2 High RTO & RPO
Larger databases result in higher RPO and RTO. Regular archival makes the database lean.

### 4.3 Poor Performance
Growing data volumes degrade query processing, response times, and critical business operations.

### 4.4 Operational Challenges
Managing large monolithic databases is complex. Existing archival is non-standardized, manual, and ad-hoc — sometimes causing outages.

### 4.5 Central Compliance, Control, Monitoring
Lack of centralized monitoring leads to gaps in data security, compliance reporting, and auditability.

### 4.6 Vertical Scaling Limits
Teams are hitting VM vertical scaling limits, making further scaling increasingly difficult.

---

## 5 Archival

**Definition:** Archival refers to systematically collecting, storing, and preserving data long-term using the most economical storage, with retrieval capabilities on demand.

### 5.1 Functional Requirements

#### 5.1.1 Automated Archival Processes
Support scheduling and executing archival tasks automatically.

#### 5.1.2 Archival Policies
- **State-based:** `shipment_status = "delivered"`
- **State + Time:** `invoice_status = "completed" AND updated_timestamp >= "30 days"`
- **Time-based:** `updated_timestamp >= "90 days"`
- **Compound:** Multiple predicates via AND/OR operators (e.g., `(status IN ('a','b') AND age > 90d) OR (status = 'c' AND age > 30d)`)

#### 5.1.3 Policy Precedence and Conflict Resolution
- Only one active policy per entity at a time.
- On-demand runs may use a temporary override policy.
- Policy updates take effect from the next scheduled run; in-flight runs use the version active at trigger time.
- Each run records the policy version used.

#### 5.1.4 Trim Down and Archive
- Users define a **field projection** (column whitelist) during onboarding.
- Default: all columns archived if no projection defined.
- Projection stored as versioned entity metadata.
- PK columns and policy-referenced columns are always included.
- Archived output includes a schema manifest per run.

#### 5.1.5 Purging
- Purge after successful archival.
- Support purge-only mode (no archival needed).
- Purge from archival store after retention period expires.

#### 5.1.6 Data Integrity
- Lossless archival: data not removed from primary DB until transfer is verified.
- **Verification:** Row-count match + checksum (SHA-256) comparison between extracted and stored data.
- Mandatory verification step between Archiving and Purging phases.

#### 5.1.7 Related Entities Archival

**In Scope:**
- Related rows in same DB (normalization). Example: `Shipments → ShipmentItems → ShipmentAttributes`
- Depth up to 3 levels (configurable). One-to-many relationships.
- **Archive order:** Children first, then parent. **Purge order:** Children first (avoids FK violations).

**Out of Scope:**
- Related entities across different databases.
- Self-referencing tables (require application-level flattening).
- Many-to-many (must decompose via junction tables).

#### 5.1.8 Retrieval Capabilities
- Locate archived data by entity + date range.
- Retrieval SLA: ≤30 min for datasets up to 10 GB.
- Only users with `archival:retrieve` permission can download.

#### 5.1.9 Audit Trails
- Detailed audit trails of all archival activities.
- Auditable events: job trigger, phase transitions, chunk completions, failures, purge operations, data retrievals, policy changes, entity onboarding/modification.
- Audit log retention: 2 years (configurable).

#### 5.1.10 Access Control and Security
- AuthN via SSO/OIDC tokens.
- RBAC with permissions: `source:manage`, `entity:manage`, `policy:manage`, `job:trigger`, `job:view`, `archival:retrieve`, `audit:view`.
- Entity ownership mapped to appId.

#### 5.1.11 Visibility and Notification
- Visibility via UI and Grafana dashboards.
- Notifications via Slack, Email, PagerDuty for failures, completions (opt-in), and warnings.

### 5.2 Non-Functional Requirements

#### 5.2.1 Data Durability
Archival store provides durable storage. Retention policy applied per entity.

#### 5.2.2 Performance
- Archival must not increase source DB P99 latency by >10%.
- Must not consume >15% of source node I/O capacity.
- Retrieval SLA: ≤30 min for ≤10 GB.

#### 5.2.3 Scalability
- Support ≥500 concurrently onboarded entities in Phase 1.
- Horizontal scaling of orchestrator and plugins.

#### 5.2.4 Reliability
- Job success rate ≥99.5% (excluding source-side outages).
- Resilient to failures with automatic retry.

#### 5.2.5 Compatibility
Extensible to other PaaS databases via plugin architecture.

#### 5.2.6 Resource Utilization
Optimized system resource usage. Minimal storage/processing/network impact on primary DB.

#### 5.2.7 Cost
Cost-effective archival storage without compromising performance.

## 6 Current Landscape of Data Archiving in Flipkart

### 6.1 Teams with Individual Archival Approaches

**Key Issues:**
- **Inconsistent Execution:** Some teams only archive when DB utilization >85%, causing sudden heavy loads and occasional outages.
- **Lack of Observability:** Cron-based jobs fail silently, contributing to unnoticed data growth.

**Benefits of Centralized Solution:** Scalability, reliability, efficiency.

### 6.2 Teams with Concerns but No Solution

**Key Issues:**
- **Unmanaged Data Growth:** Storage costs increase, DB performance degrades.
- **Operational Risks:** Databases approaching capacity limits increase outage risk.
- **Resource Strain:** Manual large-dataset management diverts from core development.

**Benefits of Centralized Solution:** Standardization, reduced operational burden, enhanced stability.

### 6.3 Teams with No Immediate Concerns
Low data volume or new deployments. Centralized solution provides future-proofing and operational efficiency.

### 6.4 Conclusion
A centralized archival approach prepares Flipkart for future scalability, improves reliability, and promotes operational consistency.

---

## 7 Archival Tool Analysis

### 7.1 Internal Tools

#### 7.1.1 SQL Databases (MySQL & TiDB)

| Capability | pt-archiver | go-pepur | Shipping-Script | Retail-Script | Scawager |
|---|---|---|---|---|---|
| Archival | Partial | No | No | No | Yes |
| Purging | Yes | Yes | No | Yes | Yes |
| Observability | No | No | No | No | Yes |
| Configurability | No | Yes | No | No | Yes |
| Teams Using | CL, Shipping | Shipping | Shipping | Retail | APL, Seller |

#### 7.1.2 NoSQL Databases (HBase)
CDC-based replication to archival clusters; no standard tool available.

### 7.2 External Solutions

#### 7.2.1 AWS SDAS
Open-source, deploys on AWS. Archives to S3 via Step Functions + Glue. Supports Oracle, SQL Server, MySQL, PostgreSQL. Offers WORM, retention periods, validation.

#### 7.2.2 Azure Data Factory
Cloud-based integration service for data movement and transformation pipelines.

### 7.3 Conclusion
Scawager meets some requirements but has limitations: no auth, standalone deployment challenges, non-partitioned table gaps, no deep hierarchy support. Comprehensive redesign is necessary.

---

## 8 DLM Solution

### 8.1 Key Design Tenets
- **One-stop Archival Solution:** Centralized platform for all DBaaS databases.
- **Controlled Execution:** Minimal impact on production environments.
- **Seamless Data Retrieval:** Date-range-based retrieval; ID-based retrieval in future phases.
- **Visibility & Monitoring:** Full process visibility and health status.
- **Security First:** AuthN, AuthZ, and encryption built in from day one.

### 8.2 Databases Supported

| Database | Type | Phase |
|---|---|---|
| Altair (MySQL) | SQL | Phase 1 |
| Rigel (TiDB) | SQL | Phase 1 |
| Yak (HBase) | NoSQL | Phase 2 |
| Scorpius (Aerospike) | NoSQL | Phase 2 |

Self-managed databases in subsequent phases.

### 8.3 Core Capabilities & Features

#### 8.3.1 User Interface
- Centralized UI for onboarding, policy setup, monitoring, manual triggers.
- Pause, resume, change schedules dynamically.
- Archival report/calendar view.
- **Dry-run mode:** Simulate a run to preview eligible records without executing.

#### 8.3.2 Archival Policy
- State-based, State+Time, Time-based, and Compound policies.
- **Policy versioning:** Every modification creates a new version. Historical versions retained. Each run tagged with the active policy version. In-flight runs unaffected by updates.

#### 8.3.3 Related Entities Archival
- Related rows in same DB archived together (e.g., `Shipments → ShipmentItems → ShipmentAttributes`).
- Users define relationships explicitly with join columns.
- Supported: one-to-many, up to 3 levels deep (configurable).
- **Archive order:** Children first. **Purge order:** Children first (avoids FK violations).
- Cross-database entities not supported.

#### 8.3.4 Automated Archival Process
- Schedule-based automatic execution.
- Retry mechanism for failed jobs (see Section 11).
- Distributed locking prevents duplicate concurrent jobs per entity.
- Status monitoring via UI.

#### 8.3.5 On-Demand Archival
- Manual trigger via UI or REST API.
- Custom temporary policy override.
- Explicit confirmation required. Purge-only mode requires double confirmation.

#### 8.3.6 Archival Storage
- **Scalability & Durability:** Managed by DLM team.
- **Retention:** Time-based (3/5/7/9 years). Data purged after retention expiry.
- **Cost-effective:** Object storage (GCS Phase 1, FDP evaluated for Phase 2+).
- **Data isolation:** Isolated paths per team/entity (see Section 10.4).
- **Storage format:**
  - SQL datastores: **Parquet** (columnar, compressed, schema-embedded).
  - NoSQL datastores: **JSON** (flexible for schemaless data).
- **Per-run output:**
  1. Data files (Parquet/JSON)
  2. Schema manifest (columns, types, nullable, PK flags)
  3. Run metadata (row count, SHA-256 checksum, policy version, timestamp range, run ID)

#### 8.3.7 Data Retrieval

**Capabilities:**
- Download archived data via UI by navigating to entity + date range.
- Metadata index of archived runs (entity, date range, row count, run ID) is searchable.
- Related parent/child table data retrieved simultaneously.
- Partial retrieval (subset of a run): not supported in Phase 1.
- Row-key-based search: planned for Phase 2 via lightweight PK index.

**Retrieval Flow:**
1. Select entity → Search by date range → Select archival run(s)
2. Initiate async download → Monitor progress → Access via pre-signed URL
3. Downloaded package: data files + schema manifest

**Retrieval SLA:** ≤30 min for ≤10 GB. Larger datasets estimated at onboarding.

**Access Control:** Only users with `archival:retrieve` permission on the entity (see Section 10).

**Queryability (SQL-over-object-storage):** Out of scope for Phase 1. Evaluated for Phase 3 using Presto/Trino.

#### 8.3.8 Purging

- Purge only after successful archival + verification (row count + checksum match).
- Batched DELETE with configurable batch size (default: 1,000) and inter-batch sleep (default: 500ms).
- Watcher dynamically adjusts batch size/sleep based on source health.

**Non-partitioned tables:**
- Batched `DELETE ... WHERE pk IN (...)` using primary key.
- Batch size and sleep configurable per entity.

**Purge-only mode:**
- Entities flagged as purge-only during onboarding.
- Double confirmation in UI. Dry-run preview available.
- Logged in audit trail with `PURGE_ONLY` flag.

**Purge verification:**
- Rows purged must match rows archived per chunk. Mismatches → warning alert + `COMPLETED_WITH_WARNINGS` status.

#### 8.3.9 Trim Down and Archive
- Field projection (column whitelist) defined during onboarding.
- Default: all columns. PK + policy-referenced columns always included.
- Applied as a Transform step between Extract and Load.
- Schema manifest records exact columns/types archived per run.
- Projection versioned alongside entity metadata.

#### 8.3.10 Visibility / Notifications
- UI dashboards + Grafana integration.
- Notification channels: Slack, Email, PagerDuty.
- Events: job failure, completion (opt-in), source unreachable, purge mismatch, watcher throttle/pause, retention expiry warning (30-day advance).

### 8.4 Constraints

#### 8.4.1 Related Entities Across Databases
Not supported. Cross-DB transactional integrity is too complex, and databases may be managed by different teams.

#### 8.4.2 No Restoration Back to DB
DLM does not restore archived data into any database. Data available in standard formats (Parquet/JSON) with schema manifests for teams to re-import with their own tooling.

#### 8.4.3 Archival Policy on Yak (Phase 2)
- **TTL-based archival:** Records archived when TTL expires.
- **Application-driven eligibility:** Application moves eligible records to a designated "archival-ready" ColumnFamily.
- **DLM's role:** Scans archival-ready CF on schedule, extracts, transfers to archival store, deletes.
- **Compaction:** DLM notifies Yak team via webhook post-purge for major compaction (best-effort).

#### 8.4.4 Archival Policy on Scorpius (Phase 2)
- **TTL-based:** Extract records approaching TTL expiry before Aerospike evicts them.
- **Set-based:** Archive entire sets based on time policy on a user-defined timestamp bin.
- **Plugin:** Uses Aerospike Scan API with filter expressions. Purge via `delete` with generation checks.
- **Limitation:** Field-level projection not supported in Phase 2 (schemaless records).

#### 8.4.5 Data Format
- SQL: Parquet. NoSQL: JSON.
- Per-run: data files + schema manifest + run metadata (row count, checksum, policy version).

#### 8.4.6 Schema Evolution
- Each run captures schema at extraction time in the manifest.
- Schema changes don't affect previously archived data (each run is self-describing).
- If a policy-referenced column is dropped, the next run fails with a clear error and alert.
- Schema diffs computed and logged between consecutive runs.

### 8.5 Onboarding Flow

Entity-centered onboarding via DLM Web UI. Each entity individually managed. Normalized entities can be configured under a single entity.

```
Entity: Shipment
  ParentEntity: Shipments
  ChildEntities: ShipmentItems, ShipmentAttributes
  Join: shipments.id = shipment_items.shipment_id
```

#### 8.5.1 Source Onboarding
1. Select appId → Select DBaaS provider → Select cluster.
2. DLM validates connectivity and creates source record.
3. Stored: connection endpoint (read-replica preferred), credentials ref (from secret store), DB type/version.

#### 8.5.2 Entity Onboarding
1. Select source DB → DLM lists databases/schemas → Select database → List tables.
2. Pick root table. Optionally specify related child tables with join columns and hierarchy level.
3. Optionally define field projection.
4. Mark as archive+purge or purge-only.

#### 8.5.3 Policy Onboarding
Configure: policy type, predicate, timestamp column, retention period, schedule (cron), chunk size (default: 10,000), purge batch size (default: 1,000), purge sleep (default: 500ms), notification channels. Policy validated before saving.

### 8.6 Archival Process

Three phases for each archival run:

#### Phase 1: Qualifying
- Identify eligible records based on policy predicate.
- Generate ordered chunk list (PK range start, end, estimated row count).
- Executed on read-only/secondary nodes.

#### Phase 2: Archiving
Per chunk: **Extract → Transform → Load → Verify.**
- Extract from source (secondary node preferred).
- Transform: apply field projection, convert to Parquet/JSON.
- Load to archival store under entity's isolated path.
- Verify: checksum + row count comparison. Failed chunks retried.
- Chunk completion checkpointed to control plane DB.

#### Phase 3: Purging
- Remove archived data from primary DB exclusively.
- Batched deletes. Watcher adjusts speed based on DB health:
  - Healthy: full batch, min sleep.
  - Degraded: reduced batch, increased sleep.
  - Unhealthy: pause; resume on recovery.
- Final verification: total rows archived vs. purged. Mismatches → warning.

#### 8.6.1 Scheduled Archival
Scheduler triggers per preconfigured schedule → job record created → plugin executes three-phase flow.

#### 8.6.2 On-Demand Archival
Manual trigger via UI/API → user reviews/overrides policy → confirms → orchestrator executes. Entity and source must be pre-onboarded.

#### 8.6.3 Atomicity
- Not fully atomic end-to-end across three phases.
- **Atomic at chunk level** within each phase.
- Cross-phase consistency via verification checkpoints.
- Purging Phase only processes verified-archived chunks.
- Incomplete chunks retried in next run.

#### 8.6.4 Application Expectation
Records eligible for archival must not be modified. Application teams ensure this.
- **Best-effort enforcement:** Qualifying Phase records fingerprint. Before Purging Phase, a sample is re-verified. Modifications detected → affected chunks skipped + alert.

### 8.7 Rate Limiting and Quotas

| Limit | Default | Configurable |
|---|---|---|
| Per-entity concurrency | 1 job at a time | No (hard limit) |
| Per-source concurrency | 2 concurrent jobs | Yes |
| Platform-wide concurrency | 20 concurrent jobs | Yes |
| Per-entity daily purge quota | 10M rows / 24h | Yes |
| Archival store write rate | Per-entity cap | Yes |

Jobs exceeding limits are queued (FIFO).

### 8.8 Migration Path from Existing Tools

1. **Assessment:** Review current archival config with migrating team.
2. **Onboarding:** Map existing rules to DLM policy syntax via UI.
3. **Parallel run:** DLM in dry-run mode alongside existing tool for 2–4 weeks.
4. **Cutover:** Disable old tool, enable DLM scheduled archival.
5. **Historical data:** Not migrated into DLM. Remains in team's existing storage.

### 8.9 New Datastore Onboarding to DLM
DLM provides plugins for DBaaS data stores. Other teams can develop plugins and integrate via the Plugin API (see Section 9.3).

### 8.10 Post-Archival Process
Post-archival DB optimization (e.g., `OPTIMIZE TABLE`, compaction) is out of scope. DLM can trigger via post-archival webhook once DBaaS solutions support it.

# DLM HLD v2 — Part 3: Section 9 (Design & Architecture)

---

## 9 Design

### 9.1 High-Level Design Philosophy

- **Each database is capable of performing its own archival** — given input, databases archive records onto an archival store.
- **The platform manages everything other than archival execution** — scheduling, storage, policy, monitoring, security.

### 9.2 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                       ARCHIVAL CONTROL PLANE                         │
│  ┌──────────┐  ┌────────────┐  ┌───────────┐  ┌────────────────┐   │
│  │  Web UI   │  │  REST API  │  │ Scheduler │  │  Notification  │   │
│  │           │──│  Gateway   │──│  Worker   │  │   Service      │   │
│  └──────────┘  └─────┬──────┘  └─────┬─────┘  └───────┬────────┘   │
│                       │               │                 │            │
│  ┌────────────────────┴───────────────┴─────────────────┴────────┐  │
│  │            Control Plane Database (MySQL / TiDB)               │  │
│  │  Sources | Entities | Policies | Jobs | Chunks | Audit | Locks │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │ Job Trigger (Message Queue / gRPC)
┌──────────────────────────────────┴───────────────────────────────────┐
│                      ARCHIVAL EXECUTION PLANE                        │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                   Orchestration Layer                          │   │
│  │  ┌────────────────┐  ┌──────────┐  ┌───────────────────────┐ │   │
│  │  │ Archival Manager│  │ Watcher  │  │   Storage Manager     │ │   │
│  │  │ (State Machine) │──│          │  │ (Bucket/Path Mgmt)    │ │   │
│  │  └───────┬────────┘  └──────────┘  └───────────────────────┘ │   │
│  └──────────┼────────────────────────────────────────────────────┘   │
│             │  Plugin API (gRPC)                                     │
│  ┌──────────┴────────────────────────────────────────────────────┐   │
│  │                      Plugin Layer                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐  │   │
│  │  │  MySQL   │  │   TiDB   │  │  HBase   │  │  Aerospike  │  │   │
│  │  │  Plugin  │  │  Plugin  │  │  Plugin  │  │   Plugin    │  │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └─────────────┘  │   │
│  └───────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
                         ┌─────────┴─────────┐
                         │   Archival Store   │
                         │   (GCS / FDP)      │
                         └───────────────────┘
```

### 9.2.1 Archival Control Plane

#### 9.2.1.1 Web UI
- Entity onboarding, policy definition, scheduling.
- Real-time job status and progress.
- Dry-run simulation.
- Archival calendar/report view.
- Data retrieval / download interface.
- Audit log viewer.

#### 9.2.1.2 REST API

**API Modules:**

| Module | Key Endpoints | Description |
|---|---|---|
| Source Manager | `POST /sources`, `GET /sources/{id}`, `DELETE /sources/{id}` | CRUD on archival sources. Connectivity validation on create. |
| Entity Manager | `POST /entities`, `GET /entities/{id}`, `PUT /entities/{id}`, `GET /entities/{id}/schema` | CRUD on entities. Schema introspection. Relationship + projection config. |
| Policy Manager | `POST /policies`, `GET /policies/{id}`, `PUT /policies/{id}`, `GET /policies/{id}/versions` | CRUD on policies. Versioned. Validation on create/update. |
| Job Manager | `POST /jobs` (trigger), `GET /jobs/{id}`, `GET /jobs?entity_id=X`, `PUT /jobs/{id}/cancel`, `PUT /jobs/{id}/pause`, `PUT /jobs/{id}/resume` | Trigger, query, control jobs. |
| Retrieval Manager | `GET /retrievals?entity_id=X&from=T1&to=T2`, `POST /retrievals/{run_id}/download`, `GET /retrievals/download/{download_id}/status` | List runs, initiate download, track progress. |
| Audit Manager | `GET /audit?entity_id=X&from=T1&to=T2&action=Y&user=Z` | Query audit logs with filters. |
| Health | `GET /health`, `GET /ready` | Liveness and readiness probes. |

All endpoints require AuthN token. AuthZ enforced per operation + target resource (see Section 10).

#### 9.2.1.3 Scheduler Worker

**Design:**
- Deployed as a K8s Deployment (not CronJob) — avoids K8s CronJob reliability issues at scale.
- Polls control plane DB every 60 seconds for due schedules.
- On due schedule:
  1. Acquires distributed lock on entity (prevents duplicate triggers).
  2. Creates job record with status `PENDING`.
  3. Enqueues job trigger to message queue.
  4. Orchestration layer picks up the message.

**Job Queue & Prioritization:**

| Priority | Source | Order |
|---|---|---|
| Critical | On-demand triggers marked urgent | First |
| Normal | Scheduled triggers | FIFO |
| Low | Retry attempts | Last |

Jobs exceeding platform-wide concurrency limit remain queued until capacity frees.

#### 9.2.1.4 Notification Service
- Channels: Slack (webhook), Email, PagerDuty.
- Events: job failure, completion (opt-in), source unreachable, purge mismatch, watcher pause, retention expiry warning.
- Per-entity configuration during onboarding.

---

### 9.2.2 Archival Execution Plane

#### 9.2.2.1 Orchestration Layer

##### 9.2.2.1.1 Archival Manager

Core orchestrator responsibilities:
- Receives job triggers from message queue.
- Loads entity + policy + source config from control plane DB.
- Executes the job state machine (see Section 9.4).
- Dispatches tasks to the plugin via Plugin API.
- Tracks chunk-level progress with checkpoints in control plane DB.
- Receives health signals from Watcher; adjusts execution params.
- Updates job status throughout execution.

**Stateless design:** All state persisted in control plane DB. Pod restart → recovers in-flight jobs from DB → resumes from last checkpoint.

**Horizontal scaling:** Multiple Archival Manager replicas can run concurrently. Each picks up different jobs from the queue. Distributed locking ensures no two managers work on the same entity simultaneously.

##### 9.2.2.1.2 Watcher

Monitors source database health and provides signals to the Archival Manager.

**Monitored Signals:**

| Signal | Collection Method | Applicable DBs |
|---|---|---|
| Replication lag (seconds) | DB-specific API / Prometheus | MySQL, TiDB |
| QPS (queries/sec) | Prometheus | All |
| P99 query latency (ms) | Prometheus | All |
| CPU utilization (%) | Prometheus / Node Exporter | All |
| Disk I/O utilization (%) | Prometheus / Node Exporter | All |
| Memory utilization (%) | Prometheus / Node Exporter | All |
| RegionServer load | HBase JMX metrics | HBase |
| Raft apply log duration | TiKV Prometheus metrics | TiDB |

**Health States:**

| State | Default Thresholds | Archival Action |
|---|---|---|
| **Healthy** | Replication lag <5s, CPU <70%, P99 <200ms, I/O <60% | Full speed: max batch size, min sleep |
| **Degraded** | Replication lag 5–30s, CPU 70–85%, P99 200–500ms, I/O 60–80% | Throttle: batch size halved, sleep doubled |
| **Unhealthy** | Replication lag >30s, CPU >85%, P99 >500ms, I/O >80% | Pause: all archival/purge operations suspended |

Thresholds are configurable per source during source onboarding.

**Communication:** Watcher runs as a sidecar or co-located service. Pushes health state updates to the Archival Manager via an in-process event bus (if co-located) or gRPC stream (if separate).

**Reaction Time:** Health state is evaluated every 10 seconds. State transitions are logged and emit metrics. Transition to `Unhealthy` also triggers a notification.

##### 9.2.2.1.3 Storage Manager

Manages the archival store (GCS/FDP):
- **Bucket management:** Creates and manages buckets per environment/zone.
- **Path structure:**
  ```
  gs://<bucket>/<env>/<app_id>/<entity_id>/<run_id>/
    ├── data/
    │   ├── <table_name>_chunk_001.parquet
    │   ├── <table_name>_chunk_002.parquet
    │   └── ...
    ├── schema_manifest.json
    └── run_metadata.json
  ```
- **Lifecycle policies:** Configures GCS object lifecycle rules based on entity retention period. Objects auto-deleted after retention expiry.
- **Encryption:** Configures encryption-at-rest (GCS default or customer-managed keys).
- **Access:** Generates time-limited pre-signed URLs for data retrieval downloads.
- **Abstraction:** Storage interface is abstracted from plugins. Changing GCS → FDP requires only updating the Storage Manager implementation, not any plugin.

##### 9.2.2.1.4 Visibility / Metrics

**Key Metrics Emitted:**

| Metric | Type | Description |
|---|---|---|
| `dlm_job_duration_seconds` | Histogram | Total job duration by entity, phase |
| `dlm_rows_archived_total` | Counter | Rows archived by entity |
| `dlm_rows_purged_total` | Counter | Rows purged by entity |
| `dlm_chunks_processed_total` | Counter | Chunks processed by entity, phase, status |
| `dlm_job_status` | Gauge | Current job status by entity (0=idle, 1=qualifying, 2=archiving, 3=purging) |
| `dlm_watcher_health_state` | Gauge | Source health state (0=healthy, 1=degraded, 2=unhealthy) |
| `dlm_purge_batch_latency_ms` | Histogram | Per-batch purge latency |
| `dlm_archive_store_write_latency_ms` | Histogram | Write latency to archival store |
| `dlm_checksum_mismatches_total` | Counter | Checksum verification failures |
| `dlm_job_failures_total` | Counter | Job failures by entity, failure_type |
| `dlm_queue_depth` | Gauge | Number of jobs waiting in queue |

**SLIs / SLOs:**

| SLI | SLO |
|---|---|
| Job success rate | ≥ 99.5% (monthly) |
| Source DB P99 latency impact during archival | ≤ 10% increase |
| Retrieval availability (download initiation) | ≤ 30 min for ≤ 10 GB |
| Notification delivery latency | ≤ 5 min from event |
| Control plane API availability | ≥ 99.9% |

##### 9.2.2.1.5 Control Plane Datastore

Stores all metadata: sources, entities, policies, jobs, chunks, audit logs, locks.

Technology: MySQL or TiDB (leveraging existing DBaaS infrastructure).

---

### 9.2.2.2 Plugin Layer (Execution)

The plugin layer contains datastore-specific archival logic.

**Responsibilities:**
- **Extract:** Identify and read candidates from source DB.
- **Transform:** Apply field projection, convert to target format.
- **Transfer:** Write to archival store (via Storage Manager abstraction).
- **Purge:** Remove archived records from source DB.
- **CQRS:** Read from replica, write/delete on primary.
- **Idempotency:** Retries produce same output (via chunk-level dedup using run_id + chunk_id).
- **Integrity Check:** Cross-validate archived data (checksum + row count).
- **Concurrency Lock:** Distributed lock per entity prevents parallel executions.
- **Reporting:** Report chunk-level and job-level success/failure to orchestrator.

---

### 9.3 Plugin API Contract

The Plugin API is the interface between the Orchestration Layer and each datastore Plugin. It is defined as a **gRPC service**.

#### 9.3.1 gRPC Service Definition

```protobuf
syntax = "proto3";
package dlm.plugin.v1;

service ArchivalPlugin {
  // Lifecycle
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);
  rpc GetCapabilities(GetCapabilitiesRequest) returns (GetCapabilitiesResponse);

  // Qualifying Phase
  rpc Qualify(QualifyRequest) returns (QualifyResponse);

  // Archiving Phase
  rpc ArchiveChunk(ArchiveChunkRequest) returns (ArchiveChunkResponse);

  // Purging Phase
  rpc PurgeChunk(PurgeChunkRequest) returns (PurgeChunkResponse);

  // Verification
  rpc VerifyChunk(VerifyChunkRequest) returns (VerifyChunkResponse);

  // Introspection
  rpc ListDatabases(ListDatabasesRequest) returns (ListDatabasesResponse);
  rpc ListTables(ListTablesRequest) returns (ListTablesResponse);
  rpc DescribeTable(DescribeTableRequest) returns (DescribeTableResponse);
}

// --- Lifecycle ---

message HealthCheckRequest {}
message HealthCheckResponse {
  enum Status { SERVING = 0; NOT_SERVING = 1; }
  Status status = 1;
  string message = 2;
}

message GetCapabilitiesRequest {}
message GetCapabilitiesResponse {
  string plugin_name = 1;          // e.g. "mysql", "tidb", "hbase"
  string plugin_version = 2;       // semver
  repeated string db_versions = 3; // supported DB versions
  bool supports_field_projection = 4;
  bool supports_compound_policy = 5;
  bool supports_checksum_verification = 6;
  int32 max_chunk_size = 7;        // max rows per chunk
}

// --- Qualifying Phase ---

message QualifyRequest {
  string run_id = 1;
  SourceConfig source = 2;
  EntityConfig entity = 3;
  PolicyConfig policy = 4;
  int32 chunk_size = 5;
}

message QualifyResponse {
  repeated ChunkDefinition chunks = 1;
  int64 total_eligible_rows = 2;
}

message ChunkDefinition {
  string chunk_id = 1;
  string pk_range_start = 2;  // inclusive
  string pk_range_end = 3;    // exclusive
  int64 estimated_row_count = 4;
}

// --- Archiving Phase ---

message ArchiveChunkRequest {
  string run_id = 1;
  string chunk_id = 2;
  SourceConfig source = 3;
  EntityConfig entity = 4;
  PolicyConfig policy = 5;
  StorageTarget storage_target = 6; // path + credentials
  FieldProjection projection = 7;   // optional
}

message ArchiveChunkResponse {
  string chunk_id = 1;
  ChunkStatus status = 2;
  int64 rows_archived = 3;
  string checksum = 4;       // SHA-256 of archived data
  string storage_path = 5;   // path where data was written
  string error_message = 6;  // populated on failure
}

// --- Purging Phase ---

message PurgeChunkRequest {
  string run_id = 1;
  string chunk_id = 2;
  SourceConfig source = 3;
  EntityConfig entity = 4;
  PurgeConfig purge_config = 5;
}

message PurgeChunkResponse {
  string chunk_id = 1;
  ChunkStatus status = 2;
  int64 rows_purged = 3;
  string error_message = 4;
}

// --- Verification ---

message VerifyChunkRequest {
  string run_id = 1;
  string chunk_id = 2;
  SourceConfig source = 3;
  EntityConfig entity = 4;
  int64 expected_row_count = 5;
  string expected_checksum = 6;
  string storage_path = 7;
}

message VerifyChunkResponse {
  string chunk_id = 1;
  bool verified = 2;
  int64 actual_row_count = 3;
  string actual_checksum = 4;
  string mismatch_detail = 5; // populated if verified=false
}

// --- Introspection ---

message ListDatabasesRequest { SourceConfig source = 1; }
message ListDatabasesResponse { repeated string databases = 1; }

message ListTablesRequest { SourceConfig source = 1; string database = 2; }
message ListTablesResponse { repeated string tables = 1; }

message DescribeTableRequest { SourceConfig source = 1; string database = 2; string table = 3; }
message DescribeTableResponse { repeated ColumnInfo columns = 1; repeated string primary_key_columns = 2; }

message ColumnInfo {
  string name = 1;
  string type = 2;
  bool nullable = 3;
  bool is_primary_key = 4;
  string default_value = 5;
}

// --- Common Types ---

enum ChunkStatus {
  SUCCESS = 0;
  FAILED = 1;
  SKIPPED = 2; // e.g., already processed (idempotency)
}

message SourceConfig {
  string source_id = 1;
  string host = 2;
  int32 port = 3;
  string read_replica_host = 4;
  int32 read_replica_port = 5;
  string credentials_ref = 6; // reference to secret store
  string db_type = 7;         // mysql, tidb, hbase, aerospike
}

message EntityConfig {
  string entity_id = 1;
  string database = 2;
  string root_table = 3;
  repeated ChildTableConfig child_tables = 4;
}

message ChildTableConfig {
  string table_name = 1;
  string join_column_parent = 2;
  string join_column_child = 3;
  int32 hierarchy_level = 4;
  repeated ChildTableConfig children = 5; // nested children
}

message PolicyConfig {
  string policy_id = 1;
  int32 policy_version = 2;
  string predicate = 3;        // SQL-like WHERE clause
  string timestamp_column = 4;
}

message PurgeConfig {
  int32 batch_size = 1;        // rows per DELETE batch
  int32 inter_batch_sleep_ms = 2;
  bool purge_only = 3;         // true if no archival was done
}

message StorageTarget {
  string base_path = 1;
  string credentials_ref = 2;
  string format = 3; // "parquet" or "json"
}

message FieldProjection {
  repeated string include_columns = 1; // empty = all columns
}
```

#### 9.3.2 Plugin Lifecycle

1. **Registration:** Plugin registers with the control plane at startup (name, version, capabilities).
2. **Health Check:** Orchestrator periodically calls `HealthCheck`. Unhealthy plugins are excluded from job dispatch.
3. **Graceful Shutdown:** On SIGTERM, plugin finishes current chunk, reports status, then exits. In-progress job is checkpointed; the next manager instance resumes.

#### 9.3.3 Plugin Development Guidelines

For teams building custom plugins:
- Implement the `ArchivalPlugin` gRPC service.
- Use the Storage Manager's pre-signed URLs or SDK (provided by DLM) for writing to archival store.
- Follow chunk-level idempotency: use `run_id + chunk_id` as dedup key.
- Report errors with structured error codes (defined in a shared error catalog).
- Emit Prometheus metrics using DLM's metric naming conventions.
- Include integration tests against a test instance of the target DB type.

---

### 9.4 Job State Machine

Every archival job follows a well-defined state machine. State transitions are persisted in the control plane DB and are the basis for failure recovery.

```
                         ┌──────────┐
                         │ PENDING  │
                         └────┬─────┘
                              │ Picked up by Archival Manager
                              ▼
                     ┌────────────────┐
                     │  QUALIFYING    │
                     └───┬────────┬───┘
                         │        │
                    Success    Failure
                         │        │
                         ▼        ▼
              ┌──────────────┐  ┌──────────────────┐
              │  ARCHIVING   │  │ QUALIFYING_FAILED │──→ (Retry or FAILED)
              └──┬───────┬───┘  └──────────────────┘
                 │       │
            Success   Failure
                 │       │
                 ▼       ▼
          ┌──────────┐  ┌──────────────────┐
          │ VERIFYING │  │ ARCHIVING_FAILED │──→ (Retry or FAILED)
          └──┬────┬──┘  └──────────────────┘
             │    │
        Pass │    │ Fail
             ▼    ▼
      ┌──────────┐  ┌───────────────────┐
      │ PURGING  │  │ VERIFICATION_FAILED│──→ (Retry or FAILED)
      └──┬────┬──┘  └───────────────────┘
         │    │
    Success  Failure
         │    │
         ▼    ▼
  ┌───────────┐  ┌──────────────┐
  │ COMPLETED │  │ PURGE_FAILED │──→ (Retry or FAILED)
  └───────────┘  └──────────────┘

  Special States:
  ┌──────────────────────┐    ┌────────────┐
  │ COMPLETED_WITH_WARNS │    │  CANCELLED │
  └──────────────────────┘    └────────────┘
  ┌──────────┐    ┌──────────┐
  │  PAUSED  │───→│ RESUMED  │──→ (returns to phase where paused)
  └──────────┘    └──────────┘
```

**State Definitions:**

| State | Description |
|---|---|
| `PENDING` | Job created, waiting in queue. |
| `QUALIFYING` | Identifying eligible records and generating chunks. |
| `QUALIFYING_FAILED` | Qualifying phase failed. Eligible for retry. |
| `ARCHIVING` | Extracting + transforming + loading chunks to archival store. |
| `ARCHIVING_FAILED` | One or more chunks failed archiving after max retries. |
| `VERIFYING` | Verifying checksums and row counts of archived chunks. |
| `VERIFICATION_FAILED` | Verification failed for one or more chunks. |
| `PURGING` | Deleting archived data from primary DB. |
| `PURGE_FAILED` | Purge failed for one or more chunks. |
| `COMPLETED` | All phases completed successfully. |
| `COMPLETED_WITH_WARNINGS` | Completed but with non-critical issues (e.g., purge count mismatch). |
| `FAILED` | Job permanently failed after exhausting retries. Requires manual investigation. |
| `PAUSED` | Job paused by user or Watcher (unhealthy source). |
| `CANCELLED` | Job cancelled by user. |

**Chunk-Level Tracking:**

Each chunk within a job has its own status:

| Chunk State | Description |
|---|---|
| `PENDING` | Not yet processed. |
| `ARCHIVED` | Successfully archived and checksum verified. |
| `ARCHIVE_FAILED` | Archive attempt failed. |
| `PURGED` | Successfully purged from source. |
| `PURGE_FAILED` | Purge attempt failed. |
| `SKIPPED` | Skipped (e.g., modification detected in pre-purge verification). |

On job retry, only chunks not in `ARCHIVED`/`PURGED` terminal states are reprocessed.

---

### 9.5 Data Model

#### 9.5.1 Entity-Relationship Overview

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Source   │──1:N│  Entity  │──1:1│  Policy  │
└──────────┘     └────┬─────┘     └──────────┘
                      │ 1:N              │ versioned
                 ┌────┴─────┐     ┌──────┴─────┐
                 │ ChildTable│     │PolicyVersion│
                 └──────────┘     └────────────┘
                      │
                      │ 1:N
                 ┌────┴─────┐     ┌──────────┐
                 │   Job    │──1:N│  Chunk   │
                 └────┬─────┘     └──────────┘
                      │ 1:N
                 ┌────┴─────┐
                 │ AuditLog │
                 └──────────┘
```

#### 9.5.2 Table Definitions

**`sources`**

| Column | Type | Description |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID |
| `app_id` | VARCHAR(128) | Application identifier |
| `db_type` | ENUM('mysql','tidb','hbase','aerospike') | Datastore type |
| `db_version` | VARCHAR(32) | Database version |
| `primary_host` | VARCHAR(256) | Primary DB endpoint |
| `primary_port` | INT | Primary DB port |
| `replica_host` | VARCHAR(256) | Read replica endpoint |
| `replica_port` | INT | Read replica port |
| `credentials_ref` | VARCHAR(256) | Secret store reference |
| `watcher_config` | JSON | Health thresholds override |
| `status` | ENUM('active','inactive') | |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |
| `created_by` | VARCHAR(128) | |

**`entities`**

| Column | Type | Description |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID |
| `source_id` | VARCHAR(36) FK | Reference to source |
| `database_name` | VARCHAR(128) | Schema/database name |
| `root_table` | VARCHAR(128) | Root table name |
| `entity_type` | ENUM('archive_purge','purge_only') | |
| `field_projection` | JSON | Column whitelist (null = all) |
| `schema_version` | INT | Incremented on schema change detection |
| `current_schema` | JSON | Last observed schema |
| `notification_config` | JSON | Slack/email/PD channels |
| `status` | ENUM('active','inactive','onboarding') | |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |
| `created_by` | VARCHAR(128) | |

**`child_tables`**

| Column | Type | Description |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID |
| `entity_id` | VARCHAR(36) FK | Reference to entity |
| `table_name` | VARCHAR(128) | Child table name |
| `parent_table` | VARCHAR(128) | Parent table (root or another child) |
| `join_column_parent` | VARCHAR(128) | Join column on parent |
| `join_column_child` | VARCHAR(128) | Join column on child |
| `hierarchy_level` | INT | 1 = direct child of root, 2 = grandchild, etc. |

**`policies`**

| Column | Type | Description |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID |
| `entity_id` | VARCHAR(36) FK UNIQUE | One active policy per entity |
| `current_version` | INT | Latest version number |
| `status` | ENUM('active','inactive') | |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

**`policy_versions`**

| Column | Type | Description |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID |
| `policy_id` | VARCHAR(36) FK | Reference to policy |
| `version` | INT | Version number |
| `policy_type` | ENUM('state','time','state_time','compound') | |
| `predicate` | TEXT | SQL-like eligibility expression |
| `timestamp_column` | VARCHAR(128) | Column used for time filtering |
| `retention_years` | INT | Years to retain in archival store |
| `schedule_cron` | VARCHAR(64) | Cron expression |
| `chunk_size` | INT | Rows per chunk (default 10000) |
| `purge_batch_size` | INT | Rows per DELETE (default 1000) |
| `purge_sleep_ms` | INT | Sleep between batches (default 500) |
| `created_at` | TIMESTAMP | |
| `created_by` | VARCHAR(128) | |

**`jobs`**

| Column | Type | Description |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID (= run_id) |
| `entity_id` | VARCHAR(36) FK | |
| `policy_version_id` | VARCHAR(36) FK | Policy version active at trigger |
| `trigger_type` | ENUM('scheduled','on_demand') | |
| `triggered_by` | VARCHAR(128) | User or "scheduler" |
| `status` | ENUM (see state machine) | Current job state |
| `phase` | ENUM('qualifying','archiving','verifying','purging','done') | Current phase |
| `total_chunks` | INT | |
| `completed_chunks` | INT | |
| `total_rows_archived` | BIGINT | |
| `total_rows_purged` | BIGINT | |
| `checksum_mismatches` | INT | |
| `retry_count` | INT | |
| `error_message` | TEXT | Last error |
| `started_at` | TIMESTAMP | |
| `completed_at` | TIMESTAMP | |
| `created_at` | TIMESTAMP | |

**`chunks`**

| Column | Type | Description |
|---|---|---|
| `id` | VARCHAR(36) PK | UUID (= chunk_id) |
| `job_id` | VARCHAR(36) FK | |
| `chunk_index` | INT | Ordered position |
| `pk_range_start` | VARCHAR(256) | Inclusive PK boundary |
| `pk_range_end` | VARCHAR(256) | Exclusive PK boundary |
| `estimated_rows` | BIGINT | From qualifying |
| `actual_rows_archived` | BIGINT | |
| `actual_rows_purged` | BIGINT | |
| `archive_checksum` | VARCHAR(128) | SHA-256 |
| `storage_path` | VARCHAR(512) | Path in archival store |
| `status` | ENUM('pending','archived','archive_failed','purged','purge_failed','skipped') | |
| `error_message` | TEXT | |
| `archived_at` | TIMESTAMP | |
| `purged_at` | TIMESTAMP | |

**`audit_logs`**

| Column | Type | Description |
|---|---|---|
| `id` | BIGINT PK AUTO_INCREMENT | |
| `entity_id` | VARCHAR(36) | Nullable (some events are global) |
| `job_id` | VARCHAR(36) | Nullable |
| `action` | ENUM('source_created','source_updated','entity_created','entity_updated','policy_created','policy_updated','job_triggered','job_completed','job_failed','job_cancelled','job_paused','job_resumed','chunk_archived','chunk_purged','chunk_failed','data_retrieved','schema_change_detected') | |
| `actor` | VARCHAR(128) | User or "system" |
| `details` | JSON | Action-specific metadata |
| `created_at` | TIMESTAMP | |

Index: `(entity_id, created_at)`, `(action, created_at)`, `(actor, created_at)`.

Retention: 2 years (configurable). Older records archived or purged.

**`distributed_locks`**

| Column | Type | Description |
|---|---|---|
| `lock_key` | VARCHAR(256) PK | e.g., `entity:<entity_id>` |
| `owner` | VARCHAR(128) | Manager pod ID |
| `acquired_at` | TIMESTAMP | |
| `expires_at` | TIMESTAMP | TTL-based auto-release |

---

### 9.6 Choice of Archival Store

| Criterion | GCS | FDP |
|---|---|---|
| Connectors | Hadoop, Spark, MR, Storm | Similar breadth |
| Lifecycle policies | Supported (timeline-based) | Supported; abstracts storage migration |
| Encryption at rest | Default or CMEK | Built-in + masking |
| Deep archival | Yes (low write cost) | Cost TBD |
| Operational ownership | DLM team manages | FDP team manages storage |
| Migration ease | DLM must manage | FDP abstracts storage backend changes |

**Decision:** GCS for Phase 1. FDP evaluated in parallel for Phase 2+. The Storage Manager (Section 9.2.2.1.3) abstracts the backend, so switching requires no plugin changes.

### 9.7 Extensibility

- Each datastore has its own plugin implementing the gRPC `ArchivalPlugin` service.
- Within a datastore, different archival strategies (partitioned vs. non-partitioned, MR-based vs. direct) can be implemented as configurable modes within the plugin.
- New datastores are onboarded by:
  1. Implementing the `ArchivalPlugin` gRPC service.
  2. Registering with the control plane.
  3. Publishing the plugin as a K8s Deployment.
- Plugin development guidelines and a reference implementation (MySQL plugin) will be provided.

---

## 10 Security Design

### 10.1 Authentication (AuthN)

All access to DLM — UI, REST API, and plugin registration — requires authentication.

**Mechanism:** SSO / OIDC token-based authentication, integrated with Flipkart's existing identity provider.

| Access Point | AuthN Method |
|---|---|
| Web UI | SSO redirect → OIDC token |
| REST API | Bearer token (OIDC JWT) in `Authorization` header |
| Plugin ↔ Orchestrator (gRPC) | mTLS (mutual TLS) with certificate-based identity |
| Archival Store (GCS) | Workload Identity Federation (WIF) service accounts |

- Tokens are validated on every request. Expired/invalid tokens return `401 Unauthorized`.
- Service-to-service calls (orchestrator ↔ plugin, orchestrator ↔ storage) use mTLS or WIF — no user credentials are passed to plugins.

### 10.2 Authorization (AuthZ)

DLM uses **Role-Based Access Control (RBAC)** with permissions scoped to specific resources.

#### 10.2.1 Roles

| Role | Scope | Description |
|---|---|---|
| `DLM_ADMIN` | Platform-wide | Full access. Manage all sources, entities, policies, jobs. View all audit logs. |
| `ENTITY_OWNER` | Per entity (mapped via appId) | Full access to entities under their appId. Trigger jobs, manage policies, retrieve data. |
| `ENTITY_VIEWER` | Per entity | Read-only access. View job status, view policies. Cannot trigger or modify. |
| `PLUGIN_SERVICE` | Per plugin | Service account for plugin registration and gRPC communication. |

#### 10.2.2 Permissions Matrix

| Permission | DLM_ADMIN | ENTITY_OWNER | ENTITY_VIEWER | PLUGIN_SERVICE |
|---|---|---|---|---|
| `source:create` | Yes | Yes (own appId) | No | No |
| `source:read` | Yes | Yes (own appId) | Yes (own appId) | No |
| `source:update` | Yes | Yes (own appId) | No | No |
| `source:delete` | Yes | No | No | No |
| `entity:create` | Yes | Yes (own appId) | No | No |
| `entity:read` | Yes | Yes (own appId) | Yes (own appId) | No |
| `entity:update` | Yes | Yes (own appId) | No | No |
| `policy:create` | Yes | Yes (own appId) | No | No |
| `policy:read` | Yes | Yes (own appId) | Yes (own appId) | No |
| `policy:update` | Yes | Yes (own appId) | No | No |
| `job:trigger` | Yes | Yes (own appId) | No | No |
| `job:read` | Yes | Yes (own appId) | Yes (own appId) | No |
| `job:cancel` | Yes | Yes (own appId) | No | No |
| `job:pause_resume` | Yes | Yes (own appId) | No | No |
| `archival:retrieve` | Yes | Yes (own appId) | No | No |
| `audit:read` | Yes | Yes (own appId) | Yes (own appId) | No |
| `plugin:register` | Yes | No | No | Yes |
| `plugin:health` | Yes | No | No | Yes |

#### 10.2.3 Ownership Mapping
- Entity ownership is determined by `appId`. The `appId` is set during source onboarding.
- Users associated with an `appId` (via Flipkart's identity/team management system) automatically receive `ENTITY_OWNER` role for all entities under that appId.
- `DLM_ADMIN` is assigned to the DLM platform team.

### 10.3 Encryption

| Layer | Method |
|---|---|
| **In transit** (UI ↔ API) | TLS 1.2+ (HTTPS) |
| **In transit** (API ↔ Plugin) | mTLS |
| **In transit** (Plugin ↔ Source DB) | TLS (where supported by DB) |
| **In transit** (Orchestrator ↔ Archival Store) | TLS |
| **At rest** (Archival Store — GCS) | AES-256 (Google-managed or CMEK) |
| **At rest** (Control Plane DB) | Encryption at rest via DBaaS provider |
| **Credentials** | Stored in a secret store (e.g., Vault). Never in control plane DB or logs. |

### 10.4 Multi-Tenant Data Isolation

Archived data for different teams/entities is isolated in the archival store:

```
gs://<bucket>/<env>/
  └── <app_id>/                    # Team-level isolation
       └── <entity_id>/            # Entity-level isolation
            └── <run_id>/          # Run-level isolation
                 ├── data/
                 ├── schema_manifest.json
                 └── run_metadata.json
```

- **Bucket-level:** One bucket per environment (prod, staging).
- **Path-level:** Each appId gets its own prefix. GCS IAM policies restrict access to the prefix.
- **Download access:** Pre-signed URLs are scoped to the specific run path and expire after a configurable TTL (default: 1 hour).
- **Cross-tenant access:** Not possible. API layer enforces appId-based filtering before generating storage paths or download URLs.

### 10.5 Credential Management

- Source database credentials are stored in a secret store (e.g., HashiCorp Vault, K8s Secrets).
- DLM stores only a `credentials_ref` (reference/path to the secret), never the credential itself.
- Plugins retrieve credentials at runtime from the secret store using the `credentials_ref`.
- WIF service accounts are used for GCS access — no long-lived keys.
- Secret rotation: When credentials are rotated in the secret store, DLM picks up the new value on the next job execution (no restart required).

### 10.6 Threat Model Summary

| Threat | Mitigation |
|---|---|
| Unauthorized access to archived data | RBAC + appId scoping + pre-signed URL expiry |
| Credential leakage | Secret store + credentials_ref (never stored in DB/logs) |
| Data exfiltration during transit | TLS / mTLS on all communication paths |
| Privilege escalation | Role hierarchy enforced at API layer. Plugin service accounts cannot access control plane APIs. |
| Tampered archival data | SHA-256 checksums verified post-write. Archival store has object versioning enabled. |
| Rogue plugin | mTLS identity verification. Plugins can only write to their assigned storage paths. |
| Audit log tampering | Audit logs are append-only. `DLM_ADMIN` cannot delete audit records (enforced at DB level). |

---

## 11 Failure Handling & Recovery

### 11.1 Failure Taxonomy

| Category | Examples | Handling |
|---|---|---|
| **Transient** | Network timeout, DB connection reset, GCS temporary unavailability, pod eviction | Auto-retry with exponential backoff |
| **Permanent (data)** | Policy-referenced column dropped, table not found, schema mismatch | Fail job, alert entity owner, require manual fix |
| **Permanent (infra)** | Source DB decommissioned, credentials revoked, archival store bucket deleted | Fail job, alert DLM admin + entity owner |
| **Partial** | Some chunks succeed, others fail | Checkpoint successful chunks, retry only failed chunks |
| **Health-triggered** | Source DB unhealthy (via Watcher) | Pause job, auto-resume when health recovers (with timeout) |

### 11.2 Retry Policy

| Parameter | Default | Configurable |
|---|---|---|
| Max retries per chunk | 3 | Yes (per entity) |
| Max retries per job (across all chunks) | 5 | Yes (per entity) |
| Backoff strategy | Exponential with jitter | No |
| Initial backoff | 30 seconds | Yes |
| Max backoff | 10 minutes | Yes |
| Retry window | Within the same job execution | — |

**Retry behavior by phase:**

| Phase | Retry Scope | What is retried |
|---|---|---|
| Qualifying | Entire phase | Full qualifying query re-executed |
| Archiving | Per chunk | Only failed chunks. Successful chunks are skipped (idempotency via run_id + chunk_id). |
| Verification | Per chunk | Only failed chunks. |
| Purging | Per chunk | Only failed chunks. Already-purged chunks are skipped. |

**After max retries exhausted:**
- Job transitions to `FAILED` state.
- Notification sent to entity owner and DLM admin.
- Audit log entry created with full error details.
- Next scheduled run will start fresh (new qualifying phase), but previously archived + purged chunks from the failed run are not re-processed.

### 11.3 Pause and Resume

**Triggers for pause:**
- **User-initiated:** Via UI or API (`PUT /jobs/{id}/pause`).
- **Watcher-initiated:** Source health transitions to `Unhealthy`.
- **Platform-initiated:** Platform-wide concurrency limit exceeded (job re-queued).

**Pause behavior:**
- Current in-progress chunk is allowed to complete (no mid-chunk interruption).
- Job transitions to `PAUSED` state.
- Checkpoint is written with the last completed chunk.

**Resume behavior:**
- **User-initiated:** Via UI or API.
- **Watcher-initiated:** Auto-resume when source health returns to `Healthy` or `Degraded`.
- **Auto-resume timeout:** If paused for >2 hours (configurable), job transitions to `FAILED` with a timeout error.
- On resume, execution continues from the last checkpointed chunk.

### 11.4 Reconciliation

If the system detects an inconsistency (e.g., data archived but purge status unknown due to crash):

1. **Detection:** On job recovery after crash, the Archival Manager checks each chunk's status in the control plane DB.
2. **Archived but not purged:** These chunks are eligible for purging. The Purging Phase picks them up.
3. **Archive status unknown (no checkpoint):** The chunk is re-archived. Idempotency ensures no duplicates — the plugin checks if data already exists at the target storage path with matching checksum. If it does, the chunk is marked `ARCHIVED` without re-writing.
4. **Purged but archive status unknown:** This should never happen (purge only runs after verified archive). If detected, an alert is raised for manual investigation.

### 11.5 Dead-Letter Queue

Jobs that fail permanently (after max retries) are moved to a dead-letter state:
- Visible in the UI under a "Failed Jobs" tab.
- DLM admin can inspect, manually retry, or permanently dismiss.
- Failed jobs older than 30 days without action are auto-dismissed with a final notification.

### 11.6 DLM Platform Disaster Recovery

| Component | DR Strategy |
|---|---|
| Control Plane DB | Standard DBaaS replication (Altair/Rigel). RPO < 1 min, RTO < 15 min. |
| Archival Store (GCS) | GCS provides 99.999999999% durability. Multi-region buckets if required. |
| Orchestrator / Scheduler pods | Stateless K8s Deployments. Auto-rescheduled on pod failure. State recovered from DB. |
| Plugin pods | Stateless. Auto-rescheduled. In-flight chunk is retried on recovery. |
| Message queue | Durable message queue (e.g., Kafka with replication). Unprocessed messages survive broker failure. |

---

## 12 Observability

### 12.1 Metrics

All DLM components emit Prometheus metrics. Key metrics:

| Metric | Type | Labels | Description |
|---|---|---|---|
| `dlm_job_duration_seconds` | Histogram | entity_id, phase, status | Job/phase duration |
| `dlm_rows_archived_total` | Counter | entity_id, db_type | Total rows archived |
| `dlm_rows_purged_total` | Counter | entity_id, db_type | Total rows purged |
| `dlm_chunks_processed_total` | Counter | entity_id, phase, status | Chunks processed |
| `dlm_job_status` | Gauge | entity_id | Current phase (0=idle..3=purging) |
| `dlm_watcher_health_state` | Gauge | source_id | Source health (0/1/2) |
| `dlm_purge_batch_latency_ms` | Histogram | entity_id | Per-batch purge latency |
| `dlm_archive_store_write_latency_ms` | Histogram | entity_id | Write latency to store |
| `dlm_checksum_mismatches_total` | Counter | entity_id | Verification failures |
| `dlm_job_failures_total` | Counter | entity_id, failure_type | Job failures |
| `dlm_queue_depth` | Gauge | priority | Jobs waiting in queue |
| `dlm_source_replication_lag_seconds` | Gauge | source_id | Source replication lag |
| `dlm_api_request_duration_seconds` | Histogram | endpoint, method, status | API latency |
| `dlm_api_request_total` | Counter | endpoint, method, status | API request count |

### 12.2 Dashboards (Grafana)

| Dashboard | Panels |
|---|---|
| **Platform Overview** | Active jobs, queue depth, job success rate (24h), total rows archived/purged (24h), failure count |
| **Entity Detail** | Job history timeline, rows archived/purged per run, run duration trend, next scheduled run, current Watcher state |
| **Source Health** | Replication lag, QPS, P99 latency, CPU/memory/disk, Watcher health state history |
| **Plugin Performance** | Chunk processing latency, archive throughput (rows/sec), purge throughput, error rate by plugin type |
| **API Health** | Request rate, latency percentiles, error rate, authentication failures |

### 12.3 Alerting

| Alert | Condition | Severity | Action |
|---|---|---|---|
| `DLMJobFailed` | Job in FAILED state | P2 | Notify entity owner |
| `DLMJobStuck` | Job in non-terminal state for >2x expected duration | P2 | Notify DLM admin |
| `DLMSourceUnhealthy` | Watcher reports Unhealthy for >15 min | P1 | Notify entity owner + DLM admin |
| `DLMQueueBacklog` | Queue depth > 50 for >30 min | P2 | Notify DLM admin |
| `DLMChecksumMismatch` | Any checksum mismatch in last 1h | P1 | Notify entity owner + DLM admin |
| `DLMHighPurgeLatency` | Purge batch P99 > 5s for >10 min | P2 | Notify entity owner |
| `DLMControlPlaneDown` | API health check fails for >5 min | P0 | Notify DLM admin (PagerDuty) |
| `DLMRetentionExpiryApproaching` | Archived data within 30 days of retention expiry | P3 | Notify entity owner |
| `DLMPluginUnhealthy` | Plugin HealthCheck returns NOT_SERVING for >5 min | P1 | Notify DLM admin |

### 12.4 Logging

- **Structured JSON logs** from all components (API, scheduler, orchestrator, plugins).
- Log fields: `timestamp`, `level`, `component`, `job_id`, `entity_id`, `chunk_id`, `message`, `error`.
- Log levels: `DEBUG`, `INFO`, `WARN`, `ERROR`.
- Logs shipped to centralized logging (ELK / Loki) for search and correlation.
- **Sensitive data is never logged** — no row data, no credentials, no PII.

### 12.5 Audit Logs

Separate from operational logs. Stored in the `audit_logs` table (Section 9.5.2).

Auditable events:
- Source/entity/policy CRUD operations
- Job triggers, completions, failures, cancellations, pauses, resumes
- Chunk-level archive and purge completions/failures
- Data retrieval downloads
- Schema change detections
- Watcher state transitions

Each audit entry records: actor (user or system), action, entity, timestamp, and action-specific details (JSON).

Retention: 2 years (configurable per compliance requirements).

---

## 13 Capacity Planning & Sizing

### 13.1 Platform Targets (Phase 1)

| Dimension | Target |
|---|---|
| Total onboarded entities | 500 |
| Concurrent running jobs | 20 |
| Max entity table size | 10 billion rows |
| Throughput per job (archive) | 10,000–50,000 rows/sec (depends on row size and DB) |
| Throughput per job (purge) | 1,000–5,000 rows/sec (throttled by Watcher) |
| Archival store capacity | 100 TB (initial provisioning, expandable) |
| Control plane DB size | ~10 GB (metadata only) |
| Job queue capacity | 1,000 pending jobs |

### 13.2 Component Sizing

| Component | Replicas | CPU | Memory | Notes |
|---|---|---|---|---|
| REST API | 3 | 2 cores | 4 GB | Stateless, behind load balancer |
| Scheduler Worker | 2 (active-passive via leader election) | 1 core | 2 GB | Lightweight polling |
| Archival Manager | 3–5 | 4 cores | 8 GB | Scales with concurrent jobs |
| MySQL Plugin | 2–4 | 4 cores | 8 GB | Scales with concurrent MySQL jobs |
| TiDB Plugin | 2–4 | 4 cores | 8 GB | Scales with concurrent TiDB jobs |
| Watcher | 1 per source (sidecar) or shared | 0.5 core | 1 GB | Lightweight metrics polling |
| Notification Service | 2 | 1 core | 2 GB | Low load |
| Control Plane DB | Per DBaaS standard | — | — | Standard Altair/Rigel cluster |
| Web UI | 2 | 1 core | 2 GB | Static SPA + API proxy |

### 13.3 Archival Store Sizing Estimate

Formula: `Annual archival volume = Σ (entity_rows_per_year × avg_row_size × compression_ratio)`

| Parameter | Typical Value |
|---|---|
| Compression ratio (Parquet) | 3:1 to 10:1 |
| Compression ratio (JSON) | 1.5:1 to 3:1 |
| Avg row size (SQL) | 500 bytes – 2 KB |
| Avg row size (NoSQL) | 1 KB – 10 KB |

Example: 500 entities × 10M rows/year avg × 1 KB avg row × 5:1 compression = **1 TB/year**.

Actual sizing will vary. GCS auto-scales; initial bucket provisioning is a cost decision, not a capacity constraint.

---

## 14 Cross-Region & Multi-Zone Considerations

### 14.1 Zone Awareness

Flipkart operates across multiple zones (HYD, CH). DLM must be zone-aware:

- **Source locality:** Archival jobs should run in the same zone as the source database to minimize cross-zone data transfer latency and cost.
- **Archival store locality:** Archived data is stored in the same region as the source by default. Cross-region replication is available but opt-in (at additional cost).
- **Control plane:** A single control plane instance manages entities across all zones. The control plane DB is replicated across zones for DR.

### 14.2 Deployment Topology

```
Zone HYD:
  - Archival Manager replicas (handles HYD-zone sources)
  - MySQL Plugin, TiDB Plugin (HYD)
  - GCS Bucket: dlm-archival-prod-hyd

Zone CH:
  - Archival Manager replicas (handles CH-zone sources)
  - MySQL Plugin, TiDB Plugin (CH)
  - GCS Bucket: dlm-archival-prod-ch

Shared (either zone, replicated):
  - Control Plane API
  - Control Plane DB
  - Scheduler Worker
  - Web UI
```

### 14.3 Cross-Zone Data Transfer

- **Default:** No cross-zone data transfer. Source → Plugin → Archival Store all in the same zone.
- **Exception:** Data retrieval downloads may cross zones if the user is in a different zone than the archival store. Pre-signed URLs are zone-specific.
- **Cost:** Cross-zone egress charges are avoided by default. If cross-region archival is needed (e.g., for DR), the cost is estimated and displayed during entity onboarding.

---

## 15 Phased Rollout

### Phase 1: SQL Foundation (Q1–Q2)

**Scope:**
- Databases: Altair (MySQL), Rigel (TiDB).
- Core features: Source/entity/policy onboarding, scheduled + on-demand archival, three-phase workflow, purging, archival storage (GCS), data retrieval (date-based), dry-run, RBAC, audit logs.
- Plugin: MySQL plugin, TiDB plugin.
- UI: Onboarding wizard, job monitoring, archival calendar, retrieval interface.
- Observability: Grafana dashboards, core alerts, structured logging.

**Milestones:**
1. Control plane API + DB schema → Week 4
2. MySQL plugin (archive + purge) → Week 8
3. TiDB plugin → Week 10
4. Web UI → Week 12
5. Watcher integration → Week 14
6. Security (RBAC, encryption) → Week 16
7. Internal dogfooding (DLM team's own entities) → Week 18
8. Beta rollout (3–5 pilot teams) → Week 20
9. GA for SQL datastores → Week 24

**Exit Criteria:**
- 5+ entities onboarded and running scheduled archival.
- Job success rate ≥ 99.5%.
- Source DB P99 impact ≤ 10% during archival.
- Data retrieval functional and within SLA.
- RBAC enforced on all API endpoints.

### Phase 2: NoSQL + Advanced Features (Q3–Q4)

**Scope:**
- Databases: Yak (HBase), Scorpius (Aerospike).
- Features: HBase plugin (TTL + CF-based), Aerospike plugin, row-key-based retrieval index, FDP evaluation, purge-only mode enhancements, schema evolution alerts.

**Milestones:**
1. HBase plugin → Week 8
2. Aerospike plugin → Week 12
3. Row-key retrieval index → Week 14
4. FDP integration evaluation → Week 16
5. GA for NoSQL datastores → Week 20

### Phase 3: Platform Maturity (Q1 next year)

**Scope:**
- SQL-over-archived-data (Presto/Trino integration) — evaluation.
- Self-managed database support.
- Post-archival webhook integration with DBaaS (OPTIMIZE TABLE, compaction).
- Advanced analytics: archival cost dashboard, storage growth projections.
- Multi-region archival store replication.

---

## 16 Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | FDP cost model for archival volumes — is it competitive with GCS? | DLM + FDP teams | Open |
| 2 | How will DLM handle entities with composite primary keys spanning multiple columns? | DLM | Open |
| 3 | Should DLM support archival of views or only base tables? | DLM | Open (likely base tables only) |
| 4 | What is the expected max row size for Yak entities? (impacts chunk sizing) | Yak team | Open |
| 5 | Should DLM own the post-archival `OPTIMIZE TABLE` for Altair or rely on Altair to expose it? | DLM + Altair teams | Open |
| 6 | Is there a compliance requirement for archival store to be in a specific geographic region? | Legal/Compliance | Open |
| 7 | Should purge-only mode require DLM_ADMIN approval in addition to ENTITY_OWNER? | DLM | Open |
| 8 | How should DLM handle entities where the PK is a UUID (non-sequential) for chunk range splitting? | DLM | Open (likely hash-based partitioning) |

---

## Appendix A: Yak (HBase) Archival — Detailed Solution

### A.1 Problem
HBase stores binary payloads. Field-based archival policies cannot be applied externally. Data may be application-encrypted.

### A.2 Solution: ColumnFamily-Based Eligibility

1. Application creates a designated "archival-ready" ColumnFamily (e.g., `cf_archive`).
2. When a record reaches terminal state (application logic), the application copies the record's row key to `cf_archive` with a marker qualifier.
3. DLM's HBase plugin scans `cf_archive` on schedule.
4. For each row key found:
   a. Extract all CFs for that row key from the source table.
   b. Serialize to JSON and write to archival store.
   c. Delete the row from the source table (all CFs including `cf_archive`).
5. Post-purge: DLM notifies Yak team via webhook for major compaction.

### A.3 TTL-Based Archival (No Application Change)

1. DLM configures a scan with a TTL filter on the source table.
2. Records within N days of TTL expiry are extracted and archived.
3. Records are allowed to expire naturally via HBase TTL (no explicit delete needed).
4. This approach requires no application changes but only works for time-based policies.

### A.4 Limitations
- Application must implement the ColumnFamily marker pattern for state-based policies.
- HBase scan performance depends on RegionServer load; Watcher integration is critical.
- Row-level consistency during archival relies on HBase's row-level atomicity.

---

## Appendix B: Scorpius (Aerospike) Archival — Detailed Solution

### B.1 Problem
Aerospike has native TTL-based eviction. Records may be evicted before archival if not captured in time.

### B.2 Solution: Pre-Expiry Extraction

1. DLM's Aerospike plugin runs a scan with filter expressions:
   - `void_time` (expiry timestamp) within the next N hours (configurable buffer, default: 24h).
   - Or: user-defined timestamp bin satisfies the time-based policy.
2. Matching records are serialized to JSON and written to archival store.
3. Purging: Explicit `delete` with generation check (ensures record was not modified after extraction).
4. If generation check fails (record was modified), the record is skipped and logged.

### B.3 Set-Based Archival
- Entire Aerospike sets can be archived based on a time policy.
- DLM scans the set, applies the policy filter, and archives matching records.

### B.4 Limitations
- Field-level projection not supported (schemaless records).
- Record ordering in archival store is not guaranteed (scan order is non-deterministic).
- Large scans may impact Aerospike cluster performance; Watcher thresholds must be tuned for Aerospike-specific metrics (e.g., queue depth, device overload).
