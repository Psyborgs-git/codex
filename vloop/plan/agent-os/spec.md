# VLOOP Agent OS v0.1 — Product & Technical Specification

Companion documents: `codex-integration.md` (validated Codex evidence base,
cited as `[CI §n]`), `settings-plane.md` (environment-variable migration, cited
as `[SP §n]`), `persistence-plane.md` (injected ORM persistence, cited as
`[PP §n]`), `plan.md` (architecture), `tasks.md` (execution DAG).

## 1. Release goal

VLOOP v0.1 is the first working local Agent OS: a graphical development
environment in which VLOOP is the **sole source of truth and the orchestrator**,
and the forked Codex repository is a component library supplying the agent loop,
tool execution, sandboxing, approvals, context injection, MCP clients and PTY
handling.

v0.1 executes a single agent per step, sequentially — but the DAG schema, the
attempt model, the resource-snapshot mechanism and the evaluation pipeline all
ship, because retrofitting them later would force schema migration and would make
the learning loop unimplementable.

The release proves one claim above all others: **Codex holds no durable state and
no configuration authority.** A full task must execute with `$CODEX_HOME`
read-only.

## 2. Scope changes from the previous revision

The previous revision descoped four requirements because the app-server seam made
Codex the owner of those subsystems. Embedding the kernel and modifying the fork
reverses all four. Recording this explicitly, because it changes what "done"
means.

| Requirement | Previous scope | This revision | Basis |
|---|---|---|---|
| FR-08 Providers | constrained config editing | **VLOOP-owned CRUD**, injected per run | `[CI §3.1]` |
| FR-09 Models | read-only catalog | **VLOOP-owned catalog CRUD** ⇒ `StaticModelsManager` | `[CI §3.2]` |
| FR-12 Tools | read-only derived inventory | **VLOOP-owned selection** per agent | `[CI §6.1]` + FC-5 |
| FR-13 Skills | user-scope enable/disable only | **VLOOP-owned catalog**, any scope | `[CI §6.2]` + FC-6/7 |
| FR-14 MCP | full-table rewrite + global reload | **VLOOP-owned definitions**, per-run | `[CI §6.3]` + FC-8 |
| FR-18 Threads | Codex-owned, re-read on demand | **VLOOP-owned** via `ThreadStore` | `[CI §4.1]` |

One constraint survives the inversion and is genuinely upstream:
`WireApi` has a single variant, `Responses` `[CI §3.1]`, so the provider form
offers no wire-protocol selector.

Six new requirements exist only because of the new model: FR-23 kernel
isolation, FR-24 thread-store consistency, FR-25 resource snapshots, FR-26 the
learning loop, FR-27 the settings plane, and FR-28 injected ORM persistence.

## 3. User stories

### Workspace
- I can open a local project and see it detected as a Git repository.
- I can browse, open, create, rename, delete and edit files.
- I can quick-open by fuzzy filename and search file contents.
- I can see Git status and diffs, stage, unstage and commit — and my commit hooks
  run.
- I can use an integrated terminal with multiple sessions.

### Resource management
- I can define model providers and models, and they are mine: nothing is read
  from or written to Codex configuration.
- I can define which tools an agent may use, and see why a tool is unavailable.
- I can define skills, including ones that exist only in VLOOP.
- I can define MCP servers per agent, and see their connection status.
- I can define policies once and reuse them across agents.

### Agents
- I can create, edit, duplicate, enable/disable and delete agents.
- I can see an agent's version history, and know which version a past run used.
- Editing an agent never changes the behaviour of a run already in flight.

### Orchestration
- I can create a task and let VLOOP plan it, or define plan steps myself.
- I can watch a run progress through its steps.
- I can see each attempt separately, including retries, with prior attempts
  intact.
- I can interrupt a run.
- I can approve or deny a requested operation — and when I deny, I can choose
  between letting the agent try something else and halting it.

### Learning
- I can see how an attempt was evaluated and on what evidence.
- I can review candidate insights and decide whether to promote them.
- I can trace any promoted insight back to the attempt that produced it, and
  disable it.

### Persistence
- Everything survives restart: workspaces, resources, agents, tasks, plans, runs,
  attempts, conversation history, evaluations and memory.
- Attempts interrupted by a crash are reconciled on startup, never left running.

## 4. Non-goals

- Cloud sync, remote or distributed workers.
- **Parallel step execution.** The DAG and workspace claims ship; concurrent
  execution is v0.3 `plan.md §8.2`.
- **Automatic insight promotion and model-graded evaluation.** Deterministic
  evaluation with manual promotion ships; automation is v0.2 `plan.md §9.3`.
- **A second runtime implementation.** The trait and `MockRuntime` ship; a real
  second runtime is v0.4.
- Debugger, LSP, marketplace, collaboration, billing, mobile app.
- **No use of Codex's own multi-agent orchestration or memory subsystems.** Both
  assume Codex-owned persistence `[CI §8]`; VLOOP owns orchestration and memory.
