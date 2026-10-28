# Codex Integration Map (validated)

Evidence base for VLOOP. `plan.md`, `spec.md` and `tasks.md` cite this file as
`[CI §n]` instead of repeating citations. The environment-variable surface and its
migration are measured separately in `settings-plane.md`, cited as `[SP §n]`.
VLOOP's injected ORM persistence architecture is specified in
`persistence-plane.md`, cited as `[PP §n]`.

All references are `path:line` into `codex-rs/` at the pinned commit. Every claim
was read from source. Unverified items are isolated in §13 and must not be
treated as facts.

**Revision note.** This document was rewritten after the ownership directive:
*VLOOP is the sole source of truth for all data and config; Codex is a Swiss
army knife whose crates we take, modify where needed, and drive.* The previous
revision assumed a subprocess + JSON-RPC integration and concluded that Codex
owned providers/models/tools/skills/MCP. That conclusion was an artefact of the
chosen seam, not a property of Codex. §1 replaces it.

---

## 1. The integration seam: in-process kernel

### 1.1 `ThreadManager::new` is a pure dependency-injection constructor

This is the single most important fact in this document.
`codex-rs/core/src/thread_manager.rs:419-434`, verified verbatim:

```rust
pub fn new(
    config: &Config,
    auth_manager: Arc<AuthManager>,
    models_manager: SharedModelsManager,
    codex_apps_tools_cache: CodexAppsToolsCache,
    session_source: SessionSource,
    environment_manager: Arc<EnvironmentManager>,
    extensions: Arc<ExtensionRegistry<Config>>,
    user_instructions_provider: Arc<dyn UserInstructionsProvider>,
    analytics_events_client: Option<AnalyticsEventsClient>,
    thread_store: Arc<dyn ThreadStore>,
    agent_graph_store: Option<Arc<dyn AgentGraphStore>>,
    installation_id: String,
    attestation_provider: Option<Arc<dyn AttestationProvider>>,
    external_time_provider: Option<Arc<dyn TimeProvider>>,
) -> Self
```

Every durable-state dependency is injected as a trait object or a plain value:

| Dependency | Trait / type | VLOOP supplies |
|---|---|---|
| Thread persistence | `pub trait ThreadStore: Any + Send + Sync` (`codex-rs/thread-store/src/store.rs:68`) | its own bridge over injected `Arc<dyn Persistence>` |
| Model catalog | `SharedModelsManager` (`codex-rs/models-manager/src/manager.rs:214`) | `StaticModelsManager` from VLOOP's catalog |
| Credentials | `Arc<AuthManager>` (`codex-rs/login/src/auth/manager.rs:2075`) | ephemeral mode + `ExternalAuth` |
| Tools / skills / MCP | `Arc<ExtensionRegistry<Config>>` (`codex-rs/ext/extension-api/src/registry.rs:123`) | contributors from a VLOOP snapshot |
| Agent graph | `Option<Arc<dyn AgentGraphStore>>` (`codex-rs/agent-graph-store/src/store.rs:17`) | `None` |
| Installation id | `String` | a VLOOP value, no file read |
| Time | `Option<Arc<dyn TimeProvider>>` | deterministic clock in tests |

There is no hidden global. Codex's own local behaviour is just the default set of
arguments that `app-server` happens to pass
(`codex-rs/app-server/src/message_processor.rs:286`, `:343-345`).

### 1.2 `CodexThread` is a public agent-loop handle

`codex-rs/core/src/codex_thread.rs:166`, with these public methods (verified):

| Method | Line | Purpose |
|---|---|---|
| `submit(&self, op: Op) -> CodexResult<String>` | `:211` | drive the loop, incl. approvals |
| `next_event(&self) -> CodexResult<Event>` | `:486` | consume the event stream |
| `start_turn_if_idle(...)` | `:295` | begin a turn |
| `steer_turn(...)` | `:351` | mid-turn steering |
| `restore_thread_settings(...)` | `:430` | **live reconfiguration** |
| `preview_thread_settings_overrides(...)` | `:418` | validate before applying |
| `environment_ready(...)` | `:712` | inject per-run environment policy |
| `inject_response_items(...)` | `:532` | inject context directly |
| `config_snapshot()` | `:647` | read effective config |
| `token_usage_info()` | `:513` | usage accounting |
| `shutdown_and_wait()` | `:225` | clean teardown |

`CodexThreadSettingsOverrides` is public at `:149`.

### 1.3 Approvals are in-process, no JSON-RPC

`Op::ExecApproval { .. }` (`codex-rs/protocol/src/protocol.rs:599`) and
`Op::PatchApproval { .. }` (`:609`) carry a `ReviewDecision`
(`:3871`, verified verbatim):

```rust
pub enum ReviewDecision {
    Approved,
    ApprovedExecpolicyAmendment { proposed_execpolicy_amendment: ExecPolicyAmendment },
    ApprovedForSession,
    ApprovedMcpPolicyAmendment,
    NetworkPolicyAmendment { network_policy_amendment: NetworkPolicyAmendment },
    Denied { rejection: String },   // agent continues, tries something else
    TimedOut,                        // automatic review timed out
    Abort,                           // agent stops until the next user command
}
```

Requests arrive as events: `EventMsg::ExecApprovalRequest` (`:1410`) and
`ApplyPatchApprovalRequest` (`:1422`).

Two consequences. The `Denied` / `Abort` distinction is richer than the
app-server surface — `Denied` carries a rejection string the agent can reason
about. And `TimedOut` is a first-class decision, so VLOOP's approval timeout has
a native representation rather than needing a synthesized cancel.

### 1.4 Why in-process, not subprocess

The previous revision proposed spawning `codex app-server`. That seam cannot
satisfy sole ownership, for a specific reason: **app-server re-reads config from
disk on every thread start / resume / fork.** `load_latest_config_for_thread`
(`codex-rs/app-server/src/config_manager.rs:147-160`) calls
`rebuild_preserving_session_layers`, which reaches `LOCAL_FS`
(`codex-rs/core/src/config/mod.rs:1774`). An injected `Config` is therefore not
authoritative across the app-server's own reload path.

Comparison of the three strategies:

| | A. spawn app-server | B. `app-server-client::InProcess` | C. drive `ThreadManager` |
|---|---|---|---|
| VLOOP owns config | ✗ re-read per request | partial | **✓** |
| VLOOP owns thread persistence | ✗ | partial | **✓ injected `ThreadStore`** |
| Protocol translation cost | JSONL codec + 2 mappings | 1 mapping | **none** |
| Approval latency | 2 process hops | in-process | **in-process** |
| Tool/skill/MCP injection | config files only | extension registry | **extension registry** |
| Fault isolation | **✓ separate process** | ✗ | ✗ |
| Experimental-API gating | applies | applies | **N/A — not a core concept** |

