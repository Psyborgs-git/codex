# VLOOP Agent OS — Architectural Plan

Companion documents:

- `codex-integration.md` — validated Codex evidence base, cited as `[CI §n]`.
  Every `path:line` citation lives there.
- `settings-plane.md` — the environment-variable inventory and the plan to move
  Codex's configuration inputs onto VLOOP-owned dynamic values, cited as
  `[SP §n]`.
- `persistence-plane.md` — injected ORM persistence, SQLite/PostgreSQL adapters,
  transactions, migrations and backend cutover, cited as `[PP §n]`.
- `spec.md` — product and technical requirements.
- `tasks.md` — execution DAG.

## 1. Purpose and posture

VLOOP is a local-first, continually learning, multi-agent orchestration engine.

**VLOOP is the orchestrator, manager and sole source of truth.** It owns every
definition and every durable record: providers, models, agents, tools, skills,
MCP servers, policies, tasks, plans, runs, thread content, artifacts, memory and
evaluations.

**The forked Codex repository is a component library** — a Swiss army knife. We
take the agent loop, tool execution, sandboxing, approval flow, context
injection, MCP clients, skill parsing and PTY handling. We modify the fork where
its seams are closed. We do not adopt its orchestration, its configuration
system, or its persistence.

The distinction that matters: *we bend the tools to the app, not the app to the
tools.* Where a Codex subsystem assumes it owns state, VLOOP disables it and
supplies its own. Where a Codex subsystem is a pure mechanism, VLOOP drives it.

## 2. The ownership inversion

The previous revision of this plan integrated with Codex by spawning
`codex app-server` and speaking JSON-RPC. Under that seam, Codex necessarily
owned providers, models, tools, skills, MCP and thread state, and VLOOP degraded
into a configuration editor. That conclusion was an artefact of the chosen seam,
not a property of Codex.

The decisive discovery: **`ThreadManager::new` is a public, pure
dependency-injection constructor taking all 14 dependencies — including
`thread_store: Arc<dyn ThreadStore>`, `models_manager`, `auth_manager` and the
extension registry** `[CI §1.1]`. Codex's local-first behaviour is simply the
default set of arguments `app-server` happens to pass. Nothing about core
requires them.

| Previous revision | This revision | Basis |
|---|---|---|
| Spawn `codex app-server`, JSON-RPC over stdio | **Embed the kernel in-process**; drive `ThreadManager` / `CodexThread` | `[CI §1.4]` |
| Codex owns providers/models/tools/skills/MCP | **VLOOP owns all definitions**; injected per run | `[CI §3, §6]` |
| Codex owns thread/turn/item persistence | **VLOOP implements `ThreadStore`**; sole store | `[CI §4.1]` |
| VLOOP is a config editor with read-verify-write | **No Codex config writes at all**; config built in memory | `[CI §2.2]` |
| Tools are a read-only derived inventory | **VLOOP-owned tool selection** via FC-5 | `[CI §6.1]` |
| Skills: user-scope enable/disable only | **VLOOP-owned catalog**, any scope | `[CI §6.2]` |
| `experimentalApi = false` forfeits live reconfiguration | **Not a core concept**; `restore_thread_settings` is public | `[CI §1.2, §1.4]` |
| Protocol drift needs pinning + schema diff in CI | **Compile-time type checking**; drift is a build error | §5 |
| Approvals need a JSON-RPC round-trip | **In-process `Op::ExecApproval`** with a richer decision enum | `[CI §1.3]` |
| Separate Cargo workspace to keep Codex pristine | **Join the `codex-rs` workspace**; we modify the fork deliberately | §4 |

Two things did *not* change, because they were validated independently of the
seam: the IDE substrate reuse verdicts `[CI §9]` and the terminal design
`[CI §10]`.

## 3. Architectural north star

```text
┌─────────────────────────────────────────────────────────────────┐
│ Presentation Plane                                              │
│ React + TS + Vite + Tailwind + shadcn/ui + Monaco + xterm.js     │
├─────────────────────────────────────────────────────────────────┤
│ VLOOP API / Event Gateway                                       │
│ HTTP CRUD + WS event stream + WS terminal stream                │
├─────────────────────────────────────────────────────────────────┤
│ VLOOP Learning Plane                                            │
│ Evaluators • Memory promotion • Prompt/policy refinement         │
├─────────────────────────────────────────────────────────────────┤
│ VLOOP Orchestration Plane                                       │
│ Planner • DAG scheduler • Agent assignment • Step attempts       │
├─────────────────────────────────────────────────────────────────┤
│ VLOOP Core (domain)                    Persistence Plane         │
│ Agents • Tasks • Plans • Runs          ORM + injected backend    │
│ Resources • Policies • Memory          SQLite | PostgreSQL       │
├─────────────────────────────────────────────────────────────────┤
│ Runtime Abstraction — AgentRuntime trait                        │
│ CodexKernelRuntime │ future ClaudeRuntime │ MockRuntime          │
├─────────────────────────────────────────────────────────────────┤
│ vloop-kernel-codex   (the only Codex-aware crate)               │
│ RuntimeManifest → Codex deps │ Event/approval translation        │
│                        ↓ injects ↓                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Codex kernel, in-process, VLOOP-injected                  │  │
│  │ ThreadManager ← ThreadStore=VLOOP, ModelsManager=VLOOP,    │  │
│  │   AuthManager=ephemeral, ExtensionRegistry=VLOOP snapshot  │  │
│  │ CodexThread: submit(Op) / next_event()                    │  │
│  │ Reused mechanisms: agent loop • tool exec • sandbox •      │  │
│  │   approvals • context injection • MCP clients • skills     │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│ VLOOP Execution Plane (user-driven, VLOOP-owned)                │
│ Files • Search • Git • PTY   — from Codex utility crates         │
└─────────────────────────────────────────────────────────────────┘
```

