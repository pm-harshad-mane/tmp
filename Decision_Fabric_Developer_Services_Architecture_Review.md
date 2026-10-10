# Decision Fabric Developer Services: SDK, Bindings, and Portability

**Architecture decision review | Discussion draft**  
**Prepared for:** PubMatic Decision Fabric / ARTF platform working group  
**Date:** October 9, 2026  
**Status:** Proposed direction; not an approved platform commitment

> **The decision in one sentence:** Should PubMatic invest in an easy-to-use developer SDK backed by managed platform services, and, if so, how can it create meaningful developer value without undermining the cross-SSP portability that makes ARTF attractive?

## Executive decision brief

**Recommended direction: proceed with developer-experience investment, but do not begin with a mandatory PubMatic SDK or silent-failure managed KV interface.** Build a portable ARTF starter-kit and standards-based operational integrations first. Pilot optional, explicitly declared service bindings for a narrow use case; make read-only data snapshots the first-class audience-lookup pattern, and treat online KV as a separately justified capability.

The goal should be **preference through operational advantage**, not lock-in through source-code dependency. A vendor may reasonably deploy the same business logic on PubMatic, Index Exchange, and other orchestrators. Winning one vendor's deployment, traffic share, or ongoing investment does not require exclusivity.

### Five conclusions

1. **The actual product is not a Go package.** An SDK is a small facade over identity, tenant isolation, provisioning, routing, security, data semantics, local emulation, quota enforcement, telemetry, support, and billing. Without those components, `decisionfabric.kv.Get()` is a misleading abstraction.
2. **“Fail silently” is not a portability strategy.** It can be acceptable for disposable convenience instrumentation, but not for core audience data or state. A no-op implementation can make an otherwise healthy-looking container make incorrect business decisions.
3. **ARTF gives the right vocabulary.** The finalized v1 specification supports named service dependencies in the manifest and says not to start a container when the orchestrator cannot support required dependencies or intents. It does not define a universal KV API or KV consistency contract. [S1]
4. **The real-time path changes the answer.** A local in-memory lookup over versioned snapshots is often better aligned with a sub-5-ms CTV audience lookup than an extra synchronous managed-KV hop. Index Cloud publicly documents a no-external-lookup auction path and mounted, independently versioned model data. [S4][S5]
5. **Cloudflare Workers is an inspiration, not a direct proof of fit.** Workers bindings are part of a tightly integrated runtime and account-resource model; ARTF is an OCI-container interoperability model across competing orchestrators. Cloudflare's local simulations are worth copying, but its platform-specific bindings do not establish that the same approach will be adopted by multi-SSP vendors. [S2][S3]

**Implementation companion:** Sections 19–27 are an affirmative, vendor-facing and platform-facing implementation blueprint **assuming the SDK decision has already been made**. They describe the SDK contract, service inventory, resource lifecycle, production request path, local environment, CSV hydration, failure handling, and delivery plan.

**Two additional nuances from the production-architecture review:** A namespace boundary is not itself a physical network hop; pod placement and service networking matter. And a “load CSV into KV at container boot” feature is a full ingestion-and-publication workflow, not a simple startup hook. [S13][S14][S15]

### What to approve now / defer

| Decision | Proposed disposition | Why |
|---|---|---|
| Improve Go examples, dev tooling, tests, documentation | **Approve** | Cross-platform value with little strategic downside |
| Standard logs, metrics, traces and runtime metadata | **Approve** | Can use OpenTelemetry and stdout without proprietary app API |
| Versioned data/artifact delivery and local mount contract | **Pilot first** | Useful to audience/model workloads; fast request path |
| Optional Decision Fabric bindings with capability discovery | **Design and validate** | Useful where a specific managed service adds measurable value |
| Mandatory `decisionfabric.kv` in business logic | **Do not require** | Portability, semantics and performance risks |
| Silent no-op for required KV or secrets | **Reject** | Masks incorrect results and causes silent revenue impact |
| New, full-featured hosted data platform as SDK prerequisite | **Defer** | High operations and cost surface before demand is proven |

## 1. Current context and intended outcomes

The team already has tested Go/Docker ARTF container examples: an audience-data provider with a packaged CSV, a related example that refreshes data from an outside source, and other use-case skeletons. These samples reduce implementation work by providing the gRPC server, protocol handling, container packaging, health checks, testing, and relevant business-logic insertion points.

The proposed next step is a PubMatic-managed library that lets a developer call methods such as `decisionfabric.KV.Get()` or `decisionfabric.Log()` while PubMatic provisions the backing infrastructure. The stated goals are faster onboarding and increased developer affinity or “stickiness.”

Those goals should be separated:

- **Onboarding success:** a vendor can go from cloning an example to a correctly functioning local test and production deployment with less engineering work.
- **Runtime value:** PubMatic gives the vendor lower latency, richer permitted auction context, reliable data freshness, useful telemetry, and less infrastructure overhead.
- **Vendor preference:** the vendor chooses to keep or expand its Decision Fabric deployment because its economics and operating experience are better—not because its code cannot run elsewhere.

**Assumption requiring validation:** Vendors need at least the same core business logic to run in multiple SSP environments. A single source tree and preferably a single OCI image, with host-specific configuration where necessary, is a useful portability target. It is not guaranteed that every SSP offers the same data, extension point, networking policy, or managed service.

## 2. What external evidence tells us

### 2.1 ARTF's portability contract is explicit—but limited

The IAB Tech Lab ARTF v1 specification emphasizes OCI containers, a standardized orchestration interface, limited network access, and named dependencies. An agent manifest identifies its required services by name; the host maps that name to its implementation. The specification says the container **should not start** if the orchestrator cannot support its requirements and intents. It does **not** standardize a cross-host KV product API, a state model, or a shared persistent namespace. [S1]

**Implication:** PubMatic can expose proprietary services as optional, declared dependencies without implying they are part of baseline ARTF. Required business dependencies need an admission-time decision, not an unobservable runtime no-op.

### 2.2 Cloudflare shows how good developer tooling feels

Cloudflare's Workers bindings give application code resource handles without embedding raw access keys. Wrangler local development defaults to locally simulated resources, with opt-in remote bindings for supported services. The same code-level API is available locally and in production. [S2][S3]

**What transfers:** declarative bindings, typed configuration, resource emulation, persistent local development state, and explicit remote-resource opt-in. **What does not automatically transfer:** control over a single runtime, binding APIs backed by one platform's semantics, and the assumption that application developers willingly target only that platform.

Also, an apparently simple method such as `KV.get` does not imply universal data semantics. Cloudflare Workers KV is eventually consistent and explicitly not designed for transactional read-modify-write patterns. [S8]

### 2.3 Index Cloud is the practical interoperability benchmark

Index Cloud markets a partner-controlled container model aligned with ARTF. Its September 2026 build guidance prohibits runtime egress for its containers, including external API or database calls during request handling, and requires packaged or platform-mounted local data. Its model-update flow supports versioned files mounted at a declared path, with validation and redeployment. [S4][S5][S6]

**Implication:** A PubMatic example whose business logic requires synchronous Decision Fabric KV could be structurally incompatible with an otherwise valuable alternate host. A portable snapshot-based example is far more likely to satisfy both platforms, while PubMatic could still differentiate through better data ingestion, distribution, and refresh orchestration.

### 2.4 Standards-based building blocks already exist

OpenTelemetry provides common telemetry conventions and a collector pattern that can route data without making the application import a host-specific logging library. Dapr demonstrates pluggable state APIs over multiple backends and local/cluster operation, though its sidecar overhead and operating model need measurement before any hot-path adoption. [S9][S10][S11]

**Implication:** Prefer established open APIs when they cover the problem. A proprietary SDK should add concrete functionality unavailable through standard APIs, not duplicate them solely to get a branded method name.

## 3. Why an SDK alone is insufficient

Consider `decisionfabric.KV.Set("vendor-key", value)`. To fulfill that call safely in production, Decision Fabric must decide which vendor, application, environment, region, data namespace, quota, and backing service the call belongs to. It must also know whether this container has permission to mutate that resource and how it should behave when the store or network is impaired.

**Underlying platform obligations:**

1. **Resource lifecycle:** define, approve, provision, modify, version, migrate and decommission each binding and its backing resources.
2. **Ownership model:** bind storage to a **vendor organization + app + environment + region/replication policy**; never assume one store per individual developer or one global store per vendor.
3. **Workload identity:** authenticate the deployed workload through a platform-injected, short-lived identity or local trusted proxy; do not accept a tenant ID claimed by the untrusted application as authorization.
4. **Entitlements:** permission check each operation, with least-privilege, protected namespace separation and auditable policy changes.
5. **Service discovery:** inject a stable binding name/endpoint and let infrastructure map it to eligible instances, rather than hard-coding server IP addresses in examples.
6. **Data behavior:** define max value size, key encoding, TTL, consistency, read-your-writes, overwrite/CAS policy, durability, replication, and failover behavior.
7. **Admission and rollout:** reject incompatible deployments before auction traffic is routed, or make a supported fallback selection explicit.
8. **Runtime protection:** enforce request timeout, bounded connections, backpressure, quotas, circuit-breaking policy and protection of other tenants.
9. **Observability and commercial controls:** meter usage, provision SLA reports, detect unusual usage/cost, surface errors and provide billing/audit trails.
10. **Developer parity:** provide a local implementation with meaningful conformance tests, fixtures and fault simulation.

In other words, the SDK project creates a **platform-services product**. Its operational ownership, SLO, costs, support load, and roadmap must be funded separately from writing the client library.

## 4. Separate services by business criticality

| Service class | Examples | Missing-service policy |
|---|---|---|
| **A. Observability conveniences** | Structured logs, spans, optional debug annotations, SDK-level counters | Can degrade to stdout/OTel or a clearly reported no-op for noncritical decoration. |
| **B. Deployment conveniences** | Typed env vars, static config, secret injection, health/ready helpers | Missing optional data gets explicit defaults; missing required secrets/config block readiness. |
| **C. Data delivery** | Versioned CSV/model snapshots, atomic mounted refresh, data-integrity checks | Required snapshot missing/stale: no readiness or business-defined safe outcome. |
| **D. Core online state** | Audience eligibility KV, decision counters used for gating, deal state | Required binding missing: block startup or deliberately switch to a tested alternate provider. Never silently return an empty audience set. |
| **E. Asynchronous platform services** | Bulk analytics export, events, refresh scheduling, report delivery | Queue/buffer within limits or degrade according to a documented delivery contract. |

**Design rule:** failure behavior should be a property of the capability contract, not of the brand of SDK. “Optional” does not mean “safe to ignore.” An optional service might still have important correctness implications for some applications.

## 5. The portability matrix the proposal must pass

| Scenario | Outcome if PubMatic calls are hard-coded | Better contract |
|---|---|---|
| PubMatic deployed; KV healthy | Appears convenient | Required capability resolved; behavior and SLO explicit |
| PubMatic KV temporarily unavailable | Uncontrolled empty result, hanging RPC, or timeout | Bounded error; business policy (cached result/no-bid/readiness) |
| Same OCI image runs on Index Cloud with no online lookup | Core audience logic may stop working | Snapshot + in-memory provider, same business interface |
| Same image runs on another ARTF host with its own KV | PubMatic method name cannot discover host KV | Host-neutral interface with alternate adapter, if available |
| Host offers telemetry but not PubMatic SDK | Missing diagnostic events or overhead | Stdout + OTel export conventions |
| PubMatic entitlement revoked | Undiagnosed runtime failures | Admission/reconciliation rejects missing required binding; clear error |
| Local Docker development | Production binding unavailable | Local service simulation + seeded fixture data |
| Local tests use real production data accidentally | Security/data-integrity incident | Local-only default; remote opt-in with safe sandbox scope |
| Cross-region rollout has inconsistent data versions | Bid decisions differ by data center | Versioned dataset metadata, checksum, freshness and rollout gates |
| “Empty result” is a legitimate business result | Cannot distinguish data absence from infrastructure error | Typed errors/status; never conflate not-found with unavailable |
| Vendor uses two SSPs with divergent data copies | Different deal eligibility decisions | Clear host-local state boundary or vendor-controlled dataset replication |
| Vendor leaves Decision Fabric | Cannot move code/state cleanly | Exportable data, portable domain logic, documented migration path |

**Crucial distinction:** Portable *code* does not guarantee portable *state*, and portable *API signatures* do not guarantee portable *consistency, latency, authorization, or permitted network behavior*.

## 6. Preferred architecture: portable core + optional host capabilities

Use layered contracts with no PubMatic types in core decisioning logic:

