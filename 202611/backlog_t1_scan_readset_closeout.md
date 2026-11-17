---
tier: tale
size: medium
title: Finish T1 scan read-set coherence and land the backlog epic
goal:
  A full scan never accepts post-read evidence for stale TextBundle or asset plans,
  retries retain one invocation date, bob-cli-mr and bob-cli-mo close after
  verification, and the prepared five-task T2 backlog epic is proposed.
proposed_by: bbugyi200.athena.bob-cli-mr.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-mr
create_time: 2026-11-17 17:36:04
status: wip
---

- **PARENT:**
  [202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md)
- **BEAD:**
  [bob-cli-mr](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mr/README.md)

# Finish T1 scan read-set coherence and landing

## Assignment and current state

This is the remaining work from the bob-cli-mr landing audit, not a new feature or a
child epic. You are responsible for the entire closeout in the final step below; there
is no separate tale lander and no dormant T1 lander will resume after you. The approved
decision is `scan_lock = narrow`; do not re-ask it.

Read `sase bead read bob-cli-mr -r "Need scope, landing evidence and follow-up triage"`
and the latest notes before editing. The audit reviewed all direct phases mr.1/.2/.3,
the nested mr.3.1 epic and its three phases, all their notes and both plans. All are
closed; mr is in_progress and the scan task mo is still ready. The root epic has no
parent_bead (also `parent_id: null` in the audited JSON). Its plan is
`plan:202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md`, resolved by
`sase bead read` to the plans sidecar. The nested scan plan is already done.

Current audited bob-cli commit is 0e83df4c (scan landing repairs), with the previous
scan work 1e8f77af, 3236ceef and 6c409ab2 and capture repair b0bddbb9. Plugin repair
b0a9c89 is in current bob-plugins a1550c8; all 179 focused regression tests pass. The
original md/me/m8/jh task beads are closed. Mac receipt docs are corrected by bfbedda9.
The T1 lander built the current debug binary and reproduced the new defect; it did not
run a full just check because a repair remains. Do not run just check-full.

Later unrelated Rust commits 99005975/48a3116d/cfc3177c add records show/update/tidy;
nightly's records step uses its existing maintenance-lock ownership. Later plugin
commits add card/schema/bank/emoji behavior, with no scan/promotion integration needed.
At start, fetch and inspect later origin commits, fast-forward a clean tree as
appropriate, and integrate only an actual new conflict with these repairs.

Read the governing decisions through `sase_memory_read`:
`decisions:open-pdf-writes-are-coordinated` and
`decisions:references-are-ingested-at-birth`. Preserve unlocked full-library planning,
no-work avoiding both locks and receipts, bob_sync.lock then scan.lock order, protected
intake-through-filing, hook-once behavior, bounded retries, shared human/JSON
presentation, collision/error shapes, and coordinated per-PDF execution. No new flags,
public schema, persistent cache, memory edits, real-vault mutations or deployment.

## 1. Repair coherent input capture

Reproduction script: file:explicit:bd831c44f897f151df7fe929. Read it with
`sase artifact read`; use a freshly built binary from cargo metadata's actual
target_directory, not a stale target/debug/bob. It uses only temporary vaults and the
existing bounded before_evidence handshake. Matching output:
file:explicit:1b54ca1c1f7d772c943b7307. Validated T2 draft:
file:explicit:e68e1976353c86235b1d0e23. Read these through sase artifact read; use sase
artifact path when a consuming tool needs the file path.

The script runs an already-converged one-PDF scan, pauses after planning but before
capture_scan_evidence, then edits only `Comment: Old content.` to
`Comment: New content.` in the selected TextBundle sidecar (same length). With text.md,
current code logs two unlocked plans and reports updated=1 with New content imported.
With text.markdown, it logs one plan, reports success with unchanged=1, and leaves the
old annotation; the next stable scan reports updated=1. This disproves coherent
planning. Do not count a harmless image-alt edit as the regression: alt text alone need
not change rendering.

