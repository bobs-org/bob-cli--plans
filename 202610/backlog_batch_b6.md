---
tier: epic
title: 'Backlog batch B6: RecoveryAssessment lint, lock-wait flake, macOS theme CI,
  capture_related perf flake, and cancel-reconcile docs'
goal: Resolve bob-cli-dt, bob-cli-dv, bob-cli-dr, bob-cli-ca, and bob-cli-dj with
  reproducible verification, then continue the bounded backlog chain.
phases:
- id: recovery-assessment-public
  title: Expose RecoveryAssessment for failed-close snapshot imports
  size: small
  depends_on: []
  description: 'recovery-assessment-public: resolve bob-cli-dt in sase by exposing
    the shared failed-close assessment type through a public name and updating _failed_close_recovery_snapshots.py
    (and any other consumers) to import that public name. Do not suppress, whitelist,
    or pragma the private _RecoveryAssessment misuse. Read SASE symvision policy first.
    Verify with focused failed-close snapshot/recovery tests plus sase tool run check
    in the opened sase checkout. Independent clean witness 13ea2a464abef73b3eab4bfee9e624d7
    already makes this KNOWN; later checks including 5d1634e22bc172fe358db97b9279961f
    still report it.'
- id: lsp-store-lock-flake
  title: Make the LSP store identity-lock wait test contend deterministically
  size: small
  depends_on: []
  description: 'lsp-store-lock-flake: resolve bob-cli-dv in sase tests/test_sase_core_wheel_cache_tool.py::test_lsp_store_rechecks_after_waiting_for_identity_lock.
    The node failed under parallel scoped load of SASE check 5d1634e22bc172fe358db97b9279961f
    (AssertionError: Waiting for the shared build lock in empty stderr) and passed
    3/3 isolated reruns on the same tree (d79366e7627ad5290e34d797cb5c6a7a, f02021819c78b654a26d3739e330ddeb,
    6d94f3e4d28d96bbaf830bde9189b048). Root cause: FAKE_CARGO_DELAY=0.3 plus a 5s
    marker poll lets the first fake cargo finish before the second store contends.
    Hold the first process in cargo until the second is observed, or otherwise make
    the lock wait deterministic. Do not suppress the node. Verify with focused serial
    and parallel reruns plus sase tool run check.'
- id: mac-unconfigured-theme-ci
  title: Restore the unconfigured-theme caret-underline XCTests on macOS CI
  size: large
  depends_on: []
  description: 'mac-unconfigured-theme-ci: resolve bob-cli-dr in bob-mac-capture.
    Master macOS 26 SwiftPM Test has failed on every push since 8b57693 with the same
    four assertions in CapturePanelModelTests.testCaretLeavingUnconfiguredThemeUnderlinesAfterAnalysis
    and CaptureUnconfiguredThemeHostedTests.testTypingUnconfiguredThemeThenMovingCaretUnderlinesWithoutFreezing
    (timeout then XCTAssertTrue at the caret-underline lines). Runs 40724077224 (8b57693),
    40728493998 (ebb798a attempted fix), and 40729693701 (e5f6604) all report four
    failures; Bundle/launch/install are skipped, so master has no green macOS signal.
    Open bob-mac-capture through sase repo open. The failures predate bob-cli-dl.7
    and touch only the unconfigured-theme dashed underline from 8b57693. Verify on
    the Mac CI path named by the task; do not invent a local green as a substitute
    for that CI signal. Never run just check-full.'