```text
  IAB ARTF request  -->  gRPC handler  -->  Vendor business logic
                                           |
                                    AudienceData interface
                                       /            \
                         Snapshot/in-memory       Managed-KV adapter
                         (portable default)       (optional PubMatic mode)
                                       \            /
                               Explicit provider selection

  Decision Fabric control plane:
    Vendor/App registry --> Capability declaration --> Admission + entitlement
       --> Resource provisioning --> Runtime binding --> Usage/SLA metering

  Decision Fabric runtime plane (when capability enabled):
    Vendor workload --> local/nearby capability endpoint --> tenant-scoped service
```

**Core SDK vs host adapter.** `AudienceData.Lookup(ctx, key)` should belong to the vendor's domain package or a small host-neutral helper, not the PubMatic SDK. A PubMatic adapter can be published independently. A single Go binary may include several adapters and select one at startup through validated runtime configuration. An alternate packaging pattern is a host-specific wrapper image, but that weakens the single-image goal.

**Illustrative Go-like API (proposal, not an existing API):**

```go
type AudienceData interface {
    Lookup(ctx context.Context, key Key) ([]DealID, error)
}

func SelectAudienceData(cfg Config, caps Capabilities) (AudienceData, error) {
    switch cfg.DataProvider {
    case "snapshot": return OpenVerifiedSnapshot(cfg.SnapshotPath)
    case "df-kv":
        if !caps.Has("kv.read") { return nil, ErrRequiredCapabilityUnavailable }
        return NewDFKVAdapter(caps.Binding("audience_data"))
    default: return nil, ErrUnsupportedProvider
    }
}
```

**Avoid embedding host decisions in the handler:** `GetMutations` should call `AudienceData.Lookup` and apply explicit timeout/error decisions. It should not contain `if PubMatic { ... } else if Index { ... }` branches.

### Binding contract example (proposed extension, NOT an ARTF standard schema)

```yaml
# decision-fabric.yaml -- illustrative only; field names not finalized
app: audience-provider
required_capabilities:
  - name: audience_data
    kind: snapshot.read
    dataset: eligibility-v3
    max_staleness: 24h
optional_capabilities:
  - name: kv-aux
    kind: kv.read
    consistency: eventual
  - name: telemetry
    kind: otlp.export
```

The ARTF manifest's declared dependency names must remain compatible with its actual manifest syntax; a separate product-specific configuration or documented manifest extension can carry richer semantics until a cross-industry binding vocabulary exists. Do not present custom keys as IAB-standard fields. [S1]

## 7. Runtime data-path options for audience lookup

The business example is `(device ID, content ID) -> deal IDs`, potentially refreshed daily, with sub-5-ms end-to-end target. The decisive comparison is **what happens on every auction request**, not merely whether a service is convenient to call.

| Pattern | Fit for hot path | Tradeoff / best use |
|---|---|---|
| Packaged CSV loaded into in-memory map on startup | **High** for bounded data | Simple, portable; image rebuild required for data change |
| Versioned dataset mounted, atomically swapped, read in-memory | **High** | Strong candidate: update data independently of code; must implement readiness, versioning and refresh verification |
| Process-local cache hydrated from PubMatic-managed snapshot | **High after hydration** | Platform differentiation in ingestion and delivery; local memory and startup behavior matter |
| Shared Redis/Aerospike/managed KV on each request | **Conditional** | Extra RPC, failure mode and QPS multiplier; potentially attractive for very large/high-churn data |
| Remote database/API lookup outside the host on each request | **Low / often disallowed** | Network and ARTF policy conflict; latency and reliability concerns |
| Asynchronous writes from the auction path | **Conditional** | Must separate durable/event delivery from synchronous bidder latency |

**Performance reasoning:** If a service performs one managed-KV operation per request at **100,000 requests/second**, that creates **8.64 billion KV operations per day** before replicas, retries or extra reads. This is an illustrative traffic scenario, not a PubMatic usage estimate. A few hundred microseconds of additional p99 overhead can be material in an auction budget. Compare p50/p95/p99/p99.9, QPS under N-1 capacity, cost/op, payload size, and tail spikes with identical business data; do not decide from an average latency or an SDK microbenchmark alone.

**Strong recommendation for the initial audience example:** make versioned, verified snapshot ingestion + in-memory lookup the default; demonstrate an optional managed KV variant only when freshness, cardinality or memory economics warrant it. Never let the “simple API” hide a prohibited or costly request-path network dependency.

### 7.1 Kubernetes: namespace boundary versus actual network distance

A specific concern is whether a vendor container inside `vendor-a` namespace calling PubMatic KV in `decision-fabric-services` creates extra network communication compared with that container calling its own Aerospike in `vendor-a`. **Namespace separation does not, by itself, add a physical hop, nor does sharing a namespace guarantee co-location.** Kubernetes Services are resolvable across namespaces, and the pod network is cluster-wide subject to network policies. [S13][S14]

- **Same process:** a Go map or memory-mapped file avoids network RPC entirely. Its costs are process memory, data warm-up, and replication across each serving pod.
- **Same pod:** a database/cache process or proxy in a sidecar shares the pod network namespace, so `localhost` is available; it still incurs serialization, IPC/TCP and its own CPU/memory usage. [S14]
- **Different pods on the same node:** service networking and client/server RPC apply even if both pods use one Kubernetes namespace. How the CNI, service implementation and encryption affect latency must be measured.
- **Different nodes, one data center:** network traversal and possibly service-proxy or overlay costs apply whether the pods have the same namespace or not.
- **Different zones / racks / data centers:** the latency and availability penalties can outweigh the distinction between namespaces; placement/locality and failover policy become design inputs. Kubernetes provides scheduling affinity and topology-aware routing mechanisms, but does not automatically enforce optimal co-location. [S16][S17]

For example, `kv.decision-fabric-services.svc.cluster.local` is a normal cross-namespace service DNS name, **if network policy grants access**. A default-deny policy must allow only the authorized vendor workloads to reach the gateway. **Do not use namespace labels alone as the authentication boundary:** a workload must also be bound to its authorized logical data resource at the server. Kubernetes NetworkPolicy limits reachable endpoints; it is not a database authorization mechanism. [S13][S15]

**Engineering question:** compare actual (a) process-local map, (b) same-pod local proxy or datastore, (c) same-node vendor Aerospike, (d) same-node shared PubMatic KV, (e) cross-node within DC, and (f) under server degradation. The conclusion should follow measured p99/p99.9 and operational cost, not namespace names.

### 7.2 Architecture options for the exact audience CSV example

Consider a vendor-supplied table `(device_id, content_id) -> deal_id[]`, delivered as a CSV inside a container image, read on every ARTF request. There is no evidence yet that this workload requires online mutability.

| Option | Where authoritative live data resides | Per-auction access | Strengths | Material cost / complication |
|---|---|---|---|---|
| **A. Embed CSV; load Go map at pod startup** | Image and each process's memory | Process-local | Portable, simple, no extra RPC | Rebuild on updates; memory multiplied by replicas; parser/startup time |
| **B. Platform delivers versioned file; process loads it** | Versioned mounted artifact and process memory | Process-local | Separates data updates from code; offline auction path; portable business logic | Artifact validation/fan-out, regional convergence, per-pod memory |
| **C. Process-local read-only DB / mmap** | Mounted binary/immutable local file | Local file / memory-mapped reads | Potentially lower heap/fast restart with well-designed indexes | Format/build pipeline, mmap/page-fault p99 risk, read-only constraints |
| **D. Vendor-owned Aerospike/Redis service** | Vendor-operated pods/volumes, typically alongside app | One or more RPCs | Control of backend/semantics; reusability across workloads | Vendor DB lifecycle, persistence, backup, HA, licensing, service QPS, host restrictions |
| **E. PubMatic shared managed KV** | Shared regional backend, logically partitioned by tenant | One or more RPCs | Central ops, capacity pooling, self-service binding | Platform-wide cost, tenant isolation, noisy neighbors, quotas, latency and portability |
| **F. PubMatic dedicated managed KV per vendor** | Per-vendor database/cluster | One or more RPCs | Stronger isolation and custom scale policy | Underutilized resources, per-vendor lifecycle and SRE workload |
| **G. Local cache backed by managed KV** | Local memory + remote backend | Cache hit local; miss RPC | Good for highly skewed hot keys and mutable data | Invalidation and miss storms, cold-start load, consistency and memory overhead |

**Do not infer that option D avoids a network call merely because the database is in the vendor namespace.** A separate Aerospike pod is a separate server; the physically relevant placement is whether it shares the application pod/node, not the namespace label. Option A, B, or C removes the per-auction RPC entirely.

For an auction extension point with external-egress prohibition, even a reachable, internally hosted database may violate a host's particular rules; Index Cloud explicitly says an auction container may not perform outbound network calls at runtime and instead supports locally mounted model data. This is an **Index-specific documented policy**, not a claim that every ARTF host prohibits internal managed services. [S4][S5]

### 7.3 Operational questions before permitting vendor-owned Aerospike

Running the vendor's own Aerospike in its namespace may sound attractive but requires an explicit hosting model: does Decision Fabric allow arbitrary multi-container deployments, StatefulSets, persistent storage, network services and database images, or only a constrained OCI ARTF execution unit? Who owns upgrades, CVEs, replication, recovery, licensing, storage I/O isolation, and incident response? Does a misconfigured database consume shared host network or memory headroom? Are sensitive audience keys allowed in persistent volumes? Does every vendor need a distinct datastore, or can one datastore support multiple vendor apps?

If the answer is that PubMatic will operate the vendor's database on its behalf, the difference between “vendor-owned Aerospike” and “PubMatic-managed KV” may be predominantly **control plane, isolation and billing**, not a fundamentally different data-plane path. This is a strong argument **for** offering managed infrastructure—but only if the use case merits a separate service at all.

### 7.4 Simple traffic and economics model (illustrative, not measured)

Assume one lookup per auction invocation at **100,000 requests/second** for 24 hours: `100,000 x 86,400 = 8.64 billion KV operations/day`. If each request plus response transfers **2 KiB**, the aggregate logical request/response payload is approximately **17.7 TB/day** before protocol, replication, retries and east-west network overhead. At **0.25 milliseconds of combined CPU time per call across client/proxies/service**, the extra synchronous path consumes roughly **25 sustained CPU cores** (`100,000 x 0.00025`) even before replication, storage and on-call overhead. These inputs are examples for sensitivity analysis, **not Decision Fabric measured data**.

A usable total-cost comparison should include: `compute for hot-path lookups + memory for replicas + storage and replicas + data-distribution traffic + network/proxy CPU + operators/on-call + failover headroom + security/compliance + vendor support + billing/metering`. A shared backend may save per-vendor idle capacity; a local in-memory dataset may save billions of RPCs; neither is always dominant.

**Break-even variables to collect:** data bytes and in-memory expansion factor; unique keys; request QPS per vendor and hot-key distribution; per-DC serving replicas; refresh frequency; desired cross-region consistency; effective cache hit ratio; hardware capacity; the vendor's willingness to pay; operational incident rate. Include **N-1 capacity** and p99.9 during background dataset refresh, not just steady-state averages.

### 7.5 When online KV genuinely adds value

Managed online state is defensible for frequent small writes that must become visible to multiple replicas, datasets too large to duplicate per application replica, high-churn control flags, coordinated caps/counters (if the chosen store actually provides correct atomicity), and shared mutable state across a vendor's own workloads. It is much less compelling for a static daily CSV used for deterministic membership checks. Do not call a volatile cache a replacement for a durable system of record, or advertise atomic counters with an eventual-consistency-only primitive.

## 8. How the managed-KV binding would actually work

### 8.1 Provisioning and mapping

Proposed entities: `VendorOrg`, `App`, `Deployment`, `Environment`, `Region`, `CapabilityBinding`, `BackingResource`, `Entitlement`, `UsageMeter`. An infrastructure owner manages the backing service; the external contract remains a stable vendor-facing binding.

A vendor declares a logical name such as `audience_state`. The control plane checks its allowed operations and region, allocates or references a tenant-isolated namespace, applies quota/storage policy, records ownership, and issues a binding descriptor at deploy time. The workload sees the **logical binding and local endpoint**, never internal Aerospike/Redis cluster topology. Across revisions or restarts the logical binding remains stable even if backend endpoints change.

### 8.2 Authentication and isolation

Prefer platform-injected short-lived workload identity with audience restrictions, or a strictly scoped local proxy that authenticates the pod without a hard-coded vendor secret. Kubernetes supports projected, short-lived, bound service-account tokens as one building block; these must not be confused with a complete external-vendor entitlement system. [S12]

Enforce tenant/resource namespace on the **server side**. The call `Get("x")` cannot authorize access merely because the client passes a vendor ID or environment string. Apply encryption in transit, audit logging, rate limits, key/value privacy classification, resource deletion and data retention policies.

