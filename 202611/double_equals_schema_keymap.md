---
tier: tale
title: "`==` opens the schema editor on album notes and bank.md"
goal:
  In Vim normal mode, `==` on an album note or bank.md opens the same card its ⊞ Schema
  / ⊞ Edit button opens.
size: medium
decisions:
  records:
    ask: Should `==` on an album's record note also open its parent album's Schema Card?
    choices:
      album_only: Only album notes and bank.md react; a record keeps Vim's line reindent
      records_too: Records open their parent album's Schema Card, like Edit album schema
    default: album_only
    why: The ⊞ Schema button you asked to mirror appears only on album notes
    answer: album_only
proposed_by: bbugyi200.apollo.ek
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 10:04:21
status: wip
---

# Plan: `==` opens the schema editor on album notes and bank.md

## Goal

In Obsidian, with Vim in normal mode in a Live Preview editor, typing `==` should:

- on an **album note**, do exactly what clicking the header's **⊞ Schema** button does
  (open the album Schema Card);
- on the **bank area note** (`bank.md`), do exactly what clicking the Setup header's **⊞
  Edit** button does (open the Bank setup card).

Everywhere else `==` keeps Vim's native behaviour (reindent the current line).

All code lives in the linked **`bob-plugins`** repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md` before editing.

## What exists today (verified)

- Album header button: `plugins/bob-ledger-tools/src/241v-plugin-album-header.js`,
  `albumHeaderOptions()` → `handlers.openSchemaCard` calls
  `this.openAlbumSchemaEntry({ filePath: sourcePath })`.
- `openAlbumSchemaEntry(options)` is in `src/242d-plugin-album-entrypoints.js`. It owns
  the desktop gate, the album/record resolver (`albumSchemaEntryTargetForFile`,
  `albumSchemaEntryKindOf`), and every teaching Notice ("not in the album report — run
  Sync albums", "needs desktop Obsidian", …). The **Edit album schema** palette command
  (`edit-album-schema`) and the bob-navigation-hotkeys Ctrl+Shift+P album fallback call
  the same function.
- Bank Setup **⊞ Edit**: `src/248c-plugin-bank-setup-hosts.js`, `bankSetupOptions()` →
  `handlers.onEdit` calls `this.openBankSetupCard({ sourcePath })`. `openBankSetupCard`
  is in `src/248l-plugin-bank-setup-card-entrypoints.js`. The **Edit bank setup**
  palette command (`edit-bank-setup`) also calls it. The hub note is
  `BANK_SETUP_HUB_PATH = "bank.md"`, and `bankSetupIsHubPath(path)` is in
  `src/248a-plugin-bank-setup-model.js`.
- **Why a vimrc mapping cannot do this.** Vault keys such as `]s` and `Z` are
  `exmap … obcommand` plus `nnoremap` lines in the vault's `obsidian_vimrc.md`. That
  route cannot reach `==`. In CodeMirror Vim (`@replit/codemirror-vim`,
  `commandDispatcher.matchCommand`), the first _full_ key match wins at once. `=` is a
  complete built-in operator
  (`{ keys: '=', type: 'operator', operator: 'indentAuto' }`), so the first `=` always
  becomes a pending operator. The second `=` then hits `processOperator`'s
  doubled-operator branch, which reindents the line linewise. A `nnoremap ==` is only
  ever a partial match and never fires. bob-vim-surround hits the same wall with `ys`,
  `cs`, and `ds` (see the comment at the top of `plugins/bob-vim-surround/main.js`). It
  solves it with a capture-phase `keydown` listener that runs before CodeMirror Vim,
  consumes the event, and clears Vim's pending operator with an injected `<Esc>`
  (`Vim.handleKey(cm, "<Esc>", "mapping")`). This plan uses that proven pattern.
- The repo already reads CodeMirror Vim `inputState` internals: `operator`, `keyBuffer`,
  `prefixRepeat`, `motionRepeat`. Examples:
  `bob-navigation-hotkeys/src/687-plugin-window-prefix.js` and
  `src/090-local-task-ids.js`. After the first `=`, Vim's state is:
  `inputState.operator === "indentAuto"`, an empty `keyBuffer`, and no
  `operatorShortcut`, because that field is only set for operators with keys longer than
  one character.

## Behaviour contract

`==` acts (opens a card) only when **all** of these hold for the second `=` keydown:

1. `event.key === "="`, with no Ctrl, Alt, or Meta. Shift is ignored because some
   layouts shift `=`. The event is not `repeat`, not `isComposing`, and not already
   `defaultPrevented`.
2. The event target is inside the `.cm-editor` of the active `MarkdownView`.
3. The view is in **Live Preview**: the editor state has `editorLivePreviewField` true,
   with a fallback to `view.getState().source === false`. That is where the header
   buttons render in the editor. Source mode, including the album/bank YAML excursion,
   keeps native `==`.
4. CodeMirror Vim for that editor shows **exactly a lone pending `=`**:
   - not in insert, visual, or replace mode;
   - `inputState.operator === "indentAuto"`;
   - `keyBuffer` is empty (accept an array or a string);
   - no count digits in `prefixRepeat` or `motionRepeat`;
   - no `registerName`.

   If `cm.state.vim` or `inputState` cannot be read, do not act.

5. The active file is an eligible note:
   - `bankSetupIsHubPath(path)` → **bank** target;
   - frontmatter kind is `"album"` via `albumSchemaEntryKindOf(frontmatter)` → **album**
     target. Gate on the note's own `type`, not on report presence, so an album note
     whose report is missing still gets the entry point's "run Sync albums" Notice
     instead of a silent reindent.
   - Record notes (kind `"record"`) follow the `records` decision below.
6. `window.CodeMirrorAdapter.Vim.handleKey` exists. Check this **before** consuming
   anything.

> [!decision] records = album_only Record notes are not eligible; `==` on a record keeps
> Vim's reindent.

> [!decision] records = records_too Kind `"record"` is an **album** target. Call
> `openAlbumSchemaEntry` with the record's own path; it resolves the parent album and
> shows its teaching Notice for a missing, cyclic, ambiguous, or non-album parent.

When it acts, in this order:

- Consume the event: `preventDefault`, `stopPropagation`, `stopImmediatePropagation`.
- Inject `<Esc>` with `Vim.handleKey(cm, "<Esc>", "mapping")` to clear the pending `=`
  operator. The document is unchanged: no reindent, no undo entry, the cursor stays put.
- Open the card through the same call its button makes:
  - album → `void this.openAlbumSchemaEntry({ filePath })`;
  - bank → `void this.openBankSetupCard({ sourcePath: filePath })`.

  Wrap the call in try/catch so a failure never throws out of the listener. Desktop,
  mobile, missing-report, and loading behaviour all stay owned by those two functions.

Everything else falls through untouched, with no consume and no `<Esc>`:

- counted `N==` and register-prefixed `"x==`;
- other operators (`d=`), and a first `=` with nothing pending;
- visual-mode `=`, insert mode, Source mode, and Reading view (which has no Vim);
- modal inputs;
- every non-eligible note.

