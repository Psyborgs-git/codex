# VLOOP Agent OS v0.1 — DAG Execution Plan

Companion documents: `codex-integration.md` (validated evidence base, cited as
`[CI §n]`), `settings-plane.md` (environment-variable migration, cited as
`[SP §n]`), `persistence-plane.md` (injected ORM persistence, cited as
`[PP §n]`), `plan.md` (architecture), `spec.md` (requirements).

Legend: `[S]` serial / critical path · `[P]` parallelizable · `GATE` milestone ·
`SPIKE` time-boxed investigation · `FC-n` fork change from `[CI §7]` ·
Risk **H** / **M** / **L**.

## 1. Shape of this graph

Two principles drive the ordering.

**De-risk the kernel before building anything on it.** The entire architecture
rests on one claim: `ThreadManager` can be constructed with VLOOP-injected
dependencies and made to hold no durable state `[CI §1.1]`. That claim is proven
in A2 — the second task — using only public API and zero fork changes. If it
fails, the architecture changes, and it is far cheaper to learn that in week one
than after the store, orchestrator and GUI are built on it.

**Ownership before features.** Tier A fork changes and the isolation test suite
land before resource management, orchestration or GUI work. Sole ownership is the
defining requirement (`spec.md` FR-23); treating it as a late hardening pass would
mean discovering write-backs after the surfaces that depend on them exist.

## 2. What changed from the previous revision

| Change | Reason |
|---|---|
| **Deleted** A5 protocol pin + TS/JSON-schema codegen | Integration is compile-time; drift is a build error `plan.md §4.1` |
| **Deleted** B1 codec / dispatch / pending modules | No JSON-RPC. `submit(Op)` / `next_event()` are direct calls `[CI §1.2]` |
| **Deleted** GATE-0 cross-workspace dependency spike | VLOOP joins the `codex-rs` workspace, removing the risk `plan.md §4.1` |
| **New** A2 kernel boot spike, and it is second | Highest-risk claim in the design `[CI §1.1]` |
| **New** Epic B: five Tier A fork changes + isolation suite | Required for sole ownership of state and policy `[CI §7]` |
| **New** Epic C: four Tier B fork changes | Required for sole ownership of resources `[CI §7]` |
| **New** D1 orchestration: DAG, steps, attempts, claims | VLOOP is the orchestrator `plan.md §8` |
| **New** D2 learning loop | The product's reason for existing `plan.md §9` |
| **New** S5, A5, A8 and GATE-P | Select an ORM/data-mapper safely, implement injected SQLite/PostgreSQL adapters, and prove cutover/conformance `[PP §3, §10]` |
| **Restored** full CRUD for providers, models, tools, skills, MCP | No longer config-constrained `spec.md §2` |
| **New** S2, S3, S4 spikes | Two recon agents did not complete; three unverified items are load-bearing `[CI §13]` |
| GATE-2 is now the **ownership** gate, not a feature gate | `spec.md §15` |
| **New** A7 + B7 + B8 + C5: the settings plane, six further fork changes | Ambient env vars cannot express per-agent config, and mutating them is UB under edition 2024 `[SP §3]` |

## 3. High-level DAG

```text
A0 Recon [DONE, partial]
   │
   ▼
A1 Workspace membership + Bazel ──► GATE-0
   │
   ▼
A2 Kernel boot spike (Tier 0, no fork changes) ──► GATE-1  ◄── the architecture
   │                                                            is proven here
   ├──────────┬──────────┬──────────┬──────────┬──────────┐
   ▼          ▼          ▼          ▼          ▼          ▼
  S2         S3         S4         S5         A3         A6
 Op/Event  layer/skill orchestr.  ORM/DB     vloop-core frontend
  enum     verify      recon      compat     domain     foundation
   │          │          │          │
   │          │          │          │          ▼
   │          │          │          │         A4 runtime trait + MockRuntime
   │          │          │          │          │
   │          │          │          └─────────►A5 ORM persistence + ThreadStore
   │          │          │                     │
   │          │          │                     ▼
   │          │          │                    A7 settings
   │          │          │                     │
   │          │          │                     ▼
   │          │          │                    A8 bootstrap/cutover ──► GATE-P
   │          │          │                     │
   └────┬─────┴──────────┼─────────────────────┘
        ▼                │
   Epic B — Tier A fork changes (sole ownership of state + policy)
   B1 FC-1 config   B2 FC-2 skills-install   B3 FC-3/4 execpolicy
   B4 FC-9 mcp-approval-write                B5 kernel adapter
   B7 FC-12/13 injected credentials          B8 FC-11 child env
        │                                          │
        └──────────────────┬───────────────────────┘
                           ▼
                    B6 Isolation suite ──► GATE-2 (sole ownership
                           │                 of state, policy, config)
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   Epic C — Tier B    D3 API gateway     E1/E2/E3 IDE
   C1 FC-5 tools      (+ auth, WS)       workspace/terminal/git
   C2 FC-6/7 skills        │                  │
   C3 FC-8 mcp             ▼                  │
   C4 resources+snapshots  D4 GUI shell       │
   C5 FC-14/15 endpoints   │                  │
        │                  │                  │
        └────────┬─────────┴──────────────────┘
                 ▼
            GATE-3 vertical slice
        (GUI → API → orchestrator → kernel → events → GUI,
         with approval, interrupt, and snapshot replay)
                 │
        ┌────────┴────────┐
        ▼                 ▼
   D1 orchestration   D2 learning loop
   DAG/attempts       eval + promotion
        └────────┬────────┘
                 ▼
            D5 polish / diagnostics
                 ▼
            GATE-4 feature complete
                 │
   ┌──────┬──────┼──────┬──────┐
   ▼      ▼      ▼      ▼      ▼
  F1     F2     F3     F4     F5
 integ  E2E   security fork   build
                      hygiene
   └──────┴──────┬──────┴──────┘
                 ▼
            F6 release gate
```

---

## 4. Epic A — Foundation and de-risking

### A0 [S] Reconnaissance — **COMPLETE (partial)**

**Deliverable:** `codex-integration.md`.

Validated: the in-process kernel seam and `ThreadManager::new` injection surface,
`CodexThread`'s public API, in-process approvals, programmatic config
construction, provider/model injection, `ThreadStore` implementability, the full
Codex write inventory, credential injection, tool/skill/MCP injection seams,
policy seams, a ten-item fork change register, IDE substrate verdicts, terminal
design, sandbox and secrets.

**Two gaps, stated honestly.** The kernel-facade and orchestration/learning recon
agents hit rate limits `[CI §13.1, §13.2]`. §1 of the evidence base was verified
directly instead, but the full `Op` and `EventMsg` variant sets were not
enumerated, and the orchestration primitives were not systematically read. S2 and
S4 close these.

---

### A1 [S] Workspace membership and Bazel wiring — Risk **M**

**Depends on:** A0

**Objective.** Add VLOOP crates to the `codex-rs` workspace with Bazel wiring
intact.

**Files.** `codex-rs/Cargo.toml` (`[workspace.members]`,
`[workspace.dependencies]`), `codex-rs/vloop/*/Cargo.toml`,
`codex-rs/vloop/*/src/lib.rs`, `codex-rs/vloop/*/BUILD.bazel`,
`MODULE.bazel.lock`, `vloop/docs/fork-changes.md` (created here).

