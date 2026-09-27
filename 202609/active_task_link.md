---
tier: epic
title: Link and start existing tasks with solo @route:id and active-task ^route:id
goal: "A capture item that is only `@route:block-id[#pomodoro][=<X>]` or
  `^route:block-id[#pomodoro][=<X>]` links an existing task into today's Pomodoro ledger
  (and optionally starts that session) atomically, and typing `^` in Bob CLI completion
  and Bob Mac Capture offers only In Progress and Next tasks, inserting the full
  `route:block-id` in one accept.

  "
phases:
  - id: link-core
    title: Solo Pomodoro-link grammar and atomic execution
    depends_on: []
    size: medium
    description: "link-core: add the shared `@`/`^` solo grammar, the keep/move/insert
      Task Link planner with queue-respecting start, the `pomodoro_link` JSON kind and
      human output, and CLI integration tests.

      "
  - id: active-task-discovery
    title: Active-task discovery module
    depends_on:
      - link-core
    size: small
    description: "active-task-discovery: build a read-only scanner that lists In
      Progress and Next tasks with block IDs from routable vault-root notes, annotated
      and ordered by today's open-Pomodoro Task Links, with query ranking and bounded
      warnings.

      "
  - id: editor-contract
    title: Parse, completion, rewrite, help, and docs for the new forms
    depends_on:
      - active-task-discovery
    size: medium
    description: "editor-contract: expose the `pomodoro_link` mode, `^` spans, needs and
      diagnostics in capture-parse, the `active_task` full-completion context in
      capture-complete, a rewrite guard, help text, and docs.

      "
  - id: mac-capture
    title: Bob Mac Capture active-task picker and link/start preview
    depends_on:
      - editor-contract
    size: medium
    description:
      "mac-capture: decode the new context, candidates and kind in bob-mac-capture,
      render an active-task picker, preview and notify link/move/start outcomes, and
      cover everything with fake-bob tests and macOS CI."
proposed_by: bbugyi200.athena.0t3
create_time: 2026-09-27 10:38:08
status: wip
---

- **PROMPT:**
  [prompts/202609/active_task_link.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/active_task_link.md)

# Plan: Link and start existing tasks with solo `@route:id` and active-task `^route:id`

## Why this exists

The user wants a solo (whole capture item) `^file:id[#pomodoro]=<X>` that "works just
like the solo `@file:id[#pomodoro]=<X>`" except that typing `^` completes only In
Progress `[/]` and Next `[*]` tasks — the tasks attached to active Task Links — and
completes the whole `file:id`, not just the file.

The solo `@file:id` form does not exist today. `bob capture '@sase:deep-fix'`,
`'@sase:deep-fix='`, and `'@sase:deep-fix#bugs='` all fail with "task text is required",
while `bob capture-parse` reports that same input as a complete `pomodoro_task`. That
disagreement is a latent bug. `=<X>` works only on body-bearing new-task captures
(`Work @sase:new=3`). Ensure Next (`@route+id[#pomodoro]`) does not accept `=<X>`. It
also never creates a missing Task Link and demotes In Progress to Next, so it cannot
serve In Progress tasks whose links sit under completed Pomodoros. This epic defines the
solo `:` form properly and adds `^` as its active-task spelling.

Real usage shaped the design. Today's ledger holds many named placeholders (GTD, SASE,
CAPTURE, DECKS, …), each with several queued Task Links. In Progress tasks are queued
there too, or appear only in earlier sessions. The main job is: "start working on this
task I already planned." Picking the task must never pull it out of the Pomodoro it is
queued in.

## The contract (shared by every phase)

### Grammar

One operation, two spellings. Both are valid only when the marker is the entire capture
item, meaning one item in a blank-line-separated batch: no body text, no authored child
bullets, no `%`/`--clip`, no `s:<N>`/`p:<N>`, and no forced destination flags.

| Item                               | Meaning                                                            |
| ---------------------------------- | ------------------------------------------------------------------ |
| `@route:block-id`                  | Link the existing task `^block-id` in `route.md` to today's ledger |
| `@route:block-id#pomodoro`         | Same, putting its Task Link under the named Pomodoro               |
| `@route:block-id[#pomodoro]=<X>`   | Same, and start that Pomodoro atomically (`<X>` mirrors `se<X>`)   |
| `^route:block-id[#pomodoro][=<X>]` | Identical execution; `^` is the active-task spelling               |