### 8.3 Data model and service guarantees

Before implementation, define: read/write/scan/batch operations; TTL and invalidation; key and value max size; namespace cardinality; consistency per region; read-your-writes expectations; atomic update or CAS availability; persistence and recovery; regional replication; backup/restore; failover; hot-key behavior; request deadlines and clear error taxonomy (`NotFound`, `Unavailable`, `PermissionDenied`, `RateLimited`, `DeadlineExceeded`).

A **cache** with tolerated eviction is not a **durable record store**. An **eventually consistent** namespace is not a correct substitute for state that requires immediate consistency. Naming everything “KV” can create more support risk than value.

### 8.4 Latency budget and overload policy

No unbounded retries inside an auction handler. A managed service call must honor remaining auction deadline and have an explicitly measured, bounded budget. If retries are supported, define whether they are client-side, proxy-side, or transport retransmissions, and demonstrate that duplicated operations cannot produce incorrect state changes. Use circuit breakers, admission, bulkheads, and overload signaling based on actual failure modes; track request amplification. A failure should not be converted into `NotFound`.

### 8.5 Who loads the vendor CSV? Four very different meanings of “on boot”

**Pattern 1: The vendor's app reads its own embedded CSV.** Each serving pod loads and verifies the data before readiness reports `SERVING`. There is no PubMatic ingestion into a remote KV. This is the simplest consistent behavior, and it can run on hosts that prohibit all request-time egress. Cost: repeated memory and load time per replica.

**Pattern 2: A Kubernetes init container downloads/prepares a read-only local file.** It can put data in a shared volume before the app starts; however, an init container runs **for each pod initialization**, not once for the overall vendor deployment. It is not by itself an appropriate mechanism for global “once-only” KV bulk provisioning. [S18]

**Pattern 3: Platform-controlled Kubernetes Job loads a managed KV dataset before application rollout.** A controller creates a **dataset-version-keyed ingestion Job** (or external ingest worker), and only admits serving pods after the dataset is validated and published. All retries must be idempotent, and repeated deployments of the same image must not duplicate/partially overwrite data. This is feasible but requires a complete control-plane/data-asset design.

**Pattern 4: Vendor uploads an independent versioned data artifact.** PubMatic validates and distributes it to each target DC/serving environment, makes it available at a declared read-only path, and publishes the revision atomically. This decouples business code from data transfer and preserves the fast local lookup pattern. Index Cloud documents a comparable partner-data approach, so this is a credible requirement and competitive baseline—not a claim of unique differentiation. [S5]

**A major unanswered question:** If the CSV exists *only inside the OCI image*, how does the PubMatic ingest service read it? Possibilities include a separately uploaded artifact, a controlled image-extraction task, or a vendor-defined command producing the dataset. Each has security and lifecycle implications. Merely deploying an image does not automatically expose its filesystem to a central KV loader.

### 8.6 Reference control-plane flow if platform-managed KV is selected

```text
Vendor portal / API                  Decision Fabric control plane
  register OCI image + dataset  -->  check org/app/region/entitlement
  declare logical binding       -->  create tenant-scoped resource mapping
  submit dataset revision       -->  verify digest, format, privacy, quota
                                 -->  provision KV capacity (if absent)
                                 -->  launch import Job outside auction path
                                 -->  validate row counts, CRC, sample queries
                                 -->  publish immutable version / active alias
                                 -->  write deployment binding and readiness gate
                                 -->  rollout app pods to authorized environment

Vendor pod (namespace vendor-a)       Decision Fabric services namespace
  ARTF gRPC handler -- Get(key) -->  approved service DNS / binding gateway
                                       authenticate workload identity
                                       authorize (org/app/env/dataset, read)
                                       route to region-local vendor partition
                                       enforce quota and call deadline
                                 <--   value | not-found | typed error
```

**Logical resource identifiers:** `tenant_id`, `application_id`, `environment`, `region`, `binding_name`, `dataset_version`, `backing_resource_id`, `entitlement`, `quota_policy`, and `rollout_state`. Expose a vendor-level logical binding (not hard-coded cluster DNS) and keep the internal mapping authoritative in the control plane. This must survive pod recreation and application upgrades without leaking another tenant's credentials or partition.

### 8.7 Data correctness and rollout timeline to specify

1. **Authorize and verify source:** image/artifact checksum, provenance, approved schema, dataset legal classification, key cardinality and tenant ownership.
2. **Create a candidate revision:** parse CSV with deterministic escaping/key normalization; validate duplicates, nulls, maximum rows, row errors and expected key sizes; do not mutate the currently serving version in place.
3. **Load and reconcile:** throttle the bulk writer independently from live auction QPS; verify row count, checksums, lookup samples and storage capacity; treat retry as an idempotent write to the same candidate revision.
4. **Publish only after success:** atomically switch a tenant's active dataset pointer or roll to a new immutable snapshot. If atomic publication is not supported, use a documented staged cutover with visibility and rollback.
5. **Gate readiness:** serving pods must report healthy only when they see an acceptable revision and the specified maximum data age; distinguish `NoMatch` from `Unavailable` and `StaleDataset`.
6. **Observe regional progression:** record the active revision per DC, startup time, lag and error rate; define whether regional skew is temporarily permitted and how long.
7. **Delete safely:** define retention, old-version garbage collection, vendor offboarding, and privacy deletion paths, including snapshots/backups and replicas.

**Scaling trap:** if 300 pods start simultaneously, 300 importers must not each bulk-load the same 10 GB CSV into one shared service. Conversely, a single initial importer must not become a single point of failure. The coordination and re-run semantics belong in a reconciler/Job, not ad hoc application startup logic.

### 8.8 A safer service exposure boundary

Place the vendor-facing binding gateway **outside PubMatic's internal SSP data stores**, even if its backing KV physically shares colocation hardware. Give the gateway a dedicated contract, resource names, SLO and security perimeter. Do not expose internal Aerospike keyspace, tenant IDs, or schema. Prefer a region-local path; do not permit inadvertent cross-DC fallback for per-request calls unless the caller's timeout, data-residency contract, and economics explicitly allow it.

A per-pod sidecar may simplify local identity handling or caching, but it does not remove the remote service call on a cache miss. A shared gateway can reduce per-pod overhead but becomes a multi-tenant scaling/incident domain. Direct client connections can be lowest overhead yet create more SDK/version/credential exposure. Benchmark all three with strict isolation.

## 9. Local developer experience: Wrangler-like, not Wrangler-dependent

The proposal needs one command that launches an ARTF example plus its **local capability substitutes**, but without giving an illusion that a local emulator reproduces an entire production region.

| Mode | Behavior | Guardrail |
|---|---|---|
| **Unit** | In-process fake store or seeded snapshot; deterministic tests | No network, no secrets, known fixture semantics |
| **Local integration** | Docker Compose with a small KV emulator/store, OTel collector, test ARTF caller | Persistent data under ignored dev directory; test failure modes |
| **Staging remote** | Explicit opt-in to a PubMatic sandbox binding | No production namespace; expiring least-privilege access and clear prompt/config |
| **Production** | Platform-injected binding based on declaration and entitlement | No developer-managed production access keys inside image |

**Illustrative CLI experience (a product proposal, not available today):**

```shell
df dev up                  # Docker Compose + local bindings + fixture dataset
df test artf               # Exercise IAB gRPC request/response and health checks
df test capabilities       # Check missing bindings, stale data, quota and auth failures
df validate                # Manifest, image, privacy and portability checks
df deploy --dry-run        # Plan resources and admission; do not deploy
```

Local state should be isolated by project, app and environment; the command should seed controlled fixtures and provide reset/export. Production-like tests must inject `Unavailable`, delays, wrong permissions, stale snapshots, quota violations, partial region rollout and process restart. A remote connector must be opt-in and clearly labeled; Cloudflare's default local simulations provide a useful precedent. [S3]

## 10. How periodic internet refresh should work

ARTF's default egress constraints mean an example that downloads arbitrary CSV files directly from the internet cannot be assumed to work unchanged in every SSP. The IAB document expects constrained egress, with exceptions negotiated by host and vendor. [S1]

Prefer an **out-of-band data pipeline**:

1. Vendor declares an HTTPS/object-storage source, update frequency, regions, schema, size limits and privacy policy through a control-plane mechanism.
2. An approved ingestion job, *outside* the auction container, downloads data through controlled egress, checks signatures/checksums, validates schema, scans and versions the artifact.
3. Decision Fabric distributes the approved artifact to each DC as a read-only snapshot/volume, without exposing PubMatic's internal object-store details.
4. The container validates version/checksum and swaps its in-memory index atomically, or is rolled to a new revision. The prior valid dataset stays available during recoverable refresh issues within an agreed freshness window.
5. Telemetry reports data age, version, row count, validation errors, rollout convergence and source-health. A breach of required freshness affects readiness or invokes an agreed business-safe policy.

This can make PubMatic attractive **without adding a proprietary method to the per-auction code**. Similar externally managed model-data delivery is already publicly described by Index Cloud, so differentiation should come from smoother integration, cadence, reliability, richer supported formats and measured operations—not merely the existence of mounting. [S5]

## 11. Should PubMatic build a sidecar, client library, or neither?

| Option | Advantages | Concerns / disposition |
|---|---|---|
| **A. Portable examples + conventions only** | Quick value, low adoption friction, no lock-in | Limited unique platform capability. **Build baseline.** |
| **B. One PubMatic Go SDK with silent no-op elsewhere** | Simple demo and marketing story | Hidden correctness failures and portability anxiety. **Do not choose.** |
| **C. Optional typed Go adapters + capability descriptor** | Incremental; portable business layer and a single image possible | Go-specific integration; provider parity to maintain. **Pilot.** |
| **D. Standard local gRPC/HTTP capability endpoint with thin language clients** | Supports Go/Rust/Java; replaces embedded secrets and host topology | Runtime service, auth, backpressure and versioning overhead. **Preferred longer-term interface where needed.** |
| **E. Dapr-like pluggable sidecar for all service calls** | Broad portable abstraction; existing open architecture | Extra hop/resource footprint, host must run it, unclear auction-path fit. **Evaluate only if measurement justifies.** |
| **F. Platform-delivered assets plus OTel, no custom runtime SDK** | Highly portable and low hot-path overhead | No synchronous mutable state. **Strong default for first release.** |

**Pragmatic hybrid:** A + F immediately; add C for one well-defined optional capability; evolve toward D if more languages and vendor use cases justify a stable service protocol. Do not force every ARTF vendor to ship a PubMatic package just to emit logs or read a local file.

## 12. Arguments and counterarguments for leadership discussion

**Claim: “Cloudflare proves developers love bindings.”**  
**Counter:** It proves integrated runtimes and tooling can be compelling. Workers and portable ARTF containers have different portability economics. **Reconciliation:** copy the declarative configuration and local workflow, but measure whether vendors voluntarily adopt host-specific capabilities.

**Claim: “The SDK will make developers sticky.”**  
**Counter:** When cross-SSP deployment is necessary, the same dependency may discourage initial adoption. A vendor may write an abstraction to isolate or avoid the library. **Reconciliation:** optimize deployment preference through lower operations cost and better business results, and measure adoption rather than assuming lock-in.

**Claim: “It will fail silently on other SSPs.”**  
**Counter:** Silent degradation cannot tell `no eligible audience` apart from `data unavailable`. It is unacceptable in revenue-affecting logic. **Reconciliation:** observability convenience can degrade transparently; required state must fail admission or use an explicitly configured, tested provider.

**Claim: “Writing a Go SDK is inexpensive.”**  
**Counter:** The SDK is the easy part; tenancy, data plane, SLAs, local simulation, support, security and capacity dominate cost. **Reconciliation:** cost the complete product before committing, and pilot one managed resource rather than a services catalog.

**Claim: “We can use a shared Redis/Aerospike cluster.”**  
**Counter:** Shared capacity requires tenant isolation, request-cost accounting, noisy-neighbor protection, key quotas, identity, schema and recovery. **Reconciliation:** prototype with isolated logical namespaces, quotas and audited access; compare per-tenant cluster overhead with pooling economics.

**Claim: “One KV API supports everyone.”**  
**Counter:** It does not standardize consistency, TTL, eviction, cross-region state, scan behavior or permitted network paths. **Reconciliation:** advertise a narrow named capability (e.g., `kv.read.eventual.v1`) with a testable contract, not generic “KV.”

**Claim: “Our existing examples already do 60% of the work.”**  
**Counter:** That is useful, but percentage is subjective; runtime and operational readiness may dominate the remaining workload. **Reconciliation:** instrument real time-to-first-success and developer effort by task, rather than use a guessed completion percentage.

