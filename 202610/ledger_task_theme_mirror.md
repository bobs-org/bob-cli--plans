---
tier: epic
title: Ledger task-theme mirror and pills
goal: Complete bob-cli-dm.5 with Rust-equivalent task-theme suggestion and field APIs,
  editable explicit-field pills across Obsidian task surfaces, and active PT6 colors.
phases:
- id: theme-api
  title: Mirror the task-theme rule and expose the synchronous API
  size: medium
  depends_on: []
  description: 'theme-api: extract the themes API factory, mirror task field parsing
    and byte-preserving writes, suggestion resolution and the deliberate-choice rule,
    add readingTheme config and known-theme lookup, consume the copied Rust vectors,
    and document the additive namespace. Include legacy reference block-ID recognition
    in the two picker display helpers identified in this plan.'
- id: task-theme-editor
  title: Render task-theme pills in Live Preview and activate PT6 colors
  size: medium
  depends_on:
  - theme-api
  description: 'task-theme-editor: build the canonical explicit-field mark model,
    safe selection folding and click reveal, lazy reason tooltips, the task-theme
    session toggle and popout gate, and the approved ledger theme-chip setup. Add
    editor, lifecycle, color, and toggle tests and document the commands and Live
    Preview behavior.'
- id: task-theme-rendered
  title: Render task-theme pills in Reading view and query rows
  size: medium
  depends_on:
  - task-theme-editor
  description: 'task-theme-rendered: reuse the mark model in Reading view, Tasks,
    and Dataview rows; preserve native content and inline markup, handle deferred
    query rendering, replace only explicit task-field Dataview chips, and support
    lazy hover, toggle restoration, and teardown. Complete cross-surface tests, docs,
    and deployment evidence for the child epic land agent.'
proposed_by: bbugyi200.apollo.bob-cli-dm.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-dm.5
create_time: 2026-10-28 17:46:52
status: done
bead_id: bob-cli-dm.5.1
---

- **PROMPT:** [prompts/202610/ledger_task_theme_mirror.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ledger_task_theme_mirror.md)
- **PARENT:** [202610/task_theme_suggestions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_theme_suggestions.md)
- **BEAD:** [bob-cli-dm.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.5.1.md)

# Ledger task-theme mirror and pills

## Outcome and boundaries

Complete the implementation assigned to **bob-cli-dm.5**, the `ledger-mirror` phase of
`plan:202610/task_theme_suggestions.md`. Ledger Tools will expose the same task-theme
suggestion and write rules as Rust, and explicit task fields will become editable PT6
pills across Live Preview, Reading view, and Tasks/Dataview results. The already
implemented ledger theme pills and their color command will be enabled.

This is a child epic because the pure semantic port, CodeMirror lifecycle, and
asynchronously rendered query DOM have distinct implementation and verification burdens.
Every phase above is one bounded coding assignment; dependencies are deliberately serial
because the later phases consume the earlier model and share the lifecycle, manifest,
exports, generated entrypoint, and documentation files.

The parent decisions are final: `ledger_theme_pills = yes`; both memory decisions are
**no**, so edit no memory notes. `seed_plex_chore = yes` belongs to the separate seed
phase and adds no vault-note changes here. Do not implement link-picker
preselection/commit behavior, Task Card editing, successor retargeting, capture, or
Swift work; those remain other phases of the outer epic. These helpers do not write task
files themselves: setters return strings and callers in later phases own the existing
guarded transactions. Never materialize inherited defaults into tasks.

All production edits belong to linked **bob-plugins**, opened with `/sase_repo`; read
the `AGENTS.md` at the path returned by `sase repo open bob-plugins`. Read bob-cli's
implementation and fixtures from the active checkout. Never edit plugins under the
deployed vault. Edit source fragments and regenerate `main.js` using `npm run build`;
every handwritten fragment must remain at most 1000 lines. Run `bob plugins sync` after
each implementation phase's changes. Do not edit versions, CHANGELOG.md, or memory. Do
not create follow-up task beads: use a `PROPOSED FOLLOW-UP:` note on the assigned phase
for the land agent to triage.

## Verified starting point