Note what is absent: no `app-server`, no JSON-RPC, no Codex config files, no
Codex thread storage, no `$CODEX_HOME` writes.

The two execution planes remain separate, and this is deliberate:

- **Agent-driven execution** — commands and edits the agent requests run inside
  the Codex kernel, where sandbox enforcement and the approval flow already
  exist. Embedding in-process preserves this completely, because the sandbox
  wraps the spawned command, not the agent loop `[CI §11.1]`.
- **User-driven execution** — the developer's own terminal, edits and Git
  operations. VLOOP owns these directly, unsandboxed, because the user is acting.

## 4. Repository layout and the workspace decision

VLOOP crates become **members of the `codex-rs` Cargo workspace.**

```text
codex-rs/
├── Cargo.toml               # VLOOP crates added to [workspace.members]
├── ...                      # existing Codex crates
├── tool-policy/             # NEW, FC-5: runtime-neutral tool selection
└── vloop/
    ├── core/                # domain model, event bus, no Codex deps
    ├── runtime/             # AgentRuntime trait, capabilities
    ├── kernel-codex/        # ONLY Codex-aware crate
    ├── orchestrator/        # task → plan DAG → step attempts
    ├── learning/            # evaluators, memory promotion
    ├── settings/            # remote settings, resolver, env materializer
    ├── persistence/         # backend-neutral repository + UnitOfWork traits
    ├── persistence-db/      # ORM/data-mapper adapter, SQLite/Postgres
    ├── store-thread/        # Codex ThreadStore bridge over Persistence
    ├── workspace/           # files, search, git, watcher
    ├── terminal/            # PTY session manager
    ├── api/                 # HTTP + WS handlers, DTOs
    └── server/              # binary, wiring, logging

vloop/                       # non-Rust assets
├── frontend/
├── docs/
├── plan/agent-os/
└── scripts/
```

### 4.1 Why join the workspace

The previous revision chose a separate workspace so that "existing Codex build
and tests remain green" would be structurally true. That rationale is void: we
now modify the fork deliberately `[CI §7]`, so the gate must change shape
regardless (§15).

Three reasons to join:

1. **It eliminates the largest unproven risk.** Cross-workspace path-dependency
   on `codex-core` — a very large crate with `version.workspace = true`
   inheritance and a wide feature graph — is unverified `[CI §13.3]`. Depending
   on it is now central rather than peripheral, so removing that risk is worth
   more than avoiding Bazel work.
2. **Fork changes and their consumers stay in one build graph.** Ten fork changes
   plus a new `tool-policy` crate would otherwise straddle two workspaces with
   two lockfiles.
3. **Protocol drift becomes a compile error.** Linking Codex types directly means
   an upstream change to `Op`, `EventMsg` or `ThreadStore` fails the build
   instead of failing at runtime. This deletes the entire pinning-and-schema-diff
   apparatus the previous revision needed.

The accepted costs, stated plainly:

- Every VLOOP dependency change requires `just bazel-lock-update`, and each new
  crate needs a `BUILD.bazel`. CI verifies lockfile drift `[CI §12]`.
- VLOOP joins the Bazel graph and the `just test` matrix, so VLOOP test time is
  Codex test time.
- Upstream merges can conflict with VLOOP crates, not just with fork changes.

Mitigation: VLOOP crates are additive and self-contained; fork changes are
minimal, individually reversible and tracked in a register (§15). GATE-0 proves
the workspace and Bazel wiring before any feature work.

## 5. Core architectural laws

1. **VLOOP is the sole source of truth.** No definition and no durable record
   lives in Codex config, `$CODEX_HOME`, or Codex thread storage.
2. **Codex holds no durable state.** Every persistence path is either injected
   into VLOOP's store or disabled, and this is a tested invariant, not a
   convention (`spec.md` FR-23).
