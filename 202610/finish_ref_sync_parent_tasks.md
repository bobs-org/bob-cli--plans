---
tier: epic
title: Finish parent-note reference sync and close bob-cli-5y.7
goal: 'Complete the remaining ref-sync-v2 work on bob-cli-5y.7, verify safe births,
  cross-file status sync, residence projection, archived reopens, and annotation routing,
  then close that phase with concrete verification evidence.

  '
phases:
- id: v2-planning
  title: Model located reading-task actions and v2 note projection
  depends_on: []
  size: medium
  description: 'v2-planning: add pure v1/v2/birth/reopen planning, parent-free sync
    snapshots, status conflict handling, and managed-embed rendering with focused
    tests.

    '
- id: v2-execution
  title: Execute reading-task writes safely across files
  depends_on:
  - v2-planning
  size: medium
  description: 'v2-execution: implement guarded insertion, adoption, line edits, and
    archive reopen execution before PDF/ref-note writes, preserving destination edits.

    '
- id: scan-integration
  title: Connect all scan entrypoints and route annotation follow-ups
  depends_on:
  - v2-execution
  size: medium
  description: 'scan-integration: share one locator index across parallel planning,
    activate v2 sync in human and JSON scans, and rebase annotation writes at execution.

    '
- id: verify-ref-sync
  title: Finish reports, documentation, and acceptance verification
  depends_on:
  - scan-integration
  size: medium
  description: 'verify-ref-sync: complete human reports and docs, cover the full acceptance
    matrix, run the repository checks, and give the land agent closure evidence.'
proposed_by: bbugyi200.athena.0z7
create_time: 2026-10-09 15:45:01
status: wip
bead_id: bob-cli-62
---

- **PROMPT:** [prompts/202610/finish_ref_sync_parent_tasks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/finish_ref_sync_parent_tasks.md)
- **BEAD:** [bob-cli-62](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-62/README.md)

# Finish parent-note reference sync and close bob-cli-5y.7

## Outcome and authority

Finish the existing `ref-sync-v2` phase of `plan:202610/ref_tasks_live_with_parent.md`.
A newly scanned reference receives one ordinary reading task in its area/project/inbox
note; its reference note displays a managed embed. Existing v2 references synchronize
with the located task, including archived tasks. Existing v1 references retain their
current behavior. Completion includes verified implementation and closing
**bob-cli-5y.7**, leaving the original epic **bob-cli-5y** to its own land agent.

This is an epic because the prior single-agent tale finished only its shared
rendering/insertion helpers. The remaining planner, executor, orchestration, and
acceptance work have distinct testable boundaries. The four phases are deliberately
sequential: each depends on the preceding implementation, and several touch the same
sync files. Each phase is bounded direct implementation work (`medium`).

Before implementation, read the current bead with
`sase bead read bob-cli-5y.7 -r "Need remaining scope and latest completion evidence"`
and read these artifacts through `sase artifact read`:

- `plan:202610/ref_tasks_live_with_parent.md`: governing design, especially design calls
  1–4, specifications 1, 2, 4, and 5, and phase `ref-sync-v2`.
- `plan:202610/ref_sync_parent_tasks.md`: the earlier implementation plan; section 1 is
  substantially implemented, sections 2–5 remain.

The original epic decisions are final: `cancel_dropped_wrapper_refs = no`,
`memory_ref_parent_decision = no`, and `memory_glossary_ref_terms = no`. Make no memory
changes and do not ask these decisions again. The two skipped-memory follow-ups are
already notes 1 and 2 on bob-cli-5y.7; do not duplicate them.

Use temporary test vaults. This work does not install bob, mutate the live vault, change
linked repos/plugins/Mac capture, implement migration or parent capture UI, add CLI
switches, or retire the legacy freshness bypass. Those are other phases of the original
epic. Do not manually set runtime bead statuses. Phase workers record out-of-scope
discoveries as `PROPOSED FOLLOW-UP:` notes on their assigned phase; do not create new
task beads. Inspect new notes on the original phase before final closure so another
worker's evidence is not lost.

## Verified starting point

The planning checkout was clean at commit `40162297a54caf743a177d85266a41e8bfe1a838`. It
includes the locator/read-side work (`9041927`), scan JSON intake fixes (`e9a0ee1`), and
the shared ref-task helpers (`4016229`). Reassess current source at implementation time;
do not overwrite work that has since landed.

