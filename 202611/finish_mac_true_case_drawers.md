---
tier: epic
title: Finish Bob Mac Capture true-case drawers
goal: Bob Mac Capture builds green again, draws Bob-annotated drawer headers as a
  tray plus the verbatim title, paints draft drawer titles and delimiters from Bob's
  spans, and types `*`, Backspace, newline, and overtype assists on Bob's authority,
  completing bob-cli-ph.8.
decisions:
  drawer_newline_keys:
    ask: Should plain Return at a childless drawer title open its nested note instead
      of capturing?
    choices:
      newline_keys: Shift-Return, Option-Return, and Ctrl-J nest at a title end; Return
        always captures
      return_too: Return also nests where Bob refuses the drawer as empty_drawer_bullet
    default: newline_keys
    why: Return stays the panel's single capture key, and Bob's refusal already teaches
      the user to nest
    answer: newline_keys
phases:
- id: annotation-build-repair
  title: Repair the drawer annotation build and cover the landed decoding
  size: small
  depends_on: []
  description: 'annotation-build-repair: fix the CaptureDrawerSummary marker decode
    that broke Mac master at b45b1951, compile every target, and add portable coverage
    for the drawer annotation decoding and verbatim destination labels already landed.'
- id: preview-tray
  title: Tray and verbatim title on annotated preview drawer headers
  size: medium
  depends_on:
  - annotation-build-repair
  description: 'preview-tray: BlockDiffCard renders a Bob-annotated drawer header
    as SF Symbol tray plus the semibold verbatim title, muted when empty, with accessible
    labels and light/dark/increased-contrast render fixtures.'
- id: draft-styling
  title: Semibold drawer titles and dimmed delimiters in the draft editor
  size: medium
  depends_on:
  - annotation-build-repair
  description: 'draft-styling: paint drawer_bullet titles semibold in the section
    family and drawer_marker delimiters in tertiary ink that returns to full ink at
    the caret, through a portable span helper the typing phases reuse.'
- id: star-expansion
  title: Bob-driven star expansion and atomic Backspace
  size: medium
  depends_on:
  - draft-styling
  description: 'star-expansion: a physical `*` asks capture-rewrite for expand_drawer_marker,
    applies it as one undoable native edit, absorbs a queued star, and deletes an
    empty `****` as one Backspace.'
- id: title-keys
  title: Drawer title newline and closing-star overtype
  size: medium
  depends_on:
  - star-expansion
  description: 'title-keys: at the end of a Bob-authorized drawer title, newline keys
    step past the closing `**` into a nested note and a typed `*` steps over the closing
    pair.'
- id: mac-acceptance
  title: Mac drawer acceptance, screenshots, and documentation
  size: small
  depends_on:
  - annotation-build-repair
  - preview-tray
  - draft-styling
  - star-expansion
  - title-keys
  description: 'mac-acceptance: verify the integrated feature on exact-SHA macOS CI,
    publish the drawer render screenshots, reconcile Mac and bob-cli docs with what
    shipped, and run every repo gate.'
proposed_by: bbugyi200.apollo.bob-cli-ph.8
proposed_with: claude/opus
parent_bead: bob-cli-ph.8
decided_by: auto
create_time: 2026-11-25 20:55:44
status: wip
bead_id: bob-cli-ph.8.1
---

- **PROMPT:** [prompts/202611/finish_mac_true_case_drawers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/finish_mac_true_case_drawers.md)
- **BEAD:** [bob-cli-ph.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.8.1.md)

# Finish Bob Mac Capture true-case drawers

This child epic finishes **bob-cli-ph.8**, the `mac-drawers` phase of
`plan:202611/true_case_drawers.md`. When it is done, a drawer in Bob Mac Capture reads
the way the parent epic designed it:

- **Preview.** A header line that Bob annotates with `drawer` shows an outline tray and
  the title exactly as Bob sent it, with no `**`.
- **Draft.** The editor shows the title semibold and the literal `**` delimiters dimmed.
  A delimiter returns to full ink while the caret touches it.
