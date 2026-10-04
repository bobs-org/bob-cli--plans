---
tier: tale
title: Ctrl-J removes a populated dash bullet from anywhere before its body
goal: In Bob Mac Capture, Ctrl-J with a collapsed caret anywhere from line start through
  the first body character of a populated `- ` row deletes the bullet prefix into
  a blank separator and keeps the body, instead of splitting the row into an empty
  bullet plus a new one.
size: small
proposed_by: bbugyi200.athena.0w5
status: done
---

# Ctrl-J removes a populated dash bullet from anywhere before its body

## Goal and scope

In Bob Mac Capture, Ctrl-J should delete a populated `- ` bullet prefix and leave a
blank separator line whenever the collapsed caret is anywhere before the bullet's body
text. That includes the case Bryan hit, with the caret right after the `- ` prefix.
Notation: `|` marks the caret and `\n` marks a real line break.

| Before Ctrl-J         | Expected              | Actual today              |
| --------------------- | --------------------- | ------------------------- |
| `+2\n- \|foo bar baz` | `+2\n\n\|foo bar baz` | `+2\n- \n- \|foo bar baz` |

This is a **tale, size small**: a one-condition change in one pure resolver, plus
regression tests and README wording. One agent can implement it directly.

All changes belong in the linked **bob-mac-capture** repository. Open it with your
`/sase_repo` skill before you read or edit anything:

```sh
sase repo open bob-mac-capture -r 'Implement the approved Ctrl-J body-start bullet removal plan'
```

Use the path that command prints. Every file path below is relative to that checkout.
bob-cli's Rust code, the `bob capture` subprocess/JSON contract, and the vault need no
changes. This is a local text edit in the editor, not capture grammar, so it stays
within the thin-client boundary.

## Root cause

- `CaptureKeyCommandRouter` maps plain Ctrl-J to `.insertBulletNewline`, and
  `CapturePanelController.perform` sends that command only to
  `insertBulletNewlineInEditableTextView`. That method applies whatever
  `CaptureBulletNewlineEditResolver.resolve(in:selectedRange:)` returns, using
  `NSTextView.insertText(_:replacementRange:)`. No other code path handles Ctrl-J in the
  editor.
- `resolve` has three branches, checked in this order:
  1. A marker-only placeholder line (`^\s*[-*+]\s*$`) at any caret position.
  2. Populated dash-prefix removal, via the private
     `removableDashBulletPrefixRange(in:lineStart:caretLocation:)`.
  3. Normal insertion of `<terminator><indent>- ` at the selection.
- Branch 2 accepts the row (leading spaces or tabs, then exactly `- `) but then applies
  `guard caretLocation <= hyphenLocation else { return nil }`. A caret after the hyphen
  or after the space therefore falls through to branch 3, which splits the row into an
  empty `- ` bullet followed by a new `- ` row. That produces exactly the reported
  output.
- That boundary was deliberate. The earlier plan `plan:202609/ctrl_j_bullet_prefix.md`
  (commit `c6273cf`, "feat: remove dash bullet prefixes with ctrl-j") kept `-| child`
  and `- |child` on the insertion path, and
  `testBulletNewlineResolverKeepsPopulatedDashBoundaryExclusionsOnNormalInsertionPath`
  plus the README still assert it. Bryan has now said that this boundary is wrong. This
  plan widens it, and those assertions and docs must change along with the code.

## New behavior

For the populated-dash branch only:

1. **Unchanged row recognition.** The physical line's content must be optional
   horizontal whitespace (ASCII space or tab, in any mix or depth), then a hyphen, then
   exactly one ASCII space, then a remainder with at least one non-whitespace character.
   An all-whitespace remainder is a placeholder and is still handled earlier by
   branch 1. The selection must be collapsed.
2. **New caret boundary.** Let the _body start_ be the offset of the first character
   after the `- ` prefix that is not a space or tab. The branch applies when the caret
   is anywhere from the physical line start through the body start, inclusive. That
   covers:
   - column zero,
   - inside leading indentation,
   - immediately before the hyphen,
   - between the hyphen and its space,
   - immediately after `- `,
   - inside any extra spaces or tabs before the body,
   - immediately before the first body character.

   A caret strictly after the body start, meaning inside or at the end of the body,
   keeps normal insertion.

