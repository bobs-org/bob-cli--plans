---
tier: epic
title: Bob Vim Yank Flash
goal: "Every Vim yank in Obsidian briefly highlights exactly the text it copied, like
  the SASE prompt input, with no change to editing behavior and nothing drawn when
  unsure.

  "
phases:
  - id: flash-plugin
    title: Build the plugin, unit tests and docs; deploy it disabled
    depends_on: []
    size: medium
    description:
      "flash-plugin: create the hand-edited bob-vim-yank-flash plugin (pushText wrapper,
      selection-text check, StateField and ViewPlugin decorations, inset-overlay
      styles), its node unit tests, and its docs page and inventory. Then add the vault
      .gitignore entry and sync it to the vault without enabling it."
  - id: obsidian-e2e
    title: Prove it in real Obsidian, then enable it in the vault
    depends_on:
      - flash-plugin
    size: medium
    description:
      "obsidian-e2e: add a CDP real-Obsidian e2e and its offline self-test, run it under
      xvfb on athena, fix what it finds, document the Obsidian upgrade check, and enable
      the plugin in the vault's community-plugins.json after a passing run."
proposed_by: bbugyi200.athena.1bd
proposed_with: claude/opus
create_time: 2026-11-11 10:46:17
status: wip
---

- **PROMPT:**
  [prompts/202611/vim_yank_flash.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/vim_yank_flash.md)

# Plan: Bob Vim Yank Flash, a brief highlight on exactly what a Vim yank copies

## Why and what exists today

In Obsidian's Vim mode a yank gives almost no feedback. `yy` leaves the text and cursor
where they were. Obsidian's patched yank prints "N lines yanked" only for more than two
lines. The vault vimrc sets `clipboard=unnamedplus`, so the extent of `yi(`, `yap`,
`y/foo<CR>` or `3yw` shows up only when Bryan pastes, often in another app. The SASE
prompt input (`sase-org/sase`) already flashes yanked text for 200 ms, and Bryan relies
on that.

The consolidated research report is
`research:202611/obsidian_vim_yank_range_flash/obsidian_vim_yank_range_flash.md`. Read
it with `sase artifact read`. Bryan accepted all of its requirement adjustments, R1–R11.
This plan implements its recommended design, a new settings-free `bob-vim-yank-flash`
plugin in `bob-plugins`, plus three refinements found while planning:

1. **Tinted task rows.** The vault snippet `.obsidian/snippets/task-statuses.css` sets
   `background` on
   `.markdown-source-view.mod-cm6 .HyperMD-task-line:is([data-task="/"], [data-task="*"], [data-task="?"], [data-task="-"])`.
   That selector is more specific than the research's
   `.cm-line.bob-yank-flash-line { background-color: … }`, so a background-only line
   flash would not show on In Progress, Next, Blocked and Cancelled rows. Those are the
   rows Bryan yanks most. Line flashes must paint above the line's own background (see
   Style).
2. **Wrapper bookkeeping.** A marker on `rc.pushText` alone can't tell whether another
   plugin has wrapped over ours. In that case the research sketch would wrap the same
   controller a second time. The plugin tracks the controllers it has hooked in each
   load instead.
3. **The vault `.gitignore` has to come first.** It allowlists `.obsidian/**/*.js` and
   `*.css`, and it lists each custom plugin folder explicitly. Without a new ignore
   entry, the first `bob plugins sync` would let `bob vault-sync` commit the plugin's
   files into the vault repository.

## Where the work lives

- **Code, tests and docs** live in the linked repository `bob-plugins`. Open it with
  your `/sase_repo` skill (`sase repo open bob-plugins -r "<why>"`) and read its
  `AGENTS.md` first. That file covers hand-edited versus fragment builds, the
  `bob plugins sync` duty, and the sync-refusal procedure.
- **No bob-cli code or docs change.** bob-cli's `docs/reference/plugins/` is a
  gitignored mount that `docs/_meta/mounts.yml` assembles from bob-plugins `docs/`, so
  the new page appears on the site automatically.
- **The live vault `~/bob/` gets two one-line config edits:** `.gitignore` in phase
  `flash-plugin` and `.obsidian/community-plugins.json` in phase `obsidian-e2e`.
  `bob vault-sync` commits vault edits, so never commit in the vault by hand. Never open
  the live vault in a test Obsidian instance.

