---
tier: epic
title: Cmd+Y Copy card — keyed paths, the task's block ID, and your own templates
goal: 'Cmd+Y in Obsidian opens a compact, filter-less Copy card where every row copies
  with one key (P Path stays the ↵ default), the cursor''s task contributes an I Block
  ID row, and copy.templates in ~/.config/bob/config.yml adds rows such as "the
  ^{block_id} task in the {path}". Preview and clipboard never disagree, and copying
  never writes the vault.

  '
decisions:
  cmd_y_twice:
    ask: Should a fresh second Cmd+Y on the open card copy the highlighted row?
    default: true
    why: Matches the decay card's re-pressed Alt+F; Cmd+Y Cmd+Y never leaves the chord.
    answer: true
  copy_card_record:
    ask: Should the config phase add a decisions-web record for the Copy card rules?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: model
    title: Pure Copy card model, template grammar, and config loader
    depends_on: []
    size: medium
    description:
      "model: add Obsidian-free fragments for the row table, the owning-task resolver,
      the single-pass template grammar, copy.templates loading and validation, the
      frozen card model, and key resolution, with exhaustive unit tests."
  - id: card
    title: Copy card modal, Cmd+Y wiring, styles, and docs
    depends_on:
      - model
    size: medium
    description:
      "card: replace YankPathPickerModal with a CopyCardModal on Bob's key-card tokens,
      wire Cmd+Y and a yank-block-id palette command through a new plugin mixin, add the
      styles, render and key tests, the copy-card docs page, then build and sync."
  - id: e2e
    title: Real-Obsidian e2e, light and dark screenshots, and polish
    depends_on:
      - card
    size: medium
    description:
      "e2e: drive the card in an isolated scratch vault over CDP, assert the clipboard
      for every key and state, capture light and dark screenshots of each state, and fix
      whatever the screenshots show is misaligned, clipped, or low-contrast."
  - id: config
    title: Ship the Task ref template and document copy.templates
    depends_on:
      - card
    size: small
    description:
      "config: add the copy.templates block with the Task ref template to chezmoi's Bob
      config and apply it, and add a copy section to bob-cli's configuration reference
      that links the plugin page."
proposed_by: bbugyi200.athena.1bg
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 11:19:05
status: wip
---

- **PROMPT:**
  [prompts/202611/cmd_y_copy_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/cmd_y_copy_card.md)

# Plan: Cmd+Y Copy card

## Why and what changes

Today `Cmd+Y` (command `copy-active-file-path`, `Mod+Y`) opens `YankPathPickerModal`, a
`FilteredPickerModal` stretched to 960×840 px around six rows behind a "Filter path
formats" input. Only `Cmd+Y ↵` is fast. "Parent directory" shows `(empty string)` at the
vault root and "Relative path" repeats "Basename". The footer says `↵ Open`, the tilde
row's notice says "Copied absolute path", and the row previews one file while activation
re-reads the active file, so preview and clipboard can drift.

This epic replaces that picker with a **Copy card**: the Obsidian twin of Bob Mac
Capture's `Cmd+Y` Copy card (fixed letters for fixed rows, digits for lists, Return for
the selected row, `I` for the ID). Bryan accepted every requirement in the research
report `research:202611/cmd_y_copy_card/cmd_y_copy_card.md`. This plan adopts that
design and tightens it in the places called out under "Design refinements over the
research".

All plugin work is in the **bob-plugins** linked repo, plugin
`plugins/bob-navigation-hotkeys` (open it with `/sase_repo`). It uses the fragment
build: edit `src/` fragments, register new ones in `src/fragments.json`, run
`npm run build`, and never hand-edit `main.js`. **Each fragment must stay at or below
1000 lines.** Several touched fragments are close to that limit: `640` (983), `200`
(960), `100` (917), `010` (941), and `700-exports.js` (exactly 1000, so do not add
anything to it). New logic goes in new fragments. New exports go in a new
`717-copy-card-exports.js` fragment that uses the
`Object.assign(module.exports.helpers, {...})` pattern from
`716-reload-history-exports.js`. After any bob-plugins change, run `bob plugins sync`
and follow the repo `AGENTS.md` rule for refused syncs (never pass `-F` on your own).

## The design (the contract every phase implements)

### Card anatomy

