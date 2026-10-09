---
tier: tale
title: Repair v2 reference sync and finish bob-cli-62 landing
goal:
  Restore the required sync and guarded-write behavior, verify integration, and close
  bob-cli-62 and bob-cli-5y.7.
size: medium
proposed_by: bbugyi200.athena.bob-cli-62.land
bead: bob-cli-62
create_time: 2026-10-09 18:43:34
status: wip
---

- **PARENT:**
  [202610/finish_ref_sync_parent_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/finish_ref_sync_parent_tasks.md)
- **BEAD:**
  [bob-cli-62](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-62/README.md)

# Repair v2 sync acceptance and finish landing bob-cli-62

Finish the remaining work on **bob-cli-62**, then close that epic and the original phase
**bob-cli-5y.7** in this coding turn. This tale has no land agent; its coder owns the
complete closeout below. Do not postpone closing until this turn's commit, push, SHA, or
CI exists. The host commits after the turn ends.

This is substantial but bounded direct implementation: repair existing planner, guard,
destination writer, and anatomy seams in one Rust repository, with focused behavioral
tests. The shared locator, resolver, renderer, insertion API, and scan context already
exist; no new architecture, linked-product implementation, or multi-agent phase graph is
needed.

## Authority, audited context, and starting evidence

Read `sase bead read bob-cli-62 -r "Need landing audit and follow-up outcomes"`, all
four closed child beads, and `bob-cli-5y.7` before changing code. Read these artifacts
with `sase artifact read`:

- `plan:202610/finish_ref_sync_parent_tasks.md`: governing completion scope and
  acceptance matrix, especially v2-execution and verify-ref-sync.
- `plan:202610/ref_tasks_live_with_parent.md`: original design calls 1–4 and ref-sync-v2
  phase contract.
- `plan:202610/ref_sync_parent_tasks.md`: previous insertion and execution contracts.

The lander's reviewed head was `61e5c47`; all four child phases are closed. Implemented
commits: `8c938cf` (planner), `4bfacc3` (executor), `b1512d6` (scan integration and
doctor fixture repair), `61e5c47` (reports/docs/tests). Existing births, shared
indexing, residence projection, collision allocation, archive reopens, and ordinary v1
fixtures should be preserved. The source review found the specific remaining gaps below.
Reproduce them with regression fixtures before fixing; do not treat the earlier green
suite as evidence they are covered.

The lander did not rerun Cargo or `just check`; phase 62.4's 3596 passing tests are
prior evidence. A fresh current-tree gate is required after repairs. A small rustc
harness using the checked-in editor functions confirmed that a quoted reading task
cannot be edited and that mixed line endings are normalized outside the edited line.
Only the nonterminal close-stamp function was stubbed in that harness; it did not
exercise full sync.

Final original decisions remain `cancel_dropped_wrapper_refs = no`,
`memory_ref_parent_decision = no`, and `memory_glossary_ref_terms = no`. Edit no memory,
cancel no wrappers, install nothing, and use only temporary test vaults. Do not change
plugins, the Mac frontend, CLI switches/schema versions, migration, or the live vault.
Keep the original four-argument insertion API compatible:

```rust
insert_ref_task(bob_dir: &Path, destination: &Path,
                line: &str, children: &[String])
    -> Result<InsertedRefTask, String>
```

`InsertedRefTask` contains `block_id`, `placement`, and `task_line`.
`insert_ref_task_with_preferred_id(..., prefer_block_id: Option<&str>)` is the additive
reopen variant. Phases 5y.9 and 5y.11 rely on these contracts.

## 1. Restore lifecycle refusal, annotation intake, and diagnostics

Primary files: `src/native/highlights_ref/sync.rs`, `reading_plan.rs`, and `report.rs`,
with existing CLI and unit fixture helpers.