## Implementation (bob-plugins, plugin `bob-ledger-tools`)

**Where it lives.** It goes in bob-ledger-tools, not bob-navigation-hotkeys.
bob-ledger-tools owns both cards, both entry functions, both resolvers, and both palette
commands, and it already registers Vim mappings (`src/300-plugin-vim-and-snippets.js`).
So the key is a third entry point beside the button and the palette command, needs no
cross-plugin API, and ships in one deploy. It needs a few small Vim-state readers of its
own. Model them on `resolveVimCodeMirror` / `getCurrentVimMode` in
`bob-navigation-hotkeys/src/610-plugin-jumps-and-counted-keys.js`, but do not import
across plugins.

bob-ledger-tools uses the committed source build. Edit `src/` fragments only, run
`npm run build`, and never hand-edit `main.js`. Each fragment must stay ≤ 1000 lines.

1. **New fragment** `plugins/bob-ledger-tools/src/248m-plugin-schema-double-equals.js`
   (about 150–250 lines). Start it with a header comment that explains the vimrc
   limitation in two lines.
   - Pure `schemaDoubleEqualsVimSnapshot(cm)` reads `cm.state.vim` into a plain object:
     `{ readable, insertMode, visualMode, replaceMode, operator, keyBufferEmpty, countPending, registerPending }`.
   - Pure `schemaDoubleEqualsGate(input)` → `{ act: boolean, reason: string }`. Its
     input is a plain snapshot: key and modifier flags, `repeat`, `isComposing`,
     `defaultPrevented`, `inEditor`, `livePreview`, the Vim snapshot, `target`
     (`"album" | "bank" | null`), and `canHandleKey`. It gives one distinct `reason` per
     fall-through rule above (for example `"not-equals"`, `"modified"`, `"repeat"`,
     `"outside-editor"`, `"not-live-preview"`, `"no-vim"`, `"not-normal"`,
     `"no-pending-equals"`, `"counted"`, `"register"`, `"not-schema-note"`,
     `"no-handle-key"`) and `"schema-key"` when it acts.
   - Pure `schemaDoubleEqualsTargetKind({ path, frontmatter })` returns `"bank"`,
     `"album"`, or `null`. It applies rule 5 with the `records` decision.
   - Mixin class `BobLedgerToolsSchemaDoubleEqualsMixin`:
     - `setupSchemaDoubleEquals()`: idempotent. Adds one capture-phase `keydown`
       listener on `window` and on `document` (when distinct), each paired with
       `this.register(() => removeEventListener(...))`, and a `WeakSet` of handled
       events so the double dispatch acts once. This is the bob-vim-surround /
       `registerOpenTaskJumpInputListeners` pattern.
     - `teardownSchemaDoubleEquals()`: marks the bridge inactive so late events do
       nothing.
     - `handleSchemaDoubleEqualsKeydown(event)`: resolves the active `MarkdownView` the
       same way `albumSchemaEntryActiveFilePath` does, and gets the CM5-compat `cm` from
       `view.editor.cm.cm` or `view.editMode.editor.cm.cm`. It reads frontmatter from
       `metadataCache.getFileCache(file)`, builds the gate input, and acts as described
       in the contract.