- `bob-cli-dm.5` is in progress, with no previous implementation/remainder notes. There
  is no reason to reopen it. `sase bead epic-symbols bob-cli-dm.5` currently reports no
  entries; repeat the check before eventual closure.
- Dependency `bob-cli-dm.1` is closed. Rust code is present in
  `src/native/task_theme.rs`, `src/native/task_fields.rs`,
  `src/native/pomodoro_themes/suggest.rs`, and the `pomodoro_homes` and config modules.
  The current shared vectors contain `suggest`, `task_field`, and `write_rule`.
- The shipped documentation calls the new contract **PT13** in
  `docs/pomodoro-themes.md`; PT12 already means configured themes. Use PT13 links and
  current code where the outer plan still says PT12.
- In bob-plugins at inspection commit `7f42bb8`, none of the new API helpers exist.
  `plugins/bob-ledger-tools/src/170-plugin-lifecycle.js` is exactly 1000 lines and still
  contains the inline `themes: Object.freeze(...)`. `onload` calls home-chip setup but
  never `setupPomodoroThemeChips()`.
- Existing templates are already substantial: `265-plugin-priority-marks.js` is 969
  lines, `266-plugin-date-marks.js` 957, and `279-plugin-pomodoro-themes.js` 895. Add
  small new fragments instead of extending those past the limit.
- Planning baseline: `npm run build:check` passed, and the existing Pomodoro themes,
  plan-budget, priority-marks, date-marks, and date-marks-tasks suites passed all 127
  tests. This is baseline evidence only; the full suite, new behavior, and actual
  Obsidian surfaces still need implementation-time verification.
- `REF_AFTER_TASK_RE` does **not** exist in Ledger Tools. It occurs in
  `block-id-prompt/src/010-core.js` and
  `bob-navigation-hotkeys/src/010-requires-and-config.js`, used by each plugin's
  `cleanTaskDisplayText`. Those cleaners remove the trailing ID before computing their
  ref flag. The narrow legacy-ID fix must inspect the original line first.

## Shared implementation contract

Use the authoritative Rust implementation and shared vectors to settle normalization,
field grammar, optional JSON members, and exact output bytes. Do not infer a default
from Dataview `task.theme`, which conflates a note default with a task field.

1. Suggestion precedence is a valid task field, reading task (`#ref` token or a trailing
   `^ref`/`^ref-*` ID), canonical daily `^gtd` review, then the first area or project
   default along the R1 residence chain. Closed tasks have no suggestion. A reading
   default is `READ` unless `plan.reading_theme` changes it or says `off`; review uses
   the first exempt name, if present.
2. Walk scalar `parent:` links through non-area/project wrappers, with the existing
   16-note/cycle bounds and path resolution. Only area/project `theme:` values count.
   Blank/absent inherits, `off` stops, invalid values warn once per note and continue.
   Daily notes have no default chain. A new-task request uses only the note rule.
3. A suggestion carries the Rust ThemeRef and `source`/`label`. `from` belongs only to
   note suggestions and contains declaring path, lowercase stem, kind, root-area hue,
   and residence-to-declarer chain. Only task-source suggestions carry `inherited`,
   including explicit null. Repair falls through and adds `repair` only when a
   suggestion exists. Preserve separately available warning diagnostics in the pure
   resolver for vector testing; the public suggestion call returns the suggestion or
   null, not a CLI response envelope.
4. Known means catalog snapshot or a component in today's ledger, including unsaved
   daily editor text. Hidden names remain eligible. Unknown names are reported with
   `known: false`. Catalog spelling wins, otherwise use Rust canonical spelling. Theme
   fields/defaults never alter hue, catalog history, preferences, or Stats. Theme hue
   comes through the existing `pomodoroThemeHueFor` API; declaring-note hue uses home
   resolution. Do not add an independent hue algorithm.
5. Parse both field delimiters with the Rust inline-field grammar. Exactly one valid
   field is set; multiple recognized fields are duplicate repair; one invalid value is
   invalid repair. Key matching and canonicalization must match Rust, including
   ASCII-only uppercase and its allowed component characters.
