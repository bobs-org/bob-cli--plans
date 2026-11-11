---
tier: tale
title: Ctrl+R redo survives disk reloads of an open note
goal:
  Redo after vim u keeps working when vault-sync, the bob CLI, or a plugin rewrites the
  open note on disk.
size: medium
decisions:
  reload_history:
    ask:
      Should every disk reload of an open note stay out of that editor's undo history?
    choices:
      always: Redo always survives a reload, and u never reverts a background write
      only_when_redo_pending:
        Only reloads during a pending redo stay out; u can still revert other background
        writes
    default: always
    why:
      CodeMirror's model for remote edits; today u reverts synced writes that sync back
      as Bryan's
    answer: always
  decision_record:
    ask:
      Record the policy as a new decisions-web record (disk-reloads-stay-out-of-undo)?
    memory:
      - decisions:disk-reloads-stay-out-of-undo
    default: false
    answer: false
proposed_by: bbugyi200.athena.1b4.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 06:02:51
status: wip
---

# Ctrl+R Redo Survives Disk Reloads Of An Open Note

## Verdict On The Suspicion

Bryan's suspicion is **right about the symptom and wrong about the cause**. The Ctrl+R
keymap is fine: Obsidian's Vim mode maps `<C-r>` to CodeMirror 6 `redo`, and no Bob
plugin intercepts it. What fails is the redo history. Whenever anything **writes the
open note on disk** between `u` and Ctrl+R, Obsidian reloads the editor. The reload is a
new undo-history event, and CodeMirror 6 throws away the whole redo branch whenever a
new event is recorded. Ctrl+R then has nothing to redo and does nothing, with no notice.
It looks random because it depends on whether a background writer touched that note
during the gap.

The same mechanism causes a second, quieter bug. A `u` pressed after a background write
**reverts the background write**, not Bryan's last edit. The reverted text is then
autosaved and synced as if Bryan had edited it.

## Evidence (real Obsidian 1.15.0, Vim mode, scratch vault, CDP keys)

The probe was adapted from `scripts/obsidian-block-id-rename-e2e.cjs` and drove real
Obsidian under `xvfb-run` through CDP key events. Each case typed an edit, pressed `u`,
waited, pressed Ctrl+R, and checked that the text matched the post-edit text. It also
wrapped the active `EditorView.update` to log every doc-changing transaction between `u`
and Ctrl+R: its `userEvent`, its `addToHistory`, and its call stack.

| Setup                                                             | Cases                                                                                                                                                                                                                                | Result                                                    |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| No plugins                                                        | `A…<Esc>`, `x`, `dd` on plain, task, BLOCKS, and Pomodoro-link lines; waits of 150 ms and 3 s                                                                                                                                        | 16/16 redo OK                                             |
| All 9 Bob plugins                                                 | Same cases, plus Depends-On `dd` / append, task-text `x`, BLOCKS `dd`, cycle-status and toggle-task commands, `uu` then two Ctrl+R, `u` then cursor moves then Ctrl+R, and a mirror-generating Depends-On edit with one or two undos | All redo OK; no plugin write lands between `u` and Ctrl+R |
| All 9 Bob plugins, external `fs` write to the open note after `u` | plain note, deps note; both waits                                                                                                                                                                                                    | **4/4 redo FAIL**                                         |
| Same, with `vault.process` on the open note after `u`             | plain note                                                                                                                                                                                                                           | **redo FAIL**                                             |

The logged reload transaction was identical every time: `userEvent: "set"`, recorded
into history, with the stack
`loadFileInternal → setData → setViewData → editor.set → dispatch`.

Classification of write paths onto the active note:

