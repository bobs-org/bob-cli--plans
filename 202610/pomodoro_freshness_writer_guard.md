---
tier: tale
title: Keep freshness stamps off Pomodoro ledger rows
goal:
  Prevent every freshness writer from stamping Pomodoro ledger rows and repair the
  polluted VIBE session through a Bryan-approved gate.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1
create_time: 2026-10-26 13:13:42
status: wip
---

- **PARENT:**
  [202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1.md)

# Keep freshness stamps off Pomodoro ledger rows

Implement assigned phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1`
(`fresh-stamp-ledger-guard`) from
`plan:202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md`. The underlying
task is `bob-cli-cf`; leave that task and all ancestor beads open for the batch lander.
Close only the assigned phase after its acceptance is met.

## Scope and sizing

This is one medium tale: a reproduced checkbox-reopen bug, two canonical placement
implementations, three small plugin delegation boundaries, existing host harnesses, and
one exact-line repair. The assigned phase has no previous work notes or unfinished
implementation. Source changes, tests, and deployment belong to one coder; the required
repair approval uses a gate turn with an explicit verification continuation, not another
planning cycle.

Honor the final parent decision `theme_field_strip = writers`. Fix the writers; do not
strip inline fields in theme parsing or change theme identity, hashing, catalogs, or
Stats parsing. No memory edits are authorized or needed. No CLI subcommands, flags,
version changes, or CHANGELOG edits are needed. The parent's `legacy_key = pin` decision
belongs to another phase.

## Evidence collected before planning

The primary checkout was clean at `6f2f1932`; the linked bob-plugins checkout was clean
at `c036597c5f05d92a1ee79f821c392c01cfbb5bf0`. Re-read current state before editing,
since other phases may land meanwhile.

The task cites vault commit `dcbe953f` from kellys-mbp, which reopened the session and
stamped its headline. Read-only Stats on 2026-10-26 still reports the row at
`2026/20261025.md:38`, session `1020-1035`, name `VIBE [fresh:: 2026-10-25]`. Worked
totals for 2026-10-25 are 205 minutes; its theme parts contain `VIBE: 65` and
`VIBE [FRESH:: 2026-10-25]: 15`. `theme_hues` contains both `vibe` and
`vibe [fresh:: 2026-10-25]`.

The defect was reproduced without filesystem writes using the existing plugin host
harness, with the actual ledger-tools placement helper adapted to its public
string-returning API:

```javascript
const {
  TaskStatusCyclerPlugin,
  createInMemoryObsidianApp,
  createTextEditor,
} = require("./scripts/task-status-cycler-harness.cjs");
const { freshnessStampLine } = require("./scripts/ledger-tools-harness.cjs");
const daily =
  "## Pomodoros\n- [x] (**1020-1035** [t:: 15m]) — VIBE\n\t- 🍅 [[Tasks#^work]]";
const h = createInMemoryObsidianApp({
  "2026/20261025.md": daily,
  "Tasks.md": "- [x] #task Work ^work",
});
const editor = createTextEditor(daily, { line: 1, ch: 5 });
const plugin = new TaskStatusCyclerPlugin();
plugin.app = h.app;
plugin.getFreshnessStampLine = () => (line) =>
  freshnessStampLine(line, "2026-10-25").line;
