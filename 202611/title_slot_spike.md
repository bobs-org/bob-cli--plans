---
tier: tale
size: medium
title: 'Title-slot spike: kit mount primitive, real-Obsidian CDP proof, and GO/NO-GO
  verdict (bob-cli-o6.1)'
goal: bob-ledger-tools ships an unused, unit-tested kit primitive that mounts a property
  card in Obsidian's title slot (after the inline title, before native Properties)
  in Live Preview and Reading, a real-Obsidian CDP spike on athena measures it across
  switches, splits, re-renders, settings, tall cards, and teardown, and bead bob-cli-o6.1
  carries a TITLE SLOT GO or NO-GO verdict note with report and screenshot artifacts
  that the collection-cards and day-period-cards phases build on.
proposed_by: bbugyi200.athena.bob-cli-o6.1
proposed_with: claude/opus
bead: bob-cli-o6.1
status: done
---

- **PARENT:**
  [202611/property_card_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_card_title_block.md)
- **BEAD:**
  [bob-cli-o6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.1.md)

# Title-slot spike and kit mount primitive (bead `bob-cli-o6.1`)

## Context

This tale implements phase `title-slot` (bead `bob-cli-o6.1`) of epic `bob-cli-o6`,
"Property cards become the note title block". Read the epic plan before coding:

- `sase bead read bob-cli-o6.1 -r "<why>"` shows the phase, the epic plan reference, and
  the final reviewer DECISIONS (memory edits are allowed only in the epic's later
  `glossary` phase, so this tale edits **no** memory files).
- `sase artifact read plan:202611/property_card_title_block.md "<why>"` is the epic
  plan. Its "Placement", "Title ownership", and `title-slot` phase sections are the
  contract this tale implements.
- `sase artifact read research:202611/property_card_title_block/property_card_title_block.md "<why>"`
  ("Resolution 3 adopts a gated title slot with body fallback") is the research
  rationale. If the read reports the research sidecar missing, run
  `sase repo open research -r "<why>"` once, then retry the read.

Why the spike exists: today every property card (record sheet, album header, bank Setup,
intention card) mounts in the note **body**, below native Properties. The epic wants the
card to be the first thing on the page: a sibling placed after the (later hidden) inline
title and before native Properties, inside Obsidian's own header slot
(`.cm-editor > .cm-scroller > .cm-sizer` in Live Preview and
`.markdown-preview-view > .markdown-preview-sizer > .mod-header` in Reading). That slot
is Obsidian-owned DOM outside CodeMirror's content, so it must be proven reliable in a
real Obsidian before the adopt phases (`collection-cards` = `bob-cli-o6.4` and
`day-period-cards` = `bob-cli-o6.5`) build on it. Both adopt phases block on this bead
and read its `TITLE SLOT:` verdict note to decide slot versus body placement. The worst
outcome is today's body layout, never a missing title.

All code work is in the **bob-plugins** linked repo, plugin `plugins/bob-ledger-tools`.
Open it with `sase repo open bob-plugins -r "<why>"` and use only the printed path. Read
its `AGENTS.md` first: edit `src/` fragments only, list new fragments in
`src/fragments.json`, run `npm run build` (never hand-edit `main.js`), keep every
hand-edited fragment at or below 1000 lines, and deploy with `bob plugins sync` (never
`-F` on your own; follow the refusal procedure in `AGENTS.md` and
`docs/develop/deployment.md`). `styles.css` is hand-edited directly.

Phase `identity-kit` (`bob-cli-o6.2`) runs in parallel in the same plugin and also adds
kit fragments after `240d`, CSS, exports, and `test:unit` entries. Before starting and
again before every build and sync: `git fetch`, merge or rebase onto `origin/master`,
keep both sides' `src/fragments.json` and `package.json` `test:unit` entries, regenerate
`main.js` with `npm run build` instead of merging it, and take the next free manifest
minor version. Never edit `identity-kit`'s fragments; put everything for this tale in
new files plus the small, listed touch points below.

## Design: the mount primitive

### Files

