---
tier: tale
title: Repair overwrite identity and narration, prove rollback, and finish landing
goal:
  Complete the remaining overwrite contract and safety work and close bob-cli-c9.4.1.9
  and its eligible ancestors.
size: medium
proposed_by: bbugyi200.athena.bob-cli-c9.4.1.9.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-c9.4.1.9
create_time: 2026-10-26 06:47:21
status: wip
---

- **PARENT:**
  [202610/overwrite_landing_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/overwrite_landing_fixes.md)
- **BEAD:**
  [bob-cli-c9.4.1.9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-c9/bob-cli-c9.4.1.9.md)

# Repair the remaining overwrite gaps and finish the nested landing

Finish the four remaining gaps in `bob-cli-c9.4.1.9`, then perform its landing and the
eligible ancestor closeout in this same coding turn. This is bounded direct
implementation for one agent. Do not create another epic for these changes.

The binding designs are `plan:202610/overwrite_landing_fixes.md`,
`plan:202610/unannotated_reference_overwrite.md`, and the overwrite contract in
`plan:202610/ref_create_force_recapture.md`. Read them through `sase artifact read`.
Read `sase bead read bob-cli-c9.4.1.9 -r "Need the landing findings and triage"` and its
three closed phases `.1`, `.2`, `.3`, including every note.

Decisions remain final: carry omitted aliases/topics/research
(`keep_user_fields = keep`), preserve raw `created` (`keep_created = yes`), and retain
legacy-only warn-and-fresh behavior (`legacy_force = no`). No memory changes, linked
client changes, live-vault operations, version bumps, or CHANGELOG edits are needed.
Annotated overwrite remains owned by `bob-cli-c9.5`.

## Evidence and completed landing work

The land review inspected source and commits `3e9da7cc`, `43f8aff5`, `66f9d67f`. At
`66f9d67f`, fetched `origin/master` has no additional commits. The only outside commit
since this epic began is `17f3881c`, which changes project `root: true` grouping in
`pomodoro_homes` and stats. It needs no overwrite integration. Recheck later drift
before landing.

All 61 existing CLI tests matching `overwrite_` passed. A temporary change to
`overwrite_title_change_keeps_old_version_suffix`, using `report.pdf` instead of
`other.pdf` as the replacement source, failed with `nothing to overwrite` for
`-N report__v2`. The land agent restored the file after the diagnostic; the coder must
add a permanent regression. The existing test comment documents its workaround.

A second temporary CLI probe made the collection fake listener write its episode, log
the audio path, and delete the selected root note via a test-specific
`FAKE_LISTEN_REMOVE` path. Listen succeeded; the subsequent recheck exited 2
(`-N tokio_internals is not a reading ...`), emitted no `kept:` line, and the logged
audio file was gone. The probe asserted its existence and failed. Test source was
restored afterward. Use an explicitly passed test path: the listener's sanitized
environment does not retain `BOB_DIR`.

Step-3 follow-up triage is complete and recorded on the epic:

- Phase `bob-cli-c9.4.1.9.2` note #1 is the PDF identity bug below. It remains epic
  work, so no separate task was created.
- Phase `bob-cli-c9.4.1.9.2` note #2 and phase `bob-cli-c9.4.1.9.3` note #1 both concern
  the existing unrelated lib timing flake `bob-cli-ca`. `/sase_new_task` search,
  recent-task and active-epic sweeps were completed; one +1 records both proposing beads
  and the draft-contract test's 855 ms versus 500 ms observation. No duplicate task was
  created. `bob-cli-bo` stays closed: it fixed the separate CLI performance gate. Carry
  this disposition into the final close note.
- No other child `PROPOSED FOLLOW-UP:` notes were present.

`sase bead epic-symbols bob-cli-c9.4.1.9` was empty. The current justfile has no
`symvision` recipe; check again at closeout.

## 1. Keep local PDF identity independent of the selected output name

In `src/native/highlights_ref/pdf_target.rs::plan_local_pdf`, a missing or rejected Info
title falls back to `humanize_stem(&stem)`, where `stem` already contains `-N`.
`create.rs` calls this function for recapture identity with `-T` deliberately removed
but still passes `-N`. For original `report.pdf` with absent Info Title or Info Title
`Report` (rejected as filename-equal), `-N report__v2` therefore looks for title
`report v2`, misses the existing family, and refuses a valid overwrite.