Strategy C is chosen. Note the last row: `experimentalApi` gates the *app-server
protocol surface*, not core. Driving `ThreadManager` directly means the entire
gating problem disappears, along with the capability losses it forced
(live reconfiguration, item listing, thread search).

The one genuine loss is fault isolation: a panic in core takes down the VLOOP
process. Mitigation in `plan.md §7.5`.

### 1.5 What must be constructed

From the app-server reference wiring
(`codex-rs/app-server/src/message_processor.rs:248-345`) and
`codex-rs/app-server/src/in_process.rs:132-165`:

- `Config` — see §2
- `Arc<AuthManager>` — §5
- `SharedModelsManager` — §3
- `CodexAppsToolsCache`, `Arc<EnvironmentManager>`
- `Arc<ExtensionRegistry<Config>>` — §6
- `Arc<dyn UserInstructionsProvider>` (`codex-rs/ext/extension-api/src/user_instructions.rs:38`)
- `Arc<dyn ThreadStore>` — §4
- `installation_id: String` — a plain string; note
  `resolve_installation_id` (`codex-rs/core/src/installation_id.rs:19-64`)
  writes a file, but it is called by app-server (`in_process.rs:419`), **not**
  by `ThreadManager`. Pass a VLOOP value and no write occurs.

Optional / `None`-able: `analytics_events_client`, `agent_graph_store`,
`attestation_provider`, `external_time_provider`.

---

## 2. Config: programmatic construction

### 2.1 Shape

`Config` — `codex-rs/core/src/config/mod.rs:560`. All fields `pub`, ~120 of them,
**no `Default`** (`:559` derives only `Debug, Clone, PartialEq`). Constructing a
literal is upgrade-brittle; go through a loader entry point.

Fields VLOOP drives: `config_layer_stack` `:563`, `model` `:569`,
`model_provider_id` `:589`, `model_provider` `:592`, `model_providers` `:808`,
`model_catalog` `:911`, `permissions` `:598`, `approvals_reviewer` `:610`,
`base_instructions` `:627`, `developer_instructions` `:633`,
`include_skill_instructions` `:651`, `mcp_servers` `:781`, `agent_roles` `:838`,
`memories` `:844`, `codex_home` `:848`, `history` `:857`, `ephemeral` `:860`,
`experimental_thread_store` `:958`, `features` `:1005`.

`ThreadStoreConfig` — `:550`: `Local` (default) | `InMemory { id: String }`.

### 2.2 Two no-disk paths

**Public today** — `Config::load_default_with_cli_overrides_for_codex_home`
(`:1812`), documented at `:1810-1811` as loading *"without reading user, project,
or system config layers."* It serializes `ConfigToml::default()` (`:1816`),
merges a CLI-override layer (`:1822-1823`), and calls the typed loader with
`ConfigLayerStack::default()` (`:1826-1832`). Limitation: input is only
`Vec<(String, TomlValue)>` — no typed `ConfigToml`, no `ConfigOverrides`.

**Ideal but crate-private** — `Config::load_config_with_layer_stack(fs, cfg:
ConfigToml, overrides: ConfigOverrides, codex_home, config_layer_stack)` at
`:3078`, verified `pub(crate)`. This is exactly the VLOOP-shaped signature. Its
typed sibling `load_from_base_config_with_overrides` (`:3060`) is `#[cfg(test)]`.

Publicizing `:3078` is fork change **FC-1** (§7).

`ConfigBuilder` (`:1309`) always goes to disk: `build_inner` calls
`find_codex_home()` (`:1382`) then `load_config_layers_state` (`:1393`), which
reads packaged defaults, cloud bundle, MDM, system requirements, system config,
user config, profile, project layers and legacy managed config
(`codex-rs/config/src/loader/mod.rs:128-487`). `LoaderOverrides`
(`codex-rs/config/src/state.rs:43`) can suppress most of it — `ignore_user_config`
`:54`, `ignore_project_config` `:56`, `ignore_managed_requirements` `:51` — with
`without_managed_config_for_tests` (`:68`) as the reference pattern.

### 2.3 Residual disk access inside the typed loader

With `ConfigLayerStack::default()`, no `model_catalog_json`, no instruction files
and no `[agents.roles]`, the only filesystem work is:

| Path | What | Conditional |
|---|---|---|
| `:3294` `resolve_root_git_project_for_trust` | git discovery for project trust | **unconditional** |
| `:3853` `InstallContext::current().bundled_zsh_path()` | install probe | unconditional |
| `:3278` `env::current_dir()` | process cwd | only if `overrides.cwd` is `None` |
| `:3620` `agent_roles::load_agent_roles` | role files | walks layers; empty stack → only `cfg.agents` (`codex-rs/core/src/config/agent_roles.rs:120`) |
| `:3859` `load_model_catalog` → `read_to_string` (`:1974`) | JSON catalog | only if `model_catalog_json` set — **bypass by assigning `Config.model_catalog` directly** (`:911`) |

### 2.4 The layer-stack trap

**Some policy is read from `ConfigLayerStack`, not from `ConfigToml`.** This is
the subtlest finding in this document.

`bundled_skills_enabled_from_stack` reads `config_layer_stack.effective_config()`
(`codex-rs/config/src/skills_config.rs:130`) and **defaults to `true`** when no
`skills` table is present (`:135`). `skill_config_rules_from_stack` inspects only
`ConfigLayerSource::User` and `SessionFlags` layers (`:150-158`).

So a `Config` built with `ConfigLayerStack::default()` and `ConfigToml.skills`
set **still** triggers a write to `CODEX_HOME/skills/.system`. VLOOP must push a
real `ConfigLayerEntry::new(ConfigLayerSource::SessionFlags, toml)`
(`codex-rs/config/src/state.rs:132`; `ConfigLayerStack::new` `:272`) carrying
`skills.bundled.enabled = false`.

Project-local layers are denied `model_provider`, `model_providers`,
`openai_base_url`, `chatgpt_base_url`, `notify`, `otel` and more
(`codex-rs/config/src/loader/mod.rs:71-84`). VLOOP layers must be
`SessionFlags` or `User`, never `Project`.

---

## 3. Providers and models

### 3.1 Definitions are injectable as data

`ModelProviderInfo` (`codex-rs/model-provider-info/src/lib.rs:95`) is fully
public with `Default`: `base_url` `:100`, `env_key` `:102`,
`experimental_bearer_token` `:110`, `auth` `:112`, `http_headers` `:122`,
`env_http_headers` `:127`, `requires_openai_auth` `:143`, `wire_api` `:117`.

