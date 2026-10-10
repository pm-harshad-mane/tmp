# Proposal: Decision Fabric Developer SDK & Managed Services

**Working proposal — for discussion and customer discovery**  
**PubMatic Decision Fabric | October 9, 2026 | Draft v0.2**  
**Status:** Idea under evaluation. No service catalog, architecture, scope, or investment decision has been approved.  
**Audience:** Decision Fabric product and architecture teams, platform engineering, SRE, security, finance, and—most importantly—vendor developers.

> **Proposal in one sentence:** Make it substantially easier for third-party developers to build and operate ARTF containers on Decision Fabric by offering a small SDK, declarative service bindings, local development tooling, and optional managed infrastructure—while preserving practical portability to other SSP execution platforms.

> **Feedback requested:** Which developer problems are worth solving? Which services should PubMatic operate, versus vendors supplying their own? What would persuade a multi-SSP vendor to adopt this? What must we learn before committing to an implementation?

## 0. Discussion guide — what this proposal is and is not

This is a **proposal to explore building** a developer-services layer, **not** an argument that it is already the right investment. Each service proposal includes reasons *for* providing it and reasons *not* to. The intent is to collect input, test assumptions, and progressively narrow the scope. Comments marked **[Feedback]**, **[Question]**, and **[Validate]** are deliberately unresolved.

The proposed SDK is not the product by itself. A convenient `df.KV.Get()` call is valuable only if the resource behind it can be provisioned, authorized, isolated, operated, tested locally, and accessed predictably in production. We should evaluate the full platform experience rather than the apparent simplicity of the library.

**Non-goals for this proposal:** AI services, model hosting, vector databases, AI-agent APIs, or making ARTF containers PubMatic-exclusive. The fact that coding assistants can generate ordinary service integrations is considered as a challenge to our differentiation, not as a new service to sell.

### Decisions we are *not* making yet

- Whether the SDK should be mandatory, optional, or only a thin helper over documented standard protocols.
- Which services, if any, deserve V1 investment.
- Whether PubMatic should run shared, dedicated, or vendor-owned instances of each stateful backend.
- Whether services should be exposed as direct endpoints, a managed gateway, a local sidecar, or combinations of these.
- Whether the adoption benefit justifies an ongoing platform/SRE product commitment.

## 1. Purpose: what are we trying to achieve?

Our existing Go/Docker ARTF example repository already provides tested container skeletons, protocol handling, health endpoints, metrics, and concrete business use cases. A developer can clone an example, add business logic, and deploy. The next opportunity may be to reduce the **infrastructure and operational work after the container skeleton is written**.

The proposed developer platform would aim to:

1. **Shorten time to first useful deployment.** Developers declare `audience-cache`, `event-queue`, or `dataset` instead of independently installing and administering infrastructure.
2. **Reduce ongoing vendor operations.** PubMatic handles agreed provisioning, credentials, upgrades, scaling, monitoring, backup, and incident response for managed capabilities.
3. **Improve local-to-production consistency.** The same logical bindings work in local Docker tests and in Decision Fabric, with explicit differences documented.
4. **Improve reliability and safety.** Standardized quotas, authentication, auditability, service health, and resource lifecycle reduce accidental operational risk.
5. **Make Decision Fabric a preferred deployment destination.** Vendors may favor it because of less effort and better runtime economics, even while maintaining a portable codebase for other SSPs.

**Hypothesis, not fact:** Managed services may increase vendor adoption or deployment depth. We need vendor evidence that *infrastructure management*, rather than business logic, certification, data access, or onboarding approvals, is a meaningful bottleneck.

### Who benefits, and how?

| End user | Job to be done | Potential benefit | Possible objection |
|---|---|---|---|
| Audience/data provider developer | Package and refresh eligibility datasets; perform fast lookups | Managed ingestion, versioning, read-only snapshots or KV | Local map/file may be faster, cheaper and more portable |
| Bid enrichment / contextual vendor | Access reference data, configuration, events | Fewer services to administer | Their stack may already work everywhere |
| DSP / bidder developer | Manage state, throttling signals, async events | Local counters, queues, observability | Hot-path RPC or state semantics may be unacceptable |
| Vendor platform / DevOps team | Deploy to several SSPs | Declarative resources, repeatable dev/test, predictable support | Proprietary SDK creates another integration matrix |
| Vendor security/compliance team | Control access to sensitive data | Centralized identities, retention, audit and network policy | Shared services raise tenancy and data-governance concerns |
| PubMatic platform/SRE | Operate third-party workloads safely | Standardized resource governance and billing | New 24×7 databases/queues/storage are significant obligations |
| PubMatic product/partner team | Onboard and retain vendors | Better partner experience and possible commercial differentiation | May not alter vendor preference without stronger proof |

**[Feedback]** Which *actual developer persona* controls the decision to use a managed service: the container author, vendor infrastructure team, security team, or commercial owner?

### Demand, volume and urgency — still to validate

Before sizing the solution, we need to understand the size and urgency of the developer problem. How many external vendor teams are expected to build or materially modify Decision Fabric containers in the next 12–24 months? How many containers per vendor? How frequently will they change datasets or infrastructure dependencies? Is the expected population tens of developers, hundreds, or more? A strong developer platform can be valuable at any scale, but the appropriate engineering investment changes significantly with expected usage.

We should also establish whether anyone is **currently blocked** because these services do not exist. If vendors are successfully shipping using embedded data, standard clients, or their own infrastructure, the proposal is primarily an optimization/differentiation investment rather than an unblocker. Conversely, if real deployments are delayed by provisioning, data refresh, credentials, queues, storage, or operational requirements, that is stronger evidence for prioritization.

**[Open questions]**
- How many real vendor developers and production containers will this serve in year 1 and year 2?
- How often will they use each proposed service?
- Which current onboarding/deployment activities are blocked versus merely inconvenient?
- What is the business impact of waiting 6 or 12 months?
- Are there committed design partners who would use the capability in production?

## 2. Why consider building this now?

There is an opportunity to go beyond a repository of example containers. A vendor may be able to generate a Go gRPC server in hours, yet still need days or weeks to establish safe service credentials, local test fixtures, data delivery, operational dashboards, and production runbooks. A coherent developer experience can eliminate this repeated platform work.

But the counterpoint is equally important: **ARTF containers are intended to run in multiple SSP environments.** Vendors may already have Redis, Aerospike, Postgres, object storage, queues, and observability in their own infrastructure or image patterns. A Decision Fabric-specific call in the core decision path may create portability and reliability costs greater than the setup it saves. The SDK should compete against a *well-documented standard integration plus a coding assistant*, not against a developer starting from zero.

