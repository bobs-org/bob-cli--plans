---
tier: tale
title: Record Card Changes skip blank spacer rows
goal:
  A record update's Changes preview shows the note being added instead of blank lines
  around a new `## Notes` heading.
size: small
decisions:
  full_preview:
    ask:
      Should the Preview pane's update diff also hide blank spacer rows, not just the
      Record Card's Changes section?
    default: true
    why:
      Preview shows the same blank rows; create previews already hide blank body lines
    answer: true
  heading_rows:
    ask: Should the compact Changes section keep added heading rows such as `## Notes`?
    choices:
      keep:
        A first note shows two rows, `## Notes` and the bullet, so the new section is
        visible
      drop:
        Card shows only note bullets; a newly created section's heading no longer
        appears
    default: keep
    why: One row names the new section and the bullet still fits in the three-row card
    answer: keep
proposed_by: bbugyi200.apollo.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 13:10:39
status: wip
---

# Show the note, not blank scaffolding, in the Record Card's Changes preview

## Problem

When you add a note to an existing record (for example
`@sase_blog_zettel/sase_is_too_complex` followed by
`- Maybe Fable 5.2 is ready to take on this role (the architect)?`), the Record Card's
**Changes** section shows:

```
+
+ ## Notes
+
+1 more
```

The three visible rows are a blank line, the new `## Notes` heading, and another blank
line. The note itself, which is the only part you care about, is hidden behind
`+1 more`.

## Root cause

- Bob reports `note_files[].lines` for a record update as the verbatim Myers diff of the
  whole note (bob-cli `record_update_lines` in `src/native/capture/record_note.rs`).
  This output is correct: it describes the exact write.
- When the record has no `## Notes` section, Bob's `append_bullets_under_notes` (bob-cli
  `src/native/records/render.rs`) writes four lines: a blank line, `## Notes`, a blank
  line, then the bullet. Notes without that section are now common: H1-less record
  creates omit an empty Notes section, and `bob records tidy` and the nightly run remove
  empty ones. The note in the screenshot ended right after its frontmatter before this
  capture.
- In bob-mac-capture, `RecordCardView.changesSection`
  (`Sources/BobMacCapture/RecordCardView.swift`) shows the first
  `CapturePanelLayout.recordCardChangesMaxRows = 3` rows from
  `CaptureRecordPreviewPresentation.liveUpdateChangedLines`. That function returns
  `CaptureNoteFile.changedLines`, which is every row whose change is not `unchanged`,
  blanks included. The bullet therefore always falls into `+N more`.
- Even when the `## Notes` section already exists, a blank separator row takes the first
  slot. The checked-in fixture `Tests/Fixtures/record-capture-update.json` reports a
  blank added row followed by `- 2026-10-30: Went back`.

The app already sets the precedent for this fix: the create preview's `plannedBody` in
`Sources/BobMacCapture/RecordNotePreviewView.swift` hides blank body lines. The fix is
display-only. The app chooses which of Bob's rows to show and never computes a diff, so
`decisions:mac-capture-is-a-thin-client` still holds.

## Scope

All changes land in the **bob-mac-capture** linked repository. Open it with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`), read its `AGENTS.md`,
and work only in the path it prints. bob-cli needs no change. Bob's JSON, the protocol,
and `note_files[].lines` stay exactly as they are, so older and newer Bob both keep
working.

## Changes

### 1. CaptureCore: a display list without blank spacer rows

In `Sources/CaptureCore/CaptureRecordPreviewPresentation.swift`:

- Add `CaptureNoteFileLine.isBlankSpacer: Bool`. It is true when `change` is `"added"`
  or `"removed"` and `text` trimmed of whitespace and newlines is empty. A `"changed"`
  row never counts as a spacer: the app does not decode `before`, and a changed row can
  be a real edit.
- Add `CaptureNoteFile.previewChangedLines: [CaptureNoteFileLine]?`, placed next to
  `changedLines` and documented:
  - It is `nil` when `changedLines` is `nil`, so the older-Bob fallback is unchanged.
  - Otherwise it is `changedLines` without blank spacers.
  - If dropping the spacers would empty a non-empty list, it returns `changedLines`
    unchanged, so a whitespace-only diff never looks like "no changes".
  - `changedLines` itself keeps its raw meaning.
- Make `liveUpdateChangedLines` return `file.previewChangedLines` (still `nil` when
  empty) and update its doc comment.

> [!decision] heading_rows = drop `liveUpdateChangedLines` also drops `"added"` rows
> whose trimmed text is an ATX heading (one to six `#` characters followed by a space).
> Do this with a separate CaptureCore helper, for example
> `CaptureNoteFile.compactChangedLines`, built on `previewChangedLines`. It uses the
> same never-filter-to-empty fallback. The Preview pane keeps heading rows.