- **No use of `codex app-server` or JSON-RPC.** The kernel is embedded
  `[CI §1.4]`.
- **No SSE transport for MCP.** Only stdio and streamable HTTP exist
  `[CI §6.3]`.
- **No OAuth-authenticated MCP servers.** No in-memory token store exists
  `[CI §11.2]`; use `bearer_token_env_var`.
- **No re-implementation of sandboxing, tool execution or the agent loop.** These
  are the point of reuse.

## 5. Functional requirements

### FR-01: Application startup
`./vloop/scripts/dev.sh` starts the backend, the frontend and the embedded kernel.

Acceptance:
- frontend served on localhost; `GET /api/health` reports backend and kernel
  status
- the kernel is constructed with VLOOP-injected dependencies: VLOOP `ThreadStore`,
  `StaticModelsManager` from VLOOP's catalog, ephemeral `AuthManager`,
  `agent_graph_store: None`, `state_db: None`, a VLOOP `installation_id`
  `[CI §1.5]`
- **startup performs no read of and no write to any Codex configuration file**
- an API bearer token is generated and passed to the frontend §11
- shutdown drains kernel threads, then terminates all PTY sessions

### FR-02: Workspace management
Create, open, list, close, remove metadata.

Opening a workspace triggers Git detection, watcher registration and resolution
of the workspace's default resource set. It does **not** trigger Codex skill
discovery or MCP config loading — those come from VLOOP's store.

### FR-03: File manager
List, read, write, create, rename, delete.

All paths are validated against the workspace root; requests resolving outside it
are rejected `[CI §9.1]`. Writes are atomic and symlink-safe. Rename is VLOOP
code because no upstream equivalent exists `[CI §9.2]`.

Watcher events deliver paths without kinds, so the backend re-stats affected paths
to distinguish modification from deletion `[CI §9.1]`.

### FR-04: Editor
Monaco with tabs, dirty state, save, syntax highlighting and diff view.

### FR-05: Search
Quick open via incremental fuzzy filename matching `[CI §9.1]`, and workspace
full-text content search — new code, since upstream matches paths only
`[CI §9.2]`. Both bounded and cancellable.

### FR-06: Terminal
Interactive PTY sessions: multiple concurrent, stdin, streaming output, resize,
interrupt, terminate. Raw bytes over a dedicated WebSocket. Each session keeps a
replay buffer bounded by **both** event count and retained bytes `[CI §10]`.

User-driven, therefore unsandboxed. Agent commands are a separate path inside the
kernel `plan.md §3`.

### FR-07: Git
Per-file status, worktree and staged diff, stage, unstage, commit, branch list,
branch create, checkout.

Only repository discovery and branch listing reuse upstream; the rest invokes
`git` directly so **user commit hooks run** — upstream disables them
`[CI §9.2]`. Arguments are passed as an argv array, never interpolated into a
shell string.

### FR-08: Providers
VLOOP-owned CRUD. Stored in VLOOP's database; injected into the kernel per run.

```text
id, name, base_url, auth_kind, credential_ref, http_headers,
request_max_retries, stream_max_retries, stream_idle_timeout_ms,
requires_openai_auth, enabled, metadata, created_at, updated_at
```

Acceptance:
- no Codex configuration file is read or written
- **no wire-protocol selector** — upstream has a single variant `[CI §3.1]`
- `credential_ref` names a VLOOP credential, never a secret value §11
- providers are injected via `ConfigToml.model_providers` at manifest assembly
  `[CI §3.1]`

### FR-09: Models
VLOOP-owned catalog CRUD.

```text
id, provider_id, slug, display_name, description, context_window,
max_output_tokens, supported_reasoning_levels[], default_reasoning_level,
service_tiers[], enabled, metadata
```

Acceptance:
- the catalog is injected as `Config.model_catalog`, producing a
  `StaticModelsManager` with **no cache file and no `/models` fetch**
  `[CI §3.2]`
- unknown reasoning-effort values are tolerated, because the upstream enum
  deserializes open-endedly `[CI §3.2]`
- a model can be seeded by importing Codex's bundled catalog once, after which
  the copy is VLOOP's

### FR-10: Agents
VLOOP-owned, versioned.

```text
id, name, description, runtime_id, model_id, reasoning_effort,
system_prompt, tool_selection_id, skill_ids[], mcp_server_ids[],
policy_id, memory_scope, context_config, environment, enabled,
current_version_id, metadata, created_at, updated_at
```

Acceptance:
- `tool_selection_id` is restored to the schema; tools are per-agent selectable
  FR-12
- every save creates an `agent_versions` row; runs reference a version, not the
  mutable agent FR-25
- editing an agent does not affect a run in flight

### FR-11: Runtime registry

```text
id, name, version, capabilities, status, last_error
```

v0.1 ships `codex-kernel` and `mock`. `capabilities` includes
`live_reconfiguration`, which the kernel runtime reports as **supported** —
`CodexThread::restore_thread_settings` is public `[CI §1.2]`.