## Behavior contract (both phases)

| #   | Rule                                                                                                                                                                                                                                                                                           | Research ref |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| C1  | Flash only the register writes made by the Vim **yank operator** (`pushText(registerName, operator, text, linewise, blockwise)` with `operator === "yank"`). That covers normal-mode motions, text objects, counts, `yy` and `Y`, and the visual modes `v…y`, `V…y`, `<C-v>…y` and visual `Y`. | R1, R2       |
| C2  | Flash for the unnamed, named (`"a`), append (`"A`) and `"+`/`"*` registers. **Never** flash for the black hole register `"_`. Upstream returns early for it, but the wrapper still runs, so the wrapper must filter it.                                                                        | R3           |
| C3  | **Never** flash for `d`, `c`, `x`, `s`, `D`, `C` or any other non-yank operator. Also never for ex `:yank`, Cmd/Ctrl+C, Bob Navigation Hotkeys' `yank-*` path commands, Reading view, or Bob Vim Surround's `ys`. Don't add a pending-operator preview or clipboard-success feedback.          | R2, R10      |
| C4  | The flashed range is exactly the live CM6 selection at the moment of the register write. Flash only when `ranges.map(r => state.sliceDoc(r.from, r.to)).join("\n") === String(text)`, and draw nothing otherwise. Never search the document and never sniff keys.                              | R6           |
| C5  | **Linewise** yanks highlight every covered line as a whole line, blank lines included. **Charwise** yanks get one mark per non-empty range. **Blockwise** yanks get one mark per row range. An empty charwise yank doesn't flash.                                                              | R4           |
| C6  | The flash lasts one 200 ms constant. A new yank replaces the flash immediately, a repeated identical yank flashes again, and a stale timer never clears a newer flash. The flash clears on any document change, on plugin unload, and when the editor is destroyed.                            | R7, R8       |
| C7  | Background-style paint only: no bold, no fade. The default colour is `rgba(var(--color-green-rgb), 0.35)`, overridable through `--bob-yank-flash-bg`. Never use `--text-highlight-bg`. The flash must stay visible on Bob's tinted task rows.                                                  | R5, R7       |
| C8  | Decorate only the lines inside `view.visibleRanges`, with no size cutoff.                                                                                                                                                                                                                      | R9           |
| C9  | Source mode and the Live Preview cursor line are the contract. Rendered widgets, folded ranges and replaced ranges are best-effort. Never unfold or scroll to show a flash.                                                                                                                    | R11          |
| C10 | **Behavior stays the same.** Register contents, clipboard delivery (vimrc-support), the cursor's end position, Obsidian's "N lines yanked" notice, `p`, `.`, macros and undo all behave exactly as with the plugin disabled. Any internal failure means no flash, never a broken yank.         | —            |
| C11 | No settings tab, no palette commands and no `data.json`. The plugin toggle is the off switch. Yanked text is never stored or logged.                                                                                                                                                           | R7           |

## Design (phase `flash-plugin` builds it; phase `obsidian-e2e` proves it)

### Files

- `plugins/bob-vim-yank-flash/manifest.json`:
  - `"id": "bob-vim-yank-flash"`, `"name": "Bob Vim Yank Flash"`, `"version": "0.1.0"`;
  - `"minAppVersion": "1.8.7"`, `"author": "Bryan"`, `"isDesktopOnly": false`;
  - `"description": "Briefly highlight the exact text a Vim yank copies in Obsidian's Vim mode."`.
- `plugins/bob-vim-yank-flash/main.js` is hand-edited like `bob-vim-surround`: no
  `src/fragments.json`, and it stays well under the 1,000-line fragment limit (expect
  about 300 lines). It requires `obsidian`, `@codemirror/state` and `@codemirror/view`
  the way `bob-ledger-tools/src/010-load-and-constants.js` does.
- `plugins/bob-vim-yank-flash/styles.css`.

### The hook

- **Resolve the controller:**
  `window.CodeMirrorAdapter?.Vim?.getRegisterController?.()`. Feature-detect a
  function-valued `pushText`, and skip silently if it's missing.
- **Install points:** `app.workspace.onLayoutReady` and every `active-leaf-change`. Both
  are registered through `registerEvent`.
  - The Vim adapter can be replaced on a vimrc reload (see
    `bob-navigation-hotkeys/src/620-plugin-dash-and-vim-jumps.js`).
  - Vim mode can also be turned on after load.
