---
tier: tale
title: Repeat the last bracket walk with ]] / [[ and move labeled links to ]n / [n
goal:
  "`]]` / `[[` repeat the last-used bracket walk type (`s` review, `w` lanes, `n` links)
  forward / backward, and the old `[[` / `]]` link keys live on `[n` / `]n`."
size: medium
decisions:
  capitals:
    ask:
      Should `]S`/`[S` and `]W`/`[W` set their lowercase type, so `]]` then steps that
      walk?
    choices:
      family:
        "`]]` steps `]s` after `]S`, and walks the landed lane with `]w` after `]W`"
      exact:
        "`S` / `W` are own types: `]]` re-jumps to POST after `]S` and cycles lanes
        after `]W`"
    default: family
    why:
      The types are named by letter, and repeating an endpoint jump never moves
      anywhere.
    answer: family
  blank_lines:
    ask:
      Should `[<Space>` / `]<Space>` (insert blank line above / below) also become a
      repeatable type?
    choices:
      exclude:
        Blank-line inserts leave the remembered type alone, so `]]` / `[[` only ever
        navigate
      include:
        After `]<Space>`, `]]` / `[[` insert blank lines below / above until another
        walk is used
    default: exclude
    why: A stray `]]` should never edit a note.
    answer: exclude
proposed_by: bbugyi200.athena.15c
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 15:56:48
status: wip
---

# `]]` / `[[` Repeat The Last Bracket Walk; Labeled Links Move To `]n` / `[n`

## Goal

Today the vault vimrc maps `[[` / `]]` to "open previous / next labeled link". This
change:

1. **Moves the link keys to `[n` / `]n`.** They keep exactly today's behavior.
2. **Gives `]]` / `[[` a new job.** They repeat the last _type_ of bracket keymap used.
   `]]` runs that type's forward key and `[[` runs its backward key.

The motivating flow: in the morning, press `]s` once to start the GTD review walk. From
then on, `]]` jumps to the current or next review task, and `[[` steps back. After a
`]w` lane walk, `]]` / `[[` act like `]w` / `[w`. After `]n`, they act like `]n` / `[n`.

The behavior should be intuitive, reliable, and invisible:

- **Intuitive.** The key decides the direction: `]]` always goes forward and `[[` always
  goes backward, whichever direction the last key went.
- **Reliable.** `N]]` behaves exactly like `N]s` / `N]w`. The remembered type survives
  plugin reloads and Obsidian restarts.
- **Invisible.** No new toast. Each walk's own landing toast already shows which walk
  ran.

All code lives in the linked **bob-plugins** repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`), read its `AGENTS.md`, and use the printed path
for every read and write. The keymap change lives in the vault file
`~/bob/obsidian_vimrc.md`. bob-cli itself does not change: its `docs/reference/plugins/`
tree is a gitignored mount of the bob-plugins docs, and no other bob-cli page documents
`[[` / `]]` as keys (checked at planning time; re-check with `grep`).

## Bracket Keymap Audit

The user asked whether any bracket types were missing. At planning time, every two-key
bracket mapping lives in `~/bob/obsidian_vimrc.md`. No plugin calls `vim.mapCommand`
with a `[` / `]` key. The `Alt+[`, `Ctrl+]`, and similar entries in
`.obsidian/hotkeys.json` are modifier chords, not bracket keymaps.

| Keys                      | Command ids (`bob-navigation-hotkeys:`)               | Type                              |
| ------------------------- | ----------------------------------------------------- | --------------------------------- |
| `[[` / `]]` → `[n` / `]n` | `open-prev-link` / `open-next-link`                   | `n`                               |
| `[s` / `]s`               | `jump-to-prev-due-task` / `jump-to-next-due-task`     | `s`                               |
| `[S` / `]S`               | `jump-to-first-due-task` / `jump-to-last-due-task`    | `s` (see DECISION `capitals`)     |
| `[w` / `]w`               | `jump-to-prev-lane-task` / `jump-to-next-lane-task`   | `w`                               |
| `[W` / `]W`               | `jump-to-prev-lane` / `jump-to-next-lane`             | `w` (see DECISION `capitals`)     |
| `[<Space>` / `]<Space>`   | `insert-blank-line-above` / `insert-blank-line-below` | none (see DECISION `blank_lines`) |

obsidian-vimrc-support also defines `[[` / `]]` as previous / next heading _motions_.
The vimrc's `nnoremap` already overrides them in normal mode, and that stays true. The
heading motions keep working in visual and operator-pending mode (for example `d]]`).
This plan does not change that.

## Behavior Contract

### Recording the type

- The family command callbacks registered in `490-plugin-lifecycle.js` record their type
  on every invocation, before they delegate. The type is recorded whatever the outcome:
  an empty queue, a swallowed press while the review walk is busy, or
  `No next link found` all still count as "used".
- Recording happens at the command level, so any surface that runs these commands
  records the type. That includes the vimrc keys, `Ctrl+Alt+J/K`, the ledger-tools
  review footer click (which runs `jump-to-next-due-task`), bob command mode, and the
  command palette. All of them are the same walk.
- Only those callbacks record. Internal callers that reach `jumpToDueTask` directly
  never record: Ctrl+Alt+F's advance, the PRE/POST `Ctrl+Enter` advance, the decay card,
  `fromAdvance` tails, and stale retries. The same holds for every other command,
  including `jump-to-next-open-task` (`Ctrl+Shift+J/K`), the section-header jumps, and
  (under `blank_lines = exclude`) the blank-line inserts.
- `]]` / `[[` never change the remembered type.

### Repeating

The two new commands look up the remembered type and delegate to the same method the
type's own key calls:

| Type | `]]` (`repeat-next-bracket-walk`)  | `[[` (`repeat-prev-bracket-walk`)  |
| ---- | ---------------------------------- | ---------------------------------- |
| `s`  | `this.jumpToDueTask(1)`            | `this.jumpToDueTask(-1)`           |
| `w`  | `this.jumpToLaneTask(1)`           | `this.jumpToLaneTask(-1)`          |
| `n`  | `this.openLabeledBodyLink("next")` | `this.openLabeledBodyLink("prev")` |

- The direction comes from the repeat key, not from the last key. For example, `[s`
  followed by `]]` runs `jumpToDueTask(1)`.
- `]]` with type `s` inherits everything `]s` does. That includes "away from the current
  review task, first return to it". This is what makes `]]` jump to "the next / current
  review task".
- **Counts.** `repeatBracketWalk(direction)` must be a plain, non-`async` method. It
  calls the delegate synchronously and returns the delegate's result, with no `await`
  before the call and no `repeat` option passed. That way the pending Vim count is still
  in the input state when `consumePendingReviewJumpRepeat` /
  `consumePendingLaneWalkRepeat` read it, so `3]]` equals `3]s` / `3]w`. Type `n`
  ignores counts, just as `]n` does today.
- **Cold start.** If no type has been remembered (and none is stored), show the Notice
  `Nothing to repeat yet — press ]s, ]w, or ]n first`. Move nothing and return `false`.
  Export the text as `BRACKET_REPEAT_EMPTY_NOTICE`.

> [!decision] capitals = exact The type set becomes `s`, `S`, `w`, `W`, `n`.
> `jump-to-first-due-task` / `jump-to-last-due-task` record `S`, and `jump-to-next-lane`
> / `jump-to-prev-lane` record `W`. Add two repeat rows: for `S`, `]]` →
> `jumpToDueTask(-1, { endpoint: "last" })` and `[[` →
> `jumpToDueTask(1, { endpoint: "first" })`. For `W`, `]]` →
> `jumpToLaneTask(1, { laneJump: true })` and `[[` →
> `jumpToLaneTask(-1, { laneJump: true })`. The arguments mirror the existing command
> callbacks exactly. Docs and tests cover the five types.

> [!decision] blank_lines = include `insert-blank-line-above` /
> `insert-blank-line-below` record type `space`. For that type, `]]` →
> `this.insertBlankLine(editor, "below")` and `[[` →
> `this.insertBlankLine(editor, "above")`, where `editor` is
> `this.getActiveMarkdownView()?.editor`. Without an active Markdown editor the repeat
> does nothing, just as the `editorCallback` would be unavailable. The cold-start notice
> text stays the same. Docs say that `]]` can edit text after `]<Space>`.

### Persistence

- Store the type device-locally with
  `this.app.saveLocalStorage(BRACKET_REPEAT_STORAGE_KEY, type)`, using
  `BRACKET_REPEAT_STORAGE_KEY = "bob-navigation-hotkeys-bracket-repeat"`. Write only
  when the type changes. Cache it in `this.bracketRepeatType`.
- On the first repeat, read the stored value lazily and synchronously with
  `this.app.loadLocalStorage(...)`, then normalize it. A stored value that is not a
  known type counts as unset.
- If either API is missing or throws, fall back to in-memory state silently. Precedent:
  `239-plugin-day-strip-lanes.js` in ledger-tools and `678-plugin-templater-trigger.js`
  here.
- The value never touches the vault, `data.json`, or vault sync. It is per device and
  per vault, and it survives the plugin reloads that `bob plugins sync` deploys require.

## Implementation (bob-plugins, `plugins/bob-navigation-hotkeys`)

- **New fragment `src/546-plugin-bracket-repeat.js`.** List it in `src/fragments.json`
  right after `545-plugin-review-journal.js`. It contains:
  - `BRACKET_REPEAT_STORAGE_KEY` and `BRACKET_REPEAT_EMPTY_NOTICE`.
  - A pure `normalizeBracketRepeatType(value)`, which returns a known type or `null`.
  - A pure `planBracketRepeat(type, direction)`, which returns a frozen
    `{ method, args }` from the repeat table above, or `null`.
  - `class BobNavigationHotkeysBracketRepeatMixin` with:
    - `rememberBracketRepeatType(type)` (normalize, cache, persist when changed)
    - `readBracketRepeatType()` (cache, else lazy load)
    - `repeatBracketWalk(direction)` (sync delegate via
      `this[plan.method](...plan.args)`, or the cold-start notice)
- **`src/490-plugin-lifecycle.js`:**
  - In `onload`, initialize `this.bracketRepeatType = undefined` next to the other state
    fields.
  - Wrap each family callback in the audit table so it calls
    `this.rememberBracketRepeatType(<type>)` and then returns exactly the call it makes
    today, with unchanged arguments.
  - Register the two new commands next to `open-next-link` / `open-prev-link`, with no
    `hotkeys`:
    - `repeat-next-bracket-walk`, named
      `Repeat last bracket walk forward (]s, ]w, or ]n)`, with callback
      `() => this.repeatBracketWalk(1)`
    - `repeat-prev-bracket-walk`, named
      `Repeat last bracket walk backward ([s, [w, or [n)`, with callback
      `() => this.repeatBracketWalk(-1)`
  - The file must stay ≤ 1000 lines (576 at planning time).
- **`src/690-install-plugin-mixins.js`:** install
  `BobNavigationHotkeysBracketRepeatMixin` right after
  `BobNavigationHotkeysLaneWalkMixin`.
- **`src/700-exports.js`:** add `normalizeBracketRepeatType`, `planBracketRepeat`,
  `BRACKET_REPEAT_STORAGE_KEY`, and `BRACKET_REPEAT_EMPTY_NOTICE` to `helpers`. The file
  is at 961 lines and must stay ≤ 1000.
- **`manifest.json`:**
  - Bump the minor version from whatever is current (2.58.0 → 2.59.0 at planning time).
  - Append this sentence to the description:
    ``[n` / `]n` (mapped in the vault vimrc) open the note's previous / next labeled link; `]]` / `[[` repeat the last bracket walk — `]s` review, `]w` lanes, or `]n` links — forward / backward, remembered per device.``
  - Adjust the sentence if a decision branch changes the type set.
- Run `npm run build`. Never hand-edit `main.js`.

## Vault vimrc

Edit `~/bob/obsidian_vimrc.md` **after** `bob plugins sync` has deployed the new
commands. Follow `~/bob/AGENTS.md`: inspect `git status` first, touch only this file,
and commit only it as that file instructs. If the vault-sync daemon commits it first,
say so in the final response. Every mapping that replays `:` must be `nnoremap`, as the
file header requires.

1. In the first block, replace

   ```
   nnoremap [[ :bob_prev_link<CR>
   nnoremap ]] :bob_next_link<CR>
   ```

   with

   ```
   nnoremap [n :bob_prev_link<CR>
   nnoremap ]n :bob_next_link<CR>
   ```

   Leave the `exmap bob_prev_link` / `exmap bob_next_link` lines as they are.

2. After the lane-walk block, add:

   ```
   " ]] / [[ repeat the last bracket walk (]s review, ]w lanes, ]n links) forward / backward.
   exmap bob_repeat_next_bracket obcommand bob-navigation-hotkeys:repeat-next-bracket-walk
   exmap bob_repeat_prev_bracket obcommand bob-navigation-hotkeys:repeat-prev-bracket-walk
   nnoremap ]] :bob_repeat_next_bracket<CR>
   nnoremap [[ :bob_repeat_prev_bracket<CR>
   ```

## Docs (bob-plugins)

- `docs/bob-navigation-hotkeys.md`:
  - Add a `## Bracket repeat` section after `## Lane walk`. It covers:
    - the types and which keys and commands set each one (including `Ctrl+Alt+J/K`, the
      review footer, and the capitals per DECISION `capitals`)
    - key-owned direction
    - the morning flow example (`]s` once, then `]]`)
    - count parity
    - device-local persistence that survives reloads
    - the cold-start notice
    - what never records (internal advances, other commands, and `]<Space>` per DECISION
      `blank_lines`)
    - that `]]` / `[[` never write
  - Add one cross-reference sentence each to `## Review walk` (`]]` / `[[` repeat `]s` /
    `[s` after any review-walk key) and to `## Lane walk` (the same for `]w` / `[w`).
  - In `## Notes, tabs, and keyboard layers`, extend the custom-vault-bindings bullet:
    the vimrc maps `[n` / `]n` to `open-prev-link` / `open-next-link` and `[[` / `]]` to
    the two repeat commands, and these are vault configuration, not registered defaults.
- Run `npm run docs:gen` so the generated command table, the plugin facts, and the
  `docs/index.md` version pick up the change. Then run `npm run docs:check`.

## Tests

Add `scripts/test-navigation-bracket-repeat.cjs` and list it in `package.json`'s `test`
script right after `scripts/test-navigation-lane-walk.cjs`. Reuse the `obsidian` stub
and `onload` command-collection pattern from `scripts/test-navigation-lane-walk.cjs`
(see its "jump-to-next-lane / jump-to-prev-lane are registered" test). Give `app` a
`Map`-backed `loadLocalStorage` / `saveLocalStorage`. Cases:

- **Pure helpers.**
  - `normalizeBracketRepeatType` accepts exactly the active type set. It rejects unknown
    strings, `""`, `null`, numbers, and the capitals unless `capitals = exact`.
  - `planBracketRepeat` returns every repeat-table row for both directions, and `null`
    for an unknown type.
- **Registration.** Both repeat ids are registered with the planned names and no
  `hotkeys`. All family ids are still registered.
- **Recording.**
  - Each family callback records its type and calls its original method with today's
    exact arguments (for example `[S` → `jumpToDueTask(1, { endpoint: "first" })`).
  - `saveLocalStorage` receives the key and type, and is not called again when the type
    does not change.
- **Repeating.**
  - Every type delegates per the table in both directions.
  - `[s` then `]]` delegates `jumpToDueTask(1)`.
  - `]s`, `]w`, then `]]` delegates to the lane walk.
  - Repeated `]]` presses never change the stored type.
- **Cold start.** With nothing stored, or with a garbage stored value, `]]` shows
  `BRACKET_REPEAT_EMPTY_NOTICE`, calls no delegate, and returns `false`.
- **Persistence.**
  - A fresh plugin instance whose storage holds `w` delegates `jumpToLaneTask(1)` on the
    first `]]`.
  - When the local-storage API is missing or throwing, the in-memory state still works
    and nothing throws.
- **Count safety.**
  - The delegate runs synchronously inside the repeat callback, before the returned
    value settles.
  - With the real `jumpToLaneTask` and a stubbed `consumePendingLaneWalkRepeat`
    returning 2, `]]` with type `w` consumes the count before any `await` and lands two
    entries ahead. This mirrors "counted ]W consumes the count before any await".
- **Non-recorders.** `refresh-task-freshness-and-advance`, `jump-to-next-open-task`, a
  direct `jumpToDueTask(1, { fromAdvance: true })`, and (under `blank_lines = exclude`)
  `insert-blank-line-*` leave the remembered type unchanged. Stub their heavy internals.
- **Regression.** The lane-walk, review-walk/advance, and vim-count suites pass
  unchanged.

## Verification And Deploy

1. In bob-plugins, these must all pass: `npm run build`, `npm test`, `npm run validate`,
   and `BOB_CLI_DIR=<the bob-cli checkout> npm run docs:check`.
2. Run `bob plugins sync` from the bob-plugins checkout, as its `AGENTS.md` requires.
   Pass `--repo <that checkout> --no-pull` if a bare sync refuses or targets another
   checkout.
3. Run the rollout smoke gate from the bob-plugins checkout:
   - `xvfb-run -a npm run smoke:obsidian -- --report /tmp/obsidian-smoke.json`
   - `npm run smoke:obsidian -- --deployed-vault ~/bob --report /tmp/obsidian-smoke-deployed.json`

   If Obsidian or Xvfb is unavailable, say so rather than claiming a pass.

4. Make the vault vimrc edit and commit it as described above.
5. Commit bob-plugins. It is an opened-repo obligation for `/sase_final`.
6. In the final response, tell the user:
   - Obsidian must reload the plugins and the vimrc before `[n` / `]n` and the new `]]`
     / `[[` work.
   - Other devices need their own `bob plugins sync` (the plugin folders are gitignored
     in the vault). Until a device has the new plugin, `]]` / `[[` do nothing there,
     while `[n` / `]n` work as soon as the vimrc syncs.
   - Restate the audit result: `s`, `w`, and `n` are the walk types, and `[<Space>` /
     `]<Space>` is the only other bracket pair, handled per DECISION `blank_lines`.

## Non-Goals

- No change to what `]s` / `[s` / `]S` / `[S`, `]w` / `[w` / `]W` / `[W`, or the link
  commands do. Their toasts, hints, and the nav `api` version stay the same.
- No relative-direction (`;` / `,`-style) repeat. The key always picks the direction.
- No repeat of modifier chords (`Alt+[` / `Alt+]`, `Ctrl+]`, `Ctrl+Shift+J/K`) or
  `<C-j>` / `<C-k>`.
- No vault writes, stats-journal events, or freshness stamps from `]]` / `[[`.
- No change to the visual / operator-pending `[[` / `]]` heading motions from
  obsidian-vimrc-support.
- No `bob` CLI, Rust, Mac Capture, or memory-note changes.
