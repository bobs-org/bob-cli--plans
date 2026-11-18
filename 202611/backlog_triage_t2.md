---
tier: epic
title: 'Backlog triage T2: notebook safety, ledger identity, midnight dating, and
  clobber evidence'
goal: 'Five remaining wrong-write and capture-identity defects are fixed, their task
  beads are resolved with evidence, and the lander continues the five-task backlog
  chain.

  '
phases:
- id: notebook-safety
  description: 'notebook-safety: reject ambiguous bullet-led @@notebook items before
    writes and close bob-cli-k9.'
  title: Refuse bullet-led notebook drafts before any checkbox write
  size: medium
  depends_on: []
- id: ledger-identity
  description: 'ledger-identity: carry explicit refs through status/name rewrites,
    report the final recorded name, and close bob-cli-dq and bob-cli-ht.'
  title: Preserve Pomodoro preview identity and report the settled name
  size: medium
  depends_on:
  - notebook-safety
- id: midnight-dating
  description: 'midnight-dating: refuse past-offset starts before today while preserving
    valid overnight ends, and close bob-cli-h1.'
  title: Refuse offset starts before the current ledger day
  size: medium
  depends_on:
  - ledger-identity
- id: clobber-evidence
  description: 'clobber-evidence: require evidence beyond a shared created timestamp
    before quarantine and close bob-cli-l0.'
  title: Require corroboration before quarantining a foreign-note clobber
  size: medium
  depends_on:
  - midnight-dating
proposed_by: bbugyi200.athena.bob-cli-mr.land--2
proposed_with: muse/muse-spark-1.3-contributor
create_time: 2026-11-17 19:17:51
status: done
bead_id: bob-cli-my
---

- **PROMPT:** [prompts/202611/backlog_triage_t2.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/backlog_triage_t2.md)
- **BEAD:** [bob-cli-my](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-my/README.md)

# Backlog triage T2

## Scope and triage evidence

This continues T1,
`plan:202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md`, after bob-cli-mr
closes. It is a separate backlog epic, not a child needed to finish T1. Use exactly five
existing tasks: bob-cli-k9, bob-cli-dq, bob-cli-ht, bob-cli-h1, bob-cli-l0. The four
phases run strictly in sequence. No new feature, memory, release, deployment, or
live-vault repair is included. These are correctness repairs with specified behavior;
there is no optional feature branch to ask about.

The T1 lander reviewed the 235 active task rows and available triage notes on 2026-11-17
and the 13 active plan scopes. Exclude the 10 snoozed tasks until their wake conditions,
and do not take work already owned by another active epic. The chosen five are ready,
not snoozed, and no active plan owns these specific defects. Other old active capture
plans are adjacent, not a reason to recreate or claim their work. No additional task was
proven obsolete by the intervening records/scan commits; partial fixes and absence of
recent flake reports are insufficient evidence to close. Recheck task state and later
notes before each phase to avoid duplicate work.

On bob-cli 0e83df4c, a freshly built debug CLI independently reproduces k9, h1, ht and
dq in isolated dry-run vaults. k9 emits a checkbox in the notebook; h1 emits 2350-0005
into the new day's note; ht exits 101 with rewritten headline has no ref; dq exits 101
with entry vanished without a ref. l0 remains source-proven at
`src/native/vault_clobber.rs:243`:
`self.created || self.prj || self.shared_new_ids >= 3`. The historical name
`is_identity` is now `matches`; re-locate by symbol. These five have higher priority
than timing flakes and optional workflow changes because they misdate/write invalid
vault rows, panic capture, or quarantine valid work. dq and ht share tracker machinery
but have distinct triggers and ht additionally misreports the settled name, so retain
both tasks and repair them together.

Read each assigned task in full with `sase bead read ID -r "Need scope and notes"`. Use
its repro and later notes. Never broaden this epic to other tracker or capture features.
Work against the workspace checkout; open any linked repository through `sase_repo`
first. No plugin or Swift change is anticipated: the JSON structures stay compatible and
clients already consume the reported final values.

## Phase notebook-safety (bob-cli-k9)

Read `decisions:notebook-routes-write-notes` through `sase_memory_read`. In
`src/native/capture_language/notebook.rs`, `classify_item` returns early when
`item_has_explicit_operation` classifies a leading bare `-` as a session token. This
lets an unrouted bullet-led item escape a draft-wide notebook route and reach the task
writer. A notebook must never receive a checkbox.

For the ambiguous `@@dev_tools` followed by a blank line and `- a` / `- b`, refuse
atomically with actionable teaching to use an explicit `@dev_tools` list form. Do not
introduce a new implicit-list meaning for @@ or change its draft-wide scope. Place the
refusal at the shared classification/validation boundary so capture-parse, preview and
execution agree. Keep actual explicit session operations on their current parser path;
do not globally redefine the `-` token or change area-route semantics. Add a final
notebook-destination invariant guard if an alternate task path can bypass the
classification. Keep valid ordinary notes, explicit notebook lists and child notes
working, and verify no writes to any file on a refused mixed batch.