VLOOP supplies its own map via `ConfigToml.model_providers`
(`codex-rs/config/src/config_toml.rs:286`), merged at
`codex-rs/core/src/config/mod.rs:3629-3631`, with the selected id resolved at
`:3633-3646` (hard error if unknown, `:3644`).

`WireApi` has exactly one variant, `Responses`
(`codex-rs/model-provider-info/src/lib.rs:63-67`), with a hand-written
`Deserialize` rejecting `"chat"` (`:78-90`). This is a genuine upstream
constraint, not an artefact of the seam: VLOOP's provider form offers no
wire-api selector.

### 3.2 Catalog is fully in-memory

Set `Config.model_catalog = Some(ModelsResponse { models })` (`:911`) and
`ModelProvider::models_manager` returns a `StaticModelsManager`
(`codex-rs/model-provider/src/provider.rs:424`), with no cache file and no
`/models` fetch. `StaticModelsManager::new(Option<Arc<AuthManager>>,
ModelsResponse)` is public (`codex-rs/models-manager/src/manager.rs:289`);
`raw_model_catalog` `:556`, `get_remote_models` `:568`,
`refresh_if_new_etag` is a no-op `:584`.

Alternatively bypass config entirely and pass VLOOP's own
`Arc<dyn ModelsManager>` to `ThreadManager::new`. The trait is public
(`manager.rs:84`), as are `ModelsEndpointClient` (`:37`) and `ModelsCache`
(`codex-rs/models-manager/src/cache.rs:25`). **No fork change needed.**

`ModelInfo` (`codex-rs/protocol/src/openai_models.rs:391-481`) is the canonical
metadata type VLOOP populates. `ReasoningEffort` (`:50-62`) deserializes
open-endedly (`:119-127`), so any VLOOP enum must tolerate unknown values.

### 3.3 Provider *behaviour* is not injectable

`ModelClient::new` constructs the provider internally via
`create_model_provider` (`codex-rs/core/src/client.rs:445`), so a VLOOP
`impl ModelProvider` cannot reach the request path. The `ModelProvider` trait
(`codex-rs/model-provider/src/provider.rs:130`) and `SharedModelProvider`
(`:283`) are public, but `ConfiguredModelProvider` is private (`:307`).

Fork change **FC-10** (`ModelClient::with_provider`) is needed only if VLOOP
wants custom retry/routing/wire behaviour. Definitions alone need nothing.

---

## 4. Persistence: VLOOP as sole owner

### 4.1 `ThreadStore` is implementable

`pub trait ThreadStore: Any + Send + Sync` —
`codex-rs/thread-store/src/store.rs:68` (verified public).

**Required:** `as_any` `:70`, `create_thread` `:81`, `resume_thread` `:109`,
`append_items` `:115`, `persist_thread` `:122`, `flush_thread` `:129`,
`shutdown_thread` `:132`, `discard_thread` `:139`, `load_history` `:142`,
`read_thread` `:188`, `read_thread_by_rollout_path` `:193`, `list_threads`
`:199`, `update_thread_metadata` `:374`, `archive_thread` `:392`,
`unarchive_thread` `:415`, `delete_thread` `:418`.

**Defaulted to `Unsupported`:** `default_history_mode` `:76`,
`load_latest_model_context` `:150`, `prepare_fork` `:164`, `revert_thread`
`:179`, sections `:202-252`, projects `:255-317`, `search_threads` `:325`,
`list_turns` `:349`, `list_items` `:358`, plus batch variants.

All parameter and return types are public
(`codex-rs/thread-store/src/lib.rs:47-101`).

Core's write funnel is one method: `Session::persist_rollout_items` →
`LiveThread::append_items` (`codex-rs/core/src/session/mod.rs:3698-3704`).

### 4.2 Two real constraints on a custom store

1. **`as_any` downcasts to `LocalThreadStore`.** `Session::new` obtains a
   `state_db` only when the downcast succeeds
   (`codex-rs/core/src/session/session.rs:857-867`). A custom store therefore
   silently forfeits every `state_db`-gated feature — memories
   (`codex-rs/memories/write/src/start.rs:49-52`) and queue persistence
   (`codex-rs/app-server/src/message_processor.rs:289-294`). Silent feature loss,
   not an error. VLOOP owns memory anyway (§8), so this is aligned rather than
   costly — but it must be a deliberate, tested decision.
2. **Paginated history is Local-only.** `LiveThread::resume` downcasts for
   paginated metadata (`codex-rs/core/src/session/live_thread.rs:156-158`);
   `local_rollout_path` returns `Ok(None)` otherwise (`:369-381`). A VLOOP store
   accepts Legacy history mode or implements the paginated methods itself.

`InMemoryThreadStore` (`codex-rs/thread-store/src/in_memory.rs:497`) is a usable
reference and a viable interim: `resume_thread` accepts caller-supplied history
(`:583-597`), `append_items` applies the same persistence filter (`:599-616`),
lifecycle methods are call-counting no-ops (`:865-895`). Caveats: legacy history
only (`:543`, `:632`, `:663`) and a **process-global registry keyed by id**
(`:49-54`), so two runtimes sharing an id share state.

### 4.3 Every Codex-owned write, and how to disable it

| Subsystem | Write path | Suppression |
|---|---|---|
| Rollout files, compression, migration | `LocalThreadStore` + workers (`codex-rs/core/src/thread_manager.rs:388-401`) | custom / in-memory `ThreadStore` |
| State SQLite | `init_state_db` (`codex-rs/core/src/state_db_bridge.rs:6`) → `codex-rs/rollout/src/state_db.rs:45` | caller-supplied `state_db: Option<StateDbHandle>` → pass `None` (`codex-rs/app-server/src/in_process.rs:152`) |
| Agent graph edges | `persist_thread_spawn_edge_for_source` (`codex-rs/core/src/agent/control.rs:761-771`) | skipped when `ephemeral` (`:766`); pass `agent_graph_store: None` |
| **Bundled skills → `CODEX_HOME/skills/.system`** | `HostSkillsService::new_with_restriction_product` (`codex-rs/ext/skills/src/host_service.rs:132-134`) → `install_system_skills` (`codex-rs/skills/src/lib.rs:77-101`) | `bundled_skills_enabled` (`codex-rs/core/src/config/mod.rs:4358`) via the **layer stack** (§2.4) — called at `thread_manager.rs:440` |
| Model cache `models_cache.json` | `FileModelsCache::store` (`codex-rs/models-manager/src/cache.rs:227-233`) | `StaticModelsManager` (§3.2) |
| Memories under `CODEX_HOME/memories` | `start_memories_startup_task` (`codex-rs/memories/write/src/start.rs:24`) | early-returns on `ephemeral`, `!Feature::MemoryTool`, or no state db (`:33-52`); only called from app-server (`turn_processor.rs:597-605`) |
| `installation_id` file | `resolve_installation_id` (`codex-rs/core/src/installation_id.rs:19-64`) | app-server only (`in_process.rs:419`); pass a `String` |
| Global `history.jsonl` | `append_entry` (`codex-rs/message-history/src/lib.rs:104`) | `HistoryPersistence::None` (`:109-117`); callers are TUI-only |
| Execpolicy `default.rules` | `append_amendment_and_update` (`codex-rs/core/src/exec_policy.rs:452`), `append_network_rule_and_update` (`:502`) | **needs FC-3** |
| MCP approval → `config.toml` | `persist_codex_app_tool_approval` (`codex-rs/core/src/mcp_tool_call.rs:2047`), `persist_non_app_mcp_tool_approval` (`:2099`) | **needs FC-9** |
| MCP OAuth tokens | `save_oauth_tokens` (`codex-rs/rmcp-client/src/oauth.rs:397`) → `CODEX_HOME/.credentials.json` (`:826`) | **no in-memory mode exists** (`codex-rs/config/src/types.rs:123-135`) — avoid OAuth MCP auth |