**Acceptance.**
- thirteen crate skeletons compile as workspace members: `core`, `runtime`,
  `kernel-codex`, `orchestrator`, `learning`, `settings`, `persistence`,
  `persistence-seaorm`, `store-thread`, `workspace`, `terminal`, `api`, `server`
- `just bazel-lock-update` run and `MODULE.bazel.lock` committed `[CI §12]`
- a `BUILD.bazel` exists per crate; `bazel build //codex-rs/vloop/...` succeeds
- `just test` passes with the new members present
- `just fmt` and `just fix` clean
- `vloop/docs/fork-changes.md` exists with the register format defined

**Tests.** Compile-only. CI must show the Codex suite green with members added.

**Non-goals.** No Codex crate dependencies yet, no domain logic.

---

### A2 [S] Kernel boot spike — Risk **H**, **the pivotal task**

**Depends on:** A1

**Objective.** Boot a Codex kernel with VLOOP-injected dependencies using **only
public API and zero fork changes**, drive one complete turn, and measure exactly
what Codex writes.

**Approach** `[CI §1.5, §2.2, §3.2, §4]`:
- `Config` via the public `load_default_with_cli_overrides_for_codex_home`
  `[CI §2.2]`
- `Config.model_catalog = Some(..)` ⇒ `StaticModelsManager`, no cache file
  `[CI §3.2]`
- `AuthManager` with `AuthCredentialsStoreMode::Ephemeral` + `set_external_auth`
  `[CI §5]`
- `InMemoryThreadStore` as the interim store `[CI §4.2]`
- `ephemeral = true`, `state_db: None`, `agent_graph_store: None`,
  a VLOOP `installation_id` string `[CI §4.3, §4.4]`
- drive `ThreadManager` → `CodexThread::start_turn_if_idle` → `next_event()` loop
- answer one approval with `Op::ExecApproval` `[CI §1.3]`

**Acceptance.**
- a real turn completes against a live provider, streaming events
- an approval round-trips in-process, and a `Denied { rejection }` decision is
  observed to let the agent continue `[CI §1.3]`
- an interrupt stops the turn
- **a filesystem audit of `$CODEX_HOME` before and after produces a complete,
  documented write list** — this is the primary deliverable
- each observed write is mapped to its suppression mechanism or to a fork change
- the residual disk reads in `[CI §2.3]` are confirmed harmless in practice

**Tests.** One integration test in `codex-rs/vloop/kernel-codex/tests/`, using
`TestCodexBuilder` conventions where applicable `[CI §12]`.

**Deliverable.** A findings section appended to `codex-integration.md`, and the
Tier A fork change list confirmed or revised.

**If it fails.** Two fallbacks, in order: embed the app-server in-process via
`app-server-client::InProcess` and accept partial ownership; or return to the
subprocess seam and accept Codex as config owner. Either is a major revision of
`plan.md` and `spec.md` — which is precisely why this task is second.

**Non-goals.** No VLOOP domain model, no HTTP, no GUI, no fork changes.

---

### GATE-0 — Workspace

**Depends on:** A1

VLOOP crates build as workspace members, Bazel is wired, the Codex suite is green.

### GATE-1 — Kernel proven

**Depends on:** A2

A Codex kernel boots with injected dependencies, completes a turn, round-trips an
approval, and its write surface is fully enumerated. **No further work proceeds
until this passes or a fallback is adopted and documented.**

---

### S2 [SPIKE] `Op` and `EventMsg` enumeration — Risk **M**

**Depends on:** A1. **Blocks** B5, D4.

**Question.** What is the complete `Op` variant set (the input surface) and
`EventMsg` variant set (the output surface)?

**Why.** Event translation and the run timeline render these. Recon confirmed only
the approval-related variants `[CI §13.2]`.

**Deliverable.** A variant table appended to `codex-integration.md`, mapped
against `spec.md` FR-19, with any unmapped event given an explicit decision:
translate, ignore, or log-only.

---

### S3 [SPIKE] Layer-stack and skills-snapshot verification — Risk **M**

**Depends on:** A1. **Blocks** B2, C2.

**Two questions.**
1. What does `ConfigLayerStack::effective_config()` return for an empty layer
   vector? The conclusion that bundled skills default to *enabled* — and
   therefore that a `SessionFlags` layer is mandatory — is an **inference**
   `[CI §13.5]`.
2. Can an external caller construct an empty `HostSkillsSnapshot`?
   `SkillLoadOutcome::from_parts` is `pub(crate)` `[CI §13.7]`. FC-7 depends on
   the answer.

**Deliverable.** Both answers with `path:line` evidence, and FC-2 / FC-7 scoped
accordingly. If (1) shows bundled skills default disabled, FC-2 shrinks. If (2) is
negative, FC-7 must add a public constructor.

---

### S4 [SPIKE] Orchestration and learning primitives — Risk **L**

**Depends on:** A1. **Blocks** D1, D2 design finalization.

**Question.** Are `agent-graph-store`, `context-fragments`, `ext/agent`,
`memories/*`, `hooks` or `rollout-trace` worth reusing as VLOOP-side libraries,
given that VLOOP owns orchestration and memory `plan.md §8`?

**Why.** The dedicated recon did not complete `[CI §13.1]`. §8 of the evidence
base is partial.

**Deliverable.** A per-crate verdict — reuse as library, reuse with wrapper, or
ignore — with persistence assumptions noted. The default is ignore; reuse must be
justified.

---

### S5 [SPIKE] ORM and backend compatibility — Risk **H**

**Depends on:** A1. **Blocks** A5.

**Questions.**
1. Which exact SeaORM version is compatible with this workspace's SQLx 0.9 graph,
   Rust toolchain and Bazel rules?
2. Can one entity/migration model target both SQLite and PostgreSQL without
   leaking backend types into VLOOP domain crates?
3. If SeaORM is not viable, is Diesel Async or a SQLx data-mapper adapter the
   lower-risk fallback behind the same persistence traits?

**Why.** The repository currently pins SQLx 0.9 with SQLite-only features
(`codex-rs/Cargo.toml:430-441`) and `codex-state` uses direct SQLite-specific
SQLx APIs, PRAGMAs and six DB files (`codex-rs/state/src/sqlite.rs:15-34`).
SeaORM compatibility has not been verified. Adding an ORM before this spike would
turn an architectural preference into dependency churn.

**Deliverable.**
- a small compile-only crate proving the selected ORM against SQLite and
  PostgreSQL features
- exact pinned versions, feature flags and Bazel dependencies
- one entity, one migration and one transaction exercised against both backends
- a decision record comparing SeaORM, Diesel Async and the SQLx data-mapper
  fallback
- `just bazel-lock-update` output for the selected dependency graph

**Acceptance.** One direction is selected with measured compile/dependency impact;
no ORM type enters `vloop-core`; failure to select an ORM does **not** block the
repository abstraction because the SQLx data-mapper fallback implements the same
interfaces `[PP §3]`.

---

### A3 [P] `vloop-core` domain model — Risk **L**

**Depends on:** GATE-1

**Objective.** Runtime-neutral domain types and the event bus, with **zero Codex
dependencies** — the proof that the domain model is portable.