The card is compact and sized to its content: `width: min(600px, 92vw)`, `height: auto`,
`max-height: min(75vh, 840px)`. It has no filter input.

Cursor in a task that has a block ID, with Bryan's template configured:

```text
╭──────────────────────────────────────────────────────────────╮
│ ▣  Copy                                                    ✕ │
│    bob.md                                                    │
│                                                              │
│▌ P   Path            ~/bob/bob.md                          ↵ │
│  A   Absolute path   /Users/bbugyi/bob/bob.md                │
│  F   File name       bob.md                                  │
│  N   Note name       bob                                     │
│ ─ TASK  Use records to start tracking bank transactions ──── │
│  I   Block ID        ^bank-records                           │
│  T   Task ref        the ^bank-records task in the ~/bob/b…  │
│                                                              │
│  ↵ copy path    ↑↓ select    esc close                       │
╰──────────────────────────────────────────────────────────────╯
```

Cursor in a task with **no** block ID (47% of `#task` lines in the vault have none):

```text
│ ─ TASK  Add event note to the recruiter Google event ─────── │
│  I   Block ID        none yet · Ctrl+6 on the task line names one │  (dimmed)
│  T   Task ref        needs a block ID                        │  (dimmed)
```

Cursor anywhere else (the common path-copy case): only the four file rows, plus any
template that does not use `{block_id}`. There is no divider and no task group.

A broken template keeps its place, shows no keycap, and states the fix:

```text
│  ·   Wiki link       unknown {task_id} · did you mean {block_id}? │  (error colour)
```

### Rows, keys, and variables

| Key               | Row                         | Copies (cursor on `^bank-records` in `bob.md`)                                                           | Template variable                   |
| ----------------- | --------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| `P` · `↵`         | Path                        | `~/bob/bob.md` (`~` only at a real home-directory boundary; a vault outside home gets the absolute path) | `{path}`                            |
| `A`               | Absolute path               | `/Users/bbugyi/bob/bob.md`                                                                               | `{absolute_path}`                   |
| `F`               | File name                   | `bob.md`                                                                                                 | `{file_name}`                       |
| `N`               | Note name                   | `bob`                                                                                                    | `{note_name}`                       |
| `I`               | Block ID (task group)       | `^bank-records`                                                                                          | `{block_id}` = `bank-records`, bare |
| `key:` or `1`–`9` | Each `copy.templates` entry | `the ^bank-records task in the ~/bob/bob.md`                                                             | —                                   |

- "Parent directory" and "Relative path" leave the card. Their palette commands
  (`yank-parent-directory` and `yank-relative-path`) stay, along with the other four
  `yank-*` commands, because someone's bindings may use those command IDs.
- `P A F N I` are **reserved even when their row is hidden**, so a key never changes
  meaning. Letters are case-insensitive, which also covers Caps Lock. Shift+letter is
  ignored (kept free for alternate payloads later).
- Row order is fixed and never depends on context: file rows, then file-only templates,
  then the TASK group (`I`, then templates that use `{block_id}`). Templates keep the
  order they have in `copy.templates`.
- The highlight starts on **Path**. If Path is unavailable (no vault base path, for
  example on mobile), it starts on the first copyable row. `↵` copies the highlighted
  row.

### Which task (the TASK group rule)

The resolver runs **once, before the card opens**, against the live editor buffer
(`editor.getValue()`), not the lagging metadata cache:

1. It uses the active `MarkdownView` only when that view is in an editing mode (source
   or live preview), has an editor, and has exactly one selection. Reading view,
   multiple cursors, or no editor mean no task. These are the guards `Ctrl+6` uses.
2. `owner = findOwningTaskLine(lines, head.line)` (`src/070-recommended-roll-batch.js`).
   This covers the task line, its child notes, and its Depends-On, BLOCKS, LINKS, and
   LINKED FROM lines, and it skips fences. For a non-empty selection, the owners of the
   anchor and the head must be equal. If they differ, the result is ambiguous and there
   is no task. Never pick one.
3. `blockId = getTrailingBlockId(lines[owner])` and
   `title = cleanTaskDisplayText(lines[owner])` (`src/020-config-load-and-tasks.js`).
   Any checkbox status is accepted, because a closed task's ID is still a valid pointer.