- `plan_pdf_sync_v2` currently sets `marker_write_needed = false` whenever `write_pdf`
  is absent. Unlike the v1 path, it never refuses a task/frontmatter contribution that
  needs marker write-back. Repro: birth a Ready ref, change its external task to Next,
  then sync twice without `--write-pdf`. The first run can write Next to
  frontmatter/base while leaving the marker Ready; the next run reads the stale marker
  as a fresh change and reverts the checkbox. Restore the existing opt-in contract for
  every non-parent synced-field contribution, including lifecycle status and
  normalization. Writing runs must refuse before any write when required marker changes
  lack opt-in; dry runs preview them. Calculate the semantic parent-free update
  separately from the optional residence-hint refresh, so parent-only moves still
  succeed without opt-in. With opt-in, update marker, note/base, and task consistently
  and prove the rerun settles. Cover task-driven close/reopen and frontmatter-driven
  edits as well as a non-status synced field.
- Capture annotation eligibility from the merged projection **before** applying the
  reading-task signal, then combine it with the post-signal status. Today both checks
  occur after the signal. Repro: sync a WIP reference, add a new sidecar `#task` bullet,
  set its reading task to `[x]` or `[-]`, then sync with PDF opt-in. The closing run
  must import the pending follow-up once. A reopen to WIP must import in that run too.
- Consume `ReadingTaskPlan.refuse_status_parent_writes`; it is currently never read by
  production code. Several live candidates or an unselectable archive-open state must
  not write task, marker, or frontmatter status/parent or create an invented birth
  embed. A clear per-PDF diagnostic failure is an acceptable conservative
  implementation. Include `multiple_open_ref_tasks` and `open_ref_task_in_done` in the
  error and preserve partial-scan continuation. Strengthen the existing ambiguity test:
  change marker status and request PDF writes, then assert affected task/ref/PDF bytes
  stay unchanged. Exercise ambiguous births and archive-open candidates too.
- Surface stored reading diagnostics in useful human/single-PDF/verbose output,
  including `open_ref_without_task` with restore-from-git/set-abandoned guidance.
  Missing tasks still gain no replacement. Keep JSON stdout as one existing-schema
  envelope; refusal details belong in its established failure message.

## 2. Make destination writes safe and retain successful dedup only

Primary files: `highlights_ref/sync.rs`, `guard.rs`, and `ref_tasks/edit.rs`.

- `guard.rs::ensure_safe_to_write` currently adds every routed intent destination to the
  Git dirty-target veto. V2 default follow-ups share reading-task destinations and must
  accept their unrelated authored changes. Exempt destinations used only by v2 default
  insertions from that veto, preserving legacy explicit-route/v1 guard behavior and all
  ref-note/PDF/asset guards. A combined default/explicit route must retain the intended
  legacy policy. Test a tracked dirty parent with unrelated task additions plus a
  reading action and default annotation insertion.
- `execute_routed_intents` rereads once but uses unguarded `atomic_write`. Route writes
  through capture's `StagedTextFile`/`write_staged_files`; use bounded retries only for
  preimage mismatch, rebuilding insertions and dedup against fresh bytes. Preserve
  authored additions, Tasks placement, missing inbox creation, CRLF, and legacy explicit
  append placement. Do not retry arbitrary IO errors.
- Commit dedup keys to `RunWriteState` only after their corresponding destination write
  succeeds, or fresh disk evidence proves the task already exists. Current code inserts
  processed ID, legacy identity, and source anchor before writing. A failed destination
  must not consume a later PDF's otherwise-identical follow-up. Successful destination
  writes remain deduped even if a later ref-note write fails. Test an **execution**
  failure followed by a valid PDF with the same legacy identity; the existing bad-PDF
  test fails planning and does not cover this.
- Before `execute_pdf_sync_v2` starts any writes, non-mutatingly validate the selected
  reading line, required ref-note preimage, and authorized PDF preimage, plus existing
  asset preconditions as applicable. Currently the reading action writes before the
  ref-note check, and the PDF check follows routed writes. Retain immediate revalidation
  at each write boundary. Tests must mutate the ref/PDF/selected line after real
  planning, then execute the real per-PDF function and assert no destination/marker/ref
  write for detected preconditions. Changed, deleted, and ambiguous selected originals
  preserve the exact reread error.
- Fix quoted external reading tasks: the locator accepts `>` prefixes, but
  `flip_checkbox_mark` does not. Reuse the locator's parsing convention while computing
  the mark's offset in the original line. Preserve the entire quote, indent, list
  marker, child block, fields, trailing ID, and existing stamps. `splice_line` must
  replace only the original line's byte span rather than joining every line with one
  chosen ending; preserve mixed LF/CRLF, final-newline state, and all unrelated bytes.
  Cover nested quotes, ordinary CRLF, and mixed endings.

