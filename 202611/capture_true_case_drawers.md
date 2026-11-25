---
tier: epic
title: True-case capture drawers and one place resolver
goal:
  Complete bob-cli-ph.6 with shared place resolution, bold container drafts, preserved
  titles, accurate completion and preview metadata, and verified capture grammar and
  documentation.
phases:
  - id: shared-place
    title: Extract the shared place lexer and resolver without changing behavior
    size: medium
    depends_on: []
    description:
      "shared-place: introduce capture_language/place.rs with TaskDrawer,
      ProjectSection, and NotebookSection adapters. Consolidate selector lexing and
      matching while preserving existing scope policies and passing existing suites
      unchanged. Remove obsolete selector helpers only after auditing their callers."
  - id: bold-drafts
    title: Admit bold container drafts across all four contexts
    size: medium
    depends_on:
      - shared-place
    description:
      "bold-drafts: replace ALL-CAPS authored container detection with whole-bold
      headers for task drawers, project-note sections, notebook creation seeds, and
      record groups. Preserve literal titles, add delimiter/title spans and the
      temporary caps_drawer_retired refusal, keep notebook route-only behavior, and make
      bullet menu replacements valid bold drafts. Advance grammar 29 to 30."
  - id: place-titles
    title: Preserve selected titles and prefer explicit catalog spellings
    size: medium
    depends_on:
      - bold-drafts
    description:
      "place-titles: complete the shared create-title rule for quoted, glued, flag, and
      slug selectors; use explicit catalog spellings and notebook-style fallback. Route
      completion and refusal decisions through the same resolver and prove replacement,
      parse, preview, and commit agree for every menu row kind."
  - id: drawer-editor-wire
    title: Add drawer typing expansion and semantic preview annotations
    size: medium
    depends_on:
      - place-titles
    description:
      "drawer-editor-wire: implement expand_drawer_marker through hypothetical parser
      admission, add marker provenance to drawer objects and drawer annotations to
      preview header lines, and cover offsets, before/after snapshots, and excluded
      contexts. Keep Mac keystroke handling and fixture regeneration in bob-cli-ph.8."
  - id: capture-acceptance
    title: Verify the complete capture contract and publish its documentation
    size: medium
    depends_on:
      - drawer-editor-wire
    description:
      "capture-acceptance: run the cross-context scratch-vault acceptance matrix, finish
      capture and JSON documentation, and run formatting, docs, and repository gates.
      Record evidence and unresolved baseline failures for the child epic land agent,
      which alone verifies and closes bob-cli-ph.6."
proposed_by: bbugyi200.apollo.bob-cli-ph.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ph.6
create_time: 2026-11-25 08:39:50
status: wip
---

- **PROMPT:**
  [prompts/202611/capture_true_case_drawers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_true_case_drawers.md)
- **PARENT:**
  [202611/true_case_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/true_case_drawers.md)

# Capture drawers with their real titles

Implement the remaining capture work of `bob-cli-ph.6`, under the accepted
`plan:202611/true_case_drawers.md`. The result is a single place-selection contract,
bold authored container headers, case-preserving creation, and semantic editor and
preview data for the Mac phase.

## Scope, evidence, and ownership

The assigned phase was read before planning. It is in progress and has no prior
implementation notes or child plan. Its dependency `bob-cli-ph.4` is closed. The
checkout already contains `8f88a52c`
(`feat(task-drawers): emit bold drawer headers from every writer`), so do not redo bold
serialization or reader rollout.

Inspection confirms the remaining work:

- `capture_drawers.rs` already has `DrawerMarker`, Unicode `drawer_key`, catalog version
  3 with per-spelling marker counts, whole-bold recognition, `can_create_drawer`, and
  bold header writers. Its capture literal normalizer, resolver creation paths, and
  catalog selection still uppercase titles or apply legacy project Title Case.
- `capture_language/drawer_bullet.rs` and `section_bullet.rs` duplicate leading selector
  lexing. `capture_language/notebook.rs` resolves sections separately.
- Task, project, notebook-seed, and record parsers still infer containers from ALL-CAPS.
  Drawer bullet completion still inserts a bare uppercase title.
- `GRAMMAR_REVISION` is 29. `SpanKind` has `DrawerBullet`, but no `DrawerMarker`.
  `RewriteRule` has the requirement expansion but no drawer expansion.
