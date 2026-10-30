---
tier: tale
title: Fix re-running RECENT commands in Bob Command Mode
goal:
  "RECENT rows in the : command sheet show commands without the bob path, and Enter /
  Ctrl+Enter rerun exactly that command."
size: medium
decisions:
  recent_enter:
    ask:
      Should Enter on a RECENT row honor its preview-first/confirm guard, with
      Ctrl+Enter as the immediate rerun?
    choices:
      guarded:
        Enter previews or asks to confirm like the catalog row; Ctrl+Enter reruns at
        once
      verbatim:
        Enter and Ctrl+Enter both rerun the recorded command at once, skipping its guard
    default: guarded
    why:
      Matches the sheet's ↵ run / ⌃↵ run now contract and keeps writers like gkeep pull
      previewed
    answer: guarded
  replay_guards:
    ask:
      Should Retry/Apply toasts, Rerun last command, and pager r stop refusing applied
      preview-first or confirm runs?
    choices:
      fix:
        Replays of an applied run skip the guard it cleared; the task-reroll Apply toast
        starts working
      leave:
        Only RECENT rows and in-sheet r change; those surfaces keep refusing applied
        runs
    default: fix
    why:
      These replays are refused today with no way forward; the Apply toast exists only
      to apply
    answer: fix
proposed_by: bbugyi200.athena.16h
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 15:29:44
status: wip
---

# Fix re-running RECENT commands in Bob Command Mode

## Problem

The `:` command sheet (Bob Command Mode plugin, `bob-plugins` linked repo,
`plugins/bob-command-mode/`) lists RECENT runs when the query is empty. These rows are
broken in three ways:

1. **The bob executable shows in each row**, for example
   `/Users/bbugyi/.cargo/bin/bob task reconcile`.
2. **Enter and Ctrl+Enter run the wrong command.** Pressing Enter on `task reconcile`
   runs `bob task reconcile reconcile`, which fails with
   `error: unexpected argument 'reconcile' found`. A RECENT row on Bryan's machine
   already shows that broken command from an earlier attempt.
3. **Selection is corrupted.** Two RECENT rows with the same entry id (or a RECENT row
   and the matching catalog row) share one row key. Both rows highlight together,
   `Ctrl+N` / `Ctrl+P` loop instead of walking the list, and Enter runs the first row
   with that key rather than the highlighted one. The screenshot shows both
   `task reconcile` rows highlighted at once.

### Root cause

- `CommandModeRunner.spawnRun` (`src/060-runner.js`, "Record the exact argv that ran")
  rewrites `record.argv` to `[<resolved bob path>, ...spawnArgs]`. Queued and
  spawn-failed records keep the bare args, without the binary.
- `cmSheetRecentRows` (`src/160-sheet-rank.js`) strips only a literal `"bob"` word. Real
  runs resolve bob to an absolute path, so the row keeps it in `path` (fault 1) and in
  `argv`.
- `rerunRecent` (`src/170-sheet.js`) passes that argv, binary included, to
  `runner.run(argv, { entryId })`. The runner then computes
  `cmAssembleArgv(entry, argv.slice(entry.path.length))`. With
  `argv = [bin, "task", "reconcile"]` and `entry.path = ["task", "reconcile"]`, the
  slice is `["reconcile"]`, which gives `task reconcile reconcile` (fault 2). It also
  ignores Enter, Ctrl+Enter, and Shift+Enter: every key runs the same unguarded call,
  which the runner then refuses for preview-first or confirm entries.
- The sheet's in-output `r` handler (`rerunOutput` in `src/170-sheet.js`) has the same
  doubling bug.
- RECENT row keys are `"entry:" + run.entry_id`, which is the catalog row's key scheme
  (`cmSheetEntryRows`). Selection, `rowEls`, and section headings are all keyed by
  `row.key` (fault 3).
- `BobCommandModePlugin.commandModeRerun` (`src/090-plugin.js`) already strips the
  binary correctly (it also checks `record.bob_path`) and drops preview and fixed args,
  but this logic is private to it. Three display helpers each repeat a regex-only binary
  strip: `cmToastArgv` (`110-toasts`), `cmChipShortArgv` (`120-status-chip`), and
  `cmPagerDisplayArgv` (`140-ansi-and-links`).