- id: capture-related-perf-flake
  title: Retune the capture_related paired lib perf gates after post-fix recurrence
  size: small
  depends_on: []
  description: 'capture-related-perf-flake: resolve bob-cli-ca in bob-cli src/native/capture_related/tests.rs.
    Both synthetic_vault_query_is_under_250ms and synthetic_vault_draft_contract_is_under_250ms
    still flake under parallel cargo test load after the paired-baseline fix 7b8c5ddb.
    Reopened by bob-cli-do.land +1 on 2026-10-28: just check f9d3fb57f9ada3da1680791bcbf3d156
    failed synthetic_vault_draft_contract_is_under_250ms with paired excess 517.54
    ms against 500 ms; isolated rerun passed and later just check 6dda6f4fd3b209fa1e6f3cff9e65b629
    passed. Keep the paired-baseline design (warmup, min(big-tiny) plus gross bound);
    retune the bound, pairing, or load isolation so the gate stays meaningful under
    parallel suite load. Verify with focused isolated and parallel runs plus just
    check / sase tool run check. Related closed bob-cli-bo and bob-cli-d9 are load-tolerance
    precedents, not duplicates.'
- id: cancel-reconcile-docs
  title: Stop claiming Cancel-row and reconcile apply the same canceled-link cleanup
  size: small
  depends_on: []
  description: 'cancel-reconcile-docs: resolve bob-cli-dj in bob-cli docs/task-status-hooks.md
    Canceled Task References. Lines ~157-161 correctly describe reconcile deleting
    the complete list item (including struck forms, authored prose, and sibling links).
    Lines ~184-189 describe the Ctrl+Shift+P Cancel row using deferral-prune dedicated-bullet/token
    semantics, then falsely claim struck links are untouched either way and that reconcile
    re-derives the same cleanup. The nav implementation matches the Cancel-row paragraph;
    reconcile matches the complete-item paragraph, so reconcile can later delete a
    struck link or mixed-prose bullet the immediate cancel kept. Correct the parity
    claim to state both behaviors. Verify with docs-check / just check. Do not change
    cancel or reconcile code in this phase unless a doc-only fix is impossible to
    state accurately.'
proposed_by: bbugyi200.athena.bob-cli-dp.land
proposed_with: grok/grok-4.6
create_time: 2026-10-28 15:35:22
status: done
bead_id: bob-cli-dw
---

- **PROMPT:** [prompts/202610/backlog_batch_b6.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch_b6.md)
- **PARENT:** [202610/backlog_batch_b5.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b5.md)
- **BEAD:** [bob-cli-dw](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dw/README.md)

# Backlog batch B6

## Context and triage

This is the required successor to `plan:202610/backlog_batch_b5.md`. B5 epic
`bob-cli-dp` is closed, its plan is done, and its four original tasks dg, dh, di, and dn
are closed with evidence. This successor is independent backlog work, not remaining B5
scope.

Listed every unresolved task with `sase bead list -t task -n 0` after those closes.
Re-read dt, dv, dr, ca, dj, dq, and 8n with audited reads. No duplicates. No active epic
owns dt, dv, dr, ca, or dj: dt/dv are SASE check failures found during B5 closeout (dt
filed by dp.land; dv filed here); dr is bob-mac-capture master CI red since 8b57693 and
is not owned by closed bob-cli-dk or bob-cli-dl; ca was reopened after the
paired-baseline fix and has no causal active epic; dj is a docs contradiction about
cancel vs reconcile.

Skipped `bob-cli-dq` for this batch: complete-label epic `bob-cli-du` commits a
status-changing capture of a source task, which is the same vanished-entry trigger, so
that panic belongs with that in-progress work rather than a parallel backlog phase.
Skipped `bob-cli-8n` (vault AGENTS.md Obsidian Sync sentence) in favor of red gates and
the already-audited docs bug dj.

Ranked red gates and flakes first (dt, dv, dr, ca), then misleading docs (dj). Skipped
snoozed ah (2026-11-04), aa (2026-11-06), cp (2026-11-16), and 6q (journal completeness
around 2026-11-09; use live snooze state). Human-dependent Mac acceptance ay/cx, crontab
30, sudo installs ax, approved write passes cm/co, and running-app UX review 20 need a
gate, question, or decision, never invented evidence. No feature or memory work is
included, so no decisions are required.

Open external repos with `/sase_repo` before source investigation: sase for dt and dv,
bob-mac-capture for dr. Include every edited repo in host completion obligations,
including sase, bob-mac-capture, and plans; reopen them on a continuation to keep
opened-repo obligations visible.

