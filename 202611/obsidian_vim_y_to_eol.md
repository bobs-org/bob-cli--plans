---
tier: tale
title: Normal-mode Y yanks to the end of the line in Obsidian
goal:
  In Obsidian's Vim normal mode, Y yanks from the cursor to the end of the line (y$)
  instead of the whole line.
size: small
decisions:
  flash_doc:
    ask:
      Also note in the Bob Vim Yank Flash doc that the vault maps normal-mode `Y` to
      `y$`?
    default: true
    why:
      Keeps Bryan's own plugin doc true to his editor; the nav-hotkeys doc notes vimrc
      bindings too.
    answer: true
proposed_by: bbugyi200.athena.1bh
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 12:44:04
status: wip
---

# Normal-Mode `Y` Yanks To The End Of The Line In Obsidian

## Goal

In Obsidian's Vim normal mode, `Y` should yank from the cursor to the end of the line,
exactly like `y$`. Today it yanks the whole line, like `yy`. The new behavior matches
`D` / `C` and Neovim's default `Y`.

| Keys                       | After this change                                                |
| -------------------------- | ---------------------------------------------------------------- |
| `Y`                        | Yanks cursor → end of line (charwise), into the system clipboard |
| `3Y`                       | Same as `3y$`: to the end of the line two lines down             |
| `"aY`                      | Same as `"ay$`                                                   |
| `yy`, visual `Y`, `D`, `C` | Unchanged (whole line / linewise / as today)                     |

## Findings (Verified At Planning Time)

- **Where the binding belongs.** Bryan's custom Vim bindings live in the vault vimrc
  `~/bob/obsidian_vimrc.md`. It is loaded by the community plugin Vimrc Support
  (`obsidian-vimrc-support` 0.10.2; `vimrcFileName` is set in
  `~/bob/.obsidian/plugins/obsidian-vimrc-support/data.json`). The file already has
  `set clipboard=unnamedplus`. Vimrc Support copies the Vim `yank` register to the
  system clipboard whatever keys filled it, so a mapped `Y` reaches the clipboard too.
- **No conflicting `Y`.** Neither the vimrc nor any bob-plugins plugin maps `Y`. The
  only `vim.mapCommand` keys are `\p \P \o \O <CR> <C-]> o O #`, and the only
  `vim.unmap` keys are `<C-o>` / `<C-i>`. The `Y` keys in the Command Mode pager, the
  delete guard, and the image-zoom viewer are modal key handlers outside CodeMirror Vim.
  They are unaffected.
- **Why `nnoremap Y y$` works.** This was checked against the CodeMirror Vim bundled in
  athena's Obsidian 1.15.0:
  - The built-in normal-mode `Y` is
    `{ keys: 'Y', type: 'operatorMotion', operator: 'yank', motion: 'expandToLine', motionArgs: { linewise: true }, context: 'normal' }`.
    A separate visual-context entry makes visual `Y` linewise.
  - `nnoremap` builds a `keyToKey` mapping with `noremap: true` and `context: 'normal'`.
    `_mapCommand` `unshift`s it to the front of `defaultKeymap`, so it beats the
    built-in entry. The vimrc's `nnoremap -` already overrides the built-in `-` motion
    this way.
  - `handleKeyNonInsertMode` pushes a count into the input state before the mapping
    replays `y$`, so `3Y` becomes `y3$`. A pending `"a` register also survives, so `"aY`
    works.
- **Yank flash still works.** Bob Vim Yank Flash wraps `pushText` and flashes any `yank`
  operator write, and `y$` is already one of its must-flash e2e cases. So the mapped `Y`
  flashes the charwise cursor → end-of-line range.
  - Its e2e (`scripts/obsidian-vim-yank-flash-e2e.cjs` in bob-plugins) builds a scratch
    vault holding only bob-plugins plugins, without Vimrc Support. Its `Y` → whole-line
    case therefore still tests stock Vim and stays correct. **Do not change the e2e.**
