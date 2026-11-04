---
tier: tale
size: medium
title:
  Open the task picker at `> +` on capture transfer lines, then close out bob-cli-go
goal:
  Typing `+` right after a capture transfer arrow (`@bob+ch-jumper-keymaps > +`) opens
  the vault-wide task picker in Bob Mac Capture instead of showing a red
  `transfer_destination_marker` error. The picker lists merge targets without the typed
  sources, and accepting a row inserts `@route+id`, which makes the line a merge
  transfer. After that, epic bob-cli-go is verified complete and closed.
proposed_by: bbugyi200.apollo.bob-cli-go.land
proposed_with: claude/opus
bead: bob-cli-go
create_time: 2026-11-04 12:55:11
status: wip
---

- **PARENT:**
  [202611/task_move_merge_split.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_move_merge_split.md)
- **BEAD:**
  [bob-cli-go](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-go/README.md)

# Open the task picker at `> +` on capture transfer lines, then close out bob-cli-go

## Why this exists

Epic `bob-cli-go` (move, merge, and split tasks) is fully implemented except for one
gap. Bryan reported it in epic note #10, with a screenshot. In Bob Mac Capture, the
draft `@bob+ch-jumper-keymaps > +` shows this error:

> capture item 1 starting on line 1: transfer destinations must use @<route>,
> @<route>+<block-id>, @<route>+<block-id>#<drawer>, or @<route>#<section>

No task picker opens. Bryan wants typing `+` after `>` to start task completion.

The epic's landing agent reproduced the gap on master `a6c55cbb` with a debug build:

- `bob capture-parse -f json -- '@bob+ch-jumper-keymaps > +'` reports
  `mode: incomplete`, `needs: []`, `spans: []`, and an error diagnostic
  `transfer_destination_marker` at `[25,26]`. `> +up` behaves the same way. With no
  `task_parent` need and no completion span under the caret, the Mac never asks bob for
  completion. Its `shouldRequestCompletion` in
  `Sources/BobMacCapture/CapturePanelModel.swift` of the bob-mac-capture repo triggers
  on a `task_parent` need or an `interactive_placeholder` span.
- `bob capture-complete -c 26 -f json -- '@bob+ch-jumper-keymaps > +'` already returns
  `context: task_parent` with a vault-scoped `parent_task` picker. Its rows are not
  scoped to the transfer: they include the source `@bob+ch-jumper-keymaps` itself, and
  no row carries the `⤵ merge into` consequence. The `@c+` merge-target picker on
  transfer lines does have that scoping.

The Mac needs no code change. It already requests completion for a `task_parent` need,
opens the parent-task picker from capture-complete's `picker` descriptor, and replaces
`marker_range` with the accepted `@route+id`. Do not edit bob-mac-capture.

This tale does the fix in bob-cli and then finishes the epic's landing. Nothing resumes
the landing after this tale, so the closeout section below is required.

## Grounding