### FR-12: Tools
VLOOP-owned selection per agent.

```text
selection_id, allow[] | deny[], exposure_overrides{}, unlisted: allow|deny
```

Acceptance:
- a tool inventory lists Codex built-ins plus MCP-provided tools, with origin
- an agent's selection determines which tools the model sees, applied inside
  `finalize_tool_router` via FC-5 `[CI §6.1, §7]`
- a denied tool is absent from the model-visible spec list **and** rejected if
  called
- when a tool cannot be enabled for an upstream reason — a model capability or a
  feature gate `[CI §6.1]` — the UI states the reason rather than silently
  ignoring the selection
- VLOOP-defined external tools can be registered through the extension
  contributor path `[CI §6.1]`

### FR-13: Skills
VLOOP-owned catalog.

```text
id, name, description, short_description, body, interface, dependencies,
scope, enabled, source, version, metadata
```

Acceptance:
- skills are stored in VLOOP and supplied to the kernel as a pre-parsed in-memory
  catalog via a `SkillCatalogProvider` registered as `Custom` `[CI §6.2]` + FC-6
- **host skill discovery is suppressed** when a VLOOP catalog is supplied, and
  bundled-skill installation never runs `[CI §2.4, §4.3]` + FC-2/FC-7
- skills can be created and edited in VLOOP — there is no upstream write API, and
  none is needed
- a skill can be imported from a `SKILL.md` file using the upstream parser
  `[CI §6.2]`
- `skills.list` / `skills.read` work against VLOOP-owned skills + FC-6
- enable/disable applies at any VLOOP scope; there is no user-scope-only
  restriction

### FR-14: MCP
VLOOP-owned definitions, injected per run.

```text
id, name, transport (stdio | streamable_http), command, args, env,
url, bearer_token_env_var, http_headers, enabled, startup_timeout_sec,
tool_timeout_sec, enabled_tools[], disabled_tools[], metadata
```

Acceptance:
- definitions are injected via `McpServerContribution::ReplaceCatalog`, so
  Codex's config-derived and plugin-derived servers are **replaced, not
  overlaid** `[CI §6.3]` + FC-8
- client-side validation mirrors upstream rules: transport-exclusive keys are
  rejected, inline bearer tokens are never submitted `[CI §6.3]`
- connection, tool listing and tool invocation reuse `codex-mcp` and
  `rmcp-client` unchanged
- per-server startup state, failure reason, tool list and auth state are shown
- approving "always allow" for an MCP tool updates **VLOOP's** policy and never
  writes Codex config `[CI §4.3]` + FC-9

### FR-15: Tasks

```text
id, workspace_id, name, description, priority, status,
plan_id, archived_at, created_at, updated_at
```

A task with no plan gets a single-step plan on execution.

### FR-16: Plans
A versioned DAG of steps. v0.1 executes a topological order sequentially.

```text
plan:      id, task_id, version, status, created_at
plan_step: id, plan_id, name, description, agent_id, depends_on[],
           inputs, success_criteria, workspace_claim, status, order_index
```

Acceptance:
- `depends_on[]` and `workspace_claim` are persisted and validated even though
  v0.1 runs sequentially `plan.md §8.2`
- a cyclic dependency graph is rejected at save time
- the sequential scheduler asserts that co-scheduled claims do not overlap

### FR-17: Runs and step attempts
A **Run** is one execution of a Plan. A **StepAttempt** is one execution of a Step
within it. Retry creates a new attempt and never mutates a prior one.

```text
run:          id, task_id, plan_id, plan_version, workspace_id,
              snapshot_id, settings_version, persistence_generation,
              status, started_at, finished_at, error, metadata
step_attempt: id, run_id, step_id, attempt_number, agent_version_id,
              runtime_id, thread_id, status, started_at, finished_at,
              error, interrupt_reason, evaluation_id, metadata
```

States for both: `queued`, `running`, `waiting_approval`, `paused`, `completed`,
`failed`, `cancelled`, `interrupted`.

Acceptance:
- terminal state is derived from kernel events; a transient provider error keeps
  the attempt `running`
- `waiting_approval` has a timeout; on expiry VLOOP sends the native `TimedOut`
  decision `[CI §1.3]`
- retry increments `attempt_number`; prior attempt rows are immutable
- a run references exactly one `snapshot_id` FR-25

### FR-18: Threads
Conversation state is **VLOOP-owned**, because VLOOP implements `ThreadStore`
`[CI §4.1]`.

Acceptance:
- `threads` and `thread_items` are VLOOP tables; the store satisfies every
  required trait method `[CI §4.1]`
- thread content is readable with the kernel stopped
- the store declares Legacy history mode, since paginated history is Local-only
  `[CI §4.2]`
- resuming an attempt rehydrates history from VLOOP's store
- **no rollout file, no Codex SQLite database and no `$CODEX_HOME` write occurs**
  FR-23

### FR-19: Runtime events
Normalized VLOOP event set:

