---
tier: tale
size: medium
title: Album card becomes the title block and owns the album title from the title slot
goal:
  On desktop Obsidian, opening an album note such as eat.md shows one title, the album
  card's, in the title slot above native Properties. The card paints at once from local
  data. The redundant H1 is hidden at render time, and touching it with the cursor
  reveals it. The schema folds to one line. A reusable adopter glue does the slot mount
  and title ownership, so the record-card phase can reuse it unchanged. Every failure
  restores native rendering.
proposed_by: bbugyi200.athena.bob-cli-o6.4.1.1
proposed_with: claude/opus
bead: bob-cli-o6.4.1.1
create_time: 2026-11-20 13:58:04
status: wip
---

- **PARENT:**
  [202611/collection_cards_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/collection_cards_title_block.md)
- **BEAD:**
  [bob-cli-o6.4.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.4.1.1.md)

# Album card becomes the title block (phase `album-card`, bead bob-cli-o6.4.1.1)

## Context and binding inputs

This tale implements phase `album-card` of the child epic
`plan:202611/collection_cards_title_block.md` (bead bob-cli-o6.4.1). That epic's phase
section "Phase `album-card`" and its "Shared rules for every phase" are binding. The
outer design contract `plan:202611/property_card_title_block.md` is also binding: "The
rule", "One skeleton for every card", the Album row of "The species", "States", "Title
ownership: hide, reveal, restore", "Placement", "Accessibility, sizing, and motion",
"Interaction", and "Naming". Before coding, read these through audited interfaces:

- `sase artifact read plan:202611/collection_cards_title_block.md "<why>"` and
  `sase artifact read plan:202611/property_card_title_block.md "<why>"`.
- `sase bead read bob-cli-o6.1 -r "<why>"`, note #1. It gives the **TITLE SLOT: GO**
  verdict and the primitive API in `240e`:
  `propertyCardMountInTitleSlot(scope, node, { path, kind, view, currentPath, onLost })`.
  The call returns a release handle, or `null`, which means body fallback.
  `onLost(reason)` passes one of `denied`, `path-changed`, `disconnected`, `unstable`,
  or `insert-failed`.
- `sase bead read bob-cli-o6.2 -r "<why>"`, note #1. It gives the identity kit API in
  `240f`–`240i`, ownership handles with `.generation`, and the
  `createPropertyCardHost({ titleOwnership: true })` opt-in.

The epic's DECISIONS are final: `memory_glossary_cards = yes` belongs to the outer
`glossary` phase, and `memory_decision_title_block = no`. **This tale edits no memory
file and creates no beads.** Record discoveries as
`sase bead note bob-cli-o6.4.1.1 'PROPOSED FOLLOW-UP: …'`. Close only bob-cli-o6.4.1.1.

All code work is in the linked **bob-plugins** repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`,
`docs/develop/architecture.md` ("Title block kit" and "Title slot"), and
`docs/develop/deployment.md` ("When sync refuses an overwrite"). Paths below are
relative to `plugins/bob-ledger-tools/` unless they start with `scripts/` or `docs/`,
which are relative to the repo root.

`bob-cli-ms` and `bob-cli-ms.8` were still open when this plan was written. Recheck
them. While either is open, **do not edit `scripts/obsidian-record-writer-e2e.cjs` or
`scripts/test-obsidian-record-writer-e2e.cjs`**.

## Facts verified while planning (bob-plugins `c5f5a1b`, ledger-tools 1.145.0)

Re-verify each fact against the current tree before relying on it.

- **No adopter yet.** No adapter calls `propertyCardMountInTitleSlot`,
  `propertyCardOwnTitle*`, `propertyCardBuildTitleBlock`, or
  `propertyCardApplyIdentity`.
- **Album card.**
  - Model: `src/241u-plugin-album-header-model.js` (`albumHeaderModel`,
    `albumHeaderOwnerChain`, `albumHeaderSignature`). It returns `null` without a report
    entry.
  - DOM and host: `src/241v-plugin-album-header.js` (538 lines; `albumHeaderBuildEl`,
    `paintAlbumHeaderReading`, `buildAlbumHeaderDecorations`,
    `albumHeaderRemoveReading`, `albumHeaderCleanEditor`, `albumHeaderOwnKeys`).
  - Schema text: `src/241w-…` (`albumHeaderSummaryText`, `albumHeaderStatsText`,
    `albumHeaderRecordCountPhrase`, `albumHeaderDefaultExpanded` returns `true`).
  - Schema DOM: `src/241x-…` (`albumSchemaPaintSummary`, `albumSchemaHeaderAppend`).
  - Mixin install: `src/310-install-methods.js`.
- **Other users of the summary text.** `albumHeaderSummaryText` is also used for the
  empty-schema text in `241x` and in `src/241yc-…` (album card editor). The editor uses
  `albumHeaderStatsText` too. Keep both functions' output unchanged; the fold gets a new
  function.
- **Title source.** `notebookHeaderNoteTitle` (`src/241e-…:164`) applies frontmatter
  `title:` only to `type: record`, then uses the first cached H1, the first alias, and
  the raw stem. It is not exported for tests. bob's album rule (bob-cli
  `src/native/albums/mod.rs` `frontmatter_title`, and `derive_title` in
  `src/native/notebooks/scan.rs`) is:
  1. a valid `title:` (a string, trimmed nonempty, no `\n` or `\r`);
  2. else the first H1 before any `##` or column-0 bullet, with wikilinks reduced to
     their display text;
  3. else the first alias;
  4. else `humanize_stem`: ASCII alphanumerics are kept, every other run becomes one
     space, and the result is trimmed, with no case change.

  An invalid `title:` falls through.

