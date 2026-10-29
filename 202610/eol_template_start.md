---
tier: tale
title: Start a @@template draft with @@@ at the end of any line
goal:
  Typing @@@ at the end of any capture line erases it and adds @@template as the draft's
  first line, in bob and the Mac app.
size: medium
decisions:
  after_absorption:
    ask:
      Should a third @ still start the template when bob has already absorbed the line's
      @route into @@route?
    choices:
      recover:
        Ask bob to start it from the pre-absorption line; `Buy milk @dev @@@` works at
        any typing speed
      literal:
        The third @ types literally (`Buy milk @@dev@`); docs say to type @@@ before the
        route
    default: recover
    why:
      Otherwise the gesture fails on lines with a plain @route unless typed faster than
      bob answers
    answer: recover
  separator_space:
    ask: Should the gesture also erase the spaces or tabs typed just before @@@?
    choices:
      strip: "`Read {0} @@@` leaves `Read {0}` with the caret at its end"
      keep:
        Only the three @ go; `Read {0} ` keeps its trailing space and the caret after it
    default: strip
    why:
      The line reads as it did before the gesture, like @@ absorption's whitespace tidy
    answer: strip
proposed_by: bbugyi200.apollo.b4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 16:55:08
status: wip
---

# Plan: Start a `@@template:` draft by typing `@@@` at the end of any line

## Goal

Today, Bob Mac Capture starts a template only when `@@@` is the whole first line (`@@@`
→ `@@template:` and a fresh line below). Add the same gesture at the **end of any
line**. When the caret sits at the end of a physical line, right after a standalone
`@@@` token that the user just typed, bob does two things:

1. It erases that `@@@` (and, per `separator_space`, the spaces or tabs before it).
2. It inserts `@@template:` plus a line ending as the draft's new first line.

The caret stays where the user was typing, mapped through both edits. The typical use: a
user writes `Read {0} @reading`, realizes it should be a template, and types ` @@@` at
the end of the line. The line becomes the template line under a new header.

The work spans two repos. Following the original feature's precedent (bob-cli `0e9f915a`
plus bob-mac-capture `ad85b85`, landed together by one agent), one agent lands both:

- **bob-cli** (this repo) owns the grammar. `bob capture-rewrite`'s `start_template`
  rule decides whether and how the draft changes.
- **bob-mac-capture** (linked repo; open it with `/sase_repo` →
  `sase repo open bob-mac-capture`, and read its `AGENTS.md` first) is a thin client
  (see `decisions:mac-capture-is-a-thin-client`). It only pre-filters the keystroke,
  asks bob, and applies bob's edits natively so Undo stays AppKit-owned.

## Behavior Contract

`start_template` fires only with `--cursor`. All of these must hold:

- The draft has no `@@template:` header. The existing early return in `rewrite_draft`
  stays.
- The whitespace-delimited token that ends exactly at the cursor (bob's
  `tokenize_with_spans` boundaries) is exactly `@@@`.
- The cursor is at the end of its physical line, using `split_physical_lines` semantics.
  The next byte is end-of-text, `\n`, or `\r`.

The result:

- **Erase range:** the `@@@` token. Under `separator_space = strip`, extend the range
  left over the run of spaces and tabs immediately before it on the same line.
- **Header:** `@@template:` followed by the draft's first line break, reused exactly
  (`\r\n`, `\n`, or `\r`), or `\n` when the draft has no line break.
- **Edits:** if the erase range starts at byte 0, emit one merged edit
  `{start: 0, end: cursor, replacement: header}`. This keeps today's JSON for `@@@` on
  line 1 byte-for-byte. Otherwise emit two sorted edits: `{0, 0, header}` and
  `{erase_start, cursor, ""}`.
- **Cursor:** `erase_start + header.len()`.
- **Summary:** the existing `"Started a @@template: draft"`. The `rule` code stays
  `start_template`, so no new rule and no schema bump. This is additive, not a contract
  break: older clients never send these drafts. Use no `!` in the commit subject.

