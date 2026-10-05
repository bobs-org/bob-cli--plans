---
tier: epic
title: Complete any open task from capture with a whole-item `!note:block-id`
goal: 'A capture item that is exactly `!note:block-id` marks that existing open task
  Done, exactly as `=x!N` would, but without closing a Pomodoro. In one atomic write
  it also closes the task''s embedded subtasks, retires its Task Links in today''s
  ledger the way `bob task reconcile` would, and unblocks dependents the way Obsidian''s
  Ctrl+Enter does. Bulk works one item per blank-line-separated block. In Bob Mac
  Capture, typing `!` at the start of an item opens a "Complete" task picker over
  every open task in the vault. Tasks with Task Links in today''s daily note come
  first, grouped by the Pomodoro they live in. A completion preview shows the struck
  task, its ledger effect, and the tasks it unblocks before Return writes anything.

  '
phases:
- id: engine
  title: Extract a shared task-completion engine (no new syntax)
  depends_on: []
  size: medium
  description: 'engine: in bob-cli, add `src/native/task_complete/`. Extract the =x
    embedded tree close into a reusable `complete_task_tree`, keeping =x byte-identical.
    Expose a scoped completed-reference retirement built on reconcile''s structural
    planner, with an opt-in dedupe that avoids the bob-cli-2l duplicate. Add an immediate
    Blocked-dependent recovery matching Ctrl+Enter, built on reconcile''s dependency-state
    definition. Unit tests only; no user-visible change.

    '
- id: grammar
  title: Lex, claim, and parse whole-item `!note:block-id`
  depends_on: []
  size: medium
  description: 'grammar: in bob-cli, generalize the `&` note-locator lexer and `replacement_for`
    over a sigil. Claim whole-item `!` tokens in execution and editor parsing. Add
    `CaptureKind::TaskComplete`, capture-parse mode/need `task_complete`, `task_complete_*`
    spans, the per-item `task_complete` object, the `invalid_task_complete` diagnostic,
    and teaching refusals for queries and padded items. Execution of a complete token
    returns a temporary refusal until `execute` lands. Grammar docs.

    '
- id: picker_contract
  title: Serve the `task_complete` picker from capture-complete
  depends_on:
  - grammar
  size: medium
  description: 'picker_contract: in bob-cli, add the vault-wide completable-task catalog
    with today''s Task Link annotations (running/worked/queued/noted), the today-first
    ordering and ranking, the `task_complete` capture-complete context with `picker`
    descriptor and continuation keys, `already_selected` and recurring guards, `complete_replacement`
    on capture-task-id, shell completion for `!`, tests, and docs.

    '
- id: execute
  title: Execute `!note:block-id` through the engine with rich JSON and human output
  depends_on:
  - engine
  - grammar
  size: medium
  description: 'execute: in bob-cli, plan `TaskComplete` items through the staged
    batch writer. Resolve the note vault-wide, validate status and recurrence, then
    run the engine (tree close, scoped ledger retirement, dependent recovery). Emit
    kind `task_complete` with the `task_complete` object and placement `completed`,
    report task_blocks roles `completed`/`unblocked`, and print green human output.
    Update `bob capture --help`, docs/capture.md, README, and CLI tests.

    '
- id: mac_preview
  title: Highlight `!` tokens and preview completions in Bob Mac Capture
  depends_on:
  - execute
  size: medium
  description: 'mac_preview: in bob-mac-capture, map the `task_complete_*` spans to
    the shared complete-green and the route/block-ID colors, decode the `task_complete`
    result, add a pure CaptureTaskCompletePresentation and the completion preview
    card (struck task, transition, subtasks, ledger, unblocked), and update footer,
    notification, and VoiceOver copy. Add real-bob fixtures, tests, and README. Commit,
    then get macOS CI green.

    '
- id: mac_picker
  title: Open the Complete picker on `!` with Today first
  depends_on:
  - picker_contract
  - mac_preview
  size: medium
  description: 'mac_picker: in bob-mac-capture, add the `.taskComplete` picker source
    and need. Add a TaskCompletePickerIndex with per-Pomodoro Today sections, then
    In Progress, Next, and per-note sections, and Today-then-All sections while filtering.
    Rows show disabled reasons and a 🍅 session count. Return inserts, Shift-Return
    inserts and starts the next `!`, Command-Return completes; Bob''s continuation
    keys apply. The Add block ID flow splices `complete_replacement`. Tests, design
    render, README, green macOS CI.

    '
proposed_by: bbugyi200.apollo.5a
create_time: 2026-10-05 15:13:22
status: done
bead_id: bob-cli-4i
---

- **PROMPT:** [prompts/202610/bang_task_complete.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bang_task_complete.md)
- **BEAD:** [bob-cli-4i](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4i/README.md)

# Problem

It is 16:40. The running `CAPTURE` session holds `[[sase#^fix-flaky]]`, and the fix just
shipped. Bob has no capture gesture that means "this task is done":

- `=x!1` completes it, but only by closing the running session.
- `@sase+fix-flaky!` toggles the Task Link; it never completes anything.
- Opening Obsidian, finding the task, and pressing Ctrl+Enter or Alt+] works, but it
  leaves capture. Afterwards a stale `[[sase#^fix-flaky]]` sits in the ledger until the
  next `bob task reconcile` pass. If `=x` runs first, it treats that line as worked-on
  and carries a Done task into the next placeholder.

Completing work is as common as starting it. The panel should finish a task as fast as
it links one, and the vault should look right the moment Return is pressed.

# Design

## Principles

1. **`!` means complete, everywhere.** `=x!2` completes Task Link 2, `=!` completes
   every link, and the close card paints completions green. A leading `!note:block-id`
   reuses that meaning and that green. The token names a task by the same vault-wide
   note locator `&note:block-id` uses, so `!` and `&` share one lexer, one resolver, and
   one replacement formatter. The user's `!file:id` is this `!note:block-id`.
2. **Whole item, always.** A complete token must be the entire capture item: no body
   text, children, `%`, `s:`, `p:`, `#name`, `=…`, `@@`, or forced destination flags.
   Anything extra is a teaching error, never a junk inbox task. Bulk means one `!` item
   per blank-line-separated block. Every item is planned against the same staged
   snapshot, and any failure rolls the whole batch back.