**Contents.** `Workspace`, `ProviderDef`, `ModelDef`, `Agent`, `AgentVersion`,
`ToolSelection`, `SkillDef`, `McpServerDef`, `Policy`, `ResourceSnapshot`,
`ResourceSnapshotId`, `Task`, `Plan`, `PlanStep`, `WorkspaceClaim`, `Run`,
`StepAttempt`, `AttemptStatus`, `ThreadRef`, `Artifact`, `Evaluation`, `Insight`,
`MemoryEntry`, `Promotion`, `ContextPack`, `VloopEvent`, `EventBus`,
`ApprovalDecision`, `ApprovalRecord`.

**Acceptance.**
- `cargo tree -p vloop-core` shows no `codex-*` dependency
- `ApprovalDecision` covers all six upstream decisions including
  `Deny { reason }` and `TimedOut` `[CI §1.3]`
- the attempt state machine rejects illegal transitions per `spec.md` FR-17
- terminal attempts are immutable by construction
- events serialize to stable camelCase JSON

**Tests.** Transition table; serialization snapshots; a CI assertion that no Codex
type is nameable here.

**Non-goals.** No persistence, no runtime, no HTTP.

---

### A4 [P] `vloop-runtime` trait and `MockRuntime` — Risk **M**

**Depends on:** A3

**Objective.** The `AgentRuntime` trait, capabilities, handles, error taxonomy,
and a scriptable mock.

**Interfaces.** As specified in `plan.md §6.3` — RPITIT with explicit `Send`
bounds, no `#[async_trait]`, no ambiguous bool or `Option` parameters `[CI §12]`.

**Acceptance.**
- `MockRuntime` drives a scripted sequence including an approval request, a denial
  that continues, an interrupt and a failure
- `RuntimeCapabilities` includes `live_reconfiguration`, reported **supported** by
  the kernel runtime `[CI §1.2]`
- trait docs explain the implementer contract per repo guidance

**Tests.** A shared attempt-lifecycle suite that both `MockRuntime` and the kernel
runtime must pass — the mechanism behind `plan.md §12`.

**Non-goals.** No Codex specifics.

---

### A5 [P] Injected persistence plane and `ThreadStore` bridge — Risk **H**

**Depends on:** A3, GATE-1, S5

**Objective.** Build VLOOP's backend-neutral persistence interfaces, the selected
ORM/data-mapper adapter for SQLite and PostgreSQL, migrations, and a Codex
`ThreadStore` bridge over the injected persistence handle.

**Crates** (`[PP §4]`):
- `vloop-persistence` — `Persistence`, `UnitOfWork`, repository traits,
  `DatabaseCapabilities`, `DatabaseProfile`, errors; **no ORM or SQLx types**
- `vloop-persistence-db` — persistence models, migrations, connection factory and
  SQLite/PostgreSQL adapters; SeaORM planned, exact implementation selected by S5
- `vloop-store-thread` — implements Codex `ThreadStore` using
  `Arc<dyn Persistence>`; contains no SQL and exposes no ORM entities

**Schema.** `spec.md §6`, including `outbox_events`, optimistic `version` columns,
claim expiry/fencing fields and `persistence_generation`.

**`ThreadStore`** `[CI §4.1]`: implement all required methods; declare Legacy
history mode because paginated history is Local-only `[CI §4.2]`.

**Acceptance.**
- `vloop-core`, runtime and API crates import no ORM/SQLx/backend type
- runtime connection selected through injected `DatabaseBootstrap` and
  `PersistenceFactory`; **no `DATABASE_URL` ambient env variable** `[PP §4.2]`
- both `embedded` SQLite and `managed` PostgreSQL profiles initialize and migrate
  from the same logical migration sequence `[PP §5.1]`
- migrations idempotent and forward-only; destructive evolution uses
  expand/backfill/contract
- `threads` / `thread_items` round-trip a full conversation on both backends
- resuming a thread rehydrates history from the injected repository
- conversation content is readable with the kernel stopped (`spec.md` FR-18)
- ordered append uses `(stream_id, sequence)` uniqueness plus idempotency keys
  `[PP §7.3]`
- run start, attempt transition, approval resolution and memory promotion use
  explicit `UnitOfWork` transactions `[PP §6]`
- events use a transactional outbox with at-least-once delivery and `event_id`
  deduplication; no exactly-once claim
- the `as_any` / `state_db` forfeiture is documented and tested `[CI §4.2]`
- `run_events` and `thread_items` do not double-record (`spec.md` FR-24)

**Tests.** The **same conformance suite** runs against SQLite and a real
PostgreSQL service: migrations, repositories, rollback, optimistic conflict,
ordered/idempotent append, `ThreadStore` lifecycle/resume, outbox deduplication,
tenant scoping, UTC/JSON round-trip `[PP §10]`.

**Non-goals.** No rewrite of Codex's `codex-state`; VLOOP disables it with
`state_db: None`. No paginated history in v0.1. No transparent dual-write or
local/remote replication `[PP §9]`.

---

### A6 [P] Frontend foundation — Risk **L**

**Depends on:** GATE-0

Vite + React + TypeScript + Tailwind + shadcn/ui, with Zustand, TanStack Query,
Monaco, xterm.js and react-resizable-panels. Shell boots against mocked data.
Types come from VLOOP's own OpenAPI generation; there is no Codex protocol surface
to track.

---

### A7 [P] `vloop-settings` foundation — Risk **M**

**Depends on:** A3, A5

**Objective.** The settings plane: scopes, typed keys, resolver, durable local
cache, and the snapshot type. No Codex changes yet.

**Contents** (`[SP §6]`): `SettingKey` with its five classes, `SettingsSource`,
`SettingsResolver`, `SettingsSnapshot`, `EnvMaterializer`, `CredentialLease`,
`ScopeSelector`, `HostEnvironment`.

**Acceptance.**
- scope precedence resolves `global → workspace → agent → agent_version → run`,
  and the resolved value records **which scope supplied it** `[SP §5.1]`
- each key class carries its own failure semantics: credential, trust and policy
  keys **fail closed**; endpoint and behavioural keys fall back to last-known-good
  `[SP §5.2]`
- `SettingsSnapshot` is immutable and version-stamped, and is embedded in
  `ResourceSnapshot` so a run resolves settings once `[SP §5.4]`
- the local settings cache is durable and usable when the remote is unreachable,
  but is not a second source of truth; it serves versioned last-known-good values
  under the per-key failure rules `[SP §5.6, PP §9]`
- credential plaintext is never written to the cache or the database; only leases
  and references `[SP §5.5]`
- a remote value cannot broaden a policy ceiling beyond the local one
  `[SP §5.5]`

**Tests.** Scope precedence table; per-class failure semantics; offline resolution
with a warm cache; a narrowing-violation rejection; a `SecretString` zeroization
check.

**Non-goals.** No Codex integration, no fork changes, no remote transport beyond
a `SettingsSource` implementation behind the trait.

---

### A8 [P] Persistence bootstrap, health and controlled cutover — Risk **H**

**Depends on:** A5, A7

**Objective.** Select and inject the authoritative persistence backend at server
or tenant-runtime construction, expose health/capabilities, and implement the
controlled backend cutover protocol. "Runtime configurable" must not mean
mutating a connection string beneath active transactions.