- New fragment `src/240e-plugin-property-card-title-slot.js`, listed in
  `src/fragments.json` immediately after `240d-plugin-property-card-host-editor.js`. If
  `identity-kit` has already landed a `240e-*` file, use the next free suffix (`240f`,
  …) and keep it directly after the kit fragments. Target about 250–350 lines.
- New export fragment `src/374a-exports-property-card-title-slot.js`, listed directly
  after `374-exports-property-card.js`, in the same style (merge into
  `module.exports.helpers`). A separate file avoids merge conflicts with
  `identity-kit`'s edits to `374`.
- One line in `src/170-plugin-lifecycle-teardown.js`: inside
  `teardownLedgerToolsSurfaces`, next to the existing record and intention card
  teardowns, `try { propertyCardTitleSlotReleaseAll(); } catch (_) {}`. Fragments share
  one script scope, so the function declaration from `240e` is callable here at runtime.
- A scoped CSS block in `styles.css` keyed only on the new mount class (see below), so
  existing cards' pixels cannot change.

Reuse the existing kit helpers from `240b-plugin-property-card-ownership.js`:
`propertyCardScopeIsEmbed`, `propertyCardDesktop`, and the direct-child lookup style
from `propertyCardNativeContainer`. Do not change their behavior.

### API (all plain function declarations in the shared script scope)

```js
// Resolve the title slot for one native view scope, or null when denied.
// Returns { mode: "live" | "reading", scope, parent, anchor } where parent is the
// .cm-sizer (Live Preview) or .mod-header (Reading) and anchor is the node the card
// goes before (null means append to parent).
propertyCardTitleSlotFor(scope);

// Mount node as the first child after the inline title and before native Properties.
// Returns an idempotent, generation-bound release function, or null when the slot is
// unavailable or insertion fails (the adapter's signal to fall back to the body).
propertyCardMountInTitleSlot(scope, node, { path, kind, view, currentPath, onLost });

propertyCardTitleSlotRelease(scope); // scope-wide release (tests, teardown)
propertyCardTitleSlotReleaseAll(); // plugin unload: remove every mounted node
propertyCardTitleSlotMountedNode(scope); // introspection: current node or null
propertyCardTitleSlotLiveCount(); // introspection: live mount entries
```

Constants: `PROPERTY_CARD_TITLE_SLOT_CLASS = "bob-property-card-title-slot"` (added to
the mounted node while mounted) and a re-insertion budget constant (for example 24
re-insertions per 1000 ms).

### Slot resolution (`propertyCardTitleSlotFor`)

Return `null` (deny) when any of these hold:

- `scope` is missing, lacks `closest`, or has `isConnected === false`;
- the scope is inside an embed or hover popover (`propertyCardScopeIsEmbed`), a canvas
  node (`.canvas-node`), or a print/export container (`.print`);
- not desktop (`propertyCardDesktop() !== true`);
- a `.cm-editor` scope whose `.markdown-source-view` ancestor lacks `is-live-preview`
  (Source mode keeps native rendering);
- the expected direct-child chain is missing: `.cm-editor > .cm-scroller > .cm-sizer`
  plus a direct-child `.cm-contentContainer`, or
  `.markdown-preview-view > .markdown-preview-sizer > .mod-header`;
- any other scope class.

Anchor rule (direct children of `parent` only, never descendants):

- Live Preview: before `.metadata-container` when present, else before
  `.cm-contentContainer`.
- Reading: before `.metadata-container` when present, else append to `.mod-header`.

This places the card after `.inline-title` (which Obsidian renders first in both
parents) for every Properties setting: visible, hidden, and source.

### Mounting (`propertyCardMountInTitleSlot`)

Model it on `propertyCardOwnKeys` in `240b`:

1. Prune entries whose scope is disconnected. Resolve the slot; on `null`, release any
   existing entry for the scope and return `null`.
2. If `node` is currently mounted for a different scope, release that entry first. If a
   different node is mounted for this scope, remove the old node.
