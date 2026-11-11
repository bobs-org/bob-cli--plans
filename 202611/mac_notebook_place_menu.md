---
tier: tale
title: Mac notebook place menu and grouped preview
goal:
  Render Bob-authored notebook section menus, new-section highlighting, and grouped
  notebook previews in Bob Mac Capture with verified macOS compatibility.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ka.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-ka.5
create_time: 2026-11-11 14:39:27
status: wip
---

- **PARENT:**
  [202611/notebook_section_bullets.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/notebook_section_bullets.md)
- **BEAD:**
  [bob-cli-ka.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ka/bob-cli-ka.5.md)

# Mac notebook place menu and grouped preview

Implement the Mac phase of `plan:202611/notebook_section_bullets.md`. A notebook section
bullet (`@dev_tools\n- #Git\n  - Use worktrees`) and a glued notebook section
(`More @dev_tools#Git`) should use the same Bob-authored place menu as drawers.
Accepting a bullet row preserves `#` and puts the caret on Bob's nested-note
continuation. The editor marks newly created sections, and a batch preview shows
consecutive notes for one notebook together with each note's section destination.

## Scope, decisions, and current evidence

This is one bounded implementation in the linked `bob-mac-capture` repository. Open it
with `/sase_repo`, use only its returned path, and read its `AGENTS.md`. Read this plan
and its parent using `sase artifact read`; read the assigned bead with
`sase bead read bob-cli-ka.5 -r "Need current phase scope and notes"` before editing.
Check current code and history again in case another turn landed work.

Planning inspected clean trees at bob-cli `1a9b68289f65f57d0b4a1bc3d9f955f82ae0291a` and
bob-mac-capture `ee26941`. The phase has no previous implementation/remainder notes.
Dependencies are closed: grammar 26 landed at `0f37849b`, completion chrome at
`53218b17`, and the notebook-aware slash alias at `f7092004`. The Mac feature has not
landed.

The accepted decisions remain final:

- `create_row_return = preselect_clean`: an otherwise clean lone create row can be
  accepted with Return; a near-miss create row stays unselected. Bob supplies this
  through `initial_selection`, not a Swift near-miss classifier.
- `typed_capitals = yes`: display Bob's title verbatim, including `API Keys`.
- `memory_decision_records = no`; glossary edits belong to the separate memory phase.
  This implementation changes no memory or generated `AGENTS.md`.

Read `decisions:mac-capture-is-a-thin-client` through `/sase_memory_read`. Bob remains
the sole source of route classification, grammar, selection policy, replacement text,
write consequences, and vault writes. The app decodes and paints. Do not add filesystem
reads or notebook parsing in Swift.

Existing scaffolding substantially bounds the work:

- `CaptureCompletionCandidate` already decodes `kind`, `group`, `line`, `note_count`,
  `detail`, `consequence`, `creates`, `refused`, and `cursor_after`. Reuse them; cover
  notebook payloads rather than creating duplicate properties.
- `CaptureNotebookSummary` already decodes `section` and `section_action`.
- `shouldRequestCompletion` already includes need `section` and span `section`. Verify
  those paths instead of adding a lexical bullet detector.
- Generic acceptance already uses `cursor_after`. Existing drawer continuation, refusal,
  alias, and hosted panel tests provide the integration pattern.
- The missing surface is `place_menu`; the drawer presentation is hard-coded,
  `applyNonActiveTaskCompletion` discards empty responses, `completionVisible`
  independently requires rows, and refreshes reset selection.

The phase note records a source-backed upstream issue: Bob's glued `section` context
currently checks candidate rows to decide whether to emit notebook chrome, so an empty
notebook may omit `place_menu`. Implement empty-hint handling whenever Bob supplies it.
Do not invent notebook identity or chrome to compensate; leave upstream repair and its
CLI regression to the parent epic's land agent.

## 1. Decode the additive protocol