await plugin.reopenActivePomodoroTask(
  editor,
  h.app.vault.getAbstractFileByPath("2026/20261025.md"),
  { pomodoroLine: 1 },
);
```

Result: the headline becomes
`- [ ] (**1020-1035** [t:: 15m]) — VIBE [fresh:: 2026-10-25]`, the child loses its
tomato, and the real task reopens with its legitimate fresh stamp. The same reproduction
also stamps `() — VIBE` and the legacy heading `Finished session`.

Confirmed source path in bob-plugins:

- `plugins/task-status-cycler/src/170-plugin-pomodoro.js`: `reopenActivePomodoroTask`
  calls `setActiveCheckboxStatus` for the headline.
- `180-plugin-editor-edits.js`: both generic local checkbox writers call
  `stampFreshnessOnEditorLine`; the Tasks-command path also stamps after a write.
- `190-plugin-targets.js`: `stampFreshnessOnEditorLine` delegates through
  `applyFreshStampLine` in `010-core.js` without checking ledger context.
- `plugins/bob-ledger-tools/src/080-freshness-placement.js`: `freshnessStampLine`,
  `freshnessKeepLine`, and `freshnessSetRefreshLine` accept any open checkbox without
  recurrence. Direct calls reproduce pollution on timed and placeholder rows. Public API
  methods in `230-plugin-freshness-api.js` return these helpers' `.line` values.
- Rust `src/native/freshness/placement.rs::stamp_inner` has the same missing ledger
  exclusion. It serves generic stamps, preserve-keeps seed stamps, and refresh writes.

## Implementation

1. Read the assigned bead and approved parent plan through audited commands. Use
   `/sase_repo` to open bob-plugins and read its `AGENTS.md`; use only the path returned
   in this run. Consult `docs.md`, `obsidian.md`, the Pomodoro glossary, and the
   freshness decision record through `/sase_memory_read` as required.

2. Add a small, explicit ledger-row exclusion to the canonical Rust and JavaScript
   freshness placement entrypoints. Recognize checkbox bodies beginning with a Pomodoro
   timespan and canonical `()` placeholders, including existing compact and
   colon-separated time formats. Reuse existing parser conventions where practical;
   anchor recognition to the checkbox body rather than matching a time mentioned later
   in ordinary task prose. Preserve the supported whitespace, indentation, and
   quote-prefix handling of the placement helpers. Timed ledger entries must be
   protected even when no file or heading context is available.

   All fresh-producing operations must refuse these rows byte-for-byte: generic
   stamping, set/clear refresh, preserve-keeps Rust stamping, and counted/uncounted JS
   keep. Use the existing non-task refusal category for ledger rows and keep public API
   return types unchanged. Already polluted lines remain unchanged by these automatic
   paths; repair is the explicit operation below. Do not alter `markUnblockedLine`
   semantics or other non-fresh writers as unrelated cleanup.

3. Guard the plugin callers as well as canonical placement. The cycler's context-aware
   editor path can use `findPomodorosSectionInLines` and `isPomodoroTaskLine` to skip
   top-level ledger rows, including legacy headings with no recognizable timespan. Cover
   its Tasks-command follow-up, local writes, and fallback metadata rewrite; do not
   merely special-case one reopen call. The actual task sources reached through Task
   Links must continue to receive legitimate freshness stamps and reopen/status updates.

   Audit and guard the shared delegation boundaries in cycler `010-core.js`, navigation
   `470-keydown-and-freshness.js` (`applyFreshStampLine` and `applyFreshRefreshLine`),
   and block-id-prompt `030-tasks-and-dependencies.js`. These should reject recognizable
   ledger rows before invoking even an older or injected stamper. Audit direct API calls
   and context-aware targets in navigation `510-plugin-link-commit-and-lane.js`,
   `520-plugin-lane-links-and-review.js`, `532-plugin-freshness-link-refresh.js`, picker
   batch writes, and decay keep paths for bypasses. Where source context is present,
   exclude top-level rows under `## Pomodoros`; retain nested real tasks and ordinary
   checklist/task rows outside the section. Keep actual field placement centralized in
   ledger-tools. Record the resulting writer inventory with which common guard covers
   each entry; do not rewrite every caller if it already shares a guard.

4. Extend the existing tests with regressions that fail before the fix:
   - Host-level Pomodoro reopen with the real freshness helper: timed row, placeholder,
     and legacy section row; assert exact headline, child-marker cleanup, and normal
     reopened task freshness. Include a linked task in the same daily note so section
     filtering cannot suppress legitimate task writes.
   - Cycler direct/local and Tasks-command stamping paths, plus plain task positive
     controls and missing/throwing API behavior.
   - Shared JS and Rust placement vectors for open/closed timed rows, planned
     placeholders, compact/colon time formats, and existing polluted rows. Run each
     fresh-producing operation against these vectors, checking no metadata is inserted,
     removed, or normalized on refusals.
   - Navigation and block-id delegation guards using an injected stamper that would
     append a field; verify it is not invoked on ledger rows. Cover navigation refresh
     and counted keep paths through the actual shared writer.
   - Ordinary tasks, quoted/nested tasks, and tasks merely mentioning a time range still
     stamp normally; linked task source rows remain eligible.

   Prefer existing files: `scripts/test-task-status-cycler-pomodoro-reopen.cjs`,
   `test-task-status-cycler-freshness.cjs`, `test-ledger-tools-freshness-placement.cjs`,
   `test-ledger-tools-freshness-keeps.cjs`, `test-navigation-stamps.cjs`,
   `test-block-id-prompt-plan-budget-and-freshness.cjs`, and Rust
   `src/native/freshness/placement_tests.rs`. Add a focused test file only if it makes
   the regression clearer, and register it in `npm test` if so.

5. Update `docs/freshness.md`'s refusal contract and placement vectors to describe
   ledger exclusion and the distinction between ledger headlines and real task targets.
   No theme parsing documentation changes or memory changes. In plugins, edit fragments,
   keep each hand-edited fragment within the 1000-line limit, update `fragments.json` if
   a split is necessary, and regenerate `main.js` with `npm run build` rather than
   editing generated files.

