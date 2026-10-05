---
tier: tale
size: medium
title: Split the block-id-prompt test suite while preserving all 179 tests
goal: Replace the block-id-prompt test monolith with a shared harness and ten cohesive
  test files of at most 1000 lines each, preserving every test body and assertion,
  registering all files in npm test, and completing only phase bob-cli-4f.2.
proposed_by: bbugyi200.apollo.bob-cli-4f.2
bead: bob-cli-4f.2
status: done
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files_1.md)
- **BEAD:**
  [bob-cli-4f.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4f/bob-cli-4f.2.md)

# Scope and ownership

Implement the already reserved, in-progress phase **bob-cli-4f.2**, whose parent is
**bob-cli-4f**. Its description requires splitting `scripts/test-block-id-prompt.cjs`
into a shared harness and per-area test files without losing tests. The parent design is
`plan:202610/split_largest_bob_plugins_js_files_1.md`; read it through
`sase bead read bob-cli-4f.2 -r "Need the phase scope and design file"` and read the
design file that command resolves.

This is a pure test-layout refactor in the linked **bob-plugins** repository. Do not
alter plugin behavior, source fragments, generated entrypoints, manifests, test names,
assertions, stub behavior, or helper names. Splitting the ledger-tools freshness and
navigation dependencies-stage suites belongs to later phases and is outside this tale.

Use the `/sase_repo` skill and
`sase repo open bob-plugins -r "Implement the block-id-prompt test split for bob-cli-4f.2"`.
Use only the returned repository path, and read its `AGENTS.md` before editing. It
requires `bob plugins sync` after repository changes and forbids editing generated
`main.js` files. The existing task-status-cycler and navigation-hotkeys harness/test
layout is the structural precedent; retain block-id-prompt's own stubs because its
`TFile`, `MarkdownView`, and notice identities are part of this suite's setup.

One coding agent can complete this bounded movement of existing tests and fixtures. The
medium tale avoids subdividing this already assigned epic phase further.

# Recorded baseline

The linked repository was clean at commit `03f0c177989871805561e84cd16f7310cdbed32f`
(`refactor(block-id-prompt): split main.js onto the fragment source build`). Phase
bob-cli-4f.1 has closed; the plugin now has `src/fragments.json`, and its generated
entrypoint is what the tests must continue loading.

At that commit the target is **4945 lines**, with **179 top-level `test(...)` calls**.
`node --test scripts/test-block-id-prompt.cjs` passed **179/179**, with no failures,
skips, cancellations, or todo cases. README is 348 lines and package.json is 14 lines.

The original whole-file SHA-256 is
`1263016351d34878639e4ad68c96660e5025f873c278f84e1482f1457f6f9d1b`. The
order-independent test-body digest is
`9137bf6de9ac5aa6755eefc5304c2ad85ac91da0f7a28e114f80a8282ac1f3d5`: extract every block
from a line beginning `test(` through the next line exactly `});`, including those lines
and original newlines; hash each block with SHA-256, sort its hexadecimal digests,
concatenate them without separators, and hash that string. All 179 blocks are distinct.
These boundaries were checked against this suite's current top-level layout.

Before implementation, re-measure and rerun the original suite. If upstream changed the
file, record the new pre-split commit, line count, full test count, and test-body
inventory, and use that untouched version for parity. The table below is a checked
mapping for the recorded baseline, not permission to cut through a test after drift.

# Chosen split

Create `scripts/block-id-prompt-harness.cjs` and these ten files. Line ranges below
refer to the original file and include complete tests. Size estimates exclude import
headers; the largest file has enough room to remain roughly 900 lines or fewer.