3. **One completion, one write.** A `!` capture leaves the vault in the state Bob's own
   writers converge to:
   - the task line matches what `=x!N` writes;
   - the ledger matches what `bob task reconcile` would make it;
   - dependents recover the way Ctrl+Enter recovers them.

   Every effect reuses an existing implementation through the `engine` phase. No step
   gets a second copy of its rules.

4. **Bob decides, the app presents** (decision `mac-capture-is-a-thin-client`). Bob owns
   the claim, the catalog, the today annotations, the order, the replacement strings,
   the continuation keys, and the preview facts. The app filters locally and renders. It
   never builds a `!` token itself.
5. **Additive contracts.** `capture-parse`, `capture-complete`, `capture-task-id`, and
   `bob capture` JSON keep `schema_version` 1 and only gain fields and enum values. An
   older app degrades gracefully: unknown spans render neutral, an unknown context falls
   back to the inline completion list, and an unknown kind gets the standard preview.

## Grammar contract

An item is claimed by `!` only when it has exactly one physical line and that line,
trimmed, starts with `!`. The rest of the line is tokenized quote-aware with the `&`
locator rules, so `!"Shopping List":milk` is one token.

| Draft item                                                                         | Reading                                                                                                                               |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `!`                                                                                | Query `""`: incomplete, `needs: ["task_complete"]`, picker opens                                                                      |
| `!fix`, `!sase:`, `!"Shopping List`, `!"Shopping List"`                            | Query: incomplete, `needs: ["task_complete"]`                                                                                         |
| `!sase:fix-flaky`                                                                  | Complete: mode `task_complete`; capture completes `^fix-flaky` in the note `sase` resolves to                                         |
| `!projects/foo:bar`, `!"Shopping List":milk`                                       | Complete: nested and quoted locators, exactly as `&` decodes them                                                                     |
| `!sase:fix-flaky more`, `!sase:fix-flaky` plus a child line, `!sase:fix-flaky p:1` | Claimed and invalid: `invalid_task_complete`                                                                                          |
| `!sase:fix-flaky=x`, `!sase:fix-flaky#bugs`                                        | Claimed and invalid, with the hint "`!` takes no `=` or `#` suffix; to complete the running session's link while closing, use `=x!N`" |
| `!sase:a !sase:b`                                                                  | Claimed and invalid, with the hint "put each completion on its own item, separated by a blank line"                                   |
| `!wow this works`, `! foo`, `Wow!`, `Buy milk !sase:x`                             | Prose, unchanged: several tokens with an incomplete first token, a space after `!`, or a non-leading `!`                              |
| `!fix` plus a child line                                                           | Prose: an incomplete token on a multi-line item, like `:dee` plus a child                                                             |
| `![[sase#^x]]`, `![alt](url)`, `!!`, `!!!`                                         | Prose: embeds, images, and `!` runs are never claimed (the second byte is `[` or `!`)                                                 |

Notes:

- **Execution refusals.** `bob capture` refuses a claimed query before any other item
  parser, with the usual `capture item K starting on line L:` prefix and exit 2:
  - `!` and `!fix` read
    `` `!fix` opens the task picker; pick a task to insert its `!<note>:<block-id>` token, or write one yourself (for example `!sase:fix-flaky`)``.
  - A claimed invalid item reads
    ``  `!sase:fix-flaky` completes an existing task and must be the whole capture item; remove `more` ``
    (or `remove its child lines`), plus the targeted hints above.
- **Shell quoting.** Docs say to single-quote: `bob capture '!sase:fix-flaky'` (zsh
  history-expands `!`).
- **`@@` declarations** never apply to a `!` item.

`capture-parse` (schema 1, additive):

- **Query items.** `mode: "incomplete"`, `needs: ["task_complete"]`, an empty body,
  `route`/`section`/`block_id` all `null`, no diagnostics, and one
  `interactive_placeholder` span over the whole token, sigil included. This mirrors `:`.
- **Complete items.** `mode: "task_complete"`, `needs: []`, `block_id: "<id>"`,
  `route: null`, and three spans:
  - `task_complete_sigil` over the `!`;
  - `task_complete_note` over the note, quotes included;
  - `task_complete_block_id` over the ID.
- **The `task_complete` object** (only on `task_complete` items):
  `{ "raw", "note", "block_id", "quoted", "range": {start, end} }`. `note` is decoded,
  and the range covers the whole token.
- **Claimed invalid items** report diagnostic code `invalid_task_complete`, with the
  same mode handling that `invalid_pomodoro_link` uses for a padded `^route:id`.

## Execution semantics

For a complete `!note:block-id` item, in this order and all against the staged batch
snapshot:

1. **Resolve.**
   - The note resolves exactly like `&`: an explicit relative path first, else a unique
     basename. Notes staged earlier in the batch also count. The same "no such note" and
     "ambiguous note … use the full relative path" usage errors apply.
   - The block ID resolves through the same lookup, with the same errors: missing ID
     with a close-match suggestion, duplicate ID, and not-a-task.
2. **Validate the status.** Ready `[ ]`, Blocked `[?]`, Next `[*]`, and In Progress
   `[/]` complete.
   - Done (any DONE-type status) is an idempotent no-op that reports
     `action: "already_done"` and writes nothing for that item. This also covers the
     same task named twice in one batch.
   - Canceled fails with
     `` `^id` in sase.md is Canceled; reopen it before completing it``.
   - Unknown or custom non-open statuses fail with
     `` `^id` in sase.md has status `[>]`; only Ready, Blocked, Next, and In Progress tasks can be completed``.
3. **Refuse recurring tasks.** A line with `[repeat:: …]`, `(repeat:: …)`, or `🔁` fails
   with
   `` `^water` in habits.md repeats; complete recurring tasks in Obsidian so Tasks writes the next occurrence``.
   This matches the Cancel picker's refusal and covers `#gtd #pre/#post` checklist rows.
4. **Close the tree** with `complete_task_tree`. The root gets `x` plus
   `  [completion:: YYYY-MM-DD]` (today's local date, two spaces, before a trailing
   `^id`, only when the line carries the global-filter tag), byte-identical to the
   `=x!N` close.
   - Embedded `![[…]]` subtasks inside the root's block close recursively under the
     existing close rules: Depends-On lines are skipped, the depth cap is 25, the target
     cap is 250, and only ` `/`*`/`/` descendants close.
   - Unlike `=x!N`, the root may be Blocked.
   - Descendants left open are reported, not silently dropped.
   - No `[fresh::]` stamp, Schedule Log, or scheduled-date change.