- **Bookkeeping invariant:** each plugin load keeps at most one _active_ wrapper per
  controller.
  - Keep per-load records `{rc, wrapped, original, hadOwn}`, keyed by controller
    identity.
  - Wrap any controller identity not yet seen.
  - Skip a controller this load already hooked, even when another plugin has since
    wrapped over ours.
  - Put a marker property on `wrapped` for diagnostics and e2e assertions.
- **The wrapper:**
  1. Call `original.apply(this, arguments)` first, and return its result unchanged.
  2. Then, only if the plugin is active, `operator === "yank"` and
     `registerName !== "_"`, call the flash routine inside `try/catch`. Swallow
     failures; a `console.debug` without yanked text is fine.
- **Unload:**
  1. Mark the plugin inactive.
  2. For each record whose `rc.pushText === wrapped`, restore the original. Use
     `delete rc.pushText` when the method wasn't an own property; in Obsidian it lives
     on `RegisterController.prototype`. Otherwise reassign the saved original.
  3. A wrapper that someone else has wrapped over stays inert inside their chain. Never
     restore a stale function over it.
- **Rejected:** `Vim.defineOperator('yank', …)` and `mapCommand`. They replace
  Obsidian's locally patched yank, including its ">2 lines" report, and unload can't
  restore it. If the real build ever fails the text check permanently, stop and report
  instead of switching.

### Choosing the editor and checking the range

- **Find the view:** `(window.activeDocument || document).activeElement`, then
  `.closest(".cm-editor")`, then `EditorView.findFromDOM(el)`. Any miss means no flash.
  This keeps pop-out windows working. Ex commands still resolve, because Vim's prompt
  panel sits inside the editor DOM.
- **Check the text:** build the selection text exactly as in C4 and compare it to the
  pushed text. The pushed text arrives _before_ `pushText` appends its linewise `"\n"`,
  so it equals the raw selection text. `yy` mid-document includes the trailing newline;
  `yy` on the last line at end of document does not.
- **Capture:** `{id, linewise, ranges: [{from, to}]}` only, where `id` is a
  per-plugin-load sequence number. Never keep the text.

### State, rendering and clearing

- **The effect:** a `StateEffect` `setFlash` carries either a new flash or
  `{id, clear: true}`. A clear applies only when its `id` matches the current flash;
  that is the CM6 version of SASE's generation counter.
- **The field:** a `StateField` holds the current flash or `null`, and drops it on
  `tr.docChanged`.
- **The view plugin:** a `ViewPlugin` builds the decorations. It rebuilds on
  `viewportChanged`, on `docChanged`, and whenever the field value changes.
  - **Linewise:**
    - The first line is `lineAt(from)`.
    - The last line is `lineAt(to)`, moved back one line when `to > from` and `to` is a
      line start.
    - An empty range, such as a blank last line at end of document, decorates
      `lineAt(from)`.
    - Decorate with `Decoration.line({class: "bob-yank-flash-line"})`, only for lines
      that intersect `view.visibleRanges`.
    - De-duplicate line starts across ranges.
  - **Charwise and blockwise:** `Decoration.mark({class: "bob-yank-flash"})` for each
    range with `to > from`.
  - Build with `Decoration.set(out, true)`.
- **Dispatch:** synchronously from the wrapper, as an effects-only transaction. It runs
  from Vim's keydown handling, not inside a CM6 update, and `try/catch` covers anything
  else.
- **Timer:** use the view's own window,
  `view.dom.ownerDocument.defaultView.setTimeout(…, 200)`, so pop-outs work. Skip the
  clear when `!view.dom.isConnected`, and dispatch the clear inside `try/catch`.
- **Registration:** `this.registerEditorExtension([field, viewPlugin])`.

### Style

```css
.bob-yank-flash {
  background-color: var(--bob-yank-flash-bg, rgba(var(--color-green-rgb), 0.35));
}

/* An inset overlay paints above the line's own background and below its text. The
   vault's task-status snippet sets a more specific `background` on tinted task rows,
   and a background-only line flash would be hidden there. */
.cm-line.bob-yank-flash-line {
  box-shadow: inset 0 0 0 100vmax
    var(--bob-yank-flash-bg, rgba(var(--color-green-rgb), 0.35));
}
```