| New test file suffix (`test-block-id-prompt-<suffix>.cjs`) | Original ranges                 |   Tests | Approximate body lines |
| ---------------------------------------------------------- | ------------------------------- | ------: | ---------------------: |
| dependencies                                               | 137-400                         |      10 |                    264 |
| markers                                                    | 401-763                         |       8 |                    363 |
| pomodoro-context                                           | 764-1086                        |       9 |                    323 |
| target-update                                              | 1087-1896                       |      35 |                    810 |
| pomodoro-insertion                                         | 1897-2493                       |      28 |                    597 |
| work-log                                                   | 2494-2727                       |      10 |                    234 |
| pomodoro-link-runtime                                      | 2728-2815, 2853-3176, 3570-3719 |      23 |                    562 |
| pomodoro-unlink-runtime                                    | 3177-3569                       |      11 |                    393 |
| task-link-deletion                                         | 3720-3723, 3789-3983, 3997-4546 |      27 |                    749 |
| plan-budget-and-freshness                                  | 4547-4945                       |      18 |                    399 |
| **Total**                                                  |                                 | **179** |                        |

The target-update area stays together because the calendar, schedule-log, planner, and
activation runtime tests form one coherent group and fit comfortably below the limit.
The old oversized runtime area is separated into link and unlink files. Keep the late
link-refusal tests with the link runtime even though unlink tests separate them in the
old file. Move complete test blocks byte-for-byte and keep the original relative order
within each new file.

# Implementation

1. **Extract the shared harness.** Move the original setup at lines 1-136, removing its
   `node:test` import because the harness must register no tests. Keep the strict assert
   import used by its helpers and the Module import used by the stubs. Preserve the
   Module._load hook for `obsidian` and `@codemirror/view`, require the existing
   `../plugins/block-id-prompt/main.js` while that hook is installed, and restore the
   original loader immediately afterward, exactly as before. Preserve the internal
   `obsidianStubs` variable, the captured stub class identities, and the single
   never-reassigned `noticeMessages` array. `resetNotices` must continue clearing that
   array in place.

   Move the additional shared fixtures without changing their bodies:
   - `createTaskModeHarness`, `LINKED_DAILY`, and `UNLINKED_DAILY` (2816-2852).
   - `selectTaskLink`, `deleteTaskLink`, and `createTaskLinkHarness` (3724-3788),
     including the explanatory comment immediately before the harness factory.
   - `NEXT_TASK` and `DAILY_WITH_LINKS` (3984-3996).

   Export `Plugin`, `helpers`, `noticeMessages`, `resetNotices`, `lastNotice`,
   `localDate`, `WORK_LOG_DATE`, `createEditor`, `applyPlannedEdits`,
   `sourceForTaskPicker`, `sourceForPomodoroLink`, `createTFile`, `createMarkdownView`,
   and all the moved shared factories and fixtures through `module.exports`. This
   harness should be about 280 lines including its exports. Keep `pomodoroLinkBase`
   (2102-2117) local to pomodoro-insertion, and `stubPlanBudgetApi`, `PLAN_BUDGET_OK`,
   and `PLAN_BUDGET_OVER` (4547-4575) local to plan-budget-and-freshness: those fixtures
   serve only one area. No test file should import another test file.

2. **Move the ten test areas.** Give each file its own imports of `node:test` and
   `node:assert/strict`, and destructure only its required harness exports. Preserve
   every original test call, body, name, assertion, async marker, fixture value, and
   explanatory comment. Put the future-scheduled activation heading (1087) with
   target-update and the selected Task Link heading (3720-3722) with task-link-deletion.
   Preserve the Depends-On compatibility comments alongside their tests. Do not add
   concurrency options: existing tests in each file rely on sequential execution and
   in-place notice resets. Node's normal file-level isolation gives each area its own
   harness/plugin instance; standalone execution must work too. Remove
   `scripts/test-block-id-prompt.cjs` after all its code has been accounted for.

3. **Register and document the split.** In package.json's explicit `scripts.test`
   command, replace the single old block-id-prompt path with all ten new paths, in table
   order. Leave every other test path and the leading build check intact; preserve
   package.json formatting. Confirm each new path occurs exactly once and neither the
   old path nor the harness occurs as a test entry.

   Add the new harness and per-area test pattern to README's scripts tree. In the
   existing testing prose, describe the area files, the harness, and the focused command
   `node --test scripts/test-block-id-prompt-*.cjs`. Mention the preserved 179-case
   coverage if that remains the pre-split count. Keep unrelated README behavior
   descriptions intact.

