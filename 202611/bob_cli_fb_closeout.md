---
tier: tale
size: small
title:
  Finish landing bob-cli-fb — fix its two test defects and close the keep-moving epic
goal:
  Make the two test defects the keep-moving epic introduced deterministic, then close
  epic bob-cli-fb normally and mark its plan done.
proposed_by: bbugyi200.apollo.bob-cli-fb.land
proposed_with: claude/opus
bead: bob-cli-fb
status: done
---

- **PARENT:**
  [202611/leaf_project_keep_moving_offers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/leaf_project_keep_moving_offers.md)
- **BEAD:**
  [bob-cli-fb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fb/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-fb.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-fb.land.md)
- **COMMITS:**
  - [c772728](https://github.com/bobs-org/bob-plugins/commit/c77272811e40879fac3d2f14926082d256fd98e1)
    — test(task-status-cycler): derive keep-moving test dates from the clock

# Finish landing bob-cli-fb (keep-moving leaf-project offers)

## Context

The land agent for epic **bob-cli-fb** ("Keep leaf projects moving") has already
verified the feature:

- All five phases are closed: contract, rust (through nested epic bob-cli-fb.2.1), nav,
  cycler, and mac.
- The Rust engine, capture JSON and human output, `plan.keep_moving`, the nav picker and
  notice v2, the cycler port, and the Mac presentation are all in place.
- The epic DECISIONS hold:
  - `offer_lanes = open`;
  - glossary-only memory edits (`leaf-project`, `successor-offer`, `task-successor`);
  - no decision record;
  - no `Cargo.toml` version or `CHANGELOG.md` edits.
- On bob-cli master, `cargo fmt --check` and `just lint` pass.
- The integration review of every commit that landed after the epic started found
  nothing that needs rewiring:
  - `=/` split and `=*` merge (ccd1ada1, bb92fb1c) close no tasks, and the KM21 net pass
    diffs the day text generically, so it already covers them.
  - 50a39221 already composes multi-note relation closes with keep-moving.
  - gcal buffers (2e49a561) are unrelated.
  - The bob-plugins commits after 053a82c do not touch the successor code.
- Follow-up triage is done and recorded as a note on bob-cli-fb. Do **not** re-triage.

Two defects were **caused by this epic** and remain epic work. They are recorded as
notes #1 and #2 on bob-cli-fb. This tale fixes both, then closes the epic.

## Step 1 — bob-cli: remove the keep-moving routing test flake

**Defect.**
`src/native/task_complete/tests/keep_moving_tests.rs::routing_is_lazy_and_narrow_with_shared_destination`
(added in 39866275 by bob-cli-fb.2.1) works like this:

- It resets three process-global `AtomicUsize` counters:
  - `ROUTING_LOAD_COUNT` (`src/native/task_complete/successors.rs`, bumped in
    `SuccessorTypeRouting::load`);
  - `FULL_ROOT_SCAN_COUNT` and `NARROW_ROOT_SCAN_COUNT`
    (`src/native/pomodoro_themes/scan.rs`, bumped in `scan_root_areas_matching`).
- It then asserts exact values: `ROUTING_LOAD_COUNT` is 0, then 1, then 2, and
  `FULL_ROOT_SCAN_COUNT` is 0.

Any other lib unit test that does a full root scan or a routing load in the same
`cargo test` process, between the reset and an assertion, makes it fail. The bob-cli-fc
land agent saw it fail in 4 of 6 runs of `cargo test --lib` on an unchanged tree, and it
always passes when run alone.

**Fix.** Make every count belong to the test's own temp vault instead of the process.

1. Add a small `#[cfg(test)]` per-vault counter in `src/native/pomodoro_themes/scan.rs`,
   for example `pub(crate) struct VaultCounter`:
   - It wraps `std::sync::Mutex<BTreeMap<PathBuf, usize>>` behind a `const fn new()`.
   - It has `bump(&self, vault: &Path)` and `get(&self, vault: &Path) -> usize`.
   - Recover a poisoned lock with `unwrap_or_else(|e| e.into_inner())`.
2. Turn `FULL_ROOT_SCAN_COUNT` and `NARROW_ROOT_SCAN_COUNT` into `VaultCounter` statics.
   Bump them with the `vault` argument of `scan_root_areas_matching`.
3. Turn `ROUTING_LOAD_COUNT` in `successors.rs` into a `VaultCounter` too, bumped with
   the `root` that `SuccessorTypeRouting::load` receives. Reuse the type from `scan.rs`
   rather than defining a second one.
4. In the test:
   - Drop the `.store(0, …)` resets. The temp vault is unique, so counts start at 0.
   - Assert with `COUNTER.get(root)` against the same `root` the test passes as
     `bob_dir`.
   - Keep the expected values (0 / 0, then 1 / 0, then 2 / 0) and the comments that
     explain them.
5. First confirm that the path the planner passes down is byte-identical to the test's
   `root` (it is `bob_dir`, which flows through `Catalog::load_without_areas`). If any
   layer canonicalizes it, key by the canonicalized path in both places. If that is not
   practical, fall back to `thread_local!` `Cell<usize>` counters
   (`const { Cell::new(0) }`). Every one of these scans runs on the calling thread.
6. Check
   `grep -rn 'ROUTING_LOAD_COUNT\|FULL_ROOT_SCAN_COUNT\|NARROW_ROOT_SCAN_COUNT' src` and
   update every remaining user. All are `cfg(test)`.

**Verify.**

- Run `cargo test --lib routing_is_lazy_and_narrow_with_shared_destination`.
- Run `cargo test --lib` at least **4 times in a row**. Every run must pass this test.
  Report the per-run pass/fail counts.
- Run `just check`, the canonical gate. Do **not** run `just check-full`. If an
  unrelated test fails, prove it fails the same way on the clean tree (`git stash`)
  before you call it pre-existing.

## Step 2 — bob-plugins: stop the keep-moving cycler test from pinning a date

Open the linked repo with the `/sase_repo` skill:

```bash
sase repo open bob-plugins -r "Fix date-pinned keep-moving cycler test for bob-cli-fb"
```

Read its `AGENTS.md` / `CLAUDE.md` first.

**Defect.** `scripts/test-task-status-cycler-keep-moving.cjs` (added by bob-cli-fb.4 in
99333a5 / 053a82c) hard-codes `const TODAY = "2026-11-01"` and
`const DAY_PATH = "2026/20261101.md"`. The real gesture paths run through
`makeRealPlugin`, and they resolve today's daily note from the real clock
(`todayDailyPath(this.app)` in `137-plugin-successor-project.js`). From 2026-11-02 on, 5
wiring tests therefore fail:

- KM1, daily-note link close;
- KM26, own-note Ctrl+Enter;
- KM26, Alt+] into Done;
- KM27, walk landing;
- KM25, Esc.

