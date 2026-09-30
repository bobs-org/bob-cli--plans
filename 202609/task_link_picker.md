---
tier: epic
title: '`:` Task Link Picker for bob capture and Bob Mac Capture'
goal: 'Typing `:` at the start of any capture item, including any item of a blank-line-separated
  batch draft, opens a fast, beautiful fuzzy picker over every open (not done or canceled)
  task in the vault''s area and project notes. Accepting a task replaces the `:` query
  with the canonical `@route:block-id` link. Tasks without a block ID get one first,
  from a prefilled suggestion. The link can then be captured as-is or started with
  `=`, so any task can be linked into today''s Pomodoros in a few keystrokes.

  '
phases:
- id: discovery
  title: Vault-wide linkable-task discovery and fuzzy ranking
  depends_on: []
  size: medium
  description: 'discovery: add the read-only `capture_link_tasks` scanner. It collects
    Ready, Blocked, Next, and In Progress tasks from area, project, and inbox notes.
    It adds groups, the canonical order, ledger annotation, block-ID suggestions,
    schedule pull-forward flags, and a deterministic tiered fuzzy ranker, all with
    unit tests.'
- id: grammar
  title: '`:` picker-query grammar in capture, capture-parse, and completion fields'
  depends_on: []
  size: medium
  description: 'grammar: add one claim predicate for single-token `:` items. Execution
    rejects them with teaching errors T1 and T2. capture-parse reports `incomplete`
    with `needs: ["task_link"]`. The completion field (`task_link` context) covers
    the sigil. Also update help text and add a claim-equivalence test.'
- id: mac_core
  title: CaptureCore task-link picker index, source, and decoding
  depends_on: []
  size: medium
  description: 'mac_core: in bob-mac-capture''s CaptureCore, decode the new candidate
    fields and add the `taskLink` picker source and need. Add `TaskLinkPickerIndex`
    with grouped and fuzzy-filtered presentations, rows keyed by route and ref, and
    a `:`-aware FuzzyQuery. Cover it with unit tests and verify on macOS CI.'
- id: complete
  title: '`task_link` completion candidates, round-trip tests, and docs'
  depends_on:
  - discovery
  - grammar
  size: medium
  description: 'complete: wire discovery into the `task_link` context of `bob capture-complete`,
    with a pinned JSON schema, human rows, and help. Prove accept-then-capture round
    trips (including capture-task-id for tasks without an ID), check real-vault latency,
    and document the feature in docs/capture.md and README.md.'
- id: mac_panel
  title: Bob Mac Capture task-link picker panel, ID prompt, and keys
  depends_on:
  - complete
  - mac_core
  size: medium
  description: 'mac_panel: route the `task_link` context into the picker card. Accept
    inserts `@route:id`, Shift-Return inserts it with `=`, and Command-Return inserts
    it and captures. Tasks without an ID open a prefilled Add block ID prompt that
    assigns the ID and splices the link, and Escape returns to the picker. Also update
    the views, README, real-bob fixtures, and tests, and get macOS CI green.'
proposed_by: bbugyi200.apollo.3j
create_time: 2026-09-30 13:00:13
status: done
bead_id: bob-cli-2v
---

- **PROMPT:** [prompts/202609/task_link_picker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/task_link_picker.md)
- **BEAD:** [bob-cli-2v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2v/README.md)

# Plan: `:` Task Link Picker

## Background

Bob already links existing tasks into today's Pomodoro ledger. A whole capture item that
is only `@route:block-id[#pomodoro][=<X>]` (or its active-task spelling
`^route:block-id…`) makes the task Next and adds or moves its `[[route#^id]]` Task Link.
With `=<X>` it also starts that session atomically. Two problems make this hard to use
today:

1. **You must already know the route and block ID.**
   - `@route:` completion only searches one note that you have already chosen.
   - The `^` picker (`active_task` context) only lists In Progress, Next, and Ready
     `#now` tasks that already have IDs.
2. **Most tasks have no block ID.** Surveying the real vault on 2026-09-30:
   - 95 vault-root area or project notes hold about 590 open tasks: 290 Blocked `[?]`,
     223 Ready `[ ]`, 52 In Progress `[/]`, and 26 Next `[*]`.
   - About 65% of those tasks have no `^block-id`.
   - `sase.md` alone holds about 258 of them.
   - Done and canceled projects currently hold 0 open tasks.
   - Every area and project note is a vault-root file, so every one is addressable by
     `@route`.

Bryan wants to type `:` at the very start of an input and fuzzy-search any open task in
any area or project note. The chosen task becomes the `@file:id` link. Bulk
(blank-line-separated) drafts must work too.

### How the pieces fit today (bob-cli, repo-relative paths)

- **Grammar**, `src/native/capture_language/`:
  - **Execution precedence** is in `item.rs`, in the function that calls
    `parse_pomodoro_equals_item`, then `parse_pomodoro_adjust_item`, then
    `parse_pomodoro_link_item` (lines ~57–80), and then falls through to `resolve_line`.
  - **The `^` family** lives in `tokens.rs`:
    - `classify_caret_token` and `classify_caret_item` handle the editor side.
    - `parse_pomodoro_link_item` handles execution.
    - A partial `^` shape is an incomplete editing state when it is the whole item and
      prose otherwise.
    - `POMODORO_LINK_INCOMPLETE_ERROR` is defined in `markers.rs`.
  - **Editor parse**, `editor_parse.rs`:
    - `parse_editor_item` has early returns for `now`-tag, close, and adjust items.
    - The caret `Partial` arm sets `mode = Incomplete`, `needs = [ActiveTask]`, and one
      `InteractivePlaceholder` span over the token.
    - The `@@` inheritance loop in `parse_for_editor` (~line 262) skips session modes.
  - **`Need` / `SpanKind`** are in `editor_model.rs`.
  - **Completion**, `completion.rs`:
    - `CompletionContext` enum and `completion_field_at`.
    - `caret_completion_field_at` is the `^` path. It only fires on the item's leading
      line and returns `None` when the cursor is at `first.start`.