**Interfaces** `[PP §4.2, §8]`: `DatabaseBootstrap`, `PersistenceFactory`,
`DatabaseProfile`, `PersistenceGeneration`, `DatabaseHealth`.

**Acceptance.**
- `embedded` profile constructs SQLite; `managed` profile constructs PostgreSQL
- no `DATABASE_URL` or database credential is read from ambient env
- endpoint, database name and TLS policy may come from a signed settings/control
  plane; the credential is a short-lived lease from the secret plane, resolving
  the bootstrap paradox explicitly `[PP §1.6]`
- startup validates backend health, schema compatibility and required
  `DatabaseCapabilities` before serving writes
- cutover pauses new scheduling and waits for all active attempts to terminate (or
  explicitly interrupts and persists them), then migrates and verifies data,
  constructs a new persistence handle with a new generation, replaces the handle
  while quiescent, retains the old backend read-only for rollback, and resumes
  `[PP §8]`
- no active attempt crosses persistence generations in v0.1; checkpoint/resume
  across generations is deferred because it needs an explicit continuation
  protocol
- managed mode does not dual-write orchestration state to local SQLite; only the
  non-secret, versioned settings cache remains local `[PP §9]`
- failed validation or migration leaves the old backend authoritative and starts
  no work on the candidate backend

**Tests.** Bootstrap for both profiles; invalid credential/TLS/schema rejection;
cutover dry run; rejection while an active attempt exists; rollback after failed
verification; generation recorded on runs before and after a successful quiescent
cutover; no-ambient-`DATABASE_URL` regression.

**Non-goals.** No offline orchestration replication for managed deployments. That
requires a separate operation log and conflict protocol, not an ORM `[PP §9]`.

---

### GATE-P — Persistence portability

**Depends on:** A5, A8

The same repository, migration, scheduler-claim and Codex `ThreadStore`
conformance suite passes against embedded SQLite and a real PostgreSQL service.
Backend bootstrap is injected, the active profile is health-checked, and a
controlled cutover/rollback rehearsal succeeds. No ORM or SQLx type crosses into
`vloop-core`, runtime or API crates.

---

## 5. Epic B — Sole ownership of state and policy (Tier A)

Eight fork changes plus the isolation suite. Each fork change follows the hygiene
rules in `plan.md §15`: minimal, reversible, documented in
`vloop/docs/fork-changes.md`, and **defaulting to current Codex behaviour when no
VLOOP snapshot is supplied**.

### B1 [S] FC-1 — programmatic config construction — Risk **L**

**Depends on:** GATE-1

**Change.** Make `load_config_with_layer_stack` public, or add
`ConfigBuilder::from_parts(ConfigToml, ConfigOverrides, ConfigLayerStack,
codex_home)` that skips `load_config_layers_state`
(`codex-rs/core/src/config/mod.rs:3078`, `:1393`) `[CI §7]`.

**Acceptance.** A typed `ConfigToml` + `ConfigOverrides` + explicit layer stack
produces a `Config` with no user, project, system or managed config read. One line
plus a thin wrapper. Codex behaviour unchanged for existing callers.

**Tests.** Config built from parts with `$CODEX_HOME` read-only.

---

### B2 [S] FC-2 — suppress bundled-skill installation — Risk **M**

**Depends on:** B1, S3

**Change.** Make the skills service injectable, or accept a
`bundled_skills_enabled` policy parameter, so `install_system_skills` never runs
(`codex-rs/core/src/thread_manager.rs:438-442`;
`codex-rs/ext/skills/src/host_service.rs:132`) `[CI §7]`.

**Why it matters.** `ThreadManager::new` writes to `CODEX_HOME/skills/.system`
**before any thread exists**, and the flag is read from the *layer stack*, not
`ConfigToml` `[CI §2.4, §4.3]`. Easy to miss in a "no persistence" audit.

**Acceptance.** No write under `$CODEX_HOME/skills` during kernel construction,
verified with a read-only directory. Scope confirmed by S3.

---

### B3 [S] FC-3 + FC-4 — injected execpolicy, no write-back — Risk **M**

**Depends on:** B1

**Change.** Make `ExecPolicyManager` and its constructor public (or add
`pub fn from_policy(Arc<Policy>)`), add a no-persist mode, and let
`SessionSpawnArgs` accept a caller-supplied manager, skipping the disk-load branch
(`codex-rs/core/src/exec_policy.rs:278,295,452,502`;
`codex-rs/core/src/session/mod.rs:414,547-567`) `[CI §7]`.

**Acceptance.**
- VLOOP builds an in-memory `execpolicy::Policy` and the session uses it verbatim
- no `*.rules` file is read at session start
- approving a rule amendment updates **VLOOP** policy and writes no file
  `[CI §4.3]`
- with no supplied policy, Codex's disk-load behaviour is unchanged

---

### B4 [P] FC-9 — suppress MCP approval config writes — Risk **L**

**Depends on:** B1

**Change.** Gate `persist_codex_app_tool_approval` and
`persist_non_app_mcp_tool_approval`
(`codex-rs/core/src/mcp_tool_call.rs:2047,2099`) `[CI §7]`.

**Acceptance.** "Always allow" for an MCP tool writes no Codex config; the
decision lands in VLOOP's policy store and enters the next snapshot.

---

### B5 [S] Kernel adapter — Risk **H**, largest single task

**Depends on:** A4, A5, B1, B2, B3, S2

**Objective.** `vloop-kernel-codex`: the only Codex-aware crate.

**Modules** (`plan.md §6`, each under 500 LoC `[CI §12]`):

| Module | Responsibility | Reference |
|---|---|---|
| `manifest.rs` | `RuntimeManifest` → Codex dependency set | `plan.md §6.1` |
| `config.rs` | in-memory `Config` assembly | `[CI §2.2]` + FC-1 |
| `layers.rs` | `ConfigLayerStack` with `SessionFlags` entries | `[CI §2.4]` |
| `models.rs` | `StaticModelsManager` from VLOOP catalog | `[CI §3.2]` |
| `auth.rs` | ephemeral `AuthManager` + `ExternalAuth` bridge | `[CI §5]` |
| `policy.rs` | VLOOP policy → `Permissions` + execpolicy | `[CI §6.4]` + FC-3/4 |
| `kernel.rs` | `ThreadManager` lifecycle, panic containment | `[CI §1.1]` |
| `session.rs` | `CodexThread` driving: `submit` / `next_event` | `[CI §1.2]` |
| `translate/event.rs` | `EventMsg` → `VloopEvent` | S2, `[CI §3]` |
| `translate/approval.rs` | request → VLOOP; decision → `ReviewDecision` | `[CI §1.3]` |
| `isolation.rs` | runtime assertions that no Codex state is written | `plan.md §6.2` |
| `capabilities.rs` | capability descriptor | — |

**Acceptance.**
- implements `AgentRuntime` and passes the shared lifecycle suite from A4
- a manifest fully determines run behaviour; nothing is read from Codex config
- all six approval decisions round-trip, with `Deny { reason }` passing the
  reason to the agent and `TimedOut` used for expiry `[CI §1.3]`
- event translation is one typed hop; timestamps normalize to RFC 3339 UTC
- output reaching the UI is byte-bounded per item; full stream to the run log
- policy translation is **provably narrowing**, over the full matrix
- **a panic inside a turn fails the attempt, not the process** `plan.md §14`
- `isolation.rs` assertions active in debug builds

