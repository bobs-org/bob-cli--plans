---
tier: tale
title: Type @@@ to start a capture template
goal:
  "In Bob Mac Capture, a third @ at the start of the draft becomes @@template: with the
  caret on a blank line below, and hints teach it."
size: medium
proposed_by: bbugyi200.athena.12d
proposed_with: claude/opus
create_time: 2026-10-20 07:18:31
status: wip
---

# Plan: Type `@@@` to start a capture template

## Goal

In the Bob Mac Capture panel, a third `@` typed at the very start of the draft (the
draft's first line is `@@` and the caret sits right after it) turns the line into
`@@template:` and moves the caret onto a new blank line below it, ready for the template
line. Add hints so people can find the gesture.

Before (`|` is the caret):

```text
@@|
```

The user presses `@`. After:

```text
@@template:
|
```

## Why bob owns the rewrite

The accepted decision `decisions:mac-capture-is-a-thin-client` says bob-cli is the only
implementation of the capture grammar. The app may filter or rank what bob returns, but
it never decides syntax itself. bob already has a command for editor typing assists,
`bob capture-rewrite`. It handles the bare `@@` absorption and the `:`/`^` block-ID
separator toggle, and the Mac app already calls it on a dedicated keystroke-intent lane,
applies the reply as one native `NSTextView` edit, and lets Undo restore the literal
typed text. `@@@` is currently a bob error (`invalid_global_destination`), so the new
rule cannot change the meaning of any valid draft.

So the work lands contract-first in two repositories:

1. **bob-cli** adds a `capture-rewrite` rule, `start_template`.
2. **bob-mac-capture** sends the third `@` to that rule, applies the result natively,
   and shows the hints.

A Swift-only snippet (like Tab's `--` → `—`) was rejected because it would put
`@@template:` header grammar into the app.

## Rule contract (bob-cli)

`start_template` fires only when all of these hold:

- `--cursor` is given and equals `3`. Without `--cursor` the rule is skipped, just like
  the separator toggle, because it is a typing gesture.
- The draft's bytes `0..3` are `@@@`, and the first physical line is exactly that. Byte
  3 must be the end of input, `\n`, or `\r\n`.

When it fires:

- **Edit:** a single edit `{start: 0, end: 3, replacement: "@@template:" + T}`. `T` is
  the first line's existing terminator: `\r\n` when the line ends in CRLF, otherwise
  `\n`.
- **Cursor:** `11 + len(T)`, which is `12` for LF. That is the start of the new blank
  line.
- **Summary:** `Started a @@template: draft`. There are no notices.
- **Examples:**
  - `@@@` with cursor 3 becomes `@@template:\n`, cursor 12.
  - `@@@\nApply {0}` with cursor 3 becomes `@@template:\n\nApply {0}`, cursor 12. The
    caret is always on a fresh blank line, and any existing lines move down.

These cases stay with the existing rules or come back unchanged:

- leading whitespace or blank lines
- `@@@@` and `@@@ x`
- `@@@` anywhere other than byte 0
- any cursor other than 3
- no cursor

The rule is idempotent. Its output has a template header, so the existing "template
drafts are never rewritten" guard returns `changed: false` when the output is fed back
in.

## Part 1: bob-cli

1. In `src/native/capture_language/rewrite.rs`:
   - Add `RewriteRule::StartTemplate` with code `start_template`.
   - In `rewrite_draft`, check it after the template-header guard and before
     `switch_block_id_separator`, only when a cursor is present. Use a small
     `fn start_template(raw_text, cursor) -> Option<DraftRewrite>` that builds the edit
     with `apply_text_edits`.
   - Update the `rewrite_draft` doc comment.
2. In `src/native/capture_rewrite.rs` `build_cli()`:
   - Change "It implements two rules" to three.
   - Add a "Template start (requires --cursor)" paragraph in the same voice as the
     others.
   - Add the example `bob capture-rewrite -c 3 -f json -- '@@@'`.
   - Follow `cli_rules.md`. There are no new flags.
3. Docs. Read the `docs.md` memory first.
   - In `docs/capture.md` `## bob capture-rewrite`:
     - name the third rule in the intro sentence
     - add a **Template start.** paragraph with the example above
     - add `start_template` to the `rule` list and to the `cursor` sentence
   - In `### Bulk templates`, add one sentence: editors that run `capture-rewrite` (the
     Mac capture panel) turn a typed `@@@` at the very start of a draft into the header,
     with the caret on the next line. Link it to the rewrite section.
   - Regenerate `docs/reference/cli/capture-rewrite.md` with `just docs-gen`. Never
     hand-edit it.
4. Tests:
   - **Unit tests** in `src/native/capture_language/tests/rewrite.rs`. Cover:
     - `@@@` and `@@@\nApply {0}` (text, single edit, cursor 12, summary)
     - CRLF terminator reuse
     - no-cursor, cursor 1, and cursor 2 are unchanged
     - ` @@@`, `\n@@@`, `@@@@`, `@@@ x`, and `x\n@@@` are unchanged
     - rewriting the output again is a no-op
   - **CLI JSON test** in `tests/cli/capture/rewrite.rs`. Assert
     `rule: "start_template"`, `edits`, `cursor`, and `summary`.

## Part 2: bob-mac-capture

Open the linked repo with `sase repo open bob-mac-capture` and read its `AGENTS.md`
before you edit.

1. **Portable helper.** Create `Sources/CaptureCore/CaptureTemplateStart.swift` with a
   `public enum CaptureTemplateStart`:
   - `rewriteRule = "start_template"`.
   - `isRewriteKeystroke(typed:text:selectedRange:) -> Bool`. It is true only when
     `typed == "@"`, the selection is collapsed at UTF-16 location 2, and the first
     physical line of `text` is exactly `@@`. In other words, the key would leave a
     first line of `@@@` with the caret after it.
   - `hintApplies(draft:cursorUTF8Offset:) -> Bool`. It is true when the first line is
     exactly `@@` and the caret is at UTF-8 offset 2.
   - Compare bytes (UTF-8 or NSString), not Swift `Character` prefixes, because `"\r\n"`
     is a single `Character`.
   - Document it as a lexical gate on _when to ask_ bob. It is the same kind of check as
     `isBareAtAtTrigger`; bob alone decides the rewrite.
2. **Keystroke intent.** Extend
   `CapturePanelController.noteSeparatorToggleIntentIfEligible` and
   `CapturePanelModel.noteSeparatorToggleIntent` to record an intent for `@` when
   `isRewriteKeystroke` holds.
   - For `@`, block only Command and Control. Allow Option, because layouts such as
     German type `@` with Option.
   - Keep the existing `:`/`^` behavior and modifier rules byte-for-byte.
   - Keep the public names, because `CaptureSeparatorToggleTests` uses them. Update
     their doc comments to say they also cover the template-start key.
   - The existing `editorTextDidChange` → `consumeSeparatorToggleIntent` →
     `startCaptureRewrite` path with its completion gate then carries the third `@`.
   - Today `isBareAtAtTrigger` already declines `@@@`, so the third `@` makes no rewrite
     call at all.
3. **Native apply.** In `applyCaptureRewrite`, send `start_template` through the same
   single-edit native path as `switch_block_id_separator`. You may rename the private
   `applySeparatorToggleRewrite` to `applyNativeSingleEditRewrite`. This path keeps Undo
   correct: Undo restores the literal `@@@`, Redo replays, and a second Undo restores
   `@@`. It must also never re-trigger, because Undo produces no keystroke intent.
   - After a `start_template` apply, close the `@@` route completion list if it is still
     visible.
   - Request no new completion at the new caret.
   - Let the existing status line announce bob's summary.
   - bob's live preview then shows its own `template_missing_rows` guidance ("write the
     template on the next line, …").
4. **Hints.**
   - **Completion-list header (primary).** Add a computed `templateStartHintVisible` on
     the model. It is true when all of these hold:
     - completion is visible and `fileUnderActive` is false
     - the context is `route`
     - the candidates include one with `kind == "template"`
     - `CaptureTemplateStart.hintApplies(draft: plainDraft, cursorUTF8Offset: collapsedSelectionUTF8Offset())`
       is true

     While it is true, `CompletionList` shows a non-selectable header at the top, using
     the existing "File under" header pattern. The header is a keycap `@` followed by
     "again to start a template", with the accessibility label "Type @ again to start a
     template".
     - Reuse the template shelf's keycap look. If needed, extract `hintKeycap` into a
       small shared view.
     - The header must not change `selectedCompletionIndex` or the scroll anchors.
     - It goes in the header, not on the `@@template:` row, because bob lists the
       template candidate last when the query is empty. With a real route list, that row
       is usually scrolled out of view.

   - **Empty template shelf.** When Control-T opens an empty shelf, both status strings
     in `CapturePanelModel` become: "No saved templates yet — type @@@ to start one;
     capturing it saves it here". Update the assertion in
     `Tests/BobMacCaptureTests/CaptureTemplateShelfModelTests.swift`.
   - **Templates footer button.** The `.help` text becomes "Reuse a saved @@template:
     draft (Control-T). Type @@@ to start a new one."
   - **Settings → Saved Templates.** Append "Type @@@ at the start of a draft to begin
     one." to the caption.

5. **Fake bob.** In `Tests/Fixtures/fake-bob`:
   - Add `capture-rewrite` replies for `@@@` and `@@@\nApply {0}` that match the
     contract above. Use the real bob output once Part 1 builds.
   - Add a `capture-complete` case for the bare `@@`, pointing to a new fixture
     `capture-complete-template-bare.json`. It has cursor 2, replacement `{2,2}`, route
     candidates first, and the template candidate last, which is bob's empty-query
     order.
6. **Tests.**
   - **`Tests/CaptureCoreTests/CaptureTemplateStartTests.swift`.** Truth tables for both
     helpers. Cover:
     - LF, CRLF, emoji, and multi-line drafts
     - caret at 0, 1, and 2
     - a non-collapsed selection
     - `@@x`, ` @@`, and `\n@@`
     - typed values `"@@"` and `"a"`
   - **`Tests/BobMacCaptureTests/CaptureTemplateStartTests.swift`.** Mirror the
     `CaptureSeparatorToggleTests` harness. Cover:
     - typing `@` into `@@` gives `@@template:\n` with caret UTF-16 12, one recorded
       `capture-rewrite` with `--cursor 3`, and completion hidden
     - the `@@@\nApply {0}` variant
     - Undo restores `@@@` without a second rewrite call, Redo replays, and a second
       Undo restores `@@` (use a window-hosted text view, as the separator undo test
       does)
     - the intent declines a caret elsewhere, `@@x`, a Command or Control modifier,
       modal fields, and marked text, but accepts Option-`@`
     - a `changed: false` bob leaves `@@@` literal
     - `templateStartHintVisible` is true for the bare `@@` completion and false for
       `@@t`, for the caret away from offset 2, and for a list without a template
       candidate

## Docs (bob-mac-capture)

- **`docs/keyboard.md`.** Add an `@` row to "Editor and completion keys":
  - **Editor column:** with the caret right after a first line of exactly `@@`, ask bob
    to start a template. `@@@` becomes `@@template:`, the caret moves to a new blank
    line, and Undo restores `@@@`. Everywhere else, `@` inserts normally.
  - **Completion column:** same, and close completion; the list header shows the hint.
  - **Prompt columns:** native behavior.
- **`docs/capture-panel.md`.** In the rewrite-lane paragraph, add the template start and
  "This gesture needs a bob build that implements `start_template`."
- **`docs/capture-previews.md`.** In § Templates, add one sentence on the gesture and
  the list-header hint.
- **`docs/saved-templates.md`.** Add one sentence on how to start a new template.
- **`docs/protocol.md`.** Add `start_template` to the `capture-rewrite` bullet and to
  its compatibility row. With an older bob, the keystroke types a literal `@`.

## Verification

- **bob-cli:** run `just check` (fmt, clippy, all tests). Then run `just docs-gen`,
  `just docs-check`, and `bob capture-rewrite -c 3 -f json -- '@@@'` using the built
  binary.
- **bob-mac-capture on athena:** run `just portable-check`. It lints every Swift file
  and builds and tests `CaptureCore`.
- **bob-mac-capture docs:** run
  `BOB_CLI_DIR=<the bob-cli checkout with Part 1> just docs-check` so that the gate sees
  the updated bob-cli pages.
- **bob-mac-capture on the Mac (best effort, required by its `AGENTS.md`):**
  1. rsync a scratch copy to `mac` (ssh alias), leaving out `.build` and `.git`
  2. run `Scripts/bundle.sh` and `just typecheck-tests`
  3. delete the scratch copy

  Never use Bryan's checkout or `just install` there. Never skip compiling Swift. If
  `mac` is offline, the final report must say the macOS targets are unverified.

- **Commits:** commit both repositories. bob-cli lands the contract and bob-mac-capture
  consumes it.

## Out of scope

- Changing what Return/Tab does on the `@@template:` completion candidate (it still
  inserts `template:` on the same line).
- Leading-whitespace or blank-line variants of the trigger.
- Installing the new `bob` on the Mac.
