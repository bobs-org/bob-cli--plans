---
tier: tale
title: Album header ancestry warning and duplicate chrome fix
goal:
  Album headers warn only on genuinely unresolvable ancestry and render once per
  surface.
size: medium
decisions:
  walk_top:
    ask: Should the owner walk end silently at the first non-area/project ancestor?
    choices:
      stop: The first resolved non-home ancestor ends the chain with no warning
      pass_through:
        Keep walking through non-home notes like pomodoroHomeResolve, collecting crumbs
        above
    default: stop
    why:
      Every real chain tops out at area.md, so both render identically; stop is the
      smaller change
    answer: stop
proposed_by: bbugyi200.apollo.do
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 15:10:38
status: wip
---

# Album Header: Remove The False Ancestry Warning And Duplicate Headers

All work is in the **bob-plugins** linked repo (open it with
`sase repo open bob-plugins -r "..."`; read its `AGENTS.md` before editing). No bob-cli
changes.

## Symptoms (both visible in the vault today)

1. **Every album header shows the warning** "Some owner ancestry could not be resolved;
   only confirmed notes are linked." even when every breadcrumb resolved. Seen on
   `~/bob/podcasts.md` (`dev › Podcasts`) and `~/bob/sase_blog_zettel.md`
   (`dev › SASE … › sase_blog › SASE Blog Zettel`).
2. **Duplicate header chrome in Live Preview.** On `~/bob/sase_blog_zettel.md` the album
   header card (breadcrumb, warning, `＋ zettel`, `⊞ Schema`, collapsed schema line)
   renders once correctly under the H1 **and three more times** inside the
   `> [!records]` callout under `## Records`, even though the album has zero records.

Both defects are in chrome the bob-cli-kr epic owns (shipped by its child epic in
`c2f6834` and reshaped by kr.3 in `f714fee`). The epic's own contract says "Reading view
and Live Preview, one header per source surface, using the notebook header
(`241e`–`241g`) lifecycle patterns" — defect 2 violates it directly.

## Root cause 1: the walk treats the conventional top-of-tree as broken

`plugins/bob-ledger-tools/src/241u-plugin-album-header-model.js`,
`albumHeaderOwnerChain()`: when a resolved ancestor's `pomodoroHomeType` is neither
`area` nor `project`, it sets `broken = true` (the
`kind !== "area" && kind !== "project"` branch) and `albumHeaderModel()` turns any of
`broken || cyclic || truncated` into the warning.

But the canonical vault convention (glossary strand "Area Note") is that a **top-level
area carries `parent: "[[area]]"`** — a link to the type-definition note `area.md`,
which has no `type:` frontmatter and is therefore kind `other`. So every album's chain
(e.g. `sase_blog_zettel → sase_blog → sase → dev → area.md`) ends at a non-home note and
every album warns. The established walk this chrome should match,
`pomodoroHomeResolve()` in `142-pomodoro-home-chips.js`, explicitly walks through
non-area/non-project notes without flagging anything.

### Fix

Rework `albumHeaderOwnerChain()` termination semantics per the `walk_top` decision:

- Reaching a resolved ancestor whose kind is not area/project is the **natural end of
  the owned tree**, never `broken`.

  > [!decision] walk_top = stop The walk simply ends at the first resolved non-home
  > ancestor; that note is never a crumb.

  > [!decision] walk_top = pass_through The walk keeps following the non-home note's
  > `parent` chain, still collecting only area/project crumbs, with the existing cycle
  > set and depth cap applied to every visited note.

- `broken` remains true only when ancestry genuinely could not be resolved: the album
  entry has no parent at all, a parent link fails to resolve, or a raw `parent` value is
  non-blank but malformed (normalizes to nothing). `cyclic` (repeated path) and
  `truncated` (depth cap `ALBUM_HEADER_MAX_ANCESTORS`) keep their meanings.
- The warning copy and the `ancestryMessage` gating in `albumHeaderModel()` stay as they
  are; after this change they fire only for those genuine cases.

Extend `scripts/test-ledger-tools-album-header.cjs` (it already covers the
`parent: "missing"` broken case at ~line 144):

- A chain `album → project → area → area.md` (top-level area parenting the type note,
  with `area.md` carrying `parent: "[[type]]"` and no `type:`) yields all home crumbs,
  `broken/cyclic/truncated` all false, and an empty `ancestryMessage`.
- A direct non-home parent (e.g. an album parented at a notebook note) ends the chain
  without a warning under `stop` (or walks through under `pass_through`).
- Unresolvable parent, malformed parent, cycle, and depth-cap cases still warn.

## Root cause 2: the reading post-processor paints into detached Live Preview chunks

