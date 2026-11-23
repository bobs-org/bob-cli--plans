---
tier: epic
title: Refresh only reference plans affected by concurrent edits
goal: Complete bob-cli-pj.2 with dependency-aware PDF replanning at all three stale
  sites, bounded retries outside writer locks, and preserved fail-closed guards.
phases:
- id: dependency_classification
  title: Track PDF dependencies and classify scan changes
  size: medium
  depends_on: []
  description: 'dependency_classification: record actual shared reads, retain per-PDF
    locator inputs, build many-owner evidence, and classify all snapshot differences
    with fail-closed precedence. Keep existing rebuild behavior until the refresh
    consumer lands.'
- id: preparation_refresh
  title: Refresh affected plans during unlocked preparation
  size: medium
  depends_on:
  - dependency_classification
  description: 'preparation_refresh: preserve unfinished preparation on coherence
    mismatches, add a bounded subset refresh with correct draft replacement and shared
    finalization, and use it for both during-planning and immediate post-preparation
    validation.'
- id: lock_refresh_and_contract
  title: Refresh outside writer locks and prove the complete contract
  size: medium
  depends_on:
  - preparation_refresh
  description: 'lock_refresh_and_contract: carry the refresh budget through lock revalidation,
    preserve recovery/intake and receipt rules, complete adversarial interleave coverage,
    document the coherence contract, and run the full repository gates.'
proposed_by: bbugyi200.athena.bob-cli-pj.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pj.2
create_time: 2026-11-22 18:51:54
status: done
bead_id: bob-cli-pj.2.1
---

- **PROMPT:** [prompts/202611/refs_incremental_coherence.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/refs_incremental_coherence.md)
- **PARENT:** [202611/tick_contention_noise.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/tick_contention_noise.md)
- **BEAD:** [bob-cli-pj.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pj/bob-cli-pj.2.1.md)

# Refresh only reference plans affected by concurrent edits

## Ownership and scope

This is a child epic implementing **bob-cli-pj.2**, phase `refs-incremental-coherence`
of `plan:202611/tick_contention_noise.md`. The phase is already in progress and has no
prior remaining-work notes. Its prerequisite is present in commit `badcd30c`: ordinary
coherence churn reports `inputs_changed`, destination-asset changes retain
`scan_failed`, and tick handles contention with the approved two-hour stall window. Do
not redo those changes or reopen the assigned phase. No code for incremental refresh has
landed yet.

Use an epic because dependency observation, optimistic retry bookkeeping, and the
writer-lock lifecycle are separate substantial units with distinct correctness tests.
Each phase above fits one coding agent and has an explicit dependency. The intermediate
phases must compile and pass their relevant checks without weakening current validation.
No new memory, public CLI option, JSON field, plugin change, release version, or
changelog edit is needed. The parent's `stall_after = two_hours` decision is final and
remains unchanged.

Child phase workers close only their assigned child phase. This child epic's land agent
verifies the combined implementation and closes **bob-cli-pj.2** with the verification
evidence; it must not close **bob-cli-pj**, any outer plan bead, or any other ancestor.
Follow the normal host lifecycle for this child epic itself. Before closing the original
phase, run `sase bead epic-symbols bob-cli-pj.2` and resolve or re-key every remaining
entry. None were present during planning. Discovered work outside this scope is recorded
with `sase bead note bob-cli-pj.2 'PROPOSED FOLLOW-UP: ...'`, never by creating task
beads in a phase worker.

## Observable result and non-goals

An edit to an existing unrelated daily note must not cause another full-library PDF
planning pass. Recompute the cheap shared Markdown-derived finalization as needed; parse
a PDF again only when its owned or shared inputs changed. Library/global changes may
still rebuild, destination-asset changes still fail closed, and sustained relevant churn
must terminate with `inputs_changed` within a fixed invocation budget. All refresh
parsing stays outside scan-owned writer locks. The one exception already in the code
remains: after an owned intake move, plan only newly intaken or late-audio PDFs under
the locks, with no retry after a move.

Do not add a persistent PDF cache, optimize the initial full-library parse, change
cron/cadences, mask network errors, or alter PDF coordination and overwrite policy.

