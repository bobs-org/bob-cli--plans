---
tier: epic
title: Complete Obsidian true-case task drawers
goal:
  Obsidian renders and types tray-marked true-case drawers, writes bold headers, and
  preserves supported drawer titles through verified project conversion, completing
  bob-cli-ph.7.
phases:
  - id: bold-writers-and-conversion
    title: Bold plugin writers and verbatim project conversion
    size: medium
    depends_on: []
    description:
      "bold-writers-and-conversion: switch new plugin drawer writes to bold, refresh
      Rust-owned fixtures, and preserve valid titles and empty sections through
      promotion and reversal."
  - id: drawer-rendering
    title: Task drawer marks in Live Preview and reading view
    size: medium
    depends_on:
      - bold-writers-and-conversion
    description:
      "drawer-rendering: render the approved tray and verbatim title using source-backed
      attributes, owner ink, folded counts, empty and closed states, and a persistent
      marks toggle."
  - id: drawer-typing
    title: Drawer marker typing and Toggle Drawer
    size: medium
    depends_on:
      - drawer-rendering
    description:
      "drawer-typing: add scoped star expansion, Enter, overtype, atomic Backspace,
      undo, and Toggle Drawer beside the existing requirement typing assist."
  - id: obsidian-acceptance
    title: Real Obsidian acceptance, screenshots, and documentation
    size: medium
    depends_on:
      - bold-writers-and-conversion
      - drawer-rendering
      - drawer-typing
    description:
      "obsidian-acceptance: exercise the integrated feature in an isolated Obsidian
      vault, inspect and publish light/dark screenshots, update the behavior docs, and
      record verified deployment evidence."
proposed_by: bbugyi200.apollo.bob-cli-ph.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ph.7
create_time: 2026-11-25 08:48:51
status: wip
---

- **PROMPT:**
  [prompts/202611/obsidian_true_case_drawers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/obsidian_true_case_drawers.md)
- **PARENT:**
  [202611/true_case_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/true_case_drawers.md)

# Complete Obsidian true-case task drawers

This child epic implements the remaining scope of **bob-cli-ph.7**, the
`obsidian-drawers` phase of `plan:202611/true_case_drawers.md`. The result is a task
drawer whose authored title survives writing, display, promotion, and reversal:
`- **API Design**` becomes `## API Design` and returns unchanged. A quiet tray replaces
the list bullet, and typing `*` at an eligible empty child bullet creates `**|**`.

The scope needs separate coding agents for serialization/conversion, two editor
integrations, and actual Obsidian verification. Each phase is medium and directly
implementable. Dependencies intentionally serialize changes to shared manifests,
exports, tests, and deployed plugin bundles. This is not a request to rerun the
already-completed reader rollout or to expand the parent epic.

## Governing context and what has landed

Read the parent plan and endorsed research with audited artifact reads:

- `plan:202611/true_case_drawers.md`, especially §§1–3, 6–8 and `obsidian-drawers`.
- `research:202611/true_case_drawers/true_case_drawers.md`.
- `sase bead read bob-cli-ph.7 -r "Need phase status, final decisions, and remaining work"`.

The following was verified while planning on 2026-11-25:

- `bob-cli-ph.7` is in progress, assigned, and has no earlier implementation notes. Its
  prerequisite `bob-cli-ph.4` is closed. The bob-cli checkout includes `8f88a52c`
  (`feat(task-drawers): emit bold drawer headers from every writer`).
- bob-plugins includes `19ac4a3e` (shared dual-read drawer readers) and `63e9e760`
  (DV/DH fixture loading); the inspected tip is `7c343149`. All four reader fragments
  already share recognition, keys, source ranges, and DH title helpers.
- Running
  `node --test scripts/test-task-drawers-twin-copies.cjs scripts/test-task-reqs-twin-copies.cjs`
  gives 19 passes and one failure: `layout.json matches bob-cli's canonical fixture`.
  This is expected work in this phase: bob-cli's writer fixtures have advanced and the
  plugin copies have not. The drawer-specific suite passes, including its explicit
  assertion that fresh plugin Notes writers still produce legacy `- NOTES`.