- The components after the sigil use exactly the existing `@route:block-id#name=<X>`
  grammar: route characters, lower-cased route, the `:`-family block-ID validator,
  Pomodoro slug selector, and `se<X>` suffix. Refactor so both sigils share one parser.
  Do not add a second one.
- `^` is recognized only as the first token of an item's first line, never trailing and
  never on an authored child line. It mirrors Obsidian's `^block-id` notation: "the
  block `id` in `route`".
- A token with the complete `^route:block-id…` shape claims its item. If anything else
  is on the item, fail with an `invalid_pomodoro_link` error: "`^route:block-id` must be
  the whole capture item; to create a new Pomodoro-linked task use
  `<text> @route:block-id`". Do not fall through to literal task text, following the
  `+N` near-miss precedent.
- Partial shapes are incomplete states when they are the whole item: `^`, `^fragment`
  (no `:`), `^route:`, and `^route:block-id#`. Strict capture fails with a helpful
  "finish the marker" error. Followed by other text, a partial shape stays ordinary
  prose. Anything else starting with `^` (`^_^`, `^^`, `^.`) stays ordinary text,
  exactly as today.
- `^…+` (project note) and `^…!` (explicit toggle) are rejected with specific messages
  naming `@route:block-id+` and `@route+block-id!`. A `@@` declaration never applies to
  a `^` item and never turns it into a task, matching `+N`.
- On a solo `@route:block-id…` item, conflicts with authored children, clipboard,
  `s:<N>`, `p:<N>`, or forced flags give specific "Pomodoro link capture cannot be
  combined with …" errors instead of "task text is required".
- Body-bearing `<text> @route:block-id[#name][=<X>]` keeps its exact current new-task
  meaning.

### What the operation does

It resolves the existing task with the task-toggle lookup, so errors are reused: missing
note, missing ID with close-match suggestion, duplicate ID, and non-task block ID. The
missing-ID error adds: "to create a new Pomodoro-linked task, add text:
`@route:block-id <text>`". The Task Link is `[[route#^block-id]]` as a dedicated,
sole-content child bullet. This is the form capture writes and Ensure Next moves.

**Task status** (route note)

| Current                    | Result                                                                                        |
| -------------------------- | --------------------------------------------------------------------------------------------- |
| Ready `[ ]`, Blocked `[?]` | Next `[*]`; `dependsOn` still warns as in Ensure Next                                         |
| Next `[*]`                 | unchanged                                                                                     |
| In Progress `[/]`          | unchanged; never demoted, matching task-status-hooks' "In Progress keeps the stronger status" |
| Done, canceled, unknown    | error: only Ready, Blocked, Next, and In Progress tasks can be linked to a Pomodoro           |

For every eligible status, a single valid future `[scheduled::]` field is retired, and
the pull-forward Schedule Log entry is written when a log already exists. Both use the
existing Ensure Next rules.

**Task Link** (daily note). Let _Q_ be the open Pomodoro whose children hold the task's
dedicated Task Link. More than one movable link is the existing write-free invariant
error. Links under completed Pomodoros are history and are never edited.

- `#pomodoro` given: use the existing named selection. That means whole-slug then prefix
  over open entries, else create the canonical named future Pomodoro with the existing
  placement. Move Q's complete subtree there, or insert a new link when there is no Q.
  If Q is already the destination, nothing changes.
- No name, Q exists: **respect the queue.** The destination is Q, and nothing moves.
- No name, no Q: use the implicit current/next Pomodoro (the single open timed entry,
  else the first open entry), exactly like `<text> @route:id`. Insert a new link.

**Start** (`=<X>`). This uses the same `se<X>` math, clock (`BOB_NOW`), and guards as
`<text> @route:id=<X>`.

- Any open timed Pomodoro fails the capture with "finish the current Pomodoro first".
  When the task's own link is in that running Pomodoro, the message names it and
  suggests `+N`/`-N`.
- `#name=<X>`: an existing match must be an untimed open placeholder, else error. A
  missing name creates a started entry with the existing placement.
- No name, Q exists: Q must be an untimed open placeholder, else error suggesting
  `#<pomodoro>=<X>`. Start Q **in place**.
- No name, no Q: the first open untimed placeholder, else create an unnamed started
  entry. This is the established new-task rule.