6. Set removes all recognized theme fields and one adjacent separator, then inserts
   `[theme:: NAME]` before the maximal trailing run of inline fields, tags, and block
   ID. Clear only removes theme fields and their one separator. Preserve all other
   bytes, including quote prefixes, spaces, tabs, Unicode titles, and a trailing CR. JS
   indexes are UTF-16: do not transplant Rust byte offsets into JS strings.
7. The pure write rule returns no change unless the choice is deliberate, a suggestion
   was shown, and the resolved destination contains exactly one non-exempt component.
   Equal to inherited clears all fields; equal to an already valid field is a no-op;
   otherwise set the chosen catalog-spelled component. Unnamed, composite, and
   all-exempt destinations do nothing. The future explicit Task Card edit can bypass the
   shown-suggestion gate at its caller; do not weaken the default write rule. A theme
   edit never changes freshness, status, or links.

## Phase theme-api

### Implementation

1. Replace the inline lifecycle namespace with `themes: this.pomodoroThemesApi()`.
   Implement the frozen factory in a new small mixin fragment. Preserve every existing
   member and `version: 1` (top-level API stays v3), and add synchronous
   `suggestFor({path, line, text})`, `taskThemeOf(text)`, `setTaskTheme(text, name)`,
   `clearTaskTheme(text)`, `planThemeChange(...)`, and `isKnownTheme(key)`. Document the
   concrete argument/return shapes so the outer link-picker and card phases have one
   stable API to consume. Prefer an object argument for the rule with field state,
   shown/inherited suggestion, resolved destination name, exempt keys, and deliberate
   flag; return explicit none/set/clear actions, with name/key on set. Keep pure parsing
   distinct from rendered-mark eligibility and never spawn from these added helpers.
2. Put field scanning, state parsing, removal/placement, and write-rule functions in a
   pure fragment. Export the tested helpers in `350-exports.js`. Use the Rust scanner
   rather than borrowing the stricter priority-mark regex: field semantics must include
   the spacing forms accepted by Rust.
3. Add a facts-returning sibling of `pomodoroHomeResolve` in a new fragment, reusing
   `pomodoroHomeReadFrontmatter`, parent/type/link helpers, and home hue resolution from
   `142-pomodoro-home-chips.js`. Do not accidentally use `pomodoroHomeFrontmatterScalar`
   to flatten a theme list into a valid name. Keep the established home-chip API stable.
   Resolve the task's path and current line from supplied live text, or the matching
   open editor when text is omitted; do not read another active file or silently
   substitute stale cached task fields. Document line numbering and failure behavior.
   Frontmatter comes from metadataCache.
4. Extend `defaultPlanCaps`, `normalizePlanCaps`, and `coercePlanCaps` in
   `050-plan-budget.js`, together with their cached load path, to carry `readingTheme`
   from `reading_theme`/`readingTheme`. Default READ, accept off, normalize exactly like
   Rust, and retain the existing invalid-block fallback and diagnostic behavior. Update
   exact-object expectations in plan-budget tests.
5. Build known-name/display lookup from the existing themes snapshot and today's parsed
   ledger, preferring any open daily editor overlay. Reuse `onChange` and current
   metadata/editor invalidation rather than adding repeated subprocess or vault-wide
   reads to suggestion/hover hot paths. Do not filter hidden themes.
6. Add the trailing ref-ID test beside the two picker `REF_AFTER_TASK_RE` definitions,
   and use it on the unstripped task line in their `cleanTaskDisplayText` functions
   (`block-id-prompt/src/030-tasks-and-dependencies.js` and
   `bob-navigation-hotkeys/src/020-config-load-and-tasks.js`). Preserve the existing
   tag-cleaning/display behavior. Ledger's suggestion reading predicate independently
   follows Rust's exact token rules. Do not broaden PDF-open or task-tag-mark behavior.
7. Register fragments/mixins in `src/fragments.json` and `310-install-methods.js`.
   Extend `docs/develop/plugin-apis.md` with examples, null/repair/warning behavior,
   live input semantics, compatibility, and all new namespace members.

### Verification

