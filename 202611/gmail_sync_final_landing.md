---
tier: tale
title: Repair Gmail sync recovery and finish the nested landing
goal:
  Gmail sync satisfies its remaining pacing, recovery, reporting, and acceptance
  requirements, then bob-cli-in, bob-cli-ii.3.1, and bob-cli-ii.3 close normally with
  verified evidence.
size: medium
proposed_by: bbugyi200.athena.bob-cli-in.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-in
status: done
---

- **PARENT:**
  [202611/gmail_sync_ii3_completion.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_sync_ii3_completion.md)
- **BEAD:**
  [bob-cli-in](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-in/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-in.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-in.land.md)
- **COMMITS:**
  - [1cf8d96](https://github.com/bobs-org/bob-cli/commit/1cf8d96b37fa250521d77990e5da19f40b79f233)
    — fix(gmail): land remaining sync retry, recovery, and watch-membership defects

# Finish the remaining Gmail sync defects and complete its landing

## Scope and authority

Implement this bounded repair directly, then finish the landing in the same coding turn
(or its verification continuation). This tale has no land agent. Do not delegate another
completion plan, wait for a commit SHA, push, or CI, or leave closeout for the agent
that proposed it. `/sase_final` commits the verified changes after closeout.

The lander reviewed **bob-cli-in**, all four closed phases and every note, commits
`3ff3c4fe`, `e5a7b2a3`, `51d572d5`, and `59ba11a6`, and the actual implementation. Read
its `LANDING REVIEW` and `FOLLOW-UP TRIAGE COMPLETE` notes before editing. Read these
artifacts through `sase artifact read`:

- `plan:202611/gmail_sync_ii3_completion.md`: the requirements this tale completes.
- `plan:202611/gmail_sync_landing_repairs.md`: the earlier repair contract.
- `plan:202611/gmail_sync_filing.md`: original filing acceptance and nested landing.

Preserve the working coordinator/raw-auth latches, adoption-before-limit, host-local
recovery, path-scoped commit, save/capture staged writers, schema-1 fields and status
strings, and permanent hidden `pull` alias. Keep all Gmail operations read-only and all
tests offline, using fake-gog and temporary vaults. Never use Bryan's vault. Do not edit
release versions, CHANGELOG.md, or memory. The inherited decisions remain
`cron_enabled=no`, `gmail_watch_record=no`, `reply_hold_record=yes` owned by outer phase
`bob-cli-ii.4`, and `inbox_glossary=yes` already completed.

The audited HEAD was `59ba11a6`, equal to freshly fetched `origin/master`. Intervening
changes already integrate configured gog (`809c0fc7`) and IDs-only `threads.list`
(`063f9c23`) through the coordinator. Capture timing/grammar changes (`f9601b70`,
`b4163548`, `572ce974`) do not replace sync's staged task insertion; relation docs
(`5ed31c13`) do not conflict. Fetch and inspect new drift before editing and before
closing; preserve outer Gmail watch/health work if it has since landed. The old
`target/debug/bob` present during landing did not even expose Gmail: build current
sources before using a binary as verification evidence.

## 1. Pace every request attempt and correct cap diagnostics

In `src/native/gmail/sync/retry.rs`, `CoordinatedRead::run` calls `pace_before_request`
once, outside `execute_with_retry`. Retries do not update `last_request`. A successful
retry can therefore be followed immediately by the next stage's request; a final
budget-limited backoff shorter than 200 ms can also issue a retry too early.

Move attempt pacing/timestamp accounting into the operation executed for each actual
read attempt. Preserve the shared wait-only budget: backoff, guard contention, and
vault-lock waits consume it; ordinary pacing never does. Keep one initial attempt at
`-r 0`, sticky auth/give-up behavior, and no double sleeping for a gap already covered
by backoff. Use the injectable clock/sleeper to assert actual call times, including a
retry followed by a different operation and a remaining backoff budget below 200 ms.

In `sync/discover.rs`, `cap_has_more = thread_index + 1 < threads.len()` still counts
trailing duplicates and invalid IDs as outstanding results. Validate and deduplicate the
remaining page entries when deciding whether to warn; preserve next-page-token handling
and the 5,000-ID bound. Test an exact cap with only duplicate/invalid trailing entries,
a genuinely new trailing ID, and a next page. Keep `threads.list` transport
normalization and its read allowlist intact.

## 2. Make recovery failures observable and journal replay idempotent

Work in `sync/file.rs` and its call site in `sync/command.rs`; reuse the shared
journal/save helpers without changing unrelated save/link/compose behavior.

- `has_pending_recovery` currently returns false on any `fs::read` failure. An idle
  `sync -l 0` with unreadable recovery evidence therefore skips `apply` and reports
  success. Make dispatch distinguish absence from an unreadable/unusable path, then
  surface the authoritative load failure as a body-free warning and exit 1 under the
  normal live guard/vault-lock boundary. Dry-run reports the error without writing. Test
  through command orchestration with no candidates/adoptions, using a directory at the
  computed evidence-file path or another portable read-error seam.
- The failed-commit branch discards `write_recovery` errors with `let _ = ...` and
  reports only a retry hint. Include both failures in the body-free diagnostic so the
  user is not told that replay evidence was retained when persistence failed. Cover this
  with a failing temporary Git pre-commit hook plus an injected evidence write failure.
- Origin events are appended as one batch, with one checkpoint only after the batch.
  Checkpoint each successful origin separately, as the approved contract requires.
  Retain failures and never enroll a thread before its save/origin stages finish.
- `persist_recovery` records an error but processing continues. If an append succeeded
  and its checkpoint failed, the old file can replay that save/origin again. Make
  checkpoint failures control the replay flow and retain enough stable, body-free
  per-event evidence to recognize an append already present in the journal on retry. Do
  not globally deduplicate legitimate independent save/import events. Keep this logic
  scoped to sync recovery and keep the public JSON contract unchanged. A bounded
  injectable I/O seam is appropriate for deterministic failure ordering.

Regression coverage must exercise two origins with a failure/interruption after the
first append, a successful save/origin append followed by failed evidence persistence
and replay, and failed final evidence deletion. Assert exact event multiplicity, pending
stages, body-free diagnostics, blocked enrollment, eventual cleanup, and no second
filing. Preserve existing hash ownership checks, custom/unrelated dirt exclusion,
auxiliary-file gating, and recovery with no search results and `-l 0`.

## 3. Report actual watch membership after recovery

`sync/report.rs::watching_count` adds every `filed`/`would_file`/`adopted` result to the
count. `file.rs` emits recovered rows even if the task was closed before replay.
Consequently the closed-task recovery test can pass duplicate-prevention assertions
while the report still claims one watcher. Save/origin failure also prevents enrollment
without placing the result in `outcome.deferred`, so it can count as a newly enrolled
watch despite the pending journal stages.

Carry explicit membership/outcome information or revalidate the needed local membership
so that recovered closed/deleted tasks do not count, and pending enrollment does not
count as completed enrollment. Preserve projected dry-run membership, newly filed
route-problem rows, the explicit apply-vs-initial snapshot choice, and existing
result/status semantics. Do not infer watch membership solely from status strings. Keep
awaiting counts consistent with the same eligible watch set.

Extend the recovery fixtures to assert the public report for closed/deleted tasks,
save/origin/enrollment failures, successful replay, and dry-run projection. Verify there
is still exactly one task and the stored enrollment event is retained.

## 4. Finish the unresolved acceptance evidence

The guard note on bob-cli-in and bob-cli-ii.3.1 is still open work. The acceptance
commit only removed allowances from `sync/guard.rs`; its note reports the same
`real_lock_contention_and_release` failure followed by passes. Diagnose the release
failure under concurrent library tests and make the coverage deterministic. The test
already uses thread-local `TestEnvGuard` and a unique temporary state directory, so do
not assume process-global environment interference. Verify the first acquisition
actually acquired a guard. Investigate descriptor/lock lifetime across concurrent
child-process creation if indicated, but establish the cause before changing release
semantics. Do not mask the failure with retries, arbitrary sleeps, serialization of the
whole suite, or ignored assertions. Preserve real contention and prompt release
semantics; use explicit fast budgets in CLI contention tests.

Complete the missing orchestration vectors from the approved coordination phase, reusing
lower-level tests where they genuinely prove the contract:

- Auth abort during metadata, hydration, and policy label lookup leaves no vault or
  journal writes, even when an earlier candidate was prepared. Hydration already has a
  CLI case; add the missing boundaries rather than duplicating it mechanically.
- Actual hydration/policy give-up defers every unprepared key, keeps earlier fully
  prepared successes, returns `ok=false`/exit 1, and retains failure diagnostics if the
  vault lock is busy. Current command tests construct outcomes directly.
- First/later permanent and malformed later search failures preserve usable earlier
  work, local adoption, and pending recovery while returning failure. Discovery unit
  outcomes alone do not verify those command effects.
- Shared nonzero wait accounting across guard/retry/vault-lock stages, a healthy
  two-thread `-r 0` through preparation, an actual planning failure's diagnostic, and
  closure/deletion between the initial scan and locked apply have concrete outcome/count
  assertions. Introduce a narrow injectable orchestration helper if needed so these run
  against fake time instead of slow real sleeps.

Keep the existing separate fixed-time sync/pull alias fixtures and all save/link/
attach/compose tests. No new CLI, configuration, memory, or documentation redesign is
needed. Read docs/CLI memory first if an actual user-visible edit becomes needed.

## 5. Verify and finish all delegated closeout

The lander completed step-3 follow-up triage before proposing this tale: every note on
bob-cli-in.1-.4 was read; there were **zero PROPOSED FOLLOW-UP entries**, no established
unrelated defect, no task creation/corroboration needed, and no declined proposal. Guard
corroboration already belongs to this epic's acceptance. These outcomes are recorded on
bob-cli-in. New unrelated findings require `/sase_new_task` and a recorded outcome;
epic-caused failures remain this work.

Run focused tests for the changed boundaries, formatting, **`just check`**, and
**`just docs-check`**. Use `/sase_monitor` for long verification, and make its
continuation own every remaining closeout action below. Never run `just check-full`. A
check-full-only failure is infrastructure, not remaining epic work. Reproduce any
purported unrelated baseline failure on the clean base before classifying it; never
claim a passing gate from stale binaries or earlier agents' notes.

After verification, the final step of this tale is the following concrete closeout:

1. Reread bob-cli-in, every descendant and note, its linked plan, and post-review drift.
   Verify all requirements above and the completed phases. Resolve this tale's own bead
   normally first if the host created one that would block parent closure. Mark its own
   linked plan done through the normal plan workflow.
2. Run `sase bead epic-symbols bob-cli-in`. Resolve every entry by wiring it up,
   privatizing it, using an appropriate non-test pragma, or deleting it; re-key only to
   a specific still-open later bead that genuinely requires the exemption. Then run
   `sase bead close bob-cli-in --note "<source and commit review, integrated drift, repairs, follow-up outcomes, and passing check/docs evidence>"`.
   If symbols remain, finish cleanup and retry. Never force a successful nested landing.
   If phases are incomplete, finish/reopen them;
   `--force --reason ... --resolution canceled|superseded` is only for deliberately
   canceled/superseded work, never to make closure succeed.
3. Run `just symvision` if available; it was absent at review, so record that fact if
   still absent. Open the plans sidecar using `/sase_repo`, use the PLAN path from
   `sase bead read bob-cli-in`, and set `status: done` in the frontmatter of
   **`plan:202611/gmail_sync_ii3_completion.md`**. Read artifacts only through
   `sase artifact read`; never hand-edit bead-store state.
4. Re-read `sase bead read bob-cli-in -r "Need the parent link"`. It had no
   `parent_bead` at review. Independently, its approved completion scope explicitly
   requires finishing **bob-cli-ii.3.1** and phase **bob-cli-ii.3**. Review
   bob-cli-ii.3.1's prior landing notes, all descendants and their notes, linked plan
   **`plan:202611/gmail_sync_filing.md`**, and post-child drift. Rerun descendant and
   linked-plan readiness checks. Once complete, resolve/re-key every
   `sase bead epic-symbols bob-cli-ii.3.1` entry, close normally with
   `sase bead close bob-cli-ii.3.1 --note "<rechecked original sync acceptance, repairs, drift, follow-up outcomes, and passing verification>"`,
   run `just symvision` if available, and set that plan's frontmatter `status: done` in
   the opened sidecar. The earlier `gmail_sync_landing_repairs.md` plan was already
   done.
5. Verify **bob-cli-ii.3**'s delegated discovery, enrollment, watch scan, adoption,
   routing, staged filing, reports, and hidden-alias scope against the code and tests.
   Confirm its child bob-cli-ii.3.1 is done. Resolve/re-key
   `sase bead epic-symbols bob-cli-ii.3` entries and close only that phase normally:
   `sase bead close bob-cli-ii.3 --note "<verified delegated sync contract and checks>"`.
   Leave containing epic **bob-cli-ii** and **`plan:202611/gmail_sync_reply_watch.md`**
   to its waiting land agent. If any parent/readiness check is incomplete or ambiguous,
   stop there, note the precise blocker on that bead, and report it; never force it
   closed. If actual parent links differ, follow the user's plan-ancestor readiness rule
   and stop at the first incomplete parent or after closing a verified parent phase.
6. Finish through `/sase_final`, declaring the code and every changed sidecar. Report
   only beads actually closed and verification actually completed. No step depends on
   this work's commit SHA, push, or CI result.