- **Typing.** On a bullet where Bob admits a drawer, typing `*` produces `- **|**`.
  Backspace on an empty `****` removes all four stars. The newline keys and a typed `*`
  step past the closing pair.

The Mac stays a thin client throughout. Bob decides grammar, eligibility, titles, and
preview data. Swift reads Bob's spans, rewrite replies, and annotations, and it only
handles keystrokes around them.

## Why an epic, and what already landed

Three earlier runs worked this phase. Read their notes with
`sase bead read bob-cli-ph.8 -r "Need phase status, notes, and final decisions"`.

**Landed in bob-mac-capture.**

- `4eb9b2e9`: the drawer fixture generator requires grammar 30 or later.
- `b45b1951`: the earlier runs added `CaptureDrawerAnnotation` and `CaptureDrawerMarker`
  (`Sources/CaptureCore/CaptureDrawerAnnotation.swift`). They added an optional `drawer`
  to `CapturePomodoroBlockLine` and `CaptureBlockDiffRow`, and plumbed it through the
  task and Pomodoro block presentations. They added an optional `marker` on
  `CaptureDrawerSummary` and removed the drawer `.uppercased()` calls in
  `CaptureTransferPresentation` and `CaptureMergeInPresentation`. They regenerated the
  drawer fixtures from bob-cli `05dd83cf` at grammar 31 and updated the Mac
  `docs/protocol.md` and `docs/capture-previews.md`.

**Landed in bob-cli.** The Mac section of `docs/task-drawers.md` landed in `f4da72ff`.

**Mac master is red.** CI run 45731912096 failed at `b45b1951`, which was never
compiled. The Build step fails at `Sources/CaptureCore/CaptureModels.swift:9857`.
`CaptureDrawerSummary.init(from:)` binds `raw` as a `String` and then calls
`raw.map(CaptureDrawerMarker.init(wireValue:))`, which maps over its Characters. The
build may have more errors after that one. Every Mac agent is blocked until this is
fixed, so the repair phase comes first and depends on nothing.

**Not yet built.**

- The `BlockDiffCard` tray branch.
- Editor span styling.
- Every typing assist.
- Regression and design tests.
- Exact-SHA verification.

The Mac docs already describe tray previews and semibold titles that do not exist yet.
Each phase makes its part of those docs true, and the acceptance phase reconciles the
whole set.

The earlier tale (`plan:202611/mac_true_case_drawers.md`) did not fit one agent. Swift
can only be type-checked on a scratch copy on athena or the mac, and every landing waits
about 25 minutes for macOS CI. So the remainder is split into phases that each fit one
agent. `preview-tray` and `draft-styling` touch disjoint files and may run in parallel.
The three editor phases run in sequence because they share
`Sources/BobMacCapture/CapturePanelModel.swift` (about 25k lines) and the keystroke
paths in `CapturePanelController.swift`.

## Governing decisions and boundaries

The parent epic's decisions are final:

- **`drawer_glyph = tray`.** The Mac draws the outline SF Symbol `tray`, never
  `tray.fill` or `cabinet`.
- **Memory decisions.** These belong to the parent's `memory` phase (`bob-cli-ph.11`).
  This epic edits no memory notes.

Read these governing records with `/sase_memory_read` before changing behavior:

- `decisions:mac-capture-is-a-thin-client`. Swift never recognizes a drawer from
  Markdown, letter case, indentation rules, or a reserved-name list. Eligibility always
  comes from Bob's `capture-parse` spans and diagnostics, `capture-rewrite` replies, or
  preview annotations.
- `decisions:capture-bullets-mirror-the-task-block`. The parent epic replaces its
  ALL-CAPS clauses with bold true-case containers. Draft, preview, and commit still name
  the same place.
- `decisions:capture-star-blocks-existing-tasks` and
  `decisions:capture-sigils-say-where-what-how`. The drawer `*` assist must not claim
  any of these: the dependency `*` sigil and its picker, `*` alias tips, Markdown
  emphasis, multiplication, or a mid-line star.
