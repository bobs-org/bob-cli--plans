---
tier: epic
title: Narrow the full highlights scan lock window
goal: Full highlights scans plan without writer locks and acquire them only for validated
  intake, recovery, or writes while preserving paired PDF and reference-note publication.
phases:
- id: prepared-scan
  title: Share full-scan planning and capture validation inputs
  size: medium
  depends_on: []
  description: 'prepared-scan: extract a shared prepared full scan, an exhaustive
    mutation predicate, and input validation for optimistic planning; retain the existing
    lock window while establishing tested helpers.'
- id: locked-apply
  title: Apply prepared scans inside a bounded mutation window
  size: medium
  depends_on:
  - prepared-scan
  description: 'locked-apply: move full-library planning and hooks outside the locks,
    validate before mutation, preserve protected intake and recovery, and make no-work
    scans finish without either lock in human and JSON modes.'
- id: scan-proof
  title: Prove scan interleavings and document the new lock window
  size: medium
  depends_on:
  - locked-apply
  description: 'scan-proof: complete adversarial concurrency coverage, update lock
    and receipt contracts, and run the full code and documentation gates with clean-base
    attribution for unrelated failures.'
proposed_by: bbugyi200.athena.bob-cli-mr.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-mr.3
create_time: 2026-11-17 12:14:05
status: wip
bead_id: bob-cli-mr.3.1
---

- **PROMPT:** [prompts/202611/highlights_scan_lock_window.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/highlights_scan_lock_window.md)
- **PARENT:** [202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md)
- **BEAD:** [bob-cli-mr.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mr/bob-cli-mr.3.1.md)

# Narrow the full highlights scan lock window

## Assignment and fixed decisions

This is a child implementation epic for **bob-cli-mr.3**, implementing the already
approved `scan_lock = narrow` branch of
`plan:202611/backlog_triage_t1_red_gates_closed_prereqs_scan_lock.md`. The defect is
tracked by `bob-cli-mo`; the original safety requirement is recorded on `bob-cli-9c.1`.
Do not re-ask the narrow/defer/cancel decision.

The planning checkout is clean at `b0bddbb9`. The preceding capture fix is present.
There are no previous implementation or remaining-work notes on bob-cli-mr.3, and the
full-scan lock change has not landed: both human and JSON paths still lock before the
hook, intake, and every PDF's planning. `sase bead epic-symbols bob-cli-mr.3` currently
reports no entries. Recheck the actual checkout and phase notes before implementing; do
not repeat work that has since landed.

Three sequential medium phases separate the shared planning substrate, the concurrency
protocol, and the cross-process proof/docs. The protocol spans two independent
orchestration paths, partial failures, recovery, and receipt ownership; this should not
be one oversized coding assignment. Each phase includes its own focused tests and must
leave a usable tree.

The child epic's land agent completes bob-cli-mr.3 after all phases and verification. It
does not close bob-cli-mr or any other ancestor. The current task instruction says to
close only bob-cli-mr.3, so provide the original epic's land agent with resolution
evidence for bob-cli-mo rather than closing that task here. Child workers close only
their assigned child phase. Do not hand-edit bead statuses or create task beads; append
`PROPOSED FOLLOW-UP:` entries to the assigned phase. No memory edits, release-version
edits, or changelog edits are authorized.

## Observable result and invariants

- A writing full scan with no intake, recovery, or actual vault/PDF writes plans the
  library without taking either `bob_sync.lock` or `scan.lock`. It completes even when
  either lock is held, without a lock-wait message. Dry runs remain read-only and take
  neither lock.
- A scan that will mutate acquires **bob_sync.lock, then scan.lock**, validates what it
  planned, and keeps both throughout the actual intake and writes. It never holds
  scan.lock while trying to acquire bob_sync.lock.
- Moving an xlib PDF into lib and filing its reference note stay in the same protected
  interval. Never release locks after the move to do slow planning. New
  optimistic-validation failures must occur before intake publication, not leave a new
  unprotected PDF-only interval. Preserve existing partial-failure accounting and
  per-PDF error isolation.
- Expensive parsing/rendering of the existing library is outside the lock, even when a
  few PDFs need updates. Do not solve validation by unconditionally repeating the full
  library's expensive planning under the lock.
- The existing `lock_busy` and `scan_busy` envelope shapes, hints, named lock path, wait
  budgets/override, JSON field order, and schema version remain unchanged. Genuine lock
  failures still write no receipt. No public flag is added.
- Keep the macOS per-PDF `.forMerging` scope, timeout budget, fresh-byte marker checks,
  editor/timeout/marker holds, and dirty-target policy. Optimistic scan validation is
  additional protection, not a replacement for those checks.

