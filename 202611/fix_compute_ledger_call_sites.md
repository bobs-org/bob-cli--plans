---
tier: tale
title: Fix compute_ledger call sites broken by the new config/raw_descs parameters
goal:
  The bob-cli workspace compiles again with zero errors and warnings and `just check`
  passes in full.
size: small
proposed_by: bbugyi200.athena.1cu
proposed_with: claude/claude-fable-5
create_time: 2026-11-15 19:46:41
status: wip
---

# Fix `compute_ledger` call sites broken by the new config/raw_descs parameters

## Problem

`just install` (and any `cargo check --all-targets`) fails with two `E0061` errors.
Commit `cfd4548a` / `b8aed5a9` extended the subscription ledger entry point
`compute_ledger` in `src/native/bank/subscriptions/ledger.rs` from four to six
parameters:

```rust
pub(crate) fn compute_ledger(
    registry: &SubscriptionRegistry,
    journal: &JournalSnapshot,
    rows: &[BankRow],
    today: NaiveDate,
    config: Option<&BankConfig>,
    raw_descs: &HashMap<String, String>,
) -> LedgerSnapshot
```

but two call sites in `src/native/bank/check_subscriptions.rs` were never updated and
still pass four arguments:

1. **Lib (breaks the install build):** line 58, inside `findings()`:

   ```rust
   let ledger = (registry.availability == SubscriptionAvailability::Available)
       .then(|| compute_ledger(registry, journal, rows, today));
   ```

2. **Test (breaks `cargo test` compilation):** line 1105, inside the
   `subscription_findings_project_ledger_and_registry_diagnostics` test:

   ```rust
   let ledger = compute_ledger(&registry, &journal, &[], date(12));
   ```

These are the only stale call sites: the other callers
(`src/native/bank/subscriptions/projection.rs:99` and the `ledger()` test helper at
`src/native/bank/subscriptions/ledger.rs:646`) already pass all six arguments. There are
no other compiler warnings in the tree — the extra "warning: build failed, waiting for
other jobs to finish..." line in the install log is cargo scaffolding that disappears
once these two errors are fixed.

## Semantics of the two new arguments

- `config: Option<&BankConfig>` lets attribution infer read-time categories from
  `bank.rules` (`resolve_txn` in `src/native/bank/subscriptions/attribution.rs`).
- `raw_descs: &HashMap<String, String>` is an **import-time** enrichment that maps new
  row IDs to their uncleaned source descriptions (see the doc comment on
  `enrich_rows_with_raw` in `src/native/bank/subscriptions/projection.rs`). Outside an
  import there are no raw descriptions, and the read-time projection path
  (`projection.rs` line 65) already passes `&HashMap::new()` for it.

`bob bank check` is a read-time path, so the correct arguments at both broken call sites
are the in-scope `config` plus an empty map.

## Changes

Both edits are in `src/native/bank/check_subscriptions.rs`. `HashMap` is already
imported at the top of the file (and via `use super::*;` in the test module), so no
import changes are needed.

### 1. `findings()` call site (line ~58)

Pass the `config` parameter that `findings()` already receives, plus an empty
raw-description map:

```rust
let ledger = (registry.availability == SubscriptionAvailability::Available)
    .then(|| {
        compute_ledger(registry, journal, rows, today, config, &HashMap::new())
    });
```

### 2. Test call site (line ~1105)

In `subscription_findings_project_ledger_and_registry_diagnostics`, move the existing
`let cfg = config("");` line (currently just below the `compute_ledger` call) above the
call, and pass `Some(&cfg)` plus an empty map so the test mirrors the production path
(which now threads `config` through; the same test already passes `Some(&cfg)` to
`append_record_findings`):

```rust
let cfg = config("");
let ledger = compute_ledger(
    &registry,
    &journal,
    &[],
    date(12),
    Some(&cfg),
    &HashMap::new(),
);
```

This is behavior-preserving for the test: `config("")` declares no `bank.rules`, so
read-time category inference contributes nothing and the asserted finding codes are
unchanged.

Do not change the `compute_ledger` signature or any other caller, and do not touch
`Cargo.toml` version or `CHANGELOG.md` (releases ride the daily train). This is a plain
`fix(bank)` — no CLI or JSON contract changes, so no `!` breaking marker.

## Validation

From the repo root:

1. `cargo check --all-targets` — fast loop; must report zero errors and zero warnings.
2. `cargo fmt --all` — make sure the edited call sites are formatted.
3. `just check` — the canonical gate (`cargo fmt --all -- --check`, gated
   `cargo clippy --workspace --all-targets --all-features`, and
   `cargo test --workspace --no-fail-fast`); must pass in full. If the gated clippy pass
   flags the edited lines (none is expected), resolve the lint rather than suppressing
   it.