5. **Retire Task Links** in today's daily note, when it exists and has a `## Pomodoros`
   section. Every completed identity (the root plus closed subtasks) is retired with
   reconcile's structural rules:
   - links are struck to `~~[[…]]~~`;
   - bullets under open entries move to the running entry, else to the last completed
     entry with children;
   - lines under completed entries are normalized in place;
   - a placeholder emptied by a move is removed;
   - with the new dedupe flag on, a childless bullet whose only link is that task is
     deleted rather than moved when the destination entry already links the task. That
     is the bob-cli-2l scenario: worked this morning, carried to a placeholder,
     completed between sessions.

   Retirement is scoped: only links that resolve to this batch's completed identities
   are rewritten.

6. **Recover dependents.**
   - **Which:** Blocked `[?]` dependents that list a completed task's `[id::]` in their
     `[dependsOn::]`, have no other open dependency in the post-completion snapshot (the
     reconcile definition), and have no strictly future `scheduled` date.
   - **Effect:** they become Ready `[ ]`, exactly like Task Status Cycler's Ctrl+Enter
     recovery. A later `bob task reconcile` remains authoritative and may promote them.
   - **Not touched:** dependents' Depends-On lines and derived fields.
7. **Leave to the existing writers:**
   - status-group moves to `### Done & Canceled` and badge rows (reconcile);
   - `^prj` frontmatter and `#hide` effects (`bob projects sync`);
   - archiving (`bob task archive`).

   Docs say so.

`plan_budget` is reported automatically whenever the Pomodoros section changed.
`pomodoro_blocks` auto-detects the touched entries.

## `bob capture` result contract

Per item, additive:

- **Common fields.**
  - Kind `"task_complete"`, `placement: "completed"`, `routed: true`, `route: null`,
    `text: ""`.
  - `relative_target` / `target` / `route_label` name the task's note (`route_label` is
    the vault-relative file name, such as `projects/foo.md`).
  - The task fields: `task_line` (post-image), `previous_task_line`, `block_id`,
    `previous_status_symbol` / `_name`, `status_symbol` / `_name`, `status_changed`, and
    `created` (date string, as every kind).
- **The `task_complete` object:**

```json
"task_complete": {
  "raw": "!sase:fix-flaky",
  "note": "sase",
  "note_path": "sase.md",
  "block_id": "fix-flaky",
  "action": "completed",
  "completion_date": "2026-10-05",
  "subtasks": [
    {"note_path": "sase.md", "block_id": "write-test", "line": 9, "text": "Write the regression test",
     "previous_status_symbol": "/", "previous_status_name": "In Progress",
     "status_symbol": "x", "status_name": "Done"}
  ],
  "subtasks_left_open": [
    {"note_path": "sase.md", "block_id": "ask-infra", "line": 11, "text": "Ask infra", "status_symbol": "?",
     "status_name": "Blocked", "reason": "blocked"}
  ],
  "ledger": {
    "day_file": "2026/20261005.md",
    "struck": 1,
    "moved": [{"from": {"line": 14, "name": "SASE", "status": "queued"},
               "to": {"line": 5, "name": "CAPTURE", "status": "running"}}],
    "deduplicated": 0,
    "removed_placeholders": [{"name": "SASE"}]
  },
  "unblocked": [
    {"note_path": "travel.md", "block_id": "book-flights", "line": 3, "text": "Book flights",
     "previous_status_symbol": "?", "previous_status_name": "Blocked",
     "status_symbol": " ", "status_name": "Ready"}
  ]
}
```

Field rules:

- `subtasks`, `subtasks_left_open`, and `unblocked` are always present and possibly
  empty.
- `ledger` is omitted when today's ledger was untouched.
- `completion_date` is omitted when `action` is `already_done`.
- `reason` is `blocked`, `recurring`, `unknown_status`, or `cap`.

Batch-level additive:

- `task_blocks` gains roles `"completed"` (the root and each closed subtask) and
  `"unblocked"` (each recovered dependent). Each is a full block with a cumulative diff,
  so the completed line shows as `changed` with `before`.
- `pomodoro_blocks` reports touched entries through the existing auto-detection.
- Dry-run JSON equals real-run JSON except for `dry_run`.

Human output (`NO_COLOR` shown; `x` markers render in the done green, and dry run says
`would complete`):

```text
✓ completed [*] → [x] Fix flaky gkeep test  sase.md ^fix-flaky
    [/] → [x] Write the regression test  sase.md ^write-test
    left Blocked [?] Ask infra  sase.md ^ask-infra
  ledger  Task Link moved SASE → CAPTURE (struck) · removed empty SASE · 2026/20261005.md
  unblocked [?] → [ ] Book flights  travel.md ^book-flights
```

- Already Done prints
  `✓ already done [x] Fix flaky gkeep test  sase.md ^fix-flaky · nothing to change`.
- Ledger line variants:
  - `Task Link struck in CAPTURE`;
  - `Task Link struck in PLAN (completed)`;
  - `Task Link moved SASE → CAPTURE (struck)`;
  - `Task Link already in PLAN; dropped the SASE copy` (dedupe).

## Picker contract (`capture-complete`, context `task_complete`)

- **When it applies.** The caret is inside or at the end of a claimed `!` token, whether
  complete or partial.
- **Response fields:**
  - `replacement`: the whole token, sigil included.
  - `query`: decoded like `&`.
  - `picker`:
    `{kind: "task_complete", scope: "vault", scope_token: "!", marker_range, trigger_removal_range}`,
    both ranges being the token.
  - `action_continuation_keys`: `["!", "["]` only when the token is the bare `!` and is
    the whole item. An editor can then hand `!!` and `![[` straight back to prose and
    embeds.
- **Catalog.** Every open task (` ` `?` `*` `/`) in every task-bearing vault note, using
  the `&` discovery scan's notes and exclusions.

Candidate fields:

- **Identity and insertion.**
  - `replacement`: `!loc:id`, quoted when needed. It is `""` for ID-less or disabled
    rows; never insert it.
  - `ref`, `note_path`, `locator`, `block_id`, `requires_block_id`,
    `block_id_suggestions`.
- **Task data.** `status_symbol` / `status_name` / `status_type`, `text`, `section`,
  `depth`, `line`, `hidden`, `scheduled` (when set).
