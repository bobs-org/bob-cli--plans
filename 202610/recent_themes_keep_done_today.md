---
tier: tale
title: Keep done-today themes in Obsidian picker Recent rows
goal:
  Every enabled Pomodoro theme appears in the Link to today and Pomodoro move/rename
  pickers, even when only a done Pomodoro used it today.
size: small
decisions:
  compound_open:
    ask:
      Should a theme shown only inside an open multi-theme Today row (BUILD in PLAN +
      BUILD) also get its own Recent row?
    choices:
      keep:
        It stays visible only inside that Today row, as today; only bob-plugins changes
      show:
        Recent drops only exact single-theme Today rows; bob-cli, vectors, and docs
        change too
    default: keep
    why:
      That theme already shows in the Today row; keep fixes the bug without reopening
      shared PT7
    answer: keep
proposed_by: bbugyi200.athena.14z
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 07:06:19
status: wip
---

# Keep themes used by done Pomodoros in the Recent rows of the Obsidian pickers

## Problem

When the Link to today picker opens with an empty query, `BUILD` is missing, even though
it is an enabled (not hidden) theme (`~/tmp/screenshots/20261028_065324.png`). `BUILD`
was the name of a Pomodoro that is already done (`[x]`) in today's daily note. A done
Pomodoro is not a Today row in this picker, because you can only link into open
Pomodoros. Its theme was still removed from Recent, so the theme appears nowhere in the
menu. The only way to reach it is to type the name.

The shared menu contract already says this is wrong. PT7 in `docs/pomodoro-themes.md`
says: "Recent rows exclude hidden themes and any theme that is a component of a **named
Today row in the same menu**". It covers the capture-complete contexts and the Obsidian
Pomodoro-name pickers. Two other places agree:

- `docs/capture.md`: "A theme completed today is Recent in this link context."
- The shared vector `completed-link-context` in
  `tests/fixtures/pomodoro_themes/vectors.json`. It expects a done-today `BOB` to come
  back as an `origin: "history"` row.

bob-cli's `theme_menu::compose` (`src/native/capture_complete/theme_menu.rs`) follows
this rule. Its link context builds exclusions from open entries only. Its start context
shows a done theme as an `again` row. Bob Mac Capture only renders what bob returns.
**Only the two Obsidian plugin callers of `api.themes.historyRows` are wrong.** Both
build their exclusion set from _every_ entry in today's ledger, done and cancelled ones
included:

1. **block-id-prompt** (the Link to today picker). `withPomodoroThemeHistoryRows` in
   `plugins/block-id-prompt/src/076-pomodoro-theme-picker.js` computes
   `blockPomodoroThemeKeysInToday(api, model.allEntries)`. Here `allEntries` contains
   done and cancelled entries, while its Today rows (`existing`) come only from
   `model.entries`, which holds open entries only.
2. **bob-navigation-hotkeys** (the Pomodoro bullet-move and whole-entry move/rename
   pickers). `createPomodoroBulletMovePickerRows` in
   `plugins/bob-navigation-hotkeys/src/160-pomodoro-reorder.js` passes `allEntries` to
   `withNavigationPomodoroThemeRows` (in
   `plugins/bob-navigation-hotkeys/src/165-pomodoro-theme-picker.js`). That function
   feeds `navigationPomodoroThemeKeysInToday`. The Today rows here are also only open
   entries.

The ledger-tools `pomodoroThemeHistoryRows` API
(`plugins/bob-ledger-tools/src/279-plugin-pomodoro-themes.js`) excludes only the keys
its caller passes, so the API itself needs no change.

Target invariant: **every enabled theme appears somewhere in the menu.** It appears
either as (a component of) a Today row the menu shows, or as a Recent row, subject to
the existing 5/12 Recent caps and query filtering. Hidden themes stay out of Recent, as
PT9 already says.

## Where the code lives