4. **Same-file invariant.** Every value on the card describes the active file. A
   `[[other#^id]]` link or Task Link under the cursor never supplies the block ID.
   Pomodoro rows are not `#task` lines, so they have no owner.
5. The result has three states: `none` (no group), `no-id` (dimmed group), and `ready`
   (live group).

### Template contract (`copy.templates`)

The templates live in `~/.config/bob/config.yml` (`XDG_CONFIG_HOME` aware, through
`getBulletPropertyConfigPath` in `src/010-requires-and-config.js`). Load them with
`parseYaml` and the mtime-and-size stat-key cache pattern used by
`loadBlocksLinesEnabled` in `src/020-config-load-and-tasks.js`, re-read on every
`Cmd+Y`, so a `chezmoi apply` takes effect on the next press. bob-cli ignores unknown
top-level keys, so the block is safe on every host.

```yaml
copy:
  templates:
    - label: Task ref # required
      key: t # optional: one letter or digit
      template: "the ^{block_id} task in the {path}" # always quote templates
```

The grammar is deliberately tiny:

- **Substitution.** `{name}` substitutes the value of a variable, where `name` matches
  `[a-z_]+`. `{{` and `}}` produce literal braces. Values are inserted in **one pass**
  and never re-scanned, so a file name containing `{block_id}` stays a file name.
- **Output is plain text**, with no shell or Markdown escaping. Newlines in a template
  are kept on the clipboard, and the preview shows each one as a faint `⏎`.
- **Errors appear on the card, never on the clipboard.** An error row has no key, is
  skipped by `↑↓`, and cannot be clicked. Each error gets a specific reason:
  - Unknown variable: `unknown {task_id} · did you mean {block_id}?` The hint comes from
    a fixed alias map (`task_id`→`block_id`, `abs_file_with_tilde`→`path`,
    `abs_file`/`absolute`→`absolute_path`, `basename`→`file_name`,
    `basename_no_ext`/`stem`→`note_name`, `block_ref`→`^{block_id}`), else the nearest
    known name within edit distance 2, else the list of valid names.
  - Unbalanced brace: `unmatched { at column N`.
  - Empty template: `template is empty`.
  - Non-string template: `quote the template — YAML read it as a mapping` when the
    template is an object (an unquoted `{path}`), or `quote the template` when it is a
    number or boolean.
  - Missing or blank label: the row is labelled `Template N` with the reason
    `add a label`.
  - Entry that is not a mapping: `expected label and template`.
- **Missing runtime values dim or hide the row. They never produce partial text** such
  as `the ^ task in …`:
  - A template that uses `{block_id}` is hidden when there is no task, and dimmed with
    `needs a block ID` when the task has no ID.
  - A template that uses `{path}` or `{absolute_path}` is dimmed with
    `needs an absolute vault path` when the vault has no base path.
- **Keys.** An optional `key` must be one `[a-z0-9]` after lowercasing (a YAML number
  `1` counts as `"1"`); anything else is an error on that row. Without a key, a
  template's key is its 1-based position in `copy.templates`, so the key only changes
  when you edit the list. Entries after the ninth need an explicit key, or they stay
  keyless with the reason `add key: to bind`.
- **Key collisions.** A collision with a reserved letter, or between templates, never
  silently remaps anything. Built-ins always keep their key. Every colliding template
  loses its keycap and shows a warning-colour reason (`key p is Path's`,
  `key t is also Wiki link's`), but it **stays copyable** with `↑↓ ↵` or a click. Only
  invalid templates can never copy.
- **Shape errors.** No file, no `copy:`, or an empty list mean no template rows. Invalid
  YAML, a `copy` that is not a mapping, or a `templates` that is not a list disable only
  the template rows. They add one diagnostic error row, for example
  `copy.templates must be a list`. The built-in rows always work.

### Interaction and reliability rules

- **Frozen snapshot.** The plugin builds one immutable model when the card opens: file
  values, task state, rendered templates, and the default row. Keys, `↵`, and clicks
  copy that model's exact `value` string and never recompute it.
- **Catch-all key scope.** Register `scope.register(null, null, handler)` returning
  `false`, as `src/672-delete-confirm-modal.js` does, so no global hotkey fires behind
  the card. Handle every key in that one handler, not also in a `contentEl` listener.
