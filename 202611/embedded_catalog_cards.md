---
tier: tale
title: Keep host catalog cards out of block embeds and hover previews
goal:
  Block embeds and hover previews of a Projects, Areas, Albums, or Notebooks host's
  blocks never paint that host's standalone catalog card, while host notes and stored
  callouts render as today.
size: medium
proposed_by: bbugyi200.athena.1gw
proposed_with: claude/opus
create_time: 2026-11-27 06:14:34
status: wip
---

# Plan: Keep host catalog cards out of block embeds and hover previews

## Problem

Opening `ref/chat/property_card_cabinets.md` (a reference note) shows a standalone
**Projects** card ("Projects 3 open · 0 blocked · 4 closed", `＋ project`, `Closed · 4`
with the two-row peek `bob_command_mode_improvements`, `bob_gtd`) above the embedded
reading task. That card belongs to `bob.md`, the note that holds the reading task. Every
reference note shows its reading task through the managed embed
`![[<residence>#^ref-<slug>]]`, so the same leak hits every reference note whose parent
is a Projects or Areas catalog host. It also hits any other note that embeds a block
from such a host, and hover previews of those blocks. In the live vault about 42
reference-note block embeds target Projects hosts and about 9 target Areas hosts.

## Root cause (reproduced in real Obsidian 1.15.1, the Mac's version)

I reproduced this in a scratch vault under Xvfb with the current `bob-ledger-tools`
build and a real `bob projects sync` / `bob areas sync`. The scratch vault shows the
same card layout as the screenshot.

1. For an embed `![[bob#^id]]`, Obsidian post-processes only the embedded fragment, with
   `ctx.sourcePath = "bob.md"`. `ctx.getSectionInfo(el)` returns
   `{ text: <only the embedded block>, lineStart: 0, lineEnd: 0 }`. The section `el` is
   still **detached** at callback time (`el.isConnected === false`,
   `el.closest(".markdown-embed") === null`). `ctx.containerEl`, however, is the live
   embed sizer and sits inside `.markdown-embed` / `.internal-embed`. A full
   Reading-view render instead returns the whole note text, frontmatter included.
2. `renderProjectsCatalogInReading`
   (`plugins/bob-ledger-tools/src/241ae-plugin-projects-catalog.js`) derives
   `allowAnchor` from that fragment. `projectsCardAnchorLine(fragment)` finds no
   `## Projects` or H1, so it falls back to the first non-blank line (0). That line lies
   inside `[0, 0]`, and the fragment has no `[!projects]` region, so `allowAnchor` comes
   out `true`. The function treats a fragment as if it were the whole host note.
3. `paintProjectsCatalogReading` then sees `bob.md`'s rows (3 open, 4 closed) →
   `needsAnchor`. `notebookCatalogAnchorExcluded(el)` cannot see the embed because `el`
   is detached. With no H2 or H1 in the fragment, it inserts `.bob-projects-anchor` at
   `el.firstChild` whenever `el` contains a `<p>`. The Tasks plugin (global filter
   `#task`) leaves a `<p>` in the task item at callback time. A paragraph block embed
   has one natively, with or without Tasks. `projectsPaintStandalone` then paints the
   header, `＋ project`, and the folded Closed peek.
4. In Live Preview, `scheduleProjectsCatalogRefresh` skips widgets inside `.cm-editor`,
   so nothing ever cleans that chrome up.
5. **Areas** (`241aj-plugin-areas-catalog.js`) has the same flaw, and it is worse.
   `areasCardAnchorLine` falls back to the frontmatter end (0), and
   `areasCatalogEnsureAnchor` inserts without any `<p>` guard. Every block embed from an
   Areas host shows an Areas card, and this reproduced without the Tasks plugin.
   **Albums** (`241q-plugin-album-catalog.js`) derive `allowAnchor` identically, though
   they paint only for non-current catalogs. **Notebooks**
   (`241d-plugin-notebook-catalog.js`) create a rendered anchor for missing or stale
   status with no section gate at all.