### 12.1 AI-assisted container development changes the value proposition

**Claim in favor of caution:** In 2026, vendors can use coding assistants to generate Go, Dockerfiles, K8s manifests and ARTF handlers from a specification, often reducing the value of generic scaffolding. Open-source gRPC, Protobuf, OpenTelemetry and container tooling already exist. A branded SDK that only saves boilerplate may not materially change the decision to integrate with an SSP. **This is a hypothesis to test, not a quantified claim about all vendors.**

**Strongest counterargument:** AI can generate code, but it does not grant authorized access to production data, produce service SLOs, satisfy a particular host's no-egress/security rules automatically, distribute a new audience dataset across regions with a verified freshness guarantee, or remove the operational burden of monitoring and recovery. Verified working examples, compatibility fixtures, automated conformance tests, stable sandbox environments and self-service managed services still remove *operational* complexity that a generated code sample cannot.

**Proposed product implication:** Design the developer kit to work well with **both human and AI coding agents**. Provide concise machine-readable protocol contracts, example golden requests/responses, a conformance checker, explicit forbidden behaviors, capability version schema, Docker-based integration tests and failure fixtures. Test whether developers or coding agents can produce a conformant audience container from scratch without PubMatic-specific assistance, and compare with using the curated starter and managed-service pilot.

**Measure the right friction:** first conformant local test, first successful deployment, number of undocumented platform interventions, production latency/reliability, time-to-dataset-refresh, and ongoing support burden. If scaffolding takes minutes with AI but onboarding still takes weeks due to access approvals, deployment certification, dataset distribution and unclear telemetry, fix those bottlenecks first.

## 13. Priority services and product surface

| Priority | Candidate | Why now / condition |
|---|---|---|
| **P0** | ARTF Go starter, official proto integration, conformance test harness | Portable prerequisite; usable even if no SDK launches |
| **P0** | Structured stdout logging, OTel metrics/traces, auto runtime dimensions | Standards-based; low vendor effort; host supplies collection and dashboards |
| **P0** | Declarative config, validated secrets, capabilities at deployment | Prevents surprises; simplifies onboarding |
| **P1** | Versioned dataset ingress, staged fan-out and read-only mount | Best fit for audience and model examples; improves freshness operations |
| **P1** | Self-service lifecycle, build/test/deploy, quota/usage/SLA visibility | Operational convenience more defensible than SDK naming |
| **P2** | Optional read-only managed KV for specific sizes/churn patterns | Only after workload demand and hot-path performance validate it |
| **P2** | Async analytics/events/feedback stream | Useful outside critical request path; delivery and privacy contracts needed |
| **Later** | Mutable transactional KV, queues, object store, remote inference API | Broad surface and complex guarantees; justified case-by-case |

**Logging proposal:** export host-neutral structured logs and OTel signals by default. The optional `decisionfabric.Log` facade, if offered, should remain a thin compatibility adapter and must not be the only way to observe a container. [S9][S10]

## 14. Threats, failure modes, and mitigations

| Risk | Impact | Preventive design / test |
|---|---|---|
| Core binding missing on another host | Wrong targeting or empty business result | Admission-time required/optional declaration; portably selected provider |
| Latency tail from managed KV | Auction deadline misses / circuit breaker | Per-call deadline, p99.9 load test, N-1 conditions, fallback policy |
| Tenant escape / key collisions | Cross-vendor data disclosure | Server-side workload identity, scoped namespaces, authorization tests |
| Stale or partially delivered dataset | Regional targeting divergence | Signed/versioned snapshots, checksums, freshness threshold, atomic rollout |
| Local emulator differs from production | Bugs found only at production scale | Shared contract tests, sandbox integration, fault injection |
| Silent no-op masks errors | Revenue discrepancy without alerts | Typed errors, health status and observable degraded state |
| Resource misprovisioned or removed | Runtime interruptions | Reconciliation controller, stable binding names, explicit migrations |
| Noisy neighbor and hot keys | Wide incident blast radius | Quotas, rate limits, isolation, overload shedding, per-tenant dashboards |
| Retry amplification | Cascading overload | Bounded budgets and explicit retry ownership; idempotent writes where required |
| High-cardinality telemetry and privacy | Cost increase / data exposure | Cardinality budgets, scrubbed fields, retention and allowed metadata |
| Multi-SSP state divergence | Inconsistent decisions across partners | Prefer source-of-truth snapshots; document host-local state boundary |
| Product roadmap overwhelms platform | Maintenance and delayed core ARTF | Stage gates, named service owner, adoption-based investment |

## 15. Pilot and evidence required before scaling

**Pilot 1 — Portable audience provider:** Same Go source and preferably the same OCI image runs locally, on Decision Fabric and in an ARTF-compatible alternate-host test environment (or a faithful constraint simulator). The default business behavior uses versioned snapshots with zero network call per auction. Compare build effort, startup time, memory footprint, freshness, throughput and auction latency.

**Pilot 2 — Optional managed KV:** Add a separately configured `df-kv` adapter. Test a genuinely needed online data case; measure the *incremental* tail latency and cost of the service (including proxy, storage, retry and regional behavior). Show that a missing required binding prevents deployment, and that a configured alternative operates elsewhere. Prove isolation and overload protections.

**Pilot 3 — Developer experience:** Ask at least three vendor profiles—audience data vendor, bidding/model vendor, and analytics/measurement vendor—to implement the same scoped task from (a) the current templates and (b) the new toolchain. Record time spent, steps requiring PubMatic assistance, documentation gaps, failed tests, and willingness to keep the integrations in their production build.

### Proposed decision gates (targets to agree, not observed outcomes)

- **G1 / Demand:** multiple design partners independently identify a managed service they would use on PubMatic *despite* multi-SSP requirements.
- **G2 / Portability:** same business logic passes conformance scenarios locally and against at least one alternate host constraint profile; absence of an optional capability never changes core behavior without a declared fallback.
- **G3 / Reliability:** required capabilities fail closed at admission/readiness, and data/state errors never masquerade as legitimate `NotFound` results.
- **G4 / Performance:** demonstrable sub-budget p99/p99.9 incremental overhead at target load, with N-1 capacity and failure injection. Numeric budgets must be jointly agreed before testing.
- **G5 / Economics:** fully loaded per-operation and platform-operations costs fit a documented pricing/metering model; no uncontrolled tenant amplification.
- **G6 / Developer productivity:** measurable improvement against current examples, such as a proposed **30% reduction** in median time-to-first-correct-deployment, subject to baseline data and leadership approval.
- **G7 / Ownership:** control plane, data plane, security, SDK, docs, local tools, SRE and partner support have named owners and an on-call model.

If G1, G4, or G5 do not pass, retain the successful portable tooling and snapshot/telemetry integrations rather than turning the experiment into a broad managed-state program.

### 15.1 Scenario-based decision tests (explicit pass conditions to agree)

| Scenario / injected condition | Expected product behavior | What a demo must prove |
|---|---|---|
| Run the unchanged OCI image on Index-like no-egress host | Correct business output via embedded/mounted data provider | No hidden PubMatic KV requests; no reliance on outbound network |
| Required KV binding unavailable at deploy | Deployment blocked or readiness held; never fake an audience miss | Clear diagnostic with remediation |
| Optional telemetry endpoint unavailable | Core auction logic continues within budget; telemetry drop accounted for | Bounded nonblocking logging and metrics |
| Start 300 pods with embedded 10 GB CSV | No 300x destructive import into a shared KV; resource budget bounded | Deterministic startup and import orchestration |
| Import contains duplicate keys or malformed rows | Candidate revision rejected or repaired under explicit policy | Previous serving revision stays active |
| One region receives dataset revision late | Visible lag and explicit freshness/error policy | Per-DC revision dashboard and rollback |
| Two vendors choose identical logical keys | Zero cross-tenant lookup/write | Server-side auth and isolation test |
| Hot tenant saturates shared KV or proxy | Other tenants retain agreed service level | Fairness, quotas and overload protection |
| Network policy blocks shared service | Typed unavailable error/blocked readiness, not ordinary `NotFound` | Controlled failure test |
| KV p99.9 spikes during normal and N-1 load | Deadline respected and business fallback is documented | Tail latency and lost-opportunity comparison |
| Vendor leaves PubMatic | Portable code, explicit export/deletion and no vendor data leakage | Offboarding runbook and artifacts |
| A coding agent creates equivalent template from scratch | Baseline includes correctness, hardening, host rules and support | Time/task comparison with curated kit |

**How to interpret the result:** A service can be commercially useful even if it is **not used by every vendor**. A strong outcome might be that the vast majority use portable snapshots while a subset of high-churn workloads opt into managed KV. That is not a product contradiction; it is a deliberate capability tier.

## 16. Suggested phased delivery with independent value

**Stage 0 — Establish baseline.** Inventory the existing container examples; make compatibility, protocol conformance, OCI behavior, offline data source, no-egress and health/readiness testable. Measure onboarding time. Produce one reference audience-provider code path without PubMatic imports in business logic.

**Stage 1 — Ship the portable developer experience.** Add repo scaffolding, local Docker Compose, sample requests, replay harness, standard OTel logs/metrics/traces, dataset fixtures, manifest checks, and capability-validation CLI. Clarify which facilities are platform-specific.

**Stage 2 — Provide platform-managed data assets.** Build the minimal secure file-ingest/validate/version/fan-out/mount pipeline and expose version/freshness to the workload. Test atomic refresh behavior and per-DC convergence; do not put this inside the auction request path.

**Stage 3 — Pilot one optional live managed capability.** Publish a versioned service contract and a Go adapter; include sandbox/local emulator, entitlement/resource provisioning, quota and operational dashboard. Compare host-neutral code, optional PubMatic acceleration, and alternate-host behavior.

**Stage 4 — Expand only with evidence.** Extend service interfaces or language SDKs if actual vendor adoption, measured economics, and operations justify them. Consider proposing a cross-industry binding vocabulary through IAB Tech Lab rather than making every SSP invent conflicting service names and semantics.

## 17. Open questions for the working group

1. Which *actual vendors and workloads* need mutable KV, as opposed to read-only snapshots or vendor-owned local memory?
2. Is developer stickiness an explicit commercial KPI, and how would it be measured without assuming exclusive deployment?
3. Must the same OCI image run across SSPs, or is common source plus host-specific packaging acceptable?
4. Which operations are legal per auction request, and what is the residual p99 budget after gRPC handling, business logic and response serialization?
5. Who is the authoritative source of audience data; what are allowed ages, refresh frequency and delete/erasure requirements?
6. What is the contract if a required dataset is missing, stale, corrupt, or only partially deployed across DCs?
7. Which region owns a KV namespace? Are writes synchronized, replicated, or intentionally host-local?
8. Who provisions and pays for data resources, and how are usage limits presented to vendors before deployment?
9. Does the runtime grant direct service access, a shared proxy, or a per-pod sidecar? What evidence decides the approach?
10. What is the vendor identity and authorization model across organizations, apps, environments, revisions and regions?
11. How are retries, circuit breakers and timeouts coordinated with auction deadlines?
12. What constitutes an optional service? How will admission, readiness, logs and SLA reporting expose degraded operation?
13. How closely must local simulation reproduce consistency, quota and fault behavior, and who maintains those contract tests?
14. Is a new `decisionfabric.Log` method valuable beyond structured stdout and OTel collection?
15. Could the most valuable differentiation instead be provisioning, secure artifact ingestion, self-service deployment, analytics and operational support?
16. Which capability semantics, if any, should be proposed as interoperable ARTF ecosystem conventions?
17. Do we mean one KV store per vendor company, application, dataset, environment, or region? Who sets the lifecycle and can multiple apps share it?
18. If the vendor's dataset is embedded inside the OCI image, who is authorized to extract it and how is its source verified?
19. Is bulk import a once-per-version platform Job or once-per-pod init task? How do retries prevent duplicate writes and partial publication?
20. Must the startup contract wait for full KV hydration, or could a previous known-good revision keep serving? Who approves data staleness?
21. For each region and DC, is a single shared KV sufficient under N-1, and what is the tolerated regional skew on refresh?
22. Is the proposed use case read-mostly, write-heavy, strongly consistent, eventually consistent, or ephemeral? What exact guarantees does `Get` promise?
23. What are worst-case cost and latency with thousands of vendors, distinct datasets, and sharply skewed QPS distributions?
24. Are shared-KV compute, storage, networking and multi-tenant support costs lower than a replicated in-process dataset at our actual footprint?
25. Could a vendor use a database inside its namespace under our security rules, and who owns upgrades, persistence, replication and CVEs?
26. Is the vendor's same-namespace datastore actually scheduled on the same node as its ARTF app? How much does locality explain observed p99?
27. Would a shared namespace/service gateway introduce risk of exposing internal PubMatic assets or of noisy-neighbor incidents?
28. If the platform provides snapshot mounts, can the vendor refresh data in an image-independent, multi-SSP-compatible way?
29. Which benefits are uniquely provided by the actual service, rather than achievable with AI-generated boilerplate and open-source dependencies?
30. Would vendors rather have a first-class compatibility/conformance CLI, richer local testing, or a managed KV binding? What do observed design partners say?
31. What are the contractual rules for downloading external datasets, object-store ingestion, and no-egress runtime execution on each target SSP?
32. Are application logs permitted to contain device identifiers or deal IDs, and how will service operators prevent unintended telemetry exposure?
33. Should PubMatic propose a cross-host capability vocabulary to IAB Tech Lab before creating incompatible provider-specific binding names?
34. What does the platform do if an entitlement is revoked while a pod is already running, and how quickly does access stop?
35. Where is source-of-truth state located when the same vendor workload executes in multiple SSP infrastructures?

