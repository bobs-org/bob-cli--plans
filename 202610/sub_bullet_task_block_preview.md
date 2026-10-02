---
tier: epic
title: Show the full parent task and its diff when capturing a sub-bullet
goal: 'When a draft adds a sub-bullet under an existing task (`@route+block-id`, with
  or without `#section`, picker task refs, and global `@@route+block-id` batches),
  the Bob Mac Capture preview shows the whole parent task as a card: the task line
  and every line of its block, exactly as Bob will write them, with the new lines
  marked as added. Bob computes the block and the diff. The app decodes and renders
  it with the same diff card the Pomodoro blocks use.

  '
phases:
- id: task_blocks_contract
  title: Emit batch-level task_blocks from bob capture
  depends_on: []
  size: medium
  description: 'task_blocks_contract: move the Pomodoro block diff helpers into a
    shared module, add a batch-level parent-task tracker fed by sub-bullet items,
    and emit the additive top-level `task_blocks` JSON (final-state block, cumulative
    diff against the note before the capture). Add unit and CLI tests and the docs/capture.md
    contract. Pomodoro block JSON stays byte-identical.

    '
- id: mac_task_block_model
  title: Decode and present task blocks in CaptureCore
  depends_on:
  - task_blocks_contract
  size: medium
  description: 'mac_task_block_model: in bob-mac-capture, add tolerant `task_blocks`
    decoding, a shared diff-row model, task-row tokenizing (tags and block IDs), a
    pure CaptureTaskBlockPresentation (caption, status, rows, folding of long quiet
    runs, covers rule, accessibility), and sub-bullet wording. Add real-bob fixtures
    and CaptureCore tests. Commit and get macOS CI green.

    '
- id: mac_task_block_view
  title: Render the parent task card in the preview pane
  depends_on:
  - mac_task_block_model
  size: medium
  description: 'mac_task_block_view: extract a shared BlockDiffCard from PomodoroBlockView,
    add TaskBlockView (status caption, status rail, diff gutter, indent guides, task
    tinting, expandable folds), render task blocks after the items, give covered sub-bullet
    items a compact header, name the parent in the status and summary, and fix the
    stale "Preview failed" footer. Add fake-bob routes, model, height, and render
    tests, the README, and green macOS CI.

    '
proposed_by: bbugyi200.athena.0vb
create_time: 2026-10-02 09:49:31
status: wip
bead_id: bob-cli-3i
---

- **PROMPT:** [prompts/202610/sub_bullet_task_block_preview.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/sub_bullet_task_block_preview.md)
- **BEAD:** [bob-cli-3i](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3i/README.md)

# Problem

Typing `Should reuse as much of PIW sase-core code as possible! @sase+capture` in the
Mac capture panel previews this today:

```text
Preview → sase.md: - Should reuse as much of PIW sase-core code as possible!
┌─────────────────────────────────────────────────────────────┐
│ sase.md  inserted  sub_bullet                               │
│ - Should reuse as much of PIW sase-core code as possible!   │
│ sase.md                                                     │
└─────────────────────────────────────────────────────────────┘
Preview failed                          Stash 2  Discard  Preview  Capture
```

Three things are wrong:

1. **The parent task is invisible.** Nothing says which task the bullet lands under,
   what that task looks like, or where in its block the bullet goes (before the Schedule
   Log? inside `REQUIREMENTS`?). That is the one thing a sub-bullet preview must answer.
2. **The card speaks in wire words.** `inserted sub_bullet` is Bob's raw `placement` and
   `kind`. The path also shows twice.
3. **The footer lies.** `Preview failed` is stale. While the user typed `@sase+captur`,
   the live dry run failed (`no task with block ID ^captur`). The next keystroke
   succeeded, but the live-preview success path only rewrites `statusText` for toggle,
   link, close, and start captures, so the failure text stays. See
   `CapturePanelModel.startLivePreview`.