- **Other album-title readers.**
  - `src/124-links-walker-footer.js:67-89` (`linksWalkerFooterAlbumRecord`) reads raw
    `frontmatter.title || stem`, capitalized, with no H1 or alias. It feeds only the
    "walk is syncing" tooltip. Align it to the album rule.
  - `src/291h-gmail-close-notice.js` reads a Gmail note's `subject`/`title`. It is not
    an album reader, so leave it unchanged and say so in the closing note.
  - `recordSheetModel` (`src/242i-…`) takes the album title from bob's report, which
    already prefers `title:`.
  - `albumHeaderOwnerChain` only walks area and project owners, so the album rule never
    applies to it.
- **Host lifecycle.** `createPropertyCardHost` lives in `src/240c-…`, and its editor
  half in `src/240d-…`. Every host `dispose()` runs only from
  `teardownLedgerToolsSurfaces()`, which `onunload()` calls
  (`src/170-plugin-lifecycle*.js`). So `propertyCardTitleReleaseAll()` in an opted-in
  host's dispose cannot strip another host's ownership at runtime, and no scoping change
  is needed. Recheck this. If any host gains a non-unload dispose, scope the release
  additively. Teardown already calls `propertyCardTitleSlotReleaseAll()`
  (`170-plugin-lifecycle-teardown.js:119`).
- **Ownership details** (`240i`).
  - `propertyCardOwnTitle(scope, { path, recognizer, view })` routes a `.cm-editor`
    scope (with `view.dom === scope`) to the editor field, and a
    `.markdown-preview-view` scope to Reading tags.
  - Editor own is deferred, idempotent for the same path and recognizer key, and returns
    `null` in Source mode or when the field is not installed.
  - Reading own bumps the generation on every call, so call it only when the path or key
    changed, and keep the latest handle.
  - `propertyCardAlbumTitleRecognizer(title)` (`240h`) hides the first body H1 whose
    cleaned text equals the title, only when it is the first non-blank body line.
- **Slot details** (`240e`).
  - There is one slot entry per scope.
  - `propertyCardTitleSlotFor(scope)` is a pure DOM read that returns `null` for embeds,
    hover popovers, canvas nodes, print, mobile, Source mode, or a missing
    `.cm-sizer`/`.cm-contentContainer`/`.mod-header`.
  - The primitive adds the `bob-property-card-title-slot` class and the
    `data-title-slot-mode`/`data-title-slot-kind` attributes to the node.
  - `propertyCardTitleSlotMountedNode(scope)` and `propertyCardTitleSlotStats(scope)`
    are exported.
  - Re-mounting the same node drops the old entry, keeps the node, and takes a new
    generation.
- **Selector consumers.** Outside the album fragments, `.bob-album-header` is used only
  by:
  - `scripts/obsidian-album-ui-e2e*.cjs`;
  - `scripts/obsidian-vim-yank-flash-e2e.cjs:912` (`waitFor` `.bob-album-header`, then a
    `yy` on the album fixture's `# Album header source row` line through the editor
    API);
  - `scripts/test-ledger-tools-property-card-ownership.cjs` (surface class).

  No other plugin queries album or property-card classes. Re-grep `plugins/` and
  `scripts/` for every selector you change.