3. **Unchanged replacement.** Replace the leading whitespace plus `- ` (and nothing
   else) with one terminator from `preferredCaptureLineTerminator(in:)`. Leave the body,
   any extra whitespace after the first space, and the row's trailing terminator in
   place. This matches what the branch does today from column zero. For example,
   `Parent\n|-   child` already becomes `Parent\n\n|  child`.
4. **Unchanged caret result.** The caret is collapsed immediately after the inserted
   terminator, so it sits before the preserved remainder, whichever qualifying position
   it started from. Compute all offsets in UTF-16, as AppKit does.

The placeholder branch, the normal insertion branch (including its zero-or-two-space
indentation copy), and noncollapsed selections stay exactly as they are. Populated `* `
and `+ ` rows, `-child`, a hyphen followed directly by a tab, and hyphens inside prose
still never trigger removal. The README calls out the dash-only scope as deliberate, and
`+N` / `-N` are capture grammar tokens.

### Expected results (now removal)

| Before Ctrl-J                   | After Ctrl-J                    |
| ------------------------------- | ------------------------------- |
| `+2\n- \|foo bar baz`           | `+2\n\n\|foo bar baz`           |
| `Parent\n-\| child`             | `Parent\n\n\|child`             |
| `Parent\n- \|child`             | `Parent\n\n\|child`             |
| `Parent\n  - \|child`           | `Parent\n\n\|child`             |
| `Parent\n\t  - \|child`         | `Parent\n\n\|child`             |
| `- \|child`                     | `\n\|child`                     |
| `Parent\n  - \|child\nNext`     | `Parent\n\n\|child\nNext`       |
| `Parent\r\n  - \|child\r\nNext` | `Parent\r\n\r\n\|child\r\nNext` |
| `Parent\r- \|child`             | `Parent\r\r\|child`             |
| `Prelude 😄\n\t- \|café`        | `Prelude 😄\n\n\|café`          |
| `Parent\n-   \|child`           | `Parent\n\n\|  child`           |
| `Parent\n- \t\|child`           | `Parent\n\n\|\tchild`           |

### Expected results (still normal insertion)

| Before Ctrl-J            | After Ctrl-J                 |
| ------------------------ | ---------------------------- |
| `Parent\n- c\|hild`      | `Parent\n- c\n- \|hild`      |
| `Parent\n  - c\|hild`    | `Parent\n  - c\n  - \|hild`  |
| `Parent\n- child\|`      | `Parent\n- child\n- \|`      |
| `Parent\n* \|child`      | `Parent\n* \n- \|child`      |
| `Parent\n+ \|child`      | `Parent\n+ \n- \|child`      |
| `Parent\n-\|child`       | `Parent\n-\n- \|child`       |
| `Parent\nword - \|child` | `Parent\nword - \n- \|child` |

## Implementation

1. **Resolver** (`Sources/BobMacCapture/CapturePanelController.swift`,
   `removableDashBulletPrefixRange`):
   - Once the `- ` check passes, start a body offset at `hyphenOffset + 2`.
   - Advance it past any spaces and tabs within `nsLine`.
   - Replace the hyphen-location guard with
     `guard caretLocation <= lineStart + bodyOffset`.
   - Keep the returned range as
     `NSRange(location: lineStart, length: hyphenOffset + 2)`.
   - Add a short doc comment stating the caret boundary (line start through body start)
     and that only leading whitespace plus `- ` is removed.

   Leave branch order, the placeholder regex, `supportedAuthoredIndent`, the
   `CaptureBulletNewlineEdit` shape, and `insertBulletNewlineInEditableTextView` as they
   are. Do not move the resolver into CaptureCore.