2. Add `"248m-plugin-schema-double-equals.js"` to `src/fragments.json` right after
   `"248l-plugin-bank-setup-card-entrypoints.js"`.
3. Add `BobLedgerToolsSchemaDoubleEqualsMixin` to the `installBobLedgerToolsMixins` list
   in `src/310-install-methods.js`.
4. Wire the lifecycle:
   - `src/170-plugin-lifecycle.js`: call `this.setupSchemaDoubleEquals?.();` beside
     `this.setupBankSetupCard()`.
   - `src/170-plugin-lifecycle-teardown.js`: call `this.teardownSchemaDoubleEquals?.();`
     inside the existing best-effort try block next to `teardownBankSetupCard()`.
5. Test exports: add a new last fragment, `src/369-exports-schema-double-equals.js`.
   Follow the `368-exports-schema-arrange.js` try/`Object.assign` pattern to expose the
   three pure helpers on `module.exports.helpers`, and append it to `fragments.json`.
6. Bump the **minor** version in `plugins/bob-ledger-tools/manifest.json`; it is a
   feature. Read the current value at implementation time rather than assuming
   `1.127.4`.

## Tests

Add `scripts/test-ledger-tools-schema-double-equals.cjs` and list it in `package.json`
`test:unit`, next to `scripts/test-ledger-tools-album-entrypoints.cjs`. Reuse that
file's `Module._load` stubs. Add a `MarkdownView` class to the stub and expose
`editorLivePreviewField`.

