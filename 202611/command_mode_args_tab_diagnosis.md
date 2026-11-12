---
tier: tale
title: Honest gcal sync echo and a working command-mode Tab
goal:
  Make the sync toast name the day it synced and make Tab in the command sheet always do
  something visible.
size: medium
decisions:
  echo_day:
    ask: Should the sync run's final human line always name the synced day?
    choices:
      always:
        Every echo reads "synced <Ddd YYYY-MM-DD> · <HH:MM> · <account>", today included
      other_day_only:
        Same-day syncs keep the short "synced <HH:MM>" echo; only other days name the
        day
    default: always
    why: One uniform line keeps every toast self-contained with no second date rule.
    answer: always
  tab_recent:
    ask:
      Should Tab on a RECENT single-run row open argument mode prefilled with that run's
      recorded args?
    choices:
      prefill:
        Recent rows with a catalog entry open argument mode with the recorded args ready
        to edit
      hint_only:
        Recent rows only show a footer hint on Tab; args stay reachable from entry and
        typed rows
    default: prefill
    why:
      Editing the last run's args is the gesture Bryan reached for when Tab "did
      nothing".
    answer: prefill
  e2e:
    ask:
      Should a real-Obsidian e2e drive an actual Tab keypress through the command sheet?
    choices:
      real_obsidian:
        CDP e2e in a scratch vault sends a trusted Tab and asserts argument mode renders
      dom_only:
        Only fake-DOM dispatch tests; cannot rule out an Obsidian-level routing failure
    default: real_obsidian
    why:
      Handler logic is already unit-tested; only a real Obsidian run settles the
      reported symptom.
    answer: real_obsidian
proposed_by: bbugyi200.athena.1bw
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 13:18:59
status: wip
---

# Command mode: "args don't work" diagnosis, honest gcal sync echo, and a working Tab

## Problem as reported