### Benefits to investigate, paired with counterarguments

| Potential advantage | Counterargument / alternative | Evidence needed |
|---|---|---|
| Faster development | Our tested starter examples already eliminate much boilerplate | Compare time-to-deploy with and without SDK |
| Less vendor infrastructure work | Vendor may already operate the same backend across SSPs | Interview vendor DevOps owners; quantify current toil |
| Stronger platform preference | Vendors cannot afford a separate business logic codebase per SSP | Ask whether optional adapters are acceptable |
| Lower total cost through pooling | Shared backend adds network, SRE, isolation and metering costs | Cost per vendor and per million operations |
| Better observability and compliance | stdout/OpenTelemetry and standard exporters may suffice | Identify genuinely missing workflows |
| Convenient local development | Docker Compose and standard SDKs already work well | Usability tests with representative vendor projects |
| Faster managed data refresh | Mounted immutable snapshots may be simpler than online KV | Compare refresh and lookup paths at production scale |

### What is the moat?

A convenient SDK by itself may not be a durable differentiator: wrappers around KV, object storage, queues, SQL, logging, or configuration can be reproduced, and AI-assisted development further reduces the cost of writing integration code. The potential moat, if one exists, is more likely to be the **complete operating experience**: low-latency services colocated with the auction path, automatic provisioning, reliable dataset delivery, strong isolation, integrated observability, predictable economics, self-service workflows, and reduced operational burden.

We should therefore separate **API convenience** from **platform differentiation**. If a vendor can obtain nearly the same outcome using a standard Redis/S3/Postgres/OTLP endpoint, that may actually be preferable: PubMatic can still provide the differentiated managed experience while the vendor retains portable application code.

### Economics and return on investment

Cost reduction should be treated as a measurable hypothesis rather than an assumed benefit. For each candidate service we should compare at least three models: vendor-bundled/self-managed infrastructure, PubMatic-managed dedicated infrastructure, and PubMatic-managed shared infrastructure. The calculation should include compute/memory, replication, storage, network traffic, idle capacity, engineering development, 24×7 SRE ownership, upgrades, security, support, metering and incident cost.

The relevant question is not only **“what percentage can shared infrastructure save?”** but also **“what is the absolute saving and is it large enough to repay the platform investment?”** A 30% reduction in a small infrastructure component may have poor ROI; a smaller percentage reduction across a very large repeated workload could be compelling.

**[Open questions]** What is current monthly infrastructure/operations cost per representative vendor? What would it cost with the managed model? What percentage and absolute dollar saving results? What engineering/SRE investment is required to achieve it? At what number of vendors or operations does the platform break even?

## 3. External inspiration: Cloudflare developer experience, not a literal copy