- **Group.** `group`: `today`, `in_progress`, `next`, or `open`.
- **`today`** (today rows only):
  - `role`: `running`, `worked`, `queued`, or `noted`;
  - `pomodoro`: `{line, name?, time_range?, status}`, omitted for `noted`;
  - `sessions`: distinct Pomodoros today that link the task.
- **Guards.**
  - `recurring: true` with
    `disabled_reason: "Recurring — complete it in Obsidian so Tasks writes the next occurrence"`.
  - `already_selected: true` (named by another `!` item in this draft) with
    `disabled_reason: "Already in this draft"`.

Today annotation:

- **Which links count.** Any unstruck block link, plain or `![[…]]`, alias allowed, 🍅
  stripped. It can sit anywhere in today's daily note outside fences and Depends-On
  lines, and must resolve with `VaultLinkResolver` to an open task.
- **Role:**
  - links under the running (single open timed) entry → `running`;
  - under a completed entry → `worked`;
  - under any other open entry → `queued`;
  - outside `## Pomodoros` → `noted`.
- A task keeps its strongest role, in precedence running > worked > queued > noted. For
  `worked`, it keeps the most recent entry.

Order:

- **Empty query.**
  1. Today rows: running in ledger order, then worked by entry most-recent-first (ledger
     order inside an entry), then queued in ledger order, then noted in document order.
  2. `in_progress`, then `next`, each by note path then line.
  3. `open` by note path, then document order.

  Within each section, hidden rows go after visible ones and recurring rows go last.

- **Non-empty query.** Every term must match (the shared tiered `match_score` over text,
  locator, `locator:block_id`, block ID, note path, section, and today Pomodoro name).
  Today matches come first by descending score, then all other matches by descending
  score, with ties in canonical order.

`capture-task-id` gains `complete_replacement` (`!loc:id`) next to
`dependency_replacement`. Shell completion offers identified, enabled rows as `!loc:id`,
with the text as description and groups `today` / `in progress` / `next` / `open`.

## Bob Mac Capture

**Highlighting.**

- `task_complete_sigil` → the existing `pomodoroCloseComplete` category (green, the same
  green as `=x!`).
- `task_complete_note` → `.route`.
- `task_complete_block_id` → `.blockID`.