- `capture/output.rs::DrawerJson` has title/key/created without marker provenance.
  `capture/block_diff.rs::BlockLineJson` has requirement annotations but no drawer
  annotation. These are the existing extension points.
- `sase bead epic-symbols bob-cli-ph.6` currently reports no entries.

The scope merits an epic because selection, authored parsing, completion, and
editor/preview contracts each have substantial existing test surfaces. Each phase below
is bounded to one implementation agent. Dependencies are deliberately serial because the
parser, completion, and documentation share files and contracts.

Read the parent plan and `research:202611/true_case_drawers/true_case_drawers.md` with
`sase artifact read`. The approved parent plan takes precedence over the research where
it differs: ALL-CAPS-with-notes is a **blocking refusal until cutover**, legacy-only
catalog spellings do **not** seed new uppercase names, and readers rolled out before
writers.

All implementation here is in bob-cli. This plan does not edit the vault, plugins, Mac
source/fixtures, release versions, CHANGELOG, or memory. The accepted `tray` glyph
decision remains final; memory decisions belong to the parent epic's memory phase. No
new decisions are needed. Keep legacy disk readers and the legacy DH title-mapping
branch until the parent cutover phase.

Child workers close only their own assigned phase beads. The child epic's **land agent**
is the designated finisher of `bob-cli-ph.6`: after its descendants close, verify the
full scope, run `sase bead epic-symbols bob-cli-ph.6`, resolve/re-key any remaining
entries, and close only that original phase with verified evidence. Never close
`bob-cli-ph` or another ancestor. Record discovered work as `PROPOSED FOLLOW-UP:` notes
on the worker's assigned bead; create no task beads.

## Contract to implement

### Selection and titles

`capture_language/place.rs` owns shared selector lexing and selection, with explicit
scope policies for `TaskDrawer`, `ProjectSection`, and `NotebookSection`. Records
consume the bold body representation but need not gain a fourth resolver adapter.

The ladder is quoted exact selection, key-form selection, unique slug selection where
the scope already supports it, ambiguity with source lines, reserved or structural
refusal, near-miss refusal with a quoted override, then creation. Keep existing
managed-name guards at their necessary precedence: a structural project heading cannot
become eligible simply because it matches. Do not silently change scope-specific
matching, slug availability, reserved sets, diagnostic nouns/codes, or near-miss
thresholds (drawer prefix/edit distance 1; notebook prefix/distance 2). Task case twins
remain ambiguous even when one exactly matches the typed spelling.

Use the existing Unicode whitespace-collapsed `drawer_key` for drawer identity.
Store/display authored titles independently. Exact literal creation collapses whitespace
but preserves capitalization and punctuation. Unquoted creation first uses a catalog's
preferred **explicit** spelling (bold or project section) when the scope has a catalog;
a legacy-only key falls back to `format_notebook_section_create_title`. With no catalog
candidate, that same rule preserves words containing typed capitals and title-cases the
other words: `API-keys` becomes `API Keys`, `git` becomes `Git`.

Keep deterministic catalog ranking/tie breaks. Marker metadata, rather than the
spelling's appearance, decides whether a spelling is explicit; a deliberately authored
bold `**API**` is eligible. A frequent legacy spelling cannot defeat an available
explicit spelling. Preserve the target's actual title and marker when selecting an
existing drawer, including legacy drawers. Never rename or re-mark an existing header
merely because a note is appended.

Selection is shared; mutation stays destination-specific. Task indentation/block
geometry, project H2 insertion, and notebook H2 insertion remain their own writers.

### Authored body matrix

| Context                                                    | Whole-bold first-level header with nested notes        | Leading `#`                      |
| ---------------------------------------------------------- | ------------------------------------------------------ | -------------------------------- |
| New task, including typed/Pomodoro task                    | Task drawer, title preserved                           | Drawer shortcut                  |
| Note-less `@route+id`, including an eligible `^prj` target | Merge by key or create drawer/eligible project section | Same target resolver             |
| New project note `@parent^id@` / offer form                | Verbatim `## Title`                                    | Project-section shortcut         |
| New notebook `%n` seed                                     | Verbatim `## Title`                                    | Notebook-section shortcut        |
| Record capture                                             | Verbatim record section group                          | Preserve existing record grammar |
| Route-only existing `@nb` list                             | Ordinary note with nested detail                       | Notebook section selector        |

