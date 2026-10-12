---
tier: tale
title: Marker-free, self-pruning task status groups
goal:
  bob task reconcile and reroll stop writing task-status-group comments and emit only
  non-empty status groups, with no loss of robustness.
size: medium
decisions:
  decision_record:
    ask:
      Add a decisions-web record saying status groups are owned by title and shape and
      exist only while they hold a task?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.6w.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-12 06:29:48
status: wip
---

# Marker-free, self-pruning task status groups

## Problem

`bob task reconcile` (and `bob task reroll`, which shares the transform) writes a hidden
ownership comment under every generated status-group heading in area/project `Tasks`
sections:

```markdown
### Blocked

<!-- bob:task-status-group:v1:blocked -->
```

Once a container is decorated, it also keeps all three headings (`Next & In Progress`,
`Blocked`, `Done & Canceled`) forever, even when they are empty. The live vault has 264
of these comments in 88 notes, and 143 of the 264 groups are empty. Bryan wants both
gone: no comment lines, and no empty sub-sections.

The research for this work is
`research:202610/task_status_group_markers/task_status_group_markers.md`. Read it with
`sase artifact read` before starting; it has the vault statistics, the prototype diff,
and the probes. This plan adopts its option A ("the heading is the marker").

## Design summary

1. **Ownership by title and shape, not by comment.** A heading directly under a `Tasks`
   container (or under an authored topic inside one) is a managed group if and only if
   its title is exactly one of the three group titles **and** its body holds only
   movable task blocks and blank lines. This is today's "safe adoption" rule for
   unmarked headings (`adoptable_group_body`), promoted to be the only rule. A canonical
   heading whose body holds anything else (prose, a plain bullet, an HTML comment, a
   stray legacy marker, an ordered-list task) makes the container fail closed with
   `ownership_collision`, and bob writes nothing there.
2. **A group exists only while it holds a task.** Empty groups are never emitted. Under
   rule 1 a group body can contain only tasks and blank lines, so dropping an empty
   group can only drop bob's own heading line and blank lines. Group order stays fixed
   (Next & In Progress, Blocked, Done & Canceled), so a returning group always lands in
   the same place.
3. **Decoration is a pure function of note content.** A container is decorated (badge
   row plus groups) if and only if it holds at least one groupable task (Next, In
   Progress, Blocked, Done, or Canceled). When it holds none, the badge row and every
   group heading go, and any Ready task found inside an old group returns to the intake
   through the existing `recovered` path. Nothing is remembered between runs.
4. **Zero-count group chips are unlinked.** A chip whose group does not exist renders as
   a plain code span (`` `🔴 0 blocked` ``). The `open` chip always links to the
   container heading. The badge-row grammar already accepts mixed linked and unlinked
   chips, so `is_badge_row` and capture placement are unaffected.
5. **Legacy comments migrate automatically.** A legacy comment is dropped only in the
   slot bob wrote it to: the first non-blank line under a heading whose title matches
   the comment's kind. Any legacy comment elsewhere fails closed (see the warning table
   below). The recognizer stays permanently, like `is_legacy_badge_marker`, so old
   backups and unsynced machines self-heal.

### Why this is safe

- Every task block is always re-emitted. Only these bytes can disappear: legacy comment
  lines in their slot, canonical heading lines whose bodies hold only tasks and blanks,
  blank lines, and the generated badge row. Anything else fails closed.
- It is strictly more conservative than today. The marker is what currently lets bob
  rewrite a group body containing non-task lines, and that rewrite silently deletes any
  line between two tasks (`split_group_prose` keeps only leading and trailing prose).
  That is bug `bob-cli-6r`. Under this design such a body fails closed, and the lossy
  code is deleted.
- The guarded-write path (lock, read-set re-check, 2-second quiet period, recovery
  copies) is unchanged and still covers every structural write.
- The research's sandbox run of the prototype on a copy of the live vault rewrote
  exactly the 88 marker-bearing notes, with 0 warnings and 0 task moves. It removed only
  marker lines, empty headings, and blank lines, and a second run changed nothing.

