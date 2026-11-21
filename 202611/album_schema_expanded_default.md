---
tier: tale
title: Album card schema starts expanded again
goal:
  Every album card's schema chip rows show expanded the first time an album note is
  opened in an Obsidian session, while the summary toggle still collapses them for the
  rest of that session.
size: small
proposed_by: bbugyi200.apollo.fg
proposed_with: claude/opus
create_time: 2026-11-21 15:15:19
status: wip
---

# Plan: Album card schema starts expanded again

## Goal

When Obsidian starts, or when an album note is opened for the first time in a session,
the album card's schema section shows its field chip rows already expanded. The
`Schema · N fields · …` summary line shows `▾`, `aria-expanded="true"`, and the chip
card is visible. Clicking the summary still collapses the section, and the section stays
collapsed for that album for the rest of the Obsidian session.

Bryan's screencast (`~/tmp/screencasts/20261121_135709.mp4`) shows the card on the
`sase_blog_zettel` album note. The summary reads
`▾ Schema · 3 fields · dated notes · 14 zettels · walks by Section` with the Status,
Claim, and Section chip rows below it, and one click folds it. Today a healthy schema
always starts folded, so the user has to click to open it in every new session.

## Root cause (verified at bob-plugins `f292fffa`, bob-ledger-tools 1.155.0)

This is a regression, and the default lives in one function:

- `plugins/bob-ledger-tools/src/241w-plugin-album-schema-model.js:113`:
  `albumHeaderDefaultExpanded(model)` returns
  `Boolean(model?.schemaInvalid || model?.emptySchema)`. That means healthy schemas
  start folded, and only invalid or empty schemas start expanded.
- `src/241v-plugin-album-header.js:105` `albumHeaderIsExpanded(path, model)` uses the
  per-session `plugin.albumHeaderExpanded` Map (keyed by album path) when it has an
  entry. Otherwise it falls back to `albumHeaderDefaultExpanded(model)`.
  `albumHeaderSetExpanded` writes the Map when the summary is clicked. `onDisposeEnd`
  clears the Map, so a plugin unload, an Obsidian restart, or `bob plugins sync` resets
  every album to the default.