- **Completion service**, `src/native/capture_complete.rs`:
  - `build_result` has an exhaustive match on the context.
  - Candidates are the untagged enum `Candidates::*`, and `SCHEMA_VERSION` is 1.
  - `active_task_candidates` and `link_candidates` are the models to follow. The latter
    filters on `matches!(status_symbol, ' ' | '?' | '*' | '/')`, the same set that
    `src/native/capture/pomodoro_link.rs:276` accepts.
  - Human rows are built in `candidate_lines`.
- **Discovery building blocks**:
  - `capture_targets::scan_capture_targets` gives, in this order: `mac_inbox` (inbox),
    area notes A→Z, then non-terminal project notes A→Z, with `CaptureTargetKind`.
    Project notes whose status is `done`, `canceled`, or `cancelled` are omitted.
  - `note_tasks::{read_settings, scan}` provides `NoteTask`, which carries:
    `status_symbol`, `status_name`, `status_type`, the cleaned `description`,
    `block_id`, `section`, `indentation`, `line_index`, and `task_ref()`.
  - `capture_tasks::{status_type_label, indentation_depth}`.
  - `capture_active_tasks::read_ledger` returns `Ledger { owners, positions }`.
    `positions` is private today.
  - `plan_budget::has_now_tag`.
  - `capture_block_ids::suggest_ids(body, marker, used_ids)` returns at most 3
    deterministic suggestions. For example, `Fix flaky gkeep test` gives
    `fix-flaky-gkeep` and `flaky-gkeep-test`.
  - `collect_done::block_ids_in_markdown` is the exact used-ID set that
    `bob capture-task-id` checks for duplicates.
  - `capture_task_toggle::find_single_future_scheduled_field` (private) decides the
    scheduled-date pull-forward that linking performs.
- **`bob capture-task-id --route R --task-ref REF --block-id ID`** is the one write that
  names an ID-less task. The ref is stale-safe.

### Bob Mac Capture (separate repo)

The app never parses grammar.

- **Opening a picker.**
  - It calls `capture-parse` on each edit, debounced by 50 ms.
  - It requests `capture-complete` only when `shouldRequestCompletion` passes: the needs
    set, or the caret inside a completion span such as `interactive_placeholder`.
  - It routes the `context` in `CapturePanelModel.handleCompletionResponse`.
- **The `^` picker** (`ActiveTaskPickerIndex` plus `CapturePickerCard`):
  - It is a full card: filter bar, grouped sticky sections, 34 pt rows with fuzzy
    highlights, a detail strip, and a key-hint footer.
  - It refetches the full list at `r.start`, then filters locally with `FuzzyMatcher` on
    every keystroke in a separate filter field. Multi-word queries therefore work once
    the card is open.
  - Accept splices the row's `insertion` into Bob's replacement range.
- **Two things break on tasks without IDs**:
  - Both picker indexes deduplicate and key rows on `replacement`, which is `""` for
    ID-less tasks.
  - The existing **Add block ID** prompt (`presentTaskIDPrompt`,
    `completeTaskIDAssignment`) ignores `block_id.suggestions` and splices only the bare
    ID.
- **Swift is not installed on the Linux hosts.** CI (`.github/workflows/ci.yml`,
  macOS 26) runs format-lint, build, test, and bundle.

## Design

### Principles

- **One sigil, one meaning.** `:` at the start of an item means "find a task to link."
  - It is only a picker query. It is never executable and never a third spelling of the
    link.
  - Accepting always produces the canonical `@route:block-id` that Bryan asked for, and
    the existing link, start (`=<X>`), and `#pomodoro` semantics apply unchanged.
- **Bob owns truth; the app owns feel.** Bob decides:
  - what is claimed,
  - which tasks are linkable,
  - their order and groups,
  - ID suggestions,
  - pull-forward effects.

  The app owns local fuzzy filtering, highlighting, keys, and the prompt, exactly like
  the `^` picker.

- **Reliable by construction.**
  - A single claim predicate is shared by execution, the editor, and completion.
  - Only statuses the link path accepts are listed.
  - IDs are validated against the same used-ID set that `capture-task-id` checks.
  - Accepting a row can only produce a link that `bob capture` will resolve.
  - A `:` query can never silently become a junk inbox task.

### Grammar (fixed across phases)

**Claim predicate.** An item is a **task-link query** exactly when both hold:

- the item has one physical line;
- that line (ignoring leading and trailing whitespace) is a single whitespace-delimited
  token that starts with `:`.

The token is the sigil `:` plus a query of any non-whitespace characters, possibly
empty. Everything else is unchanged ordinary grammar.

| Draft item                           | Reading                                                 |
| ------------------------------------ | ------------------------------------------------------- |
| `:`                                  | query `""` (picker opens on the full list)              |
| `:dee`, `:sase:out`, `:déjà`, `:)`   | query `dee`, `sase:out`, `déjà`, `)`                    |
| `:sase:deep-fix`, `:sase:deep-fix=`  | query (execution teaches `@sase:deep-fix…`; see T2)     |
| `: dee`, `:dee more`, `Buy :dee`     | prose: several tokens, unchanged                        |
| `:dee` + child line `- x`            | prose: several lines, unchanged                         |
| `:dee` as the second item of a batch | query (each item is judged alone; bulk works)           |
| `=x`, blank line, `:dee`             | a close, then a query item (a chain needs a blank line) |
| `@@work`, blank line, `:dee`         | query; a `@@` declaration never applies to it           |

The only behavior change is that a single-token, single-line item starting with `:`
(`bob capture ':)'`) used to capture a prose task in `mac_inbox.md`. It is now a
write-free error. This is deliberate, so that a picker query that was never finished
cannot create a junk task.

**Execution** (`bob capture`, and `--dry-run`) rejects a claimed item before any other
item parser, so the whole batch rolls back. Forced flags do not change this. Errors keep
the usual `capture item K starting on line L: …` prefix. The texts are fixed:

- **T1**:
  `` `{token}` opens the task picker; pick a task to insert its `@<route>:<block-id>` link, or write the link yourself (for example `@sase:deep-fix`) ``
- **T2** is used when `^{query}` parses as a complete caret link through the existing
  `parse_caret_link_token`, that is `route:block-id[#name][=<X>|=x…]`:
  `` `{token}` opens the task picker; to link that task, write `@{query}` ``
  - Example: `:sase:deep-fix=` gives ``…, write `@sase:deep-fix=` ``.

