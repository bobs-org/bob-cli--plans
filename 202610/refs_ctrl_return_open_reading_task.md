---
tier: tale
title: Bob Refs Control-Return opens the reading task
goal:
  In Bob Refs, Control-Return opens the selected reference's reading-task file in
  Obsidian, mirroring Command-Return for the note.
size: small
decisions:
  landing:
    ask:
      Should ⌃↵ put the caret on the reading-task line, not just open the file at the
      top?
    choices:
      task_line:
        Lands the caret on the task's ^block line via bob-goto-task
        (bob-navigation-hotkeys ≥ 2.32.0)
      file_top:
        Opens the file via obsidian://open like ⌘↵ does; you find the task in the note
        yourself
    default: task_line
    why:
      reading tasks live in long area notes, so landing on the line saves a search every
      time
    answer: task_line
  footer_hint:
    ask: Should the Bob Refs footer advertise `⌃↵ Task` in its widest hint variants?
    choices:
      menu_only: The footer stays as it is; ⌃↵ appears in the ⌘K menu and the docs only
      widest:
        Adds `⌃↵ Task` to the widest footer variants only; mid-width panels drop ⌘1–5
        Scope sooner
    default: menu_only
    why: the footer is width-tuned to 760 pt and the ⌘K menu already shows the shortcut
    answer: menu_only