## Verification and deployment

Run focused Rust freshness tests using the configured `$CARGO_TARGET_DIR` and the host
regressions above, then `just check` (never `check-full`), `just docs-check`, and in
bob-plugins `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`.
Use `/sase_monitor` before long commands and wait for the handoff command to exit; leave
a continuation that preserves this phase's remaining obligations. For checks failing on
the identical clean base, record reproduction evidence as a `PROPOSED FOLLOW-UP:` note
on this phase and continue; known active epic `bob-cli-cc` owns the ref-code and
source-store baseline issues identified by the parent plan. Do not silently skip
required checks or expand into their fixes.

Before deployment, fetch and rebase bob-plugins onto current `origin/master`, preserving
this turn's edits through supported Git/SASE procedures, resolve any conflicts, and
rebuild. Recheck affected regressions after integrating changes. Deploy only through
`bob plugins sync` and inspect `bob plugins list` for drift. Do not deploy a stale build
over the concurrent relation-index/dependency phases. Follow `/sase_final` for
changed-repository obligations; never run raw git commit. Record code revisions, test
outcomes, and deployment evidence on the phase before the repair handoff so a successor
need not rediscover them.

## Prepare and execute the approved one-line repair

Only after the source fix, checks, and plugin deployment are concrete, use `/sase_gate`
to author the required Bryan-approved repair gate. The parent plan explicitly says: "A
live-vault repair runs only as a command of a Bryan-approved `/sase_gate` turn." Set
`auto: false`; plan approval alone does not authorize this mutation. Explain this source
of the confirmation requirement to Bryan.

Use supported Bob read-only interfaces for live evidence. If inspecting vault repository
files/history directly, open that repository through `/sase_repo` first; never discover
or clone a second live-vault checkout by another route. Resolve the actual live vault
root through supported configuration, not an ephemeral source-repository path. Read the
documented vault synchronization/lock protocol before writing the gate command.

The gate must show the exact removal from the unique target headline:

```diff
-- [x] (**1020-1035** [t:: 15m]) — VIBE [fresh:: 2026-10-25]
+- [x] (**1020-1035** [t:: 15m]) — VIBE
```

Implement a narrow compare-before-write command that locates the exact unique line in
`2026/20261025.md` instead of trusting line 38, preserves every other byte and newline
style, and aborts on a changed/ambiguous preimage. An already repaired exact headline is
an explicit no-op, not permission for a broader cleanup. Observe the existing vault
write/lock protocol and return a structured result with the diff and write outcome. Test
it first against temporary fixture files, including drift, duplicate matches, and a
second invocation. Make preview available without answering the gate. Provide repair and
decline options, with typed results and a verification successor for both; failure must
resume to diagnose and safely re-propose rather than orphaning the phase.

The gate successor verifies the resulting vault commit through the normal vault-sync
workflow (or a specifically approved SASE commit path) and records its hash; never issue
raw git commits or stage unrelated live edits. If waiting for sync is needed, use
`/sase_monitor` with a bounded command and an explicit next prompt. Do not finish a turn
assuming a background command will survive.

After a successful repair run:

```bash
bob stats show worked -a -f json -g day -r 2026-10-25..2026-10-25
```

Verify no polluted theme part or `theme_hues` key remains. With the current unchanged
day's data, `VIBE` should be 80 minutes and total worked time should remain 205 minutes;
compare with a fresh pre-repair baseline if unrelated live activity changed the data.
Verify session name `VIBE`, unchanged duration and task children, and no other ledger
freshness pollution. Do not change the parser to obtain this result. Record human
decline distinctly, leave the live line as requested, and give the batch lander evidence
to resolve `bob-cli-cf` according to the parent's decline policy.

## Completion

Record the reproduced root cause, writer audit, code changes, test results, deployment,
and repair gate outcome/commit/Stats comparison on the assigned phase. Before closing
run:

```bash
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1
```

Resolve each listed symbol or re-key its justfile exemption to a still-open owning bead.
Then close only this phase with a verification note:

```bash
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1 --note "<verified fix, tests, deployment, and approved repair outcome>"
```

Leave `bob-cli-cf`, the parent epic, and all ancestors to their land agents. Do not
hand-edit status or create task beads. Discovered work belongs in `PROPOSED FOLLOW-UP:`
notes on the assigned phase. A human decline of the repair must be explicit in the close
note; it is never reported as a repaired vault. The final turn uses `/sase_final` as its
last action unless it successfully hands off through a supported terminating workflow.
