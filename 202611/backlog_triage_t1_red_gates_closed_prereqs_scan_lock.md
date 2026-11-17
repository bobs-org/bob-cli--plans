---
tier: epic
title: 'Backlog triage T1: bob-plugins red gates, closed-prerequisite reopen, highlights-scan
  lock window'
goal: 'bob-plugins npm test has no deterministic reds, capture never reopens a closed
  or archived prerequisite, a writing highlights scan holds bob_sync.lock only while
  it writes, and the lander chains the next five-bead triage epic until every open
  task bead is closed.

  '
phases:
- id: plugin-reds
  title: Fix the deterministic bob-plugins npm test reds
  depends_on: []
  size: medium
  description: 'plugin-reds: update the stale walkMode v5 and hex-guard pins (bob-cli-md),
    stop Platform shadowing from breaking platform tests (bob-cli-me), defuse the
    block-marks fixture date bomb (bob-cli-m8), and close all three.'
- id: closed-prereqs
  title: Never promote closed or archived prerequisites
  depends_on:
  - plugin-reds
  size: medium
  description: 'closed-prereqs: filter capture promotion seeds by graph.open, guard
    the rewrite loop, add closed/done/cancelled prerequisite tests, and close bob-cli-jh.'
- id: scan-lock
  title: Narrow the highlights-scan bob_sync.lock window
  depends_on:
  - closed-prereqs
  size: large
  description: 'scan-lock: per scan_lock, plan the library lock-free and lock only
    for intake and writes with re-validation, preserving the 9c.1 guarantee, then
    resolve bob-cli-mo.'
decisions:
  scan_lock:
    ask: Should this epic narrow how long a writing highlights scan holds bob_sync.lock?
    choices:
      narrow: Plan lock-free and lock only for intake and writes; lock_busy shrinks
        to the write phase
      defer: Skip the scan-lock phase; bob-cli-mo stays open for a later batch
      cancel: Skip the scan-lock phase and close bob-cli-mo as canceled; plugin retry
        is enough
    default: narrow
    why: On the Mac every 3 s-budget writer fails lock_busy for minutes, four times
      an hour
    answer: narrow
proposed_by: bbugyi200.athena.1dh
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 11:16:42
status: wip
bead_id: bob-cli-mr
---

- **PROMPT:** [prompts/202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md)
- **BEAD:** [bob-cli-mr](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mr/README.md)

# Backlog triage T1: bob-plugins red gates, closed-prerequisite reopen, highlights-scan lock window

## Context

This is the first epic of a chain that works through every open task bead in bob-cli, at
most five beads per epic, one phase at a time. Before this plan was written, a
2026-11-17 triage pass did two things:

- It closed 28 task beads that were verifiably obsolete (already fixed, with the fixing
  commit cited in each close note) or duplicates. bob-cli-f2 was folded into bob-cli-gd.
- It added a note starting `2026-11-17 triage` to 37 of the kept beads. These notes
  record narrowed scope, partial fixes, or open questions for Bryan.

237 task beads remained open after triage, 10 of them snoozed.

This epic takes the five most important of them:

| Bead       | Type    | Why first                                                                                                             |
| ---------- | ------- | --------------------------------------------------------------------------------------------------------------------- |
| bob-cli-md | ci      | Deterministic red in `npm test` on bob-plugins master. Every bob-plugins landing in this chain hits it.               |
| bob-cli-me | ci      | Deterministic red in `npm test` on bob-plugins master (Platform shadowing).                                           |
| bob-cli-m8 | ci      | Deterministic red in `npm test` on bob-plugins master: a date time bomb in the block-marks e2e self-test.             |
| bob-cli-jh | bug     | Silent vault corruption: capture reopens closed and `done/` prerequisites in place.                                   |
| bob-cli-mo | feature | The Mac's writing highlights scan holds `bob_sync.lock` for 80-300 s every 15 min, so other writers fail `lock_busy`. |

The red-gate phase runs first because every later phase and epic in the chain lands
through these gates.

Each phase closes the task beads it fixes. Close with
`sase bead close <id> --note "<evidence: commit SHA, tests run>"` once the fix is pushed
and verified. Phase workers never create beads; record `PROPOSED FOLLOW-UP:` notes on
the phase bead instead.

## Phase plugin-reds (bob-cli-md, bob-cli-me, bob-cli-m8)

Work happens in the linked `bob-plugins` repo. Open it with `/sase_repo` and use only
the path it prints. Read each bead with `sase bead read` first, including its +1
evidence: the line numbers there are current as of bob-plugins master 9ae8142.