## 18. A leadership-level framing for the decision

**Good aspiration:** “Developers should get production-grade, low-latency infrastructure and useful observability on Decision Fabric with minimal setup.”

**Avoid as primary justification:** “Developers will become locked in when they call our API.” Multi-SSP vendors are not looking for a new dependency they cannot use elsewhere. A lock-in attempt may paradoxically reduce first adoption.

**Proposed position:** “Make a portable vendor workload easier to develop everywhere, and demonstrably easier, faster and cheaper to operate on PubMatic.”

**Recommended immediate decision:** Fund the portable DX baseline and a versioned-data-delivery pilot. Authorize architecture exploration for an optional managed-services binding, but condition build-out of production KV on documented vendor demand, strict capability contracts, security ownership, a realistic local-dev story and measured tail-latency economics. This is a **selective yes** to the platform strategy, not an unconditional yes to a branded SDK.

## 19. Implementation blueprint if Decision Fabric commits to an SDK and managed bindings

**Purpose of this section:** Sections 1–18 evaluate *whether* to build. This section assumes leadership has decided **to build an SDK/library and at least one real managed service**, even if the commercial case is still being validated. It answers *how it should be built*, what must exist beyond the Go package, what can be delivered incrementally, and which design choices are expensive to reverse. The examples below are **proposed contracts and architecture**, not assertions about deployed PubMatic systems or fields in the IAB ARTF standard.

### 19.1 Non-negotiable implementation principles

1. **A real capability, not merely a Go convenience wrapper.** `df.KV("audience").Get(...)` must resolve to an authorized, provisioned, metered service with explicit behavior; the SDK cannot manufacture that service.
2. **Host independence of business logic.** The developer's audience-matching function should depend on a narrow business interface. The PubMatic implementation is an adapter. A workload may use host-specific deployment configuration while preserving a common OCI image.
3. **Explicit dependency semantics.** Required binding absent → deployment/admission or startup error. Optional binding absent → discoverable absence and a documented behavior. `NotFound` must never mean connectivity or authorization failed.
4. **Control plane off the hot path.** Provisioning, tenant lookup, entitlements, CSV ingestion, metering aggregation and service discovery updates are asynchronous. The per-auction path never invokes a Kubernetes API or control-plane database.
5. **Regional, deliberately constrained data plane.** Vendor-serving RPCs terminate on a dedicated vendor-facing gateway/service boundary; neither Kubernetes namespace membership nor a vendor-supplied tenant ID is sufficient authorization.
6. **No hidden synchronous networking.** Document whether each SDK call is in-process, localhost/sidecar, same-node, or another service. The developer must know the timing and failure implications.
7. **Local development is a first-class execution mode.** The same SDK method signatures, request/response contracts and error categories work in Docker on a laptop without live PubMatic credentials.
8. **One source of truth for capability policy.** A typed, versioned binding specification drives provisioning, admission, runtime configuration, local simulation, docs and contract tests. Do not implement six independently interpreted YAML/env conventions.
9. **Backward compatibility and migration exist on day one.** SDK versions, protocol versions, service semantics, binding renames, tenant migrations and data exports need published support and deprecation rules.
10. **The host—not untrusted container code—owns shared infrastructure.** Vendor images are inputs to a controlled deployment; they should not obtain cluster-admin access, direct credentials to shared backing databases, or unrestricted access to PubMatic internal services.

### 19.2 What developers see versus what PubMatic must operate

| What the vendor experiences | Production implementation that must exist | Team/owner to identify |
|---|---|---|
| `df.KV("audience").Get(ctx, key)` | Versioned Go package, binding parser, pooled RPC client, stable gateway API, regional KV backend, admission policy, SLO | SDK + runtime + data platform |
| `df.KV("state").Put(ctx, key, value)` | All the above **plus** write permissions, durability/TTL/consistency rules, quotas, retries/idempotency and billing | Data platform + SRE |
| `df.Log.Info(...)` | Safe structured logging, stdout/OTel collection, filtering/redaction, cardinality limits, retention and vendor-access controls | Observability + privacy |
| `df.Config.Get(...)` | Deployment config versioning, schema validation, safe updates and binding injection | Control plane |
| `df.Secrets.Get(...)` | Secret source, entitlement, short-lived identity or file mount, rotation, redaction and revoke semantics | Security + control plane |
| `df.Assets.Open("model")` | Artifact intake, provenance, validation, versioning, DC replication, immutable mount contract, freshness metadata | Data delivery + runtime |
| `df.Events.Publish(...)` | Bounded non-blocking publisher, queue/broker, backpressure, retention, delivery and billing contract | Eventing + SRE |
| `df dev up` | CLI, Docker/local gateway and datastore, fixtures, protocol simulator, error injection, compatibility tests | Developer experience |
| “Runs in another SSP” | Optional adapter composition, feature discovery, tested fallbacks, alternate-host config/manifest and portability conformance | SDK + partner engineering |

**Design insight:** the managed service, not the SDK method name, sets the implementation difficulty. Start with one capability that can be operated to a published service contract; do not commit to the entire table as v1 scope.

## 20. Concrete reference architecture and service boundaries

### 20.1 Recommended logical topology (initial production direction)

```text
                       VENDOR-SIDE DEVELOPMENT
  Go ARTF project --> vendor domain interface --> DF Go adapter (optional)
       |                      |                       |
       +------ df CLI + local simulator / Compose ---+
                              |
                     image + manifest + data
                              v
  +---------------- DECISION FABRIC CONTROL PLANE -----------------+
  | Vendor portal / API                                          |
  |   -> org/app/environment registry + entitlement decisions     |
  |   -> binding/catalog API + quota/plan + validation            |
  |   -> desired-state records (resource/binding/deployment)      |
  |   -> reconciliation/provisioner + dataset-import controller   |
  |   -> deployment admission + readiness / version promotion     |
  |   -> metering/billing summaries, access revocation, audit     |
  +---------------------------+----------------------------------+
                              | resolved, least-privilege binding
                              v
  +---------- KUBERNETES: VENDOR-A WORKLOAD NAMESPACE ------------+
  | Vendor ARTF pod(s)                                           |
  |   gRPC ARTF endpoint --> business logic --> SDK client        |
  |                               |             |                 |
  |                  local snapshot provider   |  (optional)     |
  |                  or local cache             |                 |
  |                      binding descriptor / workload identity  |
  +--------------------------------------------+-----------------+
                                               | gRPC/TLS
                                NetworkPolicy + authenticated RPC
                                               v
  +---------- PLATFORM-EXPOSED SERVICES (SEPARATE CONTRACT) ------+
  | DF Binding Gateway (HA, region/DC local)                     |
  |    authenticate -> authorize logical resource -> enforce      |
  |    quota/deadline -> resolve physical route -> meter          |
  +-------------------------+------------------------------------+
                            |
              +-------------+----------------+
              v                              v
       tenant-partitioned KV         versioned dataset/artifact
       (Redis/Aerospike/etc.)         store + distribution
              |
              +--- SRE dashboards / audits / per-tenant billing

  Internal SSP bidder/data stores remain behind separate internal
  trust and service boundaries; vendor gateway does not expose them.
```

**Deployment choices that must remain open until measurement:** direct SDK → gateway (recommended reference), SDK → optional localhost sidecar → gateway (if identity/caching justify it), or direct managed backend (only for exceptionally well-isolated uses; not the default). A localhost sidecar avoids reimplementing some plumbing in language SDKs but adds per-pod memory, CPU, lifecycle coupling and operational complexity. It does not eliminate downstream RPCs on misses.

### 20.2 Enumerate the buildable components (minimum product bill of materials)

| Component | Responsibilities | Durable state / dependency | Can leverage existing platform? |
|---|---|---|---|
| **1. Go SDK module** | Typed APIs, config/binding loading, deadline propagation, pooling, error mapping, capability discovery | None; vendored into Go binary | Mostly new thin code |
| **2. Binding/service contract** | Protobuf gRPC API and capability schema, semantic-version rules, generated client | Versioned spec repo | New public interface |
| **3. Capability catalog & entitlement service** | Available plans, rights, quotas, region support and compatibility checks | Control-plane DB | Extend platform control plane |
| **4. Binding registry** | Logical name → immutable resource identity, scope, desired/observed state and backend mapping | Durable DB with audit | Extend registry/control plane |
| **5. Provisioning reconciler** | Idempotent create/update/suspend/delete; repair drift; resource allocation | Registry + infrastructure API | New controller/service logic |
| **6. Admission/binding injector** | Verify required dependencies, authorize image+app, project descriptor and identity, block unsupported deployment | K8s workload API / deployment orchestrator | Extend platform deployer; webhook/CRD **optional** |
| **7. Vendor-facing binding gateway** | Workload authentication, authorization, service API, quotas, timeouts, routing, usage events | Local auth policy cache + backend route cache | New stable boundary |
| **8. KV backend and operations** | Data storage, replication, sharding, persistence, upgrades, failover, backup, N-1 | Managed Redis/Aerospike/other | Reuse if isolation and license allow |
| **9. Data-ingest/version controller** | Artifact acceptance, CSV validation, staged bulk load, active-version switch and cleanup | Durable job/version records + artifact store | Extend data-delivery capability |
| **10. Local CLI / simulator** | Run SDK calls locally with fixtures and fault injection; ARTF caller and conformance | Local Docker volumes/state | New developer tooling |
| **11. SRE, security, finance integrations** | SLIs, alerts, privacy audit, per-binding meter, quotas, cost attribution and incident workflow | Telemetry/billing pipelines | Extend existing platform systems |
| **12. Documentation and contract test kit** | API reference, no-egress policy, examples, versioned compatibility and negative tests | CI and documentation | Extend container-example repository |

**Boundary rule:** components 3–9 are **not** imported by the vendor image. The vendor image sees a declared logical resource, a small descriptor, an authenticated protocol and the SDK/client. Internal tables, Redis/Aerospike credentials, control-plane endpoints and resource names remain private.

### 20.3 Minimal internal data model and lifecycle

Proposed relational/control-plane records (not an approved schema):

```text
VendorOrg(org_id, status, billing_account)
Application(app_id, org_id, owner, artifact_digest, status)
Deployment(deployment_id, app_id, environment, region, revision, state)
CapabilityDefinition(type, contract_version, allowed_operations,
                     regions, default_limits, runtime_requirements)
CapabilityBinding(binding_id, app_id, environment, logical_name,
                  capability_type, contract_version, required,
                  backing_resource_id, lifecycle_state, generation)
BackingResource(resource_id, tenant_partition_id, physical_backend_class,
                locality, capacity_class, observed_state, revision)
Entitlement(org_id, app_id, capability_type, operations, region, limits)
DatasetVersion(dataset_id, version_id, digest, schema_id, validation_state,
               import_state, region, activated_at, expiry_policy)
UsageRecord(binding_id, region, window, reads, writes, bytes_in,
            bytes_out, throttled, errors, meter_version)
AuditEvent(actor, action, target_id, outcome, timestamp, correlation_id)
```

Recommend uniqueness of `(app_id, environment, logical_name)` for a binding *within its declared resource scope*; map that logical name to different region-local physical resources under one stable logical binding ID. Decide explicitly if bindings are scoped per org, application, deployment, or shared vendor-owned dataset. Separate deletion of a Deployment from deletion of durable KV contents. Durable storage must not be unintentionally destroyed during ordinary rolling releases.

**Reconciliation state machine:** `Requested -> Validating -> Provisioning -> Ready -> Bound -> Suspended -> Deleting -> Deleted`, with `Degraded`/`Error` observable states. Store desired and observed generations; retries must be idempotent and compensating cleanup required. If provisioning partly succeeds in DC A and fails in DC B, show `PartiallyReady`, block/limit the affected rollout, and preserve the prior serving revision according to policy.

