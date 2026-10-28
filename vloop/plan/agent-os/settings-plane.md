# VLOOP Settings Plane — migrating Codex off environment variables

Companion documents: `codex-integration.md` (Codex evidence base, cited `[CI §n]`),
`persistence-plane.md` (injected ORM persistence, cited `[PP §n]`), `plan.md`
(architecture), `spec.md` (requirements), `tasks.md` (DAG).

This document is both the evidence base and the plan for one workstream: moving
Codex's configuration inputs from process environment variables to VLOOP-owned,
remotely managed, dynamically resolved values.

All findings below were read from source directly at the pinned commit.

---

## 1. The problem, stated precisely

Environment variables are a **process-global, untyped, unversioned, single-valued**
namespace. A multi-agent orchestrator needs configuration that is **per-agent,
typed, versioned, and multi-valued**. These are irreconcilable, and the gap is not
stylistic — it produces four concrete defects (§3).

The requirement is therefore not "read settings from a database instead of env."
It is: **VLOOP resolves a typed, versioned settings snapshot per run, and
materializes it into Codex through constructor arguments and child-process
environments — never through the host process environment.**

---

## 2. Measured inventory

Reproduce with:

```sh
# occurrences, all vs non-test
grep -rn --include='*.rs' -E 'env::var(_os)?\(|env::vars\(' codex-rs | wc -l
grep -rn --include='*.rs' -E 'env::var(_os)?\(|env::vars\(' codex-rs \
  | grep -v '/tests/' | grep -v '_tests.rs' | grep -v 'test_support' | wc -l

# declared name constants
grep -rn --include='*.rs' -E 'const [A-Z0-9_]+(_ENV_VAR|_ENV|_VAR|_ENV_KEY): &str *=' codex-rs

# mutation sites
grep -rn --include='*.rs' -E 'env::(set_var|remove_var)' codex-rs
```

| Measure | Count |
|---|---|
| `env::var` / `env::var_os` / `env::vars` occurrences, all | **398** |
| the same, excluding `tests/`, `*_tests.rs`, `test_support` | **294** |
| declared env-var *name* constants | **~90** |
| `env::set_var` / `remove_var`, all | **52** |
| the same, outside test paths | **29** |

Most-referenced branded literals: `OPENAI_API_KEY` 129, `CODEX_HOME` 91,
`CODEX_CA_CERTIFICATE` 19, `CODEX_MCP_PROTOCOL_VERSION` 14,
`CODEX_AUTHAPI_BASE_URL` 13, `CODEX_ACCESS_TOKEN` 12, `CODEX_REMOTE_AUTH_TOKEN` 11,
`RUST_LOG` 10, `CODEX_API_KEY` 8, `CODEX_SQLITE_HOME` 7.

### 2.1 Non-test reads, restricted to crates VLOOP embeds

| Crate | Reads | Crate | Reads |
|---|---|---|---|
| `login` | 15 | `core-plugins` | 3 |
| `exec-server` | 11 | `core` | 3 |
| `rmcp-client` | 9 | `network-proxy` | 3 |
| `codex-mcp` | 6 | `http-client` | 2 |
| `protocol` | 5 | `git-utils` | 2 |
| `sandboxing` | 4 | `process-hardening` | 1 |
| `model-provider-info` | 4 | `model-provider` | 1 |
| `otel` | 3 | `arg0` | 1 |

**≈73 reads** across the embedded surface — not 294. That is the number this plan
must actually address, and it is tractable.

`utils/pty` reports 18 but they are concentrated in `src/tests.rs`, which the
filter above misses (it matches `_tests.rs`, not `tests.rs`). Treat as
test-inflated.

---

## 3. Four failure modes, with evidence

### (a) Process-global credentials break per-agent providers

```rust
// codex-rs/model-provider-info/src/lib.rs:331-347  (verified verbatim)
pub fn api_key(&self) -> CodexResult<Option<String>> {
    match &self.env_key {
        Some(env_key) => {
            let api_key = std::env::var(env_key)
                .ok()
                .filter(|v| !v.trim().is_empty())
                .ok_or_else(|| CodexErr::EnvVar(EnvVarError { /* … */ }))?;
            Ok(Some(api_key))
        }
        None => Ok(None),
    }
}
```