Don't use `!important`. If the flash hides behind search matches or inline code, adjust
the extension's `Prec` rather than the CSS. Phase `obsidian-e2e` checks both.

### Test seam

Export `module.exports = BobVimYankFlashPlugin;` with a `module.exports.__test = { … }`
object, as `bob-vim-surround` does. Export the pure helpers: selection text, linewise
line starts, the field update, the decoration builder, and hook install and uninstall
with an injectable controller and plugin.

### Phase graph

```text
flash-plugin  ──►  obsidian-e2e
```

The phases run in sequence. The e2e drives the real plugin, and its passing run gates
enabling the plugin in the live vault. Until then the plugin is deployed but disabled,
so it does nothing.

## Phase flash-plugin: Build the plugin, unit tests and docs; deploy it disabled

Size: medium. Depends on: nothing.

1. **Orient.** Open `bob-plugins` with `/sase_repo` and read:
   - `AGENTS.md`;
   - `plugins/bob-vim-surround/main.js`, for the lifecycle, `onLayoutReady` and
     `active-leaf-change`, and focused-editor resolution;
   - `scripts/test-vim-surround.cjs`, for its `Module._load` stubs;
   - `plugins/bob-ledger-tools/src/010-load-and-constants.js`, for its CM6 requires.
2. **Plugin.** Create the three files exactly as the Design section describes.
3. **Unit tests: `scripts/test-vim-yank-flash.cjs`.** Use `node:test`. Stub `obsidian`,
   `@codemirror/state` and `@codemirror/view` through `Module._load` with small doubles:
   - `StateEffect.define()` gives `{of, is}`; `StateField.define(spec)` gives the spec;
   - `Decoration.line` and `Decoration.mark` give objects with `range()`;
     `Decoration.set` and `Decoration.none`;
   - `ViewPlugin.fromClass` and `EditorView.findFromDOM`;
   - a small Text double built from a string, with `length`, `lineAt` and `sliceString`.

   Cover these cases:
   - **Line arithmetic:**
     - `yy` mid-document, on a blank line, and on the last line with and without a
       trailing newline;
     - `3yy` across a blank line;
     - a linewise range larger than the visible ranges, including two disjoint visible
       ranges, where only visible lines are decorated;
     - duplicate and overlapping line ranges, which are de-duplicated.
   - **Charwise and blockwise:**
     - a multi-line charwise yank gives one mark;
     - a blockwise yank gives one mark per row;
     - empty blockwise rows are skipped;
     - an empty charwise yank doesn't dispatch.
   - **Selection check:**
     - the text matches the adapter's `getSelection()` semantics, including the
       blockwise `"\n"` join;
     - a mismatch, such as the wrong view or `:yank` with a collapsed selection, means
       no dispatch;
     - no focused `.cm-editor`, or a `findFromDOM` miss, means no dispatch.
   - **Effects:**
     - a stale clear keeps the newer flash;
     - a matching clear removes it;
     - `docChanged` drops it;
     - a set replaces it.
   - **Hook lifecycle:**
     - the hook wraps once per controller;
     - a new controller identity gets wrapped;
     - a controller this load already hooked is skipped even after something else has
       wrapped over ours;
     - a prototype method is restored by `delete`, leaving no own property;
     - an own-property original is reassigned;
     - a wrapper that has been wrapped over stays inert after unload: no flash, and the
       outer chain still reaches the original;
     - `"_` and non-yank operators are ignored;
     - flash exceptions are swallowed;
     - the original is called exactly once with the same `this` and arguments, and its
       return value passes through.
   - **Timer:** the clear is scheduled on the view's own window and skipped when the
     view is disconnected.

4. **Register the test** in the `package.json` `test` list, right after
   `scripts/test-vim-surround.cjs`.
