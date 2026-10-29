---
tier: tale
title: Ctrl+Shift+- Change-parent panel for Obsidian notes
goal:
  Ctrl+Shift+- opens a pre-filled, ranked, previewing panel that safely rewrites the
  active note's parent frontmatter.
size: medium
proposed_by: bbugyi200.athena.15u
proposed_with: claude/opus
create_time: 2026-10-29 12:31:21
status: wip
---

# Plan: Ctrl+Shift+- — Change the current note's parent

## Goal

Add a `Ctrl+Shift+-` keymap to Obsidian that opens a polished **Change parent** panel
for the active Markdown note. The panel is pre-filled with the current parent's name,
ranks existing vault notes as the user types, previews where the note will land in the
note tree, and on Enter rewrites only the `parent:` frontmatter line. It completes the
existing trio: `Ctrl+-` opens the parent (`open-parent-note`), `Ctrl+=` opens a child
(`open-child-note`), and `Ctrl+Shift+-` changes the parent.

All code changes are in the linked **bob-plugins** repo (open it with `/sase_repo`:
`sase repo open bob-plugins -r "..."`), plugin `plugins/bob-navigation-hotkeys`. It uses
the committed fragment build (`src/fragments.json`): edit `src/` fragments, run
`npm run build`, never hand-edit `main.js`, keep every hand-edited fragment ≤ 1000
lines, and run `bob plugins sync` after the change. No bob-cli source, docs, or memory
changes are needed. bob-cli's `docs/reference/plugins/` copy is an untracked mount of
bob-plugins docs.

## Design principles

1. **Intuitive.** The input shows the current parent's name, fully selected. Typing
   replaces it. Backspace clears it and shows suggestions. Enter on the highlighted row
   sets the parent. The panel works like the other `bob-cnp` pickers (Ctrl+= child
   picker, Ctrl+Shift+M move picker): same keys, same row style.
2. **Reliable.** A parent can only be an existing note, so the panel never writes a
   dangling link. The write changes one frontmatter line and nothing else. Cycles, stale
   edits, and malformed frontmatter are refused with a clear notice and nothing is
   written. When the note is open in the active editor, one undo reverts the change.
3. **Beautiful.** Header breadcrumb of the current location, a live "Moves to"
   breadcrumb that highlights the new parent, typed area/project badges, small `current`
   / `up a level` / `recent` chips. Everything uses Obsidian theme variables, so it
   looks right in light and dark themes.

## Facts this design relies on (verified in the vault and repo)

- ~6.8k Markdown notes. `parent:` values appear as unquoted `[[x]]` (3,273), `"[[x]]"`
  (1,456), and `'[[x]]'` (151). About 2,700 are path-qualified (`[[2026/202608]]`,
  `[[zot/day_log.zot]]`). A few are bare strings (`obsidian_ref`) or Templater
  expressions (`_templates/`). Parents are often ordinary notes (`journal`, `2024`,
  `mind`), not only areas and projects. The candidate pool must therefore be every note,
  ranked.
- bob-cli resolves scalar `parent:` wikilinks, quoted or unquoted. The exact
  vault-relative path wins; otherwise only a unique, case-insensitive basename resolves.
  List-form parents and bare strings do not resolve (`docs/pomodoro-homes.md`, decision
  `subprojects-inherit-project-blocking`). So the writer always emits one double-quoted
  scalar wikilink.
- `app.fileManager.processFrontMatter` re-serializes the whole YAML block. It would turn
  the many unquoted `[[x]]` flow values in _other_ keys (`type: [[area]]`, …) into
  nested lists and drop comments. **Do not use it.** Edit the text with a pure line
  planner instead.
- Existing helpers to reuse:
  - `FilteredPickerModal` (`src/200-filtered-picker-modals.js`) and its
    `renderPickerSubtitle` hook.
  - `renderTypedNotePickerRow` and `getFileChildNoteInfo` (`190`/`200`).
  - `getFrontmatterLinks` and `resolveLinkTargetFile` (`670`/`680`): these give the same
    "child" meaning as `collectChildNotes`, including unquoted `[[x]]`, which Obsidian
    parses as `[["x"]]`.
  - `getFrontmatterWikiLinkToFile(file)` (`670`): `[[basename]]` at the vault root,
    otherwise `[[folder/basename|basename]]`, the same form Create project note writes.
  - `splitMarkdownContent`, `getYamlScalarText`, and `startsWithFrontmatter` (`210`).
  - `FRONTMATTER_DELIMITER_RE` (`010`).
  - `applyEditorContentTransaction` (`180`).
  - `getActiveMarkdownFile` / `getActiveMarkdownView` (`670`).
  - `isStaleRegisteredPicker` (`200`).
