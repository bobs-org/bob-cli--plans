---
tier: tale
title: Bare - and + replace the =- and =+ property-card chords
goal:
  In Live Preview Vim normal mode, bare - toggles the card Properties drawer and bare +
  opens the Property Editor; =- and =+ retire, == is unchanged, and the vimrc's -
  daily-note map is gone.
size: medium
proposed_by: bbugyi200.apollo.ft
proposed_with: claude/opus
create_time: 2026-11-24 06:59:54
status: wip
---

# Plan: Bare `-` and `+` replace the `=-` and `=+` property-card chords

## Goal

In Obsidian Live Preview, Vim normal mode, Bryan presses one key instead of a two-key
chord:

- `-` toggles the mounted property card's read-only **Properties** drawer. It replaces
  `=-`.
- `+` opens the **Property Editor** on the active Markdown note. It replaces `=+`.
- `==` is unchanged. It still opens the species card editor.
- `=-` and `=+` are retired. After a pending `=`, `-` and `+` go back to Vim, as `=`
  followed by any other motion does.
- The vault vimrc mapping `nnoremap - :bob_daily<CR>` is removed. Bare `-` no longer
  opens today's daily note. Bryan said he doesn't use it.

Context: epic `bob-cli-pf` (card-owned frontmatter) introduced the `=` chord family in
§4 of `plan:202611/card_owned_frontmatter.md`. Epic `bob-cli-pl` (area/project cards)
builds on `=+`. Both epics are still in progress, so this change must leave them a clear
note (step 6).

## Where things are today (bob-plugins)

Open bob-plugins with `/sase_repo` (it is a linked repo) and read its `AGENTS.md`.
Rebase onto the latest `origin/master` first: `bob-cli-pf.6` (takeover) is landing in
the same files this week.

- `plugins/bob-ledger-tools/src/248m-plugin-schema-double-equals.js` has the
  capture-phase `keydown` gate on `window` and `document`.
  - `handleSchemaDoubleEqualsKeydown` builds an arm for `=`, `-`, or `+`.
  - `propertyCardSecondKeyCommonGate` requires a clean pending `indentAuto` (`=`)
    operator, plus these checks: normal mode, Live Preview, readable Vim, an empty key
    buffer, no count, no register, no Ctrl/Alt/Meta, no repeat, no IME, and no
    `defaultPrevented`. Shift is allowed.
  - When the gate acts, it consumes the event, sends
    `Vim.handleKey(cm, "<Esc>", "mapping")` to cancel the pending `=`, and runs the arm.
  - The `-` arm acts only when `propertyCardCanToggleProperties(view)` finds a live foot
    in this view.
  - The `+` arm comes from `registerPropertyCardSecondKeyArm`. It is registered by
    `298m-plugin-property-editor-entrypoints.js`, and `canAct` is true for any `.md`
    path.
- Hint text: in `240o-plugin-property-card-foot-dom.js`, the disclosure `title` is
  `"Toggle properties =-"` and the drawer action is `"✎ Edit all =+"`.