### 4.4 `ephemeral` is the master switch, and it is per-thread

`Config.ephemeral` (`codex-rs/core/src/config/mod.rs:860`) is assigned at
`:4095` from `ConfigOverrides.ephemeral` (`:2527`) **only**. It is not a
`ConfigToml` field. Effects: no `LiveThread`
(`codex-rs/core/src/session/session.rs:770-772`), no `state_db` (`:857-859`), no
graph edge (`codex-rs/core/src/agent/control.rs:766`), no memories
(`codex-rs/memories/write/src/start.rs:33`), no metadata updates
(`codex-rs/core/src/thread_manager.rs:798-800`).

Because it is per-thread and caller-supplied, **any code path that starts or
forks a thread without setting it re-enables the full local persistence stack.**
This is the highest-probability regression in the whole design and is why
`spec.md` FR-23 makes it a tested invariant rather than a convention.

---

## 5. Credentials

`AuthManager::new(codex_home, enable_codex_api_key_env,
auth_credentials_store_mode, forced_chatgpt_workspace_id, chatgpt_base_url,
keyring_backend_kind, auth_route_config)` —
`codex-rs/login/src/auth/manager.rs:2075`.

**`ExternalAuth` trait** (`:254-265`): `resolve()`, `refresh(...)`,
`classify_error()`. Installed via `set_external_auth` (`:2544`), removable
(`:2572`), queryable (`:2618`).

`AuthCredentialsStoreMode::Ephemeral` (`codex-rs/config/src/types.rs:117`) is
backed by a process-local map (`codex-rs/login/src/auth/storage.rs:460-496`).
`load_auth` short-circuits before any persistent store in that mode (`:1487-1490`),
and `commit_external_auth` persists only into the ephemeral store (`:2922-2937`).

`CodexAuth` variants (`:76-84`) include `Headers(AuthHeaders)`;
`AuthHeaders::new(HeaderMap)` (`codex-rs/login/src/auth/auth_headers.rs:17`) has
a redacted `Debug` (`:26-31`). In-memory constructors: `from_api_key` (`:802`),
`from_external_chatgpt_tokens` (`:785`).

**So VLOOP injects credentials with zero fork changes**: ephemeral mode plus
`set_external_auth`.

**Two leaks to close in VLOOP's own wiring:**

1. `read_codex_access_token_from_env()` (`:1464`) is gated on
   `auth_mode_is_allowed(..., AuthMode::AgentIdentity)`, **not** on
   `enable_codex_api_key_env`. A stray `CODEX_ACCESS_TOKEN` in the environment
   wins over VLOOP's resolver during initial load.
2. `ModelProviderInfo::api_key()` reads `env_key` from the process env
   (`codex-rs/model-provider-info/src/lib.rs:334`), and `build_header_map()`
   reads `env_http_headers` (`:276`). Leave both `None` and use
   `experimental_bearer_token` or `CodexAuth::Headers`.

---

## 6. Resource injection: tools, skills, MCP

### 6.1 Tools — most coupled

Assembly chain: `built_tools` (`codex-rs/core/src/session/turn.rs:1476`,
`pub(crate)`) → `build_tool_router` (`codex-rs/core/src/tools/spec_plan.rs:121`,
`pub(crate)`) → `add_core_tool_sources` (`:893`) → MCP tools (`:152`) →
`apply_mcp_tool_exposure_policy` (`:178`) →
`append_extension_tool_executors` (`:1283`) → `append_dynamic_tool_runtimes`
(`:1232`) → `finalize_tool_router` (`:318`) → `ToolRouter::from_parts` (`:451`).

**Verified `pub(crate)`** (`codex-rs/core/src/tools/mod.rs:1-11`): `orchestrator`,
`registry`, `router`, `runtimes`, `sandboxing`, `spec_plan`, `lifecycle`,
`network_approval`, `parallel`. Only `pub use router::ToolRouter` (`:25`) escapes.
`ToolRegistry` is declared `pub struct` (`codex-rs/core/src/tools/registry.rs:277`)
but is unreachable externally. `CoreToolRuntime` is `pub(crate) trait` (`:55`).

Built-in selection is spread across ~12 `Config` / `Feature` / `model_info`
predicates: shell type (`spec_plan.rs:980`), MCP resources (`:1027`),
`update_plan_enabled` (`:1040`), `Feature::DeferredExecutor` (`:1044`),
`experimental_request_user_input_enabled` (`:1056`),
`experimental_supported_tools` (`:1068`, `:1122`), `Feature::TokenBudget`
(`:1079`), `Feature::CurrentTimeReminder` (`:1084`), `Feature::ViewImage`
(`:1127`), collaboration (`:1145-1152`).

**Public and reusable as-is** (crate `codex-tools`, re-exported at
`codex-rs/tools/src/lib.rs:21-113`): `ToolExecutor<Invocation>`
(`codex-rs/tools/src/tool_executor.rs:106`), `ToolExposure` (`:51`),
`ToolExposures` bitflags (`:17`), `ToolSpec`
(`codex-rs/tools/src/tool_spec.rs:22`).

Two additive paths already exist: `ToolContributor`
(`codex-rs/ext/extension-api/src/contributors.rs:279`, registered
`codex-rs/ext/extension-api/src/registry.rs:123`, consumed
`spec_plan.rs:1304` via `register_external`), and thread-scoped `DynamicToolSpec`
(`codex-rs/protocol/src/dynamic_tools.rs:13`) — but dynamic tools round-trip
execution to the client (`codex-rs/core/src/tools/handlers/dynamic.rs:171-245`),
so they are not in-process.