- Vault `hotkeys.json` binds `open-parent-note` to Ctrl+- and `open-child-note` to
  Ctrl+=. Nothing binds Ctrl+Shift+-, and no installed plugin claims it. Obsidian stores
  shifted punctuation by its base key, e.g. the existing Ctrl+Shift+] binding is
  `{modifiers:["Ctrl","Shift"], key:"]"}`.
- `src/700-exports.js` is at 999 lines, so new test exports need a new fragment.

## UX specification

### Keymap and command

- Register command `change-parent-note`, named **"Change parent note"**, with plugin
  default hotkey `hotkeys: [{ modifiers: ["Ctrl", "Shift"], key: "-" }]`. Use a plain
  `callback`, not `editorCallback`, so it works in Reading view too. Register it in
  `src/490-plugin-lifecycle.js` directly after `open-child-note`, with a one-line
  comment noting the Ctrl+- / Ctrl+= / Ctrl+Shift+- trio.
- With no active Markdown file, show the existing "No active markdown file" notice.
- Single instance: a second press while the panel is open focuses the open panel and
  does not open another. Track it in `this.activeParentNotePicker` using the
  `isStaleRegisteredPicker` stale-slot rule.

### Panel anatomy (reuses the `bob-cnp` chassis plus `bob-pnp-*` classes)

```
┌───────────────────────────────────────────────────────────────┐
│ [network]  Change parent                                       │
│            org › dev › ai                ← current location     │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ 🔍 [dev]            ← pre-filled, fully selected            │ │
│ └───────────────────────────────────────────────────────────┘ │
│  MOVES TO   org › bob › ai              ← live preview strip    │
│ ───────────────────────────────────────────────────────────── │
│ ▸ ◆ dev        dev.md               [Area]  current          ↵ │
│   ◆ devops     projects/devops.md   [🟢 WIP]  4 children       │
│   …                                                            │
│        Showing 50 of 214 matches — keep typing to narrow        │
│ ───────────────────────────────────────────────────────────── │
│  ↑ ↓ Navigate   ^N ^P Move   ↵ Set parent   esc Cancel          │
└───────────────────────────────────────────────────────────────┘
```

- **Size:** a compact prompt, not the full 960×840 picker. Add `bob-pnp-modal` to
  `modalEl` with `width: min(92vw, 720px); height: min(72vh, 640px)` so it reads as a
  focused prompt.
- **Header:** Lucide `network` icon. Title "Change parent". The subtitle (rendered via
  `renderPickerSubtitle`) is the note's **current ancestry breadcrumb**: the scalar
  parent walk from the note upward, rendered root-first and ending with the note itself
  in bold.
  - With no parent: the subtitle reads `No parent yet · <note>`.
  - With an unresolvable parent: that segment shows a `triangle-alert` icon and the raw
    target in `--text-error`, followed by "(not found)".
  - A `list` or `duplicate` parent state shows the matching warning text.
- **Input:** pre-filled with the current parent's name and fully selected (focus and
  `select()` in the existing deferred-focus timeout).
  - Name source: the resolved file's basename. If the parent is unresolved, the last
    path segment of the raw link target. If there is no parent, an empty string.
  - Placeholder: "Type a note name…".
- **Preview strip** (`.bob-pnp-preview`, `aria-live="polite"`) sits between the search
  box and the results and updates on every selection or query change:
  - Selected row is a new parent → label **MOVES TO**, then the selected note's ancestry
    - the selected note (accent, semibold, `is-new-parent`) + this note (bold,
      `is-self`).
  - Selected row is the current parent → label **STAYS UNDER**, same breadcrumb, no
    accent.
  - No selectable row → strip hidden.
  - Breadcrumbs longer than 5 segments collapse to `root › … › last-2 › last-1 › self`,
    with the full chain in `title` / `aria-label`.