Why only the bullet shows: a `sub_bullet` result carries `task_line` (the new bullet,
unindented), `parent_line`, `parent_text` (the cleaned description), and the parent
status. It carries none of the parent's block lines, so the app cannot show the task.
The Pomodoro work (epic `bob-cli-2r`, plan `pomodoro_full_block_preview`) solved the
same problem for Pomodoros with a batch-level `pomodoro_blocks` contract and a shared
block view. This plan applies that proven pattern to the parent task of a sub-bullet.
Bob computes the blocks and the app renders them, which keeps the accepted rule
`decisions:mac-capture-is-a-thin-client`: Bob owns grammar, placement, and vault logic,
and the app only decodes and presents.

# Design

## Principles

- **Whole task.** The card shows the parent's task line and every line of its block, as
  `note_tasks` defines the block (deeper-indented lines, with interior blank lines kept
  when deeper lines follow). The bullet is never shown out of context again.
- **Truthful.** Every line is the exact bytes Bob will write, in the after-state, diffed
  by Bob against the note before the capture. Tinting is display-only and never adds,
  drops, or reorders characters.
- **Once.** Task blocks are batch-level. A global `@@sase+capture` draft with three
  items shows one card with three added rows, not three cards.
- **One visual language.** The task card uses the same card as the Pomodoro blocks:
  status rail, `+`/`•`/`−` diff gutter, change washes, indent guides, and monospaced
  verbatim lines. The picker's task status glyphs and colors are reused, so the card
  matches the `@+` task chooser the user just used.
- **Graceful.** An older Bob without `task_blocks` previews exactly as today. A newer
  Bob with an older app is fine, because unknown keys are ignored. Malformed blocks
  decode as empty and never fail the capture decode.

## Contract: top-level `task_blocks`

`bob capture -f json` (dry run and real run alike) gains one additive, batch-level
top-level key next to `pomodoro_blocks`. It never appears per item or inside
`captures[]`, and it is omitted when empty. Against this note:

```markdown
- [/] #task Port capture to PIW sase-core [created:: 2026-09-30] ^capture
  - REQUIREMENTS
    - existing
  - 🗓️ **SCHEDULE LOG**
    - 2026-10-01 moved
```

`Should reuse as much of PIW sase-core code as possible! @sase+capture` reports:

```json
"task_blocks": [
  {
    "relative_target": "sase.md",
    "route": "sase",
    "line": 1,
    "block_id": "capture",
    "text": "Port capture to PIW sase-core",
    "status_symbol": "/",
    "status_name": "In Progress",
    "created": false,
    "roles": ["sub_bullet"],
    "lines": [
      {"text": "- [/] #task Port capture to PIW sase-core [created:: 2026-09-30] ^capture", "depth": 0, "change": "unchanged"},
      {"text": "\t- REQUIREMENTS", "depth": 1, "change": "unchanged"},
      {"text": "\t\t- existing", "depth": 2, "change": "unchanged"},
      {"text": "\t- Should reuse as much of PIW sase-core code as possible!", "depth": 1, "change": "added"},
      {"text": "\t- 🗓️ **SCHEDULE LOG**", "depth": 1, "change": "unchanged"},
      {"text": "\t\t- 2026-10-01 moved", "depth": 2, "change": "unchanged"}
    ]
  }
]
```

Field rules:

- One entry per distinct parent task that the batch's sub-bullet items wrote under, in
  first-touch order, in its final state.
- `relative_target` and `route` name the note. `line` is the 1-based task line in the
  final staged note. `block_id` is the parent's trailing block ID, omitted when it has
  none (a picker task-ref parent).
- `text`, `status_symbol`, and `status_name` describe the parent in the final state,
  with the same meanings as the per-item `parent_text`, `parent_status_symbol`, and
  `parent_status_name`.
- `created` is true only when the parent task did not exist before the batch. This
  happens when an earlier item in the same draft created it, as in `Task @sase^new`
  followed by `note @sase+new`. A created block's lines are all `added`.
- `roles` is informational and deduplicated. Today it is only `"sub_bullet"`; future
  kinds may add values. Clients must not depend on it.
- `lines` uses exactly the `pomodoro_blocks` line schema: `text` (verbatim, no
  terminator), `depth` (relative to the task line, computed by the same helper),
  `change` (`unchanged` / `added` / `removed` / `changed`), and `before` (old text on
  `changed` rows only). Removed lines are interleaved where they used to be.
- The diff is cumulative against the note before the capture. If the same draft also
  rewrites the parent's task line (for example, a task toggle of the same task), the
  task line shows as `changed` with its `before` text.