The cabinets preview gate is off on both hosts (no `data.json`), so cabinets play no
part. Callout painting inside a fragment is a separate path, so a heading embed of
`## Projects` that includes the stored callout should keep painting the callout card.
The e2e control in step 4 verifies this, because the repro did not exercise it.

No decision record governs catalog-card rendering in embeds. I read the roster per
`decisions:decision-rules-are-read-at-planning-time`. The fix brings the code in line
with the documented rule: "One card per host; standalone only without a stored callout"
(`docs/bob-ledger-tools.md`). It also matches the existing code comment in
`241p-plugin-album-catalog-dom.js`: embeds and hover previews "never own a standalone
catalog anchor."

## Fix

All work is in the linked `bob-plugins` repo. Open it with `/sase_repo` and read its
`AGENTS.md`. `bob-ledger-tools` uses the fragment source build: edit `src/` fragments
only, keep each hand-edited fragment at or below 1000 lines, and never hand-edit
`main.js`.

### 1. One shared fragment-render guard

In `plugins/bob-ledger-tools/src/241p-plugin-album-catalog-dom.js`, next to
`notebookCatalogAnchorExcluded`, add a never-throwing helper, for example
`notebookCatalogFragmentRender(el, ctx, sectionText, metadataCache)`. It returns `true`
when any of these holds:

- `notebookCatalogAnchorExcluded(el)` (the existing check).
- `notebookCatalogAnchorExcluded(ctx?.containerEl)`. `ctx.containerEl` is the live
  render container. It is undocumented but present in Obsidian 1.15, so guard it with
  `try` and a type check.
- The source note has frontmatter in the metadata cache
  (`metadataCache.getCache(ctx.sourcePath)?.frontmatterPosition`, or `.frontmatter` as a
  fallback) **and** `sectionText` has none (`notebookFrontmatterEnd(lines) === 0`). A
  full-note render always carries the host's frontmatter; embed and hover fragments
  never do. Skip this test when there is no cache entry or no section text, so untyped
  notes keep today's behaviour.

Write a short comment explaining why `el.closest()` is not enough: Obsidian
post-processes fragments on detached section nodes.

### 2. Apply it at every catalog post-processor

- `renderProjectsCatalogInReading` (241ae): `allowAnchor` becomes `false` for fragment
  renders. Keep painting callouts found in the fragment, and keep registering the widget
  so callout repaints still work. Because `widget.allowAnchor` is stored, the refresh,
  fold, and re-render paths then clear standalone chrome instead of painting it.
- `renderAreasCatalogInReading` (241aj): same change.
- `renderAlbumCatalogInReading` (241q): same change.
- `renderNotebookCatalogInReading` / `paintNotebookCatalogReading` (241d): pass a
  fragment flag. A fragment never creates the `notebookEnsureRenderedAnchor` fallback
  anchor (treat `needsAnchor` as false). Callout painting and existing leftover-chrome
  cleanup stay unchanged.

Use `this.app?.metadataCache` for the cache argument. Do not change Live Preview
decoration paths (`buildProjectsCatalogDecorations` and its siblings); they key on the
outer editor's own file and are not involved.

### 3. Unit tests (fake DOM, run in `npm test`)

Model the new tests on "missing-catalog controls occur only in the selected source
section" in `scripts/test-ledger-tools-projects-catalog.cjs`.

- **Projects:** use a host with current-catalog rows (open and closed) and a
  **detached** `el` (`div > p`, plus a `div > ul > li > p` Tasks-like variant).
  `getSectionInfo` returns a fragment
  (`{ text: "Embedded paragraph. ^para", lineStart: 0, lineEnd: 0 }`). Cover
  `ctx.containerEl` placed inside a `.markdown-embed`, inside a `.hover-popover`, and
  absent (with `metadataCache.getCache("body.md")` reporting `frontmatterPosition`).
  Each case renders no `.bob-projects-anchor`, `.bob-projects-header`, or
  `.bob-projects-closed`.