3. **Only `vloop-kernel-codex` may name a Codex type.** Enforced by CI, not
   convention.
4. **`vloop-core` compiles with zero Codex dependencies.** It is the proof that
   the domain model is runtime-neutral.
5. **Every run executes against an immutable `ResourceSnapshot`.** Mid-run edits
   to agents, tools, skills, MCP or policy never change a running attempt §7.
6. **Runtimes are swappable behind `AgentRuntime`.** Codex is the first
   implementation, never an assumption.
7. **Fork changes are minimal, documented and upstream-shaped.** Prefer new
   crates and visibility changes over logic in `codex-core` §15.
8. **Policy translation may only narrow.** A VLOOP policy can never produce a
   more permissive Codex policy than the user selected §10.
9. **Sandbox and approval enforcement are never bypassed.** VLOOP owns the
   *decision*; Codex owns the *enforcement*.
10. **User-driven and agent-driven execution stay separate paths** §3.
11. **UI state is not domain state.**
12. **Learning is explicit and reviewable.** No implicit self-modification of
    agents or policies without a recorded, inspectable promotion step §9.
13. **Configuration is resolved, never ambient.** Codex's inputs arrive as
    constructor arguments and child-process environments derived from a pinned
    settings snapshot. VLOOP never mutates its own process environment, and no
    credential, endpoint or trust value is read from ambient env `[SP §5]`.
14. **Persistence is injected and generation-pinned.** Domain code sees repository
    and `UnitOfWork` interfaces, never ORM/SQLx/backend types. No active attempt
    crosses persistence generations in v0.1; backend cutover is quiescent and
    operationally controlled `[PP §4, §8]`.

## 6. The kernel adapter

`vloop-kernel-codex` is the only crate with Codex knowledge. Structure, with
modules kept under 500 LoC per repo guidance `[CI §12]`:

```text
vloop/kernel-codex/
├── manifest.rs      # RuntimeManifest -> Codex dependency set
├── config.rs        # in-memory Config assembly (FC-1)
├── layers.rs        # ConfigLayerStack construction (the §2.4 trap)
├── models.rs        # StaticModelsManager from VLOOP catalog
├── auth.rs          # ephemeral AuthManager + ExternalAuth bridge
├── extensions/
│   ├── tools.rs     # ToolCatalogInput + ToolContributor (FC-5)
│   ├── skills.rs    # SkillCatalogProvider (FC-6)
│   └── mcp.rs       # ReplaceCatalog contribution (FC-8)
├── policy.rs        # VLOOP Policy -> Permissions + execpolicy (FC-3/4)
├── settings.rs      # SettingsSnapshot -> constructor args (FC-12..15)
├── env.rs           # EnvMaterializer: child env from snapshot (FC-11)
├── kernel.rs        # ThreadManager lifecycle
├── session.rs       # CodexThread driving: submit / next_event
├── translate/
│   ├── event.rs     # EventMsg -> VloopEvent
│   └── approval.rs  # approval request -> VLOOP; decision -> ReviewDecision
├── isolation.rs     # assertions that no Codex state is written
└── capabilities.rs  # capability descriptor
```

### 6.1 The runtime manifest

The manifest is the mechanism by which VLOOP bends the kernel. It is an immutable
value, resolved from VLOOP's store, that fully determines a run's behaviour:

```rust
/// Everything the kernel needs for one run. Resolved from VLOOP's store and
/// frozen before execution begins; see `ResourceSnapshot` in §7.
pub struct RuntimeManifest {
    pub snapshot_id: ResourceSnapshotId,
    pub persistence_generation: PersistenceGeneration,
    pub settings_version: SettingsVersion,
    pub workspace: WorkspaceRef,
    pub provider: ProviderDef,          // VLOOP-owned
    pub model: ModelDef,                // VLOOP-owned catalog entry
    pub reasoning: ReasoningSettings,
    pub instructions: Instructions,     // base + developer
    pub tools: ToolSelection,           // allow/deny/exposure  (FC-5)
    pub skills: SkillCatalogSnapshot,   // pre-parsed, in memory (FC-6)
    pub mcp: McpServerSnapshot,         // definitions           (FC-8)
    pub policy: PolicySnapshot,         // approval + sandbox + execpolicy
    pub context: ContextPack,           // VLOOP-assembled memory and context
    pub credentials: CredentialRef,     // resolved at request time, never stored
}
```

Mapping to Codex dependencies `[CI §1.5, §2, §3, §6]`:

| Manifest field | Codex target |
|---|---|
| `provider` | `ConfigToml.model_providers` + `ConfigOverrides.model_provider` |
| `model` | `Config.model_catalog` ⇒ `StaticModelsManager` |
| `reasoning` | `ConfigToml.model_reasoning_*` |
| `instructions` | `ConfigOverrides.{base_instructions, developer_instructions}` |
| `tools` | `ToolCatalogInput` via `ExtensionDataInit` (FC-5) |
| `skills` | `SkillCatalogProvider` registered as `SkillSourceKind::Custom` (FC-6/7) |
| `mcp` | `McpServerContribution::ReplaceCatalog` (FC-8) |
| `policy` | `Permissions` + `ExecPolicyManager` (FC-3/4) + `environment_ready` |
| `context` | `CodexThread::inject_response_items` |
| `credentials` | `AuthManager` ephemeral + `ExternalAuth` |
| persistence | `ThreadStore` bridge over injected `Arc<dyn Persistence>`; `ephemeral = true`; `state_db: None` |

### 6.2 Isolation invariants

Because `ephemeral` is per-thread and caller-supplied, any path that starts or
forks a thread without setting it silently re-enables Codex's full local
persistence stack `[CI §4.4]`. This is the highest-probability regression in the
design.

`isolation.rs` therefore asserts, and `spec.md` FR-23 tests:

- `ephemeral = true` on every thread start and fork
- `state_db: None`, `agent_graph_store: None`
- `ThreadStore` is the VLOOP implementation
- `HistoryPersistence::None`
- bundled skills disabled **through a `SessionFlags` config layer**, not merely
  through `ConfigToml` — the layer-stack trap `[CI §2.4]`
- no write occurs anywhere under `$CODEX_HOME` during a full run
- `installation_id` is a VLOOP value, so no id file is written

The last invariant is verified by running a complete task with `$CODEX_HOME`
pointed at a directory made read-only, which must succeed.

### 6.3 The runtime trait

Per repo guidance: RPITIT with explicit `Send` bounds, no `#[async_trait]`, no
ambiguous bool or `Option` parameters `[CI §12]`.

```rust
/// A pluggable execution engine for one agent attempt.
///
/// Implementations translate a runtime-neutral `RuntimeManifest` into a concrete
/// backend and normalize backend output into `VloopEvent`. Implementations must
/// declare support via `capabilities` rather than assuming a feature set;
/// callers must consult it before issuing an operation.
pub trait AgentRuntime: Send + Sync {
    fn id(&self) -> RuntimeId;
    fn capabilities(&self) -> &RuntimeCapabilities;
    fn status(&self) -> RuntimeStatus;

    fn start(
        &self,
        manifest: Arc<RuntimeManifest>,
    ) -> impl Future<Output = Result<RuntimeHandle, RuntimeError>> + Send;

    fn send(
        &self,
        handle: &RuntimeHandle,
        input: RuntimeInput,
    ) -> impl Future<Output = Result<TurnId, RuntimeError>> + Send;

    fn interrupt(
        &self,
        handle: &RuntimeHandle,
        turn: &TurnId,
    ) -> impl Future<Output = Result<(), RuntimeError>> + Send;

    fn resolve_approval(
        &self,
        handle: &RuntimeHandle,
        approval: ApprovalId,
        decision: ApprovalDecision,
    ) -> impl Future<Output = Result<(), RuntimeError>> + Send;

    fn stop(
        &self,
        handle: &RuntimeHandle,
        reason: StopReason,
    ) -> impl Future<Output = Result<(), RuntimeError>> + Send;

    fn subscribe(&self, handle: &RuntimeHandle) -> EventStream;
}
```

`ApprovalDecision` maps onto Codex's `ReviewDecision` `[CI §1.3]`, which is
richer than the app-server surface and must be preserved in full:

| VLOOP | Codex | Meaning |
|---|---|---|
| `Approve` | `Approved` | run it |
| `ApproveForSession` | `ApprovedForSession` | and stop asking this session |
| `ApproveWithRule(..)` | `ApprovedExecpolicyAmendment{..}` | and remember the rule |
| `Deny { reason }` | `Denied { rejection }` | refuse; **agent continues and may adapt** |
| `Abort` | `Abort` | refuse and halt until the next user turn |
| `TimedOut` | `TimedOut` | native timeout, no synthesized cancel |

`Deny { reason }` carrying a string the agent can reason about, and `TimedOut`
being first-class, are both direct benefits of the in-process seam.

### 6.4 Event translation

```text
CodexThread::next_event() -> Event { EventMsg }
        ↓ translate/event.rs   (one typed hop, no codec)
   VloopEvent  (correlated to run_id / attempt_id)
        ↓
   VLOOP Event Bus (bounded broadcast)
        ├── Store        (append run_events)
        ├── Orchestrator (attempt state machine)
        ├── Learning     (signal extraction)
        ├── API/WebSocket
        └── Logs
```

Rules carried over from the previous revision and still valid:

- Output reaching the UI is byte-bounded per item; the full stream goes to the run
  log. `DEFAULT_OUTPUT_BYTES_CAP` (1 MiB) is a reasonable starting bound
  `[CI §10]`.
- Timestamps normalize to RFC 3339 UTC once, at the boundary.
- Bus lag emits an explicit `Lagged` event rather than dropping silently.