Add regression cases for the exact k9 draft, an explicit-list success control, ordinary
notebook prose with children, and a real session command under a global route. Test
dry-run/JSON and writing capture against a temporary vault; refusal must leave the
notebook byte-identical. Read docs.md before updating only the affected notebook/capture
contract and teaching examples; do not add a CLI flag. Run the focused notebook
capture/parser tests and `just check`, then close bob-cli-k9 with concrete evidence.
Report any unrelated issue on the phase as PROPOSED FOLLOW-UP.

## Phase ledger-identity (bob-cli-dq, bob-cli-ht)

Read `decisions:open-pomodoro-names-follow-task-links`. Keep its agreement rules: open
names follow their supported task links, closed sessions settle from worked types,
prose/unknown children protect, and MEET/container exceptions stay.

Inspect `src/native/capture/pomodoro_blocks.rs`, the status-changing capture writers,
`pomodoro_past.rs`, and the shared name-agreement applier. Preserve explicit
`PomodoroBlockRef` identity through the actual edits and chronological moves. Fix both
the rewritten-headline and vanished-entry paths at their writer/identity boundary. Never
suppress debug assertions, make every row Created, or fall back to null previews as the
repair. Task rows inside ## Pomodoros must either be excluded consistently from session
identity or have explicit edit identity; prove unrelated session blocks retain correct
previews and line positions.

After the final name agreement/settle, derive the past-record summary and JSON name from
the actual final row. Under the ht fixture with default_type CHORE and READ selected,
the preview, emitted name and written row must agree according to the accepted naming
decision; do not pin the name just to silence the discrepancy. Cover `x=read2`, a
segment-pinned form, a chained form, and a close-part form. For dq, cover the exact
cancel-label draft on a task in today's ## Pomodoros and an ordinary completion/status
toggle. Add realistic config fixtures, assert debug success, meaningful non-null
previews when a session changes, final row coordinates, matching human/JSON names, and
dry-run/write parity. Include unchanged/unrelated blocks and no partial writes on
refusal. Existing past-record, segment, name-agreement and tracker tests plus
`just check` must pass. Close both tasks with evidence.

## Phase midnight-dating (bob-cli-h1)

`compute_pomodoro_start_range` in `src/native/capture/pomodoro_start.rs` currently
formats negative `start_block_minute` modulo 1440. Reject a start before minute zero of
the selected current-day ledger before planning any edits. Return the normal usage
refusal, explain that the offset reaches the previous day, and give the maximum valid
offset derived from the same rounded start anchor. Do not silently write a previous
day's session into today, invent a prior-day write, or change statistics.

Keep valid late-night sessions that START today and END after midnight. The existing
`capture_pomodoro_start_midnight_wrap_and_bob_now` test covers that distinct behavior;
do not turn the end wrap into a blanket midnight ban. Inspect signed future-start
support in `meet_event_run.rs` and preserve its existing semantics. Exercise bare, named
and task-link starts, exact-midnight boundary, just-too-large past offset, zero offset,
and large/overflow input. The `=-3 x` draft at 00:05 must refuse atomically and leave
every vault file unchanged; a valid start-and-close chain continues to work. Keep
parser/preview/write parity, update the existing start contract if needed after docs.md,
run focused start/MEET/past tests and `just check`, then close bob-cli-h1.

## Phase clobber-evidence (bob-cli-l0)

`IdentityMatch::matches` in `src/native/vault_clobber.rs` accepts matching created
alone. A minute timestamp is not unique identity. Make timestamp-only matches
insufficient; require corroborating newly shared task block IDs or a matching ^prj
identity, while retaining the existing standalone strong ^prj and >=3 shared-new-ID
cases. A created match plus at least one newly shared task ID is corroborated; a
timestamp at finer precision still cannot identify a foreign note by itself. Keep
candidate displacement screening, quarantine safety, diagnostic evidence, scoped git
handling and real-clobber preservation intact. Avoid a broad clobber algorithm rewrite
or fuzzy title heuristic.

Use scratch git vaults and existing unit seams, never the real vault. Show two unrelated
notes sharing created to the minute do not quarantine a valid body edit; also test exact
timestamp matches with seconds, created+new-ID, ^prj, and three shared-new-ID true
positives. Verify a true clobber retains recovery bytes and an unrelated valid edit is
left untouched. Adapt old tests whose only evidence was a created stamp so they exercise
the stricter contract instead of deleting protection coverage. Run focused
vault_clobber/vault_sync tests and `just check`; if docs state created-alone is
sufficient, correct only that contract after reading docs.md and run `just docs-check`.
Close bob-cli-l0 with concrete test evidence.

## Verification and completion

Every phase runs focused tests before `just check`; never run `just check-full`. Use
`sase_monitor` for long gates and finish its staged handoff through `sase_final`. If a
gate failure is unrelated, establish clean-base evidence and record the exact existing
task or PROPOSED FOLLOW-UP, not a speculative waiver. Do not weaken assertions or
increase wall-clock budgets. No manual git commit; finish with the host finalizer. Phase
workers close only their assigned phase and resolved tasks, never this epic. The lander
verifies all five task outcomes and each phase note, integrates post-start drift,
retires epic-symbol exemptions, closes normally, runs symvision when available, and sets
this plan's status done. None of those closes should depend on the same turn's future
commit SHA, push or CI: cite existing source commits and test evidence, then let
sase_final commit the completed work.

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