- `decisions:task-reqs-are-met-with-their-task`. Requirement chips must keep working,
  along with `[` expansion, overtype, Backspace, and `req_in_drawer`.
- `decisions:notebook-routes-write-notes`. Bob, not Swift, decides that a route-only
  notebook list stays literal.

**Per-phase rules.**

- **Opening the repo.** Open bob-mac-capture with
  `sase repo open bob-mac-capture -r "<phase reason>"`, use only the path it prints, and
  read its `AGENTS.md` first. All Mac paths below are relative to that checkout.
- **Compiling Swift.** Compile every Swift change before landing it. Swift lands only
  after it compiles.
  - Run `just portable-check` on a scratch copy on athena, which has Swift 6.0.3. Rsync
    without `.build` or `.git`, and delete the copy afterwards.
  - Best effort: type-check the AppKit and SwiftUI targets on a scratch copy on the mac,
    with `Scripts/bundle.sh` and then `just typecheck-tests`. Never use Bryan's checkout
    or `just install`.
  - If the mac is offline, say so in the phase note. The macOS CI gate still applies.
- **Formatting.** Run swift-format only on the lines you change, at 100 columns with
  4-space indents. Never re-indent whole files.
- **CI gate.** Before landing Swift, run `Scripts/macos-ci-check` through
  `/sase_monitor`. After the push, watch the exact-SHA macOS run to green as the repo's
  `AGENTS.md` requires. Rebase onto current master before the snapshot when a parallel
  phase has landed.
- **Async tests.** Every async test wait is bounded. A hang shows up at the CI watchdog.
- **Fixtures.** Generate fixtures only with `Scripts/generate-*-fixtures.sh`, using
  `BOB_BIN` built from a clean bob-cli checkout at current master. Never hand-edit
  generated JSON. Edge cases may use inline JSON in tests.
  - Only `draft-styling` and `star-expansion` edit `Scripts/generate-drawer-fixtures.sh`
    and `Tests/Fixtures/drawer-fixtures-README.md`.
  - Parallel phases leave that script alone.
  - The cutover phase `bob-cli-ph.10` may land during this epic. If a regenerated legacy
    fixture stops producing `legacy_caps`, keep the legacy coverage as inline decode
    payloads and update the README provenance.
- **Wire format.** Decoding stays additive. Older fixtures and an older `bob` keep
  working. A missing, malformed, or unknown drawer field changes only the drawer
  presentation, never the rest of the capture.
- **Out of scope.**
  - bob-cli grammar or Rust changes. If Bob's contract lacks a fact a phase needs,
    record a `PROPOSED FOLLOW-UP:` and do not infer the fact in Swift.
  - Plugin changes, vault edits, release or version edits, and deploying to Bryan's
    installed app.
- **Docs.** Read `docs.md` with `/sase_memory_read` before editing docs. Mac docs must
  pass `just docs-check`, including word budgets.
- **Discovered work.** Phase workers record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead and never create beads.
- **Pre-existing failures.** A gate failure that reproduces identically on the clean
  base is recorded as a follow-up, citing any existing task, and does not keep a phase
  open. The `bob-cli-ph.8` notes already record two of them: the clean-base bob-cli docs
  word-budget overages in `capture-syntax.md` and `json-outputs.md`, and the clippy
  `manual_contains` in `tests/cli/capture/complete_block_id.rs`. Confirm these still
  reproduce before citing them.

## Phase: annotation-build-repair

This phase is urgent: Mac master stays red until it lands.

### Changes

1. In `CaptureDrawerSummary.init(from:)` (`Sources/CaptureCore/CaptureModels.swift`),
   decode the optional marker string and convert it with
   `CaptureDrawerMarker(wireValue:)`.
   - An absent marker stays `nil`.
   - An unknown marker string becomes `.unknown`.
   - A malformed marker value, such as a number, must not discard the summary.

   Wrap the `init(title:key:created:marker:)` signature to 100 columns. Fix every other
   compile or lint error in the `b45b1951` files.

2. Fix forward if master already builds. Another agent may have repaired the build
   first. Then keep their fix and do the rest of this phase.
