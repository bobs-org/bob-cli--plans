---
tier: epic
title:
  "Backlog batch 11: gated live ref-id repair, Task Card close cleanup, popout mark
  gates, gkeep stdin EPIPE, and start-name again order"
goal:
  Close five backlog tasks by repairing the live vault's reference ids only through a
  Bryan-approved gate, restoring the Task Card close cleanup, showing ledger marks in
  popout windows, removing the gkeep login stdin broken-pipe race, and listing repeated
  start names newest-first.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6
phases:
  - id: ref-id-live-repair
    title: Repair the live vault's reference ids through a Bryan-approved gate
    size: small
    depends_on: []
    description:
      "ref-id-live-repair: finish bob-cli-b8 by dry-running the bob ref rename repairs
      for the remaining live reference-id warnings and running them only through a
      Bryan-approved /sase_gate turn, recording an explicit decline instead when he
      rejects."
  - id: task-card-close-cleanup
    title: Restore the Task Card cleanup in BulletPropertyPickerModal.onClose
    size: small
    depends_on: []
    description:
      "task-card-close-cleanup: resolve bob-cli-4e by folding the dead Task Card class
      removal and flag resets into the live onClose in bob-navigation-hotkeys
      src/350-picker-task-card.js, with a focused test and a bob plugins sync deploy."
  - id: popout-mark-gates
    title: Show ledger progress, unblocked, and date marks in popout windows
    size: small
    depends_on:
      - task-card-close-cleanup
    description:
      "popout-mark-gates: resolve bob-cli-9q by making the bob-ledger-tools
      progress-mark, unblocked-glyph, and Tasks date-mark body-class gates hold in
      Obsidian popout windows for both toggle states, with tests and a bob plugins sync
      deploy."
  - id: gkeep-stdin-epipe
    title: Tolerate a closed child stdin in the gkeep login test helper
    size: small
    depends_on: []
    description:
      "gkeep-stdin-epipe: resolve bob-cli-2m by making tests/gkeep_auth.rs run_login
      (and any sibling stdin-writing gkeep helper) tolerate BrokenPipe while keeping the
      exit-code, hint, and no-call assertions."
  - id: start-name-again-order
    title: List repeated completed Pomodoro names newest-first in start-name again rows
    size: small
    depends_on:
      - popout-mark-gates
    description:
      "start-name-again-order: resolve bob-cli-bn by making today_start_rows order again
      rows by latest occurrence, with an A, B, A regression in capture-complete JSON and
      the shell presenter, and shared vectors kept byte-identical with bob-plugins."
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.land
proposed_with: claude/opus
create_time: 2026-10-25 11:34:34
status: wip
---

- **PROMPT:**
  [prompts/202610/backlog_batch11_ref_repair_taskcard_popout_gkeep_again.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch11_ref_repair_taskcard_popout_gkeep_again.md)
- **PARENT:**
  [202610/backlog_batch10_port_help_doctor_mermaid_startpending.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch10_port_help_doctor_mermaid_startpending.md)

# Backlog batch 11: gated live ref-id repair, Task Card close cleanup, popout mark gates, gkeep stdin EPIPE, and start-name again order

This epic continues the bounded backlog campaign after batch 10 closed `bob-cli-c3`,
`bob-cli-c4`, `bob-cli-9i`, and `bob-cli-4k` and epic `bob-cli-an.6.6.6.5.6.5.5.6.6`.
The batch-10 lander also fixed two epic-caused ref-doctor defects (`ba523746`,
`41cd3748`). A fresh enumeration shows 123 unresolved tasks: 119 ready and 4 snoozed
(`3o`, `6q`, `aa`, `ah`). That count includes the new follow-up `bob-cli-c6`. Assign
exactly these five existing task beads, reused and never cloned:

- `bob-cli-b8` (medium, feature): the doctor half shipped in batch 10. Only the
  live-vault repair remains, and the working rules require Bryan's explicit approval for
  it.
- `bob-cli-4e` (small, bug): `BulletPropertyPickerModal.onClose` never runs the Task
  Card cleanup.
- `bob-cli-9q` (small, bug): ledger mark CSS gates hide marks in Obsidian popout
  windows.
- `bob-cli-2m` (small, flake): `gkeep_auth` login test panics with a broken pipe writing
  stdin.
