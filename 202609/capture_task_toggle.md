---
tier: epic
title: Capture-driven Obsidian task status toggle (@route+block-id with no other text)
goal: 'Submitting a capture draft that is exactly `@route+block-id` (optionally `@route+block-id#pomodoro`)
  toggles that existing Obsidian task between Ready `[ ]` and Next `[*]` and adds
  or removes its Pomodoro task link, matching the Obsidian `<ctrl+shift+enter>` keymap''s
  semantics. The `@route+block-id` completion and Add block ID prompt behave exactly
  as they already do for sub-bullet capture, and Bob Mac Capture makes the mode, the
  target task, and the exact before/after change obvious before the user commits it.

  '
phases:
- id: grammar
  title: Capture grammar and completion for the task-toggle item
  depends_on: []
  size: medium
  description: 'grammar: add the `task_toggle` item mode to the shared capture grammar,
    re-point `#` after `@route+id` at Pomodoro names while the item has no text, add
    the three `task_toggle_*` span kinds, and route completion accordingly.

    '
- id: engine
  title: Pure toggle planners for the route note and the daily ledger
  depends_on: []
  size: medium
  description: 'engine: add a new native module of pure, unit-tested planners that
    compute the route-note task mutation (status, future-schedule pull-forward, Schedule
    Log entry) and the daily-note Pomodoro task-link insertion, cleanup, and removal.

    '
- id: capture
  title: Wire the toggle into bob capture, its JSON contract, and its human output
  depends_on:
  - grammar
  - engine
  size: medium
  description: 'capture: execute `task_toggle` items inside the existing staged batch
    planner, emit the additive JSON fields, render the human before/after output,
    and cover the whole surface with CLI integration tests.

    '
- id: cli-docs
  title: bob-cli documentation for the task-toggle marker
  depends_on:
  - capture
  size: small
  description: 'cli-docs: document the toggle marker, its semantics, its errors, and
    its JSON in docs/capture.md and README.md, including the `#` mode-dependency rule.

    '
- id: mac-core
  title: CaptureCore models, presentation model, and panel wiring
  depends_on:
  - capture
  size: medium
  description: 'mac-core: decode the additive toggle fields in CaptureCore, add a
    Linux-testable toggle presentation model, and route completion, the footer verb,
    status text, announcements, and notifications through it.

    '
- id: mac-preview
  title: Bob Mac Capture toggle preview, highlighting, and documentation
  depends_on:
  - mac-core
  size: medium
  description: 'mac-preview: render the before/after toggle preview and destination
    detail, color the new `task_toggle_*` spans, and document the flow in the app
    README.'
proposed_by: bbugyi200.athena.0ir
create_time: 2026-09-10 13:19:07
status: wip
bead_id: bob-cli-1z
---

- **PROMPT:** [prompts/202609/capture_task_toggle.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/capture_task_toggle.md)
- **BEAD:** [bob-cli-1z](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-1z/README.md)

# Plan: Capture-driven Obsidian task status toggle

## Why this exists

Bob already has one way to flip an existing task onto today's Pomodoro: the Obsidian
`<ctrl+shift+enter>` keymap in the `block-id-prompt` plugin. It only works when the task
is on screen in Obsidian, with the cursor on its line. Bob Mac Capture is the frontend
that is always one hotkey away, but it can only _create_ things — it has no way to say
"the task I already have, `^goog-exit`, is what I'm working on now."

This epic gives the capture panel that verb, reusing the grammar and pickers the user
already knows. A draft that is _exactly_ a `@route+block-id` marker — no body text —
means "toggle this task", because that input is currently a hard error
(`task text is required; pass TEXT or pipe it on stdin`), so the spelling is free and
the change is strictly additive.

### The reference implementation

`<ctrl+shift+enter>` is `openPomodoroTaskLink` in `plugins/block-id-prompt/main.js` of
the `bob-plugins` repo (open it with `sase repo open bob-plugins`). Read it before
implementing `engine`. Its behavior, which this epic reproduces in Rust:

- **Ready `[ ]` or Blocked `[?]` → Next `[*]`** (`planTargetTaskUpdate` with
  `forceNext: true`), plus a `[[route#^id]]` task link under today's current/next
  Pomodoro (`planPomodoroLinkInsertion`), plus removal of the same link from _later_
  still-open Pomodoros (`planFuturePomodoroLinkCleanup`).
- **Pull-forward**: when the task carries exactly one valid, strictly future `scheduled`
  field, that field is removed, and — only when the task already owns a direct-child
  Schedule Log — a dated entry is prepended under the log marker.
- **Next `[*]` → Ready `[ ]`**, plus removal of the link from _every_ currently open
  Pomodoro (`planAllOpenPomodoroLinkCleanup`); closed history is never touched.
- **In Progress `[/]`** opens a work-summary prompt and is a different operation.
- Everything is planned against snapshots first; nothing is written on any error.

The user's prompt named this keymap as `<ctrl+enter>`. `<ctrl+enter>` is Obsidian Tasks'
done/reopen toggle; `<ctrl+shift+enter>` is the open/next + Pomodoro-link toggle this
epic mirrors, and it is what the `Work Log` glossary strand and the `bob-plugins` README
both describe. This plan targets `<ctrl+shift+enter>`.

## The design

### Grammar

An item is a **task toggle** when, after the normal marker extraction the grammar
already performs, all of the following hold:

1. its resolved marker is `@route+block-id` or `@route+block-id#name` (the existing
   `sub_bullet` marker family, block-ID form only), and
2. its capture body is empty, and
3. it has no authored child bullets, and
4. it resolved no other item-wide marker (`s:<N>`, `p:<N>`, `%...`).

That is exactly one physical line holding exactly one token. Leading and trailing forms
collapse to the same thing, so there is no leading/trailing rule to learn.

Two grammar table rows are added:

| Marker                                            | Meaning                                                                                                 |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `@route+block-id` **with no other text**          | Toggle that existing task between Ready `[ ]` and Next `[*]`, adding or removing its Pomodoro task link |
| `@route+block-id#pomodoro` **with no other text** | The same toggle, linking under a matching named open Pomodoro or a new named future Pomodoro            |

**The `#` component follows the item's mode.** After a resolved `@route+block-id`:

- while the item has **no** body text, `#name` selects a **Pomodoro**, with
  byte-for-byte the same slug matching, canonicalization, and create-a-future-Pomodoro
  rules `@route:block-id#name` already has;
- once the item **has** body text, `#name` keeps its current meaning — an ALL-CAPS
  **task section** for the sub-bullet.

This is the one genuinely new rule in the grammar, and it is the right one: a toggle
writes no bullet, so a task-section selector has nothing to select, while a sub-bullet
capture creates no Pomodoro link, so a Pomodoro name has nothing to name. Each mode
offers the only selector that can do anything. To keep it from ever being a surprise
while typing, the mode is made visible in three places at once (span color, footer verb,
preview); see the `mac-core` and `mac-preview` phases.

To keep `@route+block-id` available as literal task text, the existing escape applies
unchanged: wrap it in inline code.

### Status transitions

Bob resolves the task through the same note scan `@route+block-id` sub-bullet capture
uses (`note_tasks::scan` plus `BlockIdLookup`), so missing-ID, duplicate-ID,
non-task-ID, close-match suggestions, and the `bob capture-tasks -r <route>` hint are
all inherited verbatim.

| Current status                        | Result                                                               |
| ------------------------------------- | -------------------------------------------------------------------- |
| `[ ]` Ready                           | `[*]` Next, link added                                               |
| `[?]` Blocked                         | future `scheduled` retired when present, then `[*]` Next, link added |
| `[*]` Next                            | `[ ]` Ready, link removed from every open Pomodoro                   |
| `[/]` In Progress                     | error — needs a work summary; not toggleable from capture            |
| done, cancelled, unknown, or non-task | error                                                                |