This formula reproduces the current first-line behavior exactly, including the CRLF
case. Replace the special-case `start_template(raw_text)` with
`start_template(raw_text, cursor)`. Do not run two rules.

Worked examples (default `strip`; under `keep`, the erase range is only the `@@@` token,
so the cursor and text keep the separator):

| Input (cursor)                | Text                              | Cursor | Edits                                   |
| ----------------------------- | --------------------------------- | ------ | --------------------------------------- |
| `@@@` (3)                     | `@@template:\n`                   | 12     | `[{0,3,"@@template:\n"}]` (unchanged)   |
| `@@@\nApply {0}` (3)          | `@@template:\n\nApply {0}`        | 12     | `[{0,3,"@@template:\n"}]` (unchanged)   |
| `@@@\r\nApply {0}` (3)        | `@@template:\r\n\r\nApply {0}`    | 13     | `[{0,3,"@@template:\r\n"}]` (unchanged) |
| `Read {0} @@@` (12)           | `@@template:\nRead {0}`           | 20     | `[{0,0,"@@template:\n"},{8,12,""}]`     |
| `Buy milk\nRead {0} @@@` (21) | `@@template:\nBuy milk\nRead {0}` | 29     | `[{0,0,hdr},{17,21,""}]`                |
| `Buy milk @dev @@@` (17)      | `@@template:\nBuy milk @dev`      | 25     | `[{0,0,hdr},{13,17,""}]`                |
| `Buy\r\nRead\t@@@` (13)       | `@@template:\r\nBuy\r\nRead`      | 22     | `[{0,0,"@@template:\r\n"},{9,13,""}]`   |
| `x\n@@@` (5)                  | `@@template:\nx\n`                | 14     | `[{0,0,hdr},{2,5,""}]`                  |
| `  @@@\nApply {0}` (5)        | `@@template:\n\nApply {0}`        | 12     | `[{0,5,"@@template:\n"}]`               |

These inputs never fire: no cursor; the cursor inside the token (`@@@` at 1 or 2);
`@@@@`; `x@@@`; `@@@ x` at 3; `Read @@@ ` at 8, where the caret sits before trailing
whitespace; a cursor past the end or off a char boundary, which must not panic; and any
draft that already has a header.

Choices made here, not open for review:

- The gesture is purely lexical on any line, including child bullets, lines after a
  `@@route` declaration, and blank-line-separated items. bob's existing template
  diagnostics then explain anything the new template cannot hold, such as
  `template_global_declaration` or `template_needs_separator`.
- The caret stays on the edited line. It does not jump to the header.
- The header goes at byte 0, even above leading blank lines. Template analysis keys on
  the first nonblank line, so this is valid.
- The Mac "type @ again to start a template" list hint stays first-line-only. bob offers
  the `template:` completion candidate only for a partial first-line header, and a hint
  on every end-of-line `@@` would crowd ordinary `@@route` typing.

## bob-cli Changes

1. `src/native/capture_language/rewrite.rs`
   - Replace `if let Some(3) = cursor && let Some(rewrite) = start_template(raw_text)`
     with a cursor-taking `start_template(raw_text, position)` that implements the
     contract above. Guard `position <= raw_text.len()` and
     `raw_text.is_char_boundary(position)` first.
   - Find the line with `split_physical_lines` from `super::draft`, and the token
     boundary with `tokenize_with_spans` or an equivalent check that matches
     `char::is_whitespace`.
   - Keep it ahead of the separator-toggle, relation-attach, and absorption rules. That
     way `Buy milk @dev @@@` starts a template rather than being read as absorption.
   - Update the `rewrite_draft` and `start_template` doc comments.
2. `src/native/capture_language/tests/rewrite.rs`
   - `("\n@@@", Some(4))` and `("x\n@@@", Some(5))` in
     `rewrite_draft_template_start_requires_the_exact_typing_gesture` now start
     templates. Move them into positive tests.
   - Add the table above as positive cases, asserting text, cursor, the exact `edits`,
     and the summary. Add the new negative cases. Add an idempotence check for an
     end-of-line case: feeding the output back is a no-op.
   - Add a test that applying the edits left-to-right reproduces `text`.