Read the governing decisions through `sase memory read` before changes:
`decisions:references-are-ingested-at-birth` and
`decisions:open-pdf-writes-are-coordinated`. If implementation reveals a genuine
contradiction, record it for the parent land agent; do not edit accepted decisions.

## Current code and important traps

All implementation belongs to bob-cli. No linked plugin or Mac application change is
needed; human and JSON are two presentations of the same protocol.

- `src/native/highlights_ref/sync.rs`: `scan_library`, `scan_library_write`,
  `scan_library_dry_run`, `plan_pdfs`, `reserve_preview_ids`,
  `finalize_annotation_task_plans`, and the per-PDF executors.
- `scan_json.rs`: `scan_library_json` duplicates the lifecycle; lock helpers and
  report/error renderers also live here. Human and JSON currently even finalize
  annotation tasks and reserve IDs in different orders. Unify the shared pipeline
  without changing final IDs or task links, proving both legacy and v2 behavior.
- `model.rs`: `ScanContext` contains the whole-vault reading-task locator index and
  invocation date. `PdfSyncPlan` includes more than a ref-note and marker write: routed
  intentions, reading-task actions, annotation work, and image assets.
- `guard.rs`: `scoped_write_path_set`, `scan_dirty_preflight`, and
  `intent_write_planned`; `report.rs`: `scan_plan_changed`. Neither existing helper
  alone is a proven exact no-work predicate: the latter omits explicit reading-task
  actions, and the path set includes already-present image assets.
- `doctor.rs`: `plan_xlib_intake` discovers PDFs, sidecars, companion audio, and late
  orphan audio; `execute_xlib_intake_counting` moves before planning today.
  `collect_pdf_paths(config, true)` can read xlib for a preview, but a persisted plan
  must use the correct library paths and post-intake companion metadata.
  `io.rs::rename_no_replace` already supplies supported-platform non-clobber moves.
- `hooks.rs`: pre-scan hooks run once, from BOB_DIR, with
  `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK=1`. JSON redirects hook stdout to stderr.
- `overwrite/recovery.rs`: `load_intent` is a read-only pending-recovery check;
  `recover_with_locks_held` mutates and requires both locks. Standalone vault-sync
  already guards recovery before staging. Never call mutating recovery unlocked.
- `scan_receipt.rs`: currently only scans that acquired scan.lock write last-scan.json
  and post the Darwin notification. Preserve that lock ownership: successful lock-free
  no-work scans and failures before lock acquisition produce their normal direct output,
  but leave the previous receipt untouched and do not notify. This is an intentional
  documented consequence of the new fast path. Do not acquire a writer lock solely to
  publish a no-work receipt, or publish it unlocked over another scan's in-progress
  result.

## Phase prepared-scan

Extract a shared owned preparation result (for example in `scan_plan.rs`) that human and
JSON can consume without repeating the full scan or printing while it plans. Keep the
existing lock behavior during this preparatory phase.

1. Share ordered PDF discovery, context construction, parallel planning, ID/task
   finalization, and collision checks. Keep per-PDF planned/held/failed outcomes and
   deterministic report ordering. Separate planning from output and execution. Keep one
   invocation date across retries and one coordination budget per scan. Retain raw plan
   drafts or an explicit reset operation so finalizing the combined intake/library set
   later cannot duplicate intentions or retain stale reservations.
2. Introduce an explicit actual-work predicate covering note/marker changes, missing or
   changed assets, routed insertions, reading-task Insert/LineEdit, and annotation
   tasks. An already-present identical asset is not work. A NoWrite reading action alone
   is not work. Do not call the mutation executor on the lock-free no-work branch just
   to obtain a report. Provide a read-only way to assemble equivalent unchanged outcomes
   and preserve diagnostics.
3. Capture validation evidence together with the bytes/index used for planning.
   Inventory the complete dependency set: PDF contents and annotation source;
   present/absent sidecars and source image/audio companions; ref-note preimages;
   target-note preimages; parent/routing/config inputs; reading-task locator and
   ID-occupancy inputs; archive/capture metadata consulted by holds; library and xlib
   membership and prospective destination absence. Shared locator changes can affect
   every plan, including one previously classified unchanged.
4. Use content/preimage checks, including existence and path membership, rather than
   only mtime/size. A source changing during preparation must not yield evidence from
   newer bytes attached to an older plan. A conservative snapshot of relevant
   Markdown/config/catalog trees, compared before and after the pure planning pass, is
   acceptable; enumerate those trees explicitly in code. Reuse the PDF hash already read
   by the planner where possible. Do not hash unrelated vault media or add a persistent
   cache. Existing library PDFs can be fingerprinted without repeating PDF
   parsing/rendering.