Blocked is forced to Next exactly as the plugin forces it, even when the task is blocked
by an open dependency rather than a schedule; `bob task-status-hooks` stays the
authoritative reconciler and will return it to `[?]` on its next run. So that this is
never silent, a toggle to Next on a task line that carries any `dependsOn` field emits a
**warning** (not an error) on the existing `warnings` channel:

```
^goog-exit still declares dependencies; bob task-status-hooks may return it to Blocked
```

This is a cheap lexical check on the task line only. Do not perform a vault-wide
dependency resolution here.

### Pull-forward and the Schedule Log

When the toggle sets Next and the task line carries **exactly one** valid, strictly
future `[scheduled::YYYY-MM-DD]` field, remove that field (collapsing the surrounding
whitespace the same way the plugin's `removeSpanWithSpaceCollapse` does). Then, **only
if the task already owns a direct-child Schedule Log marker**, insert one entry
immediately below that marker (newest first):

```
<entry-indent><marker> _<old-date> → <today>_ — 🍅 pulled into today's Pomodoro
```

Never create a Schedule Log for a task that keeps none. `<marker>` is the log marker
line's own list marker; `<entry-indent>` copies the first existing entry's indentation,
falling back to the marker's indentation plus one tab.

**Byte-exactness matters here, and there is a trap.** `plugins/block-id-prompt/main.js`
sets `SCHEDULE_LOG_ENTRY_EMPHASIS = "_"` for this entry, while
`plugins/bob-navigation-hotkeys/main.js` — the `<ctrl+shift+p>` picker that
`src/native/capture_schedule_log.rs` already mirrors for `p:<N>` — uses `"*"`. Both
spellings are live in the vault today
(`_2026-09-01 → 2026-08-25_ — 🍅 pulled into today's Pomodoro` in `sase.md` versus
`*2026-09-04 → 2026-09-10* — 🎲 …` in `cash.md`). Match `block-id-prompt` and emit the
**underscore** form, so this command and the keymap it mirrors write identical bytes. Do
**not** reuse `capture_schedule_log::entry_text`, which hardcodes `ENTRY_EMPHASIS`; add
a separate, documented formatter beside it and note in a comment why the two differ.
Unifying the two plugins is out of scope for this epic — record it as a follow-up
instead.

### The Pomodoro task link

Link direction, against today's daily note (`BOB_DAY_FILE`, else
`<bob-dir>/YYYY/YYYYMMDD.md`, exactly as `@route:block-id` selects it):

- Select the destination entry with the existing `@route:block-id` rules: the single
  open top-level timed entry, else the first open top-level entry; `#name` instead
  selects a named open entry or creates a named future one, reusing
  `capture::select_named_pomodoro`, `capture::insert_named_pomodoro_child_block`,
  `capture_pomodoros::select_named`, and `capture_pomodoros::named_creation_name`.
- Insert `- [[route#^block-id]]` after the entry's existing children, reusing their
  indentation.
- **Idempotence**: when the selected entry already carries a matching link, insert
  nothing and report `pomodoro_already_linked: true`. This is a deliberate departure
  from `plan_capture_with_pomodoro_link`, which errors on
  `Pomodoro ledger already contains …`; that guard exists because a fresh capture's
  block ID must be new, and a toggle's is not.
- Remove matching links from every _later_ still-open Pomodoro, counting the removals.

Unlink direction: remove matching links from **every** currently open Pomodoro (current
and future), never from a completed entry. When a link is the sole content of its
bullet, remove the whole bullet and the sub-bullets it owns; otherwise remove only the
link token. This is `planPomodoroLinkCleanupForRanges` plus `isDedicatedLinkBullet` and
`listItemSubtreeEdit` in the plugin.

A `#name` selector on an item that turns out to be un-toggling is **inert, not an
error** — the operation is a toggle, and refusing the second press of a toggle because
of a selector that only matters on the first press would be hostile. It is reported, not
swallowed: `pomodoro_selector_unused: true` in JSON, a dim note in human output, and a
preview line in the panel.

**The task note and the daily note may be the same file.** A task that lives in today's
daily note is legitimate. Both edits must be staged onto one snapshot of that path
through the existing `CaptureBatchPlanner`. Do not copy
`plan_capture_with_pomodoro_link`'s
`routed note and Bob daily note must be different files` guard.