3. `src/native/capture_rewrite.rs`
   - Rewrite the "Template start (requires --cursor)" `long_about` paragraph to describe
     the general rule. Cover the token ending at the cursor and the end of the line,
     erasing `@@@` (plus separating whitespace per the decision), the header as the
     first line, reuse of the first line break, the caret where `@@@` was ("so a first
     line that is exactly '@@@' becomes the header with the caret on a fresh line"),
     idempotence, and skipped without `--cursor`. It still "implements four rules".
   - Add the after-help example `bob capture-rewrite -c 12 -f json -- 'Read {0} @@@'`.
   - Run `just docs-gen` to regenerate `docs/reference/cli/capture-rewrite.md`. Never
     hand-edit the generated file.
4. `tests/cli/capture/rewrite.rs`
   - Add `capture_rewrite_json_starts_a_template_at_a_line_end`, the CLI twin of
     `capture_rewrite_json_starts_a_template`, for `Read {0} @@@` at cursor 12 (text,
     changed, cursor, rule, both edits, summary).
5. `docs/capture.md`
   - In **Bulk templates**, extend the "Editors that run `capture-rewrite` … turn a
     typed `@@@` at the very start of a draft …" sentence with the end-of-line form. The
     point to make: a line you already wrote becomes the template.
   - In the `capture-rewrite` JSON section, state that the `start_template` cursor lands
     where the erased `@@@` began, after the inserted header. That is the fresh line
     when `@@@` was the whole first line. Also state that `start_template` returns one
     edit when the erasure starts at byte 0, and otherwise two: the header insertion at
     0, then the erasure.
6. Do not touch `Cargo.toml`'s version or `CHANGELOG.md`
   (`decisions:bob-cli-releases-ride-a-daily-train`). Suggested subject:
   `feat(capture): start a template from @@@ at the end of any line`.

## bob-mac-capture Changes

1. `Sources/CaptureCore/CaptureTemplateStart.swift`: widen `isRewriteKeystroke`. It
   stays a lexical prefilter only, and bob decides. All of these must hold:
   - `typed == "@"` with a collapsed selection at `loc >= 2`.
   - The two UTF-16 units before `loc` are both `@`.
   - The unit before them is absent (start of text) or whitespace or a newline
     (`CharacterSet.whitespacesAndNewlines`).
   - `loc` is the end of a physical line: end of text, `\n`, or `\r`.

   The old first-line case is a subset, so it keeps working. Leave `hintApplies`
   first-line-only and update both doc comments.

2. `Sources/CaptureCore/CaptureModels.swift` (or a small new CaptureCore file): add a
   pure helper. A suggested shape is
   `CaptureRewriteResponse.coveringEdit(in draft:) -> CaptureRewriteEdit?`. It collapses
   bob's sorted, non-overlapping edits into one covering replacement,
   `[first.start, last.end)`, whose replacement is the matching slice of `text`. It
   returns nil unless all of these hold:
   - The edits are non-empty and sorted.
   - No two edits overlap.
   - Every range is a valid UTF-8 boundary in `draft`.
   - Applying the edits left-to-right to `draft` yields `text`.

   A single edit comes back unchanged.

3. `Sources/BobMacCapture/CapturePanelModel.swift`, `applyNativeSingleEditRewrite`:
   replace the `edits.count == 1` requirement with `response.coveringEdit(in: draft)`.
   The covering edit becomes one `insertText` in `NSTextView`. That gives exactly one
   Undo step: Undo restores the literal post-key draft (`Read {0} @@@`), and Redo
   replays the rewrite.
   - Consider renaming the function to `applyNativeRewrite`.
   - Update the `CaptureNativeRewriteEdit` and `applySeparatorToggleInEditableTextView`
     doc comments in `CapturePanelController.swift` ("single-edit" → "one covering
     native replacement").
   - The existing `start_template` branch already dismisses route completion and
     announces bob's summary.

> [!decision] after_absorption = recover This is needed because typing the second `@` of
> ` @@` calls `capture-rewrite` immediately (`isBareAtAtTrigger`). On a line with one
> absorbable `@route` or `@route+id`, or a draft with one other `@@x` declaration, bob
> rewrites `Buy milk @dev @@` → `Buy milk @@dev` before the third `@` arrives.
>
> 1. Record provenance whenever `applyCaptureRewrite` applies an `absorb_local_marker` /
>    `absorb_declaration` result through `setPlainDraft`. The record holds the input
>    draft, input cursor (UTF-8), output draft, and output cursor. Clear it when the
>    visible draft stops equaling the output: in `editorTextDidChange` and in
>    `setPlainDraft` for any other text.
> 2. In `noteSeparatorToggleIntent`, when the normal gate fails for `@`, check three
>    things: the text equals the provenance output, the caret's UTF-8 offset equals the
>    output cursor, and `isRewriteKeystroke` passes on the provenance input at its input
>    caret. If all hold, record the usual intent (the expected post-key draft and caret)
>    plus a rewrite source: the input with `@` inserted at the input cursor, and the
>    input cursor + 1. One way is an optional field on `SeparatorToggleIntent`.
> 3. In `editorTextDidChange`, when the consumed intent carries a source, call bob with
>    the source draft and cursor instead of the visible one. Apply the result only if
>    every guard passes:
>    - The generation and `processClient` identity still match.
>    - `plainDraft` still equals the visible post-key draft (`Buy milk @@dev@`).
>    - The caret is unchanged.
>    - `response.input` equals the source.
>    - The response is `changed` with `rule == start_template` and a valid cursor.
>
>    Then apply one native whole-draft replacement (visible draft → `response.text`,
>    caret at `response.cursor`). Share the post-apply tail with
>    `applyNativeSingleEditRewrite` (sync `attributedDraft`, restore the selection,
>    dismiss route completion, announce the summary, schedule analysis, reset
>    `rewriteExpectedCursorUTF8`). Any other reply leaves the literal `@` and stays
>    silent.
>
> Result: `@@template:\nBuy milk @dev`, caret 25. Undo restores `Buy milk @@dev@`. When
> the third `@` arrives before the absorption reply, today's generation guard already
> drops the stale absorption, and the normal path handles `Buy milk @dev @@@`.

> [!decision] after_absorption = literal Add no provenance. Document in
> `docs/keyboard.md` and `docs/capture-panel.md` that when bob has already moved a
> line's `@route` into `@@route`, the third `@` types literally. Type `@@@` before the
> route marker, or start the template from the first line.

4. `Tests/Fixtures/fake-bob`: add `capture-rewrite` replies matching the bob examples
   above (values per `separator_space`) for `Read {0} @@@`, `Buy milk\nRead {0} @@@`,
   and, under `recover`, `Buy milk @dev @@@`. The existing `Buy milk @dev @@` absorption
   reply stays.
5. Tests:
   - `Tests/CaptureCoreTests/CaptureTemplateStartTests.swift`:
     - Accepted: the existing cases plus `Read {0} @@` at 11, `Buy milk\nRead {0} @@` at
       20, `x\n@@` at 4, ` @@` at 3, `Read\t@@\r\nmore` at 7, `Read @@\nmore` at 7, and
       `🧪 @@` at 5 (UTF-16 offsets).
     - Rejected: every existing reject plus `x@@` at 3, `@@@` at 3, `Read @@ ` at 7,
       `Read @@x` at 7, `Read @@` at 6, and a non-collapsed selection.
     - Keep the `hintApplies` tests, and add one showing it still rejects `x\n@@` at 4.
   - A new CaptureCore test for `coveringEdit`: one edit unchanged; the two-edit
     header-and-erase case; multibyte text (emoji before the erased range); and nil for
     unsorted, overlapping, out-of-range, or non-reproducing edits.
   - `Tests/BobMacCaptureTests/CaptureTemplateStartTests.swift`:
     - A third `@` at the end of `Read {0} @@` (single line) and of the last line of
       `Buy milk\nRead {0} @@`. Assert the text view and `plainDraft`, the caret
       (`textView.selectedRange()` and `collapsedSelectionUTF8Offset()`), the status
       `"Started a @@template: draft"`, the recorded argv
       (`capture-rewrite --cursor 12 --format json -- Read {0} @@@`), and that route
       completion is closed.
     - An Undo/Redo test proving one Undo restores `Read {0} @@@` and Redo replays the
       template. Note: a second Undo lands on a bare `@@`, which today legitimately
       issues an absorption request, so do not assert a total of one rewrite call after
       it.
     - Under `recover`, an absorption-then-`@` test. Type the second `@` on
       `Buy milk @dev @` and wait for `Buy milk @@dev`. Sync the standalone text view to
       the model's draft and caret, because absorption goes through `setPlainDraft`.
       Then type `@`. Assert the recorded `--cursor 17 … -- Buy milk @dev @@@` call, the
       resulting `@@template:\nBuy milk @dev` with caret 25, and that Undo restores
       `Buy milk @@dev@`. Add a negative test: after absorption, move the caret or edit,
       then type `@`, and confirm no recovered call is made.
   - `CapturePanelModelTests.testSingleAtAndTripleAtDoNotTriggerRewrite` sets the draft
     programmatically with no key intent, so it must still pass unchanged.
6. Copy and docs. Format only the lines you change:
   - `docs/keyboard.md` `@` row: describe both forms. A first line that is exactly `@@`
     becomes `@@template:` with the caret on a new line. At the end of any other line, a
     standalone `@@` plus `@` is erased and `@@template:` becomes the first line, with
     the caret staying put. Undo restores `@@@`. The list-header hint remains
     first-line-only.
   - `docs/capture-panel.md` (the "Typing a third `@` immediately after a first physical
     line…" paragraph): generalize it the same way, plus the `after_absorption`
     behavior.
   - `docs/saved-templates.md`: "To start one, type `@@@` at the very start of a draft"
     → also "or at the end of any line".
   - `docs/protocol.md` compatibility row: keep "with an older bob, the third `@` types
     literally", which is now true for both forms. Say `@@@` at a line end too.
   - `Sources/BobMacCapture/SettingsView.swift` line ~276: "Type @@@ at the start of a
     draft or the end of any line to begin one."
7. Suggested subject: `feat(capture): start a template with @@@ at the end of any line`.

## Verification

bob-cli (apollo or athena):

- `cargo test capture_language::tests::rewrite`, `cargo test --test cli capture`, then
  `just all` (fmt, lint, test).
- `just docs-gen`, then `git diff docs/reference/cli/capture-rewrite.md` (it must show
  only the new paragraph and example), then `just docs-check`.
- Smoke test: `cargo run -q -- capture-rewrite -c 12 -f json -- 'Read {0} @@@'` and
  `cargo run -q -- capture-rewrite -c 3 -f json -- '@@@'`. The second must match today's
  output.

bob-mac-capture (follow its `AGENTS.md`; never land uncompiled Swift):

- apollo has no `swift`, so run `just portable-check` on athena. That covers
  swift-format lint plus the CaptureCore tests, including the new gate and
  `coveringEdit` tests.
- On the mac (best effort, when online): rsync a scratch copy without `.build`/`.git`,
  run `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
  Bryan's checkout or `just install`. If the mac is offline, say in the final report
  that the `BobMacCapture*` targets are unverified. CI runs the XCTest suite.
- `just docs-check` in bob-mac-capture, which runs bob-cli's documentation gate over the
  mounted pages.

## Out of Scope

- Any new `rule` code, schema bump, or JSON field.
- Changing absorption itself, or the first-line hint.
- bob-plugins, which does not call `capture-rewrite`.
- Moving the caret to the header, or adding the blank separator and rows area
  automatically.