Use deterministic internal test seams for write failures/preimage races; never introduce
a production test-only environment switch or timing-dependent race. Keep the deliberate
parent-first recovery model without a multi-file transaction. Add a real late-ref-write
failure after a successful parent insertion, followed by replanning/rerunning into
adoption of the same ID without duplicate tasks.

## 3. Finish reopen routing and managed anatomy integration

- In `plan_pdf_sync_v2`, the projected/destination residence must prefer the new
  `Insert` route for a reopen, rather than the archived task's old residence. Repro:
  archive a terminal reading task, make its source project terminal, then reopen the
  marker/frontmatter to WIP with an unqualified annotation follow-up. Reading task,
  follow-up, frontmatter parent, and embed must all use `mac_inbox`; the archive and
  terminal source remain unchanged. Use the live located task's residence for ordinary
  existing refs, and inbox when there is no live residence.
- Wire `maybe_insert_audio_embed_after_managed` into existing-v2 late companion
  discovery. It is currently used only in tests; the fallback calls the old
  Highlights-heading anchor. A late companion must appear after the managed task embed
  even when authored text lies before `## Highlights`, once, preserving that text and v1
  placement. Remove transitional dead-code suppressions on wired seams and update
  comments that still say public wiring has not happened.
- Audit `ref_tasks/embed.rs::find_managed_embed` and its healing/dirty-guard callers
  against the designated slot one blank below H1. Today healing repeatedly removes every
  block embed before the first H2. Restrict managed handling to that anatomy (including
  duplicate managed lines within it), preserving an unrelated authored block embed in a
  later introductory paragraph. Such an authored embed change must not pass the
  managed-only dirty allowance. Retain no-H1 behavior, missing and stale healing,
  duplicate collapse, fenced-block exclusion, archive addresses, audio anchoring, and
  read-side identification of genuine managed embeds.
- Preserve managed-region validation on the v2 healed path. Its `Err(_) => healed`
  branch must not silently turn an invalid Highlights region into a successful sync.
  Establish whether earlier validation already covers each case; use the established
  missing/broken-region failure where it does not.

## 4. Complete acceptance and post-start integration

Add the explicitly required archive-move dedup test proposed by **62.4 note #1**: create
a v2 sidecar follow-up, move that follow-up into `done/` carrying its `[h::]` property,
rescan, and assert no recreation, unchanged archive, accurate annotation counts, and a
subsequent no-op. Preserve legacy processed-ID/source anchor dedup. Confirm archived
terminal reading sync leaves archive bytes intact.

Keep existing real temp-vault Git guards, generated tiny PDFs, fixed `BOB_NOW`,
human/single-PDF/JSON paths, no-write dry runs, jobs-1/parallel equivalence, actual-ID
reporting, custom ref dir, parent aliases, two-category collisions, born-read/abandoned
stamps, move/no-opt-in, and frozen v1 byte tests. Add missing strong assertions; do not
weaken error expectations or substitute tests of isolated helpers for tests of actual
orchestration.

Only two non-62 commits landed since the first epic commit in the reviewed history:
`9b44dc6` (successor close/recovery repairs) and `6562b71` (resolved Pomodoro agenda).
Both use ordinary task parsing/marks/links with arbitrary IDs; no `#ref` special case is
appropriate. Recheck new drift since `61e5c47` before closing. Verify a
temporary-fixture reading task with a user-renamed ID through the new agenda
(`capture-pomodoros --tasks`) and a capture/close status gesture, proving its title,
mark, address and any dependency behavior remain ordinary and survive sync with the
correct opt-in policy. Reuse their existing tests and settings helpers.

Update `docs/highlights-ref-sync.md` where necessary to explain the restored v2
opt-in/refusal and safe destination behavior. Do not expand into other epic phases.

Focused checks while implementing:

```sh
cargo test --lib ref_tasks
cargo test --lib highlights_ref
cargo test --test cli highlights
cargo test --test cli doctor_reports_ref_tasks_and_parents_rows
```