**Completion preview card** (above Bob's task and Pomodoro block cards):

```text
✓ Would complete                                   sase.md · ^fix-flaky
  [*] → [x]  F̶i̶x̶ ̶f̶l̶a̶k̶y̶ ̶g̶k̶e̶e̶p̶ ̶t̶e̶s̶t̶
  ☑︎ Closes 1 embedded subtask · leaves 1 Blocked
  ⏱ Moves its Task Link SASE → CAPTURE, struck · removes empty SASE
  🔓 Unblocks Book flights · travel.md ^book-flights      [?] → [ ]
```

- Green `checkmark.circle.fill` seal.
- The task text is struck and dimmed after the arrow.
- Fact rows use SF Symbols: `list.bullet.indent`, `timer`, `lock.open.fill`.
- Already Done reads `Already done — nothing to change`, in gray.
- The footer action is **Complete**.
- The notification title is `Completed: <task>` (batch: `Completed N tasks`), and the
  body is `sase.md · unblocked Book flights`.
- Every fact also appears in the VoiceOver summary.

**Complete picker** (empty filter):

```text
┌ !  Complete ─────────────────────────────────────────────────┐
│ Search open tasks to complete                                │
│ NOW  CAPTURE · 0920–0950                                     │
│   ◉ Fix flaky gkeep test                    sase:fix-flaky   │
│ 🍅 PLAN · 0830–0855                                           │
│   ◐ Outline the talk                 🍅 2    talks:outline    │
│ UP NEXT  SASE                                                │
│   ◉ Recovery panel                   sase:recovery-panel     │
│ IN TODAY'S NOTE                                              │
│   ○ Call the bank                       cash:call-bank       │
│ IN PROGRESS · NEXT · cash.md · projects/foo.md …             │
└ Return Insert · ⇧Return Insert & next · ⌘Return Complete ────┘
```

- **Sections.** One section per today Pomodoro: the running one with the pink NOW pill,
  completed ones recent-first with 🍅 and their time range, queued ones with `UP NEXT`.
  Then "In today's note", In Progress, Next, and one section per note.
- **While filtering**, two ranked sections: **Today** and **All open tasks**.
- **Rows.** Status glyph, text, and locator (note in accent, `:`, ID in indigo). A
  `🍅 N` capsule appears when the task was in two or more Pomodoros today. Disabled rows
  stay visible with their reason badge and no insertion.
- **ID-less rows** open the Add block ID prompt (`--note-path`) and splice Bob's
  `complete_replacement`. The buttons are **Add ID & Insert** and **Add ID & Complete**.
- **Keys.**
  - Return/Tab inserts.
  - Shift-Return inserts, appends a blank line plus `!`, and the fresh picker opens with
    the just-picked task marked "Already in this draft".
  - Command-Return inserts, then captures.
  - With an empty filter, Bob's continuation keys `!` and `[` close the picker and type
    the key.
  - Escape and Backspace behave as in `:`.

# engine

Work in bob-cli. No user-visible behavior changes in this phase.

- **New module `src/native/task_complete/`** (`mod.rs` plus focused files), registered
  in `src/native.rs`. It is the single home for "what completing a task writes". Keep
  everything `pub(crate)`.
- **Tree close.**
  - Extract `ClosePlanner::apply_embedded_tree` and its private helpers from
    `src/native/capture_pomodoro_close/linked_tasks.rs` into
    `task_complete::complete_task_tree`:
    - `embedded_children` (skips Depends-On lines);
    - the depth cap 25 and target cap 250;
    - `task_has_global_filter`, `remove_completion_fields`,
      `add_or_replace_completion_field`, and `replace_line`;
    - the `set_task_line_status` call.
  - It works over the existing `CloseVault` abstraction. The capture side already
    bridges it with `SnapshotCloseVault`.
  - Parameters: the root address (vault-relative path plus block ID), the completion
    date, and a root policy. `CloseLink` allows only ` `/`*`/`/` roots (today's
    behavior). `Explicit` also allows a `?` root.
  - It returns structured transitions: the root, closed subtasks in close order, and
    descendants left open with reason `blocked` / `recurring` / `unknown_status` /
    `cap`.
  - Rewire the close to call it with `CloseLink`. **Every existing close test must pass
    unchanged.** `=x!N` output stays byte-identical.
- **Scoped ledger retirement.**
  - Add
    `task_complete::retire_completed_links(day_text, completed, link_statuses, settings) -> LedgerRetirement`.
    It is built on `task_status_hooks::pomodoro::scan_pomodoros`,
    `structure::plan_structural_changes`, `apply_structural_plan`, and
    `plan_empty_pomodoro_removals`. Make exactly the needed items `pub(crate)`.
  - The caller passes `link_statuses`: for each link in the day file, either `Done` (its
    identity is in `completed`), `Live(status)` (it resolves to an open task), or
    omitted (anything else).
  - Build the `resolved` map from that, so liveness checks on mixed bullets stay
    correct. Done tasks completed elsewhere and not yet reconciled are left untouched.
  - Pass an empty duplicate-deletion set.
  - Add an opt-in `dedupe_into_destination` flag to `plan_structural_changes`. Reconcile
    passes `false` and stays byte-identical; retirement passes `true`. With the flag on,
    a bullet that would move, has no children, and whose only link resolves to a task
    the destination entry already links (struck or not) is deleted instead of moved. It
    is reported as deduplicated.
  - The result carries:
    - the new text;
    - counts of struck, moved (with source and destination entry line, name, and
      status), deduplicated, and removed placeholders.
- **Dependent recovery.**
  - Add `task_complete::recover_blocked_dependents(snapshot, completed_ids, today)`. The
    snapshot is an iterator of `(relative_path, text)` with staged overlays.
  - It reuses reconcile's parsed `TaskLine` / `FileScan` and the open-dependency rule of
    `task_status_hooks::sync::task_dependency_states`. Make those `pub(crate)`, or move
    them into a shared module the reconcile loop also calls, so there is exactly one
    definition.
  - A dependent recovers to Ready (`[ ]`, via `set_task_line_status`) when:
    - it is Blocked `[?]`;
    - its `dependsOn` contains a completed id;
    - no other dependency id maps to an open task;
    - it has no strictly future valid `scheduled`.
  - It returns the transitions with path, line, block ID, and text. This matches
    `docs/task-status-hooks.md`'s Ctrl+Enter recovery paragraph.
- **Unit tests** in `src/native/task_complete/tests/`:
  - a tree close with nested embeds, a Blocked descendant left open, and a recurring
    descendant;
  - `Explicit` completes a `?` root and `CloseLink` refuses it;
  - completion-field spacing with and without a trailing `^id`;
  - retirement:
    - strike in place under the running entry;
    - move from a placeholder to the running entry;
    - move to the last completed entry when nothing runs;
    - removing an emptied placeholder;
    - a mixed bullet with a live second link not moving;
    - an unrelated already-done link left untouched;
    - dedupe deleting the carried copy when the completed entry already links the task
      (the bob-cli-2l fixture), while reconcile without the flag is unchanged;
    - CRLF;
  - recovery:
    - a sole prerequisite recovers;
    - a second open prerequisite keeps it Blocked;
    - a future `scheduled` keeps it Blocked;
    - a non-Blocked dependent is untouched;
    - an unresolved id does not block.
- **Docs.** In `docs/task-status-hooks.md`, add one sentence next to the relocation
  rules: the scoped capture-side retirement uses the same planner with the dedupe flag.
  Record a `PROPOSED FOLLOW-UP:` note on this phase's bead saying that the Rust
  planner's dedupe path exists behind the flag. Enabling it for reconcile, together with
  the bob-plugins JS fix, belongs to bob-cli-2l.
- **Verification.** `just fmt`, `just lint`, and `just test` pass. Record any lint
  failure that predates this phase, and do not fix unrelated code.

# grammar

Work in bob-cli.

- **Shared locator lexer** (`src/native/capture_language/dependencies.rs`).
  - Parameterize the private scanner helpers over the sigil byte instead of hard-coding
    `b'&'`: `scan_modifier`, `scan_quoted_modifier`, `decode_quoted_note`,
    `scan_block_id`, `validate_note`, `validate_block_id`, and
    `decode_dependency_query`.
  - Expose a `pub(crate)` entry point that parses one whole `!` token into the shared
    `DependencyRef`-shaped struct (raw, note, block ID, quoted, ranges), or reports
    partial or invalid. Keep the `&` behavior and tests byte-identical.
  - Generalize `capture_dependency_tasks::replacement_for` to take the sigil, and keep a
    thin `&` wrapper.
- **Claim predicate** (`src/native/capture_language/tokens.rs`, next to
  `task_link_query_token`). Add one shared predicate for execution, the editor parse,
  and completion. It implements the Grammar contract table:
  - single-line items only;
  - a leading `!`;
  - quote-aware single token for queries;
  - complete-token-plus-extra is claimed invalid;
  - a second byte of `[` or `!` stays prose.
- **Model.** Add `CaptureKind::TaskComplete { raw, note, block_id, quoted }` in
  `capture_language/model.rs`.
- **Execution parse.** In `parse_capture_item` (`capture_language/item.rs`), claim at
  the `:` query position:
  - queries return the T1 usage error;
  - claimed invalid items return the `invalid_task_complete` usage error with the
    targeted hints;
  - complete tokens become `TaskComplete`.

  `@@` inheritance skips these items.

- **Temporary planner arm.** In `capture/plan.rs`, `TaskComplete` fails with the usage
  error `` `!note:block-id` completion is not available in this build yet``. `execute`
  replaces it. Add `capture_kind_label` → `"task_complete"`.
- **Editor parse** (`editor_model.rs`, `editor_parse.rs`).
  - Add `EditorMode::TaskComplete` (`"task_complete"`), `Need::TaskComplete`
    (`"task_complete"`), and `SpanKind` `task_complete_sigil` / `task_complete_note` /
    `task_complete_block_id`.
  - Add `parse_editor_task_complete_item` modeled on `parse_editor_task_link_item`, plus
    the claimed-invalid diagnostic.
  - Add the per-item `task_complete` object to `CaptureParseItem` in
    `src/native/capture_parse.rs`, using `skip_serializing_if`.
  - Update every exhaustive match: `rewrite.rs`, `capture_block_ids.rs`, the `@@` skip
    list, and so on.
- **Tests.**
  - Grammar unit tests cover every row of the contract table (`capture_language/tests/`:
    grammar, editor_modes, editor_spans with exact byte ranges, chain).
  - The `&` regression suite stays green.
  - New `tests/cli/capture/task_complete_parse.rs`:
    - capture-parse JSON for a query, a complete token (nested and quoted), and a
      claimed invalid item;
    - `bob capture` refusals exit 2 with exact messages, and no inbox file is created;
    - prose rows still capture as tasks;
    - `![[x#^y]]` still captures as prose.
- **Docs (`docs/capture.md`).**
  - Grammar-at-a-glance rows for `!` (query, complete, invalid).
  - A new `### Completing tasks with '!'` section after "Adding prerequisites with '&'"
    containing the Grammar contract, the quoting advice, and a placeholder sentence
    "execution lands with the task-complete writer".
  - The capture-parse mode, need, span, and object docs.
  - A Contents entry.
- **Verification.** `just fmt`, `just lint`, and `just test` pass.

# picker_contract

Work in bob-cli.

- **Catalog.** New `src/native/capture_completable_tasks.rs`:
  - `discover(bob_dir, today_day_file, draft_selected) -> Vec<CompletableTask>`. It
    reuses `capture_dependency_tasks`' note walk and exclusions, `note_tasks::scan`,
    `locator_for`, `replacement_for('!')`, and ID suggestions as the `&` catalog builds
    them.
  - Open statuses only.
  - Recurring detection uses the same rule as `execute` (put one shared helper in
    `task_complete`).
- **Today annotation.** Add `today_task_links(day_text, …)` to `task_complete` or the
  catalog module:
  - entries come from `capture_pomodoros::scan` (state, `is_current`, name, time range);
  - links come from `capture_pomodoro_close::links::wikilink_tokens`, with
    `strip_pomodoro_markers`, `range_is_struck` (skip struck), and
    `markdown::fenced_lines`;
  - skip Depends-On lines;
  - resolve with `VaultLinkResolver`;
  - assign roles, `sessions`, and the strongest placement per the contract.

  A missing day file means no today rows and no warning.

- **Order and ranking** exactly as the contract specifies, using the shared
  `capture_link_tasks::match_score`.
- **capture-complete wiring.**
  - Add `CompletionContext::TaskComplete` (`task_complete`) in
    `capture_language/completion.rs`, detected with the claim predicate before `^` and
    `&`. The replacement covers the token and the query is decoded.
  - Add `TaskCompleteCandidate` and a `Candidates` variant in
    `capture_complete/model.rs`.
  - Add `task_complete_candidates` in `candidates.rs`. The unfiltered snapshot comes
    back when the cursor sits at `replacement.start`.
  - Compute `already_selected` from the other claimed complete items in the draft,
    resolved, excluding the active item.
  - Add `PickerKind::TaskComplete` in the descriptor, with the continuation keys
    `["!", "["]` only for a bare whole-item `!`.
  - `query` is set for this context.
  - Update `engine.rs`, `render.rs`, and `shell.rs` (exhaustive matches).
- **Shell completion** (`capture_complete/shell.rs`): identified, enabled rows only,
  today rows first, under the existing 150 ms deadline.
- **capture-task-id.** Add `complete_replacement` to `CaptureTaskIDResult` for both
  `--route` and `--note-path`, plus its help text.
- **Tests.**
  - Catalog unit tests:
    - roles and precedence;
    - most-recent worked ordering;
    - `sessions` count;
    - struck, fenced, and Depends-On links ignored;
    - alias and embed links;
    - noted links;
    - hidden and recurring sinking;
    - empty-query order;
    - today-first query partition;
    - `already_selected`;
    - quoted locators.
  - CLI tests `tests/cli/capture/complete_task_complete.rs`:
    - JSON for `!`, `!fix`, and `!"Shop`;
    - descriptor ranges and continuation keys, present for `!` and absent for `!fix` and
      for a padded line;
    - a draft with two `!` items marking `already_selected`;
    - ID-less rows;
    - capture-task-id `complete_replacement`;
    - shell completion lines.
- **Docs.**
  - `docs/capture.md`: the `capture-complete` `task_complete` context, candidate fields,
    today annotation and order, and capture-task-id's new field.
  - `docs/completion.md`: `!` in the capture markers bullet.
- **Verification.** `just fmt`, `just lint`, and `just test` pass.

# execute

Work in bob-cli.

- **Planner.** Replace the temporary `TaskComplete` arm in `src/native/capture/plan.rs`
  with `plan_task_complete_item` in a new `src/native/capture/task_complete.rs`,
  following Execution semantics steps 1–7:
  - Resolve through the `&` path: `resolve_prerequisite_note` / `resolve_staged_note`,
    then `lookup_staged_task`. Share the batch's `DependencyContext` vault walk,
    building it lazily when the batch has `!` items.
  - Validate status and recurrence before any staging.
  - Run `complete_task_tree(…, Explicit)` through `SnapshotCloseVault`.
  - Compute `link_statuses` for today's staged day file from the staged vault view, then
    run `retire_completed_links` with the dedupe flag on.
  - Run `recover_blocked_dependents` over the walk's notes with staged overlays.
  - Stage every changed file through `CaptureBatchPlanner`.
  - Before staging, re-check each preimage the way `capture/dependencies.rs` does
    ("changed while planning").
  - When the task lives in today's daily note, apply the task edit and ledger retirement
    to the same staged text in sequence.
- **Results.**
  - Add the `task_complete` fields to `CaptureItemResult` (`capture/output.rs`, all
    constructors) and `Placement::Completed` (`"completed"`).
  - Feed `TaskBlockTracker` with roles `completed` and `unblocked`
    (`capture/task_blocks.rs`).
  - Pomodoro blocks keep auto-detection.
  - Reject `--route`, `--section`, `--task`, `--task-section`, and `--clip` on
    `task_complete` items with the `invalid_task_complete` message.
- **Human output.** Add `print_human_task_complete_success` per the contract, dispatched
  from `print_human_item_success`. Give `style_task_status_marker` the done-green `x`
  everywhere.
- **Help and docs.**
  - `bob capture --help` (`capture/cli.rs`): a paragraph after the `:` query paragraph
    with the syntax, the whole-item rule, the effects, single-quoting, and the
    `task_complete` kind.
  - `docs/capture.md`:
    - replace the placeholder with full Execution semantics, a worked example (a
      before/after ledger and note, a dry-run JSON excerpt, and human output), and the
      error list;
    - the kind list in "Input, stdin, and JSON output";
    - the `task_complete` result paragraph;
    - `task_blocks` roles.
  - `README.md` workflow step 3: one clause on completing work with
    `bob capture '!sase:fix-flaky'`.
- **CLI tests** (new `tests/cli/capture/task_complete.rs`, with Tasks settings,
  `BOB_NOW`, and `BOB_DAY_FILE`):
  - Next completes with a running-session strike;
  - a placeholder link moves to the running entry, and the emptied placeholder is
    removed;
  - nothing runs, and the link is deduped into the morning entry (bob-cli-2l);
  - embedded subtasks close, with a Blocked one left open;
  - a dependent unblocks;
  - a Blocked root completes;
  - already done is a no-op;
  - each refusal (Canceled, unknown status, recurring, missing note, ambiguous note,
    missing ID with suggestion, duplicate ID, forced flags) leaves the vault intact;
  - a batch of two completions plus `=x` in one draft;
  - the same task twice reports the second as `already_done`;
  - a failure in item 2 rolls item 1 back;
  - `capture_json_dry_run_matches_real`;
  - exact human output, both dry-run and real;
  - a task living in today's daily note.
- **Verification.** `just fmt`, `just lint`, and `just test` pass. Then run a manual
  sandbox check against a copy of the fixture vault and paste the human output into the
  bead notes.

# mac_preview

Work in bob-mac-capture. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<reason>"`) and use only the printed path. If the
linked checkout is unavailable on the host, `sase repo open gh:bobs-org/bob-mac-capture`
redirects or opens it.

- **Toolchain.** Run `swift build --target CaptureCore` and the `CaptureCoreTests`
  locally when a toolchain is present (export
  `PATH="$HOME/.local/share/swiftly/bin:$PATH"` explicitly in `sh` scripts). Treat macOS
  CI as the gate.
- **`CaptureModels.swift`.** Add `CaptureTaskComplete` with nested
  `Subtask`/`LeftOpen`/`Ledger`/`Move`/`Unblocked` structs, decoded with
  `decodeIfPresent` and safe defaults. Add `CaptureCommandSuccess.taskComplete`. Use a
  tolerant `try?` so a malformed object never fails the success decode. Accept
  `placement == "completed"`.
- **Highlighting.** In `captureSemanticCategory(forSpanKind:)`
  (`CompletionRowContent.swift`), map `task_complete_sigil` → `.pomodoroCloseComplete`,
  `task_complete_note` → `.route`, and `task_complete_block_id` → `.blockID`. Update the
  README color table.
- **New `CaptureTaskCompletePresentation.swift`** (CaptureCore, pure, public; nil unless
  `kind == "task_complete"`). It exposes:
  - `isDryRun`, `isAlreadyDone`;
  - `headline` (`Complete` / `Would complete` / `Already done`);
  - `destinationLabel` (`sase.md · ^fix-flaky`);
  - `transitionText` (`[*] → [x]  Fix flaky gkeep test`, reusing
    `CaptureTogglePresentation.marker` and `taskPreviewText`);
  - `previewText`;
  - `subtaskRows` and `leftOpenRows`;
  - `ledgerText` (tense-aware: `Strikes its Task Link in CAPTURE` / `Struck …`,
    `Moves its Task Link SASE → CAPTURE, struck · removes empty SASE`,
    `Task Link already in PLAN; drops the SASE copy`);
  - `unblockedRows`;
  - `chips`;
  - `primaryActionTitle` (`Complete`);
  - `statusText`, `notificationTitle`, `notificationBody`;
  - `previewAccessibilitySummary`, covering every fact.

  Wording mirrors bob's human output.

- **View** (`CapturePanelView.swift`). Add a `completePreviewItem` branch in
  `previewItem` before the toggle and link branches, per the Bob Mac Capture design:
  - green `checkmark.circle.fill` seal;
  - strikethrough plus secondary color on the post-arrow text;
  - fact rows with `list.bullet.indent`, `timer`, and `lock.open.fill`;
  - gray `Already done` styling.

  Task and Pomodoro block cards keep rendering below from the batch arrays.

- **Footer and notifications.**
  - `completeSubmit` status comes from the presentation for a single item.
  - `NotificationService.successPresentation` uses `Completed: <task>`, or
    `Completed N tasks` when every item is `task_complete`.
  - `friendlyKindLabel` gets an explicit `task_complete` label.
- **Fixtures and tests.**
  - Generate real-bob fixtures with bob-cli master's `bob` (it must include `execute`)
    against a sandbox vault, and record each generating command in the test file header:
    - running-session strike;
    - placeholder move plus removal;
    - dedupe;
    - subtasks plus left open;
    - unblocked;
    - already done;
    - a two-item batch;
    - a refusal (`ok: false`).
  - Add fake-bob routes for these drafts.
  - Tests:
    - `CaptureModelTests` decoding, including malformed or absent fields;
    - new `CaptureTaskCompletePresentationTests` (every text field, tense, already done,
      accessibility);
    - span mapping in `CompletionRowContentTests`;
    - a panel test that the completion card renders for a `task_complete` preview;
    - an `ImageRenderer` design render behind `BOB_MAC_CAPTURE_RENDER_DIR`, like the
      existing design tests.
- **README.** A "Completing tasks with `!`" section covering the preview, colors, and
  notifications.
- **Commit, then CI.** This plan explicitly instructs you to commit the bob-mac-capture
  changes with `/sase_git_commit` before waiting on CI. macOS CI only runs on the pushed
  commit, and agent hosts have no Apple Swift toolchain for the app target.
  - Use subject `feat(capture): preview task completions`. If `$SASE_BEAD_ID` is set,
    pass `-B keep`.
  - Then wait with `/sase_monitor` on the `CI` workflow run for that SHA
    (`gh run list --commit <sha> --limit 1`, then `gh run watch <id>`), with a timeout
    of at least 20 minutes.
  - If the run is red, read `gh run view <id> --log-failed`, fix forward, commit, and
    watch again.
  - The phase is done only when the latest run for its commit is green.

# mac_picker

Work in bob-mac-capture, opened as in `mac_preview`.

- **Source and need.**
  - Add `CapturePickerSource.taskComplete`:
    - `triggerByte` 33 (`!`);
    - placeholder `Search open tasks to complete`;
    - scope token `!` with caption `Complete`;
    - chip text `Pick a task to complete` with icon `checkmark.circle`;
    - key hints `Return Insert · ⇧Return Insert & next · ⌘Return Complete`;
    - its own UserDefaults key;
    - `locatorMarker` `:`.
  - Add `CapturePickerNeed.taskComplete`, add `task_complete` to `completionNeeds`, and
    add status text `Pick a task to complete — press Tab to browse`. Skip the doomed
    live dry run exactly as `task_link` does.
  - Add the `task_complete_*` span kinds to `completionSpanKinds` so Tab on a complete
    token reopens the picker.
  - Handle every exhaustive switch the compiler flags:
    - `CapturePickerIndex`;
    - `openPickerFromChip`;
    - `removePickerTrigger` (use Bob's `trigger_removal_range`);
    - `pickerMarkerHighlightRange`;
    - `cancelTaskIDPrompt`;
    - the `acceptPickerRow` announcement;
    - `CaptureCompletionContext`;
    - detail-strip `insertionText` / `teachingLine`.
- **Decoding.** Add the candidate fields `today` (`role`, `pomodoro`, `sessions`),
  `already_selected`, `recurring`, and `scheduled`, all `decodeIfPresent`. Decode
  `picker.kind == "task_complete"`.
- **Completion handler.** `handleTaskCompleteCompletion` is modeled on
  `handleDependencyCompletion`:
  - Bob's `query` seeds the filter;
  - stale-safe refetch at `replacement.start` with the same guards;
  - auto-open only on edit.
- **New `TaskCompletePickerPresentation.swift`** (CaptureCore,
  `TaskCompletePickerIndex`):
  - Rows are keyed `note_path|ref`.
  - **Empty-filter sections** in Bob's order:
    - per today Pomodoro: running with the NOW pill; completed recent-first with 🍅, the
      time range, and a `Done` capsule; queued with `UP NEXT` and its ordinal;
    - `In today's note`;
    - `In Progress`;
    - `Next`;
    - per-note sections titled with `note_path` and a `doc` icon.
  - **Filtered:** `Today` then `All open tasks`, each ranked by
    `FuzzyQuery(filter, leadingSigil: "!")` score with ties in Bob order.
  - **Rows:**
    - status glyph through `CaptureEditorPalette.taskStatus`, with Blocked as
      `pause.circle`;
    - text;
    - locator (accent note, `:`, indigo ID);
    - a `🍅 N` capsule when `sessions >= 2`;
    - a schedule capsule when present;
    - disabled rows show `disabled_reason` as a badge with `insertion == nil`;
    - ID-less rows use `pendingBlockID`.
  - **Detail strip:** `Inserts !sase:fix-flaky — completes it [*] → [x]`, a Blocked
    variant, and the disabled reason.
- **Keys** (`CaptureKeyCommandRouter`).
  - Shift-Return maps to a new `.acceptPickerRowAndContinue` for this source. It
    inserts, appends `\n\n!`, and re-analyses with completion requested so the fresh
    item auto-opens a new picker.
  - Command-Return keeps `acceptPickerRowAndSubmit`.
  - Bob's `action_continuation_keys` go through the existing `continuePickerOperator`.
- **ID-less flow.** Add `CaptureTaskIDPromptPurpose.taskComplete`, which assigns via
  `capture-task-id --note-path` and splices Bob's `complete_replacement`; never build it
  in Swift. The buttons are `Add ID & Insert` and `Add ID & Complete`.
- **Tests.**
  - `TaskCompletePickerPresentationTests` (inline JSON):
    - section order and titles for every role;
    - most-recent worked first;
    - sessions capsule;
    - filtered Today/All split;
    - disabled rows;
    - ID-less rows;
    - unknown group or role degrading to the note sections.
  - Router tests for Shift-Return and continuation keys.
  - Panel tests: `!` auto-opens the picker; accept inserts Bob's replacement;
    Shift-Return creates the next item and reopens; `!` then `[` hands off; the ID
    prompt splices `complete_replacement`.
  - Fake-bob routes for `!` completion JSON (fixtures from real bob).
  - A `CapturePickerDesignTests` render case.
- **README.** A "Complete Picker" section next to the Task Link Picker: trigger,
  sections, keys, bulk via Shift-Return, ID-less flow, and continuation keys.
- **Commit, then CI.** Same loop as `mac_preview`, with subject
  `feat(capture): open the Complete picker on !`. The phase is done only when the latest
  macOS run for its commit is green.

# Non-goals

- No Work Log entry, note text, or child bullets on a `!` item. A completion is the
  whole item; logging stays with `=x`.
- No `#pomodoro`, `=<X>`, or `=x` suffixes on `!`. Completing while closing stays
  `=x!N`.
- No new statuses, no change to `=x!N` output, and no change to reconcile behavior
  beyond the opt-in dedupe flag it does not pass. The bob-plugins JS half of bob-cli-2l
  stays with that bead.
- No status-group moves, `^prj` frontmatter, `#hide`, or archive writes from capture.
- No multi-select picker. Bulk is Shift-Return chaining plus blank-line items.
- No new CLI subcommands or options, and no SASE memory changes.

# Acceptance

- `bob capture '!sase:fix-flaky'` on a running session holding `[[sase#^fix-flaky]]`:
  - writes `[x]` with `  [completion:: <today>]` exactly as `=x!N` would;
  - strikes the link in place;
  - unblocks a dependent whose only prerequisite it was.

  The next `bob task reconcile` reports no ledger or status changes for these tasks.

- Completing a task that was worked this morning and carried to a placeholder, with
  nothing running, leaves exactly one struck link in the morning entry and removes the
  emptied placeholder.
- `!`, `!fix`, and padded or suffixed items fail with teaching errors and never create
  inbox tasks. `![[x#^y]]` and prose starting with `!` capture exactly as before.
- Bulk drafts complete every task or none. The same task twice is reported as already
  done.
- `capture-complete` for `!` lists today's tasks first, grouped by Pomodoro role, then
  In Progress, Next, and every other open task in the vault. Filtering keeps Today
  matches on top.
- In Bob Mac Capture:
  - typing `!` opens the Complete picker with Today sections;
  - Return inserts `!note:block-id`;
  - the preview shows the struck task, ledger effect, and unblocked tasks;
  - Return completes;
  - Shift-Return chains the next pick;
  - `![[` still reaches embed completion.
- bob-cli `just fmt`, `just lint`, and `just test` pass, apart from any recorded
  pre-existing lint issue. bob-mac-capture macOS CI is green for the landed revisions.