- `task-status-cycler/src/196-task-block-layout-twin.js` and navigation's
  `696-task-block-layout-twin.js` still set `TASK_BLOCK_NOTES_TITLE = "NOTES"` and
  interpolate it directly into a bullet. Navigation's `111-project-seed-reqs.js`
  separately emits `"  - NOTES"`.
- Promotion's `parseProjectSectionBulletTitle` already accepts bold titles verbatim.
  Keep it. Actual conversion still needs work: the seed requires nested content for a
  drawer section, reversal uppercases titles in `100-yank-and-paths.js`, and
  `210-rename-and-line-context.js` skips empty non-Tasks sections.
- No task-drawer rendering fragment or drawer typing assist was found. Existing
  extension points are ledger-tools' `264-plugin-task-reqs.js` and cycler's
  `199a-req-typing.js`.
- `sase bead epic-symbols bob-cli-ph.7` reports no entries at planning time.

Recheck the live tree and phase notes before implementation; keep anything that has
landed since this evidence. The research sidecar needed a refresh to resolve its
recorded reference; it now resolves at the original reference above.

## Decisions and boundaries

The parent decisions are final: **`drawer_glyph = tray`**. Use a shallow, wide open tray
with a short inner shelf line, approximately 0.95em across. Do not offer another glyph
decision. The parent authorized memory work for its own memory phase; this child epic
edits no memory notes and adds no memory decisions.

Rust remains the semantic authority. Preserve dual-read recognition, Unicode key
folding, reserved/managed exclusions, and legacy title-case promotion until the parent's
cutover phase. New writes are bold; existing bold and legacy headers are reused without
background respelling. Never create an empty Notes home merely because a requirement
exists: it is born with the first note. Explicit empty authored drawers and eligible
empty project sections remain representable.

Keep the current requirement boundary, `req_in_drawer` behavior, task ownership,
successor offers, project conversion rollback, and existing unsupported-body refusals.
This work preserves drawer titles and supported list bodies; it does not implement a
universal project-to-task representation, change capture grammar, migrate the vault,
remove legacy readers, change Mac code, or redesign the property-card drawer.

Read `docs.md` with `/sase_memory_read` before documentation changes. Open `bob-plugins`
with `/sase_repo` and use only the returned checkout, reading its `AGENTS.md` before
edits. Paths below under `plugins/` and `scripts/` refer to that repo unless explicitly
marked bob-cli. Never edit deployed plugin bundles in the vault. Edit source fragments,
keep each at or below 1000 lines, update `src/fragments.json`, and generate `main.js`
with `npm run build`.

Every implementation phase runs its focused tests, `npm run build`, and `npm test`; add
new unit-test files to `test:unit`. Run `npm run validate` when command/settings
inventory or manifests change and regenerate the inventory as required. Sync only
changed plugins with `bob plugins sync -p <id>`, following the repo's non-forcing
deployment conflict procedure. Use `/sase_monitor` for long commands. Declare all
changed repos through `/sase_final`; serialization changes use a Conventional Commit
with `!`. Do not edit bob-cli's Cargo version or CHANGELOG.

## Phase: bold-writers-and-conversion

### Changes

1. Port the established Rust header-writing behavior into the cycler layout twin; keep
   the NOTES identity key separate from the `Notes` title. Centralize bold header
   emission and copy shared twin changes byte for byte into navigation. Audit tidy,
   first-requirement scoop, note insertion, carry/close, task split, and conversion seed
   writers. Reuse the exact existing header when it is present. `^prj` continues writing
   its existing column-zero H2 Notes home.
2. Copy current canonical fixtures from bob-cli's `tests/fixtures/task_drawers`,
   `task_reqs`, and `task_split` to every consumer identified by the existing copy
   checks. Include RL/RW/SP writer expectations, keeping DV/DH/RQ unchanged unless a
   demonstrated semantic gap needs a Rust-first correction. Never edit expected fixture
   bytes independently in a plugin.