3. Insert with `parent.insertBefore(node, anchor)` only when the node is not already in
   the correct position (correct means `node.parentNode === parent` and its next element
   sibling is `anchor`, or it is the last child when `anchor` is null). Add
   `PROPERTY_CARD_TITLE_SLOT_CLASS` and `data-title-slot-mode="live|reading"` (plus
   `data-title-slot-kind` from `kind` when given). If insertion throws, clean up and
   return `null`.
4. Bump a module generation counter, store the entry
   `{ generation, node, mode, parent, path, kind, view, currentPath, onLost, observers, reinserts, budget }`
   in a module `Map` keyed by scope (a `Map`, not a `WeakMap`, so `ReleaseAll` can
   iterate; pruning on every mount keeps it bounded).
5. Observers, all created from `scope.ownerDocument.defaultView` and guarded so tests
   and old hosts without them still mount:
   - A `MutationObserver` with `childList: true` on `parent`, plus in Reading on the
     `.markdown-preview-sizer` (catches `.mod-header` replacement), plus `attributes`
     with `attributeFilter: ["class"]` on the `.markdown-source-view` ancestor in Live
     Preview (catches a switch to Source). The callback re-resolves the slot: on `null`,
     self-release and call `onLost(reason)`; when `parent` changed, move the observers
     to the new parent; when the node is out of position, re-insert and count it. When
     `currentPath` is a function and its value differs from `path`, self-release with
     reason `"path-changed"`. Exceeding the re-insertion budget self-releases with
     reason `"unstable"`, so the primitive never fights another writer forever and the
     adapter falls back.
   - A `ResizeObserver` on the node that calls `view.requestMeasure()` when `view` has
     it (Live Preview CodeMirror `EditorView`). Also call `requestMeasure` once after
     each Live Preview insertion.
   - Do not observe `subtree` on the scope; CodeMirror content mutations must never
     reach the callback.
6. Return the release function. It is a no-op unless the stored entry for the scope is
   still this generation. Otherwise it disconnects every observer, removes the node from
   the DOM, removes the class and data attributes, and deletes the entry. Calling it
   twice is harmless. `onLost` fires only for self-releases, never for an explicit
   release, and a throwing `onLost` is swallowed.

`propertyCardTitleSlotReleaseAll()` releases every entry (no `onLost`). After it runs,
no `.bob-property-card-title-slot` node and no live observer may remain.

### CSS

Add one small block to `styles.css` that targets only
`.cm-sizer > .bob-property-card-title-slot` and
`.mod-header > .bob-property-card-title-slot`. Make it `display: block`, keep it in the
column (`max-width: 100%`, `box-sizing: border-box`), and give it bottom spacing that
matches the gap between the inline title and Properties. No rule may match existing card
classes. Nothing user-visible changes, because no adapter mounts through the slot yet.

## Unit tests (fake DOM, `npm test`)

Add `scripts/test-ledger-tools-property-card-title-slot.cjs` and register it in the
`test:unit` list in `package.json` (next to
`scripts/test-ledger-tools-property-card-host.cjs`). Load
`plugins/bob-ledger-tools/main.js` with the same `Module._load` stubs as
`scripts/test-ledger-tools-property-card-ownership.cjs`, and give the fake node
`insertBefore`, `nextElementSibling`, `previousElementSibling`, `firstElementChild`, and
`isConnected`. Fake `MutationObserver` and `ResizeObserver` classes record their observe
options and let the test fire callbacks. Cover:

- **Insertion order for each Properties setting:** Live Preview with
  `.inline-title, .metadata-container, .cm-contentContainer` puts the card between the
  title and Properties; Live Preview without `.metadata-container` puts it before
  `.cm-contentContainer`; Reading with and without `.metadata-container`, including
  append to `.mod-header`; a nested (non-direct-child) `.metadata-container` is ignored.
- **Re-insertion:** remove the node, then fire the observer: it returns to position.
  When Obsidian inserts a new `.metadata-container` before the card, it is re-anchored.
  When `.mod-header` is replaced, it moves to the new header and observers follow. A
  node already in place causes no DOM write. Budget exhaustion self-releases with
  `onLost("unstable")`.