```text
RunStarted        RunStateChanged    AttemptStarted     AttemptCompleted
AgentMessage      Reasoning          ToolStarted        ToolOutput
ToolCompleted     CommandStarted     CommandOutput      FileChanged
DiffUpdated       PlanUpdated        ApprovalRequested  ApprovalResolved
ArtifactCreated   TokenUsage         EvaluationRecorded InsightProposed
RunCompleted      RunFailed          RunInterrupted     Warning
Lagged
```

Acceptance:
- every event carries `run_id` and `attempt_id`; clients never see Codex
  identifiers
- translation is a single typed hop from `EventMsg`, with no codec `plan.md §6.4`
- timestamps normalize to RFC 3339 UTC at the boundary
- UI-bound output is byte-bounded per item; the full stream goes to the run log
- bus lag emits an explicit `Lagged` event rather than dropping silently

### FR-20: Approvals
Approvals are in-process operations, not protocol round-trips `[CI §1.3]`.

Acceptance:
- command and patch approval requests are surfaced with command or diff, working
  directory, and reason
- the decision set preserves the full upstream semantics:

  | UI action | Decision | Effect |
  |---|---|---|
  | Approve | `Approved` | run it |
  | Approve for session | `ApprovedForSession` | stop asking this session |
  | Approve and remember | `ApprovedExecpolicyAmendment` | persist a rule **to VLOOP** |
  | Deny, keep going | `Denied { rejection }` | agent adapts; reason is passed |
  | Deny and stop | `Abort` | halt until the next user turn |

- the deny reason is user-editable, because the agent receives it
- an unanswered approval times out to the native `TimedOut` decision
- every decision is recorded in `approval_records` with its attempt
- a rule amendment updates VLOOP policy and enters the next snapshot; it never
  writes Codex state FR-23

### FR-21: Artifacts
Files created or modified, patches, command outputs and test results, addressed by
`attempt_id` so provenance survives retries.

### FR-22: Settings
General, appearance, providers, models, agents, tools, skills, MCP, policies,
terminal, Git, keyboard shortcuts, learning thresholds.

All settings are VLOOP-owned. There is no configuration-layer provenance to
display, because there are no layers.

### FR-23: Kernel isolation — **the defining requirement**

Codex must hold no durable state and no configuration authority.

Acceptance, each an automated test:
1. **A full task executes end to end with `$CODEX_HOME` pointed at a read-only
   directory.**
2. No Codex configuration file is read during any VLOOP operation.
3. `ephemeral = true` is asserted on every thread start and fork `[CI §4.4]`.
4. Bundled-skill installation never runs, verified with the suppression carried in
   a `SessionFlags` **config layer** rather than only in `ConfigToml` — the
   layer-stack trap `[CI §2.4]`.
5. No rollout file, Codex SQLite database, `models_cache.json`, `history.jsonl`,
   `default.rules`, installation-id file or MCP credential file is created
   `[CI §4.3]`.
6. Credentials resolve through `ExternalAuth` with an ephemeral store, and
   `auth.json` is never read `[CI §5]`.
7. With a stray `CODEX_ACCESS_TOKEN` in the environment, VLOOP's resolver still
   wins — the known precedence leak `[CI §5]`.

### FR-24: Thread-store consistency
Because VLOOP both consumes the event stream and implements `ThreadStore`, it
observes turn content twice `plan.md §6.4`.

Acceptance:
- `run_events` is the observability log; `thread_items` is the conversation of
  record; neither is derived from the other
- an attempt's `thread_items` reconstruct a coherent conversation, verified by
  resuming the thread and comparing whole objects
- no item is double-recorded in `thread_items`

### FR-25: Resource snapshots
Every run executes against an immutable snapshot.

Acceptance:
- starting a run resolves provider, model, agent version, tool selection, skills,
  MCP definitions and policy into a `resource_snapshots` row
- editing any of those mid-run does not change the running attempt
- a tool call stamped with a stale snapshot id is rejected, reusing the upstream
  catalog-revision precedent `[CI §6.3]`
- a completed run can be re-executed from its snapshot after its agent has been
  edited, and behaves identically

### FR-26: Learning loop
v0.1 ships signal extraction, deterministic evaluation and manual promotion with
full provenance `plan.md §9.3`.

Acceptance:
- attempt completion extracts signals: outcome, approval count and decisions, tool
  errors, retries, diff size, test results, tokens, wall time
- deterministic evaluators run and record an `evaluations` row: did tests pass,
  did the patch apply, did the expected file appear, did the command exit zero
- candidate `insights` are generated with a scope (`agent`, `workspace`,
  `task_type`, `global`) and a confidence
- promotion is **manual** in v0.1 and writes a `promotions` row recording the
  source attempt, evaluator and confidence
- a promoted insight is visible in the next run's `ContextPack`, disableable, and
  traceable to its origin
- memory is token-budgeted during context assembly; it competes with files and
  history rather than growing unbounded

