---
tier: epic
title: 'Bob Mac Capture Control-Return: capture, then land on it in Obsidian'
goal: 'In Bob Mac Capture, Control-Return captures the draft, then brings Obsidian
  forward with every note the capture changed open in its own tab. The caret sits
  on the exact line that changed, which gets a short highlight. bob decides what to
  open, so the app stays a thin client. One Obsidian URL carries the whole list, so
  tabs never replace each other. The panel shows beforehand what Control-Return will
  open.

  '
decisions:
  tab_policy:
    ask: Where should Control-Return open notes in Obsidian?
    choices:
      new_tab: Reuse a tab already showing the note, else open a new tab; your current
        note stays open
      replace_active: The first note replaces the active tab, like Go To; only extra
        notes get new tabs
    default: new_tab
    why: A capture should never close the note you were reading, and several notes
      need tabs anyway
    answer: new_tab
  notify_after_open:
    ask: Should a successful Control-Return skip the usual "Captured" banner once
      Obsidian has opened?
    default: true
    why: The landing highlight already confirms the capture; a banner over Obsidian
      is redundant
    answer: true
phases:
- id: bob-open-notes
  title: bob reports which notes a capture should open
  depends_on: []
  size: medium
  description: 'bob-open-notes: add the always-present, batch-level `open_notes` array
    to `bob capture --format json` (dry-run and real runs alike). Each entry carries
    an absolute path, a role, a label, and a landing line and ref taken from the final
    staged text. Document the field in docs/capture.md and test every capture kind.'
- id: obsidian-open-uri
  title: Obsidian handler that opens several notes and lands on each
  depends_on: []
  size: medium
  description: 'obsidian-open-uri: in bob-plugins, register a read-only `obsidian://bob-open-notes`
    handler in bob-navigation-hotkeys. It runs one URL at a time, waits until bob''s
    writes are visible in Obsidian, opens the notes under the tab policy, and lands
    on and highlights the primary line. Also covers notices, tests, README, version
    bump, build, and `bob plugins sync`.'
- id: mac-open-core
  title: Portable open plan and URL builder in CaptureCore
  depends_on:
  - bob-open-notes
  - obsidian-open-uri
  size: small
  description: 'mac-open-core: in bob-mac-capture''s CaptureCore, decode `open_notes`
    and add `CaptureOpenPlan`: bob''s entries, or a single shared fallback for older
    bob that replaces the two duplicated heuristics. Add the `bob-open-notes` URL
    builder and helpers that save the plan into and read it back from notifications,
    all covered by portable tests.'
- id: mac-ctrl-return
  title: Control-Return gesture and the open pipeline
  depends_on:
  - mac-open-core
  size: medium
  description: 'mac-ctrl-return: handle Control-Return (with Command-Return as an
    alias in the editor) across the editor, completion, pickers, prompts, the successor
    card, the stash, and the template shelf. Open the plan through one Obsidian-activating
    NSWorkspace call that reports failures, and switch notifications over to the plan.
    Also update the README keyboard docs, add tests, and run the compile checks.'
- id: mac-open-affordances
  title: Footer hint, hold-to-reveal destinations, and picker hints
  depends_on:
  - mac-ctrl-return
  size: medium
  description: 'mac-open-affordances: add the footer `⌃↩ Open` chip and a destinations
    capsule that appears while Control is held alone. Add Control-Return hints to
    the pickers and the successor card, accessibility text from a portable presentation
    type, render tests, and README docs.'
proposed_by: bbugyi200.apollo.8q
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-17 15:54:12
status: done
bead_id: bob-cli-8w
---

- **PROMPT:** [prompts/202610/ctrl_enter_capture_and_open.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ctrl_enter_capture_and_open.md)
- **BEAD:** [bob-cli-8w](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8w/README.md)

# Plan: Control-Return captures, then lands you in Obsidian

## Where things stand today

The phases rely on these facts, all verified in the current trees:

- **bob-mac-capture** already has a capture-and-open gesture, but it's Command-Return
  and only in the editor.
  - `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` returns `.submitAndOpen`.
  - `CapturePanelController.perform` then calls `model.submit(openAfterCapture: true)`.
  - On success, `CapturePanelModel.completeSubmit` fires one `obsidian://open?path=` per
    unique `target`, plus the day file when the capture wrote it
    (`uniqueTargetURLs`/`captureWroteDayFile`).
  - Control-Return currently falls through to a plain capture.