5. Supply validation returning unchanged/stale/error, and a bounded preparation retry
   path. It must detect created/deleted/renamed inputs as well as edits. Tests should
   directly mutate inputs between preparation and validation, so no production
   pause/debug flag is necessary.

Focused verification: existing full-scan human/JSON and annotation/reading-task tests,
plus unit tests for all categories of actual work, stable repeat scans, content changes
with unchanged length, missing/new inputs, and shared locator invalidation. This phase
must not strand unused helpers behind stale epic-symbols; integrate them into the
existing locked lifecycle or use a justified temporary entry owned by the still-open
locked-apply phase.

## Phase locked-apply

Use the prepared result from the first phase in both full-scan entry points. Keep
intake-only and create-time ingestion on their existing lock/sync/commit protocol; do
not accidentally route them through the full-scan fast path.

1. Validate layout and check pending overwrite recovery before running the hook. When
   recovery is present, acquire both locks, re-read the intent, recover, then release
   before ordinary preparation. A recovery error stops before the hook; a dry run only
   reports. Check recovery again at the mutation boundary, since an overwrite could
   start during unlocked planning. A required recovery invalidates the old prepared
   result; never reuse it afterward.
2. Run the hook once before intake discovery and expensive planning, with neither
   scan-owned lock held. Preserve --no-hooks, dry-run behavior, environment,
   stdout/stderr routing, and hook-failure semantics. Do not replay it on a stale plan
   retry. A hook invoking vault-sync now lets vault-sync own its lock and complete
   before planning; update the former silent-no-op claim. Guard nested writing full
   scans using the existing in-hook marker so moving the hook out of the locks cannot
   turn the former contention failure into unbounded recursive hook execution. Use the
   existing error envelope, not a new public option.
3. Discover xlib work read-only and prepare the existing lib outside locks. Pending
   intake is work even when the existing library is unchanged; handle xlib-only initial
   libraries and late audio. A stable result with no pending recovery, intake, or actual
   writes returns directly without either lock, preserving per-PDF holds/failures and
   the write-mode summary/exit code.
4. For work, acquire bob_sync.lock then scan.lock. Recheck layout, recovery, membership,
   intake collisions and the prepared read set before the first move/write. Recompute
   dirty-target classification under the locks. If stale, release both without moving
   anything, rebuild outside locks, and retry at most once. Persistent churn returns the
   existing generic scan_failed envelope with a clear rerun message; it must not spin or
   fall back to parsing the whole library while locked. A scan whose replan becomes
   no-work uses the fast path.
5. Once validation passes, re-discover and validate intake while still locked. No stale
   source/destination list may overwrite another writer's new bundle; use non-clobber
   moves. Move and plan only the newly intaken/late-audio-affected items inside this
   same protected scope. Existing library plans are reused. Rebuild shared ID
   reservations/finalization over the combined ordered set before execution so intake
   and existing plans do not allocate conflicting task identities. Do not rerun
   expensive planning for unrelated library PDFs. Keep both locks through all intake
   note writes and error cleanup. New retry branches are forbidden after any move;
   preserve accurate completed-intake accounting on partial failures. Reuse existing
   intake rollback protections if a new failure path needs to undo owned moves; never
   overwrite intervening data in cleanup or change the full scan into a sync/commit
   drain.
6. Execute the final plans using existing dirty-target holds, per-write preimage checks,
   cross-PDF deduplication, and MarkerWriteSession. In particular recheck PDF/annotation
   evidence for note-only writes, not just when writing a marker; otherwise a save
   during the unlocked interval could import stale annotations. The in-accessor
   coordination checks remain authoritative for an editor save after validation. Do not
   reject or bypass their supported same-marker save.
7. Keep receipt publication serialized with the mutation interval. A completed writing
   no-work scan publishes no receipt. Preflight failures now preceding lock acquisition
   also publish none. Once execution has acquired the writer locks, preserve
   success/partial/error receipt bytes and notifications; an abandoned retry must not
   publish an intermediate receipt. If recovery alone performed work, settle its scan
   result while appropriately serialized rather than emitting a later unlocked receipt.
   Document and test this case explicitly.

Focused tests in this phase must prove lock-free no-work human and JSON scans, lock
ordering/error compatibility for real work, stale-plan rejection/replanning, and intake
staying inside a single protected interval. Existing recovery and coordinated marker
tests must still pass before this phase closes.

## Phase scan-proof

