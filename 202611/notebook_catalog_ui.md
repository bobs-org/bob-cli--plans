---
tier: epic
title: Notebook catalog cards, live chips, and linked headers in Obsidian
goal:
  Complete bob-cli-j8.6 with notebook catalog cards and live metadata in Reading view
  and Live Preview, linked notebook headers, and debounced structural sync.
phases:
  - id: notebook-ui-data
    title: Notebook report cache and debounced catalog sync
    size: medium
    depends_on: []
    description:
      "notebook-ui-data: implement the notebook JSON adapter, shared report cache,
      lifecycle, frontmatter fingerprints, and single-flight three-second sync trigger
      in bob-ledger-tools, with deterministic tests for refresh races and failures."
  - id: notebook-ui-catalog
    title: Notebook catalog cards and live chips in both Obsidian views
    size: medium
    depends_on:
      - notebook-ui-data
    description:
      "notebook-ui-catalog: use the shared notebook report to style and decorate
      generated catalogs in Reading view and Live Preview, including host hue, tree
      rules, row counts, recency, add-summary navigation, and the heading sync action;
      test both surfaces and preserve Markdown bytes and native links."
  - id: notebook-ui-header
    title: Linked notebook headers, settings, and integration proof
    size: medium
    depends_on:
      - notebook-ui-catalog
    description:
      "notebook-ui-header: render linked ancestry and child-notebook chips in both
      views, add the persistent notebook-chips setting, finish plugin docs and manifest
      metadata, and verify theme, width, editing, lifecycle, and deployed-plugin
      behavior."
proposed_by: bbugyi200.apollo.bob-cli-j8.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-j8.6
create_time: 2026-11-09 15:14:59
status: wip
---

- **PROMPT:**
  [prompts/202611/notebook_catalog_ui.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/notebook_catalog_ui.md)
- **PARENT:**
  [202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_notebooks.md)

# Notebook catalog UI for bob-cli-j8.6

Implement the Obsidian portion of the approved notebook epic: a readable, colored
catalog with live metadata, a linked header inside notebooks, and automatic catalog sync
after structural frontmatter edits. This child epic fulfills **bob-cli-j8.6** only. Its
parent contract is `plan:202611/bob_notebooks.md`, section `plugins-notebook-catalog`;
the research is `research:202611/bob_notebooks/bob_notebooks.md`, sections The Notebooks
catalog and Surfaces by repository. Read these through `sase artifact read` when
implementing.

## Scope and evidence

At planning time, bob-cli-j8.6 is in progress with no prior implementation notes.
Dependency bob-cli-j8.2 is closed. The primary checkout contains notebook recognition at
`0a74731b` and catalog generation at `55468bff`. Its `docs/notebooks.md`,
`src/native/notebooks/model.rs`, `scan.rs`, and `catalog.rs` define the shipped schema
and generated Markdown. Do not reimplement those capabilities.

The linked bob-plugins checkout, inspected at `a2045cc`, has no notebook rendering or
notebook sync implementation in Ledger Tools. Recheck history and the phase notes before
each implementation phase because other phases of the outer epic are active. Open
bob-plugins with `/sase_repo` and use only the path that command returns. Paths below
are relative to that repository unless explicitly identified as bob-cli paths.

This is an epic because cache/write concurrency and two distinct view integrations each
need a bounded coding pass. All three implementation phases are medium, with a serial
dependency chain: they share the lifecycle, mixin registry, styles, and generated
bundle. Each phase must leave a working build and test its own behavior.

The outer epic's DECISIONS remain final. This plan adds no reviewer choices and edits no
memory. Creation commands, parent/task pickers, block-ID prompts, capture, Mac Capture,
vault migration, and the outer epic's memory/docs audit belong to their existing phases.
Do not edit bob-cli's version or CHANGELOG.

## Shared implementation rules

- Work in `plugins/bob-ledger-tools/src/` fragments, registering source order in
  `src/fragments.json`, mixins in `310-install-methods.js`, and focused test exports in
  `350-exports.js`. Generate `main.js` with `npm run build`; never edit it by hand.
  Every hand-edited fragment must stay within 1000 lines.