| Write path                                                                                              | `userEvent`     |
| ------------------------------------------------------------------------------------------------------- | --------------- |
| `editor.setValue`, `editor.replaceRange`, `editor.transaction` (Bob's open-note writers)                | none            |
| Vim `u` / Ctrl+R                                                                                        | `undo` / `redo` |
| External process write (bob CLI, vault-sync merge), `vault.modify`, `vault.process`, `view.setViewData` | **`set`**       |

### Prototype Fix

A prototype `EditorState.transactionExtender` added `Transaction.addToHistory.of(false)`
to doc-changing `set` transactions. It turned every failing case green:

- Redo survives external writes and `vault.process`.
- The externally written text is kept after the redo.
- `u` after an external write undoes Bryan's own edit and keeps the external line.
  Without the extender, `u` reverted the external write.
- Switching files in the same leaf still does not leak undo history.

CodeMirror's history maps both the done and undone branches through changes marked
`addToHistory: false`, which is the mechanism CodeMirror designed for remote and collab
changes.

### Who Writes Open Notes On The Mac

These writers can each land between `u` and Ctrl+R:

- `bob vault-sync -q`, every 15 s from the LaunchAgent. It merges athena, apollo, and
  agent writes into the working tree.
- Bob Mac Capture. Each capture is a `bob capture` write.
- The 15-minute and hourly crons: `bob task reconcile`, gcal, highlights, and Gmail
  sync.
- Plugin `vault.process` writes to a note that is open in another pane.

That spread is why the failure looks intermittent.

### Ruled Out

- **No Bob plugin binds Ctrl+R.** No fragment or hand-edited `main.js` registers `<C-r>`
  or `Mod-r`, or any capture-phase keydown handler for R. The full-plugin probe shows
  Ctrl+R working.
- **bob-vim-surround.** It swallows any non-pair key, Ctrl+R included, only while a
  `ys`/`cs`/`ds` target is pending. That is the intended behavior and out of scope.
- **The vault's `obsidian_vimrc.md` was not inspected.** The vault is outside this
  project's repo inventory. A vimrc `<C-r>` remap would break Ctrl+R every time, not
  intermittently, so it does not explain the symptom.

## Fix

Add one editor extension to **bob-navigation-hotkeys**. It already owns Bob's
editor-transaction helpers (`180-editor-transactions.js`) and the Vim jump-history
bridge. The extension keeps disk reloads of an open note out of the CodeMirror undo
history.

1. **New fragment `plugins/bob-navigation-hotkeys/src/182-reload-history.js`.** Insert
   it in `src/fragments.json` right after `181-editor-line-diff.js`. It holds:
   - A pure predicate `isDiskReloadTransaction(tr, Transaction)`. It is true only when
     `tr.docChanged`, `tr.isUserEvent("set")`, and
     `tr.annotation(Transaction.addToHistory) !== false`. Guard every member access,
     because Node harnesses have no CodeMirror.
   - `createReloadHistoryExtension(codeMirrorState)`. It returns
     `EditorState.transactionExtender.of((tr) => isDiskReloadTransaction(tr, Transaction) ? { annotations: Transaction.addToHistory.of(false) } : null)`,
     or `null` when `EditorState`, `Transaction`, or `transactionExtender` is missing.
   - A header comment that records the `set` provenance
     (`loadFileInternal → setViewData`), the redo-wipe mechanism, and why Bob's own
     open-note writers are unaffected (they carry no `set` event).

   > [!decision] reload_history = only_when_redo_pending The predicate also requires
   > `redoDepth(tr.startState) > 0`, taken from `require("@codemirror/commands")` inside
   > the same try/catch pattern used for `@codemirror/state` in
   > `010-requires-and-config.js`. Missing `redoDepth` falls back to the `always`
   > behavior. Docs and tests describe "reloads while a redo is pending" instead of
   > "every reload".

2. **Requires.** In `010-requires-and-config.js`, extend the `@codemirror/state`
   destructure to include `EditorState` and `Transaction`. Keep the existing try/catch
   shape.
3. **Registration.** In `490-plugin-lifecycle.js`, beside the existing
   `EditorView.updateListener` registration, call `this.registerEditorExtension(ext)`
   when the factory returns non-null. Do not wrap the extension in `Prec`; extenders run
   regardless of precedence.
4. **Exports.** Export the predicate and factory through the module's existing
   test-export fragment, the way other nav helpers are exported for the
   `scripts/navigation-hotkeys-harness.cjs` tests.
5. **Leave other behavior alone.** Do not change the dependency hand-edit mirror,
   block-ID rename follow, or any writer. Undo of Bob's own open-note writes stays
   exactly as documented, because those writes never carry `set`.
6. Bump the bob-navigation-hotkeys `manifest.json` minor version, run `npm run build`,
   and never hand-edit the generated `main.js`.

## Tests

1. **Unit test `scripts/test-navigation-reload-history.cjs`**, added to the `npm test`
   list in `package.json`. Use stub `Transaction` and `tr` objects with `docChanged`,
   `isUserEvent`, and `annotation`. Cover:
   - `set` with a doc change → annotated.
   - `set.something` → annotated, since `isUserEvent` prefix-matches.
   - No doc change, `undo`, `redo`, `input`, `delete`, and no userEvent → `null`.
   - Already `addToHistory: false` → `null`.
   - Missing CodeMirror classes → the factory returns `null`.
   - Under `only_when_redo_pending`, also the redo-depth gate.
2. **Real-Obsidian e2e `scripts/obsidian-redo-history-e2e.cjs`**, plus the offline
   `--self-test` wrapper `scripts/test-obsidian-redo-history-e2e.cjs` in `npm test`.
   Model it on `scripts/obsidian-block-id-rename-e2e.cjs`: isolated scratch vault and
   profile, all repo plugins installed, `vimMode: true`, CDP keys, and never the live
   vault. Cases:
   - `x` edit → `u` → external `fs` append to the open note → wait for the reload →
     Ctrl+R. The note must equal the edited text plus the external line.
   - The same case with `vault.process` instead of the external write.
   - `x` edit → wait for autosave → external append → `u`. The user's edit is undone and
     the external line stays.
   - Edit note A, then open note B in the same leaf → `u` leaves B unchanged.
   - Regression: plain `x` / `dd` → `u` → Ctrl+R with no external write still redoes.
   - Bob's undo contract: the block-ID rename e2e must still pass unchanged.

   Run it with `xvfb-run -a node scripts/obsidian-redo-history-e2e.cjs` (the Obsidian
   snap exists on athena), and run the block-ID rename e2e the same way. Report both
   outputs.

3. Run `npm test` and `npm run validate`. Both must pass.

## Docs

- **`docs/bob-navigation-hotkeys.md`.** Add a short "Undo history and disk reloads"
  section. Say that when a note open in the editor is rewritten on disk (by vault-sync,
  the bob CLI, Mac Capture, or another plugin's `vault.process`), the reload no longer
  enters that editor's undo history: Ctrl+R / redo survives, and `u` undoes your own
  last edit instead of reverting the background write. Also add one clause to the long
  description cell.
- **Docs inventory.** Regenerate it with `npm run docs:gen` (the version inventory in
  `docs/index.md`) and pass `npm run docs:check`.

> [!decision] decision_record Use `/sase_memory_write`, then add the
> `decisions:disk-reloads-stay-out-of-undo` strand in bob-cli in the existing record
> shape:
>
> - **Applies to:** bob-plugins (bob-navigation-hotkeys) and every Bob writer that
>   touches open notes.
> - **Claim:** a disk write to an open note never enters that editor's undo history;
>   Bob's open-note gestures must keep writing through editor transactions to stay
>   undoable.
> - **Why:** this plan's evidence.
> - **Rejected:** keeping reloads undoable (it wipes redo and lets `u` revert synced
>   writes); a per-writer fix (external processes can't be fixed).
> - **Cost:** a background write can't be undone with `u`; git history keeps it.
> - **Reopens when:** Obsidian stops tagging reloads `set`, or Bryan wants `u` to revert
>   background writes.
> - **Evidence:** this plan and the bob-navigation-hotkeys version.
>
> Then run `sase memory init`.

## Deploy And Verify

1. Commit the bob-plugins changes in the linked bob-plugins checkout, opened with
   `/sase_repo`.
2. Run `bob plugins sync`, following AGENTS.md's refusal procedure if it refuses.
3. Confirm with `bob plugins list` that bob-navigation-hotkeys shows the new version in
   sync.
4. In the final response, tell Bryan:
   - Obsidian must reload the plugin, or be restarted, before the fix applies.
   - A Ctrl+R that still fails after that is a different cause (a vimrc remap, or focus
     outside the editor), and he should report it.