3. Add portable CaptureCore tests in the existing suites, or in a focused
   `CaptureDrawerAnnotationTests`:
   - **Annotation decoding.** Cover a full payload, a missing `key`, `marker`, or
     `empty`, a malformed annotation, and unknown marker strings. A malformed annotation
     makes the line's `drawer` `nil` and leaves the line otherwise intact.
   - **Marker values.** Cover `bold`, `legacy_caps`, and `section`.
   - **Unicode titles.** Use `Café`, `C++ Notes`, `Q1: Goals`, and an astral title.
   - **Summary marker.** Cover the summary `marker` absent, known, and unknown.
   - **Generated fixtures.** Use `drawer-capture-*.json`, and assert that each annotated
     row's `title` reaches `CaptureBlockDiffRow.drawer` through
     `CaptureTaskBlockPresentation` and `CapturePomodoroBlockPresentation`. Tokens,
     content, change, and fold identity stay unchanged.
   - **Verbatim destination labels.** Transfer and merge-in labels keep mixed case
     (`› Perf Budget`). A legacy `FUTURE WORK` stays exactly as sent. ALL-CAPS task
     types and Pomodoro names are unaffected.

### Verification

- Run `just portable-check` on an athena scratch copy.
- Type-check on the mac with `bundle.sh` and `just typecheck-tests`, best effort.
- Run `Scripts/macos-ci-check` through `/sase_monitor`, then watch the post-push
  exact-SHA run to green.
- Record the green SHA and run URL in the phase close note.

## Phase: preview-tray

### Changes

1. Add a drawer branch to `Sources/BobMacCapture/BlockDiffCard.swift`, keyed only on
   `row.drawer != nil`.
   - Render the outline `Image(systemName: "tray")` plus `annotation.title`, semibold.
   - Never show the raw `**`, and never derive a title from row text.
   - Keep the indent guides and change wash. Keep the diff gutter, the removed-row
     strike and dim, and the fold behavior.
   - `empty == true` mutes the glyph and title.
   - A `.unknown` marker renders conservatively: the tray plus the verbatim title. If
     that is unsafe in some row context, fall back to the existing verbatim row.
   - An unannotated row, including a bold ordinary note, renders exactly as today.
   - Rows with `req`, `isRelationSuccessor`, `isKeepMoving`, or `isUnblocked` keep their
     current precedence. Choose the order deliberately and cover it with a test.
2. **Tint.** Use the existing section color family, or the card's Bob-authored owner
   tint where the card already has one. Never infer task type or state from Markdown.
   Add any new palette helper to `CaptureEditorPalette.swift` next to the existing
   section and requirement colors.
3. **Copy and accessibility.**
   - Text selection and copy keep the verbatim line content, and the glyph never enters
     copied text.
   - VoiceOver reads `Drawer, <title>, <change>`, plus `empty` when Bob reports `empty`.
     Do not invent note counts that Bob does not send.
   - The `.help` tooltip names the drawer.
4. **Coverage.** Apply the branch everywhere `BlockDiffCard` renders rows: task cards,
   Pomodoro cards, section blocks, and transfer and merge previews. Confirm that each
   path carries `drawer`.
5. **Tests.**
   - Extend `TaskBlockDesignTests`, or add `DrawerHeaderDesignTests`, to render
     nonempty, empty, removed, legacy-as-sent (`FUTURE WORK`), and section-marker
     headers next to a requirement row and an ordinary bold note.
   - Render a mixed-case transfer destination.
   - Write each render through `RenderFixtureWriter` in light, dark, and increased
     contrast so CI uploads them under `render-fixtures`.
   - Check for title wrapping and for no inherited bold or strike leakage.
   - Use `drawer-capture-*.json` fixtures for real rows and inline JSON for `empty` and
     `.unknown`. Do not edit the drawer fixture generator.
6. **Docs.** Make the drawer paragraph in the Mac `docs/capture-previews.md` describe
   exactly what renders.

### Verification