- The started entry keeps its position, name, checkbox, children, and line endings. Then
  the link is moved or inserted under it. The ledger is never reordered.

The route note, the daily note, and the other batch items commit atomically through
`CaptureBatchPlanner`. Byte-identical files are not staged. Dry run plans the same
result. A later batch item sees earlier staged edits, so `^a:b=` followed by `+2`
adjusts the session just started.

### Worked example (use as a test fixture)

`BOB_NOW=2026-07-10 09:02:00`. The day note:

```markdown
## Pomodoros

- [x] (**0800-0825** [t:: 25m]) — PLAN
  - [[sase#^outline]]
- [ ] () — BUGS
  - [[sase#^deep-fix]]
    - repro notes
- [ ] ()
```

`sase.md` has `[*] … ^deep-fix`, `[/] … ^outline`, and `[ ] … ^ready`.

| Item                     | Result                                                                                                              |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `^sase:deep-fix`         | True no-op: already Next, Task Link already in BUGS                                                                 |
| `^sase:deep-fix=`        | BUGS becomes `(**0905-0930** [t:: 25m]) — BUGS`; the link and `repro notes` stay                                    |
| `^sase:deep-fix#focus=3` | Started `FOCUS` (0905-0920) created after PLAN; the link subtree, child included, moves out of BUGS into it         |
| `^sase:outline=`         | Stays `[/]`; not queued, so BUGS (the first open placeholder) starts with a new `[[sase#^outline]]`; PLAN untouched |
| `@sase:ready`            | `[ ]` → `[*]`; new link under BUGS (implicit current/next)                                                          |
| `@sase:ready#bugs=-`     | `[ ]` → `[*]`; BUGS starts 0900-0925 (5-minute offset); link appended                                               |

### Output contract

`bob capture --format json` reports a new `kind: "pomodoro_link"` with schema version

1.

- It carries every key clients require: `ok`, `dry_run`, `routed`, `route`,
  `route_label`, `relative_target`, `target`, `text: ""`, the post-image `task_line`,
  `created`, `scheduled: null`, and `placement: "linked"`.
- It reuses the task-toggle vocabulary so clients can share rendering: `block_id`,
  `day_file`, `block_link`, `previous_task_line`, `previous_status_symbol`/`_name`,
  `status_symbol`/`_name`, `status_changed`, `removed_scheduled`, `schedule_log` (when
  written), `pomodoro_name` (resolved destination), and `creates_pomodoro`.
- It adds `pomodoro_link_action: "linked" | "moved" | "already_current"`.
- `pomodoro_link_source` is present only when a link existed, and is reported in the
  pre-image. `pomodoro_link_destination` is reported in the post-image.
  `pomodoro_link_placement` is present when a link was inserted or moved.
- `pomodoro_start` is the exact object `pomodoro_task` uses, and only appears with
  `=<X>`.
- It emits no `toggle_direction`/`toggle_behavior`, so no client mistakes it for a
  toggle. Existing kinds and keys are unchanged.

Human output follows the Ensure Next style:

- a `link`/`would link` or `start`/`would start` header naming the route note;
- a status line: `[ ] → [*]`, `[*] already Next`, or `[/] stays In Progress`, plus the
  text and `^id`;
- a ledger line: `Linked … under BUGS`, `Moved Task Link BUGS → FOCUS (created FOCUS)`,
  or `Task Link already in BUGS; no ledger change.`;
- the existing start phrasing, e.g. `started BUGS 0905-0930 (25m) at line 4`.

### Editor contract

- `bob capture-parse`:
  - Mode `pomodoro_link` for complete solo items of either spelling.
  - The `@` spelling keeps its `pomodoro_route`/`pomodoro_block_id`/`pomodoro_name`/
    `pomodoro_start` spans.
  - `^` uses new `active_task_route` (covering `^route`) and `active_task_block_id`
    spans, plus the existing name and start spans.
  - Lone `^`, `^fragment`, and `^route:` report `incomplete`, `needs: ["active_task"]`,
    and one `interactive_placeholder` span over the token. `^route:id#` needs
    `pomodoro_name`.
  - Near misses and conflicts report an `invalid_pomodoro_link` diagnostic with a
    precise range.
  - The additive `pomodoro_start` spec stays as today.