## 21. Specify the external SDK and wire contract before the backend

### 21.1 Proposed Go developer API (illustrative; API names subject to review)

```go
// In a vendor-owned package, no PubMatic types leak into business logic.
type AudienceStore interface {
    Lookup(ctx context.Context, deviceID, contentID string) ([]string, error)
}

// In the optional Decision Fabric adapter package:
// client, err := df.NewFromEnvironment(df.Options{ /* no raw DB keys */ })
// audienceKV, err := client.KV("audience_data") // explicit capability resolution
// value, found, err := audienceKV.Get(ctx, []byte(key))
// err = audienceKV.Put(ctx, []byte(key), encoded, df.WithTTL(time.Hour))
//
// The same vendor business interface may instead receive a snapshot adapter
// in local/offline/other-SSP execution; selection occurs during startup.
```

**Required API properties:** `context.Context` on every potentially blocking operation; typed permission/unavailable/timeout/quota errors; `found` separate from `err`; no hidden global singleton; efficient concurrent use by multiple gRPC request goroutines; pooled connections; defined limits on allocation/value size; bounded internal queues; explicit `Close`; and instrumentation that avoids logging raw audience IDs. Use `Put` only when binding contract authorizes it. Do not infer that read-only snapshot and shared mutable KV have identical semantics.

### 21.2 Service interface: speak stable gRPC, not a proprietary backing-store protocol

```protobuf
// PROPOSED Decision Fabric API, not part of the IAB ARTF protobuf.
service KVService {
  rpc Get(GetRequest) returns (GetResponse);
  rpc Put(PutRequest) returns (PutResponse);
  rpc Delete(DeleteRequest) returns (DeleteResponse);
}
message GetRequest { string binding = 1; bytes key = 2; }
message GetResponse { bool found = 1; bytes value = 2; string data_version = 3; }
message PutRequest {
  string binding = 1;
  bytes key = 2;
  bytes value = 3;
  int64 ttl_seconds = 4;
  string idempotency_key = 5; // when applicable to the chosen write semantics
}
message PutResponse { string version = 1; }
message DeleteRequest { string binding = 1; bytes key = 2; }
message DeleteResponse { bool existed = 1; }
```

This is a **shape illustration only**, not a complete .proto: define `syntax`, package/version, operation semantics, limits, errors and transport security in the actual specification. In particular, an idempotency token is useful only if the server **implements** durable request deduplication with a defined retention window; otherwise omit it rather than suggest exactly-once delivery. If writes are outside the initial scope, expose only `Get` in v1 and document that `Put` is future work.

**Error contract:** `found=false, err=nil` means legitimate missing key; `Unavailable` means a service/backend problem; `PermissionDenied` means unauthorized scope; `ResourceExhausted` means quota; `DeadlineExceeded` means budget expired; `InvalidArgument` means invalid key/size/TTL; `FailedPrecondition` means unready dataset or stale incompatible revision. The Go adapter maps gRPC status to stable typed errors while retaining machine-readable cause and retry guidance. Define whether partial `BatchGet` can succeed before adding it.

### 21.3 Exactly what a binding contains

```yaml
# df-bindings.yaml: proposed DEPLOYMENT-SPECIFIC configuration, NOT IAB schema
version: decisionfabric.dev/v1alpha1
application: audience-provider
bindings:
  - name: audience_data            # vendor-facing logical name
    capability: kv.read.v1
    required: true                  # on this PubMatic deployment
    scope: application
    region_policy: local-only
    max_key_bytes: 256              # illustrative; real limits TBD
    max_value_bytes: 65536          # illustrative; real limits TBD
    consistency: contract-defined  # select actual supported profile
    initialization:
      mode: dataset-import
      artifact: audience-v2026-10-09
      format: csv
      freshness_max: 24h
  - name: audit_events
    capability: events.publish.v1
    required: false
```

**Critical separation:** this desired-state/deployment declaration is **not** the descriptor the runtime consumes. The control plane resolves it into a **read-only generated descriptor** (binding name, effective protocol version, approved endpoint, resource capability, limits, dataset revision). The runtime descriptor does **not** carry DB root passwords, arbitrary tenant IDs or raw physical keys. Sensitive workload identity is injected through a distinct rotating mechanism. Alternative SSP deployments may supply a different host-specific declaration while shipping the same application image.

**Version strategy:** use semantic major protocol versions and feature negotiation; require both supported host capabilities and the application's declared minimum versions. Refuse unsupported required features at deployment and on startup. Publish an SDK-to-gateway compatibility matrix and support overlapping major versions long enough for vendor release cycles. Do not break old vendor containers when the backing store changes.

## 22. How `KV.Get` and `KV.Put` execute in production

### 22.1 End-to-end data-plane request walkthrough

1. **Build:** vendor compiles Go binary with SDK/adapter, packages one OCI image. No PubMatic prod credentials or internal DNS addresses are embedded.
2. **Register/deploy:** vendor declares `audience_data` and requested rights; the control plane performs entitlement, quota, version and region checks.
3. **Provision:** reconciler creates or assigns a region-local tenant partition/resource, stores binding-to-resource mapping, and waits for backend readiness. Admission does **not** rely on an arbitrary app-controlled tenant ID.
4. **Inject:** workload receives a read-only descriptor and short-lived bound identity/credentials; network policy permits only the authorized gateway and permitted host services.
5. **Initialize:** SDK parses descriptor, validates required bindings/contracts, acquires a pooled client, and ensures a usable local identity before readiness. No per-auction control-plane lookup.
6. **Request:** ARTF request reaches vendor gRPC handler; business logic asks adapter for `Get(ctx, key)`. SDK inherits remaining auction deadline, checks size, and sends an RPC to the region-local gateway.
7. **Verify:** gateway authenticates workload cryptographically, authorizes its binding/operation and regional scope, applies quotas, resolves a **cached** backend mapping and calls appropriate storage partition.
8. **Respond:** backend returns value/missing/failure. Gateway maps the status without disguising errors and emits bounded per-binding metrics. SDK delivers value or typed error to the vendor's explicit business policy.
9. **Observe:** per-app, per-binding and per-region SLIs and usage are asynchronously exported. No request-time database or Kubernetes control-plane lookup for usage accounting.
10. **Revoke/change:** control plane updates access decisions; gateway detects policy revision, invalidates cached authorization within defined revocation time, and blocks subsequent calls. In-flight-call behavior must be specified.

**A `Set/Put` has additional work**: enforce write scope, storage cap, TTL limits, durability/visibility policy, conflicting writers and idempotency semantics. If the app issues writes on every auction, throughput/cost and correctness testing must model that explicitly. `Put` is not an innocuous syntactic extension of `Get`.

### 22.2 Authentication design and threat boundaries

**Baseline choice:** short-lived, audience-scoped workload tokens injected into pods via a supported projected-volume mechanism, presented to a TLS-protected gateway, verified and associated with a binding grant. Kubernetes projected service-account tokens rotate and can be audience-scoped; they establish a Kubernetes workload identity foundation, **not** the complete vendor entitlement model. The verifier must validate token issuer, audience, expiry and workload identity, and map workload → app/deployment → specific entitled binding **on the server**. [S12][S19]

**More mature choice:** SPIFFE/SPIRE-style workload identities with mTLS and workload attestation, if the colocation platform already supports or is willing to operate its issuance/rotation stack. SPIFFE is a standards family, not a prerequisite for v1, and adding it creates its own control plane. [S20]

| Threat / mistaken shortcut | Required countermeasure |
|---|---|
| Client sends `tenant_id=other_vendor` | Gateway derives tenant from verified identity and server-side binding mapping; ignores client tenant authority |
| Any pod in `vendor-a` namespace gets full access | Distinct workload identity per app/deployment, per-binding grants, admission restrictions, NetworkPolicy defense in depth |
| Token stolen from logs/image | Never bake token in image; redaction, short lifetime, rotation, audience-scoped token, minimal filesystem access |
| Data backend exposed via service discovery | Only gateway allowed by network policy; no raw Redis/Aerospike addresses/passwords in descriptor |
| Unauthorized external network calls | Explicit egress allowlist, DNS and network restrictions; controlled ingestion outside auction workloads |
| Ingestion runs arbitrary vendor image | Do not execute untrusted image code with platform-wide credentials; isolate extraction, scan sources, use separate importer identity |
| A vendor exhausts shared service | Per-binding QPS/concurrency/storage limits, fairness, backpressure, hot-key controls and isolation tiers |
| Offboarding leaves keys behind | Revocation, data retention/deletion workflow, backups/replicas lifecycle and audit evidence |

**Practical issue:** performing an API-server TokenReview or control-plane DB query for each `Get` would itself introduce a high-QPS dependency. Verify signatures/policy using an appropriate cached or locally-verifiable design with bounded cache freshness and a measured revocation SLA; quantify token/key rotation effects under load.

### 22.3 Placement, load balancing and locality

**Do not compare architectures by namespace alone.** For example, `vendor-a` pod → `df-kv-gateway` Service in `df-services` namespace is not intrinsically slower than `vendor-a` pod → `vendor-a-aerospike` Service. Both can span nodes; both involve pod-to-pod traffic. Co-locate by placement when beneficial, use regional service discovery, test topology hints and avoid accidental cross-DC path selection. Kubernetes DNS/service discovery and topology controls support such patterns, but locality needs deliberate configuration and measurement. [S13][S16][S17]

| Topology | Calls per auction | Locality assumption | Major engineering concern |
|---|---:|---|---|
| In-process map / memory-mapped data | 0 RPC | Inside vendor process | Replicated memory and refresh correctness |
| Same-pod sidecar cache | Local RPC on hit; remote on miss | Same network namespace | Cache consistency, extra pod resources |
| Vendor-owned Aerospike StatefulSet | ≥1 network RPC | Depends on pod placement | Vendor/service operations and persistence |
| Shared DF gateway → tenant-partitioned KV | ≥2 logical service hops (client→gateway→store) | Region- or node-local if engineered | Hop overhead, shared blast radius and quotas |
| SDK directly to DF-owned store | Usually 1 RPC | Depends on placement | Backend coupling, credentials, migration and isolation |

A logical "hop" need not correspond to a physical switch hop; the gateway may be co-resident with a store, and proxy/sidecar implementation details matter. Obtain baseline numbers with and without TLS, proxy, serialization, service routing, mesh and pod churn before drawing conclusions. **A direct call can be faster, but the gateway buys backend independence and enforcement**—make the tradeoff visible.

### 22.4 Kubernetes deployment wiring: what the platform injects

The platform controller should generate the concrete Kubernetes resources from the validated binding record. **Do not ask each vendor to create or edit PubMatic ServiceAccounts, ConfigMaps, NetworkPolicies or internal Services.** These are examples of **controller output**, not instructions to a vendor or a complete installable manifest. Actual namespaces, labels and policy engines depend on the Decision Fabric runtime.

```yaml
# ILLUSTRATIVE fragments of a generated vendor Deployment (not complete).
spec:
  template:
    spec:
      serviceAccountName: df-vendor-audience
      automountServiceAccountToken: false
      containers:
        - name: artf-app
          image: registry.example/vendor/audience@sha256:REPLACE_WITH_DIGEST
          env:
            - name: DF_BINDINGS_FILE
              value: /var/run/df/bindings/descriptor.yaml
            - name: DF_IDENTITY_FILE
              value: /var/run/df/identity/token
          volumeMounts:
            - {name: binding-descriptor, mountPath: /var/run/df/bindings, readOnly: true}
            - {name: workload-identity, mountPath: /var/run/df/identity, readOnly: true}
      volumes:
        - name: binding-descriptor
          configMap:
            name: df-audience-binding-descriptor  # created IN the vendor namespace
        - name: workload-identity
          projected:
            sources:
              - serviceAccountToken:
                  path: token
                  audience: df-bindings-gateway
                  expirationSeconds: 3600
```

Kubernetes ConfigMap/Secret/projected-volume inputs are namespace-scoped to the pod; therefore the platform must create the generated descriptor **in the vendor's namespace**, even though the KV gateway/backing store is in a different platform namespace. The workload does not need access to the Kubernetes API to discover services or read the projected files. The kubelet rotates a projected bound service-account token; SDK/gateway token refresh must not assume its contents are fixed for the pod's lifetime. [S19]

```yaml
# ILLUSTRATIVE vendor-namespace egress policy fragment.
# Complete production policy must also account for explicitly allowed DNS
# and any other approved endpoints; requires a CNI enforcing NetworkPolicy.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-df-bindings-egress
  namespace: vendor-a
spec:
  podSelector:
    matchLabels: {app: audience-agent}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels: {decision-fabric-role: exposed-services}
          podSelector:
            matchLabels: {app: df-binding-gateway}
      ports:
        - {protocol: TCP, port: 7443}
```