- **Ignored input:** `event.repeat`; `isComposing` or `keyCode === 229`; Meta, Ctrl, or
  Alt chords other than `Ctrl+N`, `Ctrl+P`, and (if `cmd_y_twice`) the open-card
  `Mod+Y`.
- **Movement.** `↑`/`↓` and `Ctrl+P`/`Ctrl+N` move the highlight among copyable rows and
  wrap. Hover shows a hover tint but never moves the keyboard highlight. Clicking a
  copyable row copies it.
- **Refused keys.** Pressing the key of a dimmed row does not copy and does not close.
  The highlight moves to that row so its reason is in view, the row plays a short
  horizontal nudge (only under `prefers-reduced-motion: no-preference`), and a visually
  hidden `aria-live="polite"` region announces the reason. Clicking a dimmed row does
  the same. Pressing an unbound key, or a reserved key whose row is hidden, does
  nothing.
- **Copy order.** A busy guard ignores every key while a write is in flight. Await
  `navigator.clipboard.writeText(value)`. On success, close the card, then show
  `Copied <value>` with the value middle-truncated to about 60 characters and its
  newlines shown as `⏎`. On failure, the card **stays open** and shows
  `Couldn't copy: <reason>` in the error colour in the footer region.
- **Read-only.** No vault writes, no ID minting, no freshness stamps, no undo entries.

> [!decision] cmd_y_twice A fresh (non-repeat) `Mod+Y` while the card is open copies the
> highlighted row, so `Cmd+Y Cmd+Y` copies the Path without leaving the chord. This
> mirrors the decay card, where a fresh re-press of `Alt+F` chooses Stay. Auto-repeat
> from a held `Cmd+Y` never fires it, because repeats are ignored. With `= no`, `Mod+Y`
> inside the card is simply consumed.

### Visual spec ("beautiful by subtraction")

Reuse Bob's existing tokens; do not invent a new look.

- **Modal.** `.modal.bob-copy-card-modal`, with `.bob-key-card` on `contentEl`. Use the
  sizing above and a padding rhythm of 4 and 8 px multiples.
- **Header.** A 28 px accent-tinted rounded tile with the `copy` icon (the current
  picker's tile, which already looks right), the title **Copy** (`.bob-key-card-title`),
  and a muted subtitle line with the vault-relative path. The path goes under the title
  so it never collides with the modal's close button. The dialog gets `role="dialog"`,
  `aria-modal`, and `aria-label="Copy"`.
- **Rows.** Each row is a CSS grid
  `grid-template-columns: 28px 9.5em minmax(0, 1fr) 20px`, with `column-gap: 12px`,
  `min-height: 36px`, and `padding: 6px 10px`. The columns are:
  - **Keycap.** A compact `.bob-key-card-key` variant (24×24, centred uppercase letter).
    On the highlighted row its border and text take the accent colour. Keyless rows show
    an empty dashed cap so the columns stay aligned.
  - **Label.** In `--text-muted`.
  - **Value.** In `--font-monospace`, `--text-normal`, about 0.92em, with the full value
    in a `title` tooltip.
    - **Path values middle-truncate in pure CSS.** Two spans: the directory head
      (`min-width: 0; overflow: hidden; text-overflow: ellipsis`) shrinks, and the file
      name tail (`flex: none`) never does, so `bob.md` always stays visible. No JS width
      measuring.
    - **Template values** wrap to at most two lines (`-webkit-line-clamp: 2`).
  - **`↵` glyph.** Only on the highlighted row, in the accent colour.
- **Highlight.** The same accent treatment as today's picker: a
  `color-mix(in srgb, var(--interactive-accent) 12%, transparent)` background plus a 3
  px inset left bar.
- **TASK divider.** A hairline rule with a small-caps `TASK` label in `--text-faint` and
  the task title (from `cleanTaskDisplayText`) in `--text-muted`, ellipsized to one
  line. The divider is the reliability affordance: you see _which_ task's ID you are
  about to copy.
- **States.**
  - **Dimmed rows:** label and value at about 0.55 opacity, and the reason in italic
    `--text-faint` in the value column.
  - **Error rows:** the reason in `--text-error`.
  - **Collision rows:** the reason in `--text-warning`.
  - **Transitions:** only under `prefers-reduced-motion: no-preference`.
- **Footer.** Reuse the picker footer's chip styling (`.bob-cnp-hint` and `.bob-cnp-kbd`
  in `styles.css`) so the card reads as part of the Bob picker family. The chips are
  `↵ copy <row label, lowercased>`, `↑↓ select`, and `esc close`. The `↵` chip updates
  as the highlight moves.