- `bob-cli-bn` (small, bug): start-name again rows list a repeated name at its
  first-seen position.

Ranking evidence:

- `b8` is identity repair of live data and was already in flight.
- `4e`, `9q`, and `bn` are deterministic defects with source-confirmed root causes. The
  batch-11 lander re-checked each against current source.
- `2m` is a known-root-cause test race.

Deferred, with reasons:

- `8r` (athena hostile-process alerting) is large hardening that needs root-level athena
  changes.
- `c6` waits for active epic `bob-cli-c5`, which edits the same `openStatsForDay`.
- `5m` and `5n` are sase-tool defects.
- `v` grew to 357 clippy warnings and needs resizing; its bead has a note.
- Large flakes `5o`, `b9`, and `67` still lack a root cause.

## Phase details

### ref-id-live-repair

Baseline: `bob ref doctor` gained a `reference ids` row in `89945148`. `41cd3748` made
it accept the decision-allowed `<stem>_N` suffix
(`decisions:reference-ids-follow-file-stems`). A read-only
`bob ref list -f json -A -R all` on the live vault (vault HEAD `74d714e3`) reports two
required repairs:

1. `ref/papers/BryanBugyi_CV_extended_v2.md` records id `BryanBugyi_CV_extended`. That
   id duplicates `ref/papers/BryanBugyi_CV_extended.md` and is also a stem mismatch. A
   same-stem `bob ref rename` should re-key it to `BryanBugyi_CV_extended_v2`. An
   earlier attempt refused `not_in_sync`, so recheck it.
2. `ref/chat/obsidian_sync_footprint_audit_bob-cli-1l.md` records id
   `obsidian_sync_footprint_audit_bob-cli-1l.1_20260827`.

`ref/papers/harness_engineering.md` (id `harness_engineering_2`, beside
`ref/blogs/harness_engineering.md`) is now compliant. Renaming its stem to
`harness_engineering_2` is optional tidying that the gate may offer as a separate,
default-off option.

Steps:

1. Read `sase memory read obsidian.md` and both reference-id decisions
   (`decisions:reference-ids-follow-file-stems`,
   `decisions:reference-renames-keep-identity`). Re-run the read-only `bob ref list`
   diagnostic query with a freshly built bob from `$CARGO_TARGET_DIR`. The live set may
   have changed.
2. For each required repair, choose the exact `bob ref rename <REF> <NAME>` command. Use
   note paths as REF, because a duplicate id is ambiguous. Prove each command with `-d`
   (read-only). If a dry run refuses (for example `not_in_sync`), diagnose it read-only.
   Use `bob ref sync -d <pdf>` to see the sync plan. Add any needed write step, such as
   a scoped `bob ref sync`, to the gate's command list instead of running it.
3. Create a `/sase_gate` turn whose approve option runs exactly the dry-run-proven
   commands in order. It should stop on the first failure, then run the read-only
   diagnostic query and return the remaining id diagnostics as JSON. The reject option
   is a no-op with required feedback. The gate notes must list each command, its dry-run
   plan, and the PDF-rewrite cost from the decision record. They must also ask Bryan to
   close those PDFs in Highlights on the Mac before approving, because an open document
   could write back a stale marker. Pick the turn's `next` prompts so the follow-up
   agent records the outcome on this phase bead and closes the phase:
   - Approve: record the gate result and the clean diagnostics.
   - Reject: record Bryan's decline verbatim so the lander closes `b8` as canceled with
     the decision recorded.
4. Leave `bob-cli-b8` open for the lander. No live vault write may happen outside the
   approved gate's commands.

Acceptance: either the approved gate ran and the live diagnostic query shows no required
reference-id warnings, or Bryan's decline is recorded verbatim on the phase.

### task-card-close-cleanup

Baseline: `plugins/bob-navigation-hotkeys/src/350-picker-task-card.js`, around line 726.
The live `onClose` clears `activeBulletPropertyPicker`, cancels a pending route, resets
`pickerOpen`, `vaultStageRefreshId`, and the pending batch, and settles the review
origin. It never removes `bob-task-card-modal` / `bob-task-card-wide` from `modalEl`,
and never resets `linkResolving` or `taskCardDispatching`. That cleanup lived in a
duplicate `onClose` that has been dead since `c0ff974`. Read the bob-plugins
`AGENTS.md`.

