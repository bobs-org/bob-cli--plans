---
tier: tale
title: Split the ledger-tools freshness test suite
goal:
  Preserve all 57 freshness tests in a shared harness and six focused files under 1000
  lines, verify the split, and close only bob-cli-4f.3.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-4f.3
bead: bob-cli-4f.3
create_time: 2026-10-04 22:34:05
status: wip
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files_1.md)
- **BEAD:**
  [bob-cli-4f.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4f/bob-cli-4f.3.md)

# Split the ledger-tools freshness tests for bob-cli-4f.3

## Objective and ownership

Complete the already assigned, in-progress phase bead `bob-cli-4f.3` by replacing
`scripts/test-ledger-tools-freshness.cjs` in the linked `bob-plugins` repository with
one shared harness and six focused test files. Preserve every test name, fixture,
assertion, and plugin behavior. Every new or modified hand-edited file must have at most
1000 lines, preferably fewer than 900.

This is a `tale` of size `medium`: the source boundaries and dependencies have been
inspected, and one implementation agent can make and verify this bounded mechanical
refactor. Do not create additional beads or delegate implementation. Do not manually
change the phase status. Close only `bob-cli-4f.3` after verifying its work; do not
close parent epic `bob-cli-4f`, ancestor plans, or other beads.

The parent design is `plan:202610/split_largest_bob_plugins_js_files_1.md`, phase
`ledger-freshness-tests`. Read it with `sase artifact read` if more context is needed.
The user's phase-worker instructions override the parent design's generic suggestion to
create follow-up task beads: record follow-ups on this phase instead.

## Inspected baseline

The linked repository was clean at commit `2486da9c17b3fdade1a954ecf31e2e88acef5751`
(`refactor(test): split block-id-prompt suite`). The assigned file has 2895 lines and 57
top-level `test(` cases. Running
`node --test --test-reporter=spec scripts/test-ledger-tools-freshness.cjs` passed all
57, with zero failures, skips, cancellations, or todos. `npm run build:check` and
`npm run validate` also passed; all six plugin manifests were valid. The README
currently has 354 lines, and `package.json` has 14.

`sase bead epic-symbols bob-cli-4f.3` reported no entries during planning. Run it again
immediately before closing, because the current result is not a substitute for the
required final check.

## Chosen split

All paths below are relative to the checkout returned by `sase repo open bob-plugins`;
line ranges refer to the inspected baseline and must be rechecked against the
implementation agent's current source. Split at whole declarations, never inside test
callbacks.

| New file under `scripts/`                      | Original source ranges                        | Cases | Responsibility                                                            |
| ---------------------------------------------- | --------------------------------------------- | ----: | ------------------------------------------------------------------------- |
| `ledger-tools-harness.cjs`                     | 1-110, 265-304, 813-890, 1359-1408, 2708-2737 |     0 | Loader stubs, helper bindings, shared row/app/element fixtures            |
| `test-ledger-tools-freshness-placement.cjs`    | 111-264                                       |     4 | Placement vectors, stamping/refusals, reader lints                        |
| `test-ledger-tools-freshness-states.cjs`       | 305-812                                       |    12 | State vectors, config coercion/loading, interval precedence               |
| `test-ledger-tools-freshness-namespace.cjs`    | 891-1358 and 1409-1503                        |     8 | Public namespace, invalidation/events, status bar, synchronous API        |
| `test-ledger-tools-freshness-review-model.cjs` | 1504-2126                                     |    11 | Bucket partition, calendar dates, model/chips, memo/config/cache identity |
| `test-ledger-tools-freshness-queue.cjs`        | 2127-2382                                     |    12 | Q1/Q2, L1-L5, R1/R2, missing-created ordering, upkeep/meter               |
| `test-ledger-tools-freshness-tracking.cjs`     | 2383-2707 and 2738-2895                       |    10 | Tracker intervals/counts/queue, checklist vectors, reference capability   |

The six groups total 57 tests. Their moved source spans contain 154, 508, 563, 623, 256,
and 483 lines respectively, leaving ample room for imports. Moving Q1 into the queue
group and starting tracking at its first test (2383) corrects the parent design's
approximate boundaries. The longest group should remain around 650 lines; the harness
should remain around 350.

Use the broader name `ledger-tools-harness.cjs`: freshness-footer and freshness-mark
already duplicate the same plugin stubs, and several other ledger suites duplicate that
loader setup. No harness with this name exists. Only the six files in this split consume
it in this phase; migrating existing sibling suites is out of scope. Existing freshness
files named `footer`, `mark`, `mark-surfaces`, `keeps`, and `decision-card` remain
untouched.

## Implementation

1. Use `/sase_beads` and `/sase_memory_read` for the required bead lifecycle context,
   then reread `sase bead read bob-cli-4f.3 -r "Need the phase scope and design file"`.
   Use `/sase_repo` to open `bob-plugins` with an audit reason and read its `AGENTS.md`
   before editing. Use only the returned checkout path. Recheck git status, record the
   actual pre-refactor commit, count the source lines and tests, and run the original
   suite. Preserve any unrelated work. The other epic phase may update `package.json` or
   README; retain its entries.