- **Rows:**
  - Base: `renderTypedNotePickerRow(file, noteInfo, rowEl, query)` gives the icon,
    highlighted basename, highlighted path, and area/project status badges.
  - Trailing chips in a `.bob-pnp-chips` group: `current` (accent-tinted), `up a level`
    (the grandparent), `recent` (in `workspace.getLastOpenFiles()`).
  - A faint `N children` meta when the note already parents at least one note.
- **Capped results:** render at most 50 rows. When more match, add a non-selectable
  faint line under the rows: "Showing 50 of N matches — keep typing to narrow".
- **Empty states**, replacing the default `emptyText` and checked in this order:
  1. The query matches only excluded notes (this note or its own subtree): "“q” is
     inside this note's own subtree — choosing it would create a cycle".
  2. Nothing matches: "No note matches “q” — the parent must be an existing note".
- **Footer hints:** `↑ ↓ Navigate`, `^N ^P Move`, `↵ Set parent`, `esc Cancel`.
- **Keys:** inherit `FilteredPickerModal` behavior: arrows / Ctrl+N / Ctrl+P move, Enter
  or click commits, Esc closes. No count handling: the parent belongs to the note, and
  counts are meaningless here.

### Candidate pool and ranking (pure, testable)

Build the **parent index** once when the panel opens, from
`app.vault.getMarkdownFiles()` and `metadataCache`. For each note record: `path`,
`basename`, `noteInfo` (via `getFileChildNoteInfo`; kind area/project/plain, terminal
project status), resolved `parentPath`, `childCount`, and `recentRank` (index in
`getLastOpenFiles()`, else ∞).

- **Excluded:** the note itself, every descendant (BFS over the children map built from
  the resolved `parentPath`s, which blocks parent cycles), and `_conflicts/**`.
- **Empty query** (after Backspace), in this order, deduplicated and capped at 50:
  1. Current parent (`current`).
  2. Grandparent (`up a level`).
  3. Recent notes by recency (`recent`).
  4. The most-used parents by `childCount` desc, then path.
- **Typed query** (trimmed, case-insensitive): match tiers on the basename are 0 exact,
  1 prefix, 2 word-start (after `_ - . /` or a space), 3 substring, 4 full-path
  substring, 5 fuzzy subsequence. Non-matches drop. Within a tier, sort by:
  1. The current parent first.
  2. Kind weight: area or open project, then plain note, then terminal project.
  3. `childCount` desc.
  4. `recentRank` asc.
  5. Path length asc.
  6. Path alphabetical.

Because the input is pre-filled, the current parent is normally row 1 (exact match) and
Enter on it is a no-op. Typing a new name immediately puts the best match first.

### Commit behavior

`commitParentChange({ file, candidate, expected })`, run when a row is chosen:

1. **Cycle check:** walk up from the candidate through resolved scalar parents (≤ 16
   hops, stop on a repeat). This is authoritative at commit time and catches tree edits
   made while the panel was open. If the walk reaches the note itself, or the candidate
   is the note, refuse: "Can't make X the parent: it is inside this note's subtree".
2. **No-op:** if the candidate is the currently resolved parent, show "Parent unchanged:
   X", close, and do not write. This holds even when the stored form is unquoted or
   single-quoted.
3. **Build the link:** `getFrontmatterWikiLinkToFile(candidate)` produces the value, and
   the planner writes it as `parent: "<link>"`.
4. **Write:**
   - If the active `MarkdownView` shows this file and `getMode() === "source"`: plan
     against `editor.getValue()` and apply with
     `applyEditorContentTransaction(view.editor, content, plan.content)`. This is one
     CodeMirror transaction, so one undo reverts it, and the cursor is mapped.
   - Otherwise: `app.vault.process(file, (data) => …)` and re-plan inside the callback
     against the on-disk text, so the read-modify-write is atomic.
5. **Stale guard:** the planner gets `expectedRaw`, the raw `parent` value (or `null` if
   absent) captured when the panel opened. A mismatch refuses: "Parent changed while the
   panel was open — press Ctrl+Shift+- again".