What does **not** exist: a caller-supplied allow/deny/exposure policy over
built-ins. That is fork change **FC-5**. The good news is that everything funnels
through one `finalize_tool_router`, so a single post-registration pass yields
VLOOP-owned selection without touching the agent loop.

### 6.2 Skills — half-migrated

`SkillProvider` is **public** (`codex-rs/ext/skills/src/provider.rs:71`,
verified) with `list` / `read` / `search`, and public request types
`SkillListQuery` (`:32`), `SkillReadRequest` (`:46`), `SkillSearchRequest`
(`:57`).

Composition is public: `SkillProviders` (`codex-rs/ext/skills/src/sources.rs:72`)
with `with_provider` (`:81`), `with_host_provider` (`:86`),
`with_executor_provider` (`:92`), `with_orchestrator_provider` (`:98`).
Catalog types have public builders: `SkillCatalogEntry::new`
(`codex-rs/ext/skills/src/catalog.rs:193`), `SkillCatalog` (`:286`),
`SkillReadResult` (`:314`), `SkillSourceKind::Custom` (`:18`, ctor `:22`).
Install: `install_with_providers` (`codex-rs/ext/skills/src/extension.rs:637`).

A VLOOP `SkillProvider` registered as `Custom` is always consulted
(`sources.rs:52`) and `HostSkillProvider` is inert without a snapshot
(`codex-rs/ext/skills/src/provider/host.rs:39-43`). Context injection —
`AvailableSkillsInstructions`, `SkillInstructions`, budgeting, mentions — is
fully reusable (`codex-rs/ext/skills/src/extension.rs:351-535`).

**Three blockers:**

1. Core performs host discovery unconditionally per turn:
   `new_turn_context_from_configuration` calls `snapshot_for_config`
   (`codex-rs/core/src/session/turn_context.rs:861-884`), and
   `TurnContext::skills_snapshot()` asserts presence with `unreachable!`
   (`:198-203`). Session startup also warms it
   (`codex-rs/core/src/session/session.rs:552-571`, called `:1114`).
2. A legacy core path injects host skills *before* extensions:
   `build_skills_and_plugins` (`codex-rs/core/src/session/turn.rs:740`, host
   snapshot `:782`, mentions `:790`, MCP deps `:792`, prompts `:808`, prepended
   `:820-874`) and `required_mcp_servers_for_input` (`:712-734`).
3. `skills.list` / `skills.read` are unavailable to custom providers:
   `skill_tools` early-returns (`codex-rs/ext/skills/src/tools/mod.rs:66-70`) and
   `SkillToolAuthority::from_authority` returns `None` for `Host`/`Custom`
   (`:234-246`).

Fork changes **FC-6** and **FC-7**.

### 6.3 MCP — cleanly layered, minor blocker

`codex-mcp` is genuinely reusable. Public: `McpCatalogBuilder`
(`codex-rs/codex-mcp/src/catalog.rs:296`) with `register` (`:302`), `disable`
(`:308`), `build` (`:407`); `McpServerRegistration::from_config` (`:129`);
`ResolvedMcpCatalog` (`:494`); `McpConfig`
(`codex-rs/codex-mcp/src/mcp/mod.rs:120`, all fields public);
`McpRuntimeInput` (`codex-rs/codex-mcp/src/runtime.rs:67`);
`McpRuntime::{empty:166, new:236, replace:243}`; `McpBinding`
(`codex-rs/codex-mcp/src/binding.rs:31`); `effective_mcp_servers`
(`mcp/mod.rs:266`).

`McpServerConfig` (`codex-rs/config/src/mcp_types.rs:181`) is a plain public
struct — constructible in memory, no TOML. Two transports only, `Stdio` (`:516`)
and `StreamableHttp` (`:528`); **no SSE**. Inline `bearer_token` is rejected
(`codex-rs/config/src/mcp_edit.rs:33-49`).

`McpServerContributor` is **public**
(`codex-rs/ext/extension-api/src/contributors.rs:69`, verified) with
`McpServerContribution` (`codex-rs/ext/extension-api/src/contributors/mcp.rs:130`,
verified) whose `Set { name, config }` variant is precisely "VLOOP owns the
definition, Codex owns the connection".

**Blocker:** contributions *overlay* rather than replace.
`McpManager::runtime_config_with_context` (`codex-rs/core/src/mcp.rs:152`) always
starts from `Config::to_mcp_config_with_loaded_plugins`
(`codex-rs/core/src/config/mod.rs:1628`), registering config servers (`:1665`)
and plugin servers (`:1653`), plus an Apps compatibility registration
(`codex-rs/core/src/mcp.rs:246-260`). Fork change **FC-8** adds a
`ReplaceCatalog` variant.

Precedent for standalone use outside core:
`codex-rs/app-server/src/request_processors/apps_processor/installed.rs:88` and
`codex-rs/core/src/connectors.rs:257` both call `McpRuntime::new` directly.

### 6.4 Policy and approvals

Mostly already parameterized; the gap is construction, not plumbing.

`StepContext.approval_policy` / `approvals_reviewer`
(`codex-rs/core/src/session/step_context.rs:30,33`) come from
`turn_context.approval_policy()` (`codex-rs/core/src/session/turn_context.rs:240`)
and `config.approvals_reviewer`. `Permissions` lives in `Config`
(`codex-rs/core/src/config/mod.rs:283`, field `:598`) with
`Constrained<AskForApproval>` (`:285`) whose `set` enforces validators
(`codex-rs/config/src/constraint.rs:174`, `allow_only` `:107`).

`ToolOrchestrator::run` (`codex-rs/core/src/tools/orchestrator.rs:135`) takes
`approval_policy` as a parameter, resolves the requirement (`:168-170`), requests
approval (`:188-225`), selects the sandbox (`:229-279`) and retries on denial
(`:381-420`). `Session::request_approval`
(`codex-rs/core/src/tools/approvals.rs:485`) runs permission hooks (`:503-519`)
then routes guardian vs user (`:549-579`).

Injectable without fork changes: approval policy, reviewer and permission
profile, via `Permissions::from_approval_and_profile`
(`codex-rs/core/src/config/mod.rs:315`) or per-thread
`CodexThreadSettingsOverrides` (`codex-rs/core/src/codex_thread.rs:149`) applied
through `restore_thread_settings` (`:430`). Per-run environment policy flows as
data through the public `CodexThread::environment_ready` (`:712`) carrying
`EnvironmentConfig.exec_policy` / `.mcp_policy`
(`codex-rs/protocol/src/environment.rs:24-34`).