**Tests.** Manifest → dependency mapping compared as whole objects; the full
approval matrix; policy narrowing over every combination; event translation
against fixtures captured in A2; a panic-injection test proving attempt-level
containment.

**Non-goals.** No HTTP, no GUI, no orchestration logic.

---

### B7 [S] FC-12 + FC-13 — injected credentials — Risk **H**

**Depends on:** A7, B1

**Objective.** Remove the two multi-agent credential defects: process-global
provider keys, and the ungated auth precedence path. This is Wave 1 of
`[SP §8]`.

**Changes.**
- **FC-12**: `ModelProviderInfo::api_key()` accepts an injected credential
  resolver instead of reading `std::env::var(env_key)`; the env path is retained
  as a fallback when no resolver is supplied
  (`codex-rs/model-provider-info/src/lib.rs:331-347`) `[SP §7]`
- **FC-13**: gate the `CODEX_ACCESS_TOKEN` read on an explicit flag, closing the
  precedence leak (`codex-rs/login/src/auth/manager.rs:1460-1470`) `[SP §7]`

**Acceptance.**
- **two agents run concurrently in one process against the same provider with
  different credentials**, and neither observes the other's value — the test that
  directly falsifies defect (a) `[SP §3]`
- a stray `CODEX_ACCESS_TOKEN` in the environment does not outrank an injected
  credential `[SP §3]`
- credentials arrive as short-lived leases; no plaintext reaches the database or
  the settings cache `[SP §5.2]`
- credential resolution failure **fails the run closed**, before the kernel starts
- with no resolver supplied, Codex's existing env behaviour is unchanged, so its
  own test suite passes

**Tests.** Concurrent per-agent divergence; precedence-leak regression; lease
expiry mid-run; fail-closed on resolution failure.

**Non-goals.** No child-environment work (B8). No endpoint or trust migration
(C5).

---

### B8 [S] FC-11 — child-process environment materialization — Risk **M**

**Depends on:** A7, B5

**Objective.** Make every child environment a pure function of
`(settings snapshot, host-local allowlist, shell policy)` rather than of the host
process environment. Wave 2 of `[SP §8]`.

**Change (FC-11).** Redirect the seven non-test `create_env` call sites to
`create_env_from_vars` with a supplied variable map, keeping `create_env` for
existing callers `[SP §7]`:
`core/src/exec_env.rs:35`, `core/src/tasks/user_shell.rs:162`,
`core/src/tools/handlers/shell/shell_command.rs:110`,
`core/src/unified_exec/process_manager.rs:1221`,
`exec-server/src/local_process.rs:684`,
`app-server/src/request_processors/command_exec_processor.rs:152`,
`cli/src/debug_sandbox.rs:260`.

**Read `ShellEnvironmentPolicy`'s set/include/exclude precedence before
implementing** — FC-11's correctness depends on semantics this plan has not
verified `[SP §10]`.

**Acceptance.**
- `EnvMaterializer` output feeds `create_env_from_vars`, then
  `scrub_non_inheritable_env_vars`, so existing protections are layered on rather
  than routed around `[SP §4]`
- `NON_INHERITABLE_ENV_VARS` remain absent from materialized child environments
  `[SP §9]`
- host-local variables (`PATH`, `TMPDIR`, `TERM`, locale) still pass through;
  `CODEX_SANDBOX*` handling is **untouched** `[SP §5.3]`
- two concurrent agents produce different child environments from one process
- **no VLOOP crate calls `std::env::set_var` or `remove_var`**, enforced by CI
  grep, since edition 2024 makes these `unsafe` and the failure is a race rather
  than a compile error `[SP §3]`

**Tests.** Materialization as a pure function over fixtures; scrubbing preserved;
per-agent child-env divergence; a CI grep for host env mutation.

---

### B6 [S] Isolation test suite — Risk **H**

**Depends on:** B5, B7, B8

**Objective.** Prove `spec.md` FR-23 and FR-27 — that Codex holds no durable
state, no configuration authority, and no ambient environment dependency.

**Acceptance**, each an automated test:
1. **A full task executes with `$CODEX_HOME` read-only.**
2. No Codex config file is read during any VLOOP operation.
3. `ephemeral = true` asserted on every thread start *and fork* `[CI §4.4]`.
4. Bundled-skill install never runs, with suppression carried in a `SessionFlags`
   layer, not only `ConfigToml` `[CI §2.4]`.
5. None of these is created: rollout file, Codex SQLite database,
   `models_cache.json`, `history.jsonl`, `default.rules`, installation-id file,
   MCP credential file `[CI §4.3]`.
6. Credentials resolve via `ExternalAuth`; `auth.json` never read `[CI §5]`.
7. With a stray `CODEX_ACCESS_TOKEN` in the environment, VLOOP's resolver still
   wins — the known precedence leak `[CI §5]`.
8. **Hermetic-env test: a full run executes with the process environment reduced
   to a minimal host-local allowlist.** Any credential, endpoint or trust value
   arriving from ambient env fails the test. This is the primary guard for the
   settings plane, and it also catches transitive reads inside dependencies
   `[SP §9]`.
9. **No VLOOP crate calls `std::env::set_var` or `remove_var`**, enforced by CI
   grep, because edition 2024 makes these `unsafe` and the failure is a race
   rather than a compile error `[SP §3]`.
10. **Env-read ratchet:** the non-test `env::var*` count in the embedded crates
    does not exceed a recorded ceiling, lowered each migration wave. Prevents
    regrowth `[SP §9]`.
11. `NON_INHERITABLE_ENV_VARS` remain absent from materialized child environments,
    so existing scrubbing is layered on rather than routed around `[SP §4]`.

**Tests.** A dedicated suite that must run in CI on every change, since both the
`ephemeral` regression and ambient-env leakage are silent by nature.

---

### GATE-2 — Sole ownership

**Depends on:** B6, GATE-P

Codex holds no durable state, no configuration authority, and no ambient
environment dependency for credentials, endpoints or trust material. The
persistence layer is injected and portable across the two supported backends.
This is the primary release gate (`spec.md §15`), promoted here because everything
downstream assumes it.

---

## 6. Epic C — Sole ownership of resources (Tier B)

### C1 [P] FC-5 — VLOOP-owned tool selection — Risk **M**

**Depends on:** GATE-2

**Change.** New crate `codex-rs/tool-policy` with `ToolSelection`,
`ToolCatalogInput`, `ResourceSnapshotId`; applied in `finalize_tool_router` after
`:324` via `entries_mut` / `remove`
(`codex-rs/core/src/tools/spec_plan.rs:318`;
`codex-rs/core/src/tools/registry.rs:386,396`) `[CI §7]`.

Delivered to core through `ExtensionDataInit` at thread start, mirroring
`WaitForEnvironmentToolConfig` `[CI §6.1]`.

**Acceptance.**
- an allow list, a deny list, `unlisted: deny`, and per-tool exposure overrides
  all take effect
- a denied tool is absent from model-visible specs **and** rejected if called
- denied tools are absent from tool search and code mode `[CI §6.1]`
- a VLOOP allow list wins over config and feature gates that would add built-ins
- **new crate, not `codex-core`** `[CI §12]`; `CoreToolRuntime` stays private
- with no `ToolCatalogInput`, tool assembly is unchanged