Root: `src/native/highlights_ref/scan_plan.rs::prepare_from_inputs` snapshots only vault
.md, config and PDF hashes before planning. `capture_scan_evidence` tracks
text.markdown, nested image assets and other non-.md inputs only afterward; subsequent
validation compares those newer bytes to themselves. Membership and archive inputs also
need the same coherent treatment, not a post-hoc hash attached to an old plan. The
existing same_length_edits_before_evidence_replan test changes sidecar, ref, config and
parent simultaneously and adds a PDF; any one detected mutation masks missing coverage
of every other input.

Use one explicit read-set snapshot before the planning reads, including both supported
TextBundle text filenames, present/absent sidecars, relevant nested source assets,
selected destination assets, audio companions and the archive metadata consulted by
holds. Reuse/refactor the current helpers; do not hash unrelated media or parse the
whole library under locks. Ensure inputs discovered through a plan are either captured
at the read that informed that plan, or included conservatively in the initial set.
Compare existence, membership and bytes before/after planning, preserving the actual
planner PDF hashes and per-write guards. If safe coherence cannot be established, return
the existing bounded scan_failed refusal before mutation.

Close the adjacent discovery gap: prepare_full_scan discovers `pdfs`/`intake` before
prepare_from_inputs takes `pre_lib`/`pre_xlib`. If a new PDF arrives in that interval,
both pre/post snapshots include it but the planned PDF list can omit it. Make discovery
part of the same snapshot or explicitly validate that the prepared list matches the
snapshot's relevant PDF membership. Keep audio members distinct from the PDF list. Cover
creation/deletion/rename and configured library locations, including an absolute lib
path outside bob_dir so vault-only Markdown enumeration cannot provide false coverage.
Fail closed on enumeration errors rather than treating an unreadable tree as unchanged.
Avoid a general unrelated scan redesign.

## 2. Pin invocation date and strengthen independent regressions

`prepare_full_scan_with_retry_date` retries a during-planning error with the original
Option<String>. For the initial None call this invokes current_local_date a second time.
The comment claiming day granularity preserves the date is false across midnight.
Resolve the date once before the first preparation, then pass that exact value through
every retry/rebuild/recovery path. Keep the finite retry budget explicit; no retry may
happen after an owned intake move. Use a deterministic unit seam or injected date
source; never wait for midnight.

Add focused independent tests, each changing ONE meaningful input at the handshake:
text.markdown annotation, text.md control, source image bytes, destination asset
existence/content, and membership at the discovery/snapshot boundary. Add another needed
case only if the repaired read-set design has a distinct boundary. Assert an observable
current result (new annotation/asset or fresh complete PDF set) or the specific safe
scan_failed refusal with no mutation; generic success and nonzero
created+updated+failures are insufficient. Include a no-work candidate so stale
classification cannot skip the executor's safety checks. Keep handshakes bounded and all
children reaped. Demonstrate that regressions fail on 0e83df4c and pass after the
repair; do not leave tests for unsupported input shapes or weaken existing tests.

Run scan_plan unit tests, scan_interleave, capture prerequisite regressions and relevant
scan/marker/overwrite/ingest suites. Preserve all existing locked-intake and recovery
receipt tests. Then run `just check`; do not substitute check-full. Use sase_monitor
when the gate is long, and place the complete remaining closeout in its --next so
verification resumes this same assignment. Do not lose the closeout by assuming a tale
has a separate lander. A known unrelated failure needs exact clean-base attribution, not
a speculative waiver. If you change docs, read docs.md and run just docs-check.

## 3. Complete the T1 epic and continue the already-prepared chain

Do this after the final code and verification in this assignment, without depending on
this turn's future commit SHA, push, or CI. The host commits through sase_final after
the turn; same-turn epic closure with final code is intentional.

