# CLAUDE.md — InferaDB KMS

## Project Overview

InferaDB KMS is the trust root for the InferaDB platform — a standalone, internal-only credentials, certificates, and key-management service that ledger, the ledger SDK, control, and engine consume as clients. It sits at the bottom of the trust chain: KMS has **no runtime dependency on any other InferaDB service**.

KMS deploys as a federation of regional clusters: a **GLOBAL** cluster acts as the federation root (cluster-wide trust anchors, system policies, the audit-chain anchor, the root issuer); per-region **REGIONAL** clusters hold intermediate CAs (cross-signed by GLOBAL), per-region transit keys, KV secrets, and the regional bootstrap-token kill-list. There is no cross-region Raft replication; federation is identity-only. This pattern mirrors `inferadb/ledger`'s `Region`/multi-Raft architecture verbatim.

The full design lives in `docs/superpowers/specs/2026-05-05-inferadb-kms-design.md`. The v1.x feature roadmap (procurement-gated additions) lives alongside it.

## Tech Stack

| Layer                  | Technology                                                                   | Version / Notes                                                                                                                                  |
| ---------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Language               | Rust                                                                         | 1.92 (2024 edition); pinned `+1.92` for build/clippy/test, `+nightly` for fmt                                                                    |
| Consensus              | `inferadb-consensus` (extracted from ledger to `inferadb/common`)            | No openraft. Custom in-house multi-shard Raft with deterministic simulation harness.                                                             |
| Wire protocol          | `inferadb-wire` + `inferadb-wire-transport` (extracted to `inferadb/common`) | QUIC via `quinn` 0.11; rustls 0.23 (`ring` default; `aws-lc-rs` under `--features fips`); postcard serialization; `inferadb-wire-macro` codegen. |
| Storage                | Custom encrypted B-tree, AES-256-GCM segmented WAL                           | Derived from ledger's `crates/store/` and `crates/consensus/src/wal/`.                                                                           |
| Errors (server crates) | `snafu`                                                                      | `types`, `store`, `state`, `raft`, `services`, `server`, engines, auth modules.                                                                  |
| Errors (SDK)           | `thiserror`                                                                  | `inferadb-kms-sdk` only — consumer-facing types.                                                                                                 |
| Builders               | `bon`                                                                        | Both `#[derive(bon::Builder)]` and `#[bon::bon] impl`.                                                                                           |
| Task runner            | `just`                                                                       | `just --list` is the source of truth.                                                                                                            |

## Repo Structure

```text
creds/                           # to be renamed `kms/` in Phase 1 of implementation
├── CLAUDE.md                    # You are here.
├── AGENTS.md                    # Symlink → CLAUDE.md.
├── Justfile                     # Every command. Run `just --list`.
├── crates/
│   ├── types/                   # Newtype IDs, errors, config schemas.
│   ├── store/                   # Encrypted B-tree page store.
│   ├── state/                   # Application state (mounted engines, sealed/unsealed).
│   ├── raft/                    # KMS Raft orchestration over inferadb-consensus.
│   ├── services/                # gRPC service implementations (Sys, Auth, Kv, Transit, Pki, Lease, Policy, Audit).
│   ├── server/                  # `inferadb-kms` binary. Subcommands: init, server, admin, pki, token.
│   ├── sdk/                     # `inferadb-kms-sdk` Rust client (the one crate using thiserror).
│   ├── agent/                   # `inferadb-kms-agent` operator-side renewer binary.
│   ├── test-utils/              # Cluster harness, fixtures, simulation seeds.
│   ├── engines/{kv,transit,pki}/  # Per-engine implementations.
│   ├── auth/{token,mtls}/       # Authentication backends.
│   ├── keyprovider/{file,aws-kms,gcp-kms}/  # Master-key bootstrap providers.
│   ├── audit/{file,syslog,stdout}/  # Audit sinks.
│   ├── lease/                   # Central lease manager.
│   └── policy/                  # Path-based policy engine.
└── proto/                       # Wire-service definition source.
```

Each crate has its own `CLAUDE.md` (symlinked to `AGENTS.md`) extending — never relaxing — the golden rules below.

---

## GOLDEN RULES

**Non-negotiable. Every agent, contributor, and reviewer follows these. If a rule looks wrong, raise it explicitly — never silently violate.**

1. **Never run state-mutating `git` commands from an agent.** `.claude/settings.json`'s `PreToolUse` hook blocks `git commit`, `git push`, `git stash`, `git tag`, `git rebase`, `git reset`, `git checkout`, `git merge`, and other state-mutating verbs. The human operator handles all git operations manually when work is ready. Read-only commands (`git status`, `git log`, `git diff`, `git show`, `git blame`) are permitted.

2. **Never hand-edit `Cargo.lock`.** It is generated by cargo. The `PreToolUse` hook blocks edits. Modify `Cargo.toml` and run `cargo +1.92 build --workspace` (or `cargo update -p <crate>`) to regenerate.

3. **KMS does not depend on `inferadb-ledger-*` or `inferadb-control-*` at compile time or at runtime.** Enforced via a `cargo deny` rule in `kms/.deny.toml`. CI fails the build on any direct or transitive dependency reaching into ledger or control crates. Shared infrastructure (wire, consensus) lives in `inferadb/common`, where both ledger and KMS depend on it independently. **The trust direction is one-way and load-bearing.**

4. **Wire-protocol types and dispatch are macro-generated; never hand-edit emitted code.** RPC request/response types live in `crates/services/...` (or whatever final layout `proto/` adopts); the canonical service catalog is a `define_protocol!` invocation in `crates/wire-services/...` (when the workspace adopts the same pattern as ledger). Add or rename an RPC by editing the wire types + the macro invocation, never by hand-editing macro output.