### FR-27: Settings plane — no ambient configuration

Codex's configuration inputs arrive from a VLOOP-owned, remotely managed,
versioned settings store — never from the ambient process environment. Full
inventory, rationale and migration waves in `settings-plane.md`.

Why this is a requirement and not a preference: environment variables are
process-global and single-valued, so two agents needing different credentials for
one provider is **unrepresentable** (`ModelProviderInfo::api_key()` reads
`std::env::var`, `[SP §3]`). The obvious workaround is unavailable — the workspace
is edition 2024, where `std::env::set_var` is `unsafe`, and Codex's own mutations
are justified by single-threadedness `[SP §3]`.

Acceptance:
- settings resolve per run through the scope chain
  `global → workspace → agent → agent_version → run`, and each resolved value
  records **which scope supplied it** `[SP §5.1]`
- each key class carries its own failure semantics: **credential, trust material
  and policy ceilings fail closed**; endpoint and behavioural keys fall back to
  last-known-good cache `[SP §5.2]`
- **two agents run concurrently in one process against the same provider with
  different credentials**, and neither observes the other's value `[SP §9]`
- **VLOOP never calls `std::env::set_var` or `remove_var`**; child environments are
  materialized as a pure function of snapshot, host-local allowlist and shell
  policy `[SP §5]`
- existing non-inheritable-variable scrubbing is preserved, not bypassed
  `[SP §4]`
- settings are **pinned into the run's `ResourceSnapshot`**: a remote change
  mid-run cannot affect an in-flight run, and a replay resolves the original
  `settings_version` `[SP §5.4]`
- a remote value **cannot broaden** a sandbox or approval ceiling beyond the local
  one `[SP §5.5]`
- credential plaintext never reaches the database or the settings cache — leases
  and references only `[SP §5.2]`
- **a run started fully offline with a warm cache and a valid credential lease
  succeeds** `[SP §5.6]`
- machine-local values (`CODEX_HOME`, `PATH`, `TMPDIR`, `TERM`, `RUST_LOG`) and
  `CODEX_SANDBOX*` remain environment-derived by design `[SP §5.3]`

### FR-28: Injected ORM persistence

VLOOP persistence is selected and injected at server or tenant-runtime
construction. Domain services depend on repository and `UnitOfWork` interfaces,
not SQL strings, SQLx pools, ORM entities or a specific database backend. Full
design in `persistence-plane.md`.

Acceptance:
- two profiles are supported: `embedded` SQLite and `managed` PostgreSQL
  `[PP §1.5]`
- connection configuration arrives through injected `DatabaseBootstrap` and
  `PersistenceFactory`; **no ambient `DATABASE_URL`** `[PP §4.2]`
- database credentials are short-lived leases; plaintext never appears in the
  authoritative database, settings cache or logs — the bootstrap credential
  cannot live only inside the database it unlocks `[PP §1.6]`
- `vloop-core`, runtime and API crates import no ORM, SQLx or backend-specific type
  `[PP §4]`
- the selected ORM/data-mapper adapter and exact pinned versions are chosen only
  after S5 proves workspace, SQLx and Bazel compatibility `[PP §3.1]`
- one logical migration sequence is tested on both backends; destructive changes
  use expand/backfill/contract and managed startup does not auto-apply a
  destructive migration `[PP §5.1]`
- run start, ordered thread append, attempt transition, approval resolution and
  memory promotion are atomic `UnitOfWork` operations `[PP §6]`
- committed domain events use a transactional outbox with **at-least-once**
  delivery and `event_id` deduplication; no exactly-once claim `[PP §6]`
- mutable aggregates use optimistic versions; multi-worker step claims use
  expiry and fencing tokens `[PP §7]`
- the same repository, migration, scheduler and Codex `ThreadStore` conformance
  suite passes against SQLite and a real PostgreSQL service `[PP §10]`
- a backend change follows the quiescent cutover protocol: pause new scheduling,
  wait for active attempts to terminate (or explicitly interrupt and persist
  them), migrate and verify, inject a new `PersistenceGeneration`, retain the
  prior backend read-only for rollback, then resume `[PP §8]`
- **no active attempt crosses persistence generations in v0.1**; checkpoint/resume
  across generations is a future protocol, not an implicit connection swap
- managed mode does not dual-write orchestration state to SQLite. Its only local
  database is the non-secret, versioned last-known-good settings cache
  `[PP §9]`

## 6. Data model

| Domain | Tables |
|---|---|
| Workspaces | `workspaces` |
| Resources | `providers`, `models`, `tools`, `tool_selections`, `skills`, `mcp_servers`, `policies` |
| Agents | `agents`, `agent_versions` |
| Snapshots | `resource_snapshots` |
| Orchestration | `tasks`, `plans`, `plan_steps`, `runs`, `step_attempts` |
| Threads | `threads`, `thread_items` |
| Observability | `run_events`, `approval_records` |
| Outputs | `artifacts` |
| Learning | `evaluations`, `insights`, `memory_entries`, `promotions` |
| Settings | `settings_scopes`, `settings_values`, `credential_refs` |
| System | `settings`, `schema_migrations`, `outbox_events` |