**Security qualifications:** This example does not itself establish default-deny across every pod or allow DNS; the platform must supply its baseline policies, including matching gateway-side ingress policy. Operators must control who can set protected namespace/pod labels, since policies use selectors. A NetworkPolicy controls permitted traffic and does not authorize a vendor to read a particular KV resource. Use server-side entitlement/identity checks regardless. [S15]

### 22.5 Lifecycle subtleties of projected bindings

- **Rotation:** a generated binding descriptor can be versioned per Deployment and immutable for that revision; on changes, a controlled rollout is often simpler than expecting all SDKs to hot-reload security-sensitive endpoint and capability changes. Identity tokens, however, must refresh without redeployment.
- **Secrets:** a projected workload token authenticates to the *gateway*, rather than carrying a Redis/Aerospike master key. Restrict the gateway's own storage credentials and rotate them independently.
- **Replica identity:** all replica pods may share an application-level entitlement but should remain individually attributable for debugging/revocation. Backend data scope is bound to the logical binding, not a transient pod IP.
- **Failure to inject:** if descriptor or identity is missing, required SDK initialization fails and readiness stays false; do not fall back to an unrestricted global KV endpoint.
- **Cross-namespace access:** vendor pods make an approved service-network request to a gateway in a platform namespace. Kubernetes does not require copying the KV Service into each namespace; DNS and network policies provide connectivity while identity/authorization provide trust separation.

## 23. Implementing KV tenancy, storage guarantees and capacity

### 23.1 Choose a tenant-isolation tier per workload, not one for everybody

| Mode | Allocation | Appropriate when | Key limits and tradeoffs |
|---|---|---|---|
| **Shared physical backend, logical partition per binding** | Resource ID maps to tenant-scoped keyspace/record set | Many low-QPS/small-data vendors | Strong gateway auth; avoid uncontrolled shared scans, cardinality and hot keys |
| **Shared backend, dedicated shard/namespace per large vendor** | Placement and budgets isolated by tenant | High-QPS or noisy/large vendors | Rebalancing and shard allocation complexity |
| **Dedicated backend cluster per vendor** | App/tenant gets isolated managed service | Custom SLO, regulated dataset or very large footprint | Expensive capacity floor, operator/SRE overhead |
| **Vendor-operated database workload** | Vendor owns DB containers/persistence where allowed | Vendor demands backend control and host permits it | Host must define security, allowed deployment types, metering and operational ownership |
| **Local read-only snapshot** | Per-pod local file/process state, platform may distribute | Read-mostly or daily refresh auction workloads | Memory multiplication, slower refresh but no hot-path service calls |

Backend choice (Aerospike/Redis/etc.) must be made against *measured workload operations* and deployment constraints. Expose product guarantees at the gateway, not implementation names. In particular, a Redis-compatible interface should not casually promise Aerospike consistency or vice versa. Inventory exact current licensing/edition entitlements before embedding a vendor-accessible service into enterprise software; existing internal use rights do not automatically imply a new service offering.

### 23.2 Publish two clearly different capability profiles if needed

- **`kv.read.snapshot.v1`** — immutable or versioned, preloaded reads; no synchronous vendor writes, active dataset revision exposed, freshness contract. Candidate for audience eligibility.
- **`kv.mutable.v1`** — defined `Get`/`Put`/`Delete`/TTL limits; specify durability, eviction, consistency, atomic mutation availability and replication scope. Candidate for changing vendor state.

For each profile specify: max keys/bytes; key encoding; missing-key semantics; allowed TTL units and expiry clocks; eviction/recovery; behavior under failover; read consistency and write visibility; intra-/inter-DC replication; cost per request/GB; billing on failed calls; batch/cursor support or explicit non-support; and privilege separation between import jobs and serving pods. Never use a “generic KV” description in place of a testable contract.

### 23.3 Cost model, not merely cluster count

Estimate per vendor and per DC: `R = auction_invocations/s * KV_operations/invocation`; request and response bytes; gateway CPU/RPC; datastore read/write CPU; memory/SSD and replication; memory duplicated across in-process readers; failure-spare/N-1 capacity; dataset import bandwidth; idle resource floor; logging/telemetry; storage recovery; operational/SRE effort; and vendor onboarding/support. For write-heavy stores, include storage amplification and durability acknowledgments. For snapshot mode, include per-pod RAM, startup distribution and full refresh traffic.

**Do not assume shared KV saves money**: pooling can reduce idle database overhead, but it creates recurring request costs and a shared availability domain. Conversely, do not assume per-vendor databases are cheaper because they share a namespace. Create a break-even spreadsheet with actual QPS, cardinality, replica count, data size and 99th-percentile targets. Section 7.4 provides one illustrative scale point.

## 24. Vendor CSV ingestion and rollout: the specific example, end to end

### 24.1 Decide what the source artifact actually is

A CSV *inside the submitted OCI image* is different from an independently supplied dataset. The platform needs a well-defined discovery mechanism: vendor-declared `image:/app/data/audience.csv` path, separately submitted object, or a controlled vendor-defined export action. The platform **cannot** assume a CSV embedded in any arbitrary image is safe or trivial to extract. Never execute untrusted image startup code just to obtain data from its filesystem. Prefer independent data artifacts for repeated refreshes.

**Proposed first contract:** vendor uploads a versioned CSV/Parquet/etc. alongside an OCI image (separate digest), with declared schema, region, ownership, size, update cadence and deletion policy. Support embedded-image extraction later through an isolated read-only OCI filesystem extraction utility, if necessary to preserve the onboarding story. Use immutable digests, not mutable `latest` tags.

### 24.2 Once per dataset revision—not once per vendor pod

```text
[1] Vendor registers CSV version + digest
                  |
                  v
[2] Intake / policy / schema / data-validation service
                  |
                  v
[3] Durable dataset revision = IMPORTING
                  |
                  v
[4] Per-DC import worker/Job -> new tenant-scoped inactive namespace
                  |
                  v
[5] Check counts + digest + sample query + quorum/replica readiness
                  |
                  v
[6] Dataset version = VERIFIED; stage activation policy
                  |
                  v
[7] Atomic active-version pointer switch (if backend supports it)
                  |
                  v
[8] Reconcile serving app readiness / warm caches / regional rollout
                  |
                  v
[9] Retain last known good version; later GC under retention policy
```

Kubernetes init containers run per pod, while Jobs have explicit execution/retry/cleanup lifecycle. Thus creating an importer in every scaled serving pod is an architectural mistake for shared state; reconcile an import once per unique `(org, dataset, version, DC)` using a durable desired-state record and idempotent workers. Even a Kubernetes Job is **not** an exactly-once transaction guarantee. [S18][S21]

**Import correctness rules:** deterministic normalization for `(device_id, content_id)` composite keys; escaping and canonical encodings; duplicate-key conflict rule; empty or malformed rows; compressed/uncompressed size limits; max row count; schema evolution; sample and full checksums; version-specific key prefixes/partitions; rate-limited bulk import; import cancellation; atomic activation or documented staged cutover; rollback to previously complete snapshot; background garbage collection and deletion/erasure verification. Avoid mutating the live namespace during a multi-hour CSV load.

**If the file changes while container stays up:** specify who detects updates, whether refresh is a timed ingestion outside the pod, whether the same binding transparently starts serving version N+1, whether an app-local cache has to invalidate, and whether regional skew is permissible. Expose `data_version` and `loaded_at` for each binding so an operator can prove what produced a bid decision.

### 24.3 Startup, refresh, scale-out and failure scenarios

| Scenario | Correct design response | Acceptance proof |
|---|---|---|
| First deployment; no dataset yet | App remains unready or rollout blocked until required dataset is verified | No false audience miss while importing |
| 200 pods are scaled up in seconds | One import per revision; pods only establish binding/read state | Count of import Jobs is independent of replica count |
| CSV has invalid row 5 million | Reject candidate or documented partial-row policy; active dataset unchanged | Prior version continues serving |
| Two imports for same revision retry | Idempotent allocation and import; no double-activation or overwrite | Forced worker/job restart test |
| Revision N+1 reaches 3 of 4 DCs | Per-DC visibility and policy-driven activation gate | DC skew dashboard and rollback |
| Gateway/storage unavailable during startup | Backoff *outside auction traffic* up to startup deadline; reported dependency error | Explicit readiness failure, no crash loop storm |
| Gateway down after readiness | Bounded call failure, optional degraded policy/circuit breaker; recovery checks | Business correctness and SLI under chaos test |
| Vendor replaces image but not dataset | Retain same logical dataset binding if compatible | App rolling release does not delete state |
| Vendor deletes a dataset or is offboarded | Revoke access, retention/deletion workflow for versions and backups | Audit trail, no residual unauthorized reads |
| Another SSP runs same image | Explicit alternative snapshot/provider, or clear required-capability refusal | No fake empty-targeting output |

## 25. Local development and cross-SSP portability: executable contract

### 25.1 Proposed `df` CLI behavior (new tooling, not currently available)

```sh
df init --template audience-provider
# Adds an example, contract declaration, fixture schema and Makefile.

df dev up --profile local
# Starts the vendor app + ARTF caller + binding gateway emulator +
# local KV implementation with project-scoped persistent dev state.

df dev seed --binding audience_data --file ./fixtures/audience.csv
df dev inspect audience_data --key 'idfa|content'
df test artf                   # golden requests / responses / tmax

df test capabilities           # denied, absent, stale, timed out, throttled

df test portability --profile no-egress
# Same business logic with offline/snapshot provider and no DF endpoint.

df deploy plan --env sandbox    # entitlements, expected resources and cost
```

`df dev up` can internally generate Docker Compose; it should not require developers to install Kubernetes. Local state is intentionally isolated from production, and a command is available to reset it. Test reproducibility is more important than making the emulator simulate the entire production datastore cluster. Cloudflare's documented default local binding simulation and explicit remote-binding option are a useful design precedent. [S2][S3][S22]

**Implementation approaches:** use (a) an in-process fake for unit tests, (b) a lightweight local gRPC binding emulator with SQLite or a matching open-source KV engine for contract tests, and (c) a sandbox test against the actual production gateway/backend for integration and performance. Do not claim the SQLite emulator has Aerospike/Redis failure, concurrency or consistency semantics. Maintain a *parity matrix*: operations, limits, not-found, TTL, concurrency, writes, error codes and protocol versions must match; failover, multi-DC and p99 require sandbox testing.

### 25.2 Portability has three distinct levels

1. **Portable source:** business logic remains host-neutral; local and SSP-specific adapters are injected at composition/startup.
2. **Portable image:** one Go binary includes snapshot provider and optional DF adapter; runtime selection reads validated configuration; lack of DF binding does not automatically imply a no-op.
3. **Portable state/semantics:** dataset revisions and state are explicitly exportable and can be hydrated on another host. Mutable records do **not** automatically replicate to other SSPs, and some hosts prohibit request-time network calls entirely.

For a non-PubMatic deployment, provide a tested **offline snapshot provider** when business logic allows it; or require a host-provided equivalent adapter with a documented semantic translation. If neither can satisfy required audience functionality, declare the deployment unsupported rather than silently produce empty results. No-op implementations are permissible only for genuinely non-critical optional capabilities such as best-effort debugging counters, and any dropped telemetry must be visible in status.

## 26. Resilience, deadlines, logging and other overlooked runtime hazards

### 26.1 One auction budget, one explicit retry owner

For request-time `Get`, propagate the incoming gRPC/ARTF deadline through the SDK into gateway and store calls. Set a call-specific upper bound *strictly less* than remaining auction budget. Avoid eager retries, per-layer exponential retries, hedging, and automatic cross-region failover for the hot path by default. Determine which layer owns any allowed retry and enforce the total attempt budget. gRPC may perform **transparent retries even without a configured retry policy**; if strict no-RPC-retry behavior is desired, disable retries in client channel options and verify actual behavior. Transport-level TCP retransmissions remain a distinct mechanism. [S23]

For `Put`, a retry after a response is lost can duplicate non-idempotent effects; plain replacement `Put` may be safe under defined last-writer-wins semantics, whereas increments/CAS/event publication demand stronger controls. Do not advertise at-most-once, exactly-once or transaction guarantees without an implementation and test contract. Include queue saturation, connection storms, DNS failures, token refresh and load-shedding tests.

### 26.2 Probes must express application correctness

- **Startup probe**: permits initial dataset opening / binding initialization without premature restart.
- **Readiness probe**: only returns ready when all *required* binding/data freshness contracts are met; conditions change on sustained runtime unavailability if serving cannot proceed correctly.
- **Liveness probe**: checks whether process is stuck, **not** whether a remote KV is healthy; otherwise a datastore incident can restart every vendor pod and amplify the outage.

