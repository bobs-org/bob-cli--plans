---
tier: tale
title: Fix the StoreWriteParams.verified build break
goal: Make `just install` build again at HEAD and restore a green `just check` on
  master.
size: small
decisions:
  restore_green_check:
    ask: Should this tale also repair the three pre-existing lib unit-test failures
      so `just check` and master CI go green?
    choices:
      repair: Also fix the three red lib tests; `just check` and CI pass so the release
        train can ship
      install_only: Fix only the build; master CI stays red and a ci bead tracks the
        three lib tests
    default: repair
    why: master CI has been red since fb28faa1, and the release train needs full CI
      on the exact SHA
    answer: repair
  ref_code_catalog:
    ask: Should `bob ref code` ship as a command-catalog entry rather than an exclusion?
    choices:
      entry: Read-only Report row like ref-show; command mode can run it; entry counts
        go 52 to 53
      excluded: Terminal-only row; command mode refuses typed `ref code` argv; entry
        counts stay 52
    default: entry
    why: it mirrors `ref-show`, and docs/command-catalog.md says every useful public
      leaf ships as an entry
    answer: entry
proposed_by: bbugyi200.apollo.ab
proposed_with: claude/opus
decided_by: auto
status: done
---

# Plan: Fix the `StoreWriteParams.verified` build break and restore a green `just check`

## Root cause

`just install` (`cargo install --path . --locked`) fails at HEAD `36ae0ef0` with:

```
error[E0063]: missing field `verified` in initializer of `StoreWriteParams<'_>`
  --> src/native/highlights_ref/create.rs:3566
```

This is a semantic merge race between commits that agents landed in parallel. Each
commit compiled on its own base, but nobody rebuilt the combined tree:

- `a6d3ecbf` (12:21, "overwrite source store and doctor ref sources") added two new
  `source_store::StoreWriteParams { .. }` struct literals:
  - `stage_overwrite_source_store` in `src/native/highlights_ref/create.rs` (production
    code, the `bob highlights create -f` / recapture overwrite path).
  - the `attach_staged_store` unit-test helper in
    `src/native/highlights_ref/overwrite/execute.rs` (around line 3329).
- `36ae0ef0` (13:14, `bob ref attach-source`, bead `bob-cli-cc.6`) was written against
  an older base. It added a required field `pub(crate) verified: Option<StoredVerified>`
  to `StoreWriteParams` in `src/native/highlights_ref/source_store.rs`, and updated only
  the literals that existed on its base (`write_for_create` and two source_store tests).
  The two literals from `a6d3ecbf` were never updated.

`cargo check --all-targets` reproduces exactly these two E0063 errors (the lib and the
lib-test targets) on Linux too, so nothing here is macOS-specific. Upstream `master`
(`d7b87d41`, which only touches `sase/sase.yml`) does not fix it.

**Correct value:** `verified` records the text-match counts that `bob ref attach-source`
gets when it verifies an existing source against an already-filed PDF (route
`attached`). The overwrite path renders a brand-new PDF _from_ the source, exactly like
the create path, which already passes `verified: None`. The test helper uses route
`markdown`. Both new literals must pass `verified: None`. Adding `Default` or making the
field optional at the type level is not wanted: an explicit field keeps every future
call site honest.

### Pre-existing red CI (same race pattern, does not block `cargo install`)

CI on `master` has failed (or been cancelled) on every commit since `fb28faa1`. With the
two-line fix applied, every integration binary passes (including the `ref` help snapshot
that `36ae0ef0` already repaired), but three lib unit tests still fail:

1. `native::catalog::tests::public_leaves_are_entries_or_excluded`: public leaves
   `["ref", "code"]` (from `fb28faa1`, landed 28 minutes after the catalog coverage test
   in `0bec19d8`) and `["ref", "attach-source"]` (from `36ae0ef0`) are neither catalog
   entries nor exclusions.
2. `native::completion::kinds::tests::every_value_arg_has_a_decision`: `ref code:number`
   and `ref code:page` (u32 value args from `fb28faa1`) lack a completion-kind decision.
3. `native::highlights_ref::source_store::tests::inspect_reports_dangling_mismatch_missing_orphan_and_history`:
   `a6d3ecbf` wrote this test against a base without the `read_store` digest check that
   `fb28faa1` added. Its "missing catalog" fixture lives in `missing--bbbbbbbbbbbb`, but
   the bytes `# no catalog\n` do not hash to `bbbbbbbbbbbb`. So `read_store` returns
   `SourceInvalid` (digest mismatch) before it ever looks for `code.json`, and
   `health.saved` is 1, not 2.

A scratch probe of every change below gave: lib 3625 passed / 0 failed; `cli` catalog
tests passed; `cargo fmt --check` clean. Clippy under the repo's `rustc_gate` reports
warnings only, so `just lint` passes.