- Run `just portable-check` on athena.
- Type-check on the mac with `bundle.sh` and `just typecheck-tests`, best effort.
- Run Mac `just docs-check`.
- Run `Scripts/macos-ci-check` through `/sase_monitor`, then watch the post-push
  exact-SHA run to green.
- Download that run's `render-fixtures` artifact and look at the drawer images before
  closing. Correct visible defects.

## Phase: draft-styling

### Changes

1. **Span helper.** Add a portable CaptureCore helper, for example
   `CaptureDrawerSpanGeometry`, that works over validated `CaptureSpan`s and the draft.
   - Pair each `drawer_bullet` title with its opening and closing `drawer_marker` spans,
     and recognize the empty `****` skeleton: two marker spans with a zero-width
     `interactive_placeholder` between them.
   - Report each title's byte range and each marker's range.
   - Report whether a collapsed caret touches a marker. Touching includes both
     endpoints.
   - Find the closing marker that starts exactly at a caret.
   - Verify only that a marker's bytes are `**` inside a Bob-authorized range. No regex
     over the draft, and no structural inference.
   - The typing phases reuse this helper. Add portable tests for it.
2. **Editor paint.** In `CapturePanelModel.paintHighlighting` and the palette:
   - `drawer_bullet` title bytes are semibold in the existing section color.
     `CompletionRowContent` already maps `drawer_bullet` to `.section`.
   - `drawer_marker` bytes use tertiary label ink. They switch to normal full ink while
     the collapsed caret touches that marker.
   - Add a `drawer_marker` category to `CompletionRowContent`, or an equivalent mapping,
     so completion rows and the editor agree.
   - Markers always keep their characters. There is no concealment, no attachment, and
     no change to attributed-string length.
   - Any new attribute, such as a font weight, is cleared in
     `clearEditorHighlightAttributes` so it never leaks into typed text.
   - Increased contrast keeps the dim markers legible.
3. **Caret repaint.** Caret movement without a text change must repaint marker ink, and
   must not rewrite the draft, move the selection, or break undo. Commit `0b458fb1`
   (later replaced by `c6c9c13e`) shows a lockstep caret repaint you can learn from;
   keep the repaint cheap.
4. **Fixtures.** Extend `Scripts/generate-drawer-fixtures.sh` with `capture-parse`
   replies for:
   - the empty skeleton `Prepare release\n- ****`, with the caret between the pairs;
   - a note-less existing task;
   - a project note;
   - a `%n` seed;
   - a record group, if the fixture vault can host a record route; otherwise reuse the
     record fixture generator's vault setup;
   - Unicode and astral titles, such as `Café` and `🚀 Launch`, with astral text before
     the line.

   Record the provenance in `drawer-fixtures-README.md`.

5. **Hosted tests.** Add `CaptureEditorHighlightTests`, or a focused
   `CaptureDrawerHighlightTests`, covering:
   - the title is semibold and the markers are dim;
   - caret-adjacent markers return to full ink at both endpoints;
   - attributes clear after the caret leaves, and after the drawer is deleted;
   - text and offsets stay identical;
   - stale or malformed spans leave ordinary text;
   - requirement chips are unchanged.
6. **Docs.** Update the Mac `docs/capture-panel.md` editor-highlighting text.

### Verification

Run the same Swift, docs-check, and exact-SHA CI gates as `preview-tray`. Also run Mac
`just check-scripts` for the generator change.

## Phase: star-expansion

### Changes

1. **Intent.** In `CapturePanelController`, record a drawer-star intent for a physical
   `*` keystroke in the main draft editor, alongside
   `noteRequirementMarkerExpansionIntentFromEvent`. Require all of the following:
   - no marked text;
   - a collapsed selection;
   - no Command, Control, or Option (Shift is allowed);
   - not a key repeat.

   A cheap byte pre-filter is allowed, such as the star following `- ` at a line start.
   The pre-filter only avoids needless calls. It never decides eligibility.

