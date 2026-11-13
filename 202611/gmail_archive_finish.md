---
tier: tale
size: medium
title: Finish Gmail archive safety checks and nested landing
goal: Repair the remaining Gmail archive defects, verify the contract, and close the
  repair epic and eligible parents normally.
proposed_by: bbugyi200.athena.bob-cli-kx.2.1.8.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-kx.2.1.8
status: done
---

- **PARENT:**
  [202611/gmail_archive_landing_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_archive_landing_repairs.md)
- **BEAD:**
  [bob-cli-kx.2.1.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kx/bob-cli-kx.2.1.8.md)

# Finish Gmail archive safety checks and nested landing

Complete the remaining work in epic `bob-cli-kx.2.1.8` and finish its landing in this
tale. There is no later land agent for this tale. This is bounded work in the existing
Gmail archive modules and fake transport, followed by verification and the concrete
nested closeout below. Do not recreate the four completed phases.

## Context and binding scope

Read `plan:202611/gmail_archive_landing_repairs.md` and
`plan:202611/gmail_archive_core.md` through `sase artifact read`, and read
`bob-cli-kx.2.1.8`, its four phase beads, and parent `bob-cli-kx.2.1` through
`sase bead read`. The repair commits are `dcd908ab`, `642e1580`, `6b9261a7`, and
`a8dd8204`. The landing audit was on `f508aa46`.

The accepted decisions on `bob-cli-kx.2` remain binding: Cancelled `[-]` archives like
Done; the receipt defaults to `<sync.label>/closed`; `archive_max` defaults to 10;
`bob-cli-i4` is superseded; no memory changes are authorized. Preserve frozen saved IDs,
one-shot ownership, consent, independent let-go, exact per-thread batch-modify fencing,
schema version 1, additive JSON, and exit codes 0/1/2. Use only fake gog and temporary
synthetic vaults. Do not touch the real vault/account, deploy, change package versions,
or edit CHANGELOG.md. Gmail contract changes need a `fix(gmail)!:` or `feat(gmail)!:`
commit declaration.

The four phase notes were reviewed. Most repairs are present, but the source checks and
property-test run below show that the epic is not ready to close. Non-epic commits after
`dcd908ab` are `78c0e085` (capture record-field completion) and `f508aa46` (album
verification evidence). Neither consumes archive machinery. The shared
`docs/reference/json-outputs.md` retains capture grammar 27 alongside the archive
additions, so no integration edit was needed. Recheck later drift.

## 1. Make the final vault check fail closed, and test the actual check

In `src/native/gmail/sync/archive_apply.rs`, the per-thread `fresh_evidence` closure
calls `scan_archive_evidence`, then extracts `fresh_scan.evidence()` and discards
scan-wide completeness. `ArchiveScan::scan_complete()` expressly says an unreadable note
can hide a linker for any thread. A newly unreadable or invalid-UTF-8 note between
preview and mutation can therefore be ignored while the remaining per-thread evidence
still matches the close.

Keep the fresh scan's completeness and safety checks at the mutation boundary. An
incomplete fresh scan must hold that thread, emit a bounded body-free warning, mark the
run unsuccessful, and issue no message-label mutation or terminal event for that thread.
Keep lock acquisition bounded and retain the lock for only one thread's write. Do not
broaden frozen IDs or weaken the current fingerprint, task-locator/status, identity, or
planned-decision comparisons.

Replace the production `BOB_GMAIL_ARCHIVE_TEST_REOPEN_THREAD` shortcut and its test in
`tests/gmail_archive_apply.rs`: it currently returns the expected hold before reading
anything, so the test passes even if real revalidation is broken. Use an isolated
fake-gog hook or an injected test collaborator that actually changes a synthetic vault
after planning and before the fresh scan. Cover both an actual task reopen and a new
unreadable/invalid-UTF-8 note that could hide a linker, asserting zero message writes,
no terminal event, failure, and retained filing rows. Keep fault controls per test/child
process rather than mutating the test runner's global environment. Likewise replace
`BOB_GMAIL_ARCHIVE_TEST_FORCE_ARCHIVABLE` with a real metadata transition in the fake
transport to prove a planned Kept row cannot upgrade into a new archive. Retain
reply-after-final-read coverage and exact frozen message argv assertions.