These are four creation families; new and existing task targets are two forms of the
task family. Same-line existing-task notes and already-glued drawer items do not acquire
another container layer. Whole-bold nested content remains a note.

Reuse the existing title recognizer/creation policy. Valid task drawer titles are
nonempty single-line titles, contain an alphanumeric, contain no `*`, have no interior
edge whitespace, and do not end in `:`. Reserved and managed whole-bold lines stay notes
where the recognizer says they are notes; explicit attempts to select/create a forbidden
drawer retain the appropriate refusal. Checkbox lines, fields, trailing tags/IDs,
quoted/fenced text, and ordinary bold prose are not promoted into containers. Keep
project/notebook/record structural policies explicit.

An eligible whole-bold container with no notes gets `empty_drawer_bullet`, teaching the
nested `  - …` form and the `__Title__` escape. Under-task requirements still use their
existing grammar and `req_in_drawer`; Notes is born only with its first note. A
childless header already on disk remains a valid target to fill.

Until the later cutover, legacy ALL-CAPS-with-nested-notes in a container position gets
`caps_drawer_retired` and a concrete bold replacement hint. Prefer the existing target
spelling on a matching key; otherwise use the new creation spelling. Do not silently
file it as a task, section, or loose legacy drawer. Do not extend this refusal to the
route-only notebook note/detail case or ordinary non-container text.

### Completion, editor, and preview

- Bullet drawer rows insert `**Title**`, plus `\n  - ` only when no nested child
  follows. Glued selectors retain their `#slug` / quoted-selector syntax. Preserve
  correct replacement ranges and UTF-8 cursor offsets.
- Existing, recent, and new rows use the same resolver as commit, including refused
  rows. Displayed intent, applied replacement, parse, dry run, and committed target must
  agree. A plain notebook section menu keeps its notebook syntax.
- Add `drawer_marker` spans for each opening/closing `**`; `drawer_bullet` spans cover
  title bytes only. The parser remains the authority on context admission.
- Add marker provenance to serialized drawer objects in parse/dry-run output; retain
  `capture-drawers`' existing marker contract. Markers describe the resolved destination
  (`bold`, `legacy_caps`, `section`), not merely the typed shortcut.
- Annotate task preview header lines with `drawer: {title, key, marker, empty}`. Omit
  the field for ordinary lines. Recognize against real task/note snapshots, including
  empty headers and eligible `^prj` sections, rather than testing strings for stars.
  Removed lines use the old snapshot; changed/added/unchanged lines use the final
  snapshot.
- `expand_drawer_marker` transforms a lone `*` on an empty eligible bullet into `**|**`.
  Support a closing star already supplied by editor auto-pairing. Admit the edit only
  when the same parser accepts a hypothetical bold title plus nested note in that
  position. No cursor means no typing assist. Do not rewrite paste, templates, or
  ordinary `bob capture` submissions. Respect the `*task` blocking sigil,
  multiplication, emphasis, nested notes, and notebook route-only content.
- Smart Enter, overtype, atomic Backspace, rendering, and Mac fixture regeneration
  belong to `bob-cli-ph.8`; this phase supplies the spans/rewrite response they need.

## Implementation phases

### shared-place

1. Map all callers of `resolve_drawer_for_write`, `resolve_h2_for_write`,
   `resolve_notebook_section`, the two leading scanners, and glued selector token
   parsing. Read current tests before extracting.
2. Introduce the pure core in `src/native/capture_language/place.rs`. Use shared
   selector/result types carrying title, key, source line, and outcome; adapt existing
   public/internal result types at their boundaries to avoid unrelated wire churn. Scope
   policy retains today's differences, including the broader notebook token syntax. Keep
   I/O, catalog loading, and mutation outside the core.
3. Share lexing across `drawer_bullet.rs`, `section_bullet.rs`, and applicable glued
   parsing in `tokens.rs`/`editor_classify.rs`. Quoted escapes, incomplete selectors,
   smart quotes, escaped hashes, and byte ranges must retain current behavior.
4. Preserve pre-change casing in adapters for this refactor. Remove dead
   `select_drawer*`, `match_drawer`, and `suggest_drawer` helpers only after a complete
   caller audit; do not remove live public command functionality.