- `bob capture-complete`:
  - New context `active_task`. It is active while the cursor is in the `route:block-id`
    part of a solo leading `^` token, including an empty part.
  - `replacement` runs from just after `^` to the end of that part, and always stops
    before `#`/`=`, so typed suffixes survive an accept. `query` is the text from after
    `^` to the cursor.
  - Candidate example:

    ```json
    {
      "replacement": "sase:deep-fix",
      "ref": "6:279bb4ad",
      "route": "sase",
      "block_id": "deep-fix",
      "status_symbol": "*",
      "status_name": "Next",
      "status_type": "ON_HOLD",
      "text": "Fix deep bug",
      "section": "Tasks",
      "pomodoro": { "line": 4, "name": "BUGS", "time_range": null, "is_current": false }
    }
    ```

    `pomodoro` is `null` when the task is not queued.

  - `#name` after `^route:id` completes Pomodoro names exactly as it does after
    `@route:id`. The `=<X>` part is never a completion field. Existing `@route:`
    (`pomodoro_block_id`) completion is unchanged.

### Deliberate choices

1. `^` and the solo `@` form write identically. `^` is an editor affordance, so
   execution does not reject a non-active task (the preview shows its transition).
2. **Respect the queue**: with no `#name`, an already-queued task stays in its Pomodoro,
   and `=` starts that Pomodoro. Pulling a queued task into the first placeholder would
   break named planning.
3. `:` means "link", so the solo form creates a missing link. `@route+id` Ensure Next
   stays unchanged, including its no-create and In-Progress-to-Next behavior.
4. Starting happens in place and never reorders the ledger. This matches `#name=` and
   the Obsidian `se<X>` snippet.
5. An unqueued `=` keeps the established new-task rule (first open untimed placeholder)
   for consistency. The Mac preview shows exactly which Pomodoro would start.
6. It is a new `pomodoro_link` kind, not an overloaded `task_toggle`/`pomodoro_task`, so
   older clients degrade to a neutral preview instead of a wrong one.

Non-goals:

- changing `<text> @route:id`, Ensure Next, `!`, project notes, task-status-hooks,
  Obsidian plugins, or `@route:` completion for body-bearing captures;
- adding new CLI subcommands or options.

## Phase: link-core

Implement the grammar and the transaction in bob-cli: `src/native/capture_language.rs`,
`src/native/capture.rs`, `src/native/capture_task_toggle.rs`, and
`src/native/capture_pomodoros.rs` if it needs helpers.

1. Add `CaptureKind::PomodoroLink { block_id, pomodoro_name, start, spelling }`, where
   `spelling` is `At` or `Caret`.
   - Let a leading or sole `@route:id…` token route with an empty body, the way the `+`
     toggle candidates do. Resolve the finished item into `PomodoroLink` or the specific
     conflict errors.
   - Add an item-level `^` pre-parse alongside `parse_pomodoro_adjust_item` that
     implements the recognition, near-miss, and partial-shape rules.
   - Factor the post-sigil component parser so `@` and `^` share it.
2. Generalize `plan_task_next`, or add a sibling that shares its code, with a policy
   that keeps In Progress while still retiring future schedules and writing pull-forward
   logs.
3. Add a pure ledger planner that finds dedicated open links (reuse
   `find_movable_task_links`), resolves the destination with the queue preference and
   optional named creation, and then moves the subtree (`move_subtree_to_entry`) or
   inserts a link.
   - Factor destination selection, placeholder timing, and started-entry creation out of
     `plan_pomodoro_start`/`create_started_pomodoro_entry`, so the new-task `=<X>` path
     and this path share one implementation and one placement rule. The existing
     `capture_pomodoro_start_*` tests must pass unchanged.
   - Expose a small read-only helper that lists each open entry's dedicated Task Links;
     the discovery phase reuses it.
4. Add `plan_pomodoro_link_capture` on `CaptureBatchPlanner`:
   - existing day-file checks (it must exist and must not be the routed note);
   - staging only changed files;
   - warnings;
   - the JSON kind and fields;
   - human output;
   - `capture_kind_label`.
