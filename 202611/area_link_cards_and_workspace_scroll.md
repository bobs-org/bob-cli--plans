---
tier: tale
title:
  Fix dead and duplicated catalog-card links and the white workspace after area-link
  clicks
goal:
  Area and project row links open their notes, each host shows one card, and the
  workspace never scrolls away.
size: medium
decisions:
  scroll_guard:
    ask:
      Should bob-ledger-tools also undo any scroll of Obsidian's never-scrolling
      workspace containers and log the cause?
    choices:
      guard:
        Add the guard; the white screen can't stick whatever the trigger, and each catch
        logs its cause.
      skip:
        Fix only the confirmed card, click, and focus defects; a recurrence needs fresh
        debugging.
    default: guard
    why:
      The white-screen trigger didn't reproduce on Linux; these containers never
      legitimately scroll.
    answer: guard
proposed_by: bbugyi200.apollo.ec
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 06:50:16
status: wip
---

# Plan: Fix dead and duplicated catalog-card links and the white workspace after area-link clicks

All code changes live in the **bob-plugins** linked repo (open it with `/sase_repo`;
read its `AGENTS.md` first). Edit only `plugins/bob-ledger-tools/src/` fragments, then
regenerate `main.js` with `npm run build`; never hand-edit `main.js`.

## What Bryan sees

After clicking area links in a note's `## Areas` section, the Obsidian window on the Mac
goes white. The ribbon, tab bar, and editor are gone; a thin strip at the very top shows
a Dataview `[root:: gtd]` field (the "ROOT gtd" chip). Only the floating status bar
stays visible. The status bar matched `bob.md` (9 properties, about 35.5k characters).
The Mac's `.obsidian/workspace.json` shows one tab group with pinned `2026/20261116` and
`dash` tabs, both sidebars collapsed, and the recent trail `area.md → dev.md → bob.md`.

## Diagnosis

The investigation used real Obsidian on athena (`/snap/obsidian/current/app/obsidian`
under `xvfb-run`) with a scratch copy of the vault, the deployed plugins, and real CDP
mouse clicks. These defects were **reproduced**:

1. **Misplaced Areas cards.** `renderAreasCatalogInReading`
   (`src/241aj-plugin-areas-catalog.js`) is a Markdown post-processor. When
   `ctx.getSectionInfo(el)` is null, it sets `allowAnchor = true`. `areasPaintReading`
   then inserts a full standalone `.bob-areas-anchor` card at `el.firstChild` of any
   render that has no `[!areas]` callout. Its "CodeMirror owns this" guard is
   `el.closest(".cm-editor")`, which misses Live Preview callout renders because the
   post-processor runs on a detached element. Observed results:
   - A full Areas card painted inside the `## Notebooks` callout of `cash.md` and
     `love.md` in Live Preview.
   - A full Areas card painted inside a Tasks-plugin task description
     (`span.task-description`) in `cash.md` Reading view.
2. **Reading-view duplicates.** Every Areas host (`area`, `gtd`, `love`, `cash`,
   `inbox`) got a standalone card right under `## Areas` _and_ the painted `[!areas]`
   callout. The no-callout test is `callouts.length` of the one post-processed section,
   not the whole note.
3. **Dead links in standalone cards.** `areasPaintStandalone`
   (`src/241ai-plugin-areas-catalog-dom.js`) deletes and rebuilds `ul.bob-areas-list` on
   every paint. The Live Preview view plugin (`createAreasCatalogExtension().update`)
   schedules a paint on every CodeMirror update, including the click's own mousedown.
   The link under the pointer is therefore replaced before mouseup, and the browser
   never fires `click`. The CDP trace showed pointerdown, mousedown, pointerup, mouseup,
   no `click`, and no `file-open`. Links in the stored `[!areas]` callout still work.
   The misplaced card under `## Notebooks` looks exactly like the real one, so users
   click its dead links.
4. **Same gating flaw in Projects and Albums.** `renderProjectsCatalogInReading`
   (`src/241ae-plugin-projects-catalog.js`) and `renderAlbumCatalogInReading`
   (`src/241q-plugin-album-catalog.js`) use the same
   `allowAnchor = anchorLine === null || …`. Observed: a hover page preview of `bob.md`
   painted a standalone Projects header onto the note's H1 in addition to the callout
   card. Both have partial guards (`!el.closest(".markdown-embed, .cm-editor")` and the
   heading/title host) that Areas lacks.