- Keep the notebook implementation in small new model, data, render, editor, and header
  fragments. Wire setup/teardown into `170-plugin-lifecycle.js`; preserve the existing
  synchronous startup and defensive optional imports. Define any notebook refresh
  StateEffect lazily, matching existing heading refresh effects.
- Reuse `bobSpawnJson` in `236-plugin-stats-data.js` for subprocesses, including
  executable discovery, vault path, bounded output, timeout, and structured errors. Do
  not shell-interpolate note names or run a subprocess from an editor paint loop.
- Use the cache/lifecycle pattern in `241-plugin-reading-board-data.js`, heading widgets
  in `220-plugin-heading-and-review.js`, and sync behavior in
  `291g-plugin-gmail-sync.js` / `291-plugin-gcal-events.js` as precedents.
- Read `docs/develop/validation.md` and `docs/develop/deployment.md`. Add new tests to
  the explicit `package.json` test list and use the ledger harness where suitable.
- Render with safe text nodes and actual navigation targets. No HTML from report
  strings. Neither decorations nor navigation may write note text. Only the explicit
  sync subprocess writes catalogs; the plugin never repairs generated regions itself.
- Register rendered components with cleanup, retain source paths per view, support
  simultaneous panes and embedded notes, and prevent duplication after rerenders. Source
  mode keeps the original Markdown. Plugin unload releases timers, subscriptions,
  components, and decoration state and prevents late async results from repainting.

## notebook-ui-data

Build the shared data and refresh foundation without adding new CLI fields.

1. Fetch `bob notebooks list -f json`. Validate schema version 1 and the report shape,
   indexing notebooks by path and stem and catalogs by host path. Preserve `notes`,
   `sections`, `last_note`, `summary`, `parent`, `owner`, `root`, `children`, and
   findings. A well-formed list report with `ok: false` is usable data containing
   notebook errors, not a failed transport. Invalid JSON/schema and spawn errors are
   failures, never zero counts or a current catalog.
2. Keep one vault-scoped report, one in-flight list request, a vault-generation counter,
   and a debounced revalidation path shared by every mounted notebook surface. Show the
   last good report while refreshing. Coalesce a generation change during a fetch into
   one subsequent fetch; do not mark newer demand satisfied by an older response. A
   failed refresh retains the last report with an explicit unavailable/stale state.
   Retry on new demand or explicit refresh without a loop. Revalidate on view show/focus
   and relevant Markdown create/change/delete/rename; filter obvious unrelated files.
   Never scan or spawn once per row. Recompute recency at local-day rollover without
   inventing an mtime-based date.
3. Seed frontmatter fingerprints after metadata/layout readiness, without a sync storm
   on startup. Compare normalized `type`, `parent`, and `summary`, accepting the
   notebook type encodings supported by Bob. A notebook's parent/summary/type change, or
   any note changing to/from notebook, schedules a trailing **3000 ms** sync. Remember
   the old type so notebook-to-other transitions still trigger. Body-only edits and
   unchanged metadata events invalidate display data as needed but do not trigger sync.
   Color changes repaint hue without rewriting catalogs.
4. Run `bob notebooks sync -f json` through one shared sync function for automatic and
   manual requests. At most one sync runs. Skip new automatic sync scheduling while it
   runs, as the parent epic specifies; update fingerprints even when skipping so
   self-generated vault events cannot restart a loop. Always revalidate the list report
   after sync settles, including a partial failure. Clear busy state in every exit path.
   Keep damaged/stale states visible after refusal and surface a concise failure hint;
   missing bob, unsupported command, timeout, malformed response, and mobile without a
   subprocess runtime must degrade gracefully.
5. Expose internal fetch/subscribe/invalidate/manual-sync methods for the next two
   phases; no public plugin API is required. Keep data freshness and catalog state
   separate: a failed fetch cannot assert the written catalog is current.