1. **bob-cli-md.** Commit d0726f6 shipped nav `api.walkMode` v5 and hex fallbacks for
   the attention tiers, but the tests still pin the old contract.
   - Update the pinned walkMode expectations in
     `scripts/test-navigation-bracket-repeat.cjs` to v5. There are three nodes, around
     lines 644, 755 and 771. Also update the daily-aliases node in
     `scripts/test-navigation-period-links.cjs`.
   - Confirm that any v5 shape changes are asserted, not merely tolerated.
   - In `scripts/test-navigation-hotkeys-priority-notice.cjs:351`, either stop the hex
     guard at the next section marker or allowlist the attention-tier fallbacks. The
     test already allowlists `#d9822b` / `#3f8f5f`.
   - Do not weaken the guard for the priority-notice section itself.
2. **bob-cli-me.** c725370 added a module-level `const Platform = obsidian.Platform;` in
   `plugins/bob-navigation-hotkeys/src/010-requires-and-config.js`. It shadows
   `globalThis.Platform`, so three platform tests can no longer simulate mobile or
   macOS: `test-navigation-bracket-mode-indicator.cjs:330` and
   `test-navigation-highlights-hints.cjs` nodes near 397 and 655.
   - Prefer the smallest correct fix. Either the tests set the platform through the
     harness's `obsidian.Platform` stub, or runtime code reads Platform through the
     guarded lookups that the bracket-mode and highlights helpers already use.
   - If runtime source changes, check that `windowPrefixIsMacPlatform` and the
     tab-numeral phone gate still behave. Run the window-prefix suites.
3. **bob-cli-m8.** `scripts/obsidian-block-marks-e2e.cjs:1093` scans the generated
   fixture for the literal `2026-11-20`. Relative dates such as `addDays(today, N)`
   collide with that literal on some calendar days.
   - Replace the byte scan with a check that the fixture is built from relative date
     helpers. One way is to assert that no contract-date literal appears in the
     generator source, or to compare against a fixture generated with a fixed fake
     "today".
   - Prove the self-test passes with today pinned to 2026-11-16 and to 2026-11-17, and
     on a clock far from either date.
4. **Verify.** Run each touched test file alone, then the full `npm test`.
   - Every remaining failure must be an already-tracked flake: bob-cli-3w, ju, ki, io,
     km, f8, ej, m1, lp, ks. Name each one in the phase notes.
   - Any new deterministic red goes in a `PROPOSED FOLLOW-UP:` note.
   - If plugin runtime source changed, deploy with `bob plugins sync`, never by
     hand-editing `~/bob`.
5. Commit and push bob-plugins. Close bob-cli-md, bob-cli-me and bob-cli-m8 with the
   commit SHA and the test evidence.

## Phase closed-prereqs (bob-cli-jh)

All work is in bob-cli. The bead's description has the full root-cause trace. Line
numbers in it date from 2026-11-10, so re-locate them.

1. In `src/native/capture/dependencies.rs`, filter the promotion seeds by `graph.open`
   on both non-`^prj` paths: the existing-dependent path
   (`else { merged_identities.clone() }`) and the new-task path (`&merged_identities`).
   This matches what the `^prj` branch already does.
2. Add a defensive `graph.open` check in `plan_promotions`' final rewrite loop, so a
   closed seed can never reach `replace_status_char`.
   - `lane_for_status` returns `None` for `x`/`X`/`-`, which is how closed rows
     currently slip through.
   - Archive rows (`is_archive_path`) must also never be rewritten. The BLOCKS mirror's
     skip shows the pattern.
3. Add capture tests in which a Next (`[*]`) dependent and an In Progress (`[/]`)
   dependent gain a dependency on:
   - (a) a closed `[x]` prerequisite in a live area note;
   - (b) an archived `[x]` prerequisite in `done/`;
   - (c) a cancelled `[-]` prerequisite.

   In every case the prerequisite line stays byte-identical, the Depends-On line is
   still written, and open prerequisites are still promoted as before.

4. Reproduce end to end in a temp vault with `bob capture --dry-run`, using the repro in
   the bead, before and after the fix.