5. Run existing drawer, project-H2, notebook resolver and selector grammar suites
   unchanged. Add meaningful cross-scope tests for precedence, ambiguity, reserved
   names, and each near-miss threshold. This phase changes no user-visible grammar.

### bold-drafts

1. Switch authored container parsing in `drawer_bullet.rs`, `editor_parse.rs`,
   `capture/plan.rs`, and `capture_project_note.rs` to the shared bold title
   representation. Direct bold titles must survive as authored; merge by key and keep
   existing target spelling. Use shared exact-title resolution rather than a second
   matching path. Adjust literal normalization where needed to preserve quoted/exact
   titles instead of uppercasing them.
2. Apply the same representation in `section_bullet.rs` (replace the seed-only
   `all_caps_as_section` behavior), `capture_language/record.rs`, and
   `records/render.rs`. Preserve record fields, grouping, notebook seed planning,
   project task IDs/moves/requirements, and batch atomicity. Remove legacy title-case
   conversion from these authored paths; retain its legacy DH caller.
3. Add shared empty/caps diagnostics and parser spans, keeping parse and execution on
   the same admission rule. Incomplete empty `****` typing states should remain
   editable, with useful needs/diagnostics rather than a fabricated container write.
4. Update the basic drawer bullet replacement to bold when grammar admission lands, so
   existing completion tests remain meaningful. This is the narrow spelling change;
   catalog preference and the full menu invariant belong to `place-titles`.
5. Bump `GRAMMAR_REVISION` from 29 to 30 and update dependent grammar assertions. If
   another accepted change already advanced it, inspect that change and preserve
   monotonicity rather than resetting the value. Mark behavior-breaking commits
   `feat(capture)!`. Update the shipped draft behavior in `docs/task-drawers.md`.
6. Verify task new/existing/`^prj`, project creation, notebook seed, and record cases;
   ordinary existing-notebook lists; empty/caps refusals with no writes; unchanged
   requirement refusals and placement; and Unicode title/delimiter byte ranges.

### place-titles

1. Finish the common creation rule and remove remaining capture-only uppercase or legacy
   project-title transformations in drawer literal, CLI `--drawer`, glued token, and
   editor-title paths. Keep Pomodoro/type casing untouched.
2. Use catalog-v3 `spelling_markers` to choose explicit spellings. Retain the cache
   version and scan geometry already delivered. If an existing cached field depends on
   the old preference, recompute the presentation from marker counts or invalidate that
   derivation correctly; never consume a stale uppercase display field blindly.
3. Update `capture_complete/drawers.rs`, relevant `candidates.rs` paths, and resolver
   warnings/refusals. Ensure new project targets no longer use obsolete ALL-CAPS literal
   validation. All accepted replacements must pass the writer's validation. Keep
   existing-target selection distinct from creation eligibility: legacy names such as
   `NOTES:` remain targetable where the existing selector grammar allows them, but
   cannot be created as invalid bold headers. A bullet replacement that cannot express
   its legacy target must be refused with useful copy rather than silently becoming a
   note or a different drawer.
4. Add round-trip tests covering task/recent/new rows, fresh/existing/project targets,
   matching legacy and bold drawers, duplicate key variants, quoted punctuation,
   near-miss overrides, catalog explicit-vs-legacy ranking, and legacy-only fallback.
   Include all three resolver scopes and route-only notebook completion regression.
5. Document the shared resolution and creation rule in `docs/task-drawers.md` and
   relevant capture contract sections as it ships.

### drawer-editor-wire

1. Add `RewriteRule::ExpandDrawerMarker` in `capture_language/rewrite.rs`, following
   `expand_req_marker`'s parser-verified pattern. Extend route-policy loading at the
   command boundary only if needed; rewrite remains free of filesystem I/O.
2. Test lone/autopaired stars, cursor positioning, Unicode before the edit, no-cursor
   calls, every eligible creation family, note-less task targeting, and all excluded
   positions. Keep ordinary sigil alias/requirement rewrite behavior green.
3. Extend `DrawerJson` and parse representations additively. Source `marker` from actual
   selection/creation, including legacy targets and project H2s.
4. Add optional preview drawer metadata to `capture/block_diff.rs` and annotate it where
   `capture/task_blocks.rs` builds real task previews. Follow the existing requirement
   before/after annotation pattern. Check shared line-model consumers
   (`pomodoro_blocks`, record previews) without marking non-task headings as drawers.
