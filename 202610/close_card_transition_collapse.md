---
tier: tale
title: Stop the close-card transition column from collapsing in Bob Mac Capture
goal:
  Long Task Link titles truncate on one line while the close card's status transitions
  stay legible.
size: small
decisions:
  wikilink_titles:
    ask:
      Should close- and start-card task titles also hide wikilink markup, as the Task
      Link picker does?
    choices:
      display:
        "`[[ref/x|Alias]]` titles read `Alias`, so long ref titles truncate less often"
      raw: Titles keep Bob's raw task text; only the column collapse is fixed
    default: display
    why:
      Matches the picker that chose these tasks; raw ref links pushed these rows past
      the width
    answer: display
proposed_by: bbugyi200.apollo.86
decided_by: auto
create_time: 2026-10-16 14:11:43
status: wip
---

# Stop the Close-Card Transition Column From Collapsing in Bob Mac Capture

## Symptom

With a draft like `-2 =!1`, Bob Mac Capture's Pomodoro close card shows some Task Link
rows broken: the monospaced `[*] → [/]` transition becomes a one-glyph-wide vertical
stack of characters under the row, each broken row grows about 130pt tall, and the
trailing locator shrinks to a stray glyph. Rows whose titles fit (a short
`[[ref/blogs/diataxis…]]`, `[/] Release v0.18.0!`) render correctly. That is why it only
happens "sometimes": it depends on title length versus panel width.

## Root Cause (verified)

In bob-mac-capture `Sources/BobMacCapture/CapturePanelView.swift`, the task row in
`closePreviewItem(_:success:index:total:)` is an `HStack` of, in order: the number
badge, `closeTaskGlyph(for:)`, the title
`Text(row.taskText).strikethrough(…).layoutPriority(1).lineLimit(1)`, an optional
`Text(tag)`, `Spacer(minLength: 4)`, and the `lineLimit(1)` locator.

For worked and mentioned rows, `closeTaskGlyph(for:)`'s `.transition` case renders
`Text(row.transitionText)` in monospaced callout with neither `lineLimit(1)` nor
`fixedSize`. SwiftUI's `HStack` first offers the priority-1 title all the width that
remains after every priority-0 sibling's _minimum_ width. A wrappable `Text` can shrink
to about one glyph wide, so a long title takes nearly the whole row and the transition
is left with its minimum width. It then wraps one character per line. The function's own
comment ("the task text and transition always stay legible") was never enforced. The
optional `Text(tag)` (`linked` / `new` on link and new-task closes) has the same
exposure.

The diagnosis was reproduced on the Mac with a scratch SwiftUI copy of this row at the
card's real width (about 720pt), using the screenshot's titles:

- **As-is:** the three-row stack measured 258pt tall and rendered exactly like the
  screenshot.
- **With the transition pinned**
  (`.lineLimit(1).fixedSize(horizontal: true, vertical: false)`): it measured 78pt,
  every row stayed one line, and only the title and locator truncated.

Neighbors checked and **not** affected:

- The start card's glyphs are all SF Symbols.
- The Unblocked, still-blocked, toggle, link, and task-complete rows have no priority-1
  sibling, so they share width normally.

bob-cli's `pomodoro_close` JSON is correct. **No bob-cli changes are needed.**

## Where the Work Happens

All changes land in the linked repo `bob-mac-capture`. Open it with the `/sase_repo`
skill (`sase repo open bob-mac-capture -r "<reason>"`), use only the path it prints, and
read its `AGENTS.md` before editing. Commit in that repo. This bob-cli checkout stays
untouched.

## Changes

1. **Pin the transition glyph.** In `closeTaskGlyph(for:)`'s `.transition` case, add
   `.lineLimit(1)` and `.fixedSize(horizontal: true, vertical: false)` to
   `Text(row.transitionText)`. Leave the image cases alone; images are already
   inflexible.
2. **Pin the tag.** In the close task row, give `Text(tag)` the same
   `.lineLimit(1).fixedSize(horizontal: true, vertical: false)`.
3. **Make the comment true.** Reword the layout comment at the top of `closePreviewItem`
   to state the enforced order: the badge, transition, and tag never shrink. At narrow
   widths the locator truncates first, then the one-line title.
