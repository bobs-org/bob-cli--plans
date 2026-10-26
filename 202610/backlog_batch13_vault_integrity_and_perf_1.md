---
tier: epic
title: 'Backlog batch 13: vault plugin integrity, popout mark gates, CI flakes, and
  Tasks query cost'
goal: Close five existing tasks by preventing host-to-host overwrites of deployed
  bob-articles files, mirroring the remaining ledger-tools mark gates into Obsidian
  popouts, diagnosing two CI flakes, and reducing native Tasks query time and memory
  cost.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6
phases:
- id: vault-plugin-ignore
  title: Stop vault sync from tracking deployed bob-articles files
  size: medium
  depends_on: []
  description: 'vault-plugin-ignore: resolve bob-cli-cd by fixing the source-of-truth
    ignore rule and coordinating untracking/redeployment across hosts; never untrack
    on one host alone.'
- id: popout-mark-gates
  title: Mirror the remaining ledger-tools mark gates into popout windows
  size: medium
  depends_on:
  - vault-plugin-ignore
  description: 'popout-mark-gates: resolve bob-cli-cb by routing priority, task-tag,
    freshness, and Today-off body classes through the shared popout mirror, testing
    both toggle states, and deploying after the ignore fix.'
- id: capture-kick-flake
  title: Diagnose the capture ref-jobs kick timing flake
  size: large
  depends_on: []
  description: 'capture-kick-flake: resolve bob-cli-5o by capturing the failure mode
    under full-suite load and fixing the demonstrated cause without blindly widening
    the timeout.'
- id: refs-refresh-flake
  title: Diagnose the RefsLibrary extra follow-up call on macOS CI
  size: large
  depends_on: []
  description: 'refs-refresh-flake: resolve bob-cli-67 by explaining the extra ref
    list call on the same trigger pattern, then fixing refresh/backfill sequencing
    or test setup without weakening its expectation.'
- id: tasks-query-cost
  title: Reduce native bob query --tasks time and memory cost
  size: large
  depends_on: []
  description: 'tasks-query-cost: resolve bob-cli-6k by profiling the native Tasks
    query path, finding the source of repeated note reads and memory amplification,
    and proving improvements with regression coverage.'
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.land
proposed_with: codex/gpt-6-luna
create_time: 2026-10-26 10:47:08
status: wip
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch13_vault_integrity_and_perf_1.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch13_vault_integrity_and_perf_1.md)
- **PARENT:** [202610/backlog_batch12_perf_baseline_fakenode_daystrip_blocks_notice.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch12_perf_baseline_fakenode_daystrip_blocks_notice.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.md)

# Backlog batch 13: vault plugin integrity, popout mark gates, CI flakes, and Tasks query cost

This epic continues the bounded backlog campaign. Batch 12 closed bob-cli-ca,
bob-cli-8z, bob-cli-c7, bob-cli-c6, and bob-cli-c8. Its lander filed bob-cli-cd while
checking the vault plugin ignore rules.

A fresh no-limit task enumeration after batch 12 closed shows 117 unresolved tasks: 113
ready and four snoozed (3o, 6q, aa, ah). Assign these five existing task beads, reused
and never cloned:

- bob-cli-cd: vault sync tracks deployed bob-articles files, so one host's plugin sync
  can overwrite another host's deployed files. Fix the ignore rule and coordinate
  untracking and redeployment across hosts.
- bob-cli-cb: four remaining ledger-tools mark gates are main-window-only; 9q's shared
  BobLedgerToolsMarkGateMixin provides the popout mirror to reuse.
- bob-cli-5o: capture_with_kick_returns_before_the_clip_finishes failed once under
  full-suite load and passed on the unchanged tree in isolation. Its cause is
  unconfirmed.
- bob-cli-67: RefsLibraryTests.testTriggersDuringRefreshRunExactlyOneFollowUp made one
  extra ref list call on one of three attempts on the same SHA, then passed twice. Do
  not weaken the assertion.
- bob-cli-6k: native bob query --tasks took 6.3-7.1 seconds and about 1.58 GB RSS for
  trivial filters on a 6,234-note vault; the lazy-sandbox fix for 33 is already in
  place.

## Ranking and ownership