One process holds one value per variable name. Two agents orchestrated
concurrently against the same provider with different keys, quotas, or tenants is
**unrepresentable**.

### (b) The obvious workaround is undefined behaviour

The workspace is `edition = "2024"` (`codex-rs/Cargo.toml:145`), where
`std::env::set_var` is `unsafe`. Setting a variable per agent at runtime is a data
race the moment more than one thread exists.

Codex's own non-test mutations are justified *by single-threadedness*:

> `// It is safe to call set_var() because our process is`
> `// single-threaded at this point in its execution.`
> — `codex-rs/arg0/src/lib.rs:315-318`

That justification does not survive a multi-agent host. **Per-agent env mutation
is not an available strategy**, which is why §5 routes everything through
constructor arguments and child environments instead.

Note the asymmetry that makes the design work: mutating *our* environment is
unsafe; setting a *child's* environment via `Command::env` is not. Divergence is
achievable at the child boundary.

### (c) Startup silently rewrites the environment

- `codex-rs/arg0/src/lib.rs:310-320` loads `.env` files through `dotenvy` into the
  process environment.
- `codex-rs/process-hardening/src/lib.rs:126-131` removes variables by prefix.

The environment VLOOP observes is not the environment it was launched with. Any
reasoning about "what is set" is unreliable by construction.

### (d) Auth precedence has an ungated env path

In `load_auth` (`codex-rs/login/src/auth/manager.rs:1425-1490`), the API-key read
is gated on `enable_codex_api_key_env`, but the `CODEX_ACCESS_TOKEN` read is gated
only on `auth_mode_is_allowed(.., AuthMode::AgentIdentity)`. A stray variable in
the environment can outrank a credential VLOOP injected deliberately.

---

## 4. Seams that already exist

Three, which is why this migration is mostly mechanical.

**Child-process env is already parameterized.**

```rust
// codex-rs/protocol/src/shell_environment.rs:61-68
pub fn create_env_from_vars<I>(
    vars: I, policy: &ShellEnvironmentPolicy, thread_id: Option<&str>,
) -> HashMap<String, String>
where I: IntoIterator<Item = (String, String)>
```

`create_env` (`:54-59`) is merely `create_env_from_vars(std::env::vars(), ..)`.
Non-test call sites on the process-env path: `core/src/exec_env.rs:35`,
`core/src/tasks/user_shell.rs:162`,
`core/src/tools/handlers/shell/shell_command.rs:110`,
`core/src/unified_exec/process_manager.rs:1221`,
`exec-server/src/local_process.rs:684`, `cli/src/debug_sandbox.rs:260`,
`app-server/src/request_processors/command_exec_processor.rs:152`. Only
`rmcp-client/src/stdio_server_launcher.rs:745` already uses the injectable form.

**Credentials are already injectable.** `ExternalAuth` +
`AuthCredentialsStoreMode::Ephemeral` `[CI §5]` lets VLOOP resolve credentials
without touching disk or env.

**Brokered values are an established in-repo pattern.** `network-proxy`'s
credential broker holds real credentials in-process, hands **dummy** values to
children via env, and substitutes at the proxy layer
(`codex-rs/network-proxy/src/credential_broker.rs:10-30`).

**Scrubbing must be preserved.** `NON_INHERITABLE_ENV_VARS` and
`scrub_non_inheritable_env_vars` (`codex-rs/protocol/src/shell_environment.rs:14-50`)
already strip identity and auth-token variables from model-reachable children. The
materializer in §5 layers on top of this, never around it.

---

## 5. Target architecture

```text
Remote settings service / managed PostgreSQL settings repository
(source of truth, versioned)
        │  authenticated pull, ETag/version
        ▼
VLOOP local durable cache (through the injected persistence/cache adapter)
        │  last-known-good, per-key TTL
        ▼
SettingsResolver — scope precedence, typing, validation
        │
        ▼
SettingsSnapshot  (immutable, version-stamped, pinned into ResourceSnapshot)
        │
        ├──► Codex constructor arguments (Config, AuthManager, ModelsManager, …)
        ├──► child-process env maps via create_env_from_vars  (per child)
        └──► credential leases (short-lived, never persisted in plaintext)

                 ✗ never std::env::set_var in the host process
```

