---
tier: tale
title: Finish and land note-pull capture
goal:
  Repair the remaining note-pull contract and latency gaps, verify integration, and
  close bob-cli-gj.2.1 and parent phase bob-cli-gj.2.
size: medium
proposed_by: bbugyi200.athena.bob-cli-gj.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gj.2.1
status: done
---

- **PARENT:**
  [202611/note_pull_cli.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/note_pull_cli.md)
- **BEAD:**
  [bob-cli-gj.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gj/bob-cli-gj.2.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-gj.2.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-gj.2.1.land.md)
- **COMMITS:**
  - [4db3200](https://github.com/bobs-org/bob-cli/commit/4db3200cd6367c43a252322b66e3c6652e8a7295)
    — fix(capture)\!: finish note-pull contract and landing

# Finish and land note-pull capture

## Scope and starting evidence

Finish only the remaining work in **bob-cli-gj.2.1**, then complete its landing in this
tale. Its parent **bob-cli-gj.2** is a phase of **bob-cli-gj**. This tale has no later
land agent: do not wait for another agent to close these beads.

The landing audit inspected all four closed child phases and every note, their commits
`c673d144`, `a95d5d8f`, `3d1822be`, and `093dcead`, and intervening commits through
`093dcead`. The grammar, single-item planner, explicit picks, batch offer traversal,
additive JSON, task/Pomodoro blocks, real CLI fixtures, and NP1–NP28 coverage exist. Do
not reimplement those phases. Read the latest audit and triage notes with
`sase bead read bob-cli-gj.2.1 -r "Need landing findings and triage"`. Read every child
with `sase bead read <id> -r "Recheck phase scope and notes"`: `bob-cli-gj.2.1.1`, `.2`,
`.3`, and `.4` (each suffix uses the full epic prefix).

Read the actual approved context through audited artifact reads:

- `sase artifact read plan:202611/note_pull_cli.md "Need the CLI acceptance contract"`
- `sase artifact read plan:202611/note_pull.md "Need parent decisions and budget"`
- `sase bead read bob-cli-gj.2 -r "Need parent phase readiness and decisions"`

The current source and fixtures reveal the bounded gaps below. These are code-review
findings, not claims of newly executed tests. The audit did not build this checkout or
rerun its gates; the implementation must establish the regression failures and then
verify the repaired tree.

## Fixed constraints

- Keep the accepted decisions: `lanes = open_lanes`, `empty_pull = refuse`,
  `memory_decision_record = no`, `memory_successor_offer = no`. No memory edits.
- Pending, Next, Ready in that order; no automatic picks, no configuration switch, no
  new command or flag. Preserve existing status, freshness, dependency, and
  staged-transaction helpers. Do not use `plan_task_link` for picks or retire schedules.
- Only capture into synthetic fixtures or an isolated vault copy with explicit `-b`,
  `BOB_DAY_FILE`, `BOB_NOW`, and isolated configuration. Never capture into the live
  vault. Use `/sase_repo` before accessing another repository, including the vault.
- Keep capture JSON additive/unversioned and preserve the **current** grammar revision.
  Revision 8 superseded note-pull's original 7 in `b8688d9e`. Revision 9 is reserved for
  target-run activation; do not downgrade it if that work has landed.
- Do not edit Cargo versions or CHANGELOG. The original grammar subject was
  `feat(capture!):` rather than the required `feat(capture)!:`, although its breaking
  footer is present. Do not rewrite history; declare the final repair commit as
  `fix(capture)!: finish note-pull contract and landing` with a `BREAKING CHANGE:`
  footer restating that a lone `@route` now pulls offers and parses as `note_pull`.

## 1. Repair prerequisite proof and fold classification

In `src/native/capture/note_pull.rs`, `needs_snapshot` currently skips dependency rows
that are future-scheduled, hidden, or already linked. The classification loop then
checks prerequisites **before** these folds, and treats a missing snapshot as blocked.
Thus a source containing only a hidden task with a closed prerequisite reports
`blocked: 1` instead of `hidden: 1`; adding an unrelated visible dependency row changes
that result by causing the snapshot to load. The same defect affects scheduled and
already-planned folds.

Make snapshot acquisition and classification follow the same first-match rules. Known
blocked symbols/project tags can short-circuit; ignored/closed/custom rows must not
trigger proof. Every remaining dependency-bearing row whose fold depends on prerequisite
state needs proof even when a later filter excludes it. Reuse the batch snapshot and
staged overlay; unavailable/incomplete proof remains blocked. Dependency-free pulls and
non-pull drafts retain their no-snapshot guarantees.

`blocked_project_reason` currently counts raw `prj.depends_on.len()` without proving
which prerequisites remain open. Derive its reason using the existing project-block
reason/state helpers and the same completeness rule. Do not say a project waits on two
tasks when one prerequisite is closed. Preserve the approved scheduled-project wording
and never offer/count the source's own `^prj`.

Add focused regressions for closed/open/unavailable prerequisites combined with hidden,
scheduled, and live-linked residents; cover a source whose only dependency row is
filtered, and the same source with another eligible dependency row. Verify fold totals
and unchanged bytes. Add a blocked project with one closed and one open prerequisite,
and staged prerequisite changes. Keep the existing NP5/NP11/NP22 tests.

## 2. Repair selected repeated-pull ownership

In `plan_note_pull_item`, `offers.is_empty()` refuses before frozen-key ownership
deduplication. With a single eligible task and draft `@sase\n\n@sase`, the existing test
covers no picks only. With `-p all`, the first item links the task; the second folds it
already-planned, refuses `nothing_to_pull_in`, and aborts the transaction. Numeric
selection has a successful unselected probe followed by that failing replan. The same
edge can follow an earlier selected hand-off that owns every source row.

Recognize earlier offer ownership by stable pre-pick keys and post-mint block IDs before
deciding whether this is a genuinely empty pull. If all otherwise relevant rows belong
to earlier offers, prune this item/group without a refusal, duplicate write, fold-only
group, or phantom title. A genuinely empty source still refuses atomically, including
one made empty solely by an earlier explicit link that was not an offer. Do not weaken
missing-source or invalid-pick refusals.

Extend the repeated-source test across no selection, `none`, `all`, numeric and key
picks, ID-less minting, and duplicate descriptions. Cover selected hand-off then
note-pull ownership, and an earlier pull followed by a relation offer of the same task,
so cross-group ownership is enforced in both orders. Assert one offer owner, one link
per selected task, stable numbers/keys between probe and submission, and identical
dry-run/real-run semantics. Exercise same-draft creation in a newly staged note and an
existing note, including equal-description rows; ensure only created rows skip
freshness. If those expose an identity bug, fix it using stable row provenance, not
whole-line equality.

## 3. Restore exact human copy and maintain integration

`print_pull_in_hint` in `src/native/capture/output.rs` joins every offer number;
`tests/fixtures/note_pull/np1/human.txt` pins `-p 1,2,3`. The approved parent copy table
explicitly uses the first and last numbers: `-p 1,3 or -p all`, and only `-p 1` for one
offer. Implement that rule for note-pull-only groups, preserving mixed-group copy and
suppressing the hint when picks exist. Add a two-note case whose numbers continue across
the draft. Regenerate actual CLI fixtures with the generator; never hand-author JSON and
call it real output.

Review selected/unselected row output against `docs/task-dependencies.md` §12.20 and the
parent copy table as part of this same small presentation fix. Preserve Pending / Next /
Ready names, source locators, new-session labels and next-up information, and
partial-selection reporting. Extend human assertions beyond all/none to a subset,
choice-required refusal, two notes, and a mixed successor group. Keep unrelated
successor copy unchanged.

Recheck `git log` since `093dcead` and integrate only newly landed interactions:

- `3b1cf138` added clipboard-only children; the three initializer collision notes are
  already fixed by `de01143e`. In `plan_capture_item`'s sub-bullet result literal,
  preserve exactly one **leading** `note_pull: None`. Never move it to the trailing
  fields and recreate the prior rebase collision.
- `cf1e4644` added ordered batch moves, and `b94eec2c` / `b8688d9e` added move-bullet
  parsing/completion. At the audit tip, project-note execution still refuses staged move
  bullets. If execution has landed, test note pulls before/after moves, selected
  ownership, final block/link locations and source/destination write order; use the
  shared move/remap engine. Do not implement another epic's unfinished execution.
- `d1f34842` added target-run parsing behind a disabled production option. Preserve that
  boundary. If activation has landed, verify lone routes remain note pulls, list headers
  still work, and preceding expanded items keep note-pull offer order.
- Attachment origin writes are clip-only; pulls have no clip. Period-tree changes retain
  `day_file_for`/staged day resolution. TODAY, `pomodoro --last`, and merge/split
  contract-only changes did not require a new note-pull path at the audit.

## 4. Meet the existing latency budget without weakening correctness

`tests/fixtures/note_pull/performance.md` records release p95 **82.60 ms** for `@sase`
against the accepted **70 ms** limit on 6,853 Markdown notes; `@mac_inbox` is an empty
refusal at 44.88 ms. This is remaining epic work, not a task to defer. The last phase
reduced catalog reads, then attributed the remaining scan to an eligible
dependency-bearing resident. Its fold optimization also caused step 1; restore correct
proof first and profile that correct path.

Use the existing `tests/fixtures/note_pull/benchmark.py` harness and an isolated source
opened with `/sase_repo`; retain representative data size, route sizes, fixed day/clock,
command, warm-up/sample counts, host and build revision in the report. Inspect
`prepare_snapshot_proof`, `DependencyContext::dependents_snapshot`, and the shared byte
pass for redundant parsing, repeated settings reads, rebuilding indexes or dependency
state. Reuse valid batch-local work; do not introduce a stale cross-process cache, skip
safety proof, weaken the threshold, or benchmark a toy vault as acceptance. Preserve
completeness, staged overlays, and zero additional work on ordinary drafts. Keep
ordinary successor behavior and performance protected.

Refresh the representative release results after repair. Reach p95 <=70 ms for both
routes. If a genuine external limitation prevents measurement or acceptance, record the
precise blocker on bob-cli-gj.2.1 and keep it open; never turn an unmet criterion into a
successful close or quietly move it to unrelated backlog.

## 5. Verify the integrated tree

Run formatting, focused note-pull/parser/CLI tests, then
`cargo check --all-targets --locked` and the required **`just check`**. Do not run
`just check-full`. Use `/sase_monitor` for long commands and carry the remaining
closeout into its continuation; do not declare the bead done just to launch a check.
Test failures caused by the repairs remain this tale's work. A check-full failure after
check passes is test infrastructure, not incomplete epic scope.

Run `just docs-check` for presentation/help/fixture contract changes, reading docs
memory first. Phase .4 reported two attachment published-URL obligations identical on
its clean base, not a note-pull failure. Distinguish a reproduced external failure from
new note-pull documentation regressions; route any newly discovered unrelated work
through `/sase_new_task`. Do not repair unrelated attachments or weaker-test
configuration under this epic. Update the fixture README with actual regression test
names and record producing-tree/fixture details on `bob-cli-gj.3` if its handoff
changed.

## 6. Complete this epic's landing, then its parent phase

All following closeout occurs in the same turn as the final repaired code. It must not
wait for this work's commit SHA, push, or CI result. The host finalizer commits after
the turn; that ordering is normal. No successful nested landing uses `--force`.

1. Reread bob-cli-gj.2.1, all descendants/notes and its linked plan readiness; account
   for the exact tested tree and post-audit drift. Include all triage outcomes below in
   the close note. Keep an unfulfilled item open rather than forcing the close.
2. Run `sase bead epic-symbols bob-cli-gj.2.1`. For every listed entry, resolve the
   symbol (wire it up, privatize it, use a valid non-test pragma, or delete it per
   policy), or re-key only to a still-open later bead that actually needs it. The audit
   found none; check again immediately before closing.
3. Close normally with
   `sase bead close bob-cli-gj.2.1 --note "<concrete source, regression, integration, performance, gate, symbol, and triage verification>"`.
   If symbols prevent closure, finish cleanup and retry. If an unfinished phase is
   named, complete or deliberately reopen it. Never force a successful nested close.
4. Run `just symvision` if the recipe is available (it was absent at audit time). Open
   the plans repository with `/sase_repo`, use audited artifact reads for the plan
   content, and set **`status: done`** in **`plan:202611/note_pull_cli.md`**
   (`202611/note_pull_cli.md` inside that repository). Use the opened path, never a
   guessed sidecar path. Include this repository's edit in `/sase_final`.
5. Run `sase bead read bob-cli-gj.2.1 -r "Need the parent link"` and reread
   `bob-cli-gj.2`. It is a **phase**, not another plan. Verify this child completed its
   CLI grammar/planner/picks/output/NP scope, then close only that parent phase:
   `sase bead close bob-cli-gj.2 --note "<CLI scope and child landing verification>"`.
   Leave containing epic **bob-cli-gj** and its plan to its waiting land agent; do not
   close it or mark `plan:202611/note_pull.md` done. If the parent phase is incomplete
   or ambiguous, record its blocker and report it instead.
6. Finish through `/sase_final`, declaring all modified repositories. Report the actual
   closed IDs, verification and any remaining blocker.

Follow-up triage is already complete and recorded on bob-cli-gj.2.1:

- **bob-cli-gj.2.1.2 note #1:** decline a new lint task. The proposed `nonminimal_bool`
  / `result_large_err` failures used blanket `-D warnings`, which the actual
  `just lint`/`just check` deny list does not use. The audit inspected that expansion,
  searched CI then all task types, swept last week's tasks, inspected active epics and
  the distinct `bob-cli-fh` needless-return report. Phase .4 also reported a later
  canonical pass. There is no established current failure to file or corroborate. This
  is not a claim that the audit ran Clippy.
- **bob-cli-gj.2.1.4 note #3:** keep the latency miss in this epic, addressed in step 4.
  `bob-cli-gc` concerns pre-existing `!` close latency, not this new note-pull path; it
  is not a reason to close this epic with an unmet budget. No new task and no
  unsupported +1 were created.