- **Gate table:** one assertion per contract rule and `reason`. Cover:
  - album and bank act;
  - record follows the decision;
  - area, project, and daily notes fall through;
  - counted (`prefixRepeat: ["3"]` and `motionRepeat`), register, `operator: "delete"`,
    and no pending operator;
  - insert and visual modes, Source mode, and outside-editor targets;
  - Ctrl, Alt, and Meta; repeat, composing, and `defaultPrevented`;
  - missing `handleKey`;
  - Shift with `key "="` still acts.
- **Integration with fakes:** use a fake `window` and `document` that record
  capture-phase listeners, a fake `CodeMirrorAdapter.Vim` that records `handleKey`
  calls, a fake workspace whose active `MarkdownView` holds `editor.cm.cm` with
  `state.vim.inputState`, and fake `metadataCache`/`vault`. Assert:
  - on an album note: the event is consumed, `<Esc>` is injected exactly once,
    `openAlbumSchemaEntry` is called once with `{ filePath: "<album>.md" }`, and the
    editor content is unchanged;
  - on `bank.md`: `openBankSetupCard({ sourcePath: "bank.md" })` is called;
  - on an ordinary note: nothing is consumed or injected, and nothing opens;
  - window plus document dispatch of the same event acts once;
  - a throwing entry function does not escape the listener;
  - after unload, the registered cleanups remove both listeners.

## Docs (plugin reference pages in bob-plugins `docs/`)

- `docs/albums.md`, in the "Open the card from the header's **⊞ Schema** …" paragraph:
  add "`==` in Vim normal mode in Live Preview on the album note". Note that a counted
  `N==`, Source mode, and other notes keep Vim's reindent. Under `records_too`, add that
  a record note opens its parent album's card.
- `docs/bob-ledger-tools.md`, in the Bank Setup line (currently "**⊞ Edit**, **Edit bank
  setup**, or Command Mode opens the card"): add "`==` (Vim normal mode, Live Preview,
  on `bank.md`)".
- `docs/bob-navigation-hotkeys.md`, in the "Editor and Vim handlers" bullet: add one
  sentence. Bob Ledger Tools owns normal-mode `==` on album notes and `bank.md`, because
  a vimrc `nnoremap ==` cannot beat Vim's built-in `=` operator.
- Run `npm run docs:gen` to refresh the generated facts and version in
  `docs/bob-ledger-tools.md` / `docs/index.md`. No command is added, so the command
  table does not change.

## Validate and deploy

From the bob-plugins checkout:

```bash
npm run build
npm run build:check
npm run docs:gen
npm run docs:inventory:check
npm test
npm run validate
BOB_CLI_DIR=<path of the bob-cli checkout you are running in> npm run docs:check
```

Then run `bob plugins sync`. If it refuses to overwrite another checkout's deploy,
follow the bob-plugins `AGENTS.md` procedure exactly: never pass `-F` on your own. A
real-Obsidian run is not available on apollo. The unit tests cover the gate and wiring,
and the capture-before-Vim precedence is the same mechanism bob-vim-surround's
`ys`/`cs`/`ds` bridge depends on in production. In the final report, list the manual
acceptance checks below for Bryan.

## Manual acceptance (Bryan, live vault)

1. Open an album note (for example `sase_blog_zettel`) in Live Preview, in normal mode,
   and type `==`. The Schema Card opens, the cursor line is unchanged, and `u` has
   nothing new to undo.
2. Type `==` on `bank.md`. The Bank setup card opens.
3. Type `==` in a daily or project note. The line reindents as before.
4. Type `3==` on the album note. Three lines reindent; no card opens.
5. In Source mode on the album note, `==` reindents.

## Non-goals

- No change to the vault's `obsidian_vimrc.md`, no new palette command, and no new
  default hotkey.
- No Reading-view `==`: Reading view has no Vim layer.
- No `==` on bank month or day notes, or on other area notes.
- Popout windows behave like bob-vim-surround (main window listeners only).