The new cd report is a silent cross-host overwrite path. The cb gate mismatch is
deterministic and has a known shared implementation pattern. The 5o and 67 flakes have
full-suite or same-SHA evidence; 67 has the stronger same-SHA reproduction. The 6k
performance report has independent corroboration and substantial interactive and memory
cost. The current ownership audit found no active plan owner for these five tasks.

The hostile-process alert bob-cli-8r and vault-sync halt alert bob-cli-9x outrank
ordinary backlog work. They each require their own large planned launch and must not be
folded into this batch as a phase; the next lander must preserve that priority and plan
them separately. Other deferred candidates include bob-cli-az until active epic c9
lands, bob-cli-8q and bob-cli-b9 while active epic 3j owns related completion work,
bob-cli-b4 pending Bryan's choice about the Multivisor UI, and bob-cli-ax pending
confirmation that clipster is retired.

## Phase details

### vault-plugin-ignore

The vault's custom-plugin ignore block includes the other custom plugins but omits
.obsidian/plugins/bob-articles/. git check-ignore finds no matching rule, and git
ls-files shows main.js, manifest.json, and styles.css tracked. Vault sync therefore
commits per-host deployed plugin files, allowing one host's version to overwrite
another's. The ignore source is managed outside bob-plugins. Use /sase_repo before
touching that repository. Determine the canonical source and coordinated safe untracking
sequence; never run git rm --cached on one host in isolation. Ensure all configured
hosts can deploy their own bob-articles copy after the ignore rule is active. Record
which hosts were covered and any host that could not be coordinated.

### popout-mark-gates

Reuse BobLedgerToolsMarkGateMixin for bob-priority-marks, bob-task-tag-marks,
bob-fresh-marks, and bob-today-marks-off. Register each family once with its effective
state; for the Today opt-out class, pass the off state as enabled. Add tests showing
each class holds in popout windows in both toggle states. Run npm run build, npm test,
and npm run validate, then deploy with bob plugins sync only after vault-plugin-ignore
has completed. Record the deployed plugin version. bob-dep-chips is out of scope unless
a CSS rule keyed on it appears.

### capture-kick-flake

The test is
tests/cli/capture/ref.rs::capture_with_kick_returns_before_the_clip_finishes. It runs
bob capture with ref-jobs enabled and a 20-second sleep adapter, expects return in under
10 seconds, and checks that one file was queued. Prior observations show a full just
check failure and unchanged-tree isolated passes; the failing assertion was not
captured. Reproduce under parallel load, including the reported 64-thread CLI test with
a concurrent build when practical. Record the assertion, elapsed time, process
placement, queue result, and pipe behavior before selecting a fix. Keep the behavior
contract and meaningful timing guard; do not simply raise the 10-second budget. Run just
check, never just check-full.

### refs-refresh-flake

The affected test is
Tests/BobMacCaptureTests/RefsLibraryTests.swift::testTriggersDuringRefreshRunExactlyOneFollowUp.
On macOS CI run 38010286522, the same SHA failed once with five ref list calls instead
of four and passed twice. A plausible cause is the follow-up starting before git-date
backfill publishes, but that remains unconfirmed. Reproduce or instrument the
backfill/refresh order on macOS, identify the source of the extra call, and test the
corrected sequencing. Do not change the expected call count or widen a timeout without
evidence. If a macOS toolchain is not available, document the limitation and propose the
smallest supported follow-up rather than claiming a fix.

### tasks-query-cost

The report measured about 6.3-7.1 seconds and 1.58 GB RSS for native bob query --tasks
over a 6,234-note, 13.5 MB live vault; it opened 12,467 Markdown paths. bob freshness
list on the same vault took about 1.62 seconds and 35 MB. The root cause is unknown and
the bob-cli-33 eager sandbox fix is already present. Profile index construction, note
reads, task cloning, and output assembly before changing implementation. Keep query
semantics and output contract, add regression coverage, and report wall time plus peak
RSS before and after using a comparable vault scale. Do not create a new live-vault
mutation.

## Acceptance

- bob-cli-cd, bob-cli-cb, bob-cli-5o, bob-cli-67, and bob-cli-6k are closed normally
  with verification evidence. If Bryan declines one, close it as canceled with his
  decision.
- The batch-13 plan is validated and proposed through sase plan propose; its reference
  and ownership are recorded on the batch-12 epic bead.
- Commit every changed repository, including the plans sidecar, through
  /sase_git_commit.

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