- Comments: `240p-plugin-property-card-foot-state.js:190` ("Future editor hook for
  `=+`") and the header comments of `248m` and `298m`.
- Tests:
  - `scripts/test-ledger-tools-schema-double-equals.cjs`: the arm-gate table test, the
    "=- toggles …" and "=- falls through …" tests, and the "+ arm" test.
  - `scripts/test-ledger-tools-property-card-foot.cjs:356`.
  - `scripts/test-ledger-tools-property-editor-entrypoints.cjs:431`.
- Real-Obsidian e2e:
  - `scripts/obsidian-card-foot-e2e.cjs`: `dispatchFootKeys` and the `=`, `-`
    minus-toggle loop near line 715.
  - `scripts/obsidian-property-editor-e2e-surfaces.cjs` near lines 334–345: the `=+` and
    Shift+`=` chord checks, and the "Edit all =+" check name.
- Docs that name the chords:
  - `docs/record-card.md`, `docs/albums.md`, `docs/intention-card.md`,
    `docs/bob-ledger-tools.md`, `docs/property-editor.md`.
  - `docs/bob-navigation-hotkeys.md` (around line 696).
  - `docs/develop/architecture.md` (the arm-table section around lines 124–151).
  - `docs/develop/validation.md` (card-foot and property-editor sections).
- bob-cli's `docs/reference/plugins/*` is a gitignored mount of these docs, so bob-cli
  needs no doc edit. No memory note names `=-` or `=+` today.

This list is a snapshot. At implementation time, re-run
`git grep -n -E '=-|=\+' -- plugins/*/src scripts docs` and filter out the JavaScript
operators (`+=`, `-=`, `==`, `!=-`, numeric `=-1`). Any phase that landed in the
meantime may have added more mentions.

## Design

### 1. Bare-key arms in the existing gate (no vimrc mapping)

The new keys stay in the plugin gate rather than in `obsidian_vimrc.md`. A vimrc map
would fire everywhere, so it could not fall through to Vim's own `-`/`+` motion when
there is no card or no registered editor. It would also bypass the tested guards.

In `248m`, split dispatch by key:

- **`=`:** exactly today's path. Keep the pending-`indentAuto` gate, the species
  registry, and the `<Esc>` injection. All existing `==` tests stay green unchanged.
- **`-` and `+`:** a new `propertyCardBareKeyGate(input)`. It acts only when every one
  of these holds:
  - no Ctrl, Alt, or Meta (Shift is allowed, because `+` is Shift+`=` on US layouts; an
    unshifted `+` from other layouts and the numpad also acts);
  - not `repeat`, not `isComposing`, not `defaultPrevented`;
  - **the event targets the main editor's own content element**, as described below;
  - Live Preview;
  - readable Vim in normal mode (not insert, visual, or replace mode);
  - **no pending operator** (`operator === null`). This covers `d-`, `y+`, `c-`, and a
    pending `=`, which is how `=-` and `=+` retire;
  - an empty key buffer, so `g+`, `g-`, `f-`, `t+`, `r-`, `m-`, `` `- ``, `[`/`]`
    prefixes, and `"-` keep Vim's meaning;
  - no count (`3-` and `3+` stay Vim motions) and no register pending;
  - the arm exists and `canAct(context)` returns true.

  When the gate acts, consume the event the same way as today (`preventDefault`,
  `stopPropagation`, `stopImmediatePropagation`, and the shared `WeakSet` dedupe across
  window and document) and run the arm.
  - **Do not** inject `<Esc>`: nothing is pending.
  - Do not require `canHandleKey`.
  - Return distinct `reason` strings, such as `operator-pending`, `not-content`, and
    `unregistered-arm`, so the tests can name each fall-through.

**Content-target guard.** The current `schemaDoubleEqualsEventInEditor` check is
`closest(".cm-editor")`. That is too loose for a bare key. Vim's `/`/`?` search prompt
panel, the native Properties inputs, the inline title, and widget inputs all live inside
`.cm-editor`. Vim also still reports normal mode while focus is in them. Typing `-` or
`+` into any of those inputs must type the character.

- Require `event.target` to be the resolved editor's `contentDOM`
  (`view.editor.cm.contentDOM`, falling back to `cm.cm6.contentDOM`).
- If real Obsidian dispatches from a descendant, accept a descendant only when it is not
  an `input`, `textarea`, or `select` and its closest `[contenteditable]` is that same
  `contentDOM`. Nested table-cell editors have their own `contentDOM`, so they fall
  through.
- Leave the `=` path's existing check alone.

**Arm table:**

| Key | Acts when                                                          | Otherwise        |
| --- | ------------------------------------------------------------------ | ---------------- |
| `-` | `propertyCardCanToggleProperties(view)`: a live foot in this view  | Vim's `-` motion |
| `+` | a `+` arm is registered (Property Editor) and the note is Markdown | Vim's `+` motion |

**Public API.** Keep `registerPropertyCardSecondKeyArm` /
`unregisterPropertyCardSecondKeyArm` and the `"+"` key unchanged. `298m` and the
in-flight epics call them. Update only their comments and docs: the `+` arm now runs on
a bare normal-mode `+`, not after `=`. The arm context keeps the same fields.

Keep `248m` under 1000 lines. If the bare gate pushes it over, move the pure gate
helpers into a new adjacent fragment and register it in
`plugins/bob-ledger-tools/src/fragments.json` before `248m`.

### 2. Hint text

- Disclosure tooltip: `"Toggle properties =-"` becomes `"Toggle properties (-)"`.
- Drawer action: `"✎ Edit all =+"` becomes `"✎ Edit all (+)"`. Its `aria-label` stays as
  is.
- Update the comment in `240p` (`=+` becomes `+`) and the header comments in `248m` and
  `298m`.

### 3. Vault vimrc

Edit `~/bob/obsidian_vimrc.md` in place. It lives in the vault and is the only source of
truth.

- Delete `nnoremap - :bob_daily<CR>`.
- Keep `exmap bob_daily …`, so `:bob_daily` still works from Command Mode, and keep the
  `open-today-daily-note` command and its Command Mode row.
- Next to the existing `<C-w>` comment, add one comment line:
  `" - (Properties drawer) and + (Property Editor) belong to bob-ledger-tools' key gate in Live Preview — keep them unmapped here.`
- Do not commit inside `~/bob`. Vault sync owns those commits. Running `bob vault-sync`
  once publishes the change right away.

## Steps

1. Rebase bob-plugins, then re-grep as described above.
2. Implement §1 in `248m`, then update `298m`'s registration comment.
3. Implement §2.
4. **Unit tests**
   1. In `scripts/test-ledger-tools-schema-double-equals.cjs`, replace the shared-gate
      table test with a bare-gate table that covers:
      - acting with `-` and with Shift and unshifted `+`;
      - `modified` (Ctrl, Alt, Meta), `repeat`, `composing`, and `default-prevented`;
      - `not-content`, for a target that is an `<input>` inside `.cm-editor` (the Vim
        search panel, a native property input) and for the inline title;
      - not Live Preview, unreadable Vim, and insert, visual, or replace mode;
      - `operator-pending` with `indentAuto` and with `delete`;
      - a non-empty key buffer (`g`, `f`);
      - a count and a pending register;
      - an unregistered arm and an arm that cannot act.
   2. Rename and adapt the `=-` toggle and fall-through tests to bare `-`. Assert that
      `handleKeyCalls` stays empty, because no `<Esc>` is injected.
   3. Adapt the `+` test to bare `+` (Shift and unshifted), also with no `<Esc>`.
   4. Add a test that pending `=` followed by `-` or `+` now falls through: the event is
      not consumed, and no arm or `<Esc>` runs.
   5. Keep every `==` test untouched and green.
   6. Update the two hint-string assertions.
5. **Real-Obsidian e2e** (run them as `docs/develop/validation.md` describes, with
   `BOB_BIN` from current bob-cli master where those docs require it)
   1. In `obsidian-card-foot-e2e.cjs`, send bare `-` for the toggle in both
      `dispatchFootKeys` and the minus loop. Assert that cursor and selection are
      unchanged. Add a check that typing `a-b` into Vim's `/` search prompt (followed by
      Esc) leaves the drawer state unchanged. Keep the `==` check.
   2. In `obsidian-property-editor-e2e-surfaces.cjs`, make bare `+` and the
      Shift+`=`-shaped `+` payload open the editor on `ordinary.md`. Add a check that
      `=` followed by `+` does **not** open it. Rename the "Edit all =+" check.
   3. Keep each offline `--self-test` wrapper green.
