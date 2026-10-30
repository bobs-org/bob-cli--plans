---
tier: tale
title: Hide _conflicts copies from Bob note pickers
goal:
  Vault-sync conflict copies under _conflicts/ never appear in Obsidian's project/area
  note pickers.
size: small
decisions:
  subtree:
    ask:
      Should the sub-project map also skip _conflicts/ copies, so ^prj cascades never
      write into them?
    choices:
      skip: Filter buildProjectChildrenMap too; plugin cascade matches bob projects sync
      leave:
        Fix only the pickers; a sub-project's conflict copy can still get cascade writes
    default: skip
    why: bob-cli's project scan already skips _conflicts/; the plugin should agree
    answer: skip
  generated:
    ask:
      Should Open child note also hide _generated/ notes, like the two dev_links query
      pages?
    choices:
      hide: Hide _generated/ children too; only hand-authored child notes are listed
      keep: Keep listing them; only _conflicts/ copies disappear
    default: keep
    why: Only _conflicts/ was requested; generated query pages can be handy children
    answer: hide
proposed_by: bbugyi200.athena.16b
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 10:50:41
status: wip
---

# Hide `_conflicts/` copies from Bob's project/area note pickers

## Problem

`bob vault-sync` keeps the remote side of a conflicted note in place and saves the local
side as a full copy under `_conflicts/` (for example
`_conflicts/sase.kellys-mbp-2026-10-23T224729-0400.md`, a copy of `sase.md`). The copy
keeps the original frontmatter (`parent: "[[dev]]"`, `type: "[[project]]"`,
`status: wip`), so Obsidian pickers treat it as a real wip project under `dev`. In the
reported screenshot, **Open child note** on `dev` lists it first with a WIP badge and
counts it in "23 notes under dev · 4 projects · 4 wip".

## Where it leaks (verified)

Everything that leaks is in the `bob-navigation-hotkeys` plugin of the linked
`bob-plugins` repo:

1. **Open child note**: `collectChildNotes(parentFile)` in
   `plugins/bob-navigation-hotkeys/src/640-plugin-notes-and-task-move.js` keeps every
   Markdown file whose `parent:` resolves to the active note. It has no path filter.
2. **Ctrl+Shift+M "Move tasks to note"**:
   `collectTaskMoveDestinations(files, sourcePath, getNoteInfo)` in
   `src/240-cancel-and-lanes.js` keeps every area and open project, skipping only
   `TASK_MOVE_TEMPLATE_PATHS`. That one function also feeds the **inbox-route "where
   does it go" picker** (`src/655-plugin-inbox-route.js`) and block-id-prompt's Pomodoro
   inbox route, which calls nav's shared inbox-route core. One fix covers all three.
3. **Sub-project map** (see decision `subtree`):
   `buildProjectChildrenMap(files, frontmatterOf)` in `src/217-project-subprojects.js`
   treats a conflict copy of a sub-project as a real sub-project. The `^prj` cascade
   (`src/218-plugin-project-subtree.js`, `src/219-plugin-project-subtree-writers.js`)
   and dependency-stage descendants (`src/590-plugin-dependency-stage.js`) inherit that.
   This list is not shown to the user directly, but the cascade can write
   `#prj_is_blocked` into a `_conflicts/` file.

Already excluded, so no change: **Change parent note** (Ctrl+Shift+-) uses
`isConflictsNotePath` in `src/208-parent-note-picker-model.js`. The dependency and
relation stages, task-status-cycler, and bob-ledger-tools have their own `_conflicts`
filters.

**bob-mac-capture is already clean.** Its note prompts (route `@`, File-under,
project-note `@route^`, inbox-route card) all come from `bob capture-targets`, which
reads only top-level vault notes. Wikilink `[[` completion comes from
`bob capture-complete`, which skips bob-cli's `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES`
(includes `_conflicts`, see `src/native.rs`). Against the live vault,
`bob capture-targets -f json`, and `bob capture-complete` for `@sase`, `[[sase.kel`,
`+sase`, `!`, `:`, `/`, and `((sase`, return no `_conflicts/` path. The Mac app needs no
Swift change. The bob-cli regression test below keeps it that way.

## Changes

### bob-plugins (`bob-navigation-hotkeys`). Open the linked repo with `/sase_repo`.

Edit `src/` fragments only, never `main.js`. Keep each hand-edited fragment at or below
1000 lines. `src/700-exports.js` is already at exactly 1000 lines, so put any new export
in `src/701-exports-parent-change.js` (`Object.assign(module.exports.helpers, {...})`).

1. **One shared predicate.** Move `isConflictsNotePath(path)` from
   `src/208-parent-note-picker-model.js` into the shared path-helper fragment
   `src/100-yank-and-paths.js`, with a one-line comment. Fragments share one script
   scope, so the parent picker keeps working unchanged. Keep its semantics: an exact
   `_conflicts` path segment, as vault-sync writes it and as bob-cli matches it. Export
   it from `701-exports-parent-change.js` so tests can call it directly.
