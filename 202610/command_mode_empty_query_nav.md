---
tier: tale
title: "Make Ctrl+N / Ctrl+P walk the : command sheet as soon as it opens"
goal:
  "On a freshly opened : command sheet with run history, Ctrl+N / Ctrl+P (and the
  arrows) walk every row one at a time without typing first."
size: small
proposed_by: bbugyi200.athena.16h.f0
proposed_with: claude/opus
create_time: 2026-10-30 15:37:46
status: wip
---

# Make Ctrl+N / Ctrl+P walk the `:` command sheet as soon as it opens

## Problem

Bryan opens the Bob Command Mode sheet with `:` and presses `Ctrl+N` / `Ctrl+P`. The
selection does not walk the list. Navigation only works after he types something.

### Root cause (reproduced)

On an empty query the sheet shows a RECENT section above the catalog sections. In the
**bob-plugins** linked repo (`plugins/bob-command-mode/`):

- `cmSheetRecentRows` (`src/160-sheet-rank.js`) gives each RECENT row
  `key: "entry:" + run.entry_id`. `cmSheetEntryRows` uses the same `"entry:" + entry.id`
  scheme for catalog rows.
- Two runs of one command with different argv (for example `task reconcile` and the
  broken `task reconcile reconcile`, or `gkeep pull` and `gkeep pull --dry-run`) make
  two RECENT rows with the same key. Each RECENT row also shares its key with that
  command's catalog row further down.
- `CommandModeSheetModal.select(delta)` (`src/170-sheet.js`) finds the current row with
  `visible.findIndex(row => row.key === selectedKey)`, which always returns the first
  row with that key. `renderRow` highlights every row with `selectedKey`.

So `Ctrl+N` moves from row 0 to row 1. If row 1 shares row 0's key, the next press
starts again from row 0. Walking down to a catalog row whose key matches a RECENT row
jumps back to the top. Bryan's screenshot shows this: both `task reconcile` RECENT rows
are highlighted at once.

Typing hides RECENT, because RECENT rows are only built for an empty query. That is why
navigation works once he types.

The cause was confirmed with a scratch script against the existing test harness:

- Seed three records with an absolute bob path: `plan --dry-run`, `ready`, `plan`.
- Open the sheet and press `Ctrl+N` six times. The selected index goes
  `0 → 1 → 0 → 1 → 0 → 1 → 0`.
- Rewrite only the RECENT keys to a `recent:` namespace and repeat. `Ctrl+N` walks all
  18 rows in order and wraps, and `Ctrl+P` walks back. This holds with FOR THIS NOTE and
  FROM CLIPBOARD sections present.

No other blocker exists: no focus, key-alias, or scope problem. The existing navigation
tests pass only because their runs directory is empty, so the sheet shows no RECENT
section.

### Relation to the approved RECENT rerun tale

The approved tale `command_mode_recent_rerun` covers the same bug as its "fault 3". It
gives RECENT rows `key: "recent:" + JSON.stringify({ argv: replay.argv, preview })`,
built on its new `cmReplayRun` helper. Its navigation test only checks a single `Ctrl+N`
step between two RECENT rows. This plan:

- makes sure the key fix is present whichever tale lands first, and
- adds the end-to-end regression test that would have caught this bug: a full walk on a
  freshly opened sheet with realistic history.

It must not re-implement anything else from that tale: path stripping, replay, guards,
or dedupe changes.

## Where to work

All changes are in the **bob-plugins** linked repo. Open it with your `/sase_repo` skill
(`sase repo open bob-plugins -r "<reason>"`), use the printed path, and read its
`AGENTS.md`. The plugin uses the fragment build:

- Edit `plugins/bob-command-mode/src/*.js` only, and keep each fragment at or below 1000
  lines.
- Regenerate `main.js` with `npm run build`. Never hand-edit `main.js`.

bob-cli needs no change.

## Steps

### 1. Ensure RECENT rows have their own key namespace

Check `cmSheetRecentRows` in `src/160-sheet-rank.js`.

- **Already fixed:** RECENT keys already start with `"recent:"` because the approved
  tale has landed. Leave the code alone and go to step 2.