- **Release generations:** an old handle after a re-mount is a no-op; release removes
  the node, class, and attributes and disconnects every observer; double release is
  harmless; re-mounting the same node on the same scope is idempotent (one node, one
  entry); mounting the node on scope B releases scope A; mounting a different node on
  the same scope removes the old node.
- **Denials return `null` with no DOM change:** null or non-element scope, embed, hover
  popover, canvas node, print, mobile (`Platform.isDesktopApp = false` on the stub),
  Live Preview scope in Source mode (no `is-live-preview`), missing `.cm-sizer`,
  `.mod-header`, or `.cm-contentContainer`, and a disconnected scope.
- **Self-release:** the source-view class losing `is-live-preview` fires
  `onLost("denied")` and removes the node; a `currentPath` mismatch fires
  `onLost("path-changed")`; a throwing `onLost` is swallowed.
- **Measure:** firing the ResizeObserver calls `view.requestMeasure` (Live Preview);
  Reading or a missing `view` never throws.
- **Teardown:** `propertyCardTitleSlotReleaseAll()` leaves zero mounted nodes, zero
  connected observers, and `propertyCardTitleSlotLiveCount() === 0`; the plugin
  instance's `teardownLedgerToolsSurfaces()` (or `onunload()`) calls it. Assert through
  a mounted fake rather than by grepping source.
- Insertion that throws returns `null` and leaves no entry.

## The real-Obsidian spike

### Files

- `scripts/obsidian-title-slot-spike-lib.cjs`: offline helpers, including the help text,
  `parseArgs` (`-e/--executable`, `-o/--out`, `-r/--report`, `-s/--self-test`,
  `-h/--help`, matching `scripts/obsidian-intention-card-e2e-lib.cjs`), fixture builder,
  page-side source strings (probe adapter, DOM probes), the check inventory,
  `computeVerdict(report)`, and `selfTest()`.
- `scripts/obsidian-title-slot-spike.cjs`: the runner. Reuse the shared isolated-vault
  and CDP helpers exported by `scripts/obsidian-record-card-e2e.cjs` (`freePort`,
  `selectTarget`, `seedProfile`, `initScratchGit`, `writeNote`, `trustCommunityPlugins`,
  `waitFor`, `capture`, `setViewport`, `collapseSidebars`, `setTheme`, `sha256File`,
  `hashSourceTree`, `sleep`) and `keyEvents` from
  `scripts/obsidian-intention-card-e2e-lib.cjs`. Follow the launch, report, and cleanup
  shape of `scripts/obsidian-intention-card-e2e.cjs`. Neither bob nor `BOB_BIN` is
  needed. Install only `bob-ledger-tools` from the working tree, and record plugin
  provenance (HEAD, dirty paths, file hashes) in the report. Never open the live vault.
- `scripts/test-obsidian-title-slot-spike.cjs`: the offline wrapper, registered in
  `test:unit`. It runs the runner with `--self-test` (expects
  `offline self-test complete`), unit-tests `computeVerdict` (every-check pass → GO,
  only Reading passes → partial GO naming Live Preview as fallback, both fail → NO-GO, a
  missing required check → fails), checks `parseArgs` errors, and compiles every
  page-side source string with `new Function` without running it.

### Vault, profile, and settings

Use a scratch vault and profile in `os.tmpdir()`, scratch Git, sidebars collapsed, a
1400×1000 viewport, and core plugins `file-explorer`, `command-palette`, `page-preview`,
`properties`, and `canvas`. Start from `app.json` values `livePreview: true`,
`showInlineTitle: true`, `propertiesInDocument: "visible"`, `readableLineLength: true`,
and `vimMode: true`. Fixtures:

- `alpha.md`: five frontmatter keys, an H1, about 160 numbered body lines
  (`Line 001 alpha` …), and `## Section 01`–`## Section 08` headings spread through the
  body;
- `beta.md`: different frontmatter and body;
- `nofm.md`: no frontmatter;
- `embedder.md`: embeds `![[alpha]]` and links `[[alpha]]` (for the hover popover);
- `board.canvas`: one file node showing `alpha.md`.

Hash every fixture before launch and again at the end. Rendering must change no bytes.