2. **Tests** (`Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, using the existing
   `applyBulletEdit` helper). Assert the full resulting text and the exact caret
   `NSRange` in every case.
   - Rename `testBulletNewlineResolverRemovesPopulatedDashPrefixAtOrBeforeMarker` to
     `...AtOrBeforeBodyStart`. Extend its caret loop from `0...hyphenOffset` to
     `0...(hyphenOffset + 2)` for every existing indent (`""`, `"  "`, `"\t"`, `"\t  "`,
     `"    "`).
   - Add an exhaustive caret loop for `Parent\n-   child` (offsets `0...4` all yield
     `Parent\n\n  child`) and for `Parent\n- \tchild` (offsets `0...3` all yield
     `Parent\n\n\tchild`). Add a check that offset `5` and offset `4`, respectively,
     fall back to normal insertion.
   - Add a dedicated regression test for Bryan's reported draft: `+2\n- foo bar baz`
     with the caret after `- ` yields `+2\n\nfoo bar baz`, caret at
     `"+2\n\n".utf16.count`.
   - Extend `testBulletNewlineResolverRemovesPopulatedDashPrefixAcrossDocumentShapes`
     and `...WithExistingLineTerminators` with caret-after-prefix variants of the
     first-line, before-next-row, CRLF, CR-only, and non-BMP/Unicode rows from the
     removal table.
   - Rewrite
     `testBulletNewlineResolverKeepsPopulatedDashBoundaryExclusionsOnNormalInsertionPath`
     as `...KeepsInsertionInsidePopulatedDashBody`. Drop the old `afterHyphen` and
     `afterPrefix` expectations, because those now remove the prefix. Keep
     `middleOfBody`, and add the first-body-character+1 cases (top-level and nested) and
     the end-of-body case from the insertion table.
   - Extend `testBulletNewlineResolverLeavesNonDashPrefixesOnNormalInsertionPath` with
     caret-after-prefix cases for `* child`, `+ child`, `-child`, and `word - child`.
   - Add one `@MainActor` native integration test that runs the reported draft through
     `CapturePanelController.insertBulletNewlineInEditableTextView`. It must assert the
     resulting text, the caret, and that an open completion is dismissed.
   - Leave the placeholder, selection-replacement, responder-rejection, and key-router
     tests unchanged. They must still pass.
3. **README** (`README.md`):
   - Keyboard table, Ctrl-J row: change "at or before a populated dash bullet's marker"
     to wording that says the removal applies anywhere before the populated dash
     bullet's body.
   - Multiline-editing paragraph that begins "Ctrl-J starts the next canonical `- `
     row": describe the new boundary (column zero, leading whitespace, either side of
     the hyphen, after `- `, through the first body character). Keep the
     `Parent\n|- child` example and add `Parent\n- |child` → `Parent\n\n|child`.
   - Replace the sentence "A caret after the hyphen, after the space, or inside the body
     keeps the normal new-row insertion behavior" with one saying that only a caret
     inside or after the body keeps new-row insertion.
   - Leave the picker-navigation Ctrl-J mentions alone.

## Validation

The AppKit editor target and its tests build only on macOS 26+ (`Package.swift` gates
them behind `#if os(macOS)`). Linux `swift test` covers only CaptureCore and does not
exercise this change, so never cite Linux results as validation of it.

On a macOS host with Xcode 26+ or Command Line Tools 26+, run these from the checkout:

```sh
./Scripts/xcode-swift.sh test --filter 'BobMacCaptureTests.BobMacCaptureTests/test.*BulletNewline'
just format-lint
just build
just test
```

If no macOS host is reachable, say so explicitly in the final report. The repository's
GitHub Actions `CI` workflow (`macos-26`) runs format-lint, build, and test on every
push to `master`. After your commit lands, check that run (for example with
`gh run list --repo bobs-org/bob-mac-capture --limit 3`) and report its result, or
report that it is still pending.

If a running app is available, smoke-test Ctrl-J on `- foo bar baz` at:

- column zero,
- just before the hyphen,
- between the hyphen and the space,
- just after `- `,
- one character into the body.

Only the last one should insert a new row. Also confirm that a single native Undo
restores the removed prefix, and that an empty `- ` placeholder still collapses to one
separator.

## Done when

- The reported draft and every row of the removal table produce the specified text and
  caret.
- Every row of the insertion table, and every existing placeholder and selection
  behavior, is unchanged.
- The README describes the new boundary.
- The diff touches only the resolver, its tests, and `README.md` in bob-mac-capture.
- macOS build, test, and format results are reported, or clearly marked as pending on a
  capable host or CI.