### 5.1 Scope precedence

```text
global  →  workspace  →  agent  →  agent_version  →  run override
```

Later wins. The resolved value **and its origin scope** are recorded in the
snapshot, so "why did this run behave that way" is answerable.

### 5.2 Key classes, with distinct failure semantics

This distinction is the core of the design. Treating all settings alike is how a
remote settings service becomes an outage amplifier.

| Class | Examples | Cacheable | On resolution failure |
|---|---|---|---|
| **Credential** | provider API keys, MCP bearer tokens, `CODEX_ACCESS_TOKEN` equivalents | plaintext **never** persisted; short-lived lease only | **fail closed** — run does not start |
| **Endpoint** | `CODEX_AUTHAPI_BASE_URL`, `OPENAI_BASE_URL`, proxy URLs | yes | fail open to last-known-good; warn |
| **Behavioural** | `CODEX_MCP_PROTOCOL_VERSION`, retry counts, timeouts, feature toggles | yes | fail open to compiled default |
| **Trust material** | `CODEX_CA_CERTIFICATE`, pinned certs | yes, integrity-checked | **fail closed** |
| **Policy ceiling** | sandbox mode, approval policy, writable roots | yes | **fail closed to most restrictive** |
| **Host-local** *(not migrated)* | `CODEX_HOME`, `PATH`, `TMPDIR`, `TERM`, `RUST_LOG`, `CODEX_SANDBOX*` | n/a | remains environment-derived |

### 5.3 What deliberately does **not** move

Being explicit, because over-migrating is the likelier failure than
under-migrating:

- **Machine-local paths** (`CODEX_HOME`, `TMPDIR`, `PATH`) describe the host, not
  policy. A remote value would break portability and, for `TMPDIR`, sandbox
  correctness.
- **`CODEX_SANDBOX` / `CODEX_SANDBOX_NETWORK_DISABLED`** are set *by* the sandbox
  for its children and read by tests to self-skip `[CI §11.1]`. Repo rules forbid
  modifying code related to these, and they are an output of enforcement, not an
  input to it. Untouched.
- **OS/terminal variables** (`TERM`, locale, `COLORTERM`) are genuinely
  environmental.
- **Tooling variables** (`RUST_LOG`, Bazel runfiles, `CARGO_*`) belong to the
  developer's shell.

### 5.4 Version pinning is the actual answer to the stated risk

The concern raised was that values "might change and cause failures in the
future." Remoteness alone makes that *worse*, not better — a remote value can
change under a running orchestration.

The mitigation is pinning, not location: **a run resolves settings once, stamps
`settings_version` into its `ResourceSnapshot`, and reads nothing further.** A
mid-flight remote change cannot affect an in-flight run, and a completed run can
be replayed against the exact settings it used. This composes with the existing
snapshot mechanism in `plan.md §7` rather than adding a parallel one.

### 5.5 Security constraints on remote settings

Remote configuration that can influence sandboxing or trust material is a
privilege-escalation channel. Three non-negotiables:

1. **The remote plane may only narrow policy, never broaden it.** A local ceiling
   is configured on the host; remote values intersect with it. This extends
   `plan.md §10`'s narrowing rule to the settings source.
2. **Trust material is integrity-checked** before use; a CA bundle that fails
   validation fails the run rather than falling back to system roots.
3. **Credentials are leased, not stored.** Plaintext never lands in the local
   cache or in VLOOP's database — only references, matching `[CI §11.2]`.

Transport is authenticated and the response is version-stamped; an unauthenticated
or unverifiable settings response is a fail-closed condition for credential, trust
and policy classes.

### 5.6 Offline behaviour

VLOOP is local-first, so a network dependency for basic operation is unacceptable.
The local cache is **durable and usable when the remote is unreachable**, but it
is not a second authoritative settings store: it serves only versioned
last-known-good values under the failure rules in §5.2. Behavioural and endpoint
keys fall open to last-known-good, while credential, trust and policy keys fail
closed. A run started fully offline with a warm cache and a valid credential lease
must succeed; this is a tested requirement, not best-effort behaviour. Managed
mode never dual-writes orchestration state into this cache (`persistence-plane.md`
§9).