proposed_by: bbugyi200.apollo.a5
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.a5](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.a5.md)
- **COMMITS:**
  - [2750827](https://github.com/bobs-org/bob-mac-capture/commit/27508279b4b66b4b82a55dfa32d7a61acc8a1e25)
    — feat(refs): Control-Return opens the reading task

# Plan: Bob Refs Control-Return opens the reading task's file in Obsidian

## Goal

In the Bob Refs panel (bob-mac-capture), a new **Control-Return** (`⌃↵`, including
keypad Enter) opens the file that holds the selected reference's **reading task** in
Obsidian. It works whether the selected row came from a ref note or a PDF, because every
Refs row is a reference and carries the reading-task location `bob` reported for it.

It mirrors the existing **Command-Return** (`⌘↵`), which opens the reference note
itself.

All work is in the **bob-mac-capture** linked repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<reason>"`), and read its `AGENTS.md` before
editing. bob-cli and bob-plugins need no changes. The mounted copies of the Mac docs
under bob-cli's `docs/reference/mac-capture/` are gitignored and regenerate from
bob-mac-capture.

## How Command-Return works today (the model to copy)

- `Sources/BobMacCapture/Refs/RefsKeyRouter.swift`: in the
  `KeyCode.return, KeyCode.keypadEnter` arm, while not IME-composing,
  `flags == .command` → `.open(.note)`, `flags == .option` → `.open(.reveal)`, and no
  modifiers → `.open(.highlights)`. Control-Return currently returns `nil` and passes
  through to the search field.
- `Sources/BobMacCapture/Refs/RefsPanelModel.swift`:
  - `RefsOpenTarget` is `highlights | note | reveal | defaultApp`.
  - `perform(.open(target))` calls `openSelected(target)`, which guards in this order:
    no selection (returns `false`), vanished id (warning banner), not in the listing
    (`false`), and a synthetic Tabs row (`showNoteActionWarning` for every
    non-Highlights target). Then
    `case .note: opener.openNote(library.noteURL(for: item)); return true`.
  - `RefsOpening` is the opener protocol. `WorkspaceRefsOpener.openNote(_:)` opens
    `ObsidianOpenURL.url(forAbsolutePath:)`, which is `obsidian://open?path=…`.
  - Note opens never count as opens (no `signals.opens` record) and never hide the panel
    themselves.
  - `openStored(_:)` routes `.note, .reveal` back through `openSelected`.
  - `performAction(.openNote)` is the ⌘K menu path.
- `Sources/BobMacCapture/Refs/RefsActionsMenu.swift`: `RefsAction.openNote` is titled
  "Open Note", with key equivalent `"\r"` and modifier `.command`. It sits in the first
  ("opens") section between `.openHighlights`/`.closeTab` and `.reveal`.

## Where the reading task comes from (no bob change)

`bob ref list` already reports a `task` object on each row. The Mac app decodes it as
`RefTask` in `Sources/RefsCore/RefsDecoding.swift` and exposes it as `RefItem.task`. Its
fields:

- `path`: the vault-relative residence. That is a root area, project, or inbox note
  (`sase.md`); a `done/` archive (`archived: true`); or, for a frozen v1 tracker, the
  ref note itself with block `ref`.
- `blockID`: the task's trailing block ID (`ref-<slug>`, or `ref` for v1), possibly nil.

`task` is `null` when the reference has no reading task, or when an older `bob` predates
the field. The inspector's "Reading task" fact row and the Copy card's `R` key already
read this object. The Mac app stays a thin client: it opens what `bob` reported and
parses nothing.

## Behavior

| Situation                                                           | ⌃↵ result                                                                                                                                 |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Row with a reading task                                             | Opens `<vault>/<task.path>` in Obsidian (see `landing`); consumed; no open is recorded; the panel is not hidden by the model (same as ⌘↵) |
| Archived task (`done/…`)                                            | Opens the archive file the same way                                                                                                       |
| v1 tracker (task path is the ref note)                              | Opens the ref note at its `^ref` tracker                                                                                                  |
| No task, or an unsafe path (absolute, empty, or any `..` component) | Consumed; `beep()` plus a VoiceOver announcement "No reading task for this reference"; nothing opens                                      |
| Nothing selected                                                    | Returns `false` and passes through, exactly like ⌘↵                                                                                       |
| Vanished row / synthetic Tabs row                                   | Same banners as ⌘↵, via the existing guards in `openSelected`                                                                             |
| IME composing                                                       | Passes through (the existing Return-arm guard)                                                                                            |
| Copy, Rename, or Link to Today card open                            | Unchanged: those cards route keys before the panel router                                                                                 |

> [!decision] landing = task_line `WorkspaceRefsOpener.openTask(_:blockID:)` dispatches
> `ObsidianOpenURL.taskGotoURL(path: url.path, line: 1, ref: nil, blockID: blockID)`.
> With no `ref`, the bob-navigation-hotkeys handler skips digest matching. It lands on
> the task line that carries `^block`, and otherwise on line 1 with no notice (see
> bob-plugins `docs/goto-task-uri.md`, "Re-finding the line"). If that builder returns
> nil (for example, a block ID with characters outside `[A-Za-z0-9-]`), retry with
> `blockID: nil`, then fall back to `ObsidianOpenURL.url(forAbsolutePath:)`.

> [!decision] landing = file_top Do not add `openTask` to `RefsOpening`. The `.task`
> case calls `opener.openNote(taskURL)`, which opens the file via `obsidian://open`
> exactly like ⌘↵. The test fakes only need to tell the two opens apart by URL. Skip the
> `taskDeepLink` helper and its test, and skip the protocol.md plugin-version sentence.

## Implementation

### 1. RefsCore: one pure, portable accessor

`Sources/RefsCore/RefsModel.swift`: add to `RefItem`:

```swift
/// The vault-relative file holding this reference's reading task (an
/// area/project/inbox note, a `done/` archive, or the ref note itself
/// for a frozen v1 tracker), or nil when bob reported no task or the
/// path is not a safe vault-relative path.
public var readingTaskPath: String? {
    guard let path = task?.path, !path.isEmpty, Self.isSafePDFPath(path) else {
        return nil
    }
    return path
}
```

`isSafePDFPath` already encodes the generic rule: vault-relative, with no `..`
component. Reuse it; do not rename it.

### 2. Library URL

`Sources/BobMacCapture/Refs/RefsLibrary.swift`: next to `noteURL(for:)` and
`pdfURL(for:)`, add:

```swift
/// The absolute URL of the file holding the reading task, or nil when
/// the row has no safe reading-task path.
public func taskURL(for item: RefItem) -> URL? {
  item.readingTaskPath.map { vaultRoot().appendingPathComponent($0) }
}
```

### 3. Router

`Sources/BobMacCapture/Refs/RefsKeyRouter.swift`: in the Return/keypad-Enter arm, after
the `.command` check, add `if flags == .control { return .open(.task) }`. Use the same
exact-flags comparison as its `⌘↵` and `⌥↵` siblings, so `⌃⇧↵` and `⌃⌘↵` stay unbound.
Mention Control-Return in the `command(keyCode:modifiers:context:)` doc comment.
`isPanelSwitch` (Ctrl-Tab) is untouched.

### 4. Model and opener

`Sources/BobMacCapture/Refs/RefsPanelModel.swift`:

- Add `case task` to `RefsOpenTarget`. Update its doc comment so it lists "the reading
  task's file in Obsidian", and keep "Only Highlights and default-app opens count as
  opens" true.
- In `openSelected`, add a `.task` arm:

  ```swift
  case .task:
    guard let url = library.taskURL(for: item) else {
      noReadingTaskFeedback()
      return true
    }
    opener.openTask(url, blockID: item.task?.blockID)  // file_top: opener.openNote(url)
    return true
  ```

- Add `private func noReadingTaskFeedback()`. It calls `beep()` (the injectable closure
  ⌥N already uses) and posts
  `AccessibilityNotification.Announcement("No reading task for this reference")`. This
  matches the Copy card's existing `R` disabled reason text.
- In `openStored`, widen the `case .note, .reveal:` arm to
  `case .note, .reveal, .task:`.
- In `performAction`, add `case .openTask: _ = openSelected(.task)`.
- (task_line) Add `func openTask(_ url: URL, blockID: String?)` to `RefsOpening`.
- (task_line) Implement it in `WorkspaceRefsOpener` through a pure helper,
  `static func taskDeepLink(for url: URL, blockID: String?) -> URL?`, which holds the
  fallback chain from the `landing = task_line` callout. Then `openTask` calls
  `NSWorkspace.shared.open` on the helper's result. It never builds a shell command,
  matching the struct's doc comment.

### 5. ⌘K Actions menu

`Sources/BobMacCapture/Refs/RefsActionsMenu.swift`:

- Add `case openTask` right after `case openNote`, titled `"Open Reading Task"`. Its key
  equivalent is `"\r"` (share the arm with `.openHighlights, .openNote, .reveal`) and
  its modifier mask is `.control`. Update the `keyEquivalent` doc comment so it reads
  "the opens carry ↵, ⌘↵, ⌃↵, and ⌥↵".
- In `sections(for:audioURL:canCloseTab:newState:)`, build the opens section as
  `.openHighlights`, an optional `.closeTab`, then `.openNote`, then `.openTask` only
  when `item.readingTaskPath != nil`, then `.reveal`. Update the function's doc comment
  to say "Open Reading Task appears only when bob reports a reading task".
- `RefsPanel.swift` needs no change: menu items map back through `tag` and
  `performAction`.

### 6. Footer (decision `footer_hint`)

> [!decision] footer_hint = menu_only Leave
> `Sources/BobMacCapture/Refs/RefsFooter.swift` and `RefsFooterTests.swift` unchanged.

> [!decision] footer_hint = widest In `RefsFooter.hintVariants`, insert `  ⌃↵ Task`
> right after `⌘↵ Note` in the first (widest) variant of each list: library, Newest, and
> Tabs. Leave every narrower variant byte-for-byte unchanged. Update the exact-string
> assertions in `Tests/BobMacCaptureTests/RefsFooterTests.swift` for `hintsText`
> (variant 0). Add assertions that variant 0 contains `⌃↵ Task` and that every later
> variant does not.

### 7. Tests

Update every `RefsOpening` conformer. Today those are `FakeOpener` in
`Tests/BobMacCaptureTests/RefsPanelModelTests.swift` and the two `StubOpener`s in
`RefsInspectorTests.swift` and `RefsPanelDesignTests.swift`; grep `: RefsOpening` to
confirm. Under task_line, `FakeOpener` records
`taskOpens: [(url: URL, blockID: String?)]` under its lock, and the stubs get a no-op.

- `Tests/RefsCoreTests/RefsModelTests.swift`, which runs on Linux via portable-check:
  `readingTaskPath` returns `sase.md`, `done/sase_done.md`, and a v1 `ref/chat/x.md`. It
  returns nil for a nil task, an empty path, `/abs.md`, `../outside.md`, and
  `a/../b.md`.
- `Tests/BobMacCaptureTests/RefsKeyRouterTests.swift`: extend
  `testReturnOpensInHighlights` so `.control` on Return and on keypad Enter maps to
  `.open(.task)`, and `[.control, .shift]` and `[.control, .command]` map to nil. Extend
  `testReturnPassesThroughWhileComposing` with the Control variant.
- `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`: seed rows through
  `FakeRefsFetcher` and `makeRenameHarness(fetcher:)`. The
  `RefRecord(… task: RefTask(path: "sase.md", blockID: "ref-attention", link: "[[sase#^ref-attention]]", mark: "*") …)`
  pattern already exists near the Today-link tests. Cover:
  - A v2 task: `perform(.open(.task))` is true. The opener got `<vault>/sase.md` with
    block `ref-attention` (file_top: in `noteOpens` with the task URL), the reference
    note itself was not opened, `dismissed == 0`, and `signals.opens.count(id) == 0`.
  - An archived `done/sase_done.md` task opens that file.
  - No task, and a `../outside.md` task path: true, `beeps == 1` (inject `model.beep`),
    and nothing opened.
  - `performAction(.openTask)` takes the same path.
  - `actionsSections().first` holds `[.openHighlights, .openNote, .openTask, .reveal]`
    for a task row and `[.openHighlights, .openNote, .reveal]` without one.
  - If an existing test asserts `showNoteActionWarning` for `.open(.note)` on a
    synthetic Tabs row, add the same assertion for `.open(.task)`.
- `Tests/BobMacCaptureTests/RefsPanelGeometryTests.swift`
  (`testActionsMenuShowsOpenShortcutHints`): `RefsAction.openTask` has key equivalent
  `"\r"` and modifier `.control`, and its title is "Open Reading Task".
- `Tests/BobMacCaptureTests/RefsInspectorTests.swift`
  (`testActionsMenuVariesWithURLsAndAudio`): assert the opens section with and without
  `.openTask`, building a fixture item with and without a `RefTask`.
- (task_line) Add a `WorkspaceRefsOpener.taskDeepLink` test. `/vault/sase.md` with
  `ref-attention` gives exactly
  `obsidian://bob-goto-task?path=/vault/sase.md&line=1&block=ref-attention`. A nil block
  omits `block=`, and a block of `bad id` falls back to the no-block URL. Put it in
  whichever BobMacCaptureTests file already tests Refs opener or URL code, or else in
  `RefsPanelModelTests.swift`.

### 8. Docs (bob-mac-capture `docs/`, mounted into Bob's site)

- `docs/refs.md`: in the "Key | Bob Refs panel" table, add a row right after
  Command-Return:
  `| Control-Return | Open the reading task's file in Obsidian, landing on the task line; rows without a reading task beep |`.
  Under file_top, drop ", landing on the task line". Where nearby prose says what ⌘↵
  does on an unmatched Tabs row, note that ⌃↵ behaves the same.
- `docs/keyboard.md` "Bob Refs shortcuts" table: add
  `| Control-Return | Open the reading task's file in Obsidian (at the task line) |`
  (file_top: without the parenthetical).
- `docs/refs-inspector.md` ⌘K paragraph: after "Open Note (⌘↵)," insert "Open Reading
  Task (⌃↵, only when `bob` reports a reading task),".
- `docs/protocol.md`:
  - In the capability table row for `` `task` on a Refs row ``, append "; ⌃↵ opens the
    reading task" to the with-support cell and "; ⌃↵ beeps" to the older-bob cell.
  - (task_line) Extend the "Go To and Control-\ today-Pomodoro landing also require …
    bob-navigation-hotkeys ≥ 2.32.0" sentence to also name Bob Refs Control-Return. The
    handler's `block` landing shipped with the handler in 2.32.0.

No decision record is needed, since decision records exclude keymaps. No SASE memory
note states anything this change makes false.

## Verification

This host has no `swift`. Follow bob-mac-capture's `AGENTS.md` exactly:

1. Run `just portable-check` on a tailnet host with Swift (athena). Use your
   `/sase_memory_read` skill on `tailnet.md` for SSH access. This gate format-lints
   every Swift file and builds and tests RefsCore, including the new `readingTaskPath`
   test.
2. `Sources/BobMacCapture*` and `Tests/BobMacCapture*` change, so compile a scratch copy
   on the mac (best effort). rsync without `.build`/`.git`, run `Scripts/bundle.sh`,
   then `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
   `just install`. If the mac is offline, say in the final report that the macOS targets
   are unverified.
3. Run `just docs-check` for the docs edits.
4. After pushing, check CI. CI runs the BobMacCapture XCTest suites, which the Mac's
   Command Line Tools cannot. On a red **Lint Swift formatting** step, check the later
   steps too.

Do not land Swift that has not compiled.

## Out of scope

- Any bob-cli or bob-plugins change. `bob ref list` already emits `task.path` and
  `task.block_id`, and `obsidian://bob-goto-task` already lands by `block`.
- Opening follow-up or annotation tasks: ⌃↵ targets only the reference's one reading
  task.
- Rebinding ⌘↵, or making ⌃↵ fall back to the reference note when there is no task. That
  would blur the two keys, so it beeps instead.