6. **Success notice:**
   - Parent replaced: `Parent: dev → bob`.
   - No previous parent: `Parent set: bob`.

   Close the panel.

7. **Other refusals** close the panel with a notice and write nothing:
   - Unclosed frontmatter: "Can't change parent: the frontmatter block has no closing
     ---".
   - Duplicate keys: "Can't change parent: the frontmatter has N parent: keys — fix them
     by hand first".
   - Unquotable link: "Can't change parent: X's name can't be written as a quoted link".
   - Write error: "Could not change parent: <message>".

Reparenting is an ordinary note write. Root-parent grouping is computed at read time
from the new tree. For project notes, inherited project blocking (`#prj_is_blocked`) is
derived and repaired by the next `bob task reconcile`, as the
`subprojects-inherit-project-blocking` decision prescribes for non-Task-Card writers.
The panel triggers no extra writes.

## Write contract: `planParentFrontmatterEdit(content, linkText, options)`

This is a pure function with no Obsidian dependencies. It returns
`{ ok: true, content, changed, action, previousRaw, previousTarget }` or
`{ ok: false, reason }`. Its sibling `readParentFrontmatterField(content)` returns
`{ state, raw, target, lineIndex }`, where `state` is one of `no-frontmatter`,
`unclosed`, `absent`, `scalar`, `list`, or `duplicate`. The panel uses it for pre-fill,
the subtitle, and `expectedRaw`.

Rules:

- Split with `splitMarkdownContent` and rejoin with the detected line ending, so CRLF
  files stay CRLF. Every byte outside the edited span is preserved.
- Frontmatter exists only if line 0 opens it (`startsWithFrontmatter`) and a later line
  matches `FRONTMATTER_DELIMITER_RE`. An opener without a closer gives
  `reason: "unclosed-frontmatter"`.
- Only **column-0** `parent` keys count: `/^parent[ \t]*:(.*)$/`. Indented `  parent:`,
  `parents:`, `parent_task:`, `Parent:`, and body text are ignored. More than one
  column-0 key gives `reason: "duplicate-parent-keys"` (with a count).
- Value forms:
  - **Double- or single-quoted scalar:** the value ends at the matching close quote, and
    any trailing `[ \t]+#…` comment is kept.
  - **Unquoted `[[…]]`:** the value ends at the first `]]`, and a trailing comment is
    kept.
  - **Plain scalar:** a trailing comment is removed with the `getYamlScalarText` rule
    and kept.
  - **Empty value followed by indented continuation lines** (block list or block scalar,
    `state: "list"`): replace the key line _and_ every following
    indented/blank-then-indented continuation line up to the next column-0 line or the
    closing delimiter.
- The target is the inner text before `|` / `#` for wikilinks, or the trimmed scalar.
  `previousTarget` reports it.
- **Replace:** write `parent: "<linkText>"` plus the preserved comment, if any, and
  return `action: "replace"`.
- **Key absent, frontmatter present:** insert `parent: "<linkText>"` as the first line
  after the opening delimiter, matching `_templates/new_note.md`, and return
  `action: "insert-key"`.
- **No frontmatter:** prepend `---`, `parent: "<linkText>"`, `---` before line 0, keep
  the body unchanged, and return `action: "create-frontmatter"`. An empty file becomes
  exactly those three lines plus a trailing line ending.