## 2. Preserve partial archive results and make failure accounting truthful

`apply_live` accumulates rows in `LiveArchiveOutcome` but returns only `Err(GmailError)`
on an auth abort or journal error. `command.rs` then sets `live_archive` to `None`.
Successful archive/let-go/recovery rows earlier in the same run disappear from JSON and
human reports even when their journal entries and Gmail mutations are durable. The
parent contract requires partial progress to remain clearly reported. Preserve the
accumulated outcome with the fatal error (for example, a typed outcome plus optional
error), attach both to the sync report, and keep existing filing/watch rows, hints,
warnings, and counts. Abort every later Gmail mutation on `invalid_grant`; never
manufacture a success row for an unjournaled write.

The current `boundary_invalid_grant_on_second_thread_aborts_later_writes` test archives
its first thread in a previous run, fails during the next run's preview, and checks only
on-disk filing/journal existence. It does not prove same-run partial progress. Add a
deterministic fake fault that allows planning to finish, successfully archives thread A,
then fails thread B at its mutation-time read or write. Assert A remains in `archive[]`
and `summary.archived`, filing/watch rows remain, B has no false terminal record, and no
later archive, let-go, or label creation occurs. Cover a journal-append failure after an
earlier successful row as well, and retain crash reconciliation on the next run.

Fix these related accounting gaps within the archive path:

- `archive_preview.rs` handles `RemoteVerdict::Held` with `failed = true` only for
  `HOLD_FETCH_FAILED`, not `HOLD_UNRESOLVABLE`. For a new close not yet persisted, a
  wrong thread/message identity is omitted from the plan and never reaches the live
  loop's failure accounting. Make unresolvable identity fail both preview and live runs,
  with zero mutations for the bad row. Test a fresh close as well as an
  already-journaled close.
- The repair plan requires exit 2 for archive auth errors, but the shared
  `google/gog.rs` maps `invalid_grant` to runtime exit 1, the archive report forwards
  it, and the integration test explicitly expects 1. Normalize archive authentication
  errors to the promised setup/auth exit 2 without casually changing other Google
  commands' contracts. Preserve typed kinds, sanitized warnings, and the login hint.
  Keep runtime failures, holds, and confirmation mismatch at exit 1. Cover human and
  JSON output with retained earlier rows.

## 3. Repair the property-test generator, preserving production validation

The new property test in `archive_transport.rs` draws `label_suffix` from `1u16..10000`
and constructs `Label_{label_suffix}` against enrollment ID `Label_1`. At
`label_suffix = 1`, `ReceiptLabel::new(...).unwrap()` panics. The landing reproduced
this with:

```sh
PROPTEST_CASES=65536 PROPTEST_DISABLE_FAILURE_PERSISTENCE=1 cargo test --lib label_diff_properties_hold_for_independent_message_labels -- --nocapture
```

The test failed after 1,793 successful cases; its minimal input was one message
`("0", {})`, `label_suffix = 1`, `name_suffix = ","`. This is epic-caused test work, not
an unrelated CI follow-up. The complete landing log is
`file:explicit:1e7c765d5bd3a77bc9aa19a1`; read it with `sase artifact read`. Construct
valid non-aliasing receipt IDs by design. Keep separate explicit rejection coverage for
aliases; do not weaken `ReceiptLabel::new` or swallow invalid construction.

Make the independently generated per-message label sets include arbitrary INBOX and
receipt membership as well as unrelated literal labels. The current generator only emits
lowercase characters/punctuation, so its generic let-go check never sees an INBOX or
receipt label. Retain arbitrary valid unique message IDs, commas/backslashes in literal
label names, idempotence, archive/restore reversal under its stated precondition, and
proof that let-go never changes INBOX and no operation changes unrelated labels. Run the
repaired property test with enough cases to exercise the previously failing boundary,
plus deterministic boundary coverage so success does not depend on random sampling.

## 4. Verify, then finish this epic and the eligible parent chain

Run formatting and relevant unit/integration tests, including all Gmail suites required
by the repair plan and the new real fault cases. Run `just check`; never run
`just check-full`. Use `/sase_monitor` when the canonical check needs a handoff,
preserving this tale's remaining closeout obligations. Do not order closeout after this
work's own commit SHA, push, or CI: closing with the final code in the same turn is
normal. Update the C1-C20/fault coverage table to point to tests that actually exercise
each behavior. If changing CLI/help/docs, first read `cli_rules.md` and `docs.md`;
regenerate and check documentation as required.