`plugins/bob-ledger-tools/src/241v-plugin-album-header.js`,
`renderAlbumHeaderInReading()`: it paints immediately per post-processed element. Its
two guards cannot catch Live Preview callout content because Obsidian renders callout
markdown on **detached** elements before inserting them into the editor widget:
`el.closest(".cm-editor")` is null while detached, and
`notebookHeaderReadingSurface(el)` falls back to the chunk itself, so each rendered
chunk of the `[!records]` callout becomes its own "surface" and receives its own header
(prepended, since the chunk has no H1). The CM decoration
(`buildAlbumHeaderDecorations`) already owns the one legitimate Live Preview header.

The notebook header solved exactly this in `241f-plugin-notebook-header.js`: it
coalesces post-processor calls per `sourcePath` into a pending batch flushed on
`notebookSetTimeout(…, 0)`, then `notebookHeaderChooseReadingSurface()` picks the first
element **attached** to a real `.markdown-preview-view` / `.markdown-preview-sizer` /
`.markdown-embed` / `.internal-embed` surface and paints once.

### Fix

Adopt that lifecycle in `241v` (reuse the shared helpers where they fit — the fragments
share one script scope):

- Coalesce `renderAlbumHeaderInReading` calls per `sourcePath`; on flush, choose one
  attached surface the way `notebookHeaderChooseReadingSurface` does.
- Unlike 241f's fallback, **never paint when no batched element is attached to a
  preview/embed surface** — for an album note, a fully detached batch is Live Preview
  callout content whose header the CM decoration already provides. Register the widget
  and re-derive the surface from the live element on refresh
  (`refreshAlbumHeaderReading` / the `apiAlbumSubscribe` repaint), bailing whenever the
  element now sits inside `.cm-editor`, so a genuine Reading-view surface that attaches
  late still gets its single header on the next refresh.
- Keep `albumHeaderRemoveReading`'s ownership rules; make repaint idempotent per
  surface. On unload of the bound `MarkdownRenderChild`, remove only that surface's
  header (existing behavior).
- Sweep any strays: a repaint of a surface removes headers that ended up inside
  `.cm-widgetBuffer` / `.cm-content` under it (the `albumHeaderNodeBelongs` CM checks
  already identify them once attached).

Tests:

- Unit (`scripts/test-ledger-tools-album-header.cjs` or a sibling): post-processor
  invoked with several detached chunks for the same album source paints zero reading
  headers; invoked with one chunk attached to a preview surface paints exactly one, and
  a second pass stays at one (idempotence).
- E2E (`scripts/obsidian-album-ui-e2e-lib.cjs` walkthrough): on an album note containing
  a `> [!records]` callout, assert exactly one `.bob-album-header` in Live Preview and
  one in Reading view, and that the standard fixture chain shows no
  `.bob-album-header-ancestry-note`. Keep the offline wrapper
  (`scripts/test-obsidian-album-ui-e2e.cjs`, part of `npm test`) green.

## Docs

`docs/albums.md` already promises "incomplete or cyclic ancestry shows only confirmed
links". Add one sentence stating that the breadcrumb ends at the top-level area and the
warning appears only when a parent link cannot be resolved, is malformed, cycles, or
exceeds the depth cap. Read the `docs.md` reference memory before touching the docs
tree; run `npm run docs:check`.

## Non-goals

- The `album_child_reparent` lint bullets ("this child lives outside …") rendering as
  identical anonymous duplicates: that is schema-chrome lint territory (`241w`/`241x`)
  where the kr.4.1 Schema Card worker is active, and the epic lander already holds
  Bryan's instruction to improve that surface. Recorded on the epic bead; not touched
  here.
- No changes to `bob albums list -f json`, the Schema Card (`241w`–`241z`, `242a/b`), or
  any bob-cli code.
- No vault writes beyond what `bob plugins sync` deploys.

## Coordination

- The bob-cli-kr epic is in progress: kr.4 (editable Schema Card) has an active worker
  in `241w`/`241x`/`241y` territory, and the bob-cli-k3 landing tale is changing
  `bob albums list -f json` (additive `singular`) and the `[!records]` tally. This tale
  touches only `241u`, `241v`, their tests, the e2e scripts, and `docs/albums.md` —
  disjoint files — but rebase before landing if the epic lands first, and preserve
  `albumSchemaHeaderAppend()`'s call contract from `241v` untouched.
- `bob-ledger-tools` uses the committed source build: edit `src/` fragments only, keep
  each fragment at or below 1000 lines, run `npm run build`, never hand-edit `main.js`.

## Verification gauntlet

From the bob-plugins repo root: `npm run build`, `npm run build:check`, `npm test`,
`npm run docs:check`, then `bob plugins sync` (follow the AGENTS.md refusal protocol —
never `-F` on your own). Confirm in the vault that `~/bob/sase_blog_zettel.md` and
`~/bob/podcasts.md` show one header each with no ancestry warning, in both Reading view
and Live Preview, and that the `[!records]` callout contains no header chrome.