### Probe adapter (page side)

Inject `window.__bobTitleSlotSpike` via `Runtime.evaluate`. It models a real adapter:

- It reads the primitive from
  `app.plugins.plugins["bob-ledger-tools"].constructor.helpers` on **every** call, so
  disabling and re-enabling the plugin picks up the fresh module. When the plugin is
  absent, it does nothing.
- `refresh()` visits every markdown leaf. The scope is the view's `.cm-editor`
  (`view.editor.cm.dom`) in Live Preview and the `.markdown-preview-view` that is, or is
  inside, `view.previewMode.containerEl` in Reading (probe which one in this Obsidian
  build and record it). It mounts one probe card per visible scope with
  `{ path, kind: "probe", view: cm (Live Preview only), currentPath: () => view.file?.path, onLost }`
  and releases mounts for scopes that are hidden, belong to a different path, or are no
  longer visible. The probe card is a `div.bob-property-card.bob-title-slot-probe` with
  `data-path`, a visible text label, and a configurable height (default 120 px,
  `setHeight(px)` changes live cards).
- It registers `refresh` on workspace `layout-change`, `file-open`,
  `active-leaf-change`, and metadataCache `changed` events, and keeps its own event refs
  so `stop()` removes them. `pause()`/`resume()` suspend refresh so a scenario can prove
  the primitive's own survival without adapter help.
- It counts mount calls, `null` returns per reason, and `onLost` reasons for the report.
- `tryDenied()` attempts mounts on every `.cm-editor` and `.markdown-preview-view` in
  the document that is inside an embed, hover popover, or canvas node. Each must return
  `null`.

### DOM probe (page side)

For one leaf, return the mode, the inline title, Properties, the probe card, and the
content element (`.cm-contentContainer` or the first section after `.mod-header`).
Include their DOM order (`compareDocumentPosition`), bounding rects, computed
visibility, and parent classes. Also return the document-wide count of
`.bob-property-card-title-slot` and `.bob-title-slot-probe` nodes, connected nodes
outside a valid slot parent (orphans), and `propertyCardTitleSlotLiveCount()`.

**Placement passes** when all of the following hold:

- the card's parent is `.cm-sizer` (Live Preview) or `.mod-header` (Reading);
- in DOM order, the card follows `.inline-title` and precedes `.metadata-container`
  (when present) and the content;
- in geometry, the card's top is at or below the inline title's bottom when that title
  is visible, and its bottom is at or above Properties' top and the content's top (2 px
  tolerance);
- the card stays within the column with no horizontal overflow, and its width is at
  least 90% of the inline title's or Properties' width when either is visible;
- exactly one probe exists per visible pane, none is orphaned, and none is in a hidden
  mode's scope.

### Scenarios and required checks

Tag every check with its surface (`live`, `reading`, or `both`) and record measurements
in its detail. Required checks (each must appear in the report; a missing row fails):

1. **Survival**
   - `switch-file-live`, `switch-file-reading`: alpha → beta → nofm → alpha in one leaf.
     Placement passes each time, and no stale probe carries an old path.
   - `mode-live-reading-source`: Live Preview → Reading → Source → Live Preview. In
     Source, no probe is visible in the leaf, native inline title and Properties are
     visible, and `onLost`/release fired. The others place correctly.
   - `split-panes`: alpha left and beta right, each with its own correct probe.
   - `same-file-two-panes`: alpha in two panes, one probe per pane. Closing one pane
     leaves the other intact and drops `liveCount` after the next mount (pruning).
   - `metadata-rerender-live`, `metadata-rerender-reading`: with the adapter **paused**,
     `app.fileManager.processFrontMatter` adds, changes, and removes a key, and
     frontmatter is added to `nofm.md`. The card survives, stays before the re-rendered
     Properties, and `reinserts` is recorded. Bytes then return to the fixture text
     through the vault API, and the final hash check covers that.
   - `window-resize`: with the adapter paused, set the viewport to 1400, 900, and 375,
     then back to 1400. Placement passes at each width.