2. Extract the shared declarations into `scripts/ledger-tools-harness.cjs` without
   rewriting their implementations. It owns `TestMarkdownView`, `TestPlugin`,
   `TestNotice` (and its existing messages array), the temporary `Module._load` hook,
   `LedgerToolsPlugin`, `helpers`, the original named helper destructure, `D`, `CFG`,
   `sRow`, `laneRow`, `readyRow`, `makeFreshnessTask`, `makeFreshnessApp`,
   `withMissingConfig`, `makeStatusEl`, and `checklistTaskRow`. Keep the stubs installed
   before requiring the plugin and restore the loader immediately afterward, exactly as
   today. Export these bindings explicitly. Keep the harness free of `node:test`
   registration; retain `assert` where its fake metadata-cache methods use assertions.

3. Create the six files above, each requiring `node:assert/strict`, `node:test`, and its
   required bindings from `./ledger-tools-harness.cjs`. Move the original test blocks
   and placement vectors byte-for-byte. Preserve test order within each area, all nested
   fixtures, and the existing `finally` cleanup. In particular, keep
   `withMissingConfig`'s environment restoration, plugin `onunload` calls, and
   filesystem monkey-patch restoration unchanged. Do not replace dates, rename helpers,
   normalize assertions, or fix incidental bugs. Keep the conformance documentation
   comments with the relevant definitions. Delete
   `scripts/test-ledger-tools-freshness.cjs` after transferring everything.

4. Replace exactly its explicit entry in `package.json`'s `scripts.test` with all six
   new test paths in the table's order. Keep every other entry and the initial
   generated-bundle check. Do not add the harness as a test entry or broaden the list to
   a glob. Update README's scripts tree and testing section with the harness, six areas,
   a focused-run command, and the preserved 57 cases. A glob for the focused command
   must select just these six files, or list them explicitly; `freshness-*.cjs` also
   includes pre-existing sibling suites. No plugin source, generated bundle, manifest,
   or version bump is needed.

## Verification and completion

1. Compare the original and new test-name multisets and require identical membership and
   multiplicity, totaling the freshly measured baseline (57 unless it changed). Compare
   the moved test blocks and fixtures against the recorded base to verify their contents
   are unchanged, allowing only new module imports/exports and moved comments. Review
   `git diff --check` and the final diff for unrelated edits. Confirm the old filename
   has no remaining executable or documentation references.

2. Run `node --test` on each of the six files separately: the expected pass counts are
   4, 12, 8, 11, 12, and 10. Then run all six together with an explicit file list:
   require exactly 57 passes, zero failures, and no skipped, cancelled, or todo tests.
   Ensure `package.json` includes every new test once and excludes the removed file and
   harness. Run `wc -l` on every new or modified hand-edited file, including README and
   `package.json`; all must be at most 1000 lines.

3. Run `npm run build:check`, `npm test`, and `npm run validate` in the opened linked
   checkout. Do not regenerate bundles for this test-only refactor. Fix failures caused
   by this split. If a failure reproduces identically on a clean snapshot of the
   recorded base, record evidence as
   `sase bead note bob-cli-4f.3 'PROPOSED FOLLOW-UP: <summary — detail>'` (cite an
   existing tracking bead if known) and proceed with closing the phase, as requested by
   the user. Use a temporary base snapshot from this checkout's own git history, without
   resetting or overwriting the working tree.

4. Record the demonstrated duplication as a follow-up note on this phase:
   `PROPOSED FOLLOW-UP: Adopt ledger-tools-harness in existing sibling tests — freshness-footer and freshness-mark duplicate the same loader stubs; migrate compatible siblings in a separate refactor while preserving specialized surface stubs.`
   Do not create a task bead or perform that migration here.

5. Follow the linked repo's required sync step using
   `bob plugins sync --no-pull --repo <opened-bob-plugins-checkout>` after the edits are
   complete. Supplying the actual returned checkout and skipping a pull ensures this
   command uses the reviewed local changes. Respect the sync command's normal handling
   of vault edits; do not add `--force`.

6. Run `sase bead epic-symbols bob-cli-4f.3` immediately before closing. Resolve any
   remaining symbol or re-key its Justfile entry to an appropriate still-open
   parent/later-phase bead; never leave a reference to this phase after close. Close
   only `bob-cli-4f.3` with
   `sase bead close bob-cli-4f.3 --note "<verified preservation, per-file and combined counts, line limits, build/test/manifest results, and sync result>"`.
   Do not close an ancestor or manually set any bead status.

7. Use `/sase_final` as the last action before a normal final answer and declare the
   linked-repository changes for commit, with a conventional subject such as
   `refactor(test): split ledger-tools freshness suite`. Do not run raw `git commit`.
   Report the phase closure and meaningful verification results.