**Editor** (`bob capture-parse`), for a claimed item:

- `mode: "incomplete"`, `needs: ["task_link"]`
- `body: ""`, `route: null`, `section: null`, `block_id: null`
- no diagnostics
- one `interactive_placeholder` span over the whole token, sigil included

This holds at the top level (for a single-item draft) and in that item's `items[]`
entry. There is no new span kind, so older apps still request completion there.

### Discovery and ordering (fixed across phases)

**Notes.** Use the `capture_targets::scan_capture_targets` targets: `mac_inbox` (inbox,
only if the file exists), area notes, and non-terminal project notes. This is the same
set that `@` route completion offers.

- `targets.issues` become bounded warnings.
- A missing inbox file is silent.
- An unreadable note becomes one bounded warning, and the other notes are still listed.

**Tasks.**

- Each note is scanned with `note_tasks::scan` (the `#task` global filter and statuses
  come from the Tasks plugin settings).
- Keep tasks whose `status_symbol` is `' '`, `'?'`, `'*'`, or `'/'`. These are exactly
  the statuses the link path accepts. Extract that set as one shared
  `pub(crate) fn is_linkable_status(symbol: char) -> bool` and use it in
  `pomodoro_link.rs`, `link_candidates`, and the new scanner.
- Done, canceled, and unknown statuses are excluded.
- Nested subtasks are included, with their `depth`.

**Groups.** Assign each task one `group`, in this precedence:

1. `queued`: an identified task whose dedicated `[[route#^id]]` link sits under an open
   Pomodoro today (`read_ledger` owners).
2. `in_progress`: `/`.
3. `next`: `*`.
4. `now`: Ready or Blocked with `#now`.
5. `note`: everything else.

**Canonical order** (the empty query):

- `queued` in ledger order (entry order, then child order);
- then `in_progress`, `next`, and `now`, each ordered by route and then line, exactly
  like the `^` picker;
- then `note` tasks grouped by note in the targets order (inbox, areas A→Z, projects
  A→Z), in document order within each note.

**Per-task facts**:

- `now`: `has_now_tag(text)`.
- `scheduled`: the first strict `[scheduled:: YYYY-MM-DD]` value on the raw line, or
  null.
- `pulls_forward`: true exactly when
  `find_single_future_scheduled_field(raw_line, today)` is `Some`. Make it `pub(crate)`.
  `today` comes from the capture clock (`BOB_NOW`, else the local date), the same clock
  `bob capture` uses. This is the date that linking would retire.
- `block_id_suggestions`, for ID-less tasks only: `suggest_ids(text, ':', used)`, where
  `used` is `collect_done::block_ids_in_markdown` of that note. Every suggestion is
  therefore free in that note at scan time.

