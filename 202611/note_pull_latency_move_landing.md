---
tier: tale
title: Land note-pull capture after the latency and move-integration fixes
goal:
  Bring note-pull capture under its 70 ms p95 budget, keep note-pull rows correct when a
  later item in the same draft moves a pulled task, then close bob-cli-gj.2.1 and its
  parent phase bob-cli-gj.2.
size: medium
proposed_by: bbugyi200.athena.bob-cli-gj.2.1.land
proposed_with: claude/opus
bead: bob-cli-gj.2.1
status: done
---

- **BEAD:**
  [bob-cli-gj.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gj/bob-cli-gj.2.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-gj.2.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-gj.2.1.land.md)
- **COMMITS:**
  - [cbc35ad](https://github.com/bobs-org/bob-cli/commit/cbc35adf64a843f25a8adc9855e823f696b088ed)
    — feat(capture): optimize note-pull landing with move-aware ownership

# Land note-pull capture after the latency and move-integration fixes

## Context

This tale finishes epic **bob-cli-gj.2.1** (note-pull capture in bob-cli) and lands it
in the same turn. Its parent **bob-cli-gj.2** is a _phase_ of epic **bob-cli-gj**. No
later land agent will act on this tale, so its final step closes both beads.

Start by reading the state:

- `sase bead read bob-cli-gj.2.1 -r "Need landing audits and triage"`. Note #7 holds the
  second landing audit this plan is built on.
- `sase bead read <id> -r "Recheck phase scope and notes"` for `bob-cli-gj.2.1.1`,
  `bob-cli-gj.2.1.2`, `bob-cli-gj.2.1.3` and `bob-cli-gj.2.1.4`.
- `sase artifact read plan:202611/note_pull_cli.md "Need the CLI acceptance contract"`.
  Section 7 has the latency gate.
- `sase artifact read plan:202611/note_pull.md "Need parent decisions and budget"`.
  Section 3.11 has the 70 ms budget.
- `sase artifact read plan:202611/note_pull_landing.md "Need prior landing constraints"`.
- `sase bead read bob-cli-gj.2 -r "Need parent phase scope and decisions"`.

**Verified at master `b9fe9d0c`; do not redo:**

- `cargo check --all-targets --locked` passes.
- `cargo test --lib note_pull` passes 38/38 and `cargo test --test cli note_pull` passes
  8/8.
- Every `CaptureItemResult` literal in `src/native/capture/plan.rs` has exactly one
  leading `note_pull: None`. Never move it to the trailing fields.
- The capture grammar revision is now **10**. Transfer parsing in `706eab67` advanced
  it. A lone `@route` still parses as `note_pull`. Do not change the revision.
- Target runs (`@a+x @b+y`) before or after a pull keep capture order and offer
  ownership.
- `sase bead epic-symbols bob-cli-gj.2.1` lists nothing.

Repair commit `4db3200c` already fixed fold-proof gating, selected repeated-pull
ownership, blocked-project counts, cross-group dedup and the pull-only hint. Do not redo
those. Two items remain.

## Fixed constraints

- Keep the accepted decisions: `lanes = open_lanes`, `empty_pull = refuse`,
  `memory_decision_record = no`, `memory_successor_offer = no`.
- Make no memory edits.
- Make no automatic picks, and add no new flag or command.
- Do not edit `Cargo.toml` versions or `CHANGELOG.md`. Use a conventional commit
  subject, `fix(capture): …` or `perf(capture): …`. These changes break no contract, so
  add no `!`.
- Capture JSON stays additive and unversioned.
- Never capture into the live vault. Use synthetic fixtures, or the isolated copy that
  `tests/fixtures/note_pull/benchmark.py` makes from the path
  `sase repo open gh:bobs-org/bob -r "<reason>"` prints.
- For the latency budget, do not add a cross-process cache or skip any dependency or
  safety proof. Do not weaken the 70 ms target, and do not substitute timings from a toy
  vault.
- Do not change behavior for `!` successor, keep-moving or relation groups. If you
  reproduce the move problem from step 2 in those groups, route it through
  `/sase_new_task` as described in step 5.
- Do not run `just check-full`. The file-change gate is `just check`.

## 1. Meet the ≤ 70 ms p95 budget

The baseline was measured at `b9fe9d0c` on athena, with a release build and an isolated
6,907-note copy under `$CARGO_TARGET_DIR` (ext4). hyperfine, 30 runs:

- `@sase`: 78.4 ± 3.3 ms, maximum 87.1 ms.
- `@mac_inbox`: 38.9 ms.
- `hello`: 17.7 ms.

The previously recorded p95 was 87.67 ms. Temporary `Instant` marks (since reverted)
gave this steady-state breakdown for `@sase`, about 75 ms in process:

| Stage                                                              |  Cost |
| ------------------------------------------------------------------ | ----: |
| `NoteCatalog::walk` → `capture_dependency_tasks::vault_note_paths` | 19 ms |
| `NoteCatalog::walk` → `NoteIndex::from_paths`                      | 10 ms |
| `NoteCatalog::walk` → basename diagnostics                         |  2 ms |
| Shared byte pass (`SharedNoteBytes::build`)                        | 11 ms |
| First `placement_for_task` → `Catalog::load` (theme catalog)       | 13 ms |
| Net post-pass and pull-in assembly                                 |  4 ms |
| `append_unconfigured_theme_warnings` → second `Catalog::load`      | 11 ms |

Apply these fixes, keeping every observable output identical:

1. **`collect_note_paths`** in `src/native/capture_dependency_tasks.rs`. Today
   `entries.sort_by_key(|entry| entry.path())` allocates two `PathBuf`s per comparison.
   Replace it with `entries.sort_by_cached_key(|entry| entry.file_name())`. Siblings
   share a parent, so path order equals file-name order. The depth-first walk therefore
   still emits component-wise sorted paths, and the final `relative_paths.sort()` stays
   a cheap check. A reverted experiment measured 19 → 7 ms. Add a unit test that pins
   the walk order for an interleaved `a/`, `a.md`, `a-b.md`, `A.md` tree against a plain
   sorted `Vec<PathBuf>`.
2. **`NoteIndex::from_paths`** in `src/native/vault_links.rs`. Collect the paths into a
   `Vec` and bulk-build `relative_paths` with `collect::<BTreeSet<_>>()` instead of one
   `insert` per path. Keep the first-seen and ambiguous basename semantics exactly. The
   experiment measured 10 → 2.7 ms. The existing `vault_links` tests, including
   `with_additional_unions_staged_notes_without_rebuilding`, must still pass.
3. **`append_unconfigured_theme_warnings`** in `src/native/capture/budget.rs`. It calls
   `Catalog::load` on every capture, which costs about 11 ms even for `hello`, and for a
   note pull it repeats the batch's own theme catalog. Proceed as follows:
   - Read the config warning and `themes` from the cheap `config::load_theme_sources()`
     first. Those are the only `Catalog` fields used before the daily-note loop.
   - Return early when no themes are configured, or when no staged text file is a daily
     note whose Pomodoros component counts grew.
   - Load the full catalog only in the remaining case.
   - Preserve the dedup of the config warning and the structured `theme_not_configured`
     output.
   - Today a theme-journal read failure inside `Catalog::load` returns before the config
     warning is pushed. Keep that ordering if any test pins it.

   Add a regression that a capture with no grown daily-note component does not load the
   catalog, using the existing per-context instrumentation pattern or an equivalent test
   hook. Also check the existing tests that cover a theme warning on a grown component.

4. With the per-batch catalog and snapshot work already reused, look for any remaining
   duplicate theme-catalog loads on the note-pull path, such as `append_theme_refs` when
   picks name entries. Remove one only when the batch already holds an equivalent
   catalog.

Expected result: about 56 ms in process after fixes 1–2, and about 45 ms after fix 3.
Then rebuild with `cargo build --release --locked --bin bob` and run
`python3 tests/fixtures/note_pull/benchmark.py --vault-source "$SOURCE"`, where
`$SOURCE` is the path `sase repo open gh:bobs-org/bob` prints. The run refreshes
`tests/fixtures/note_pull/performance.md`. Acceptance is p95 ≤ 70 ms for both `@sase`
(the largest root area/project note) and `@mac_inbox`. If either still misses, profile
again with temporary marks and keep going. Never record a miss as a pass.

## 2. Keep note-pull rows when a later item moves a pulled task

Since the note-pull code landed, project-note move bullets (epic bob-cli-gl: `6650faa3`,
`14a57732`, `b738d8f2`, `7cf07746`) and transfer moves (`b9fe9d0c`, `@a+x > @b`) have
landed. A later item in the same draft can now relocate a task that an earlier `@route`
pull offered or picked.

To reproduce, make a synthetic vault with area `work.md` holding
`- [/] #task Alpha ^alpha`, `- [*] #task Beta ^beta` and `- [ ] #task Gamma ^gamma`. Use
a day note with an open `FIX` Pomodoro, a missing `BOB_CONFIG_FILE`, and fixed
`BOB_NOW`. Then run:

```
bob capture --dry-run -p all -b <vault> -- $'@work\n\nDocs push @work^docs-push@\n- ^gamma'
bob capture --dry-run -p all -b <vault> -- $'@work\n\n@work+gamma > @bob'
```

Today Gamma is picked: it goes Ready → Next, gets a fresh stamp, and its link follows to
`[[work_docs_push#^gamma]]`. The report is still wrong:

- It says `Pull from work · 2 of 2 pulled in`, and `pull_in.offer_count` and
  `picked_count` are both 2, although three links are added.
- Gamma's Pomodoro line has no `reason: "note_pull"`.
- The no-pick probe never offers Gamma, so `-p 3` is refused, while `-p all` links it.

That makes the preview and the execution disagree. The cause is the SL20 net pass,
`apply_successor_net_post_pass` in `src/native/capture/task_complete.rs`.
`keep_unblocked_row` and `successor_task_lookup` look the row up by its original
`(note_path, block_id)`. The move has left that note, so the row is dropped.

Fix it for **note-pull rows only**:

- **Build a batch-level move map** from the old `(relative path, block id)` to the new
  `(relative path, block id)`. Draw it from two sources:
  - Each item's `project_move_landings`. `plan_capture_batch` currently
    `std::mem::take`s them for `task_tracker.remap_project_moves`, so record a copy
    there.
  - Each item's `transfer_move_output.moves`: `source_path`, `destination_path`,
    `old_block_id` and `new_block_id`.

  Only consume these structures. Do not change project-note or transfer execution.
  Follow chains, because one task may move twice in a draft.

- **Remap before the per-row checks.** In the note-pull loop of the net pass, remap each
  row whose `(note_path, block_id)` was moved. Update `note_path`, `block_id` and `line`
  to the final location, then run the existing open, live, re-blocked and linked-survive
  checks against that identity. Keep `offer.key` frozen at its pre-pick value, because
  numeric and key picks map by key and must stay stable between probe and submission.
- **Dedup on final identity.** Earliest-wins dedup must also compare the final identity,
  not only the key. Otherwise a later pull of the destination note re-offers a task an
  earlier pull still owns under a different key, for example `@work` / move /
  `@work_docs_push`.
- **Keep the Pomodoro reason.** `surviving_successor_link_reasons` must match the
  remapped link, so Gamma's Pomodoro line keeps `reason: "note_pull"`. Task-block roles
  already follow moves; keep them correct.
- **Leave unresolvable rows alone.** A row the move map cannot resolve, such as an
  ID-less row that cannot be a move target, keeps today's drop behavior.

Add regressions at two levels:

- Unit or batch tests in `src/native/capture/note_pull.rs`.
- CLI tests in `tests/cli/capture/note_pull.rs`, on synthetic temp vaults only.

They must cover:

- A pull with `-p all` followed by a project-note move of a picked row. Expect `3 of 3`,
  the row's `note_path` at the destination, one link with `reason: "note_pull"`, and
  freshness and status written once.
- The same draft with no picks. Gamma stays offered with a stable `n`, and `-p <n>` and
  `-p <key>` select it.
- A pull followed by a transfer move (`@work+gamma > @bob`), with the same assertions.
- A pull of the source, a move, then a pull of the destination. Expect one offer owner
  and one link.
- A move followed by a pull of the source. Today's behavior stays: the moved task is not
  offered.
- Dry-run versus real-run parity, and an unchanged-bytes rollback when a later item
  fails.

Update the human output only as far as the existing locator rendering already shows the
row's `note_path`.

## 3. Docs and fixtures

Read `docs` memory first: `sase memory read docs.md -r "<why>"`.

- If `docs/task-dependencies.md` §12.20 or §12.21 states net-pass behavior for note
  pulls, add one sentence: rows a later item moves follow the move and keep their offer
  key. Do this only if the section would otherwise be wrong.
- Update the coverage table in `tests/fixtures/note_pull/README.md`. Add the `4db3200c`
  regression names that are still missing:
  - `fold_gating_closed_prereq_keeps_later_filter`
  - `blocked_project_counts_only_open_prerequisites`
  - `batch_repeated_source_selected_ownership_prunes`
  - `batch_selected_handoff_owns_row_later_pull_omits`
  - `target_runs_and_one_note_pull_keep_capture_order_and_offer_ownership`
  - `note_pull_human_subset_two_notes_and_mixed_groups`

  Add the new move regressions as well.

- Regenerate saved CLI fixtures only with `tests/fixtures/note_pull/generate.py`, and
  only if their output changes. Never hand-edit them.
- If any fixture changes, add a note on `bob-cli-gj.3` (the Mac phase) naming the
  producing tree and the changed files.

## 4. Gates

1. Run `cargo fmt --all`.
2. Run the focused tests:
   - `cargo test --lib note_pull`
   - `cargo test --lib vault_links`
   - `cargo test --lib capture_dependency_tasks`
   - the theme-warning tests in `budget` and `pomodoro_themes`
   - `cargo test --lib capture_language`
   - `cargo test --test cli note_pull`
   - `cargo test --test cli theme`
3. Run `cargo check --all-targets --locked`.
4. Run `just check` and `just docs-check`.

The previous tale reported two failures it reproduced on a clean base:
`catalog document_shape` (69 vs 67) and `protocol_1`. If either still appears, reproduce
it on a clean `git stash` base before treating it as pre-existing. Failures the repairs
cause are this tale's work. Long builds must still run in the foreground, with a larger
timeout if one is killed.

## 5. Close this epic and its parent phase in this same turn

Do every step below in the turn that finishes the code. Do not wait for this work's
commit SHA, push or CI. Never use `--force`.

1. Reread `bob-cli-gj.2.1` and its descendants. Confirm that steps 1–4 are met against
   the tested tree.
2. Follow-up routing:
   - If you reproduced the move-drop in `!` successor or relation groups, run
     `/sase_new_task` with details naming the `bob-cli-gj.2.1` landing as the proposer.
     That gap traces to the bob-cli-gl move integration.
   - Otherwise file nothing.
   - Add a note on `bob-cli-gc` (`!` close latency) with the before/after timings for
     the catalog walk and the per-capture theme warnings. Its unattributed ~60 ms
     includes both.
3. Run `sase bead epic-symbols bob-cli-gj.2.1`. Resolve every listed entry by wiring it
   up, privatizing it, adding a valid non-test pragma, or deleting it. Re-key an entry
   only to a still-open bead that truly needs it.
4. Close the epic with `sase bead close bob-cli-gj.2.1 --note "<verification>"`. The
   note must state:
   - the source fixes and their regressions;
   - the move integration;
   - the before and after benchmark p95 for both routes, with host and revision;
   - the gates run;
   - the symbol check;
   - the triage outcomes. bob-cli-gj.2.1.2 #1 stays declined: blanket `-D warnings` is
     not the canonical gate. bob-cli-gj.2.1.4 #3 is resolved here by step 1. List any
     task filed in step 5.2.

   If symbols block the close, clean them up and retry. If an unfinished phase is named,
   finish it. Do not force.

5. Run `just symvision` if the recipe exists.
6. Open the plans repository with `/sase_repo` and use the path it prints. Set
   `status: done` in the frontmatter of `202611/note_pull_cli.md`
   (`plan:202611/note_pull_cli.md`). Declare that repository in `/sase_final`.
7. Run `sase bead read bob-cli-gj.2.1 -r "Need the parent link"` and reread
   `bob-cli-gj.2`. It is a phase. Verify that its CLI scope is complete: grammar,
   planner, picks, net pass, refusals, inbox routing, JSON, human output and NP vectors.
   Then close only that phase with
   `sase bead close bob-cli-gj.2 --note "<CLI scope and child-landing verification>"`.
   Leave epic `bob-cli-gj` and `plan:202611/note_pull.md` alone; their own land agent
   handles them. If the phase is incomplete or ambiguous, note the blocker on it and
   report it instead.
8. Finish through `/sase_final`, declaring every modified repository. Report the closed
   IDs, the verification, and any remaining blocker.
