---
tier: epic
title: (( task wikilink picker in Obsidian
goal: 'Typing (( anywhere prose is allowed in an Obsidian note opens a fast, calm,
  keyboard-first picker over every task in the vault. Choosing a task replaces the
  token with the same [[note#^id]] block link Bob Mac Capture would insert, naming
  an ID-less task first. Under an open Pomodoro the link obeys Task Link rules. Nothing
  else ever changes by surprise.

  '
decisions:
  pomodoro_blocked:
    ask: Under an open Pomodoro, should Enter on a prerequisite-blocked task raise
      its prerequisites?
    choices:
      raise: Raise them to Next like Ctrl+Shift+Enter, then offer them in the picker
        to link
      refuse: Disable the row with the reason; the picker never writes other tasks
    default: raise
    why: Every plugin link gesture already consumes Blocked targets this way (PDG1)
    answer: raise
  caret_parity:
    ask: Should ^^ links under an open Pomodoro adopt the same Task Link rules as
      ((?
    default: true
    why: One commit path; decisions:capture-link-drops-blocking-deps already says
      ^^ keeps PDG1
    answer: true
  decision_record:
    ask: Add a decisions record explaining why Obsidian (( mirrors bob's rules in
      JS instead of spawning bob?
    memory:
    - decisions:obsidian-task-wikilinks-mirror-bob
    default: false
    answer: false
phases:
- id: model
  title: Shared task wikilink rules and parity vectors
  depends_on: []
  size: medium
  description: 'model: pin bob''s wikilink_task pool, order, link-target, and guard
    rules as shared vectors asserted in bob-cli, and mirror them in a pure block-id-prompt
    model.'
- id: trigger
  title: Note grammar for the (( trigger
  depends_on: []
  size: small
  description: 'trigger: pure (( token detection for Markdown notes, its note-only
    exclusions, and the written-second-paren auto-open rule, pinned by vectors.'
- id: index
  title: Warm vault task index
  depends_on:
  - model
  size: medium
  description: 'index: an incrementally maintained, mtime-validated vault task index
    with live-editor overlays, streaming cold builds, and performance budgets.'
- id: picker
  title: Task Wikilink Picker card and identified-link insert
  depends_on:
  - trigger
  - index
  size: medium
  description: 'picker: wire the trigger to the new card, render it beautifully, and
    insert links for identified rows, with Escape, Backspace, the reopen command,
    and reference docs.'
- id: naming
  title: Name-then-link for ID-less tasks
  depends_on:
  - picker
  size: small
  description: 'naming: the Block ID prompt''s wikilink mode, its back-to-picker lifecycle,
    closed-task support, and stale-line relocation.'
- id: pomodoro
  title: Task Link rules under an open Pomodoro
  depends_on:
  - naming
  size: medium
  description: 'pomodoro: Pomodoro mode (no history, lane/schedule/type consequences,
    blocked handling, duplicate guard, budget notice) plus the ^^ alignment.'
- id: review
  title: Real-Obsidian verification, visual polish, and docs
  depends_on:
  - pomodoro
  size: medium
  description: 'review: Chromium previews and a real-Obsidian e2e on athena, polish
    driven by the screenshots, the final docs pass, and deployment.'
proposed_by: bbugyi200.apollo.b5
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-30 16:10:18
status: wip
bead_id: bob-cli-eq
---

- **PROMPT:** [prompts/202610/obsidian_task_wikilinks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/obsidian_task_wikilinks.md)
- **BEAD:** [bob-cli-eq](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-eq/README.md)

# Plan: `((` task wikilink picker in Obsidian

## Why and what

Epic `bob-cli-e7` (plan `202610/wikilink_task_picker.md`) gave Bob Mac Capture the
Roam/Logseq-style `((` gesture. It listed "`((` inside Obsidian itself (bob-plugins)" as
a non-goal. This epic closes that gap, so Bryan can type the same gesture wherever he
writes:

```text
Note:   - [ ] Follow up with Kelly about ((        ← typing the second "(" opens the picker
Pick:   "Stop idling when no agents run"   (sase_cleanup.md, ^no-idle)
Note:   - [ ] Follow up with Kelly about [[sase_cleanup#^no-idle]]|   ← caret after "]]"
```

`((` is a trigger, never stored syntax. What lands in the note is always a plain
Obsidian block link. Obsidian, `bob task archive` link repair, Task Mentions
(`glossary:task-mention`), and every Bob reader therefore keep working unchanged. In a
task's text the link becomes a Task Mention on its own: nav converges the target's
LINKED FROM line when the cursor leaves the line. When the link is the sole content of a
sub-bullet under an open Pomodoro, it is a Task Link (`glossary:task-link`) and follows
Task Link rules ([Pomodoro mode](#pomodoro-mode)).

**Naming.** In docs and UI this is the **Task Wikilink Picker**, the same name as on the
Mac. Code identifiers use `taskWikilink` / `task-wikilink`, and CSS uses `bid-twl-*`.
Never call the picker or its links "Task Links" except in Pomodoro mode, where the
inserted link really is one.

## Design principles

These are the tie-breakers for every detail below.

1. **One link everywhere.** The link target follows bob's `cross_note_link_target`. That
   is the shortest unique basename, else the vault path without `.md`, and `done/`
   always keeps its path. A link typed in Obsidian is byte-identical to one captured on
   the Mac (e7 decision `link_form = shortest`). Same-note links still name the note (no
   `[[#^id]]`), so a line moved by task move or inbox routing keeps resolving.
2. **Never surprise.**
   - The picker auto-opens only when the edit you just made wrote the second `(`.
   - Escape leaves exactly the text you typed.
   - One editor undo restores `((query))`.
   - Every write outside the cursor is stated in the detail strip before Enter, and
     announced by a Notice after.
3. **Quiet when nothing else changed.** Inserting a link to a task that already has an
   ID shows no Notice; the link is the feedback. Adding an ID or raising a lane does
   show one, because another note changed.
4. **Verify, then write.** Every commit re-reads its target authoritatively and checks
   the identity it relies on. When the target changed, the picker stays open with
   refreshed rows and a Notice saying why. It never guesses.
5. **Instant.** A warm open paints in under 50 ms, and typing never touches the disk.

## Architecture

**Owner: `plugins/block-id-prompt`.** That plugin already owns `^^` (the per-note task
picker), the Block ID prompt, source-link completion (`completeTaskSourceLink`), target
writes (`planTargetTaskUpdate` → `applyTargetTaskPlan`), Pomodoro activation
(`sourceQualifiesForPomodoroActivation`), future-link cleanup, and PDG1 consumption
(`planTaskLinkBlockConsumption`). `((` is the vault-wide sibling of `^^` and reuses that
pipeline instead of growing a second one.

**A JS mirror of bob's rules, not a `bob` subprocess.** The Mac follows
`decisions:mac-capture-is-a-thin-client`, a rule whose scope is capture grammar on the
Mac. Obsidian note editing has always been implemented in the plugins, and parity is
pinned by shared vectors. For example, nav's dependency ranker ports
`capture_link_tasks` and is pinned by DK1–DK8. Spawning `bob` is the wrong transport
here:

- `block-id-prompt` is not desktop-only, and mobile cannot spawn.
- bob reads disk, so the note you are typing in (unsaved for up to about 2 s) would be
  stale exactly in the "This note" group.
- A spawn costs about 0.6 s and 1.8 MB of JSON per open (the e7 measurement), against an
  in-memory index.

Parity is enforced mechanically: one `vectors.json` is asserted by a bob-cli Rust test
over `capture_task_wikilinks::candidates` and by the plugin's JS tests over its model.

**A modal card, not an inline `EditorSuggest`.**

- The vault holds about 4,300 tasks across about 6,500 notes. A useful picker needs
  grouped sections, rich rows, and a detail strip that states the consequence of Enter.
  An inline popover has no room for these, and per-row selection hooks need private
  `EditorSuggest` internals.
- A card matches the existing `^^` card (`TaskLinkPickerModal`, `bid-tlp-*`) and the Mac
  `((` card.
- Text typed before the card appears seeds its filter, as on the Mac. The card opens
  synchronously from the warm index, so nothing is lost while typing.

**Data flow**

```text
EditorView.updateListener ─► scheduleScan: record user-input written ranges
        │  (debounced SCAN_DEBOUNCE_MS)
        ▼
inspectActiveEditor ─► findTaskWikilinkToken (trigger) ─ second "(" just written? ─► open
        ▼
TaskWikilinkPickerModal ◄── taskWikilinkIndex.snapshot(ownerPath, liveOverlays)   (index)
        │                      └─ order/rank/group/guards from the pure model     (model)
   Enter on a row
        ├─ identified ─► verify target ─► one-transaction splice ─► caret after "]]"
        ├─ ID-less ────► BlockIdPromptModal (wikilink mode) ─► submitLinkTaskBlockId   (naming)
        └─ Pomodoro mode ─► gate (closed/blocked/duplicate) ─► Ctrl+Shift+Enter target plan
                           (forceNext, schedule pull-in, type, freshness) ─► splice   (pomodoro)
```

**Reuse map.** Existing `plugins/block-id-prompt/src` pieces to reuse, not copy:

| Need                    | Reuse                                                                                                                                                                                           |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Task line parsing       | `getObsidianTaskLineMatch`, `cleanTaskDisplayText`, `getTrailingBlockId`, `hasHideTaskTag`, `hasProjectBlockedResidentTag` (030)                                                                |
| Fences, lines           | `lineIsInsideCodeFence` (020), `forEachMarkdownLineOutsideCodeFence` (050)                                                                                                                      |
| Managed-line guards     | `isTaskDependencyLine`, `isMalformedTaskDependencyLine`, `isTaskBlocksLine`, `isMalformedTaskBlocksLine` (060), `isTaskRelationLine` (061), `isPomodoroEntryLine` (040), their notice constants |
| Scan lifecycle          | `scheduleScan`, `inspectActiveEditor`, `promptOpen`, `lastPromptKey`, `suppressEditorScans` (100/150)                                                                                           |
| Target read and resolve | `readAuthoritativePromptTargetContent` (095), `readDestinationForValidation`, `resolveReferenceDestination` (150)                                                                               |
| Writes                  | `planTargetTaskUpdate` (070), `applyTargetTaskPlan`, `adjustSourceForPlan`, `completeTaskSourceLink`, `taskLineStillPresent` (140)                                                              |
| ID naming               | `BlockIdPromptModal` (090), `resolvePromptBlockIdSuggestion` (095), `submitLinkTaskBlockId` (110), `BLOCK_ID_RE`                                                                                |
| Pomodoro                | `sourceQualifiesForPomodoroActivation`, `findPomodoroSourceContext` (040), `planFuturePomodoroLinkCleanup` (060), `planTaskLinkBlockConsumption` (030), `resolvePomodoroLinkThemeWrite` (078)   |
| Notices                 | `activationSuccessSuffix` (070), `futureLinkCleanupNoticeSuffix` (140), the plan-budget suffix used by Ctrl+Shift+Enter (130)                                                                   |
| Card look               | `bid-tlp-*` header, search, status pill, highlight, and footer styles (`styles.css`), `appendHighlighted`, `taskStatusClass` (080)                                                              |

New fragments slot into `src/fragments.json` so that classes are defined before any
module-level use (functions hoist, classes do not). Keep each hand-edited fragment at or
under 1000 lines. Export the pure helpers through `src/170-exports.js` for tests.
Install plugin methods as a mixin in `src/160-install-methods.js`. Suggested files are
below. Equivalent splits are fine.

## Experience spec

### Trigger

- **Opener.** `((` at line start or after a space or tab, not followed by a third `(`.
  This is exactly bob's `is_valid_opener`. It works in task text, sub-bullets, Work,
  Schedule, Cancel and Rotten Log entries (links there become Task Mentions), plain
  paragraphs, headings, blockquotes, and table cells. Mid-word `f((x` never triggers.
- **Token.** These rules are bob's `wikilink_task_tokens` and `field_at_token`,
  unchanged.
  - From `opener+2`, a `))` closes the token.
  - An unclosed query stops at the first `(`, `)`, `[`, `]`, backtick, or line end.
  - Closed `((q))`: the replacement covers `((q))` with the caret anywhere inside it.
    Obsidian's default bracket auto-pair turns `((` into `((|))`, so this is the common
    shape.
  - Unclosed: the replacement runs from the opener to the end of the non-whitespace run
    at the caret.
  - The query is the text from `opener+2` to the caret, never including `))`.
- **Never a token** (a note-only addition to bob's rules):
  - inside YAML frontmatter, fenced code, a `$$` math block, a `%%` comment block, or an
    HTML comment;
  - inside an inline code span, inline `$…$` math, inline `%%…%%`, an existing `[[…]]`
    (alias included), a Markdown link label or destination, an autolink, or a bare URL;
  - on a Pomodoro entry line;
  - in a reading view, with multiple cursors, or during IME composition.
- **Managed lines** (Depends-On, BLOCKS, LINKS, and LINKED FROM, malformed shapes
  included) never open the picker. If you just wrote the second `(` there, show that
  line's existing guard notice once. Those notices already point to the right gesture
  (for example Ctrl+Shift+P for prerequisites).
- **Auto-open only when the edit wrote the second `(`.**
  - Each docChanged update whose transactions are user input (`tr.isUserEvent("input")`,
    covering type, paste, drop, and compose) and not `undo` or `redo` records its
    inserted ranges. Later changes map them forward, and each scan clears them.
  - The picker opens only when the cursor's token has its second `(` (`opener+1`) inside
    a recorded range.
  - Typing `((`, an auto-paired `((|))`, a paste, and a fast burst such as `((fo` all
    open it.
  - Caret moves, Backspace into an old token, undo/redo, other plugins' `replaceRange`,
    and vault-sync reloads never do.
  - No separate suppression state is needed: after Escape the token stays, and it can
    only reopen when you deliberately retype a `(`.
- **Reopen on demand.** Add the command `link-task-wikilink`, **Link a task here**, with
  no default hotkey.
  - Inside a token, it opens the picker on that token.
  - With a single-line selection, it seeds the filter with the selection and replaces
    the selection on accept.
  - Otherwise it inserts at the caret.
  - In an excluded context it shows `Task wikilinks can't go here` (or the managed-line
    notice).

### The card

The card reuses the `bid-tlp` frame (`min(92vw, 900px)` × `min(82vh, 760px)`, radius
`--radius-l`). The modal root carries both `bid-tlp-modal` and `bid-twl-modal`, so it
inherits the shared header, search, and footer styles, and `bid-twl-*` adds the rest.
Everything is built from Obsidian CSS variables, so it reads correctly in light, dark,
and community themes.

```text
╭──────────────────────────────────────────────────────────────────────────────╮
│ ⟮((⟯  Link to a task                                                           │
│       Inserts [[note#^id]] in sase_cleanup.md                                 │
│ ┌──────────────────────────────────────────────────────────────────────────┐ │
│ │ ⌕  opus blen                                               3 of 4,323 ▏ │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
│ ▸ [ ] Try Opus with Blender. 💡                    mac_inbox  ⊕ Add ID       │
│   [*] Render the Blender turntable                 blender#^turntable        │
│   ── Completed history ──                                                    │
│   ✓  Install Blender on athena            Done     done/2026/blender#^inst   │
├──────────────────────────────────────────────────────────────────────────────┤
│ Try Opus with Blender. 💡                                                     │
│ Open · ⌂ mac_inbox · › Inbox · line 12                                        │
│ ↵ names it ^try-opus-blender in mac_inbox.md, then links it                   │
├──────────────────────────────────────────────────────────────────────────────┤
│ ↑↓ move   ↵ insert link   esc clear · close   ⌫ back to "("                  │
╰──────────────────────────────────────────────────────────────────────────────╯
```

- **Header.**
  - The icon tile, a 36 px accent-tinted square as in `bid-tlp-header-icon`, shows a
    monospaced `((`. It ties the card to the gesture.
  - The title is **Link to a task**.
  - The subtitle is `Inserts [[note#^id]] in <source basename>`. Pomodoro mode changes
    both lines ([Pomodoro mode](#pomodoro-mode)).
- **Search.**
  - The placeholder is _Search every task by text, note, or ^id_.
  - A trailing monospaced, tabular-numerals count sits in a fixed-width slot so the
    field never jumps: `4,323 tasks` when empty, `12 of 4,323` when filtered, and
    `No matches` when nothing matches.
- **Cold index.** The count slot reads `Indexing · 812 of 1,104 notes`, and a 2 px
  accent progress line runs along the bottom edge of the search field. Rows stream in
  with at most one repaint per 120 ms, and the selection is kept by row key
  (`path|line`).
- **Empty filter: grouped, in bob's canonical order.**
  - **This note** (the note you are typing in)
  - **In progress**
  - **Next**
  - **Other open tasks**, one sub-section per note
  - **Completed history**
  - Inside each group, hidden (`#hide`) rows trail visible ones, as bob's order
    dictates. In "Other open tasks" they gather under a quiet **Hidden** run instead of
    repeating note headers.
  - Section headers are sticky while scrolling.
  - A per-note header shows a note-kind icon, the locator with its folder part faint and
    middle-truncated so the filename always survives, and a count pill.
  - Note-kind icons: daily `calendar-days`, `ref/` `book-open`, `done/` `archive`, a
    note holding a `^prj` task `folder-kanban`, an inbox note `inbox` (nav `inboxRoute`
    when present, else the inbox file name), and anything else `file-text`.
- **Non-empty filter.** One ranked flat list. Spaces are AND terms. Open matches come
  first, then a quiet `Completed history` divider, then completed matches, which is
  bob's `order_for_picker`.
- **Rows** (34 px, one line, left to right):
  1. **Status pill**, reusing the `bid-tlp-status-pill` tokens:
     - `[ ]` muted, `[/]` In Progress orange, `[*]` Next blue, `[?]` Blocked;
     - `✓` Done and `✕` Cancelled, both dimmed.
  2. **Title.** The display text, rendered safely as DOM nodes and never `innerHTML`:
     - `code` is monospaced on a faint fill;
     - `[[a|b]]` and `[[a#^x]]` are tinted like links, showing the alias or the target;
     - a Markdown link `[label](url)` shows only its label, so Keep tasks read
       `Try Opus with Blender. 💡`;
     - a bare URL shows its host;
     - `📖` stays on reference tasks;
     - matched characters are semibold accent.
  3. **Badges.**
     - `🔒 Blocked` on Blocked rows;
     - a small `eye-off` mark on hidden rows, whose row is also subdued to 55% opacity;
     - a quiet `Done` / `Cancelled` capsule on history rows.
  4. **Locator, drawn as the link it will insert.** The note text is in `--link-color`,
     `#^` is in `--text-faint`, and the ID is in `--text-accent`. ID-less rows show the
     note text plus a dashed `⊕ Add ID` capsule. A guarded row is dimmed and shows its
     short reason badge (for example `archived`, `duplicate ^x`, or
     `the task you're in`).
- **Rendering.** The list renders in windows of 100 rows, adding the next window as
  scrolling or keyboard movement nears the end. End and Mod+↓ render through the last
  row. A 4,300-row vault therefore never builds thousands of DOM rows up front.
- **Empty states.** `No tasks match "foo" · try fewer words`, and
  `No tasks in this vault yet`.
- **Accessibility.**
  - The input is a `combobox` whose `aria-activedescendant` names the selected row. The
    list is a `listbox`, and rows are `option`s with `aria-selected` and
    `aria-disabled`.
  - A polite live region announces the count ("12 of 4,323 tasks") and the selected row.
  - Color is never the only signal: status pills carry glyphs, and badges carry words.
  - `prefers-contrast: more` strengthens fills and borders, and `prefers-reduced-motion`
    removes transitions.
- **Narrow layouts** (under 620 px): the locator keeps only the basename and ID, and the
  footer drops its last hint.

### Detail strip and footer

- **Detail strip.** A fixed 84 px band that never resizes the card. It has three parts:
  1. The full task text, two lines at most, rendered the same way as the row title.
  2. A meta line: status name, note icon and locator, `› section`, and `line N`.
  3. An action line stating exactly what Enter does:
     - identified: `↵ inserts [[sase_cleanup#^no-idle]]`, colored like the locator;
     - ID-less: `↵ names it ^try-opus-blender in mac_inbox.md, then links it`, using the
       JS `suggestPromptBlockId` suggestion;
     - guarded: the full reason, for example
       `Archived tasks can't get an ID · it lives in done/2026/…`;
     - Pomodoro consequences ([Pomodoro mode](#pomodoro-mode)).
- **Footer.** Uses the `bid-tlp-footer` hint styling: `↑↓ move`, `↵ insert link`,
  `esc clear · close`, and `⌫ back to "("`. The last hint shows only while the filter is
  empty.

### Keys

| Key                                             | Behavior                                                                                                                                                                                         |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Printable keys, ←/→, Mod-A/C/V/X/Z in the field | Native filter editing. The note is never touched while filtering.                                                                                                                                |
| ↓ / Ctrl-N / Ctrl-J, ↑ / Ctrl-P / Ctrl-K        | Move the selection, wrapping and skipping section headers.                                                                                                                                       |
| Page Down / Page Up, Mod-↓ / Mod-↑              | Move ten rows, or jump to the last / first row.                                                                                                                                                  |
| Enter, keypad Enter, Tab, click                 | Accept the selected row. A disabled row does nothing (the strip already says why).                                                                                                               |
| Shift-Enter, Alt-Enter, Shift-Tab               | Consumed, no-op.                                                                                                                                                                                 |
| Escape / Ctrl-[                                 | Clear a non-empty filter. Otherwise close: the note and caret are unchanged and `((query))` stays literal.                                                                                       |
| Backspace on an empty filter                    | Undo the trigger as the editor would. Closed `((q))` becomes `()` with the caret inside, and unclosed `((q` becomes `(` with the caret after it. The query fragment goes too. The picker closes. |

### Naming an ID-less task

Accepting an ID-less row hides the card and opens `BlockIdPromptModal` in a new
**wikilink mode**. Its source is `kind: "link-task-complete"`, `trigger: "(("`,
`targetText: <link target>`, and `aliasSuffix: ""`, plus the task row and a picker state
snapshot (query, selected key, scroll).

- **Title:** **Name this task to link it**.
- **Subtitle:** `Adds ^<id> to mac_inbox.md now`. It updates live, and it is honest that
  the note changes immediately.
- **Task preview:** the rendered display text.
- **The ID field.** It is prefilled by `resolvePromptBlockIdSuggestion` and fully
  selected, so Enter takes the good default and typing replaces it.
  - A live line reads `Inserts [[mac_inbox#^try-opus-blender]]`, colored like the
    locator. It is composed for display only; the bytes actually spliced come from the
    same `sourceReplacement` the pipeline uses.
  - Inline validation disables the button and states the problem:
    `Use letters, numbers, and dashes` (`BLOCK_ID_RE`), or
    `^x already exists in mac_inbox.md`, checked against the note's block IDs from a
    fresh read.
- **Buttons:** **Back** (Escape) and **Add ID & link** (Enter, call to action).
  - Back, Escape, and an outside click all reopen the picker exactly where you were. No
    path through the prompt reverts or edits the token. This differs from `^^`, whose
    cancel reverts its marker; keep that branch for `^^` only.
  - Success splices the link, puts the caret after `]]`, and shows the Notice
    `Added ^try-opus-blender to mac_inbox.md`.
  - Failure keeps the prompt open with the reason, and the note is unchanged.
- **Closed tasks.** Done and Cancelled tasks outside `done/` can be named, because
  `planTargetTaskUpdate` appends IDs without activation.
- **Guarded rows never open the prompt.** These mirror bob's `disabled_reason`:
  - `done/` archives: `Archived tasks can't get an ID`;
  - the previous daily note: `The previous daily note is read-only`;
  - duplicate IDs (always guarded): `Duplicate block ID ^x in note.md`;
  - Obsidian-only additions: a note path containing `[`, `]`, `|`, `#`, or `^`
    (`This note's name can't be linked`), and the task line holding the caret
    (`The task you're typing in`).

### Pomodoro mode

Pomodoro mode applies when the token is the sole content of a sub-bullet under an
**open** Pomodoro entry, which is the existing `sourceQualifiesForPomodoroActivation`
test applied to the token range. The inserted link is then a Task Link, so the card
shows and enforces Task Link rules. A link under a closed Pomodoro is logged history and
uses the ordinary mode.

- **Header.** The icon tile shows 🍅. The title is **Link a task to this Pomodoro**. The
  subtitle is `FIX · 09:20–09:50 · Ready tasks become Next`, using the entry's name and
  time (or `planned` for a `()` placeholder).
- **Pool.** Completed history is omitted, because closed tasks can't join a Pomodoro.
  The count reads `1,482 open tasks`.
- **Action lines** add consequences, joined with `·`:
  - `↵ links it here`;
  - `Ready → Next`;
  - `pulls it into today (was scheduled Nov 4)`, when exactly one future `scheduled`
    would be removed;
  - `logs the schedule change`;
  - `type becomes FIX`, `type cleared (inherits WRITE)`, or `keeps WRITE`, from Ledger
    Tools `themes.suggestFor` and `planRetype`, exactly as Link to today computes them.
    Omit the type clause when the themes API is absent.
- **Guards:**
  - **Already linked in this entry:** disabled with `Already in this Pomodoro`.
  - **Project-blocked** (`#prj_is_blocked`, an inherited project block, or a project
    date): disabled with the same wording the consumption notices use, for example
    `Blocked by its project projects/foo · move the ^prj Schedule`.
  - **Prerequisite-blocked:**

> [!decision] pomodoro_blocked = raise
>
> The row is selectable, with a `🔒 waits on 2` badge. Its action line reads
> `↵ raises 2 prerequisites to Next · it stays unlinked`.
>
> Enter runs PDG1 consumption for that task, the same `planTaskLinkBlockConsumption`
> plan and per-file `applyTargetTaskPlan` writes as `consumeBlockedTaskPomodoroLink`. It
> feeds that plan only the task-bearing documents the index knows, not all 6,500 notes.
>
> The picker then stays open:
>
> - the touched notes are refreshed in the index;
> - a pinned **Raised for ‹task›** section lists the raised prerequisites, now Next,
>   with the first one selected;
> - a header banner reads `Raised 2 prerequisites to Next · pick one to link`;
> - the Notice uses Ctrl+Shift+Enter's wording.
>
> If dependency status can't be checked, nothing is written and the existing "could not
> be checked" Notice shows.

> [!decision] pomodoro_blocked = refuse
>
> The row is disabled with `Blocked by 2 prerequisites · link one of them instead`, and
> the picker never writes other tasks.

- **Commit.** Use the Ctrl+Shift+Enter target plan: `planTargetTaskUpdate` with
  `forceNext: true`, the retype `themeEdit`, a freshness stamp, and a Schedule Log entry
  when the marker exists. Then call `completeTaskSourceLink`, which also performs
  future-link cleanup. A re-check at commit repeats every guard against fresh content.
- **Notice.** Mirrors Link to today, for example
  `Linked under FIX · type FIX · set Next · plan 3/3 · 7/10`. It includes the
  plan-budget suffix and `futureLinkCleanupNoticeSuffix`.
- **Non-goals here.** The keep-type `⌥↵` merge and the inbox-route gate stay on
  Ctrl+Shift+Enter.

> [!decision] caret_parity
>
> `^^` completions under an open Pomodoro route through the same Pomodoro gate and
> commit plan. Closed targets are refused, prerequisite-blocked targets are consumed
> (PDG1, and the `^^` marker reverts as on cancel), project-blocked targets are refused,
> and the target plan uses `forceNext` with the retype. This brings the code in line
> with `decisions:capture-link-drops-blocking-deps` ("Plugin gestures (Ctrl+Shift+Enter,
> `^^`) keep PDG1 consumption"). If declined, `^^` keeps today's `activationEligible`
> plan unchanged and only `((` uses the gate.

### Notices (complete list)

- **Identified insert, ordinary mode:** none.
- **ID added:** `Added ^id to note.md`. In Pomodoro mode it is followed by the Pomodoro
  clause.
- **Pomodoro link:** as above.
- **Stale target:** `Task wikilink not inserted: ‹reason›`. Reasons:
  - `note.md no longer has ^id`;
  - `^id is now duplicated in note.md`;
  - `the task line changed in note.md`;
  - `the ((…)) text changed`.

  The picker stays open, and its rows refresh.

## Phase `model`: shared rules and parity vectors

**bob-cli (this repo).**

- Add `tests/fixtures/task_wikilinks/vectors.json` in the style of
  `tests/fixtures/task_mentions/vectors.json`.
- Each case has `name`, `now` (pinned via `BOB_NOW` or a date parameter), `vault` (a
  path → content map, including a Tasks `data.json` whose global filter is `#task`),
  `owner` (the note path with `.md`), `query`, and the expected ordered `rows`. Each row
  records `note_path`, `line`, `group`, `hidden`, `block_id`, `locator`, `link_target`,
  `replacement`, `disabled_reason`, `status_symbol`, and `text`.
- Cover at least:
  - every group and the hidden-last rule in each;
  - per-note grouping order and path/line tiebreaks;
  - open-before-completed at every score;
  - AND terms, each match tier, and score-tie canonical order;
  - all excluded directories (dot-dirs, `_conflicts`, `_generated`, `_templates`, and
    `.bob`);
  - `ref/`, daily, and `done/` notes;
  - a duplicate basename (path target), an archive path, and a unique basename;
  - every guard: a duplicate ID, an ID-less `done/` row, and an ID-less previous-daily
    row;
  - ID-less rows having an empty `replacement`;
  - status symbols ` `, `/`, `*`, `?`, `x`, `X`, and `-`;
  - `#task #ref` lines;
  - a CRLF note;
  - a multi-byte title.
- Add a Rust test beside `src/native/capture_task_wikilinks.rs`, loading the fixture the
  way `task_relations/mentions.rs` does. For each case it builds a temp vault and
  asserts `candidates()` plus `cross_note_link_target` against the expected rows. The
  test is the contract: if bob's behavior differs from a hand-written expectation, read
  the code and fix the vector, never bob.
- Commit as `test(capture): …`. This is test-only, so do not touch `Cargo.toml` or
  `CHANGELOG.md` (`decisions:bob-cli-releases-ride-a-daily-train`).
- Gate: `just check`.

**bob-plugins** (open with `/sase_repo`; read its `AGENTS.md`).

- Mirror the fixture verbatim to `scripts/fixtures/task_wikilinks/vectors.json`. The JS
  test header records the bob-cli source path and the file's sha256, and the test
  asserts that sha256 so local edits fail loudly.
- Add pure fragments, for example `src/031-task-wikilink-rules.js` and
  `src/032-task-wikilink-rank.js`:
  - `parseNoteTaskRows(path, content)`. It returns all `#task` checkbox lines of every
    status outside frontmatter and fences, with `line` (0-based internally and 1-based
    on the wire like bob), `rawLine`, `status`, `blockId` (a trailing `^id` only),
    `hidden`, `prjBlocked`, `section` (the nearest heading), and `depth`. It also
    returns `searchText`, equal to bob's task `description` for the ranker, and
    `displayText` (`cleanTaskDisplayText`) for presentation. It flags `duplicateId`
    within the note.
  - `isTaskWikilinkNotePath(path)`, mirroring `collect_note_paths` and
    `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES`.
  - `buildBasenameIndex(paths)`, with `locatorFor(path)` and `linkTargetFor(path)`
    mirroring `locator_for` and `cross_note_link_target` (case-insensitive uniqueness,
    archive keeps its path).
  - `groupFor`, `canonicalKey`, and `matchScore` / `searchFields`, mirroring
    `capture_link_tasks::match_score` and `dependency_search_fields` (the five tiers and
    AND terms; `route:id`, `id`, text, locator, note path, section).
  - `orderForPicker(rows, query, ownerPath)`.
  - `disabledReasonFor(row, {previousDailyPath})`, mirroring
    `capture_task_wikilinks::disabled_reason` and `previous_daily_note` (no read-only
    check in JS; a failed write reports instead).
  - `previousDailyNotePath(paths, now)`.
- Add `scripts/test-block-id-prompt-task-wikilink-model.cjs`. It runs every vector
  through the JS model with deep equality, and adds unit cases for parsing edge cases.
  Add it to the explicit `package.json` test list.
- Gates: `npm run build`, `npm test`, and `npm run validate`, then `bob plugins sync`
  per `AGENTS.md` (`--repo <checkout> --no-pull` for the opened checkout, per
  `docs/develop/deployment.md`).

## Phase `trigger`: note grammar

bob-plugins only. This phase changes no behavior yet: nothing opens until `picker` wires
it.

- Add a pure fragment, for example `src/022-task-wikilink-token.js`:
  - `findTaskWikilinkToken(lineText, ch)` returns
    `{opener, closer, queryStart, queryEnd, replaceStart, replaceEnd, query, removal: {start, end, insert, caretCh}}`
    or `null`, implementing [Trigger](#trigger) exactly.
  - `taskWikilinkContextAt(docLines, line, ch)` classifies the multi-line and inline
    exclusions and the managed and Pomodoro-entry lines. It returns `{allowed, notice}`,
    where `notice` is the managed line's guard text.
  - `taskWikilinkWrittenSecondParen(token, writtenRanges, lineStartOffset)`.
  - A small pure written-range accumulator (`addChanges(ranges, changeSet)`,
    `mapRanges(ranges, changeSet)`) that tests can drive with fake change sets.
- Add `scripts/fixtures/task_wikilinks/trigger_vectors.json` and
  `scripts/test-block-id-prompt-task-wikilink-token.cjs`. Mirror bob's
  `capture_task_wikilinks` unit cases for the shared core (openers, `(((`, closers, stop
  bytes, mid-word caret, query clamping, and multi-byte text before the token; JS
  offsets are UTF-16 `ch`). Add every note-only exclusion, the auto-pair shapes `((|))`
  and `((fo|))`, and the written-paren rule (typed, pasted, burst, undo, programmatic,
  caret-only, and Backspace-into).
- Gates as in `model`.

## Phase `index`: warm vault task index

bob-plugins only.

- Add a fragment, for example `src/036-task-wikilink-index.js`, with a
  `TaskWikilinkIndex` (pure core plus a thin Obsidian adapter):
  - **Store.** `Map<path, {mtime, size, rows, blockIds}>` plus the basename index. Keep
    rows and each note's block-ID set (for the detail strip's `suggestPromptBlockId`
    preview), not note texts.
  - **Warm-up.** Start at `workspace.onLayoutReady`, in idle chunks of at most 12 ms
    (`requestIdleCallback`, falling back to `setTimeout`).
    - Prefilter with `metadataCache.getFileCache(file)` and skip files with no task list
      items. Read the rest with `vault.cachedRead`. A file with no cache yet is read
      anyway.
    - It must never block the UI for more than 16 ms.
  - **Maintain.**
    - `metadataCache.on("changed", (file, data))` re-parses from `data` with no extra
      read.
    - `vault.on("rename" | "delete")` re-keys or drops rows and recomputes the basename
      index.
    - Use `registerEvent` everywhere.
  - **Open-time validation.** `snapshot({ownerPath, overlays})` compares every indexed
    path's `stat.mtime` and `stat.size` with the vault and queues changed files for
    background re-parse. It overlays live text from every open `MarkdownView` editor,
    always including the source editor. It then returns the ordered model input
    immediately, with `{ready, scanned, total}` progress and a change subscription for
    streaming repaints.
  - **Reads.** Expose `taskBearingDocuments()` (path plus fresh text, read on demand)
    for PDG1 consumption, and `refreshPaths(paths, texts?)` for post-write refresh.
  - **Warnings.** Bounded unreadable-file warnings, surfaced as a quiet footer note.
- Add `scripts/test-block-id-prompt-task-wikilink-index.cjs` and
  `scripts/test-block-id-prompt-task-wikilink-index-performance.cjs`, in the style of
  `test-navigation-dependencies-stage-performance.cjs`. Use a synthetic 6,500-note,
  5,000-task fake vault and assert:
  - the cold build completes in at most 1.5 s, in chunks of at most 16 ms;
  - a warm `snapshot` plus order takes at most 50 ms;
  - one keystroke's rank of 5,000 rows takes at most 12 ms;
  - no vault reads happen while filtering;
  - event-driven updates (changed, rename, delete) and mtime-driven revalidation work;
  - overlays win over disk;
  - growth stays near-linear (in the style of the 6× step check).

  Stub the Obsidian APIs as `scripts/block-id-prompt-harness.cjs` does.

- Gates as in `model`.

## Phase `picker`: card and identified-link insert

bob-plugins only. This is the first phase users see.

- **Wiring** (`src/100-plugin-lifecycle.js` plus a new mixin, for example
  `src/128-plugin-task-wikilinks.js`):
  - `scheduleScan` accumulates written ranges from `update.transactions` and
    `update.changes`.
  - `inspectActiveEditor` checks the `((` token before the `^^` and marker branches.
    `((` and `^^` never overlap, because `^^` needs `[[`.
  - Open `TaskWikilinkPickerModal` synchronously from the index snapshot. Don't await
    I/O before `open()`.
  - Register `link-task-wikilink`. The index starts in `onload`, and its events are torn
    down in `onunload`.
- **Modal** (for example `src/087-task-wikilink-picker-modal.js` for state and keys, and
  `src/088-task-wikilink-picker-view.js` for pure row, section, and detail render plans
  plus DOM builders), implementing [The card](#the-card),
  [Detail strip and footer](#detail-strip-and-footer), and [Keys](#keys). Selection,
  scroll, and query survive streaming repaints and the round trip through the ID prompt.
- **Identified accept** (ordinary mode):
  1. Verify the token text at its range.
  2. Read the target authoritatively and confirm `^id` occurs exactly once as a block
     ID.
  3. Confirm `metadataCache.getFirstLinkpathDest(linkTarget, sourcePath)` resolves to
     the row's path. If it does not, use the path form.
  4. Replace the token in one editor transaction (`editor.transaction` with one change),
     so one undo restores `((query))`. `suppressEditorScans()`, then put the caret after
     `]]`.
  5. Show no Notice.

  On any failure, show the stale Notice, refresh that note, and keep the picker open.

- **Escape and Backspace** behave as specified. They never call `^^`'s
  `revertTaskPickerMarker`.
- **ID-less rows** in this phase open nothing yet: the action line reads as specified,
  and Enter is a no-op until `naming`. Pomodoro sources use the ordinary mode until
  `pomodoro`. Guarded rows are disabled.
- **CSS:** `bid-twl-*` in `plugins/block-id-prompt/styles.css`, following the
  conventions of the existing `bid-tlp` block (variables only, a reduced-motion guard,
  and a `max-width: 700px` media block).
- **Tests:**
  - `scripts/test-block-id-prompt-task-wikilink-picker-view.cjs` covers the render
    plans: groups, the Hidden run, sticky header data, row anatomy and locator parts,
    action lines, counts, empty and indexing states, windowing, and narrow layout.
  - `scripts/test-block-id-prompt-task-wikilink-picker-runtime.cjs`, through
    `scripts/modal-harness.cjs` and `block-id-prompt-harness.cjs`, covers:
    - auto-open only on a written second `(`, including the auto-pair and closed shapes;
    - seeding;
    - every key in the table;
    - Escape leaving the text;
    - Backspace for both shapes;
    - accept, with the caret and a single change;
    - the stale paths keeping the picker open;
    - the command's three entry modes;
    - managed-line notices;
    - `^^` regressions unchanged.
- **Docs:**
  - Add a **Linking any task with `((`** section to `docs/block-id-prompt.md` covering
    the trigger, the card, keys, and how `((` differs from `^^` (vault-wide, includes
    history, and names the note in every link).
  - Bump `plugins/block-id-prompt/manifest.json` (minor) and add one sentence about `((`
    to its description.
  - Run `npm run docs:gen`; the facts and commands tables are generated.
  - Run `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`.
- Gates as in `model`, plus the docs check.

## Phase `naming`: name-then-link

bob-plugins only.

- **`BlockIdPromptModal` wikilink mode** (`src/090-prompt-modals.js`). When
  `source.trigger === "(("`, render [Naming an ID-less task](#naming-an-id-less-task):
  the title, live subtitle, preview, prefilled and selected field, live insert line,
  inline validation, and the **Back** / **Add ID & link** buttons. Other kinds render
  exactly as today.
- **Lifecycle.** `cancelBlockIdPrompt` and `onClose` reopen the picker from
  `source.pickerState` for `((`, and keep the `^^` revert for `^^`.
- **Pipeline** (`submitLinkTaskBlockId`, `src/110-plugin-block-id-submit.js`, and
  helpers in 140):
  - Accept closed (non-archived) tasks.
  - Relocate a moved task line by a unique exact `rawLine` match in the fresh content
    before refusing.
  - Resolve the target from `targetText` and assert it is the row's path.
  - Emit the `Added ^id to note.md` Notice for `((`. Keep `^^` notices unchanged.
- Enable Enter on ID-less rows in the picker.
- **Tests:** `scripts/test-block-id-prompt-task-wikilink-naming.cjs`. Cover:
  - prefill and selection;
  - live line and validation;
  - success in the same note and across notes (including `adjustSourceForPlan` line
    shifts);
  - Back, Escape, and outside-click restoring the picker state;
  - a stale line relocated, or refused when ambiguous;
  - a closed-task ID;
  - guarded rows never prompting;
  - `^^` prompt behavior unchanged.
- **Docs:** extend the `((` section. Gates as in `picker`.

## Phase `pomodoro`: Task Link rules

bob-plugins only.

- **Pomodoro mode.** Detect it at open and re-check it at commit. Implement
  [Pomodoro mode](#pomodoro-mode): the header, pool, action-line consequences,
  duplicate-in-entry guard, project and blocked handling per `pomodoro_blocked`, commit
  through the Ctrl+Shift+Enter target plan, the retype through
  `resolvePomodoroLinkThemeWrite` (fed the picked task's location and text), and the
  Notice with the plan budget.
- **Shared helper.** Factor the gate and target plan into one helper that both `((` and
  (per `caret_parity`) `^^` call from `completeTaskLinkWithExistingId` and
  `submitLinkTaskBlockId`.
- **Tests:** `scripts/test-block-id-prompt-task-wikilink-pomodoro.cjs`. Cover:
  - qualifying and non-qualifying sources (closed Pomodoro, prose bullet, top-level
    line);
  - each consequence clause;
  - the retype set / clear / keep outcomes, and absence when the themes API is missing;
  - the duplicate guard;
  - project refusals;
  - the blocked branch selected by `pomodoro_blocked` (raise: writes, index refresh, the
    pinned section, the banner, and a later link of a raised prerequisite);
  - future-link cleanup;
  - stale re-check.
- **Regressions.** Extend `test-block-id-prompt-pomodoro-context.cjs` and
  `test-block-id-prompt-target-update.cjs` for the `^^` behavior that `caret_parity`
  selects.
- **Docs:** `docs/block-id-prompt.md`. Add the Pomodoro-mode subsection, and update the
  `^^` / Guards text if `caret_parity` changes `^^`. Gates as in `picker`.

## Phase `review`: real-Obsidian verification, polish, docs

"Beautiful" and "reliable" are verified on real renders and in real Obsidian, not
assumed. athena has `/snap/bin/obsidian` and `/usr/bin/google-chrome`. Run there over
SSH (see the `tailnet` reference memory). If a prerequisite is missing, say so; never
convert a skipped check into a pass.

- **Static previews.** Add `scripts/preview-task-wikilink-picker.cjs` with a
  `.template.html`, in the style of `scripts/preview-command-mode.cjs`. It renders the
  production view fragments and `styles.css` in Chromium with Obsidian's default light
  and dark variables. Capture 9 states × 2 widths (900 and 600) × 2 themes, 36 PNGs:
  1. grouped, empty filter;
  2. filtered with highlights;
  3. ID-less selected;
  4. guarded selected;
  5. completed history;
  6. indexing;
  7. Pomodoro mode with retype and blocked rows;
  8. the "Raised for" state;
  9. the ID prompt in wikilink mode.

  Look at every PNG. Fix alignment, truncation (the filename must survive), locator
  color balance, dark-mode contrast, header and footer rhythm, and consistency with the
  `^^` card. Repeat until the card reads calm and consistent.

- **Real Obsidian e2e.** Add `scripts/obsidian-task-wikilink-e2e.cjs`, reusing
  `scripts/obsidian-startup-smoke-lib.cjs` (isolated scratch vault and profile, owned
  process, CDP, never the live vault). Run it under `xvfb-run -a` with vim mode on and
  bracket auto-pair at its default, in Live Preview and in Source mode. Drive real key
  events through `Input.dispatchKeyEvent` and assert note text and caret through CDP:
  - typing `((` auto-opens the picker on the auto-paired `((|))`;
  - a burst `((opus` seeds the filter;
  - Enter inserts the expected link, the caret lands after `]]`, and one undo restores
    `((opus))`;
  - Escape leaves the text, and a further keystroke does not reopen;
  - Backspace leaves `()`;
  - an ID-less task is named end to end in both notes;
  - Pomodoro mode in a daily note links, promotes, and retypes;
  - a fenced-code `((` and a Depends-On `((` never open.

  Take `Page.captureScreenshot` of the live picker in light and dark, and review those
  too. Also run
  `xvfb-run -a npm run smoke:obsidian -- --report /tmp/obsidian-smoke.json` and record
  the verdict.

- **Docs.**
  - Do a final accuracy pass of `docs/block-id-prompt.md` against the code.
  - In bob-cli, read the `docs.md` reference memory first, then add one sentence to step
    1 of `docs/guides/see-task-mentions.md` saying the fastest way to write such a link
    is `((` in Obsidian or Bob Mac Capture.
  - Run `just docs-check` in bob-cli (with `BOB_DOCS_BOB_PLUGINS_DIR` per the plugins'
    `docs:check`) and `npm run docs:check`.

> [!decision] decision_record
>
> Through `/sase_memory_write`, add the bob-cli decisions strand
> `decisions:obsidian-task-wikilinks-mirror-bob` and run `sase memory init`.
>
> - **Applies to:** bob-plugins, bob-cli.
> - **Claim:** Obsidian `((` mirrors bob's `wikilink_task` pool, order, link-target, and
>   guard rules in plugin JS, pinned by `tests/fixtures/task_wikilinks/vectors.json`,
>   which both repos assert. The plugin never spawns `bob` for it.
> - **Why:** mobile, unsaved buffers, instant opens, and plugin precedent. The rejected
>   alternatives are a `bob` subprocess and an inline `EditorSuggest`.
> - **Cost:** two implementations kept in step by vectors.
> - **Reopens when:** the plugins gain an in-process bob library, or the vectors keep
>   drifting.
> - **Evidence:** this plan and the phase commits.
>
> When this decision is off, write nothing to memory.

- **Deploy.** Run `npm run build`, `npm test`, and `npm run validate`, then
  `bob plugins sync` per `AGENTS.md`. Record each command's verdict in the final report.

## Gates and conventions (every phase)

- Before editing bob-plugins, open it with `/sase_repo` and follow its `AGENTS.md`:
  - edit `src/` fragments, run `npm run build`, and never hand-edit `main.js`;
  - keep fragments at or under 1000 lines;
  - list new tests explicitly in `package.json`;
  - run `bob plugins sync` after changes.
- Commit types: bob-plugins `feat(block-id-prompt): …` (or `test(…)`); bob-cli
  `test(capture): …` / `docs(guides): …`.
- Never read Bryan's vault for fixtures. All fixtures are synthetic.
- If a phase discovers out-of-scope work, record it as `PROPOSED FOLLOW-UP:` on its
  phase bead.

## Rollout and risks

- **Order.** `model` and `trigger` run in parallel, then `index`, `picker`, `naming`,
  `pomodoro`, and `review`. Users first see `((` at `picker`, which already inserts
  links to identified tasks. Until `naming` lands, an ID-less row explains itself
  instead of acting.
- **Auto-pair and vim.** Bryan's vault runs `vimMode: true` with default bracket
  auto-pair. The closed-token grammar and the e2e cover both. Vim `(` motions in normal
  mode are not edits, so they never trigger.
- **Index staleness.** Stale rows are bounded three ways: mtime revalidation at open,
  `changed` events, and verify-before-write. The worst case is a refused insert with an
  accurate Notice and refreshed rows.
- **Plugin interplay.**
  - Nav's mention convergence and ledger chips react to the inserted link as to any
    typed link.
  - Obsidian's own `[[` suggester does not reopen, because the caret lands after `]]`.
  - `suppressEditorScans` keeps the plugin from re-scanning its own splice.
- **Performance.** The budgets above are enforced by the index performance test, and the
  card renders in windows.
- **Divergence from bob.** Any deliberate difference is Obsidian-only and listed here:
  - same-note links still name the note (bob does the same for drafts);
  - read-only notes are detected at write time, not as a row guard;
  - two extra guards (unlinkable note names, and the task holding the caret).

## Non-goals

- Link aliases (`[[note#^id|text]]`), embeds (`!((`), and heading or note links.
- Any change to Obsidian's native `[[` suggester, to `^^` outside the `caret_parity`
  decision, or to the Mac `((` contract.
- An inline `EditorSuggest` variant, a default hotkey for **Link a task here**, a
  cross-plugin `api.taskWikilinks`, and an editor decoration for unresolved `((` tokens.
- In Pomodoro mode, the keep-type `⌥↵` merge, the inbox-route gate, and review-walk
  advancement. These stay on Ctrl+Shift+Enter.
- Changing the JS block-ID suggestion algorithm (`015-prompt-block-id-naming.js` already
  skips URLs and link destinations).