- The tests did not catch this because the shared runner harness resolves bob to the
  bare word `"bob"`.

Adjacent bug (decision `replay_guards`): `commandModeRerun` never passes `skipGuards`,
so the runner refuses the following with "Run the preview first, then apply.":

- toast **Retry** on an applied preview-first writer (gkeep pull, gcal sync, task
  reroll)
- toast **Apply** (`rerun-with-carry` from the task-reroll adapter)
- `Bob: Rerun last command` on such a run
- the standalone pager's `r` on such a run

Confirm-policy runs are refused the same way. The existing test in
`scripts/test-command-mode-outcomes-api.cjs` stubs the runner, so it never exercises the
guard.

## Where to work

All changes are in the **bob-plugins** linked repo. Open it with your `/sase_repo` skill
(`sase repo open bob-plugins -r "<reason>"`), use the printed path, and read its
`AGENTS.md`. The plugin uses the fragment build: edit
`plugins/bob-command-mode/src/*.js` only, keep each fragment at or below 1000 lines, and
regenerate `main.js` with `npm run build`. Never hand-edit `main.js`. bob-cli needs no
change: its `docs/reference/plugins/` copy is a gitignored mount of bob-plugins docs.

## Design

### 1. One record-replay helper (`src/050-run-store.js`)

Add two pure helpers next to the other run-record code and export them from
`src/990-exports.js` for tests:

- `cmRecordArgs(record)` returns the record's argv without a leading bob binary. Strip
  `argv[0]` only when it is non-empty and is one of: `"bob"`, `record.bob_path`, or a
  path matching `/(?:^|\/)bob$/`. Otherwise return the argv unchanged, because queued
  and spawn-failed records hold bare args. The result is the user-facing command without
  `bob`.
- `cmReplayRun(record, catalog)` returns `{ argv, preview, entryId }`, the exact inverse
  of `cmAssembleArgv` (path + preview + fixed + user args):
  - Start from `words = cmRecordArgs(record)`.
  - Find the entry with `cmEntryById(catalog, record.entry_id)`.
  - When the entry exists and `words` starts with `entry.path`: drop `entry.preview`
    positionally right after the path, only when `record.preview` is true and those
    words are present, then drop `entry.fixed` positionally. Return
    `{ argv: [...entry.path, ...rest], preview: Boolean(record.preview), entryId: entry.id }`.
  - Otherwise (unknown entry or a path mismatch) return
    `{ argv: words, preview: false, entryId: null }`. Any raw preview flags stay in
    `words`, the runner re-resolves the entry and keeps its exclusion guard, and the
    runner never gets an entry id it would mis-slice.
  - Do not filter preview flags anywhere else in the args. A non-preview run whose user
    args happened to include `--dry-run` must replay unchanged, never silently turning
    into a real write.

### 2. RECENT rows (`src/160-sheet-rank.js`)

Change `cmSheetRecentRows(runs, nowMs)` to `cmSheetRecentRows(runs, nowMs, catalog)`,
and pass `this.catalog` from `rebuildRows` in `src/170-sheet.js`. For each run, compute
`replay = cmReplayRun(run, catalog)` and the entry it resolved, then build:

- `key: "recent:" + JSON.stringify({ argv: replay.argv, preview: replay.preview })`.
  This key never collides with `entry:` catalog keys. Also dedupe on this signature
  instead of the raw argv, so a bob-path change or a binary-less queued record does not
  repeat a row.
- `path: cmRecordArgs(run)`. This display keeps preview, fixed, and user args, so a
  recorded preview reads `gkeep pull --dry-run`, but never includes the binary.
- `argv: replay.argv`, `preview: replay.preview`, `entryId: replay.entryId`.
- Entry metadata so the normal activation pipeline can apply guards:
  - `policy`: `cmNormalizePolicy(entry.policy)`, or `"run"` without an entry.
  - `presentation`: the entry's presentation, else `run.presentation`, else `"action"`.
  - `hasPreview` and `confirm`, computed the same way as `cmSheetEntryRows`.
  - `required: []`, because recorded args are complete.
