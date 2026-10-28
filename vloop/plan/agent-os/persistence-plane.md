# VLOOP Persistence Plane — injected ORM, switchable database backends

Companion documents: `codex-integration.md` (`[CI §n]`),
`settings-plane.md` (`[SP §n]`), `plan.md`, `spec.md`, `tasks.md`.

This document clarifies an earlier comment and defines the persistence
architecture. The target is **not** "replace SQLite with a remote database." The
target is:

- VLOOP owns all durable state;
- persistence is injected through runtime-neutral interfaces;
- repositories use an ORM rather than handwritten SQL;
- embedded SQLite and managed PostgreSQL are supported backends;
- runs are pinned to a persistence/settings generation so a backend or setting
  cannot change underneath them.

All repository references below were inspected directly.

---

## 1. Clarification: what creates the additional risk

Moving state *into SQL* is not inherently risky. SQLite is already a transactional
SQL database and is a good local backend. The concern was narrower:

> Making a remotely managed database the live source of configuration and
> orchestration decisions can add availability, latency, consistency and security
> failure modes unless values are resolved and pinned before execution.

The risks come from **remoteness and coordination**, not from SQL itself.

### 1.1 Availability becomes part of the execution path

With embedded SQLite, a read is an in-process filesystem operation. With a managed
PostgreSQL database, each read can fail because of DNS, TLS, routing, pool
exhaustion, credential expiry, failover or maintenance.

If an agent reads model, policy, skills and credentials from the remote database
throughout a turn, one transient outage can strand the run in a partially applied
state. Therefore:

- resource and settings data are resolved before an attempt starts;
- the immutable snapshot is used for the full attempt;
- event and thread writes are retried idempotently;
- loss of the authoritative store pauses new scheduling rather than silently
  changing behaviour.

### 1.2 Latency amplifies across an agent loop

One remote query is cheap; dozens per model/tool iteration are not. N agents
multiplied by M tool calls multiplied by configuration reads creates a hidden
latency and connection-pool tax.

The runtime hot path therefore uses in-memory snapshots. The database is used for:

- initial resolution;
- durable event/thread append;
- state transitions;
- checkpoints and recovery.

It is not consulted for every tool selection, policy decision or prompt fragment.

### 1.3 Mutable remote values create run drift

A remotely managed row can change while a run is active. Without version pinning,
the same run can use one model key before a tool call and another afterwards, or a
policy can become broader halfway through execution.

The answer is the same as in `settings-plane.md` §5.4: location is not the
correctness mechanism; **version pinning is**. Every run records:

```text
resource_snapshot_id
settings_version
persistence_generation
```

A running attempt never resolves those again.

### 1.4 Distributed concurrency is different from local concurrency

A single-process SQLite deployment can serialize scheduling in memory. A managed
PostgreSQL deployment may have multiple VLOOP workers competing for the same
queued step. That requires durable compare-and-swap or leases, fencing tokens,
heartbeats, idempotency keys and recovery of expired claims.

An ORM maps rows; it does **not** design this protocol.

### 1.5 A local-first product can become network-dependent

If remote PostgreSQL is mandatory, opening a local workspace can fail because the
network is down. That contradicts local-first operation.

VLOOP therefore supports two deployment profiles:

| Profile | Authoritative backend | Intended use |
|---|---|---|
| `embedded` | SQLite | single-user local/offline development |
| `managed` | PostgreSQL | multi-device, multi-worker, centrally managed VLOOP |

Switching profile is an operational migration/cutover, not a transparent
mid-transaction toggle (§8).

### 1.6 Secrets and the bootstrap paradox

A database cannot be the only place that stores the credential needed to connect
to that database. Some root of trust must exist outside it.

VLOOP represents that explicitly with an injected `DatabaseBootstrap`:

- endpoint/database/TLS identity may come from a signed control-plane response or
  a local sealed profile;
- the database password/token is a short-lived credential lease from the OS
  keyring or remote identity service;
- the database URL is never read from ambient environment variables;
- plaintext credentials are never persisted in VLOOP tables or local cache.

This is an unavoidable bootstrap boundary, not configuration debt.

---

## 2. Current repository state

The repository already uses SQLx, but it is **not an ORM**.

- workspace dependency: `sqlx = 0.9.0`, SQLite-only feature set
  (`codex-rs/Cargo.toml:430-441`): `sqlite-bundled`, `chrono`, `json`, `macros`,
  `migrate`, `runtime-tokio`, `tls-rustls`, `time`, `uuid`;
- `codex-state` depends directly on SQLx
  (`codex-rs/state/Cargo.toml:10-25`);