**Cloudflare Workers + Wrangler** offers a useful developer-experience pattern: define resource bindings declaratively, access them by stable logical names, run code locally against simulated resources, and deploy with platform-provisioned resources. Cloudflare exposes Workers KV, Durable Objects, D1 SQL, R2 object storage, Queues, service bindings, and additional capabilities. Wrangler documents automatic provisioning for several resources and local/remote binding choices. Sources: [Cloudflare bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/), [local development](https://developers.cloudflare.com/workers/local-development/), [automatic provisioning](https://developers.cloudflare.com/workers/wrangler/configuration/), [local support matrix](https://developers.cloudflare.com/workers/local-development/bindings-per-env/).

**The attractive pattern:**

```text
Declare logical resources -> develop with local equivalents -> deploy
         |                              |                     |
   KV / DB / object / queue      same logical API       platform binds
                                                        real resources
```

**Why this comparison has limits:** Cloudflare controls the Workers runtime and service-binding environment. Decision Fabric runs vendor-authored OCI/ARTF containers that may also target competing SSPs. We do not automatically have the ability to inject runtime-native bindings or mandate a Cloudflare-like programming model. Also, local emulation is not identical to distributed production behavior (latency, consistency, failures, security).

**AWS comparison, framed carefully:** Cloudflare's tightly integrated workflow may *feel* simpler for small projects or prototypes than composing multiple AWS services, IAM permissions, and SDK configurations. However, it would be inaccurate to claim Cloudflare is mainly for hobby projects while AWS is for production: both serve production workloads, and both can involve substantial operational complexity. The relevant product question is whether **our specific vendor developers** value Wrangler-like simplicity enough to adopt it for production ARTF workloads. **[Validate with customer interviews rather than treating this as a market fact.]**

### What to borrow—and what to avoid

Borrow: named bindings, declarative manifests, sensible defaults, local emulators, isolated development state, quick deploy/test, resource explorer, one-command fixtures, explicit remote-dev opt-in, clear docs and error messages. Avoid: hidden data dependencies, implicit production writes from local tests, a new proprietary data model for every existing standard service, and an assumption that portability does not matter.

## 4. Proposed experience: what would a vendor actually do?

**Illustrative, not a committed API or ARTF manifest schema.** A developer selects an ARTF starter example, declares optional capabilities, tests locally, and deploys the same image with environment-specific bindings.

```yaml
# decision-fabric.yaml — proposed product configuration, NOT an IAB-standard schema
app: audience-eligibility
runtime: artf-grpc
bindings:
  audience_data:
    kind: dataset.snapshot
    required: true
    max_age: 24h
  lookup_cache:
    kind: kv
    required: false
    access: read
    consistency: eventual
  events:
    kind: queue
    required: false
    delivery: at-least-once
  logs:
    kind: telemetry
    required: false
```

```go
// Conceptual Go API. Service binding names, errors and contracts are TBD.
client, err := df.OpenFromEnvironment(ctx)
if err != nil { return err }
store, ok := client.KV("lookup_cache")
if ok {
    value, err := store.Get(ctx, key) // timeout, not-found, unavailable are distinct
    _ = value; _ = err
}
// Required audience_data is checked before readiness; no silent no-op.
```

An equally important **SDK-free path** should be available: `DF_BINDING_LOOKUP_CACHE_ENDPOINT`, standard Redis/HTTP/gRPC/S3-compatible endpoints, and workload-scoped credentials. A vendor can then change endpoints/configuration when deploying to another SSP, *provided that the destination permits network access and supports compatible semantics*. Endpoint substitution alone does **not** guarantee portability across hosts with different egress rules, consistency, auth, or resource availability.

### Three possible developer models to compare

| Model | Developer experience | Portability | PubMatic work | Discussion |
|---|---|---|---|---|
| **A. PubMatic-specific SDK** | `df.KV.Get()` and `df.Queue.Publish()` | Requires adapters or conditionals | SDK + full services + tooling | Most branded, potentially least portable |
| **B. Standard protocol + managed endpoint** | `redis.Get()`, `sql.DB`, S3 client, OTLP, HTTP/gRPC | Often strongest where protocol is accepted | Provisioning, identity, bindings, operations | Could deliver most convenience without custom SDK |
| **C. Host-neutral interface + provider adapters** | `storage.Get()` selected via config | Strong if semantics and alternate backends are tested | SDK adapters + managed services + conformance tests | More abstraction work, but preserves business logic |

**[Question]** Would we gain more adoption from a good **developer CLI, binding spec, and managed service catalog** than from a PubMatic-branded Go API? We should test all three models, not presuppose A.

## 5. Candidate services — each requires an independent case

The service catalog below is inspired by Cloudflare's developer-platform patterns, but tailored to ARTF workloads. A service should not be included merely because Cloudflare offers it. For each, compare a PubMatic-managed version with vendor-provided equivalents, standard APIs, and a no-service option.

### 5.1 Key-value store / cache (Cloudflare Workers KV analogue)

**Use cases:** audience eligibility lookup, vendor configuration, counters (only with suitable semantics), mapping IDs to deal IDs, low-churn reference tables.

**Possible approaches:** (A) process-local Go map/mmap; (B) versioned snapshot with local index; (C) vendor-operated Redis/Aerospike; (D) shared PubMatic Redis/Aerospike/other KV with tenant partitioning; (E) dedicated managed KV per vendor; (F) local cache backed by remote managed KV. Offer a standard Redis-compatible endpoint where appropriate, rather than requiring a proprietary client.

**Why PubMatic-managed helps:** automatic provisioning, HA, pooling, access control, quota management, standardized backup/monitoring; developers avoid running a stateful cluster. **Why vendor-owned may be better:** no platform-specific state dependency, choice of consistency/indexing, existing vendor expertise, independent operations, easier reuse on other SSPs. **Important counterpoint:** same Kubernetes namespace does not guarantee low latency; same-node placement and network path matter more. A remote managed KV adds a synchronous RPC regardless of namespace. For sub-5-ms auction workloads, local immutable data may be preferable.

**Design questions:** required consistency? max key/value size? eviction/TTL? persistence? read-only vs write? bulk load? p99 under N-1? tenant quotas? replication and locality? failure policy? allowed network path on alternate SSPs? **[Validate]** Does an actual vendor need mutable online KV, or only daily snapshot refresh?

### 5.2 Versioned datasets / read-only snapshots (an especially relevant Decision Fabric service)

**Use cases:** audience CSV, device/content eligibility tables, contextual catalogs, mapping tables, fraud rule sets. **Approaches:** vendor packages CSV in image; vendor downloads at startup; PubMatic ingests a signed artifact and mounts a versioned file; PubMatic distributes a compact indexed/mmap representation; vendor uses a remote data endpoint.

**Managed advantages:** separates code from data release, validates schemas/checksums, schedules refresh, publishes atomically, supplies provenance and rollback, keeps auction reads process-local. **Vendor-managed advantages:** simplest container portability when data is bundled, vendor controls update schedule and format, no new platform data-plane dependency. **Costs:** ingest/validation pipeline, per-DC fan-out, storage, large warmups, stale-data alarms, consistency across replicas, sensitive-data retention. **[Question]** Could this be our highest-value service even if we never expose online KV?

### 5.3 Object storage (Cloudflare R2 analogue)

**Use cases:** CSV imports, large static datasets, snapshots, diagnostic artifacts, export files, historical batch results. **Approaches:** S3-compatible object endpoint backed by existing object infrastructure; signed upload/download URLs; mounted read-only artifact; vendor-operated object storage with approved ingestion. **Managed advantages:** simpler upload and lifecycle, standardized encryption/retention, locality, reproducible datasets. **Vendor alternative:** existing S3/MinIO-compatible tools, vendor-owned storage, easier cross-host sharing. **Risks:** egress policy, large-object fan-out, cost of replicas, object versioning, tenant leakage, hot-path misuse. Prefer artifact staging and local reads for auction traffic; do not turn an object GET into an auction-time dependency.

### 5.4 Relational database / SQL (Cloudflare D1 analogue)

**Use cases:** control/configuration, metadata, job tracking, moderate transactional state; usually *not* per-auction reads. **Approaches:** managed PostgreSQL-compatible endpoint; tenant database/schema with strict isolation; dedicated instance for sensitive/high-load tenants; embedded SQLite for local or immutable state. **Managed advantages:** migrations, backup, credentials, HA, consistent SQL access, shared tooling. **Vendor alternative:** vendor's existing Postgres/SQLite schema and deployment pipeline; less cross-SSP state migration. **Risks:** connection storms from many pods, schema isolation, noisy neighbors, upgrades, recovery objectives, SQL injection/permissions, regional consistency. **[Question]** Is there a demonstrated need for transactional DB inside an ARTF decision container, or is it mainly an offline/control-plane need?

### 5.5 Durable stateful objects / per-key coordination (Cloudflare Durable Objects analogue)

**Use cases:** per-entity counters, pacing/throttling coordination, rate limiters, small serialized state machines. **Approaches:** sticky key-sharded actor service, Redis atomic operations/Lua, partitioned stream processor, vendor-owned stateful service; possibly *no* stateful primitive in V1. **Managed advantages:** simplifies serialized updates and per-key coordination if backed by a credible consistency model. **Vendor alternative:** vendor controls state machine and fault model; can use familiar databases and avoid lock-in. **Risks:** the hardest service here—placement, migration, failover, hot keys, consistency, exactly-once illusions, state restoration, cross-DC behavior, very high QPS. A Durable Objects-like API is **not** just KV with another method. **[Question]** Is there a real auction use case whose latency and consistency requirements justify building this?

### 5.6 Queues and asynchronous events (Cloudflare Queues analogue)

**Use cases:** async analytics events, noncritical enrichment signals, audit events, delayed processing, dataset refresh triggers. **Approaches:** managed Kafka/Redpanda-compatible topics; HTTP/gRPC publish endpoint; local bounded buffer/sidecar; vendor-operated queue. **Managed advantages:** durable delivery, backpressure, centralized metering, consumers and dead-letter handling. **Vendor alternative:** vendor already operates Kafka and can retain end-to-end ownership. **Risks:** enqueue latency on request path, delivery semantics (at-least-once means duplicates), ordering/partition keys, data retention, quota abuse, consumer lifecycle and compliance. Favor nonblocking bounded emission for auction-time producers, with explicit drop/backpressure policy; do not imply a fire-and-forget call is guaranteed durable.

### 5.7 Logging, metrics, traces and diagnostics (Cloudflare analytics/observability inspiration)

**Use cases:** per-container error logs, request counts, response types, latency, resource consumption, per-DC health, vendor debugging. **Approaches:** standard stdout/stderr + OpenTelemetry OTLP exporters and collector; PubMatic SDK convenience wrapper; vendor-provided instrumentation. **Managed advantages:** uniform dashboards, access control, correlated deployment and request metadata, reduced vendor setup. **Vendor alternative:** standard logging and OpenTelemetry already exist, no need for branded SDK. **Risks:** cardinality, PII leakage, log-volume costs, backpressure, telemetry loss. **Candidate principle:** platform captures baseline telemetry without vendor SDK; SDK adds only optional convenience. Optional telemetry may degrade gracefully, with counters for dropped records.

### 5.8 Secrets, configuration and environment bindings

**Use cases:** API credentials, data source settings, feature toggles, endpoint discovery. **Approaches:** Kubernetes Secret projected to a workload, CSI secret provider, short-lived workload credentials, config API or mounted files; local `.env` with development-only values. **Managed advantages:** vendor need not embed static credentials in image, rotation and auditing become possible. **Vendor alternative:** familiar Kubernetes/secret tooling, but the vendor may not control the host cluster. **Risks:** tenant identity, secret rotation without restart, accidental log exposure, local-prod divergence, permissions to external systems. This is a foundational capability, not necessarily a proprietary SDK call. Missing required secrets must fail readiness/deployment.

### 5.9 Scheduled jobs, workflows and dataset refresh (Cloudflare Workflows / scheduled triggers inspiration)

**Use cases:** daily audience CSV refresh, schema validation, index build, periodic sync, TTL cleanup, export. **Approaches:** platform CronJob/Job or workflow controller; vendor-provided refresh logic in its container; external vendor scheduler calling a documented ingestion API. **Managed advantages:** refresh executes once per revision rather than once per replica, retry/audit/rollback are centralized, container serving stays stateless. **Vendor alternative:** no dependence on PubMatic scheduler, control remains with vendor. **Risks:** idempotency, job overlap, backfill, per-DC convergence, data source credentials, publish gating, failed updates. Avoid each ARTF serving pod independently downloading and loading a large CSV on startup.

### 5.10 Service-to-service invocation and discovery (Cloudflare service bindings inspiration)

**Use cases:** one vendor component invoking another approved vendor service, data loader publishing to serving container, controlled internal APIs. **Approaches:** Kubernetes Service DNS + mTLS/identity, gateway-mediated gRPC/HTTP, host-provided logical service name. **Managed advantages:** stable discovery, auth, observability, network policy, rollout controls. **Vendor alternative:** normal Kubernetes service discovery if vendor is permitted to deploy multiple services. **Risks:** cross-tenant reachability, cyclic dependencies, latency and retries, prohibited host-to-host or egress access on other SSPs. **[Question]** Do we want to support multi-service vendor applications in V1, or only a single ARTF container with managed backing services?

### 5.11 Rate limiting / quota counters

**Use cases:** protect vendor endpoints, enforce per-tenant budgets, optional request shaping. **Approaches:** platform gateway limits, in-process token bucket, shared atomic store, vendor logic. **Managed advantages:** prevents noisy-neighbor incidents and gives common visibility. **Vendor alternative:** standard middleware libraries already solve many local cases. **Risks:** globally consistent counters cost network round trips; fail-open/fail-closed choice affects revenue; platform-enforced quotas should not depend on the vendor voluntarily calling an SDK. Treat enforcement as a host responsibility, with SDK only for visibility.

### 5.12 Data export, usage and billing metering

**Use cases:** per-container usage reports, dataset delivery audit, events export, discrepancy analysis. **Approaches:** host-owned telemetry and metering API, periodic object exports, Kafka/OTLP streams, optional SDK annotations. **Managed advantages:** trustworthy standardized reporting and support; less integration work. **Vendor alternative:** vendor uses its own logs and reconciliation. **Risks:** accuracy, tenant attribution, retention, contractual SLAs, high-cardinality event volumes. Metering must be host-generated and independently verifiable, not based solely on self-reported SDK calls.

### Initial service prioritization — **illustrative, for feedback**

| Service | Developer value hypothesis | Hot-path sensitivity | Build/operate complexity | Portability potential | Open question |
|---|---|---|---|---|---|
| Versioned datasets / mounts | High for audience providers | Low if read local | Medium–high | High with standard file contract | Do vendors need this most? |
| Logs / metrics / traces | Broad | Low if async | Medium | Very high via OTel | What isn't solved already? |
| Secrets / config / discovery | Broad prerequisite | Usually low | Medium–high | High with env/mounts | Host baseline or SDK product? |
| Object storage / import | Medium–high | Keep off hot path | Medium–high | High via S3-compatible API | Build or reuse existing? |
| KV / cache | Use-case dependent | **High if per request** | High | Medium with Redis/protocol adapters | Can local snapshots replace it? |
| Queues / events | Medium | Medium if sync publish | High | High with Kafka/HTTP | Who operates consumers? |
| SQL database | Unclear for ARTF hot path | High if per request | High | High via PostgreSQL | Is demand real? |
| Scheduled jobs / refresh | High for data vendors | Off hot path | Medium–high | Medium | What refresh SLA is needed? |
| Durable stateful objects | Unproven | Very high | **Very high** | Low–medium | Is there a strong use case? |
| Service discovery / invocation | Conditional | Potentially high | Medium–high | Medium | Multi-service vendors in scope? |
| Rate limiting | Host need | High if remote | Medium | High for local logic | Platform feature, not SDK? |
| Usage / exports | Broad commercial need | Off hot path | High | High via standard formats | Which SLAs are promised? |

**[Feedback]** Please mark each service as **Must-have / Useful / Not needed / Unknown** and provide one actual vendor use case.

## 6. Proposed high-level architecture

```text
   Vendor developer                          Decision Fabric control plane
   ----------------                          -----------------------------
   Go ARTF starter + optional SDK            App / vendor / deployment registry
   df.yaml binding declarations   -------->  Capability validation + entitlements
   df dev (local fixtures)                    Resource catalog + provisioner
   df test / deploy                           Identity / secrets / policy
          |                                   Resource lifecycle / billing
          |                                              |
          v                                              v
   +----------------------- Kubernetes / per DC --------------------------+
   | Vendor namespace                                                    |
   |   ARTF serving pod(s) -- gRPC request/response --> SSP auction path  |
   |       |                                                              |
   |       +-- process-local data / mounted versioned snapshots           |
   |       +-- optional standard client / SDK adapter                     |
   |                  |                                                    |
   |                  +-- authenticated binding endpoint / gateway         |
   |                           |                                           |
   |   Platform services namespace / dedicated data-plane clusters        |
   |      KV/cache | object/artifact | SQL | queue | telemetry | secrets   |
   |      (shared, isolated, or dedicated according to service policy)    |
   +---------------------------------------------------------------------+
```

**Critical boundary:** the vendor-facing SDK, API contracts, binding service, and service endpoints should be separate from PubMatic's internal SSP applications, databases, schemas, and operational interfaces. Internal assets are exposed only through reviewed product contracts/adapters. No vendor container should obtain implicit access to an internal PubMatic datastore merely because it runs in our Kubernetes cluster.

### Components we would need to implement or productize

| Component | Responsibility | Challenge / decision |
|---|---|---|
| Go SDK (optional) | Typed helpers, timeouts, errors, capability discovery | Versioning, compatibility, avoiding proprietary business logic |
| Declarative binding contract | Logical names, requirements, semantics, versions | ARTF alignment; extension vs separate manifest |
| CLI + local runtime | `init`, `dev`, `test`, `seed`, `inspect`, `deploy` | Faithful local behavior, Docker-only development |
| Resource catalog | Supported service classes and SLAs | Service-specific limits and pricing |
| Provisioning controller | Create/update/delete per-tenant resources idempotently | Reconciliation, rollback, garbage collection |
| Binding resolver | Map logical name to authorized endpoint/resource | Identity, region/DC affinity, rotation |
| Workload identity | Bind vendor/app/deployment to entitlements | No trusting user-supplied tenant IDs |
| Data-plane gateway or direct endpoints | Transport, routing, auth, quota, telemetry | Per-hop p99, failure blast radius |
| Backend service operators | Redis/Aerospike, object store, SQL, queues, etc. | Upgrades, HA, backup, capacity, N-1 |
| Dataset ingestion/distribution | Validate, build, publish, roll back artifacts | Large datasets, atomicity, freshness |
| Observability + metering | Usage, SLA, tracing, billing, vendor dashboards | Attribution, PII, high cardinality |
| Security/policy controls | NetworkPolicy, mTLS, secrets, egress policy | Cross-tenant isolation and audit |
| Conformance / portability tests | Run same workload with multiple adapters | Semantics, host constraints, error handling |

### Binding and identity lifecycle (illustrative)

1. Vendor declares logical resource `audience_data` with access, durability, consistency, size and freshness needs.
2. Control plane validates service availability and vendor entitlement in each deployment region/DC.
3. Reconciler provisions or assigns an isolated resource, attaches quota and policy, and records a **binding ID** distinct from internal backend IDs.
4. Admission controller validates required bindings. Missing required services block deployment/readiness with actionable errors.
5. Workload identity (for example service account identity plus short-lived token/mTLS) is mapped server-side to allowed binding IDs and operations. The client cannot choose another tenant's resource by guessing an ID.
6. Pod receives a scoped endpoint/configuration; SDK or standard client connects. Binding gateway/endpoint enforces authorization on every operation.
7. Rollouts, scaling, credential rotation, dataset revisions and resource deletion are reconciled independently from application code.
8. Metering records authorized usage by vendor/app/deployment/DC; support tooling can explain which resource and revision served a request.

**[Design decision]** Gateway versus direct backend: a gateway offers stable contracts and enforcement but adds a hop and operational bottleneck. Direct Redis/Postgres endpoints offer standard compatibility but require strong backend auth, credential isolation, network controls and safe client connection limits. A local sidecar can simplify identity and buffering but adds pod CPU/memory and still needs upstream networking. We should benchmark all relevant paths before selecting one.

### Kubernetes considerations

Kubernetes namespaces are logical administrative/security boundaries, **not physical network-locality guarantees**. Cross-namespace traffic can remain on the same node; two pods in the same namespace can be on different nodes. Measure pod placement, service routing/CNI, gateway overhead, serialization, TLS, cross-node hops, and p99.9 under failure. Apply default-deny NetworkPolicies, workload identity, quotas, anti-affinity/topology placement, and backend authorization. Shared service failures can impact many vendors; dedicated instances reduce blast radius but increase cost.

## 7. Worked example: audience CSV → managed KV versus local snapshot

**Scenario:** vendor provides `(device_id, content_id) -> deal_ids[]` as a CSV, refreshed daily. An ARTF request needs a lookup within a very small latency budget.

**Option A — portable default:** vendor packages CSV or receives a versioned mounted dataset; container validates and loads an in-memory map or local read-only index; lookup is process-local. **Pros:** no RPC, simple business logic, works on restrictive SSPs. **Cons:** replica memory multiplication, warmup time, coordinated updates, dataset fan-out.

**Option B — managed online KV:** vendor uploads a CSV once per revision; PubMatic validates it, creates a staging namespace/database, bulk-loads it, verifies counts/checksums, and atomically publishes a binding to the new dataset; serving pods query KV. **Pros:** centralized dataset lifecycle, potentially less per-pod memory, easier mutable data. **Cons:** per-auction RPC, shared-backend tail latency and blast radius, cross-host portability, platform SRE obligations.

**Option C — managed ingestion + local serving:** PubMatic handles upload, validation, compact-index build and versioned distribution; pods continue using local data. **Pros:** preserves fast path while relieving vendor of refresh operations. **Cons:** requires an artifact distribution and publication platform, still uses per-pod memory or local disk.

**Option D — vendor-owned KV:** vendor deploys Redis/Aerospike in allowed vendor scope or connects to an approved endpoint. **Pros:** backend ownership and standard APIs. **Cons:** the vendor or host must manage persistence, HA, capacity and permissions; may be prohibited by another SSP's runtime rules. Same namespace does not eliminate an RPC.

### The lifecycle we must solve if we offer managed ingestion

```text
Vendor artifact upload / signed URI / packaged file
  -> authentication + size/type/PII policy
  -> checksum + schema validation + duplicate-key policy
  -> staging data revision / build index / bulk load
  -> data count, integrity and performance verification
  -> publish immutable revision / switch active binding atomically
  -> serving readiness confirms active revision
  -> monitor age + health; rollback on bad revision
  -> retention / deletion / billing / audit
```

**Do not** have every serving pod independently ingest the same CSV into a shared KV on boot: scaling, rolling updates and restarts would duplicate writes and create races. Use a revision-keyed Job/controller with idempotency and a clear owner for publication. Define behavior when the latest refresh fails: continue serving the last known good revision until a documented maximum age, then apply an explicit business policy (e.g. not-ready, no-bid, or controlled fallback). Do not silently treat unavailable audience data as an empty match.

**Performance illustration:** 100,000 ARTF requests/second × one KV read each = 8.64 **billion** reads/day, before retries, multiple keys or cross-DC replication. This is a hypothetical load, not an estimate of PubMatic traffic. Compare process-local, same-pod, same-node, cross-node and degraded-backend p50/p95/p99/p99.9, CPU, memory, cost and operational burden.

## 8. Local development and production parity: a Wrangler-like path

A developer should be able to run `df dev` with Docker and **no access to PubMatic production infrastructure**. The tool reads declared bindings, starts local equivalents (e.g. Redis, Postgres, MinIO, queue broker), mounts sample datasets, injects local endpoint variables and fake-but-scoped development credentials, and launches the ARTF container with the same logical binding names.

```bash
# Illustrative CLI, not an existing tool
 df init --example audience-lookup
 df dev --seed ./fixtures/audience.csv
 df test --artf ./fixtures/requests.json
 df bindings inspect
 df deploy --environment staging
```

**Local behavior requirements:** persistent or resettable state; seeded fixtures; deterministic test data; offline mode; no production credentials; local resource inspector; reproducible Docker Compose setup; versioned service emulators; network-failure injection; documented differences in durability/consistency/latency. Remote staging bindings should be an **explicit opt-in**, with warnings for real writes and costs. A local SQLite/Redis simulation must not falsely promise production semantics.

**Production behavior requirements:** provision resources before pods become ready, validate required capabilities at admission, bind scoped credentials/endpoints, set timeouts and resource budgets, monitor service health, and support rollback/rotation. The *same image* should ideally work with different environment configs. A compatibility suite must cover local, PubMatic staging, production-like degraded conditions, and an alternate-host configuration.

**[Feedback]** Would vendor developers prefer a CLI they can run independently, or documented Docker Compose files and standard service SDKs? Do they want a managed remote sandbox, or would that introduce unwanted approval/network friction?

## 9. Multi-SSP portability: design it deliberately

ARTF standardizes an execution and interaction model, **not** universal semantics for Redis, object storage, queues or databases. A vendor may target Decision Fabric and Index Cloud or another SSP. We should distinguish three portability levels:

- **Source portability:** business logic stays unchanged; host adapters may differ.
- **Image portability:** the exact OCI image runs with host-specific environment/configuration.
- **Operational/state portability:** data, refresh jobs, credentials, retention, consistency and failure handling can move as well. This is the hardest level.

### Four concrete options

1. **Standard protocol first:** provide S3-compatible, PostgreSQL, Redis, Kafka/HTTP, OTLP endpoints when feasible. Developer changes endpoint and credentials on another platform. **Caveat:** another SSP may forbid external network access or lack an equivalent backend; protocol compatibility does not imply same latency/consistency.
2. **Vendor-owned domain interfaces:** `AudienceLookup`, `EventPublisher`, `ConfigStore` interfaces with `snapshot`, `redis`, `df-managed` adapters selected by config. PubMatic can supply adapter packages without putting host-specific calls into decision logic.
3. **Optional SDK conveniences:** SDK adds capability discovery, local binding resolution, telemetry annotations, typed errors. Missing *optional* telemetry can degrade visibly; missing *required* state must block startup or follow an explicitly configured fallback.
4. **Portable data contracts:** versioned files and standard formats for snapshots; import/export and migration procedures for managed persistent state. Test migration rather than assuming endpoint replacement is enough.

### Failure semantics cannot be silent for core services

| Dependency | If unavailable | Suggested behavior |
|---|---|---|
| Required audience dataset / KV | Decisions may be wrong | Reject deployment/readiness, or use an explicitly validated last-good snapshot; business-specific response policy |
| Required secret / config | May leak or misroute | Fail startup/readiness with clear diagnostic |
| Optional log annotation | Business result unaffected | Continue, count dropped telemetry, expose health |
| Optional metrics export | Business result unaffected | Continue with bounded buffer/drop policy |
| Async event publication | Depends on event importance | Explicit best-effort or durable delivery contract; no hidden guarantee |
| Rate limiter / counter | Revenue/safety implications | Explicit fail-open/fail-closed policy agreed per use case |

**[Question]** Should we publish a **host-neutral service-binding specification** (with standard transports and conformance tests) and then implement PubMatic as one provider? This may increase adoption even if it reduces exclusivity. Preference can come from quality of service, price, proximity and support rather than proprietary syntax.

## 10. Challenges and unresolved risks

An important architectural consequence of convenience is **responsibility transfer**. When a vendor calls `df.KV.Get()` instead of operating its own data path, complexity has not disappeared; much of it has moved to PubMatic. We would own more of the provisioning, availability, capacity, isolation, upgrades, credential rotation, backup/recovery, debugging and incident response. That may be the right product choice, but the operational obligation should be part of the feature decision.

| Area | What may make this difficult | Mitigation or experiment |
|---|---|---|
| Actual developer demand | Vendors may not want a new SDK | Interview real container developers before choosing scope |
| AI-assisted implementation | Coding assistants can generate Go/Docker/Redis/OTel integrations quickly | Measure end-to-end setup, ops and support savings, not lines of code |
| Cross-platform deployment | Proprietary stateful dependency blocks other SSPs | Standard protocols, adapters, portable snapshots, conformance tests |
| Production versus local | Local emulator differs in failure, consistency, scale | Fault injection, documented parity matrix, staging tests |
| Latency and QPS | Extra RPC on every auction adds tail latency and backend load | Benchmark realistic p99.9, no hidden retries, local-first patterns |
| Security | Shared state may expose one vendor to another | Identity, authz at data plane, network policies, audit and penetration tests |
| Provisioning | Stateful resources need lifecycle/cleanup | Reconciliation controller, quotas, idempotency, owner references |
| Data residency and privacy | Audience/device identifiers may be sensitive | Purpose limits, encryption, retention, region and contractual review |
| Multi-tenancy | Noisy neighbors and blast radius | Per-tenant limits, isolation tiers, admission and load testing |
| Durability/consistency | KV/SQL/queues have different guarantees | Explicit service-specific contracts; no universal `Get` semantics |
| Platform economics | Free-looking SDK hides 24×7 services | Chargeback/metering, total-cost and break-even analysis |
| Support and upgrades | SDK versions + backend versions become an API promise | SemVer, compatibility matrix, deprecation window, runbooks |
| Responsibility transfer | Developer simplicity moves provisioning, reliability, recovery and support work to PubMatic | Quantify ownership and on-call requirements per service |
| Shared-service blast radius | A vendor-owned dependency may affect one vendor; a shared managed service could affect many | Isolation tiers, cell architecture, per-tenant limits, failure-domain design |
| Platform complexity | SDK, control plane, bindings, emulators and stateful services enlarge Decision Fabric itself | Compare against simpler standard-endpoint and vendor-owned models |
| ROI / opportunity cost | Engineering effort may exceed infrastructure or onboarding savings | Break-even model and comparison with alternative roadmap investments |
| Vendor trust | Proprietary platform dependency may be viewed as lock-in | Optionality, transparent costs, export and exit paths |
| Standards alignment | Custom manifests may diverge from ARTF | Separate extension, standards review, never label custom fields as IAB-standard |

### What if we do **not** build the SDK or managed-service catalog?

This is a credible alternative, not a negative outcome. We could invest in: (a) excellent portable Go/ARTF examples, (b) a documented service and data-delivery contract using standard endpoints, (c) versioned dataset mounts, (d) OpenTelemetry/stdout support, (e) self-service deployment/testing and vendor dashboards, and (f) certification and performance tools. Vendors would bring their own Redis/Aerospike/SQL/queue implementations where allowed, or use only local data. PubMatic would avoid becoming the operator of numerous stateful multi-tenant products. **Tradeoff:** onboarding may involve more vendor setup and we may miss differentiation where fully managed resources are genuinely valued.

A middle path is also plausible: **build the developer experience and resource bindings, but initially expose only existing, well-operated PubMatic platform capabilities through standard contracts**—without promising a full Cloudflare-like service catalog.

## 11. Customer discovery and market evidence before architecture commitment

### Has anyone asked for this, and what do competing platforms provide?

We should distinguish an internally attractive architecture from demonstrated market demand. Before treating the managed-service layer as a differentiator, collect evidence of direct vendor requests, onboarding escalations, blocked deployments and repeated infrastructure pain. “This would be convenient” is weaker evidence than “we would deploy this service in production if available.”

We should also perform a focused competitive review of other SSP/container platforms, including Index Cloud and any other environments supporting ARTF-style vendor workloads. If competitors expose comparable KV, storage, queues, databases or developer bindings, understand their API model, portability, pricing and adoption. If they do **not** provide these capabilities, do not automatically interpret that as whitespace: it could represent an opportunity, or it could indicate limited customer demand or unfavorable economics.

**[Evidence to collect]** customer requests/tickets; current blocked deployments; competitor capability matrix; vendor usage/adoption where discoverable; and interviews with developers who have deployed the same workload to multiple SSPs.

We should interview **actual developers who author and operate vendor ARTF containers**, not only business stakeholders or internal enthusiasts. Suggested sample: 2–3 audience/data providers, 2 DSP/bidder teams, 1–2 contextual/verification vendors, plus partner SRE/security owners. This is a proposed sampling plan, not a completed study.

Ask them to walk through their current project rather than simply asking whether they “like” an SDK:

1. How do you build, test, package, deploy and update the same container across SSPs today?
2. Which steps consumed the most engineering time? How many hours and which teams?
3. Which backend services are already inside your container or external infrastructure?
4. Would you accept a PubMatic-managed Redis/Postgres/queue endpoint if it reduced operations? Under what SLA, price, and access rules?
5. What would you do on an SSP without that service or without egress?
6. Would you adopt a PubMatic Go SDK, a standard endpoint, or a host-neutral adapter? Why?
7. What are your request-path latency budgets, throughput, data freshness and failure semantics?
8. How do you refresh datasets today? Would versioned file delivery be more useful than KV?
9. What local development workflow would you actually run? Docker Compose, CLI, remote sandbox?
10. What data isolation, audit, billing and portability commitments are non-negotiable?
11. Which service would you use **in production in the next quarter**, not just in a demo?
12. What is the maximum extra operational dependency you are willing to take on for convenience?
13. Have you ever been blocked from deploying to an SSP because KV, object storage, queues, database, secrets, scheduling, or similar managed services were unavailable?
14. Which of these capabilities have you explicitly asked an SSP to provide before?
15. If PubMatic offered a standard Redis/S3/Postgres/OTLP-compatible endpoint instead of a PubMatic SDK, would that solve the problem equally well or better?
16. What would make you choose PubMatic's managed service over bringing your existing implementation?
17. Would this capability influence where you deploy more workloads, or is multi-SSP deployment non-negotiable regardless of developer experience?

**Output:** anonymized interview notes, a ranked problem list, evidence of actual adoption intent, and at least two design-partner candidates. Do not equate enthusiastic feature requests with production commitment.

## 12. Proposed evaluation / proof-of-concept plan (not an approved roadmap)

**Phase 0 — Discovery and baseline.** Interview vendors; instrument current example-repo onboarding; quantify time to first local test, time to production, ongoing toil and pain points. Identify existing platform capabilities that could be exposed without new databases.

**Phase 1 — Compare three developer experiences.** Implement the *same* audience lookup with (A) no SDK + local snapshot, (B) standard Redis endpoint + declarative binding, (C) SDK + managed binding. Use one source tree where possible. Have external developers try each without coaching.

**Phase 2 — Prove runtime economics.** Benchmark realistic QPS, p99/p99.9, pod/node/DC placement, startup, N-1, failures, credential rotation, dataset refresh, memory, egress, resource cost, and cross-tenant contention. Include another SSP-style deployment that lacks PubMatic bindings.

**Phase 3 — Choose a narrowly scoped service pilot.** Candidates: versioned dataset delivery, object import, optional KV, or queue/event export. Select based on evidence, not feature-list completeness. Require service ownership, on-call coverage, security sign-off and billing model.

**Phase 4 — Reassess.** Expand, revise, or stop the managed-service investment based on measurable developer adoption and total cost. Continuing with standard tooling alone remains a valid decision.

### Suggested success measures (targets TBD with stakeholders)

| Measure | Baseline needed | What improvement would justify investment? |
|---|---|---|
| Time to first correct local ARTF response | Existing examples | Meaningful reduction with independent vendors |
| Time to production deployment | Current vendor onboarding | Reduced developer and platform effort |
| Vendor production adoption | Number of vendors actually using service | Repeat use, not only POCs |
| Portability | Same source/image on alternate SSP | No core logic rewrite; explicit fallbacks |
| Auction latency | p99/p99.9 current container | No unacceptable added tail latency |
| Operational load | Vendor + PubMatic SRE hours | Net reduction, not cost transfer alone |
| Cost | $/vendor, $/million operations, DC capacity | Sustainable with clear pricing/chargeback |
| Reliability | Availability, recovery, data freshness | Meets explicit service-level objectives |
| Security | Isolation and audit tests | No unauthorized cross-tenant access |

## 13. Open decisions and feedback register

| ID | Question | Options to consider | Owner / feedback |
|---|---|---|---|
| D1 | What problem is the first release solving? | Boilerplate, operations, data refresh, observability | **[Open]** |
| D2 | Do vendors want this in production? | Interview + design partners | **[Open]** |
| D3 | SDK or standard endpoints first? | Proprietary Go API, standards, adapters | **[Open]** |
| D4 | Mandatory or optional? | Optional capabilities strongly favored for evaluation | **[Open]** |
| D5 | Which services in V1? | Dataset, KV, objects, queue, telemetry, secrets | **[Open]** |
| D6 | How are resources isolated? | Shared tenant partitions, dedicated instances, vendor-owned | **[Open]** |
| D7 | How does a vendor obtain identity? | Workload identity, mTLS, scoped token | **[Open]** |
| D8 | Where are services placed? | Same pod, node, rack/DC, separate cluster | **[Open]** |
| D9 | What is the required hot-path overhead? | Explicit per-service latency budget | **[Open]** |
| D10 | What happens if a required service fails? | Not-ready, last-good data, explicit business response | **[Open]** |
| D11 | What does local development run? | CLI + Compose, emulators, optional staging | **[Open]** |
| D12 | How is dataset refresh coordinated? | Revisioned Job/controller and atomic publish | **[Open]** |
| D13 | What is portable across SSPs? | Source, image, state, behavior | **[Open]** |
| D14 | Who pays and who supports it? | Usage meter, quotas, vendor tiers, SRE ownership | **[Open]** |
| D15 | What if demand is weak? | Stop at standard tools and snapshots | **[Open]** |
| D16 | Who are the end users and what is expected volume? | Personas, vendors, containers, operations, growth | **[Open]** |
| D17 | Is anyone blocked today? | Blocker, measurable toil, or future convenience | **[Open]** |
| D18 | What is the moat? | SDK API, managed operations, locality, economics, DX, none | **[Open]** |
| D19 | What is the ROI? | Monthly savings vs build + SRE + support investment | **[Open]** |
| D20 | Has the market asked for it? | Requests, design partners, production commitments | **[Open]** |
| D21 | What do competitors offer? | Equivalent services, standard endpoints, no service | **[Open]** |
| D22 | Can the same convenience be portable? | Standard protocols + endpoint injection + host-neutral bindings | **[Open]** |
| D23 | What responsibility moves to PubMatic? | SLA, state, recovery, support, security, blast radius | **[Open]** |
| D24 | Is added platform complexity justified? | Compare full SDK platform with simpler service exposure | **[Open]** |

### Suggested review prompts for the working group

- **Product:** Which specific vendor would choose Decision Fabric more often because of this? Who asked for it, who is blocked today, what is the expected volume/urgency, and what evidence supports the claim?
- **Architecture:** What service needs to be an RPC, and what can be a mounted artifact or standard endpoint?
- **Standards:** Can a host-neutral binding contract be proposed upstream or documented separately from ARTF?
- **Security/privacy:** Which vendor data is acceptable to persist, in what region, for how long, and who can access it?
- **SRE/infra:** Which managed services can we responsibly operate at multi-tenant, multi-DC scale today? What new failure domains, blast radius, on-call responsibility and operational complexity would PubMatic accept?
- **Finance:** What is the real cost of provisioning, replication, support, and idle capacity? What percentage and absolute monthly saving do we expect, when do we break even, and is that return better than alternative investments?
- **Vendor developers:** Which service and development workflow would you actually adopt for a production container? Would standard endpoints solve the same problem while keeping your code portable across SSPs?

## 14. Material retained and reframed from the earlier architecture review

The previous review's useful implementation work has been folded into this proposal rather than presented as already-approved architecture: **Kubernetes namespace versus physical network placement** (Section 6), **audience CSV ingestion and immutable revisions** (Section 7), **SDK and binding implementation components** (Section 6), **Wrangler-style local development** (Section 8), **required versus optional capability behavior** (Section 9), **multi-SSP portability** (Section 9), **latency and economics** (Sections 7, 10 and 12), and **engineering/operational sign-off** (Sections 10–13). The earlier categorical go/no-go recommendations have been replaced with explicit hypotheses, counterarguments, experiments and open decisions.

### Research references (external examples, not endorsements)

- [Cloudflare Workers bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/) — breadth of resource bindings and API model.
- [Cloudflare Wrangler configuration](https://developers.cloudflare.com/workers/wrangler/configuration/) — declarative resources and automatic provisioning.
- [Cloudflare local development](https://developers.cloudflare.com/workers/local-development/) — local simulations and optional remote bindings.
- [Cloudflare binding support by environment](https://developers.cloudflare.com/workers/local-development/bindings-per-env/) — demonstrates that local parity is service-specific.
- [Cloudflare local data](https://developers.cloudflare.com/workers/local-development/local-data/) — seeding local KV, R2 and D1.
- [Cloudflare Queues local development](https://developers.cloudflare.com/queues/configuration/local-development/) — queue developer experience and caveats.
- [IAB Tech Lab Agentic Real-Time Framework](https://iabtechlab.com/) — validate current ARTF manifests, protobuf and orchestration semantics against the published standard during design.
- [OpenTelemetry](https://opentelemetry.io/) — standard observability APIs and collectors.
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/) and [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) — routing and policy, not data-plane authorization by themselves.

---

**Document posture:** **Proposal / working discussion.** Nothing above represents a committed PubMatic feature or an approved architecture. The next version should incorporate actual vendor feedback, concrete performance measurements, platform service ownership, and a prioritized pilot scope.