- `ref_tasks/line.rs` already implements rendering, alias sanitization, slug/ID
  allocation, managed embed formatting, and close stamps for arbitrary trailing IDs.
- `ref_tasks/insert.rs` exports
  `insert_ref_task(bob_dir: &Path, destination: &Path, line: &str, children: &[String]) -> Result<InsertedRefTask, String>`,
  returning `block_id`, `placement`, and `task_line`. It uses capture's insertion and
  staged-file writer, with bounded preimage retries. Preserve this interface for the
  migration phase.
- `RefTaskIndex::build`, `select`, `candidates`, `LocatedRefTask`, `find_trackers`, and
  `find_managed_embed` exist. Located paths include `.md` and are vault-relative; tasks
  carry exact original line bytes, line index, mark, block ID, archive flag, and
  residence. Path-qualified ref links are indexed even before the note exists.
- `parent_notes::resolve_parent` supplies canonical area/project/inbox routes and
  aliases. `mac_inbox` is an available pinned target even before its file exists.
- `highlights_ref/sync.rs` still parses only the in-note `^ref` tracker, and
  `note.rs::default_note_body` still births that tracker. None of the new task helpers
  is integrated into Highlights sync.
- `execute_pdf_sync` currently writes the marker and ref note before routed task notes.
  `finalize_annotation_task_plans` groups by destination and assigns stale whole-file
  snapshots to the first owning PDF. These must change for v2 writes.
- Human scan and `scan_json.rs` have separate orchestration paths; both must use the
  same context, finalization, and execution behavior. Preserve the JSON contract,
  including PDF-only intake reporting from `e9a0ee1`.
- Bead note 3 reports 24 passing `cargo test --lib ref_tasks` tests and a baseline
  failure in `doctor_reports_ref_tasks_and_parents_rows`. This planning turn did not
  rerun those tests; treat the failure as prior evidence, not a fresh verification.
- `sase bead epic-symbols bob-cli-5y.7` currently reports no entries.

## Shared design contracts

Keep the public activation in `scan-integration`, after both the new planner and
executor exist. Earlier phases add small internal modules and exercise their seams
directly while the existing command path remains functional. Do not introduce a
user-facing feature flag, temporarily enable incomplete v2 writes, or silently use the
old renderer for a v2 plan. Record the concrete interfaces on each completed phase for
the next worker; remove transitional unused helpers at integration.

Use explicit internal state, with equivalent names permitted:

- A command context containing one immutable `RefTaskIndex`, shared by all PDF planning
  workers, plus the invocation date.
- A reading-task plan distinguishing legacy, existing/adopted v2, birth, reopen, and
  diagnostic/missing states. Actions distinguish no write, insert, and exact line edit;
  carry the selected task and original line for optimistic validation.
- Separate full marker metadata, parent-free v2 sync projection/base/hash, and the
  residence parent rendered into note frontmatter. Never silently drop `parent` from
  required PDF marker validation.
- Execution results containing actual destination, final ID, task line, and action;
  final note rendering and reports use those results rather than preview IDs.
- Routed annotation insertion intentions with originating PDF ownership, not stale full
  destination snapshots. Per-PDF failures must not misattribute later PDFs' annotation
  writes to an earlier owner.

Reuse the existing lifecycle, capture insertion, staged writer, and locator code. Do not
put a second vault walker inside per-PDF planning. Preserve configured ref directories,
CRLF, authored fields/text, image assets, tombstones, created provenance, status
normalization, and existing `--prefer` semantics.

## Phase `v2-planning`: model, status resolution, and note anatomy

Primary files: `highlights_ref/model.rs`, `projection.rs`, `frontmatter.rs`, `note.rs`,
`region.rs`, `audio.rs`, and focused new helpers/tests under `highlights_ref/` as
needed. Add an index-aware planning seam usable by the executor and tests; wire the
public entrypoints only in `scan-integration`.

1. Classify missing notes as v2 births. Existing notes are v2 when a task is located
   outside the reference note or the body contains the managed embed. Existing in-note
   `^ref` and trackerless legacy notes retain the v1 branch exactly.
2. For births, adopt a uniquely located orphan before considering insertion. Otherwise
   resolve the marker parent through the shared resolver. Unresolvable, ambiguous, or
   terminal parents fall back to `mac_inbox` with the child bullet
   `⚠️ parent '<hint>' is not an open area or project · refile me with Ctrl+Shift+M`.
   Preserve canonical alias resolution and the original hint in the warning.