Repro at bob-plugins HEAD 885c560:
`node scripts/test-task-status-cycler-keep-moving.cjs` reports 5 failures. That keeps
bob-plugins `npm test` red.

**Fix.** Derive every date from the clock. This is the pattern
`scripts/test-task-status-cycler-successor-wiring.cjs` already uses.

1. Replace the constants:

   ```js
   const TODAY = helpers.formatLocalDate();
   const DAY_PATH = helpers.todayDailyPath({}, new Date(`${TODAY}T12:00:00`));
   const LATER = helpers.formatLocalDate(new Date(Date.parse(`${TODAY}T12:00:00`) + 30 * 86400000));
   ```

2. Replace every hard-coded date literal:
   - each future `[scheduled:: 2026-12-01]` becomes `${LATER}`;
   - each "today" `[scheduled:: 2026-11-01]` becomes `${TODAY}`;
   - the `/\[fresh:: 2026-11-01\]/` assertion becomes one built from `TODAY`, for
     example `new RegExp(`\\[fresh:: ${TODAY}\\]`)`.

   Find them all with `grep -n '2026' scripts/test-task-status-cycler-keep-moving.cjs`.
   That grep must return nothing afterward, except text that has nothing to do with a
   date.

3. Keep every vector id and every assertion's meaning unchanged. KM13/KM17 still need a
   resident scheduled after today (`LATER`) and one scheduled today (`TODAY`).

**Verify.**

- Run `node scripts/test-task-status-cycler-keep-moving.cjs`. It must report 0 failures.
- Prove the fix does not depend on the date: run the file once more under a shifted
  clock, for example with `faketime '2027-03-15 12:00:00'` if it is installed. If it is
  not, use a temporary `-r` preload that overrides `Date`; do not commit it. Report
  which method you used.
- Run `npm run build:check` and `npm run docs:check`.
- Run `npm test`. These failures are pre-existing, already tracked, and not caused by
  this tale:
  - bob-cli-f1, command-mode DOM without Chromium;
  - bob-cli-et, gcal-events and freshness-namespace;
  - bob-cli-eh, note-ready and relation-index;
  - bob-cli-ej, the Live Preview perf bound under load.

  Any other failure must be proven identical on the clean tree.

- Run `bob plugins sync`, as the repo rules require after any change. If a protected
  dirty deploy from another checkout blocks it, report that outcome. Do not force it.

## Step 3 — close out epic bob-cli-fb

1. Run `sase bead epic-symbols bob-cli-fb`. It reported none during landing. If any
   entry appears now, resolve it (wire it up, privatize it, add a non-test pragma, or
   delete it), or re-key its justfile line to a still-open bead that needs it.
2. Close the epic normally, **without** `--force`:

   ```bash
   sase bead close bob-cli-fb --note "<verification>"
   ```

   The note must summarize:
   - **Verification.** All five phases and nested epic bob-cli-fb.2.1 are closed. The
     DECISIONS hold. The code and tests exist in bob-cli, bob-plugins, and
     bob-mac-capture.
   - **Integration review.** Use the findings from Context above.
   - **Follow-up triage.** Point to the earlier LAND TRIAGE note on bob-cli-fb:
     - +1s on bob-cli-f1, bob-cli-et, bob-cli-eh, and bob-cli-ej;
     - new task bob-cli-fm for the cycler localeCompare vs Rust byte ordering;
     - the task-link-marks-motion item declined because it now passes;
     - the fb.2.1 proposals already resolved by that landing.
   - **Fixes from this tale**, with commit subjects and the test evidence from Steps 1
     and 2.

3. Run `just symvision`. This justfile currently has no `symvision` recipe. If
   `just --list` still lacks it, say "symvision unavailable (no just recipe)" in your
   final report.
4. Set `status: done` in the YAML frontmatter of the epic's plan file,
   `plan:202611/leaf_project_keep_moving_offers.md`. It currently says `status: wip`.
   Find its path in the `PLAN` section of
   `sase bead read bob-cli-fb -r "Need the plan file path"`. It lives in the plans
   sidecar.
5. bob-cli-fb has no `parent_bead`, so there is no ancestor to close.

## Done when

- `routing_is_lazy_and_narrow_with_shared_destination` passes on every one of at least 4
  consecutive `cargo test --lib` runs.
- `just check` passes, or fails only on failures proven to be pre-existing.
- `scripts/test-task-status-cycler-keep-moving.cjs` passes, both on the real date and
  under a shifted clock.
- bob-cli-fb is closed with the note above.
- Its plan file says `status: done`.
- The bob-cli, bob-plugins, and plans-sidecar changes are all part of the turn's final
  declaration.