4. **Wikilink display titles.**

   > [!decision] wikilink_titles = display
   >
   > - Add a computed
   >   `public var displayTitle: TaskDisplayText { TaskDisplayText(parsing: taskText) }`
   >   to both `CapturePomodoroClosePresentation.TaskRow` and
   >   `CapturePomodoroStartPresentation.TaskRow` in `Sources/CaptureCore`. It is
   >   computed, so `Equatable` and the memberwise init are unchanged.
   > - In both cards' task rows (`closePreviewItem` and `startPreviewItem` in
   >   `CapturePanelView.swift`), render `Text(row.displayTitle.text)` instead of
   >   `Text(row.taskText)`. Keep `.strikethrough(row.isStruck)`, `.layoutPriority(1)`,
   >   `.lineLimit(1)`, and `.textSelection(.enabled)`.
   > - Use plain display text, not `CapturePickerRichText`: its `.body`-sized code font
   >   and accent link tint would clash with the callout card and with struck or dimmed
   >   rows.
   > - Leave `taskText` and every accessibility label/summary unchanged.

   > [!decision] wikilink_titles = raw
   >
   > Skip this step entirely. Titles keep rendering `row.taskText`.

## Tests

1. **Layout regression (BobMacCaptureTests).** Add `PomodoroCloseRowLayoutTests.swift`,
   or extend `CapturePreviewFullHeightTests.swift` to reuse its
   `hostedPreviewHeight(for:width:)` and `readyModel(for:)` helpers. Mark tests
   `@MainActor` and `@available(macOS 26.0, *)` like those helpers. Build each
   long-title variant from the raw fixture JSON:
   - Read the fixture `Data`, then mutate it with `JSONSerialization`.
   - Set a task's `text` to a long plain title of 300+ characters with no wikilinks, so
     both decision branches exercise truncation.
   - Decode the result as `CaptureCommandResponse`.

   Cases:
   - `pomodoro-close-worked.json`: lengthen task index 1, the worked `[*] → [/]` row.
   - `pomodoro-close-link.json`: lengthen the row whose `block_id` is `ready`. That is
     the `linked`-tagged worked row.

   For each case, at widths 620 and 760, host `PreviewPane` in an `NSHostingView`.
   Assert that the long-title `fittingSize.height` equals the original fixture's height
   within 1pt. Without the fix, the reproduction grew by about 130pt per row, so the
   test fails loudly.

2. **Display titles (CaptureCoreTests, decision `display` only).** In
   `CapturePomodoroClosePresentationTests` and `CapturePomodoroStartPresentationTests`,
   cover a row whose task text is ``[[ref/chat/x|Alias Title]] and `code` ``. Use those
   files' existing fixture/JSON construction style. Assert that `displayTitle.text` is
   `Alias Title and code` while `taskText` stays raw.

## Docs

In bob-mac-capture's `README.md` close-card paragraph (the one listing "up to six task
rows with per-role glyphs, transitions (`[*] → [/]`, …)"), add one sentence: badges,
transitions, and tags never wrap or shrink, and at narrow widths the locator truncates
first, then the one-line task title.

Under decision `display`, also note there and in the start-card paragraph that row
titles show wikilink display text (`[[path|Alias]]` reads `Alias`), like the pickers.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

- `swift` is not installed on apollo. Run `just portable-check` on athena over tailnet
  SSH, in a scratch copy of the edited tree. It parse-lints every Swift file and builds
  and tests `CaptureCore`, which includes the display-title tests. Delete the copy
  afterward.
- `portable-check` cannot type-check `Sources/BobMacCapture*` or `Tests/BobMacCapture*`,
  so compile those on the Mac (best effort):
  1. rsync a scratch copy there, excluding `.build` and `.git`.
  2. Run `Scripts/bundle.sh` (compiles the app target).
  3. Run `just typecheck-tests` (type-checks the new layout test).
  4. Delete the copy.

  Never use Bryan's checkout or `just install` there. If the Mac is offline, say in the
  final report that the macOS targets are unverified.

- CI runs the BobMacCaptureTests layout regression. After pushing, if **Lint Swift
  formatting** is red, check the later CI steps too.