3. Finish conversion using `heading_for_drawer` / `drawer_for_heading`:
   - Keep bold titles byte for byte, including punctuation, Unicode, and internal
     whitespace. Use the existing legacy Title Case mapping only for legacy input.
   - New reversal bullets are `- **Title**`, never uppercased. Propagate named
     `not_a_drawer_title` / `reserved` refusals before writes; retain existing generated
     catalog/Tasks handling and unsupported-body refusals.
   - Preserve eligible empty H2 sections and explicitly authored empty drawers. Do not
     drop them because `bodyLines` or `baseIndent` is empty. Trace through
     `111-project-seed-reqs.js`, section insertion in `110-project-seed.js`, and
     reversal planning/serialization in `210-rename-and-line-context.js`.
   - The Notes role still promotes to `## Notes` as specified by the parent; it must not
     create an empty Notes section without an explicit empty Notes drawer.
   - Receipts and candidate labels carry authored titles. Preserve the existing preimage
     checks, rollback, task IDs, requirement structure, and link handling.
4. Leave legacy casing helpers only where legacy behavior actually calls them. Do not
   reintroduce local grammar or casing in receipts or conversion callers.
5. Add a stable bundle comment marker `bob-task-drawer-bold-writes-v1` to the
   implemented writer paths in task-status-cycler and bob-navigation-hotkeys. Record
   these markers for the parent's later writer rollout gate.

### Verification

Run the drawer/requirement twin-copy checks and vector runners, project-from-task,
project-reversal, project creation bytes, promotion requirement, and split suites. Add
assertions for existing legacy header preservation, mixed-case existing headers,
first-note birth, and no duplicate Notes home. Pure DH tests and full supported
promotion/reversal tests cover `API Design`, `C++ Notes`, `Q1: Goals`, `Café`,
`iOS / OAuth 2.0`, and empty sections. Include invalid/reserved headings, managed lines,
fences, quotes, nested bold notes, and LF/CRLF; a refusal leaves all files untouched.
The initial canonical fixture mismatch must be gone, not waived.

Record the changed plugin IDs and passing checks on this phase. Document the shipped
writer/conversion contract in the relevant existing pages; the final acceptance phase
consolidates the full docs.

## Phase: drawer-rendering

### Changes

1. Add small task-drawer model and integration fragments in bob-ledger-tools beside
   `264-plugin-task-reqs.js`. Build rows from the shared contextual recognizer and real
   owner task, not from rendered bold text or a second title regex. Cache by document
   identity, as the requirement renderer does, and rebuild on relevant edits, folds,
   viewport, mode, settings, and owner type/state changes.
2. Use `Decoration.line` in Live Preview and a source-mapped reading-view postprocessor.
   Stamp `data-bob-drawer`, `data-bob-drawer-key`, `data-bob-drawer-count`,
   `data-bob-drawer-empty`, and `data-bob-drawer-owner-state`, plus the owner's
   `data-bob-type-hue` and `data-bob-type-state`. Reuse `taskReqOwnerIdentity` /
   `taskReqTypeAttrs` or extract the shared type-attribute plumbing without changing
   requirement ink. Count direct note entries; nested detail belongs to its entry and
   does not inflate the count. Exclude fence contents from structural counting.
3. Add attribute-scoped `bob-task-drawer-*` CSS. Replace the existing bullet slot with
   the approved tray using a CSS mask and owner hue softened toward muted ink. Keep the
   title semibold (600) and verbatim. Let native Live Preview reveal `**` at the caret;
   no replacement widgets, inserted DOM nodes, text transformations, or custom delimiter
   concealment.
4. Folded drawers show a muted tabular `· n`; empty drawers get a hollow dashed tray and
   muted title; `[x]` and `[-]` owners desaturate the glyph and title. Scope typography
   to the header so nested notes retain normal weight and ink. Preserve native folding,
   list indentation, and the distinct silhouettes of task squares, requirement diamonds,
   Task Link rings, and note dots.
5. Add tooltip and accessible label on existing elements, for example
   `Drawer · API Design · 2 notes · in Upgrade Postgres`. Add a persistent **Drawer
   marks** setting/toggle, default on, beside the existing marks controls. Turning it
   off, switching files/modes, or unloading removes only owned attributes and styles,
   restoring any original labels. It does not mutate Markdown.