The last-known-good `settings_cache` is a **separate local cache store**, even in
managed/PostgreSQL mode. It is not an authoritative table and is never a general
write-behind replica of orchestration state `[PP §9]`.

Constraints:
- UUID primary keys; RFC 3339 UTC timestamps.
- JSON metadata columns for forward compatibility.
- Foreign keys throughout; `ON DELETE` is restrictive for anything a run
  references, so snapshots and attempts cannot be orphaned.
- Soft archive for tasks and runs; hard delete only for drafts.
- Append-only `run_events`, indexed by `(run_id, seq)`; `thread_items` indexed by
  `(thread_id, seq)`.
- `step_attempts` rows are immutable once terminal.
- Secret **references** only; never a secret value §11.
- persistence is accessed through injected repository and `UnitOfWork` interfaces;
  no domain or API crate imports ORM or SQLx types `[PP §4]`
- embedded SQLite and managed PostgreSQL adapters pass the same repository and
  `ThreadStore` conformance suite `[PP §10]`
- the `ThreadStore` implementation tolerates concurrent append from the kernel
  while the API reads; backend-specific locking remains inside the adapter

## 7. Resource resolution

The previous revision specified a config read-version-write-verify protocol. It is
deleted: VLOOP writes no Codex configuration. It is replaced by resolution.

```text
User edits a resource        →  VLOOP database (immediately authoritative)
Run starts                   →  resolve all resources → ResourceSnapshot (frozen)
                             →  build RuntimeManifest from the snapshot
                             →  construct Config, ModelsManager, ExtensionRegistry
                                in memory  [CI §1.5, §2.2, §6]
Kernel executes              →  no disk read of Codex config, no write anywhere
```

Two rules:

1. **Resolution failure blocks the run with a specific error.** A missing model, a
   disabled provider or an unsatisfiable tool selection fails before the kernel
   starts, with a message naming the offending resource.
2. **A snapshot is never mutated.** Edits after resolution apply to the next run.

## 8. API contract

All routes require authentication §11. Errors use the model in §10.

### Health and runtimes
```
GET /api/health
GET /api/runtimes
GET /api/runtimes/:id/capabilities
```

### Workspaces
```
GET/POST       /api/workspaces
GET/PUT/DELETE /api/workspaces/:id
POST           /api/workspaces/:id/open
POST           /api/workspaces/:id/close
```

### Resources
```
GET/POST       /api/providers
GET/PUT/DELETE /api/providers/:id
GET/POST       /api/models
GET/PUT/DELETE /api/models/:id
POST           /api/models/import          seed from the bundled catalog
GET            /api/tools                  inventory with origin
GET/POST       /api/tool-selections
GET/PUT/DELETE /api/tool-selections/:id
GET/POST       /api/skills
GET/PUT/DELETE /api/skills/:id
POST           /api/skills/import          parse a SKILL.md file
GET/POST       /api/mcp
GET/PUT/DELETE /api/mcp/:id
GET            /api/mcp/status
GET/POST       /api/policies
GET/PUT/DELETE /api/policies/:id
```

### Agents
```
GET/POST       /api/agents
GET/PUT/DELETE /api/agents/:id
POST           /api/agents/:id/duplicate
GET            /api/agents/:id/versions
```

### Orchestration
```
GET/POST       /api/tasks
GET/PUT/DELETE /api/tasks/:id
GET/POST       /api/tasks/:id/plans
PUT            /api/plans/:id
POST           /api/tasks/:id/run
GET            /api/runs
GET            /api/runs/:id
GET            /api/runs/:id/events            replay, paginated
GET            /api/runs/:id/attempts
POST           /api/runs/:id/interrupt
POST           /api/runs/:id/replay            re-execute from its snapshot
GET            /api/attempts/:id
POST           /api/attempts/:id/retry
POST           /api/attempts/:id/approvals/:approval_id
GET            /api/attempts/:id/artifacts
GET            /api/snapshots/:id
```

### Threads
```
GET /api/threads
GET /api/threads/:id
GET /api/threads/:id/items
```
Served from VLOOP's store, so they work with the kernel stopped.

### Learning
```
GET  /api/attempts/:id/evaluation
GET  /api/insights
POST /api/insights/:id/promote
POST /api/insights/:id/dismiss
GET  /api/memory
POST /api/memory/:id/disable
GET  /api/promotions
```

### Files, search, Git
```
GET/PUT        /api/files/content
GET            /api/files
POST           /api/files
POST           /api/files/rename
DELETE         /api/files
GET            /api/search/files
GET            /api/search/content
GET            /api/git/status
GET            /api/git/diff
POST           /api/git/stage
POST           /api/git/unstage
POST           /api/git/commit
GET/POST       /api/git/branches
POST           /api/git/checkout
```