**Tests.** Table test over selection shapes comparing the whole spec list; a
denied-call dispatch test; a test that feature-enabled built-ins are still
excluded.

---

### C2 [P] FC-6 + FC-7 — VLOOP-owned skill catalog — Risk **M**

**Depends on:** GATE-2, S3

**Change.** Add `SkillCatalogProvider` + `SkillCatalogSnapshot`, blanket-adapted
to the public `SkillProvider`; add `SkillToolAuthority::Custom` so
`skills.list` / `skills.read` serve VLOOP skills; gate host discovery and the
legacy core injection path when a VLOOP snapshot is present
(`codex-rs/ext/skills/src/provider.rs:71`, `tools/mod.rs:221,234,66-70`;
`codex-rs/core/src/session/turn_context.rs:861-884`;
`session.rs:552`; `turn.rs:712-734,782-808`) `[CI §7]`.

**Acceptance.**
- an in-memory catalog renders the same fragments as host skills, verified by
  `insta` snapshot
- host discovery does not run; `TurnContext::skills_snapshot()` stays infallible
  `[CI §6.2]`
- `skills.list` / `skills.read` work against VLOOP skills, with pagination
- a `$mention` for a host skill absent from the VLOOP snapshot injects nothing
- context injection, budgeting and mention resolution are reused unchanged
- with no VLOOP snapshot, host skill behaviour is unchanged

**Note.** Scope depends on S3 question 2; a public `SkillLoadOutcome` constructor
may be required `[CI §13.7]`.

---

### C3 [P] FC-8 — VLOOP-owned MCP catalog — Risk **M**

**Depends on:** GATE-2

**Change.** Add `McpServerContribution::ReplaceCatalog { servers }`; honour it in
`runtime_config_with_context` by starting from an empty builder and skipping
plugin and Apps registration
(`codex-rs/ext/extension-api/src/contributors/mcp.rs:130`;
`codex-rs/core/src/mcp.rs:152,245-260`) `[CI §7]`.

**Acceptance.**
- VLOOP definitions **replace** rather than overlay config- and plugin-derived
  servers `[CI §6.3]`
- `codex_apps` is absent even with the feature enabled and ChatGPT auth present
- connection, tool listing and invocation reuse `codex-mcp` / `rmcp-client`
  unchanged
- the catalog-revision guard still rejects stale prepared calls — not publicized,
  not weakened `[CI §7]`
- two threads with different snapshots see disjoint tool sets
- with no `ReplaceCatalog`, MCP resolution is unchanged

---

### C4 [S] Resource management and snapshots — Risk **M**

**Depends on:** A5, C1, C2, C3

**Objective.** VLOOP-owned CRUD for all resources, plus the snapshot mechanism.

**Sub-tasks.**
- **C4.1** Providers and models (`spec.md` FR-08, FR-09), including one-time
  import of the bundled catalog. No wire-protocol selector `[CI §3.1]`.
- **C4.2** Agents with versioning (`spec.md` FR-10); every save creates an
  `agent_versions` row.
- **C4.3** Tool inventory and selections (FR-12), including a stated reason when a
  tool cannot be enabled for an upstream cause.
- **C4.4** Skills CRUD plus `SKILL.md` import using the upstream parser (FR-13).
- **C4.5** MCP CRUD with client-side validation mirroring upstream rules (FR-14).
- **C4.6** Policies (FR-22) and **resource snapshots** (FR-25).

**Acceptance.**
- resolution produces a `resource_snapshots` row; failure blocks the run with a
  specific error naming the resource `spec.md §7`
- editing a resource mid-run does not affect the running attempt
- a stale-snapshot tool call is rejected `[CI §6.3]`
- a run replayed from its snapshot after its agent was edited behaves identically
- no Codex config is read or written by any CRUD operation

**Tests.** Snapshot immutability under concurrent edit; replay equivalence;
resolution failure cases; validation parity with upstream MCP rules.

---

### C5 [P] FC-14 + FC-15 — endpoints, trust material, behavioural settings — Risk **M**

**Depends on:** A7, B8

**Objective.** Move the remaining non-host-local configuration inputs onto the
settings plane. Wave 3 of `[SP §8]`.

**Changes.**
- **FC-14**: accept CA/trust material as bytes from a caller rather than only via
  `CODEX_CA_CERTIFICATE` / `SSL_CERT_FILE`
  (`codex-rs/http-client/src/custom_ca.rs:61-62`) `[SP §7]`
- **FC-15**: accept MCP protocol version, startup timeouts and endpoint overrides
  from config or caller rather than env (`codex-rs/codex-mcp/src/mcp/mod.rs:64`,
  `codex-rs/rmcp-client/src/utils.rs`) `[SP §7]`

**Acceptance.**
- trust material is **integrity-checked before use**; a bundle that fails
  validation fails the run rather than silently falling back to system roots
  `[SP §5.5]`
- endpoint and behavioural keys fall back to last-known-good cache when the remote
  is unreachable; trust keys fail closed `[SP §5.2]`
- per-agent MCP protocol version and timeouts are expressible
- with no supplied value, existing env behaviour is unchanged

**Tests.** Corrupt-CA rejection; per-agent MCP version divergence; offline
fallback per key class.

**Deferred.** **FC-16** — a `HostEnvironment` abstraction so embedded crates read
from an injected view rather than `std::env` — is deliberately last and may not
land in v0.1. FC-11..FC-15 remove the multi-agent defects on their own; the
residual reads are then either host-local by design `[SP §5.3]` or low-risk
behavioural. Decide with the ratchet count from F1 in hand.

---

## 7. Epic D — Orchestration, API, GUI, learning

### D1 [S] Orchestration — Risk **M**

**Depends on:** A4, A5, C4, S4, GATE-P, GATE-P

**Objective.** `vloop-orchestrator`: tasks, plan DAGs, sequential scheduling,
attempts.

**Acceptance.**
- a task with no plan gets a single-step plan on execution
- `depends_on[]` and `workspace_claim` persist and validate even though execution
  is sequential `plan.md §8.2`
- a cyclic graph is rejected at save time
- the scheduler asserts co-scheduled claims do not overlap
- each attempt maps 1:1 to a `CodexThread`
- retry creates a new attempt; prior rows are immutable
- the `Scheduler` trait returns a `Vec<StepId>` even though v0.1 returns at most
  one — the parallel contract is present from the start
- interrupt propagates to the kernel and reaches `cancelled`
- an approval timeout sends the native `TimedOut` decision `[CI §1.3]`

**Tests.** DAG validation including cycles; claim overlap detection; attempt
lifecycle; retry immutability; interrupt mid-stream; approval timeout. Uses
`MockRuntime` for determinism.

---

### D2 [P] Learning loop — Risk **M**

**Depends on:** D1, S4

**Objective.** `vloop-learning`: signal extraction, deterministic evaluation,
manual promotion with provenance (`spec.md` FR-26).

**Acceptance.**
- attempt completion extracts signals: outcome, approval decisions, tool errors,
  retries, diff size, test results, tokens, wall time
- deterministic evaluators record an `evaluations` row — tests passed, patch
  applied, expected file present, command exit zero