5. **Every `*_PREFIX` / `*_KEY` constant on `SystemKeys` has a matching `KEY_REGISTRY` entry.** A missing entry silently disables tier validation for that key. Storage paths are **cluster-scoped** (`/cluster/<id>/sys/...`, `/cluster/<id>/orgs/<org>/...`) so future federation can replicate selected scopes without state migration. Hand-construction of paths via `format!()` at the call site is a bug — every storage write goes through a `SystemKeys::*` builder.

6. **No PII enters KMS.** KMS is a credentials/keys service; it stores wrapped key material, certificate metadata, policy documents, audit events, and bootstrap-token hashes — none of which are PII. Any new field that even _could_ contain PII (an email address, a real name, a free-text field) is a design error to flag immediately. The `data-residency` problem ledger solves does not apply here, but the discipline (audit every storage write for what it actually contains) does.

7. **Server crates use `snafu` only. The SDK uses `thiserror`.** `anyhow` is banned everywhere. Every variant with a `source` field includes `#[snafu(implicit)] location: snafu::Location`. Propagate via `.context(XxxSnafu)?` — never manually construct an error variant. The wire crate's `WireError` is the canonical RPC-boundary error type returned to clients.

8. **No `unsafe`, `panic!`, `todo!()`, `unimplemented!()`, or `TODO`/`FIXME`/`HACK`/`XXX` comments in production code.** No placeholder stubs, no backwards-compat shims, no feature-flag dead paths. `.unwrap()` / `.expect()` outside `#[cfg(test)]` requires a `SAFETY:` comment stating why the call cannot fail. KMS holds the most security-sensitive material in the platform; defensive defaults are the floor, not the ceiling.

9. **Never introduce `openraft` or any external Raft crate.** Consensus is `inferadb-consensus` (extracted from ledger to common). Custom in-house Raft with deterministic simulation. `grep openraft` across the workspace must return zero dependency references.

10. **Configuration minimalism (opinionated defaults over knobs).** A new CLI flag, env var, or config field requires a documented operator-facing reason — a real production scenario where operators have demonstrably needed to tune that value, or genuinely environment-specific input (paths, endpoints, IAM identities). Pure performance-tuning knobs and "just in case" flags are rejected. The full v1 knob list lives in `docs/runbooks/configuration.md` and is intentionally short. CI lints the config crate's public surface against an allowlist.

11. **Bootstrap, seal, and sign operations are audited from day one.** Every `Sys.Seal` / `Sys.Unseal` / `Sys.RotateRootKey` / `Pki.IssueWithBootstrapToken` / `Pki.Revoke` / `Auth.CreateBootstrapToken` emits an HMAC-chained tamper-evident audit event before any side effect commits. Audit-failure-in-fail-closed-mode rejects the request (with one explicit exception documented in §9.4 of the spec — unseal does not re-seal on audit failure). Skipping the audit emit, even temporarily for "ease of testing", is a hard no.

12. **Every issued credential is leased.** PKI leaves create lease records at `/cluster/<id>/sys/leases/pki/<issuer>/<serial>`; tokens create lease records at `/cluster/<id>/sys/leases/token/<token-id>`. `Pki.Revoke` and `Lease.Revoke` are functionally equivalent paths; both add the serial/token-id to the regional kill-list and emit a watch event. Code that produces credentials without lease records is a bug.

13. **Server integration tests are a single binary.** `crates/server/Cargo.toml` sets `autotests = false` and declares exactly one `[[test]] name = "integration"`. Every test file is a submodule of `crates/server/tests/integration.rs` using `use crate::common::` — never `mod common;`. Audited by `test-isolation-auditor`.

14. **`Shard` / consensus state machines return `Action` values and perform no I/O.** Any blocking call, disk read, or network send inside a consensus state machine is a correctness bug. All I/O executes in the `Reactor`. WAL writes are batched with a single `fsync` per batch — never per proposal. (Inherited from `inferadb-consensus`.)

15. **Document non-obvious invariants where they live.** Every load-bearing decision in the spec (§5 architectural invariants, §7.6 RK rotation barrier, §10.4.5 revocation propagation SLO, §16.4 idempotency keys) has a corresponding inline comment in the code that asserts the invariant or links to the spec section. Future contributors should not need to read the spec from scratch to understand why a piece of code looks the way it does.

---

## Escalation

If a rule looks wrong, or a task seems to require violating one, **stop and raise it explicitly**. Don't rationalize a workaround. The human operator decides whether a rule needs to change, with all the context. Silent violations are how trust roots become breach roots.

## Skill / Agent Catalog

`.claude/skills/` and `.claude/agents/` carry KMS-adapted versions of ledger's review and authoring infrastructure. Universal skills (`define-error-type`, `use-bon-builder`, `audit-claude-md`, `just-ci-gate`) work without modification. KMS-specific replacements:

- `agents/kms-residency-auditor.md` — replaces ledger's `data-residency-auditor` for KMS's cluster-scoped path conventions and PII-free invariants.
- `agents/wire-reviewer.md` — minor adaptation; checks KMS's wire surface for the same patterns ledger checks.
- `agents/consensus-reviewer.md` — minor adaptation; reviews KMS's `crates/raft/` orchestration over `inferadb-consensus`.
- `skills/add-system-key/SKILL.md` — replaces ledger's `add-storage-key`; uses cluster-scoped path conventions.
- `skills/new-rpc/SKILL.md` — copied from ledger; KMS adopts the same `define_protocol!` pattern.

When invoking these, the skill/agent file documents the workflow. Read the file first.
