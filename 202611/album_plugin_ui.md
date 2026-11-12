---
tier: epic
title: Obsidian album cards, live chips, headers, and navigation
goal:
  Complete bob-cli-k3.7 with report-driven album and record cards, album headers,
  correct parent and task-home pickers, and a tested, documented plugin deployment.
phases:
  - id: album-data
    title: Shared album reports and debounced sync
    size: medium
    depends_on: []
    description:
      "album-data: implement schema-v1 report and sync decoders, shared
      stale-while-revalidate service, filtered frontmatter invalidation, three-second
      sync debounce, lifecycle wiring, real CLI fixtures, and tests."
  - id: owner-cards
    title: Owner album cards and lead-group navigation
    size: medium
    depends_on:
      - album-data
    description:
      "owner-cards: implement hue-accented albums callouts in Reading view and Live
      Preview, count/lead-group/peek/age/health chips, responsive layout, accessible
      group navigation, and focused renderer tests."
  - id: records-cards
    title: Record index cards with folds and date chips
    size: medium
    depends_on:
      - owner-cards
    description:
      "records-cards: implement styled records tally, view-only folding after the first
      group above twelve rows, record date chips, refresh-safe view state, group-jump
      integration, and renderer tests."
  - id: album-header
    title: Album breadcrumb and read-only schema header
    size: medium
    depends_on:
      - records-cards
    description:
      "album-header: implement path-resolved breadcrumb, copy-capture affordance,
      report-driven schema card, album-only record-property hiding, Reading/Live Preview
      lifecycle, and header tests."
  - id: album-navigation
    title: Album and record kinds in navigation
    size: medium
    depends_on:
      - album-header
    description:
      "album-navigation: implement accessible album/record icons, album and record
      parent-picker rules with commit-time checks, task-home exclusions, and focused
      navigation regressions."
proposed_by: bbugyi200.athena.bob-cli-k3.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-k3.7
create_time: 2026-11-11 23:10:16
status: wip
---

- **PROMPT:**
  [prompts/202611/album_plugin_ui.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_plugin_ui.md)
- **PARENT:**
  [202611/album_and_record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_and_record_notes.md)

# Album cards, headers, and navigation in Obsidian

## Scope and completion owner

This is the child implementation epic for **bob-cli-k3.7**, the `plugins-albums` phase
of **bob-cli-k3**. The assigned phase is still in progress. Its description and notes
were read before planning; there are no earlier work notes. The implementation is absent
from the inspected plugin checkout, so this plan starts with the shipped notebook
patterns, not a partial album UI.

The work spans a shared asynchronous service, two callout renderers, a dual-view header,
navigation rules, and real-Obsidian integration. Each phase below is bounded to one
coding agent. Phases run sequentially because they share generated bundles, manifests,
test registration, and deployment to the same vault. The child epic's land agent owns
the integrated proof and completion of **bob-cli-k3.7**. Phase workers close only their
own assigned child phase. Nobody working this child epic closes **bob-cli-k3** or its
other phases.

Read the accepted context through audited artifact reads:

- `plan:202611/album_and_record_notes.md`, especially the contract and `plugins-albums`
  section.
- `research:202611/album_and_record_notes/album_and_record_notes.md`, especially “The
  album summary” and “The Albums section in area and project notes.”

The accepted epic and shipped CLI contract take precedence over older research examples.
The parent decisions are final: `ledger_mining=yes`, the two glossary decisions are yes,
and `memory_albums_decision=no`. This child epic changes no memory; the outer wrap-up
phase owns the authorized glossary work. It does not change capture grammar, migrate the
vault, alter CLI schemas, edit chezmoi, or work on Bob Mac Capture.

## Starting evidence and repository discipline

The inspected bob-cli base is `0c8df969`. Album creation, owner catalogs, record
indexes, list/check JSON, and sync already landed, including `aef23542`, `0d2f433d`,
`3dce8a9e`, and their subsequent fixes. Do not redo those phases. The inspected linked
bob-plugins base is `d94c6bb`; a source/docs/test search and commit-subject search found
no album implementation. Recheck current history and assigned-bead notes when each
worker starts, because other work continues to land. There were no epic-symbol entries
for bob-cli-k3.7 when this plan was authored.