### 2. Record Card (`Sources/BobMacCapture/RecordCardView.swift`)

No logic change should be needed. `policy` (through `changeLineCount`) and
`changesSection` already read `liveChangeLines`, so the height policy, the row count,
`+N more`, and the VoiceOver label `Changes, N lines` all follow the filtered list.
Confirm this. For the screenshot's capture the card should show `+ ## Notes` and
`+ - 2026-11-18: Maybe Fable 5.2 …` with no `+N more` row, and the card should be one
row shorter. Keep `recordCardChangesMaxRows` at 3.

### 3. Preview pane (`Sources/BobMacCapture/RecordNotePreviewView.swift`)

> [!decision] full_preview In `noteBody`, render `file.previewChangedLines ?? []`
> instead of the inline `lines.filter { $0.change != "unchanged" }`. Keep showing **No
> changes** only when the raw changed list is empty; the fallback rule keeps the two
> cases equivalent. The accessibility label joins the same filtered rows. **View
> Markdown** still shows the exact bytes.

> [!decision] full_preview = no Leave `noteBody` untouched.

### 4. Tests (`Tests/CaptureCoreTests/CaptureRecordPreviewTests.swift`)

Build the lines in code, as the existing tests do. Do not regenerate fixtures:
`Scripts/generate-record-fixtures.sh` rewrites every record fixture from a live Bob.

- **Screenshot shape.** Use a few `unchanged` frontmatter rows plus `added` rows `""`,
  `"## Notes"`, `""`, and
  `"- 2026-11-18: Maybe Fable 5.2 is ready to take on this role (the architect)?"`.
  Assert that `previewChangedLines` texts are `["## Notes", "- 2026-11-18: …"]`, and
  that `liveUpdateChangedLines` returns the same rows for a ready, matching update
  preview. Model the preview on `testLiveUpdateChangedLinesRequiresReadyMatchingDraft`.
  Under `heading_rows = drop`, `liveUpdateChangedLines` returns only the bullet.
- **Existing fixture.** For `record-capture-update.json`, assert that
  `previewChangedLines` is `["- 2026-10-30: Went back"]` and that `changedLines` still
  has two rows.
- **Spacer rule.** Added or removed rows of `"   "` or `"\t"` are dropped. A `"changed"`
  row with empty text is kept.
- **Fallback.** A diff made only of blank added rows returns those rows unchanged.
- **Older Bob.** Missing `lines` gives `nil`, and an empty `lines` gives `[]`.
- Existing tests in `CaptureRecordPreviewTests` and
  `Tests/BobMacCaptureTests/CaptureRecordCardDesignTests.swift` must pass unchanged.

### 5. Docs (`docs/capture-previews.md`, the record preview paragraph)

- Replace "renders changed `note_files[].lines` verbatim with `+` / `~` / `−` gutters"
  with wording that says changed lines render with those gutters while blank added or
  removed spacer rows are skipped (adjust for the `full_preview` decision).
- Make the Changes clause read "up to three non-blank lines, then `+N more`". Under
  `heading_rows = drop`, also say that heading rows are left out of the card.
- Keep the **View Markdown** exact-bytes sentence.
- Do not edit bob-cli's mounted copy under `docs/reference/mac-capture/`; its source
  repository owns it.

## Verification

Follow bob-mac-capture's `AGENTS.md`:

- `just portable-check` must pass. It runs CaptureCore tests on Linux. If `swift` is
  missing on this host, run it on athena.
- `RecordCardView.swift` and `RecordNotePreviewView.swift` are in
  `Sources/BobMacCapture`, which portable-check cannot type-check. Compile a scratch
  copy on the Mac (best effort): rsync without `.build`/`.git`, run
  `./Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
  Bryan's checkout or `just install`. If the Mac is offline, say in the final report
  that the macOS targets are unverified.
- Format only the lines you changed (`.swift-format`: 4-space indentation, 100 columns).
- Run `just docs-check` for the docs edit.
- Before landing, run `Scripts/macos-ci-check`. After any push, watch the exact-SHA
  macOS run to green with `/sase_monitor`.

## Out of scope

- Bob's `lines` payload, the record note write format, and the `## Notes`-creation
  spacing stay unchanged.
- The Changes row cap stays at 3.
- No fixture regeneration.