- Dry-run JSON equals real-run JSON except for `dry_run`. Human output does not change.
  Per-item `parent_*` fields stay as they are.

## How Bob computes task blocks

- **Shared diff helpers.** Move `line_maps`, `pair_lines`, `block_depths`, and the line
  JSON types out of `src/native/capture/pomodoro_blocks.rs` into a new
  `src/native/capture/block_diff.rs`. Both trackers use them. Renaming the Rust types
  (for example, to `BlockLineJson` and `BlockLineChange`) is fine, but the serialized
  `pomodoro_blocks` JSON must stay byte-identical, and the existing Pomodoro tests must
  pass unchanged.
- **Refs.** `PlannedCaptureItem` gains a never-serialized
  `task_block_refs: Vec<TaskBlockRef>`. It is set only by the sub-bullet branch of
  `plan_capture_item`; every other constructor passes an empty vec. A ref names the
  target `PathBuf`, `relative_target`, `route`, the parent's 0-based line in the item's
  post-state text, and the parent `block_id`. The sub-bullet planner always inserts
  below the parent line, so that line is the parent's pre-item `line_index`.
  Debug-assert that the post-state scan has a task there.
- **Tracker.** Add `TaskBlockTracker` in a new `src/native/capture/task_blocks.rs`,
  owned by `plan_capture_batch` next to the `PomodoroBlockTracker`:
  - Before each item, peek the current text of every tracked target with
    `planner.peek_text`.
  - After the item, forward each tracked parent whose note changed through the Myers
    `old_to_new` map. If the task line is unmapped (it was rewritten), re-find the task
    by `block_id` in a `note_tasks` scan of the post text. Failing that, use the nearest
    task line with identical bytes. Failing that, drop the block with a `debug_assert!`.
    Only compute line maps for notes the item actually changed.
  - Merge the item's refs: a ref that matches a tracked parent (same target and line)
    adds its role; otherwise it starts a new tracked parent.
  - `finish`: for each tracked parent, take `(original, final)` from
    `planner.loaded_texts(target)` and scan both with `note_tasks` (read the settings
    once per batch). The final block is the task line through its `block_end`. Map the
    final task line back to the original through a whole-note `new_to_old`. If it is
    unmapped, look up the `block_id` in the original scan: `Missing` means `created`;
    otherwise use the nearest identical task line. If there is still no match, skip the
    block with a `debug_assert!` rather than guess a diff. The before block is the
    original task's block. Then run `pair_lines`.
- **Output.** `PlannedCaptureBatch` and `CaptureResult` carry `task_blocks`, serialized
  with `skip_serializing_if = "Vec::is_empty"` and threaded through
  `CaptureResult::from_items` like `pomodoro_blocks`. `print_human_success` is
  untouched.
- **Performance.** Live preview spawns `bob` on every keystroke. Keep the work linear:
  one scan per tracked note at finish and Myers only on changed notes. A one-line
  insertion into a several-thousand-line note must stay well inside the existing ~5 ms
  budget.

## The task card (Mac)

The preview for the opening example becomes:

```text
Preview → sase.md › ^capture: - Should reuse as much of PIW sase-core code as possible!
┌──────────────────────────────────────────────────────────────────────────┐
│ ↳ Sub-bullet under ^capture                                              │
│ ◐ In Progress · sase.md · line 1                                         │
│ ┃ - [/] #task Port capture to PIW sase-core [created:: 2026-09-30] ^capture
│ ┃   │ - REQUIREMENTS                                                     │
│ ┃   │   │ - existing                                                     │
│ ┃ + │ - Should reuse as much of PIW sase-core code as possible!   (green) │
│ ┃   │ - 🗓️ **SCHEDULE LOG**                                              │
│ ┃   │   │ - 2026-10-01 moved                                             │
└──────────────────────────────────────────────────────────────────────────┘
Ready                                   Stash 2  Discard  Preview  Capture
```

- **Item header (covered sub-bullet items only).** One line: an `arrow.turn.down.right`
  glyph (secondary), then **Sub-bullet** (semibold), then `under ^capture` (secondary).
  Add ` › REQUIREMENTS` when `parent_section` is set. Use `under task` when the parent
  has no block ID. Keep the multi-item index number and the existing `local override`
  and `scheduled` tokens. The trailing `relative_target` line is dropped, because the
  caption names the note. The header says _what_; the card says _where_. `placement` and
  `kind` wire words no longer show for these items.