The parent design has overlapping wording: its classifier says **any `ref_dir` change is
global**, while its test list says a sidecar _or_ ref-note edit replans one PDF.
Preserve the explicit conservative global rule, and exercise the targeted test with a
sidecar. An existing ref-note edit therefore still rebuilds. Document this exception
rather than promising that every owned input is local.

## Verified code and read inventory

Relevant files are under `src/native/highlights_ref/` unless stated otherwise:

| Input/read                                                                                                                           | Current implementation                                                                                         | Planned handling                                                                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| PDF, `.md`/TextBundle sidecars, both TextBundle text filenames, nested assets, named screenshot sources, audio, prospective ref note | `scan_plan.rs::snapshot_pdf_companion_states`, `sync.rs::plan_pdf_sync`                                        | Exact path/content evidence, including absence, with one or more PDF owners                                                |
| Routed writes/intents, reading Insert destination, reading LineEdit residence, archive metadata                                      | `capture_scan_evidence`, `track_archive_for_note`                                                              | Add the owning PDF to every tracked path, retaining owners from both pre- and post-plan observations                       |
| Reading-task candidates                                                                                                              | `plan_pdf_sync` reads `ctx.index.candidates(ref_note_rel)` before selecting V1/V2                              | Retain that per-PDF candidate vector and compare it with the fresh context                                                 |
| Underline/section redirect links                                                                                                     | `sidecar_render.rs` calls `vault_links::markdown_links_to_block_ids`                                           | Set `consulted_vault_links` only when a nonempty ID set actually causes the walk                                           |
| Birth/reopen parent lookup                                                                                                           | `sync.rs` archived-source probe and `plan_reading_task` resolver closure                                       | Set `consulted_parent_targets` when the resolver is invoked, including misses                                              |
| Reread lineage                                                                                                                       | `birth.rs::derive_birth_lineage` calls `ref_library::build_index`, reads predecessor notes and follows lineage | Set `consulted_lineage` when this derivation is attempted; conservatively include these plans for Markdown content changes |
| Saved source at birth                                                                                                                | `derive_birth_source_markdown` reads `.bob/ref-sources/<type>/<stem>--*/code.json` and source bytes            | Explicitly snapshot this hidden tree and classify its membership/content changes as global                                 |
| Annotation target route                                                                                                              | `annotation_tasks.rs::route_name_to_note_path` checks `<bob>/<route>.md` existence/type                        | Markdown membership and owned target evidence cover this; no broad content dependency                                      |
| Annotation deduplication and preview-ID occupancy                                                                                    | `finalize_annotation_task_plans`, `reserve_preview_ids`, archive-pair census                                   | Always rerun finalization over all retained/replanned drafts with the fresh context on a refresh                           |

Additional findings that implementation must account for:

- `parent_notes::parent_candidates` filters records/notebooks out of
  `capture_targets::scan_capture_targets`. Area/project candidates come from root files.
  Record discovery may walk more broadly, but those targets and its diagnostics do not
  feed the parent candidates. There is no current `VaultSnapshot::load` in this path. A
  failed notebook/album parent lookup can follow `HomeResolver` for its refusal
  diagnostic; keep failed outcomes conservative instead of treating them as
  dependency-free.
- `RefTaskIndex` derives `Clone`, and `LocatedRefTask` derives `PartialEq/Eq`. Its whole
  `Debug` includes candidates, follow-ups, orphans, and archive-pair facts. `line_index`
  and the original candidate line are meaningful writer inputs: a line shift for a PDF's
  own candidate must replan that PDF. A follow-up/orphan or other PDF's line shift must
  not force every PDF through planning. Fresh archive-pair facts still feed shared
  finalization.
- `markdown_links_to_block_ids` returns immediately on an empty ID set; the underline
  call currently invokes it even for an empty set. Setting its flag merely because that
  call site was reached would unnecessarily mark nearly every sidecar plan.
- `snapshot_vault_md_states` uses `capture_dependency_tasks::vault_note_paths`, which
  excludes more directory names than `vault_links` does. Make the coherence snapshot
  cover the union of Markdown paths actually read by planning/finalization, using
  checked enumeration. Do not narrow link semantics to the smaller walker or silently
  treat an unreadable tree as an unchanged empty tree. Hidden source-store files need
  their own snapshot because both ordinary Markdown walkers exclude `.bob`.