# Verification and acceptance

All of the following are required before declaring this phase complete:

1. **Prove exact test preservation.** Read the original through
   `git show <pre-split-commit>:scripts/test-block-id-prompt.cjs`, not the deleted
   working-tree path. Use a temporary or inline script to compare Counters of the
   complete original and new top-level test blocks byte-for-byte. Require the same count
   (179 at the recorded baseline), identical names, and no missing, duplicate, or
   modified block. Check the recorded digest when using the recorded baseline. A simple
   equal test count is insufficient. Review the moved fixture/setup bodies against the
   original as well; only imports, exports, and file placement should differ. Do not add
   a permanent test that mirrors the refactor.

2. **Run the focused suite and every file independently.**
   `node --test scripts/test-block-id-prompt-*.cjs` must pass all 179 tests with no
   skips or cancellations. Then run `node --test <file>` for each of the ten files
   separately. Verify each file's pass count matches the table and their sum is 179. No
   standalone file may depend on another test file having run first.

3. **Run the repository checks.** In bob-plugins, run `npm run build:check`, `npm test`,
   and `npm run validate`; all introduced failures must be fixed. Plugin source is
   unchanged, so a build regeneration and patch-version bump are not part of this phase.
   Review `git diff --check` and confirm that no unrelated suite was removed from the
   explicit npm command. Run the primary checkout's standard check if it defines one.
   Use `/sase_monitor` for a command expected to run long, and wait for the monitor
   handoff command itself to exit; do not leave a live exec session when ending the
   turn.

4. **Measure the hand-edited files.** Run `wc -l` for the harness, all ten area files,
   README.md, and package.json. Every file created or modified by this phase must be at
   most 1000 lines; aim for roughly 900 or fewer. Confirm the original monolith is
   deleted and only this phase's files changed.

5. **Deploy the completed change.** Run `bob plugins sync` once the phase's file changes
   and checks are complete, as required by bob-plugins/AGENTS.md. Wait for its
   completion and record the result. Finalize the modified linked repository through
   `/sase_final`, using a Conventional Commit message such as
   `refactor(test): split block-id-prompt suite`; the host finalizer commits the
   declared repository. Do not use raw git commit.

If a check fails, reproduce the failure on the untouched recorded base before calling it
pre-existing. Preserve evidence of an identical clean-base failure and record it as a
`PROPOSED FOLLOW-UP:` note on bob-cli-4f.2, citing an existing tracking task if known.
Such a failure does not justify leaving this phase open. Fix failures caused by this
split; do not fix unrelated defects as part of this pure refactor.

# Phase completion and lifecycle constraints

The phase is already assigned and in_progress. Never set its status by hand. Do not
create beads manually. Record discovered out-of-scope work only with
`sase bead note bob-cli-4f.2 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. The
epic's land agent triages those proposals.

Immediately before closing, run `sase bead epic-symbols bob-cli-4f.2`. The planning
inspection reported no entries, but this must be checked again. Resolve every remaining
symbol or re-key its Justfile line to a verified still-open parent epic or later phase;
do not close with symbols still keyed to this phase.

After verification and sync succeed (or identical clean-base failures have been
documented under the user's exception), close **only** the requested phase:

```bash
sase bead close bob-cli-4f.2 --note "<original/current count preserved byte-for-byte; focused and independent pass counts; repository check results; file-size results; sync result; epic-symbol cleanup result>"
```

Do not close bob-cli-4f, any ancestor plan bead, or later phases. Instructions from this
tale do not grant a phase worker authority to close its ancestors. Follow the mandatory
`/sase_final` declaration as the last action before the normal final answer; include
each modified repository's commit obligation. Report the phase closure and the concrete
coverage/validation results without claiming later phases are complete.
