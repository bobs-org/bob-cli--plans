---
tier: tale
title: Section headers in the Mac Refs inspector and Copy card
goal:
  Complete bob-cli-bd.8 with section-aware inspector and copy presentation while
  preserving flat entry indices, pagination, and keyboard selection.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-bd.8
proposed_with: codex/gpt-6-astra
bead: bob-cli-bd.8
status: done
---

- **PARENT:**
  [202610/ref_note_annotation_sections.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_note_annotation_sections.md)
- **BEAD:**
  [bob-cli-bd.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bd/bob-cli-bd.8.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-bd.8](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-bd.8.md)
- **COMMITS:**
  - [f9cd88a](https://github.com/bobs-org/bob-mac-capture/commit/f9cd88add5f2b4bdb8ca5a0dffde9b42617b7253)
    — feat(refs): present annotation sections in the inspector and Copy card

# Section headers in the Mac Refs inspector and Copy card

Complete the Mac presentation phase `bob-cli-bd.8` of
`plan:202610/ref_note_annotation_sections.md`. When `bob ref show -f json` supplies
sections, the inspector shows their titles, descriptions, and membership, and the Copy
card preserves their grouping. Older responses retain the current appearance and
behavior. Headers must never change annotation indices or keyboard numbering.

## Scope and settled decisions

- The assigned bead is already `in_progress`, depends on closed `bob-cli-bd.5`, and had
  no prior implementation or remainder notes when this plan was authored. The inspected
  Mac base is `6d41b2e`; it has no section decoding or presentation yet. Re-read the
  bead and current code before implementation to accommodate intervening work. Do not
  claim it or change status by hand.
- This is a single, bounded frontend change: additive data decoding, one pure feed
  projection, two presentation surfaces, and their existing test infrastructure. It
  needs no new parser, endpoint, persistence, or agent phase graph.
- Use `/sase_repo` to open `bob-mac-capture`; read its `AGENTS.md`. All source paths
  below are relative to that returned checkout unless explicitly marked bob-cli. Do not
  locate or use a developer checkout on a remote host.
- Read the parent plan with `sase artifact read` and consult required reference memory
  using `/sase_memory_read`, including `docs.md`,
  `decisions:mac-capture-is-a-thin-client`, and `tailnet.md` for remote verification.
- The epic's decisions are final: `decision_record = no`, `glossary_term = no`. Edit no
  memory notes. At completion, record the skipped decision strand and glossary term as
  `PROPOSED FOLLOW-UP:` notes on this phase, unless already recorded there. Do not ask
  those decisions again.
- Keep the app a thin client: consume sections reported by bob. Do not parse `#`
  declarations or `@` membership, read the vault to discover sections, or mutate it. No
  CLI/schema-version changes, release/version edits, or app installation.

## Existing contract and integration points

bob-cli's `src/native/ref_library/show.rs` already emits schema version 1 with optional
row `sections` and optional annotation `section`. Each section contains `id`, `title`,
`tag`, `block_id`, `link`, and `count`, plus optional `description`, `pages`, and
`images`. A member's `section` is the public ID, not its `s-` block ID. Sections are in
declaration order; annotations are in managed-region order, with section members first
and ungrouped annotations afterward. Counts/pages describe the full section, including
when annotation filtering yields fewer members.

`Sources/RefsCore/RefsInspector.swift` currently decodes the response and projects
`RefsInspectorFeed` from a flat array. `RefsNotesPage`, feed `total`, fades, and reveal
anchors all count entries. `Sources/BobMacCapture/Refs/RefsInspectorLoader.swift` stores
annotations in `RefsInspectorContent` and constructs its feed; decoding alone will not
reach the UI.

`RefsCopyFormat.allNotes` currently joins `yourNotes` and appends `### Notes` for
`own_notes`. `RefsHighlightPicker` selects and windows a flat array of copyable
annotations. `RefsCopyCardView.pickerRows` assigns digit keys from that window. The
picker height budget currently assumes only 36 pt selectable rows, calculated in
`RefsCopyCardLayout` and supplied by `RefsPanelView`/`RefsPanelModel`.

## Implementation

### 1. Decode sections without breaking older or malformed responses

In `Sources/RefsCore/RefsInspector.swift`:

- Add `RefsShowSection` with the JSON fields above, conforming to the existing
  `Decodable`, `Equatable`, and `Sendable` conventions. Map `block_id` appropriately.
  Treat absent descriptions/pages/images as empty or nil, as appropriate.
- Add `RefsShowRow.sections`, defaulting to `[]`, and
  `RefsShowAnnotation.section: String?`, defaulting to nil. Preserve existing
  initializer call sites through default arguments.
- Decode the new fields lossily with `try?`/`decodeIfPresent`: missing, null, or
  wrong-type section metadata must not discard a row or annotation. Prefer a lossy
  section-element wrapper, following `LossyShowRow`, so one malformed section does not
  discard valid sibling sections. Ignore unusable/duplicate section IDs safely without
  traps; keep the first usable descriptor. Never invent membership from text.
- Add `Tests/Fixtures/refs-show-sections.json`: multiple populated sections, a
  leading/middle empty section, a trailing empty section scenario, equal page labels
  across groups, ungrouped entries, comments, a standalone note, a tucked group, and an
  image annotation. Use small fixture variants in unit tests when the ungrouped tail and
  truly trailing empties need different orderings. Retain canonical JSON names/omissions
  and `schema_version: 1`.
- Exercise the existing `FAKE_BOB_REFS_SHOW_FIXTURE` route in `Tests/Fixtures/fake-bob`;
  it already accepts arbitrary fixture filenames, so change the script only if needed.

### 2. Project headers while retaining every flat entry index

Extend `RefsInspectorFeed` with section metadata (default `[]`) and a `FeedHeader` value
representing `.section(...)` or `.other`. Add `Entry.leadingHeaders` and enough resolved
section context for accessibility on every member, including members that do not
introduce a header. Keep `highlights`/entries in their existing order.

- Build header placement from the complete filtered feed, then take the visible entry
  prefix. Never insert headers into that prefix or its index space.
- Attach a populated section's header to its first displayed member. Accumulate
  empty/no-displayable-member section headers in declaration order and attach them to
  the next entry; at the ungrouped boundary, attach pending sections before `.other`. A
  truly trailing set has no entry and is exposed separately only when all entries have
  been revealed. With zero entries, expose the section headers immediately so an empty
  scaffold can be seen.
- Attach `.other` exactly once, before the first ungrouped displayed entry, only when
  usable sections exist. An unknown section ID has no invented heading and safely falls
  into the ungrouped presentation. If there are no usable sections, retain today's feed
  exactly.
- A header resets the page-label run, even when its first member has the same page label
  as the previous group. Preserve within-group runs and nil-page behavior.
- Disclosure text gains `· N sections` when hidden entries carry section starts; count
  section headers, not `.other`, and preserve the existing note/image and reveal
  clauses. Include the section information in the disclosure accessibility label too.
  Trailing empty headers never become hidden entries or create a reveal.
- Keep entry `index`, `total`, note-card totals, `RefsNotesPage` ranges/generations,
  `RefsInspectorNoteFade`, and `RefsInspectorRevealAnchor` keyed exactly as before.
- Include the section title in accessibility labels for highlights, notes, images, and
  tucked group pieces, while retaining existing quote, continued-page, note, underline,
  and unavailable-image wording. Unsectioned labels stay unchanged.

Thread the decoded sections through `RefsInspectorContent`, its defaulted initializer,
loader response/cache publication paths, and `feed`. Check all `RefsShowRow` to
`RefsInspectorContent` construction paths; reuse cached metadata rather than issuing
another subprocess call.

### 3. Render native inspector headers

In `RefsInspectorView.swift` and `RefsVisualTokens.swift`:

- Render leading headers before each member and trailing headers after all revealed
  entries. Render the HIGHLIGHTS container when it has headers but no annotations. Keep
  headers outside the member's combined accessibility element so VoiceOver retains a
  separately navigable heading. The reveal anchor should include the new entry's leading
  headers at the reveal boundary, using the same entry index.
- A section header uses a rounded 3 pt leading rule spanning title and description,
  adaptive `Color.purple` at 0.6 opacity (1.0 under Increase Contrast).
- Title: `.subheadline.weight(.semibold)`. Trailing `@tag · N`:
  `.caption2.monospacedDigit()`, tertiary. Description: `.callout`, secondary, clamped
  to 3 lines in compact mode or 5 in wide mode, `.help(fullDescription)`.
- Add `inspectorSectionGap = 14` above every header after the first. Avoid stacking the
  ordinary page-run gap on top of the section gap. Preserve existing member rendering,
  image sizing, underlines, and continuation seams.
- `.other` reads `OTHER HIGHLIGHTS N` in the existing HIGHLIGHTS caption style; its
  count is the number of ungrouped displayed entries.
- Use `.isHeader` and labels such as `Section Memory, 2 highlights. <description>`, with
  singular/plural handling. Titles and typography convey grouping independently of
  color. Do not add jump chips, a new expand control, or section-copy shortcuts.

### 4. Preserve grouping in the Copy card

In `Sources/RefsCore/RefsCopy.swift`:

- For top-level A, partition `row.yourNotes` by valid section ID, in declaration and
  member order. For each section with at least one copyable note, emit `### <Title>`,
  its description as Markdown blockquote lines, and the existing note/standalone-note
  blocks. Preserve blank lines in multiline descriptions.
- Emit `### Other highlights` only when sections exist and ungrouped copyable notes
  remain. Append `### Notes` and `own_notes` as today. Empty sections and sections
  containing only uncommented quotes do not create empty A groups.
- Keep all unsectioned output byte-for-byte identical, individual copy payloads and
  provenance unchanged, and the H picker's existing A/all-highlights payload semantics
  unchanged. `G` remains unbound.

In `RefsHighlightPicker.swift` and `RefsCopyCardView.swift`:

- Carry section group metadata alongside the flat selectable entries. Draw section
  titles as non-selectable group rows before visible group starts; show current group
  context at the top of a window that begins midway through a section. Use an Other
  highlights group for an ungrouped tail when sections are present. Omit empty groups
  from the picker, which has no selectable members for them.
- Group rows have header accessibility traits, no keycap, selection, click-to-copy
  action, or digit slot. Arrow navigation, wraparound, Return, Option alternates,
  window-relative 1–9, and counts remain based exclusively on entries.
- Account for header height in the window budget so the card still fits. Extend
  `RefsCopyCardLayout` and its `RefsPanelView`/model caller only as necessary; use a
  deterministic bounded allowance for section headers rather than a measurement feedback
  loop. An unsectioned picker keeps its current capacity, and a sectioned picker always
  keeps its selected entry visible and at least one selectable entry when space allows.
  Cover many one-entry sections and a window starting mid-group.

### 5. Tests, design renders, and documentation

Add focused regression tests to the existing RefsCore suites (or a dedicated
`RefsSectionsTests.swift` when clearer):

- Full fixture decoding; absent/null/wrong-type fields; malformed section elements;
  valid siblings surviving; old fixtures unchanged.
- Header attachment for leading, middle, and trailing empties, only-empty sections,
  filtered-out members, and an Other tail. Every header appears once in the inspector at
  the correct reveal boundary. Same-page labels restart across headers.
- Flat indices, totals, note counts, pagination ranges, disclosure text, and entry
  accessibility, including tucked group pieces. A pagination boundary immediately before
  a section header must reveal the original range and index.
- Exact Copy A Markdown, multiline descriptions, own notes, no empty groups, legacy
  output, unchanged copy-one, and H navigation/digit mapping across group boundaries.
  Include header-aware capacity and unchanged unsectioned capacity.

In `Tests/BobMacCaptureTests/RefsInspectorTests.swift` and
`RefsInspectorScrollTests.swift`, verify loader-to-feed section propagation and
reveal/scroll anchor behavior with leading headers. Preserve bounded waits in async
tests. Extend Copy component coverage where the changed UI layout warrants it.

In `RefsPanelDesignTests.swift`, add a dedicated section render test and
`renderInspectorSections`, using the existing `RenderFixtureWriter`. Produce light and
dark renders at 880 and 1344 pt, plus at least one large accessibility text-size
variant. Include long descriptions, an empty section, page-label resets, an Other group,
and a reveal boundary. Add a grouped H picker render to existing Copy card design
coverage to inspect heading fit and digit placement. Keep these synthetic.

Update `docs/refs-inspector.md` for HIGHLIGHTS section headers, empties, paging, and
accessibility, and `docs/refs.md` for Copy A grouping and H group rows. These files are
mounted into the bob-cli site: do not edit bob-cli's generated
`docs/reference/mac-capture/` tree or README front doors.

## Verification and completion

1. Run the targeted RefsCore tests during development, then `just portable-check` in the
   Mac checkout. This host currently lacks Swift; if still true, use the configured
   `athena` SSH alias and an isolated scratch copy of the opened checkout. Do not
   substitute static inspection for compilation. Retain concise command and exit-status
   evidence. Use `/sase_monitor` for long checks/CI waits as required by the single-turn
   runner, with continuation instructions that finish this bead.
2. Follow the Mac `AGENTS.md`: attempt an isolated scratch rsync build on `mac`,
   excluding `.build` and `.git`, then run `./Scripts/bundle.sh --identity -` and
   `just typecheck-tests`. Remove only the scratch copy afterward on success or failure.
   Never run `just install` or use Bryan's checkout. The Command Line Tools host lacks
   XCTest; stub typechecking is compilation evidence, not executed tests. If
   unreachable, explicitly report the macOS targets as unverified; portable compilation
   is still required.
3. Execute macOS XCTest/design renders on an Xcode-equipped environment or the existing
   macOS CI for the resulting change. Set `BOB_MAC_CAPTURE_RENDER_DIR` and inspect the
   generated section and picker PNGs for spacing, clipping, contrast, empty headers, and
   large text. If local Mac execution is unavailable, retain the render test definitions
   and identify CI/test/image review still outstanding for the parent land agent. Do not
   report test compilation as test execution or claim visual review without viewing
   images. The epic land remains responsible for passing macOS CI on the integrated
   change.
4. Run the Mac checkout's `just docs-check` with `BOB_CLI_DIR` explicitly pointing to
   the current authorized bob-cli workspace, so it checks the mounted changes without
   cloning another repo. Open any additional required source repository through
   `/sase_repo`. Run the applicable bob-cli `just check` gate required by the parent
   epic, and check the final diffs/formatting. Do not repeat a vault-wide sync or modify
   a real reference for this presentation-only phase.
5. For a failure that reproduces identically on a clean base, record a
   `PROPOSED FOLLOW-UP:` note on `bob-cli-bd.8` with base/change commands and evidence
   (and an existing task ID if known); it does not keep this phase open. Fix failures
   introduced here. File no new beads. Record other out-of-scope discoveries using the
   same note format for the parent land agent to triage.
6. Record the two skipped memory changes as required by the epic's DECISIONS. Before
   closing, run `sase bead epic-symbols bob-cli-bd.8` (none existed at planning time).
   Resolve any entries now present, or re-key their Justfile line to an appropriate
   still-open parent/later bead. Then close only the assigned phase:
   `sase bead close bob-cli-bd.8 --note "<implemented behavior and verified checks; explicitly identify any permitted verification limitations>"`.
   Never close `bob-cli-bd` or any ancestor plan bead.
7. Use `/sase_final` as the final action for a normal provider-turn completion; the
   modified linked Mac repository belongs in the finalizer's repository declaration. Do
   not manually commit outside the authorized SASE flow. Report behavior, checks, and
   any Mac/CI/visual limitations clearly to the parent land agent.
