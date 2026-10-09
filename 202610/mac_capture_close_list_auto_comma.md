---
tier: tale
title: Auto-insert commas between close task numbers in Bob Mac Capture
goal:
  Typing 1-9 right after a task number in an =x/=*/=! close list inserts the comma
  itself when the running Pomodoro has fewer than 10 numbered Task Links.
size: medium
proposed_by: bbugyi200.apollo.61.w1
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.61.w1](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.61.w1.md)
- **COMMITS:**
  - [5601235](https://github.com/bobs-org/bob-cli/commit/56012352f4cdec6e00e4c163b6c953c69ae4f62b)
    — feat(capture): report task_link_count in capture-pomodoros JSON output

# Auto-insert commas between close task numbers in Bob Mac Capture

## Goal

In the Bob Mac Capture editor, typing a task number right after another task number in a
Pomodoro close selection inserts the separating `,` itself. Today the user types
`=x1,2!3,4`. After this change they type `=x12!34` and the editor shows `=x1,2!3,4`.

Rules (from the request):

- Trigger: the user presses a digit key `1`–`9` while the caret sits right after a task
  number that is already in a close-selection group. The groups are the `=x` keep list
  (`<N>`), `*<P>` park, `!<M>` complete, and `~<K>` drop. This covers the `=*…` and
  `=!…` aliases, any mix of groups such as `=x2!1,4*3`, and the `@route:id=x…`,
  `^route:id=x…`, and `<text> @route:id=x…` link and new-task forms, because they all
  use the same list grammar. The editor then inserts `,` followed by the digit instead
  of the digit alone.
- Gate: this happens only when the running Pomodoro has **fewer than 10** numbered Task
  Links. With 10 or more, a multi-digit index like `12` is legitimate, so digits insert
  normally.
- No trigger: the first number of a group (`=x` + `1`, `=x2!` + `1`), the `0` key, the
  inline Work Log number (`=x 1 …`), Work Log bullet numbers (`- 1 …`), start-drop lists
  (`=~2`, `=x =~2`), and every non-close context. In all of these the digit is typed
  exactly as today.

## Architecture: stays a thin client

The accepted decision `mac-capture-is-a-thin-client` applies here. bob-cli is the only
implementation of capture grammar and vault facts, and the app "may filter or rank what
bob returned, but it never decides syntax itself". This feature needs two facts. Both
must come from bob:

1. **Where the close-selection lists are.** bob already reports them.
   `bob capture-parse --format json` emits one span per group, with byte offsets in
   UTF-8: `pomodoro_close_in_progress` (the bare `<N>` list), and `pomodoro_close_park`
   / `pomodoro_close_complete` / `pomodoro_close_drop` (each including its `*` / `!` /
   `~` sigil). These spans also appear on link and new-task close forms and inside
   chains and batches. Verified shapes:
   - `=x1` → `pomodoro_close` [0,2), `pomodoro_close_in_progress` [2,3)
   - `=*1` → `pomodoro_close` [0,1), `pomodoro_close_park` [1,3)
   - `=x2!1,4*3~5` → in_progress [2,3), complete [3,7), park [7,9), drop [9,11)
   - `^bob:foo=x1` → in_progress [10,11)
   - `hello\n\n=x3` → in_progress [9,10)
   - `=x1,` → in_progress [2,3) plus `interactive_placeholder` [3,4) (mode `incomplete`)
   - `=x 1` → no list span (the inline Work Log number)
   - `=x1\n- 1 did` → `pomodoro_close_log_index` for the bullet number (not a list kind)
   - a lexically malformed list such as `=x1,1` reports only the `pomodoro_close` span
     plus a diagnostic. With no list span the assist does nothing, so typing stays
     plain. That is acceptable.

   No bob-cli change is needed for this part. The app only filters bob's spans: a list
   span contains the caret, and the byte before the caret is `1`–`9`.

2. **How many numbered Task Links the running Pomodoro has.** This is a vault fact that
   today only appears inside a close dry run (`pomodoro_close.task_links`). A dry run is
   tied to the draft and arrives after the 50 ms analysis debounce, so it is often
   missing exactly when a fast typist hits the second digit. bob-cli therefore gains one
   **additive** JSON field on the existing read-only `bob capture-pomodoros` command.
   The app prefetches it when the panel shows and refreshes it on vault changes. The
   field matches the request's wording ("task links in the current pomodoro") and does
   not depend on the draft.

Why the count of the running session is enough even for link forms: a link close indexes
the lineup after the link step, which can be one larger than the running count. At the
boundary (9 running, so 10 after the link) the only two-digit index is `10`. It is typed
`1` then `0`, and `0` never triggers the assist, so nothing is corrupted. With 10 or
more running links the assist is off anyway. The only known miss is a batch that links
two or more new tasks into a 9-link session before its `=x`. Accept that and note it in
the docs.

Why decide on the keystroke instead of rewriting afterwards: `bob capture-rewrite` is
documented as purely lexical and vault-free, and it never rewrites close items. An async
rewrite would also flash `=x12` before fixing it. Deciding synchronously on the key
event, from bob-supplied data, gives instant and deterministic behavior. When the app's
data is stale or missing, the digit types normally, so the failure mode is just today's
behavior.

## Part 1 — bob-cli: `task_link_count` on `bob capture-pomodoros`

Files: `src/native/capture_pomodoros.rs`, possibly `src/native/capture_pomodoro_close/`
(visibility or a small helper), `docs/capture.md`, tests.

1. In `list_capture_pomodoros` (the `capture-pomodoros` command path only), compute the
   size of the numbered Task Link lineup for the `is_current` entry. Use the same
   numbering a plain `=x` close uses:
   `capture_pomodoro_close::selection::number_task_links(contents, &RunningPomodoro)`.
   That function reads only the running headline's line, so either build a
   `RunningPomodoro` from the entry (`line`, `name`, `time_range`) or extract a
   line-number variant. Make it reachable from `capture_pomodoros.rs` (re-export or
   `pub(crate)` path) without duplicating the classification logic.
2. Serialize it as `task_link_count` on **every** JSON entry: an integer on the
   `is_current` entry and explicit `null` on every other entry. That includes completed
   entries under `--all`, and every entry when there is no current session (none
   running, or multiple open timed Pomodoros). This keeps the shape stable for decoders.
   Do **not** add the field to the shared `PomodoroEntry` / `scan()`. Those are reused
   by `capture-pomodoro-name`, `capture-complete`, task toggles, and plan code, whose
   JSON and performance must not change. Instead wrap entries in a
   `capture-pomodoros`-only output struct, for example
   `#[serde(flatten)] entry: PomodoroEntry` plus `task_link_count: Option<usize>`. Keep
   `schema_version` 1 because the change is additive. Human output is unchanged.
3. Extend the `long_about` help sentence ("…current-session status, and child link
   counts for picker callers") to mention the current entry's numbered Task Link count.
   Keep it one clear clause, and update any help fixture that captures it.
4. Tests:
   - Unit tests in `capture_pomodoros.rs`. Build a fixture day with a running timed
     Pomodoro whose children include a plain link, a deferred link, an embedded link, a
     struck link, a prose note, and a nested detail. `task_link_count` must equal the
     numbered lineup, so the struck link and the note are excluded. Also cover: a
     placeholder or completed entry gets `null`; no running session gives all `null`;
     multiple open timed entries give all `null`; a session with 10 or more links
     reports the full count (≥ 10).
   - A CLI parity test, next to
     `capture_pomodoros_json_lists_open_entries_with_stable_picker_shape` in
     `tests/cli/capture/pomodoro_name.rs` or a new `tests/cli/capture/pomodoros.rs`
     registered in `tests/cli/capture/mod.rs`. On the same fixture (`BOB_DAY_FILE` /
     `BOB_NOW` as existing close tests do), assert that `bob capture-pomodoros -f json`
     current `task_link_count` equals the length of
     `bob capture --dry-run --no-clip -f json -- '=x'` → `pomodoro_close.task_links`.
   - Assert that `capture-pomodoro-name` / `capture-complete` JSON is unchanged.
     Existing tests should already cover this; add one assertion if not.
5. Docs: in `docs/capture.md` § "Discovery commands", in the `capture-pomodoros` JSON
   paragraph, add `task_link_count`. Describe it as the size of the numbered Task Link
   lineup that a plain `=x` close indexes, the same count as
   `pomodoro_close.task_links`, present as an integer only on the `is_current` entry and
   `null` elsewhere. Say editors use it to know whether close task numbers can be
   multi-digit.
6. Run `just check` (fmt, clippy, all tests).

## Part 2 — Bob Mac Capture

Open the repo with `/sase_repo` (`sase repo open bob-mac-capture`). If the linked
checkout is unavailable, use `sase repo open gh:bobs-org/bob-mac-capture`. Use the
printed path. Linux agent hosts have no Swift toolchain, so CI is the compiler; see
Verification.

### 2a. Decode the count (CaptureCore)

- `Sources/CaptureCore/CaptureModels.swift`: add `CapturePomodorosResponse` (`ok`,
  `schemaVersion`, `pomodoros`) and a minimal entry model (`line`, `name`, `isCurrent`,
  `taskLinkCount: Int?`). Decode tolerantly with `decodeIfPresent`, so an older bob
  without `task_link_count` decodes to `nil`. Conform to `SchemaVersioned` (see the
  extension list at the bottom of `BobProcessClient.swift`). Add a computed
  `currentTaskLinkCount: Int?`: the `isCurrent` entry's `taskLinkCount`, or `nil` when
  there is no current entry or the field is absent.
- `Sources/CaptureCore/BobProcessClient.swift`: add
  `capturePomodoros() async throws -> CapturePomodorosResponse` running
  `["capture-pomodoros", "--format", "json"]`, `expectedSchema: 1`, lane `"pomodoros"`.
  Give `captureParse` an optional `lane:` parameter (default `"parse"`, so existing
  callers are unchanged) so the assist can parse on its own lane without cancelling the
  debounced analysis parse.
- Fixtures under `Tests/Fixtures/`: `pomodoros-current-3.json`,
  `pomodoros-current-12.json`, `pomodoros-none-running.json`, and
  `pomodoros-legacy-no-count.json` (no `task_link_count` key). Add decoding tests in
  `Tests/CaptureCoreTests/` (a new file or `CaptureModelTests.swift`).

### 2b. Pure decision helper (CaptureCore, no AppKit)

New `Sources/CaptureCore/CaptureCloseTaskCommaAssist.swift`:

- `public struct CaptureParseSnapshot: Equatable, Sendable { draft: String; spans: [CaptureSpan] }`.
- `public enum CaptureCloseTaskCommaAssist` with:
  - `static let listSpanKinds: Set<String> = ["pomodoro_close_in_progress", "pomodoro_close_park", "pomodoro_close_complete", "pomodoro_close_drop"]`
  - `static let singleDigitLineupLimit = 10` (the assist runs only when `count < 10`)
  - `static func isArmed(taskLinkCount: Int?) -> Bool`: true only for a known count
    below the limit.
  - `static func edit(typed: String, text: String, selectedRange: NSRange, snapshot: CaptureParseSnapshot?, taskLinkCount: Int?) -> CaptureCloseTaskCommaEdit?`
    returns `replacementRange` (the collapsed caret), `replacementText` (`","` plus the
    digit), and `resultingSelection` (caret after the digit). The shape matches
    `CaptureSnippetEdit`. It returns `nil` unless **all** of these hold:
    1. `typed` is exactly one character in `1`…`9`;
    2. `isArmed(taskLinkCount:)`;
    3. `selectedRange.length == 0` and the location is valid in `text` (UTF-16);
    4. `snapshot?.draft == text`, meaning the spans describe exactly this text;
    5. converting the caret to a UTF-8 offset `c` (use the helpers in
       `CaptureTextRanges.swift` or `String.Index(utf16Offset:in:)`), the byte at
       `c - 1` is ASCII `1`–`9`;
    6. some span in `snapshot.spans` has `kind ∈ listSpanKinds` and
       `span.start < c && c <= span.end`. This covers the end of a list and a caret
       placed right after a number in the middle of a list.
  - Doc comment: the helper never recognizes close syntax itself. It only filters bob's
    spans, and Bob stays authoritative.
- Tests in `Tests/CaptureCoreTests/CaptureCloseTaskCommaAssistTests.swift`, with spans
  copied from real `bob capture-parse` output (the shapes listed above):
  - `=x1` + `2` → `,2`; `=*1` + `2`; `=!1` + `2`; `=x2!1` + `4` → `,4` inside the
    complete span; `=x2!1,4*3` + `5` at the end of the park span; `=x2!1,4*3~5` + `6` in
    the drop span; `^bob:foo=x1` + `2`; `Draft docs @bob:draft-docs=x1` + `2`;
    `hello\n\n=x3` + `4` (multi-item offsets); caret after `2` in `=x2!1` (end of the
    in-progress span, before `!`) + `3`.
  - Declines: count `nil`, count `10`, count `12`; typed `0`; typed a non-digit; the
    first digit of a group (`=x`, `=x2!`, `=*`) where the byte before the caret is a
    sigil or `x`; `=x1,` (byte before the caret is `,`); `=x0` + `2` (byte before is
    `0`); inline `=x 1` + `2` and bullet `- 1` + `2` (no list span); `=x1,1` with no
    list span; a stale snapshot (`snapshot.draft != text`); a non-collapsed selection; a
    multibyte prefix such as `é =x1` to prove the UTF-16/UTF-8 conversion.

### 2c. Model state (`Sources/BobMacCapture/CapturePanelModel.swift`)

- `private(set) var currentPomodoroTaskLinkCount: Int?` and
  `var closeTaskCommaArmed: Bool { CaptureCloseTaskCommaAssist.isArmed(taskLinkCount: currentPomodoroTaskLinkCount) }`.
- `func refreshCurrentPomodoroTaskLinkCount()`: run `processClient.capturePomodoros()`
  in a `Task`. On success set the count to `response.currentTaskLinkCount`. On any
  failure set it to `nil`, which is safe because the assist turns off. Do not surface
  errors in the status or error UI; this is a silent typing aid. Ignore it when
  `processClient` is nil, and clear the count in `setProcessClient(nil)`.
- `private var closeListParseSnapshot: CaptureParseSnapshot?`, updated only when the
  parsed draft still equals `plainDraft`:
  - in `applyParse(_:draft:)`, the debounced analysis parse;
  - by a new immediate assist parse. In `editorTextDidChange`, at the point where normal
    edits reach `scheduleAnalysis`, when `closeTaskCommaArmed` is true, the caret is
    collapsed, and the byte before the caret is ASCII `1`–`9`, start
    `captureParse(draft, lane: "close-task-comma")` right away with no debounce. Store
    the snapshot if the draft is still current. Swallow errors. This keeps the snapshot
    fresh within one subprocess spawn of the last digit, well under human inter-key
    time, so `=x` + `1` + `2` typed quickly still gets its comma.
  - Clear the snapshot in `resetAnalysisState()` and wherever the draft is replaced
    wholesale.
- `func closeTaskCommaEdit(typed: String, text: String, selectedRange: NSRange) -> CaptureCloseTaskCommaEdit?`
  forwards to the pure helper with the snapshot and count.

### 2d. Wiring: refresh triggers, routing, and insertion

- `Sources/BobMacCapture/AppDelegate.swift`: call
  `panelModel?.refreshCurrentPomodoroTaskLinkCount()` in `showCapturePanel()`, next to
  `refreshTargetsWhenPossible()`. Also call it from the vault watcher's `onChange`
  callback when the capture panel is visible, so a capture or an Obsidian edit that
  changes the lineup is picked up. Keep the hotkey path free of synchronous subprocess
  work.
- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`:
  - New `CaptureKeyCommand.insertCloseTaskNumber(String)` and
    `CaptureKeyRoutingContext.closeTaskCommaArmed` (default `false`).
  - In the main editor branch only, which comes after the stash, prompt, and picker
    branches that already own digits, return `.insertCloseTaskNumber(characters)` when
    `context.closeTaskCommaArmed`, the modifiers contain none of
    `[.command, .control, .option]`, and `event.characters` is exactly one character in
    `1`…`9`. Check `characters`, not key codes, and do not reject Shift. This accepts
    the numeric keypad and layouts such as AZERTY that type digits with Shift. Leave
    every existing key path unchanged.
- `Sources/BobMacCapture/CapturePanelController.swift`:
  - Pass `closeTaskCommaArmed: self.model.closeTaskCommaArmed` into the routing context
    in `installKeyMonitorIfNeeded()`.
  - Add
    `static func insertCloseTaskNumberInEditableTextView(_ digit: String, firstResponder: NSResponder?, model: CapturePanelModel) -> Bool`,
    modeled on `applySnippetExpansion`. It needs `editableTextView(firstResponder)` and
    `!textView.hasMarkedText()`, and it asks
    `model.closeTaskCommaEdit(typed:text:selectedRange:)` using `textView.string` and
    `textView.selectedRange()`. On a decline it returns `false`, so the key event falls
    through to AppKit and the digit types natively. Otherwise it calls
    `textView.insertText(edit.replacementText, replacementRange: edit.replacementRange)`
    and `setSelectedRange(edit.resultingSelection)`, then returns `true`. Going through
    `NSTextView` keeps undo, IME, and accessibility native, and the normal
    `editorTextDidChange` path then re-arms the assist parse.
  - Handle `.insertCloseTaskNumber(let digit)` in `perform(_:)`.
- Tests in `Tests/BobMacCaptureTests/`, using the existing `NSEvent.keyEvent` and
  static-helper patterns (for example `CaptureTaskCompletePanelTests.keyEvent`, and the
  `insertBulletNewlineInEditableTextView` tests in `BobMacCaptureTests.swift`):
  - Router: a digit with `closeTaskCommaArmed` maps to `.insertCloseTaskNumber`; with
    the flag off, or with Command, Control, or Option held, it returns `nil`; `0`
    returns `nil`; with the picker, stash, or a prompt visible their own digit handling
    still wins.
  - Controller helper against a real `NSTextView`, with a model seeded through
    test-visible setters or a fake-bob fixture: `=x1` + `2` becomes `=x1,2` with the
    caret at the end; a decline leaves the text untouched and returns `false`; marked
    text declines.
  - Model: the snapshot is stored only for the current draft, a stale snapshot never
    arms an edit, and a `capture-pomodoros` failure or a legacy response yields
    `closeTaskCommaArmed == false`.

### 2e. Docs (bob-mac-capture `README.md`)

- Keyboard table: add a `1`–`9` row. In the editor: right after a task number in a close
  task list (`=x`, `=*`, `=!`, and their `*`/`!`/`~` groups, including link closes),
  insert `,` before the digit (`=x1` + `2` → `=x1,2`) when the running Pomodoro has
  fewer than 10 numbered Task Links; otherwise the digit types normally. The other
  columns are native and unchanged.
- Runtime Contract / close section: the app runs `bob capture-pomodoros --format json`
  when the panel shows and on vault changes while it is visible, and reads the current
  entry's `task_link_count`. An older bob without the field disables the assist. The
  decision uses bob's `capture-parse` close-list spans, so the app still never
  recognizes close syntax itself. Note the accepted limitation: a batch that links
  several new tasks before its `=x` can push a 9-link session past 10.

## Out of scope

- Start-drop lists (`=~2,3`, `=x =~2`). They index the next session's lineup, which
  `task_link_count` does not describe.
- Work Log numbers (inline `=x 1 …` and bullets `- 1 …`) and the `0` key.
- Paste, and changing what `bob capture` accepts. Commas stay required by the grammar,
  and the assist only types them for the user.
- Any change to `capture-rewrite`, `capture-parse`, or the SASE memory/decision records.
  The feature follows `mac-capture-is-a-thin-client` as written.

## Verification

- bob-cli: `just check` passes; the parity test shows the `capture-pomodoros`
  `task_link_count` equals the `=x` dry run's `pomodoro_close.task_links` length.
- bob-mac-capture: this repo cannot be built on Linux agent hosts. Read the new Swift
  carefully against the surrounding code (Swift 6 concurrency annotations, `nonisolated`
  static helpers, `swift-format` line width). After the change is committed and pushed,
  watch the `macOS 26 SwiftPM` GitHub Actions run (format lint, build, test, bundle,
  launch smoke). Use `/sase_monitor` for the wait, and fix any failure in a follow-up
  commit until it is green.
- Manual check on the Mac, for Bryan: with a running Pomodoro of 3 links, typing
  `=x12!3` shows `=x1,2!3`; `=x2!14*3` shows `=x2!1,4*3`; `=x 12 fixed` stays as typed;
  with 10 or more links `=x12` stays `=x12`.