- `docs/capture.md` documents the transfer contract in "Moving and merging tasks with
  `>`" (around line 6108). It documents transfer completion in the capture-complete
  section (the paragraph "On a `>` transfer line the same contexts serve each
  component", around line 8621). The bare `+` parent picker is documented in "Choosing a
  parent task with `+`" (around line 1843).
- `src/native/capture_language/transfer.rs` is the shared transfer lexer. Look at
  `TransferNeed`, `TransferScan::Incomplete`, `scan_with_arrow`, and
  `parse_destination_token`.
- `src/native/capture_language/editor_parse.rs` (around line 1585) maps
  `TransferScan::Incomplete` to spans, needs, and the `incomplete_transfer` diagnostic.
  See also `transfer_need` and `add_incomplete_transfer_selector_span`.
- `src/native/capture_language/draft.rs` (around line 1391) maps execution-time
  incomplete needs to codes: `incomplete_transfer`, `incomplete_transfer_block_id`, and
  so on.
- `src/native/capture_language/tokens.rs::parent_task_selector_tokens` holds the bare
  `+query` selector rule. The last token on the line must start with `+`. The query must
  not start with `+`, `-`, or an ASCII digit, and the token must not sit in protected
  markup.
- `src/native/capture_language/completion.rs`:
  - `parent_task_completion_field_at` already claims `> +` and `> +up` as `TaskParent`
    before the transfer check in `completion_field_at` runs.
  - `TransferCompletionRole`, `transfer_completion_role`, and
    `transfer_role_for_incomplete` decide the transfer scoping role.
- `src/native/capture_complete/transfer_assist.rs` applies role scoping.
  `scope_merge_targets` shows the existing rules for `@c+` rows: drop ID-less rows, drop
  typed sources, and add the
  `⤵ merge into [[route#^id]] — cuts the source notes under this task and retires the source`
  consequence.
- `src/native/capture_complete/model.rs::TaskParentCandidate` has no `consequence` field
  yet. `ProjectTaskMoveCandidate` and the `task` rows already carry one.
- Tests to extend:
  - `src/native/capture_language/tests/transfer.rs` (lexer vectors)
  - `tests/cli/capture/complete_transfer.rs` (fixture `transfer_vault`, plus
    `transfer_merge_targets_drop_sources_and_carry_consequences`)
  - `tests/cli/capture/transfer.rs`
  - `tests/cli/capture/complete_parent_task.rs` (the ordinary `+` picker, which must not
    change)
- Help text: `src/native/capture_parse.rs` (the long help sentence about an unresolved
  parent-task selector `+query`) and `src/native/capture_complete/cli.rs` (around line
  266, the transfer completion sentence). `docs/reference/cli/*.md` is generated from
  these files.
- Before editing `docs/` or user-visible CLI help, read the `docs.md` reference memory
  with `/sase_memory_read`. Read `cli_rules.md` too, in case any option changes; none
  are planned.

## Behavior contract

1. **Lexer.**
   - In `scan_with_arrow`, a destination slot that is exactly one token, `+` or
     `+query`, counts as a picker selector. The token must also pass the `+query`
     selector rule. Extract that rule from `parent_task_selector_tokens` into one shared
     predicate in `tokens.rs`, so the two call sites cannot drift apart.
   - A selector destination returns `TransferScan::Incomplete` with a new
     `TransferNeed::Task` (test label `task_parent`). Its span is the token, and its
     message is `transfer destination needs a task: pick one with +, or type @route+id`.
   - Everything else that starts with `+` keeps today's error:
     - `> +5`, `> ++`, and `> +-x` keep `transfer_destination_marker`.
     - A selector followed by more tokens (`> +up more`) also keeps that error.
     - A selector inside protected markup keeps it as well.
   - Sources, the duplicate check, the `\>` escape, and glued arrows do not change.
2. **Execution.** `bob capture '@inbox+bench > +'` refuses through the existing
   incomplete-transfer path, with code `incomplete_transfer_task` and the message above.
   It exits with the same code as the other incomplete transfers and leaves the vault
   unchanged. `--dry-run` refuses the same way.
3. **capture-parse.** For the new need, report:
   - `mode: incomplete` and `needs: ["task_parent"]` (map it in `transfer_need`).
   - `transfer_source` spans and the `transfer_arrow` span.
   - One `interactive_placeholder` span over the whole `+query` token. Use it instead of
     a `transfer_destination` span, because spans never overlap and the ordinary `+`
     selector reports this same placeholder.
   - The `incomplete_transfer` error diagnostic over the token.
   - No `transfer` object, matching the other incomplete transfers.

   Example: `@inbox+bench > +` gives `transfer_source [0,12)`, `transfer_arrow [13,14)`,
   and `interactive_placeholder [15,16)`.

4. **capture-complete.**
   - On a transfer line, a cursor inside the `+query` token keeps the `task_parent`
     context: the vault-scoped `parent_task` picker, the query stripped of `+`, and a
     replacement and `marker_range` covering the whole token.
   - Add `TransferCompletionRole::VaultMergeTarget { exclude }`.
     `transfer_completion_role` returns it for this shape, and `exclude` holds the typed
     sources.
   - `transfer_assist::apply` scopes `Candidates::TaskParent` rows for this role:
     - Drop ID-less rows (`requires_block_id`), as `scope_merge_targets` does.
     - Drop any row that is a typed source. Compare the route case-insensitively plus
       the block ID. If a candidate's `route` spelling can differ from the typed route
       (nested notes), compare resolved note paths with `capture::route_label`.
     - Set each remaining row's new `consequence`.
   - Extract one `merge_consequence(route, id)` helper so `@c+` rows and `> +` rows
     share exactly one string.
   - Add `consequence: Option<String>` to `TaskParentCandidate` with
     `#[serde(skip_serializing_if = "Option::is_none")]`. JSON for ordinary `+` pickers,
     such as `Call the bank +`, must stay byte-for-byte unchanged.
   - Shell completion inherits the filtered rows.
   - Accepting a row (`@db+vacuum`) turns the draft into a valid `task_merge` transfer.
5. **No grammar revision bump.** `task_parent` is a need that every client at grammar 5
   or later already handles, and this change only turns an error into an incomplete
   need. Do not edit `docs/reference/compatibility.md`. Its grammar prose belongs to
   active epic bob-cli-h0.

## Steps

1. Implement the lexer, execution, parse, and completion changes above with focused unit
   tests in `src/native/capture_language/tests/transfer.rs`:
   - `@a+x > +` → Incomplete, need Task, span `(7,8)`.
   - `@a+x @b+y > +up` → span over `+up`.
   - `@a+x > +5`, `@a+x > ++`, `@a+x > +-x`, and `@a+x > +up more` → still Invalid.
   - `@a+x \> +` → still Escaped.

   Add completion-role unit coverage wherever `transfer_completion_role` is already
   tested.

2. Add CLI tests:
   - **capture-parse JSON** (`tests/cli/capture/transfer.rs`): the exact mode, needs,
     spans, and diagnostic for `@inbox+bench > +` and `@inbox+bench > +up`.
   - **capture-complete** (`tests/cli/capture/complete_transfer.rs`, `transfer_vault`
     fixture):
     - The `> +` cursor returns `task_parent` with a vault picker. Its candidates
       exclude the typed source and ID-less rows, and every row has the exact
       `⤵ merge into [[<route>#^<id>]] — …` consequence.
     - `> +<query>` filters.
     - Round trip: splice an accepted `replacement` into the draft; capture-parse then
       reports `task_merge`, and `bob capture --dry-run` plans the merge.
   - **bob capture refusal**: `incomplete_transfer_task`, vault unchanged.
   - **Regression guard** in `complete_parent_task.rs` or a nearby file: an ordinary
     `Call the bank +` picker row has no `consequence` key.
3. Update the docs and help text:
   - `docs/capture.md` transfer section: say that `> +` or `> +query` opens the
     vault-wide task picker for a merge target. Accepting a row inserts `@route+id`, and
     `bob capture` refuses the unresolved selector.
   - `docs/capture.md` completion paragraph: add the `> +` row scoping and one worked
     example, `bob capture-complete -c 16 -f json -- '@inbox+bench > +'`.
   - "Choosing a parent task with `+`": one sentence pointing to transfer lines.
   - The capture-parse and capture-complete long help in `src/native/capture_parse.rs`
     and `src/native/capture_complete/cli.rs`.
   - The capture-complete row in `docs/reference/json-outputs.md`: `task_parent` rows on
     a transfer line drop sources and ID-less rows and add `consequence`.

   Regenerate the generated references with the repo's docs generator (`just docs-gen`).
   Keep the capture-syntax page under its word budget, and add no new token rows. If
   `just docs-check` token coverage asks for a literal, add the exact query to
   `docs/search_smoke.yml`.

4. Verify:
   - `just check` must pass. Do not run `just check-full`.
   - `just docs-check` must pass.
   - Smoke test a scratch vault with the just-built `bob`. Use one area note with two
     identified tasks plus a project note holding `^ch-jumper-keymaps`, a `#task` global
     filter, and `BOB_DAY_FILE`/`BOB_NOW` set. The screenshot draft
     `@bob+ch-jumper-keymaps > +` must report `needs: ["task_parent"]` from
     capture-parse. capture-complete at the end of the draft must list tasks without
     `@bob+ch-jumper-keymaps`, each with the merge consequence. Splicing a row must give
     a `task_merge` parse.

   Timing-only failures that pass alone are tracked by bob-cli-gd, bob-cli-bo, and
   bob-cli-gz. Note them but do not treat them as caused by this change.

5. **Close out epic bob-cli-go.** This is the final step, and it runs in the same turn
   as the code; it never waits for this work's own commit SHA, push, or CI.
   1. Run `sase bead epic-symbols bob-cli-go`. The landing agent found no entries. For
      any entry that appears, resolve the symbol (wire it up, make it private, add a
      non-test pragma, or delete it per the Symvision epic-whitelist policy). Re-key the
      Justfile line to another bead only if a still-open later bead needs that
      exemption.
   2. Close the epic with `sase bead close bob-cli-go --note "<verification>"`. In the
      note, summarize:
      - The landing review: all 13 phases plus child epics bob-cli-go.4.1 and
        bob-cli-go.10.1 were verified against source and commits (`8d2eb09f` through
        `a6c55cbb`).
      - The follow-up triage recorded on the epic's latest landing note: bob-cli-8a,
        bob-cli-gd, and bob-cli-bo corroborated, and the declined or resolved proposals.
      - The integration review: commits from bob-cli-gj, -gl, -gm, -h0, gcal,
        attachments, and periods since the epic began needed no transfer changes. The h0
        move-consequence builder covers project-note `^`/`:` moves only.
      - This tale's `> +` fix and its verification results.

      Never use `--force`. If the close is refused, fix the cause it names and close
      again.

   3. Run `just symvision` if the recipe exists. This repo currently has none, so check
      with `just --list` and say so in the final response if it is absent.
   4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
      file, `plan:202611/task_move_merge_split.md`. Its path is the PLAN line of
      `sase bead read bob-cli-go`. That file lives in the plans sidecar, so open it with
      `/sase_repo` before editing if your workspace requires it.
   5. bob-cli-go has no `parent_bead`, so there is no parent phase or plan to close.