Add deterministic adversarial tests using injected Rust callbacks or subprocess
handshakes with bounded waits. Do not use narrow wall-clock pass thresholds or
sleep-and-hope races. Keep subprocesses reaped, lock paths and state isolated, and hooks
local to temporary vaults.

Required scenarios (combine fixtures when appropriate):

1. A populated, converged full scan succeeds while both locks are externally held;
   verify each held lock independently too. Assert no mutation, no wait line, correct
   unchanged counts, and unchanged sentinel receipt. Include an annotated PDF with
   already-existing assets, plus an empty valid library.
2. A paused expensive planning stage permits an independent maintenance-lock writer to
   complete. With one planned update among unchanged PDFs, assert the existing library's
   parser does not run again under the mutation locks. This proves the contention
   improvement structurally without a timing benchmark.
3. With real work pending, a held maintenance lock wins over a held scan lock; error
   shape/hint/path stay exact. With only scan.lock held, report scan_busy. No intake
   move, note write, or receipt occurs on either lock-acquisition failure.
4. Between preparation and acquisition, edit a PDF's annotations, a ref note, a sidecar,
   and a reading/routed destination. Also create/remove an input and move a reading
   task. The result is freshly planned or safely refused, never a stale overwrite,
   duplicate task, or stale no-work classification. Repeated churn terminates, and the
   hook runs once throughout.
5. Two scans prepare the same queued PDF. Only one can intake it; the other revalidates
   and reports the resulting state without duplicate work or an overwritten destination.
   Cover PDF plus companion sidecar/audio and late audio alone. Pause between move and
   note filing and prove a vault-sync-like lock taker cannot enter until the paired note
   exists. Exercise the existing failure/hold cases without introducing an unlock gap.
6. Pending overwrite recovery happens under both locks before a hook capable of syncing;
   an intent arriving during preparation invalidates the plan. Keep dry-run recovery,
   mismatched/invalid intents, and recovery errors covered.
7. Human, verbose human, JSON, and their error paths agree. JSON stdout stays one
   envelope even for a chatty hook. No-work/prelock failures do not replace the receipt;
   real writes and post-lock hard failures still do. Replace the current
   output-collision receipt fixture (now prelock) with an actual post-lock failure for
   that guarantee, and retain an assertion for the new prelock behavior.
8. Run existing coordinated marker, dirty-hold, replaced-capture, v1/v2 reading,
   annotation deduplication, create ingestion, and intake-only suites. The new lifecycle
   must not change their write semantics or require real Highlights.

Update behavior contracts after reading `docs.md` through `sase memory read`:
`docs/highlights-ref-sync.md` (hook, writer lock, receipt, recovery, troubleshooting)
and `docs/vault-git-sync.md` (shared-lock window and custom-hook paragraphs). Check
`docs/ref.md`, `docs/reference/mac-capture/refs.md`, and other docs for claims that
every writing scan publishes a receipt. Adjust only claims made inaccurate; keep
published URLs. Update lock/receipt comments in the Rust modules. No memory decision
needs changing: protected intake and coordinated per-PDF writes remain.

Run targeted tests first, for example `cargo test --test cli highlights::scan`,
`cargo test --test cli highlights::marker_write`,
`cargo test --test cli highlights::overwrite_recovery`,
`cargo test --test cli highlights::ingest`, and the new module's unit tests (verify the
filters select tests). Then run `just check` and `just docs-check`. Use `/sase_monitor`
for long gates and follow its staged handoff/finalizer procedure. Do not deploy, change
host cron, mutate the real vault, or require a new real-Mac acceptance run for an
unchanged coordination implementation.

## Landing and completion evidence

The child epic's land agent verifies the three phases and the exact implemented tree's
code/docs gates. Record concrete test names, final commit references, the lock-order
proof, the no-work result, and remaining limitations on bob-cli-mr.3. Reproduce any
unrelated gate failure identically on a clean base using an isolated checkout permitted
by the repository workflow; record it as `PROPOSED FOLLOW-UP:` with an existing task ID
if one already tracks it. An identical base failure does not leave this assignment open.
Do not silently waive a failure newly introduced by these phases.

Before each child phase closes, run `sase bead epic-symbols <assigned-phase>` and
resolve or re-key every entry to an appropriate still-open bead. Before completing
bob-cli-mr.3, run `sase bead epic-symbols bob-cli-mr.3` again. Close child work first,
then close bob-cli-mr.3 with its verified evidence through the authorized SASE
completion flow; do not close bob-cli-mr or other ancestor plans. Attach a clear
recommendation and evidence for bob-cli-mo to the parent phase's notes for the original
epic's land agent. Finish every root provider turn through `/sase_final`.
