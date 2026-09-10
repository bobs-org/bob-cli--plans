---
tier: tale
title: Remove the leading bullet prefix with Ctrl-J at or before its hyphen
goal:
  Ctrl-J turns a populated dash bullet into a separated body line when the caret is at
  or before its marker, preserving the body and existing editor behavior.
size: small
proposed_by: bbugyi200.athena.0iz
create_time: 2026-09-10 15:49:50
status: wip
---

# Ctrl-J removes a bullet prefix when the caret is at or before its marker

## Goal and scope

In Bob Mac Capture, extend Ctrl-J so a collapsed caret at or before the hyphen of a
line-leading `- ` bullet replaces that prefix and all its leading whitespace with a line
terminator. Preserve the bullet body and place the caret at the start of the resulting
body line. For example, `Parent\n|- child` becomes `Parent\n\n|child`. Here and below,
`|` denotes the caret and escaped newlines denote actual line breaks.

Use a **tale, size small**: this is one focused editor change in an existing pure
resolver, with regression tests and a README update. One implementation agent can
complete it directly. This planning work is large under the SASE authoring rules; the
follow-up implementation is small.

All implementation changes belong in the linked **bob-mac-capture** repository. Before
reading or editing it, use `/sase_repo` and run:

```sh
sase repo open bob-mac-capture -r 'Implement the approved Ctrl-J bullet-prefix plan'
```

Use the returned checkout and follow any applicable repository instructions. Every file
path below is relative to that repository. The capture subprocess/JSON contract and
bob-cli's Rust implementation require no changes.

## Current behavior and cause

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` maps exactly Ctrl-J to
  `.insertBulletNewline`. Ctrl-Shift-J is a separate vertical-navigation command.
- `CaptureBulletNewlineEditResolver.resolve(in:selectedRange:)` near the top of
  `Sources/BobMacCapture/CapturePanelController.swift` computes physical-line bounds
  with `NSString.getLineStart` and returns a UTF-16 `NSRange` edit and caret range.
- Its existing placeholder branch already applies at **any collapsed caret position** on
  a line containing only whitespace and one `-`, `*`, or `+` marker. It replaces that
  row with one terminator, reusing the row's terminator when present. Thus `Parent\n- `
  becomes `Parent\n\n`; it does not insert two additional terminators.
- A bullet containing body text misses that branch and reaches the normal insertion
  fallback. At `Parent\n|- child`, that fallback inserts `\n- ` before the old marker,
  producing `Parent\n\n- |- child`. The missing operation is prefix replacement on a
  populated bullet, not caret-position support on an empty placeholder.
- `CapturePanelController.insertBulletNewlineInEditableTextView` dismisses completion,
  applies the returned edit with `NSTextView.insertText(_:replacementRange:)`, and sets
  the caret. Keep using that native edit path.
- Existing coverage lives in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`,
  including `testBulletNewlineResolver...`, `testInsertBulletNewline...`, and the
  `applyBulletEdit` helper. The README describes Ctrl-J in its keyboard table and
  multiline editing discussion.

## Exact behavior

Interpret “at or before `- `” as a caret on the current physical line at an offset less
than or equal to the hyphen's offset. This includes column zero and positions inside
leading indentation. A caret between the hyphen and its space, immediately after the
prefix, or inside the body retains normal bullet insertion for populated rows. The
existing placeholder rule continues to apply anywhere on a marker-only row.

For the new branch:

1. Require a collapsed selection and a current line beginning with optional horizontal
   whitespace followed by the exact ASCII prefix `- `. Accept arbitrary leading
   whitespace, including tabs, mixed indentation, and more than two spaces; the normal
   insertion fallback's zero-or-two-space indentation rule does not limit removal.
2. Require the caret to be at or before that hyphen. Match only within the physical
   line's content; never consume a preceding newline or whitespace from another row.
3. Replace from the physical line start through the single space in `- ` with one
   terminator chosen by the existing `preferredCaptureLineTerminator(in:)` helper. Do
   not remove any body text, additional whitespace after that first space, or the body's
   existing trailing line terminator.
4. Set a collapsed caret immediately after the inserted terminator, before the preserved
   remainder. Derive all offsets and lengths in UTF-16, matching AppKit.

The newline already before a continuation row combines with the inserted terminator to
create the blank separator. At the first physical line there is no preceding newline, so
replacement creates one leading blank line. Preserve existing neighboring blank lines
without normalizing them.