Because VLOOP also implements `ThreadStore`, it sees turn items through *two*
paths: the event stream and `append_items`. These must not double-record.
`run_events` is the observability log; `thread_items` is the conversation of
record. `spec.md` FR-24 makes their consistency a tested property.

## 7. Resource snapshots and immutability

A run must not change behaviour because a user edited an agent mid-flight. Codex
rebuilds tools and skills per sampling step with no version identity
`[CI §6.1, §6.2]`, so without intervention a mid-run edit silently takes effect.

VLOOP resolves every definition into an immutable, content-addressed snapshot
before a run starts:

```text
Agent (mutable, versioned)  ──resolve──►  ResourceSnapshot (immutable)
Tools / Skills / MCP / Policy               ├── snapshot_id
                                            ├── agent_version_id
                                            └── frozen definitions
                                                      │
                                        RuntimeManifest for every attempt
```

Snapshots are retained for the lifetime of the runs referencing them, which makes
a run reproducible and gives evaluation a stable subject.

Pinning reuses an existing Codex precedent rather than inventing one:
`PreparedMcpCall` stores a `catalog_revision` and rejects execution when the live
revision has moved `[CI §6.3]`. VLOOP stamps `ResourceSnapshotId` into the same
position, so a tool call from a stale snapshot is rejected rather than silently
honoured.

## 8. Orchestration

VLOOP owns orchestration outright. Codex's own multi-agent spawning is not used:
it assumes Codex-owned persistence and an `AgentGraphStore` `[CI §8]`, and
adopting it would reintroduce exactly the ownership we removed. VLOOP instead
treats the **single-agent loop as its execution primitive** and composes above it.

### 8.1 Model

```text
Task
 └── Plan (versioned, DAG of steps)
      └── PlanStep  (agent_id, depends_on[], inputs, success criteria)
           └── StepAttempt  (one execution; retries create new attempts)
                └── RuntimeHandle -> CodexThread (1:1)
```

Five entities, with a deliberate distinction: a **Run** is one execution of a
Plan; a **StepAttempt** is one execution of a Step within it. Retry creates a new
attempt, never mutating history. This is what makes evaluation and learning
possible, and it is why v0.1 ships the DAG schema even though it executes
sequentially.

### 8.2 Scheduling

v0.1 executes a topological order sequentially. The scheduler interface is
designed for parallelism from the start:

```rust
/// Selects steps whose dependencies are satisfied and whose resource claims do
/// not conflict. v0.1 returns at most one step; the contract permits many.
pub trait Scheduler: Send + Sync {
    fn ready_steps(&self, run: &RunGraph, limits: &ConcurrencyLimits)
        -> Vec<StepId>;
}
```

The hard problem parallelism introduces is **shared workspace conflict**: two
agents editing the same files. v0.1 does not solve it; it *represents* it. Each
step declares a `WorkspaceClaim` (read or write, path-scoped), and the sequential
scheduler asserts claims are non-overlapping. When parallel execution lands, the
same claims gate concurrency, and the escalation path is per-attempt git
worktrees. Deferring the mechanism is fine; deferring the data model would force
a schema migration later.

### 8.3 Agent assignment

An agent is assigned to a step either explicitly or by capability match against
required tools, skills and policy. Assignment resolves to an `agent_version_id`,
which enters the snapshot, so a plan's behaviour is reproducible even after the
agent is edited.

## 9. Continual learning

This is the product's reason for existing, so the design is explicit rather than
emergent. Codex's memory subsystem is left disabled — it is gated on `ephemeral`,
a state DB and `Feature::MemoryTool`, and the startup task is only invoked from
app-server `[CI §8]`, so driving `ThreadManager` directly starts no memory task
at all. VLOOP owns memory.

### 9.1 The loop

```text
StepAttempt completes
   ↓
Signal extraction      outcome, approvals, tool errors, retries, diffs,
   ↓                   test results, tokens, wall time
Evaluation             deterministic checks first; model-graded second
   ↓
Candidate insights     scoped: agent | workspace | task-type | global
   ↓
REVIEW GATE            auto-promote only above a confidence threshold and
   ↓                   only for reversible scopes; otherwise queue for a human
Promotion              memory entry, prompt refinement, policy adjustment,
   ↓                   or skill suggestion
Next run consumes via ContextPack
```

### 9.2 Design constraints

- **Deterministic evaluators run before model-graded ones.** Did tests pass, did
  the patch apply, was the file created, did the command exit zero. These are
  cheap, trustworthy and sufficient for the golden path.
- **Every promotion is recorded with its provenance** — which attempt, which
  evaluator, which confidence. A memory entry whose origin cannot be inspected
  is a liability.
- **Promotion is reversible.** Each promoted insight can be disabled, and the
  affected runs identified.