- **Caption.** This mirrors the Pomodoro caption (`Running · line 27`): the picker's
  status glyph and color from
  `CaptureEditorPalette.taskStatus(CapturePickerTaskStatus(symbol:name:))` (◐ orange In
  Progress, ◉ blue Next, ○ Todo, ⏸ Blocked, ✓ green Done, ⊗ Canceled), then
  `In Progress · sase.md · line 1` in caption weight. The status name is Bob's
  `status_name`, unless it is empty or `Unknown`; then use the canonical
  `CapturePickerTaskStatus.displayName`. A created parent gets the Pomodoro view's pink
  **New** capsule on a dry run and **Created** once committed.
- **Card.** This is the Pomodoro card, shared: a 3 pt rail in the task's status color, a
  quinary background, a 7 pt continuous corner with a hairline border, the 12 pt diff
  gutter (`+` green, `•` accent with `Was: …` on hover, `−` red and struck), 18 pt
  indent guides, change washes (0.10 / 0.12 / 0.08, or 0.22 under increased contrast),
  and monospaced callout text that wraps and is never truncated.
- **Task tinting.** Tinting is display-only and byte-preserving:
  - list markers and checkbox brackets use the tertiary label color;
  - the checkbox symbol uses `CaptureEditorPalette.checkboxSymbolColor`;
  - `#tags` are secondary;
  - `[key:: value]` fields are secondary;
  - a trailing ` ^block-id` is indigo (`CaptureSemanticCategory.blockID`);
  - wikilinks, inline code, `**bold**` delimiters, and `~~strikes~~` render exactly as
    in Pomodoro rows;
  - the task line's prose is semibold, so the eye lands on the task first.
- **Folding long quiet runs.** This is a deliberate design call. Unlike a Pomodoro, a
  task block is unbounded: every worked Pomodoro appends a Work Log entry, and
  long-lived tasks reach dozens of lines. The panel grows to fit its preview, so a
  60-line Work Log would stretch the window to the screen edge on every keystroke.
  - Blocks of at most `foldThreshold = 24` rows always render in full. That is
    effectively every task.
  - Above the threshold, the visible rows are:
    - the task line;
    - every non-`unchanged` row;
    - `contextRadius = 2` rows on each side of a change;
    - each change's structural ancestors (the nearest preceding row at each shallower
      depth, so `REQUIREMENTS` stays visible above a bullet added inside it).
  - Any other run of at least `minimumFoldRun = 4` unchanged rows collapses into one
    fold row. The fold row reads `⋯ 31 unchanged lines` (secondary, caption) at the
    run's shallowest depth, with indent guides.
  - Clicking a fold expands it in place and returns focus to the editor
    (`model.requestFocus(.editor)`). Expansions reset when the card's identity
    (`relative_target`, `line`, `block_id`) changes.
  - The card offers an accessibility action, **Show all lines**. Its accessibility
    summary always reads every row, so nothing is hidden from VoiceOver.
  - Every line stays one click away. Setting the threshold to `Int.max` removes folding
    entirely, should Bryan prefer strictly unfolded cards.
- **Placement.** Task blocks render once, after the item stack and before the Pomodoro
  blocks: route-note changes first, then day-file changes. Use the same 10 pt spacing
  and 4 pt top padding. They dim to 0.6 with a pending close card, exactly like Pomodoro
  blocks.
- **Covers rule.** A `sub_bullet` item is covered when `taskBlocks` is non-empty and
  every `previewBlockLines` entry, with leading whitespace stripped, matches a distinct
  `added` or `changed` row, likewise stripped, of a task block with the same
  `relativeTarget`. Matching is one-to-one, so duplicate bullets in one item each need
  their own row. Stripping is required because Bob's `task_line` is unindented while the
  block line carries the parent's child indentation. Only covered items switch to the
  compact header and drop the verbatim stack. Uncovered items, which only an older Bob
  produces, render exactly as today.
