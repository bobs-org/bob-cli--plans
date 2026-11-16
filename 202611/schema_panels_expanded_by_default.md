---
tier: tale
title: Expand album schema summaries and the bank Setup header by default
goal:
  Album notes and bank.md open with their schema/Setup panels already expanded; the
  toggle still collapses them per session.
size: small
proposed_by: bbugyi200.athena.1d5
proposed_with: claude/opus
create_time: 2026-11-16 16:09:11
status: wip
---

# Plan: Expand album schema summaries and the bank Setup header by default

## Goal

Bryan's screencast (album note `sase_blog_zettel`) shows the album header's schema
summary (`▸ Zettel · 3 fields · dated notes`), which he must click on every visit to see
the field chip rows (Status, Claim, Section, ...). The `▸ Setup` header on the bank hub
`bank.md` behaves the same way. Make both panels start **expanded**. The existing toggle
still collapses them for the rest of the session.

## Where the work lives

Every change is in the linked `bob-plugins` repo, plugin `bob-ledger-tools`. Open the
repo with `/sase_repo` (`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md`
first. The plugin uses the committed source build: edit `plugins/bob-ledger-tools/src/`
fragments, then run `npm run build`. Never hand-edit `main.js`. Each hand-edited
fragment must stay at or below 1000 lines.

bob-cli itself needs no change: its `docs/reference/plugins/` is an untracked assembly
mount of the bob-plugins `docs/` tree. No SASE memory note or decision record states
these defaults, so memory stays unchanged.

## Root cause

- **Album.** `albumHeaderIsExpanded(path, model)` in `src/241v-plugin-album-header.js`
  reads the per-path in-memory `albumHeaderExpanded` Map. When that has no entry, it
  falls back to `albumHeaderDefaultExpanded(model)` in
  `src/241w-plugin-album-schema-model.js`, which returns
  `Boolean(model && model.schemaInvalid)`. So a healthy schema paints its chip card with
  `hidden`, sets `aria-expanded="false"` on the summary, and shows `▸`.
- **Bank.** `bankSetupModelFor(sourcePath)` in `src/248c-plugin-bank-setup-hosts.js`
  passes `expanded: key && expandedMap.has(key) ? expandedMap.get(key) === true : false`
  into the pure `bankSetupModel` (`src/248a-plugin-bank-setup-model.js`). That function
  forces `true` only when `state === "invalid"`. `bankSetupToggleExpanded` flips
  relative to the current model's `expanded` value, so it works with any default.

## Changes

1. **Album default.** Make `albumHeaderDefaultExpanded` return `true` for every painted
   header. This subsumes the invalid-schema case. Keep the function and its export in
   `src/352-exports-album.js`. Update the header comment of
   `src/241x-plugin-album-schema-dom.js` ("Collapsed schema summary, expandable chip
   view") so it says the summary starts expanded.
   - Reading view and Live Preview both reach the default through `albumHeaderOptions` →
     `albumHeaderIsExpanded`, so both pick up the change.
   - The CodeMirror widget signature already includes `expanded`.
   - Embeds and hover previews of an album note also start expanded, because state is
     keyed by album path. This is accepted.
2. **Bank default.** Change the fallback in `bankSetupModelFor` from `false` to `true`.
   Keep the ternary grouping explicit:
   `(key && expandedMap.has(key)) ? expandedMap.get(key) === true : true`.
   - Put the default in one obvious place, for example
     `const BANK_SETUP_DEFAULT_EXPANDED = true;` next to `BANK_SETUP_HUB_PATH` in
     `248a`, and use it in `248c`.
   - Do **not** change the pure `bankSetupModel` default (`options.expanded === true`).
     The pure-model tests and the Bank setup card (`248i`, which already passes
     `expanded: true`) rely on explicit options.
   - Invalid configs keep their forced expansion.
   - The `missing` state now shows its "Copy starter" body by default. This is intended.
   - The mobile, loading, unsupported, and refresh-failed states never paint a body, so
     they do not change.
3. **No persistence change.** A collapse choice remains session view state in the
   in-memory Maps and resets on plugin reload. This keeps the documented rule "Collapse
   state lives in view state keyed by the album, never in frontmatter." Painting still
   never writes vault bytes.

## Tests

- `scripts/test-ledger-tools-album-header.cjs`
  - Rewrite "schema summary stays collapsed by default and expands per album in view
    state" as "schema summary starts expanded and collapses per album in view state".
    - Before any click: `aria-expanded="true"`, the summary text starts with `▾`, and
      `.bob-album-schema-card` has no `hidden` attribute.
    - After a click: `plugin.albumHeaderIsExpanded("eat.md")` is `false`, the summary
      has `aria-expanded="false"`, and the card has `hidden=""`.
    - A second pane painted for `eat.md` is also collapsed.
  - In "theme accent, schema button, and keyboard labels stay on the header", expect
    `aria-expanded="true"`.
  - Assert that the exported `albumHeaderDefaultExpanded` returns `true` for both a
    healthy model and an empty-schema model.
  - Keep the broken-schema test's `"true"` expectation unchanged.
- `scripts/test-ledger-tools-bank-setup.cjs`: add a host-level test using the existing
  `makeSetupPlugin`, `seedReport`, and `hubSurface` helpers. Check this sequence:
  1. The first `paintBankSetupReading(el, "bank.md")` paints a `.bob-bank-setup-toggle`
     with `aria-expanded="true"` and text `▾ Setup`, plus a `.bob-bank-setup-body`
     containing category tiles.
  2. Call `plugin.bankSetupToggleExpanded("bank.md")` and repaint: the toggle has
     `aria-expanded="false"` and there is no body.
  3. Toggle again and repaint: the panel is expanded again.

  Leave the existing pure-model tests as they are.

- **Real-Obsidian bank e2e** (`scripts/obsidian-bank-ui-e2e-setup.cjs` and
  `scripts/obsidian-bank-ui-e2e-lib.cjs`). The live flow currently asserts
  `live-header-collapsed` and then clicks the toggle expecting it to expand. Rework it:
  - Replace `assertLiveCollapsed` with a default-state helper, for example
    `assertLiveDefault`. Update its export and the `--self-test` call sites (around the
    `collapsed helper failed` check).
    - The helper requires `expanded === "true"`.
    - It still checks the summary-line grammar (`🏦`, `categor`, `no rules`). The
      `.bob-bank-setup-collapsed` span paints in both states.
  - Record the default check as `live-header-expanded-default`.
  - While the panel is expanded, run the tile checks (`live-tiles-present`,
    `live-tile-aria`, `live-hotkey-none-dot`, `live-derived-keycap`) and take the
    expanded screenshots.
  - Then click the toggle, wait for `expanded === "false"`, record
    `live-header-collapses`, and take the collapsed screenshots.
  - Click again to restore expansion before any later step that needs the body.
  - Keep case names stable wherever their meaning is unchanged.
  - Leave the missing-mapping block alone; it already clicks only when not expanded.
  - The offline `--self-test` must pass. `npm test` runs it through
    `scripts/test-obsidian-bank-ui-e2e-setup.cjs`.
- **Album e2e** (`scripts/obsidian-album-ui-e2e.cjs`) already clicks the summary only
  when `aria-expanded !== "true"`. Confirm that no other album e2e step assumes the
  collapsed state, and change only what does.

## Docs (bob-plugins `docs/`)

- `docs/albums.md`
  - In the frontmatter `description`, change "a collapsed schema header" to "an expanded
    schema header".
  - In the body, change "a collapsed-by-default schema summary" to "an
    expanded-by-default schema summary".
  - Change "Expanding it shows one chip row per field ..." so it says the chip rows show
    by default and the summary's `▾` toggle collapses them for the session.
  - Keep the sentence "Collapse state lives in view state keyed by the album, never in
    frontmatter."
  - In the e2e walkthrough sentence, change "the collapsed header" to "the schema
    header".
- `docs/bob-ledger-tools.md`, in the bank Setup paragraph: change "It starts collapsed,
  auto-expands on errors" to say it starts expanded, `▾ Setup` collapses it for the
  session, and errors keep it expanded.
- Run `npm run docs:gen` if the inventory picks up page descriptions, then run
  `npm run docs:check`.

## Version, build, verify, deploy

1. Bump the minor version in `plugins/bob-ledger-tools/manifest.json` to match recent
   practice: `1.120.0` → `1.121.0`, or the next minor above whatever master holds at
   implementation time.
2. Run `npm run build`, then `npm run build:check`.
3. Run the targeted tests:
   `node --test scripts/test-ledger-tools-album-header.cjs scripts/test-ledger-tools-bank-setup.cjs scripts/test-obsidian-bank-ui-e2e-setup.cjs scripts/test-obsidian-album-ui-e2e.cjs`.
4. Run the full `npm test` and `npm run docs:check`. Report any failure verbatim.
5. If the host has an Obsidian executable, run the real bank Setup e2e
   (`node scripts/obsidian-bank-ui-e2e-setup.cjs`; see `docs/develop/`) and report its
   per-case verdicts. Otherwise, state that it was not run.
6. Run `bob plugins sync` to deploy to the vault. If sync refuses an overwrite, follow
   the bob-plugins `AGENTS.md` rules and never pass `-F` on your own.

## Acceptance

- Opening any album note (for example `sase_blog_zettel.md`) in Reading view or Live
  Preview shows `▾ <Label> · N fields ...` with its field chip rows visible, with no
  click needed. Clicking the summary collapses it, and that album stays collapsed until
  the plugin reloads.
- Opening `bank.md` shows `▾ Setup` with its accounts, category tiles, and rules
  visible. Clicking the toggle collapses it.
- Invalid album schemas and invalid bank configs still paint expanded, with their
  diagnostics.
- Painting writes no vault bytes, and all existing tests still pass.