- The existing debug-only `scan_test::log_plan_pdfs` already records
  `locked=0|1 count=N <paths...>`, and `scan_interleave.rs` can attach its log. Reuse it
  for exact subset/count assertions. No public receipt field is required.

## Internal design

Names below are suggestions; the data ownership and invariants are requirements.

### Evidence and classifications

Retain a structured pre-plan snapshot independently of the eventual finalized evidence.
It includes discovered library/xlib membership, planned PDFs, config, intake states,
Markdown membership and bytes, source-store membership and bytes, PDF hashes, and
companion/target/archive states. Keep a many-to-many
`BTreeMap<PathBuf, BTreeSet<PathBuf>>` owner map. Directory-tree descendants inherit
their companion owner; shared target files have every owner, not the last owner. Build
ownership before planning for all PDFs (including Held/Failed), then augment it with
paths discovered by successful plans and finalization. Never assign the entire vault
snapshot to every PDF.

Retain the `ScanContext` or equivalent per-PDF candidate snapshots used by the plans.
When selected PDFs are replaced, their candidate baseline and dependency flags must be
replaced too. On a stable refresh, retained candidates must equal the new context; do
not adopt a changed index as evidence for an unrefreshed plan.

Replace first-mismatch/string-driven control flow with a complete typed change
classification. Keep human reasons deterministic, and preserve the existing error
constructors/messages/codes. A useful result is `Unchanged`, `Refresh { pdfs, reason }`,
`Rebuild { reason }`, or a hard failure. A `Refresh` with an empty PDF set means shared
finalization/evidence refresh only, not another `plan_pdfs` call.

Inspect **all** changed/added/removed paths before deciding, with this precedence:

1. Any destination-asset existence/content change fails with `scan_failed`, even when a
   locator or global change also occurred. Do not hide it behind the first sorted path
   or bless the modified asset as a fresh baseline.
2. Global changes require a full rebuild: library/xlib or planned-PDF membership,
   config, intake states, Markdown membership, any observed `ref_dir` input, and
   `.bob/ref-sources`. Missing/unclassified evidence falls back conservatively to global
   handling, not acceptance.
3. Other owned content changes select all owning PDFs; PDF-byte changes select the PDF
   itself. Source companions remain local unless a higher-priority rule applies.
4. Markdown content changes also select vault-link consumers and lineage consumers;
   root-note content selects parent consumers. Apply this union even when a Markdown
   path has an owner: a note owned by PDF A can also affect PDF B's link or parent
   lookup. Unowned Markdown with no such consumers selects no PDF.
5. Diff each PDF's old and fresh `index.candidates(ref_note_rel)` vector. Select only
   PDFs with different candidates. Changes to other index fields require fresh shared
   finalization, not wholesale PDF parsing.

For Failed outcomes whose partial dependency observations are unavailable, select them
conservatively on shared Markdown/index changes; do not retain a stale missing parent
failure forever. Held outcomes retain their companion/archive dependencies. Where
recording partial flags through the planning wrapper is simpler, it is also valid, but
no failure may lose a dependency merely because it lacks `PdfSyncPlan`.

Preserve the existing distinction between newly observed absent paths and files that
actually appeared. The ordinary pre/post comparison must not treat a planner naming an
absent target as creation. Preserve the explicit expected-intake destination exception
where it applies; never use it to excuse external edits before an owned move.

### One refresh algorithm, bounded for the invocation

`PreparedFullScan` must survive a non-global during-planning mismatch, rather than being
discarded by `prepare_from_inputs`. Separate initial discovery/draft assembly from
convergence so all three stale sites can use the same refresh primitive.

For every non-global refresh:

1. Take a fresh pre-snapshot and fresh `ScanContext` with the pinned invocation date.
   Compare the **last trusted snapshot** to this pre-snapshot again and union newly
   affected PDFs with those already selected. This catches edits between validation and
   refresh; never replace the baseline first and silently bless them. Recheck
   destination/global precedence before doing incremental work.
2. `reset_prepared_finalization`, restoring every retained plan's raw candidates and
   pre-finalization rendered body/action. Remove both draft-map entries for every
   selected PDF before replacing it, even if its new outcome is Held or Failed.