- **Status and summary.** For a single `sub_bullet` capture, `captureStatus` and
  `captureSummary` name the parent: `Preview → sase.md › ^capture` and
  `Preview → sase.md › ^capture: - Should reuse…`. Captured results get the same with
  `Captured`. A parent without an ID uses `› task`.
- **Stale failure fix.** A successful live preview that sets no kind-specific status
  clears a stale `Preview failed` footer. `statusText` becomes empty, so the footer
  reads `Ready`. Any other status text, such as rewrite notices, is left untouched.

# task_blocks_contract

Work in bob-cli.

- **Code.**
  - New `src/native/capture/block_diff.rs`, holding the shared helpers and types moved
    out of `pomodoro_blocks.rs`.
  - New `src/native/capture/task_blocks.rs`, holding `TaskBlockRef`, `TaskBlockRole`
    (`SubBullet` serializes as `"sub_bullet"`), `TaskBlockTracker`, and `TaskBlockJson`.
  - Wire them through `mod.rs`, `plan.rs` (the batch loop and the sub-bullet branch),
    and `output.rs` (`CaptureResult`). Follow the "How Bob computes task blocks" section
    exactly.
- **Unit tests** (`task_blocks.rs`), each asserting full `lines` (text, depth, change):
  - a plain `@route+id` insertion before a Schedule Log: one added row at depth 1;
  - a `#section` insertion: the added row at depth 2 inside the section;
  - authored children plus `p:<N>`: the new bullet, its children, and the generated
    Schedule Log are all `added`, in order, with correct depths;
  - a global `@@route+id` batch with two items: one block, two added rows, `roles`
    deduplicated;
  - two parents in one note, where the second item inserts above the first parent: both
    blocks are correct after forwarding, in first-touch order;
  - `Task @sase^new` followed by `note @sase+new`: `created: true`, every row `added`;
  - a sub-bullet plus a task toggle of the same parent in one batch: the task line is
    `changed` with `before`;
  - if the draft grammar rejects either of the two batches above, cover the same
    behavior with a direct tracker test that stages the equivalent texts between items;
  - a `--task-ref` parent with no block ID: `block_id` omitted, block present;
  - a nested (indented) parent task: depths are relative to its line;
  - a CRLF note: verbatim texts without terminators;
  - a task whose block has interior blank lines.
- **CLI tests.**
  - Add a `tests/cli/capture/` family (a new `task_blocks.rs` or an extension of
    `authored.rs`/`routing.rs`, matching the module layout). Seed the sandbox vault with
    an Obsidian Tasks `data.json`, the way existing status-name tests do, so
    `status_name` reads `In Progress`.
  - Assert:
    - dry-run JSON carries the expected `task_blocks`;
    - real-run JSON equals dry-run JSON except for `dry_run`;
    - non-sub-bullet captures omit the key;
    - human output is unchanged;
    - a batch that touches both a Pomodoro and a sub-bullet parent reports both arrays
      independently.
- **Docs (`docs/capture.md`).**
  - Add a `#### Task blocks` subsection after `#### Pomodoro blocks`, with the contract
    and field rules from this plan's Design section.
  - Add a sentence to "Sub-bullet results additionally include…" and to "Sub-bullets
    under existing tasks" pointing at it.
  - Update the Contents list if it lists `####` entries.
- **Verification.** `just fmt`, `just lint`, and `just test` pass. If a lint failure
  predates this phase, record it and do not fix unrelated code.

# mac_task_block_model

Work in bob-mac-capture. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<reason>"`) and use only the printed path. This
phase touches only `CaptureCore`, which builds and tests on Linux. Run
`swift build --target CaptureCore` and the `CaptureCoreTests` locally when a toolchain
is present (export `PATH="$HOME/.local/share/swiftly/bin:$PATH"` explicitly in `sh`
scripts), but treat macOS CI as the gate.

- **`CaptureModels.swift`.**
  - Add `CaptureTaskBlock` (Codable, Equatable, Sendable). Every field decodes with
    `decodeIfPresent` and a safe default. Its `lines` reuse `CapturePomodoroBlockLine`.
    Add a neutral `public typealias CaptureBlockLine = CapturePomodoroBlockLine`, and
    use it in new code.
  - Add `CaptureCommandSuccess.taskBlocks: [CaptureTaskBlock]`, decoded with
    `try? … ?? []` exactly like `pomodoroBlocks`, plus an init parameter defaulting to
    `[]`.