3. Existing v2 notes with a missing task never receive an automatic replacement. Surface
   `open_ref_without_task` and restore-from-git/set-abandoned guidance. Duplicate live
   candidates refuse status/parent writes. Report open archive candidates without
   modifying them. Preserve the locator's other diagnostics; do not guess a missing
   residence or block ID.
4. Feed the selected mark into the status policy, including `[?]` as an open-lane
   overlay. In the v2 branch compare the task status with the stored base to tell an
   unchanged checkbox from a new task gesture. Marker/frontmatter-only changes can
   therefore drive a checkbox edit; incompatible changed inputs still conflict. Preserve
   the v1 algorithm. Task-driven marker updates still require
   `--write-pdf`/`--write-pdfs`; read-side pending status remains meaningful.
5. A closed archive task supplies terminal state. A deliberate marker/frontmatter reopen
   plans a fresh open task in the archive's source parent, falling back to the inbox and
   warning when that parent cannot accept work. The old terminal mark must not override
   an explicit reopen. Scan never plans an archive mutation.
6. For v2 only, remove parent from both input projections, stored base, and hashes.
   Normalize the old stored base/hash consistently before conflict detection so a
   residence move does not look like a two-sided edit. Preserve existing conflict
   refusal when there is insufficient base evidence rather than blessing changes. Render
   frontmatter parent from residence separately. Normal scans preserve the marker birth
   hint; PDF opt-in also refreshes a stale hint. A parent-only move must never trigger
   the missing-write-opt-in error.
7. Birth bodies show an embed instead of an in-note tracker. Heal a missing/stale embed
   to exactly one `![[<task-path-without-md>#^<id>]]`, one blank line below H1. Archive
   paths use `done/...`; embeds are views, not identity. Preserve authored material
   around the managed slot and do not invent a target when no task exists. `region.rs`
   excludes the managed embed from own notes, and companion audio anchors after it.
   Limit embed recognition to the managed anatomy; preserve unrelated authored embeds
   elsewhere.

Acceptance for this phase: unit tests for classification/adoption/diagnostics,
status/base conflicts and `[?]`, parent-free hashes and first v2 transition,
birth/closed birth/archived rendering, embed healing, audio placement, and frozen v1
byte stability. Keep new helpers small and publish their types in the phase note.

## Phase `v2-execution`: guarded writes and recovery

Primary files: `ref_tasks/insert.rs`, a focused cross-file task editor,
`highlights_ref/sync.rs`, `guard.rs`, and tests. Implement v2 plan execution without
activating it in the public scan entrypoints yet.

1. Reuse `insert_ref_task` for births. Allocate IDs against fresh destination bytes, its
   `done_tasks` archive, and earlier successful insertions. Keep the completed rendering
   rules: `#task #ref`, full ref-note link, sanitized title alias,
   `[created::YYYY-MM-DD]`, no `#hide`/`[fresh::]`, and closed-at-birth stamps. Audit
   its archive lookup using absolute destination paths, and propagate archive read
   errors other than not-found instead of ignoring possible collisions.
2. Add a compatible internal insertion variant for a preferred reopen ID. The current
   helper always derives a new ID from the linked ref stem and cannot preserve an
   arbitrary user-renamed address. Prefer the old ID only when free under the same
   destination/archive collision rules; otherwise allocate a unique ID. Keep the
   existing four-argument API for migration callers. Archived bytes remain untouched,
   including when an old ID must be suffixed in the live note.
3. For line edits, reread immediately before writing. Use the planned index if the exact
   original line still matches there; otherwise require one unique equal line elsewhere.
   Changed/deleted/ambiguous originals fail with
   `reading task changed during sync; rerun`. Change only the checkbox and, on close,
   the missing completion/cancellation stamp before the trailing ID. Preserve all other
   bytes, children, fields, IDs, and line endings.
4. Destination notes have no git-dirty veto. Stage writes through capture's
   preimage-checked writer and rebuild from fresh bytes on bounded preimage retries;
   rerun the exact-line check on a retry. Unrelated user additions survive. Do not retry
   arbitrary IO/permission errors or overwrite a changed task line.
5. Retain the ref-note/PDF/asset guards. The v2 ref-note guard additionally allows
   tracked modifications confined to the managed embed (including deletion, insertion,
   and repointing) plus permitted frontmatter edits. A residence-only frontmatter change
   is allowed even though it is excluded from sync contribution. Unrelated body edits
   still refuse; do not weaken the v1 guard globally.