In `Sources/CaptureCore/CaptureModels.swift`, add an optional place-menu model to
`CaptureCompletionResponse`, including initializer plumbing, coding keys, and
`decodeIfPresent`. Its fields are `kind`, `title`, `caption`, ordered
`groups: [{id, title}]`, `new_row_prefix`, optional `empty_hint`, `results_label`, and
`footer_hint`. Preserve omitted-field defaults and unknown string values; older Bob
responses must still decode and render.

Add optional `notebook` and `section_action` metadata to `CaptureSpan`; `title` already
exists. Add `notebookSectionBullet` to `CaptureCompletionContext` in
`CompletionRowContent.swift` and update all exhaustive switches and portable row
fallbacks. Treat notebook section rows as section-colored rows.

Test both real grammar-26 responses and explicit legacy payloads missing these fields.
Updating a fixture must not erase coverage of the old drawer contract.

## 2. Generalize the drawer presentation and view

Generalize `CaptureDrawerMenuPresentation` and `CaptureDrawerMenuView` into one place
presentation/view. Renaming them is appropriate; update all production and test
references together. Keep fallback drawer copy for `context == "drawer"` without
`place_menu`. A non-notebook `section` without chrome keeps its current generic
completion surface.

Use the presence of `place_menu` to select the new surface. Render its title, caption,
ordered group headers, prefix, result noun, empty hint, and footer hint verbatim.
Preserve Bob's row order and candidate indexes; empty group titles are untitled groups.
Unknown group IDs must not drop rows. Preserve the old `drawers`/`task` grouping
behavior for legacy drawer payloads.

The real current wire value of `new_row_prefix` is only `"＋ "`; candidates carry labels
such as `New section “Docker”`. Compose those once. Do not assume the prefix already
says "New section", add a second plus, or title-case Bob's text. Selected-row footer
precedence remains refusal, consequence, then Bob's footer hint. Use old unselected and
selected copy only in the legacy drawer fallback.

Drawers retain `tray.full`; notebook rows use a section glyph with the existing section
color. Draft-origin rows show a small `new` capsule using Bob's supplied detail; create
rows retain `plus.square.dashed` and the dashed treatment. Refused rows remain visible
and dimmed. Return, Tab, and click on a refused row consume the action without changing
or submitting the draft; its footer explains why.

Build accessible row/list labels from the destination caption, Bob's title, detail,
consequence, and refusal. Preserve a clear existing/new distinction without generating a
Swift interpretation of the write. Keep captions and tooltips consistent with those same
decoded strings.

In `CapturePanelView.swift`, replace all drawer-specific routing together: completion
body, viewport height/group-header count, generic-list suppression, footer, and
accessibility. Keep its position among the existing specialized surfaces. An empty hint
needs real height and visibility, including when the alias tip is present and on short
screens.

## 3. Integrate menu lifetime, selection, and acceptance

In `CapturePanelModel.swift`, retain an empty completion when its decoded
`place_menu.empty_hint` is nonempty. Update `completionVisible` as well as
`applyNonActiveTaskCompletion`, and audit downstream candidate-count guards so the empty
surface actually renders. It must still dismiss on Escape and when the caret leaves the
field. With no candidates, navigation/acceptance must be safe and must not swallow
ordinary typing or fabricate a choice. Preserve warning handling and
stale-generation/draft checks.

Honor `initial_selection: "none"`. When Bob allows selection, choose the first enabled
ordinary section row, or the enabled create row when no ordinary row exists. Do not
inspect `detail` text for near-miss logic: Bob already asks for no selection in that
case. Apply this narrowly to notebook place menus so other contexts retain their current
selection semantics.

Preserve an explicitly active row across refreshes of the same menu using
`group + title + kind`, not array index or replacement text. Scope preservation to the
same destination/context/field; clear it on dismissal or a different menu. If the row
disappears or becomes refused, fall back to Bob's initial selection. Do not resurrect a
selection for a fresh empty query whose response asks for none. Avoid changing the
global candidate `id` for unrelated pickers.