- **Shared row model.** Add a public `CaptureBlockDiffRow` with `content`, `depth`,
  `change`, `beforeContent`, `isHeadline`, and `tokens`. Make
  `CapturePomodoroBlockPresentation.Row` a typealias to it, so existing Pomodoro code
  and tests compile unchanged.
- **Tokenizer (`CapturePomodoroLineTokens.swift`).**
  - Add roles `.tag` and `.blockID`.
  - Add `tokenizeTaskRow(_ content: String, isHeadline: Bool)`:
    - an optional list marker (`-`, `*`, `+`, `N.`, `N)`) as `.syntax`;
    - then an optional `[c]` checkbox as `.checkbox(c)`;
    - then the existing inline scan;
    - then a byte-preserving post-pass that splits `#tag` words (`#` at the start or
      after whitespace, followed by `[A-Za-z0-9_/-]+` containing at least one non-digit)
      into `.tag`, and a trailing ` ^[A-Za-z0-9-]+` into `.blockID`.
  - The Pomodoro `tokenize` output must not change. The hard invariant still holds:
    `tokens.map(\.text).joined() == content`.
- **New `CaptureTaskBlockPresentation.swift`** (pure, public), with
  `init(block:dryRun:)` exposing:
  - `status` (`CapturePickerTaskStatus`), `statusText` (with the `Unknown` fallback),
    `captionText` (`In Progress · sase.md · line 1`), and `badgeText` (`New`, `Created`,
    or nil);
  - `rows` (stripped content, depth clamped to
    `CapturePomodoroBlockPresentation.maxDepth`, `isHeadline` for row 0, and task-row
    tokens);
  - `identity` (`relativeTarget`, `line`, `blockID`);
  - folding:
    - the constants `foldThreshold = 24`, `contextRadius = 2`, `minimumFoldRun = 4`;
    - `enum Item { case row(CaptureBlockDiffRow), fold(FoldRun) }`, where `FoldRun` has
      a stable `id` (the first hidden row index), `count`, `depth`, and `rows`;
    - `func items(expandedFolds: Set<Int>) -> [Item]`, following the folding rules in
      the Design section. A block with no changed rows never folds.
  - `accessibilitySummary`, in the Pomodoro wording style, for example
    `Port capture to PIW sase-core, In Progress, sase.md line 1: …; added - Should reuse…; …`.
    It always covers every row.
  - `static func covers(_ capture: CaptureCommandSuccess, blocks: [CaptureTaskBlock]) -> Bool`,
    implementing the one-to-one, whitespace-stripped covers rule from the Design
    section.
- **New `CaptureSubBulletPresentation.swift`** (pure, public, nil unless
  `kind == "sub_bullet"`), exposing:
  - `headerDetail` (`under ^capture`, `under ^capture › REQUIREMENTS`, `under task`);
  - `parentLabel` (`^capture` or `task`) for the status and summary strings;
  - an accessibility phrase.
- **Fixtures (`Tests/Fixtures/`).**
  - Build `bob` from bob-cli at a revision that contains `task_blocks_contract`: pull,
    then `cargo build` in your bob-cli workspace.
  - Generate real JSON against sandbox vaults with `BOB_DIR`, `BOB_NOW`, and an Obsidian
    Tasks `data.json`, so status names are real.
  - Commit the fixtures with `"dry_run": false`, as the existing fixtures do; fake-bob
    flips it for `--dry-run`.
  - Document the vaults and exact commands in a header comment in the decoding test
    file, as `BlockIDDecodingTests.swift` does.

  | Fixture                               | Scenario                                                                  |
  | ------------------------------------- | ------------------------------------------------------------------------- |
  | `sub-bullet-task-block.json`          | the Design section's example: insertion before a Schedule Log             |
  | `sub-bullet-section-task-block.json`  | `Postgres 17 minimum @foo+bar#requirements` (the docs/capture.md example) |
  | `sub-bullet-children-task-block.json` | an item with two authored children (nested added rows)                    |
  | `sub-bullet-global-task-block.json`   | `@@sase+capture` with two items: one block, two added rows                |
  | `sub-bullet-long-task-block.json`     | a parent with a 30-entry Work Log, so the block folds                     |