6. Before each PDF starts writing, validate its required ref-note/PDF/task
   preconditions. Revalidate an adopted or otherwise selected task before using its
   address/status to write the v2 ref note, even when no checkbox edit is needed; a
   moved or altered task must not produce a stale healed embed or parent. Execute
   destination reading-task and routed insertion actions, then an explicitly authorized
   PDF marker write, then the ref note, using the actual final ID and refreshed
   metadata. Preserve asset guards and rehash a PDF after marker changes. Validate again
   at the existing write boundaries.
7. This is deliberately not a multi-file transaction or birth journal. If a later
   marker/ref write fails after the parent succeeded, a rerun must adopt that existing
   task and finish without duplication. Do not claim an all-or-nothing guarantee across
   a late IO failure. A detected changed reading line must fail before this PDF's
   destination/marker/ref writes begin.

Acceptance: direct planner/executor tests for dirty parents, moved line index,
deleted/changed/duplicate lines, CRLF, close-stamp preservation, archive immutability,
reopen into original/inbox parents, renamed IDs and collisions, forced parent-first
interruption then adoption, and ref/PDF guards. Use deterministic test seams rather than
timing-dependent races or production test-only environment switches.

## Phase `scan-integration`: shared context and annotation execution

Primary files: `sync.rs`, `scan_json.rs`, `annotation_tasks.rs`, `model.rs`, and
integration tests registered in `tests/cli/highlights/mod.rs`.

1. Activate the index-aware planner and executor for single-PDF sync, human scans, and
   JSON scans. Build one locator index after intake and before PDF planning; share it
   across scoped threads. Preserve the scan writer lock, hooks, layout and asset
   collision checks, deterministic PDF order, and partial-failure behavior. Dry-run
   performs no destination/PDF/ref writes or intake moves.
2. After parallel planning, reserve preview IDs in deterministic PDF order against each
   destination/archive and earlier previews. Actual execution always allocates against
   current disk contents. Human previews can name IDs; writes and embeds must report the
   actual result when a race causes allocation to differ.
3. Preserve annotation eligibility before and after the status signal so closing and
   reopening runs import the same pending work as today. For v2, an unqualified
   follow-up targets the live reading task's residence; without one, use `mac_inbox`,
   never an archive. Keep explicit `@name` behavior and v1 local-note defaults. Use
   full-path `[[ref/<type>/<stem>#^h-…|🔖]]` links for v2 follow-ups.
4. Replace stale routed-note snapshots with insertion intentions that reread and rebase
   at execution. For v2 default destinations use capture's Tasks insertion behavior,
   including existing notes without that heading and missing inboxes. Keep explicit
   legacy route behavior compatible. Mixed v1/v2 actions sharing a destination must not
   overwrite or invalidate one another's successful writes.
5. Keep processed-ID and legacy deduplication, including follow-ups moved to `done/`.
   Distinguish deterministic planning reservations from actually successful writes, so a
   failed PDF does not silently consume another PDF's follow-up work. Preserve correct
   owning-PDF failure/report attribution; do not attach later PDFs' tasks to the first
   destination group's owner just to reduce writes.
6. Render the final ref note with the result of its reading-task action and annotation
   execution. A repeated run after successful sync is a no-op. Continue after an
   individual bad PDF exactly as supported today. Keep the existing JSON version, field
   order, and field meanings; `summary.tasks` remains annotation tasks. Do not print
   human task messages onto JSON stdout.

Acceptance: subprocess fixture tests with fixed `BOB_NOW`, real temp-vault git guards,
and generated tiny PDFs. Cover two births to one parent, two categories with the same
stem, destination/archive collisions, alias and fallback parents, birth with/without
Tasks heading, orphan recovery, born-read/abandoned tasks, moved task residence, sidecar
follow-ups, archive reopen, multiple-open/missing diagnostics, mixed
births/edits/follow-ups in one destination, and a failed PDF followed by a valid one.
Compare `--jobs 1` and parallel results. Exercise human, single-PDF, JSON, intake, and
dry-run paths. Update new-birth fixtures to assert v2 placement while keeping explicit
v1 compatibility fixtures.

## Phase `verify-ref-sync`: reports, docs, and complete acceptance

Primary files: `report.rs`, `scan_json.rs` only as needed for contract parity,
`docs/highlights-ref-sync.md`, and tests for the integrated behavior.