5. **Docs.**
   - In `scripts/docs-inventory.mjs`, add `PURPOSES` and `PAGE_DESCRIPTIONS` entries for
     `bob-vim-yank-flash` first, so the scaffold is specific. Then run
     `npm run docs:gen`; it creates `docs/bob-vim-yank-flash.md` and updates
     `docs/index.md`.
   - Write handwritten sections outside the generated regions:
     - what flashes, as a table with examples;
     - what never flashes;
     - the colour override, with a CSS snippet example setting `--bob-yank-flash-bg`;
     - Live Preview and widgets (best-effort);
     - how it works, including why it wraps `pushText` rather than overriding the
       operator;
     - settings: none, and the toggle is the off switch;
     - a Related link to Bob Vim Surround (`ys` never flashes).
   - Add a `docs/nav.yml` entry after Bob Vim Surround. Add a reciprocal Related bullet
     to `docs/bob-vim-surround.md`.
   - Correct the plugin counts to "ten": `README.md` says "nine", and
     `docs/develop/architecture.md` and `docs/develop/deployment.md` still say "eight".
6. **Gates.** Run `npm run build:check`, `npm test` and `npm run validate`. Then run
   `npm run docs:check` with `BOB_CLI_DIR` pointing at your SASE host bob-cli checkout.
7. **Vault ignore, before any sync.** Add `.obsidian/plugins/bob-vim-yank-flash/` to
   `~/bob/.gitignore`. Put it in the "Bryan's custom Obsidian plugins" block, after
   `.obsidian/plugins/bob-vim-surround/`. Don't commit it; `bob vault-sync` does.
8. **Deploy disabled.**
   - Run
     `bob plugins sync --repo <bob-plugins checkout> --no-pull -p bob-vim-yank-flash`,
     following the `AGENTS.md` refusal procedure if it refuses.
   - Confirm that `bob plugins list -f json` shows the plugin synced at `0.1.0`, **not
     enabled**, with no `vault_ignore` warning.
   - Don't edit `community-plugins.json` in this phase.

**Acceptance:**

- the plugin, unit tests and docs land in `bob-plugins` with every gate green;
- the vault ignores the plugin folder;
- the plugin is deployed but disabled.

## Phase obsidian-e2e: Prove it in real Obsidian, then enable it in the vault

Size: medium. Depends on: `flash-plugin`.

1. **The e2e script: `scripts/obsidian-vim-yank-flash-e2e.cjs`.** Model it on
   `scripts/obsidian-redo-history-e2e.cjs`, and reuse
   `scripts/obsidian-startup-smoke-lib.cjs`.
   - Keep the same CLI: `-e/--executable`, `-o/--out`, `--only`, `-r/--report`,
     `-s/--self-test` and `-h`. Exit code 2 means unrun.
   - Use an isolated scratch vault and profile with an owned Obsidian process and CDP
     key events. Install every repo plugin, so Bob Vim Surround is present.
   - Set `app.json` to `vimMode: true`. Use Source mode by default, and open a Live
     Preview leaf for the cursor-line cases.
   - Add `scripts/test-obsidian-vim-yank-flash-e2e.cjs` as an offline `--self-test`
     wrapper (copy the redo wrapper) and add it to `npm test`.
2. **Observation.** A flash lives only 200 ms, so don't poll for it after the fact.
   1. Before sending keys, install a `MutationObserver` with `requestAnimationFrame`
      sampling on the focused editor's `.cm-content`.
   2. Record every `.bob-yank-flash` and `.bob-yank-flash-line` element seen, with its
      line number (through `EditorView.findFromDOM(…).posAtDOM`) and its text.
   3. Compare the record to the expected spans for the fixture.
   4. Cross-check against the unnamed register,
      `CodeMirrorAdapter.Vim.getRegisterController().getRegister('"')`.
   5. Wait at least 350 ms with no input and assert that no flash class remains.
   6. Assert that a flashed line's computed style really differs from an unflashed
      line's.
