---
tier: epic
title: Show the full Pomodoro block in the Mac capture preview
goal: 'Whenever a capture touches, creates, or reports a Pomodoro, the Bob Mac Capture
  live preview shows that Pomodoro in full: its headline and every nested child line,
  exactly as Bob will write it, with the lines the capture changes clearly marked.
  Bob computes the blocks and the diff. The app only decodes and renders them in one
  consistent, polished block view.

  '
phases:
- id: blocks_tracker
  title: Emit batch-level pomodoro_blocks from bob capture
  depends_on: []
  size: medium
  description: 'blocks_tracker: add the block-range and depth helpers, the per-item
    block-ref tracker (ref resolution, auto-detection, cross-item forwarding, line
    diff), and the top-level `pomodoro_blocks` JSON. Add explicit refs for adjust,
    shift, whole-item and named starts, and whole-item close (closed and next). Add
    unit and CLI tests and the docs/capture.md contract.

    '
- id: blocks_refs
  title: Report every remaining Pomodoro-touching capture
  depends_on:
  - blocks_tracker
  size: medium
  description: 'blocks_refs: add explicit refs for link and task starts, close link
    and task forms, Pomodoro task and link captures, and Ensure Next, including mention-only
    cases. Prove that toggles, Pomodoro notes, and project notes are auto-detected.
    Add a coverage-invariant test helper used by every Pomodoro CLI test family.

    '
- id: mac_block_model
  title: Decode and present Pomodoro blocks in CaptureCore
  depends_on:
  - blocks_tracker
  size: medium
  description: 'mac_block_model: add tolerant `pomodoro_blocks` decoding, a pure block
    presentation, a display-only line tokenizer, and the covers-preview-lines rule.
    Add real-bob fixtures for adjust, shift, start, named start, close, and a chain,
    plus CaptureCore tests. Commit and get macOS CI green.

    '
- id: mac_block_view
  title: Render the Pomodoro block view in the preview pane
  depends_on:
  - mac_block_model
  - blocks_refs
  size: medium
  description: 'mac_block_view: add PomodoroBlockView (status rail, diff gutter, indent
    guides, syntax tint) and render blocks after the item stack. Drop verbatim lines
    the blocks already cover, dim blocks with a pending close card, and remove the
    duplicated path in the destination summary. Add link, move, and note fixtures,
    fake-bob routes, model, height, and render tests, the README, and green macOS
    CI.'
proposed_by: bbugyi200.apollo.3c
create_time: 2026-09-30 07:52:26
status: done
bead_id: bob-cli-2r
---

- **PROMPT:** [prompts/202609/pomodoro_full_block_preview.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/pomodoro_full_block_preview.md)
- **BEAD:** [bob-cli-2r](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2r/README.md)

# Problem

The Mac capture panel's live preview (the pane at the bottom of the window) often shows
only a Pomodoro's headline. Typing `+5` against today's ledger previews:

```text
2026/20260930.md  toggled  pomodoro_adjust
0620–0710 (50m) → 0620–0735 (75m), +25m  CLEANUP · line 27
- [ ] (**0620-0735** [t:: 75m]) — CLEANUP
2026/20260930.md
```

The real Pomodoro, however, is a block:

```markdown
- [ ] (**0620-0735** [t:: 75m]) — CLEANUP
  - [[sase#^re-launch-failed]]
  - [[sase#^clean-prompt-history]]
  - [[bob#^decision-web]]
```

A Pomodoro is a unit: its headline plus its Task Links, 🍅 marks, strikes, deferred
markers, notes, and nested Work Log lines. Every preview that involves a Pomodoro should
show that whole unit, so the user always sees which session they are acting on and what
it will contain afterwards.

Why only the headline shows today: Bob's capture JSON describes Pomodoros only by
headline line numbers and summary fields. The adjust/shift `task_line` is the headline.
Close and start carry resolved task rows. Link and toggle kinds carry endpoints. None of
them carries the block's child lines. The fix therefore spans both repositories. Bob
computes the blocks, and the app renders them. That split keeps the existing rule that
Bob owns all grammar, ledger, and vault logic and the app only decodes and presents.

# Design

## Principles

- **Full.** Every Pomodoro the capture touches, creates, or reports is shown with every
  line of its block. Nothing is capped, collapsed, or truncated. Long lines wrap. The
  window grows to the screen limit, and after that the existing auxiliary region
  scrolls.
- **Truthful.** Lines are the exact bytes Bob will write, in the after-state. Bob
  computes a per-line diff against the before-state, and the app never re-derives ledger
  content. Styling is display-only: tinting never adds, drops, or reorders characters.
- **Once.** Blocks are batch-level. A chain such as `+2 =x` or `=x =` touches the same
  Pomodoro several times, but it is shown once, in its final state, with the cumulative
  diff against the ledger before the capture.