## Changes

### 1. Fix the build (always)

- `src/native/highlights_ref/create.rs`, `stage_overwrite_source_store`: add
  `verified: None,` after `catalog,` in the `super::source_store::StoreWriteParams`
  literal.
- `src/native/highlights_ref/overwrite/execute.rs`, test helper `attach_staged_store`:
  add `verified: None,` after `catalog: &catalog,` in the
  `source_store::StoreWriteParams` literal.
- Grep for any other `StoreWriteParams {` literal; there should be none beyond these two
  and the already-correct ones in `source_store.rs` and `ref_library/attach.rs`.

> [!decision] restore_green_check = install_only Stop after step 1 and run only the
> build verification below. Then use `/sase_new_task` to file one `ci` task bead for the
> three lib failures listed above, with the root causes and the fixes from steps 2–4 in
> the description (size `small`). Skip steps 2–4.

> [!decision] restore_green_check = repair Also do steps 2–4.

### 2. Completion kinds for `bob ref code` (`src/native/completion/kinds.rs`)

Right after the existing `path: &["ref", "code"], arg: "ref"` entry, add two entries,
`arg: "number"` and `arg: "page"`, both `kind: Kind::FreeText`. This matches how other
numeric args (`limit`, `days`, `offset`, `min-score`) are classified.

### 3. Command catalog coverage (`src/native/catalog/table.rs`)

- Add an `Excluded` row for `["ref", "attach-source"]` with the reason
  `"Batch source backfill; run it from a terminal"`. Place it before the
  `["ref", "migrate-tasks"]` exclusion. Like the migrations, it is a terminal-run batch
  writer whose required `-s PATH` search roots are not a catalog slot.

> [!decision] ref_code_catalog = entry Add a `ref-code` row right after the `ref-show`
> row:
> `Row { slots: &[("REF", "ref"), ("ref", "ref")], ..reads("ref-code", &["ref", "code"], "Show saved code blocks", &["code", "blocks", "source", "markdown"], Presentation::Report, 60) }`.
> Bump the pinned entry counts from 52 to 53 in `src/native/catalog/tests.rs`
> (`document_shape`: `catalog.entries.len()` and `ENTRIES.len()`) and in
> `tests/cli/catalog.rs` (`entries.len()`).

> [!decision] ref_code_catalog = excluded Add an `Excluded` row for `["ref", "code"]`
> with the reason
> `"Pipe-oriented reader for exact code blocks; run it from a terminal"`, next to the
> attach-source exclusion. Entry counts stay at 52.

The catalog change is additive (`schema_version` stays 1; clients ignore unknown rows),
so no client-side change in bob-mac-capture or bob-plugins is needed, and it is not a
contract break.

### 4. Repair the stale source-store inspect fixture (`src/native/highlights_ref/source_store.rs`)

In `inspect_reports_dangling_mismatch_missing_orphan_and_history`, name the
missing-catalog store after the real digest of its bytes, mirroring how the `saved`
fixture already builds its path:

```rust
let missing = b"# no catalog\n";
let missing_hash12 = &sha256_hex(missing)[..12];
let missing_ptr =
    format!(".bob/ref-sources/papers/missing--{missing_hash12}/source.md");
let missing_dir = bob_dir
    .join(format!(".bob/ref-sources/papers/missing--{missing_hash12}"));
// write `missing` to missing_dir/source.md; no code.json
```

Then pass `Some(&missing_ptr)` to its `inspect_row`. Keep every assertion unchanged: the
test intends a valid source with no catalog (`NoCatalog`, which counts toward `saved`
and `missing_catalog`), and the `mismatch--cccccccccccc` fixture keeps covering the
digest-mismatch case. Do not loosen `read_store`'s digest check.

## Verification

Run these from the repo root:

1. `cargo build --locked`, then `just install-smoke`. It performs the same
   `cargo install --path . --locked` as `just install`, into a temp root, then runs
   `bob --help`.
2. With `restore_green_check = repair`: `just check` (`cargo fmt --check`, clippy with
   the repo `rustc_gate`, `cargo test --no-fail-fast`) must pass with zero test
   failures. With `install_only`: `cargo check --all-targets --all-features` must report
   no errors (the three lib test failures are expected and tracked by the new bead).
3. Run `cargo fmt` if needed. Do not edit the `Cargo.toml` version or `CHANGELOG.md`
   (release-train rule). Use a `fix(...)` conventional commit type, not breaking (`!`).

## Out of scope

- Process hardening so that land/merge flows rebuild the combined tree before pushing.
  This is the systemic cause of all four breaks. It belongs to SASE landing tooling, not
  this repo.
- The ~325 clippy lints that are warnings under the repo gate.