- **Tests.**
  - `CaptureModelTests`:
    - decoding each new fixture;
    - the key absent (existing fixtures) or malformed, which yields empty `taskBlocks`
      and still decodes.
  - New `CaptureTaskBlockPresentationTests`:
    - caption, glyph status, and status-name fallback per status;
    - the badge;
    - rows and depth clamp;
    - folding:
      - no folds at or below the threshold;
      - the long fixture folds its Work Log tail but keeps the task line, ancestors, and
        context;
      - runs shorter than 4 never fold;
      - expanding by id inlines the rows;
      - no changes means no folds;
    - accessibility covers every row;
    - `covers`:
      - true for each fixture's items;
      - false without blocks;
      - false when the relative targets differ;
      - duplicates need distinct rows.
  - `CapturePomodoroLineTokensTests`:
    - the concatenation invariant over every line of every new fixture;
    - targeted task-row cases: open and done checkboxes; `#task` and `#a/b` tags; `#123`
      is not a tag; `C#` is not a tag; a trailing `^id` versus a mid-line `^`; fields;
      wikilinks; code; `🗓️ **SCHEDULE LOG**`;
    - the Pomodoro `tokenize` cases unchanged.
  - New `CaptureSubBulletPresentationTests`.
- **Commit, then CI.** This plan explicitly instructs you to commit the bob-mac-capture
  changes with `/sase_git_commit` before waiting on CI, because macOS CI only runs on
  the pushed commit and this host has no Apple Swift toolchain for the app target.
  - Use subject `feat(capture): decode and present sub-bullet task blocks`. If
    `$SASE_BEAD_ID` is set, pass `-B keep`.
  - Then wait with `/sase_monitor` on the `CI` workflow run for that SHA
    (`gh run list --commit <sha> --limit 1`, then `gh run watch <id>`), with a timeout
    of at least 20 minutes.
  - If the run is red, read `gh run view <id> --log-failed`, fix forward, commit, and
    watch again.
  - The phase is done only when the latest run for its commit is green.

# mac_task_block_view

Work in bob-mac-capture, opened as in `mac_task_block_model`.

- **Shared card.**
  - Extract the card body from `PomodoroBlockView.swift` into a reusable `BlockDiffCard`
    view. It covers the rail, row stack, gutter glyphs, indent guides, change washes,
    the `Was:` hover, removed-row styling, and token coloring.
  - It takes a rail tint, `[CaptureBlockDiffRow]` (or the task presentation's items, for
    fold rows), and a headline-emphasis flag.
  - `PomodoroBlockView` becomes its caption plus `BlockDiffCard` and must render
    identically. Re-run its PNG test and compare the images when a Mac is reachable.
  - Map the new `.tag` and `.blockID` roles in the shared token color switch.
- **New `TaskBlockView.swift`.**
  - The caption row, card, folds, emphasis, and accessibility follow the Design section
    exactly.
  - Fold rows are plain borderless buttons with a pointing-hand cursor. Expansion is
    `@State Set<Int>` reset by `.onChange(of: presentation.identity)`.
  - Add `.accessibilityAction(named: "Show all lines")`.
  - Use `.textSelection(.enabled)` on rows, honor `colorSchemeContrast`, and add no
    animation.
- **`CapturePanelView.swift` `PreviewPane`.**
  - In `previewContent`, render `success.taskBlocks` with `TaskBlockView` after the
    items and before the Pomodoro blocks, with the same spacing, padding, and
    pending-close dimming.
  - Pass the task blocks into `previewItem` → `standardPreviewItem`. When
    `CaptureTaskBlockPresentation.covers` is true, render the compact sub-bullet header
    instead of `standardHeader`, and omit both the verbatim `previewBlockLines` stack
    and the trailing `relativeTarget` line.
  - VoiceOver announces the item summary once, from the header.
- **`CapturePanelModel.swift`.**
  - Make the sub-bullet-aware `captureStatus` / `captureSummary` wording use
    `CaptureSubBulletPresentation.parentLabel`.
  - Add the stale `Preview failed` fix in `startLivePreview`'s success branch.