- That gesture has four problems:
  1. Several `obsidian://open` URLs fired back to back replace one another in the active
     tab, so a two-note capture shows only one note.
  2. Notes open at the top (or the last scroll position), never on the captured line.
  3. Nothing on screen mentions the gesture.
  4. Which notes to open is decided by a client heuristic that exists twice:
     `CapturePanelModel.uniqueTargetURLs`/`captureWroteDayFile` and
     `NotificationService.notificationTargetPaths`/`dayFileChanged`. The two copies are
     currently identical.
- **bob-cli**'s capture JSON has no landing line for a plain new task. `task_line` is
  text only. `task_blocks[].line` (1-based, in final staged text) exists only for
  sub-bullets, dependency writes, `!` completions, and dependency drops. The batch-level
  `pomodoro_blocks[].line` gives Pomodoro headlines in the same final coordinates.
  - The final staged text of every touched file is in the planned batch's `text_files`
    (`src/native/capture/batch.rs`). `capture()` in `src/native/capture/mod.rs`
    assembles `CaptureResult::from_items` after the optional commit.
  - Dry and real runs produce identical plans.
  - Task refs are `<line>:<first 8 hex of SHA-256(line.trim_end())>`
    (`src/native/note_tasks.rs`, `TaskRef` and `line_digest`, with shared vectors in
    `tests/fixtures/line_digest/vectors.json`).
- **bob-plugins**: `bob-navigation-hotkeys` registers the read-only
  `obsidian://bob-goto-task` handler (`src/675-plugin-goto-task-uri.js`).
  - What it does well: maps absolute path to vault path (with a realpath fallback),
    re-finds the line by digest, reuses a leaf, lands via `focusTaskAnchor`, and records
    a `<C-o>` jump.
  - Its limits: it opens in the active tab, has no highlight, and several URLs in a row
    race, with the last one silently winning. It is the only protocol handler in the
    monorepo.

## Design

### Three layers, each in the repo that owns it

1. **bob decides** which notes a capture should open and where to land in each. This
   follows `decisions:mac-capture-is-a-thin-client`: new capture behavior lands in
   bob-cli and `docs/capture.md` first, and the app decodes and presents it.
2. **One Obsidian URL** carries the whole list. A new plugin handler opens every note as
   a tab, waits until bob's write is visible, lands, and highlights.
3. **The Mac app** binds the key, passes bob's list through unchanged (it never computes
   lines or digests), and shows beforehand what will open.

### The gesture: Control-Return finishes, then shows you

Return captures; Control-Return captures and opens what the capture changed. This
follows the convention already used across Bob ("`Ctrl+Enter` … (`Cmd+Enter` alias)",
bob-cli `docs/projects.md`). So Command-Return remains a full alias in the editor and
completion contexts, and keeps its own "& Capture" meaning inside pickers.