3. Call `plan_pdfs` only for the selected ordered subset; skip it for an empty set.
   Splice outcomes back into the canonical discovery order. Capture raw draft maps for
   newly Planned outcomes before finalization. Replace dependencies/owners for those
   outcomes without dropping ownership needed to compare the prior evidence.
4. Finalize **all** Planned outcomes in the existing order:
   `finalize_annotation_task_plans`, `reserve_preview_ids`, then
   `validate_planned_asset_collisions`. Keep output-collision checks where membership is
   discovered/rebuilt and in the existing intake path. Use the fresh context for
   finalization, even when zero PDFs were reparsed.
5. Capture post evidence, compare pre/post membership and content, compare the actual
   context inputs, and rehash PDFs on disk. Successful plan hashes must still match
   `marker.source_pdf_sha256`; Held/Failed PDFs retain direct hash comparisons. Never
   compare an old planner hash to itself as proof that disk bytes are unchanged.
6. On stable evidence, return the updated prepared scan. On another mismatch, repeat
   classification and consume the remaining budget; hard errors abort immediately.

Use a named budget: **four incremental refresh passes and at most one full rebuild** for
ordinary coherence over the whole invocation. Initial planning is not a refresh. An
empty-subset finalization pass consumes a refresh pass too. A global change uses the
full-rebuild allowance immediately; after incremental passes are exhausted, use that
allowance if it remains. Do not reset allowances when crossing functions or reacquiring
locks. A full rebuild uses current discovery/config-dependent reads and the same pinned
date; persistent mismatches after the remaining allowance is spent return
`inputs_changed`. This bounds ordinary PDF planning calls to initial planning, four
subset calls, and one rebuild (fewer if some subsets are empty). Existing
recovery/intake work remains separately guarded and must not reset this budget.

The three consumers are:

- During-planning comparison in `prepare_from_inputs`.
- Immediate validation in `prepare_full_scan_with_retry_date`.
- `scan_apply.rs::lock_validate_intake_and_execute`: classify while locked, drop both
  locks before any refresh/rebuild, then reacquire in maintenance/scan order and
  validate again. A refresh that eliminates all work uses the existing no-work return.

Do not turn hard finalization/I/O/collision errors into retries. Preserve pinned-date
tests, dry-run behavior, `-w` policy, configured PDF coordination, hooks running once,
recovery before hooks, receipt ownership, and output/receipt byte agreement. After any
owned intake move, no new refresh/rebuild branch is legal.

## Phase work and acceptance

### `dependency_classification`

Implement dependency observations in `model.rs`, `sync.rs`, and `sidecar_render.rs`,
with narrow helper changes as needed. Keep flags on the plan (or one nested observation
struct). Track link reads in renderer results, actual parent resolver invocations, and
lineage calls, including empty/missing results. Snapshot the saved-source tree
explicitly and cover the Markdown walker union.

Refactor evidence in `scan_plan.rs` into the structured snapshots/owner map and typed
classifier described above. Store per-PDF candidate vectors or a retained context.
Initially translate `Refresh` and `Rebuild` back to existing stale/rebuild behavior at
production call sites, so this phase changes observation fidelity without an unfinished
retry path. Avoid introducing unused public helpers or broad lint allows.

Acceptance tests cover exact owner sets (including shared targets), own PDF/source
changes, unowned/root Markdown consumer selection, owned-plus-shared dependencies,
per-PDF locator changes and line shifts, global membership/config/ref/source changes,
and destination-asset precedence when other mismatches coexist. Include an existing note
in a directory read by `vault_links` but excluded by the dependency walker. Verify empty
link sets do not set a broad dependency. Keep existing scan tests green.

### `preparation_refresh`

Implement reusable draft replacement/finalization helpers, checking
`integrate_intake_plans` as the existing retained-plan example. Add the budget and
convergence loop, keep provisional preparations available, and wire both unlocked stale
sites. Share helper logic with intake where appropriate without adding retries to
intake. Retained outcomes and preview reservations must remain deterministic.

Add focused unit/interleave coverage for edits at `after_snapshot`, `planning`, and
`before_evidence`: an existing daily note with no consumers causes zero additional PDF
parses; one sidecar or PDF selects one of two PDFs; a second edit affecting a different
PDF between detection and refresh is included; finalization-only changes update
deduplication/ID reservations without reparsing unaffected PDFs. Test a
Planned-to-Failed/Held replacement and a later recovery without stale draft entries.
Prove the bounded preparation fallback and pinned date. Add a narrow debug pause before
refresh snapshot/after initial preparation validation only if required to exercise a
timing window not covered by the existing seams.