5. Check that `docs/task-dependencies.md` already states the contract ("promotes the
   task's open prerequisites" and "Closed lines are never edited") and that no doc
   claims otherwise.
6. Check whether bob-plugins has a parallel JS promotion writer with the same seed rule.
   If it does, fix it here only when the fix is mechanically identical; otherwise record
   a `PROPOSED FOLLOW-UP:`.
7. Run `just check`. Close bob-cli-jh with the commit SHA and test names.

## Phase scan-lock (bob-cli-mo)

> [!decision] scan_lock = narrow Implement the change below.

> [!decision] scan_lock = defer Make no code changes. Note the decision on bob-cli-mo,
> leave it open, and close the phase bead.

> [!decision] scan_lock = cancel Make no code changes. Close bob-cli-mo as `-R canceled`
> with the decision as the reason, and close the phase bead.

This is a lock-protocol change, so the phase is `large` and plans before it implements.
Read bob-cli-mo in full. Read closed phase bob-cli-9c.1 for the guarantee it must
preserve: vault-sync can never push a library PDF without its ref note. Also read
`decisions:open-pdf-writes-are-coordinated` and
`decisions:references-are-ingested-at-birth`.

1. In `src/native/highlights_ref/sync.rs::scan_library`, move the pre-scan hook, xlib
   intake discovery, and the parallel per-PDF planning ahead of
   `acquire_scan_maintenance_lock` and `scan.lock`. Dry runs already take neither lock;
   this keeps them unchanged.
2. When the plan has no intake and no writes, return `writes: none` without taking
   either lock.
3. When there is work, take `bob_sync.lock` then `scan.lock`, in the existing order.
   Re-validate the plan under the lock, because a PDF, ref note, or annotation
   fingerprint may have changed since planning. Then perform intake and writes. Re-plan
   only the stale items, or the whole scan if that is simpler and still bounded.
   - If the design needs a different lock order, justify it in the plan and in
     `docs/vault-git-sync.md`.
4. Keep the `lock_busy` JSON shape and the lock path it names unchanged for clients: the
   Mac cron, `host_catalog::run_locked` callers, and the plugin retry.
5. Tests:
   - A no-write scan takes neither lock. One way to show this is to hold `bob_sync.lock`
     from the test and see the scan still finish.
   - A writing scan still takes both locks in order.
   - A plan invalidated between planning and locking is re-validated, not blindly
     written.
   - The 9c.1 PDF-without-note guarantee still holds.
6. Update the docs that state the lock window. Check `docs/vault-git-sync.md`, the
   highlights or ref scan docs, and `docs/` contract text. If the change contradicts
   `decisions:open-pdf-writes-are-coordinated` or another decision, do not edit the
   record. Record a `PROPOSED FOLLOW-UP:` for a superseding decision instead.
7. Run `just check`. Close bob-cli-mo with the commit SHA and the evidence.

## Landing

The land agent:

1. Verifies each phase's beads are closed, or left open as the `scan_lock` decision
   directs. Spot-check the evidence: the bob-plugins red nodes now pass, and the bob-cli
   tests named in the close notes exist and pass.
2. Triages every `PROPOSED FOLLOW-UP:` note into task beads through `/sase_new_task`,
   which catches duplicates.
3. **Continues the chain.** List the open task beads with `sase bead list -t task -n 0`.
   Leave out snoozed beads that have not reached their wake time, and beads already
   owned by an active epic. If any eligible beads remain:
   1. **Re-triage.** Close every bead that is verifiably obsolete or duplicate. Concrete
      evidence means a fixing commit SHA, a current `file:line`, a passing targeted
      test, an existing memory key, or the duplicate bead id; "probably fixed" is never
      enough. Use the default resolution for already-fixed beads, `-R superseded` for
      duplicates and overtaken designs, and `-R canceled` for no-longer-applicable
      beads. Read each bead's `2026-11-17 triage` note, and any later triage notes,
      first. Parallel read-only helper agents are fine for this sweep; the land agent
      does the closing.
   2. **Pick the five most important** remaining beads, in this priority order:
      1. wrong or lost vault writes;
      2. deterministic red gates on master;
      3. high-+1 flakes that keep blocking landings;
      4. daily-workflow pain;
      5. everything else.

      Several beads may share one phase, but there are at most five beads per epic.

   3. **Author the next epic plan with `/sase_plan`**:
      - at most five phases, run strictly in sequence (each phase depends on the one
        before);
      - a `decisions:` block kept to a minimum, but one that lets Bryan opt out of work
        that may not be wanted, especially feature beads, UX polish, and memory beads
        (for example the "Record the decision …" beads, and beads whose triage notes say
        they were overtaken or need re-scoping);
      - this whole Landing section copied forward unchanged, so the chain continues.

      Then `sase plan propose` it. When it reports Staged, finish through `/sase_final`.

   4. If no eligible beads remain, say so in the final report. List any snoozed beads
      with their wake times, and stop the chain.

### Suggested next queue (non-binding; re-triage first)

- **Bugs with wrong vault writes:**
  - bob-cli-k9: a notebook item led by a bullet writes a checkbox task.
  - bob-cli-h1: offset Pomodoro starts just after midnight are misdated.
  - bob-cli-ht: an `x=` record names the wrong Pomodoro, and debug builds panic.
  - bob-cli-l0: vault-sync's clobber identity relies on `created:` alone.
- **Daily workflow:**
  - bob-cli-6g: `@route` capture ignores `project_name_aliases` and creates a stray
    note.
  - bob-cli-8a: unknown capture flags are swallowed as task text.
  - bob-cli-ia: a plain Gmail Reply ignores Reply-To.
- **Shared-root-cause flakes:** bob-cli-3w (+30), ju (+18), ki (+17), io (+13) and km
  (+11) are single-sample wall-clock budgets under parallel `npm test`. bob-cli-f8
  (+18), ej, m1 and lp are related. One measurement fix may clear most of them, so a
  whole epic could take this cluster.
- **Docs caps blocking edits:** bob-cli-il (the ledger-tools page is at 6000/6000 words,
  +9) and bob-cli-ma (the task-pickers page is at 5999/6000).
