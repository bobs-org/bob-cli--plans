---
tier: tale
size: small
title: Finish and land the card foot drawer epic
goal: Drawer tag chips search their tag, aliases stay plain chips, =- heals in any
  pane, docs match the shipped =- entry points and foot kit API, and epic bob-cli-pf.1.1
  plus its parent phase bob-cli-pf.1 are verified and closed.
proposed_by: bbugyi200.apollo.bob-cli-pf.1.1.land
proposed_with: claude/opus
bead: bob-cli-pf.1.1
status: done
---

- **PARENT:**
  [202611/card_foot_drawer.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/card_foot_drawer.md)
- **BEAD:**
  [bob-cli-pf.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/bob-cli-pf.1.1.md)

# Finish and land the card foot drawer epic (bob-cli-pf.1.1)

This tale finishes the last work on epic `bob-cli-pf.1.1` ("Card foot row, properties
drawer, and equals-minus entry points") and then lands it. That includes closing the
epic, closing its parent phase `bob-cli-pf.1`, and marking the plan done. There is no
other land agent: **you** do the closeout in the same turn as the code.

All code and docs work is in the linked **bob-plugins** repository. Open it with
`/sase_repo` (`sase repo open bob-plugins -r "<why>"`), use only the path it prints, and
read its `AGENTS.md`. Paths below are relative to bob-plugins unless they say bob-cli.
Edit fragments under `plugins/*/src/` and regenerate `main.js` only with
`npm run build`; never hand-edit `main.js`. Keep each fragment at or below 1000 lines.
Register any new test file in `package.json`'s explicit `test:unit` list.

## What the lander already verified (do not redo)

- All five phases (`bob-cli-pf.1.1.1`–`.5`) are closed. Their commits are on bob-plugins
  master: `538d4213`, `15d82db7`, `b3188ab5`, `175283d3`, `d3827858`, and `c8deb534`.
  Source review confirmed:
  - the created formatter (`240m`) and source-ordered inventory (`240n`);
  - the foot DOM (`240o`), per-note session state and toggle runtime (`240p`), and
    shared adapter (`240r`);
  - the Record Card migration (`242kc`; `recordSheetFolds` and the old footer are gone);
  - album, intention, and bank feet (`241va`, `296da`, `248ca`);
  - the `=` second-key arm table with the `-` arm, a registration-only `+`, the palette
    command `toggle-card-properties`, and the Command Mode row (`248m`,
    `bob-command-mode/src/160-sheet-rank.js`);
  - CSS scoped under `.bob-property-card`;
  - the real-Obsidian runner `scripts/obsidian-card-foot-e2e.cjs` with its offline
    self-test. Its report (116/116 checks) and 27-screenshot matrix are retained as
    `file:explicit:dcd27f31fa7fb913287fdb6a` and
    `file:explicit:c7135143af9a4a7d2df51467`.
- On master, `npm run build:check` passes, and the focused epic suites pass 154/154.
  Those suites are the property-card created, inventory, foot, and foot-adapters tests,
  plus record-sheet, schema-double-equals, command-mode-record-surface, the card-foot
  e2e self-test, and the intention title block.
- Integration: none of the commits since the epic started touch the foot. They are the
  task-requirement close card work, copy-card, and bob-cli `bob props`. bob-cli
  `src/native/props/roles.rs` adds `review_*` to its derived set, but the plan says the
  drawer's Managed grouping is display-only and is not the writer role table, so leave
  it alone.
- `sase bead epic-symbols bob-cli-pf.1.1` and `sase bead epic-symbols bob-cli-pf.1` list
  no entries.
- Phase follow-ups were triaged, with the outcomes recorded in a note on
  `bob-cli-pf.1.1`:
  - the timing flakes are tracked as `bob-cli-km`, `bob-cli-io`, and `bob-cli-pg`;
  - the period-notes clock failure is new task `bob-cli-pm`;
  - the docs mount refusal is tracked as `bob-cli-it`.

## Remaining defects to fix (caused by this epic)

### 1. Drawer tag chips open search without the tag

`propertyCardFootAdapterSearchTag` (`240r-plugin-property-card-foot-adapter.js`) and
`recordSheetSearchTag` (`242kc-plugin-record-sheet-foot.js`) are copies of each other.
Both only run the `global-search:open` command and drop the tag, so a tag chip opens an
empty search. The plan says tag chips must open a search for that tag.

- Replace the two bodies with one shared kit helper in `240r`, for example
  `propertyCardFootOpenTagSearch(app, tag)`, and make `recordSheetSearchTag` delegate to
  it.
- Build the query as `tag:#<name>`, stripping any leading `#` from the value.
- Open it through
  `app.internalPlugins.getPluginById("global-search")?.instance?.openGlobalSearch(query)`.
- If that API is missing, fall back to the current `global-search:open` command. Return
  whether anything opened. The helper must never throw or write to the vault.

### 2. `aliases` values render as tag chips

`PROPERTY_CARD_FOOT_TAG_KEYS_RE` in `240o-plugin-property-card-foot-dom.js` is
`/^(tags?|aliases?)$/i`. That makes every `aliases` item a "Search …" tag button, but
aliases are note names, not tags.

- Restrict the key match to `tags?`, so alias items render as ordinary list chips.
- Keep detecting literal `#tag` text as a tag.

### 3. Two-pane mount-heal gap in the `=-` toggle

`propertyCardFootEnsureViewMount(view, path)` in `240p` returns `true` early whenever
_any_ live mount exists for the path. When a second pane on the same note still has a
registered mount, the heal never scans this view. The caller then finds no mount in this
view, so `=-` there falls through to Vim's `=-` reindent operator.

- Change the early return so it only short-circuits when a live mount for the path is
  inside this view (`propertyCardFootViewContains`).
- Otherwise scan this view's DOM as it does today.
- Keep every existing guard: connected node, non-zero rect, and a matching
  `data-foot-source`.

### 4. Stale and missing docs

- These pages still say "the future `=-` key" although `=-` shipped in phase 4:
  - `docs/intention-card.md` (around line 98);
  - `docs/record-card.md` (around line 141);
  - `docs/albums.md` (around line 115).

  Reword each to name the real entry points: the disclosure, `=-` in Live Preview Vim
  normal mode, and the **Toggle card properties** palette or Command Mode command.

- In the `docs/record-card.md` "Entry points" table, add two rows next to the Properties
  disclosure row:
  - `=-` in Vim normal mode in Live Preview toggles the drawer. Counts, registers,
    Source mode, insert and visual modes, and views without a mounted foot keep Vim's
    reindent.
  - **Toggle card properties** (palette or Command Mode) toggles the drawer.
- Hotkey reference: `docs/bob-navigation-hotkeys.md` (around line 694) says Bob Ledger
  Tools owns `==` on albums, records, and `bank.md`. Add that it also owns `=-`, which
  toggles the mounted card's Properties drawer.
- `docs/develop/architecture.md` documents the `-` arm and
  `propertyCardFootSetEditorHook`, but not the foot kit that the parent's `editor-core`
  and `takeover` phases build on. Add a short "Foot kit" paragraph or subsection next to
  the arm-table text that names:
  - `propertyCardFormatCreatedStamp(...)` → `{text, tooltip, state, dateOnly}`;
  - `propertyCardInventory(...)` →
    `{state, reason, rows, groups, count, issues, recovery}`;
  - `propertyCardFootBuildEl(doc, spec)`;
  - the per-path state and listener API (`propertyCardFootIsOpen`,
    `propertyCardFootToggle`, `propertyCardFootOnToggle`) and the mount registry;
  - `propertyCardToggleProperties(view)` and `propertyCardCanToggleProperties(view)`;
  - the `240r` adapter helpers (`propertyCardFootAdapterSpec`, `…Append`, `…Track`,
    `…Bind`, `…Unbind`).

  Take the exact signatures from the source. Keep within the page's docs:check word
  budget.

## Tests

Add focused cases to the existing suites. Each must fail before its fix and pass after:

- `scripts/test-ledger-tools-property-card-foot-adapters.cjs` (or the foot suite):
  - the shared tag-search helper calls `openGlobalSearch("tag:#phase")` for both `phase`
    and `#phase`;
  - it falls back to the command when `internalPlugins` is absent;
  - the record sheet path uses the same helper.
- `scripts/test-ledger-tools-property-card-foot.cjs`:
  - an `aliases` list renders plain chips, not tag buttons;
  - `tags` still renders tag buttons;
  - two-pane heal: two fake views show the same path, and pane B's foot is registered.
    Pane A's foot is connected and visible but missing from the registry.
    `propertyCardToggleProperties(viewA)` must return `true`, and a view whose foot
    carries a different `data-foot-source` must still return `false`.

## Verify, build, and deploy

1. Run the focused suites with `node --test` on the files above. Also run record-sheet,
   schema-double-equals, and command-mode-record-surface.
2. Run `npm run build`, then `npm run build:check`, `npm run validate`, and
   `npm run docs:inventory:check`.
3. Run `npm run docs:check` with `BOB_CLI_DIR` set to your bob-cli workspace root.
   - If it refuses with "refusing to overwrite authored file …/docs/reference/plugins/…"
     or the stale-index error, that is known task `bob-cli-it`: a missing
     `target/docs/mount-manifest.json` next to stale gitignored copies.
   - Move the gitignored copies under bob-cli `docs/reference/plugins/` and
     `docs/develop/plugins/` aside to a temp directory (do not delete tracked files),
     rerun, and mention it in your close note.
4. Run `npm test`. Failures in only the known tracked nodes are not blockers:
   - `test-ledger-tools-event-link-times.cjs` (`bob-cli-km`);
   - `test-ledger-tools-frees-marks.cjs` (`bob-cli-io`);
   - the block-id-prompt taskLinks p95 test (`bob-cli-pg`);
   - `test-ledger-tools-period-notes.cjs` (`bob-cli-pm`).

   Check any other failure against the clean base before treating it as unrelated.

5. Rerun the real-Obsidian foot walkthrough (`node scripts/obsidian-card-foot-e2e.cjs`,
   following its docs in `docs/develop/validation.md`). Use `/sase_monitor` if it runs
   long.
   - It covers the two-pane toggle path you changed.
   - If it cannot launch for environmental reasons unrelated to the code, record that
     and rely on the unit tests.
6. Deploy with `bob plugins sync`. On a protected-deploy refusal, follow bob-plugins
   `AGENTS.md` and `docs/develop/deployment.md`; never pass `-F` on your own.

## Closeout (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-pf.1.1`. For each entry, resolve the symbol or
   re-key it to a still-open bead. The lander found none.
2. Close the epic with `sase bead close bob-cli-pf.1.1 --note "<verification>"`. The
   note should summarize:
   - the phase verification above;
   - the integration finding (no conflicting changes);
   - the four fixes above, with their test evidence;
   - gate results, sync output, and the e2e result;
   - the follow-up triage outcomes (`bob-cli-km`, `bob-cli-io`, `bob-cli-pg`, and
     `bob-cli-it` corroborated or noted; `bob-cli-pm` created; none declined).

   Never pass `--force` just to make the close succeed.

3. Run `just symvision` in bob-cli if that recipe exists. At planning time it did not,
   so skip it and say so.
4. Set `status: done` in the frontmatter of the epic's plan file: the EPIC PLAN path
   shown by `sase bead read bob-cli-pf.1.1`, which is `202611/card_foot_drawer.md` in
   the plans repo.
5. Close the parent. `bob-cli-pf.1` is a **phase** bead ("Card foot row, drawer, created
   stamp, and =- (bob-plugins)") of epic `bob-cli-pf`.
   - Verify this epic delivered everything that phase requires. The `foot-drawer`
     section of the parent plan `202611/card_owned_frontmatter.md` lists it: the
     formatter, inventory, foot and drawer on every adopted card, the record fold
     migration, the `=-` arm, the palette and Command Mode toggle, tests, the e2e, and
     docs.
   - Run `sase bead epic-symbols bob-cli-pf.1`, then close only that phase:
     `sase bead close bob-cli-pf.1 --note "<what you verified>"`.
   - Do **not** close `bob-cli-pf` or any other ancestor; its own land agent owns it.
6. Finish through `/sase_final` and include every changed linked repository (bob-plugins
   and the plans repo) in the declaration.