Known unrelated gates are recorded on existing beads: `bob-cli-l2` (GCal alias test),
`bob-cli-kl` (subprocess pipe-drain flake), and `bob-cli-l7` (docs lychee path). Do not
treat a new Gmail failure as one of these. The parent landing note also records older
catalog/glossary issues owned by `bob-cli-l5`; recheck whether they remain before
reporting them. Triage genuinely new unrelated failures with `/sase_new_task`; keep
epic-caused defects in this work.

Follow-up triage is already complete for this repair epic: the only proposal is
`bob-cli-kx.2.1.8.4` note #1, duplicated by `bob-cli-l7`. The lander searched all
statuses, swept recent CI tasks, reviewed active epic scopes, confirmed the bad relative
path at `docs/gcal-notes.md:22`, and added a +1 to `l7`. No proposal was declined or new
task created. Repeat this outcome in the close note. Parent `bob-cli-kx.2.1` note #1
already dispositions all seven original phase proposals; read it and recheck stale
outcomes rather than duplicating them.

The final step of this tale is the following concrete closeout, not a handback:

1. Re-read `bob-cli-kx.2.1.8` and all four children and notes. Verify every repair above
   and every original acceptance item against source, tests, and post-audit commits.
   Record any remaining blocker and leave the epic open rather than claiming completion.
   Run `sase bead epic-symbols bob-cli-kx.2.1.8`; resolve every listed symbol or re-key
   its Justfile exemption only to a still-open later bead that actually needs it. The
   landing audit found no entries.
2. Close normally with
   `sase bead close bob-cli-kx.2.1.8 --note "<verified source, decisions, tests, integration, and follow-up outcomes>"`.
   If symbols reject close, clean them and retry. If phases are incomplete,
   finish/reopen them; never force a successful nested landing. Run `just symvision` if
   available (this checkout currently has no recipe), then set `status: done` in
   `plan:202611/gmail_archive_landing_repairs.md`. Use `sase repo open plans` before
   editing the sidecar and the PLAN path from a fresh bead read; do not hard-code a
   workspace path. Read artifacts only with `sase artifact read`.
3. Re-read the parent link. Its current parent is plan bead `bob-cli-kx.2.1`. Review its
   previous landing notes #1/#2, every descendant and note, and
   `plan:202611/gmail_archive_core.md`; recheck descendant and linked-plan readiness
   after the child close, and all drift since the parent audit. The original phases `.1`
   through `.7` are closed; this child is its remaining repair epic. If still complete,
   resolve `sase bead epic-symbols bob-cli-kx.2.1`, close it normally with
   `sase bead close bob-cli-kx.2.1 --note "<rechecked parent contract, all prior defects, descendants/plans, tests and drift>"`,
   run `just symvision` if available, and set `status: done` in its PLAN file
   `plan:202611/gmail_archive_core.md`. If incomplete or ambiguous, record a blocker on
   this parent and stop ascending; do not force it closed.
4. The next parent is phase `bob-cli-kx.2`, within outer epic `bob-cli-kx`. Re-read it
   and `plan:202611/gmail_archive_on_task_close.md` to verify the core phase
   requirements and final DECISIONS. Its note #1 calls out the approved
   `bead_i4 = close` decision, which remains unfinished: `bob-cli-i4` was READY at
   audit. Read `i4` again, and fulfill that decision by closing it normally with
   resolution `superseded`, a reason citing this accepted design, and a note explaining
   that removing enrollment would erase archive consent and a nested-label phone queue
   remains an alternative. Do not implement unlabeling. After verifying the full phase
   scope, resolve any phase symbols and close only `bob-cli-kx.2` normally with a
   verification note. Leave containing epic `bob-cli-kx` to its waiting land agent; its
   receipts/polish/enable work is outside this tale. If this phase is incomplete, note
   the blocker and stop.
5. Finish through `/sase_final`, declaring all changed repositories including
   plan-sidecar status changes. Do not manually commit or wait for this work's commit
   before closing. Never use `--force` to advance successful completion; a genuine
   cancellation/supersession of unfinished descendants needs a deliberate reason and the
   appropriate non-done resolution.