5. Add tests in `tests/cli.rs`:
   - every worked-example row;
   - named existing, prefix, and created destinations, with and without `=`;
   - a missing link inserted versus moved versus already-current;
   - descendant preservation;
   - In Progress kept and Ready/Blocked promoted;
   - future-schedule retirement with a Schedule Log;
   - `dependsOn` warning;
   - Done/canceled, missing, duplicate, and non-task IDs, plus the missing-ID hint;
   - multiple movable links;
   - the running-Pomodoro error when the task is queued in it and when it is not;
   - a non-placeholder Q;
   - named creation blocked by multiple timed entries;
   - no Pomodoros section and a missing day file;
   - CRLF and no final newline;
   - dry run;
   - a mixed batch with `^…=` then `+2`, and late-failure rollback of both notes;
   - `@@` never applying to `^`;
   - every conflict and near-miss message;
   - `^_^` and `^ text` staying literal;
   - body-bearing `@route:id=` regression.

Run `cargo clippy --all-targets --all-features` and `cargo test`. Format only the code
you touched. Repo-wide rustfmt drift is the known task bob-cli-24, so do not reformat
unrelated files.

## Phase: active-task-discovery

Add a read-only module, `src/native/capture_active_tasks.rs`, that the editor-contract
phase wires into completion.

- Enumerate vault-root `*.md` notes whose stem is a valid capture route. Reuse
  `capture_targets`' eligible-filename predicate rather than re-deriving it. Do not
  limit this to area/project notes.
- Read Tasks settings once. Scan with `note_tasks`. Keep open tasks that have a block ID
  and a status symbol of `/` or `*`, filtering by symbol as task-status-hooks does.
- Annotate each task using today's daily note (`pomodoro::day_file_for`) and the
  link-core helper. The annotation is the first open entry holding the task's dedicated
  `[[route#^id]]` link, with `line`, `name`, `time_range`, and `is_current`. When the
  ledger has more than one open timed entry, `is_current` is false everywhere.
- Order with an empty query:
  1. queued tasks in ledger order (entry order, then child order);
  2. unqueued In Progress tasks;
  3. unqueued Next tasks.

  Within groups 2 and 3, order by route, then line.

- Provide a `rank` function. Prefix matches rank before substring matches. The fields
  are `route:block-id`, `block-id`, task text, section, and Pomodoro name. Order is
  stable inside each tier, so `^dee`, `^sase:dee`, and `^outline` all find their tasks.
- Warnings are bounded and never contain task text: a missing day file, a missing
  Pomodoros section, and unreadable notes. A failed or partial read still returns the
  other candidates.
- Add unit tests with temp vaults covering:
  - ordering;
  - ranking;
  - nested tasks;
  - tasks without an ID excluded;
  - done, Blocked, and Ready excluded;
  - non-route and uppercase filenames skipped;
  - the `mac_inbox.md` inclusion;
  - duplicate links annotated with the first owner;
  - the warning paths.

## Phase: editor-contract

Wire the contract into bob-cli's editor commands.

- **`capture-parse`** (`capture_language.rs`, `capture_parse.rs`):
  - Add `EditorMode::PomodoroLink`, `Need::ActiveTask`, the two span kinds, and the
    `invalid_pomodoro_link` diagnostic.
  - Pin these inputs: lone `^`, `^frag`, `^route:`, `^route:id#`, `^route:id=`, complete
    forms, `=` malformations, near misses, literal lookalikes, non-first-line `^`, and
    multi-item drafts.
  - Also pin the solo `@route:id…` mode and conflict diagnostics, which fixes today's
    parse/capture disagreement.
- **`capture-complete`** (`capture_complete.rs`, `capture_language.rs`):
  - Add `CompletionContext::ActiveTask`, the field detection for a solo leading `^`, and
    `ActiveTaskCandidate` JSON backed by the discovery module.
  - Add human rows such as `sase:deep-fix  [*] Fix deep bug  · BUGS`.
  - Keep replacement ranges ending before `#`/`=`, give `#name` completion after `^`,
    and return an empty success inside `=<X>`.
  - Unit and protocol tests must pin the JSON shape and prove that `@route:` completion
    is unchanged.
- **`capture-rewrite`**: `pomodoro_link` is non-absorbable, with the notice "@@ cannot
  take a Pomodoro link: leave … on this item, or delete it". `^` items are never
  rewritten.
- **Help**:
  - `bob capture --help`: concise grammar lines and examples such as
    `bob capture '^sase:deep-fix='`.
  - `bob capture-complete --help`: add `active_task` to the Contexts list and an example
    `-c 1 -- '^'`.
  - `bob capture-parse --help`: list the mode if the help lists modes.