- **Nothing self-modifies silently.** Law 12. Prompt and policy changes require
  explicit promotion; the review gate defaults to human-in-the-loop for anything
  affecting policy.
- **Memory is scoped and budgeted.** `ContextPack` assembly has a token budget;
  memory competes with files and history rather than growing without bound.

### 9.3 v0.1 scope

Signal extraction, deterministic evaluators, the attempt/evaluation schema, and
manual promotion with full provenance. Automatic promotion and model-graded
evaluation are v0.2. This keeps the schema honest while deferring the parts that
need real run data to tune.

## 10. Policy

VLOOP policy is runtime-neutral and translated by `policy.rs`. Translation is
one-directional and **narrowing only**, asserted by unit tests over the full
matrix.

| VLOOP policy | Codex target `[CI §6.4, §11.1]` |
|---|---|
| `filesystem: workspace-read` | `SandboxMode::ReadOnly` |
| `filesystem: workspace-write` | `SandboxPolicy::WorkspaceWrite { writable_roots }` |
| `filesystem: unrestricted` | `DangerFullAccess` — explicit per-run opt-in |
| `network: disabled / allowed` | `network_access: false / true` |
| `approval: always` | `AskForApproval::UnlessTrusted` |
| `approval: on-risk` | `AskForApproval::OnRequest` |
| `approval: never` | `AskForApproval::Never` |
| `command_rules[]` | in-memory `execpolicy::Policy` (FC-3/4) |
| per-run restrictions | `EnvironmentConfig.exec_policy` via `environment_ready` |

Two write-backs must be suppressed for VLOOP to be the sole policy owner:
execpolicy amendments writing `default.rules` (FC-3) and MCP approvals writing
`config.toml` (FC-9) `[CI §4.3]`. Until both land, the claim of sole ownership is
false — which is why they are Tier A.

Approval decisions that carry amendments (`ApproveWithRule`) are applied to
VLOOP's policy store and re-enter the next snapshot, rather than mutating Codex
state.

## 11. Persistence

VLOOP's persistence plane is the sole source of truth. It is **injected at
runtime** and accessed through backend-neutral repository and `UnitOfWork`
interfaces. Full design: `persistence-plane.md`, cited `[PP §n]`.

Two supported deployment profiles:

| Profile | Authoritative backend | Use |
|---|---|---|
| `embedded` | SQLite through the ORM adapter | local, single-user, offline-first |
| `managed` | PostgreSQL through the ORM adapter | central management, multiple workers/devices |

SQLite remains the default local backend; it is no longer hard-coded into domain
or repository code. Switching profiles is a **quiescent controlled cutover** with
a new `PersistenceGeneration`, never a live connection-string mutation under
active transactions `[PP §8]`.

```text
vloop-core (domain types only)
        ↑
vloop-persistence
  Persistence • UnitOfWork • repository traits • capabilities
        ↑
vloop-persistence-db
  persistence models • migrations • SQLite/PostgreSQL adapters
  SeaORM planned; S5 may select the fallback
        ↑
vloop-store-thread
  Codex ThreadStore bridge, no SQL or ORM entities exposed
```

SeaORM is the planned adapter, subject to an early compatibility spike against
the workspace's SQLx 0.9 dependency and Bazel graph. If that combination is not
viable, the fallback is Diesel Async or a SQLx data-mapper implementation behind
the **same** persistence interfaces `[PP §3]`. The architecture does not depend
on an ORM brand.

| Domain | Tables |
|---|---|
| Workspaces | `workspaces` |
| Resources | `providers`, `models`, `tools`, `skills`, `mcp_servers`, `policies` |
| Agents | `agents`, `agent_versions` |
| Snapshots | `resource_snapshots` |
| Orchestration | `tasks`, `plans`, `plan_steps`, `runs`, `step_attempts` |
| Threads | `threads`, `thread_items` — the `ThreadStore` implementation |
| Observability | `run_events`, `approval_records`, `outbox_events` |
| Outputs | `artifacts` |
| Learning | `evaluations`, `insights`, `memory_entries`, `promotions` |
| Settings | `settings_scopes`, `settings_values`, `settings_cache`, `credential_refs` |
| System | `settings`, `schema_migrations` |

`threads` / `thread_items` are VLOOP's own because VLOOP implements
`ThreadStore` `[CI §4.1]`. The bridge depends on `Arc<dyn Persistence>`, not on a
concrete connection or ORM entity, so Codex sees the same store contract under
SQLite and PostgreSQL.

Two constraints inherited from `ThreadStore` remain `[CI §4.2]`:

- `as_any` downcasts to `LocalThreadStore` gate Codex's `state_db`; the VLOOP
  bridge intentionally forfeits Codex memories and queue persistence because
  VLOOP owns those domains.
- paginated history is Local-only. v0.1 declares Legacy history mode; adding
  paginated methods later does not change the persistence boundary.