- **CSS.** Album header rules are at about `styles.css:8757-8830`
  (`.bob-album-header-nav` and `.bob-album-header-current` will be dead), plus a media
  rule at about 9750. The kit section starts at about 11680:
  - inline-title hide under `bob-property-card-owns-title`;
  - `.bob-property-card-title-source` hide;
  - the album material, an 8% tint plus a header band applied when content follows the
    title block;
  - narrow container rules (about 12004) and reduced motion (about 12059).
- **Tests.**
  - `scripts/test-ledger-tools-album-header.cjs` (1363 lines) paints Reading into fake
    roots without `.mod-header`, so those paths become the body fallback.
  - `scripts/test-ledger-tools-property-card-host.cjs:262` asserts 3 editor extensions
    for album, record, and bank. The album's `titleOwnership: true` may add the kit
    field when the test's CodeMirror stubs allow it.
  - `scripts/test-ledger-tools-property-card-title-slot.cjs` shows how to fake the Live
    Preview and Reading slot DOM.
- **Album e2e.**
  - `scripts/obsidian-album-ui-e2e-lib.cjs` provides `assertHeaderIntegrity` and
    `albumUiProbe` (`crumbs` =
    `.bob-album-header-crumb, .bob-album-header a.internal-link`).
  - `scripts/obsidian-album-ui-e2e.cjs` runs the `header-crumb` (href matches `body`),
    `header-copy`, `header-schema`, `live-header-body-sized-and-contained` (root font
    12–16px, weight below 600), and `header-375-wrap` checks.
  - The child-warnings flow clicks the summary when `aria-expanded` is not `"true"`.
  - Fixtures: `eat.md` has `# Eat` and no `title:`. `h1less-album.md` has no H1 and no
    `title:`.
- **Previews.** `scripts/preview-property-cards.cjs` renders the real
  `albumHeaderBuildEl` from a fake DOM. It compares baseline CSS (`-b 36d4801`) against
  current CSS, and fails unless every case is pixel-identical.
- **Docs.** `docs/albums.md`: the frontmatter description says "an expanded schema
  header", and the album card paragraph is at about lines 73-84.
  `docs/develop/architecture.md` has "Title block kit" (106) and "Title slot" (130).

## Parallel work in the same plugin

Two sibling phases run in parallel in bob-ledger-tools and land on bob-plugins
`origin/master` independently:

- **bob-cli-o6.5** (day and period cards). It may land an equivalent adopter glue or a
  heading-demotion helper.
- **bob-cli-o6.4.1.3** (bank-and-editors). It may add a kit identity-badge fragment
  (possibly `240j`), the `ADOPTED_KINDS` set in `preview-property-cards.cjs`, and a
  manifest bump.

At the start, and again before every build-for-deploy, run `git fetch` and merge
`origin/master` into the linked checkout. If a sibling already landed a helper you need,
reuse and extend it additively instead of adding a second one. When merging:

- keep both sides' `src/fragments.json` and `package.json` `test:unit` entries;
- regenerate `main.js` instead of merging it;
- take the next free `manifest.json` minor version and rerun `npm run docs:gen`;
- rebuild.

Name new fragments with the next free name after `240i` (and `374a` for exports) at
implementation time.

## Implementation

### 1. Album title rule (`src/241e-…`, `src/124-…`)

- In `notebookHeaderNoteTitle`, add an album branch for notes whose frontmatter `type`
  resolves to album, decided the way the album fragments already decide it
  (`albumTypeNames(type).includes("album")`). Check this against bob's album type rule.
  Inside the branch:
  - A valid `title:` wins: a string, trimmed nonempty, with no line break. A non-string
    (number, list, map), blank, or multi-line `title:` falls through.
  - Next comes the first cached level-1 heading that precedes the first level-2 heading
    and the first list item (`cache.headings` and `cache.listItems` positions). Clean it
    with `propertyCardCleanInlineText`.
  - Then the first alias.
  - Then a JS port of bob's `humanize_stem`.
- Leave the record branch and non-album behavior byte-for-byte unchanged.
- Export `notebookHeaderNoteTitle` (and the stem humanizer) through an exports fragment
  for tests.
- Align `linksWalkerFooterAlbumRecord` to `notebookHeaderNoteTitle(albumPath, …)` and
  keep its first-letter capitalization.
- Do not change `291h` (not an album reader). `recordSheetModel` crumbs and
  `albumHeaderOwnerChain` already agree.

### 2. Local identity model (`src/241u-…`)