- **Docs**: in `docs/capture.md`, update:
  - the grammar table and `#` table rows;
  - a new section after "Starting the session atomically", with the status, queue, and
    start rules, the worked example, errors, and JSON;
  - the kind list (`pomodoro_link`);
  - the Interactive editor markers section (the `^` picker);
  - the `capture-parse` and `capture-complete` sections.

  Also update the capture summary in `README.md`.

Run `cargo clippy --all-targets --all-features` and `cargo test`.

## Phase: mac-capture

Open the linked repo with `/sase_repo` (`sase repo open bob-mac-capture`) and read its
`AGENTS.md` if one is present. Keep Swift free of grammar, clock, and ledger logic;
everything comes from Bob's JSON.

- **Decode** (`Sources/CaptureCore/CaptureModels.swift`, `CompletionRowContent.swift`):
  - Add `CaptureCompletionContext.activeTask` (`"active_task"`), a tolerant `pomodoro`
    candidate object, and the `pomodoro_link` success fields (`pomodoro_link_action`
    `"linked"` included).
  - Older Bob binaries that omit these must still decode.
- **Gate** (`CapturePanelModel.swift`):
  - Add need `active_task` and span kinds `active_task_route`/`active_task_block_id` to
    `shouldRequestCompletion`.
  - Keep them **out of** `routeSpanKinds`, so cached route completion never intercepts
    `^` and `routeReplacementRange` never overwrites it.
  - Add no Swift-side `^` sniffing; Bob's `needs` covers lone `^`.
  - After accepting a candidate, do not immediately re-open the popup for an exact
    `route:block-id`.
  - Map the new spans to the route and block-ID palette categories.
- **Picker row**:
  - A calm, legible `.activeTask` branch.
  - Distinct In Progress versus Next glyph and tint from the existing palette.
  - Primary text is the task text with the query emphasized.
  - The secondary line is `route:block-id`, plus the section when present.
  - The Pomodoro badge reads `Now · NAME HHMM-HHMM` for the current entry, the name for
    a queued entry, `Planned` for an unnamed placeholder, or `Not queued`.
  - The context label is "Active Task".
  - Accept uses the generic replacement path. A typed `#name`/`=<X>` suffix survives.
- **Preview and notifications**:
  - Add a pure `CapturePomodoroLinkPresentation` in CaptureCore. It covers the status
    transition, the ledger outcome (linked/moved/already-there, created destination),
    and the destination label.
  - Reuse `CapturePomodoroStartPresentation` for the session row, and reuse endpoint
    labels from the toggle presentation where practical.
  - Set `primaryActionTitle` to "Start" when `pomodoro_start` is present, else "Link".
  - Include an accessibility summary.
  - Wire it into the panel preview dispatch, the footer, and `NotificationService`, for
    single and batch captures, target paths, and `friendlyKindLabel`.
  - Add notification titles: "Started NAME", "Linked to NAME", "Moved to NAME", and
    "Already in NAME".
- **Tests**:
  - Extend `Tests/Fixtures/fake-bob` with parse, complete, and dry-run/commit responses
    for:
    - `^`, `^dee`, and a full accept;
    - `^sase:deep-fix#b` name completion;
    - linked, moved and created, already_current, and start variants;
    - a near-miss error;
    - the solo `@sase:deep-fix=`.
  - Add model, row-content, decoding, presentation, and notification tests, including
    older-Bob compatibility and suffix preservation.
- **README**: update the runtime contract, completion, the draft grammar, the
  row-presentation notes, and notifications.

Run the CaptureCore tests locally. The app target needs macOS, so push and confirm the
`macOS 26 SwiftPM` GitHub Actions run passes: format lint, build, and the full
`swift test`. Report the run ID, and fix any failure before finishing.

## Acceptance

- Every worked-example row behaves as specified, from both the CLI and a Bob Mac Capture
  preview and submit on the same fixture. The dry-run preview and the committed JSON
  agree.
- `^` in the Mac panel lists only In Progress and Next tasks, in ledger order. Accepting
  a row inserts `route:block-id` in one step. Adding `=` shows the exact session that
  would start.
- `capture-parse` and `capture` agree on every solo, near-miss, and literal input.
- Failed items write nothing, and batches stay atomic.
- `cargo clippy`, `cargo test`, and macOS CI pass on the landed revisions.
