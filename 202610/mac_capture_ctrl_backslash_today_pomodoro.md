---
tier: tale
title: Bob Mac Capture Control-\ opens today's note on the current Pomodoro
goal:
  In the capture panel, Control-\ opens today's daily note in Obsidian with the caret on
  the running Pomodoro line, keeping any draft.
size: medium
decisions:
  no_current:
    ask:
      When no Pomodoro is running, should Control-\ still land on the nearest Pomodoro
      line instead of the note's top?
    choices:
      nearest:
        Land on the first open Pomodoro, else the last completed one, else the note's
        first line
      top: Open today's note at its first line whenever nothing is running
      refuse: Keep the panel open with "No Pomodoro is running" and open nothing
    default: nearest
    why: Between sessions, the next planned Pomodoro is where work resumes
    answer: nearest
proposed_by: bbugyi200.apollo.8p
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.8p](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.8p.md)
- **COMMITS:**
  - [ff96c07](https://github.com/bobs-org/bob-plugins/commit/ff96c07db7ec3e3e37113a9feaa8fb463fa9f63f)
    — docs: name Control-backslash as a bob-goto-task caller

# Bob Mac Capture: Control-\ opens today's note on the current Pomodoro

## Goal

In the Bob Mac Capture panel, pressing **Control-\\** opens today's daily note in
Obsidian with the caret on the running Pomodoro's entry line, then hides the panel
(keeping any draft, as Escape does). Nothing in the vault is written.

## Scope and non-goals

- **In-panel key only.** It goes into `CaptureKeyCommandRouter` next to Control-S and
  Control-T. It is **not** a global Carbon hotkey: a global Control-\\ would take the
  key from every terminal (SIGQUIT) and from Vim's `Ctrl-\ Ctrl-N`. Bob Refs is
  unchanged.
- **No bob-cli or plugin code changes.** Everything needed already ships:
  - `bob capture-pomodoros --format json [--all]` (present since `cc4c9a38`, 2026-08-28)
    reports the top-level absolute `day_file` and, per entry, `ref` (`LINE:digest8`,
    made by the same `note_tasks::line_digest` as task refs), the 1-based `line`,
    `state` (`open` / `completed`), `name`, and `is_current` (bob marks the single open
    timed entry as current). See `src/native/capture_pomodoros.rs` in bob-cli
    (`CapturePomodorosResult`, `PomodoroEntry`, `scan`).
  - bob-navigation-hotkeys' read-only `obsidian://bob-goto-task?path=&line=&ref=`
    handler (bob-plugins
    `plugins/bob-navigation-hotkeys/src/675-plugin-goto-task-uri.js`) accepts any line.
    It lands on the planned line when the digest matches, falls back to a unique digest
    match among checkbox lines (Pomodoro entries are column-0 checkboxes), and otherwise
    clamps. `ref` is optional, and with no `ref` it clamps silently.
    `focusTaskAnchor(..., { ensureEditing: true })` switches Reading view to editing,
    puts the cursor on the line, focuses the editor, and records a jump. The Go To
    picker already requires bob-navigation-hotkeys ≥ 2.32.0, so Control-\\ adds no new
    requirement.
- Follows decision `mac-capture-is-a-thin-client`: the app only runs `bob`, chooses
  among the entries bob returned ("filter or rank what bob returned"), and dispatches a
  URL. It never reads the daily note.
- No footer button or hint; the footer is already full.

## Design

### Data source: a fresh read-only fetch on each press

Each press runs `bob capture-pomodoros --format json --all` through the existing
`BobProcessClient` "pomodoros" lane. It does not use the cached idle-agenda snapshot,
because:

- the cache can lag a Pomodoro started or closed in Obsidian moments ago (watcher
  latency), and "current" is the one fact this key exists to get right;
- the agenda cache is not loaded when bob predates `--tasks`, and it is skipped entirely
  when the agenda setting is off.

The plain call (no `--tasks`) only scans today's daily note, so it is cheap. `--all`
includes completed entries for the fallback.

### Landing rule

All logic lives in one pure CaptureCore function (unit-tested on Linux) that reads the
decoded response:

1. `day_file` missing, empty, or not absolute: no destination. Status
   `bob sent no daily-note path — update bob`; the panel stays open.
2. An entry with `is_current == true`: land on it.
3. No current entry:

> [!decision] no_current = nearest Land on the first entry (document order) whose
> `state` is `open`; else the last entry whose `state` is `completed`; else the note's
> first line (no `ref`). This also covers bob's "multiple open timed Pomodoros" warning
> case: no entry is current, so the first open one wins.

> [!decision] no_current = top Land on line 1 of `day_file` with no `ref`. Completed and
> open entries are ignored.

> [!decision] no_current = refuse No destination. Status `No Pomodoro is running`; the
> panel stays open with nothing dispatched. An empty Pomodoros list (including a missing
> daily note) also refuses with the same status.

A chosen entry builds
`ObsidianOpenURL.taskGotoURL(path: dayFile, line: entry.line, ref: entry.ref)`. If the
entry's `ref` is missing or invalid (older or odd bob), the URL is built from the line
alone, with no `ref`. A note-top landing builds `path` plus `line=1` with no `ref`. When
the daily note does not exist, bob still reports `day_file` with an empty list. The URL
then goes out, and the plugin shows its own `Bob Go To: … not found` notice. That is
acceptable; the app does not stat vault files.

### Panel behavior

- **Router** (`Sources/BobMacCapture/CaptureKeyCommandRouter.swift`): add
  `KeyCode.backslash = 42` (`kVK_ANSI_Backslash`; the router already matches physical
  key codes, e.g. `leftBracket = 33`) and a new `CaptureKeyCommand.openTodayPomodoro`.
  - Main editor branch (with or without completion visible, with or without the picker
    reopen chip): exactly `modifiers == .control` returns `.openTodayPomodoro`. Place it
    beside the Control-S and Control-T checks, before the `switch`.
  - Modal surfaces consume exact Control-\\ (`.consumeKey`), matching the "finish or
    cancel first" rule Control-S and Control-T follow there. Covered: the stash picker
    and template shelf (their `printableModalCommand` fallback returns nil for the
    control character, so this needs an explicit case), `isolatedPromptCommand` (Add
    block ID and Name Pomodoro), `pullInChoiceCommand`, and `pickerCommand`. A picker
    dispatch in flight already consumes every key.
  - Plain `\`, Shift-Control-\\, Command-\\, and Option-Control-\\ keep their current
    routing (nil / native).
- **Controller** (`CapturePanelController.perform`): `.openTodayPomodoro` calls
  `model.openTodayPomodoroInObsidian()` and returns `true`. The switch is exhaustive, so
  this case is required.
- **Model** (`CapturePanelModel`): new `openTodayPomodoroInObsidian()`:
  - `isSubmitting`: status `Wait for the capture to finish`; nothing else.
  - A press while one is already in flight does nothing (`todayPomodoroGotoInFlight`,
    `private(set)` so tests can read it). The flag clears on completion, in
    `prepareForDismissal()`, and wherever `taskGotoDispatchInFlight` is reset in
    `resetAnalysisState()`.
  - `processClient == nil`: status `Bob is not resolved` (existing wording).
  - Otherwise set the flag, set status `Opening today's note…`, and run
    `processClient.capturePomodoros(includeCompleted: true)` in a `Task`. Back on the
    main actor:
    - If the panel was hidden in the meantime (`panelVisible == false`), or a request
      generation counter (bumped on each dispatch and on dismissal) no longer matches,
      drop the result silently.
    - bob failure (any thrown error): status `Couldn't read today's Pomodoros`; the
      panel stays open.
    - No destination: the landing rule's status; the panel stays open.
    - Otherwise dispatch through the existing injectable `taskGotoOpener` (`NSWorkspace`
      with `activates = true`; tests stub it). On success: `prepareForRetainedClose()`
      then `panelDismisser()`, so a nonempty draft is retained exactly like Escape. It
      is never stashed, discarded, or captured. On failure: status
      `Couldn't open Obsidian`; the panel and draft stay.
  - Use `announceStatus` for every status, as `acceptTaskGotoRow` does.

### CaptureCore changes

- `CaptureModels.swift`:
  - `CapturePomodorosResponse` gains `dayFile: String?` (`day_file`).
  - `CapturePomodorosListEntry` gains `ref: String?` and a state that decodes `open` and
    `completed` and tolerates unknown values (a small enum with `.other`, or an optional
    raw string; follow the file's existing decode-with-defaults style).
  - All of these use `decodeIfPresent` with defaults so legacy payloads still decode.
    Keep the existing public initializers source-compatible by giving the new parameters
    default values.
- `BobProcessClient.capturePomodoros(includeCompleted: Bool = false)`: appends `--all`
  when true. Existing callers (agenda store's plain-count lane) stay unchanged.
- `ObsidianOpenURL.taskGotoURL(path:line:ref:blockID:)`: make `ref` a `String?`. A nil
  ref omits the `ref=` query item. A non-nil invalid ref still returns nil, so every
  existing assertion in `TaskGotoPickerPresentationTests` keeps passing. The Go To
  caller passes a non-optional `String` and keeps compiling. Update the file's header
  comment to name both callers.
- New `Sources/CaptureCore/CaptureTodayPomodoroGoto.swift`: the pure landing rule.
  Suggested shape: a `resolve(_ response: CapturePomodorosResponse)` that returns either
  a destination (`url`, plus a landing kind of current / first open / last completed /
  note top, for tests and status) or a refusal reason that maps to the status strings
  above. Names are the implementer's call; keep the status strings as written.

## Tests

CaptureCore (portable, Linux):

- `CaptureTodayPomodoroGotoTests` (new) covers:
  - current entry: the URL carries `day_file`, its line, and its ref;
  - the decision's no-current branches: none running with open entries; only completed
    entries; an empty list;
  - missing or relative `day_file`: no destination;
  - an entry with a missing or invalid `ref`: line-only URL;
  - percent-encoding of a `day_file` containing a space.

  Decode inline JSON or new fixtures.

- `CapturePomodorosTests` / decoding: existing fixtures (`pomodoros-current-3.json`,
  `pomodoros-none-running.json`) now expose `dayFile`, `ref`, and `state`;
  `pomodoros-legacy-no-count.json`-style payloads without them still decode, with nil
  defaults.
- `TaskGotoPickerPresentationTests`: a nil `ref` builds a URL with no `ref=` item; an
  invalid non-nil ref still returns nil.
- `BobProcessClientTests`: `capturePomodoros(includeCompleted: true)` records
  `argv=capture-pomodoros --format json --all` via `FAKE_BOB_RECORD_PATH`, and the
  default call records no `--all`.
- New fixtures in `Tests/Fixtures/` as needed, in the plain `capture-pomodoros` shape
  (for example `pomodoros-all-completed.json` and `pomodoros-no-daily-note.json`).
  `fake-bob`'s `capture-pomodoros` branch already ignores `--all` and serves
  `FAKE_BOB_POMODOROS_FIXTURE`, so no fake-bob change is needed unless a test wants one.

BobMacCaptureTests (macOS):

- Router tests in `BobMacCaptureTests.swift`:
  - Control-\\ (keyCode 42, `.control`) maps to `.openTodayPomodoro` in the editor, with
    completion visible, and with the picker chip visible;
  - it maps to `.consumeKey` with the stash picker, template shelf, Add block ID prompt,
    Name Pomodoro prompt, pull-in card, or picker open;
  - plain `\`, Command-\\, and Shift-Control-\\ do not route to it.
- Model tests in a new `CaptureTodayPomodoroPanelTests.swift`. Use the
  `CaptureTaskGotoPanelTests` pattern (stub `taskGotoOpener` and `panelDismisser`) plus
  a fake-bob `BobProcessClient` with `FAKE_BOB_POMODOROS_FIXTURE`, awaiting the async
  hop with an expectation. Cover:
  - current fixture: exactly one expected URL, dismissed once, nonempty draft retained
    (not stashed), flag cleared;
  - opener failure: status `Couldn't open Obsidian`, no dismiss;
  - `FAKE_BOB_POMODOROS_FAIL=1`: status `Couldn't read today's Pomodoros`, no open;
  - a double press opens once;
  - nil client: status `Bob is not resolved`;
  - `isSubmitting`: no open;
  - the panel hidden before the result lands: no open;
  - the chosen `no_current` branch against the none-running and all-completed fixtures.

## Docs

- bob-mac-capture `README.md`:
  - **Keyboard** table: add a `Control-\` row. In the editor: open today's daily note in
    Obsidian with the caret on the running Pomodoro (state the chosen no-current rule),
    hiding the panel and retaining a nonempty draft. While completion is visible: same,
    and close completion. Add block ID / Name Pomodoro columns:
    `Consume the key; finish or cancel the prompt first`.
  - New `### Opening today's Pomodoro (Control-\)` subsection under Keyboard. It covers:
    - the fresh `capture-pomodoros --all` read;
    - the landing rule;
    - the `obsidian://bob-goto-task` dispatch with the entry's `ref`;
    - that nothing is written;
    - draft retention;
    - the failure statuses;
    - the double-press and hidden-panel drops;
    - that the key is consumed in the stash, shelf, prompts, pull-in card, and pickers;
    - that it is in-panel only, not a global hotkey.
  - The Requirements sentence that names bob-navigation-hotkeys ≥ 2.32.0 for Go To:
    extend it to Control-\\.
- bob-plugins `README.md`, in "Go to task URI (`obsidian://bob-goto-task`)": change "Bob
  Mac Capture's `/` Go To picker dispatches this URI" to also name the Control-\\
  today-Pomodoro landing. State that `line`/`ref` may name any line, including a
  Pomodoro entry, which is not a `#task` line. This protects the contract from being
  narrowed to `#task` rows later. This is a doc-only change: per bob-plugins
  `AGENTS.md`, still run `npm run build:check` and `bob plugins sync` afterwards.
- No bob-cli changes and no SASE memory changes. No memory note becomes false: the
  thin-client and idle-agenda decisions still hold, because the app never reads the
  vault.

## Repos and verification

Open both linked repos with `/sase_repo` (`bob-mac-capture`, `bob-plugins`) and read
each repo's `AGENTS.md` first. Commit each touched repo separately.

bob-mac-capture's `AGENTS.md` requires compiled Swift. Never skip compiling.

1. `just portable-check` (swift-format lint, then build and test CaptureCore and
   RefsCore). The planning host has no `swift`, so run it on a tailnet host with Swift
   (athena), as `AGENTS.md` says.
2. `Sources/BobMacCapture*` and `Tests/BobMacCapture*` change, so also compile a scratch
   copy on the Mac (best effort):
   - rsync without `.build` and `.git`;
   - run `./Scripts/bundle.sh` and `just typecheck-tests`;
   - delete the copy.

   Never use Bryan's checkout or `just install` there. If the Mac is offline, say in the
   final report that the macOS targets are unverified.

3. If CI shows a red **Lint Swift formatting** step, check the later steps too: a lint
   parse error skips Build, Test, Bundle, and the install gates.

Manual acceptance (Bryan, after install):

- Mid-session, open the panel and press Control-\\: Obsidian comes to the front on
  today's note, with the caret on the running `- [ ] (**HHMM-HHMM** …)` line.
- Between sessions: the press lands per the chosen rule.
- A typed draft is still there on the next panel show.
