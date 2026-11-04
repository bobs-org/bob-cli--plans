---
tier: epic
title: 'Go To picker: Command-X toggles completed tasks'
goal: 'In Bob Mac Capture''s `/` Go To picker, Command-X toggles whether completed
  tasks appear in the results. bob supplies them through a new additive `capture-complete
  --closed-tasks` flag, and the Mac app renders them.

  '
phases:
- id: bob
  title: 'bob-cli: capture-complete --closed-tasks for task_goto'
  depends_on: []
  size: medium
  description: 'bob: add the -C/--closed-tasks flag and append newest-first closed
    rows from the vault catalog to task_goto, with tests and docs.'
- id: mac
  title: 'bob-mac-capture: Command-X toggle in the Go To picker'
  depends_on:
  - bob
  size: medium
  description: 'mac: route Command-X in the Go To picker to a refetch with --closed-tasks,
    render a trailing Completed section, degrade on older bob, and update tests and
    docs.'
decisions:
  completed_scope:
    ask: Should completed rows come from every vault note, including done/ archives,
      ref notes, and daily notes?
    choices:
      vault: 'All closed tasks the &/(( catalog finds: ~3.8k rows, ~170 ms per fetch'
      capture_notes: Only closed tasks from the notes / lists, plus their done/ archives
    default: vault
    why: Most finished tasks live in done/ archives or ref notes; the catalog already
      exists
    answer: vault
  canceled_rows:
    ask: When completed tasks are shown, should canceled tasks show too?
    choices:
      closed: Done and canceled tasks both list; canceled rows keep their glyph
      done_only: Only done tasks list; canceled tasks stay hidden
    default: closed
    why: Both are closed history; the & and (( completed groups already include both
    answer: closed
  sticky_toggle:
    ask: Should the toggle stay on for later / searches until you turn it off?
    choices:
      session: Each new / draft starts open-only; the toggle lasts for that draft
      sticky: The app remembers the last state until you toggle it off or quit
    default: session
    why: Open tasks are the usual target; a forgotten toggle would bury them
    answer: session
proposed_by: bbugyi200.athena.180
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 11:50:51
status: wip
bead_id: bob-cli-h3
---

- **PROMPT:** [prompts/202611/goto_completed_toggle.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/goto_completed_toggle.md)
- **BEAD:** [bob-cli-h3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h3/README.md)

# Go To picker: Command-X toggles completed tasks

## Context

In Bob Mac Capture, typing `/` as the whole draft opens the **Go To picker**
(`task_goto` completion context). The Mac app gets the rows from
`bob capture-complete --all-tasks --cursor N --format json -- '/'`. It filters them
locally and opens the chosen row with
`obsidian://bob-goto-task?path=…&line=…&ref=…[&block=…]`.

Today bob builds `task_goto` rows in
`src/native/capture_complete/candidates.rs::task_goto_candidates`. It uses
`capture_link_tasks::discover`, which lists only linkable **open** tasks (statuses ` `,
`?`, `*`, `/`) in routable inbox, area, and non-terminal project notes. As Bryan
suspected, completed tasks never appear; the code confirms it.

This plan adds **Command-X** to the Go To picker. It toggles whether the results include
completed tasks. Bob supplies those rows behind a new `capture-complete` flag. The Mac
app only toggles, refetches, and renders them. That follows the accepted decision
`mac-capture-is-a-thin-client`.

### What already exists (reuse, do not rebuild)

- `src/native/capture_dependency_tasks.rs::discover`. A vault-wide catalog of every
  task: open and closed, hidden, terminal projects, ref notes, daily notes, and `done/`
  archives. Each `DependencyTask` has an `open: bool`, an exact `note_path`, a display
  `locator`, `status_*`, `text`, `section`, `depth`, 1-based `line`, `task_ref`,
  `block_id`, `hidden`, and `task_kind`. The `&` and `((` pickers use it and show closed
  rows in a trailing "completed history" group.
- Closed task lines carry `[completion:: YYYY-MM-DD]` or `[cancelled:: YYYY-MM-DD]`.
  `task_fields::inline_fields(raw_line, name)` and
  `task_fields::parse_strict_calendar_date` read them.
  `capture_link_tasks::scheduled_facts` uses the same approach.
- The `obsidian://bob-goto-task` handler (bob-plugins, `docs/goto-task-uri.md`) opens
  any Markdown note in the vault by absolute path and line, `done/` archives included.
  **No plugin change is needed.**
- Mac precedents:
  - `DependencyPickerPresentation.swift` has a separate "Completed history" section,
    ranks open matches before completed ones, and keys rows by `note_path|ref`.
  - `BobProcessClient.capturePomodoros(includeCompleted:)` shows how a client method
    takes an optional flag.
  - `BobProcessClient.isUnsupportedOptionFailure` detects an older bob.
- Mac key routing: `CaptureKeyCommandRouter.pickerCommand` gets every picker key and
  already handles Command chords (Command-Up/Down/Backslash). Command-X currently
  returns nil, so the filter field cuts text natively. The `KeyCode` table has no `x`
  yet (macOS keyCode 7).

### Contract (shared by both phases)

`bob capture-complete` gains `--closed-tasks` / `-C`, a boolean flag.

- It affects only the `task_goto` context. Every other context ignores it, and output
  without the flag stays byte-identical to today.
- With the flag, `task_goto` returns today's open rows unchanged, in the same order.
  After them come closed rows from the vault-wide catalog, filtered by the query with
  the same `match_score` matcher. Open matches always come before closed matches.
  - > [!decision] completed_scope = vault: closed rows are every `!task.open` task from
    > `capture_dependency_tasks::discover`.
  - > [!decision] completed_scope = capture_notes: keep only closed tasks whose
    > `note_path` is a capture target from `scan_capture_targets`, or a `done/` archive
    > whose source (`collect_done::source_relative_path_from_archive`) is one.
  - > [!decision] canceled_rows = done_only: keep only rows whose status type is DONE.
    > Canceled and other terminal types drop out. Name the flag `--done-tasks` / `-D`
    > everywhere this plan says `--closed-tasks` / `-C`.
- Closed-row order for an empty query: newest close date first. The close date is the
  first strict `completion` field for DONE rows and the first `cancelled` field
  otherwise. Rows with no date come after dated rows. Ties break by `note_path`, then
  `line`. For a non-empty query, sort by score descending, then by this order.
- Each closed row uses the existing `TaskGotoCandidate` shape plus additive changes:
  - `group: "completed"`. Add a `Completed` variant to
    `capture_link_tasks::LinkTaskGroup`. Only `task_goto` emits it.
  - `route`: the catalog `locator` (unique basename, else the vault-relative path
    without `.md`).
  - `note_kind`: becomes `Option<CaptureTargetKind>` with
    `skip_serializing_if = "Option::is_none"`. Open rows still serialize it, so their
    bytes do not change. Closed rows omit it.
  - `note_path`, `target` (`bob_dir.join(note_path)`), `ref`, `line`, `block_id`,
    `status_*`, `text`, `section`, `depth`, `task_kind`: as for open rows.
  - `scheduled: null`, `pomodoro: null`.
  - New additive `closed: "YYYY-MM-DD"` (omitted when absent) so the Mac can show the
    date.
- Warnings from the vault catalog are appended to the response's bounded warnings.
- Without the flag, an older Mac app sees no change. With the flag, an older bob fails
  with clap's `unexpected argument '--closed-tasks'`. The Mac detects that and degrades
  (phase 2).

## Phase 1 — bob-cli: `capture-complete --closed-tasks` for `task_goto`

Repository: bob-cli (this project's primary repo).

1. `src/native/capture_complete/cli.rs`
   - Add `closed_tasks_arg()`: long `closed-tasks`, short `C`, `SetTrue`. Help: "Include
     done and canceled tasks from the whole vault (task_goto context only)". Adjust the
     wording to the chosen decisions.
   - Register it in alphabetical order: all-tasks, bob-dir, closed-tasks, cursor,
     format, help.
   - Add an example: `bob capture-complete -C -c 1 -f json -- '/'`.
   - Extend the long-about `task_goto` sentence with one sentence on the flag.
   - Pass the flag through `run` into `build_result`.
2. `src/native/capture_flag_guard.rs::capture_complete_spec`: add
   `long("closed-tasks", false)` and add `C` to `short_flags`. Otherwise the
   unknown-flag guard rejects it.
3. `src/native/capture_complete/engine.rs`: thread a `closed_tasks: bool` (or a small
   options struct, if that is cleaner than a fifth positional bool) from `build_result`
   to the `CompletionContext::TaskGoto` arm. Update every `build_result` caller,
   including shell completion and tests, with `false`.
4. `src/native/capture_complete/candidates.rs::task_goto_candidates`: when the flag is
   set, append closed rows as described in the Contract above. Keep the closed-row
   selection, close-date extraction, and ordering in a small helper. Either add a
   `closed_on: Option<String>` field to `DependencyTask`, filled in `discover` from the
   raw line, or put a function in `capture_dependency_tasks`. Do not re-read note files.
5. `src/native/capture_complete/model.rs`:
   - `TaskGotoCandidate.note_kind` becomes optional and skipped when None.
   - Add `closed: Option<String>`, skipped when None.
   - Add `LinkTaskGroup::Completed`. Fix any exhaustive matches, for example
     `render.rs::task_parent_line`; that arm is unreachable there but must compile.
   - Update the doc comment.
6. `src/native/capture_complete/render.rs::task_goto_line`: the human detail for a
   closed row appends `· closed DATE` when `closed` is present.
7. Tests:
   - `tests/cli/capture/task_goto.rs`: the worked example already has `- [x] … ^old-fix`
     and `- [-] … Dropped idea` in `sase.md`. Add a `done/` archive note and a ref-style
     note. Cover:
     - The JSON without the flag is byte-identical to today.
     - With `-C` and `--closed-tasks`: open rows first and unchanged, then closed rows
       newest-first with `group: "completed"`, `closed`, `note_path`, absolute `target`,
       a `locator` route, and `note_kind` omitted.
     - A query filters closed rows, and open matches still come first.
     - Other contexts (`:` `task_link`) ignore the flag.
     - The unknown-flag guard accepts `-C`.
   - Add unit tests for the close-date order: dated vs undated rows, and `completion` vs
     `cancelled`.
8. Docs:
   - `docs/capture.md`, Go To section (around "`bob capture-complete` serves context
     `task_goto`"): document the flag, the trailing closed rows, their order,
     `group: "completed"`, `closed`, and optional `note_kind`.
   - Run `just docs-gen` to refresh `docs/reference/cli/capture-complete.md`.
   - Then run `just docs-check`.
   - Do not edit `CHANGELOG.md` or the `Cargo.toml` version. That is the release train's
     job (decision `bob-cli-releases-ride-a-daily-train`). The change is additive, so
     the commit is a plain `feat(capture):`.
9. Gates: `cargo fmt`, `cargo clippy --all-targets`, `cargo test`, and
   `just docs-check`. Also time the real-vault fetch with `--closed-tasks` (read-only),
   e.g. `bob capture-complete -C -c 1 -f json -- '/'` against `~/bob`. It should stay
   near the `((` catalog cost of about 0.2 s on athena. Record the number in the phase
   notes.

## Phase 2 — bob-mac-capture: Command-X toggle in the Go To picker

Repository: the `bob-mac-capture` linked repo. Open it with `/sase_repo`. Read its
`AGENTS.md` first: compile rules, the scratch-copy Mac typecheck, `.swift-format`, and
the docs gate.

1. `Sources/CaptureCore/BobProcessClient.swift`:
   - `captureComplete(_:cursor:includeClosedTasks: Bool = false)` appends
     `--closed-tasks` when true. Keep `--all-tasks`.
   - Map clap's unexpected-argument failure for `--closed-tasks` through
     `isUnsupportedOptionFailure` to the existing `BobClientError.unsupportedOption`
     path (see `captureAgenda`).
2. Key routing, `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`:
   - Add `KeyCode.x = 7`.
   - Add a `CaptureKeyCommand.toggleGotoClosedTasks`.
   - In `pickerCommand`, `KeyCode.x` with `modifiers == .command` returns it **only**
     when `context.pickerSourceIsTaskGoto`. Return `.consumeKey` for key repeats.
   - Every other picker keeps native Command-X cut. In the Go To filter field Command-X
     no longer cuts; Command-C, Command-A, and Delete still work. Update the
     `pickerCommand` doc comment, which lists Command-X among native keys.
   - The existing `pickerDispatchInFlight` consume still applies.
3. State, `CapturePickerState.swift` / `CapturePanelModel.swift`:
   - Add `includesClosedTasks: Bool` to `CapturePickerState` and
     `CapturePickerChipState`, defaulting to false. Escape → chip → reopen keeps it.
   - > [!decision] sticky_toggle = session: a fresh `/` session (new draft, auto-open
     > from the editor) starts false.
   - > [!decision] sticky_toggle = sticky: also keep a panel-model
     > `gotoIncludesClosedTasks` that new Go To sessions seed from. It lives in memory
     > only (not `AppSettings`) and resets when the app quits.
   - Handling `.toggleGotoClosedTasks`: flip the desired state, then refetch with
     `captureComplete(draft, cursor: r.start, includeClosedTasks: new)`. That is the
     same full-snapshot cursor `handleTaskLinkCompletion` uses.
   - Guard the response like the existing refetch: current draft unchanged,
     `context == "task_goto"`, and `replacement == r`. A generation counter drops stale
     responses, so the last press wins.
   - Rebuild `.taskGoto(TaskLinkPickerIndex…)` with the new candidates. Keep
     `filterText`. Keep the selected row if its key still exists, otherwise select the
     first row. Recompute the row budget as a fresh open would.
   - Turning the toggle off refetches without the flag, so there is one code path and
     the open rows are fresh.
   - The chip-reopen refetch (the `needsRefetch` block near the `case .taskGoto:` chip
     branch) passes the chip's `includesClosedTasks`.
   - If bob reports `unsupportedOption`, keep the picker as it was, leave the toggle
     off, and show "Update bob to show completed tasks" in the picker's warning/status
     line.
   - If the fetch fails any other way, keep the picker and show Bob's error the same way
     other picker refetch failures do.
   - VoiceOver: announce "Showing completed tasks, N tasks" or "Hiding completed tasks,
     N tasks" (see `appearedAnnouncementPrefix`).
4. Presentation, `Sources/CaptureCore/TaskLinkPickerPresentation.swift`:
   - `TaskLinkGroup` gains `.completed` (wire `"completed"`).
   - In `.taskGoto` mode, key rows by `note_path|ref`, falling back to `route|ref` when
     `note_path` is missing. Closed rows from different `done/` notes can share a
     locator. `.taskLink` keys stay as they are.
   - Grouped (empty-filter) view: open sections as today, then one trailing section
     titled "Completed" (kind `.other`, like the dependency picker's "Completed
     history"). It lists rows in Bob's order.
   - Filtered view: like `DependencyPickerIndex`, rank open matches before completed
     matches at any score. Within each tier sort by score, then `bobIndex`.
   - Closed rows show the status glyph (`CapturePickerTaskStatus` already maps `x` →
     done and `-` → canceled), the `locator` route, and `blockID`.
   - Closed rows put `Closed Oct 3` in the slot the scheduled capsule uses, reusing
     `formattedScheduledDate`, or put it in the detail summary instead. Pick whichever
     fits the existing row layout without new view types.
   - The Go To detail action line ("↩ Opens note.md at ^id / line N") works unchanged
     because `taskGotoDestination` comes from `note_path`, `target`, `line`, and `ref`.
   - Decode the new optional `closed` field on `CaptureCompletionCandidate` in
     `CaptureModels.swift`. Missing means nil.
5. Header and hints, `Sources/CaptureCore/CapturePickerPresentation.swift` and
   `CapturePickerView.swift`:
   - The scope capsule reads `Go To` while the toggle is off and `Go To · + Completed`
     while it is on.
   - The Go To footer hints (`keyHintItems`, the `.taskGoto` case) add
     `("⌘X", "Show Completed")` or `("⌘X", "Hide Completed")` to match the state. Change
     `keyHintItems` to take a parameter if needed. Keep the other sources' hints
     byte-identical.
   - Empty states: with the toggle on and no rows, use the existing "no tasks to open"
     wording.
6. Tests:
   - Update `Tests/Fixtures/fake-bob` so `capture-complete … --closed-tasks` returns a
     new fixture `Tests/Fixtures/task-goto-complete-closed.json`: open rows from
     `task-goto-complete-full.json` plus completed rows, including a `done/` row, a
     canceled row, and two closed rows with the same locator in different notes.
   - Router tests: Command-X in Go To returns `.toggleGotoClosedTasks`; in `:`/`^`/`&`
     it returns nil (native cut); a repeat is consumed.
   - `TaskGotoPickerPresentationTests`: the trailing Completed section; open matches
     before completed ones; unique keys for same-locator rows; the closed-date text.
   - `CaptureTaskGotoPanelTests`:
     - Toggle on → refetch with the flag, rows appear, filter and selection kept.
     - Toggle off → completed rows gone.
     - Escape → chip → reopen keeps the state.
     - A new `/` draft starts off (session default).
     - A stale response is dropped.
     - An unsupported-option failure keeps the picker and shows the update hint.
     - Accepting a completed row dispatches `bob-goto-task` with the `done/` absolute
       path.
   - Async tests must never await an untimed continuation (repo AGENTS.md).
7. Docs (this repo owns them; bob-cli mounts them):
   - `docs/task-pickers.md` Go To section: describe Command-X, the Completed section,
     ordering, the capsule, and the update hint. Add Command-X to the "Go To Picker
     keys" line.
   - `docs/keyboard.md` "Capture picker keys": a sentence or row for Command-X in Go To,
     noting that it replaces native cut there.
   - `docs/protocol.md`:
     - Add `capture-complete --closed-tasks` to the command list near the existing
       `capture-complete --all-tasks` lines.
     - Add a row to the older-bob degradation table: "Completed rows in Go To |
       Command-X shows the update hint | Upgrade `bob`".
   - Run `just docs-check`.
8. Gates:
   - Run `just portable-check`, on athena if `swift` is missing locally.
   - On the Mac, best effort: compile a scratch rsync copy (no `.build`/`.git`) with
     `bundle.sh` and `just typecheck-tests`, then delete the copy. Never use Bryan's
     checkout or `just install`. See `docs/develop/build.md` and `AGENTS.md`.
   - If the Mac is offline, say in the final report that the macOS targets are
     unverified.
   - Format only the lines you changed with `.swift-format`.