Add a pure `albumHeaderIdentityModel(path, { metadataCache, vault, sourceText })`. It
returns `null` unless the note is album-typed: by `albumHeaderIsAlbumSource(sourceText)`
when text is given (Live Preview), else by the cached frontmatter `type`. Unlike
`albumHeaderFileTypeIsAlbum`, it never defaults to true when `type` is missing.
Otherwise it returns
`{ kind: "album", path, stem, title, emoji, summary, owners, ancestryBroken }`:

- `title` comes from `notebookHeaderNoteTitle`;
- `emoji` is the trimmed frontmatter `emoji` string, or empty;
- `summary` is the frontmatter `summary` when it is a plain string (trimmed);
- `owners` come from
  `albumHeaderOwnerChain({ path, parent: <normalized frontmatter parent> }, …)`.
  Normalize the parent with `pomodoroHomeParentTarget`/`albumNormalizeParent`, as the
  chain does.

Add `albumHeaderCardModel(path, state, sourceText)` that returns `{ identity, body }`:

- `identity` is required; without it the card model is `null`.
- `body` is today's report-backed `albumHeaderModel`, or `null` while the report is
  pending, failed, or has no entry.
- The title always comes from `identity`, never from the report.
- The subtitle is `identity.summary`. If the report entry carries a validated string
  `summary`, it may refine the subtitle.

Extend `albumHeaderSignature` (or add a card signature) to cover identity (title, emoji,
subtitle, owner paths and titles), body presence, the body signature, `expanded`, and
`desktop`.

### 3. Title block DOM (`albumHeaderBuildEl`, `src/241v-…`)

Build from the card model (update every caller, including `preview-property-cards.cjs`):

- The root stays `section.bob-album-header.bob-property-card`, with `data-album-header`,
  `data-album-source`, and `data-root-hue`.
- Call `propertyCardApplyIdentity(root, { kind: "album", title })`. The aria-label
  becomes `Album: <title>` and role becomes `group`.
- The first child is `propertyCardBuildTitleBlock(doc, …)` with:
  - `kind: "album"`;
  - `tile: { emoji }` (a stack tile, else the `library` icon);
  - `kicker`: the owner chain as `{ text: owner.title, target: owner.path }`;
  - `title`;
  - `subtitle`;
  - `onLink`: today's crumb opener.
- After building, add `bob-album-header-crumb` to every
  `.bob-property-card-kicker-link`, so e2e probes still resolve.
- Make the opener modifier-aware: pass `Keymap.isModEvent(event)` as `newLeaf` to
  `openLinkText`, following the guarded pattern in
  `src/241aj-plugin-areas-catalog.js:104`.
- When `body` is present, append `＋ <noun>` (`.bob-album-header-copy`) and `⊞ Schema`
  (`.bob-album-header-schema`, desktop only) to the title block's `actions` slot. Keep
  their classes, data attributes, labels, and handlers. Before the report there are no
  actions.
- Remove the crumb `nav` and the current-album crumb.
- Then, only when `body` is present, append:
  - the ancestry note (`body.ancestryMessage`);
  - the schema summary and chips (`albumSchemaHeaderAppend`);
  - the mobile hint.
- Report-pending and failed states show the title block alone. The broken and
  invalid-schema body states are today's.

Add an additive kit helper that **demotes and promotes the kit heading**
(`role="heading"` plus `aria-level="1"` on `.bob-property-card-title`). Put it in the
new glue fragment unless a sibling already landed one. Nodes start promoted, as the kit
builds them. The glue (step 5) and the fallback paths demote whenever ownership is not
held.

### 4. Schema fold (`src/241w-…`, `src/241x-…`)

- `albumHeaderDefaultExpanded(model)` returns
  `Boolean(model?.schemaInvalid || model?.emptySchema)`: healthy schemas start folded,
  and invalid or empty schemas keep today's expanded default. The per-session toggle in
  `plugin.albumHeaderExpanded` is unchanged.
- Add `albumHeaderFoldText(model)` for healthy schemas:
  `Schema · N field(s)[ · <grouping clause>][ · dated notes] · <record-count phrase>`.
  The record-count phrase comes from `albumHeaderRecordCountPhrase`. Example:
  `Schema · 5 fields · grouped by Location · dated notes · 104 restaurants`.
- The summary button shows `▸`/`▾` plus `albumHeaderFoldText` for healthy schemas.
  Invalid and empty schemas keep `albumHeaderSummaryText` and its `is-error` styling.
- Do not change `albumHeaderSummaryText` or `albumHeaderStatsText` output (the editor
  and empty text use them). Export the new helper.