- If `linkText` contains `"`, `\`, or a newline, return `reason: "unquotable-link"`.
- `options.expectedRaw`, when provided (`string` or `null`), must equal the current raw
  value. A raw value is the exact text after `parent:` up to the end of the value (the
  joined lines for `list`), and `null` when the key is absent. A mismatch returns
  `reason: "stale"`.
- If the new line equals the old one, return `changed: false` and the content unchanged.
  The planner never writes in that case.

## Implementation layout (bob-plugins, `plugins/bob-navigation-hotkeys`)

Fragments share one script scope, and `fragments.json` order is the build order.
Function declarations hoist, but `class … extends` needs its base defined first.

| Fragment (new)                        | Place in `fragments.json`              | Contents                                                                                                                                                                                                                                                                                                |
| ------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/211-parent-frontmatter-plan.js`  | after `210-rename-and-line-context.js` | `readParentFrontmatterField`, `planParentFrontmatterEdit`, value-span parsing helpers                                                                                                                                                                                                                   |
| `src/208-parent-note-picker-model.js` | after `207-image-zoom-modal.js`        | `buildParentPickerIndex(records, context)`, `rankParentCandidates(index, query, options)` (tiers, exclusions, 50-cap with total), `buildParentAncestry(index, path, maxHops=16)`, `collapseParentBreadcrumb(segments, max=5)`, and the `PARENT_PICKER_HINTS` constant                                   |
| `src/209-parent-note-picker-modal.js` | after `208-…`                          | `class ParentNotePickerModal extends FilteredPickerModal`: pre-filled/selected input, `renderPickerSubtitle` breadcrumb, preview strip (override `renderResults` to call `super` then refresh the strip and capped-count line), chips, empty states, `onClose` clearing the plugin's active-picker slot |
| `src/682-plugin-parent-change.js`     | after `681-plugin-quick-roll.js`       | `class BobNavigationHotkeysParentChangeMixin`: `openParentChangePicker()`, `collectParentPickerRecords()` (metadataCache to plain records for the pure model), `commitParentChange(...)`, `parentChangeLinkFor(file)`                                                                                   |
| `src/701-exports-parent-change.js`    | after `700-exports.js`                 | `Object.assign(module.exports.helpers, { … })` for the new pure functions, the modal class, and `PARENT_PICKER_HINTS` (`700-exports.js` is at 999 lines)                                                                                                                                                |

Edits to existing files:

- `src/490-plugin-lifecycle.js`: register the command (see above) and initialize
  `this.activeParentNotePicker = null` in `onload`.
- `src/690-install-plugin-mixins.js`: add `BobNavigationHotkeysParentChangeMixin` to the
  install list.
- `styles.css`: append a `/* --- Parent note picker (Ctrl+Shift+-) --- */` section,
  using only Obsidian CSS variables:
  - `.modal.bob-cnp-modal.bob-pnp-modal`: the compact size above.
  - `.bob-pnp-preview`: flex row, `var(--background-secondary)`, `var(--radius-m)`,
    padding `var(--size-4-2) var(--size-4-3)`, margin
    `0 var(--size-4-4) var(--size-4-3)`, `var(--font-ui-small)`, overflow hidden.
  - `.bob-pnp-preview-label`: uppercase, `letter-spacing: .04em`,
    `var(--font-ui-smaller)`, `var(--text-faint)`.
  - `.bob-pnp-crumbs`: nowrap with ellipsis. `.bob-pnp-crumb` uses `var(--text-muted)`.
    `.bob-pnp-crumb-sep` is `›` in `var(--text-faint)`. `.is-new-parent` uses
    `var(--interactive-accent)` and `var(--font-semibold)`. `.is-self` uses
    `var(--text-normal)` and `var(--font-bold)`. `.is-missing` uses `var(--text-error)`.
  - `.bob-pnp-chip`: a small pill with `var(--radius-s)` and `var(--font-ui-smaller)`.
    `.is-current` uses `color-mix(in srgb, var(--interactive-accent) 14%, transparent)`
    with accent text. `.is-up` and `.is-recent` use `var(--background-modifier-hover)`
    with muted text.
  - `.bob-pnp-children`: faint meta text.
  - `.bob-pnp-more`: a centered faint capped-count line.
  - Under the existing mobile media query, hide `.bob-pnp-chips` except `.is-current`.
- `manifest.json`: bump `version` 2.61.0 → **2.62.0** and append one sentence to
  `description`: "Ctrl+Shift+- changes the current note's parent: a panel pre-filled
  with the current parent ranks existing notes, previews the new ancestry, refuses
  cycles, and rewrites only the `parent:` frontmatter line as a quoted wikilink in one
  undo."
- `package.json`: add `scripts/test-navigation-parent-change.cjs` to the explicit `test`
  list.
