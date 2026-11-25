---
tier: tale
title: Complete Bob Mac Capture true-case drawers
goal:
  Render Bob-authored drawer annotations and deliver native drawer typing assists while
  preserving title case and editor behavior.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ph.8
proposed_with: codex/gpt-6-astra
bead: bob-cli-ph.8
create_time: 2026-11-25 18:39:56
status: wip
---

- **PARENT:**
  [202611/true_case_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/true_case_drawers.md)
- **BEAD:**
  [bob-cli-ph.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.8.md)

# Complete Bob Mac Capture true-case drawers

## Outcome and scope

Complete phase **bob-cli-ph.8** of `plan:202611/true_case_drawers.md`. The Mac draft
shows Bob's true-case drawer titles semibold, dims the literal `**` delimiters away from
the caret, and provides the approved star, Return, overtype, and Backspace assists.
Preview headers show an outline `tray` and the exact title Bob returns. Transfer and
merge destinations keep that title's case. JSON compatibility, native editing, and
existing requirement behavior survive.

This is one medium tale: the Rust contract has landed and the remaining work is one
frontend integration with existing decoder, preview, and native rewrite seams. The phase
had no implementation or remaining-work notes when inspected; there is no unfinished
earlier coding attempt to subdivide. Do not repeat the Rust place resolver, writer, or
grammar work.

Implementation belongs in the linked **bob-mac-capture** repository, opened with
`sase repo open bob-mac-capture -r "Implement bob-cli-ph.8"`; use only its returned path
and read its `AGENTS.md`. The only intended primary-repository edit is the Mac behavior
section of `docs/task-drawers.md`. No vault migration, deployment to Bryan's installed
app, plugin changes, release/version edits, or memory changes.

## Governing evidence and decisions

Before implementation, read the assigned phase with
`sase bead read bob-cli-ph.8 -r "Confirm assigned scope and final decisions"`. Read the
parent plan and required research with `sase artifact read`:

- `plan:202611/true_case_drawers.md`, especially §§6–7 and `mac-drawers`.
- `research:202611/true_case_drawers/true_case_drawers.md`.

If the research artifact initially reports missing, open `bob-cli--research` through
`/sase_repo` and repeat the audited artifact read; that resolved the initial stale
checkout during planning. Never read the sidecar file directly.

The epic's final `drawer_glyph = tray` decision means SF Symbol `tray`, never
`tray.fill` or `cabinet`. Memory decisions belong to phase `.11`; they are not
permission for this phase to edit memory.

Governing records, read through `/sase_memory_read`:

- `decisions:mac-capture-is-a-thin-client`: Bob alone owns grammar, eligibility,
  completion replacement ranges, preview data, and writes. Swift interprets validated
  annotations and handles native keystrokes; it does not recognize drawers by Markdown,
  case, reserved-name lists, or indentation rules.
- `decisions:capture-bullets-mirror-the-task-block`: preserve the shared
  draft/preview/commit destination, with the parent epic's explicit replacement of its
  obsolete ALL-CAPS clauses by bold true-case containers.
- `decisions:task-reqs-are-met-with-their-task`: keep requirement chips, requirement
  editing, `req_in_drawer`, and the first-note birth policy intact.
- `decisions:capture-star-blocks-existing-tasks` and
  `decisions:capture-sigils-say-where-what-how`: the new assist must not claim
  multiplication, emphasis, dependency `*`, or existing sigil picker gestures.
- `decisions:notebook-routes-write-notes`: `%n` seed containers differ from a route-only
  notebook list; Bob decides which may expand.

The decisions roster was inspected through `sase memory web show decisions`: the
installed SASE rejected `sase memory read roster:decisions` as an unknown web. The
governing strand bodies were read with the audited memory command.

## What has already landed

At planning time bob-cli HEAD was `6a3e5df6`; prerequisite `bob-cli-ph.6` is closed,
with its landing evidence in note #2 and commit `2d2f200d`. Mac HEAD was `fc0d4551`.
Recheck changes since those revisions before editing.

The existing CLI provides:

- `drawer_marker` spans over each `**` pair; `drawer_bullet` over the title alone. An
  empty `****` task-container draft has two marker spans and a zero-width
  `interactive_placeholder` between them. Use actual replies for every family.
- `capture-rewrite` rule `expand_drawer_marker`: a physical `- *` can produce `- **|**`;
  an auto-paired `- *|*` is also supported. Bob returns exact UTF-8 edits and cursor.
  See `src/native/capture_language/rewrite.rs`, its tests, and `docs/capture.md` under
  drawer marker expansion.
- Optional `drawer` on preview block lines: `{title, key, marker, empty}`. `marker` is
  `bold`, `legacy_caps`, or `section`. Per-item drawer summaries also have additive
  `marker` beside `{title, key, created}`. See `src/native/capture/block_diff.rs`,
  `src/native/capture/output.rs`, and
  `tests/cli/capture/drawer_bullets.rs::drawer_reports_carry_marker_provenance_and_preview_annotations`.
- Bold completion replacements and title-preserving creation. Current capture grammar is
  31, incorporating the drawer contract from grammar 30.

Existing Mac code already preserves menu and sub-bullet capsule titles in
`CapturePlaceMenuPresentation.swift` and `CaptureSubBulletPresentation.swift`. Retain
that work. The remaining `.uppercased()` drawer conversions are `containerLabel` in
`CaptureTransferPresentation.swift` and `CaptureMergeInPresentation.swift`.

## Implementation

### 1. Carry Bob's drawer annotation through the preview

In CaptureCore, introduce a small drawer annotation model for the four wire fields
above. Extend `CapturePomodoroBlockLine` (also `CaptureBlockLine`) in
`CaptureModels.swift` with an optional annotation, preserving constructor defaults,
Codable behavior, and existing callers. Extend `CaptureDrawerSummary` with optional
marker provenance. Missing or malformed additive values must not discard an otherwise
valid capture; unrecognized marker values render conservatively.

Propagate the optional annotation into `CaptureBlockDiffRow`, through
`CaptureTaskBlockPresentation` and every relevant constructor that carries block rows
(including section/transfer/merge paths). Keep original text, tokens, before-text, diff
state, and fold identity unchanged. Never derive an annotation from `**`, uppercase
text, or an unannotated row.

Add the drawer branch in `BlockDiffCard` using a small reusable view/helper if useful.
An annotated header displays outline `tray` plus `annotation.title` semibold, without
raw delimiters. Empty drawers are muted; removed/closed states retain the card's
muted/diff treatment. Use existing Bob-authored owner/theme presentation when available;
fall back to the section palette without deriving task type or status from raw Markdown
or introducing a new wire contract. Keep the raw row available to source/copy/diff
behavior; cosmetic glyphs must never enter the draft or stored text. Accessibility names
it as a drawer and uses the verbatim title; do not announce invented note counts absent
from Bob. Absent/malformed annotations retain the existing verbatim preview.

Remove only the drawer `.uppercased()` conversions in the transfer and merge-in
presentations. Preserve ALL-CAPS task types, Pomodoro names, and identity keys.

### 2. Render the draft from semantic spans

Use the existing string-kind `CaptureSpan` model and validated UTF-8 range conversion;
no grammar parser is needed. In `CompletionRowContent.swift`,
`CaptureEditorPalette.swift`, and `CapturePanelModel`'s highlighting helpers:

- Set `drawer_bullet` title spans semibold in the existing section color family.
- Paint `drawer_marker` delimiters with tertiary label ink; use full normal ink when the
  collapsed caret is on or immediately adjacent to that marker range, including both
  endpoints. The markers always occupy their original characters.
- Repaint on caret movement even when text has not changed. Reuse the existing
  current-parse selection path, and clear every added style on subsequent paints so
  semibold/dim attributes do not leak into ordinary typed text.
- Never conceal marker bytes, substitute attachments in the editor, or change
  attributed-string character counts. Invalid/stale spans leave ordinary text.