Use Bob's `query` for notebook match emphasis when present and keep legacy drawer
extraction as fallback. Add regressions showing that need `section` opens the menu at a
bare bullet hash, a caret inside a `section` span requests completion, and
`section_notes` allows the user to type the nested note.

Acceptance must reuse the generic UTF-8 replacement/`cursor_after` splice. Verify a
bullet row inserts `#Git\n  - ` while preserving surrounding text and the leading hash;
a glued row inserts no continuation. Exercise quotes and a multibyte title. `/` must use
Bob's rewrite response and converge on the same menu; pasted slash, same-key double-tap,
Escape, and Undo keep their contracts.

## 4. Paint new sections and group notebook previews

Extend `paintHighlighting` with a subtle dotted underline only for a notebook `section`
span whose `section_action == "create"`. Keep its existing purple section color.
Repainting an existing section must clear the underline and any new owned attributes;
avoid affecting drawer, theme, and diagnostic styling. Expose the resolved title and
"new section" in editor VoiceOver/help text using the existing editor presentation
boundary, without modifying draft bytes.

Extend `CaptureNotebookPresentation` with portable grouping/headline helpers. Group only
consecutive `notebook_note` results with the same nonempty `notebook.route`; preserve
source order and original item indexes. Missing metadata, route changes, ordinary
captures, notebook creation, and sub-note captures form boundaries. Do not group
separated occurrences of the same route.

Render one block headed `📓 N notes → Title`, followed by each note's content and
destination: `› Git`, `› Docker` plus `New section`, or `· top`. Use Bob's per-note
`section_action` for the badge and preserve named-note identity and authored children.
Keep `CaptureNotebookPresentation` the shared source of headline/copy. Preserve the
existing single-item notebook and creation view.

Integrate the grouping into `CapturePanelView`'s preview batch iteration beside the
current past-Pomodoro runs and list groups. Suppress covered rows only once; avoid
duplicate list headers/dividers and keep each non-notebook preview's existing behavior.
Preserve `pairIndexed` file pairing by original item index so notebook/record
create-then-use batches still show the correct Markdown. Extend
`CaptureNotebookPreviewView.swift` or add a small grouped view consuming the portable
presentation, with useful VoiceOver ordering.

## 5. Real fixtures, focused tests, and docs

Build Bob from a current bob-cli master checkout containing the dependency commits; use
the current authorized workspace or `/sase_repo` for another checkout. Pass that
executable through `BOB_BIN` and its exact full SHA through `BOB_COMMIT`. Use the
generators' isolated scratch vault/XDG paths, never the real vault. Extend and run both
`Scripts/generate-notebook-fixtures.sh` and `Scripts/generate-drawer-fixtures.sh`;
generated JSON is not hand-authored. Record provenance in both fixture READMEs.

Add fixture scenarios for section-bullet and glued menus, an empty bullet menu,
existing/draft/new groups, a clean create, a near-miss create, refused duplicate
headings, staged `%n` sections, parse create spans, slash rewrite, and a batch with
existing/new/top destinations. Add corresponding `Tests/Fixtures/fake-bob` dispatches.
Keep generated source drafts discoverable in the script/README.

Extend the existing suites (rename drawer suites with the implementation):

- `CaptureDrawerMenuPresentationTests`, `CompletionRowContentTests`, and
  `CaptureCompletionMenuTests`: decoded chrome and row order, one prefix, refusals,
  Bob's strings, empty state, clean/near-miss selection, and legacy compatibility.
  Include a refused row before an enabled row.
- `CaptureDrawerMenuHostedTests` and `CaptureHashAliasDoubleTapTests`: real completion
  request paths, the section continuation twin, stable selection across
  reordering/changed replacements, destination reset, stale replies, empty hint
  visibility/Escape, and notebook slash behavior. Hosted async tests require bounded
  waits, never untimed continuations.