2. **Rewrite.** In `CapturePanelModel`, consume the intent when AppKit's inserted text
   matches.
   - Call `startCaptureRewrite` with `holdInputUntilRewrite: true`, mirroring the
     requirement path.
   - Add `expand_drawer_marker` to the native single-edit rules in the rewrite reply
     handler, so `applyNativeSingleEditRewrite` applies Bob's edit as one undoable
     native edit at Bob's cursor.
   - Undo restores the typed `- *`, and Redo restores `- **|**`.
   - Paste, template load, programmatic changes, and undo or redo never call the
     endpoint.
   - The existing generation, draft, caret, and process-client guards drop late or
     mismatched replies.
   - Title keys held during the request land in order, without loss or duplication.
   - An unchanged or failed reply leaves the typed `*` and releases the hold.
   - Bob supports an auto-paired `- *|*` form, but the Mac has no auto-pairing. Do not
     add any.
3. **Queued star.** After Bob's expansion, a one-shot arm bound to the exact draft and
   caret absorbs one immediately queued `*` at `**|**`. Mirror
   `RequirementMarkerOvertypeArm`. Any other key, selection change, or draft change
   disarms it.
4. **Atomic Backspace.** At `**|**`, Backspace deletes exactly the four stars as one
   undoable edit and leaves `- `. Mirror `consumeRequirementMarkerBackspace` and
   `deleteRequirementMarkerInEditableTextView`.
   - Authorize the deletion with the bound expansion arm, which bridges the parse gap.
   - Otherwise, authorize it with a fresh parse whose two marker spans and empty
     placeholder surround the caret. A fresh parse has `input == text`, the current
     generation, and the current client. Use the `draft-styling` helper.
   - A nonempty title or a selection keeps native Backspace.
5. **Fixtures.** Add `capture-rewrite --cursor` replies to the drawer generator.
   - **Expands:** new task, note-less existing task, project note, `%n` seed, record
     group, the auto-paired form, and astral text before the caret.
   - **Stays literal:** a nested note, a route-only notebook list, a plain mid-line star
     (`Buy 2 * 3`), a checkbox body, and a trailing dependency `*` after task text.
   - Route each fixture through `Tests/Fixtures/fake-bob`'s `capture-rewrite` branch.
6. **Tests.** Add hosted pipeline tests modeled on `CaptureRequirementPipelineTests`,
   using `CaptureRewriteTestSupport`. Cover:
   - expansion from each family;
   - rapid title typing during the hold;
   - undo and redo;
   - the queued-star arm;
   - Backspace before and after the parse arrives;
   - stale replies after the caret moves, marked text, paste, template load, and
     timeout;
   - every negative reply keeping native typing;
   - the dependency `*` picker and `*` alias tip still working on their own drafts;
   - requirement `[` expansion unchanged.

   Add `CaptureRewriteResponseTests` decoding of the real replies.

7. **Docs.** Document the star and Backspace behavior in the Mac `docs/capture-panel.md`
   (or `docs/typing-tips.md`) and `docs/keyboard.md`, within the word budgets.

### Verification

Run the same Swift, check-scripts, docs-check, and exact-SHA CI gates.

## Phase: title-keys

> [!decision] drawer_newline_keys = newline_keys Shift-Return, Option-Return, and Ctrl-J
> nest at a drawer title end. Plain Return keeps its capture meaning everywhere, and a
> childless drawer is refused by Bob with `empty_drawer_bullet`, as today.

> [!decision] drawer_newline_keys = return_too Plain Return also nests at a drawer title
> end, but only when Bob's current parse reports an `empty_drawer_bullet` diagnostic
> whose range covers that drawer line. Every draft Bob accepts still captures on Return.
> A stale parse or a missing diagnostic means Return captures.

### Changes

1. **Authorization.** A key is authorized only when the collapsed caret sits at the end
   of a nonempty `drawer_bullet` title, immediately before its closing `drawer_marker`.
   Either of these sources counts:
   - a fresh parse, checked through the `draft-styling` helper;
   - an expansion-session record from `star-expansion`: Bob's expansion reply plus
     exactly the contiguous title bytes the user has typed since at the caret, with no
     other edit and no selection change.

   Anything else keeps native behavior. Swift never infers a drawer from text.