### Terminal
```
POST           /api/terminals
GET            /api/terminals
POST           /api/terminals/:id/write
POST           /api/terminals/:id/resize
POST           /api/terminals/:id/signal
DELETE         /api/terminals/:id
```

### Streams
```
WS /api/events                    normalized domain events
WS /api/terminals/:id/stream      raw PTY bytes
```
Two endpoints deliberately: terminal throughput must not share a backpressure
domain with domain events.

## 9. Frontend requirements

### Shell
Dark-first, dense, resizable panels, command palette, kernel status indicator.

### Explorer
File tree, Git status markers, quick open, content search, workspace selector.

### Editor
Monaco, tabs, dirty indicator, save, diff view.

### Run view
Plan DAG with per-step status; attempt list including retries; event timeline with
markdown, tool cards and bounded command output; approval cards; interrupt.

### Agent panel
Agent and model selection, task prompt, live event stream, token usage.

### Management screens
Providers, Models, Agents, Tools, Skills, MCP, Policies, Tasks, Runs, Insights,
Settings.

### Learning view
Attempt evaluation detail, candidate insights with confidence and provenance,
promote/dismiss actions, memory browser.

### UI state
Zustand for ephemeral state: `activeWorkspace`, `openTabs`, `activeTab`,
`panelSizes`, `activeAgent`, `activeRun`, `activeAttempt`, `terminalTabs`,
`commandPalette`, `modals`, `connectionState`. TanStack Query for all server data.

Types come from VLOOP's own OpenAPI/TS generation. There is no Codex protocol
surface to track, because integration is compile-time `plan.md §4.1`.

## 10. Error model

```json
{
  "code": "RESOURCE_RESOLUTION_FAILED",
  "message": "Model 'gpt-5-codex' is disabled and cannot be used by agent 'Backend Engineer'",
  "subsystem": "orchestrator",
  "recoverable": true,
  "details": { "agent_id": "…", "model_id": "…" }
}
```

Required codes include `KERNEL_UNAVAILABLE`, `KERNEL_PANIC`,
`RESOURCE_RESOLUTION_FAILED`, `SNAPSHOT_STALE`, `PLAN_CYCLE_DETECTED`,
`WORKSPACE_CLAIM_CONFLICT`, `PATH_OUTSIDE_WORKSPACE`, `APPROVAL_TIMEOUT`,
`APPROVAL_UNKNOWN`, `ATTEMPT_NOT_INTERRUPTIBLE`, `GIT_NOT_A_REPOSITORY`,
`TERMINAL_NOT_FOUND`, `CREDENTIAL_UNAVAILABLE`.

The frontend renders an action for every recoverable error and never shows a raw
stack trace by default.

## 11. Security

- Localhost-only binding by default.
- **Bearer-token authentication on every route and both WebSocket endpoints**,
  generated at startup. Mandatory, not hardening: the API spawns PTY sessions, and
  loopback binding alone does not stop a local page from connecting. Codex's own
  execution socket pairs loopback with `Origin` rejection and still does not treat
  that as authentication `[CI §10]`.
- WebSocket handshakes validate `Origin` against an allowlist.
- Credentials are injected through `ExternalAuth` with an ephemeral store
  `[CI §5]`. `env_key` and `env_http_headers` are left unset so the process
  environment cannot supply credentials implicitly.
- VLOOP-owned secrets use the keyring-backed secrets manager `[CI §11.2]`.
  `credentials_refs` stores references, never values.
- `$CODEX_HOME/auth.json` is never read, logged or copied.
- `redact_secrets` is applied before any command output is logged or displayed.
- **The kernel shares VLOOP's address space.** Decrypted secrets are not retained
  in process memory beyond the request that needs them.
- Sandbox and approval enforcement remain active; embedding preserves them because
  the sandbox wraps spawned commands `[CI §11.1]`.
- Policy translation may only narrow `plan.md §10`.
- Unrestricted filesystem access requires explicit per-run opt-in.
- No remote exposure in v0.1.

## 12. Observability

Structured logs carrying `run_id`, `attempt_id`, `task_id` and `thread_id`.

```text
~/.vloop/logs/backend.log
~/.vloop/logs/kernel.log                  captured Codex tracing output
~/.vloop/logs/runs/<run_id>.log           full unbounded event stream
~/.vloop/logs/frontend.log
```

Because the kernel is in-process, VLOOP installs its own `tracing` subscriber and
captures the `codex_otel.log_only` and `codex_otel.trace_safe` targets directly
`[CI §11.3]`. Optional OTLP export tags spans with the VLOOP run id.

## 13. Performance targets

- The UI shell renders without waiting on the kernel.
- Resource CRUD feels immediate. Embedded mode may use optimistic UI updates;
  managed mode confirms versioned writes and surfaces compare-and-swap conflicts
  rather than pretending a remote commit is local `[PP §7.1]`.