6. Register a stable `bob-task-drawer-render-v1` bundle comment marker after the
   implementation is complete.

### Verification

Add model and renderer tests through `ledger-tools-harness.cjs`: both markers,
empty/nonempty/folded states, owner completion/cancellation and hue changes, duplicate
title keys under different owners, fences/quotes, managed and reserved lines, nested
bold ordinary notes, non-ASCII offsets, and setting/unload cleanup. Assert no extra DOM
nodes and no document edits. Cover both Live Preview line attributes and reading-view
source mapping, including repeated titles and stale rendered elements. Run the existing
requirement and task theme suites to protect shared ink. Full screenshots and the actual
Obsidian matrix belong to acceptance.

## Phase: drawer-typing

### Changes

1. Put pure planners and a thin CodeMirror adapter beside cycler's `199a-req-typing.js`;
   reuse its input-event and history patterns. Eligibility is an empty non-checkbox
   bullet directly under a real task, outside frontmatter, fences, quotes, and another
   note/drawer/requirement. Determine the position through the shared contextual reader,
   using a hypothetical valid header where needed; do not require content before
   allowing marker entry.
2. On a typed `*`, produce `- **|**`, handling Obsidian `autoPairMarkdown` both on and
   off without duplicated delimiters. One Undo returns the literal `- *` so the user can
   continue an italic note. Keep this separate from paste, templates, programmatic
   edits, IME composition, nonempty selections, and multicursor edits.
3. Smart Enter at the end of a valid title immediately before closing `**` steps over
   the delimiters and inserts a correctly indented nested note bullet. A typed `*` there
   steps over both stars. Backspace inside empty `**|**` deletes all four stars and
   returns to the empty bullet. Preserve the list marker and indentation, including
   tabs, and do not override unrelated Enter/Backspace handling.
4. Add the **Toggle Drawer** command with no default hotkey. It wraps or unwraps an
   eligible direct child bullet, preserving title bytes and descendants. Reject checkbox
   lines, managed lines, reserved names, invalid/trailing-colon titles, and deeper
   positions with an actionable notice and no mutation. A legacy drawer may be
   deliberately marked bold by this explicit gesture without recasing it; unwrapping
   bold produces its plain title (which can still be a legacy drawer during dual-read if
   it is ALL-CAPS). Do not silently recase or invent an escape.
5. Preserve requirement typing and Vim normal-mode bindings; support Vim insert mode
   through the same adapter. Use the parent's `- *` plus Space fallback only if a real
   Obsidian reproduction demonstrates that immediate-star expansion cannot reliably
   compose with autopair. Record that evidence and update tests and help together
   instead of silently changing the trigger.
6. Add `bob-task-drawer-typing-v1` to the completed bundle and expose needed pure
   helpers through the existing test exports.

### Verification

Add a typing suite modeled on `test-task-status-cycler-req-typing.cjs`, covering
autopair on/off, exact document and caret after every key, Undo/Redo, overtype, atomic
Backspace, Enter indentation, ordinary italic outside the position, non-eligible
contexts, and Toggle Drawer refusal/no-write behavior. Include caret movement/edit
disarming, paste/programmatic input, selection/composition, and Vim insert versus
normal-mode behavior. Run the existing requirement typing/toggle suites. Acceptance
verifies actual native autopair and history, not just stubs.

## Phase: obsidian-acceptance

1. Add `scripts/obsidian-task-drawers-e2e.cjs` and bounded helper fragments modeled on
   `obsidian-task-reqs-e2e.cjs` and `obsidian-task-reqs-look-e2e.cjs`. Use the owned
   scratch vault/profile, CDP, Xvfb, exact fixture setup, bounded waits, report output,
   and cleanup conventions. Add an offline self-test and register its unit wrapper.
   Never run acceptance mutations against the live vault.
2. Exercise real key events with `autoPairMarkdown` on/off and Vim on/off, including
   expansion, Undo, Enter, overtype, Backspace, Toggle Drawer, refusal cases, and
   unchanged requirement typing. Inspect both caret and saved bytes after actions.