| Context                                                                        | Control-Return (new)                                                                                                                                                                                                                   | Command-Return                                                                                    |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Editor                                                                         | Capture, open, close the panel                                                                                                                                                                                                         | Same as Control-Return (same meaning as today, now using the new open path)                       |
| Completion visible                                                             | Accept the selected completion, then capture and open. This happens only when the accept leaves no completion, picker, or prompt open and the candidate was not the `@@template:` header; otherwise it only accepts                    | Same as Control-Return. Today it only accepts; the README already documents "Accept, then submit" |
| Picker: task link, dependency, blocks, complete, parent, active task, block ID | Insert the row, capture, open (Command-Return's path plus the open). A row with no ID opens Add block ID with a follow-up that keeps the open intent. A dependency or blocks row with no owner gets the same refusal as Command-Return | Unchanged (Insert & Capture)                                                                      |
| Go To picker                                                                   | Same as Return (Go To already opens Obsidian)                                                                                                                                                                                          | Unchanged                                                                                         |
| Add block ID / Name Pomodoro prompts                                           | Same as Return, as today; a prompt opened by a Control-Return picker accept keeps the open intent                                                                                                                                      | Consumed, as today                                                                                |
| Successor pull-in card                                                         | Pull In (Return's commit), then open                                                                                                                                                                                                   | Consumed, as today                                                                                |
| Stash picker, template shelf                                                   | Consumed (never reaches AppKit)                                                                                                                                                                                                        | Unchanged                                                                                         |

Control-Return means exactly Control. Control-Shift-Return and Control-Option-Return
still insert a newline, and Control-Command-Return follows Command-Return. Keypad Enter
behaves like Return. Success hides the panel and failure keeps the draft, as today.

### What opens (bob decides)

- Process items in source order. For each item, open its `target` (skipped when empty,
  as for a queued link). Then open the day file, if this batch changed it.
- Deduplicate by path. The first item to touch a path decides where that note lands.
- Incidental writes do **not** open: BLOCKS mirrors, the closed tasks' notes on `=x`,
  and prerequisite notes. These can fan out across many notes. The item's subject note
  plus today's ledger is what you want to see, and that matches today's target set.

| Item kind                                                                      | Where its note lands                                     | Where the day note lands     |
| ------------------------------------------------------------------------------ | -------------------------------------------------------- | ---------------------------- |
| `task` (new task, any route or `^id`)                                          | The new task line                                        | First touched Pomodoro block |
| `bullet` (`@route#Section`)                                                    | The new bullet line                                      | —                            |
| `sub_bullet`                                                                   | The first added child line                               | —                            |
| `task_toggle`, `link_guard`, `pomodoro_link`, `pomodoro_task`                  | The task line carrying `^block_id`                       | First touched Pomodoro block |
| `pomodoro_start`/`_adjust`/`_shift`/`_close`/`_reset` (target is the day file) | First touched Pomodoro block                             | (same entry)                 |
| `task_complete` (`!`)                                                          | The completed task line (`task_blocks` role `completed`) | First touched Pomodoro block |
| `task_dependency` (`&`, `*` owner)                                             | The owner task line (`task_blocks` role `dependency`)    | —                            |
| `project_note`                                                                 | The `^prj` line of the new note                          | First touched Pomodoro block |
| `ref` already in the library or intake                                         | No line: open the note or PDF                            | —                            |
| `ref` queued                                                                   | Nothing opens (empty target)                             | —                            |

"First touched Pomodoro block" means the first batch-level `pomodoro_blocks` entry for
that file whose role is not `removed`, in first-touch order; its `line` is the headline.

### How it looks

**In Obsidian:**

- Notes open as tabs according to the tab policy, in entry order, and the first note
  ends up active and focused.
- The caret goes to column 0 of the landed line, which is centred.
- That line gets a short highlight, about 1.2 s: an accent tint with a slim accent bar
  on its left edge, fading out. With reduced motion, a static tint shows briefly
  instead.
- Secondary tabs open in the background, already scrolled to their landing lines.

**In the panel:**

- A quiet `⌃↩ Open` keycap chip sits beside the primary button, and its tooltip lists
  where it goes.
- Holding Control alone for 400 ms tints the chip and fades in a small "Opens in
  Obsidian" capsule listing each destination. Control chords never flicker the UI.
- Pickers and the successor card show a `⌃↩ … & Open` hint.
- The status reads "Capturing and opening…".

## Contracts

### `open_notes` in `bob capture --format json`

- A top-level, batch-level array that appears on **every** success, dry-run and real run
  alike, and is never repeated per item.
- An empty array means nothing to open, for example a queued link.
- When the key is absent, the `bob` is older than this feature.

```json
"open_notes": [
  {"path": "/Users/bryan/bob/mac_inbox.md", "relative_path": "mac_inbox.md",
   "role": "note", "label": "inbox", "line": 42, "ref": "42:35a733bd"},
  {"path": "/Users/bryan/bob/2026/20261017.md", "relative_path": "2026/20261017.md",
   "role": "day", "label": "Today", "line": 18, "ref": "18:0c1d2e3f"}
]
```

| Field           | Rule                                                                                                                                              |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `path`          | Always absolute. A relative `BOB_DAY_FILE` is joined onto the vault directory.                                                                    |
| `relative_path` | Vault-relative when the file is under the vault, otherwise the absolute path.                                                                     |
| `role`          | `"day"` when the path is the batch's day file, else `"note"`. Clients must tolerate unknown roles.                                                |
| `label`         | `"Today"` for the day file. Otherwise the contributing item's `route_label` when non-empty, else `relative_path` without `.md`.                   |
| `line`          | Optional. A 1-based line in the file's final text (staged text for touched files, on-disk text for untouched ones). Omitted when bob cannot land. |
| `ref`           | Present exactly when `line` is: `<line>:<digest>`, with the digest computed from that final line exactly as `note_tasks` computes it.             |
| `block_id`      | Optional. Present when the landed line carries `^block_id`.                                                                                       |

### `obsidian://bob-open-notes` URI v1

```
obsidian://bob-open-notes?count=<1..8>&path1=<abs>[&line1=<n>][&ref1=<LINE:digest>][&block1=<id>]…[&total=<n>]
```

| Param    | Rule                                                                                    |
| -------- | --------------------------------------------------------------------------------------- |
| `count`  | Required, matching `^[1-8]$`.                                                           |
| `pathK`  | Required for every K from 1 to `count`. An absolute filesystem path.                    |
| `lineK`  | Optional. A positive 1-based integer.                                                   |
| `refK`   | Optional. Only valid together with `lineK`; must match `^[1-9][0-9]*:[0-9a-f]{8}$`.     |
| `blockK` | Optional. Must match `^[A-Za-z0-9-]+$`.                                                 |
| `total`  | Optional. An integer of at least `count`, sent when the client truncated a longer list. |
| other    | Ignored.                                                                                |

- Encoding matches `bob-goto-task`: RFC 3986 unreserved characters and `/` stay literal,
  and every other byte (including `+`, `&`, `=`, `#`, `%`, `:`, and spaces) is
  percent-encoded.
- Indexed keys avoid depending on how Obsidian collapses repeated query keys.
- Entry 1 is the primary note.

## Phase bob-open-notes: bob reports which notes a capture should open

Repo: bob-cli (the primary repo).

1. Add a module `src/native/capture/open_notes.rs` that builds `Vec<OpenNoteJson>` after
   the batch is planned. Run it before the commit and the move of `batch.items`, so dry
   and real runs match. Inputs:
   - the planned items' results (`target`, `relative_target`, `day_file`, `block_id`,
     `task_line`, `kind`, `route_label`, `project_note`);
   - the batch's `text_files`, using `current` as the final text;
   - the batch-level `task_blocks` and `pomodoro_blocks`.

   Read the on-disk text only for a path the batch did not stage (for example a toggle
   whose route note did not change).

2. Choose entries and roles exactly as described under "What opens".
   - A day file counts as changed when its `BatchTextFile` is present with
     `current != original`.
   - The day file is the one `pomodoro::day_file_for` returns for this run.
3. Choose landing lines per the "What opens" table.
   - Prefer positions the planner already knows: batch-level `task_blocks` /
     `pomodoro_blocks` lines, which are already in final coordinates, and the
     `^block_id` task line found with the existing task-line helpers.
   - Otherwise fall back to the existing new-task locator in `dependencies.rs` (the
     helper behind `locate_new_task_ref`: the last exact full-line match of
     `task_line`).
   - Never use per-item line numbers. A later item in the same draft can shift them.
4. Compute `ref` with `note_tasks` `TaskRef` / `line_digest` over the final line. Never
   add a second digest implementation.
5. Add `open_notes: Vec<OpenNoteJson>` to `CaptureResult` (`output.rs`).
   - Serialize it always, without `skip_serializing_if`.
   - Pass it through `from_items`.
   - Human output does not change.
6. Document the contract in `docs/capture.md` next to `task_blocks` / `pomodoro_blocks`
   under "Input, stdin, and JSON output": the field table, the selection and landing
   rules, the empty-array meaning, and the statement that clients should pass entries
   through rather than re-derive them.
7. Add tests in a new `tests/cli/capture/open_notes.rs`, registered wherever its
   siblings are registered. Cover:
   - Each row of the "What opens" table: a new inbox task, a routed task, a section
     bullet, a sub-bullet, an Ensure Next toggle that links into a Pomodoro, `=`, `=x`,
     `=x0`, `+N`, `!` with a ledger change, a project note with task links, a queued
     link (empty array), a library-hit link (no line), and `&` / `*` dependencies.
   - A batch with three items into one note plus a day-file touch: dedupe, order, and
     the first item's landing.
   - A template draft.
   - Dry run equals real run.
   - After a real run, every `ref` digest equals the digest of the line at `line` in the
     written file, and every `path` is absolute.

   Run `just check` (or `just all`) until it is green.

## Phase obsidian-open-uri: Obsidian handler that opens several notes and lands on each

Repo: bob-plugins. Open it with `/sase_repo` and read its `AGENTS.md`. Edit the `src/`
fragments, never `main.js`, and keep every fragment at or below 1000 lines.

1. Add a new fragment `plugins/bob-navigation-hotkeys/src/676-plugin-open-notes-uri.js`
   and list it in `fragments.json`.
   - Register `bob-open-notes` beside `bob-goto-task` in `490-plugin-lifecycle.js`, with
     the same "only when the API exists" guard.
   - Install the mixin in `690-install-plugin-mixins.js` and add test exports in
     `700-exports.js`.
   - Reuse the goto helpers (path mapping, `bobLineDigest`, `refindGoToTaskLine`,
     `focusTaskAnchor`, the leaf helpers). Extract shared pieces rather than copying
     them.
2. **Validation** follows the URI v1 table and is all-or-nothing.
   - Any invalid parameter shows `Bob Open: invalid link`, and nothing opens.
   - A missing file or one outside the vault is a per-entry failure, and the other
     entries still open.
3. **One URL at a time.** Chain invocations on a promise, so two quick Control-Return
   captures never race. The later URL ends up active.
4. **Freshness.** After `onLayoutReady`, wait under one shared 2000 ms deadline.
   - What to wait for: each path must resolve to a vault file, which covers a note bob
     just created such as a new project note or the first day file. For each entry with
     a `ref`, its digest must also be found by re-find steps 1–3 in the content Obsidian
     will show: the open editor buffer when the note is open, else `vault.cachedRead`.
   - Re-check on vault `create`/`modify` events and on a 50 ms timer.
   - After the deadline, proceed. An unresolved file gets
     `Bob Open: <vault path> not found`. An unresolved digest lands on the clamped line
     with `Task changed since capture — landed near line N`.
5. **Tabs.** Tab-strip order follows entry order, and the primary note ends up active.

   > [!decision] tab_policy = new_tab For each entry, reuse a leaf in any split or
   > window that already shows the file. For the first entry only, an active empty tab
   > (no file) may be used. Otherwise open the entry in a new tab in the active tab
   > group. Secondary entries open in the background. Finally, activate and focus the
   > primary leaf.

   > [!decision] tab_policy = replace_active Entry 1 opens through Go To's
   > `openMarkdownFileWithLeafReuse`: reuse a leaf already showing the note, else
   > replace the active tab. Entries 2..N open like `new_tab`: reuse a leaf, else a new
   > background tab.

6. **Landing.**
   - **Primary note:** land through `focusTaskAnchor` (column 0, centred, Reading view
     switched to editing, editor focused). Record one `<C-o>` jump from the markdown
     cursor that was active before the handler ran.
   - **Secondary markdown notes:** open scrolled to their re-found line (for example
     through `openFile`'s `eState: { line }`) without stealing focus.
   - **Non-markdown files** (an intake PDF): open them with no landing.
7. **Highlight** (primary landing only).
   - A CodeMirror line decoration (StateField plus StateEffect, through
     `registerEditorExtension`) adds `bob-landing-flash` to the landed line. A cleanup
     effect removes it after the animation.
   - It must never change the document, its undo history, or the selection.
   - Add the CSS to `plugins/bob-navigation-hotkeys/styles.css`: a roughly 1.2 s
     ease-out keyframe from an accent tint
     (`color-mix(in srgb, var(--interactive-accent) 24%, transparent)`) plus a 2px inset
     left bar in `var(--interactive-accent)`, fading to transparent.
   - Under `@media (prefers-reduced-motion: reduce)`, show a static tint for about 600
     ms with no animation.
   - It must work in light and dark themes.
8. **Notices.**
   - `Bob Open: invalid link`
   - `Bob Open: <path> is outside this vault`
   - `Bob Open: <vault path> not found`
   - `Bob Open: could not open <name>`
   - `Bob Open: opened N of M notes`, when `total` > `count`
   - `Task changed since capture — landed near line N`
9. **Read-only.** The handler never writes the vault. It does not park or advance the
   review walk beyond what any note navigation does, which is the same as Go To. Add a
   test for this.
10. **Tests.** Add a new `scripts/test-navigation-open-notes-uri.cjs`, added to
    `package.json`'s `test` list, modelled on
    `scripts/test-navigation-goto-task-uri.cjs` and its harness. Stub
    `@codemirror/state` if needed. Cover:
    - registration;
    - every validation rule;
    - path mapping, including realpath;
    - the tab policy (reuse, empty-tab reuse, new tabs in order, primary active);
    - freshness (a file that appears late, a digest that appears late, the deadline
      fallback);
    - re-find;
    - opening a non-markdown file;
    - the truncation notice;
    - two invocations running one after the other;
    - no vault writes;
    - the highlight added and removed on the primary note only.
11. **Docs and release.**
    - Add a README section "Open notes URI (`obsidian://bob-open-notes`)" after "Go to
      task URI", mirroring its URI, re-find, Notices, and Terminal check subsections.
    - Add one sentence to the plugin table row.
    - Bump the bob-navigation-hotkeys minor version in `manifest.json` and in the README
      table.
    - Run `npm run build`, `npm test`, and `npm run validate`.
    - Deploy with `bob plugins sync` as the repo requires. From a linked checkout, pass
      `--repo <checkout>` per bob-cli `docs/plugins.md`.

## Phase mac-open-core: Portable open plan and URL builder in CaptureCore

Repo: bob-mac-capture. Open it with `/sase_repo` and read its `AGENTS.md`. This phase
touches only `Sources/CaptureCore` and `Tests/CaptureCoreTests`, so
`just portable-check` fully verifies it. Run it on athena if this host has no `swift`.

1. Decode `open_notes` on the top-level capture success. This is the type that exposes
   `normalizedCaptures` and `globalDestination`.
   - Add `openNotes: [CaptureOpenNote]?` via `decodeIfPresent`: `nil` means an older
     bob, and `[]` means nothing to open.
   - `CaptureOpenNote` has `path`, `relativePath`, `role` (`.note`, `.day`, or
     `.other(String)`), `label`, `line: Int?`, `ref: String?`, and `blockID: String?`.
   - Drop an invalid optional value on its own (non-positive `line`, malformed `ref` or
     `block_id` per `ObsidianOpenURL`'s validators) and keep the entry.
   - Drop an entry with an empty or non-absolute `path`.
2. Add `Sources/CaptureCore/CaptureOpenPlan.swift`:
   - Properties: `notes`, `source` (`.bob` or `.legacy`), and `isEmpty`.
   - Initializer:
     `init(openNotes: [CaptureOpenNote]?, captures: [CaptureCommandSuccess])`. With
     bob's entries, keep them in order, deduplicated by path.
   - Legacy fallback: move the current heuristic here unchanged. For each capture,
     include its `target`, then its `dayFile` when any of these holds: a reset, a close,
     a session start, `toggle.dayFileChanged`, a task completion with a ledger change or
     a linked successor, `link.dayFileChanged`, or a project note with task links.
     - Legacy entries have no lines and get role `.day` for the day file.
     - Their label is `routeLabel`, else `relativeTarget`.
   - Leave the two existing copies in place for now; the next phase deletes them.
3. Add `ObsidianOpenURL.openNotesURL(for: CaptureOpenPlan) -> URL?`:
   - Return nil for an empty plan.
   - Send at most the first 8 notes, plus `total` when truncated.
   - Use the same encoder as `taskGotoURL`.
   - Emit `lineK` when valid, `refK` only alongside `lineK`, and `blockK` when valid.
4. Add a notification payload round trip: `CaptureOpenPlan.notificationPayload`, a
   plist-safe array of dictionaries, and `init?(notificationPayload:)`.
5. Tests:
   - `Tests/CaptureCoreTests/CaptureOpenPlanTests.swift`: decoding of a present, absent,
     empty, or malformed field; legacy parity, porting the existing model and
     notification target-path cases; dedupe; round trip.
   - URL tests: spaces, `+`, `&`, `#`, `:` in refs, non-ASCII paths, `count`, truncation
     with `total`.
   - If the repo keeps real-bob JSON fixtures, add one from the new bob: a batch with a
     linked task.

## Phase mac-ctrl-return: Control-Return gesture and the open pipeline

Repo: bob-mac-capture (open with `/sase_repo`).

1. **Router** (`CaptureKeyCommandRouter.swift`). Implement the gesture table.
   - New commands: `.acceptCompletionThenSubmitAndOpen`,
     `.acceptPickerRowAndSubmitAndOpen` (not for Go To, where Control-Return is
     `.acceptPickerRow`), and `.confirmPullInChoiceAndOpen` (non-repeat, like the other
     card keys).
   - Stash picker and template shelf: `.consumeKey`.
   - Command-Return in the editor and completion maps to the same commands as
     Control-Return. Update the accept-only Command-Return completion assertion in
     `BobMacCaptureTests.swift`.
2. **Controller and model.**
   - **Completion:** accept, then submit with open only when, right after the accept, no
     completion, picker, or inline prompt is visible, the accepted candidate was not
     `@@template:`, and `hasDraft` holds.
   - **Pickers:** give the picker accept and the Add block ID follow-up an open-carrying
     variant, for example a follow-up enum with `.select`, `.submit`, and
     `.submitAndOpen`. Update every `submit(openAfterCapture: false)` follow-up site to
     honor it: task link, dependency, complete, blocks, and parent picker.
   - **Pull-in card:** set the stored `openAfterCapture` to true, then confirm with
     Return semantics (marked offers, else the highlighted one).
3. **Open pipeline** in `CapturePanelModel`.
   - Replace `targetOpener` with
     `noteOpener: (URL, @escaping @MainActor (Bool) -> Void) -> Void`. In production it
     uses `NSWorkspace.OpenConfiguration` with `activates = true`, as `taskGotoOpener`
     does.
   - On success with `openAfterCapture`, build
     `CaptureOpenPlan(openNotes: success.openNotes, captures:)` and open its single URL.
     An empty plan opens nothing.
   - If the opener fails, post a notification "Captured, but Obsidian didn't open"
     carrying the error, because the panel is already gone.
   - Delete `uniqueTargetURLs` and `captureWroteDayFile`.
4. **Notifications** (`NotificationService.swift`).
   - Store the plan payload under a new userInfo key. Keep reading the old
     `targetPath`/`targetPaths` keys for notifications that were already delivered.
   - The body click and the Open Note(s) actions open the plan's single URL. The action
     is titled by note count.
   - Delete `notificationTargetPaths` and `dayFileChanged` in favour of the plan.

   > [!decision] notify_after_open When Control-Return (or its alias) opened Obsidian
   > successfully, post no success banner. Failures still notify: a capture failure
   > keeps the panel and the draft, and an open failure posts the notice above. Return
   > without open still notifies as today. To keep this reliable, post nothing until the
   > opener reports its result.

   > [!decision] notify_after_open = no Keep posting the success notification exactly as
   > today after Control-Return. Its Open action now uses the plan URL.

5. **README.**
   - Keyboard table: add a Control-Return row. Make the Command-Return row read "Same as
     Control-Return" for the editor and completion columns, which also corrects the
     completion cell.
   - Add Control-Return to each picker key list: active task, task link, Go To, parent,
     dependency, blocks, and complete.
   - Update the successor-card paragraph.
   - Replace "Command-Return opens every unique returned target in source order" with
     the open-plan behavior: bob's `open_notes`, one `bob-open-notes` URL, tabs, and
     landing. Note the requirement on the new bob-navigation-hotkeys version, and that
     an older bob opens notes without landing.
   - Update the `in_library`/`in_intake` reference paragraph and the Notifications
     section.
6. **Tests** (`BobMacCaptureTests`). Cover:
   - router cases for every context, including the exact-Control rule and keypad Enter;
   - completion accept-then-capture, plus the guarded stop for template and follow-up
     surfaces;
   - picker Insert, Capture & Open, including a row with no ID going through Add block
     ID;
   - the pull-in card with Control-Return;
   - the opener receiving exactly one `bob-open-notes` URL with activation;
   - the legacy fallback, an empty plan, and the opener-failure notice;
   - notification payload and click routing (old and new keys);
   - the `notify_after_open` branch.
7. **Verification**, following `AGENTS.md`.
   - Run `just portable-check`, on athena if this host lacks `swift`.
   - Best-effort scratch compile on the Mac: rsync without `.build`/`.git`, run
     `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
     Bryan's checkout or `just install`.
   - If the Mac is offline, say in the final report that the macOS targets are
     unverified.

## Phase mac-open-affordances: Footer hint, hold-to-reveal destinations, and picker hints

Repo: bob-mac-capture (open with `/sase_repo`).

1. **Preview plan.** Keep the latest live preview's plan (`previewOpenPlan`), built from
   the same dry-run success (`open_notes`, else the legacy fallback). Clear it wherever
   the preview results are cleared.
2. **Portable presentation:** `Sources/CaptureCore/CaptureOpenHintPresentation.swift`,
   tested in `CaptureCoreTests`. It provides:
   - `isAvailable`;
   - `accessibilityLabel`, for example "Control-Return: capture and open inbox at line
     42 and Today in Obsidian";
   - `rows`: a symbol (`doc.text` for notes, `calendar` for the day note), the label,
     and a detail ("line 42" or none), with at most 4 rows followed by "+N more".
3. **Footer chip** (`CapturePanelFooter` in `CapturePanelView.swift`), placed just
   before the primary button.
   - Style: a borderless button styled like the picker key hints, with the keycap `⌃↩`,
     "Open", and an `arrow.up.forward` glyph, in secondary foreground.
   - Shown when the primary action is enabled and either the preview plan is non-empty
     or this draft has no preview yet. Hidden when the current preview's plan is empty,
     as for a queued link.
   - Clicking it runs `submit(openAfterCapture: true)`.
   - Its `.help` tooltip lists the destinations.
   - It is the first element dropped under `ViewThatFits` in narrow panels.
4. **Hold to reveal.**
   - Extend the panel's local event monitor in `CapturePanelController` to
     `.flagsChanged`. The precedent is `RefsPanel.swift`, which monitors
     `[.keyDown, .flagsChanged]` and feeds `RefsPanelModel.updateCopyModifiers`.
   - When Control alone has been held for 400 ms with no keyDown, set
     `openHintRevealed`.
   - Clear it at once on any keyDown, any modifier change, panel hide, or submit.
   - While revealed, the chip takes the accent tint, and an overlay capsule fades in and
     rises 4 pt over 160 ms above the footer's trailing edge. The capsule uses the
     completion list's material, a 10 pt corner radius, and its shadow, with the header
     "Opens in Obsidian" and then the rows. Because it is an overlay, the footer never
     reflows.
   - Reduce Motion: crossfade only.
   - The capsule is hidden from accessibility, because the chip's label already carries
     the destinations.
5. **Picker and card hints.**
   - `CapturePickerPresentation.keyHintItems` gains a Control-Return item for every
     source except Go To: "Capture & Open", or "Complete & Open" for the complete
     picker. It has the lowest priority, so it is dropped first, and the card
     accessibility hints mention Control-Return.
   - The successor card shows a `⌃↩ Pull In & Open` hint at the lowest priority. When
     the card will already open notes, the Pull In and Skip buttons show a trailing
     `arrow.up.forward` glyph and that hint is omitted.
6. **Render tests** through `RenderFixtureWriter`.
   - A new editor-footer fixture, which nothing renders today: the chip at rest, and
     revealed with the capsule. Light and dark, at 760 and 620 pt.
   - Picker hints that include Control-Return.
   - The successor card showing the open glyph.
7. **README:** describe the chip, the hold reveal, and the hints.
8. Run the same verification as the previous phase.

## Rollout and manual check

- **Deploy order:** the plugin syncs in its own phase. Bryan then installs the new bob
  and the app on the Mac (`just install-all`). Agents never run `just install` on the
  Mac.
- **Partial upgrades:**
  - An older bob means a legacy plan: the notes still open as tabs, without landing.
  - An older plugin means Obsidian ignores the unknown action. The README names the
    required plugin version.
- **Manual checklist for Bryan:**
  - a new inbox task;
  - a task linked into a Pomodoro (two tabs, inbox active);
  - `=x`;
  - `!` completion;
  - a three-item batch;
  - a queued link (nothing opens, and the chip is hidden);
  - Obsidian cold-launched by Control-Return;
  - two quick Control-Return captures in a row;
  - Control chords in the editor (no flicker);
  - holding Control alone (the capsule appears).

## Out of scope

- Changing `bob-goto-task`, Go To, or review-walk landings. The highlight could be
  offered there later.
- Opening incidental notes (BLOCKS mirrors, closed tasks' notes).
- A setting to choose the opener, or plain `obsidian://open`.