### Errors

Every one of these is detected during planning, before any write, and rolls back the
whole batch:

- unresolvable route note, missing block ID, duplicate block ID, non-task block ID —
  inherited from the sub-bullet path, with its existing suggestions and hints;
- `[/]` In Progress —
  `task ^<id> is In Progress; toggling from In Progress needs a work summary, so use <ctrl+shift+enter> in Obsidian`;
- done, cancelled, or unknown status —
  `task ^<id> is <Status Name>; only Ready, Blocked, and Next tasks can be toggled`;
- missing daily note, missing `Pomodoros` section, no eligible open entry, or multiple
  open timed entries when the selection is implicit or a named entry must be created —
  reuse the `@route:block-id` messages;
- combining a toggle item with `s:<N>`, `p:<N>`, `%...`, `--clip`, authored children, or
  any forced destination flag — usage errors naming the conflict.

### JSON contract

`kind` gains `"task_toggle"`. Every field an older Bob Mac Capture build decodes as
**required** stays populated, so an old app never fails to decode a new response:
`routed: true`, `route`, `route_label`, `relative_target`, `target` (the route note),
`text: ""`, `created` (today's local date, the same anchor the Schedule Log entry uses),
`placement: "toggled"` (a new `Placement` variant), and `task_line` set to the
**resulting** task line — so even a stale client's `previewBlockLines` shows something
true.

Additive fields:

| Field                                            | Meaning                                                      |
| ------------------------------------------------ | ------------------------------------------------------------ |
| `toggle_direction`                               | `"next"` or `"open"`                                         |
| `previous_task_line`                             | the task line before the toggle                              |
| `status_symbol`, `status_name`                   | resulting status                                             |
| `previous_status_symbol`, `previous_status_name` | prior status                                                 |
| `block_id`                                       | the toggled task's block ID                                  |
| `day_file`                                       | resolved daily note                                          |
| `block_link`                                     | `[[route#^block-id]]`                                        |
| `pomodoro_link_placement`                        | link direction only                                          |
| `pomodoro_name`                                  | canonical name when named or created                         |
| `creates_pomodoro`                               | link direction; a new named future entry was planned         |
| `pomodoro_already_linked`                        | link direction; the selected entry already had it            |
| `removed_pomodoro_links`                         | count removed (future duplicates, or all on un-toggle)       |
| `removed_scheduled`                              | the retired future date, when one was retired                |
| `schedule_log`                                   | existing `ScheduleLog` shape, only when an entry was written |
| `pomodoro_selector_unused`                       | `#name` was given on an un-toggle                            |

`sub_bullets`, `clip`, `priority`, `priority_label`, `parent_*`, and `scheduled` are
omitted for a toggle.

### Human output

```
✓ toggled  cash.md
  [ ] → [*]  Finish Google Exit Packet!  ^goog-exit
  2026/20260910.md · under CODING
  + [[cash#^goog-exit]]
```

```
✓ toggled  cash.md
  [*] → [ ]  Finish Google Exit Packet!  ^goog-exit
  2026/20260910.md
  − removed 2 Pomodoro task links
```

`--dry-run` says `would toggle`. Status markers use the existing
`style_task_status_marker`; `+`/`−` are green/red; the route and daily-note labels stay
cyan, matching the surrounding commands. Extra dim chips are appended when they apply:
`removed future schedule`, `logged schedule change`, `already linked`,
`#<name> not used when clearing`.

### Bob Mac Capture

The panel keeps its identity: one editor, one Return, no dialogs. What changes is that
the panel tells the truth about which of two verbs Return currently means.

- **Completion is unchanged where the user asked for it to be unchanged.** The `@route+`
  route side, the task list (including `--all-tasks` missing-ID rows), the inline **Add
  block ID** prompt, `bob capture-task-id`, and stale-safe task refs all behave
  byte-for-byte as they do for a sub-bullet capture. This already works today:
  `bob capture-complete --all-tasks` returns the full task list for `@route+` against an
  empty body.
- **`#` after a resolved `@route+block-id` with an empty body** opens the existing
  Pomodoro-name completion, including the `creates_pomodoro` **Create** row and the
  inline **Name Pomodoro** prompt. The app already implements all of it for the `:`
  family; it is keyed on `needs`/context, so it follows Bob's answer.
- **Three new span kinds** — `task_toggle_route`, `task_toggle_block_id`,
  `task_toggle_pomodoro_name` — let the editor color the marker distinctly the instant
  the item becomes a toggle, and back again the instant body text is typed. That colour
  flip is the earliest possible signal that Return now means something different.
- **The footer verb changes** from **Capture** to **Set Next** or **Set Open**, so the
  primary action always names the thing it will do.
- **The preview shows the transition**, not a line to be written:

```
cash.md · ^goog-exit
  [ ] → [*]  Finish Google Exit Packet!
2026/20260910.md · under CODING
  + [[cash#^goog-exit]]
```

Because `Sources/BobMacCapture` cannot be built or tested on Linux (only `CaptureCore`
can; the macOS 26 GitHub Actions job covers the app target), every decision above that
can be expressed as a pure function belongs in `CaptureCore` with tests, and the
SwiftUI/AppKit layer stays a thin renderer. That split is a requirement of the
`mac-core` and `mac-preview` phases, not a suggestion.

## Non-goals

- **In Progress `[/]` and the Work Log.** The toggle refuses `[/]` with an actionable
  message. Capturing a work summary through the panel is a separate, larger design.
- **`--route` + `--task` / `--task-ref` as a toggle spelling.** A toggle needs a block
  ID to build `[[route#^id]]`, which is why the Add block ID prompt exists; there is no
  consumer for a flag-driven toggle. The typed marker is the only spelling.
- **A toggle inheriting a `@@route+block-id` declaration.** A toggle item is exactly one
  marker token, and an item that is only a declaration is already a separate error.
- **Unifying `_` and `*` Schedule Log entry emphasis** across the two plugins. Match the
  keymap being mirrored, record the divergence, move on.
- **Changing `bob task-status-hooks`.** It stays the authoritative reconciler.

## Phases

### Capture grammar and completion for the task-toggle item

Files: `src/native/capture_language.rs`, `src/native/capture_parse.rs`,
`src/native/capture_complete.rs`, `src/native/capture_rewrite.rs`.

1. Add `CaptureKind::TaskToggle { block_id, pomodoro_name: Option<String> }` and
   `EditorMode::TaskToggle` (`"task_toggle"`).
2. Resolve the mode as a **post-pass over the finished item**, not in the per-token
   marker shape: when the resolved kind is `SubBullet` with a
   `SubBulletTarget::BlockId`, the body is empty, `sub_bullets` is empty, and no
   clip/schedule/priority marker was absorbed, convert it to `TaskToggle` and re-kind
   its spans from `sub_bullet_route`, `sub_bullet_block_id`, `sub_bullet_section` to
   `task_toggle_route`, `task_toggle_block_id`, `task_toggle_pomodoro_name`. The current
   `missing_text_error()` on an empty parent body must no longer fire for this shape.
3. Validate a toggle's `#name` with the **Pomodoro** selector character set
   (`is_pomodoro_selector_component`, i.e. the task-section set plus `+`), reporting
   `invalid_pomodoro_name` rather than `invalid_sub_bullet_section`, so
   `@route+id#deep+work` is accepted while the item has no text.
4. For `@route+block-id#` with an empty body, report `mode: "incomplete"` with
   `needs: ["pomodoro_name"]` (today: `["task_section"]`) and keep the trailing `#` as
   an `interactive_placeholder` span. With a nonempty body, `needs` stays
   `["task_section"]`. Cover both, and the flip in each direction, with tests.
5. Keep an item that is _only_ a marker but is **not** a block-ID `+` form (`@route`,
   `@route#Sec`, `@route^id`, `@route+#`, a bare `#`, `@@…`) failing exactly as it does
   today.
6. `capture-complete`: route the `#`-after-`@route+id` context to
   `CompletionContext::PomodoroName` when that item's body is empty, and leave
   `CompletionContext::TaskSection` otherwise. The `@route+` task context and the route
   context are unchanged; assert that in tests rather than editing them.
7. `capture-rewrite`: a toggle item's marker is **not** absorbable into `@@` — absorbing
   it would delete the only content of the item. Add it to the non-absorbable notice
   list beside `@route#Section` and `@route:block-id`, with wording in the same shape:
   `@@ cannot take a task toggle: leave @cash+goog-exit on this item, or delete it`.
8. In a multi-item draft, a toggle item participates normally: it is a real item for
   index, range, and `items[]` purposes, and other items keep their own modes.

Verify with `just fmt lint test` and by hand:

```bash
cargo run --bin bob -- capture-parse --format json -- '@cash+goog-exit'
cargo run --bin bob -- capture-parse --format json -- '@cash+goog-exit#bugs'
cargo run --bin bob -- capture-parse --format json -- '@cash+goog-exit#'
cargo run --bin bob -- capture-parse --format json -- 'note @cash+goog-exit#'
```

The first two must report `mode: "task_toggle"`; the third `needs: ["pomodoro_name"]`;
the fourth `needs: ["task_section"]`.

**Beware a stale `target/debug/bob`**: this checkout can carry a prebuilt binary that
`cargo build` considers fresh but that predates recent commits. Confirm
`cargo run --bin bob -- capture-parse --format json -- 'Body @dev:focus-123#bugs'`
reports `mode: "pomodoro_task"` with `section: "bugs"` before trusting any probe; if it
reports `invalid_pomodoro_block_id`, build into a clean `CARGO_TARGET_DIR`.

### Pure toggle planners for the route note and the daily ledger

New file: `src/native/capture_task_toggle.rs`. Pure functions over `&str` note contents
plus unit tests. **No wiring into `bob capture` in this phase** — that is `capture`'s
job, and keeping them separate is what lets the two run in parallel.

Read `plugins/block-id-prompt/main.js` in `bob-plugins` first
(`sase repo open bob-plugins`); `planTargetTaskUpdate`, `planTargetTaskOpenUpdate`,
`planPomodoroLinkInsertion`, `planFuturePomodoroLinkCleanup`,
`planAllOpenPomodoroLinkCleanup`, `planPomodoroLinkCleanupForRanges`,
`isDedicatedLinkBullet`, and `listItemSubtreeEdit` are the reference.

1. `plan_task_next(contents, task_line_index, today) -> TaskTogglePlan` — retire a
   single strictly future `scheduled` field with whitespace collapse, set `[*]`, and
   prepend the Schedule Log entry when and only when a direct-child log marker already
   exists. Returns the postimage plus `removed_scheduled`, `schedule_log`, and
   `previous_status_symbol`.
2. `plan_task_open(contents, task_line_index) -> TaskTogglePlan` — set `[ ]` and nothing
   else.
3. `plan_link_insertion(day_contents, block_link, pomodoro_name, …)` — destination
   selection, idempotent insertion, and later-open-Pomodoro duplicate cleanup, returning
   `already_linked`, `placement`, `pomodoro_name`, `creates_pomodoro`, and
   `removed_links`.
4. `plan_link_removal(day_contents, block_link, …)` — remove matching links from every
   open Pomodoro, whole-bullet-with-subtree when the link is the bullet's sole content,
   token-only otherwise, returning `removed_links`.
5. The pull-forward Schedule Log formatter, with the underscore emphasis and the
   `🍅 pulled into today's Pomodoro` reason, plus a comment explaining the divergence
   from `capture_schedule_log::ENTRY_EMPHASIS`.

Reuse rather than reimplement: `capture_pomodoros::scan`, `select_named`,
`named_creation_name`, `format_named_placeholder_line`, `canonicalize_pomodoro_name`,
and
`capture::{first_direct_managed_log_start, parse_managed_task_log_marker, list_item_body, list_marker_len, leading_whitespace, first_child_indentation, nearest_shallower_list_item_parent}`.
Preserve CRLF and the note's existing indentation in every edit, and add a test that
proves it.

Cover at minimum: no future schedule; exactly one future schedule; two schedule fields
(retire nothing); a past or today schedule (retire nothing); a task with a Schedule Log;
a task without one; a link already present on the selected entry; a duplicate link on a
later open Pomodoro; a link inside a completed Pomodoro (untouched); a sole-content link
bullet with sub-bullets; a link sharing a bullet with other text; CRLF notes.

### Wire the toggle into bob capture, its JSON contract, and its human output

Files: `src/native/capture.rs`, `tests/cli.rs`.

1. Add `plan_task_toggle_capture` alongside `plan_sub_bullet_capture`, driven by
   `CaptureKind::TaskToggle`. Resolve the task with `note_tasks::scan` +
   `BlockIdLookup`, reusing the sub-bullet path's error text, close-match suggestions,
   and `bob capture-tasks -r <route>` hint.
2. Branch on the resolved status per the transition table and call the `engine`
   planners. Stage both notes through `CaptureBatchPlanner`; when the route note and the
   daily note are the same path, stage one merged postimage. Emit the `dependsOn`
   warning when setting Next.
3. Add `Placement::Toggled` and the additive `CaptureItemResult` fields, all
   `skip_serializing_if` so every other kind's JSON is byte-identical to today.
4. Render the human output described above, with `would toggle` under `--dry-run`.
5. Reject `s:<N>`, `p:<N>`, `%...`, `--clip`, authored children, and forced destination
   flags on a toggle item with usage errors.
6. Confirm batches work: a toggle beside ordinary items, two toggles of different tasks
   in one draft, and a toggle whose Pomodoro was created by an earlier item in the same
   draft — all against staged snapshots, with any failure rolling every target back.
7. Integration tests in `tests/cli.rs`, following the existing capture fixtures'
   conventions: both directions; `#name` selecting an open named Pomodoro; `#name`
   creating a future one; `#name` inert on an un-toggle; idempotent re-link;
   pull-forward with and without a Schedule Log; the task living in the daily note; each
   error; `--dry-run` writing nothing; and rollback leaving every note byte-identical.

### bob-cli documentation for the task-toggle marker

Files: `docs/capture.md`, `README.md`.

1. Add the two grammar-table rows and a `### Task status toggle` section under
   `bob capture`, placed after "Sub-bullets under existing tasks" so the `+` family
   reads as one story.
2. State the `#` mode-dependency rule in one sentence, in both the grammar table's `#`
   reading guide and the new section.
3. Document the transition table, the pull-forward and Schedule Log rule (including that
   a log is never created), link idempotence, the un-toggle's all-open-Pomodoro cleanup,
   the `dependsOn` warning, and every error message.
4. Document the `task_toggle` JSON `kind` and every additive field in the "Input, stdin,
   and JSON output" section, and the `task_toggle` mode plus the three new span kinds
   and the `pomodoro_name`/`task_section` `needs` flip in the `bob capture-parse`
   section.
5. Document the `capture-rewrite` non-absorbable notice and the `capture-complete`
   context rule.
6. Keep the contents list, table alignment, and prose voice consistent with the
   surrounding document. Do not renumber or reflow unrelated sections.

### CaptureCore models, presentation model, and panel wiring

Open the app with `sase repo open bob-mac-capture`. Files:
`Sources/CaptureCore/CaptureModels.swift`,
`Sources/CaptureCore/CompletionRowContent.swift`, a new
`Sources/CaptureCore/CaptureTogglePresentation.swift`,
`Sources/BobMacCapture/CapturePanelModel.swift`,
`Sources/BobMacCapture/NotificationService.swift`, plus tests.

1. Decode every additive toggle field on `CaptureCommandSuccess` with `decodeIfPresent`,
   and keep `previewBlockLines` sensible for a `task_toggle` result.
2. Add `CaptureTogglePresentation`: a pure value type built from a
   `CaptureCommandSuccess` that exposes the before/after task lines and status symbols,
   the destination labels, the added or removed link lines, the dim chips, the primary
   action verb (`Capture` / `Set Next` / `Set Open`), the status text, the VoiceOver
   announcement, and the notification title and body. **All of it is unit-tested in
   `CaptureCoreTests`, which runs on Linux.**
3. Map the three `task_toggle_*` span kinds in `captureSemanticCategory(forSpanKind:)` —
   route to `.route`, block ID to `.blockID`, Pomodoro name to `.section` — so
   completion rows and editor text agree.
4. In `CapturePanelModel.shouldRequestCompletion`, add `pomodoro_name` to the needs set
   and `task_toggle_route`, `task_toggle_block_id`, `task_toggle_pomodoro_name`, and
   `pomodoro_name` to the span-kind set. `pomodoro_name` is missing from both sets
   today, so typing a name after `#` only re-triggers completion incidentally; fix that
   while here and cover it with a test.
5. Route the footer verb, status text, announcements, notification content, and
   Command-Return target opening (the route note, and the daily note when it changed)
   through `CaptureTogglePresentation`. The panel layer must contain no toggle string
   literals of its own.

`swift build`/`swift test` on Linux exercises `CaptureCore` only; state plainly in the
phase's report which parts were verified locally and which rest on macOS CI.

### Bob Mac Capture toggle preview, highlighting, and documentation

Files: `Sources/BobMacCapture/CapturePanelView.swift`,
`Sources/BobMacCapture/CaptureEditorPalette.swift`, `README.md`, and
`Tests/BobMacCaptureTests`.

1. Give `previewItem` a toggle branch that renders the before/after transition, the
   destination and daily-note labels, and the `+`/`−` link line, using the model built
   in `mac-core` and rendering nothing it does not supply. Keep it inside the existing
   auxiliary region's scrolling contract — the preview must not nest another scroll view
   — and keep a batch containing a toggle rendering in the same ordered stack with the
   same dividers as any other batch.
2. Keep the accessibility label informative: position, kind, destination, and the
   transition read as one sentence.
3. `CaptureEditorPalette` needs no new case if the three span kinds map onto existing
   categories in `mac-core`; if a distinct toggle accent reads better in the running
   app, add exactly one adaptive system color and use it for all three.
4. Update the app README: the toggle draft shape, the `#` mode-dependency rule, the
   footer verb change, the preview treatment, the unchanged `@route+` completion and Add
   block ID prompt, and the added `bob` version requirement in the Requirements list —
   including how an older `bob` behaves (it reports `task text is required`, which the
   panel surfaces unchanged).
5. Run `just format-lint` and `just test` on macOS if a macOS host is available;
   otherwise state that CI is the verification and make sure the workflow will exercise
   it.

## Verification

- `just all` passes in `bob-cli` (`cargo fmt --check`,
  `cargo clippy --all-targets --all-features`, `cargo test`).
- `just format-lint build test` passes for `bob-mac-capture` on macOS 26 CI;
  `swift test` passes for `CaptureCore` on Linux.
- Against a scratch vault, `bob capture --format json -- '@<route>+<id>'` flips a Ready
  task to `[*]` and adds the link, and running it again flips it back and removes it,
  leaving both notes byte-identical to their pre-toggle state apart from any Schedule
  Log entry the pull-forward legitimately wrote.
- `--dry-run` on both directions writes nothing.
- Every existing capture kind's JSON output is byte-identical to `master`'s for the same
  input.

## Follow-ups to record, not fix here

- `plugins/block-id-prompt/main.js` and `plugins/bob-navigation-hotkeys/main.js`
  disagree on `SCHEDULE_LOG_ENTRY_EMPHASIS` (`_` versus `*`), and both spellings exist
  in the vault. Worth reconciling in `bob-plugins`, with a vault migration.
- The Obsidian `<ctrl+shift+enter>` keymap and this marker will now be two
  implementations of one behavior. If they drift, the fixture-style parity tests that
  `scripts/test-navigation-hotkeys.cjs` uses for the Schedule Log are the pattern to
  extend.