Run **`just check`** after final changes. Do not run `just check-full`. Use the SASE
tool/monitor workflow for long execution, including a concrete continuation that still
performs section 5 after successful verification. Do not bind a prepared completion that
ends the agent before bead/plan closeout. Fix epic-caused failures; record and triage
only independently proven unrelated issues under `/sase_new_task`.

## 5. Finish the landing in the same coding turn

Follow-up triage is already complete in bob-cli-62's landing notes; include every
outcome in the final close note. No separate tasks were created:

- **62.1 #2, 62.2 #2, 62.3 #1:** one doctor fixture defect, declined as already repaired
  by `b1512d6` adding fixture `lib/` (62.3 #3), with prior 62.4 green gate. Repeat its
  focused test to supply fresh evidence. The original report is already outer epic 5y
  note #1; do not file a duplicate unless a new reproduction remains.
- **62.4 #1:** archive-move dedup coverage, declined as a separate task because
  completion-plan verify-ref-sync item 3 requires it. This tale implements it.
- Original **5y.7 #1/#2** skipped-memory notes remain for outer epic 5y's landing; this
  plan forbids duplicating/implementing them.

After all repairs and the current-tree `just check` pass:

1. Re-read bob-cli-62, all 62.1-.4 notes and statuses, and bob-cli-5y.7 for newly
   arrived evidence. Ensure this tale's acceptance work is done. If SASE created a
   separately assigned repair bead that is a descendant, normally close that completed
   bead first so descendant readiness is satisfied.
2. Run `sase bead epic-symbols bob-cli-62`. Resolve every entry keyed to 62 or its
   phases: wire, privatize, delete, or apply the permitted non-test pragma. Re-key only
   when a concrete still-open later bead still needs the exemption. Check
   `sase bead epic-symbols bob-cli-5y.7` and resolve its entries too. Both were empty
   during the land audit; recheck now. Never leave cleanup for another agent.
3. Run
   `sase bead close bob-cli-62 --note "<source/commit review, integration, repaired acceptance, actual check outcomes, all follow-up outcomes>"`.
   Do not force a successful landing. If rejected, fix the named symbol/descendant issue
   and retry normally; never force merely to make the command succeed.
4. Run `just symvision` if the recipe exists, and confirm the whitelist is clean. The
   reviewed justfile had no symvision recipe; explicitly record unavailability if it
   remains absent.
5. Audited-read `plan:202610/finish_ref_sync_parent_tasks.md`, open the `plans` sidecar
   through `/sase_repo`, resolve the PLAN path via `sase artifact path`, and set only
   its frontmatter `status: done`, preserving the rest. Use the dynamically resolved
   path; never hard-code a numbered workspace directory.
6. Run `sase bead read bob-cli-62 -r "Need the parent link"`. Reviewed JSON had
   `parent_id: null`; however its approved scope explicitly requires closing original
   **bob-cli-5y.7**, even without a structural parent link. Verify its ref-sync-v2
   obligations against the completed code/tests and append concrete insertion/reopen API
   and verification evidence. Then run
   `sase bead close bob-cli-5y.7 --note "<implemented behavior and actual verification>"`
   and audited-read it to confirm `closed`, resolution `done`. Clean its symbols before
   this close and rerun available `just symvision` afterward. Leave **bob-cli-5y** and
   `ref_tasks_live_with_parent.md` open for its waiting lander. Do not mark the outer
   epic plan done.
7. If a newly observed structural parent of 62 is a phase, close only that phase after
   verifying its scope (if it is 5y.7, the preceding close handles it). If it is instead
   a plan, review its previous landing note, every descendant and note, linked plan, and
   post-child drift; rerun descendant/linked-plan readiness, retire symbols, normally
   close and run available symvision, mark that plan done, and repeat only through fully
   complete directly parented plan ancestors. Stop at incomplete/ambiguous scope and
   record its blocker. Never force or close the containing outer epic merely because
   this child finished.
8. Use `/sase_final` as the final action before a normal final response, declaring
   commits for every edited repository (including plans). No manual commit is requested.
   Report actual verification, whether 62 and 5y.7 closed, and any material limitation.
   Do not wait for this turn's commit or CI to close them.