- `runKey: cmSheetRunKey(run)`, used by `cmSheetMedianDuration` for RECENT rows instead
  of rebuilding a signature from `row.argv`. That rebuild would otherwise break once
  `row.argv` is normalized.
- Keep `title`, `about` (`run.summary`), `outcome`, `glyph`, `age`, `duration`, and
  `icon` unchanged. Do not add effect chips; the RECENT visual stays the same apart from
  the path.

Also make `cmSheetRunKey` key no-entry runs on `cmRecordArgs(run)`, so their history no
longer depends on the bob path.

### 3. Activating a RECENT row (`src/170-sheet.js`)

Delete `rerunRecent` and let RECENT rows go through `activateRow`, so Enter,
Shift+Enter, and Ctrl+Enter mean the same thing they mean on catalog rows:

- **Recorded previews.** If `row.kind === "recent" && row.preview && row.hasPreview`,
  every key calls `this.runPreview(row, row.argv)`. The preview reruns in the sheet, and
  Ctrl+Enter in that output applies it through the existing `applyPreview`. Without
  `hasPreview`, fall through and never re-enter `runPreview`. Its no-preview fallback
  calls `activateRow`, so this guard prevents infinite recursion.
- **All other RECENT rows** use the existing pipeline unchanged: the missing-positional
  check (a no-op because `required` is `[]`), Shift+Enter → `runPreview`, preview-first
  policy → `runPreview` unless forced, confirm policy → confirm card unless forced,
  report → `runReport` in the sheet, otherwise
  `runner.run(argv, { entryId, title, ...(force ? { skipGuards: true } : {}) })`. Pass
  `trigger: "sheet-recent"` for RECENT rows (today's value) and `"sheet"` for everything
  else.

> [!decision] recent_enter = guarded Enter follows the guard above, and Ctrl+Enter
> (`force`) reruns at once with `skipGuards`. Policy-`run` commands such as
> `task reconcile` or `plan` rerun immediately on either key. With `verbatim`, treat
> every non-preview RECENT activation as forced: Enter behaves like Ctrl+Enter, and
> Shift+Enter still previews.

`args-vim`'s `activateRowWithArgsVim` override already passes RECENT rows to the
original `activateRow`. Tab on a RECENT row keeps its current no-op.

### 4. Other rerun surfaces

- **In-sheet output `r`** (`rerunOutput` in `src/170-sheet.js`): use
  `cmReplayRun(record, this.catalog)` and call
  `runner.run(replay.argv, { entryId, title, trigger: "sheet-rerun", preview: replay.preview })`.
  Keep its existing follow-the-new-run handling.
- **`commandModeRerun`** (`src/090-plugin.js`): replace the inline strip/inverse logic
  with `cmReplayRun(current, runner.catalog())`. Keep its `carry` and `preview` override
  semantics and its `trigger: "retry"`. The existing "rerun-last preserves arguments and
  preview mode; Apply carries the replay seed" test must still pass unchanged.

> [!decision] replay_guards = fix In `commandModeRerun` and `rerunOutput`, pass
> `skipGuards: true` whenever the effective run is not a preview. A record only exists
> for a run that cleared its guard: it was previewed then applied, confirmed, or forced.
> An Apply (`preview: false` over a preview record) is the toast equivalent of the
> sheet's `applyPreview`, which already skips the gate. Add a short comment saying so.
> With `leave`, pass no `skipGuards` here, exactly as today.

### 5. Display helpers

Make `cmToastArgv`, `cmChipShortArgv`, and `cmPagerDisplayArgv` call
`cmRecordArgs(record)` instead of each repeating the regex strip. They keep formatting
with `cmFormatArgv(["bob", ...args])`, so the toast, chip, pager header, and the `Y` /
Copy text stay `bob task reconcile`. They now also handle a configured bob path whose
basename is not `bob`.

Stored records keep the honest `[bob path, ...args]` argv; there is no schema change and
no migration. Old broken records (such as the `task reconcile reconcile` failure) age
out through normal retention.

## Tests

Add these to the existing command-mode suites, which already run in `npm test`:

- **Pure helpers** (`scripts/test-command-mode-sheet.cjs` or the catalog suite):
  - `cmRecordArgs` strips `"bob"`, `/Users/x/.cargo/bin/bob`, and a non-`bob`-named
    `record.bob_path` such as `/tmp/fake-bob.cjs`, and leaves a binary-less queued argv
    untouched.
  - `cmReplayRun` round-trips:
    `cmAssembleArgv(entry, replay.argv.slice(entry.path.length), { preview: replay.preview })`
    deep-equals `cmRecordArgs(record)` for:
    - a plain run
    - a preview run
    - an entry with both `preview` and `fixed` args
  - `cmReplayRun` cases with no round-trip:
    - A non-preview record whose user args include `--dry-run` keeps it.
    - An unknown entry id returns the raw words with `entryId: null` and
      `preview: false`.
- **RECENT rows** (`cmSheetRecentRows`, absolute-path runs):
  - No row path contains the binary.
  - Keys are unique and never start with `entry:`.
  - Two runs that differ only in bob path collapse to one row.
  - A preview run's path shows `--dry-run` while its `argv` omits it.
  - Rows carry `policy`, `hasPreview`, and `confirm` from the entry.
- **Sheet regression with the real fake bob** (`sheetHarness`, which already resolves
  bob to the absolute `FAKE_BOB` path). Run `vault-sync run` (policy `run`, two-word
  path, the `task reconcile` stand-in) through the runner, then reopen the sheet:
  - The RECENT row text does not contain `FAKE_BOB`.
  - Enter produces a new record whose `cmRecordArgs` is exactly `["vault-sync", "run"]`,
    never `["vault-sync", "run", "run"]`.
  - Ctrl+Enter does the same.
  - With two distinct RECENT rows for one entry, `Ctrl+N` moves from the first to the
    second and only one row renders as selected.
- **Guards on RECENT rows:**
  - An applied `ref-rename` run: Enter enters the in-sheet preview (with
    `recent_enter = guarded`), and Ctrl+Enter runs immediately with no `--dry-run`.
  - A recorded `ref-rename` preview: Enter reruns it as a preview.
  - A `task-archive` run: Enter shows the confirm card and Ctrl+Enter runs.
- **In-sheet `r`:** rerunning a `vault-sync run` output does not double the path.
- **`commandModeRerun` with a real runner** (with `replay_guards = fix`):
  - Retry of an applied preview-first record is not refused.
  - `rerun-with-carry` Apply on a preview record runs with the carry and without
    `--dry-run`.

## Docs

Update `docs/bob-command-mode.md` in bob-plugins:

- In **Command sheet**, add one or two sentences for RECENT rows:
  - They show the command without the `bob` path.
  - Enter reruns it under the command's preview or confirm guard, Ctrl+Enter reruns it
    now, and a recorded preview reruns as a preview.
  - Word it to match the chosen `recent_enter` branch.
- If `replay_guards = fix`, extend the `Bob: Rerun last command` sentence in **Run
  commands** to say a replay of an applied run does not ask for its preview or
  confirmation again.

Run `npm run docs:check`. The bob-cli guide (`docs/guides/run-bob-from-obsidian.md`)
stays correct as written, so leave it alone.

## Verification

From the bob-plugins checkout:

1. `npm run build`, then `npm test` (it starts with `build:check` and the docs inventory
   check). All tests must pass.
2. `npm run docs:check`.
3. `bob plugins sync` to deploy to the vault (required by the bob-plugins `AGENTS.md`).
4. Report the manual check for Bryan: reload the plugin, press `:`, and confirm that:
   - RECENT shows `task reconcile` with no path.
   - Enter and Ctrl+Enter both rerun it successfully.
   - `Ctrl+N` walks every RECENT row one at a time.

## Out of scope

- Tab (argument mode) on RECENT rows.
- `?` help for RECENT rows.
- Changing the on-disk record schema.
- Pruning existing broken records.