- `codex-state` is intentionally SQLite-specific: `SqlitePool`,
  `SqliteConnectOptions`, WAL, synchronous mode and auto-vacuum
  (`codex-rs/state/src/sqlite.rs:15-23`);
- it maintains six SQLite files with versioned names
  (`codex-rs/state/src/sqlite.rs:29-34`);
- runtime code contains direct SQLx queries and SQLite PRAGMAs, e.g.
  `PRAGMA integrity_check` and `PRAGMA data_version`
  (`codex-rs/state/src/runtime.rs:362`,
  `codex-rs/state/src/runtime/queued_items.rs:44`).

VLOOP already disables this subsystem by injecting `state_db: None` and its own
`ThreadStore` `[CI §4]`. **We do not rewrite `codex-state` to an ORM.** That would
be broad, unrelated churn in a subsystem VLOOP does not use.

The ORM work belongs in VLOOP's persistence crates.

---

## 3. ORM decision

### 3.1 Selected direction: SeaORM, subject to an early compatibility spike

SeaORM is the planned adapter because it is async, uses SQLx underneath, supports
SQLite and PostgreSQL, has entity models, migrations and transactions, and fits
Tokio. The exact pinned version must be selected in task S5 after verifying
compatibility with this workspace's SQLx 0.9 dependency and Bazel rules.

This is intentionally a **planned choice, not a fabricated compatibility claim**.
If the compatible SeaORM release pulls an incompatible SQLx major or creates
unacceptable duplicate driver graphs, S5 compares:

1. SeaORM with an isolated compatible SQLx dependency;
2. Diesel + `diesel-async`;
3. a repository/data-mapper layer on SQLx as the fallback.

The requirement is ORM/data-mapper portability and dependency injection. SeaORM
is preferred, but not at the expense of an unbuildable workspace.

### 3.2 What the ORM does and does not buy

**It does buy:**

- typed entities and relationships;
- reusable CRUD/query composition;
- backend-neutral transactions for common operations;
- migrations managed in code;
- less duplicated SQL and decode logic;
- testability through an injected connection.

**It does not buy:**

- identical locking semantics;
- identical JSON, UUID or timestamp storage;
- identical index/constraint features;
- transparent live switching of an active database;
- distributed scheduling correctness;
- zero backend-specific migrations or queries.

The plan therefore uses a common ORM core plus explicit backend capabilities and
small backend-specific implementations where semantics genuinely differ.

---

## 4. Crate and dependency architecture

```text
vloop-core
   ↑ domain types only; no ORM, no SQLx, no backend type

vloop-persistence            traits, UnitOfWork, repository bundle,
   ↑                         DatabaseCapabilities, health/errors
   │
   ├── vloop-persistence-db
   │      persistence models, migrations, repository implementations,
   │      connection factory, SQLite/PostgreSQL adapters
   │      (SeaORM planned; S5 may select the fallback)
   │
   └── vloop-store-thread
          implements codex_thread_store::ThreadStore using
          Arc<dyn Persistence>, no SQL in the adapter

vloop-server
   └── receives injected Arc<dyn Persistence>
       and passes repositories to orchestrator, API, learning, settings cache
```

No SeaORM type crosses into `vloop-core`, `vloop-runtime` or the API DTOs.

### 4.1 Runtime injection

Runtime selection requires trait objects. Native RPITIT methods (`-> impl Future`)
are not dyn-compatible, so this boundary deliberately follows Codex's own
`ThreadStoreFuture` pattern (`codex-rs/thread-store/src/store.rs:55-68`): explicit
boxed futures, **not** `#[async_trait]` and not RPITIT. This is a justified
exception to the repository's RPITIT preference, not an accidental violation.