5. **Focus that scrolls the workspace (latent).**
   - `notebookHeaderRestoreFocus` (`src/241g-plugin-notebook-header-live.js`) focuses
     the first matching header link anywhere in the document. It does not check that the
     link is in the owning pane and does not pass `preventScroll`.
   - `paintNotebookHeaderReading` (`src/241f-plugin-notebook-header.js`) and
     `notebookFocusSummaryProperty` (`src/241d-plugin-notebook-catalog.js`) also call
     `.focus()` without `preventScroll`.
   - When a focused element lies outside its container, Chromium scrolls even
     `overflow: hidden` ancestors to show it.

**Not reproduced: the white screen.** Measured in real Obsidian:

- `body` is `overflow: clip`.
- `.status-bar` is `position: fixed`.
- `.horizontal-main-container`, `.workspace`, `.workspace-tabs`, and
  `.workspace-tab-container` are `overflow: hidden`.

A white window that keeps only the status bar therefore means something scrolled
`.workspace` or `.horizontal-main-container` programmatically. Likely sources are a
CodeMirror `scrollIntoView` (it scrolls every overflowing ancestor), `Element.focus()`,
or `scrollIntoView()`. The same click path on Linux (`area.md → [[dev]] → [[bob]]`, Mac
tab layout, 1429×864) left every container at scroll 0. So the plan removes the
confirmed triggers (items 1–5) and, under the `scroll_guard` decision, makes the symptom
impossible and self-reporting.

## Changes

### 1. Paint standalone catalog anchors only in the host's own Reading sections

Areas (`241aj`), Projects (`241ae`), and Albums (`241q`) post-processors:

- A **null** section info, or a section that does not contain the catalog's anchor line
  (`## Areas` / `## Projects` / the album anchor), means **no standalone anchor**. Keep
  painting any stored callout found in `el` exactly as today.
- No standalone anchor when the whole note (`section.text`) already has the stored
  callout. Areas already has `areasCardHasCallout(text)`. Add the equivalent check for
  Projects and Albums if they lack it, reusing their existing callout matchers.
- Never insert a standalone anchor when `el`, or the node it would attach to, is inside
  a callout, `.markdown-embed`/`.internal-embed`, a hover popover (`.hover-popover`,
  `.popover`), a Tasks render (`.plugin-tasks-list-item`, `.tasks-list-text`,
  `.block-language-tasks`), or a Dataview render (`.dataview`,
  `.block-language-dataview`, `.block-language-dataviewjs`). Re-check this at refresh
  time, because post-processing can happen while `el` is detached:
  - `scheduleAreasCatalogRefresh` and the Projects/Albums refresh loops must skip such
    widgets.
  - They must also remove any standalone chrome already painted there, keeping callout
    chrome that shares the class names, as `paintProjectsCatalogReading` already does.
- Put the shared predicate in one helper in an existing shared catalog fragment (the
  notebook or album DOM helpers already shared by these cards) instead of three copies.
  Keep every hand-edited fragment at or below 1000 lines.
- Live Preview hosts without a stored callout still get the standalone widget from
  `buildAreasCatalogDecorations` and the Projects/Albums equivalents. Do not change that
  path's placement.

### 2. Make standalone Areas rows stable so clicks land

- `areasPaintStandalone` must not rebuild `ul.bob-areas-list` when nothing changed. Key
  the list on the row models, fold map, and source path (as `albumReplaceChrome` keys
  chrome) and reuse the existing nodes on an unchanged key. Rows only change on a real
  report, fold, or path change.
- Confirm that a standalone row link opens its note on click in both Reading view and
  the Live Preview widget. The widget's `ignoreEvent()` returns true, so Obsidian's
  delegated link handler may not see it. If either surface does not open, give
  standalone links:
  - an `href` like the notebook header crumbs,
  - a click handler calling
    `app.workspace.openLinkText(link, sourcePath, Keymap.isModEvent(evt))`,
  - a `hover-link` trigger on mouseover so page preview matches callout rows.
- Apply the same no-rebuild-on-unchanged rule to any Projects standalone list painted
  the same way.

### 3. Focus without scrolling, scoped to the owning pane

- Every catalog/header focus restore (`241f`, `241g`, `241d`; grep `src/241*` for
  `.focus(`) uses `focus({ preventScroll: true })`.
- `notebookHeaderRestoreFocus` restores focus only inside the header of the pane that
  held focus: the active leaf's container, or the surface being repainted. It never
  focuses the first match anywhere in the document.

