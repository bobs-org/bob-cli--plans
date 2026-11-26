---
tier: tale
title: "Copy card: replace Absolute path with Note link and Task link rows"
goal:
  Cmd+Y drops the A row and copies [[note]] and, for a task with a block ID,
  [[note#^id]].
size: medium
decisions:
  row_keys:
    ask: Which keys should the new Note link and Task link rows use?
    choices:
      l_k:
        L copies the note link, K the task link; home-row neighbors, clear of your t
        template
      w_l: W copies the note link (wikilink), L the task link, the more frequent row
      l_b: L copies the note link, B the task link (Obsidian calls it a block link)
    default: l_k
    why: L reads as link on the always-present row; K sits beside it and dodges t
    answer: l_k
proposed_by: bbugyi200.athena.1gq
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 12:18:11
status: wip
---

# Copy card: drop Absolute path, add Note link and Task link rows

## Goal

The `Cmd+Y` Copy card (`bob-navigation-hotkeys:copy-active-file-path`) currently lists
`P` Path, `A` Absolute path, `F` File name, `N` Note name, then a TASK group with `I`
Block ID, `copy.templates` rows, and an optional `C` chip row. Change it to:

1. Remove the `A` Absolute path row.
2. Add a **Note link** row to the file group that copies a wikilink to the active note,
   e.g. `[[foo]]` for `~/bob/foo.md`.
3. Add a **Task link** row to the TASK group that copies a block link to the selected
   task, e.g. `[[foo#^bar]]` for the `^bar` task in `~/bob/foo.md`. It is shown **only**
   when the selected task has a block ID.

"Task Link" is the glossary term for a block link to a task, so the row label matches
Bob's vocabulary.

## Where the work lives

All changes are in the **`bob-plugins`** linked repo, so open it first with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md`. bob-cli needs no
change. Its `docs/reference/plugins/copy-card.md` is a gitignored mirror that the docs
site assembles from `bob-plugins:docs/copy-card.md`.

Relevant files (all under `plugins/bob-navigation-hotkeys/` unless noted):

- `src/101-copy-card-model.js`: `COPY_CARD_BUILTIN_ROWS`, `COPY_CARD_RESERVED_KEYS`
  (derived from the rows), `buildCopyCardBuiltinRow`, and `buildCopyCardModel`. **978
  lines; the hand-edited fragment cap is 1000.**
- `src/101a-copy-card-keys.js`: key resolution. Reserved-but-hidden keys are consumed,
  and unbound keys return `null`.
- `src/101c-copy-card-task-link.js`: `buildCopyCardRow`, `isCopyableCopyCardRow`, and
  the Task Link resolver.
- `src/641-plugin-copy-card.js`: mixin `openCopyCardSnapshot` gathers inputs for
  `buildCopyCardModel`.
- `src/080-dependency-identity.js`:
  `canonicalDependencyLink(target, sourcePath, markdownFiles)` is the shared link-note
  rule. It uses the basename unless another markdown note shares it (case-insensitively)
  or the note lives under the `done/` archive, in which case it uses the vault path
  without `.md`. **987 lines; do not edit.**
- `src/590-plugin-dependency-stage.js`: the `readDependencyVaultFileList()` plugin
  method lists vault-relative markdown paths.
- `src/717-copy-card-exports.js`: test exports.
- `src/fragments.json`: fragment order.
- Repo root: `docs/copy-card.md`, `docs/bob-navigation-hotkeys.md`,
  `scripts/test-navigation-copy-card-model.cjs`,
  `scripts/test-navigation-copy-card-view.cjs`, `scripts/obsidian-copy-card-e2e.cjs`,
  and `scripts/obsidian-copy-card-e2e-lib.cjs`.

Governing decision: `decisions:q-and-ctrl-bracket-mirror-esc`. `q` stays the reserved
close key, so no new row may use `q`, and Esc / `q` / `Ctrl+[` handling is unchanged.

## Behavior

Keys below use the `row_keys` decision. The default is `l_k`, and "note-link key" /
"task-link key" refer to whichever keys are chosen.

### Removed: Absolute path

- Drop the `absolute` builtin row. `a` stops being reserved. With no template bound to
  it, pressing `a` does nothing. A `copy.templates` entry with `key: a` now binds
  normally instead of reporting a collision.
- Keep the `{absolute_path}` template variable and its aliases (`abs_file`, `absolute`).
  Keep the `yank-absolute-path` / `yank-absolute-path-tilde` palette commands. Existing
  templates keep working.

### New: Note link (file group)

- Row id `note-link`, label `Note link`, the note-link key. It goes immediately after
  `Note name`, so the file rows are Path, File name, Note name, Note link.
- Value: `[[<note>]]`. `<note>` is the canonical link-note form of the active file from
  `canonicalDependencyLink`: the basename (`[[foo]]`), or the vault path without `.md`
  when a rival basename exists or the note is under `done/`.
- Ready whenever the active file has a vault-relative path. It does not depend on the
  vault base path or home directory, so it works where absolute paths are unavailable.
  It is not a path row (`isPath: false`), so it gets the plain preview and no head/tail
  split.

### New: Task link (TASK group)

- Row id `task-link`, label `Task link`, the task-link key. It goes immediately after
  `Block ID` and before TASK-group templates.
- Shown only when the resolved task state is `ready`. That means either the cursor's
  owning `#task` has a trailing `^id`, or a Task Link bullet at the cursor resolved to a
  `#task` with that ID.
- Hidden for `none`, `no-id`, and `unresolved`. In those states the existing `I` Block
  ID behavior stays unchanged, including the dimmed
  `none yet · Ctrl+6 on the task line names one` row.
- Value: `[[<note>#^<blockId>]]`, using the same canonical rule for the task's note. For
  an owned task that is the active file. For a resolved Task Link it is the linked note
  (`taskPath` in `buildCopyCardModel`).
- The copied link always uses the full `[[note#^id]]` form, never the same-note
  `[[#^id]]` form, even for a same-note Task Link bullet, so the copy pastes correctly
  into any note.
- The task-link key stays reserved while the row is hidden, the same way `I` and `C`
  are. Pressing it then is consumed and never falls through to a template, so a key
  never changes meaning.

### Unchanged rules

- Path stays the default highlight. Row order becomes: file builtins (Path, File name,
  Note name, Note link), file-only templates, the diagnostic row, the TASK / LINKED TASK
  group (Block ID, Task link, task templates), then CHIP.
- `↑`/`↓`/`Ctrl+N`/`Ctrl+P` still move over copyable rows. The footer chip derives
  `copy note link` / `copy task link` from the label automatically.
- `COPY_CARD_RESERVED_KEYS` becomes `p f n l i k c` under the default (plus `q` for
  close). Collision reasons read `key l is Note link's` / `key k is Task link's`.
- Copying still never writes the vault. Preview and clipboard stay identical, and the
  copied Notice shows the link text.

## Implementation

1. **Builtin rows** (`101-copy-card-model.js`):
   - Replace the `absolute` spec with `note-link` (after `note`) and add `task-link`
     (after `block-id`). Neither has a template `variable`.
   - In `buildCopyCardBuiltinRow`, make `isPath` true only for `path`, and drop the
     `absolute_path` branch of the unavailable-reason text.
   - Leave the `values` / `taskValues` tables and `COPY_CARD_VARIABLE_NAMES` alone, so
     the template grammar is unchanged.
2. **New fragment `src/101d-copy-card-links.js`**, registered in `fragments.json` right
   after `101c-copy-card-task-link.js`:
   - Use function declarations only. Fragments share one script scope, and a top-level
     `const` that reads another fragment at load time would break order independence.
   - It holds a `copyCardLinkNote(path, markdownPaths)` helper that wraps
     `canonicalDependencyLink`. Use its `.note` for the Note link and its `.text` for
     the Task link, with a source path that can never equal a vault note (for example
     `""`), so it never returns `[[#^id]]`.
   - It also holds the row builders for `note-link` and `task-link`, built with
     `buildCopyCardRow` (group `file` / `task`).
   - This keeps `101-copy-card-model.js` under 1000 lines. Do not edit `080`.
3. **Model wiring** (`buildCopyCardModel`):
   - Accept an optional `markdownPaths` array. When it is missing, the canonical-rule
     fallback still yields the basename, or the full path for `done/`.
   - Route `note-link` into `fileBuiltins`. Route `task-link` into `taskBuiltins` only
     when `taskState === "ready"`.
4. **Mixin** (`641-plugin-copy-card.js`): in `openCopyCardSnapshot`, pass
   `markdownPaths: this.readDependencyVaultFileList() || []`. First confirm that the
   method is reachable on the same plugin instance; if it is not, call
   `app.vault.getMarkdownFiles()` the same way.
5. **Exports** (`717`): export the new helper(s) the tests use.
6. Run `npm run build` to regenerate `main.js`. Never hand-edit `main.js`.

## Tests

- `scripts/test-navigation-copy-card-model.cjs`:
  - Update the builtin-row and reserved-key assertions.
  - Remove or retarget every `absolute` row assertion. Keep the
    `{absolute_path}`-template and alias-hint assertions.
  - Add Note link cases: root note → `[[bob]]`; nested unique note → basename; rival
    basename → vault path; `done/` note → vault path; missing `markdownPaths` →
    basename; no base path → still ready.
  - Add Task link cases:
    - Owned ready task → `[[bob#^id]]`.
    - Task Link bullet to another note → `[[other#^id]]`.
    - Same-note `[[#^id]]` bullet → `[[bob#^id]]`.
    - `no-id` → row absent while `I` stays dimmed.
    - `unresolved` and `none` → absent.
    - Row order with templates and chip.
  - Add key cases:
    - The note-link and task-link keys copy.
    - The task-link key is consumed while hidden.
    - `a` with no template returns `null`.
    - A template with `key: a` binds.
    - Templates keyed to the new keys report the new collision reasons.
- `scripts/test-navigation-copy-card-view.cjs`:
  - Footer test: ArrowDown from Path now shows `copy file name`.
  - Replace the `"a"` keypress in the busy test with a still-bound key.
  - Retarget the click-on-"Absolute" test (around line 1218) to another ready row.
  - Add a render test: Note link and Task link rows show their link previews, and no
    `Absolute path` text appears.
- `scripts/obsidian-copy-card-e2e.cjs` and `-lib.cjs`:
  - Update the expected row-id lists (around lines 602 and 613).
  - Replace `copy-absolute-A` with note-link and task-link key copies.
  - ArrowDown / Ctrl+N from Path now land on File name.
  - Drop `expected.absolute` and add the expected Task link value.
  - Rename the fixture template currently labelled `Note link` (`[[{note_name}]]`), for
    example to `Bare note link`, so it does not duplicate the builtin label. Keep its
    positional `2` coverage.
  - Check fixture template keys against the chosen `row_keys` (the full config binds `b`
    and `t`).

## Docs

- `docs/copy-card.md`:
  - Update the intro.
  - Keys table: remove `A`, add the Note link and Task link rows with example values
    (`[[bob]]`, `[[bob#^bank-records]]`).
  - Reserved-keys sentence and row-order paragraph.
  - Task Links section: the Task link row uses the linked note.
  - States paragraph: `no-id` and `unresolved` hide the Task link row.
  - Add a short note on the link form (basename unless a rival or `done/`).
  - Add a tip: to keep an absolute-path row, add a template
    `{label: Absolute path, key: a, template: "{absolute_path}"}`.
- `docs/bob-navigation-hotkeys.md`: update the Copy card summaries near lines 8, 34, and
  729 so they mention links (for example "path, link, block ID, or template").

## Validation

From the `bob-plugins` checkout:

1. `npm run build`, then `npm run build:check`.
2. `node --test scripts/test-navigation-copy-card-model.cjs scripts/test-navigation-copy-card-view.cjs`,
   then the full `npm test`.
3. `npm run docs:check`.
4. Run `xvfb-run -a node scripts/obsidian-copy-card-e2e.cjs`; Obsidian and `xvfb-run`
   exist on athena. If Obsidian cannot launch, run it with `--self-test` and report that
   the live run was skipped.
5. `bob plugins sync` to deploy to the vault. If it refuses an overwrite, follow the
   bob-plugins `AGENTS.md` procedure. Never pass `-F` on your own.

## Out of scope

- New template variables for links (for example `{note_link}`).
- Changes to the `yank-*` palette commands.
- Changes to the Mac Capture or PDF Copy cards.
- Any bob-cli change.
- A manifest version bump. Prior Copy card features did not bump it.