### 5. Reusable adopter glue (new kit fragment after `240i`, exports in a new `374*`)

Write it generically (any `kind`, any recognizer) so the record-card phase reuses it
unchanged. The API is additive and backward compatible; say so in the commit. Proposed
surface:

- `propertyCardAdoptTitle(scope, node, { path, kind, view, recognizer, currentPath, onLost, defer })`
  returns `{ mode: "slot" | "fallback", release }`. `view` is the EditorView in Live
  Preview and `previewMode.renderer` in Reading. `defer` is optional (default
  `queueMicrotask`) and is used by tests.
- `propertyCardAdoptedEntry(scope)` returns
  `{ node, path, kind, mode, owned, generation }` or `null`.
- `propertyCardAdoptRelease(scope, path?)` is a scope-wide, optionally path-bound
  release, and `propertyCardAdoptReleaseAll()` releases everything at unload. Call
  `propertyCardAdoptReleaseAll()` from teardown next to
  `propertyCardTitleSlotReleaseAll()`.
- `propertyCardAdoptPatch(target, fresh)` swaps `fresh`'s children into `target` and
  syncs its attributes. It keeps the slot-owned `bob-property-card-title-slot` class and
  the `data-title-slot-*` attributes, so a body or title update happens inside the same
  mounted node.
- `propertyCardReadingViewFor(app, scope)` finds the markdown leaf whose
  `view.previewMode.containerEl` contains the scope. It returns
  `{ view: previewMode.renderer, currentPath: () => leaf.view.file?.path || "" }` or
  `null`.
- `propertyCardLiveCurrentPath(view)` returns the `editorInfoField` file path.

Behavior:

1. **Tracking.** Track one entry per scope:
   `{ generation, node, path, kind, mode, slotRelease, titleRelease, titleKey, owned, pending }`.
   Also remember a per-scope `{ path, reason }` for `unstable`/`insert-failed`, so the
   same path is not retried. That memory clears on a new path or an explicit release.
2. **Same node, same path, still mounted.** If the slot still holds the node (or the
   mount is pending), do not remount. Re-own only if the recognizer key changed, keeping
   the latest handle. Re-sync the heading. Return the same `mode` and `release`.
3. **Fallback.** If the slot is unavailable (`propertyCardTitleSlotFor(scope)` is
   `null`), or the path is remembered as unstable or failed, release any previous entry
   for the scope. Demote the node's heading and return `{ mode: "fallback" }`.
4. **Mounting.** Reading mounts synchronously. Live Preview (`.cm-editor`) returns
   `{ mode: "slot" }` at once, then mounts and owns in a deferred callback. **Never
   dispatch or mutate DOM inside a CodeMirror update.** If the deferred mount returns
   `null`, mark the path as failed, demote the heading, and call the adapter's `onLost`
   with `"insert-failed"`, deferred. That lets the adapter's refresh switch the editor
   to its fallback widget.
5. **Ownership.** After a successful mount, own the title with
   `propertyCardOwnTitle(scope, { path, recognizer, view })`. Promote the heading only
   while that returns a handle. A `null` ownership keeps the slot mount, and the node
   stays demoted.
6. **Losing the slot.** Wrap the primitive's `onLost`. A stale generation is ignored.
   Otherwise:
   - release title ownership in the same frame, so the H1 and inline title return;
   - demote the heading;
   - drop the entry, remembering `unstable` or `insert-failed`;
   - call the adapter's `onLost(reason)`, deferred, so it can remount through its
     refresh.
7. **Release.** `release()` is idempotent and generation-bound. An old handle never
   clears a newer mount. It releases the slot (the node leaves) and title ownership, and
   demotes the heading, all in the same frame. It never calls `onLost`.

### 6. Mounting the album card (`src/241v-…`)

- **Host.** Set `titleOwnership: true` on `albumPropertyCardHost`. On both surfaces, the
  adapter's `onLost` calls `scheduleAlbumHeaderRefresh()`.