- `CaptureEditorHighlightTests`: create versus existing, Unicode byte ranges, attribute
  cleanup, help/accessibility description.
- `CaptureNotebookPresentationTests` and `CaptureNotebookPreviewDesignTests`: note
  counts, ordered section/top/new rows, route and kind boundaries, missing-metadata
  fallback, mixed create/use file pairing, and hosted layout and accessible content.

Read docs guidance through `/sase_memory_read` before editing. Update the linked repo's
`docs/capture-panel.md` (Place menu, grammar-26 behavior, stale grammar-3 wording),
`docs/capture-previews.md` (grouped notebook preview), `docs/protocol.md` (grammar 26,
context, chrome, span modifiers, older-Bob fallbacks), `docs/keyboard.md`, and
`docs/typing-tips.md` (`hash_menu`). Update internal drawer-menu links and preserve the
published `#drawer-menu` anchor when renaming the heading. No CLI flags, versions, or
release files change.

## 6. Verify, land, and finish this phase

Format only changed Swift lines. Run focused portable tests while implementing, then
`just portable-check`. Point `BOB_CLI_DIR` at the authorized current bob-cli checkout
and run the linked repo's `just docs-check` so its wrapper does not discover an
unrelated checkout or clone a cache. Check changed fixture generator shell syntax and
the fake-Bob dispatches.

Portable checks cannot compile AppKit/SwiftUI targets. Follow the linked `AGENTS.md`:
read tailnet memory before remote access, attempt a disposable scratch copy on the Mac
excluding `.build` and `.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, and
delete the scratch copy. Never use Bryan's working checkout or install an app there. Do
not use `--build-tests` against Command Line Tools without XCTest. If the Mac is
unavailable, record that fact and rely on the mandatory macOS CI gate to compile and run
the actual targets.

Before landing Swift, run `Scripts/macos-ci-check` in the linked repo through
`/sase_monitor`. It snapshots the current tree onto a unique temporary branch, verifies
the exact SHA's required macOS workflow steps, and cleans up that branch. Put
inspection/fixes and completion in the monitor's follow-up. Wait for the monitor staging
command itself to exit, then finish the turn through `/sase_final`; do not wait for the
staged command in that turn. After every subsequent push, also watch the exact pushed
SHA's macOS run through `/sase_monitor`. A formatting failure may hide later skipped
compile/test gates; require evidence of the actual Build/Test/Bundle coverage before
calling it verified. Never land uncompiled Swift.

Use `/sase_final` for repository commit declarations; do not invoke raw Git commits or
the manual commit skill. Keep the assigned bead open during an intermediate verification
handoff. If a failure reproduces identically on a clean base, record its evidence as
`PROPOSED FOLLOW-UP:` on this phase, citing an existing tracking bead if known, and
follow the user's baseline-failure exception rather than leaving the phase stranded. An
unrun compile is not a clean-base test failure. Fix failures introduced by this work.

When the complete Mac scope is verified, run `sase bead epic-symbols bob-cli-ka.5` again
(none existed during planning). Resolve leftovers or re-key them to a still-open owner
before closing. Close only `bob-cli-ka.5` using
`sase bead close bob-cli-ka.5 --note "<verification summary with linked-repo SHA and exact macOS run evidence>"`.
Do not set status by hand, reopen the bead, create task beads, or close `bob-cli-ka` or
any ancestor plan bead. Record discoveries as phase `PROPOSED FOLLOW-UP:` notes for the
parent land agent. Finish through `/sase_final` with every edited repository
represented.

Completion means both notebook entry forms use Bob's menu, continuation and selection
obey its response, empty hints remain visible, new-section styling and grouped previews
agree with Bob's results, older Bob still works, fixtures and docs describe the shipped
behavior, and compiled macOS evidence is recorded.