Derive source identity from the original file and its metadata, independently of the
requested output stem and replacement `-T`. Keep naming and family matching separate; do
not select an unrelated family just because its stem matches. Audit the other callers of
`plan_local_pdf` and existing naming tests so fresh capture, exact-output, explicit
title and reread behavior remain intentional.

Extend `tests/cli/highlights/overwrite_local.rs` with absent-Info and
filename-equal-Info cases. File `report` and `report__v2`, then use the original
same-stem source to select v2 with `-N`, with and without replacement `-T`. Check real
and dry-run selection, unchanged root note/PDF, preserved task address and version
suffix, and no accidental extra reading. Retain the free-`-N` refusal and
unrelated-family controls. Remove the obsolete workaround comment once resolved.

## 2. Keep completed narration on every subsequent error

`collection.rs::capture_and_install_overwrite` calls `stage_overwrite_audio`, then uses
`?` for `recheck_collection_overwrite` and `marker_from_pdf` before entering its
`post_listen_error` handler. A selection change during narration drops the scratch
directory and its finished episode. `create.rs::finish_local_overwrite` and
`clip.rs::capture_article_overwrite` similarly extract the final marker after narration
but outside the preservation handler.

Once narration successfully produces audio, put all subsequent fallible rechecks, marker
extraction, staging validation and execution under the existing preservation path, using
a small shared helper or a scoped result if useful. Preserve the useful failure
diagnostic and print `kept: <path>` with a usable rebind hint. Do not retain scratch on
success or change failed-listen semantics. Keep explicit `-a` separate from newly
generated narration.

Add a deterministic CLI regression using the existing fake listener in
`tests/cli/highlights/create.rs::highlights_create_collection_overwrites_exact_set_with_listen`:
after writing the episode, make the selected collection unavailable or change its source
set so the post-listen recheck fails. Record the produced audio path in the fake log;
assert a nonzero exit, `kept:` names that still-existing episode, its bytes are intact,
and no overwrite commit/publication occurred. Snapshot the state after the simulated
concurrent edit so the test does not demand undoing another writer. Also exercise a
post-listen marker-read failure through a local or article route using the existing fake
process seams; keep the current executor-failure test. Any injected fault hook added to
Rust must be test-only, not a release environment variable or CLI option.

## 3. Remove the unused selector implementation and test the live path

The selection phase explicitly required wiring or deleting obsolete selectors.
`overwrite.rs::select_url_member` and `select_stem_member` remain unused under new
`#[allow(dead_code)]` annotations. They retain `recapture.rs::select_overwrite_for_url`
and `select_overwrite_for_stem` as a parallel selection implementation, while routes use
`preflight_for_url_outcome`, `preflight_for_stem` and `finish_selection`.

Prefer deleting the unused wrappers and selectors, moving their meaningful tests onto
the live preflight APIs. The executor fixture also currently selects through
`select_overwrite_for_stem`; migrate it. Keep live-path coverage for default newest,
explicit older member, ambiguity/nonmember names, partial collection overlaps and
overwrite at the `__v999` reread limit. Add no dead-code suppressions to preserve the
obsolete implementation.

## 4. Make executor failpoints prove rollback of actual writes

`overwrite/execute.rs::filed_execute_fixture` creates only a generated note, PDF and
unchanged ready task. It has no authored prose, sidecar or replacement audio, and the
staged title/status are unchanged. Thus `AfterFrozenNote`, `AfterSidecarRemoval`,
`AfterAudio` and `AfterTaskEdit` currently trip after no-op steps. The checkpoint
plumbing is useful; extend its fixture and assertions.

Exercise a real prose archive, an unannotated tracked `.textbundle` or sidecar, an audio
extension swap with an old companion and a newly created destination, and a title/status
edit of a reading task with preserved children. Rebuild preflight after fixture setup.
Assert that each relevant mutation really occurred before the injected failure, then
compare all affected files, creations, task content, index, HEAD and intent against the
old state after rollback. Preserve unrelated staged or dirty files. For crash recovery,
leave real applied files and recover them. For commit-reported-as-error and post-sync
failure, assert the complete expected committed new state and correct result/reporting,
not only changed note/PDF bytes. Use existing failpoint infrastructure and fixture
helpers; do not invent a new test framework. Fix any transaction defects these stronger
cases reveal as epic work.