| Before Ctrl-J                   | After Ctrl-J                    |
| ------------------------------- | ------------------------------- |
| `Parent\n\|- child`             | `Parent\n\n\|child`             |
| `Parent\n\|  - child`           | `Parent\n\n\|child`             |
| `Parent\n \| - child`           | `Parent\n\n\|child`             |
| `Parent\n  \|- child\nNext`     | `Parent\n\n\|child\nNext`       |
| `Parent\n\|\t  - child`         | `Parent\n\n\|child`             |
| `\|- child`                     | `\n\|child`                     |
| `Parent\r\n  \|- child\r\nNext` | `Parent\r\n\r\n\|child\r\nNext` |

The new populated-row behavior targets `- ` as requested. Keep existing marker-only
handling for `-`, `*`, and `+` unchanged. Nonempty `* ` and `+ ` rows, `-child`, bare
hyphens within prose, and other nonmatching rows keep their current insertion behavior.
Noncollapsed selections also keep the current selected-text replacement behavior.

## Implementation

1. Extend `CaptureBulletNewlineEditResolver` in `CapturePanelController.swift` with a
   narrowly scoped branch **after** the existing placeholder branch and **before** the
   normal insertion fallback. Detect the leading horizontal whitespace and `- ` using
   UTF-16-compatible ranges on the already bounded `lineContent`. Compare the caret with
   the absolute hyphen location and return one `CaptureBulletNewlineEdit` covering
   exactly the prefix. A small private helper is appropriate if it clarifies prefix
   discovery; avoid restructuring unrelated editor commands or moving the resolver into
   CaptureCore just to test it on Linux.
2. Add regression coverage to `BobMacCaptureTests.swift`, using `applyBulletEdit` and
   table-driven cases where useful. Assert both the full resulting draft and the exact
   caret range. Add one native text-view integration test for the new branch.
3. Update the README keyboard-table Ctrl-J entry and multiline editing paragraph to
   explain populated-bullet prefix removal, its caret boundary, body preservation, and
   the existing marker-only behavior. Include a concrete before/after example.

## Regression coverage

- On a populated `- child` row, exercise every caret position from physical line start
  through the hyphen for no indentation, two spaces, tabs, mixed whitespace, and deeper
  indentation. All qualifying positions must yield the same edit.
- Cover a populated bullet on the first line, at EOF, and before another row. Preserve
  neighboring rows and existing blank lines. With `-   child`, preserve the two spaces
  after the consumed `- ` prefix.
- Cover LF, CRLF, and CR-only drafts, including a final populated row without a trailing
  terminator. Include a non-BMP character before the target row and Unicode body text to
  catch UTF-16 caret/range mistakes.
- Assert boundary exclusions on populated rows: `Parent\n-| child` must still become
  `Parent\n-\n- | child`, and `Parent\n- |child` must still become
  `Parent\n- \n- |child`. Also retain the normal middle-of-body insertion behavior.
- Verify that nonempty alternate markers, inline hyphens, and `-child` do not trigger
  removal. Keep noncollapsed selection replacement, including a selection beginning
  before the marker and a multiline selection, unchanged.
- Expand existing placeholder tests to cover all caret offsets on `- ` and an indented
  placeholder, preserving the existing `*`/`+`, trailing-whitespace, EOF, and
  line-terminator cases. This explicitly protects behavior already present.
- Through `insertBulletNewlineInEditableTextView`, confirm a populated indented prefix
  is replaced, the caret lands correctly, and an open completion is dismissed. Existing
  responder-rejection and key-router tests should continue to pass.

## Validation and completion

The editor target and its tests are macOS-only under `Package.swift`. Linux Swift tests
exercise CaptureCore and cannot validate this change. The planning host is Linux, so
report any unavailable macOS validation explicitly; do not claim Linux-only results
cover the AppKit resolver.

On a macOS 26 host with compatible Apple developer tools, run from the linked checkout:

```sh
./Scripts/xcode-swift.sh test --filter 'BobMacCaptureTests.BobMacCaptureTests/test.*BulletNewline'
just format-lint
just build
just test
```

Use the repository's Xcode wrapper rather than a potentially shadowed Swift executable.
The existing macOS CI runs these checks as well. If only Linux is available, leave the
macOS checks explicitly pending for a capable host or CI; avoid refactoring production
code merely to bypass the platform requirement.

When a running macOS app is available, smoke-check Ctrl-J at column zero, within
indentation, immediately before a populated bullet's hyphen, and after its prefix; also
check an empty placeholder and completion dismissal. Confirm a single native Undo
restores the text changed by the prefix replacement.

Completion requires the specified text/caret behavior, preserved existing editor
semantics, updated README, and the macOS build/test/format results or a clear account of
checks still unavailable. Review the final diff to keep it limited to the resolver, its
tests, and the README. No implementation files are changed during this planning turn;
validate this plan with `--explain`, revalidate without it until successful, and submit
with `sase plan propose` for the requested approval handoff.