- **Accessibility.** The list has `role="listbox"` with `aria-activedescendant`. Rows
  are `role="option"` with `aria-selected` and `aria-keyshortcuts`; dimmed and error
  rows also get `aria-disabled="true"`.
- **Narrow widths (< 480 px).** The label column shrinks to 7em; nothing overflows
  horizontally.

### Design refinements over the research

1. **No opening-chord guard.** The research suggested the decay card's
   `openingChordKeys` pattern, but it would be wrong here. macOS does not deliver
   `keyup` for a key pressed while Cmd is held, so a held-key set seeded with `y` could
   never clear, and the card would ignore every key. `Y` is not a row key anyway.
   Ignoring `event.repeat` and modified chords covers auto-repeat completely.
2. **The path sits in the header subtitle**, not right-aligned, so it never collides
   with the modal's close button.
3. **Refused keys explain themselves.** They move the highlight, nudge the row, and
   announce the reason, instead of being a silent no-op.
4. **Key collisions keep the row usable.** A colliding row is keyless with a warning
   reason but still copyable. Only invalid templates never copy.
5. **The palette `yank-*` commands report the copied value as well**
   (`Copied ~/bob/bob.md`), which fixes the misleading "Copied absolute path" for both
   path rows.
6. **The feature gets its own docs page** (`docs/copy-card.md`), like `task-card.md`,
   instead of prose buried in the hotkeys page.

## Phase `model`: pure Copy card model, template grammar, and config loader

Create these Obsidian-free fragments in `plugins/bob-navigation-hotkeys/src/` and
register them in `src/fragments.json` right after `100-yank-and-paths.js`:

- **`101-copy-card-model.js`**
  - `COPY_CARD_BUILTIN_ROWS`: a frozen table of `{id, key, label, variable, needs}` for
    `path`, `absolute`, `file`, `note`, and `block-id`, plus `COPY_CARD_RESERVED_KEYS`.
  - `resolveCopyCardTask(lines, selections)`: implements the task rule (steps 2–5) and
    returns `{state: "none"|"no-id"|"ready", line, blockId, title}`. The caller passes
    `[]` or more than one selection to force `none`, which keeps the editor guards
    (step 1) in the plugin.
  - `parseCopyTemplate(text)`: returns either `{ok: true, tokens, variables}` or
    `{ok: false, reason}`, including the did-you-mean logic.
  - `renderCopyTemplate(tokens, values)`: substitutes in one pass and returns `null`
    when any value it needs is unavailable.
  - `buildCopyCardModel({filePath, basePath, homePath, task, templates})`: returns a
    deeply frozen model:
    - `title`: `"Copy"`.
    - `subtitle`: the vault-relative path.
    - `rows`: each row is
      `{id, group, key, label, value, preview, previewParts, state, reason, isPath}`.
      - `state` is one of `ready`, `unavailable`, `error`, or `collision`.
      - `value` is the exact clipboard string, or `null` when the row cannot copy.
      - `previewParts` is `{head, tail}` for path rows (the CSS truncation needs it).
    - `defaultRowId`.
    - `task`: `{state, title}`.

    Reuse `getYankPathText`, `compactHomePath`, `joinFilesystemPath`, and the
    `getVaultPath*` helpers. Do not duplicate path logic.

  - `resolveCopyCardKey(model, event, ui)`: `ui` is `{highlightId, busy}`. It returns
    one of `{type: "copy", rowId}`, `{type: "move", rowId}`,
    `{type: "refuse", rowId, reason}`, `{type: "close"}`, `{type: "consume"}`, or
    `null`, and it applies every rule under "Interaction and reliability rules",
    including `cmd_y_twice`.
  - `formatCopiedNotice(value)`: returns `Copied <value>`, middle-truncated to about 60
    characters with newlines shown as `⏎`.

- **`102-copy-card-config.js`**
  - `loadCopyCardTemplates(options)`: returns either `{templates: [...]}` or
    `{templates: [], diagnostic}`. Every entry is normalized and validated, with its key
    resolved (explicit key or position) and its collision state computed. It caches on
    `configPath|statKey`, mirroring `loadBlocksLinesEnabled`, and accepts `env`,
    `osModule`, `fsModule`, and `parseYaml` injection for tests. It never throws.
  - Also add `resetCopyCardTemplatesCache()`.