### What is given up (accepted)

- Prose inside a generated group is no longer supported; it fails closed until it moves
  to the intake or an authored topic. The vault has 0 such groups, and the current
  support is lossy.
- A hand-typed empty `### Blocked` (etc.) under `Tasks` is removed on the next run
  (after the quiet period).
- A vanished group's Obsidian fold state is gone when the group returns.

## Code changes (`src/native/task_status_groups/`)

### `mod.rs`

- Delete `MARKER_ACTIVE`, `MARKER_BLOCKED`, `MARKER_CLOSED`, and `GroupKind::marker()`.
  Rename `MARKER_PREFIX` to something like `LEGACY_GROUP_MARKER_PREFIX` and keep
  `GroupKind::from_marker_id` as the legacy recognizer. Add a doc comment mirroring
  `is_legacy_badge_marker` ("kept permanently so old backups and unsynced machines
  self-heal"). Tests that need legacy input write the comment text literally, or use a
  `#[cfg(test)]` helper.
- `rewrite_container`:
  - `should_decorate = parsed.has_groupable()` (drop `|| classified.has_groups()`).
  - `should_rewrite = should_decorate || parsed.has_badges || classified.has_groups()`,
    so a container whose groups exist but are no longer needed is rewritten plain.
  - Report every rewritten container: push a `GroupedSection` whenever
    `grouping_changed`, not only when `should_decorate`. A container that returns to a
    plain layout is then reported with all three group counts at 0. Its `open` count and
    `moved_blocks` (recovered tasks moving to `intake`) come from the plan as today.
    This matches the documented contract ("`grouped_task_sections` contains only
    containers whose bytes changed for grouping"). It also stops the human report from
    saying "already in sync" after a structural write. No JSON field is added.
  - Apply the section-end rule below to the emitted section on the rewrite path.
- `classify_children`, for a child whose title is a group title:
  - Keep the existing `badge_row_in_span` check (`misplaced_badge_row`) and the
    child-heading check (`authored_heading_in_group`, pushed as `Authored`) as they are.
  - Keep `Mismatch` → `renamed_marked_heading` and `Malformed` → `malformed_ownership`.
  - Merge `MarkerState::Valid(kind)` (legacy comment in the slot) and
    `MarkerState::Missing` into one path. If `adoptable_group_body(child, ctx)`, it is a
    `Group` (with the existing duplicate check); otherwise `ownership_collision`.
  - **Precedence:** `ownership_collision` must never overwrite a `misplaced_badge_row`
    already found for the same child. A badge row in a group body makes
    `adoptable_group_body` false, and the prototype let the generic code win. Keep the
    specific code.
  - Non-group-title children with a legacy comment as the first non-blank line keep
    failing with `renamed_marked_heading`; `stray_marker_in_exclusive` keeps reporting a
    legacy comment in the intake as `malformed_ownership`.
- Update the `OwnershipCollision` message. "unmarked" no longer means anything; use
  something like
  `"status-group heading in {title:?} at line {line} contains non-task content"`. Keep
  every warning **code** string unchanged.

### `parse.rs`

- `adoptable_group_body` parses from `group_body_start(node, ctx.lines)` instead of
  `node.body_span.start`, so a legacy comment in the slot is skipped by the shape check.
  A legacy comment anywhere else in the body is an `Other` piece with non-whitespace
  text, so the body is not adoptable and the container fails closed.
- Delete the lossy prose plumbing, which is unreachable once every group body is tasks
  plus whitespace: `split_group_prose`, `ParsedGroup::{leading, trailing}`, and helpers
  that become unused (`piece_start`/`piece_end` if nothing else uses them). This removes
  the `bob-cli-6r` code path entirely.
- Update the module doc comment ("Marker and badge parsing") to say that group markers
  are legacy-only.

### `emit.rs`

- Delete `GroupingPlan::{group_leading, group_trailing}` and their population in
  `grouping_plan`.
- `emit_grouped_body`: when decorating, emit only the groups whose task list is
  non-empty, in `GroupKind::ALL` order.
- `emit_group`: emit `"{hashes} {title}{newline}{newline}"` followed by the tasks. There
  is no comment line and no leading or trailing prose. Remove the now-dead
  leading/trailing branches and any helper they alone used (`trim_blank_edges`, for
  example).
- `render_badges`: a group chip with count 0 renders unlinked
  (`!linked || (group.is_some() && count == 0)`). The `open` chip keeps linking to the
  container. The existing all-unlinked fallback for `#` in the ancestry is unchanged.
- **Section-end rule** (fixes the prototype's extra blank line at EOF). The rewritten
  container section must end with exactly the trailing run of whitespace-only lines that
  the original section ended with. Strip trailing whitespace-only lines from the emitted
  body, keeping the last content line's line ending, then append the original section's
  trailing whitespace-only lines byte-for-byte. Today the emitter also eats the blank
  line between the last group and a following heading (for example `- [x] #task done`
  directly followed by `## Notes`); this rule fixes that too. The rule is idempotent and
  causes no churn on already-grouped notes, because today's output always ends a
  decorated section with 0 trailing blank lines. Handle LF and CRLF, and leave the
  document-level `apply_final_newline` as is.

### Help text and other callers

- `src/native/task_status_hooks/mod.rs` long help (around line 134): replace "generated
  groups use hidden ownership comments" with wording such as "generated groups exist
  only while they hold a task".
- `src/native/task_status_hooks/compose.rs` and `src/native/randomize_plan.rs` need no
  logic change. `bob task reroll` inherits the behavior through `transform`.
- `src/native/capture/sections.rs` needs no change (`is_badge_row` already accepts mixed
  rows).

## Warning codes after the change

| Code                     | Trigger                                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `ownership_collision`    | A group-title heading whose body holds anything but movable task blocks and blank lines, including a legacy comment that is not in the slot |
| `malformed_ownership`    | Legacy comment with an unknown kind, or a legacy comment in a container intake                                                              |
| `renamed_marked_heading` | Legacy comment as the first line under a different group title or a non-group title                                                         |
| `misplaced_badge_row`    | Badge row or legacy badge comment inside a group body (still wins over `ownership_collision`)                                               |
| others                   | Unchanged                                                                                                                                   |

## Tests

### Rewrite existing tests (`src/native/task_status_groups/tests.rs`)

- `GOLDEN_OUTPUT` and `tests/fixtures/task_status_groups/layout.grouped.md`: drop the
  three comment lines. All three groups are non-empty there, so the badge row is
  unchanged.
- `authored_child_containers_get_independent_badges_and_anchors`: Frontend's next/wip
  chip becomes the unlinked `` `🔵 0 next/wip` ``, and its open chip stays linked.
- `safe_legacy_adoption_completes_a_partial_set`: rename it. Unmarked `### Blocked` plus
  a Next task in the intake yields `Next & In Progress` and `Blocked` only, with no Done
  heading and no comments.
- `prose_inside_managed_groups_is_preserved`: replace it with fail-closed coverage (see
  new test 6).
- `empty_groups_are_retained_once_created`: replace it with new test 3.
- `heading_only_first_setup_is_a_change`: expect the badge row plus `### Blocked` only;
  assert there is no Next/Done heading and no `task-status-group` text.
- `nested_children_travel_with_parent_status`: there is no Done root task, so assert
  that `TITLE_CLOSED` is absent instead of splitting on it.
- `orphaned_badge_block_is_removed_without_creating_groups`: the removal is now reported
  as one `GroupedSection` with zero group counts.
- `misplaced_badge_rows_fail_closed`: keep the legacy-marker inputs, add a marker-free
  variant, and still assert `MisplacedBadgeRow` (precedence).
- `malformed_and_duplicate_markers_fail_closed` and
  `authored_heading_inside_a_managed_group_fails_closed`: keep them, and add marker-free
  duplicate and authored-heading variants.
- Anything else that references `MARKER_*` or expects empty headings: update it to the
  new layout.

### New unit tests

1. **Legacy migration, LF and CRLF.** A legacy section with three commented groups, one
   of them empty, comes out with no `task-status-group` text, without the empty group,
   with the tasks conserved, and idempotent.
2. **No comments on fresh grouping.** Golden output has no `<!--`.
3. **Vanish and return.** A group disappears with its last task and reappears with its
   first, in fixed order. For example, a Blocked task in a Next+Done note produces
   `### Blocked` between them, and its chip becomes linked again.
4. **Un-decoration.** When the last groupable task leaves (closed tasks archived away,
   or a Done task reopened to Ready inside the Done group), the badge row and every
   group heading are removed, Ready tasks are recovered to the intake, and the change is
   reported with zero group counts. A second pass is a no-op.
5. **Zero chips.** Unlinked zero chips, a linked `open` chip at 0, and `is_badge_row`
   accepts the emitted row.
6. **Interior content fails closed (`bob-cli-6r`).** A plain bullet, a prose paragraph,
   an HTML comment, or a legacy comment between two tasks in a group, and prose above or
   below the tasks, each leave the bytes unchanged with `OwnershipCollision`. Test both
   marker-free and legacy-commented groups.
7. **Legacy comment out of slot.** In the intake → `MalformedOwnership`; under the wrong
   group title → `RenamedMarkedHeading`; under a non-group title →
   `RenamedMarkedHeading`. All leave the bytes unchanged.
8. **Hand-typed empty group heading is removed**, both in a decorated container and in a
   Ready-only container (which ends plain).
9. **Renamed heading becomes an authored topic.** `### Waiting` holding a `[?]` task
   gets its own badge row and `#### Blocked`.
10. **Section-end rule.** No extra blank line at EOF after a trailing group vanishes or
    after un-decoration; the blank line before a following `## Notes` survives grouping;
    CRLF variants; all idempotent.
11. **Blank lines between tasks in a group** are still adoptable and normalize away, as
    today.

### Other test updates

- `src/native/capture/tests/placement.rs`
  (`tasks_section_inserts_below_generated_status_badges`): drop the comment from
  `GROUP`, and add a case whose row has unlinked zero chips.
- `src/native/randomize_plan.rs`
  (`project_postimage_is_byte_exact_with_blocked_move_and_badge`): the new expected
  postimage keeps the badge row with unlinked next/wip and done chips and a linked
  blocked chip, followed by a blank line, `### Blocked`, a blank line, and the task with
  its schedule log. There is no Next or Done heading and no comment.
- `tests/cli/task_status_hooks/sync.rs`:
  - In the grouping flow test, drop the comment from the ordered-text assertions, assert
    that no `bob:task-status-group` text appears in any written note, and assert that
    the area note (Next tasks only) has no `### Blocked` or `### Done & Canceled`. Keep
    the "already in sync" re-run assertion.
  - In `task_status_hooks_reports_grouping_warnings_without_noop_text`, convert the
    `badges.md` fixture to the marker-free layout (badge row inside a
    `### Next & In Progress` body). It must still report `misplaced_badge_row` and
    `grouped_task_sections: []`.
  - Add a CLI case for un-decoration: a note whose legacy-commented groups hold only a
    Ready task. The JSON report lists it in `grouped_task_sections` with zero group
    counts, the human report does not say "already in sync", and the written note is a
    plain `Tasks` section.

Known pre-existing failures: two `percent_listen` CLI tests fail on clean master
(`bob-cli-6m`). Do not chase them; mention them in the final report if they still fail.

## Docs

- `docs/task-status-hooks.md` § Status Grouping:
  - Remove "with hidden ownership comments" and the comment lines from the example.
  - Describe ownership by title and shape, the appear/vanish rule, un-decoration, and
    unlinked zero chips. Replace "Empty buckets still show `0` so the row keeps stable
    link targets" and "all three headings are retained even when one is empty".
  - Rewrite the warnings paragraph to match the table above.
  - Say that legacy `<!-- bob:task-status-group:v1:* -->` comments are removed on the
    next rewrite.
  - Add one upgrade sentence: an older `bob` re-adds the comments and empty groups
    (losslessly), so upgrade every host that runs reconcile or reroll together.
  - Update the JSON example's `ownership_collision` message and the
    `grouped_task_sections` paragraph (it now includes containers that returned to a
    plain layout, with all group counts 0).
- `docs/randomize.md`: remove the comment line from the "After" example.
- `README.md` (around line 596): mention that only non-empty groups are emitted.
- Run `rg -n "task-status-group|ownership comment" src tests docs README.md` afterwards.
  The only matches should be the legacy recognizer, its migration tests, and the docs'
  legacy sentence.

> [!decision] decision_record Use `/sase_memory_write`, then add a decisions-web strand
> (for example `status-groups-are-owned-by-shape`): "Status groups are owned by title
> and shape and exist only while they hold a task". Applies to bob-cli
> `task_status_groups` (reconcile and reroll). It should record the claim (design
> summary points 1–4), why (the marker's only power was prose-in-groups, used 0 times
> and lossy per `bob-cli-6r`; no hidden state; layout is a pure function of content),
> the rejected alternatives from the research §4 (badge-row ownership, per-host state,
> fingerprinted headings, `%%` comments, keeping markers), the cost (no prose in groups,
> hand-typed empty headings removed, fold state lost), and what would reopen it (wanting
> prose inside groups, which is option B). Cite the research ref and the landing commit.
> Then run `sase memory init`.

## Verification

1. `just check` passes (fmt, clippy with all targets, every test binary), apart from the
   known `bob-cli-6m` failures.
2. Read-only check against the live vault with the freshly built binary:
   `cargo run --quiet -- task reconcile --dry-run --format json --bob-dir ~/bob`. Expect
   `grouping_warnings` to be empty and roughly 88 entries in `grouped_task_sections`,
   with no unexpected `moved_blocks`. Investigate any warning before finishing.
3. Sandbox live run (recommended, as the research did). Copy the vault's Markdown and
   `.obsidian` config into a temp dir with mtimes preserved. Run the new binary live
   twice with isolated `BOB_DIR`/`--bob-dir`, `XDG_STATE_HOME`, `XDG_CACHE_HOME`, and
   `BOB_VAULT_SYNC_LOCK_FILE`. Confirm three things:
   - the second run applies 0 files;
   - `rg -c 'bob:task-status-group'` finds nothing in the copy;
   - comparing the multiset of non-blank lines before and after shows only removed
     legacy comments, removed empty canonical headings, and changed badge rows.

   Keep any helper scripts outside the repo.

4. Do **not** run a live reconcile against `~/bob`, and do not install the binary
   (`just install`). The MacBook cron runs an older `bob` that would re-add the comments
   and every empty group. That is lossless, but it would churn the 88 notes through
   vault-sync until every host is upgraded.
5. After everything passes, close `bob-cli-6r` with
   `sase bead close bob-cli-6r --note "<the fail-closed tests that cover it>"`.

## Rollout (for Bryan; include in the final report)

1. Install the new `bob` on every host that runs `bob task reconcile` or
   `bob task reroll`, the MacBook cron first (or pause its cron line until it is
   upgraded).
2. Preview with `bob task reconcile --dry-run --format json`.
3. Run once live. Then `rg -c 'bob:task-status-group' ~/bob` should find nothing, and a
   second dry run should report no grouping changes.

## Out of scope

- Prose inside generated groups (research option B).
- Changing chip order, labels, emojis, or the badge-row grammar.
- Capture placement, the move-task gesture, and the Ctrl+Shift+P picker, which already
  insert into the intake.