2. **Settings matrix** (`settings-matrix-live`, `settings-matrix-reading`): every
   combination of `propertiesInDocument` visible, hidden, and source; `showInlineTitle`
   on and off; `readableLineLength` on and off; and light and dark (`setTheme`). Apply
   settings with `app.vault.setConfig` and reopen the note. One row records each
   combination's placement result; the check passes only when all 24 pass per surface.
   Capture at least eight representative screenshots.
3. **Tall and growing card** (Live Preview unless noted; the probe at 600 px, then grown
   to 1400 px after mount, then shrunk to 200 px):
   - `click-accuracy`: for three body lines (top, middle, and one needing scroll),
     scroll the line into view, click its `.cm-line` with CDP `Input.dispatchMouseEvent`
     (pressed and released), and assert `editor.getCursor().line` is that line. Repeat
     after the growth and after the shrink.
   - `coords-at-pos`: for the same lines, `cm.coordsAtPos(line.from).top` equals the
     `.cm-line` rect top within 2 px, before and after growth and shrink. Also record
     the same measurement for an unmeasured control mount (no `view` option) as
     non-mandatory evidence of what `requestMeasure` buys.
   - `scroll-restore-live`, `scroll-restore-reading`: scroll to about line 120, save
     `view.getEphemeralState()` and the top visible line, open beta, reopen alpha with
     that `eState` (and separately through `leaf.history.back()`), and assert the top
     visible line is within ±1 line of the saved one.
   - `heading-jump-live`, `heading-jump-reading`: open alpha with
     `eState: { subpath: "#Section 06" }`. The heading lands in the viewport, near the
     top (within the native tolerance you measure for the same jump with the adapter
     stopped; record both).
   - `vim-gg-G`: focus the editor and send `G` (Shift) through `keyEvents`. The cursor
     reaches the last line, which is visible. Then `gg`: the cursor is on line 0 and
     that line is visible. Record whether the card top is visible after `gg`
     (informational).
4. **Denials** (`deny-embed`, `deny-hover-popover`, `deny-canvas`): open `embedder.md`
   in Reading and Live Preview, trigger a hover popover on `[[alpha]]` (for example
   `app.workspace.trigger("hover-link", …)` or a CDP mouse hover with Ctrl), open
   `board.canvas`, and run `tryDenied()`. Every attempt returns `null`, and no probe
   appears inside an embed, popover, or canvas node.
5. **Teardown** (`teardown-plugin-disable`): with probes mounted in two panes, run
   `app.plugins.disablePlugin("bob-ledger-tools")`. Zero `.bob-property-card-title-slot`
   and zero probe nodes remain in the document. Then edit metadata and wait 500 ms:
   still zero, so no zombie observer re-inserts. Native inline title and Properties are
   visible. Re-enable the plugin, call `refresh()`, and probes mount again through the
   new module. `teardown-leaf-close`: detaching a leaf leaves no connected orphan.
6. **Hygiene:** `bytes-unchanged` (every fixture hash matches at the end),
   `screenshots-captured` (at least 12: Live Preview and Reading, light and dark, wide
   and 375 px, tall card, and Properties hidden and source), and
   `no-console-exceptions`.

### Verdict

`computeVerdict(report)` returns `GO` when every required `live`, `reading`, and `both`
check passes. It returns a partial GO when every `both` check and every check for one
surface pass, naming the other surface as falling back to the body (for example
`GO (reading only; live preview falls back to body)`). Otherwise it returns `NO-GO`.

When a check fails because of a primitive bug, fix the primitive and rerun. Downgrade
the verdict only when the evidence shows an Obsidian limitation that the primitive
cannot work around, and describe that limitation in the note.

### Running on athena

```bash
npm run build
xvfb-run -a node scripts/obsidian-title-slot-spike.cjs \
  -o <scratch-dir>/shots -r <scratch-dir>/title-slot-spike-report.json
```

The default Electron path is `/snap/obsidian/current/app/obsidian` (Obsidian 1.15.1 at
planning time), or pass `-e`. Use a scratch directory under `/tmp`, never inside a repo
checkout. The spike takes a few minutes; run it in the foreground with a generous
timeout, or hand it to `/sase_monitor`. Open and inspect every screenshot with the Read
tool, and record a one-line visual verdict for each in the report's `screenshots`
entries before attaching.