Open `bob-plugins` with `/sase_repo` and read its `AGENTS.md`. All plugin paths below
are relative to that returned checkout. Edit `src/` fragments and the
`src/fragments.json` order; rebuild `main.js`, never hand-edit it. Every edited fragment
must remain at or below 1000 lines. Prefer new cohesive fragments to expanding
already-large notebook fragments. Share small existing helpers where appropriate; a
generic catalog-framework rewrite is outside this plan.

Use the primary bob-cli checkout for CLI fixtures and docs checks. Set `BOB_CLI_DIR` to
that checkout when running `npm run docs:check`, so the wrapper does not discover or
clone a different repository. Read `docs.md` through `/sase_memory_read` before
documentation changes. Every phase updates the reference text for its own behavior,
regenerates the docs inventory when needed, runs build/tests/docs checks, and runs
`bob plugins sync` as required by the linked repo. Follow that repo's
deployment-conflict instructions; do not force an overwrite. Use `/sase_monitor` for
long verification commands and finish staged handoffs through `/sase_final`.

## Shared implementation contract

Bob owns stored `[!albums]` catalog rows and `[!records]` index bytes. The plugin adds
presentation only. Painting, folding, peeking, copying a route, and changing the local
day must not modify note contents. Only explicit parent changes and the shared
`bob albums sync -f json` operation invoke their existing writers. No UI code rebuilds
catalog Markdown or parses record schemas from YAML.

The current contract lives in bob-cli's `src/native/albums/report.rs`, `catalog.rs`,
`grouping.rs`, `index.rs`, and `sync.rs`; CLI fixtures and assertions live in
`tests/cli/albums.rs`, `albums_sync.rs`, and `albums_create.rs`.

- List JSON is schema version 1, with `ok`, `albums`, `catalogs`, and `counts`.
  `ok: false` can be a valid report: invalid albums remain visible with their counts and
  issues. Transport failure is different from invalid vault content.
- Album rows include path/stem, title, nullable emoji/summary/noun/label/plural,
  nullable parent/parent_kind/owner/root, display-label `breadcrumb`, ordered `fields`,
  `group_field`, `index_by`, `index_facets`, total `records`, ordered `groups`,
  `needs_status`, nullable `last_record`, `index`, and `issues`.
- **`index` is `{ "state": "current|missing|stale|damaged" }`, not a string.** An
  invalid schema is represented by its issues and damaged state; do not invent an
  `invalid` wire enum. Issue severity also accepts `info`.
- A field exposes key, label, kind, required, list, and ordered `{value,label}` choices.
  A group exposes value, label, count, and up to three title strings in `peek`. Display
  labels and peek titles are never navigation targets.
- Owner `catalogs[]` entries expose host/path/root/state/albums/issues. Surface
  catalog-only damage even if there are no usable album rows.
- `last_record` means latest valid record **created** date. Label its tooltip “last
  added”; it is not the latest visit or watch date.
- The report does not include individual record dates or paths. Resolve rendered record
  links with Obsidian metadataCache and read their cached frontmatter for row ages. Do
  not add a subprocess per row or scan record bodies.
- Album membership is structural: immediate Markdown children of the folder named by an
  album stem, including invalid and legacy-typed members. A broken album must not
  disappear from cache indices or event classification.

Use the notebook implementation as a behavioral reference:
`241a-plugin-notebook-data.js`, `241b` through `241d` for cards, `241e` through `241g`
for headers, and the existing notebook harness tests. Wire setup/teardown in
`170-plugin-lifecycle.js`, mixins in `310-install-methods.js`, and test exports in
`350-exports.js` or new export fragments where the size limit requires them. Register
new tests explicitly in `package.json`, in the appropriate unit stage.

## album-data

Deliver the shared report and sync service; later phases consume this interface instead
of making their own CLI calls.

