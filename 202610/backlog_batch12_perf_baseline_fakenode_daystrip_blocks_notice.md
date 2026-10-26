---
tier: epic
title: 'Backlog batch 12: capture-related perf baseline, FakeNode assert OOM, Day
  strip new-tab and legend overflow, and the BLOCKS-line ! notice'
goal: Close five backlog tasks by making the capture_related lib perf gates load-tolerant,
  making failing FakeNode asserts report instead of exhausting the heap, letting a
  modified click on the Day strip open a new Stats tab, keeping long Day strip legend
  labels from painting over each other, and refusing ! on a generated BLOCKS line
  with the documented notice.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6
phases:
- id: capture-related-perf-baseline
  title: Give the capture_related synthetic-vault lib perf gates a paired baseline
  size: small
  depends_on: []
  description: 'capture-related-perf-baseline: resolve bob-cli-ca by moving both src/native/capture_related
    synthetic-vault wall-clock gates onto the warmed, paired-baseline pattern from
    0e75ca56, proving they still catch a real slowdown and no longer fail under parallel
    just check load.'
- id: blocks-line-bang-notice
  title: Refuse ! and N! on a generated BLOCKS line with the BLOCKS notice
  size: small
  depends_on: []
  description: 'blocks-line-bang-notice: resolve bob-cli-8z by making the bob-navigation-hotkeys
    ! / N! handler show BLOCKS_GENERATED_LINE_NOTICE on a generated BLOCKS line, with
    a test and a bob plugins sync deploy.'
- id: fakenode-inspect
  title: Make failing FakeNode asserts in ledger-tools suites report instead of OOM
  size: small
  depends_on: []
  description: 'fakenode-inspect: resolve bob-cli-c7 by giving every bob-plugins ledger-tools
    test FakeNode a short util.inspect.custom descriptor, restoring the plan-host
    identity asserts, and adding a bounded-time failing-assert regression.'
- id: day-strip-new-tab
  title: Open a new Stats tab on a modified Day strip click even when Stats is open
  size: small
  depends_on:
  - fakenode-inspect
  - blocks-line-bang-notice
  description: 'day-strip-new-tab: resolve bob-cli-c6 by making openStatsForDay skip
    the reuse branch for newLeaf requests, with a host test, the docs/stats.md wording
    fix, and a bob plugins sync deploy.'
- id: day-strip-legend-overflow
  title: Keep long Day strip legend labels from painting over each other
  size: medium
  depends_on:
  - day-strip-new-tab
  description: 'day-strip-legend-overflow: resolve bob-cli-c8 by keeping each Day
    strip legend item at its content width and moving whole items that do not fit
    into the overflow counter, with a preview-harness check that catches the overlap
    and a bob plugins sync deploy.'
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.land
proposed_with: claude/opus
create_time: 2026-10-26 08:32:06
status: wip
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch12_perf_baseline_fakenode_daystrip_blocks_notice.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch12_perf_baseline_fakenode_daystrip_blocks_notice.md)
- **PARENT:** [202610/backlog_batch11_ref_repair_taskcard_popout_gkeep_again.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch11_ref_repair_taskcard_popout_gkeep_again.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.md)

# Backlog batch 12: capture-related perf baseline, FakeNode assert OOM, Day strip new-tab and legend overflow, and the BLOCKS-line ! notice