- Event streaming never blocks editor or file interaction.
- Terminal and command output are bounded in the UI; raw logs are retained.
- `thread_items` append must not become the bottleneck in a streaming turn:
  appends are batched and must sustain a fast turn without stalling the loop.
  No database transaction is held across a model request or tool execution
  `[PP §11]`.
- managed-mode pool saturation applies backpressure and emits metrics; it never
  causes an unrecorded state transition.
- Quick open stays responsive at 50,000 files.
- Snapshot resolution completes in well under a second for a typical agent.

## 14. Golden-path acceptance test

1. Launch VLOOP; health reports backend and kernel ready.
2. Open a Git workspace; repository detected, tree populated.
3. Create a provider with a credential reference; create a model.
4. Create a `Full Stack Engineer` agent: model, tool selection, one VLOOP-defined
   skill, a workspace-write policy, approval-on-request.
5. Create the task: `Create hello.txt containing hello world and verify it exists.`
6. Start the run; observe a snapshot being created.
7. Observe `AttemptStarted`, streaming `AgentMessage` and `Reasoning`, then
   command and file-change events.
8. Approve the requested command; confirm the decision is recorded.
9. Observe `FileChanged`, then `RunCompleted`.
10. Open `hello.txt` in Monaco; inspect the diff.
11. Confirm conversation history is in `thread_items` and readable.
12. **Confirm nothing was written under `$CODEX_HOME`** FR-23.
13. Run a command in the terminal; resize it; interrupt it.
14. Stage and commit; confirm a commit hook ran.
15. Edit the agent's system prompt, then replay the original run from its
     snapshot; behaviour is unchanged FR-25.
16. Start a second run, deny an approval with "keep going", and observe the agent
     adapt rather than halt.
17. Start a third run and interrupt it; it reaches `cancelled` and the kernel
     stays healthy.
18. Let an approval time out; the attempt fails with `APPROVAL_TIMEOUT`.
19. Review the attempt's evaluation; promote one insight; confirm provenance.
20. Restart VLOOP; all state is present, no attempt is left `running`, and the
     promoted insight appears in the next run's context.
21. Run the same persistence conformance scenario against embedded SQLite and
     managed PostgreSQL; verify equivalent domain results.
22. Rehearse a PostgreSQL candidate cutover whose verification fails; confirm the
     prior persistence generation remains authoritative and scheduling resumes
     there.

## 15. Release gates

### Ownership gate — the primary gate
A full task executes with `$CODEX_HOME` read-only. No Codex configuration file is
read or written. No Codex durable state is created. FR-23.

### Settings gate
A full run executes with the process environment reduced to a host-local
allowlist: no credential, endpoint or trust value arrives from ambient env. Two
agents run concurrently in one process against the same provider with different
credentials. No VLOOP crate calls `std::env::set_var` or `remove_var`. The
env-read ratchet is at or below its recorded ceiling. FR-27, `[SP §9]`.

### Persistence portability gate
The same migration, repository, scheduler-claim and Codex `ThreadStore`
conformance suite passes against embedded SQLite and a real PostgreSQL service.
No ORM, SQLx or backend-specific type appears in `vloop-core`, runtime or API
crates. Bootstrap is injected without `DATABASE_URL`; controlled cutover and
rollback preserve `persistence_generation`. FR-28, `[PP §10]`.

### Architecture gate
`vloop-core` has zero Codex dependencies. A CI grep confirms no Codex type name
outside `vloop-kernel-codex`. `MockRuntime` passes the same attempt-lifecycle
suite as the kernel runtime.

### Codex non-regression gate
**Redefined for this revision.** The full Codex test suite passes after all fork
changes, and Codex's own behaviour is unchanged when no VLOOP snapshot is
supplied `plan.md §15`. Bazel lockfile and `BUILD.bazel` updates are present for
every dependency change `[CI §12]`.

### Fork hygiene gate
Every fork change is documented in `vloop/docs/fork-changes.md` with rationale,
diff scope and an upstream note, and is individually reversible. No change adds
VLOOP-specific logic to `codex-core`. No forbidden symbol was publicized
`[CI §7]`. Every behaviour gate defaults to current Codex behaviour when no VLOOP
value is supplied, including the settings-plane changes FC-11..FC-15 `[SP §7]`.

### Backend gate
`cargo check` and the VLOOP test suite pass; clippy clean. CI runs persistence
migrations and conformance against SQLite and a real PostgreSQL service. Managed
startup rejects incompatible schema versions and a failed cutover leaves the old
backend authoritative.

### Frontend gate
Typecheck, lint and production build pass.

### Reproducibility gate
A run replayed from its snapshot after its agent was edited behaves identically
FR-25.

### Security gate
Unauthenticated requests rejected on every route and both WebSocket endpoints;
`Origin` rejection verified; path traversal rejected; Git argv construction
injection-safe; no secret value in any log; `auth.json` never read; environment
credential precedence leak covered by a test.

### UX gate
No dead-end buttons or placeholder screens on the golden path. Every deferred
capability — parallel execution, automatic promotion — states its status in the UI
rather than appearing broken.