1. Add album-specific list and sync decoders, with fixtures obtained from the current
   bob binary in a temporary `BOB_DIR` and controlled `BOB_NOW`. Use `bobSpawnJson` for
   executable resolution, vault selection, timeout, buffer limits, desktop handling, and
   safe argv. Preserve valid partial/refused sync reports and revalidate the list
   afterwards even when sync failed.
2. Expose consistent internal methods analogous to the notebook methods:
   `apiAlbumReport({force, wait, requiredGeneration})`, `apiAlbumSubscribe(callback)`,
   `apiAlbumInvalidate(reason)`, `apiAlbumSync(options)`, and `albumReportState()`. Keep
   one list flight, one sync flight, and indexed maps by album path, stem, and catalog
   host path. No public cross-plugin API version is needed by later phases.
3. Preserve last-good data while revalidating; retain a visible stale/error state on
   timeout, malformed JSON, unsupported version, and missing bob. An old flight cannot
   mark a newer generation current. Mounting several panes shares the same fetch. Unload
   cancels timers/subscriptions and prevents late completions from painting or
   scheduling more work.
4. Seed frontmatter fingerprints once when metadata is ready. Compare relevant content
   rather than metadata position offsets. Preserve schema/choice order in fingerprints.
   Detect album identity/schema/frontmatter changes and structural record-member
   frontmatter changes; debounce automatic sync for three seconds. Cover creates,
   deletes, and renames across membership boundaries using old and new identities.
   Unknown/invalid schemas still trigger refresh and remain diagnosable.
5. Keep unrelated Markdown edits, record body-only edits, and generated index writes
   from starting sync loops. Do not discard a real frontmatter change that arrives
   during a sync: coalesce a trailing sync when needed. Changes to an owner's ancestry
   or type invalidate relevant reports; retain enough dependency information to refresh
   the affected catalog/hue. Startup seeding itself must not schedule a vault-wide sync.
6. Follow the existing lightweight revalidation and day-rollover patterns so visible
   ages refresh at local midnight without writing Markdown. A record date change also
   notifies mounted index views.

Focused tests cover actual JSON shape (including info issues and null fields), broken
album alongside a healthy one, no-album catalog damage, unsupported schema, fetch
coalescing, generation races, the 3-second debounce, deletion and rename, changes during
sync, no feedback loop, partial sync failure, vault argv, midnight, and teardown. Add
concise service notes to `docs/develop/plugin-apis.md` only if that page needs to
describe the new internal service; user behavior belongs with the cards.

## owner-cards

Decorate `[!albums]` in owner notes in Reading view and Live Preview, retaining the
notebook card's understated border, spacing, and root-area hue resolver.

- Resolve the album row's link path against its actual source note. Project grouping
  rows are headings, not albums, and receive no album chips.
- Right-align total count, first nonempty declared group's tally, last-added age, and
  health. The lead tally uses the existing label grammar, such as `🔖 11 to try`; do not
  sort groups in the plugin. Its hover/focus peek lists at most the three titles
  supplied by bob, safely inserted as text.
- A lead-chip click or keyboard activation opens the correct album and scrolls to that
  top-level group. Groups are list items, not Markdown headings: resolve the generated
  callout and group line rather than guessing `#Status` anchors or deriving paths from
  titles. Keep normal modifier navigation and jump-history behavior. Factor the jump
  target/activation helper for reuse by records-cards, which adds fold-state
  integration. Missing group fallback opens the Records region with a useful hint,
  without writing an anchor.
- Distinguish stale cached data, stale/missing generated regions, and invalid schema or
  damaged regions. Offer a single shared sync action for repairable state and preserve
  bob's diagnostic/hint for unrepairable state. During sync disable duplicate actions;
  expose retry for read errors. Do not represent an unavailable report as zero counts.
- Chips wrap below the summary in narrow panes. Retain native internal links, hover
  previews, accessible buttons and tooltips, keyboard focus through repaint, and normal
  editor selection/caret behavior. Source mode remains editable Markdown. Paint
  idempotently and clean up on file switch/unload.