- History:
  - Commit `67073ac5` ("expand album schema and bank Setup headers by default",
    ledger-tools 1.121.0) set the default to `true` for every schema. Its commit message
    says: "The existing toggle still collapses them for the rest of the session."
  - Commit `a68e29be` ("album card owns the title block from the title slot",
    ledger-tools 1.147.0) switched healthy schemas back to folded. That plan was
    `plan:202611/album_card_title_block.md`, following the design contract
    `plan:202611/property_card_title_block.md` ("Album body: The schema chip rows fold
    to one line by default").
  - Bryan's request overrides that design default. The one-line summary text, the
    walks-by pill, and the toggle stay exactly as they are; only the initial state
    changes.
- The bank Setup panel's `BANK_SETUP_DEFAULT_EXPANDED = true`
  (`src/248a-plugin-bank-setup-model.js:10`) was not regressed. Leave it alone.

## Behavior after the fix

- With no session toggle recorded, every album schema starts expanded: healthy, invalid,
  or empty.
- A healthy schema keeps today's summary text: `albumHeaderFoldText` plus the
  `· walks by <Label>` pill when `walk_by` is set. It now leads with `▾` and has
  `data-schema-fold="open"`. After a click it leads with `▸` and has
  `data-schema-fold="folded"`, and the chip card gets `hidden`.
- Invalid and empty schemas are unchanged: they still start expanded and keep their
  diagnostic and teaching text.
- Session memory is unchanged. A collapse is keyed by album path and applies in every
  pane. It lasts until the plugin unloads (Obsidian restart, plugin disable/reload, or
  `bob plugins sync`), and after that the album starts expanded again. No state is
  written to frontmatter, the vault, or plugin data.
- No CSS rule changes are needed. `.bob-album-schema-summary[data-schema-fold]` keeps
  its muted look in both states, and `.bob-album-schema-card[hidden]` still hides the
  folded state.

## Implementation (all in the linked bob-plugins repo)

Open the repo with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and read its
`AGENTS.md` before editing. Paths below are relative to that checkout's root. Recheck
each line number against the current tree before relying on it.
`plugins/bob-ledger-tools` uses the committed source build. Edit `src/` fragments, then
run `npm run build`, and never hand-edit `main.js`.

1. **Default.** In `plugins/bob-ledger-tools/src/241w-plugin-album-schema-model.js`,
   make `albumHeaderDefaultExpanded(model)` return `true` for every model. Keep the
   function and its `model` parameter, because it is exported from
   `src/352-exports-album.js` and called from `241v`. Replace the comment above it
   ("Healthy schemas start folded to one line; …") with one that says every schema
   starts expanded and the summary toggle collapses it for the session.
2. **Comments.** Update the header comment of
   `plugins/bob-ledger-tools/src/241x-plugin-album-schema-dom.js` (lines 1–4) so it says
   the summary starts expanded and the collapse is a session view-state flag. Also
   update the CSS comment above `.bob-album-schema-summary[data-schema-fold]` in
   `plugins/bob-ledger-tools/styles.css` (about line 9039) so it describes the summary
   line without saying the schema "folds … by default". Do not change any CSS rules.
3. **Unit tests** in `scripts/test-ledger-tools-album-header.cjs`:
   - The test "healthy schema summary starts folded to one line and expands per album in
     view state" (about line 499): rename it to say the summary starts expanded and
     collapses per album. On first paint, assert `aria-expanded="true"`, the exact text
     `"▾ Schema · 2 fields · grouped by Status, sub-grouped by Location · dated notes · 104 restaurants"`,
     `data-schema-fold="open"`, a chip card with no `hidden` attribute, and 2 field
     rows. Click the summary, then assert that `plugin.albumHeaderIsExpanded("eat.md")`
     is `false`, `aria-expanded="false"`, the text matches `/^▸ Schema · 2 fields · /`,
     `data-schema-fold="folded"`, and the chip card has `hidden=""`. The second pane's
     paint must report `"false"`, because the toggle is per album, not per pane. Flip
     the `albumHeaderDefaultExpanded` helper assertion for a healthy model to `true`.
     Keep the empty and invalid ones `true`.
   - The walk_by test "a walk_by album's folded schema line ends with …" (about line
     542): reword "folded" in the name. Expect
     `"▾ Schema · 2 fields · grouped by Status · 12 restaurants · walks by Section"`
     first. After the click, expect `albumHeaderIsExpanded("eat.md") === false` and
     `/^▸ Schema · .* · walks by Section$/`.
   - The test "theme accent, schema button, and keyboard labels stay on the header"
     (about line 786): change the expected `aria-expanded` to `"true"`.
   - The title-slot body-join test (about line 944): change
     `/^▸ Schema · 1 field · 3 restaurants$/` to
     `/^▾ Schema · 1 field · 3 restaurants$/`.
   - Add a restart regression test. Collapse a healthy album through the summary click,
     then call `plugin.teardownAlbumHeader()` and `plugin.setupAlbumHeader()`, or build
     a fresh fixture if teardown and setup don't fit the fixture. Repaint and assert
     that the schema starts expanded again (`aria-expanded="true"`, `▾`, chips visible).
   - Calls to `plugin.albumHeaderSetExpanded("eat.md", true)` that existed only to open
     a folded schema can stay; they are harmless.
   - Then grep `scripts/` for other album-schema assumptions of the folded default
     (`▸ Schema`, `aria-expanded`, `data-schema-fold`, `.bob-album-schema-card` +
     `hidden`) and fix any that `npm test` reports. Do not touch the record-index group
     folds (`data-bob-records-folded`), the record card, or the bank Setup tests.
4. **Real-Obsidian walkthrough** in `scripts/obsidian-album-ui-e2e.cjs` (about lines
   1247–1273):
   - Reword the comment to "a healthy schema starts expanded with its chips shown".
   - Rename the `schema-fold-default` check to `schema-expanded-default`. Require
     `summaryExpanded === "true"`, `/^▾ Schema · /`, the trailing
     `/ · \d+ restaurants?$/`, and `chipsShown === true`.
   - Update the matching name in the list in `scripts/test-obsidian-album-ui-e2e.cjs`
     (about line 131).
   - The child-warnings flow (about line 1416) already clicks the summary only when it
     is not expanded, so it needs no change. Confirm that no later step in the
     walkthrough clicks the summary expecting it to open.
   - The preview scripts (`scripts/preview-property-cards.cjs`,
     `scripts/preview-title-block-matrix.cjs`, `scripts/preview-title-block.cjs`) pass
     `expanded` explicitly or use static strings. Leave them unchanged.
5. **Docs** in `docs/albums.md`:
   - In the album card paragraph (about lines 86–103), say the schema summary line (for
     example
     `▾ Schema · 5 fields · grouped by Location · dated notes · 104 restaurants`) starts
     expanded with the chip rows shown. Say that the `▾` toggle collapses them to that
     one line for the session, keyed by album and never written to frontmatter, and that
     a new Obsidian session (or plugin reload) starts expanded again.
   - Keep the walks-by, invalid-schema, and empty-schema sentences accurate.
   - In the walkthrough-coverage sentence (about line 273), change "the folded schema
     line" to "the expanded-by-default schema line".
   - Leave the frontmatter `description` as it is unless it mentions folding.
6. **Version.** Bump `plugins/bob-ledger-tools/manifest.json` by one patch version
   (1.155.0 → 1.155.1 at planning time; bump from whatever is current). Then run
   `npm run docs:gen` so the generated inventory rows in `docs/index.md` and
   `docs/bob-ledger-tools.md` match.
7. **Build and verify** from the bob-plugins root: `npm run build`,
   `npm run build:check`, `npm test`, `npm run validate`, and `npm run docs:check`. All
   must pass. If this host has Obsidian (for example `/snap/bin/obsidian`), a verified
   Bob binary, and `xvfb-run`, also run the real walkthrough as documented in
   `docs/albums.md`:
   `BOB_BIN=<verified bob> xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs -e /snap/bin/obsidian -r /tmp/album-ui.json -o /tmp/album-ui-shots`.
   Confirm that `schema-expanded-default` passes and look at
   `album-records-reading.png`. If those prerequisites are missing, say in the final
   report that the real-Obsidian run was skipped and why.
8. **Deploy.** Run `bob plugins sync`. If it refuses to overwrite another checkout's
   deploy, follow the bob-plugins `AGENTS.md` procedure exactly: never pass `-F` on your
   own.

Suggested commit subject:
`fix(ledger-tools): start the album card schema expanded again`.

## Non-goals

- Persisting a collapse across Obsidian restarts. The user asked for expanded on first
  open, and the session-only Map already gives that.
- Changing the summary text, the walks-by pill, the `⊞ Schema` editor, or any CSS rule.
- The record-index group folds, the record card fold, and the bank Setup panel (already
  expanded by default).
- Any bob-cli change. bob-cli's `docs/reference/plugins/` is an untracked mount
  assembled from bob-plugins `docs/`.
- Memory changes. The glossary's Album Card entry does not describe the schema fold
  default, and no decision record governs it.

## Acceptance criteria

- After an Obsidian restart, or right after `bob plugins sync` reloads the plugin,
  opening `sase_blog_zettel` (or any healthy album) shows the schema chip rows under a
  `▾ Schema · …` summary without a click.
- Clicking the summary collapses it to `▸ Schema · …`. The album stays collapsed when
  reopened in the same session, in any pane. It is expanded again in the next session.
- Invalid and empty schemas behave as before. No note bytes change.
- `npm run build:check`, `npm test`, `npm run validate`, and `npm run docs:check` pass.
  The bob-ledger-tools patch version is bumped, the inventory is regenerated, and the
  vault deploy is synced.