---

## 6. Interfaces

In `vloop-settings` — a new VLOOP crate, not a Codex one. RPITIT with explicit
`Send` bounds, per repo convention `[CI §12]`.

```rust
/// Typed identity of a setting. Variants carry their class (§5.2), so failure
/// semantics are a property of the key rather than of each call site.
pub enum SettingKey {
    Credential(CredentialKey),
    Endpoint(EndpointKey),
    Behavioural(BehaviouralKey),
    TrustMaterial(TrustKey),
    PolicyCeiling(PolicyKey),
}

/// Source of truth for settings. The remote service and the local cache are both
/// implementations; the resolver composes them.
pub trait SettingsSource: Send + Sync {
    fn fetch(
        &self,
        scope: &ScopeSelector,
    ) -> impl Future<Output = Result<ScopedSettings, SettingsError>> + Send;
}

/// Resolves scope precedence, validates types, and freezes a snapshot.
pub trait SettingsResolver: Send + Sync {
    fn resolve(
        &self,
        scope: &ScopeSelector,
    ) -> impl Future<Output = Result<SettingsSnapshot, SettingsError>> + Send;
}

/// Immutable, version-stamped. Pinned into `ResourceSnapshot` (§5.4).
pub struct SettingsSnapshot {
    pub settings_version: SettingsVersion,
    pub resolved: BTreeMap<SettingKey, ResolvedValue>, // value + origin scope
    pub resolved_at: DateTime<Utc>,
}

/// Produces a child-process environment from a snapshot plus host-local
/// variables. The only place VLOOP composes an environment map.
pub trait EnvMaterializer: Send + Sync {
    fn materialize(
        &self,
        snapshot: &SettingsSnapshot,
        host: &HostEnvironment,
        policy: &ShellEnvironmentPolicy,
    ) -> HashMap<String, String>;
}

/// Short-lived credential handed to exactly one consumer.
pub struct CredentialLease {
    pub value: SecretString,   // zeroized on drop
    pub expires_at: DateTime<Utc>,
}
```

`EnvMaterializer` output feeds `create_env_from_vars` (§4) and then
`scrub_non_inheritable_env_vars`, preserving existing protections.

---

## 7. Fork changes

Extends the register in `[CI §7]`. Same hygiene rules as `plan.md §15`: new crates
over `codex-core` growth, behaviour gates default to current behaviour, each change
individually reversible.

| ID | Change | Location | Tier |
|---|---|---|---|
| **FC-11** | Redirect the seven `create_env` call sites to `create_env_from_vars` with a supplied var map; keep `create_env` for existing callers | `core/src/exec_env.rs:35`, `core/src/tasks/user_shell.rs:162`, `core/src/tools/handlers/shell/shell_command.rs:110`, `core/src/unified_exec/process_manager.rs:1221`, `exec-server/src/local_process.rs:684`, `app-server/src/request_processors/command_exec_processor.rs:152`, `cli/src/debug_sandbox.rs:260` | A |
| **FC-12** | `ModelProviderInfo::api_key()` accepts an injected credential resolver instead of reading env; env path retained as fallback when none supplied | `model-provider-info/src/lib.rs:331-347` | A |
| **FC-13** | Gate the `CODEX_ACCESS_TOKEN` read on `enable_codex_api_key_env` (or a new explicit flag), closing the precedence leak §3(d) | `login/src/auth/manager.rs:1460-1470` | A |
| **FC-14** | CA/trust material accepted as bytes from a caller, not only via `CODEX_CA_CERTIFICATE` / `SSL_CERT_FILE` | `http-client/src/custom_ca.rs:61-62` | B |
| **FC-15** | MCP protocol version, startup timeouts and endpoint overrides accepted from config/caller rather than env | `codex-mcp/src/mcp/mod.rs:64`, `rmcp-client/src/utils.rs` | B |
| **FC-16** | Optional: a `HostEnvironment` abstraction in a new `codex-host-env` crate so embedded crates read from an injected view rather than `std::env` | new crate + call sites in `login`, `exec-server`, `otel` | C |

