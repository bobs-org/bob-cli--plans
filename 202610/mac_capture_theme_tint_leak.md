---
tier: tale
title: Stop Bob Mac Capture's theme tint from leaking past the theme
goal:
  The capture editor tints only the theme bytes bob reports, never inherited neighbors,
  and relation-label italics stay on the label.
size: small
proposed_by: bbugyi200.apollo.aa
proposed_with: claude/opus
create_time: 2026-10-26 11:37:04
status: wip
---

# Plan: Stop Bob Mac Capture's theme tint from leaking past the theme

## Symptom

In the capture panel, a draft like

```text
Add new working time bar to day strip that groups by theme! @bob:theme-day-strip-bar#vibe=

Fix theme highlighting in bob-mac-capture! @bob
```

paints the VIBE theme tint behind `vibe=` **and** the whole second item, not just behind
`vibe`.

## Diagnosis (verified)

**bob is correct.** `bob capture-parse --format json` on that draft returns:

| kind                | bytes      | notes                                                    |
| ------------------- | ---------- | -------------------------------------------------------- |
| `pomodoro_route`    | `[8, 12)`  |                                                          |
| `pomodoro_block_id` | `[13, 32)` |                                                          |
| `pomodoro_name`     | `[33, 37)` | `themes: [{key: vibe, hue: orange, start: 33, end: 37}]` |
| `pomodoro_start`    | `[37, 38)` | the `=`; no theme ref                                    |
| `route`             | `[64, 68)` | `@bob` on the second item                                |

So bob asks for a tint over `vibe` only. (Those offsets are for the shortened draft
`Add bar @bob:theme-day-strip-bar#vibe=\n\nFix theme highlighting! @bob`, which the
regression test below reuses.)

**The leak is in bob-mac-capture's editor repaint**
(`CapturePanelModel.applyHighlighting(parse:draft:)` in
`Sources/BobMacCapture/CapturePanelModel.swift`):

1. The draft editor is a SwiftUI `TextEditor(text: $model.attributedDraft, selection:)`.
   A typed character takes the attributes of the character before the caret. Typing `=`
   right after a tinted `vibe` gives the `=` the tint's `backgroundColor`. Each later
   keystroke (Return, Return, `Fix theme…`) inherits it from the character before it.
2. Before painting bob's spans, each parse repaint resets only `foregroundColor`,
   `underlineStyle`, and `underlineColor` across the whole draft. The PT6 theme-tint
   commit (`a306745`) added the `backgroundColor` paint but no matching reset. The
   inherited tint is never removed, and every repaint re-adds the real tint on `vibe`.
   Syntax foreground colors never leak lasting color only because they _are_ reset.
3. The same latent bug exists for `relation_label` spans. They set
   `inlinePresentationIntent = .emphasized`, and that is never reset either, so text
   typed after a relation label stays italic.

It happens only "sometimes" because the leak needs text typed directly after a tinted
known theme, for example a `=` start suffix, or continuing a bulk draft after `#name=`.
Text placed by `setPlainDraft` (completion acceptance, rewrites, stash restore) replaces
the attributed value wholesale and starts clean. That is also why `@bob` was untinted
until more text was typed next to it.

## Work

All changes are in the **bob-mac-capture** linked repo. Open it with your `/sase_repo`
skill (`sase repo open bob-mac-capture -r "<reason>"`), use only the path it prints, and
read that repo's `AGENTS.md` before editing. bob-cli needs no change.

### 1. Repaint from a clean slate

In `Sources/BobMacCapture/CapturePanelModel.swift`:

- Add one helper, for example
  `static func clearEditorHighlightAttributes(_ text: inout AttributedString)`. It nils
  every attribute the editor painters own, over the whole draft: `foregroundColor`,
  `backgroundColor`, `underlineStyle`, `underlineColor`, and `inlinePresentationIntent`.
  Give it a short doc comment saying why. Typed text inherits the previous character's
  attributes, so every attribute a painter writes must also be cleared here. Put the
  list in one place so the next painter attribute added cannot miss its reset.
- Move the body of the
  `attributedDraft.transform(updating: &editorSelection) { text in … }` closure in
  `applyHighlighting` into a static, test-drivable function, for example
  `static func paintHighlighting(_ text: inout AttributedString, parse: CaptureParseResponse, draft: String, ranges: [ValidatedCaptureSpan]) -> Bool`.
  It returns whether a malformed span was ignored. Its first statement calls
  `clearEditorHighlightAttributes(&text)` in place of the three partial resets. Keep
  everything else the same: palette colors, underline/italic categories, the known-theme
  tint loop, the template-diagnostic red underline, and the early stop on an unmappable
  index. `applyHighlighting` keeps its guards, the `isApplyingProgrammaticDraft`
  bracket, and the status text, and calls the new function inside the transform. Mark
  the statics `nonisolated` only if the compiler needs it.