Bryan ran `gcal sync -D yesterday` from the Obsidian command sheet (the
`bob-command-mode` plugin's `:` sheet) and got a notification saying 2026-11-12 (today)
was synced, so he believes command-mode args are ignored. He also reports that `Tab` —
documented as entering argument mode — "does nothing".

## Verified diagnosis (do not re-derive; evidence below)

**Args DO work. The notification is misleading.** The host-local run records on the Mac
(`~/.local/state/bob-cli/command-mode/runs/20261112-125315-zryt.*`) prove the typed-pin
flow ran exactly `bob gcal sync -D yesterday` (entry_id `gcal-sync`, title "Run typed
bob command", exit 0) and its stdout header is `bob gcal sync · 2026/20261111.md` —
yesterday's note, synced correctly. The record's `summary` is
`↻ synced Thu 2026-11-12 12:53 · bryanbugyi34@gmail.com`.

Why that summary misleads: `src/native/gcal/sync.rs` `print_human` ends every day report
with the stamp echo (`sync.rs:2223-2225`), and `Stamp::when_text`
(`src/native/gcal/stamp.rs:31-37`) prints the sync's **wall-clock datetime**, including
the full date exactly when it differs from the note day — i.e. precisely when syncing a
non-today day. The plugin's toast summary is the last meaningful stdout line
(`plugins/bob-command-mode/src/070-outcomes.js`, `cmClassifyRun`, exit-0 branch), so the
toast names today's date and never the synced day. The toast title is the record title
"Run typed bob command", which adds no context, and the catalog title "Sync Calendar
events to today" (`src/native/catalog/table.rs:1291-1303`) reinforces the misreading.

**Tab has a real silent no-op path.** In `plugins/bob-command-mode/src/175-args-vim.js`
(`handleSheetKeydown`, the list-stage `action.type === "args"` branch near the file's
end), Tab is consumed but does nothing unless the selected row's kind is `entry` or
`typed`. On an empty query the default selection is the first RECENT row (`170-sheet.js`
`rebuildRows`/`recomputeVisible`), and recent/surface/plain/ sequence rows all hit the
silent branch — no stage change, no hint. The args-vim override also bypasses the base
sheet's hint fallback (`170-sheet.js:453-467`). Everything else checked out: the
deployed `main.js` on the Mac is byte-identical to the repo build, `bob __catalog` and
`bob __complete bash --protocol 1` work on the Mac's bob 0.5.0, and the whole typed-argv
pipeline (`cmResolveTypedBobArgv` → typed pin → `cmSheetRowArgv` → runner
`cmAssembleArgv`) preserves user args. Whether Tab also fails on entry/typed rows in
real Obsidian is unconfirmed — the unit tests call
`enterArgumentStage`/`cmSheetListAction` directly and never push a real keydown through
the modal; the e2e below settles it.

## Scope

Two repos:

- **bob-cli** (this repo): make the sync run's final human line name the synced day;
  day-neutral catalog title. Human output only — the JSON report and the in-note Events
  stamp grammar (`*↻ synced <when> · [<account>](<day-url>)*`) must NOT change
  (`bob-ledger-tools` paints/parses the note stamp, and `stamp.rs` parsing round-trips
  it).
- **bob-plugins** (linked repo — open with `/sase_repo`; read its `AGENTS.md` first):
  fix Tab's silent no-op, give typed runs an honest toast title, and add Tab regression
  coverage including a real-Obsidian e2e.

## bob-cli changes

1. **Self-contained sync echo** (`src/native/gcal/sync.rs`, `src/native/gcal/stamp.rs`):
   change the terminal echo printed at the end of `print_human` so it names the synced
   day (the report's `day`), per the `echo_day` decision:
   - > [!decision] echo_day = always New uniform echo:
     > `↻ synced <Ddd YYYY-MM-DD> · <HH:MM> · <account>`, where the date is the **note
     > day** and the time is the sync wall-clock time (minute-truncated, as today).
     > Example for the reported run:
     > `↻ synced Wed 2026-11-11 · 12:53 · bryanbugyi34@gmail.com`.
   - > [!decision] echo_day = other_day_only Keep `↻ synced <HH:MM> · <account>` when
     > the sync runs on the note's own day; otherwise use the new day-first form above.
   - Implement this in the echo path only: either rework `Stamp::echo` (it has exactly
     one production call site, `sync.rs:2224`) or inline the new line in `print_human`.
     `Stamp::when_text` is shared with the note-stamp `render()` — the note stamp must
     keep its current grammar, so do not change `render()` or the parser.
   - Multi-day runs (`-D a -D b`) print one block per day, each now ending with a
     self-contained echo. Additionally, when more than one day synced in human non-quiet
     mode, print one final combined line after the last block (e.g.
     `↻ synced 2 days · <account>` — exact copy yours) so the toast's last line is
     honest for batched runs too.
   - Update the echo assertions in `stamp.rs` tests (lines ~376, ~393) and any
     `sync.rs`/`plan/tests.rs` human-output tests that match the old echo.
2. **Day-neutral catalog title** (`src/native/catalog/table.rs:1297`): retitle
   `gcal-sync` from "Sync Calendar events to today" to a day-neutral title (e.g. "Sync
   Calendar events"); keep the about/keywords consistent and update catalog
   snapshot/tests (`src/native/catalog/tests.rs`) as needed.
3. **Docs**: `docs/gcal.md` documents the human echo (lines ~615-670, "the human report
   echoes `↻ synced 14:05 · you@example.com`") — update the sample transcript and prose.
   Before touching `docs/`, read the SASE docs memory:
   `sase memory read docs.md -r "..."`. The note-stamp paragraph stays as is.
4. **Contract note**: human output only; no JSON or note-format change, so no `!`
   breaking marker is needed. Never edit `Cargo.toml` version or `CHANGELOG.md` (release
   train owns both).

## bob-plugins changes (plugin `bob-command-mode`)

All edits go in `plugins/bob-command-mode/src/` fragments (committed source build —
never hand-edit the generated `main.js`; run `npm run build` after).

5. **Tab never silently no-ops** (`175-args-vim.js`, list-stage `args` branch):
   - > [!decision] tab_recent = prefill A RECENT single-run row whose record maps to a
     > catalog entry (`row.entryId` set; `cmReplayRun` in `050-run-store.js` already
     > produced `row.argv = [...entry.path, ...userArgs]` with preview/fixed words
     > stripped) enters argument mode prefilled: derive
     > `userArgs = row.argv.slice(entry.path.length)` and hand `enterArgumentStage` a
     > row it accepts (either construct a `typed`-shaped row or extend
     > `enterArgumentStage` to accept such recent rows). Sequence recent rows
     > (`recent-seq:` keys) and recents with no `entryId` are not eligible.
   - > [!decision] tab_recent = hint_only Recent rows get only the hint below on Tab.
   - Every other non-eligible row (surface, plain, vim, excluded, ineligible recent)
     sets a visible footer hint on Tab (e.g. "No argument mode here — pick a bob command
     row") via `this.hint` + render + refocus, mirroring how the base sheet hinted,
     instead of consuming the key silently.
6. **Honest toast title for typed runs** (`175-args-vim.js` typed pin in
   `recomputeVisible`, and/or `170-sheet.js` `activateRow`): when a typed run resolves
   to a catalog entry, pass the entry's title through to the run record (so the toast
   header reads e.g. "Sync Calendar events" instead of "Run typed bob command"). The
   sheet row may keep displaying "Run typed bob command"; only the recorded/toast title
   changes.
7. **Unit tests** (`scripts/test-command-mode-args-vim.cjs` /
   `test-command-mode-sheet-keys.cjs`): cover (a) Tab on an eligible recent row → args
   stage with prefilled query; (b) Tab on surface/plain/sequence rows → hint set, no
   stage change; (c) Tab on entry and typed rows → args stage (through
   `handleSheetKeydown` with a synthetic Tab event, and where the harness allows, by
   dispatching a keydown event through `modalEl.addEventListener` wiring rather than
   calling the method directly); (d) typed-run record title.
8. **Real-Obsidian Tab e2e**:
   - > [!decision] e2e = real_obsidian New `scripts/obsidian-command-mode-tab-e2e.cjs`
     > (+ `test-obsidian-...` wrapper following the existing `test-obsidian-*-e2e.cjs`
     > pattern, skipping gracefully when the Obsidian binary is unavailable): isolated
     > scratch vault + profile per `obsidian-startup-smoke.cjs`, a stub `bob` executable
     > that answers `__catalog --protocol 1` with a minimal valid catalog (one entry
     > with a path and empty `fixed`/`args`) and `__complete` with a couple of
     > candidates, plugin configured to that stub; open the sheet
     > (`bob-command-mode:open`), type a query matching the stub entry, send a
     > **trusted** Tab via CDP `Input.dispatchKeyEvent`, and assert the argument stage
     > rendered (stage pill / `Arguments…` placeholder). If this e2e reproduces a
     > genuine routing failure on entry rows (Tab swallowed before the modal listener),
     > diagnose and fix it in the same change (likely candidates: Obsidian scope/focus
     > handling — register Tab on the modal scope or handle it at capture phase), and
     > keep the e2e as the regression gate.
   - > [!decision] e2e = dom_only Skip the new e2e; rely on the DOM-dispatch unit tests
     > from step 7 and note in the PR that real-Obsidian Tab routing on entry rows
     > remains unverified.
9. **Deploy**: run `bob plugins sync` after building (per the repo's `AGENTS.md`,
   including its refusal-handling rules). Note in the PR that vault plugin deploys are
   host-local (`~/bob/.gitignore` excludes `.obsidian/plugins/bob-command-mode/`), so
   Bryan's Mac picks the fix up on its next `bob plugins sync` there — the plugin fix is
   not live on the Mac merely by merging.

## Validation gates

- bob-cli: `cargo fmt --check`, `cargo clippy`, and the test suite (use the `justfile`
  targets if present); scoped runs for `gcal` first, full suite before finishing.
- bob-plugins: `npm run build` then `npm test` (includes `build:check`, docs inventory
  check, unit + perf); `npm run docs:check` if docs change. Run the new e2e locally
  (athena has `/snap/bin/obsidian`).
- Re-verify the reported flow end to end where practical: with the new bob built,
  `bob gcal sync -D yesterday --dry-run` output's final line must name yesterday, not
  today (dry-run avoids vault writes; the echo prints for real syncs — asserting the new
  echo via the Rust tests is sufficient if auth or network makes a live run
  impractical).

## Non-goals / guard rails

- Do not change the in-note Events stamp grammar, the gcal JSON report, the
  `__catalog`/`__complete` protocols, or capture behavior.
- Do not touch `Cargo.toml` version or `CHANGELOG.md`.
- The decision records `mac-capture-is-a-thin-client` and the command-mode design
  (binary decides, clients present) stay honored: the fix makes bob's own output
  self-contained rather than teaching the plugin to reinterpret it.