FC-16 is deliberately last. It is the "correct" general fix but touches the most
code; FC-11 through FC-13 remove the multi-agent defects on their own, and the
remaining reads are then either host-local (§5.3) or low-risk behavioural.

---

## 8. Migration waves

Each wave ends with the env-read ratchet (§9) lowered and the isolation suite
extended.

**Wave 0 — measure and fence.** Land the ratchet test at the current count.
No behaviour change. Establishes that the number cannot silently grow.

**Wave 1 — credentials (removes defect (a) and (d)).** `vloop-settings` with
credential class only; `ExternalAuth` bridge; FC-12 and FC-13. Proves two agents
running concurrently against one provider with different keys.

**Wave 2 — child environments (removes defect (c) for agent-visible processes).**
FC-11 plus `EnvMaterializer`. Child environments become a pure function of
`(snapshot, host-local, policy)`.

**Wave 3 — endpoints, trust, behavioural.** FC-14 and FC-15. Remote endpoint and
protocol-version control, integrity-checked CA material.

**Wave 4 — residual reads.** FC-16 if the remaining surface justifies it;
otherwise document each remaining read as host-local by design.

---

## 9. Verification

The invariants are silent when broken, so each needs a test rather than a
convention.

1. **Hermetic-env test.** A full run executes with the process environment reduced
   to a minimal host-local allowlist. Any credential, endpoint or trust value
   arriving from env fails the test. This is the primary guard.
2. **Per-agent divergence test.** Two agents in one process, same provider,
   different credentials, executing concurrently — both succeed and neither
   observes the other's value. Directly falsifies defect (a).
3. **No-host-mutation test.** `std::env::set_var` / `remove_var` do not appear in
   VLOOP crates. Enforced by CI grep, since edition 2024 makes these `unsafe` and
   the failure is a race rather than a compile error §3(b).
4. **Env-read ratchet.** A CI check asserts the non-test env-read count in the
   embedded crates does not exceed a recorded ceiling, lowered each wave. Prevents
   regrowth.
5. **Offline test.** Run to completion with the remote unreachable and a warm
   cache; behavioural and endpoint keys fall back, credential keys fail closed if
   no valid lease §5.6.
6. **Snapshot pinning test.** Change a remote value mid-run; the running attempt is
   unaffected. Replay the run afterwards; it uses the original
   `settings_version` §5.4.
7. **Narrowing test.** A remote policy value attempting to broaden sandbox or
   approval policy beyond the local ceiling is rejected §5.5.
8. **Scrubbing-preserved test.** `NON_INHERITABLE_ENV_VARS` are still absent from
   materialized child environments §4.

---

## 10. Risks and unverified items

1. **Inventory precision.** §2 counts are `grep`-based: they include commented-out
   code and doc strings, and miss dynamically constructed variable names. The
   `utils/pty` figure is test-inflated §2.1.
2. **`ShellEnvironmentPolicy` semantics not fully read.** §4 verifies the
   `create_env_from_vars` signature and its call sites, not the policy's
   set/include/exclude precedence. FC-11's correctness depends on it; read before
   implementing.
3. **Transitive env reads inside dependencies** (`reqwest` proxy variables,
   `rustls` cert discovery, `dotenvy`) are not covered by an audit of `codex-rs`
   alone. The hermetic-env test §9.1 is what actually catches these.
4. **A remote settings service is a new availability and security dependency** for
   a local-first product. §5.2 and §5.6 bound the blast radius, but the tradeoff is
   real and should be an explicit product decision, not a silent one.
5. **`OPENAI_API_KEY` has 129 references**, many in tests and fixtures. Migrating
   the non-test subset without breaking Codex's own suite needs the fallback path
   in FC-12 to remain functional.
6. **Whether Codex's own tests depend on env values** that FC-11..FC-13 would
   change is unverified. The non-regression gate (`spec.md §15`) covers it, but
   expect iteration.
7. **No builds or tests were run** during this research; all findings are from
   source reading.