## 5. Verify the combined repair

Run focused behavior regressions and the existing overwrite, recapture and rename tests
affected by the shared paths. Run `just check` as the file-change gate. Do not run
`just check-full`. If docs/help need adjustment, first read `docs.md` through
`/sase_memory_read`; the binding epic also requires `just docs-gen` and
`just docs-check`, so run them before closeout and include generated changes.

Use `/sase_monitor` before long-running checks and provide the successor enough context
to finish this plan and its closeout. If only the known `bob-cli-ca` timing tests fail
under parallel load, confirm the unchanged-tree isolated result and record it against
that task per the existing policy; do not expand this epic into performance-test
infrastructure work. A `just check` pass with a `just check-full` failure is an
infrastructure issue, not outstanding epic scope. Never claim a failed gate passed.

## 6. Finish this epic and the eligible ancestors in the same turn

There is no separate land agent for this tale. The coder must complete this step; do not
wait for this tale's own commit SHA, push, or CI. The host commits after the turn ends,
and closing with final verified code still uncommitted is normal.

1. Re-read `bob-cli-c9.4.1.9` and every descendant, all notes, its linked plan, final
   decisions and post-review drift. Verify all four gaps above are fixed, all named
   phases remain completed, and every proposal has the disposition above. Run
   `sase bead epic-symbols bob-cli-c9.4.1.9`. Resolve every entry by wiring,
   privatizing, adding a justified non-test pragma, or deleting the symbol. Re-key only
   if a still-open later bead actually needs the exemption. Do not defer it.
2. Close normally with
   `sase bead close bob-cli-c9.4.1.9 --note "<verified source/commits and decisions, identity/narration repairs, live selector coverage, actual-write rollback proof, integration/drift review, gate results and all follow-up dispositions>"`.
   Run `just symvision`; if the recipe is still absent, record that it is unavailable.
   Use `/sase_repo` to open the plans sidecar before editing, read plans through
   `sase artifact read`, and set `status: done` in
   `plan:202610/overwrite_landing_fixes.md` at the PLAN path reported by the bead.
3. Read `bob-cli-c9.4.1.9` again with reason `Need the parent link`. Its parent plan is
   `bob-cli-c9.4.1`. Review that parent's three prior landing notes, all descendants
   (`bob-cli-c9.4.1.1` through `.8` and `.9`, including deeper children) and every note,
   the linked `plan:202610/unannotated_reference_overwrite.md`, and drift since its
   prior `94abe3d9` review. Rerun descendant and linked-plan readiness checks; do not
   infer readiness solely from closed statuses. If complete, run
   `sase bead epic-symbols bob-cli-c9.4.1`, retire/re-key every entry as above, then
   `sase bead close bob-cli-c9.4.1 --note "<rechecked full scope, descendants, repaired landing findings, decisions, drift and verification>"`.
   Confirm with `just symvision` (record absence if unavailable), and set `status: done`
   in its linked `plan:202610/unannotated_reference_overwrite.md`.
4. That plan's parent is phase `bob-cli-c9.4`. Read it, confirm its unannotated
   overwrite scope is satisfied, resolve `sase bead epic-symbols bob-cli-c9.4`, and
   close only that phase with
   `sase bead close bob-cli-c9.4 --note "<verified unannotated overwrite implementation and completed child plans, integration and checks>"`.
   Stop at the phase boundary: leave containing epic `bob-cli-c9` and its
   `ref_create_force_recapture.md` plan to the waiting outer land agent; annotated
   overwrite is still separate work.

If any close reports leftover symbols, clean them and retry. If descendants or named
phases are incomplete, finish/reopen them; never force a successful nested landing.
Cancellation/supersession requires a genuine deliberate outcome with
`--force --reason ... --resolution canceled|superseded`, never a way around readiness.
Stop at the first incomplete or ambiguous ancestor, append a blocker note there and
report it. Submit `/sase_final` only after all work and commands are finished; declare
all changed repositories, including the plans sidecar, for host commit.