Atomic domain transitions use `UnitOfWork`: run start, ordered thread append,
attempt transition, approval resolution and memory promotion. Domain events are
written to a transactional outbox and published after commit; delivery is
at-least-once with `event_id` deduplication, not falsely claimed exactly-once
`[PP §6]`.

ORM portability is constrained deliberately: application UUIDs, UTC timestamps,
validated string enums, relational join tables, portable constraints, and no
correctness dependency on backend-native enums, extensions or triggers
`[PP §5]`. Backend-specific SQL is permitted only behind an adapter with a
portable semantic equivalent and conformance tests.

For multi-worker scheduling, mutable aggregates use optimistic versions, and step
claims carry expiry plus a fencing token. PostgreSQL may optimize claiming with
`SKIP LOCKED`; SQLite uses portable compare-and-swap. Both pass the same scheduler
suite `[PP §7]`.

The database connection itself is injected through `DatabaseBootstrap` and
`PersistenceFactory`; no `DATABASE_URL` environment variable. The database
credential is a short-lived lease from the settings/secret plane, resolving the
bootstrap paradox explicitly `[PP §1.6, §4.2]`.

## 12. Swappability

Law 6 is only credible if it is exercised. Three mechanisms:

1. **`MockRuntime`** implements `AgentRuntime` with scripted events and passes
   the same attempt-lifecycle test suite as the kernel runtime. It is the
   deterministic substrate for orchestrator and E2E tests.
2. **`vloop-core` has zero Codex dependencies**, verified in CI. If the domain
   model needed a Codex type, the abstraction would be leaking.
3. **A CI grep** asserts no Codex type name appears outside `vloop-kernel-codex`.

Within the kernel, the same principle applies to components: tools, skills and
MCP are injected per run, so an individual mechanism can be replaced without
touching the loop. That is what "keep runtime tools swappable instead of tightly
coupled" means concretely.

## 13. Security

- Localhost-only binding by default.
- **Bearer-token authentication on every HTTP route and both WebSocket
  endpoints**, generated at startup. This is mandatory, not hardening: the API
  spawns PTY sessions, and loopback binding alone does not stop a local page from
  connecting. Codex's own execution socket pairs loopback with `Origin` rejection
  and still does not treat that as authentication `[CI §10]`.
- WebSocket handshakes validate `Origin` against an allowlist.
- Codex sandbox and approval enforcement stay active; embedding preserves them
  because the sandbox wraps spawned commands `[CI §11.1]`.
- Policy translation may only narrow §10.
- Credentials are injected via `ExternalAuth` with an ephemeral store `[CI §5]`.
  Two leaks must be closed in VLOOP's wiring: `CODEX_ACCESS_TOKEN` in the process
  environment can outrank the resolver, and `env_key` / `env_http_headers` read
  the process environment — so both are left unset.
- `$CODEX_HOME/auth.json` is never read, logged or copied.
- `redact_secrets` is applied before any command output is logged or displayed.
- **In-process embedding means the agent shares VLOOP's address space.** VLOOP
  must not hold decrypted secrets in long-lived process memory beyond the request
  that needs them.

## 14. Failure modes

| Failure | Behaviour |
|---|---|
| **Panic inside the kernel** | This is the real cost of in-process embedding. Turn driving is wrapped so a panic fails the attempt rather than the process; the runtime is marked degraded and the supervisor decides on retry. A panic that poisons shared state escalates to process restart with attempts reconciled on startup. |
| Approval unanswered | Attempt enters `waiting_approval` with a timeout; on expiry VLOOP sends the native `TimedOut` decision `[CI §1.3]` |
| Transient provider error | Surfaced as a warning; the attempt stays running |
| Step fails | Retry creates a **new attempt**, never mutating the old one §8.1 |
| VLOOP restarts with attempts `running` | Startup recovery marks them `interrupted` with a recovery note; no attempt is left falsely running |
| Workspace claim conflict | Scheduler refuses to co-schedule; sequential v0.1 asserts non-overlap §8.2 |
| Snapshot drift mid-run | Stale-snapshot tool calls are rejected §7 |
| Event bus lag | Explicit `Lagged` event; client refetches |
| Terminal process dies | Session marked exited with code; replay buffer retained until the tab closes |
| Codex state written unexpectedly | Isolation test fails the build §6.2 |
| Remote settings unreachable | Behavioural and endpoint keys fall back to last-known-good cache; credential, trust and policy keys **fail closed** `[SP §5.2]` |
| Remote setting changes mid-run | No effect: settings are pinned into the run's snapshot `[SP §5.4]` |
| Ambient env supplies a credential | Hermetic-env test fails the build `[SP §9]` |
| Managed database unavailable before run start | New scheduling pauses; health reports degraded; no unpinned run starts `[PP §1.1]` |
| Managed database fails during an attempt | The attempt may continue only until its next required durable transition; it pauses/fails rather than executing unrecorded work. No transaction is held across model/tool execution `[PP §11]` |
| Connection pool exhausted | Apply bounded backpressure and surface a retryable persistence error; never open unbounded connections |
| Schema version incompatible | Refuse writes and scheduling; managed mode never auto-applies a destructive migration `[PP §5.1]` |
| Backend cutover verification fails | Candidate is discarded; old generation remains authoritative and read/write service resumes there `[PP §8]` |