Tests: report with errors, malformed/schema-mismatched reports, shared requests, change
during fetch, failure retaining old data, unload during fetch, startup baseline,
3-second coalescing, parent/summary edits, both type transitions, body-only edits, sync
already running, self-write events, partial failure, and explicit retry. Use fake timers
and stubbed subprocesses; do not mutate the live vault in tests.

## notebook-ui-catalog

Decorate the existing Markdown callout; Bob remains the authority for rows and their
order. Match notebook links by resolved target path/stem, never display title. Do not
decorate project group rows as if they were notebooks.

1. Style `.callout[data-callout="notebooks"]` as a soft card with a visible
   notebook-tabs icon, hidden redundant callout title, semibold linked notebook titles,
   muted summaries after the em dash, thin nested tree rules, and 12–16 px row rhythm.
   Keep the icon visible independently of the hidden title. Preserve real wikilinks,
   native folding, selection, and keyboard navigation. Scope CSS narrowly so ordinary
   lists and other callouts do not change.
2. Resolve the host's hue through existing Pomodoro home ancestry and
   `pomodoroHomeChainHue` (`142-pomodoro-home-chips.js` and
   `142a-pomodoro-home-colors.js`). Set `data-root-hue` on the actual callout in Reading
   view and Live Preview, using the existing `--bob-area-*` tokens and a neutral
   fallback for absent/muted hues. No color is written to Markdown.
3. Add right-aligned tabular `N notes · M §` and age chips from the report. Section
   count is `sections.length`; recency uses only valid `last_note` calendar dates. Use
   `today`, `1d`, `2d`, etc.; omit unknown dates and do not show negative ages for
   future dates. Null/absent report data is unavailable, not zero.
4. Add an amber **add summary** action for a missing/blank summary. Resolve and open
   that notebook, then reveal its properties and focus `summary` when the installed
   Obsidian API permits. Use the actual app capability, with a visible fallback that
   opens the notebook and explains how to edit its summary. Do not depend on another Bob
   plugin or synthesize undocumented DOM keystrokes.
5. Attach a subtle **↻ sync** action to the host's `## Notebooks` H2 whenever its
   catalog is missing, stale, or damaged. Show busy/disabled state during the shared
   sync. For a missing catalog without an H2, put a small rendered-only sync action at
   the host title/header so the missing state remains actionable; do not insert a
   heading into the source. After success revalidate; damage stays visible until Bob
   reports it fixed. Both mouse and keyboard invoke exactly the same action.
6. Reading view uses a postprocessor and registered render children. Live Preview uses
   CodeMirror decorations gated by `editorLivePreviewField` and source-path context,
   following the heading widget precedent. Verify the real Obsidian callout DOM:
   source-line decorations alone are not proof that a folded callout receives its
   hue/chips. Preserve editing of active callout source and avoid decorations in fenced
   examples. Never create a widget that covers unrelated text.

Tests: CSS structure and focus rules, nested catalogs and project group rows, duplicate
titles, escaped malicious summaries, all catalog states, both view renderers, real link
targets, repeat mounting, source mode/fenced examples, busy sync, property navigation
fallback, hue-only changes, and zero note-byte writes. Use real-Obsidian scratch-vault
coverage for callout integration where the Node harness cannot prove the DOM behavior.

## notebook-ui-header

1. Render a compact header inside notebooks in Reading view and Live Preview, below the
   document title (or at the first content position after frontmatter when there is no
   H1). Show `📓 dev › Developer tools › Git`-style ancestry and a separate row of
   child-notebook chips, with each resolved crumb and child linking to its real note.
   Tint the root/owner context with the existing hue.