- **`717-copy-card-exports.js`**: export every function above through
  `Object.assign(module.exports.helpers, {...})`, and register it in `fragments.json`
  after `716-reload-history-exports.js`.

Add `scripts/test-navigation-copy-card-model.cjs` (it loads `helpers` through
`scripts/navigation-hotkeys-harness.cjs`) and list it in `package.json` `test`. Cover:

- **The task rule:**
  - the task line, a child note, and a Depends-On / LINKS child line;
  - a closed `[x]` task (resolves);
  - a task without `^id` (`no-id`);
  - a cursor inside a code fence (`none`);
  - a Pomodoro Task Link line `- [[other#^x|…]]` (`none`);
  - a two-task selection (`none`);
  - multiple selections (`none`).
- **The grammar:**
  - escapes (`{{` and `}}`);
  - single pass, using a file named `a{block_id}.md`;
  - every error reason, including the did-you-mean hints for `{task_id}` and
    `{abs_file_with_tilde}`;
  - a YAML mapping and a number as `template`;
  - an empty template;
  - a multi-line template, which keeps its `\n` in `value` and shows `⏎` in `preview`.
- **The model:**
  - Bryan's `the ^{block_id} task in the {path}` yields exactly
    `the ^bank-records task in the ~/bob/bob.md` for `bob.md` in `/Users/bbugyi/bob`
    with `HOME=/Users/bbugyi`;
  - a vault outside home gives the absolute path for `{path}`;
  - no base path dims the path rows, and the default row becomes File name;
  - the TASK group is hidden for `none` and dimmed for `no-id`, and the `I` row value is
    `^bank-records`.
- **Keys:**
  - letters are case-insensitive;
  - Shift+letter, repeats, IME (`isComposing`, `229`), and modified chords are ignored;
  - `Ctrl+N`/`Ctrl+P` and arrows wrap and skip rows that cannot copy;
  - `↵` follows the highlight;
  - the key of a dimmed row is refused;
  - a reserved key whose row is hidden is consumed;
  - digit keys default by position;
  - explicit keys work;
  - entries past the ninth are keyless;
  - every reserved and duplicate collision makes the row keyless but copyable;
  - busy consumes every key;
  - `Mod+Y` follows `cmd_y_twice`.
- **The config loader:**
  - missing file, missing `copy`, and an empty list (no rows);
  - invalid YAML, a non-mapping `copy`, and a non-list `templates` (one diagnostic);
  - the cache hit, and the reload after an mtime or size change.

This phase changes no UI and no command, so it does not need `bob plugins sync`. Run
`npm run build`, `npm run build:check`, and `npm test`.

## Phase `card`: Copy card modal, Cmd+Y wiring, styles, and docs

- **`103-copy-card-modal.js`**: `class CopyCardModal extends Modal`. Model it on
  `FreshnessDecayCardModal` (`src/060-decay-card-modal.js`) for structure,
  `moveFreshnessDecayModalToAppDocument`-style document handling, and cleanup. Model it
  on `DeleteFileConfirmModal` (`src/672-delete-confirm-modal.js`) for the catch-all
  scope.
  - It receives the frozen model and an async `copy(row)` callback that resolves to
    `{ok, reason}`.
  - It renders exactly the anatomy and visual spec above, and dispatches every key
    through `resolveCopyCardKey`.
  - It owns only the highlight, busy, and failure UI state. It never reads the
    workspace.
- **`641-plugin-copy-card.js`**: a new `BobNavigationHotkeysCopyCardMixin`, installed in
  `src/690-install-plugin-mixins.js`.
  - `openCopyCard()`:
    1. Take the active Markdown file (the existing `getActiveMarkdownFile()` notice
       applies when there is none).
    2. Apply the editor guards from the task rule (step 1).
    3. Read `editor.getValue()` and `editor.listSelections()`.
    4. Get the vault base path from `getVaultBasePath()`, the home path from
       `getHomeDirectoryPath()`, and the templates from `loadCopyCardTemplates()`.
    5. Build the model and open the card.
  - `copyCardRow(row)`: writes `row.value` to the clipboard and returns `{ok, reason}`
    without its own notice. The card closes, then the mixin shows
    `formatCopiedNotice(row.value)`.
  - `yankTaskBlockId()`: the palette command. It copies `^id` for the cursor's owning
    task. Otherwise it shows a notice: `No task at the cursor`, or
    `This task has no block ID yet · Ctrl+6 on the task line names one`.
