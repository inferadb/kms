# `.claude/` — InferaDB KMS

This directory carries Claude Code project configuration adapted from `inferadb/ledger/.claude/`. Most files are **copied as-is** from ledger; some need **KMS-specific adaptation** during Phase -1 of implementation.

## Files

### Configuration

- **`settings.json`** — KMS-specific. The `PreToolUse` hook denies `git commit/push/stash/tag/rebase/reset/checkout/merge/cherry-pick/...` (anything state-mutating); read-only git is allowed. Cargo.lock edits are blocked. `PostToolUse` runs `cargo +nightly fmt`, `cargo +1.92 check`, prettier on markdown, and a writing-check linter. The `SessionStart` hook surfaces the KMS-specific golden rules.

### Agents (copied from ledger; **adapt during Phase -1**)

- **`agents/snafu-error-reviewer.md`** — _universal_. No adaptation needed. Reviews `snafu` error handling.
- **`agents/unsafe-panic-auditor.md`** — _universal_. No adaptation needed. Audits production code for `unsafe`, `panic!`, `todo!()`, `unwrap()` without `SAFETY:` justification.
- **`agents/wire-reviewer.md`** — needs minor adaptation. Replace ledger-specific service references with KMS's service surface. Currently references ledger's 14 services and the `inferadb-ledger-wire-services` crate; KMS uses fewer services and `inferadb-wire-services` (post-Phase 0 extraction).
- **`agents/consensus-reviewer.md`** — needs minor adaptation. Same pattern; references should target KMS's `crates/raft/` orchestration over `inferadb-consensus`, not ledger's apply pipeline.
- **`agents/test-isolation-auditor.md`** — needs minor path adaptation. The single-binary-tests rule applies identically; only paths differ.
- **`agents/documentation-reviewer.md`** — substantial adaptation. The 5-partition model (A: README/CONTRIBUTING/SECURITY, B: CLAUDE.md, C: docs/, D: code-doc cross-references, E: in-code rustdoc) survives; the partition-D/E content lists are ledger-specific and need a KMS pass.

### Agents (KMS-specific replacements; **author during Phase -1**)

- **`agents/kms-residency-auditor.md`** — replaces ledger's `data-residency-auditor`. Audits cluster-scoped path conventions (`/cluster/<id>/sys/...`, `/cluster/<id>/orgs/<org>/...`) and the PII-free invariant. Core checks: every storage path includes a `/cluster/<id>/` prefix; no field contains plaintext PII (emails, names, free text); `KEY_REGISTRY` entries match `*_PREFIX` / `*_KEY` constants 1:1; cluster-scoping is verified for any new storage operation.

### Skills (copied; **most universal**)

- **`skills/define-error-type/SKILL.md`** — _universal_. No adaptation.
- **`skills/use-bon-builder/SKILL.md`** — _universal_. No adaptation.
- **`skills/just-ci-gate/SKILL.md`** — _universal_. Adapt the example just-recipe names to KMS's `Justfile` once it exists.
- **`skills/audit-claude-md/SKILL.md`** — _universal_. Reads each crate's CLAUDE.md against root rules.
- **`skills/new-rpc/SKILL.md`** — needs adaptation. Replace ledger service-catalog references with KMS's. Same `define_protocol!` pattern.

### Skills (KMS-specific replacements; **author during Phase -1**)

- **`skills/add-system-key/SKILL.md`** — replaces ledger's `add-storage-key`. Walkthroughs for adding a new entry to KMS's cluster-scoped `SystemKeys` registry, with KEY_REGISTRY entry, tier classification (GLOBAL vs REGIONAL cluster), and validate-tier call discipline.

### Commands

- **`commands/validate.md`** — copied from ledger. Adapt to KMS's `Justfile` recipe names.

### Skipped (ledger-specific, do not copy)

- `agents/data-residency-auditor.md` — ledger's PII residency model doesn't apply (KMS holds no PII). Replaced by `kms-residency-auditor`.
- `skills/add-new-entity/SKILL.md` — ledger's entity-versioning model doesn't apply.
- `skills/debug-integration-test/SKILL.md` — ledger-specific test-debugging walkthrough.
- `skills/add-storage-key/SKILL.md` — replaced by `add-system-key`.

## Phase -1 (project setup) checklist

These items must be completed before any code work begins:

- [ ] Verify `.claude/settings.json` `PreToolUse` git-deny hook fires on `git commit` attempts.
- [ ] Author `agents/kms-residency-auditor.md`.
- [ ] Author `skills/add-system-key/SKILL.md`.
- [ ] Adapt `agents/wire-reviewer.md`, `agents/consensus-reviewer.md`, `agents/test-isolation-auditor.md`, `agents/documentation-reviewer.md`.
- [ ] Adapt `skills/new-rpc/SKILL.md`, `skills/just-ci-gate/SKILL.md` (example commands).
- [ ] Adapt `commands/validate.md`.
- [ ] Confirm CLAUDE.md golden rules are crate-aware (each crate gets its own CLAUDE.md extending — never relaxing — the workspace rules).
