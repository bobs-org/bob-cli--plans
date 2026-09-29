---
tier: tale
size: medium
title: Finish landing bob-cli-2k (=x<N>!<M> close selection) and close the epic
goal:
  The =x<N>!<M> close selection has no remaining epic-caused defects in row numbering,
  listed warnings, grammar diagnostics, block-ID completion, or docs, and epic
  bob-cli-2k is closed with its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-2k.land
bead: bob-cli-2k
status: done
---

- **PARENT:**
  [202609/close_task_selection.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/close_task_selection.md)
- **BEAD:**
  [bob-cli-2k](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2k/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2k.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2k.land.md)
- **COMMITS:**
  - [afb2e5c](https://github.com/bobs-org/bob-cli/commit/afb2e5c19174b902f1d34ea6f03bf594e686b8cb)
    — fix(capture): land epic bob-cli-2k selection follow-ups

# Finish landing epic bob-cli-2k

Epic `bob-cli-2k` ("Choose each Task Link's outcome while closing a Pomodoro with
`=x<N>!<M>`", plan `plan:202609/close_task_selection.md`) has five closed phases. Their
commits are `6f45d38`, `1838779`, `2c32a91`, and `b3405bd` in bob-cli, plus `7e672cc`
and `f6eae0b` in bob-mac-capture, where macOS CI run 36615238978 is green. The land
review confirmed that the feature works: `cargo test` is fully green,
`cargo fmt --check` is clean, and clippy has no new warnings. It also found the defects
below, which the epic caused and so must be fixed before the epic closes. No unrelated
commits landed in either repo since the epic started, so there is no other integration
work.

**Follow-up triage is already done and recorded on `bob-cli-2k`.** Do not create task
beads for any of these. The pre-existing clippy deny at
`tests/cli/capture/pomodoro_name.rs:808` (`|| true`) belongs to epic `bob-cli-28`, and
the clippy warnings belong to `bob-cli-v`, so
`cargo clippy --all-targets --all-features` still exits non-zero because of that one
untouched test line. Your acceptance bar is no new clippy warnings or errors in files
you touch. The gkeep_auth flake is `bob-cli-2m`, and the `^route:id` deferred-link
duplicate belongs to `bob-cli-28`. Leave all of those alone.

The design contract for everything below is the epic plan
(`sase/repos/plans/202609/close_task_selection.md`, sections "The contract" and "Output
contract"). Read it first. The shared fixture is `close_worked_vault` in
`tests/cli/capture/pomodoro_close.rs`, run with `BOB_NOW=2026-09-28 09:37:00`.

## 1. Planner and output: numbered rows and listed warnings

Work in `src/native/capture_pomodoro_close/linked_tasks.rs` and
`src/native/capture/output.rs`.

1. **Row indices come from task identity, not from the row's first line.**
   `assign_task_indices` only matches a numbered link whose `line` equals the row's
   `ledger_line`. A task gets one row, registered from the first ledger line that
   mentions it. So when an earlier unnumbered line (mentioned or struck) links the same
   task, the row ends up with `index: null` even though the task is numbered. Example: a
   running Pomodoro with `\t- see [[bob#^blocked]] for context` followed by
   `\t- [[bob#^blocked]]`.

   Fix it by mapping every `NumberedTaskLink` to the row that owns its task:
   - resolve `path_part` with `vault.resolve_target(day_path, …)`;
   - for `Found(path)`, the key is `(path, block_id)` in `planner.task_indices`;
   - for missing or ambiguous targets, the key is `"{path_part}#^{block_id}"` in
     `planner.unresolved_indices`.

   Then set `row.index` to the lowest number among the links mapped to that row.
   `Subtask` rows keep `None`. Compute this link→row mapping once and reuse it in
   item 2.

2. **Listed-status warnings are keyed per listed link.** `emit_listed_status_warnings`
   looks up only the row's (lowest) index, so it misses a listed higher duplicate. For
   example, with 1 = `![[T]]` and 2 = `[[T]]`, `=x!2` on a Blocked `T` gives no warning.
   Iterate over the **listed** links, find each link's row through the item-1 mapping,
   and warn at most once per row, using the lowest listed number on that row.

   Also decide by status **type**, not by symbol. Read the row's task through
   `task_for_key((resolved_path, block_id), …)`:
   - warn "…so it was not started" when an in-progress outcome's task is not of type
     `TaskStatusType::InProgress`;
   - warn "…so it was not completed" when a complete outcome's task is neither of type
     `TaskStatusType::Done` nor symbol `x`. This matches `retire_closed_embeds`.

   Keep the existing message text and the rule that unresolved rows never warn twice.

3. **The human index column turns on only when a row is numbered.** In
   `print_human_pomodoro_close_success`, `numbered`/`index_width` currently come from
   `close.task_links`. Per the contract ("When at least one row is numbered … width =
   digits of the highest number"), compute both from `close.tasks[].index`. With zero
   numbered rows the output must be byte-identical to before the epic.

**Tests** (unit tests in `selection_tests.rs`/`linked_task_tests.rs`, CLI tests in
`tests/cli/capture/pomodoro_close_selection.rs`) must assert, never comment:

- a mentioned-first then bare link to a Blocked task, with `=x1`: the row has `index: 1`
  and gets the "is Blocked, so it was not started" warning;
- the `![[T]]` / `[[T]]` duplicate with `=x!2` on a Blocked task produces the "not
  completed" warning;
- a custom In Progress status whose symbol is not `/`, listed in `<N>`, gets no warning;
- `tasks[].index` carries the lowest number when one task is numbered on two
  same-outcome lines;
- an unresolved listed row warns exactly once;
- plain `=x` human output is byte-identical to the pre-epic format when no row is
  numbered, including the mentioned-first case, which now gets numbered and so shows the
  column. Also assert a fixture with no bare Task Links at all.

## 2. Grammar fixes

Keep diagnostic text in `capture_language/markers.rs`. `bob capture` and `capture-parse`
must agree on every message and range. Pin every fix below with a protocol test in
`tests/cli/capture/parse_pomodoro.rs` (and `complete_editor.rs` where noted), plus a
`bob capture` error test for each execution-visible change.

1. **A `^` link close ending in `!` is incomplete, not a toggle.** In
   `src/native/capture_language/tokens.rs`, both `parse_caret_link_token` (the
   `rest.ends_with('!')` check) and `classify_caret_token` (`tail.ends_with('!')`)
   reject a trailing `!` as `POMODORO_LINK_TOGGLE_ERROR` before the close suffix is
   examined. So `^r:id=x!`, `^r:id=x1!`, `^r:id=x0!`, and `^r:id=x1,3!` all say "is a
   task toggle". Run the toggle check only when the tail does not carry a close-shaped
   `=x…` suffix; the `@` forms already get this right.
   - Expected in `capture-parse`: mode `incomplete`, need `pomodoro_close_task`, the
     partial spec, and a placeholder span over the `!`.
   - Expected in `bob capture`:
     `` `=x1!` is incomplete: type a task number after `!` ``.
   - `^r:id!` (no `=`) must still give the toggle error.

2. **Extra-text range.** In `src/native/capture_language/editor_pomodoro.rs` (the
   whole-item close's extra-text branch, around `parent_text.find(rest)`), the offset of
   `rest` is found by substring search, so it can land inside the close token itself.
   Compute it from the first token's end plus the skipped whitespace instead. Expected
   ranges:
   - `=x1 1` → [4,5);
   - `=x1,3 3` → [6,7);
   - `=x1 x1` → [4,6);
   - `  =x1 1` → [6,7).

3. **Empty-element ranges and messages** (`close_selection.rs`, `parse_number_list` and
   the dangling-separator path):
   - `=x1,,` must point at the second `,` ([4,5)), matching `=x1,,2`;
   - `=x1!2,,` → [6,7);
   - `=x1,!2` must report `` expected a task number after `,` `` (a new shared message)
     on the `,`, not "before `,`".

4. **`=x0,` is an error, not an editing state.** Nothing typed after `0,` can be valid,
   and a lexical error wins over the incomplete state. Report the `0`-alone message on
   the `0`, exactly as `=x0,2` does. `=x0!` stays incomplete, because `=x0!2` is valid.

5. **Only a literal `0` means "none".** Today `=x00` (and `=x000`) is accepted as
   `in_progress: []`. Give a zero-valued `<n>` other than a lone `0` in `<N>` the
   `0`-alone message on that number. In `<M>`, zeros already give
   `task numbers start at 1`; keep that.

6. **Execution and editor agree on invalid link closes with conflicts.**
   - For `^r:id=x1,1 s:2` and `^r:id=x1, s:2`, `bob capture` currently says
     "`^route:block-id` must be the whole capture item…". `capture-parse` instead
     reports "Pomodoro link capture cannot be combined with s:<N>".
   - Likewise for `^r:id=x1,1` followed by a child line.
   - Make execution report the same conflict message the editor reports, as the comment
     in `editor_parse.rs`'s `CloseInvalid` arm already promises ("a conflict wins …
     exactly like execution").
   - Extend `editor_agrees_with_execution_for_resolved_captures` (or a sibling parity
     test in `capture_language/tests/editor_modes.rs`) with these inputs. Make the
     existing parity check also compare `in_progress`/`complete`, not only `raw`.

7. **Block-ID completion inside an incomplete close list.** In
   `src/native/capture_block_ids.rs` `detect_colon_intent`, an item counts as a link
   only when its mode is `PomodoroLink` with no diagnostics. So `@r:id=x1,` with the
   cursor in the block ID returns intent `new` with 0 candidates, while `@r:id=x1` and
   `@r:id=x1!2` return `link`. Also treat an item whose mode is `incomplete` and whose
   `needs` is exactly `pomodoro_close_task` as the link item it will become. Pin this in
   `tests/cli/capture/complete_editor.rs`: cursor in the block ID of `@r:id=x1,` and of
   `^r:id=x1!`.

8. The `raw` doc comment on `PomodoroCloseSpec` (`capture_language/model.rs`) says
   `x`/`X` plus lists. It is actually the `=`-prefixed token (`=x1,3!2`). Fix the
   comment.

## 3. Docs and help

1. `docs/capture.md`: the `=x2` block (near line 1068) and the `=x0` block (near
   line 1117) show the plan's folded entry line (`— CAPTURE - Designed the …`). Replace
   both with the real bytes the binary writes, which are pinned in
   `tests/cli/capture/pomodoro_close_selection.rs`. The entry line stays
   `- [x] (**0920-0940** [t:: 20m]) — CAPTURE`, and the orphaned notes follow as
   sub-bullets:

   ```markdown
   - [x] (**0920-0940** [t:: 20m]) — CAPTURE - Designed the `=x` grammar - chose `x` for
         done - Wrote the plan
   ```

   The `=x2` block then continues with `\t- 🍅 [[bob#^web-capture]]` and the rest.
   Regenerate each block with the debug binary on a copy of `close_worked_vault`, and
   copy the output verbatim.

2. `src/native/capture/cli.rs` (near line 217): the help prints a stray line that
   contains a literal `\n`. The source has `history expansion applies.\n\n\\n\n\`.
   Remove the stray `\\n\n`, keeping exactly one blank line between paragraphs.
3. `cli.rs` (near line 195) says the task numbers are "the numbers `capture-parse` and
   the human output show". `capture-parse` never opens the vault and cannot number Task
   Links. Say the numbers are the ones `bob capture` shows, in human output, in
   `--dry-run`, and as JSON `task_links`.
4. `README.md` (near lines 276-277): "`bob capture '=x1!2'` also completes task 2" reads
   as if task 2 were both kept in progress and completed. Reword it:
   `bob capture '=x1!2'` keeps task 1 in progress and completes task 2.
5. `docs/capture.md` (near lines 1097-1099): the completed `^web-capture` line is an
   inline code span broken across lines, which renders a single space before
   `[completion::`. Put that line in a fenced code block, or keep the code span on one
   line.
6. `docs/capture.md` Contents: add `Choosing each Task Link's outcome` (nested under
   "Closing the running Pomodoro"), plus the missing `Starting the session atomically`
   and `Linking and starting existing tasks` headings at their correct nesting. Check
   that every anchor resolves.
7. `docs/capture.md` structure: the plain-`=x` worked example and its "Other rows"
   paragraph now sit under `#### Choosing each Task Link's outcome`. Move them back
   under "Closing the running Pomodoro", before the new subsection, so the subsection
   holds only selection material.
8. Document the item-2 changes where the docs list diagnostics or incomplete states:
   - `^r:id=x1!` is incomplete;
   - `=x0,` and `=x00` get the `0` message;
   - `=x1,!2` gets the "after `,`" message.

   Verify every changed doc example against the built binary.

## 4. Test gaps from the land review

In `tests/cli/capture/pomodoro_close_selection.rs`, assert these fully; do not use
`contains` where a full post-image is available:

- `=x0`: the `sase.md` post-image.
- `=x1,2`: `task_links`, `tasks[].index`, and the `sase.md` post-image.
- `^bob:ready=x3`:
  - `tasks[]` index, role, and status;
  - the `sase.md` and full `bob.md` post-images;
  - `^web-capture` stays `[*]`;
  - `next_pomodoro.created`.
- `Draft docs @bob:draft-docs=x0`: `tasks[]` and the full `bob.md` post-image.
- Batches: the file effects of `=x0` + `^sase:recovery-panel=`, and of `-2` + `=x1!2`.

In `src/native/capture_pomodoro_close/selection_tests.rs`:

- make `fenced_links_are_unnumbered` exercise a fence **inside** the running Pomodoro's
  sub-bullet range (an indented fence), so the fenced-skip branch in `number_task_links`
  actually runs;
- make the outcome-table test assert outcomes, not only sources, for the "otherwise →
  ledger" row.

## 5. Validation

- `cargo fmt --check` is clean.
- `cargo clippy --all-targets --all-features` shows no new warnings or errors. The only
  error must be the known, untouched `tests/cli/capture/pomodoro_name.rs:808` deny.
- `cargo test` passes.

This repo has no `just check` recipe. Do not run `just check-full`.

Commit the work in bob-cli with a message that references `bob-cli-2k`. No
bob-mac-capture change is expected: the Mac app follows Bob's `needs` and `index` fields
generically. If a Mac fixture would change, which it should not because the
worked-example vault has no mentioned-first line, regenerate it from the real binary and
confirm macOS CI (`gh run list -R bobs-org/bob-mac-capture`) goes green.

## 6. Final step: close out epic bob-cli-2k

Do this only after sections 1-5 are committed and green.

1. Run `sase bead epic-symbols bob-cli-2k`. At the land review it printed "No
   --epic-symbol entries for bob-cli-2k". Handle any entry that has appeared since:
   - resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it,
     per the Symvision epic-whitelist policy;
   - re-key the justfile line to a bead only when a still-open later bead needs the
     exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-2k --note "<verification>"
   ```

   The note must say:
   - that all five phases and their notes were verified, citing the commits listed at
     the top of this plan and the green macOS CI run;
   - what this tale fixed, citing its commits;
   - that there was no drift from unrelated commits since the epic started;
   - the follow-up triage outcomes, all already recorded on `bob-cli-2k`:
     - the clippy deny: DISCOVERED ISSUE note on `bob-cli-28`;
     - the clippy warnings: +1 on `bob-cli-v`;
     - the gkeep flake: new task `bob-cli-2m`;
     - the plan's folded post-images: fixed here as epic work;
     - the `^route:id` deferred-link duplicate: DISCOVERED ISSUE note on `bob-cli-28`;
   - the validation results.

   Never use `--force` merely to make the close succeed.

3. Run `just symvision` if the recipe exists. It did not exist at the land review, so
   record that it is unavailable.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path that
   `sase bead read bob-cli-2k -r "<why>"` shows:
   `sase/repos/plans/202609/close_task_selection.md`).
5. `bob-cli-2k` has no `parent_bead`, so there is no parent to handle. Confirm with
   `sase bead read bob-cli-2k -r "Need the parent link"` (it shows no PARENT section),
   then finish.