**Ranking** (Bob side; the CLI and the app's partial-snapshot fallback use it):

- Split the query into lowercase whitespace-separated terms. Every term must match.
- A term's tier is its best match over these fields: `route:block-id` (identified only),
  `block_id`, `text`, `route`, `section`, and the Pomodoro name. Tiers, best first:
  - 3: prefix of the field;
  - 2: prefix of a word, meaning the preceding character is not alphanumeric;
  - 1: substring;
  - 0: in-order subsequence.
- Order by the sum of tiers, descending, then by canonical order. The sort is stable.
- An empty query keeps the canonical order.

The app ranks locally with its own `FuzzyMatcher`. The two rankers need not agree byte
for byte.

### `capture-complete` contract (fixed across phases; additive, `schema_version` stays 1)

**Context `task_link`.** It applies on a claimed item when the cursor is anywhere in
`[token.start, token.end]`. That includes a cursor just before the sigil, so clients can
refetch the full list at `replacement.start`.

- `replacement` = `{token.start, token.end}`. It **includes the `:` sigil**, because
  accepting rewrites the query into the canonical marker.
- The query is the token text between the sigil and the cursor. It is empty when the
  cursor is at or just after the sigil.
- ID-less tasks are always included. `--all-tasks` does not affect this new context.
- There is no top-level `block_id` object. Bounded `warnings` carry ledger, target, and
  read warnings.

**Candidate** (every key is always present unless it is marked omittable):

```json
{
  "replacement": "@sase:deep-fix",
  "ref": "7:1a2b3c4d",
  "route": "sase",
  "note_kind": "project",
  "block_id": "deep-fix",
  "requires_block_id": false,
  "block_id_suggestions": [],
  "status_symbol": "*",
  "status_name": "Next",
  "status_type": "on_hold",
  "text": "Fix deep bug",
  "section": "Bugs",
  "depth": 0,
  "line": 7,
  "group": "queued",
  "scheduled": null,
  "pomodoro": { "line": 5, "name": "BUGS", "time_range": null, "is_current": false }
}
```

Field notes:

- `status_type` comes from `capture_tasks::status_type_label`.
- `note_kind` is `inbox`, `area`, or `project`.
- `group` is `queued`, `in_progress`, `next`, `now`, or `note`.
- `now: true` and `pulls_forward: true` are omitted when false.
- `pomodoro` has the same shape as `active_task` and is non-null only for `queued`.
- For an ID-less task: `replacement: ""` (clients must never insert it),
  `block_id: null`, `requires_block_id: true`, and `block_id_suggestions` holds up to 3
  suggestions, possibly `[]`.

**Human rows** (`candidate_lines`) put the left column in cyan and the rest dim:

- Identified: `@sase:deep-fix` then `[*] Fix deep bug  · BUGS`. The tail is the Pomodoro
  name, `Planned`, `In Progress`, `Next`, `#now`, or the note label (`sase.md`).
- `  · scheduled 2026-10-03` is appended when `scheduled` is set.
- ID-less: `@sase:…` then
  `[ ] Fix flaky gkeep test  · sase.md · needs ID (^fix-flaky-gkeep)`.

### Worked example (shared by the Bob phases)

`BOB_NOW=2026-09-30 09:02:00`, standard Tasks settings (the `active_task` test settings,
including `?` Blocked), and TAB indentation.

- `mac_inbox.md` (`type: "[[area]]"`): `- [ ] #task Call the bank [created::2026-09-29]`
- `health.md` (area): `## Errands` then
  `- [?] #task Book dentist [scheduled::2026-10-03]`
- `bob.md` (project, `status: wip`): `- [ ] #task Polish capture picker ^polish`, then a
  TAB-indented `- [ ] #task Tune fuzzy weights`
- `sase.md` (project, wip), with these lines:
  - `## Bugs`
  - `- [*] #task Fix deep bug ^deep-fix`
  - `- [ ] #task Fix flaky gkeep test`
  - `- [x] #task Old fix ^old-fix`
  - `## Writing`
  - `- [/] #task Draft outline ^outline`
  - `- [ ] #task Ship blog post #now ^blog`
  - `- [-] #task Dropped idea`
- `archive.md` (project, `status: done`): `- [ ] #task Leftover ^leftover`. It is
  excluded.
- `scratch.md` (no frontmatter): `- [ ] #task Loose end ^loose`. It is excluded.
- Ledger `2026/20260930.md`: `## Pomodoros`, `- [ ] () — BUGS`, then TAB
  `- [[sase#^deep-fix]]`.

`bob capture-complete -c 1 -f json -- ':'` gives 8 candidates, in this order:

| #   | `replacement`    | group         | Notes                                                               |
| --- | ---------------- | ------------- | ------------------------------------------------------------------- |
| 1   | `@sase:deep-fix` | `queued`      | `pomodoro.name` = `BUGS`                                            |
| 2   | `@sase:outline`  | `in_progress` |                                                                     |
| 3   | `@sase:blog`     | `now`         | `now: true`                                                         |
| 4   | `""`             | `note`        | mac_inbox, Call the bank, `requires_block_id`                       |
| 5   | `""`             | `note`        | health, Book dentist, `scheduled` 2026-10-03, `pulls_forward: true` |
| 6   | `@bob:polish`    | `note`        |                                                                     |
| 7   | `""`             | `note`        | bob, Tune fuzzy weights, `depth: 1`                                 |
| 8   | `""`             | `note`        | sase, Fix flaky gkeep test, suggestions start `fix-flaky-gkeep`     |

More queries against the same vault:

- `-c 4 -- ':dee'` returns only row 1, with `replacement` `{0,4}`.
- `':bug'` returns rows 1 and 8, through section `Bugs` and the name `BUGS`.
- `':sase out'` is not claimed (two tokens), but the ranker unit test uses the query
  `sase out` and gets `@sase:outline` only.
- `':dntst'` returns Book dentist through a subsequence match.
- Accepting row 1 turns the draft into `@sase:deep-fix`, and `bob capture` then succeeds
  with `pomodoro_link`, `already_current`.
- For row 8,
  `bob capture-task-id --route sase --task-ref <ref> --block-id fix-flaky-gkeep`
  followed by capturing `@sase:fix-flaky-gkeep=` links the task, makes it Next, and
  starts BUGS.

### Bob Mac Capture experience

The app reuses the `^` picker's card. The new source is `taskLink`, with scope capsule
`:` and caption **Open Tasks**:

```text
┌─────────────────────────────────────────────────────────────────────┐
│ [: Open Tasks]  Search tasks by text, note, or ^id          8 tasks │
├─────────────────────────────────────────────────────────────────────┤
│ ① BUGS                                                       1 task │
│   ◉ Fix deep bug                                      sase:deep-fix │
│ In Progress                                                  1 task │
│   ◐ Draft outline                                      sase:outline │
│ This Week's Bets                                             1 task │
│   ○ Ship blog post #now                          NOW      sase:blog │
│ ▣ mac_inbox.md · Inbox                                       1 task │
│   ○ Call the bank                              mac_inbox: ⊕ call-bank│
│ ▣ health.md · Area                                           1 task │
│   ⏸ Book dentist                           📅 Oct 3  health: ⊕ book-… │
│ ▣ bob.md · Project                                          2 tasks │
│   ○ Polish capture picker                               bob:polish │
│       ○ Tune fuzzy weights                       bob: ⊕ tune-fuzzy-… │
├─────────────────────────────────────────────────────────────────────┤
│ Fix deep bug                                                        │
│ Next · sase.md › Bugs · Queued in BUGS (#1)                         │
│ ↩ inserts @sase:deep-fix · ⇧↩ adds = to start it                    │
├─────────────────────────────────────────────────────────────────────┤
│ ↑↓ Move   ↩ Link   ⇧↩ Link & Start   ⌘↩ Link & Capture   esc Clear  │
└─────────────────────────────────────────────────────────────────────┘
```

**Typing.**

- Typing `:` on an empty item auto-opens the card on the full list. The seed query is
  any text typed after `:` before the card appeared.
- Typing then filters locally. Spaces separate AND terms, and matched characters are
  highlighted.
- Tasks in the filtered view show a Pomodoro chip when they are queued.

**Keys.**

| Key                  | Effect                                                                                                                           |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| ↩ / Tab             | Replace the `:` query with `@route:block-id`                                                                                     |
| ⇧↩                  | Insert `@route:block-id=`. The live preview then shows the start, or "finish the current Pomodoro first", before Return captures |
| ⌘↩                  | Insert, then capture                                                                                                             |
| Esc                  | Clear the filter, then cancel to the reopen chip                                                                                 |
| ⌫ on an empty filter | Remove the `:` token                                                                                                             |

**ID-less rows** show `route:` plus a dim `⊕ suggested-id` locator. Accepting one opens
the Add block ID prompt in link mode:

```text
┌ Add block ID ───────────────────────────────────┐
│ ○ Fix flaky gkeep test            sase.md › Bugs │
│ ^ [fix-flaky-gkeep                  ]  (selected) │
│   ✦ fix-flaky-gkeep   ✦ flaky-gkeep-test          │
│ Inserts @sase:fix-flaky-gkeep                     │
│ Letters, numbers, and hyphens                     │
│                     [Cancel]   [Add ID & Link]    │
└──────────────────────────────────────────────────┘
```

- **Return** runs `bob capture-task-id` and then splices `@sase:fix-flaky-gkeep`. It
  appends `=` if the prompt was opened with ⇧↩, and it captures if it was opened with
  ⌘↩.
- **Tab / ⇧Tab** cycle the suggestions.
- **Esc** returns to the picker with the same filter and selection.

### Decisions flagged for review

1. **Done and canceled projects are excluded**, which matches `capture-targets` and `@`
   route completion. Today that excludes 0 open tasks. Including them would be a
   one-line change to the note set.
2. **A single-token, single-line `:…` item is always a picker query.**
   `bob capture ':)'` is now a teaching error instead of a prose inbox task.
3. **Accept rewrites `:query` to `@route:block-id`.** `:route:id` is never executable,
   and T2 teaches the `@` spelling.
4. **ID-less tasks are named before linking.** The prompt is prefilled with the first
   suggestion, so this costs one extra Return. As with today's Add block ID flow, the ID
   write lands before the capture is submitted and persists even if the draft is later
   discarded.
5. **⇧↩ inserts `=` but does not capture**, so the start or its guard error is
   previewed before anything is written.

## Phase `discovery`: linkable-task scanner and ranker (bob-cli)

1. Create `src/native/capture_link_tasks.rs`, registered in `src/native.rs`, with a
   module doc comment in the style of `capture_active_tasks.rs`.
   - `#[serde(rename_all = "snake_case")] pub(crate) enum LinkTaskGroup { Queued, InProgress, Next, Now, Note }`.
   - `pub(crate) struct LinkTask` with fields:
     - `route`
     - `note_kind: CaptureTargetKind`
     - `block_id: Option<String>`
     - `block_id_suggestions: Vec<String>`
     - `status_symbol`, `status_name`, `status_type: &'static str`
     - `text`, `section`, `depth`, `line` (1-based)
     - `task_ref`
     - `now`, `scheduled: Option<String>`, `pulls_forward`
     - `pomodoro: Option<ActiveTaskPomodoro>`
     - `group`
   - `LinkTask::replacement()` returns `@route:id`, or `""`.
   - `pub(crate) struct LinkTaskResult { tasks, warnings }`.
   - `pub(crate) fn discover(bob_dir)` uses `pomodoro::day_file_for` and the capture
     clock.
   - `pub(crate) fn discover_at(bob_dir, day_file, today)` is for tests.
   - `pub(crate) fn rank<'a>(tasks: &'a [LinkTask], query: &str) -> Vec<&'a LinkTask>`.
2. Implement the note set, filtering, groups, canonical order, per-task facts, and
   ranking exactly as the Design describes.
   - Extract `is_linkable_status` and use it at the three call sites.
   - Make `find_single_future_scheduled_field` `pub(crate)`. Expose ledger positions,
     for example as `pub(crate) fn position(&self, key) -> Option<(usize, usize)>`,
     rather than duplicating `read_ledger`.
   - Share or reuse `bounded_warning`. Do not add a third copy.
3. **Unit tests** in the module, using temp vaults in the `capture_active_tasks` test
   style:
   - The worked-example order and every field of all 8 rows.
   - Exclusions: done and canceled statuses, an unknown status symbol, the terminal
     project, the untyped note, a nested-directory note, and a non-routable root file.
   - Suggestions:
     - they avoid a used ID, including a non-task `^anchor` line;
     - identified tasks get `[]`.
   - `pulls_forward`:
     - a future date gives true;
     - a past date and two scheduled fields give false.
   - A missing ledger or missing `## Pomodoros`: a warning, nothing is `queued`, and the
     tasks are still listed.
   - An unreadable note gives a warning and the others are still listed.
   - Ranker:
     - each tier;
     - AND across terms;
     - subsequence (`dntst`);
     - stability of ties;
     - an empty query keeps the order;
     - a `route:block-id` prefix (`sase:out`).
4. `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test`
   must be clean, apart from pre-existing warnings. Nothing wires into
   `capture-complete` in this phase, except that `link_candidates` uses the extracted
   predicate.

## Phase `grammar`: the `:` query claim (bob-cli)

1. **Claim predicate.** Add it once, in `tokens.rs` or a new
   `capture_language/task_link.rs`:
   `pub(super) fn task_link_query_token<'a>(item: &CaptureItem<'a>) -> Option<Token<'a>>`.
   It implements the Design's rule. Execution, editor, and completion must all call it.
   No other code may re-derive the rule.
2. **Execution.** Add the T1 and T2 builders to `markers.rs`.
   - In `item.rs`, check the predicate before `parse_pomodoro_equals_item`, and return
     `Err(T1|T2)`. T2 is chosen when
     `parse_caret_link_token(&format!("^{query}")).is_ok()`.
   - Update the precedence doc comment in `parse_pomodoro_equals_item`.
3. **Editor.**
   - Add `Need::TaskLink` (label `task_link`) in `editor_model.rs`.
   - Add an early return in `parse_editor_item` that produces the Design's incomplete
     outcome, with one `InteractivePlaceholder` span over the token.
   - Make sure the `@@` inheritance loop never applies to it. Extend the skip condition,
     or mark the item as having a local destination.
4. **Completion field.**
   - Add `CompletionContext::TaskLink` (`task_link`) in `completion.rs`.
   - Add `task_link_completion_field_at`, and call it in `completion_field_at` on the
     leading line, before the caret path. It returns the query and `replacement` as the
     Design describes, including a cursor at `token.start`.
   - In `capture_complete.rs`, add the arm to every exhaustive match and to
     `context_label` (`"task_link"`). For now it returns empty candidates
     (`Candidates::Task(vec![])`). Phase `complete` replaces that.
5. **Help.**
   - `src/native/capture/cli.rs` `long_about`: add one short paragraph. A lone `:` item
     is a task-picker query, it is never captured, and accepting it in a picker inserts
     `@route:block-id`.
   - `src/native/capture_parse.rs` `long_about` and the `Needs:` list: add `task_link`.
   - Keep option lists alphabetical, per the CLI rules.
   - Update `tests/cli/help.rs` if it pins these texts.
6. **Tests.**
   - `capture_language/tests/grammar.rs`: every row of the claim table. Exact T1 and T2
     texts, including `:sase:deep-fix`, `:sase:deep-fix=`, `:sase:deep-fix#bugs=3`, and
     `:sase:deep-fix=x1`. The prose rows are byte-identical to today.
   - `tests/editor_modes.rs` / `editor_spans.rs`:
     - the mode, needs, and span for `:`, `:dee`, `  :dee`, and `:déjà`;
     - a batch `Buy milk\n\n:dee` with a correct `items[]`;
     - `@@work\n\n:dee`, where the item stays incomplete with `route: null`.
   - `tests/completion.rs`:
     - the query and replacement at the cursor before the sigil, just after it, in the
       middle, and at the end;
     - the second item of a batch;
     - multi-token and multi-line cases give `None`.
   - **Claim-equivalence test.** Over one table of about 20 inputs, these three are
     equivalent: execution returns T1 or T2, the editor needs `task_link`, and
     completion at the token end gives `task_link`.
   - `tests/cli/capture/`:
     - `bob capture ':dee'` exits with the usage error code and T1 on stderr;
     - `-f json` gives `{"ok":false,…}`;
     - `bob capture` with `Buy milk\n\n:dee` writes nothing (batch rollback);
     - `capture-parse -f json -- ':dee'` shape.
7. `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test`
   must be clean.

## Phase `mac_core`: CaptureCore picker model (bob-mac-capture)

**Repository.**

- Open it with `/sase_repo`: `sase repo open bob-mac-capture`. On Linux hosts the linked
  checkout is missing, so fall back to `sase repo open gh:bobs-org/bob-mac-capture`.
- Use only the printed path, and read its `AGENTS.md` if one exists.
- The contract comes from the Design. Hand-written JSON samples must match it exactly.
  Phase `mac_panel` later regenerates real-bob fixtures.

1. **Decoding** (`Sources/CaptureCore/CaptureModels.swift`):
   - Add these to `CaptureCompletionCandidate`, using `decodeIfPresent` with defaults:
     - `noteKind: String?` (`note_kind`)
     - `blockIDSuggestions: [String]` (`block_id_suggestions`, `[]`)
     - `group: String?`
     - `scheduled: String?`
     - `pullsForward: Bool` (`pulls_forward`, false)
   - `requiresBlockID`, `depth`, `line`, `taskRef`, `now`, and `pomodoro` already exist;
     reuse them.
   - Add `CaptureCompletionContext.taskLink` (`"task_link"`) in
     `CompletionRowContent.swift`, so a stray inline rendering reads "Task" rather than
     neutral.
2. **Source and need** (`CapturePickerPresentation.swift`):
   - Add `CapturePickerSource.taskLink`, with an arm in every switch:

     | Property                     | Value                                                                                                            |
     | ---------------------------- | ---------------------------------------------------------------------------------------------------------------- |
     | `triggerByte`                | `58`                                                                                                             |
     | `filterPlaceholder`          | "Search tasks by text, note, or ^id"                                                                             |
     | `filterAccessibilityLabel`   | "Search open tasks"                                                                                              |
     | `scopeSymbolText`            | `:`                                                                                                              |
     | `scopeCaption`               | "Open Tasks"                                                                                                     |
     | `chipLabel`                  | "Browse open tasks"                                                                                              |
     | `chipIcon`                   | `magnifyingglass`                                                                                                |
     | `chipHelp`                   | "Reopen the Task Link Picker for the : item (Tab)."                                                              |
     | `cardAccessibilityLabel`     | "Task link picker"                                                                                               |
     | hint                         | "Arrow keys move, Return inserts the task link, Shift-Return inserts it and starts its session, Escape cancels." |
     | `appearedAnnouncementPrefix` | "Open tasks"                                                                                                     |
     | `pickerUsedDefaultsKey`      | `org.bobs.bob-mac-capture.task-link-picker-used`                                                                 |

   - `keyHintItems`: `("↑↓","Move")`, `("↩","Link")`, `("⇧↩","Link & Start")`,
     `("⌘↩","Link & Capture")`, `("esc","Clear / Cancel")`. Add a matching
     accessibility label.
   - Add `CapturePickerNeed.taskLink`, with status "Pick any open task — press Tab to
     browse". Precedence: `activeTask`, `taskLink`, `pomodoroID`, `blockID`,
     `pomodoroStart`.
   - Add the `CapturePickerSectionKind` cases for the `now` and per-note sections.
   - Give `CapturePickerRow` two optional additions:
     - `pendingBlockID: CapturePickerPendingBlockID?`, holding `route`, `taskRef`, and
       `suggestions`. It is set on ID-less rows, which stay `isSelectable: true` with
       `insertion: nil`.
     - `scheduledText: String?`, for example `Oct 3`, with the year appended when it is
       not the current year.

     Existing sources pass nil, so their behavior is byte-identical.

3. **Fuzzy query** (`FuzzyMatcher.swift`):
   - Let `FuzzyQuery` strip a caller-chosen leading sigil. The default stays `^`, so the
     `^` and Block ID behavior is unchanged.
   - The task-link index strips `:`.
4. **Index.** Add `Sources/CaptureCore/TaskLinkPickerPresentation.swift` with
   `TaskLinkPickerIndex(candidates:)`, modeled on `ActiveTaskPickerIndex`:
   - Rows are deduplicated and keyed by `"\(route)|\(ref)"`, never by `replacement`.
   - **Grouped mode (empty filter)** follows Bob's order and `group`:
     - `queued` → one section per Pomodoro line. Reuse the `^` header semantics:
       ordinal, name or "Unnamed Pomodoro", `formattedTimeRange`, and the NOW pill for
       the current entry.
     - `in_progress` → "In Progress".
     - `next` → "Next".
     - `now` → "This Week's Bets".
     - `note` → one section per route, titled `route.md`, with subtitle "Inbox", "Area",
       or "Project" from `noteKind`.
     - A missing or unknown `group` falls back to `note`.
   - **Filtered mode**: one `.matches` section. Every token must match.
     - Weights: text 100, `route:blockID` 90, blockID 90, route 80, Pomodoro name 70,
       section 40.
     - Ties go to Bob order.
     - Highlight ranges cover text, route, and blockID.
   - **Rows**:
     - `glyph: .task(status)`, with the display text from `TaskDisplayText`.
     - `badgeText: "NOW"` when `now`.
     - `scheduledText`.
     - A Pomodoro `chipText` in filtered mode only.
     - `depth` in grouped `note` sections only.
     - Identified rows: `insertion` = Bob's `replacement`.
     - ID-less rows: `pendingBlockID`, and the locator shows `route` plus the first
       suggestion.
   - **Detail**:
     - `insertionPrefix "@"`.
     - Identified action line: `↩ inserts @sase:deep-fix · ⇧↩ adds = to start it`.
     - ID-less action line:
       `↩ adds ^fix-flaky-gkeep to sase.md, then inserts @sase:fix-flaky-gkeep`. When
       there is no suggestion: `↩ names this task, then inserts its link`.
     - When `pullsForward`: `Scheduled Oct 3 — linking pulls it forward`.
   - **Budget**: `min(max(rows + headers, 4), 11)`, or 4 when empty.
   - **Empty states**: "No open tasks in your area or project notes", and the existing
     no-matches state.
5. **Tests** (`Tests/CaptureCoreTests/TaskLinkPickerPresentationTests.swift`), using a
   JSON sample of the worked example:
   - decoding, including the defaults when keys are absent;
   - grouped section order and titles;
   - four ID-less rows staying distinct;
   - depth only in note sections;
   - filtered ranking (`sase deep`, `flaky`, `dntst`) and its highlights;
   - a seed query with a leading `:`;
   - an unchanged `^` `FuzzyQuery`;
   - `scheduledText` formatting;
   - the detail action lines;
   - every `taskLink` source string and the key hints;
   - need precedence.
6. **Verify on macOS.** Swift is not available on Linux.
   - First try `ssh -o ConnectTimeout=8 mac true`. If it works, rsync the checkout
     (excluding `.build`) to `mac:/tmp/bob-mac-capture-task-link/` and run
     `just format-lint build test` there.
   - Otherwise iterate on GitHub Actions. This plan explicitly instructs you to use
     `/sase_git_commit` for each CI iteration in bob-mac-capture, with conventional
     `feat(capture): …` or `fix(capture): …` subjects.
   - Find the run with `gh run list -L 3`. Wait in the foreground with
     `gh run watch <id> --exit-status` and a tool timeout of about 30 minutes, or hand
     the wait to `/sase_monitor`.
   - Read failures with
     `gh run view <id> --log-failed | grep -E " error: |error: -\[|failed \("`.
   - Done means one `macOS 26 SwiftPM` run with every step green. Record the run ID.
     Never weaken or skip an assertion to go green.

## Phase `complete`: `task_link` candidates and docs (bob-cli)

1. **Candidates** (`capture_complete.rs`):
   - Add `TaskLinkCandidate` with exactly the Design's JSON keys and serde attributes,
     plus `Candidates::TaskLink`.
   - Add `task_link_candidates(bob_dir, query)`, which calls
     `capture_link_tasks::discover` and `rank`. Replace the `grammar` placeholder with
     it, and pass the warnings through.
   - Add the human rows in `candidate_lines`.
   - Help:
     - add `task_link` to the `Contexts:` list;
     - add the example `bob capture-complete -c 1 -- ':'` next to the `^` example;
     - add one `long_about` sentence on the context and the sigil-inclusive replacement.
2. **Unit tests**:
   - the pinned JSON for one identified row and one ID-less row, with the omitted `now`
     and `pulls_forward`;
   - the worked-example order;
   - `:dee` → `{0,4}`;
   - the cursor at the sigil;
   - a batch second item;
   - the human rows under `NO_COLOR`;
   - a missing daily note, where the warning is present and the candidates are intact.
3. **CLI integration.** Add `tests/cli/capture/complete_task_link.rs`, registered in
   `mod.rs`.
   - JSON and human output on the worked-example vault.
   - **Round trip, identified.** Take row 1's `replacement` and range, splice it into
     the draft, and run `bob capture -f json`. The result is `pomodoro_link` success.
   - **Round trip, ID-less.**
     1. Run `bob capture-task-id` with row 8's `route`, `ref`, and first suggestion.
     2. Splice `@sase:<id>=` and capture.
     3. The task is Next and BUGS started.
   - Also: `@health:<id>` retires the future `[scheduled::2026-10-03]`, which agrees
     with `pulls_forward: true`.
   - A batch of two `:` items: accept both, then capture. Both are linked.
4. **Latency check (read-only).**
   - Run `bob capture-complete -c 1 -f json -- ':'` with `hyperfine`, or 10 timed runs,
     against the real `~/bob`.
   - Report the median in the final response. The target is under 150 ms.
   - If it is slower, profile and fix, for example by avoiding reading every note twice.
     Do not cap the candidate count.
5. **Docs.**
   - `docs/capture.md`:
     - Contents entry.
     - A Grammar-at-a-glance row for `:<query>`.
     - Example rows: `:`, `:dee`, `:sase:deep-fix` → T2, and `: dee` → prose.
     - A new subsection, `### Picking any open task with ':'`, after "Linking and
       starting existing tasks". It covers: the claim rule and table, T1 and T2, which
       notes and statuses are included and why, groups and order, ranking, ID-less tasks
       and `capture-task-id`, and bulk drafts.
     - The capture-parse section: the `task_link` need and state.
     - The capture-complete section: the `task_link` context paragraph, the candidate
       schema, and the contexts list.
   - `README.md` Capture section: add wherever it enumerates markers, needs, or
     contexts.
6. `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test`
   must be clean.

## Phase `mac_panel`: wiring, prompt, views, and README (bob-mac-capture)

Open the repo exactly as in phase `mac_core`. Build `bob` from the bob-cli checkout
(`cargo build`) and generate real JSON on the worked-example vault for the fixtures:

- `capture-parse` for `:`, `:dee`, `Buy milk\n\n:`, and `@sase:deep-fix=`;
- `capture-complete -c 1 -- ':'`, `-c 4 -- ':dee'`, and the batch second item;
- `capture-task-id` success and the duplicate-ID error;
- `capture --dry-run -f json` for `@sase:deep-fix` and `@sase:deep-fix=`.

Add the matching `Tests/Fixtures/fake-bob` cases.

1. **Index routing** (`CapturePickerState.swift`): add
   `CapturePickerIndex.taskLink(TaskLinkPickerIndex)`, and handle every switch on
   `CapturePickerSource` and `CapturePickerIndex`.
2. **Completion gating and calm state** (`CapturePanelModel.swift`):
   - Add `task_link` to the `shouldRequestCompletion` needs.
   - `pickerNeed(in:)` returns `.taskLink`, which goes through
     `applyQuietIncompletePicker`. That skips the doomed dry run and shows the status
     text.
3. **Opening.**
   - In `handleCompletionResponse`, route `context == "task_link"` to a new
     `handleTaskLinkCompletion`. Mirror `handleActiveTaskCompletion`:
     - dismiss the inline list;
     - auto-open only on `.edit` and when not suppressed;
     - show the chip on `.selection`;
     - when the caret is past `r.start`, refetch the full list at `r.start`, falling
       back to `snapshotIsPartial`.
   - Per-source helpers:
     - `pickerQuery` = draft bytes `(r.start + 1)..<caret`;
     - `pickerMarkerHighlightRange` = `[r.start, r.end)`;
     - `removePickerTrigger` deletes `[r.start, r.end)` when the byte at `r.start` is
       `:`.
4. **Accept.**
   - Add `acceptPickerRowAndStart` alongside `acceptPickerRow` and
     `acceptPickerRowAndSubmit`.
   - Identified row: splice `insertion` (plus `=` for start) into
     `picker.replacementRange`, keeping the stale-draft guard. Place the caret after the
     insertion, set `suppressedCompletionAcceptanceDraft`, analyze without completion,
     and submit for ⌘↩.
   - Announcements: "Inserted @sase:deep-fix", or "Inserted @sase:deep-fix= — starts its
     session when captured".
   - ⇧↩ on other sources stays consumed.
5. **Link-mode Add block ID prompt**, for an ID-less row:
   - `CaptureTaskIDPromptState` gains a purpose:
     - `.parentTask`, which is today's `@route+` flow, byte-for-byte unchanged;
     - `.taskLink(route, taskRef, suggestions, followUp: .none|.start|.submit, returnPicker: CapturePickerState)`.
   - **Prompt contents:**
     - the field is prefilled with `suggestions.first`, fully selected;
     - suggestion chips are clickable, and Tab / ⇧Tab cycle them (a new router command,
       used only in this purpose);
     - a live `Inserts @route:<typed>[=]` line in route and blockID colors;
     - the button reads "Add ID & Link", "Add ID & Start", or "Add ID & Capture".
   - **Return:**
     1. Validate locally with `isValidBlockID`.
     2. Call `assignCaptureTaskID(route:taskRef:blockID:)`.
     3. On success, splice `"@\(success.route):\(success.blockID)" + ("=" if .start)`
        over the saved `:` range, and place the caret after it.
     4. Announce "Added ^id to route.md and inserted @route:id".
     5. Submit if `.submit`.
   - On failure, keep the prompt open with Bob's error.
   - **Escape** closes the prompt and restores `returnPicker` (filter, selection, and
     budget) without a refetch.
   - Keep `completeTaskIDAssignment`'s bare-ID splice for `.parentTask` only.
6. **Keys** (`CaptureKeyCommandRouter.swift`):
   - Add `pickerSourceIsTaskLink` (or the source) to `CaptureKeyRoutingContext`.
   - Picker mode: Shift-Return and Shift-keypad-Enter map to `.acceptPickerRowAndStart`
     only for `taskLink`. Otherwise `.consumeKey`, as today.
   - Prompt: Tab / Shift-Tab cycle suggestions only in link purpose.
7. **Views**:
   - `CapturePickerView.swift`:
     - section headers for `now` and per-note sections, with a note-kind SF Symbol from
       the targets-cache mapping and a count;
     - the ID-less trailing locator: `route:` in accent, a `plus.circle` glyph, and the
       suggestion dim and italic, middle-truncated at 260 pt;
     - the `scheduledText` capsule (`calendar` glyph, secondary);
     - the detail-strip action and pull-forward lines.
   - `CapturePanelView.swift`: the `TaskIDPromptCard` link-purpose elements.
   - Follow the existing palette, fills, and Increase Contrast variants. The layout
     constants pinned by `CapturePickerDesignTests` must not change.
8. **README**:
   - A new "Task Link Picker" subsection after "Active Task Picker": the trigger,
     included notes and statuses, order, fuzzy filtering, keys, the ID-less flow, and
     bulk drafts.
   - Keyboard table and prose for ⇧↩.
   - The link-mode Add block ID behavior.
   - Bump the required Bob features in Requirements.
9. **Tests** (`CapturePanelModelTests`, `BobMacCaptureTests`, and the design tests):
   - **Opening:**
     - auto-open on typing `:`, and on the batch's second item;
     - a chip only on a selection move;
     - a refetch at `r.start`;
     - local filtering with no subprocess per keystroke.
   - **Keys and accept:**
     - ↩ gives `@sase:deep-fix` with the caret at 14;
     - ⇧↩ gives `@sase:deep-fix=`;
     - ⌘↩ submits;
     - ⌫ on an empty filter gives an empty item;
     - Esc: first clear, then the chip.
   - **ID-less flow:**
     - prompt prefilled and selected;
     - Tab cycling;
     - success splices `@sase:fix-flaky-gkeep`, with the `=` and submit variants;
     - failure keeps the prompt;
     - Esc returns to the picker with the filter intact;
     - the `@route+` prompt flow is unchanged.
   - **Other:**
     - router tables for Shift-Return per source;
     - marker-highlight range;
     - calm status text;
     - key hints in the design tests;
     - an optional render-to-PNG state for the task-link card.
10. **Verify on macOS** exactly as in phase `mac_core`, step 6. Record the green run ID.

## Out of scope and follow-ups

- Chaining `:` after a session token on one line (`=x :dee`). A blank line works today.
- Most-recently-picked ranking, and remembering the last filter.
- Offering "link with the existing ID" when `capture-task-id` reports that another
  editor already named the task.
- An Obsidian plugin equivalent, and folding the `^` picker into `:`.