### `lock_refresh_and_contract`

Wire the same refresh operation and remaining budget into the lock-window stale branch.
Preserve the existing recovery and fresh-intake comparisons as full-rebuild causes; keep
their current receipt/error ownership and finite guard. Ensure a recovery-triggered
rebuild cannot grant an unbounded new coherence budget. Add no retry after
`execute_xlib_intake_counting` starts moving files.

Extend `tests/cli/highlights/scan_interleave.rs` using real handshakes and the current
plan log. Adapt `repeated_churn_after_prepare_terminates`, which currently assumes
exactly two `before_lock` pauses, to drive every allowed retry and assert a finite upper
bound without test-handshake timeouts. Where behavior is expected to converge, assert
success and exact counts rather than allowing either success or refusal.

The final regression matrix must include:

| Scenario                                                                                   | Required evidence                                                                     |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| Edit an existing daily-like `2026/day.md` during planning and between prepare/lock         | Success; unchanged unrelated PDF parse count; original full pass only                 |
| Edit one of two PDFs or one TextBundle sidecar                                             | Only its owning PDF reparses; latest content reaches the output                       |
| Add a link in an existing unowned note to a block whose plan consulted links               | Only relevant shared-read consumers replan; redirect/stub reflects the new link       |
| Edit a root parent alias/status for a birth; include an owned root shared by another birth | Parent consumers replan; route/result reflects the current root note                  |
| Move/change one PDF's located task; alter unrelated index follow-up/orphan data            | Candidate change reparses its PDF; unrelated index data does not reparse every PDF    |
| Change annotation dedup or archive ID occupancy without changing per-PDF inputs            | Shared finalization updates; IDs/intents are neither duplicated nor retained stale    |
| Add/remove vault Markdown, library/xlib entry, or change config/ref/source-store input     | Global rebuild, with new discovery and current values                                 |
| Change a destination asset, also with locator/global changes                               | `scan_failed`, no refresh that accepts the damaged bytes, no unsafe overwrite         |
| Relevant churn through preparation and repeated lock reacquisition                         | `inputs_changed` within the shared budget, no intermediate abandoned-retry receipt    |
| Refresh while a maintenance writer can acquire its lock                                    | Every refresh plan-log line has `locked=0`; only intake subset planning may be locked |

Retain all existing tests for no-work under held locks, same-length edits, source
images, missing/new TextBundle inputs, two scans racing on intake/late audio, recovery,
prospective collisions, dry-run/human/JSON agreement, and receipts.

Update `docs/highlights-ref-sync.md` in the scan/writer-lock section: unrelated content
edits avoid PDF replanning, owned non-global inputs refresh their consumers, explicit
global categories rebuild, and sustained churn yields `inputs_changed`. Preserve the
destination-asset `scan_failed` exception and explain that finalization can rerun
without parsing PDFs. Follow `docs.md` memory; no memory changes are authorized.

## Verification and landing

Each worker runs formatting and the focused tests for its code. Useful commands:

```sh
cargo fmt --all -- --check
cargo test --lib highlights_ref
cargo test --test cli highlights::scan_interleave
cargo test --test cli highlights
```

The final implementation must pass `just check` (format, Clippy and the complete
workspace test suite) and `just docs-check`. Use `/sase_monitor` for long checks,
waiting for the staging command itself to exit before finalizing the turn. Do not launch
a long check and end the provider turn while it remains an unmonitored child.

Record command results and exact replan-count evidence on the applicable child bead and
original phase as needed for landing. If a gate fails identically on a clean base,
record a `PROPOSED FOLLOW-UP:` on bob-cli-pj.2 with the reproduction and any existing
tracking bead, then finish this phase as instructed; do not leave completed work open
solely for that baseline failure. All new failures must be fixed.

The child land agent checks the combined change, both gates, all three refresh sites,
the finite budget, and remaining epic symbols before closing bob-cli-pj.2 with a
verification note. The outer epic's land agent retains responsibility for bob-cli-pj.