3. Exercise the existing project conversion commands end to end on supported
   scratch-note bodies with `API Design`, `C++ Notes`, `Q1: Goals`, `Café`, and an
   eligible empty section. Verify saved output and receipts, not only pure helper
   results. Include a named refusal that makes no write.
4. Render Live Preview and reading view in light and dark themes. Put open, folded,
   empty, done-owner, and cancelled-owner drawers beside a requirement, ordinary note,
   and Task Link. Check labels, count updates, owner hue, setting-off fallback, native
   delimiter reveal, and native fold behavior. Measure the marker slot and indentation;
   visually inspect screenshots for inherited bold/strike leakage, shape distinction in
   grayscale, and title legibility. Correct observed defects.
5. Publish the screenshots and structured report with `/sase_artifact`; read its skill
   before creating artifacts. Attach refs to this phase and to bob-cli-ph.7 with a
   concise acceptance note. Screenshots are genuine Obsidian output, not generated
   mockups. Store deploy evidence with changed plugin IDs, exact plugin commit(s), and
   each emitted marker so `bob-cli-ph.9` can run its writer gate. If the current turn's
   host commit is not yet available, have the land agent resolve that SHA after
   finalizers settle instead of recording a guessed value.
6. Update existing plugin references (ledger marks, cycler typing, navigation conversion
   and any generated inventory) and bob-cli `docs/task-drawers.md`
   (rendering/typing/writers) and `docs/projects.md` (conversion). State the limited
   title round-trip guarantee and dual-read behavior accurately. Preserve concurrent
   capture-phase documentation; do not claim ALL-CAPS cutover shipped. Do not add bulk
   docs or make unrelated information architecture changes.
7. Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check` in
   bob-plugins; run `just docs-gen`, `just docs-check`, and `just check` in bob-cli, per
   the parent gate requirements. Use `/sase_monitor` for long runs. Refresh the changed
   plugin deployments with the documented per-plugin sync.

If a gate fails, repair failures caused by this work. A failure reproduced identically
on an unchanged clean base is recorded with command, baseline revision, diagnostic, and
any existing task reference as `PROPOSED FOLLOW-UP:` on the assigned phase; it does not
keep otherwise-complete work open. `bob-cli-ph.4` already recorded clean-base docs
word-budget failures in `plugin-apis.md` and `bob-navigation-hotkeys.md`; reconfirm
rather than assume those persist. The planned fixture drift and any new drawer failure
are in scope and must be fixed. Missing actual Obsidian acceptance is unfinished work,
not a passing unit-test substitute.

## Land and completion responsibility

Implementation phase workers close only their assigned child phase after checking
`sase bead epic-symbols <their-phase-id>`. They never create follow-up task beads;
record `PROPOSED FOLLOW-UP:` notes for triage. They do not close bob-cli-ph.7 or
bob-cli-ph. The child epic land agent gathers their evidence and completes the remaining
integration checks.

The child epic land agent verifies:

- Every planned behavior is present, generated bundles and byte-identical canonical
  fixture copies are current, and all required tests/gates have results.
- The real-Obsidian report and inspected light/dark screenshots are durable artifact
  refs, including actual conversion and typing evidence.
- Changed plugins are deployed and exact commits/markers are recorded on
  **bob-cli-ph.7** for the later writer gate.
- No background drawer rewrite, capture grammar change, memory edit, or migration
  slipped into this work; legacy reading remains available.

This child epic exists to finish bob-cli-ph.7. Its land agent, acting as that phase's
completion owner, runs `sase bead epic-symbols bob-cli-ph.7`, resolves any leftovers or
re-keys them to an appropriate still-open later bead, and closes **only bob-cli-ph.7**
with
`sase bead close bob-cli-ph.7 --note "<verified behavior, tests, deployment, and artifact refs>"`
once its child work is closed. It may close its own child epic according to the normal
land lifecycle, but must never close bob-cli-ph or any higher ancestor. Planning and
approval leave bob-cli-ph.7 in progress.