2. **Open child note.** In `collectChildNotes`, drop any file where
   `isConflictsNotePath(file.path)` is true, before the `parent:` check. The
   single-child auto-open, the empty "No child notes found" notice, and the modal's
   subtitle counts (`getChildNoteSummary`) then all ignore conflict copies.
   > [!decision] generated = hide Also drop paths with a `_generated` segment here, and
   > here only. Add a small sibling predicate next to `isConflictsNotePath`, or a
   > combined `isChildNotePickerExcludedPath`. Leave all other pickers unchanged.
3. **Move / inbox-route destinations.** In `collectTaskMoveDestinations`, add
   `!isConflictsNotePath(path)` to the existing path filter, beside the
   `TASK_MOVE_TEMPLATE_PATHS` check. Do not change `validateTaskMoveDestination`: a
   conflict copy can no longer be offered, so the commit-time validator needs no new
   refusal.
4. **Sub-project map** (decision `subtree`).

   > [!decision] subtree = skip In `buildProjectChildrenMap`, skip conflict paths in the
   > first loop so they enter neither `records` nor `notePaths`. A conflict copy is then
   > never a child and never resolves as a parent. Update the header comment's parity
   > note to name the `_conflicts` exclusion.

   Under `subtree = leave`, skip this step and its test.

5. **Tests.** Add to existing files, which `npm test` already lists, so `package.json`
   needs no change:
   - `scripts/test-navigation-hotkeys-task-move-plan.cjs`: add a
     `_conflicts/Wip.kellys-mbp-2026-10-23T224729-0400.md` wip project to the
     `collectTaskMoveDestinations` fixture and assert it is absent.
   - `scripts/test-navigation-inbox-route.cjs`: assert the inbox-route destination list
     built from the same files omits the conflict copy.
   - A child-note test, in `scripts/test-navigation-parent-change.cjs` or a nearby file
     that already builds `new NavigationHotkeysPlugin()` with a fake `app`. Fake
     `vault.getMarkdownFiles()` and `metadataCache` (`getFileCache` with
     `parent: "[[dev]]"`, plus whatever `resolveLinkTargetFile` needs) for `dev.md`,
     `dev_tools.md`, and `_conflicts/sase.kellys-mbp-….md`. Assert that
     `plugin.collectChildNotes(devFile)` returns only `dev_tools.md`. Under
     `generated = hide`, add a `_generated/queries/query/links/dev_links.md` child and
     assert it is absent too.
   - `scripts/test-navigation-project-subprojects.cjs` (subtree = skip): a conflict copy
     with `parent: "[[sase]]"` and `type: "[[project]]"` is absent from the children
     map.
   - Unit-test `isConflictsNotePath`: `_conflicts/x.md` and `a/_conflicts/x.md` are
     true; `_conflicts.md`, `my_conflicts/x.md`, and `x.md` are false.
6. **Docs.** In `docs/bob-navigation-hotkeys.md`, add one sentence near the
   `Ctrl+Shift+M` paragraph: the move and inbox-route destination lists, and Open child
   note, never offer `bob vault-sync` conflict copies under `_conflicts/`. Add the
   `_generated` clause only under `generated = hide`. Do not change the manifest
   description, so the docs inventory stays unchanged.
7. **Build and deploy.** Run `npm run build` and `npm run build:check`, then the touched
   test files with `node --test …`, then the full `npm test`, `npm run validate`, and
   `npm run docs:check`. Finally, run `bob plugins sync` as bob-plugins' AGENTS.md
   requires.

### bob-cli (this repo): regression guard for the Mac app

8. Extend `conflict_directory_is_skipped_by_vault_walkers` in `tests/cli/vault_sync.rs`,
   or add a sibling test in `tests/cli/capture/targets.rs`. Write a top-level `area.md`
   (`type: [[area]]`) and a `_conflicts/proj.kellys-mbp-…md` wip project with a `^prj`
   task. Assert that:
   - `bob capture-targets -f json` lists no `_conflicts/` path.
   - `bob capture-complete -f json -c <n> -- '[[proj'` returns no `_conflicts/`
     candidate. Use `BOB_DIR`/`-b` and a temporary `HOME`, like the existing test does.
     Run `cargo test --test cli` on the touched test, then `just check`. bob-cli product
     code and docs need no change: `docs/capture.md` already documents the `_conflicts`
     exclusion for completions.

## Out of scope

- bob-mac-capture Swift code: no change needed (see above).
- Obsidian core surfaces (Quick Switcher, search, graph). Hiding conflict copies there
  needs the vault's `Excluded files` setting (`userIgnoreFilters` in
  `.obsidian/app.json`), which Bryan configures by hand. Even then, Obsidian only
  demotes the copies.
- Resolving or deleting the existing conflict copy. That stays Bryan's call, through
  `_conflicts/sync_conflicts.md`.
- No SASE memory note or decision record changes. Nothing on file becomes false.