Handle Unicode through the existing UTF-8/UTF-16 conversion helpers, including titles
such as `Café`, `Straße`, `iOS / OAuth 2.0`, and `C++ Notes`.

### 3. Add physical-key typing assists using Bob authority

Use `CapturePanelController`'s native event handling, `CapturePanelModel`'s
requirement-marker intent/Backspace patterns, the guarded rewrite lane, and
`CaptureNativeRewriteEdit`. Put reusable span/range calculations in a focused
CaptureCore helper instead of expanding the already-large model unnecessarily.

**Star expansion.** Record an intent for a physical `*` in the main editor, outside
marked text/composition and with a collapsed selection. Ask `capture-rewrite` about the
resulting draft and byte cursor. A cheap local keystroke filter may avoid irrelevant
calls but must not decide eligibility. Apply `expand_drawer_marker` as one native
undoable edit, via the existing `applyNativeSingleEditRewrite` route, preserving Bob's
returned cursor. Undo after expansion restores the typed `- *`; redo restores the
skeleton. Do not trigger on paste, template load, programmatic changes, or undo/redo.

Reuse generation, draft, cursor, and process-client guards so a late rewrite cannot
replace newer input, a moved selection, a closed panel, or another Bob client. Route
rapid following title keys through the existing native input-hold mechanism if
necessary; never lose, reorder, or duplicate those keys. Failure or an unchanged reply
leaves native typing intact and releases any hold.

**Closing-pair overtype.** At the end of a title immediately before a validated closing
`drawer_marker`, a physical `*` advances over both closing stars without inserting text.
It does not remove stars or arm unrelated sigil aliases.