- **bob-cli needs no change.** No bob-cli page documents the editor `Y`
  (`docs/guides/run-bob-from-obsidian.md` mentions only the Command Mode pager's `Y`).
  `docs/reference/plugins/` is a gitignored mount of the bob-plugins docs. Re-check with
  `grep -rn '`Y`' docs` before finishing.

## Change 1: Vault Vimrc (`~/bob/obsidian_vimrc.md`)

Follow `~/bob/AGENTS.md`:

- Run `git -C ~/bob status --short` before editing.
- Never stage, revert, or commit unrelated pre-existing changes.
- Never use `/sase_git_commit` in the vault.

Insert this block directly after the `set clipboard=unnamedplus` line, with one blank
line before it and one after it:

```vim
" Y yanks from the cursor to the end of the line, like D and C (Neovim's default).
" yy still yanks the whole line, and visual-mode Y stays linewise.
nnoremap Y y$
```

Rules:

- Use `nnoremap`, not `nmap`, `map`, or `noremap`:
  - Normal mode only, so visual `Y` stays linewise and operator-pending mode is
    untouched.
  - Non-recursive, so a future `y` or `$` remap cannot hijack it.
- Add no `vnoremap` / `xnoremap Y`, and touch no other line in the file.
- Commit only this file in the vault:
  `git -C ~/bob add obsidian_vimrc.md && git -C ~/bob commit -m 'feat(vimrc): make normal-mode Y yank to end of line'`.
  - Add the commit attribution trailer your harness requires.
  - If `bob vault-sync` auto-commits the edit first (subject
    `vault(<host>): 1 file - obsidian_vimrc.md`), do not make an empty commit; say so in
    the final response.
  - Leave pushing to `bob vault-sync`.

## Change 2: Yank Flash Doc Row (bob-plugins)

> [!decision] flash_doc Do this section only when `flash_doc` is yes. When it is no,
> skip it entirely and do not open bob-plugins.

1. Open the linked repo with `/sase_repo` (`sase repo open bob-plugins -r "<reason>"`),
   read its `AGENTS.md`, and use the printed path for every read and write.
2. In `docs/bob-vim-yank-flash.md`, table `## What flashes`, replace the row

   ```
   | `Y` | The whole current line |
   ```

   with

   ```
   | `Y` | The whole current line in stock Vim. Bob's vault vimrc maps normal-mode `Y` to `y$`, so there it flashes like `y$` |
   ```

3. Change nothing else: no plugin code, no manifest version, and not the e2e. The visual
   `Y` row stays as it is.
4. Run `npm run docs:check` from the bob-plugins checkout. If it needs the bob-cli
   checkout, pass `BOB_CLI_DIR=<bob-cli checkout>`.
5. Run `bob plugins sync`, as bob-plugins' `AGENTS.md` requires after any change. If it
   refuses an overwrite, follow that `AGENTS.md` (never pass `-F` on your own).
6. Commit the doc change in bob-plugins. This is an opened-repo obligation for
   `/sase_final`.

## Verification

1. `grep -nE '^[a-z]*map Y ' ~/bob/obsidian_vimrc.md` prints exactly one line:
   `nnoremap Y y$`.
2. `git -C ~/bob show --stat HEAD` (or the vault-sync commit that carries the edit)
   touches only `obsidian_vimrc.md`. Its diff is the three added lines plus their blank
   separator.
3. If `flash_doc` is yes, `npm run docs:check` passes in bob-plugins, and
   `git diff HEAD~1 --stat` there touches only `docs/bob-vim-yank-flash.md`.
4. No automated harness here loads Vimrc Support with the vault vimrc. **Do not claim a
   live Obsidian test.** Instead, give Bryan this manual check:
   - Reload Obsidian on each device after the vault syncs: command palette → "Reload app
     without saving", or disable and re-enable Vimrc Support.
   - On a line `alpha beta` with the cursor on `b`, `Y` then `p` pastes `beta`, and the
     system clipboard holds `beta`.
   - `yy` still yanks the whole line, and `V` then `Y` still yanks whole lines.

## Final Response

Tell Bryan:

- `Y` now runs `y$` in normal mode. Restate the behavior table from Goal.
- athena, apollo, and the Mac get the vimrc through `bob vault-sync`, but each needs an
  Obsidian reload, because Vimrc Support reads the file only when it loads.
- Whether the vault commit was yours or vault-sync's, and (when `flash_doc` is yes) the
  bob-plugins commit.
- The mapping was verified against Obsidian's bundled CodeMirror Vim source but not
  exercised in a live Obsidian window. Give him the manual check above.

## Non-Goals

- No change to `yy`, visual `Y`, `D`, `C`, or the `clipboard` setting.
- No `Y` binding in plugin code (`vim.mapCommand`). The vault vimrc is where Bryan's
  custom Vim bindings live.
- No new or changed e2e, plugin version bump, bob-cli code or doc change, Mac Capture
  change, or memory-note change.