- **Consistent.** One block view is used for every Pomodoro kind: adjust, shift, start,
  named start, close, link, task, Ensure Next, toggle, note, and project note. The
  existing cards (close, start, link, toggle, and the adjust/shift rows) stay unchanged
  as the "what this step does" summary. The block view below them answers "what the
  Pomodoro will look like".

## Contract: top-level `pomodoro_blocks`

`bob capture -f json` (dry run and real run alike) gains one additive, batch-level
top-level key. It sits next to `plan_budget`, never appears per item or inside
`captures[]`, and is omitted when empty:

```json
"pomodoro_blocks": [
  {
    "relative_target": "2026/20260930.md",
    "line": 27,
    "name": "CLEANUP",
    "time_range": "0620-0735",
    "status": "running",
    "created": false,
    "roles": ["adjusted"],
    "lines": [
      {"text": "- [ ] (**0620-0735** [t:: 75m]) — CLEANUP", "depth": 0,
       "change": "changed", "before": "- [ ] (**0620-0710** [t:: 50m]) — CLEANUP"},
      {"text": "\t- [[sase#^re-launch-failed]]", "depth": 1, "change": "unchanged"},
      {"text": "\t- [[sase#^clean-prompt-history]]", "depth": 1, "change": "unchanged"},
      {"text": "\t- [[bob#^decision-web]]", "depth": 1, "change": "unchanged"}
    ]
  }
]
```

Field rules:

- `relative_target`: the day file relative to the vault, produced by
  `capture_pomodoros::relative_day_file`.
- `line`: the 1-based headline line in the **final** staged day file.
- `name` and `time_range`: taken from `capture_pomodoros::scan` of the final text at
  that line (`time_range` is plain `HHMM-HHMM`). Each is omitted when the entry has
  none.
- `status`:
  - `completed` when the scan state is `Completed`;
  - `running` when the entry is open with a time range;
  - `queued` when it is open without a time range.
- `created`: true when the entry did not exist before the batch.
- `roles`: an informational, deduplicated list in first-touch order. The vocabulary is
  `adjusted`, `shifted`, `started`, `closed`, `next`, `linked`, `unlinked`, and
  `changed` (auto-detected). The app does not depend on it.
- `lines`: the block in document order, with removed lines interleaved where they used
  to be:
  - `text` is the verbatim line with no terminator.
  - `depth` is the nesting level relative to the headline:
    - the headline is 0;
    - a list item whose nearest shallower list-item parent (found with
      `nearest_shallower_list_item_parent`) is the headline is 1;
    - its children are 2, and so on;
    - a non-list continuation line gets its parent list item's depth + 1;
    - a blank line is 0.
    - Removed lines get their depth from the before-block.
  - `change` is `unchanged`, `added`, `removed`, or `changed`.
  - `before` holds the old text and appears only on `changed`.
- A created block's lines are all `added`.
- Block extent: the headline plus every following line up to, but not including, the
  first non-blank zero-indent line or the end of the `## Pomodoros` section. Trailing
  blank lines are trimmed. This is the same boundary `direct_child_count` walks.
- Order: blocks appear in first-touch order across the batch. Within one item, explicit
  refs come first in the order the planner pushed them, followed by auto-detected blocks
  in document order.

The `bob capture` human output does not change.

## How Bob computes blocks

**Refs from planners.** Every planner that selects, creates, moves, or reports a
Pomodoro pushes `PomodoroBlockRef { role, before, after }` onto its
`PlannedCaptureItem`. The field is not serialized; `PlannedCaptureItem` has no derives.