5. Cover added, unchanged, changed, removed, and empty drawer headers, duplicate text at
   unrelated depths, CRLF/tab indentation, and normal lines with omitted fields. Dry-run
   and committed responses must describe the same destination.
6. Update `docs/reference/json-outputs.md` and the capture rewrite contract for the
   additive fields/rule. Record exact wire examples for the parent Mac phase.

### capture-acceptance

1. Re-read `bob-cli-ph.6` and all child evidence. Audit remaining capture references to
   ALL-CAPS casing, `is_project_section_title`, `format_project_section_title`, dead
   selector helpers, and stale menu/span descriptions. Keep legitimate legacy
   reader/DH/Pomodoro/type uses, with comments that explain the boundary.
2. Run the acceptance matrix below using temporary vault fixtures. Finish behavioral
   gaps within scope; record unrelated discoveries on the assigned phase bead.
3. Update `docs/capture.md` (drawer, sub-bullet, project, notebook section, rewrite
   sections), `docs/reference/capture-syntax.md`, `docs/reference/json-outputs.md`,
   `docs/projects.md`, `docs/notebooks.md`, `docs/records.md`, and
   `docs/task-drawers.md`. Explain the dual-read/retired-draft distinction and the
   notebook route-only exception. Read `docs.md` memory first. No glossary or
   decision-memory edits here.
4. Run `just fix`, `just docs-gen`, `just docs-check`, and `just check`. Use
   `/sase_monitor` for long verification, and wait for staging commands themselves to
   exit. Do not change release numbers or CHANGELOG. Verify generated docs for
   unintended churn. Expand testing only for failures or new concerns.
5. Record commands, results, relevant commit IDs, and concrete acceptance evidence for
   the child land agent. A failure reproduced identically on the clean base tree is a
   `PROPOSED FOLLOW-UP:` with the reproduction and any known task ID; it does not leave
   this phase indefinitely open. Fix failures caused by this work.
6. Before closing any worker bead, run its `epic-symbols` check and resolve/re-key
   entries to an appropriate still-open bead. This phase does not close ancestors.

## Acceptance matrix for the child land agent

Use the existing unit/CLI fixtures under `capture_language/tests`,
`capture_complete/tests`, and `tests/cli/capture`. Add cases to the matching suites; use
real CLI round trips where they verify independent parser/writer boundaries.

- A new task and note-less existing task receive `**API Design**` and retain it exactly.
  Existing `- NOTES` and `- **Notes**` retain their bytes when targeted by any matching
  spelling. Case twins refuse instead of picking one.
- Direct bold and quoted titles `iOS / OAuth 2.0`, `C++`, `Q&A`, `Café`, and `Q1: Goals`
  work in every eligible family; `Straße`/`STRASSE` pin Unicode drawer identity. No
  output derives display text from a key.
- Project capture produces `## API Design`; notebook creation produces `## Git`; record
  grouping preserves `API Design`. An existing notebook route-only list preserves a bold
  note with detail, while `- #Git` still selects a section.
- Childless valid bold containers refuse; empty existing drawers can be filled.
  `__Title__`, bold leads/trailing colons, checkbox lines, managed lines, nested bold
  notes, and content outside container positions retain their intended meanings.
- ALL-CAPS-with-notes is refused only in the designated creation contexts, with a usable
  hint and no staged/committed partial writes. Legacy disk drawers still resolve and
  never get re-marked on append. Requirement tests still pass.
- Explicit catalog spellings beat legacy spellings; legacy-only/no-catalog creation
  yields the notebook title rule. Quoted creation and `--drawer` preserve case.
  Near-miss messages teach a quoted override that creates the advertised title.
- Apply every completion row kind and verify parse, dry-run JSON, and final bytes agree
  on key/title/target. Validate replacement/cursor ranges with non-ASCII input.
- Validate title and both delimiter spans; rewrite returns `**|**` only in admitted
  positions. Preview drawer annotations have correct title/key/marker/empty values for
  before and after lines and are absent on ordinary notes.
- Grammar and JSON documentation agree with emitted responses; the repository and
  documentation gates pass, or unrelated failures have clean-base evidence.

Only after all of these hold does the child epic land agent close `bob-cli-ph.6` with
its verification note. The parent epic's Mac, migration, cutover, and memory phases
remain owned by their existing agents.
