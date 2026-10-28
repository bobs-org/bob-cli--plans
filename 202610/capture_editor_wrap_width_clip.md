---
tier: tale
title: Stop the capture editor from clipping soft-wrapped lines
goal:
  The Bob Mac Capture draft editor always grows to show every soft-wrapped line, because
  its hidden sizing text wraps at the same width as the backing NSTextView, and a
  live-panel regression test guards that.
size: small
proposed_by: bbugyi200.athena.150
proposed_with: claude/opus
create_time: 2026-10-28 07:11:43
status: wip
---

# Plan: Fix the capture editor clipping a soft-wrapped line (sizer wraps wider than the NSTextView) clips a soft-wrapped line (sizer wraps wider than the NSTextView)

All changes are in the linked **bob-mac-capture** repo (open it with `/sase_repo`; read
its `AGENTS.md` first). bob-cli itself does not change. This is a pure presentation bug,
so it stays on the app's side of the thin-client boundary
(`decisions:mac-capture-is-a-thin-client`).

## Symptom

Bryan's screenshot shows the capture editor at its one-line height (42 pt) while the
draft
`Make it so all themes need to be configured (or a warning shows)! @bob:configure-themes`
(87 characters, one logical line) soft-wraps to **two** visual lines. The `NSTextView`
scrolls to the caret line, so the first visual line is cut off at the top of the box.
The live preview proves the draft is one line (no child row), so this is a soft wrap,
not a newline. It happens only "sometimes" because it needs a visual line whose width
falls in a narrow ~10 pt band (see below).

## Root cause (confirmed empirically on the Mac)

`AutosizingCaptureEditor` (`Sources/BobMacCapture/CapturePanelView.swift`, around lines
1227–1330) does not measure the real editor. It predicts the height by laying out a
hidden SwiftUI `Text(verbatim: draft + "\u{200B}")` (`sizingText(width:)`) at

```swift
let usableTextWidth = max(1, proxy.size.width - textInset * 2)
```

and feeds that height through `CaptureEditorHeightPolicy.resolvedHeight`.

But the `TextEditor`'s backing `NSTextView` keeps AppKit's default
`NSTextContainer.lineFragmentPadding` of **5 pt on each side**. Its text container is
exactly `usableTextWidth` wide, but glyphs wrap at `usableTextWidth − 10`. The
placeholder already accounts for this (`.padding(.leading, textInset + 5)`); the sizer
does not.

A throwaway AppKit/SwiftUI probe run on the Mac (macOS 26.5, default 760 pt panel, so
the editor is 724 pt wide) measured:

- body monospaced font `.AppleSystemUIFontMonospaced-Regular` 13 pt, advance 8.036 pt,
  so the 87-char draft is 699.1 pt wide;
- `NSTextView` frame 704 pt wide, `textContainerInset` (0, 0), container width 704,
  `lineFragmentPadding` 5, so it wraps at 694 pt: TextKit 2 lays out **2 lines, 32 pt**;
- sizing `Text` at 704 pt: **16 pt (1 line)**. The same `Text` at 694 pt: 32 pt.

So the policy resolves `max(16, 22) + 20 = 42` pt. The `TextEditor` gets a 22 pt frame
and shows one of its two lines, which is exactly the screenshot.