- **Live Preview (`buildAlbumHeaderDecorations`).**
  - No path, or no identity: release the glue for `view.dom`, release the keys, and
    return `Decoration.none`.
  - A glue entry exists for `view.dom` with the same path: if the card signature
    changed, build a fresh element and `propertyCardAdoptPatch` it into the mounted node
    (restore focus by `data-album-header-focus`). Then re-adopt, which is a no-op unless
    the title changed. A slot entry returns `Decoration.none`, with no card widget.
  - Otherwise, build the node and adopt with:
    - `view`;
    - `recognizer: propertyCardAlbumTitleRecognizer(identity.title)`;
    - `currentPath: () => propertyCardLiveCurrentPath(view)`;
    - `kind: "album"`.

    On `fallback`, keep today's widget path exactly (anchor at the end of the first `# `
    line, `eq` by signature) with the heading demoted.

  - Own the `record` row keys only when `body` is painted, as today. Otherwise release
    them.
  - `albumHeaderCleanEditor` also calls `propertyCardAdoptRelease(view.dom)`.

- **Reading (`paintAlbumHeaderReading`).**
  - Card model `null`: remove and release, then return `null` (native).
  - The scope is a non-embed `.markdown-preview-view` with a glue entry for this path:
    patch in place when the signature changed, and keep the node.
  - Otherwise build and adopt with `propertyCardReadingViewFor(app, scope)` (renderer
    and `currentPath`).
  - On `fallback` (embeds and popovers), keep today's insert after the first non-code H1
    (or at the root start), with the heading demoted. Patch an existing fallback node in
    place too, rather than removing and re-inserting it.
  - Own the `record` row keys only when `body` is painted.
  - `albumHeaderRemoveReading(root, scope, path)` releases the glue entry for that scope
    and path before removing body nodes. The slot node lives in `.mod-header`, inside
    the sizer root, so today's sweep would otherwise tear it out without releasing
    ownership.
- **Placement outcomes to preserve.**
  - An album with `title:` and no H1 hides nothing in the body; the recognizer finds no
    H1.
  - An H1 that differs from the title, follows prose, or sits in a fence stays visible.
  - Source mode keeps native rendering, because `albumHeaderLivePath` returns `null` and
    the kit field drops its owner.
  - Mobile, embeds, hover popovers, and canvas get the body fallback with the heading
    demoted.
- **Fragment size.** Keep `241v` at or below 1000 lines. If needed, move the mount
  methods into a new album fragment registered through `src/310-install-methods.js`.

### 7. CSS (`styles.css`)

- Use the album material from the kit (`[data-card-kind="album"]`, 8% wash, header band
  when the body follows). Tune it only if the previews show a defect.
- Style `.bob-album-header-copy` and `.bob-album-header-schema` as compact pills in
  `.bob-property-card-actions`, aligned right on the kicker row.
- Style the folded summary as one muted line. Healthy and folded: no top rule and no
  chips. It wraps, and never overflows the card.