- **`Tests/Fixtures/fake-bob`.**
  - Route each new fixture's draft, preferring drafts no existing route uses.
  - Where an existing route or pinned string must change (for example, a sub-bullet
    `Preview → …` summary), update its tests in the same commit.
- **Tests.**
  - `CapturePanelModelTests`:
    - a live preview of the example draft yields one task block and the `› ^capture`
      status and summary;
    - after a failed live preview, a successful one shows `Ready`, not `Preview failed`;
    - a failed dry run clears task blocks with the card;
    - an older-Bob sub-bullet fixture still renders the old item lines.
  - `CapturePreviewFullHeightTests`:
    - `PreviewPane` with `sub-bullet-task-block.json` at widths 724 and 620 is taller
      than the same success with `taskBlocks` emptied;
    - the long fixture is shorter folded than with every fold expanded.
  - New `TaskBlockDesignTests`, a `BOB_MAC_CAPTURE_RENDER_DIR`-gated PNG render of
    `PreviewPane` for every new fixture.
    - Use light and dark appearances at widths 760 and 620, at scale 2.
    - Skip the test when the variable is unset.
    - Follow `PomodoroBlockDesignTests`.
  - Visual review is best-effort:
    - If `ssh -o ConnectTimeout=5 mac true` succeeds, clone the pushed commit into a
      fresh temporary directory on that Mac, run just the render tests with
      `BOB_MAC_CAPTURE_RENDER_DIR` set, copy the PNGs back, inspect them, and iterate on
      spacing, contrast, wrapping, fold rows, and guide alignment.
    - Touch nothing else on that machine, and delete the clone afterwards.
    - If the Mac is unreachable (it is offline unless its lid is open) or lacks a
      toolchain, skip the review and say so on the bead.
- **`README.md`.**
  - Next to the Pomodoro block paragraph, describe the task card:
    - batch-level, after the items and before the Pomodoro blocks;
    - the compact sub-bullet header;
    - the status caption and New badge;
    - the shared diff card and task tinting;
    - folding above 24 rows and click-to-expand;
    - the covers rule.
  - Add the parent-naming status and summary wording.
  - Note that an older Bob without `task_blocks` previews exactly as before.
  - Mention `TaskBlockDesignTests` and its environment variable.
- **Commit and CI.** Follow the same commit-then-CI loop as `mac_task_block_model`, with
  subject `feat(capture): show the full parent task when capturing a sub-bullet`. The
  phase is done only when the latest macOS run for its commit is green.

# Non-goals

- No new capture syntax, no placement change, and no change to `bob capture` human
  output, the `@+` task picker, completion, notifications, or the task-ID and
  Pomodoro-name prompts.
- No task blocks for other kinds yet: toggles, links, `=x` Work Log writes, and project
  notes. The contract's `roles` field leaves room for them in a follow-up.
- No redesign of the close, start, link, or toggle cards, and no visual change to
  Pomodoro blocks beyond extracting the shared card.
- No auto-scrolling of the preview. Bob inserts above managed logs, so the added row
  sits near the top of the block, and folding keeps long blocks compact.
- No changes to bob-plugins, Obsidian, or SASE memory.

# Acceptance

- The opening example previews the whole `^capture` task:
  - the task line, `REQUIREMENTS`, and the Schedule Log are visible;
  - the new bullet is a green `+` row at depth 1 just above the Schedule Log;
  - the item header reads `↳ Sub-bullet under ^capture`;
  - the caption reads `◐ In Progress · sase.md · line N`;
  - the summary names `sase.md › ^capture`;
  - the footer reads `Ready`.
- `#section` captures show the bullet inside its section with the section visible.
  Global batches show one card with every added row. Created parents show a New badge
  and all rows added. A task toggled in the same draft shows its task line as changed.
- Blocks above 24 rows fold quiet runs into `⋯ N unchanged lines` rows that expand on
  click. Smaller blocks are never folded.
- Dry-run JSON equals real-run JSON. `pomodoro_blocks` JSON is byte-identical to before.
  An older Bob without `task_blocks` previews exactly as today.
- bob-cli `just fmt`, `just lint`, and `just test` pass, apart from any recorded
  pre-existing lint issue. bob-mac-capture macOS CI is green for the landed revision.