A sweep across 4,984 cases compared the live `TextEditor` (TextKit 2
`usageBoundsForTextContainer` height) with the sizing `Text`. It covered every prefix of
five corpora (prose with Bob syntax, a long URL, tab-indented lines, hard newlines,
emoji/CJK/accents, with alternating italic and colored spans like the highlighter's) at
editor widths 584, 640, 700, 724, 760, 901.5, and 1013 pt:

- sizer at `width − 20` (current): **83 mismatches**, every one an under-measure, which
  clips a line;
- sizer at `width − 30` (also subtracting 2 × 5 pt line-fragment padding): **0
  mismatches**.

Line heights already agree (16 pt per line in both engines), so the wrap width is the
only divergence. Fixing the width makes the existing synchronous sizer exact.

## Design choice

Keep the synchronous hidden-`Text` sizer and make it wrap at the same width the
`NSTextView` does. Do not measure the `NSTextView` directly. Observing AppKit layout
would add an asynchronous hop, so the editor would visibly scroll for a frame before
growing. The sweep shows the corrected sizer matches the real layout exactly. A
live-panel regression test (below) guards against future drift, such as a macOS release
changing SwiftUI's `TextEditor` defaults.

## Changes

### 1. `Sources/BobMacCapture/CapturePanelView.swift`: one wrap-width source of truth

In `enum CapturePanelLayout`, next to `editorContentPadding` and `editorLineHeight`,
add:

```swift
/// AppKit's default `NSTextContainer.lineFragmentPadding`, which SwiftUI's
/// `TextEditor` keeps on its backing `NSTextView`: glyphs sit this far inside
/// each side of the editor's text container.
static let editorLineFragmentPadding: CGFloat = 5

/// Width at which the editor's `NSTextView` wraps a line, for an editor
/// `editorWidth` points wide. The hidden sizing text must wrap here too: if it
/// wraps wider, a line that just fits the sizer soft-wraps in the editor and
/// the box clips one line.
static func editorTextWrapWidth(editorWidth: CGFloat) -> CGFloat {
    max(1, editorWidth - (editorContentPadding + editorLineFragmentPadding) * 2)
}
```

In `AutosizingCaptureEditor.body`:

- Replace `let usableTextWidth = max(1, proxy.size.width - textInset * 2)` with
  `let wrapWidth = CapturePanelLayout.editorTextWrapWidth(editorWidth: proxy.size.width)`
  and pass `wrapWidth` to `sizingText(width:)`. Keep the sizer's `.padding(textInset)`,
  `.accessibilityHidden(true)`, `.allowsHitTesting(false)`, and `.opacity(0)` unchanged.
  Only its frame width changes, and the sizer is invisible, so it needs no extra
  positioning.
- In the placeholder `Text("Type to capture…")`, replace the literal
  `.padding(.leading, textInset + 5)` with
  `.padding(.leading, textInset + CapturePanelLayout.editorLineFragmentPadding)`, so the
  placeholder and the sizer read the same constant. Leave
  `.padding(.top, textInset + 3)` as is.

Do not touch `CaptureEditorHeightPolicy`, `CaptureEditorHeightBudget`, panel metrics, or
`CaptureEditorTextConfiguration`. Height math and the screen budget are correct; only
the measured width was wrong.

Follow `.swift-format` (4-space indent, 100 columns) on the changed lines only; never
re-indent whole files.

### 2. Unit test (portable arithmetic): `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`

Next to `testEditorHeightPolicyUsesOneLineMinimumAndInjectedCap`, add
`testEditorTextWrapWidthSubtractsInsetAndLineFragmentPadding`:

- `CapturePanelLayout.editorTextWrapWidth(editorWidth: 724) == 694`, the default 760 pt
  panel minus 2 × 18 root padding.
- `editorTextWrapWidth(editorWidth: 584) == 554`, the 620 pt minimum panel.
- `editorTextWrapWidth(editorWidth: 10) == 1`, the floor.
- Assert `CapturePanelLayout.editorLineFragmentPadding == 5`, with a one-line comment
  that it mirrors AppKit's default.

### 3. Live regression test: new `Tests/BobMacCaptureTests/CaptureEditorWrapHeightTests.swift`

Follow the native-composition style of `CapturePanelAgendaLayoutTests.swift` and
`CapturePanelEyeLineTests.swift`: `@MainActor`, the production
`CapturePanelController` + `NSPanel` + `NSHostingView`, and
`XCTSkip("no visible frame on this host")` when
`panel.screen?.visibleFrame ?? NSScreen.main?.visibleFrame` is nil. Copy the small
private helpers it needs (`visibleFrame(for:)`, the bounded `settlePanel(_:)` loop,
`editableTextViews(in:)`) instead of making the agenda suite's helpers shared. Use a
plain `CapturePanelModel(debounceNanoseconds: 0)` with no process client, so no `bob` is
spawned. Set drafts with `model.plainDraft = text` and then call `settlePanel(panel)`.

Shared assertion `assertEditorShowsWholeDraft(panel, label:)`:

- Find the single editable `NSTextView` in `panel.contentView`, and fail if there is
  none.
- Call `textLayoutManager!.ensureLayout(for: documentRange)` and read
  `contentHeight = textLayoutManager!.usageBoundsForTextContainer.height`.
- Read `visibleHeight = textView.enclosingScrollView!.contentView.bounds.height`.
- `XCTAssertLessThanOrEqual(contentHeight, visibleHeight + 0.5, info)`, and
  `XCTAssertEqual(enclosingScrollView!.contentView.bounds.origin.y, 0, accuracy: 0.5, info)`.
  Use a diagnostic `info` string with the label, panel content width, `textView.frame`,
  `contentHeight`, `visibleHeight`, and the clip origin.

Tests:

1. `testScreenshotDraftShowsBothWrappedLines`: at the default panel width, set the exact
   screenshot draft
   `"Make it so all themes need to be configured (or a warning shows)! @bob:configure-themes"`
   and assert the whole draft is visible. Before the fix this fails with content 32 vs
   visible 22.
2. `testLineLengthsAroundWrapBoundaryNeverClip`: for each panel content width in
   `[CapturePanelLayout.panelInitialContentWidth, CapturePanelLayout.panelMinimumContentWidth]`,
   call `panel.setContentSize(NSSize(width: w, height: panel.contentLayoutRect.height))`
   and settle. Compute
   `wrap = CapturePanelLayout.editorTextWrapWidth(editorWidth: w - 2 * CapturePanelLayout.rootPadding)`.
   Take the advance of `"x"` in
   `NSFont.monospacedSystemFont(ofSize: NSFont.systemFontSize, weight: .regular)` (used
   only to aim the sweep) and set `boundary = Int(wrap / advance)`. For `n` in
   `(boundary - 1)...(boundary + 2)`, set `String(repeating: "x", count: n)`, settle,
   and assert the whole draft is visible. That range covers every length inside the old
   10 pt band: 87 at 760 pt, and 69 and 70 at 620 pt. Then repeat the sweep once with a
   space-separated word draft (for example `"word "` repeated and trimmed to `n`
   characters), so word wrapping is covered as well as character wrapping.

Keep the sweep to these bounded lengths (about 16 settles in total) so the suite stays
fast. Never await an untimed continuation (see `AGENTS.md`).

### 4. Docs

None. `docs/capture-panel.md` already promises that the editor "grows and shrinks with
rendered content". The fix makes the app match that promise.

## Verification

Per bob-mac-capture's `AGENTS.md`, never land Swift you have not compiled:

1. `just portable-check` on the Linux host. If `swift` is missing there, run it on
   athena. This parse-lints every file, including the changed ones.
2. Compile the macOS targets on the Mac (`ssh mac`; best effort, macOS tailnet host).
   rsync a scratch copy of the repo without `.build`/`.git` to a temp directory, run
   `./Scripts/bundle.sh` and `just typecheck-tests` there, then delete the scratch copy.
   Never use Bryan's checkout or `just install` on the Mac. If the Mac is offline, say
   in the final report that the macOS targets are unverified.
3. Run the new tests in real CI before committing: `Scripts/macos-ci-check` from the
   repo (see `docs/develop/ci.md`). It can take many minutes, so hand it to
   `/sase_monitor` rather than blocking. The Mac's Command Line Tools have no XCTest, so
   CI is the only place the live test runs. Require the `Lint Swift formatting`,
   `Build`, `Test`, and later macOS steps to pass. If the live test fails, read the
   diagnostic `info` before changing anything. Do not weaken the assertion to make it
   pass.

## Out of scope

- No change to editor height policy, the screen budget, the eye line, or panel sizing.
- No measurement of the `NSTextView` itself, and no pinning of AppKit's
  `lineFragmentPadding` (see Design choice).