**Smart Return.** With the caret at the end of a nonempty `drawer_bullet` title before
its closing marker, Return passes the pair and starts `\n  - ` at the next authored
level (using the editor's existing list-indent mechanics). Keep existing following
content and established submission/menu key precedence. Ordinary Return elsewhere is
unchanged. An empty skeleton must not bypass Bob's empty-container refusal by
manufacturing a valid drawer.

**Atomic Backspace.** At `**|**`, delete exactly the four stars in one native undoable
edit, preserving the `- ` prefix. Authorize it with matching current marker spans plus
the empty placeholder; just after an accepted Bob expansion, an exact draft/cursor-bound
response record may bridge the asynchronous parse gap. Discard that record on any
incompatible edit, selection, focus/client change, paste, or composition. Existing
hand-typed skeletons work when Bob's current spans authorize them. Nonempty titles and
selected ranges remain native.

Do not add a Swift reserved-title list, container classifier, or speculative `**...**`
regex. The helpers verify the byte shape within Bob-authorized ranges only. Retain
requirement marker expansion/Backspace/overtype and the existing dependency-star picker
behavior.

### 4. Regenerate fixtures and prove the user paths

Update `Scripts/generate-drawer-fixtures.sh` to generate current bold drafts from a
newly built Bob from the known CLI checkout. The present script still feeds ALL-CAPS
multi-note drafts, so fix those inputs rather than hand-editing their resulting JSON.
Preserve representative legacy destination headers in the scratch vault to prove
appending does not recase or rewrite them. Record source revision/version/commands in
`Tests/Fixtures/drawer-fixtures-README.md`; retain its cleared environment, isolated
vault/config/cache, and pinned clock. Require the drawer capability (grammar >= 30)
without assuming it is the newest grammar. If the generator requires a clean CLI tree,
run it before changing the primary docs page. Add real parse and rewrite fixtures where
needed.

Meaningful regression coverage:

- Portable decoding: new/old payloads, missing/malformed/unknown annotation fields,
  `bold`/`legacy_caps`/`section`, empty headers, Unicode. Unannotated bold notes remain
  ordinary rows. Raw row/token bytes and fold IDs are retained.
- Pure presentation: title casing survives menu, capsule, transfer, merge-in, and
  annotated block paths; an existing legacy title remains exactly as sent.
- Editor paints: title-only semibold, dim versus caret-adjacent markers, identical
  text/offsets, malformed ranges, and clearing after leaving a drawer.
- Native hosted edits: expansion, rapid title typing, Return, overtype, Backspace before
  and after parse, undo/redo, astral Unicode before the caret, moved caret/stale
  replies, marked text, paste, template load, and timeout.
- Negative protocol/gesture cases: nested notes, plain/checkbox bullets, route-only
  notebook bodies, reserved/ineligible containers, emphasis, multiplication, and
  dependency `*`. Bob supplies no eligible rewrite/spans; Swift must preserve ordinary
  handling. Include new task, note-less existing task, project-note, `%n` seed, and
  record-group examples from real replies.
- Extend the existing hosted design tests with annotated nonempty/empty drawers and
  mixed-case transfer destinations in light/dark/increased contrast. Check title
  wrapping, caret visibility, and accessible drawer labels. All async test waits must be
  bounded, as required by the Mac `AGENTS.md`.

Existing useful suites include `CaptureRewriteResponseTests`,
`CaptureTaskBlockPresentationTests`, `CaptureTransferPresentationTests`,
`CaptureMergeInPresentationTests`, `CapturePlaceMenuPresentationTests`,
`CaptureSubBulletPresentationTests`, `CaptureEditorHighlightTests`,
`CaptureEditorLeaderTests`, `CaptureRequirementPipelineTests`, and
`TaskBlockDesignTests`. Add focused drawer tests where that is clearer than overloading
unrelated suites. Keep old fixtures testing backward compatibility.

### 5. Document, verify, and finish only this phase

Read `docs.md` through `/sase_memory_read`. Update the Mac's `docs/protocol.md` for
annotation/span/rewrite fields and compatibility behavior, and
`docs/capture-previews.md` for true-case tray previews and typing. Update bob-cli's
`docs/task-drawers.md` Mac section to describe the implemented behavior. Do not update
memory: the parent epic's memory phase owns that work.

Follow the opened Mac `AGENTS.md` in full. Format only changed Swift lines. Compile
every Swift change. On a host without Xcode, run `just portable-check`; if Swift is
absent, read the tailnet reference memory and use a scratch copy on athena. Portable
checking does not compile the AppKit/SwiftUI targets. Best effort compile a scratch copy
on the Mac with `Scripts/bundle.sh` and `just typecheck-tests` (rsync without
`.build`/`.git`, then delete that scratch copy). Never use Bryan's checkout or
`just install`; Command Line Tools lack XCTest, so `--build-tests` is not a substitute
there. Report Mac unavailability honestly, and still require the exact-SHA macOS CI gate
before landing Swift.

Run the Mac `just check-scripts` for fixture-script changes and `just docs-check`. Run
primary `just docs-gen`, `just docs-check`, and `just check`, as required by the parent
plan. Use `/sase_monitor` for long checks and handoffs; wait for each staging command to
exit and never end with a live unmanaged subprocess.

Before landing, run the linked repo's `Scripts/macos-ci-check` through `/sase_monitor`.
It snapshots the worktree onto a disposable branch and watches the exact SHA through
macOS lint/build/test/bundle/install coverage. A successful portable run is not enough.
After any subsequent push, monitor that exact SHA's macOS run to green as well. Record
the verified SHA/run evidence on the phase.

For a failing check, reproduce it on the clean relevant base before calling it
pre-existing. Fix failures caused by this work. A failure identical on the clean base is
recorded with reproduction evidence as
`sase bead note bob-cli-ph.8 'PROPOSED FOLLOW-UP: ...'`, citing an existing task when
known; it does not keep the phase open. Do not create beads yourself.

When the complete scope passes its applicable gates, run
`sase bead epic-symbols bob-cli-ph.8` (none existed at planning time). Resolve any new
entries, or re-key a justified outstanding suppression to a still-open bead. Then close
**only** `bob-cli-ph.8` with `sase bead close bob-cli-ph.8 --note` summarizing the tests
and exact macOS evidence. Never hand-set its status, reopen it, or close `bob-cli-ph` or
another ancestor. Finish with `/sase_final`, declaring every repository changed during
implementation.