1. Re-read mr and its descendants and notes for late issues and confirm all descendants
   remain closed; verify the nested scan plan remains done. Check post-audit drift and
   confirm the four earlier fixed tasks md/me/m8/jh remain complete. Re-run only checks
   justified by intervening changes. Record final test evidence and the repair outcome.
2. The lander already used sase_new_task and finished all proposed-follow-up triage.
   Carry these dispositions verbatim in substance into the final close note: mr.3.1.1 #1
   and mr.3.1.2 #1 generated decisions drift declined as fixed by 48a3116d, with
   generate_reference(check=True) returning []; mr.3.1.3 #1 JSON reference budget
   attached to causal active epic bob-cli-ms, not a new task (6112/6000 at 0e83df4c,
   versus 6089 before records tidy; ms notes #4/#6 assign the pending 5233-word repair
   to eo.f0--code); mr.3.1.3 #2 Mac receipt wording completed in bfbedda9; mr.1's old
   ju/ki load failures remain tracked there, with no fresh failing full-suite claim.
   Recheck whether the pending docs commit landed before repeating the budget claim. Any
   new unrelated follow-up you discover must go through sase_new_task; defects caused by
   this epic must be completed here.
3. Resolve bob-cli-mo normally with
   `sase bead close bob-cli-mo --note "<verified narrow lock lifecycle, coherent reads, protected intake, tests and existing commit refs>"`.
   Existing refs include 3236ceef, 6c409ab2 and 0e83df4c; do not wait for this repair's
   own finalizer commit to obtain a SHA.
4. Run `sase bead epic-symbols bob-cli-mr`. Resolve every entry (wire up, privatize, add
   a justified non-test pragma or delete per policy), or re-key only to a still-open
   later bead that actually needs it. The audit found none; recheck. Then close
   normally:
   `sase bead close bob-cli-mr --note "<all phase/decision/source verification, intervening-commit integration, final tests, and every follow-up disposition>"`.
   If rejected for epic-symbol entries, finish them and retry. If a named phase is
   incomplete, finish or reopen it; never force to advance a successful nested landing.
   Force is only for a deliberate canceled/superseded outcome with an explicit reason,
   never merely to make a command succeed.
5. After closing run `just symvision` if the recipe exists (absent at audit). Open the
   plans repo with sase_repo, use the PLAN path from mr's audited read, and set only the
   frontmatter status of
   `202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md` to done. Read the
   artifact through sase artifact read before changing it. Submit the changed plans repo
   in the finalizer declaration. Re-read mr for its parent link; it was null. If
   unexpectedly linked to a parent phase, verify this work completes that phase and
   close only that phase normally; leave its containing epic to its waiting lander. If a
   new parent plan exists, review its prior landing note, all descendants/notes, linked
   plan and post-child drift, rerun readiness checks, retire its symbols, then close
   normally, run symvision and mark its plan done only if fully complete; repeat through
   direct plan parents. Stop and note the first incomplete/ambiguous parent.
6. Propose the prepared T2 draft file:explicit:e68e1976353c86235b1d0e23. Read its
   artifact into a fresh sase_plan_backlog_triage_t2.md (preserve YAML at byte zero),
   recheck the five task statuses and intervening triage notes, and preserve the
   original Landing section unchanged. The draft is a separate top-level epic: ensure it
   is not parented as unfinished work of now-complete mr. Its five existing tasks are
   k9, dq, ht, h1, l0, in four sequential medium phases. Four were freshly reproduced in
   isolated dry runs and l0's timestamp-only identity is source-proven. The earlier
   sweep reviewed 235 active task rows/triage notes and 13 active plan scopes, excluding
   snoozed/owned work; no additional obsolete task was proven. Revalidate with
   --explain, make any necessary drift edit, revalidate without --explain, and run sase
   plan propose. Wait for that command itself to exit. On Staged, finish through
   sase_final, declaring commits for every changed repository. Do not wait for T2
   approval or its successor. Nothing about T1's closeout waits on T2.