- `docs/bob-navigation-hotkeys.md`: add a `## Parent note` section after "Navigation and
  counts". Cover the Ctrl+- / Ctrl+= / Ctrl+Shift+- trio, the panel (pre-fill, Backspace
  suggestions, ranking, chips, preview), the write contract (only `parent:`, canonical
  `"[[…]]"` with path-qualified alias for non-root notes, comments and line endings
  kept, insert or create when missing), refusals, undo, and the note that root parents
  and inherited project blocking follow the new tree (reconcile repairs
  `#prj_is_blocked`). Then run `npm run docs:gen` so the generated commands table (new
  row with default hotkey `Ctrl+Shift+-`) and facts table update.

## Tests: `scripts/test-navigation-parent-change.cjs` (node:test, `navigation-hotkeys-harness.cjs`)

Planner/reader vectors. Each asserts exact output text, and asserts that every byte
outside the edited span is unchanged:

1. `parent: "[[dev]]"` → `parent: "[[bob]]"`.
2. `parent: '[[dev]]'` → the canonical double-quoted form.
3. Unquoted `parent: [[dev]]` → canonical.
4. `parent: "[[2026/202608|202608]]"`: `previousTarget` is `2026/202608`.
5. `parent: "[[x]]"     # projection of the task's residence (R4)`: the comment is kept.
6. Block list (`parent:` + `  - "[[a]]"` + `  - "[[b]]"`) followed by `created: …`:
   collapsed to one line, and `created` is kept.
7. Frontmatter without `parent`: inserted as the first key.
8. No frontmatter: the block is created and the body is identical.
9. A CRLF file stays CRLF.
10. An empty file.
11. Unclosed frontmatter → `unclosed-frontmatter`, content untouched.
12. Two column-0 `parent:` keys → `duplicate-parent-keys`.
13. Indented `  parent:`, `parents:`, `parent_task:`, and body `parent:` lines are
    ignored.
14. A Templater `parent: "<% … %>"` value is replaced cleanly.
15. An `expectedRaw` mismatch → `stale`.
16. A `linkText` containing `"` → `unquotable-link`.
17. The same link → `changed: false`.

Ranking/model vectors:

- Tier order exact > prefix > word-start > substring > path > fuzzy.
- The current parent wins ties.
- Kind weight orders area/open project > plain > terminal project.
- `childCount` and recency tie-breaks.
- Self, descendants (multi-level), and `_conflicts/` are excluded.
- A cycle through an unquoted `[[x]]` parent is detected.
- Results cap at 50 and report the true total.
- Empty-query order: current → up a level → recent → popular, deduplicated.
- The ancestry walk stops at 16 hops and on repeats.
- Breadcrumbs collapse at 5 segments.
- Excluded-only matches pick the cycle empty-state text.

Plugin/modal tests:

- The command is registered with id `change-parent-note` and default hotkey
  `{modifiers:["Ctrl","Shift"], key:"-"}`.
- Opening on a note with `parent: "[[dev]]"` pre-fills `dev` and selects it, and the
  subtitle breadcrumb ends with the note.
- A commit through the `vault.process` path writes the expected text and shows the
  notice `Parent: dev → bob`.
- A commit through the editor path issues exactly one editor transaction and does not
  call `vault.process`.
- No-op → `Parent unchanged: dev` and no write.
- A commit-time cycle (the tree changed after open) is refused.
- A stale `expectedRaw` is refused.
- A second command press while open does not create a second modal.
- No active Markdown file → the existing notice.

## Verification

From the bob-plugins checkout:

```bash
npm run build
npm run build:check
npm run docs:gen
npm run docs:inventory:check
node --test scripts/test-navigation-parent-change.cjs
npm test
npm run validate
npm run smoke:obsidian   # startup smoke, if the environment supports it
bob plugins sync
```

Also confirm that `docs/bob-navigation-hotkeys.md` now lists
`| Change parent note | change-parent-note | Ctrl+Shift+- |` and that no fragment
exceeds 1000 lines.

## Non-goals

- Creating a new note from the panel. The parent must already exist. A typed unknown
  name shows the explanatory empty state.
- Removing a parent. Vault notes must keep a `parent`, so the panel offers no "clear"
  action.
- Reparenting several notes at once, counts, or acting on a Task Link's target note.
- Running `bob task reconcile` or rewriting descendants. Inherited project blocking is
  repaired by the next reconcile, per the existing decision.
- Any bob-cli, Bob Mac Capture, or vault `hotkeys.json` change. The binding is a plugin
  default.