- **`640-plugin-notes-and-task-move.js`**:
  - Delete `openYankPathPicker()`.
  - Make `writeTextToClipboard`'s success notice use `formatCopiedNotice(text)` so the
    palette `yank-*` commands show the copied value. Keep the label for the failure
    notice.
  - Keep net growth ≤ 0.
- **`200-filtered-picker-modals.js`**: delete `YankPathPickerModal`.
- **`100-yank-and-paths.js`**: delete `createYankPathPickerItem` and
  `getYankPathPreviewText` once nothing uses them.
- **`010-requires-and-config.js`**: delete `YANK_PATH_PICKER_TITLES`. Keep
  `YANK_PATH_COMMANDS` and `YANK_PATH_NOTICE_LABELS`, because the docs inventory parses
  the `YANK_PATH_COMMANDS.forEach` registration.
- **`490-plugin-lifecycle.js`**:
  - Keep the command ID `copy-active-file-path` and `Mod+Y`. Rename the command to
    `Copy card: path, block ID, or template` and point it at `openCopyCard()`.
  - Add a literal `addCommand` for `yank-block-id` (`Yank task block ID`, no hotkey)
    next to the `YANK_PATH_COMMANDS.forEach` loop.
- **`styles.css`**: add a `.bob-copy-card*` block next to the key-card tokens. Do not
  alter the shared `.bob-key-card*` or `.bob-cnp-*` rules.
- **Tests.** Add `scripts/test-navigation-copy-card-view.cjs` (FakeNode, modelled on
  `scripts/test-navigation-task-card-view.cjs` and
  `scripts/test-navigation-delete-guard.cjs`) and list it in `package.json` `test`.
  Cover:
  - the rendered rows, keycaps, ARIA attributes, the TASK divider and title, and the
    dimmed, error, and collision rows;
  - the footer `↵` chip following the highlight;
  - a key copies and closes after the write resolves, then notices `Copied …`;
  - a rejected write keeps the card open with the error;
  - a double press during an in-flight write writes once;
  - **the snapshot**: the card opens on A, the active file switches to B, and a key
    still copies A's value;
  - the refused key moves the highlight and fills the live region;
  - `Esc` closes without writing;
  - `openCopyCard()` with reading view, multiple cursors, and no editor yields no TASK
    group;
  - `yank-block-id`, with and without an ID.
- **Docs.**
  - Add `docs/copy-card.md` to bob-plugins: what `Cmd+Y` shows, the keys table, the task
    rule and its three states, the variables, the template grammar and errors, and a
    `copy.templates` example. Add it to `docs/nav.yml` after Task Card.
  - Add one sentence in `docs/bob-navigation-hotkeys.md` prose linking to it.
  - Regenerate the command inventory with `npm run docs:gen`, then run
    `npm run docs:check`.
- **Checks and deploy.** Run `npm run build`, `npm run build:check`, `npm test`, and
  `bob plugins sync`. Then confirm with `wc -l` that every touched fragment is ≤ 1000
  lines.

## Phase `e2e`: real-Obsidian e2e, light and dark screenshots, and polish

Add `scripts/obsidian-copy-card-e2e.cjs` with its `-lib.cjs` and an offline self-test,
`scripts/test-obsidian-copy-card-e2e.cjs`, listed in `package.json` `test`. Follow
`scripts/obsidian-decay-ladder-e2e.cjs` and its lib: an isolated scratch vault and
profile, an owned Obsidian process (`/snap/bin/obsidian`) under `xvfb-run -a`, and CDP
probes. Never open the live vault.

- **Scratch vault.** One note with:
  - a task with `^bank-records` and a child note;
  - a task without an ID;
  - a closed task with an ID;
  - a Pomodoro-style Task Link line;
  - plain prose.

  Point `XDG_CONFIG_HOME` at a scratch dir whose `bob/config.yml` holds:
  - Bryan's Task ref template (`key: t`);
  - a file-only template (`"[[{note_name}]]"`, keyed by position);
  - a broken `{task_id}` template;
  - an unquoted-mapping template;
  - a template that collides with `key: p`.