## Recording the verdict

1. Read `sase memory read sase_artifacts.md -r "<why>"`. Attach the report and the key
   screenshots, at least one Live Preview and one Reading in each theme, the tall card,
   and the 375 px capture:
   `sase artifact create -p <file> -l "<label>" --bead bob-cli-o6.1`.
2. Add exactly one verdict note:
   `sase bead note bob-cli-o6.1 'TITLE SLOT: GO — <evidence summary>'` (or
   `TITLE SLOT: NO-GO — …`, or a partial GO that names each surface that falls back to
   the body). The note must give adopters what they need:
   - the artifact refs;
   - Obsidian version, plugin version, and commit;
   - the pass counts per surface and key measurements: click and coords deltas,
     scroll-restore deltas, and reinsert counts;
   - any surface or setting that falls back;
   - the primitive's API in one line:
     `propertyCardMountInTitleSlot(scope, node, { path, kind, view, currentPath, onLost })`
     returns a release handle, or `null` meaning body fallback, and `onLost(reason)`
     means remount or fall back.
3. Record any discovered out-of-scope work as
   `sase bead note bob-cli-o6.1 'PROPOSED FOLLOW-UP: <summary — detail>'`. Do not create
   beads.

## Docs

Run `sase memory read docs.md -r "<why>"` first. Add a short "Title slot" paragraph to
the "Property-card extension API" section of `docs/develop/architecture.md` in
bob-plugins. It should say what the primitive does, where it inserts, that `null` means
body fallback, the release and `onLost` contract, teardown, and the spike script plus
its offline test. User docs do not change, because nothing user-visible ships.

## Gates, version, commit, deploy

In bob-plugins, after merging `origin/master`:

- `npm run build`, `npm run build:check`, `npm test`, and `npm run validate`.
- `npm run docs:check`, with `BOB_CLI_DIR` pointing at this project's bob-cli primary
  workspace checkout (the agent's own workspace root).
- Bump `plugins/bob-ledger-tools/manifest.json` to the next free minor version (1.143.0
  at planning time, or higher if `identity-kit` took it), then rebuild.
- The bob-plugins changes become a repo obligation of the turn and are committed through
  the normal SASE finalization (`/sase_final` commit decision). Use a conventional
  message, for example
  `feat(ledger-tools): add title-slot mount primitive and real-Obsidian spike`.
- Run `bob plugins sync -p bob-ledger-tools` and confirm the vault copy is
  byte-identical to the built files. On a refusal, follow `AGENTS.md` and never pass
  `-F` on your own.

A gate failure that reproduces identically on a clean `origin/master` checkout does not
block this tale. Record it as a `PROPOSED FOLLOW-UP:` note (citing any task bead that
already tracks it) and continue.

## Closing the bead

Run `sase bead epic-symbols bob-cli-o6.1` (at planning time there were none). Then close
only this bead:
`sase bead close bob-cli-o6.1 --note "<verdict, gates, version, deploy evidence>"`.
Never close `bob-cli-o6` or any other phase, and never run `sase bead open` on this
bead. Never edit any memory file.

## Acceptance

- `240e` (or the next free suffix) exports the six API functions. Unit tests cover
  insertion order for every Properties shape, re-insertion, release generations,
  denials, self-release, measure, and teardown. All of it is green in `npm test`.
- The spike runs on athena under xvfb against the built plugin. Its report lists every
  required check with surface tags and measurements, and the screenshots were inspected.
- Bead `bob-cli-o6.1` carries one `TITLE SLOT:` verdict note with evidence and artifact
  refs. A partial verdict names each fallback surface.
- No existing card changes placement or pixels. Fixture bytes are unchanged by the
  spike. Plugin disable leaves no slot nodes or observers.
- Gates are green (or red only on documented pre-existing failures), the version is
  bumped, the changes are committed, the vault is synced, and the bead is closed with
  its evidence.