Kubernetes distinguishes startup, readiness and liveness probes and cautions against probe choices that create cascading failures. [S24]

### 26.3 Logging, metrics and billing are not the same thing

Use `slog`/structured stdout and OpenTelemetry where possible. If `df.Log` is offered, make it a thin non-blocking convenience wrapper. Enforce rate/cardinality limits, redaction of device IDs and keys, and output budgets. **Do not issue a synchronous RPC to a central logger on each auction request.** Aggregate metering from gateway counters/events into a durable billing pipeline with reconciliation; telemetry loss must not silently change billed usage. OTel includes guidance on attribute and metric cardinality limits, but platform-level privacy and cost policies must be defined separately. [S9][S10][S25]

### 26.4 Incident and lifecycle runbooks required before production

Document gateway outage, partial region outage, backend failover, failed bulk import, expired identity, entitlement revocation, quota exhaustion, vendor code bug causing hot key, rogue write/delete, data restoration, disaster recovery, SDK vulnerability and protocol version retirement. For each: symptom, blast radius, alert threshold, safe runtime behavior, rollback, tenant communication, estimated recovery objective (to be negotiated), and audit requirements. A vendor-facing service is not production-ready until these operations are rehearsed.

## 27. Engineering delivery plan: dependencies, milestones and acceptance criteria

**This is an order of work, not a staffing or calendar estimate.** Control-plane and data-plane delivery can proceed in parallel after the external contract freezes; a nice-looking SDK demo must not be called a completed managed service.

| Step | Workstream and deliverable | Predecessor / important detail | Exit proof |
|---|---|---|---|
| **0** | Choose exact KV product contract: read-only vs mutable; consistency/durability; tenancy scope; allowed operations; SLO | Vendor workload and latency budget agreed | Reviewed contract + threat model |
| **1** | Define binding manifest/descriptor + versioned gRPC schema + Go interfaces | Separate ARTF standards from DF extensions | Contract tests and examples |
| **2** | Build reference Go SDK with fake and local gRPC client | No backend required for unit tests | Correct `found` vs `error`, concurrency/deadline tests |
| **3** | Provisioning registry, entitlement checks and lifecycle reconciler | Durable desired/observed state and resource scope | Repeated deploy/delete is idempotent; stale resources reclaimed |
| **4** | Build authenticated regional gateway and choose physical backend | Identity, route cache, quota policy, SRE ownership | Cross-tenant denial; N-1 load/failure benchmark |
| **5** | Binding injection, network restrictions and startup/readiness integration | Pod/workload identity and app mapping | No raw backend credentials; missing required binding blocks serving |
| **6** | Dataset intake, validation, import and version activation | Independent artifact source preferred | Import once per dataset revision; rollback on bad CSV |
| **7** | Local CLI and simulator, Docker and ARTF conformance harness | Same wire schema and error contract | Local get/put/seed + known failure fixtures |
| **8** | Usage/SLA reports, vendor/admin UI, audit, alerts and support runbooks | Metering attribution and per-tenant dashboard | Vendor can diagnose own binding; platform can bill and revoke |
| **9** | Portability & reliability test release candidate | Snapshot alternative or equivalent on alternate host | Same business outputs without silent no-op; chaos, p99.9 tests |
| **10** | Limited production pilot; phased region rollout | Approved legal/privacy/security and support owners | Demand, cost and operational gates from Section 15 |

**Suggested implementation ownership:** developer platform (SDK/CLI/docs); runtime (Kubernetes admission/deployment); control plane (binding lifecycle, portal, entitlements); data platform (KV + import/distribution); security/privacy (identity, egress, audit); SRE (availability/performance/capacity); finance/analytics (usage accounting); vendor engineering (feedback and portability testing). Cross-team contract review is mandatory before independent implementation begins.

### 27.1 Definition of done for the *first* SDK-backed KV deployment

- [ ] A developer builds a real ARTF Go audience sample and uses a documented `Get` contract without any cluster credentials or backend endpoint knowledge.
- [ ] PubMatic can provision, map and authorize a named KV binding per organization/application/environment/region and surface its lifecycle in an operator view.
- [ ] **Tenant A cannot read/write Tenant B's data**, even with identical keys, forged logical names, stolen stale tokens or misconfigured namespace selectors.
- [ ] Production resource persists across app rolling updates and can be explicitly deleted/exported according to the vendor's contract.
- [ ] Vendor CSV validation, idempotent import, per-DC activation, version inspection and rollback work without duplicate imports on pod scale-out.
- [ ] All `Get` outcomes distinguish missing data, denied access, quota, timeout, service outage and stale/missing required dataset.
- [ ] SDK does not extend an auction's deadline and has a tested policy for retries, cancellation, connection pooling, circuit breaker and rate limits.
- [ ] Local Docker/CLI simulation accepts identical SDK calls and deterministic fixture data; the sandbox tests service-specific semantics.
- [ ] A missing required binding on another SSP is either replaced by an explicitly configured portable provider or causes a clear startup/deployment refusal.
- [ ] At intended QPS and N-1 failure load, p99/p99.9 increment, network traffic, CPU, memory, storage and cross-tenant fairness are measured against agreed limits.
- [ ] Dedicated vendor-facing services are separate from internal SSP APIs/data stores and subject to reviewed privacy, egress, encryption, identity and audit controls.
- [ ] Gateway/datastore have named on-call ownership, usage metering, per-vendor diagnostics, capacity model, backup/restore and tested incident runbooks.
- [ ] Protocol/SDK versions, contract tests, upgrade process, deprecation window and issue-response ownership are published.

### 27.2 Decisions that should be resolved in architecture review, even after committing to build

| Decision needing sign-off | Options that materially change cost and design | What decides |
|---|---|---|
| **SDK or local protocol?** | Go-only direct client; stable gRPC gateway with thin Go client; shared sidecar | Number of languages, p99 budget, identity needs |
| **What is KV?** | Read-only snapshot KV; mutable cache; durable key-value database | Correctness, freshness and real vendor state needs |
| **Tenant granularity?** | Org-wide, application, deployment, dataset, or shared named asset | Ownership, reuse, deletion and billing boundaries |
| **Storage isolation?** | Shared partitions, isolated shards, dedicated vendor cluster | QPS, dataset size, customer tier and SLO |
| **How does binding identity work?** | K8s projected token; SPIFFE/mTLS; proxy identity | Existing platform capability, attack model and rotation |
| **What loads embedded CSV?** | Independent artifact upload; isolated image extraction; vendor export command | Supply chain security, image format and refresh cadence |
| **Who owns bulk import?** | Platform reconciler + Job; external pipeline; vendor-managed job | Idempotency, failure recovery and delivery model |
| **How is refresh published?** | Immutable version pointer; per-DC activation; rolling pod updates | Storage semantics, staleness and correctness |
| **How do apps behave off PubMatic?** | Snapshot provider; other-host backend; unsupported required capability | Interoperability commitments and alternate-host policy |
| **What counts as SDK adoption?** | SDK download; active production bindings; usage; vendor-saved work | Avoid mistaking dependency for value |
| **How do we protect auction p99?** | Local cache, batching, co-placement, read-only snapshot, reduced hops | Load tests under N-1 and failure scenarios |
| **Who carries operational responsibility?** | PubMatic SRE, platform data team, vendor or shared responsibility | Contract, escalation, incident and billing readiness |

**Implementation recommendation under the stated commitment:** deliver a **small, versioned Go SDK plus a stable authenticated gRPC binding gateway**, initially for one narrowly specified KV profile, backed by a durable binding registry and Kubernetes reconciliation. Ship a **local emulator and explicit snapshot adapter** alongside it. Keep the optional gateway hop off the critical path for audience workloads unless benchmarks justify it. This creates a functional managed-service platform without putting PubMatic-specific assumptions into every vendor's business logic.

---

## Sources and evidence notes

Public external sources were reviewed on **October 9, 2026**. Product capabilities and third-party rules should be rechecked before implementation. Examples and field names identified as *proposed* are not documented PubMatic capabilities.

- **[S1]** IAB Tech Lab, *Agentic Real Time Framework v1.0* (final; released November 12, 2025), especially Requirements, Integration into the Container Ecosystem, Manifest Requirements, Service Naming. https://github.com/IABTechLab/agentic-real-time-framework/blob/main/Agentic_Real_Time_Framework_Version_1_0_FINAL.md
- **[S2]** Cloudflare, *Bindings (env)*. https://developers.cloudflare.com/workers/runtime-apis/bindings/
- **[S3]** Cloudflare, *Local development* and *Adding local data*. https://developers.cloudflare.com/workers/local-development/ and https://developers.cloudflare.com/workers/local-development/local-data/
- **[S4]** Index Exchange, *Building a Docker container* and *Build your logic*. https://kb.indexexchange.com/index_cloud_partners/real_time_data_integration/building_your_container.htm and https://kb.indexexchange.com/index_cloud_partners/real_time_data_integration/build_overview.htm
- **[S5]** Index Exchange, *Updating models in containers without rebuilding* (September 29, 2026). https://kb.indexexchange.com/index_cloud_partners/real_time_data_integration/updating_models_and_manifests_without_rebuilding.htm
- **[S6]** Index Exchange, *Introducing Index Cloud* (April 21, 2026; updated July 28, 2026). https://www.indexexchange.com/2026/04/21/introducing-index-cloud/
- **[S8]** Cloudflare, *Workers KV — Consistency*. https://developers.cloudflare.com/kv/concepts/how-kv-works/
- **[S9]** OpenTelemetry, *Semantic Conventions*. https://opentelemetry.io/docs/concepts/semantic-conventions/
- **[S10]** OpenTelemetry, *Collector: Agent deployment pattern*. https://opentelemetry.io/docs/collector/deploy/agent/
- **[S11]** Dapr, *Building blocks* and *Components*. https://docs.dapr.io/concepts/building-blocks-concept and https://docs.dapr.io/concepts/components-concept
- **[S12]** Kubernetes, *Managing Service Accounts* (bound service account tokens). https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/

- **[S13]** Kubernetes, *DNS for Services and Pods* (cross-namespace service discovery). https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- **[S14]** Kubernetes, *Services, Load Balancing, and Networking* (pod network model, same-pod localhost). https://kubernetes.io/docs/concepts/services-networking/
- **[S15]** Kubernetes, *Network Policies* (default connectivity, isolation rules and CNI prerequisites). https://kubernetes.io/docs/concepts/services-networking/network-policies/
- **[S16]** Kubernetes, *Assigning Pods to Nodes* (pod affinity and topology controls). https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- **[S17]** Kubernetes, *Topology Aware Routing* (zone-locality hints with safeguards). https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/
- **[S18]** Kubernetes, *Init Containers* (per-pod ordered initialization and retries). https://kubernetes.io/docs/concepts/workloads/pods/init-containers/


- **[S19]** Kubernetes, *Projected Volumes* and *Service Accounts* (rotating, audience-bound workload tokens and volume constraints). https://kubernetes.io/docs/concepts/storage/projected-volumes/ and https://kubernetes.io/docs/concepts/security/service-accounts/
- **[S20]** SPIFFE, *SPIFFE Overview* and *SPIRE Concepts* (workload identities, issuance, attestation). https://spiffe.io/docs/latest/spiffe-about/overview/ and https://spiffe.io/docs/latest/spire-about/spire-concepts/
- **[S21]** Kubernetes, *Jobs* and *Automatic Cleanup for Finished Jobs* (controller execution/retry behavior and cleanup). https://kubernetes.io/docs/concepts/workloads/controllers/job/ and https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/
- **[S22]** Cloudflare, *Supported bindings per development mode* and *KV bindings* (local simulation/default and remote connection choices). https://developers.cloudflare.com/workers/local-development/bindings-per-env/ and https://developers.cloudflare.com/kv/concepts/kv-bindings/
- **[S23]** gRPC, *Retry* (transparent retry, retry policy and explicit disabling). https://grpc.io/docs/guides/retry/
- **[S24]** Kubernetes, *Configure Liveness, Readiness and Startup Probes* (avoiding cascading restart behavior). https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
- **[S25]** OpenTelemetry, *Metrics SDK* and *Common Specification Concepts* (cardinality and attribute limits). https://opentelemetry.io/docs/specs/otel/metrics/sdk/ and https://opentelemetry.io/docs/specs/otel/common/

**Evidence boundary:** External documentation supports claims about public ARTF, Cloudflare, Index, Dapr and Kubernetes behavior. Recommendations, staged options, proposed acceptance thresholds, internal object names and proposed API syntax in this memo are architectural analysis—not claims about existing PubMatic implementation or measured vendor demand.