- candidate `insights` carry scope and confidence
- promotion is **manual**, writing a `promotions` row with source attempt,
  evaluator and confidence
- a promoted insight appears in the next run's `ContextPack`, is disableable, and
  is traceable to origin
- memory is token-budgeted and competes with files and history

**Non-goals.** No automatic promotion, no model-graded evaluation — v0.2
`plan.md §9.3`.

---

### D3 [S] API and event gateway — Risk **M**

**Depends on:** A5, B5, C4

**Objective.** `vloop-api` and `vloop-server`: HTTP CRUD, domain event WebSocket,
terminal WebSocket, authentication.

**Acceptance.**
- the contract in `spec.md §8` is served
- **bearer-token auth on every route and both WebSocket endpoints**, token
  generated at startup `spec.md §11`
- WebSocket handshakes validate `Origin` against an allowlist
- two separate WS endpoints; terminal throughput does not share a backpressure
  domain with domain events
- the error model in `spec.md §10` is returned uniformly
- the event bus is bounded and emits an explicit `Lagged` event
- thread endpoints serve from VLOOP's store and work with the kernel stopped

**Tests.** Unauthenticated rejection on every route; origin rejection; error-model
shape; lag behaviour; thread read with the kernel stopped.

---

### D4 [P] GUI — Risk **M**

**Depends on:** A6, D3, S2

**Sub-tasks.** Shell and navigation; explorer; editor; run view with plan DAG,
attempt list and event timeline; agent panel; approval UI; management screens;
learning view; command palette.

**Acceptance.**
- the shell renders without waiting on the kernel
- streamed deltas concatenate correctly per item
- the timeline is virtualized and output-bounded
- **approval UI exposes all six decisions**, with deny-and-continue versus
  deny-and-stop unambiguous, and an editable deny reason `[CI §1.3]`
- unknown enum values render without error `[CI §3.2]`
- retries display as separate attempts
- the command palette uses the upstream fuzzy matcher, remembering it is
  **lower-score-is-better** `[CI §9.1]`
- deferred capabilities state their status rather than appearing broken

---

### GATE-3 — Vertical slice

**Depends on:** C4, C5, D1, D3, D4

End to end: GUI → API → orchestrator → kernel → event stream → GUI, proving
resource resolution into a snapshot, a streaming turn, an approval round-trip
including deny-and-continue, an interrupt, snapshot replay, and persistence of
conversation content in VLOOP's store.

---

### D5 [P] Polish and diagnostics — Risk **L**

**Depends on:** E1, E2, E3, D1, D2

Loading, error and empty states; reconnect handling; structured logs carrying run,
attempt, task and thread ids; captured kernel tracing via VLOOP's own subscriber
`[CI §11.3]`; bounded output with full retention in run logs; keyboard shortcuts;
optional OTLP export tagging spans with the run id.

---

## 8. Epic E — IDE surfaces

### E1 [P] Workspace, files, editor, search — Risk **M**

**Depends on:** D3, D4

**Sub-tasks.** Workspace open/close with Git detection; file operations including
**rename** (new code, `[CI §9.2]`) behind a root guard with symlink-safe atomic
saves `[CI §9.1]`; Monaco with diff view via `similar` `[CI §9.3]`; watcher
integration with debounce and **re-stat to infer event kind** `[CI §9.1]`; quick
open `[CI §9.1]`; **full-text search — new code** `[CI §9.2]`.

**Acceptance.** Paths outside the root rejected; quick open responsive at 50,000
files; external modification invalidates the editor buffer; the inverted fuzzy
score convention handled correctly `[CI §9.1]`.

---

### E2 [P] Terminal — Risk **M**

**Depends on:** D3, D4

`vloop-terminal` on `codex-utils-pty` `[CI §10]`, with a control-channel enum
mirroring app-server's shape, raw byte streaming, and a replay buffer bounded by
**both** event count and retained bytes `[CI §10]`.

**Acceptance.** Multiple concurrent sessions; stdin, resize, interrupt, terminate;
reconnect resumes from the buffer; teardown kills the process group with no
orphans; the stream is authenticated and separate from the domain bus.

**Non-goals.** No sandboxing — user-driven execution `plan.md §3`.

---

### E3 [P] Git — Risk **M**

**Depends on:** E1

Per-file status, worktree and staged diff, stage, unstage, commit, branch list,
create, checkout. Upstream reuse limited to repo discovery and branch listing;
the rest invokes `git` directly so **user commit hooks run** `[CI §9.2]`.

**Acceptance.** Porcelain parsed into per-file staged/unstaged entries; commit runs
hooks; non-repository returns `GIT_NOT_A_REPOSITORY`; **argv arrays, never shell
interpolation**.

**Tests.** Fixture repo covering added, modified, deleted, renamed, untracked; hook
execution assertion; argument escaping for paths with spaces and quotes.

---

### GATE-4 — Feature complete

**Depends on:** C4, C5, D1, D2, D3, D4, D5, E1, E2, E3

All requirements in `spec.md §5` pass, including the 20-step golden path
(`spec.md §14`).

---

## 9. Epic F — Verification and release

### F1 [S] Backend integration suite — Risk **M**
**Depends on:** GATE-4

Kernel construction with injected dependencies; event translation against
fixtures; the full approval matrix including deny-and-continue and timeout;
interrupt; panic containment; `ThreadStore` conformance; resume equivalence;
snapshot immutability and replay; resource resolution failures; DAG validation;
attempt lifecycle and retry immutability; startup recovery; terminal lifecycle;
file operations and traversal rejection; Git against a fixture repo; evaluation and
promotion provenance.

**Persistence plane** `[PP §10]`: run the identical migration, repository,
transaction rollback, optimistic-conflict, ordered/idempotent append,
step-claim/expiry/fencing, Codex `ThreadStore`, outbox/deduplication, tenant-scope,
and UTC/JSON round-trip suite against embedded SQLite and a real PostgreSQL
service. Include managed-store outage, pool saturation and generation-pinned
cutover/rollback tests.

**Settings plane** `[SP §9]`: hermetic-env run; per-agent credential divergence
under concurrency; per-class failure semantics; offline run with a warm cache;
settings pinning and replay; policy-narrowing rejection; scrubbing preserved. Also
**records the env-read ratchet count** used by F5.

Run via `just test` scoped per crate `[CI §12]`, integration-first per repo
guidance.

### F2 [P] Frontend E2E — Risk **M**
**Depends on:** GATE-4

The golden path against a fixture repository using `MockRuntime` for determinism,
plus one live-kernel pass. Must include approval, deny-and-continue, interrupt and
replay.

### F3 [P] Security verification — Risk **H**
**Depends on:** GATE-4

Auth on every route and both WS endpoints; origin rejection; traversal rejection;
Git argv safety; no secret value in any log; `auth.json` never read; the
`CODEX_ACCESS_TOKEN` precedence leak covered `[CI §5]`; sandbox and approval
enforcement demonstrably active; secrets not retained in process memory beyond
their request `plan.md §13`.

**Settings plane:** no credential, endpoint or trust value reachable from ambient
env under the hermetic-env run; credential plaintext absent from the database and
the settings cache; a remote value cannot broaden a sandbox or approval ceiling;
trust material is integrity-checked before use `[SP §5.5]`.