- `before` is `At(i)` (the 0-based headline index in that item's pre-state day text),
  `Resolve`, or `Created`.
- `after` is `Some(i)` (the 0-based index in that item's post-state text) or `None`,
  meaning resolve it.

A planner pushes what it already knows and leaves the rest to resolution.

**Tracker in the batch loop.** `plan_capture_batch` (`src/native/capture/plan.rs`) owns
a `PomodoroBlockTracker` for `pomodoro::day_file_for(&request.bob_dir)`.

Before each item, peek at the planner's current day text **without** loading it. Add a
non-loading accessor on `CaptureBatchPlanner`; `current_contents` would add a new error
path for batches that never touch the day file.

After the item, if the day file is now loaded, take:

- `pre`: the peeked text, or the file's `original` if it was first loaded during this
  item;
- `post`: `current`.

Then:

1. **Line map.** When `pre != post`, run
   `similar::capture_diff_slices(Algorithm::Myers, pre_lines, post_lines)`. `similar` is
   already a dependency. Build `old_to_new` and `new_to_old` maps from the `Equal` ops.
2. **Resolve refs.** Fill a `Resolve` before with `new_to_old[after]`. Fill a `None`
   after with `old_to_new[before]`. Both are valid because a headline that a ref does
   not describe as changed stays byte-identical. Every resolved `after` must be an entry
   headline in the post scan, and every `At` before must be one in the pre scan.
   Otherwise, debug-assert and drop the ref.
3. **Auto-detect unreported touches.** For each entry in the post scan that no resolved
   ref claims, check whether its block contains a line with no `new_to_old` mapping, or
   whether its mapped pre-block lost a line. If so, push an implicit `changed` ref:
   - `before = At(new_to_old[headline])` when the headline is mapped;
   - `Created` when every line of the block is unmapped.

   A modified headline with no ref is a planner bug. Handle it as follows:
   - `debug_assert!` so that tests catch it;
   - in release builds, emit the block with every line marked `unchanged`, never a
     guessed diff.

   The same check covers a pre-entry that vanished without a ref.

4. **Match and forward.** Each tracked block keeps its current headline index.
   - When a resolved ref's `before == At(current)`, that item touched the tracked block:
     set `current = after` and append the ref's role.
   - Otherwise map `current` through `old_to_new`. A tracked headline that cannot be
     mapped is the same planner bug as above: debug-assert and drop it.
   - A ref that matches no tracked block starts one. Store `before_lines`, the block
     extracted from `pre` at `before` (nothing when `Created`), and `created`.
   - Two refs of one item with the same `after`, such as `AlreadyCurrent` source ==
     destination, merge into one block.
5. **Finish.** After the loop, handle each tracked block in first-touch order:
   - Re-extract the block from the final day text at `current`.
   - Read the scan metadata. If there is no entry at that line, debug-assert and drop
     the block.
   - Diff `before_lines` against the final lines with the same Myers call, and pair each
     `Replace` op: the first `min(old, new)` lines become `changed` with `before`, the
     leftover old lines become `removed`, and the leftover new lines become `added`. If
     a `similar` version yields no `Replace` op, group adjacent `Delete` + `Insert` runs
     the same way.
   - Store the result on `PlannedCaptureBatch`. `capture()` passes it through
     `CaptureResult::from_items`, mirroring how `plan_budget` flows.

Block content cannot change after a Pomodoro's last touch, because any later change
would itself be a touch. Only its position can move, and forwarding tracks the position,
so the final extraction is exact. Dry-run and real-run JSON stay identical because both
use the same planner.

## The block view (Mac)

The preview pane renders `success.pomodoroBlocks` once, after the item stack (after the
last item and before the "Clipboard markers stay literal" caption), with 10 pt between
blocks. Every block uses the same view:

```text
 ▶ Running · line 27                                            (caption row)
╭──────────────────────────────────────────────────────────────╮
┃• - [ ] (**0620-0735** [t:: 75m]) — CLEANUP                   ┃ ← accent-tinted row
┃    │ - [[sase#^re-launch-failed]]                             ┃
┃    │ - [[sase#^clean-prompt-history]]                         ┃
┃    │ - [[bob#^decision-web]]                                  ┃
╰──────────────────────────────────────────────────────────────╯
```

Closing the docs worked example (`=x`) shows two blocks:

```text
 ✓ Completed · line 4
┃• - [x] (**0920-0940** [t:: 20m]) — CAPTURE        (changed, accent)
┃•   │ - 🍅 [[bob#^capture-stop]]                    (changed, accent)
┃    │   │ - Designed the `=x` grammar
┃    │   │   │ - chose `x` for done
┃    │   │ - Wrote the plan
┃−   │ - [[bob#^web-capture]]#                       (removed: red, struck, dimmed)
┃    │ - ~~[[sase#^axe-restart]]~~
┃    │   │ - Restarted axe
┃    │ - quick note
 ◌ Queued · line 13   [New]
┃+ - [ ] () — CAPTURE                                (added, green)
┃+   │ - [[bob#^capture-stop]]
┃+   │ - [[bob#^web-capture]]
```

Visual specification:

- **Caption row.** A status glyph, then `Running · line N`, `Completed · line N`, or
  `Queued · line N` in `.caption` semibold secondary. Created blocks add a pink capsule:
  **New** on a dry run, **Created** once committed. It is the start card's
  `createdBadgeText` capsule style.

  | Status    | Glyph                   | Tint                                                     |
  | --------- | ----------------------- | -------------------------------------------------------- |
  | Running   | `play.circle.fill`      | `CaptureEditorPalette.color(for: .pomodoroStart)` (pink) |
  | Completed | `checkmark.circle.fill` | green                                                    |
  | Queued    | `circle.dashed`         | secondary                                                |

- **Card.**
  - Inset on the pane's thin material with
    `.background(.quinary, in: RoundedRectangle(cornerRadius: 7, style: .continuous))`
    and a 0.5 pt `.separator` hairline border.
  - A 3 pt leading status rail in the status tint, clipped by the rounded shape.
  - 6 pt vertical and 8 pt horizontal padding.
- **Rows.**
  - One row per line, monospaced `.callout`, text selectable.
  - Rows wrap and never truncate: `fixedSize(horizontal: false, vertical: true)`. The
    wrapped text keeps a hanging indent under the content start.
  - Each row has a fixed 12 pt gutter, then `depth` indent steps of 18 pt, then the
    content. The content is the line with its leading whitespace dropped.
  - Every ancestor level draws a 1 pt `.separator` indent guide that spans the full row
    height, so guides join into continuous lines across rows and wrapped lines. Clamp
    depth at 8.
- **Changes.**
  - Every changed row gets a full-width 3 pt-radius background, and its gutter glyph is
    `.caption` bold.

  | Change    | Background                       | Gutter glyph | Other treatment                                 |
  | --------- | -------------------------------- | ------------ | ----------------------------------------------- |
  | added     | green, 10% opacity               | `+`, green   | none                                            |
  | changed   | `Color.accentColor`, 12% opacity | `•`, accent  | `.help("Was: <before content>")`                |
  | removed   | red, 8% opacity                  | `−`, red     | content struck through, secondary, 0.55 opacity |
  | unchanged | none                             | none         | none                                            |
  - With `colorSchemeContrast == .increased`, raise the tints to 22% and draw the
    hairline at full `.separator`.

- **Syntax tint.** This is display-only, and every token keeps its bytes.

  | Tokens                                                                  | Styling                                                                                                            |
  | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
  | List dash, `[`/`]`, `(`/`)`, `**`, `~~`, backticks, deferred `#` suffix | `.tertiary`                                                                                                        |
  | Checkbox symbol                                                         | `x` green, `/` orange, `*` blue, space `.tertiary`, other secondary                                                |
  | Headline time range                                                     | pink, semibold                                                                                                     |
  | `[t:: 75m]` and any `[key:: value]` inline field                        | secondary                                                                                                          |
  | Headline name after `— `                                                | primary, semibold                                                                                                  |
  | Wikilinks (`[[`/`![[`/`]]`, target, `#Heading`, `#^id`, `\|alias`)      | `CaptureEditorPalette` wikilink colors: delimiter, target (accent), heading (purple), block (indigo), alias (teal) |
  | `~~…~~` inner text                                                      | struck through, secondary                                                                                          |
  | Inline code                                                             | faint `Color.secondary.opacity(0.12)` background                                                                   |
  | 🍅 and plain text                                                       | primary                                                                                                            |

- **Behavior.**
  - No animation; the pane updates on every keystroke.
  - Blocks dim to 0.6 together with the close card while `closePendingText` is set.
  - When `pomodoro_blocks` is absent (an older Bob), the pane renders exactly as it does
    today.
  - The standard item (adjust, shift, Pomodoro note) omits its verbatim
    `previewBlockLines` when the blocks already cover them. That keeps the headline from
    showing twice.
- **Accessibility.** Each block is one element. Its label names the Pomodoro (or
  "Unnamed"), the status, the line, and New when created, then reads each line with an
  `added` / `removed` / `changed from …` prefix where one applies.

Why this design: verbatim Markdown matches the pane's existing "shows every block Bob
will write" promise and the Obsidian source the user reads daily. Tinting that reuses
the editor palette ties the `+5` the user typed (pink) to the time range it changes
(pink). The diff gutter makes the effect of a capture legible at a glance: the user sees
which line changes, what arrives, and what leaves. Showing each block once, at batch
level, keeps chains calm and truthful.

# blocks_tracker

Work in bob-cli.

- **Block helpers.**
  - `capture_pomodoros.rs`: add
    `pub(crate) fn pomodoro_block_range(lines, headline_index, section_end) -> Range<usize>`
    implementing the extent rule. It shares its boundary walk with `direct_child_count`
    but keeps fenced lines and interior blanks verbatim.
  - New `src/native/capture/pomodoro_blocks.rs`:
    - depth computation via `nearest_shallower_list_item_parent`, so mixed tabs and
      spaces work;
    - `PomodoroBlockRef`, `PomodoroBlockRole`, and the tracker;
    - the diff/pairing code;
    - the serialized `PomodoroBlockJson` / `PomodoroBlockLineJson`, with
      `skip_serializing_if` on `name`, `time_range`, and `before`, and snake_case enums.
- **Plumbing.**
  - Add `pomodoro_refs: Vec<PomodoroBlockRef>` to `PlannedCaptureItem`; set it to empty
    at every construction site that has nothing to report.
  - Add the non-loading `CaptureBatchPlanner` peek.
  - Add `pomodoro_blocks` to `PlannedCaptureBatch`, to `CaptureResult`
    (`#[serde(skip_serializing_if = "Vec::is_empty")]`, placed after `plan_budget`), and
    as a `from_items` parameter.
- **Explicit refs in this phase.** The indices are those the explorer confirmed at
  planning time; verify them against the current code. Lines reported in JSON are
  1-based; locals are 0-based.

  | Planner                                                    | Role       | before                                                                                                                                              | after                             |
  | ---------------------------------------------------------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
  | `plan_pomodoro_adjust_item`                                | `adjusted` | `At(session.line - 1)`                                                                                                                              | `Some(session.line - 1)`          |
  | `plan_pomodoro_shift_item`                                 | `shifted`  | `At(session.line - 1)`                                                                                                                              | `Some(session.line - 1)`          |
  | `plan_pomodoro_start_item`                                 | `started`  | `At(index)` of `next_future_pomodoro`                                                                                                               | `Some(moved_index)`               |
  | `plan_named_pomodoro_start_item`, `Found`                  | `started`  | `At(index)`; add it to the tuple the `match` returns                                                                                                | `Some(moved_index)`               |
  | `plan_named_pomodoro_start_item`, missing / completed-only | `started`  | `Created`                                                                                                                                           | `Some(moved_index)`               |
  | `plan_pomodoro_close_item`, the closed entry               | `closed`   | `At(running.line - 1)`                                                                                                                              | `Some(summary.pomodoro_line - 1)` |
  | `plan_pomodoro_close_item`, `next_pomodoro` when present   | `next`     | `Created` when `created`, else `Resolve`. Push it even when the next entry is unchanged: a Pomodoro the card names is a Pomodoro the preview shows. | `Some(next.line - 1)`             |

  One close edge case needs care. When a linked task lives in the day file above
  `## Pomodoros`, close inserts Work Log lines there, and that can leave `running.line`
  and `next.line` stale against the final staged text. Check the refs' after indices
  against the post-state text. Where they can drift, locate the entry the way
  `retire_closed_embeds` does (by its rewritten headline text), and add a CLI test for
  that layout.

- **Tests.**
  - Unit tests in the new module:
    - range: tabs, two-space, mixed indentation, interior and trailing blanks, a fenced
      block, the section end, and the last entry;
    - depth, including continuation lines;
    - pairing: replace becomes changed with `before`, pure insert, pure delete, uneven
      replace, and a created block;
    - resolution, auto-detect of an unreported child insert, and forwarding across an
      item that inserts lines above a tracked block;
    - the debug-assert paths, using `#[should_panic]` under `debug_assertions`.
  - CLI tests in the matching `tests/cli/capture/*.rs` files, pinning the full
    `pomodoro_blocks` value with `assert_eq!(…, json!(…))` on:
    - the screenshot ledger (`CLEANUP` 0620-0710 with the three links, then
      `- [ ] () — GTD` with `[[#^gtd]]`, at `BOB_NOW=2026-09-30 07:20:00`) under `+5`;
    - the same ledger under `++1`;
    - `=` starting a placeholder that has children and moves (block moved, headline
      changed, children unchanged, new `line`);
    - a named start that creates its entry (`created: true`, all lines added);
    - the `docs/capture.md` close worked example (closed block plus a created next
      block, as in the mock above);
    - `=x` with an unchanged existing next (all lines `unchanged`);
    - `+2 =x` (one `CLEANUP` block with the cumulative diff against the original, then
      `GTD`);
    - `=x =` (the closed `CLEANUP` block, then one `GTD` block with roles
      `["next", "started"]`).

    Each of these also asserts that dry-run JSON equals real-run JSON except for
    `dry_run`.

  - Add `pomodoro_blocks` to the absent-key list in `json_success_shape_is_stable`.

- **Docs.** In `docs/capture.md`, add a "Pomodoro blocks" subsection under "Input,
  stdin, and JSON output" covering the contract, rules, and example above, and
  cross-link it from the adjust, shift, start, and close JSON paragraphs. While there,
  add the missing `pomodoro_shift` and `pomodoro_start` entries to that section's `kind`
  list.
- **Verify.** Run `just fmt`, `just lint`, and `just test`. If `just lint` fails only on
  a pre-existing issue this phase did not touch, record it on the bead instead of fixing
  it.

# blocks_refs

Work in bob-cli. Add explicit refs (with the same caveats about verifying indices) for:

- **`plan_pomodoro_start`** (link and task starts, `pomodoro_start.rs`): thread the pre
  index out of each branch.

  | Branch                         | before      | after          |
  | ------------------------------ | ----------- | -------------- |
  | named `Found`                  | `At(index)` | `moved_index`  |
  | named missing / completed-only | `Created`   | `created_line` |
  | next-future                    | `At(index)` | `moved_index`  |
  | fallback create                | `Created`   | `created_line` |
  - The role is `started`.
  - Add `linked` for the same entry when a Task Link is also appended.

- **Close link and task forms** (`plan_pomodoro_close_link_item`,
  `plan_pomodoro_close_task_item`):
  - `closed`:
    - before = `At` of the **pre-link** running index. The link form has it as
      `destination.line - 1`. The task form discards `_dest`, so keep it or recompute it
      with `find_running_pomodoro`.
    - after = `summary.pomodoro_line - 1`.
  - `unlinked` for a moved source: before = `At(source.line - 1)`; after = `None`
    (resolve it).
  - `next` as in the whole-item close.
- **Pomodoro task and link captures** (`plan_capture_with_pomodoro_link`,
  `plan_pomodoro_link_capture`, `plan_pomodoro_link_with_start`, and
  `plan_pomodoro_link_ledger`):
  - Destination `linked`:
    - after = `destination.line - 1`, which is post-state on these paths;
    - before = `Created` when the entry was created, otherwise `Resolve`, or `At(...)`
      when the destination also started and moved.
  - Source `unlinked`: before = `At(source.line - 1)` (pre-state); after = `None`.
  - Mention-only cases still push their ref, so the block shows with every line
    `unchanged`:
    - `pomodoro_already_linked` destinations;
    - `AlreadyCurrent`;
    - a queued entry that is itself the destination.
- **Ensure Next** (`ensure_next.rs`): the destination `linked` ref (from
  `destination_endpoint_after_move`, with `Created` when it creates) and the source
  `unlinked` ref, with after resolved.
- **Auto-detected paths** (two-way toggles, including `remove_matching_links` cleanup
  across several entries, Pomodoro notes, and project notes): do not plumb refs here.
  Add CLI tests proving that each yields the right blocks:
  - toggle insert into an existing entry;
  - toggle insert that creates a named entry (`created: true`);
  - Open-direction removal from two entries (two blocks, one removed line each);
  - `quick note #` under the running entry, and under the last completed entry;
  - a project note linking two `:` tasks into one Pomodoro.
- **Coverage invariant.** Add a test helper, for example
  `assert_pomodoro_blocks_cover_changes(original_day, final_day, json)`, in the capture
  CLI test support module. Run a Myers line diff of the original against the final day
  file and check both directions:
  - every inserted or changed line inside the final `## Pomodoros` section belongs to a
    block listed in `pomodoro_blocks`, matched by final headline `line`;
  - every deleted pre-state line belongs to an entry whose final headline is listed.

  Call it from at least one test in every Pomodoro test family:

  | Test families                                                                                |
  | -------------------------------------------------------------------------------------------- |
  | `pomodoro_adjust`, `pomodoro_shift`                                                          |
  | `pomodoro_start`, `pomodoro_start_named`, `pomodoro_whole_item`                              |
  | `pomodoro_close`, `pomodoro_close_selection`, `pomodoro_chain`                               |
  | `pomodoro_link`, `ensure_next`, `task_toggle`, `project_note`, `sub_bullet` (Pomodoro notes) |

- **Pinned JSON.** Pin the full `pomodoro_blocks` JSON for:
  - `Write outline @sase:outline` into the running entry;
  - a link move `^sase:x#gtd` (destination first, then the source with a removed line);
  - a link that creates a named Pomodoro.
- **Docs.** In `docs/capture.md`, note under the link, toggle, Ensure Next, Pomodoro
  note, and project note JSON paragraphs that their Pomodoros appear in
  `pomodoro_blocks`.
- **Verify.** Run `just fmt`, `just lint`, and `just test`, with the same lint caveat.

# mac_block_model

Work in bob-mac-capture. Open it with `/sase_repo`. The linked name fails on hosts
without `~/projects/github/bobs-org/bob-mac-capture`, so open the GitHub checkout there:

```sh
sase repo open gh:bobs-org/bob-mac-capture -r "Decode and present Pomodoro blocks"
```

Use only the printed path, and read its `AGENTS.md` if it names one. This phase changes
only `CaptureCore` and its tests.

- **`Sources/CaptureCore/CaptureModels.swift`.**
  - Add public `CapturePomodoroBlock`:
    - fields: `relativeTarget`, `line`, `name?`, `timeRange?`, `status`, `created`,
      `roles`, `lines`;
    - `status` is an enum `running` / `queued` / `completed`, and unknown values decode
      to `.other`;
    - it has a tolerant `init(from:)` with `decodeIfPresent` defaults, like
      `PomodoroShiftSummary`.
  - Add public `CapturePomodoroBlock.Line`:
    - fields: `text`, `depth`, `change`, `before?`;
    - `change` is `unchanged` / `added` / `removed` / `changed`, and unknown values
      decode to `.unchanged`;
    - the decoder is tolerant too.
  - Add `pomodoroBlocks: [CapturePomodoroBlock]` on `CaptureCommandSuccess`, decoded
    with `(try? container.decodeIfPresent(...)) ?? []`, so a malformed value can never
    fail the capture decode. Add the coding key and a memberwise-init parameter that
    defaults to `[]` last, so existing call sites compile unchanged.
- **New `Sources/CaptureCore/CapturePomodoroBlockPresentation.swift`** (pure, public):
  - `init(block:dryRun:)`;
  - `status`, `statusSymbolName`, and `captionText` (`Running · line 27`);
  - `badgeText`: `New` on a dry run, `Created` when committed, nil unless `created`;
  - `rows`, each with `content` (leading spaces and tabs dropped), `depth` (clamped
    0…8), `change`, `beforeContent`, `isHeadline` (`depth == 0 && !content.isEmpty`),
    and `tokens`;
  - `accessibilitySummary`, following the design above;
  - `static func covers(_ capture: CaptureCommandSuccess, blocks: [CapturePomodoroBlock]) -> Bool`,
    true when blocks is non-empty and every `capture.previewBlockLines` entry equals the
    `text` of a non-removed line in a block with the same `relativeTarget`.
- **New `Sources/CaptureCore/CapturePomodoroLineTokens.swift`** (pure, public):
  - `tokenize(_ content: String, isHeadline: Bool) -> [Token]`, where `Token` is `text`,
    `role`, `struck`, and the roles are `syntax`, `checkbox(Character)`, `timeRange`,
    `field`, `name`, `wikilinkDelimiter`, `wikilinkTarget`, `wikilinkHeading`,
    `wikilinkBlock`, `wikilinkAlias`, `code`, and `text`.
  - It implements the tinting rules in the design section, with one left-to-right scan
    and no regex backtracking over whole lines.
  - A legacy or unrecognized headline falls back to `text` after the checkbox.
  - The hard invariant is `tokens.map(\.text).joined() == content`.
- **Fixtures.**
  - Build `bob` from bob-cli at a revision that contains `blocks_tracker`: use
    `cargo build` in your bob-cli workspace after pulling. Generate real JSON against
    sandbox vaults with `BOB_DIR`, `BOB_DAY_FILE`, and `BOB_NOW`.
  - Commit the fixtures with `"dry_run": false` as the existing fixtures do; fake-bob
    flips it for `--dry-run`.
  - Document the vault and exact commands in a header comment, as
    `BlockIDDecodingTests.swift` does.

  | Fixture                                    | Scenario                                                              |
  | ------------------------------------------ | --------------------------------------------------------------------- |
  | `pomodoro-adjust-blocks.json`              | the screenshot ledger above, `+5` at 07:20 on 2026-09-30              |
  | `pomodoro-shift-blocks.json`               | the same ledger, `++1`                                                |
  | `pomodoro-start-blocks.json`               | `=` against a ledger whose running entry is closed, starting `GTD`    |
  | `pomodoro-start-named-blocks-created.json` | a named start that creates its entry                                  |
  | `pomodoro-close-blocks.json`               | the `docs/capture.md` worked example at `BOB_NOW=2026-09-28 09:37:00` |
  | `pomodoro-chain-blocks.json`               | `+2 =x` on the screenshot ledger                                      |

- **Tests.**
  - `CaptureModelTests`: decoding of each fixture; decoding with the key absent (the
    existing fixtures) and malformed, which yields empty blocks and still succeeds;
    unknown status and change values.
  - New `CapturePomodoroBlockPresentationTests`: captions and glyphs per status, the
    New/Created badge, rows and depth clamp, the before text, accessibility, and
    `covers` (true for adjust and shift; false for a `pomodoro_task` item, whose line
    lives in the route note; false without blocks).
  - New `CapturePomodoroLineTokensTests`: the concatenation invariant over every line of
    every new fixture, plus targeted cases:
    - running, placeholder, completed, and legacy headlines;
    - plain, 🍅, deferred `]]#`, struck `~~[[…]]~~`, embed `![[…#^id]]`, and dropped
      `~[[…]]` links;
    - `[[#^gtd]]`, `[[x#Heading]]`, `[[x|alias]]`;
    - backtick code;
    - a note containing a bare `[[note]]` link.
- **Commit, then CI.** This plan explicitly instructs you to commit the bob-mac-capture
  changes with `/sase_git_commit` before waiting on CI, because macOS CI only runs on
  the pushed commit and this Linux host has no Apple Swift toolchain.
  - Use subject `feat(capture): decode and present Pomodoro blocks`. If `$SASE_BEAD_ID`
    is set, pass `-B keep`.
  - Then wait with `/sase_monitor` on the `CI` workflow run for that SHA
    (`gh run list --commit <sha> --limit 1`, then `gh run watch <id>`), with a timeout
    of at least 20 minutes.
  - If the run is red, read `gh run view <id> --log-failed`, fix forward, commit again,
    and watch again.
  - The phase is done only when the latest run for its commit is green.

# mac_block_view

Work in bob-mac-capture, opened as in `mac_block_model`.

- **New `Sources/BobMacCapture/PomodoroBlockView.swift`.**
  - It implements the visual specification in the design section exactly: caption row,
    card, status rail, gutter, indent guides, change tints, and syntax tint.
  - Token roles map to colors through `CaptureEditorPalette`. Add the checkbox-symbol
    color helper there, next to `taskStatus`.
  - Honor `colorSchemeContrast`.
  - Use `.textSelection(.enabled)`.
  - Add no animation.
- **`CapturePanelView.swift` `PreviewPane`.**
  - In `previewContent`, after the items `ForEach`, render
    `ForEach(success.pomodoroBlocks)` with `PomodoroBlockView`, 10 pt spacing, and 4 pt
    extra top padding. Wrap it so it dims to 0.6 while `model.closePendingText != nil`,
    matching `closePreviewItem`'s `isPending` treatment.
  - Pass the blocks to `previewItem` → `standardPreviewItem`. When
    `CapturePomodoroBlockPresentation.covers(success, blocks:)` is true, omit the
    verbatim `previewBlockLines` stack.
  - VoiceOver must still announce the item summary once. Move the item's accessibility
    label to its header row, or keep the label without the duplicated lines.
- **`CapturePanelModel.captureSummary`.** For one capture, drop the parenthetical
  `(relative_target)` when it equals the display label. Today it reads
  `Preview → 2026/20260930.md (2026/20260930.md): …`; after the fix it reads
  `Preview → 2026/20260930.md: …`. Update the tests that pin the old string.
- **Fixtures.** Generate these from bob-cli at a revision that contains `blocks_refs`,
  with the same method as in `mac_block_model`:

  | Fixture                          | Scenario                                             |
  | -------------------------------- | ---------------------------------------------------- |
  | `pomodoro-task-blocks.json`      | `Write outline @sase:outline` into the running entry |
  | `pomodoro-link-move-blocks.json` | a queued link moved to `#gtd`                        |
  | `pomodoro-note-blocks.json`      | `quick note #`                                       |
  | `pomodoro-toggle-blocks.json`    | a two-way toggle insert                              |

  In `Tests/Fixtures/fake-bob`:
  - Route each new fixture's draft to it, including the `mac_block_model` fixtures.
  - Prefer drafts that no existing route uses. Where an existing route must change,
    update the tests that pin its old rendering in the same commit.

- **Tests.**
  - `CapturePanelModelTests`:
    - a live preview of `+5` yields one block and the deduplicated destination summary;
    - a failed dry run clears the blocks along with the card;
    - an older-Bob fixture renders no blocks.
  - `CapturePreviewFullHeightTests`: hosting `PreviewPane` with
    `pomodoro-close-blocks.json` at widths 724 and 620 is taller than the same success
    with `pomodoroBlocks` emptied, and the loading-hold behaviour still holds.
  - Extend `CapturePickerDesignTests`, or add `PomodoroBlockDesignTests`, with a
    `BOB_MAC_CAPTURE_RENDER_DIR`-gated PNG render.
    - Render `PreviewPane` for the adjust, close, link-move, created-named-start, and
      chain fixtures.
    - Use light and dark appearances at widths 760 and 620, at scale 2.
    - Skip the test when the variable is unset.
    - Follow `testRenderPickerCardStatesToPNG`.
  - Visual review is best-effort:
    - If `ssh -o ConnectTimeout=5 mac true` succeeds, you may clone the pushed commit
      into a fresh temporary directory on that Mac, run just that test with
      `BOB_MAC_CAPTURE_RENDER_DIR` set, copy the PNGs back, inspect them, and iterate on
      spacing, contrast, wrapping, and guide alignment.
    - Touch nothing else on that machine, and delete the temporary clone afterwards.
    - If the Mac is unreachable or lacks a toolchain, skip the review and say so on the
      bead.
- **`README.md`.**
  - In the "Preview shows every block Bob will write" paragraph, describe the block
    view:
    - batch-level;
    - after the items;
    - the status caption and New badge;
    - diff gutter colors;
    - verbatim tinted lines with indent guides;
    - no truncation;
    - dimming with a pending close;
    - dropping covered verbatim lines.
  - In the older-Bob fallback paragraph, add that an older Bob that omits
    `pomodoro_blocks` previews exactly as before.
  - Mention the render test and its environment variable.
- **Commit and CI.** Follow the same commit-then-CI loop as `mac_block_model`, with
  subject `feat(capture): show full Pomodoro blocks in the preview`. The phase is done
  only when the latest macOS run for its commit is green.

# Non-goals

- No change to `bob capture` human output, notifications, completion lists, the Name
  Pomodoro prompt, or the explicit Preview button's status text.
- No redesign of the close, start, link, or toggle cards. They keep their rows, caps,
  and wording. The blocks add the full Pomodoro beneath them.
- No collapse, expand, or settings toggle. The full block is always shown.
- No new per-item JSON inside the existing nested objects. `pomodoro_close` and the
  start lineup rows are pinned byte for byte.
- No changes to bob-plugins or Obsidian.

# Acceptance

- `+5` on the screenshot ledger previews the whole `CLEANUP` block: the changed headline
  is accent-marked and the three Task Links are below it. The verbatim headline is no
  longer printed twice, and the summary reads `Preview → 2026/20260930.md: …`.
- Every capture kind that touches, creates, or reports a Pomodoro shows each such
  Pomodoro exactly once, in full, in its final state, with correct added, removed, and
  changed markers against the pre-capture ledger. Chains show cumulative results.
- The coverage invariant holds across every Pomodoro CLI test family. Dry-run JSON
  equals real-run JSON.
- An older Bob without `pomodoro_blocks` previews exactly as today.
- bob-cli `just fmt`, `just lint`, and `just test` pass, apart from any recorded
  pre-existing lint issue. bob-mac-capture macOS CI is green for the landed revision.
