---
tier: tale
title: Split the Task Status Cycler test suite
goal:
  Preserve all 185 cycler tests in per-area files and a shared harness under 1000 lines,
  verify the refactor, and close only bob-cli-47.5.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-47.5
bead: bob-cli-47.5
status: done
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files.md)
- **BEAD:**
  [bob-cli-47.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-47/bob-cli-47.5.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-47.5](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-47.5.md)
- **COMMITS:**
  - [b854201](https://github.com/bobs-org/bob-plugins/commit/b8542019284ca72e2dfe36b60c523edbe47b36ed)
    — refactor(test): split task-status-cycler suite

# Split the Task Status Cycler test suite for bob-cli-47.5

Complete only assigned phase bead `bob-cli-47.5`, whose parent epic is `bob-cli-47`.
Split `scripts/test-task-status-cycler.cjs` in the linked `bob-plugins` repository into
a shared harness and per-area test files, each no longer than 1000 lines. Preserve all
test names, test bodies, fixture/helper bodies, and plugin behavior. This is one bounded
extraction using the harness convention already established by phase `bob-cli-47.4`; a
medium tale is sufficient and needs no further phases.

## Context and measured baseline

Read the phase with:

```sh
sase bead read bob-cli-47.5 -r "Need the phase scope and design file"
sase artifact read plan:202610/split_largest_bob_plugins_js_files.md "Need the cycler test phase design"
sase repo open bob-plugins -r "Implement the cycler test split for bob-cli-47.5"
```

Use only the linked checkout path printed by `sase repo open`; read its `AGENTS.md`
before edits. Do not locate another checkout or edit deployed files in the vault. Do not
change the bead's status by hand: it is already reserved and in progress.

At linked-repo commit `5d074dc340173f94cb9353f11a0241d9a8717af5`:

- `scripts/test-task-status-cycler.cjs` is 7168 lines and contains 185 distinct tests.
  The epic's original 6987-line/179-test measurement predates six additional
  `completeTaskAtCursor` tests. Preserve the current 185 tests, including API v2
  coverage, rather than using the epic's old count as a target.
- `node --test --test-reporter=dot scripts/test-task-status-cycler.cjs` passes.
- `npm test` passes all 1764 tests with no failures, skips, or cancellations.
- `npm run validate` passes the generated-bundle checks and validates all six plugins.
- `scripts/navigation-hotkeys-harness.cjs` is 767 lines. Navigation area files import
  `node:test`, `node:assert/strict`, and the shared harness. `package.json` retains an
  explicit test-file list. Reuse those conventions.
- Both the primary and linked checkouts were clean when inspected. The preceding
  navigation test split, commit `5d074dc`, changed tests, `package.json`, and README; it
  did not bump a plugin manifest for a test-only refactor.
- `sase bead epic-symbols bob-cli-47.5` currently reports no entries. Repeat it just
  before closing; the earlier result is not a substitute for the closing check.

Re-measure if the file changes before implementation. Save the exact current source and
its test-name list before extraction so validation uses that actual baseline.

## Implementation

### 1. Script a complete extraction

Use a throwaway extraction script and scratch files within the current workspace; remove
only that scratch material when finished. Work from one saved input byte string and
slice complete declarations/test statements by line range. Do not hand-copy or reformat
bodies. The ranges below refer to commit `5d074dc` and include the newline separators
between declarations. Recompute them if the source changes.

Assert every input line is assigned exactly once, without gaps or overlaps. The four
harness ranges plus the test ranges in the table cover all 7168 source lines. Any
regrouping must preserve complete test statements and ascending source order within each
destination file. If origin changes during the work, rerun extraction against the new
source rather than hand-merging moved code.

### 2. Create the shared harness

Create `scripts/task-status-cycler-harness.cjs`, without the `test-` prefix, using these
original ranges in their existing order:

| Original lines | Content                                                                                                                                                              |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1–176          | Imports, original loader, stub-assigned classes, notices, focused DOM state, DOM-node builder, loader override, plugin require with try/finally, and helpers binding |
| 357–478        | In-memory app, embedded-target assertion, text editor, active Markdown view, Vim action registration, and async flush helpers                                        |
| 4455–4549      | Shared demotion/promotion note fixtures, heading/prompt resolvers, picker opener, and picker acceptance helpers                                                      |
| 6302–6342      | `createCompleteAtCursorPlugin` and `installTasksCloseCommand`                                                                                                        |

Keep the existing `Module._load` stub and try/finally restoration byte-for-byte. Export
`TaskStatusCyclerPlugin`, `helpers`, `MarkdownView`, `TestModal`, `notices`,
`createTestDomNode`, and the fixture/helper functions from these ranges after the plugin
require has completed. Direct class exports are safe at that point: the stub has
assigned both classes. Keep `focusedEl` and `originalLoad` private to the harness.

Every test and helper that creates a view must use this same exported `MarkdownView`
constructor. Do not reconstruct the classes in area files, re-run the loader stub,
replace the existing modal with `modal-harness.cjs`, or change the DOM implementation.
The cycler's stub has distinct semantics that must remain intact. The harness must
register no tests of its own. Its moved ranges total 434 lines before export glue.

### 3. Create eleven area files

Each destination is `scripts/test-task-status-cycler-<area>.cjs`. Prepend only the
necessary `node:test`, `node:assert/strict`, and destructured harness imports. Move the
listed ranges unchanged after these imports. Use the same harness module everywhere.

| Area                  | Original line ranges        | Tests | Moved lines |
| --------------------- | --------------------------- | ----: | ----------: |
| `dependencies`        | 177–356, 479–584, 6868–7168 |    24 |         587 |
| `ctrl-enter`          | 585–1129, 3643–3965         |    24 |         868 |
| `reference-lifecycle` | 1130–1368, 2085–2580        |    23 |         735 |
| `pomodoro-completion` | 1369–2084                   |    17 |         716 |
| `pomodoro-work-log`   | 2581–2977                   |    16 |         397 |
| `pomodoro-reopen`     | 2978–3642                   |    15 |         665 |
| `blocked-schedule`    | 3966–4454                   |    17 |         489 |
| `demotion-picker`     | 4550–5161                   |    10 |         612 |
| `promotion`           | 5162–6050                   |     9 |         889 |
| `task-links-api`      | 6051–6301, 6343–6644        |    19 |         553 |
| `freshness`           | 6645–6867                   |    11 |         223 |

Eleven areas keep feature boundaries intact, especially the separate freshness contract,
while the largest file has adequate headroom below 1000 lines. Combining non-adjacent
clusters is safe because each test file runs in its own process; preserve the original
order inside each file. Keep existing notices resets, global-window restoration, real
clocks, dates, and asynchronous behavior untouched.

### 4. Wire the suite and document the layout

Replace exactly the `scripts/test-task-status-cycler.cjs` token in the `package.json`
test command with the eleven new test paths, listed in table order. Preserve the
build-check prefix and every unrelated explicit test path. Do not switch to globbing or
include the harness as a test entry. Delete the original monolithic test file after
extraction parity is established; do not retain a duplicate runner.

Update the README Layout and testing documentation to describe the cycler area files and
`scripts/task-status-cycler-harness.cjs`, including the focused test command. Do not
touch plugin source fragments, generated entrypoints, manifests, dependency versions, or
the existing navigation split. No manifest bump is warranted for this test-only change,
matching the immediately preceding test split.

## Verification and completion

1. Use temporary verification code to compare the old and new suites. Match the exact
   multiset of 185 unique test names and compare every complete top-level test statement
   byte-for-byte, for example with hashes keyed by decoded test title. Check that all 19
   moved fixture/helper function bodies are also byte-identical. Use correctly delimited
   statements rather than including the old interspersed helper declarations in a test's
   comparison span. Retain the original snapshot until these checks pass.
2. Syntax-check the new harness and area files with `node --check`. Use `wc -l` to
   confirm every created or touched hand-edited file is at most 1000 lines, including
   README and `package.json`. Verify the old suite is absent and no references to its
   exact filename remain.
3. Run `node --test scripts/test-task-status-cycler-*.cjs` and compare the runtime test
   names with the saved baseline. Require the same 185 tests to pass with no renames,
   omissions, duplication, skips, or cancellations.
4. Run `npm test`, `npm run validate`, and `npm run build:check` in the opened linked
   checkout. Expect the full suite to retain its 1764-test baseline, subject only to
   independently arrived changes on the implementation base. Review `git diff --check`
   and the final diff for any changes beyond moved code, import/export glue, explicit
   test registration, and README documentation.
5. Honor the linked repo's mandatory sync rule even though deployed plugin bytes did not
   change. Before working with vault synchronization, read applicable Obsidian reference
   memory through `sase memory read obsidian.md -r "Need vault sync rules"`. Run
   `bob plugins sync --no-pull --repo <opened-linked-checkout>` from the workspace,
   using the actual path returned by `sase repo open`. This verifies the exact checkout
   under test and avoids an unrelated pull. If sync refuses dirty managed vault files,
   stop and report the blocker; never use `--force` or edit the deployed files directly.
6. If a check fails, fix extraction/import/glue errors within this scope. For an
   unrelated failure, reproduce it on the exact clean implementation base without
   disturbing other work. If identical there, record the evidence with
   `sase bead note bob-cli-47.5 'PROPOSED FOLLOW-UP: <summary — failure and clean-base evidence, plus an existing task ID if known>'`
   and close this phase anyway as directed by the user. Do not create any beads.
7. Immediately before closing run `sase bead epic-symbols bob-cli-47.5`. Resolve each
   returned symbol or re-key its Justfile entry to a verified still-open bead, such as
   parent epic `bob-cli-47`, then repeat the check until no entries remain.
8. Close only the assigned phase with
   `sase bead close bob-cli-47.5 --note "<verified identity and body preservation, focused/full test results, line limits, build/manifest checks, sync result, and symbol check>"`.
   Do not close the parent epic, any ancestor plan bead, or unrelated work. Record
   discovered issues only as `PROPOSED FOLLOW-UP:` notes on this phase.
9. Remove extraction scratch files and complete the SASE final declaration as the last
   action before the final response. The modified linked checkout is a commit
   obligation; include it in `/sase_final` with a concrete commit message. Do not
   manually invoke `git commit`. Report the phase closure and meaningful verification
   results.

The phase is complete when all current cycler tests remain intact and pass from the new
files, hand-edited files meet the line limit, required checks and sync are handled, and
only `bob-cli-47.5` has been closed with verification evidence.