## 15. Fork strategy

Ten fork changes are registered in `[CI §7]`, tiered: Tier 0 needs none, Tier A
(FC-1..4, FC-9) buys sole ownership of state and policy, Tier B (FC-5..8) buys
sole ownership of resources, Tier C (FC-10) is optional. Six further changes
(FC-11..16) move configuration inputs off ambient environment variables and are
registered in `[SP §7]`.

Hygiene rules, so the fork stays maintainable:

1. **Prefer new crates over changes to `codex-core`**, per its own guidance
   `[CI §12]`. FC-5 introduces `tool-policy` rather than putting selection logic
   in core.
2. **Prefer visibility changes and behaviour gates over new logic.** FC-1 is one
   line. FC-2, FC-7, FC-9 are gates on an injected value.
3. **Every fork change is individually reversible** and recorded in
   `vloop/docs/fork-changes.md` with rationale, diff scope and an upstream note.
4. **Behaviour gates default to current behaviour.** With no VLOOP snapshot
   supplied, Codex behaves exactly as before — which is what keeps its own tests
   passing.
5. **Do not touch** `CODEX_SANDBOX*` handling, or publicize `CoreToolRuntime`,
   `McpConnectionSet`, `McpBinding::new` or `PreparedMcpCall::new` `[CI §7]`.
6. **The non-regression gate becomes**: the full Codex test suite passes after
   our changes, and no Codex-default behaviour changes when no snapshot is
   supplied.

## 16. Evolution path

- **v0.1** — Kernel ownership proven: VLOOP-owned config, resources, policy and
  thread storage; injected ORM persistence with SQLite/PostgreSQL conformance;
  single-agent sequential execution; DAG and attempt schema in place;
  deterministic evaluation with manual promotion; IDE surfaces.
- **v0.2** — Sequential multi-agent DAG; automatic memory promotion;
  model-graded evaluation.
- **v0.3** — Parallel execution with workspace claims enforced via git worktrees;
  concurrency limits.
- **v0.4** — Second runtime behind `AgentRuntime`; runtime routing by capability
  and cost.
- **v0.5** — Full context engine: retrieval, budgeting, cross-run knowledge.
- **v1.0** — Durable long-running orchestration, policy learning, artifact
  lineage, extensible GUI.

## 17. Architectural acceptance criteria

1. `vloop-core` compiles with zero Codex dependencies, verified in CI.
2. A CI grep confirms no Codex type name outside `vloop-kernel-codex`.
3. **A full task executes with `$CODEX_HOME` read-only**, proving Codex holds no
   durable state §6.2.
4. No Codex config file is read or written during any VLOOP operation.
5. Thread content is retrievable from VLOOP's store with the kernel stopped.
6. `MockRuntime` passes the same attempt-lifecycle suite as the kernel runtime.
7. A run is reproducible from its `ResourceSnapshot` after its agent is edited.
8. Retry produces a new attempt; prior attempt records are immutable.
9. Policy translation is provably narrowing, over the full matrix.
10. The Codex test suite passes after all fork changes, and Codex behaviour is
    unchanged when no VLOOP snapshot is supplied §15.
11. Every fork change is documented with rationale and is individually
    reversible.
12. A promoted insight is traceable to the attempt and evaluator that produced
    it, and can be disabled.
13. **A full run executes with the process environment reduced to a host-local
    allowlist**, proving no credential, endpoint or trust value is ambient
    `[SP §9]`.
14. Two agents run concurrently against one provider with different credentials,
    in one process, without observing each other's values `[SP §9]`.
15. No VLOOP crate calls `std::env::set_var` or `remove_var`; enforced by CI,
    because edition 2024 makes these `unsafe` and the failure is a race rather
    than a compile error `[SP §3]`.
16. A mid-run change to a remote setting does not affect the running attempt, and
    a replayed run resolves the original `settings_version` `[SP §5.4]`.
17. The same repository, migration, scheduler-claim and `ThreadStore` conformance
    suite passes against SQLite and PostgreSQL `[PP §10]`.
18. No ORM, SQLx, SQLite or PostgreSQL type crosses into `vloop-core`, runtime or
    API crates `[PP §4]`.
19. A cutover rehearsal rejects switching while attempts are active, succeeds once
    the runtime is quiescent, and can roll back to the prior backend after
    candidate verification fails `[PP §8]`.
20. No database transaction spans a model request or tool execution `[PP §11]`.