- Narrow widths (375px or less, using the kit's container query): the actions wrap below
  the title, the tile shrinks, and the species word stays.
- Visible `:focus-visible` rings on the kicker links (`.bob-album-header-crumb`), copy,
  schema, and summary buttons.
- `forced-colors: active`: borders and stripe use `CanvasText`. Reduced motion disables
  transitions.
- Delete the dead `.bob-album-header-nav` and `.bob-album-header-current` rules, and the
  now-unused margin on `.bob-album-header-actions`, including the media rule near 9750
  if it is dead.

### 8. Unit tests

- **New glue suite** (for example
  `scripts/test-ledger-tools-property-card-title-adopt.cjs`, added to `test:unit`), on
  the fake slot DOM from the title-slot suite. It covers:
  - Reading slot success: the node sits before `.metadata-container` in `.mod-header`,
    the scope carries `bob-property-card-owns-title`, the H1 is tagged, and the heading
    is promoted.
  - Live Preview with a sync `defer`: the node sits in `.cm-sizer` and the ownership
    effect is dispatched.
  - Slot `null` (embed, mobile, Source mode) gives `fallback` with the heading demoted
    and no ownership.
  - `onLost("path-changed")` releases the title in the same call, notifies the adapter
    once (deferred), and a re-adopt remounts.
  - `release()` restores native rendering (node gone, class gone, H1 untagged).
  - An old handle after a newer adopt is a no-op (generations).
  - `unstable` gives a sticky fallback for the same path that clears on a new path.
  - A same-node re-adopt does not remount. A changed recognizer key re-owns.
  - `propertyCardAdoptPatch` keeps the slot class and attributes.
  - Exactly one `[aria-level="1"]` (or untagged visible `h1`) remains in the owning
    scope.
- **Album title rule:**
  - a valid `title:` wins;
  - blank, multi-line, and non-string `title:` fall through;
  - then the H1 (an H1 after a `##` or bullet is skipped);
  - then the alias;
  - then the humanized stem;
  - the record branch and non-album notes are unchanged;
  - the walker-footer album title.
- **Album card** (update `scripts/test-ledger-tools-album-header.cjs`):
  - the new DOM (title block, kicker links with the crumb class and `data-href`, actions
    in the kicker row, no crumb nav), the `Album: <title>` aria-label, and
    `data-card-kind`;
  - local paint before the report (title block only, no actions, `record` row not
    hidden);
  - the body joining with no remount (the same node object, the slot entry generation
    unchanged, the `record` row hidden after the join);
  - fold defaults (healthy folded with the new text; invalid and empty expanded with
    today's text) and the per-session toggle;
  - fallback paths keep today's placement with the heading demoted;
  - teardown releases everything.
- **Host test.** Update `scripts/test-ledger-tools-property-card-host.cjs` if the
  extension count changes.
- Keep every other suite green. Re-grep for asserts on `album details` or crumb DOM.

### 9. Album e2e (`scripts/obsidian-album-ui-e2e-lib.cjs`, `scripts/obsidian-album-ui-e2e.cjs`)

Update the album e2e, and `scripts/obsidian-album-properties-e2e.cjs` only if needed.
Then:

- Extend the probe and `assertHeaderIntegrity` to check, in both Reading and Live
  Preview:
  - the active card sits in the title slot (`.cm-sizer` / `.mod-header`) before
    `.metadata-container`;
  - it has `data-card-kind="album"` and aria-label `Album: Eat`;
  - the inline title is hidden;
  - exactly one **visible** `aria-level="1"` node is in the active pane (count only
    rendered nodes; the other mode's scope stays in the DOM, hidden).
- Keep the crumb, copy, and schema checks (the kicker links keep
  `bob-album-header-crumb`).
- Apply the font check (`live-header-body-sized-and-contained`) to the card root, not to
  the new heading.
- **H1 hide and reveal.**
  - Reading: `# Eat` carries the kit title-source class and is not displayed.
  - Live Preview: the H1 line is hidden after open, with the cursor in the frontmatter.
  - Placing the cursor on that line through the editor API reveals it.
  - Moving the cursor away hides it again.
- **Fold.** The healthy summary starts with `aria-expanded="false"`, its text starts
  with `▸ Schema · ` and ends with the record-count phrase, and the chips are hidden.
  The existing click-to-expand flows still work.
- **`record` row.** It is hidden only after the report has painted the body.
- **Post-tidy shape.** Add or reuse a fixture with `title:` and no H1: the card shows
  that title and no body H1 is hidden.
- Keep the offline self-test and wrapper (`scripts/test-obsidian-album-ui-e2e.cjs`)
  green in `npm test`.

### 10. Previews (`scripts/preview-property-cards.cjs`)

- Add an `ADOPTED_KINDS` set containing `album`. If the bank-and-editors phase already
  added it, merge and keep both sides' kinds.
- Adopted kinds leave the baseline pixel-identity gate and instead get structural
  assertions:
  - `data-card-kind`;
  - exactly one `aria-level="1"`;
  - the species word in `.bob-property-card-kicker-species`;
  - the tile form (`stack`).
- Non-adopted kinds stay pixel-identical.
- Render the album from the card model (report-backed body, folded and expanded) in
  light and dark, at 375px and wide. Add a report-pending album case (title block only).
- Run the script under `xvfb-run -a` and inspect **every** album capture. Fix every
  defect you see in the owning fragment or CSS.

### 11. Docs

Read `docs.md` with `/sase_memory_read` first. Describe behavior only, keep each page
under its word budget, and use the contract's names: album card, card editor, and
"Schema Card" only as the editor's former name.

- **`docs/albums.md`.** Rewrite the album card paragraph:
  - it is the note's title block in the title slot above Properties, with the stack tile
    (emoji or icon), the `ALBUM · <owners>` kicker with links, the title, and the
    `summary` subtitle;
  - the title source: `title:`, then the H1, then the alias, then the stem;
  - the redundant H1 is hidden at render time, never deleted, and touching it with the
    cursor reveals it;
  - the folded schema line and its toggle;
  - `＋ <noun>` and `⊞ Schema` in the kicker row;
  - native rendering in Source mode, embeds, and mobile.

  Also update the frontmatter `description`, which mentions "an expanded schema header".

- **`docs/develop/architecture.md`.** Add a short "Adopting the title block" subsection
  after "Title slot". Cover:
  - the glue API and modes;
  - deferral in Live Preview;
  - `onLost` remounting;
  - the in-place patch;
  - the heading demotion rule;
  - the record-card phase reusing the glue.
- Run `npm run docs:gen` if the inventory or version changed.

## Gates, real Obsidian, deploy, close

1. In bob-plugins:
   - `npm run build`
   - `npm run build:check`
   - `npm test`
   - `BOB_CLI_DIR=<the bob-cli workspace checkout> npm run docs:check`

   Rerun any failing test in isolation. Known load-sensitive flakes (recorded on
   bob-cli-o6.1 and bob-cli-o6.2):
   - the frees-marks 25 ms budget (bob-cli-io);
   - the event-link 25 ms budgets;
   - the navigation dependencies-stage performance check;
   - the command-mode sheet-sequence 30 s timeout.

   A failure that reproduces identically on a clean `origin/master` worktree gets a
   `PROPOSED FOLLOW-UP:` that cites any tracking bead, and does not block.

2. **Real Obsidian on athena.** Run commands in the foreground with long explicit
   timeouts, and use `/sase_monitor` only if a run cannot fit.
   - Confirm that `bob albums tidy --help` works for the bob you pass, then run:
     `BOB_BIN="$(command -v bob)" xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs -r <report.json> -o <shots>`.
   - Also run the consumer check
     `xvfb-run -a node scripts/obsidian-vim-yank-flash-e2e.cjs --only <the group holding the album header case>`.
   - Run `scripts/obsidian-album-schema-writer-e2e.cjs` too, since it opens album notes.
   - Fix what fails. Read `sase_artifacts.md` with `/sase_memory_read`, then register
     the e2e report, the key screenshots (Reading and Live Preview of `eat.md`, the
     375px header, and the reveal), and the preview report and index with
     `sase artifact create --bead bob-cli-o6.4.1.1`.
   - If athena cannot run Obsidian, record the run as skipped with the exact reason,
     keep the offline wrappers green, and leave a `PROPOSED FOLLOW-UP:`.
3. **Deploy.**
   - Fetch and merge `origin/master` (see "Parallel work"), take the next free minor
     version in `plugins/bob-ledger-tools/manifest.json`, then rerun `npm run docs:gen`
     and the build.
   - Run `bob plugins sync -p bob-ledger-tools -d`, then the real sync. Never pass `-F`
     on your own; follow the `AGENTS.md` refusal procedure.
   - Confirm that the vault's `main.js`, `styles.css`, and `manifest.json` under
     `~/bob/.obsidian/plugins/bob-ledger-tools/` match the build byte for byte.
   - The bob-plugins commit
     (`feat(ledger-tools): album card owns the title block from the title slot`, noting
     that the kit additions are additive and backward compatible) lands through the
     turn's `/sase_final` repo obligation.
4. **Close.**
   - Run `sase bead epic-symbols bob-cli-o6.4.1.1`, and resolve or re-key any entries to
     a still-open bead.
   - Close with `sase bead close bob-cli-o6.4.1.1 --note "…"`. List:
     - the commit and version;
     - the gates, with any cited flakes;
     - the e2e and preview artifact refs;
     - each changed e2e expectation;
     - the walker-footer alignment and the `291h` no-change finding;
     - the dispose-scope verification.
   - Do not close the epic or any ancestor.

## Acceptance

- Opening `eat.md` in Live Preview and Reading on desktop shows exactly one visible
  title: the card's `Eat`, in the title slot before Properties. The inline title and
  `# Eat` are hidden. The cursor on the H1 line reveals it in Live Preview, and the card
  stays.
- The card's tile is a stack, and its kicker reads `ALBUM · Body` with a working link
  that honors modifier keys. The subtitle shows `summary`. `＋ Restaurant` and
  `⊞ Schema` sit on the kicker row. The aria-label reads `Album: Eat`.
- Before the report, the title block alone paints with no flash. When the report
  arrives, the body joins inside the same node. The `record` row hides only then.
- The healthy schema starts folded as
  `▸ Schema · N fields · grouped by … · <count phrase>`. Invalid and empty schemas start
  expanded with today's text.
- Source mode, embeds, hover popovers, canvas, mobile, plugin disable, file switch, and
  a failed mount each restore native rendering. Fallback cards carry no
  `aria-level="1"`.
- Album crumbs, record kickers (via bob), the walker-footer tooltip, and the card agree
  on the album title, before and after `bob albums tidy`.
- The glue is generic and documented, and its tests run on the fake DOM.
- Gates pass (or fail only on cited known flakes). The album e2e report is attached. The
  vault deploy is byte-identical to the build.