2. **Newline keys.** Shift-Return, Option-Return, and Ctrl-J at an authorized title end
   move past the closing `**` and insert a newline plus the drawer line's leading
   whitespace plus two spaces plus `- `. This is one undoable edit, and the caret lands
   after the new `- `.
   - Keep the existing precedence for completion menus, pickers, the lone `@file` header
     assist, and template-draft Ctrl-J. Only the authorized drawer-title case changes.
   - Implement the selected `drawer_newline_keys` branch.
   - The empty `****` skeleton has no title, so these keys never manufacture a drawer
     there.
3. **Overtype.** A physical `*` at an authorized title end moves the caret past both
   closing stars and inserts nothing. It never removes stars and never arms an alias.
4. **Tests.** Add hosted tests for each newline key and for overtype, under both
   authorization sources. Cover:
   - nested indentation for an indented drawer line;
   - undo;
   - menu-open precedence;
   - Return capturing in the `newline_keys` branch, or nesting only under Bob's
     diagnostic in the `return_too` branch;
   - unchanged behavior for a nonempty selection, a caret mid-title, a stale parse, an
     ordinary bullet, and a bold note that Bob does not annotate.
5. **Docs.** Update the key tables in the Mac `docs/keyboard.md` and the drawer typing
   text.

### Verification

Run the same Swift, docs-check, and exact-SHA CI gates.

## Phase: mac-acceptance

1. **CI evidence.** Confirm that master contains every phase's landed commit and that
   the latest master SHA's macOS run is green. Rerun `Scripts/macos-ci-check` through
   `/sase_monitor` if any doubt remains. Record the SHA and run URL.
2. **Screenshots.** Download that run's `render-fixtures` artifact. Look at the drawer
   images in light, dark, and increased contrast:
   - glyph distinct from the task checkbox and the requirement diamond;
   - legible titles;
   - muted empty drawers;
   - no `**` in the preview.

   Read the `/sase_artifact` skill first, then publish the inspected images as SASE
   artifacts. Attach the refs to this phase and to `bob-cli-ph.8` with a concise
   acceptance note. This is the parent epic's Mac screenshot evidence.

3. **Docs.** Reconcile the documentation with what actually shipped:
   - In the Mac repo: `docs/protocol.md` (spans, the rewrite rule, annotation fields,
     and compatibility rows), `docs/capture-previews.md`, `docs/capture-panel.md`, and
     `docs/keyboard.md`.
   - In bob-cli: the Mac Capture section of `docs/task-drawers.md` (rendering, editor
     paint, typing assists, and the selected newline-key branch).
   - Remove any claim of behavior that did not ship. Do not claim ALL-CAPS cutover
     unless `bob-cli-ph.10` has closed.
4. **Gates.**
   - Mac: `just docs-check` and `just check-scripts`.
   - bob-cli: `just docs-gen`, `just docs-check`, and `just check`.

   Use `/sase_monitor` for long runs.

## Land and completion responsibility

Phase workers close only their own child phase, after checking
`sase bead epic-symbols <phase-id>`. They never close `bob-cli-ph.8`, `bob-cli-ph`, or
any ancestor.

The child epic's land agent verifies the following:

- Every phase is closed, with exact-SHA green macOS evidence for each Swift landing.
- The drawer screenshots are durable artifact refs.
- No Swift-side grammar, drawer recognition, memory edit, Rust change, or vault write
  slipped in.
- Requirement typing and the dependency `*` picker still pass their suites.

Then, as `bob-cli-ph.8`'s completion owner, the land agent does the following:

1. Run `sase bead epic-symbols bob-cli-ph.8`, and resolve or re-key any leftovers to a
   still-open later bead.
2. Close only `bob-cli-ph.8`:
   `sase bead close bob-cli-ph.8 --note "<verified behavior, tests, CI SHAs, artifact refs>"`.
   It may close this child epic through the normal land lifecycle, but never
   `bob-cli-ph` or any higher ancestor.

Planning and approval leave `bob-cli-ph.8` in progress.