Tests use the existing DOM/editor harness patterns: nested project groups, duplicate
display titles with different paths, aliases, malicious-looking labels,
stale/invalid/empty/no-report states, missing catalog with an actionable host anchor,
group jump, multiple panes, repaint cleanup, and no note writes. Add the owner-card
reference text in a focused album page linked from `docs/bob-ledger-tools.md`; reuse
that page in later phases rather than growing the already-large Ledger Tools reference
indefinitely.

## records-cards

Style `[!records]` as the album's record-index card, preserving its stored tally, order,
subgroup labels, facets, links, and empty-album teaching text.

- Read the actual generated callout structure and associate it with the source album.
  Ignore matching examples in fences/frontmatter and unrelated embeds. Work in Reading
  view and Live Preview using the notebook rendering and CodeMirror lifecycle patterns;
  do not implement Live Preview by mutating editor document text.
- The first top-level group starts expanded. Later groups start folded only when their
  record count exceeds 12. Count records, not subgroup list items. Store user fold
  choices only in view state keyed by source album and group identity; refreshes
  preserve a user's choice, and panes are independent. Flat ungrouped albums and empty
  indexes remain readable. A group-jump action from owner-cards expands the intended
  group before scrolling/focusing.
- Add age chips from valid `visited` and `watched_on` cached frontmatter dates on
  rendered record links, reusing the notebook calendar-age helper. If both are present,
  show distinct labeled tooltips for the two facts; do not silently choose a different
  meaning or substitute mtime/created. Omit missing, invalid, or future date ages.
  Update on relevant metadata and day rollover, without per-record subprocesses or body
  reads.
- Give fold controls accessible labels and expanded state. Preserve internal link
  navigation and keyboard access; cleanup removes decorations and fold restrictions on
  teardown without leaving hidden rows behind.

Test 12 versus 13 rows, a first group larger than 12, later large sub-grouped groups,
the needs-status group, empty/flat albums, fold persistence versus pane isolation,
navigation into folds, date changes and rollover, and byte-identical note content after
interaction. Extend the album reference page.

## album-header

Add album-only header chrome, based on `241e`–`241g`, in Reading view and Live Preview.
Use the report's title and fields; no YAML schema parser is needed.

- Show the owner ancestry and album title, for example `Body › Restaurants`. Resolve
  links using actual parent paths and metadata, as the notebook header does.
  `breadcrumb` strings are display labels only. Missing/cyclic ancestry degrades safely
  instead of creating misleading links.
- `＋ restaurant` copies exactly `Name @eat/` using the valid report noun and stem. It
  does not launch capture or mutate the vault. Show clipboard success or failure through
  the existing notice conventions. Invalid schema/route state keeps the header visible
  but disables this affordance with its reason.
- Render a read-only schema card in declaration order, including field labels, kinds,
  list markers, required marks, and choice labels. Treat all content as text. An absent
  fields list has a helpful empty-schema message; issues remain visible and cannot be
  mistaken for an empty healthy schema.
- Hide `.metadata-property[data-property-key="record"]` only within an album note's own
  surface. A view-scoped class and cleanup must restore visibility on type change, file
  switch, unload, and non-album notes. Do not delete or rewrite the property. Source
  mode must still allow editing raw frontmatter.
- Handle H1-less albums, Properties panel rerenders, embeds, multiple panes, light/dark
  themes, and focus retention using existing header patterns.

Tests cover path identity versus labels, broken reports, nullable noun, empty schema,
ordered choices, clipboard failure, one header per source surface, Properties isolation,
type changes, view switches, and unload. Extend the album reference page with the
schema-editing and capture-copy behavior.

## album-navigation

Extend note-kind presentation and parent eligibility in Navigation Hotkeys. The current
`parentRuleKindOf` classifies any unreserved typed note as a record, so it incorrectly
includes albums; make album classification explicit before the legacy record fallback.
Preserve compatibility for existing noun-typed records while treating `[[record]]`
directly.

Work primarily in `190-summary-modals.js` (kind information/presentation),
`208-parent-note-picker-model.js`, `209-parent-note-picker-modal.js`, and
`682-plugin-parent-change.js`. Extract small fragments if the line cap requires it. Use
Lucide `album` and `file-text` (with accessible “Album” and “Record” labels), and
integrate them into the existing child/parent note icon surfaces. Keep search and
summary presentation coherent with the added kinds.