2. `breadcrumb` in the shipped JSON is an array of display labels, not paths, and
   currently stops at the owner. Never turn those labels into guessed link targets.
   Follow the report's `parent` identities for notebook ancestors, using paths from the
   index, and the existing metadata/link resolver for the owner-to-root context. Use
   report titles for notebook labels. Study `buildParentAncestry` and
   `collapseParentBreadcrumb` in Navigation's `208-parent-note-picker-model.js` for
   bounded, cycle-safe presentation; do not import another plugin's private script scope
   or add a hard dependency on that plugin being enabled. Broken/cyclic ancestry
   produces a bounded, honest header, never infinite walking or links inferred from
   ambiguous titles. Child chips follow report identities and deterministic title/path
   ordering. Empty children omit the row.
3. Reuse component registration and refresh from the first two phases. Retyping,
   reparenting, title changes, and deletion update/remove headers. Avoid duplicate
   headers across rerenders and correctly use embedded-note source context. Preserve
   keyboard focus when refreshed. Headers never appear in Source mode or task-only note
   types.
4. Add a default-on persistent **Notebook chips** setting in the Ledger Tools settings
   tab, using plugin `loadData`/`saveData` without discarding unrelated settings.
   Turning it off removes live numeric/age/add-summary/sync chips and child chips, while
   leaving basic catalog styling and ancestry navigation intact. Automatic structural
   sync remains active: the preference controls presentation. Make the toggle update
   mounted views immediately and document its exact scope. Read-only styling/navigation
   still work on mobile when bob cannot be spawned.
5. Update `docs/bob-ledger-tools.md`, generated inventory via `npm run docs:gen`, and
   the plugin's own manifest version/description. Preserve concurrent feature metadata.
   Describe live data, property navigation, the sync trigger, failures, the toggle, and
   mobile limitations. Do not redo the outer epic's docs audit.

Tests: deep ancestry, non-root project owner versus root, duplicate titles, missing
parents, cycles, empty/renamed children, aliases/path targets, embeds, both modes,
setting persistence/refresh, and lifecycle cleanup. Include an isolated visual fixture
with nested rows, long titles and summaries, missing summary, stale/damaged state, and a
notebook with children. Inspect Reading and Live Preview in light/dark themes at desktop
and 375 px widths; confirm no horizontal overflow, visible focus, readable muted text,
native links, and editable source. Capture evidence and report the actual results. An
offline harness pass is not a real-Obsidian pass.

## Verification, deployment, and completion

Every implementing phase runs `npm run build`, its focused tests, `npm run build:check`,
and `npm run validate`; add meaningful regressions to the explicit test list. After
changing this linked repo, deploy the source actually edited with
`bob plugins sync --repo "$PWD" --no-pull` from its checkout, as its AGENTS.md requires.
Respect deployment protection; never silently force over an unrelated deployment. Read
applicable vault memory first if diagnosis needs vault access. Use `/sase_monitor` for
long-running checks and wait for staging to exit.

Final integration runs the complete `npm test`, `npm run validate`, and the mounted docs
gate with `BOB_CLI_DIR` pointing to the current primary checkout. Use the documented
isolated Obsidian startup smoke and notebook visual fixture; record any missing
prerequisite explicitly. Verify deployed managed-file hashes against the edited source
after sync. Plugin changes and regenerated bundles belong in the linked repository's
final declaration; every root turn uses `/sase_final`.

At completion, assemble evidence for the assigned phase: working card/chips/header in
both views, sync concurrency and byte-preservation tests, visual checks, settings, docs,
version, and deployment. Run `sase bead epic-symbols bob-cli-j8.6` immediately before
its eventual close and resolve or re-key any remaining entries. There were none at
planning time.

Only this child epic's land/completion owner may close the original assigned phase
`bob-cli-j8.6` once its entire scope is verified, with a note recording that evidence.
Implementation phase workers close only their assigned child phase; any ancestor close
instruction is evidence for the responsible land agent, not authorization for them.
Never close `bob-cli-j8` or another ancestor. Do not manually change bead status or
reopen the reserved phase.

Do not create discovered-work beads. Record `PROPOSED FOLLOW-UP:` notes on the assigned
phase for the outer land agent. If a check fails identically on the clean base, record
the comparison and any existing tracking ID as a proposed follow-up; that pre-existing
failure alone does not leave a completed phase open.