**Execpolicy is the sharp edge.** `ExecPolicyManager` is `pub(crate) struct`
(`codex-rs/core/src/exec_policy.rs:278`) with `pub(crate) fn new(Arc<Policy>)`
(`:295`) — both verified. `Session::spawn_internal` migrates prefix rules and
loads `*.rules` from disk (`codex-rs/core/src/session/mod.rs:539-568`), and
approvals write back (`exec_policy.rs:452`, `:502`). Meanwhile
`codex-execpolicy` itself is fully public and in-memory constructible:
`Policy::{new:72, empty:88, add_prefix_rule:128, merge_overlay:178, check:225}`,
`PolicyParser::{new:49, parse:57}`.

Fork changes **FC-3** and **FC-4**.

---

## 7. Fork change register

Ten changes, tiered by necessity. Per `codex-rs/AGENTS.md` ("resist adding to
`codex-core`"), new surface goes in new crates; core gets visibility changes and
behaviour gates only.

### Tier 0 — none

Prove the kernel boots with injected stores using only public API:
`Config::load_default_with_cli_overrides_for_codex_home` +
`StaticModelsManager` + a VLOOP `ThreadStore` + ephemeral `AuthManager` +
`ephemeral = true` + `state_db: None` + `agent_graph_store: None`.

### Tier A — required for sole ownership of state and policy

| ID | Change | Location | Size |
|---|---|---|---|
| **FC-1** | Make `load_config_with_layer_stack` public, or add `ConfigBuilder::from_parts(ConfigToml, ConfigOverrides, ConfigLayerStack, codex_home)` that skips `load_config_layers_state` | `codex-rs/core/src/config/mod.rs:3078`, `:1393` | 1 line + wrapper |
| **FC-2** | Injectable skills service / `bundled_skills_enabled` policy param so `install_system_skills` never runs | `codex-rs/core/src/thread_manager.rs:438-442`; `codex-rs/ext/skills/src/host_service.rs:132` | ~10 lines |
| **FC-3** | `ExecPolicyManager` + `new` public (or `pub fn from_policy`), plus a no-persist mode | `codex-rs/core/src/exec_policy.rs:278,295,452,502` | ~30 lines |
| **FC-4** | `SessionSpawnArgs` accepts a caller-supplied `ExecPolicyManager`; skip the disk-load branch | `codex-rs/core/src/session/mod.rs:414,547-567` | ~15 lines |
| **FC-9** | Gate MCP approval config write-backs | `codex-rs/core/src/mcp_tool_call.rs:2047,2099` | ~10 lines |

### Tier B — required for sole ownership of resources

| ID | Change | Location | Size |
|---|---|---|---|
| **FC-5** | New crate `codex-tool-policy` with `ToolSelection` / `ToolCatalogInput` / `ResourceSnapshotId`; apply in `finalize_tool_router` after `:324` via `entries_mut` / `remove` | new crate + `codex-rs/core/src/tools/spec_plan.rs:318`, `registry.rs:386,396` | new crate + ~60 lines |
| **FC-6** | `SkillCatalogProvider` + `SkillCatalogSnapshot`, blanket-adapted to `SkillProvider`; `SkillToolAuthority::Custom` | `codex-rs/ext/skills/src/provider.rs:71`; `tools/mod.rs:221,234,66-70` | ~80 lines |
| **FC-7** | Gate host skill discovery + the legacy injection path when a VLOOP snapshot is present | `turn_context.rs:861-884`; `session.rs:552`; `turn.rs:712-734,782-808` | ~40 lines |
| **FC-8** | `McpServerContribution::ReplaceCatalog { servers }`; honour it by starting from an empty builder and skipping plugin + Apps registration | `contributors/mcp.rs:130`; `codex-rs/core/src/mcp.rs:152,245-260` | ~40 lines |

### Tier C — optional

| ID | Change | Location | When |
|---|---|---|---|
| **FC-10** | `ModelClient::with_provider(SharedModelProvider, …)`; keep `new` delegating | `codex-rs/core/src/client.rs:430,445` | only if VLOOP needs custom provider behaviour |

### Explicitly avoid

- Publicizing `CoreToolRuntime` (`codex-rs/core/src/tools/registry.rs:55`) or the
  registry mutators — exports the whole hook/telemetry contract and freezes core
  internals.
- Publicizing `McpConnectionSet` (`codex-rs/codex-mcp/src/connection_manager.rs:179`),
  `McpBinding::new` (`binding.rs:53`) or `PreparedMcpCall::new` (`:189`) — these
  enforce the catalog-revision safety property VLOOP wants to reuse.
- Faking "no discovery" with an empty or temp `CODEX_HOME`. It still installs
  bundled skills (`host_service.rs:220`) and runs execpolicy migration
  (`session/mod.rs:552-561`), so the invariant cannot be asserted.
- Adding VLOOP-specific types to `codex-core`.

---

## 8. Orchestration and memory primitives

**Partial coverage.** The dedicated recon for this area did not complete (§13.1).
What is verified:

- `AgentGraphStore` is a public injectable trait
  (`codex-rs/agent-graph-store/src/store.rs:17`), passed as `Option` to
  `ThreadManager::new` (`thread_manager.rs:430`) and `None` when no state DB
  (`:409-415`). Spawn edges are written by
  `persist_thread_spawn_edge_for_source` (`codex-rs/core/src/agent/control.rs:761-771`),
  skipped under `ephemeral` (`:766`).
- Codex has its own multi-agent spawning in `codex-rs/core/src/agent/control.rs`,
  with `inherited_exec_policy` decided at `:689-706`.
- Memories are gated on `ephemeral`, `Feature::MemoryTool`, root-agent status and
  a state DB (`codex-rs/memories/write/src/start.rs:33-52`), and the startup task
  is invoked only from app-server
  (`codex-rs/app-server/src/request_processors/turn_processor.rs:597-605`).
  Driving `ThreadManager` directly therefore starts no memory task at all.
- `MemoriesConfig` is a `Config` field (`codex-rs/core/src/config/mod.rs:844`)
  with `generate_memories` / `use_memories` toggles
  (`codex-rs/config/src/types.rs:291-301`).

**Design consequence:** VLOOP owns orchestration and memory outright. Codex's
agent-spawning and memory subsystems are left disabled rather than adapted,
because both assume Codex-owned persistence. VLOOP reuses the *single-agent loop*
as its execution primitive and composes multi-agent behaviour above it. The open
spike (`tasks.md` S4) is whether `agent-graph-store` and `context-fragments` are
worth reusing as VLOOP-side libraries.

---

## 9. IDE substrate: reuse verdicts

Unchanged by the ownership inversion; these were validated independently.

### 9.1 Reusable as-is

| Need | Crate + API | Source |
|---|---|---|
| Interactive PTY | `codex-utils-pty` (§10) | `codex-rs/utils/pty/src/lib.rs:12-47` |
| Fuzzy quick-open | `codex-file-search`: `create_session`, `update_query` | `codex-rs/file-search/src/lib.rs:164-169,149` |
| File watching | `codex-file-watcher`: `new`, `add_subscriber`, `register_paths`, `DebouncedWatchReceiver` | `codex-rs/file-watcher/src/lib.rs:379,409,308,266` |
| Palette filtering | `codex-utils-fuzzy-match::fuzzy_match` | `codex-rs/utils/fuzzy-match/src/lib.rs:12` |
| Atomic + symlink-safe save | `write_atomically`, `resolve_symlink_write_paths` | `codex-rs/utils/path-utils/src/lib.rs:122,47` |
| Path boundary checks | `PathUri::starts_with`, `relative_path_from`; `AbsolutePathBuf::from_absolute_path_checked` | `codex-rs/utils/path-uri/src/lib.rs:324,358`; `utils/absolute-path/src/lib.rs:66` |

Two API traps: fuzzy score direction is **inverted** between `FileMatch.score`
(higher better, `codex-rs/file-search/src/lib.rs:326-339`) and `fuzzy_match`
(lower better, `codex-rs/utils/fuzzy-match/src/lib.rs:4-5`); and `file-watcher`
**discards event kinds**, delivering paths only
(`codex-rs/file-watcher/src/lib.rs:761-766,30-33`), so consumers must re-stat.

### 9.2 Must build new

| Gap | Evidence |
|---|---|
| Workspace full-text search | `file-search` matches paths only (`match_paths()` `codex-rs/file-search/src/lib.rs:189`; `IndexedEntry` `:71-74`). No grep engine in `Cargo.lock` (verified: 0 matches for `grep-searcher`/`ripgrep`). `ignore 0.4.23` available (`codex-rs/Cargo.toml:344`) |
| Git per-file status | `get_has_changes_in_repo` collapses porcelain to `Option<bool>` (`codex-rs/git-utils/src/status.rs:29-41`) |
| Git worktree / staged diff | absent from the public surface (`codex-rs/git-utils/src/lib.rs:16-50`) |
| Git commit, unstage, branch create/checkout | absent |
| Path-based staging | `stage_paths` is diff-driven and returns `Ok` on non-zero exit (`codex-rs/git-utils/src/apply.rs:326-349`) |
| File rename/move | no `rename` on `ExecutorFileSystem` (`codex-rs/file-system/src/lib.rs:412-507`) |

**Git decision.** `codex-git-utils` shells out to `git`
(`codex-rs/git-utils/src/operations.rs:117`) but injects
`-c core.hooksPath=/dev/null` (`:104-112`, verified), so **user commit hooks do
not run**, and `run_git_for_stdout` is `pub(crate)` (`:76-92`). For an IDE where
the user commits their own work, silently skipping hooks is wrong. VLOOP invokes
`git` itself and reuses only `get_git_repo_root`
(`codex-rs/git-utils/src/info.rs:35`) and `local_git_branches` (`:844`).

### 9.3 Not reused

- **`codex-file-system` / `codex-exec-server` FS impls** — trait-only crate whose
  impls live in `exec-server`; `PathUri` boundary conflicts with the `PathBuf`
  used by search/watcher/git; `walk` caps at 50k entries / 4 MiB
  (`codex-rs/file-system/src/lib.rs:39-40`). VLOOP uses `tokio::fs` behind a
  root guard.
- **`codex-apply-patch`** — Codex applies patches during a turn; VLOOP renders
  diffs with `similar`, already a workspace dep
  (`codex-rs/git-utils/Cargo.toml:25`).
- **`codex-terminal-detection`** — sniffs the *host* emulator
  (`codex-rs/terminal-detection/src/lib.rs:25-54`). VLOOP *is* the emulator.

---

## 10. Terminal

`codex-utils-pty` (`codex-rs/utils/pty/Cargo.toml:4`) is directly reusable:
`portable-pty` plus a bundled ConPTY path on Windows
(`codex-rs/utils/pty/src/pty.rs:113-123`), depending only on `anyhow`,
`portable-pty`, `tokio` and platform crates — **no `codex-core`, no event bus.**

```rust
pub async fn spawn_pty_process(program, args, cwd, env, arg0,
                               size: TerminalSize, inherited_fds: &[i32])
    -> anyhow::Result<SpawnedProcess>;                       // pty.rs:127-135

impl ProcessHandle {                                          // verified:
    pub fn writer_sender(&self) -> mpsc::Sender<Vec<u8>>;    // process.rs:163
    pub fn resize(&self, size: TerminalSize) -> Result<()>;  // :186
    pub fn signal(&self, ProcessSignal) -> io::Result<()>;   // :229
    pub fn terminate(&self);                                 // :246
}
pub const DEFAULT_OUTPUT_BYTES_CAP: usize = 1024 * 1024;     // lib.rs:12
```

`Drop` calls `terminate()` (`process.rs:273-277`). Output is raw `Vec<u8>` with
no ANSI processing — what xterm.js wants. The crate `README.md:7-9` is **stale**
(omits `inherited_fds`); trust the source.

Reference design for multi-session management:
`codex-rs/app-server/src/command_exec.rs` — session map (`:49-51`) driven by a
`CommandControl { Write, Resize, Terminate }` channel (`:76-85`). It is
`pub(crate)`; copy the pattern.

**exec-server is not an option for a browser terminal:** no `resize` anywhere in
its protocol (`codex-rs/exec-server-protocol/src/protocol.rs:213-245`), and its
WS listener returns `403` for **any** request carrying an `Origin` header
(`codex-rs/exec-server/src/server/transport.rs:202-216`, verified) — which every
browser sends. Its `ExecProcessEventLog`
(`codex-rs/exec-server/src/process.rs:63-159`) is still worth copying: a replay
buffer bounded by **both** event count and retained bytes (`:130-139`).

---

## 11. Sandbox and secrets

### 11.1 Three policy layers

| Layer | Type | Source |
|---|---|---|
| User-facing | `SandboxMode { ReadOnly, WorkspaceWrite, DangerFullAccess }` | `codex-rs/protocol/src/config_types.rs:86-96` |
| Effective | `SandboxPolicy` | `codex-rs/protocol/src/protocol.rs:1003-1051` |
| Mechanism | `SandboxType { None, MacosSeatbelt, LinuxSeccomp, WindowsRestrictedToken }` | `codex-rs/sandboxing/src/manager.rs:37-42` |

Selection is `get_platform_sandbox` (`:62-72`). Linux requires shipping the
`codex-linux-sandbox` helper binary (`codex-rs/sandboxing/src/lib.rs:65-67`).
`WritableRoot` carries read-only subpaths so `.codex` and `.git/hooks` stay
unwritable inside a writable root (`codex-rs/protocol/src/protocol.rs:1053-1060`).

**In-process embedding preserves sandboxing.** The sandbox wraps the *spawned
command*, not the agent loop, so VLOOP inherits enforcement unchanged.

### 11.2 Secrets — sanctioned mechanisms

1. **Provider keys:** VLOOP resolves credentials through `ExternalAuth` (§5) and
   leaves `env_key` / `env_http_headers` unset.
2. **MCP HTTP auth:** `bearer_token_env_var` only; inline tokens are rejected
   (`codex-rs/config/src/mcp_edit.rs:33-49`). Avoid `McpServerAuth::OAuth` — no
   in-memory token store exists (`codex-rs/config/src/types.rs:123-135`).
3. **VLOOP-owned secrets:** `codex_secrets::SecretsManager`
   (`codex-rs/secrets/src/lib.rs:98-157`), keyring-backed, scoped via
   `SecretScope::Environment(environment_id_from_cwd(cwd))` (`:53-56,159-180`).
4. **Never read `$CODEX_HOME/auth.json`.** It may hold a plaintext key protected
   only by `0o600` (`codex-rs/login/src/auth/storage.rs:40-61,202-219`).
5. Apply `redact_secrets` (`codex-rs/secrets/src/lib.rs:21`) before logging or
   displaying command output.

### 11.3 Observability

`[otel]` supports `OtlpGrpc` / `OtlpHttp` with custom endpoint, headers and mTLS
(`codex-rs/otel/src/config.rs:88-105,81-85`). `span_attributes` and `tracestate`
(`:60-61`) let VLOOP tag spans with a run id. Structured events go to
`codex_otel.log_only` and `codex_otel.trace_safe`
(`codex-rs/otel/src/targets.rs:1-11`). In-process embedding means VLOOP can
install its own `tracing` subscriber and capture these directly.

`codex-rs/diagnostics` is a **process-metrics** crate (gauges, RSS), not config
diagnostics (`codex-rs/diagnostics/src/lib.rs:76-106`).

---

## 12. Repository conventions VLOOP must honour

From `codex-rs/AGENTS.md` and `justfile`:

- **Never run `cargo test` directly.** Use `just test`, scoped `-p <crate>`
  (nextest-backed).
- Run `just fmt` after changes; `just fix -p <crate>` before finalizing.
- **Resist adding to `codex-core`.** Prefer new crates; core gets visibility
  changes and behaviour gates only.
- Edition 2024; inline `format!` args; collapse `if`s; method references over
  closures; exhaustive `match`.
- No `#[async_trait]`, no `#[allow(async_fn_in_trait)]`. Use RPITIT with explicit
  `Send` bounds: `fn foo(&self) -> impl Future<Output = T> + Send;`
- Avoid bool / ambiguous `Option` parameters; use `/*param_name*/` comments for
  unavoidable positional literals.
- Modules under 500 LoC; new modules rather than growing files past ~800.
- New test modules in sibling files: `#[path = "..._tests.rs"] mod tests;`
- `pretty_assertions::assert_eq`, whole-object comparisons.
- **Integration tests over unit tests for agent logic**; `core/suite` with
  `test_codex` / `TestCodexBuilder::build_with_auto_env()`.
- **Any change to `codex-rs/Cargo.toml` or `Cargo.lock` requires
  `just bazel-lock-update`**, and new crates need `BUILD.bazel`. CI verifies
  lockfile drift. This is the standing cost of the fork changes in §7.
- If `ConfigToml` changes, run `just write-config-schema`.
- Do not add product docs to `codex-rs/docs/`. VLOOP docs live under `vloop/`.

---

## 13. Unverified — do not treat as fact

1. **Orchestration / learning primitives recon did not complete** (rate limit).
   `agent-graph-store`, `ext/agent`, `context-fragments`, `memories/*`,
   `ext/memories`, `hooks`, `analytics`, `rollout-trace` were not systematically
   read. §8 is partial. Resolved by `tasks.md` S4.
2. **Kernel-facade recon did not complete** (rate limit); §1 was verified
   directly by me instead, but only for `ThreadManager::new`, `CodexThread`'s
   public method list, and the approval ops/enum. The full `Op` variant set and
   `EventMsg` variant set were **not** enumerated. Resolved by `tasks.md` S2.
3. **Cross-workspace path dependency on `codex-core`.** Whether a foreign Cargo
   workspace can path-depend on `codex-core` and resolve its
   `version.workspace = true` inheritance and full feature graph is unproven, and
   is now far more load-bearing than in the previous revision. `tasks.md` GATE-0.
4. `merge_configured_model_providers` / `validate_model_providers` bodies not
   read; "built-in IDs cannot be overridden" rests on the doc comment
   (`codex-rs/config/src/config_toml.rs:283-284`) and the call site
   (`codex-rs/core/src/config/mod.rs:3629-3631`). Enforcement semantics — error
   vs silent ignore — unconfirmed.
5. `ConfigLayerStack::effective_config()` behaviour for an empty layer vector not
   read. The §2.4 conclusion (bundled skills default enabled) is an **inference**
   from `codex-rs/config/src/skills_config.rs:131-136`. Confirm before relying.
6. Whether `resolve_root_git_project_for_trust`
   (`codex-rs/core/src/config/mod.rs:3294`) can be short-circuited.
7. `SkillLoadOutcome::from_parts` is `pub(crate)`
   (`codex-rs/ext/skills/src/host_outcome.rs:30`); whether an external caller can
   build an empty `HostSkillsSnapshot` is unconfirmed — relevant to FC-7.
8. Guardian interaction with a VLOOP-owned approval policy
   (`codex-rs/core/src/guardian/review.rs:327`) not traced.
9. Subagent inheritance of VLOOP snapshots: `inherited_exec_policy`
   (`codex-rs/core/src/session/mod.rs:414`) and `dynamic_tools` propagation
   (`codex-rs/core/src/session/review.rs:168`) suggest snapshots must be
   explicitly inherited; spawn paths not enumerated.
10. Analytics / OTEL / feedback network egress not audited beyond noting
    `Config.analytics_enabled` (`:1029`), `feedback_enabled` (`:1033`),
    `otel` (`:1039`). Open data-egress review item.
11. `codex-state` internals (schema, WAL, file-creation timing) not audited;
    only that `state_db: None` avoids constructing it.
12. Plugin marketplace writes under `CODEX_HOME/plugins`
    (`PluginsManager::new_with_options`,
    `codex-rs/core-plugins/src/manager.rs:512`) — write set not enumerated.
    Note `ThreadManager::new` constructs a `PluginsManager` unconditionally
    (`codex-rs/core/src/thread_manager.rs:445`).
13. Package name of `codex-rs/utils/path-utils` — contents verified, declared
    `[package] name` not.
14. **No builds or tests were run.** All findings are from source reading.