Copy the **current** bob-cli `tests/fixtures/pomodoro_themes/vectors.json` byte-for-byte
to `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/vectors.json`; preserve
other sections added by concurrent outer phases. Record provenance in its README. Extend
`scripts/test-ledger-tools-pomodoro-themes.cjs` to execute every `suggest`,
`task_field`, and `write_rule` case (not just assert fixture contents). Feed realistic
Obsidian parsed frontmatter into the mini-vault adapter, including unquoted wikilinks,
quoted scalars, and invalid lists; compare warnings and JSON optional-member rules. Use
Rust's vector test setup for catalog/config construction.

Add focused cases for live unsaved task text, today's uncached component becoming known,
hidden suggestions, missing metadata, null inherited, Unicode/surrogate-pair title
preservation, and config defaults/off/invalid. The existing picker ref-display tests
must cover legacy ID-only references and ordinary follow-ups. Prove all prior namespace
methods still exist and the new calls neither spawn nor mutate vault text. Run the
focused suites, `npm run build`, `npm test`, `npm run docs:gen`, `npm run docs:check`,
and `bob plugins sync` (generate inventory before npm test if changed docs/manifests
require it). Report checks accurately.

## Phase task-theme-editor

### Implementation

1. Add a pure explicit task-theme mark model/DOM builder and a separate Live Preview
   mixin. Use `135-priority-marks.js` for the one-matcher/model/widget structure and the
   semantic scanner from phase theme-api. No field means no mark even if a
   note/reading/review default exists. A stored field on a closed task can still display
   its historical value. Repair must be visible as repair, never presented as a
   trustworthy chosen theme: decorate recognized explicit spans with repair styling,
   without inventing a default pill or hiding malformed unmatched text.
2. Use `.bob-pomo-theme`, `data-theme-key`, and `data-theme-hue` with normal text, 4px
   radius, approximately 0.3em padding and existing PT6 16% light/24% dark tints. No
   glyph or `theme` label. Unknown values are neutral with dotted underline and
   `Unknown theme`; repair uses the existing dashed repair treatment and `Needs repair`.
   Use the existing hue/configured/exempt semantics; a configured explicit hue still
   wins under the current hue contract.
3. Fold with `dateMarkSelectionFold` from `136-date-marks.js`: touching the field
   reveals source; a selection endpoint in any folded whitespace prevents hiding that
   endpoint. Clicking the widget places the caret in the field and reveals it. Rebuild
   on selection, document, viewport, path, and relevant refresh changes; source mode,
   fenced code, and inline code stay raw. Do not copy priority marks' older single-space
   caret handling blindly.
4. Resolve inherited tooltip text lazily on hover/focus, not while building every
   decoration. Example:
   `Task theme FIX · sase_blog suggests WRITE · Ctrl+Shift+P t to change`;
   reading/review/no-default use the shared reason phrases. Never put `::` into
   tooltip/aria attributes that Dataview rescans. Keep DOM construction safe with text
   content and ownerDocument. Design hover dispatch so the same model can survive
   Dataview DOM cloning in the next phase.
5. Add default-on session command `toggle-task-theme-marks`, gate
   `bob-task-theme-marks`, and popout mirroring through `applyMarkGateClass` and
   `registerMarkGatePopoutSync` in `266-plugin-mark-gate-mirror.js`. Toggling must
   reveal raw fields and never write them. Initialize and tear down hooks, classes,
   scheduled work, and listeners with the existing lifecycle.
6. Implement the already-approved ledger-pill branch: call `setupPomodoroThemeChips()`
   in onload. Use the mark-gate helpers for its `bob-pomo-theme-colors` body class too,
   including popouts and unload cleanup. Keep the color toggle independent of the
   task-field visibility toggle: disabling colors leaves readable task pills; disabling
   task marks restores raw fields. Ledger names, the plan-block star line, and task
   pills share the existing tint gate. Avoid duplicate registration or new persisted
   settings.
7. Wire fragments, mixins, test exports, and generated bundles. Update the manifest
   description and `docs/bob-ledger-tools.md` for the task-theme command, current Live
   Preview behavior, independent toggles, and the now-active ledger colors. Let
   `npm run docs:gen` refresh generated inventory; do not hand-edit generated command
   tables.

### Verification