3. **Cases.** Use the research's verification table:
   - **Must flash:**
     - `yy`, `3yy`, `yw`, `y$`, `yiw`, `yi(`, `yap`, `yb`, `yG`;
     - `ygg` from deep in a note, decorating only rendered lines;
     - `Y`, `"ayy`, `"Ayy`, `Vjy`, `vey`, and `<C-v>jly` (two marks);
     - `yy` on a blank line, and a repeated identical `yy`.
   - **Must not flash:**
     - `dd`, `dw`, `cw<Esc>`, `x`;
     - `"_yy`, `:y<CR>`, `fy`, `ty`, `ry`, `"yp`;
     - `ysiw)`, with surround still working;
     - one `yank-*` path command, run through `app.commands.executeCommandById`.
   - **Unchanged behavior.** Compare against the same keys with the plugin disabled
     (`app.plugins.disablePlugin` / `enablePlugin`):
     - register text;
     - cursor position;
     - document text;
     - the "3 lines yanked" notice for `3yy`.
   - **Lifetime:**
     - a second yank within 200 ms;
     - no input after a yank, where the flash must still clear;
     - `yyp` and `yyP`;
     - a tab switch;
     - a split pane;
     - a pop-out window. This one is best-effort: if CDP can't drive it, record
       `skipped` with the reason, never a pass.
   - **Geometry:**
     - repeated `- [ ]` rows;
     - emoji and UTF-16 surrogate pairs;
     - tabs;
     - wrapped lines;
     - a folded section;
     - Live Preview paragraph, list, heading and link on the cursor line.
   - **Refinement checks:**
     - **Tinted task row:** the fixture enables a CSS snippet that reproduces
       `task-statuses.css`'s
       `.markdown-source-view.mod-cm6 .HyperMD-task-line:is([data-task="/"])`
       background. `yy` on a `- [/]` row must still show a visible line flash.
     - **Search match:** `yy` on a line with an active search match must show a visible
       flash.
     - **Unload:** disabling the plugin leaves `pushText` with no own property and no
       marker, and re-enabling flashes again.
   - **Screenshots:** save light and dark screenshots (`moonstone` and `obsidian`
     themes) of a linewise, a charwise, a blockwise and a tinted-row flash into the
     `--out` directory.
4. **Run it on athena,** which has the Obsidian 1.15.0 snap, `xvfb-run` and Node 22:
   `xvfb-run -a node scripts/obsidian-vim-yank-flash-e2e.cjs -e /snap/bin/obsidian -r /tmp/bob-yank-flash-e2e.json -o /tmp/bob-yank-flash-e2e-shots`.
   - Look at the screenshots yourself.
   - If any case fails, fix the plugin and its unit tests, bump the manifest patch
     version, rerun `npm run docs:gen`, and rerun until every non-skipped case passes.
   - **Stop condition:** the real build may never satisfy the text check, or `pushText`
     may never see yanks. If so, don't switch to a `defineOperator` override and don't
     enable the plugin. Report the evidence and file a task bead through your
     `/sase_new_task` skill.
5. **Docs.**
   - In `docs/develop/deployment.md`, add an "Obsidian upgrade check" paragraph: after
     any Obsidian upgrade, run this e2e. The plugin depends on undocumented Vim
     internals, and a broken hook must fail the check rather than silently stop
     flashing.
   - Mention the e2e in `docs/develop/validation.md`.
   - Add a "Verification" section with the command to `docs/bob-vim-yank-flash.md`.
6. **Gates.** Rerun `npm run build:check`, `npm test`, `npm run validate` and
   `npm run docs:check`.
7. **Enable, only after a passing real run.**
   - Append `"bob-vim-yank-flash"` to `~/bob/.obsidian/community-plugins.json`. Keep its
     existing two-space JSON array formatting, and don't commit; `bob vault-sync`
     propagates the change.
   - Run `bob plugins sync` again.
   - Confirm that `bob plugins list -f json` shows the plugin enabled and synced.
8. **Final report to Bryan:**
   - the e2e report path, the per-group verdicts, and anything skipped;
   - each running Obsidian must restart to load the plugin;
   - the Mac first needs `bob plugins sync` (for example through `just install-all`),
     because custom plugin folders are gitignored. Toggling plugins in an
     already-running Obsidian before that restart may rewrite `community-plugins.json`
     without the new entry;
   - a two-minute manual pass on the Mac with his own theme, in light and dark: `yy` on
     a `- [/]` row, `vey`, `<C-v>jly` and `"_yy`.

**Acceptance:**

- the real-Obsidian e2e passes on athena, with skips listed and justified;
- its offline self-test runs in `npm test`;
- the upgrade-check docs have landed;
- the plugin is enabled in the vault.

## Non-goals

- ex `:yank`, Cmd/Ctrl+C, Bob's path-yank commands, Reading view, a pending-operator
  (`y/foo`) preview, and clipboard-success feedback (R10);
- a settings tab, per-mode options, a configurable duration, or a fade (R7);
- folding this into `bob-vim-surround`, because the separate plugin keeps its own kill
  switch;
- proposing an upstream `vim-yank` event to `replit/codemirror-vim`;
- mobile verification.