This epic continues the bounded backlog campaign. Batch 11 (epic
`bob-cli-an.6.6.6.5.6.5.5.6.6.6`) closed `bob-cli-b8` (the live reference ids were
repaired through Bryan's approved gate), `bob-cli-4e`, `bob-cli-9q`, `bob-cli-2m`, and
`bob-cli-bn`. Its lander filed `bob-cli-cb` and corroborated `bob-cli-ca` and
`sase-1el`.

A fresh enumeration after that landing shows 121 unresolved tasks: 117 ready and 4
snoozed (`3o`, `6q`, `aa`, `ah`). Assign exactly these five existing task beads, reused
and never cloned:

- `bob-cli-ca` (small, flake): the two `capture_related` synthetic-vault lib perf gates
  fail about half of parallel `just check` runs on athena. The batch-11 lander's first
  `just check` failed only on these tests (774 ms and 698 ms against a 500 ms budget at
  load average 15). They passed when run alone.
- `bob-cli-8z` (small, bug): nav `!` on a generated 🧱 BLOCKS line says "No links found
  on current line" instead of the documented BLOCKS notice.
- `bob-cli-c7` (small, bug): a failing `assert.equal` / `deepEqual` on two live
  FakeNodes exhausts the heap instead of reporting.
- `bob-cli-c6` (small, bug): Ctrl/Cmd-click or middle-click on the Day strip never opens
  a new Stats tab while Stats is open anywhere.
- `bob-cli-c8` (small, bug): long Day strip legend labels paint over each other in the
  wide layout, and the preview harness reports `pass`.

Ranking evidence:

- `ca` is the recurring CI flake with a known remediation and the most corroboration:
  three phase reports plus two independent +1s.
- `c8` is a visible defect on every daily note whose top roots have long labels.
- `c6`, `8z`, and `c7` are deterministic defects with source-confirmed root causes. The
  batch-11 lander re-checked each against current source. `c6` became actionable once
  epic `bob-cli-c5` landed.
- No active epic owns these files. Active epic `bob-cli-c9.5.1` only cites `bob-cli-ca`
  as a known flake.

Deferred, with reasons:

- `8r` (athena hostile-process alerting, security) and `9x` (cross-host vault-sync halt
  alerting) are large. They need root-level athena changes and delivery-channel
  decisions. Each belongs in its own planned `sase bead work` launch (large tasks plan
  first), not in a bounded backlog batch. `8r` should reuse the Alertmanager email route
  recorded on its bead.
- `az` (coordinate rename/stamp/`create --force` PDF writes) overlaps active epic
  `bob-cli-c9`, which owns `create --force`. Revisit once `c9` lands.
- `cb` (the remaining popout body-class gates) is low severity and was filed today. It
  is a strong next candidate.
- `b4` (Multivisor web UI 500) needs Bryan to choose between reinstalling the web assets
  and dropping the UI.
- `8q` belongs to active epic `bob-cli-3j`.
- `5m` and `5n` are sase-tool defects.
- `v` needs resizing; its bead has a note.
- The large flakes `5o`, `b9`, and `67` still lack a root cause.

Phase ordering: every bob-plugins phase that deploys runs `bob plugins sync` from its
own checkout. Deploying phases are therefore serialized (`blocks-line-bang-notice` →
`day-strip-new-tab` → `day-strip-legend-overflow`), and each one rebases onto the latest
bob-plugins master before building and deploying. `fakenode-inspect` changes only test
scripts and does not deploy. It precedes the Day strip phases so their new DOM asserts
report readably.

## Phase details

### capture-related-perf-baseline

Baseline: `src/native/capture_related/tests.rs` has two tests.
`synthetic_vault_query_is_under_250ms` starts around line 513, with its assert around
line 539. `synthetic_vault_draft_contract_is_under_250ms` starts around line 553, with
its assert around line 580. Each times one debug-build run of the 40-note × 38-task
synthetic vault and asserts it is under 500 ms. Commit `0e75ca56` (bob-cli-bo) gave the
CLI gate in `tests/cli/capture/related.rs` a load-tolerant design: an untimed warmup,
then paired timed runs against a trivial baseline, gating the minimum difference plus a
gross bound.

1. Read `0e75ca56` and apply the same design to both lib tests. Share the helper between
   them rather than duplicating it. Keep the gate meaningful: the paired minimum stays
   near the current isolated budget, and the gross bound still fails an
   order-of-magnitude regression.
2. Prove both directions:
   - Inject a temporary artificial slowdown (do not commit it) and show the gate fails.
   - Run the two tests repeatedly under parallel load (for example alongside a
     concurrent `cargo test --lib`) and report pass counts.
3. Run `just check`. Never run `just check-full`.

### blocks-line-bang-notice

Baseline:
`plugins/bob-navigation-hotkeys/src/500-plugin-transclusion-and-link-picker.js`. The `!`
/ `N!` handler (around lines 201–209) refuses a Depends-On line with its own notice but
never calls `parseBlocksLine`. On a generated `- 🧱 **BLOCKS:** [[#^t1]]` line,
`findTransclusionToggleTargets` (`180-editor-transactions.js`, around line 407) finds no
targets, so the handler shows "No links found on current line". bob-cli
`docs/task-dependencies.md` §14.8 says `!` / `N!` and Ctrl+Shift+Enter are refused with
nav's `BLOCKS_GENERATED_LINE_NOTICE`. Read the bob-plugins `AGENTS.md`.

1. Before `toggleLineTransclusions`, show `BLOCKS_GENERATED_LINE_NOTICE` and return
   `false` when `parseBlocksLine(lineText, {}).verdict !== "not-a-line"`.
2. Add a test: `!` on a generated BLOCKS line shows the BLOCKS notice and writes
   nothing. `!` on an ordinary link line still toggles. Confirm read-only that
   Ctrl+Shift+Enter already refuses on a BLOCKS line. If it does not, record a
   `PROPOSED FOLLOW-UP` rather than widening this phase.
3. Rebase onto the latest bob-plugins master. Run `npm run build`, `npm test`, and
   `npm run validate`. Deploy with `bob plugins sync` and record the deployed
   bob-navigation-hotkeys version.

### fakenode-inspect

Baseline: seven bob-plugins `scripts/*.cjs` suites declare their own `class FakeNode`
(`grep -l '^class FakeNode' scripts/*.cjs`). Each has `parentNode`, `children`,
`ownerDocument`, and listener maps, so every node graph is cyclic. When a
`node:assert/strict` comparison of two live nodes fails, Node builds the diff from the
whole reachable graph. On an 85-node tree under `--max-old-space-size=1024`, it aborts
with a heap OOM after about 10 s (exit 134). Epic `bob-cli-c5.3` worked around it in
`scripts/test-ledger-tools-day-strip-plan-host.cjs` with `assert.ok(a === b, ...)`
identity checks.

1. Give every FakeNode-style fake DOM class in the ledger-tools suites a
   `[require("node:util").inspect.custom]()` method. It returns a short descriptor: tag,
   classes, child count, and a short text prefix. Include similarly cyclic fakes under
   other names. Keep the change local to each suite. Migrating the suites onto a shared
   harness belongs to `bob-cli-4g` and stays out of scope.
2. Convert the c5.3 workaround identity checks back to `assert.equal` so a failure names
   both nodes.
3. Add a regression: a deliberately failing `assert.equal` on two nodes of an 85-node
   FakeNode tree throws an `AssertionError` within a few seconds under
   `--max-old-space-size=1024`, and its message names both descriptors. Show the
   regression fails (OOM or timeout) without the descriptor.
4. Run `npm test` and `npm run validate`. No plugin source changes, so no build bump or
   deploy.

### day-strip-new-tab

Baseline: `openStatsForDay` in
`plugins/bob-ledger-tools/src/239-plugin-day-strip-fence.js` (around line 101) first
finds an open `dash_stats` leaf through `findOpenMarkdownLeafByPath` (around line 175).
It reveals that leaf and returns, so `opts.newLeaf` only reaches the
`workspace.openLinkText("dash_stats", sourcePath, opts.newLeaf === true)` fallback
(around line 196). `239-plugin-day-strip-render.js` turns Ctrl/Cmd-click and
middle-click into `newLeaf: true`. Every other ledger-tools link passes `newLeaf`
straight to `openLinkText`. bob-cli `docs/stats.md` (around line 949) documents the
current behavior: "An already-open Stats tab is reused; otherwise Ctrl/Cmd-click and
middle-click open a new tab".

1. Skip the reveal-and-reuse branch when `opts.newLeaf === true`. The pinned day and
   focused metric must still apply to the new tab. A plain click keeps reusing an open
   Stats tab.
2. Extend `scripts/test-ledger-tools-day-strip-hosts.cjs`:
   - An already-open `dash_stats` leaf plus `newLeaf: true` opens a new leaf with the
     pinned state.
   - An already-open leaf without `newLeaf` is still reused.
3. In bob-cli, reword the `docs/stats.md` Interaction text: a plain click reuses an open
   Stats tab, and Ctrl/Cmd-click or middle-click always opens a new one. Read the `docs`
   reference memory first and run `just docs-check`.
4. Rebase onto the latest bob-plugins master. Run `npm run build`, `npm test`, and
   `npm run validate`. Deploy with `bob plugins sync` and record the deployed
   bob-ledger-tools version.

### day-strip-legend-overflow

Baseline: in `plugins/bob-ledger-tools/styles.css` (around line 4305),
`.bob-day-strip-legend` is an `overflow: hidden` flex row. Its
`.bob-day-strip-legend-item` children are `inline-flex` with `min-width: 0`. Long items
shrink below their content width, but their nowrap text stays visible and paints over
the next item. The narrow container query hides the rank-3 item and shows the `+N`
counter. `docs/stats.md` (around lines 878–903) promises that labels never ellipsize and
that the counter's `title` lists the hidden roots.
`node scripts/preview-day-strip.cjs --out <dir>` renders the matrix. Its `longLabels`
cell visibly overlaps, yet the matrix reports `pass` with 0px legend overlap: the check
compares the shrunken item boxes, not each item's `scrollWidth`.

1. Keep every visible legend item at its content width, so no item paints over another.
   When the wide legend cannot fit every item, hide whole trailing items. The overflow
   counter must count them and list them in its `title`, matching the narrow behavior.
   Labels never ellipsize.
2. The phase may choose the mechanism: CSS-only, if the counter stays accurate, or a
   measured fit pass after paint. A measured pass must not loop on repaint and must stay
   correct on container resize.
3. Add a harness check that fails if a visible legend item's `scrollWidth` exceeds its
   box or if two visible items intersect. Show it fails on the old CSS for `longLabels`.
   Inspect `longLabels` at 410, 540, and 700 in light and dark.
4. Update `docs/stats.md` only if the documented wide-layout overflow behavior changes,
   and run `just docs-check` if it does.
5. Rebase onto the latest bob-plugins master. Run `npm run build`, `npm test`, and
   `npm run validate`. Deploy with `bob plugins sync` and record the deployed
   bob-ledger-tools version.

## Acceptance

- `bob-cli-ca`, `bob-cli-8z`, `bob-cli-c7`, `bob-cli-c6`, and `bob-cli-c8` are each
  closed normally with verification evidence. If Bryan declines one, it is closed as
  canceled with his decision recorded.
- Epic symbols are clean, the batch-12 plan file says `status: done`, and all changed
  repositories are committed via `/sase_git_commit`.
- Batch 13 passes validation and is proposed through `sase plan propose`, with its
  reference recorded on the batch-12 epic bead.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include:
   - the remaining popout body-class gates `bob-cli-cb`;
   - Athena `bob-cli-b4` and `bob-cli-ax`;
   - `bob-cli-az` once active epic `bob-cli-c9` has landed;
   - flakes `bob-cli-5o`, `bob-cli-b9`, and `bob-cli-67`;
   - a resized `bob-cli-v`.

   Security `bob-cli-8r` and content-loss alerting `bob-cli-9x` outrank these. Each is
   large and should get its own planned launch rather than a backlog-batch phase.
   `bob-cli-8q` belongs to active epic `bob-cli-3j` while that epic owns the PTY
   completion drivers. This is guidance, not a preassigned batch, and concurrent work
   may obsolete it.

4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes **and** live hooks report `legacy_dependency_children: 0`.
   Verify real prerequisites, not just wake timestamps. Work other actionable tasks
   first. If only gated tasks remain, the next bounded epic must own the readiness check
   and supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.

The successor epic must also carry the working rules batch 9 used:

- Phase workers leave target tasks open for the lander and record baseline, fix, tests,
  and limitations on their phase.
- Use `/sase_repo` before touching another repository, and `sase artifact read` for
  artifacts.
- No live vault edits, live feed publishes, or credential handling. There are two
  exceptions:
  - bob-plugins phases deploy only through `bob plugins sync`.
  - A live-vault repair runs only as a command of a Bryan-approved `/sase_gate` turn.
- Build bob through `$CARGO_TARGET_DIR`.
- Follow the daily release train: no version or `CHANGELOG.md` edits.
- Run `just check`, never `just check-full`.
- The lander must commit every changed repository, including the plans sidecar status
  edit, with `/sase_git_commit` before proposing its own successor. This plan explicitly
  authorizes the skill for those obligations.