- **Drive real keystrokes.**
  - `Mod+Y` then each of `P A F N I T 2`, plus `↵`;
  - `↓ ↵` and `Ctrl+N ↵`;
  - `Esc`;
  - `Mod+Y Mod+Y` (per `cmd_y_twice`);
  - a held `Mod+Y` with auto-repeat, which must not copy.
- **Assert the clipboard after every copy** by reading it in the renderer (Electron
  `clipboard.readText()` or `navigator.clipboard.readText()`). Also assert:
  - that no note's bytes changed;
  - that the card is closed after a copy and open after a refused key.
- **Screenshot every state in light and dark themes:**
  - (1) the cursor in prose (file rows plus the file template);
  - (2) the cursor on the ID'd task;
  - (3) the cursor on the no-ID task;
  - (4) the broken and collision rows;
  - (5) a narrow 460 px window.

  Write them to an `--out` directory outside the repo.

- **Polish pass.** Open every screenshot with the Read tool and fix what it shows in
  `styles.css` or the modal:
  - column alignment across row states;
  - truncation keeping the file name;
  - two-line template clamping;
  - divider spacing;
  - keycap centring;
  - contrast of dimmed, error, and warning text in both themes;
  - nothing clipped by the close button.

  Re-run the e2e until both themes look right. List the final screenshot paths in the
  phase's final response so Bryan can review them.

- Run `npm run build`, `npm test`, and `bob plugins sync` after any polish change.

## Phase `config`: ship the Task ref template and document `copy.templates`

- **chezmoi** (linked repo; open it with `/sase_repo` and read its `AGENTS.md`). Add
  this block to `home/dot_config/bob/config.yml`, after `capture:`, with a comment in
  the file's existing style:

  ```yaml
  # Obsidian Cmd+Y Copy card (bob-navigation-hotkeys; docs/copy-card.md in
  # bob-plugins). bob-cli ignores this block. Variables are the card's row labels:
  # {path} (~/…), {absolute_path}, {file_name}, {note_name}, and {block_id}
  # (bare, so write ^{block_id}). {{ and }} are literal braces. Always quote
  # templates. `key` is one letter or digit; without it, an entry's key is its
  # 1-based position (1–9). A template using {block_id} shows only in a task.
  copy:
    templates:
      - label: Task ref
        key: t
        template: "the ^{block_id} task in the {path}"
  ```

  Then follow that repo's apply rule (`chezmoi update -a --force` once the commit has
  landed). Bryan's Mac runs Obsidian and picks the block up on its next
  `chezmoi update`; say so in the final response.

- **bob-cli** (this repo). Add a `### copy` subsection to
  `docs/reference/configuration.md` under "Supported keys". It needs:
  - a one-row table: `copy.templates`, a list of `{label, template, key?}`, default
    `[]`, effect "adds rows to the Obsidian `Cmd+Y` Copy card; the CLI ignores it";
  - a short example;
  - a link to the mounted plugin page. From `docs/reference/configuration.md` that is
    `plugins/copy-card.md` (pages in `docs/start/` use `../reference/plugins/…`).

  Run `just docs-check`. If the URL contract or search smoke lists need the new mounted
  page, follow what the check reports.

> [!decision] copy_card_record If accepted, the config phase also uses
> `/sase_memory_write` to add one record to the `decisions` web. The rule it records:
> "Copy cards key fixed rows by letter and lists by digit; copying never writes the
> vault; every value on a card describes one target (the active file and its owning
> task)." It cites this epic, the research report, and the Mac Copy card. Then run
> `sase memory init`. With `= no`, no memory changes.

## Verification (end of epic)

On Bryan's Mac after `bob plugins sync` and `chezmoi update`:

- `Cmd+Y ↵` still copies `~/bob/bob.md`.
- `Cmd+Y T` on `^bank-records` copies exactly
  `the ^bank-records task in the ~/bob/bob.md`.
- `Cmd+Y I` copies `^bank-records`.
- On a task without an ID, `I` is dimmed and points to `Ctrl+6`.
- In prose, there is no TASK group.
- "Parent directory" and "Relative path" are gone from the card but still in the
  palette.