### F4 [P] Fork hygiene audit — Risk **M**
**Depends on:** GATE-4

Every fork change documented in `vloop/docs/fork-changes.md` with rationale, diff
scope and upstream note; individually reversible; no VLOOP logic in `codex-core`;
no forbidden symbol publicized `[CI §7]`; **every behaviour gate verified to
default to current Codex behaviour when no VLOOP snapshot is supplied**.

### F5 [P] Build and non-regression — Risk **M**
**Depends on:** F1, F2, F3, F4

The full Codex suite passes after all fork changes; Bazel lockfile and
`BUILD.bazel` current `[CI §12]`; `just fmt` / `just fix` clean; frontend
typecheck, lint and build pass; `just write-config-schema` run if `ConfigToml`
changed; **the env-read ratchet is at or below its recorded ceiling** `[SP §9]`;
CI provisions a real PostgreSQL service and runs GATE-P's migration/conformance
suite; the selected ORM/data-mapper versions match S5's pinned decision; the dev
script starts and stops both embedded and managed profiles; documented setup works
from a clean clone.

### F6 [S] Release gate — Risk **L**
**Depends on:** F5

Release checklist; known limitations naming each deferred capability from
`spec.md §4`; architecture notes; the fork change register; developer quickstart;
resolved status of every unverified item in `[CI §13]`.

---

## 10. Parallelization matrix

| Workstream | Unblocked after |
|---|---|
| Workspace + Bazel | A0 |
| **Kernel boot spike** | A1 |
| S2, S3, S4, S5 spikes | A1 |
| `vloop-core` domain | GATE-1 |
| Runtime trait + mock | A3 |
| Persistence interfaces + ORM adapters + `ThreadStore` | A3 + GATE-1 + S5 |
| Persistence bootstrap/cutover | A5 + A7 |
| Persistence portability gate | A5 + A8 |
| `vloop-settings` foundation | A3 + A5 |
| Frontend foundation | GATE-0 |
| FC-1 config | GATE-1 |
| FC-2 skills install | B1 + S3 |
| FC-3/4 execpolicy | B1 |
| FC-9 MCP writes | B1 |
| Kernel adapter | A4 + A5 + B1..B3 + S2 |
| FC-12/13 injected credentials | A7 + B1 |
| FC-11 child env materialization | A7 + B5 |
| Isolation suite | B5 + B7 + B8 |
| FC-5 tools, FC-6/7 skills, FC-8 MCP | GATE-2 (C2 also needs S3) |
| Resources + snapshots | A5 + C1 + C2 + C3 |
| FC-14/15 endpoints and trust | A7 + B8 |
| Orchestration | A4 + A5 + C4 + S4 + GATE-P |
| Learning | D1 + S4 |
| API gateway | A5 + B5 + C4 |
| GUI | A6 + D3 + S2 |
| Workspace/editor/search, Terminal | D3 + D4 |
| Git | E1 |
| Verification | GATE-4 |
| Release | F1..F4 |

## 11. Critical path

```text
A0 → A1 → GATE-0 → A2 → GATE-1
   → A3 → A4/S5 → A5 → A7 → A8 → GATE-P
   → B1 → B2/B3 → B5 → B7/B8 → B6 → GATE-2
   → C1/C2/C3 → C4 → D1 → D3 → D4 → GATE-3
   → D2 → D5 → GATE-4 → F1..F4 → F5 → F6
```

A2 is the pivot. Everything after it assumes the kernel can be owned; nothing
before it costs much to discard.

## 12. Execution batches

- **Batch 0 — Prove the architecture.** A1, GATE-0, A2, GATE-1. Then S2, S3,
  S4 and S5 in parallel.
- **Batch 1 — Foundations and persistence portability.** A3, A4, A5, A6, A7,
  A8 → GATE-P.
- **Batch 2 — Own the state and the config.** B1, B2, B3, B4, B5, B7, B8, B6
  → GATE-2.
- **Batch 3 — Own the resources.** C1, C2, C3, C4, C5.
- **Batch 4 — Orchestrate.** D1, D3, D4 → GATE-3.
- **Batch 5 — IDE and learning.** E1, E2, E3, D2, D5 → GATE-4.
- **Batch 6 — Verify and release.** F1–F6.

Batch 0 exists because its spikes can invalidate downstream design. Running them
first is far cheaper than discovering an incompatible ORM graph, skill invariant
or event surface inside A5, B5 or C4.

## 13. Definition of an executable task

Every leaf task must state: objective; dependencies; files and crates affected;
interfaces touched; acceptance criteria; tests required; explicit non-goals.

Specific to this project:

- **Cite the owning evidence document rather than re-deriving facts.** Use
  `codex-integration.md` for Codex seams, `settings-plane.md` for ambient-env
  migration, and `persistence-plane.md` for ORM/backend semantics. If a needed fact
  is absent or unverified, raise a spike instead of assuming.
- **Fork changes are deliberate and registered.** Any change to `codex-rs` outside
  the FC register requires explicit approval and a register entry. Every change
  must default to current Codex behaviour when no VLOOP snapshot is supplied.
- **Never add VLOOP logic to `codex-core`.** Prefer new crates, visibility
  changes and behaviour gates `[CI §12]`.
- **Never let Codex own state.** Any new code path that starts or forks a thread
  must set `ephemeral = true` and is covered by the isolation suite.
- **Never read Codex state files directly** `[CI §4.2]`.
- **Never expose ORM, SQLx or backend types outside persistence adapters.** Domain,
  runtime, API and learning code use repository and `UnitOfWork` interfaces only
  `[PP §4]`. Backend-specific queries require a documented portable semantic
  equivalent and dual-backend conformance tests.
- **Never hold a persistence transaction across a model request, tool execution or
  user approval wait** `[PP §11]`.
- **Never switch the persistence handle underneath an active run.** Use the
  generation-pinned cutover protocol `[PP §8]`.
- **Never call `std::env::set_var` or `remove_var` in VLOOP code.** Edition 2024
  makes these `unsafe` and the failure is a data race, not a compile error
  `[SP §3]`. Per-agent divergence belongs at the child-process boundary.
- **Never source a credential, endpoint or trust value from ambient env.** Resolve
  it from the settings snapshot; cite `settings-plane.md` for the key class and its
  failure semantics `[SP §5.2]`.
- Honour repo conventions `[CI §12]`: scoped nextest runs, `just fmt`, clippy
  clean, RPITIT over `#[async_trait]`, modules under 500 LoC, sibling test files,
  whole-object assertions, integration-first for agent logic, Bazel lockfile
  updates for dependency changes.

## 14. Definition of done

A node is complete only when:

1. Code is implemented with no placeholder or TODO on the critical path.
2. Specified tests pass.
3. The **full Codex suite passes**, and Codex behaviour is unchanged when no VLOOP
   snapshot is supplied.
4. Acceptance criteria are verified, not assumed.
5. Any fork change is recorded in `vloop/docs/fork-changes.md` with rationale, diff
   scope, upstream note and reversal instructions.
6. Any Codex fact discovered during implementation is added to
   `codex-integration.md` with a `path:line` citation.
7. Bazel lockfile and `BUILD.bazel` are current if dependencies changed.
8. Architecture changes land in `plan.md` and scope changes in `spec.md`, in the
   same change set.