- **Projects control:** a full-note section text with frontmatter, no callout, and a
  normal `containerEl` still anchors as today.
- **Projects re-render:** a widget painted earlier and re-rendered as a fragment is
  cleaned on the next `scheduleProjectsCatalogRefresh`.
- Add the equivalent fragment cases (embed `containerEl` and frontmatter-mismatch) to
  `scripts/test-ledger-tools-areas-catalog.cjs` (current catalog with child areas; no
  `<p>` needed), `scripts/test-ledger-tools-album-catalog.cjs` (non-current catalog),
  and `scripts/test-ledger-tools-notebook-catalog.cjs` (missing catalog, no rendered
  anchor).
- Unit-test the helper itself: `ctx` missing, `containerEl` missing, `getCache`
  throwing, and a BOM-prefixed frontmatter fence.

### 4. Real-Obsidian acceptance (required)

Extend `scripts/obsidian-projects-ui-e2e.cjs`, which already seeds Projects and Areas
hosts:

- Add a fixture note, for example `embeds.md`, containing:
  - a paragraph block embed from `project-host`. Add a trailing paragraph with a block
    ID to `hostNote()` without disturbing existing checks; the paragraph form reproduces
    without the Tasks plugin.
  - a task block embed from `area.md` (`![[area#^area-task]]`), which reproduces the
    Areas leak.
  - a heading embed of a host's `## Projects` section that includes its stored callout,
    as a control.
- Open it in Live Preview and in Reading view. Record checks that every
  `.markdown-embed` holds zero `.bob-projects-anchor`, `.bob-projects-header`, and
  `.bob-areas-anchor` outside a `[data-callout]` callout. Also check that the embedded
  text is visible, and that the control heading embed still paints its callout card.
  Capture a screenshot of each mode.
- Keep `--self-test` (the offline wrapper in `npm test`) passing. Extend its fixture
  assertions for the new note.
- Run the real walkthrough as `docs/develop/validation.md` describes. Build bob from the
  current bob-cli checkout (`cargo build`), then set `BOB_BIN` and `BOB_CLI_DIR` and run
  under `xvfb-run -a` with `-e /snap/obsidian/current/app/obsidian`. Inspect every
  screenshot and confirm all checks pass, including the pre-existing ones.

### 5. Docs, version, build, deploy

- In `docs/bob-ledger-tools.md`, add one terse sentence each to "Projects catalog card"
  and "Areas": embeds and hover previews of a host's blocks never show a standalone
  card, while an embedded stored callout still paints. Run `npm run docs:gen` if the
  inventory changes. Then run `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`.
- Bump the `plugins/bob-ledger-tools/manifest.json` patch version from whatever is
  current at commit time (it is a fix).
- Run `npm run build`, `npm run build:check`, `npm run docs:inventory:check`,
  `npm test`, and `npm run validate`. All must pass.
- Run `bob plugins sync` to deploy to the vault. If it refuses an overwrite, follow the
  `AGENTS.md` procedure; never pass `-F` on your own.
- The Mac deploys from its own bob-plugins checkout, so the fix reaches the screenshot
  host on that machine's next sync. Say so in the final report.

## Acceptance criteria

- The Projects card no longer appears in `ref/chat/property_card_cabinets.md`. No block
  embed or hover preview of a Projects, Areas, Albums, or Notebooks host's blocks paints
  standalone catalog chrome, in Live Preview or Reading view, with or without the Tasks
  plugin.
- Host notes themselves are unchanged: standalone cards and missing-catalog actions
  still appear in Reading view and Live Preview where they do today. Stored callouts,
  including ones inside heading embeds, still paint.
- New unit tests fail on the current code and pass with the fix. The full `npm test` and
  `npm run validate` pass. The extended real-Obsidian Projects walkthrough passes with
  screenshots inspected.
- No vault note bytes change, and no new Bob process is started by painting.