### 4. Workspace scroll guard

> [!decision] scroll_guard = guard Add the guard below. With `skip`, omit this section
> and its tests and docs line.

- On load, bob-ledger-tools registers a capture-phase `scroll` listener with
  `registerDomEvent` on the main window's document and each popout window (via the
  workspace `window-open` event).
- When the target is `.horizontal-main-container`, `.workspace`, `.workspace-split`,
  `.workspace-tabs`, `.workspace-tab-container`, `.workspace-leaf`, or
  `.workspace-leaf-content` and `scrollTop` or `scrollLeft` is non-zero, reset both
  to 0.
- Never touch `.view-content`, `.cm-scroller`, `.markdown-preview-view`,
  `.workspace-tab-header-container-inner`, or anything else that legitimately scrolls.
- On each reset, log one rate-limited `console.warn` with:
  - the container class and offsets,
  - the active file path,
  - `document.activeElement` (tag, classes, short text),
  - whether a hover popover is open.

  The next Mac occurrence then names its trigger.

- No setting or command is needed. Document the guard in one short line.

### 5. Tests (node harnesses, part of `npm test`)

Extend `scripts/test-ledger-tools-areas-catalog.cjs`,
`scripts/test-ledger-tools-projects-catalog.cjs`, and
`scripts/test-ledger-tools-album-catalog.cjs` with these cases:

- Post-processing with null section info inserts no standalone anchor.
- A host whose note text contains the stored callout gets no standalone anchor in any
  section.
- A host without a callout still gets the anchor in the section holding its heading.
- An `el` inside a callout, embed, popover, Tasks render, or Dataview render gets no
  anchor, and a refresh removes one painted there earlier.
- Repainting an unchanged standalone Areas card keeps the same `a.internal-link` node
  identity. A fold or report change rebuilds it.
- Focus restore uses `preventScroll` and stays in the owning surface.
- Guard: a scroll event on a `.workspace` stub resets it to 0 and warns once per window;
  `.view-content` and `.cm-scroller` stubs are untouched.

### 6. Real-Obsidian acceptance (athena)

apollo has neither Obsidian nor `xvfb-run`; athena has both. Reach athena with
`ssh athena` (Tailscale SSH).

Extend the existing Projects real-CDP walkthrough
(`scripts/obsidian-projects-ui-e2e.cjs` and its offline wrapper), or add an Areas
walkthrough with an offline wrapper registered in `npm test` following the same pattern.
The fixture needs:

- an Areas host with `## Areas` + `[!areas]`, `## Projects` + `[!projects]`,
  `## Notebooks` + `[!notebooks]`, and a `## Tasks` list;
- a `dev`-like area hosting a `bob`-like project;
- a pinned daily-note tab.

Assert, in Live Preview and Reading view:

- exactly one Areas card per host;
- no `.bob-areas-anchor` inside any other callout or task description;
- a real CDP click on an Areas row link, then a Projects row link, opens the target
  (`file-open` fires);
- afterwards `.workspace` and `.horizontal-main-container` stay at scroll 0.

Under `guard`, also scroll `.workspace` programmatically and assert it snaps back to 0
with one warning. Run it with
`xvfb-run -a node <script> -e /snap/obsidian/current/app/obsidian`. Review the
screenshots and record the result. Wait for "Indexing vault…" to finish before the first
click: clicks during indexing are unreliable in a large vault.

### 7. Build, docs, deploy

- In the bob-plugins checkout, run `npm run build`, then `npm test` (which includes
  `build:check`). Bump the bob-ledger-tools manifest version per repo convention.
- Update `docs/bob-ledger-tools.md` "## Areas" (and the Projects card section if wording
  changes):
  - one card per host;
  - a standalone card appears only when the note has no stored callout;
  - row links open the area.

  Add the guard line under `guard`. Run
  `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`.

- Run `bob plugins sync`. If it refuses an overwrite, follow the repo's documented
  refusal protocol, never `-F`.
- In the final report:
  - state plainly that the white screen itself was not reproduced on Linux;
  - list which confirmed triggers were removed;
  - say what the guard logs, so Bryan can check the Mac console if a reset ever fires.

## Out of scope

- bob-cli's Areas report/sync and the stored `[!areas]` bytes (`docs/areas.md` ownership
  rules) are unchanged.
- No memory notes or decision records change.