- Album parent candidates are areas or projects of any status, never notebooks, other
  albums, or records. Record parent candidates are albums only. Notebook rules remain
  unchanged. Keep cycle and descendant checks and clear empty states; verify eligibility
  again using live data before applying a parent write, including the parent shortcut
  path if it shares this writer.
- Do not move a record file or rewrite its schema when changing a parent. Structural
  mismatch remains a `bob records check` finding; that broader record-move feature is
  not part of this phase.
- Task-move already uses an area/open-project whitelist in `240-cancel-and-lanes.js`;
  preserve it and prove album/record exclusion. Audit inbox routing and Task Card
  relocation consumers of the same destination logic so no alternate task-home picker
  admits the new kinds. Navigation to ordinary non-record children of albums remains
  legal.

Extend parent-change, notebook-gesture, task-move, inbox-route, and relevant note-kind
tests. Cover scalar/list/wikilink type forms, legacy nouns, picker ranking and typed
queries, stale candidate rejection at commit time, closed project album parents, and
album/record task-home exclusion. Update the Navigation Hotkeys reference for the new
parent rules without changing the existing hotkeys or unrelated navigation behavior.

## Integration and land-agent acceptance

The child epic's land agent audits all five deliveries against bob-cli-k3.7, fixes
integration defects, and owns the following bounded final proof:

1. Extend the scratch-vault/CDP pattern from `scripts/obsidian-notebook-ui-e2e.cjs` with
   an album-specific scenario and an offline fixture test registered in `npm test`. Use
   actual bob JSON and generated callout bytes from a temporary vault, not the personal
   vault. Include a healthy grouped album, >12-row later group with subgroups, empty
   album, invalid album, missing/stale catalog, notebook neighbor, record dates, an
   H1-less album, and an unrelated `record` property.
2. Run real Obsidian checks in Reading view and Live Preview: light/dark, a narrow pane
   around 375 px, owner chips/peek/group jump, fold persistence, date-chip refresh,
   breadcrumb navigation, copy action, schema ordering, scoped property hiding, report
   failure/retry, and file-switch/unload cleanup. Capture screenshots, inspect them, and
   fix overlap, focus, or visibility problems. Assert presentation-only interactions
   preserve fixture note bytes; separately verify frontmatter edits coalesce one
   debounced sync.
3. In the linked repo run `npm run build`, `npm run docs:gen`, `npm test`,
   `npm run validate`, and `BOB_CLI_DIR=<primary-checkout> npm run docs:check`. Inspect
   changes to bundles/inventory/manifests and retain only intended generated updates.
   Run the primary checkout's required `just check` gate using the normal monitor flow
   if it is long. A failure reproduced identically on clean base is recorded with
   evidence as a proposed follow-up rather than leaving the assigned phase open.
4. Run `bob plugins sync` from the opened linked checkout and verify deployment using
   `docs/develop/deployment.md`. Each implementation phase also obeys this repo's
   mandatory sync rule; final sync verifies the integrated build.
5. Record implemented features, exact checks, screenshot/artifact references, deployment
   outcome, and any clean-base failures on bob-cli-k3.7. Read/produce durable artifacts
   through the SASE artifact workflow. Do not create task beads: append discoveries as
   `PROPOSED FOLLOW-UP:` notes on the assigned phase for the outer land agent to triage.
6. Once every child phase has landed and acceptance passes, run
   `sase bead epic-symbols bob-cli-k3.7`. Resolve remaining entries or re-key genuine
   unfinished symbols to a still-open owning bead. Close only the assigned phase via its
   prescribed completion workflow, and have this child epic's land agent close
   **bob-cli-k3.7** with a verification note when the child-descendant guard permits it.
   Never close **bob-cli-k3** or another ancestor; the outer epic land agent owns that
   completion. Every turn ends through `/sase_final`, declaring all changed
   repositories.

The completion criterion is the whole Obsidian behavior promised by `plugins-albums`,
deployed and verified. The outer epic still owns vault cutover, config retirement, and
the accepted memory updates.