Add meaningful model/editor tests (new scripts included in the root `npm test` command)
for bracket/paren fields, duplicates and invalids, no inherited ghost pill, unknown and
known hues, tooltip copy, selection endpoints in the field and folded whitespace,
multiselection, click reveal, source/code exclusion, closed history, and cache refresh.
Extend lifecycle/startup tests to exercise actual onload command registration. Test both
toggles independently and together, initial/new popout documents, and unload cleanup.
Run build, focused suites, npm test, docs inventory generation/checks, and
`bob plugins sync` before closing this child phase.

## Phase task-theme-rendered

### Implementation

1. Add separate Reading-view and query-row fragments using the same canonical mark
   model, never another field parser. Follow priority marks' render/postprocessor
   structure and `267-plugin-date-marks-tasks.js` for bounded frame scheduling when
   Tasks attaches or finalizes rows after postprocessing. Avoid a global observer.
2. Restrict transformation to actual explicit fields on task rows. Skip prose, code,
   embeds/children owned by other rows, existing Bob pills, and unrelated fields.
   Preserve surrounding links, emphasis, tags, nested lists, and text byte-for-byte; do
   not replace a whole row's innerHTML. Handle bracket/paren forms split across rendered
   text nodes and native Dataview inline-field markup.
3. Replace Dataview's `theme | FIX` presentation only when it represents an explicit
   task-line field. Page-default `task.theme` is insufficient evidence. Recover the
   originating note/task context from the supported renderer metadata; if provenance
   cannot be established, leave native output intact rather than inventing a pill or
   inheritance reason from the dashboard's path. Keep original/native content restorable
   for toggles and unload, and avoid one listener per cloned row.
4. Reuse lazy inherited-reason hover/focus with source-note context and delegated events
   scoped to registered document/root hosts, including popouts. Avoid raw inline-field
   syntax in tooltips/attributes that Dataview parses. Repeated render, catalog refresh,
   toggles, detached retries, and reattachment must be idempotent. Bound retries and
   cancel/drop queued work after disabling or unloading.
5. Finish Reading/Tasks/Dataview docs and API examples, regenerate inventory, and ensure
   the public contract consistently cites PT13. Keep documentation scoped to shipped
   helpers and marks; later picker/card behavior remains later work.

### Verification

Add rendered-model and DOM tests for text-node splitting, nested links/formatting,
code/prose exclusions, native Dataview chips, no chip from inherited page fields,
correct origin note versus dashboard path, invalid/duplicate repair, unknown themes,
deferred detached/pending Tasks rows, idempotence, DOM cloning, lazy hover, and
toggle/unload restoration. Extend the existing fake DOM only as necessary.

Run `npm run build`, `npm run docs:gen`, `npm test`, `npm run docs:check`, and
`bob plugins sync`. If Obsidian is available, manually inspect one fixture with a known
override, unknown field, repair, inherited-only task, both delimiters, and a formatted
title in source/Live Preview/Reading/Tasks/Dataview and a popout. Verify click/caret
reveal, hover reasons, and both toggles. If unavailable, say exactly which manual
surfaces were not checked; do not claim manual verification from unit tests. Do not
insert fake fields into Bryan's real tasks to test the feature.

## Land and closure evidence

The child epic land agent owns end-to-end integration and the completion evidence for
**bob-cli-dm.5**. Child phase workers close only their assigned child phase, never dm.5
or an ancestor. Re-read the original phase and final decisions, confirm all three phases
landed and that copied shared vectors still match the canonical source (re-copy/run
relevant tests if the outer completion phase extended them), review API compatibility
and generated output, and run the final npm/doc checks and plugin sync. Report any
unavailable manual verification.

For check failures, establish whether they reproduce identically on an isolated clean
base without reverting current work. Record independent base failures as
`PROPOSED FOLLOW-UP:` notes with an existing tracking bead if known; those do not leave
a completed phase open. Fix regressions caused by this work.

Before the permitted final closure of the original assigned phase, run
`sase bead epic-symbols bob-cli-dm.5`, resolve entries or re-key them to a still-open
bead, and close **only bob-cli-dm.5** with a note listing the implementation and
verification evidence, after the nested work has been finalized through the normal SASE
land flow. Never close outer parent epic **bob-cli-dm** or another ancestor plan bead.
Do not hand-set any status. The currently empty epic-symbol result is planning evidence,
not a substitute for the final check.