All plugin edits are in the linked `bob-plugins` repo. Open it with your `/sase_repo`
skill (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`.
block-id-prompt and bob-navigation-hotkeys both use the committed fragment build. Edit
only their `src/` fragments, regenerate with `npm run build`, and never hand-edit
`main.js`. Each hand-edited fragment must stay at or below 1000 lines.

## Changes

### 1. block-id-prompt: exclude only themes of the Today rows the picker shows

In `withPomodoroThemeHistoryRows` (`076-pomodoro-theme-picker.js`), build the exclusion
set from `model.entries` (today's open entries, before query filtering) instead of
`model.allEntries`. Done (`[x]`/`[X]`) and cancelled (`[-]`) entries no longer remove
their themes from Recent. Leave the rest of the composition alone:

- Exclusions still come from **all** open entries, not just the query-filtered
  `existing` rows. PT7 excludes "before filtering the query", as the vector
  `today-component-exclusion-before-query` shows.
- Exact Recent matches still fold into the create row, and hidden ghosts and caps work
  as before.
- `buildPomodoroLinkPickerRows` still uses `allEntries` for `againOf` and for the
  creation anchor. Leave both alone.

Rename the local `allToday` variable and add a short comment that states the rule ("only
themes the menu already shows as a Today row leave Recent; a done Pomodoro is not a
Today row here").

After this change, the screenshot case shows `New Pomodoro BUILD` in Recent. Its
`last_used` is today, so it sorts first.

### 2. bob-navigation-hotkeys: same rule for the move and rename pickers

In `createPomodoroBulletMovePickerRows` (`160-pomodoro-reorder.js`), pass the open
entries to `withNavigationPomodoroThemeRows` instead of `allEntries`. Those entries are
the ones the picker can show as Today rows. In `entry` (rename) mode, also pass the
source entry itself, whatever its status: a "Rename to <own name>" Recent row would do
nothing. An open source entry is already among the open entries, so no special case is
needed for it. A done source in bullets mode is not excluded: moving its bullets into a
new Pomodoro with the same name is a real action.

Keep the existing early return on `=` queries, and keep the query-independent exclusion.

### 3. Comments and docs

- In `bob-ledger-tools/src/279-plugin-pomodoro-themes.js`, change the
  `pomodoroThemeHistoryRows` comment from "today's components excluded by the caller" to
  "components of the caller's displayed Today rows excluded by the caller". Make the
  same change to the matching comment above `pomodoroThemeRankHistory` in
  `279-plugin-pomodoro-theme-mirrors.js`. These are comment-only edits; the API is
  unchanged.
- bob-plugins `docs/block-id-prompt.md` ("Linking tasks to today") and
  `docs/bob-navigation-hotkeys.md` (the paragraph "Both pickers keep today's open
  Pomodoros first …"): add one sentence each. A theme that today's ledger uses only on a
  done or cancelled Pomodoro still shows as a Recent row; only themes of the open Today
  rows (and, for rename, the entry being renamed) are left out. Then run
  `npm run docs:gen` if the inventory changes, and `npm run docs:check`.
- bob-cli `docs/pomodoro-themes.md` PT7: after the sentence "Recent rows exclude hidden
  themes and any theme that is a component of a named Today row in the same menu.", add
  one clarifying sentence. Only rows the menu actually shows count. A theme used today
  only by a done or cancelled Pomodoro is not a Today row in a link or move picker, so
  it stays in Recent. In the start context it shows as an "again" row. Before editing,
  read the docs conventions with `sase memory read docs.md -r "<why>"` (this is a
  behavior-contract clarification), and run `just docs-check`.

> [!decision] compound_open = show Do everything above. In addition, narrow the
> exclusion everywhere it is computed: a Today row excludes a theme only when the row's
> whole name is that single theme (`split_components(name).len() == 1`). This means:
> bob-cli `theme_menu::compose` (both the link and start contexts, the `exclusions`
> chain), `blockPomodoroThemeKeysInToday`, and `navigationPomodoroThemeKeysInToday`.
> Update the vector `today-component-exclusion-before-query` in
> `tests/fixtures/pomodoro_themes/vectors.json` so that `DECKS` comes back as an
> `origin: "history"` row and the separate create row disappears. Copy the file
> byte-for-byte to
> `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/vectors.json`. Rewrite the
> exclusion sentence in `docs/pomodoro-themes.md` PT7, `docs/capture.md` (around "Recent
> rows exclude hidden themes and components of any Today row"), and the
> `capture-complete` long help in `src/native/capture_complete/cli.rs` ("excludes hidden
> themes and components already represented by Today"). Then run `just docs-gen` so
> `docs/reference/cli/capture-complete.md` regenerates. Add Rust unit coverage in
> `src/native/capture_complete/tests/pomodoros.rs` or the existing theme_menu tests, and
> run `cargo test --test cli complete_themes` plus the unit tests. Commit bob-cli
> changes with `fix(themes)!:` only if the JSON candidate set counts as a contract break
> under `decisions:bob-cli-releases-ride-a-daily-train`. Otherwise use `fix(themes):`.
> Never edit `Cargo.toml` versions or `CHANGELOG.md`.

## Tests

Add focused regressions next to the existing theme-row tests, using each file's fake
`createThemesApi`:

- `scripts/test-block-id-prompt-pomodoro-picker-view.cjs`. Its `DAILY` fixture already
  contains `- [x] () — DONE`. Add a theme `DONE` (most recent `last_used`) and an
  open-row theme `CAPTURE` to the fake catalog, then assert:
  - The empty-query rows are `CAPTURE`, `LATER`, `New Pomodoro DONE`, …, and no Recent
    `CAPTURE`.
  - Typing `done` gives exactly the Recent `DONE` row (no separate create row).
    Shift+Enter resolves to `{ kind: "new", name: "DONE", preserveSpelling: true }`.
  - Typing a query that filters `CAPTURE` out of the Today rows still keeps the
    `CAPTURE` theme out of Recent (exclusion happens before filtering).
  - A cancelled `[-]` entry's theme also stays in Recent. Use a small extra daily
    fixture for this.
- `scripts/test-navigation-hotkeys-pomodoro-move-pickers.cjs`. Use a ledger with an open
  `SOURCE`, an open `DESTINATION`, and a done `BUILD`, plus catalog themes for all
  three. Assert that `BUILD` is a Recent row in both bullets mode
  (`PomodoroBulletMovePickerModal` or the `createPomodoroBulletMovePickerRows` helper)
  and entry mode, and that `DESTINATION` and `SOURCE` are not Recent rows in either
  mode. Also add an entry-mode case where the source entry itself is done: its own theme
  must not appear as a "Rename to" Recent row.
- `scripts/test-block-id-prompt-pomodoro-targets.cjs` already exercises
  `buildPomodoroLinkPickerRows`. Add a pure-model assertion there too:
  `buildPomodoroLinkPickerRows(model, "", { themesApi })` keeps a theme that today's
  ledger uses only on a done entry.

Make sure every test file you add is listed in the `npm test` script. Modified existing
files already are.

## Verification

In bob-plugins:

```bash
npm run build          # regenerate block-id-prompt and nav main.js
npm test               # includes build:check, docs inventory, all node tests
npm run docs:check
bob plugins sync       # deploy to ~/bob (required after any bob-plugins change)
```

In bob-cli: `just docs-check` for the PT7 sentence. Under the `show` decision, also run
the cargo commands above and `just lint`.

Optionally, read-only: make sure today's real daily note has a done `BUILD` Pomodoro.
After sync, Bryan can reopen Link to today and confirm that `New Pomodoro BUILD` leads
the Recent rows.

## Commits

Commit bob-plugins (for example
`fix(pomodoro-pickers): keep done-today themes in Recent rows`) and the bob-cli docs
clarification as separate commits in their own repos.

## Out of scope

- Ledger-tools `api.themes` itself, bob-cli `capture-complete` (already correct under
  `keep`), and Bob Mac Capture (a thin renderer of bob's rows).
- The themes manager's Active/Hidden/All views. They are not completion menus and
  already list every theme.
- Hide/unhide semantics (PT9 and `decisions:theme-hide-is-recent-only`) stay unchanged.
  Hidden themes are not "enabled" and stay out of Recent.
- No SASE memory note states the old plugin behavior, so no memory change is needed.