- **Not fixed yet:** RECENT keys still use `"entry:" + run.entry_id`. Make the smallest
  change that fixes navigation:
  - Set the RECENT row key to `"recent:" + signature`. `signature` is the
    `JSON.stringify(run.argv)` the function already dedupes on, so keys are unique
    within RECENT and never collide with `entry:`, `ctx:`, `surface:`, or `plain:` keys.
  - Remove the `"argv:" + signature` fallback, because the new key covers runs without
    an entry id.
  - Change nothing else in the row. `cmSheetMedianDuration` already uses `row.argv` for
    RECENT rows, and no other code reads the `entry:` prefix of a RECENT row.
    `rerunRecent` uses `row.argv` and `row.entryId`, not the key.
  - Update the hand-built RECENT fixture in the "empty query groups RECENT, note,
    clipboard, then catalog sections" test (`scripts/test-command-mode-sheet.cjs`). It
    hard-codes `key: "entry:plan"`; use a `recent:` key.
  - If the approved tale lands after this change, its rebase replaces this key line with
    its own `recent:` key. The tests below hold under either key.

In both cases, add one short comment above the `key:` line in `cmSheetRecentRows`. Merge
it with the approved tale's comment if one is already there. The comment says row keys
must be unique across the whole sheet, because selection, highlighting, `rowEls`, and
section headings are keyed by `row.key`.

Do not add a runtime de-duplication or key-suffixing pass in `recomputeVisible`. It
would hide a bad key scheme instead of failing a test, and it would detach renamed rows
from their per-key frecency and duration data.

### 2. Regression test: navigate a freshly opened sheet with real history

Add a test to `scripts/test-command-mode-sheet-keys.cjs` (the "reliable Ctrl+N/P"
suite). Its setup follows the suite's `makeSheet`. Before `modal.open()`, seed the
runner's runs directory (`base.runner.runsDir()`) with finished records:

- Create each record with `helpers.cmNewRecord`, then write it with
  `helpers.cmWriteRecord`. Use `bobPath: "/Users/x/.cargo/bin/bob"` and
  `outcome: "completed"`, and give each its own `id` and increasing `startedAt`.
- Seed three records:
  - `[BIN, "plan", "--dry-run"]` with entry `plan` and `preview: true`
  - `[BIN, "ready"]` with entry `ready`
  - `[BIN, "plan"]` with entry `plan`
- With these records, RECENT holds two rows for `plan`, and both collide with the
  catalog's `plan` row under the old key scheme.
- Open with context
  `{ notePath: "refs/a.md", noteKind: "ref", clipboardUrl: "https://example.com/x" }`,
  so FOR THIS NOTE and FROM CLIPBOARD also render. Extend `makeSheet` with an optional
  `records` argument if that keeps the setup readable.

Assert, without typing anything:

- Every `modal.visible` key is unique, and RECENT has at least three rows.
- `modal.selectedKey === modal.visible[0].key`, and exactly one rendered row is
  selected. Count the rendered `bob-cmd-row` elements under `modal.listEl` that carry
  `is-selected`; do not count `rowEls` entries, because that map collapses duplicate
  keys.
- Pressing `Ctrl+N` `visible.length` times selects indexes `1, 2, …, n-1, 0` in order.
  After every press, exactly one row is selected and focus stays on `modal.inputEl`.
- Pressing `Ctrl+P` `visible.length` times walks the same rows in reverse.
- `ArrowDown` and `ArrowUp` each move one row.
- Typing a character and then deleting it with the input's `input` event brings RECENT
  back, and `Ctrl+N` still moves from row 0 to row 1.

The test must fail on the old `entry:` keys and pass after step 1. If step 1 changed the
code, confirm the failure by running the new test once before the fix. If the approved
tale had already landed, the scratch reproduction above already shows the old failure
(`0 → 1 → 0 → 1`), so just make sure the test passes.

### 3. Docs

`docs/bob-command-mode.md` already says `Ctrl+P` / `Ctrl+N` move between rows in the
list, so no docs change is needed. Still run `npm run docs:check`.

## Verification

From the bob-plugins checkout:

1. `npm run build`, then `npm test`. `npm test` starts with `build:check` and the docs
   inventory check. All tests must pass.
2. `npm run docs:check`.
3. `bob plugins sync` to deploy to the vault (required by the bob-plugins `AGENTS.md`).
4. Report this manual check for Bryan:
   - Reload Bob Command Mode.
   - Press `:` and, without typing, press `Ctrl+N` repeatedly. Each press highlights
     exactly one row and moves down through RECENT, FOR THIS NOTE, and the catalog
     sections, then wraps.
   - `Ctrl+P` walks back up.

## Out of scope

Everything else in the approved `command_mode_recent_rerun` tale:

- bob-path display
- rerun argv and guards
- replay helpers
- dedupe on the replay signature
- the toast and pager rerun fixes