```rust
pub type PersistenceFuture<'a, T> =
    Pin<Box<dyn Future<Output = T> + Send + 'a>>;

/// Backend-neutral persistence boundary injected into every VLOOP service.
pub trait Persistence: Send + Sync {
    fn capabilities(&self) -> &DatabaseCapabilities;
    fn health(
        &self,
    ) -> PersistenceFuture<'_, Result<DatabaseHealth, PersistenceError>>;
    fn begin(
        &self,
    ) -> PersistenceFuture<'_, Result<Box<dyn UnitOfWork>, PersistenceError>>;
    fn repositories(&self) -> &RepositorySet;
}

/// Transaction-bound repositories. Commit/rollback consume the unit of work;
/// callers cannot accidentally reuse a completed transaction.
pub trait UnitOfWork: Send {
    fn repositories(&self) -> &RepositorySet;
    fn commit(
        self: Box<Self>,
    ) -> PersistenceFuture<'static, Result<(), PersistenceError>>;
    fn rollback(
        self: Box<Self>,
    ) -> PersistenceFuture<'static, Result<(), PersistenceError>>;
}

pub struct RepositorySet {
    pub workspaces: Arc<dyn WorkspaceRepository>,
    pub resources: Arc<dyn ResourceRepository>,
    pub agents: Arc<dyn AgentRepository>,
    pub orchestration: Arc<dyn OrchestrationRepository>,
    pub threads: Arc<dyn ThreadRepository>,
    pub events: Arc<dyn EventRepository>,
    pub learning: Arc<dyn LearningRepository>,
    pub settings: Arc<dyn SettingsRepository>,
}

The authoritative repositories above do not include the last-known-good settings
cache. Managed mode injects a **separate local cache adapter** for that purpose;
it is not a dual-write replica of orchestration state (§9).

pub enum DatabaseKind { Sqlite, Postgres }

pub struct DatabaseProfile {
    pub kind: DatabaseKind,
    pub endpoint: DatabaseEndpoint,
    pub credential_ref: Option<CredentialRef>,
    pub tls: DatabaseTlsPolicy,
    pub pool: PoolSettings,
    pub generation: PersistenceGeneration,
}
```

The root `RepositorySet` is pool-bound and used for reads or individually atomic
repository operations. The `RepositorySet` returned by `UnitOfWork` is
transaction-bound; its implementations must route every operation through that
unit's transaction rather than silently falling back to the pool. Repository
traits use the same object-safe boxed-future pattern as `Persistence`.

Repository traits return domain types, not ORM entities.

### 4.2 Bootstrap and factory

```rust
pub trait PersistenceFactory: Send + Sync {
    fn connect(
        &self,
        profile: DatabaseProfile,
        credentials: Option<CredentialLease>,
    ) -> PersistenceFuture<'_, Result<Arc<dyn Persistence>, PersistenceError>>;
}

pub trait DatabaseBootstrap: Send + Sync {
    fn profile(
        &self,
    ) -> PersistenceFuture<'_, Result<DatabaseProfile, BootstrapError>>;
}
```

`vloop-server` receives these by constructor injection. No `DATABASE_URL` env var.

---

## 5. Portable schema rules

To keep SQLite and PostgreSQL genuinely interchangeable, the schema uses a
portable subset by default:

- application-generated UUIDs; never auto-increment IDs;
- UTC timestamps represented consistently by the ORM;
- enums stored as validated strings, not backend-native enum types;
- JSON restricted to fields that are not used for critical relational joins;
- normalized join tables for tool, skill, MCP and dependency relationships;
- explicit unique and foreign-key constraints supported by both backends;
- no correctness dependency on collation-specific string comparisons;
- no database trigger required for core business logic;
- no SQLite PRAGMA or PostgreSQL extension in common migrations.

Backend-specific indexes are allowed as optional performance migrations, never as
correctness requirements.

### 5.1 Migrations

One logical migration version sequence is applied to both backends. Every
migration is tested on both SQLite and PostgreSQL in CI.

Rules:

- forward migrations are mandatory;
- destructive changes use expand → backfill → contract across releases;
- application startup never auto-applies destructive migrations in managed mode;
- one migrator acquires a backend-specific migration lock;
- the app refuses to serve writes when the schema is newer or older than its
  supported range;
- migration state is stored in the authoritative backend, not process memory.

---

## 6. Transaction boundaries

The ORM's main architectural value is an explicit `UnitOfWork` around domain
transitions.

Operations that must be atomic:

1. **Start run:** create `run` + `resource_snapshot` reference + initial
   `step_attempt` + `RunStarted` outbox/event row.
2. **Append conversation:** append ordered `thread_items` + advance thread sequence.
3. **Attempt transition:** compare expected version/status + write new status +
   append event + enqueue evaluation if terminal.
4. **Approval resolution:** record decision + update attempt state + append event.
5. **Promotion:** create promotion + memory entry/version + audit event.

The WebSocket bus is not part of the transaction. A transactional outbox row is
written atomically; a dispatcher publishes it after commit. Delivery is
**at-least-once**, so consumers deduplicate by `event_id`.

Exactly-once delivery is not claimed.

---

## 7. Multi-agent and multi-worker concurrency

### 7.1 Optimistic concurrency

Mutable aggregate rows carry a monotonically increasing `version`.
Updates use compare-and-swap semantics:

```text
UPDATE aggregate
SET ..., version = version + 1
WHERE id = ? AND version = expected_version
```

The ORM repository exposes this as `save_if_version`. Zero affected rows means a
conflict, not success.

### 7.2 Step claiming

A queued step is claimed with:

```text
claim_owner
claim_token         random fencing token
claim_expires_at
heartbeat_at
```

Correctness uses a portable conditional update. PostgreSQL may use
`FOR UPDATE SKIP LOCKED` as an optimized backend-specific implementation;
SQLite uses the portable compare-and-swap path. Both must pass the same scheduler
conformance suite.

A worker must present the current fencing token when completing or heartbeating a
claim, preventing an expired worker from committing after a replacement has taken
over.

### 7.3 Ordered appends

`thread_items` and `run_events` use `(stream_id, sequence)` unique constraints.
The next sequence is allocated transactionally. Retries carry an idempotency key,
so a network timeout after commit cannot duplicate an item.

---

## 8. Runtime backend selection and switching

"Configured/injected at runtime" means **selected when constructing a VLOOP
server or tenant runtime**, not swapped under active transactions.

Safe cutover in v0.1 is deliberately quiescent:

1. pause new scheduling;
2. wait for all active attempts to reach a terminal state, or explicitly interrupt
   them and persist that terminal transition;
3. migrate/copy data;
4. verify counts, hashes and foreign keys;
5. construct and health-check a new `Arc<dyn Persistence>` with a new
   `PersistenceGeneration`;
6. replace the server/tenant runtime's handle while no attempt is active;
7. retain the old backend read-only for rollback;
8. resume scheduling.

Runs record `persistence_generation`. **No active attempt crosses generations in
v0.1.** Checkpoint-and-resume across generations is a future feature because it
requires an explicit continuation protocol; it is not smuggled into a connection
swap.

For multi-tenant deployments, the injected persistence handle may be per tenant,
but tenant selection occurs before repository access. Every repository is scoped
by a mandatory `TenantContext`; callers cannot issue an unscoped query. PostgreSQL
row-level security is an optional second defence, not the only defence, because
SQLite has no equivalent.

---

## 9. Remote authority and local cache

The managed profile uses PostgreSQL as authoritative. The embedded profile uses
SQLite as authoritative. VLOOP does **not** transparently dual-write both.

The only planned local cache in managed mode is the settings cache described in
`settings-plane.md`; it stores non-secret, versioned, last-known-good settings for
offline resolution. It is not a general write-behind replica of orchestration
state.

Why not general dual-write:

- two independent commits create split-brain;
- conflict resolution for ordered thread/event streams is hard;
- "local-first then sync" requires an explicit replication protocol, not an ORM;
- stale local orchestration state can execute work twice.

If offline orchestration against a managed deployment becomes a requirement, it
is a separate replication project with globally unique operation IDs, a log and
conflict rules — not part of v0.1.

---

## 10. ORM conformance and escape hatches

All repository behaviour is tested against both backends. The conformance suite
covers:

- CRUD and relationships;
- migrations from every supported version;
- transaction rollback;
- optimistic concurrency conflicts;
- ordered append and idempotent retry;
- step claim/expiry/fencing;
- `ThreadStore` create, append, flush, shutdown, resume and read;
- outbox delivery and deduplication;
- tenant isolation;
- UTC timestamp and JSON round-trip.

Backend-specific SQL is permitted only behind a repository implementation and
must have a portable fallback. Every escape hatch is documented with:

- why ORM composition was insufficient;
- which backend it applies to;
- its portable semantic equivalent;
- tests proving both implementations agree.

---

## 11. Risks

1. **ORM compatibility.** The workspace currently pins SQLx 0.9 with SQLite-only
   features. SeaORM compatibility is unverified; task S5 must pin an exact version
   before dependencies are added.
2. **Portability illusion.** Passing CRUD tests does not prove scheduler locking
   semantics. The claim/fencing conformance tests are the real guard.
3. **Remote database outages.** Existing runs can continue from in-memory
   snapshots only until they need a durable transition or event append. The system
   must pause rather than continue unrecorded.
4. **Connection-pool exhaustion under many agents.** Pool limits, transaction
   duration metrics and backpressure are required. Never hold a DB transaction
   across a model call or tool execution.
5. **Migration coordination.** Multiple server instances must not race migrations.
6. **Bootstrap circularity.** The database credential cannot live only inside the
   database; `DatabaseBootstrap` is unavoidable (§1.6).
7. **SQLite test bias.** Fast local tests can hide PostgreSQL-specific failures;
   CI must run the same conformance suite against a real PostgreSQL service.
8. **ORM entity leakage.** Returning SeaORM entities from repositories would
   recouple the domain to the adapter and make replacement cosmetic. CI/module
   boundaries must prevent it.
9. **No implementation validation yet.** This is an architecture plan; no ORM
   dependency was added and no build/test was run.