- Keep the trailing `applyPickerMarkerHighlight()` call in `applyHighlighting`. Now that
  the reset clears `backgroundColor`, it also wipes an open picker's accent wash, and
  this call puts the wash back in the same parse application. Update the comment there
  to say so.

### 2. Clear stale styling when the draft goes whitespace-only

`editorTextDidChange` returns early on a whitespace-only draft (after
`resetAnalysisState()`), and no parse ever repaints it. Spaces or newlines that
inherited a tint from deleted text would keep it. In that branch, after
`resetAnalysisState()`, run `clearEditorHighlightAttributes` on `attributedDraft`
through `attributedDraft.transform(updating: &editorSelection)`, bracketed by
`isApplyingProgrammaticDraft = true/false`. This is the same attribute-only pattern
`clearPickerMarkerHighlight` uses: no text, selection, or undo change. An empty draft
needs nothing.

### 3. Regression tests

Add `Tests/BobMacCaptureTests/CaptureEditorHighlightTests.swift` (a `@MainActor`
`XCTestCase` that imports `CaptureCore`, `SwiftUI`, `XCTest`, and
`@testable import BobMacCapture`). Use a helper that lists the substrings of runs
carrying a given attribute, modeled on `washedStrings(in:)` in
`CapturePickerMarkerHighlightTests.swift`:

- **Screenshot regression.** Draft
  `Add bar @bob:theme-day-strip-bar#vibe=\n\nFix theme highlighting! @bob`, with the
  five spans from the table above
  (`CaptureSpan(…, themes: [CapturePomodoroThemeRef(key: "vibe", name: "vibe", hue: .orange, hueSource: "hash", start: 33, end: 37)])`
  on the name). Build the `AttributedString`. To simulate inherited typing attributes,
  give bytes `33..<end` some `backgroundColor` and `.emphasized`. Get `ranges` from
  `validatedSpanRanges(in:spans:)`, build a `CaptureParseResponse`, and call
  `paintHighlighting`. Assert that the only background run is `["vibe"]` and no run
  carries `inlinePresentationIntent`.
- **Unknown theme drops the tint.** Same seeded draft, but the theme ref has
  `known: false`. Assert there are no background runs.
- **Relation label italic stays on the label.** Use a small relation draft with
  hand-built `relation_sigil` / `relation_note` / `relation_block_id` /
  `relation_label_separator` / `relation_label` spans, and seed the text after the label
  with `.emphasized`. If `bob capture-parse --format json -- '<draft>'` is available,
  check the offsets against it. Assert that exactly the label substring is emphasized.
- **Whitespace-only edit clears inherited tint.** Use a `CapturePanelModel()` with no
  process client. Assign `model.attributedDraft` an `AttributedString("  ")` with a
  `backgroundColor`, call `model.editorTextDidChange(cursorUTF8Offset: 2)`, and assert
  there are no background runs.

### Out of scope

- A character typed right after a colored or tinted token can still flash the inherited
  style until the debounced parse repaints it (about 50 ms plus the bob call). Syntax
  foreground colors have always behaved this way. Stripping attributes on every
  keystroke would mean a per-keystroke attribute transform during `TextEditor` edits,
  which risks IME/marked-text input, for a cosmetic flash.
- No bob-cli, protocol, or docs change. bob's spans are correct, and the
  `docs/protocol.md` description of editor tints stays true.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

- Run `just portable-check` (format lint, plus build and test of the portable targets).
  If this host has no `swift`, run it on athena.
- The changed files live in `Sources/BobMacCapture` and `Tests/BobMacCaptureTests`,
  which `portable-check` cannot type-check. Compile a scratch copy on the `mac` tailnet
  host (best effort): rsync the repo without `.build`/`.git`, run `./Scripts/bundle.sh`
  and `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
  `just install` there. If the mac is offline, say in the final report that the macOS
  targets are unverified. CI runs the XCTests.
- Commit in bob-mac-capture, for example
  `fix(capture): reset theme tint and italic before each editor repaint`, and say in the
  body that the tint leaked through inherited typing attributes.