1. Add the approved concise human lines:
   `📖 N reading tasks created · sase.md (1) · bob.md (1)` and
   `↻ N reading tasks updated`. When open v1 references remain, print
   `N open v1 ref tasks · run bob ref migrate-tasks`. Use actual successful actions;
   adoption is not creation, unchanged tasks are not updates, and failures do not count
   as completed writes. Verbose, dry-run, and single-PDF reports name destination, ID,
   adoption, and planned/performed line edits. Preserve concise no-op behavior and clean
   JSON stdout; no new JSON fields are required.
2. Update the sync guide's overview, generated-note shape, located-task status policy,
   parent projection, birth/adoption and reopen behavior, annotation Tasks placement,
   PDF write opt-in, and dirty-note rules. Label retained v1 examples explicitly. This
   is sync documentation only; broader epic docs and migration instructions remain the
   original closeout/migration phases' responsibility.
3. Audit all original `ref-sync-v2` acceptance cases against actual tests. Add missing
   behavioral coverage, including a custom configured ref directory, fixed birth dates,
   sidecar-free birth, parent move with unchanged PDF bytes and no opt-in error, parent
   refresh with opt-in, managed-embed deletion, unchanged v1 bytes, archive
   immutability, and follow-up dedup after moving a task into an archive. Test that a
   changed located line causes no marker/ref write and that a late ref failure is
   recoverable through adoption. Check reports alongside disk effects.
4. Adapt old tests that assume all newly born notes contain `#hide ^ref` or local
   follow-up tasks. Do not weaken error/guard/conflict assertions, drop no-op tests, or
   delete v1 compatibility coverage to make the suite pass.
5. Run the checks below and fix regressions caused by this work. Hand the completion
   epic's land agent an acceptance checklist, concrete API signatures, exact check
   outcomes, any baseline failure evidence, and any remaining unrelated follow-ups. This
   phase does not prematurely close bob-cli-5y.7 or any ancestor.

## Verification and final closure

Each phase runs meaningful focused tests for its changes. The final integrated gate is
`just check`, which runs `cargo fmt --check`,
`cargo clippy --all-targets --all-features`, and `cargo test --no-fail-fast`. Useful
focused commands include `cargo test --lib ref_tasks`,
`cargo test --lib highlights_ref`, `cargo test --test cli highlights`, and
`cargo test --test cli ref_library`. The integration harness is `tests/cli/main.rs`;
ensure each filter actually executes its intended tests. Use `/sase_monitor` for long
commands and include the exact continuation/remaining work; never end a provider turn
with an ordinary shell session still running.

Before the first phase changes implementation files, run
`cargo test --test cli doctor_reports_ref_tasks_and_parents_rows` on the unchanged
starting tree and record the result for the previously reported doctor failure. If it
still fails after implementation, compare the failure with that baseline; do not label
new failures pre-existing merely because their names match. Record independently
reproduced unrelated failures with reproduction details and an existing tracking
reference if available. A proven unchanged baseline failure does not require unrelated
product changes, but report the full gate honestly as failing rather than claiming it
passed. All failures caused by this implementation must be fixed.

The **completion epic's land agent** owns closure of the requested original bead:

1. Read the current original phase and all completion-phase notes. Verify all required
   behavior is implemented, tests/check evidence is current, and no acceptance work
   remains. Verify the shared insertion API is still compatible with original downstream
   phases bob-cli-5y.9 and bob-cli-5y.11.
2. Inspect `sase bead epic-symbols bob-cli-5y.7` and the completion epic's phase
   symbols; resolve this work's temporary integration markers. Do not mask unfinished
   scope by transferring its symbols to an unrelated phase.
3. Append a concise implementation and verification note to bob-cli-5y.7, including the
   final insertion/reopen interfaces, command outcomes, and any proved baseline failure.
   Do not repeat the two existing skipped-memory follow-ups.
4. Finish and close this completion epic through its normal land workflow after its
   phase beads are closed. If it is attached below bob-cli-5y.7, it must be closed first
   because bead closure never cascades.
5. Run
   `sase bead close bob-cli-5y.7 --note "<implemented behavior and verification actually established>"`,
   then audited-read the bead to confirm `closed` with resolution `done`. Never use
   `--force`, hand-edit status, close other original phases, or close the original epic
   bob-cli-5y. If another worker already closed the phase, verify its evidence and
   append only genuinely new results.

No manual git commit is requested. Follow `/sase_final` before a normal provider
response; skill handoffs follow their own lifecycle. The final user report must say
whether bob-cli-5y.7 was closed and summarize actual validation and any material
remaining limitation.