1. Fold the class removal and the two flag resets into the live `onClose`, before
   `super.onClose()`, without reordering the existing route-cancel and review-settle
   behavior.
2. Add a focused test that opens the Task Card, closes it, and asserts the classes and
   flags are cleared on that instance.
3. Run `npm run build`, `npm test`, and `npm run validate`, deploy with
   `bob plugins sync`, and record the deployed bob-navigation-hotkeys version.

### popout-mark-gates

Baseline: `plugins/bob-ledger-tools/styles.css` gates
`.bob-date-mark[data-host="tasks"]` (around line 1416), `.bob-progress-mark` (around
1872), and `.bob-unblocked-glyph` (around 1908) on `body:not(.<class>)`. The setup and
toggle methods in `src/266`, `src/268`, and `src/271` add those classes only to the main
window's `document.body`. `src/276-plugin-today-marks-reading.js` already listens for
`workspace` `window-open`.

1. Make each toggle state hold in every window. Mirror the current classes onto each
   popout body on `window-open` and on every toggle, iterating open popout documents.
   Alternatively, use opt-out classes if they keep toggle-off working in popouts.
   Inverting the gate alone would ignore toggle-off in popouts, so do not ship that.
2. Add tests that cover a popout opened before and after a toggle, for both states.
3. Rebase onto the latest bob-plugins master (which includes `task-card-close-cleanup`)
   before deploying. Run `npm run build`, `npm test`, and `npm run validate`, deploy
   with `bob plugins sync`, and record the bob-ledger-tools version.

### gkeep-stdin-epipe

Baseline: `tests/gkeep_auth.rs`, around lines 99-122. `run_login` writes stdin with
`.expect("write stdin")`. When `gkeep.email` is missing, the child exits 2 before
reading, so the write can hit EPIPE.

1. Tolerate `ErrorKind::BrokenPipe` on the stdin write and keep failing on any other
   error. Keep asserting exit code 2, the `gkeep.email` hint, and
   `fake.call_count() == 0`.
2. Apply the same treatment to any other stdin-writing helper in `tests/gkeep_*.rs`.
3. Prove the race deterministically, for example with a child that exits before reading,
   or by repeating the focused test many times. Then run `just check`.

### start-name-again-order

Baseline: `src/native/capture_complete/pomodoros.rs` `today_start_rows`. Around line
286, a repeated key replaces its entry in the first-seen slot
(`again_entries[position] = entry`), and the vector is reversed around line 291.

1. Order again rows by latest occurrence, newest-first, without changing Today ordering
   (PT7).
2. Add an A, B, A regression asserting A precedes B in `bob capture-complete`
   `pomodoro_start_name` JSON and in the shell presenter.
3. If `tests/fixtures/pomodoro_themes/vectors.json` `menu` vectors change, update the
   byte-identical bob-plugins copy in
   `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/vectors.json` in the same
   change. Fix any plugin mirror (`src/279-plugin-pomodoro-theme-mirrors.js`) that
   orders again rows the same way. Rebase onto the latest bob-plugins master and deploy
   with `bob plugins sync` only if plugin source changed.
4. Run `just check` and `just docs-check` if docs change.

Active epic `bob-cli-c5` edits the Pomodoro homes and Day strip code, so rebase before
committing.

## Acceptance

- `bob-cli-4e`, `bob-cli-9q`, `bob-cli-2m`, and `bob-cli-bn` are each closed normally
  with verification evidence.
- `bob-cli-b8` is closed done, with the approved gate's result and clean live
  diagnostics. If Bryan declined, it is closed canceled with his decision recorded.
- Epic symbols are clean, the batch-11 plan file says `status: done`, and all changed
  repositories are committed via `/sase_git_commit`.
- Batch 12 passes validation and is proposed through `sase plan propose`, with its
  reference recorded on the batch-11 epic bead.

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
   - flakes `bob-cli-5o`, `bob-cli-b9`, and `bob-cli-67`;
   - small deterministic plugin bugs `bob-cli-8z` and `bob-cli-c6` (`c6` once active
     epic `bob-cli-c5` has landed);
   - Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and `bob-cli-9x`;
   - a resized `bob-cli-v`.

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