6. **Docs.** Rewrite every chord mention found by the grep to the new keys:
   - "`-` in Live Preview Vim normal mode toggles the drawer";
   - "`+` opens the Property Editor";
   - `==` is unchanged;
   - counts, registers, operator-pending keys, Source mode, insert and visual modes, and
     views without a mounted foot or registered editor keep Vim's `-`/`+` motions;
   - the `-`/`+` keys act only when the editor content has focus.

   In `docs/develop/architecture.md`, describe the split: `=` uses the pending-operator
   arm, while `-` and `+` use bare arms with the content-target guard. Also note there
   that the registration API name is historical. In `docs/bob-navigation-hotkeys.md`,
   note that the vimrc leaves `-` and `+` unmapped. Then run `npm run docs:check`.

7. **Finish bob-plugins.**
   1. Run `npm run build`, then `npm test`.
   2. Run `bob plugins sync`. If it refuses, follow the AGENTS.md overwrite rule.
   3. Edit the vault vimrc (§3).
8. **Coordinate the in-flight epics.**
   1. Append one note each with `sase bead note bob-cli-pf "<text>"` and
      `sase bead note bob-cli-pl "<text>"`. The note says: the `=-`/`=+` chords are now
      bare `-`/`+` in Live Preview Vim normal mode; `==` is unchanged; the
      `registerPropertyCardSecondKeyArm("+")` API is unchanged; remaining phases' code,
      hints, tests, docs, and authorized memory strands must say `-`/`+`. Cite this
      plan.
   2. Do not edit memory. If a glossary strand names `=-` or `=+` by then (pf.6 and pl.9
      are authorized to write such strands), file a `memory` task bead with
      `/sase_new_task` naming the strand and the fix.

## Acceptance

- In Live Preview, Vim normal mode, on a card note with a mounted foot, `-` toggles the
  drawer, never moves the cursor, writes nothing, and spawns no bob.
- On any Markdown note, `+` (shifted or not) opens the Property Editor.
- `==` still opens the species editor. `=-` and `=+` no longer act.
- These all keep plain Vim behavior:
  - `-` with no mounted foot, and `+` with no registered editor;
  - counted, register, operator-pending, and key-buffer-prefixed `-`/`+` (`3-`, `d-`,
    `g+`, `f-`);
  - Source mode, insert mode, and visual mode;
  - typing `-`/`+` in the Vim search prompt, native property inputs, and the inline
    title.
- The vault vimrc no longer maps `-`.
- `npm test` passes, both e2e runs pass, `npm run docs:check` passes, and
  `bob plugins sync` has deployed the build.
- Both epics carry the coordination note.

## Out of scope

- Renaming the `registerPropertyCardSecondKeyArm` API.
- Reading-view keys. The palette commands and the disclosure still cover Reading view.
- Any change to `==` or to the editor's internal `=` hand-off key.