## Phase recovery-assessment-public

Open sase and read bob-cli-dt, including the private import of `_RecoveryAssessment`
from `_failed_close_recovery_assess.py` into `_failed_close_recovery_snapshots.py`,
pre-B5 source dcb7d2a6e7, and KNOWN witness `13ea2a464abef73b3eab4bfee9e624d7`. Expose
the shared type through a public name and update consumers. Do not suppress or
whitelist. Verify with focused failed-close tests plus `sase tool run check`.

## Phase lsp-store-lock-flake

Open sase and read bob-cli-dv. Make
`test_lsp_store_rechecks_after_waiting_for_identity_lock` wait on real lock contention
instead of a 0.3s fake-cargo race. Prove isolated and parallel stability. Verify with
`sase tool run check`.

## Phase mac-unconfigured-theme-ci

Open bob-mac-capture and read bob-cli-dr, including the three consecutive master CI runs
40724077224, 40728493998, and 40729693701. Restore the two unconfigured-theme
caret-underline XCTests. Master currently has no green macOS signal; this phase exists
to recover that signal.

## Phase capture-related-perf-flake

Read bob-cli-ca, including the post-fix recurrence on 2026-10-28 (paired excess 517.54
ms vs 500 ms under `just check`, isolated pass). Retune the paired lib gates so they
remain meaningful under parallel suite load. Verify with focused runs plus `just check`.

## Phase cancel-reconcile-docs

Read bob-cli-dj and `docs/task-status-hooks.md` Canceled Task References. Correct the
false claim that the Cancel row and reconcile apply the same cleanup. Docs-only unless
accuracy is impossible without a code comment pointer.

## Landing and successor rule

Review all phases/notes, their source and commits, and changes landed since this batch
began in all touched repositories. Finish epic-caused gaps before closing. Verify each
acceptance with its focused suite plus the required repository gate
(`sase tool run check` in sase, bob-mac-capture, and bob-cli as applicable; never claim
green on NEW/UNKNOWN failures). For genuine unrelated proposals use `/sase_new_task`,
record every disposition and proposing bead, and avoid duplicating active epic work.
Close dt, dv, dr, ca, and dj normally with evidence only once each acceptance is met.
Never force a successful close.

Before closing this epic, resolve or re-key every entry from
`sase bead epic-symbols <this-epic>`, close with a detailed verification note, run
`just symvision` if available, and set its plan frontmatter `status: done`. Follow any
parent_bead link under the normal nested-landing rules; this successor is authored
without one.

As the lander's final action, use `/sase_plan` to author and propose the next backlog
epic, and copy this successor rule into it. Continue until no unresolved task beads
remain:

1. List every unresolved task with `sase bead list -t task -n 0`; re-read candidates
   with audited reads. Close only verified duplicates (`superseded`) or work already
   fixed (`done`), with evidence. Recheck this triage instead of trusting it.
2. Choose at most five important remaining beads in at most five phases, with unique
   slugs and explicit depends_on. Rank red gates/flakes, then correctness and data
   safety, misleading memory/docs, then features. Open external repos with `/sase_repo`
   before source investigation, including sase for remaining runner defects and the
   vault for 8n.
3. Feature work gets a local yes/no decision. Memory changes require
   `/sase_memory_write` and its memory decisions; no unsupported permission assumption.
   Keep other decisions to a minimum.
4. Skip snoozed tasks until wake: ah 2026-11-04, aa 2026-11-06, cp 2026-11-16, and 6q
   (still snoozed; journal completeness around 2026-11-09; use live snooze state).
   Human-dependent Mac acceptance ay/cx, crontab 30, sudo installs ax, approved write
   passes cm/co, and running-app UX review 20 need a gate, question, or decision, never
   invented evidence.
5. Validate with --explain, repair, revalidate without --explain, and propose the
   validated plan. Wait for the proposal command itself to exit; its successful handoff
   ends the turn.
