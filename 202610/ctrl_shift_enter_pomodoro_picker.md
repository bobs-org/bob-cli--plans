---
tier: epic
title: Ctrl+Shift+Enter Pomodoro picker
goal: 'Ctrl+Shift+Enter on an unlinked task asks which of today''s Pomodoros to link
  it into: Enter takes the current/first future Pomodoro, typing filters, and a new
  name creates a Pomodoro with bob capture''s rules, in a reliable, stale-safe, and
  polished picker.

  '
phases:
- id: core
  title: Pomodoro target core (pure model and explicit-target planner)
  depends_on: []
  size: medium
  description: 'core: add pure block-id-prompt helpers for capture-parity Pomodoro
    names, today''s open-entry model, ranked picker rows, the create intent, and an
    explicit-target link planner (existing or new named entry), with unit tests; no
    behavior change yet.

    '
- id: picker-ui
  title: Pomodoro link picker modal and styles
  depends_on:
  - core
  size: medium
  description: 'picker-ui: build the promise-based "Link to today" modal with timeline
    rows, progress ring, create/invalid/blocked rows, plan-meter footer, keys, accessibility,
    bid-ppk styles, and DOM-stub view tests; not yet wired.

    '
- id: gesture
  title: Wire the picker into Ctrl+Shift+Enter, notices, docs, release
  depends_on:
  - core
  - picker-ui
  size: medium
  description: 'gesture: move the inbox-route helpers out of the full fragment, add
    the picker mixin and preflight, route the choice through the link flow, name the
    destination in Notices, update the harness and tests, add runtime tests, bump
    to 1.24.0, update docs in both repos, and sync.'
proposed_by: bbugyi200.apollo.5h
create_time: 2026-10-07 09:22:14
status: wip
bead_id: bob-cli-54
---

- **BEAD:** [bob-cli-54](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-54/README.md)

# Ctrl+Shift+Enter asks which Pomodoro: the "Link to today" picker

## Goal

Today, `Ctrl+Shift+Enter` on an unlinked open `#task` line (block-id-prompt's
`link-task-to-pomodoro` command, `openPomodoroTaskLink`) silently links the task under
today's implicit current/next Pomodoro. Make the link direction **ask which Pomodoro**
in today's daily note, with:

- `↵` on open taking the current Pomodoro, else the first future one — exactly today's
  implicit target, so the old behavior stays one key away;
- type-to-filter over today's open Pomodoros;
- typing a name that does not exist offers **New Pomodoro NAME**, created with the same
  rules `bob capture '@route:id#name'` uses.

The design goals are intuitive (one picker, always first, `↵` = today's behavior),
reliable (pure planners, stale-safe commit, capture parity, nothing written on any
cancel or refusal), and beautiful (a compact timeline picker with a live progress ring
on the running Pomodoro and a plan-cap preview on the create row).

## What does not change

- The **unlink** direction (task already linked under an open Pomodoro), the In Progress
  Work Log prompt, task-link mode (cursor on a Task Link), Depends-On refusals, the `^^`
  task picker, and `Ctrl+6` are untouched and never open the picker.
- Lane rules (Ready/Blocked → Next, Next/In Progress stay), future-schedule
  pull-forward, Schedule Log entry, freshness stamp, inbox routing, and review-walk
  advance semantics are unchanged.
- No decision record changes: the `answering-advances-the-walk` record still holds (a
  committed link advances once; `Esc` in the picker stays).

## Repositories

- **bob-plugins** (linked repo): all code, styles, tests, the plugin README row. Open it
  with the `/sase_repo` skill (`sase repo open bob-plugins`) and work only in the path
  it prints. Read its `AGENTS.md` first. Plugins with `src/fragments.json` are built
  from `src/` fragments: edit fragments, run `npm run build`, never hand-edit `main.js`;
  every hand-edited fragment must stay at or below 1000 lines. All fragments share one
  script scope, and `main.js` is not strict mode, so a duplicate top-level `function`
  name silently overrides the earlier one: `grep` the plugin's `src/` before adding any
  top-level name. Add every new test file to the explicit `test` list in `package.json`.
  After any change: `npm run build`, `npm test`, `npm run validate`, then
  `bob plugins sync` to deploy to the vault.
- **bob-cli** (primary repo): docs only, in the final phase.

## UX specification

### Gesture flow (link direction)

1. `Ctrl+Shift+Enter` resolves the task and link presence exactly as today. A linked
   task takes the unchanged unlink path.
2. **Preflight** (new, before any modal): resolve today's daily note and read its
   snapshot (the live editor value when the task note _is_ the daily note). Refuse with
   today's existing Notice texts and open nothing when the note is missing
   (`Task link blocked: today's daily note could not be found`), unreadable
   (`Task link blocked: <path> could not be read`), or has no `## Pomodoros` section
   (`Task link blocked: <path> has no Pomodoros section`).
3. **Pomodoro picker** opens. `Esc`/`Ctrl+[`/click-outside cancels: nothing is written,
   no Notice, and a review-walk origin settles with `null` (stays).
4. On a choice, the target rides on the link source (`source.pomodoroTarget`), then the
   existing flow continues unchanged: the block-ID prompt when the task has no ID (its
   `Esc` still cancels the whole toggle), then the inbox route picker on inbox notes,
   then the write.

Modal order is therefore always **Pomodoro picker → (block-ID prompt) → (inbox route
picker) → write**: the picker is the one constant first step of the link gesture, and
cancelling it can never leave a half-assigned block ID.

### Picker anatomy

- **Header**: accent icon tile (lucide `timer`), title `Link to today`, subtitle = the
  task's display text (ellipsized). Right side: a lane chip — `Ready → Next`,
  `Blocked → Next` (green-tinted), `stays Next`, `stays In Progress` (muted) — plus a
  small accent-outline `+ block ID` chip when the task has no block ID (tooltip
  `You'll name the block ID next`).
- **Search**: placeholder `Filter Pomodoros or name a new one`.
- **Warning banner** (only with more than one open timed entry):
  `2 Pomodoros are running — pick one; new Pomodoros are off until one finishes`.
- **Rows**, today's open Pomodoros in document order (the day's timeline):
  - _Glyph_: the running entry shows a **progress ring** (conic-gradient filled to the
    elapsed fraction of its time range) with a softly breathing center dot; the next-up
    entry a 2px accent ring; other open entries a faint hollow ring; the create row a
    dashed accent ring with a `plus`; invalid or blocked rows `circle-alert` in orange.
  - _Title_: the entry's name (ALL-CAPS, semibold, slight letter-spacing); unnamed
    entries fall back as defined in the core phase and render muted italic.
  - _Meta_ (muted, `·`-separated, tabular numbers): time range `09:20–09:50` when timed;
    `15m left` (accent) or `5m over` (orange) on the running entry; `3 tasks` / `1 task`
    / `empty`; then up to two child Task Link block IDs and `+k`
    (`fix-flaky, web-capture +1`).
  - _Badges_: `Running` (accent tint) or `Next up` (accent outline).
  - The selected row has the accent left rail + tint (same language as the `^^` picker)
    and a `corner-down-left` keycap at the far right.
- **Create row** (bottom, separated by a dashed divider when matches precede it): title
  `New Pomodoro NAME`; meta `after CAPTURE · becomes next up`, `after PLAN`, or
  `first in Pomodoros`, and `PLAN is done — starts a fresh PLAN` for a completed-only
  name; badge `+1 theme` (muted), turning red `+1 theme · 4/3` when it would exceed the
  plan theme cap.
- **Invalid row** (typed text is not a valid name and nothing matches):
  `Names use A–Z, 0–9, spaces and & ' ( ) + , . / -`, meta
  `Start with a letter or digit`. **Blocked row** (valid new name while several
  Pomodoros run): `Can't add a Pomodoro while 2 are running`. Neither is selectable.
- **Empty state** (no open Pomodoros, empty query): `timer-off` icon,
  `No open Pomodoros today`, `Type a name to plan one`.
- **Footer**: key hints left — `↑↓ select`, `↵ link`, `⇧↵ new NAME` (shown only when the
  typed text has a create intent), `esc cancel`; the current plan meter right
  (`plan 2/3 · 6/10`, red when over, `aria-live="polite"`), omitted when
  bob-ledger-tools' `planBudget` api is unavailable.

### Keys

| Key                        | Action                                                                                                                      |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `↑`/`↓`, `Ctrl+P`/`Ctrl+N` | Move selection (wraps; skips non-selectable rows)                                                                           |
| `↵`                        | Choose the selected row (existing → link there; create row → create and link)                                               |
| `Ctrl+Shift+Enter`         | Same as `↵` — so a habitual double tap links to the current/next Pomodoro                                                   |
| `⇧↵`                       | Create from the typed text even when other rows match; an exact open name links there instead (never a duplicate open name) |
| `Esc`, `Ctrl+[`            | Cancel; nothing written                                                                                                     |
| Click                      | Choose that row                                                                                                             |

Clearing the query restores the default selection. Mobile works by tap; the modal is
responsive (badges wrap under the text below ~600px).

### Notices

Picker-driven links name the destination: `Linked to CAPTURE · Next · plan 2/3 · 7/10`,
`Linked to SASE · stays In Progress`, `Linked to new TAXES · Next · plan 4/3 · 7/10 🔴`.
Existing chips (removed future schedule, logged schedule change, removed current/future
links), the plan suffix, the inbox `· moved to <dest>` suffix, and the review-walk
`link-today` outcome keep their current shape. Unlink texts are unchanged.

## Semantics (shared contract for all phases)

- **Entry recognition** is block-id-prompt's own (`findPomodorosSectionRange`,
  `computeFencedLineFlags`, `isPomodoroEntryLine`, `pomodoroEntryStatus`,
  `pomodoroEntryEndLine`), so the picker can only offer entries the planner can resolve.
  Open = `isOpenPomodoroStatus`. Cancelled and completed entries are never rows.
- **Default** mirrors `selectPomodoroInsertionTarget`: the single open timed entry, else
  the first open entry. With several open timed entries, the first of them is
  highlighted but flagged ambiguous (no `Running` vs `Next up` distinction claimed;
  banner shown); an explicit pick is allowed, matching capture ("an existing open match
  is explicit").
- **Next up** mirrors capture's `next_future_pomodoro`: the first open, placeholder,
  untimed entry.
- **Name grammar** mirrors bob capture (`canonicalize_pomodoro_name`,
  `is_pomodoro_name`, `selector_slug`): collapse whitespace runs to one space, trim,
  ASCII-uppercase; valid when the first char is `A-Z`/`0-9`, the rest are `A-Z 0-9`,
  space, tab, `& ' ( ) + , . / -`, and at least one letter is present. Slug =
  whitespace-split words, ASCII-lowercased, joined with `-`.
- **Ranking** for a non-empty query: exact name slug > name slug prefix > title/label
  substring > other substring (time in `0920-0950` and `09:20–09:50` forms, `#N`, child
  link block IDs), stable by document order; so the top hit for a typed name is exactly
  what capture's `#name` would select. The create row is appended when the canonical
  name is valid and no open entry has that exact canonical name.
- **Creation** mirrors capture's named-future-entry rule: the new `- [ ] () — NAME`
  entry (with the Task Link as its only child) goes after the running entry's block;
  else after the last completed (`[x]`/`[X]`, never `[-]`) entry's block; else before
  the first open entry; else at the top of the section. "After a block" means after its
  last nonblank descendant line, before trailing blank separators. Several open timed
  entries make the anchor ambiguous, so creation is refused. Child indentation reuses
  the anchor's child indent, else any child indent in the section, else `\t`
  (`CANONICAL_CHILD_INDENT_UNIT`; capture's fallback is two spaces — this plugin keeps
  its canonical tab).
- **Commit is stale-safe**: an existing choice re-resolves in the fresh daily snapshot
  by `(entryLine, entryText)`, else by a unique identical entry line elsewhere; a
  missing or ambiguous target refuses, and a target that closed meanwhile refuses. A new
  choice whose exact canonical name became open meanwhile links into that entry (capture
  parity, no duplicate). Every refusal happens before any write, so nothing is written —
  not even a new block ID.
- **Cleanup for explicit targets**: matching links under every _other_ open Pomodoro are
  removed (in the link direction there normally are none; this keeps "linked under
  exactly the chosen open Pomodoro" true under races). An already-linked destination is
  an idempotent no-op insertion, as today. The implicit (no-target) planner path keeps
  today's later-only cleanup.

## Phases

### Phase: Pomodoro target core (pure model and explicit-target planner)

All in bob-plugins `plugins/block-id-prompt`. No user-visible behavior change; the
gesture still links implicitly after this phase.

1. Add `src/075-pomodoro-link-targets.js` (insert after
   `070-daily-and-activation-plans.js` in `src/fragments.json`). Pure functions only (no
   Obsidian APIs). If it would exceed 1000 lines, split it into
   `075-pomodoro-link-target-model.js` and `076-pomodoro-link-target-plans.js`.
   - `POMODORO_NAME_USAGE` (capture's message text),
     `canonicalizePomodoroLinkName(raw) → { valid, name, error }`,
     `isPomodoroLinkName(name)`, `pomodoroLinkSelectorSlug(text)` per the grammar above.
   - `parsePomodoroEntryParts(lineText) → { name, label, range }`: `name` = the em-dash
     tail (`— NAME`) after the line's placeholder/time-range parenthetical, trimmed, or
     null; `label` = legacy leftover text with the list/checkbox prefix, the
     parenthetical, and any tail removed (`- [ ] Current (10:00-10:25)` → `Current`;
     `- [ ] Later ()` → `Later`), or null; `range` =
     `{ startMinutes, endMinutes, text: "09:20–09:50" }` via `parsePomodoroTimeRange`,
     or null.
   - `collectPomodoroLinkEntries(content)` → `{ hasSection, section, entries }` with
     every recognized unfenced top-level entry in document order: `entryLine`,
     `entryText` (CR stripped), `position` (1-based among all recognized entries),
     `status`, `open`, `timed`, `placeholder`, `name`, `label`, `slug`, `range`, `links`
     (non-struck wiki block references in the entry block: `{ blockId, targetText }`; a
     reference wrapped in `~~…~~` is struck), `linkCount`, `running`, `nextUp`, `title`.
     Title = name, else label, else range text when timed, else `Pomodoro #N`.
   - `pomodoroRunProgress(entry, now) → { fraction, minutesLeft }` from local minutes
     (`fraction` clamped to 0–1; `minutesLeft` negative when over; an end before start
     wraps past midnight).
   - `buildPomodoroLinkPickerModel(content, { now })` →
     `{ ok: false, error: "no-section" }` or
     `{ ok: true, entries (open only), allEntries, defaultIndex (−1 when none), defaultReason ("running" | "next" | "ambiguous" | null), multipleRunning, creation: { allowed, blockedReason, anchor: { kind: "running" | "completed" | "first-open" | "section-top", title }, becomesNextUp } }`.
   - `buildPomodoroLinkPickerRows(model, rawQuery) → { rows, selectedIndex }` with row
     kinds `existing` (`entry`, `isDefault`, `match`), `new` (`name`, `againOf`,
     `anchorTitle`, `becomesNextUp`), `invalid` (`message`; only when nothing matches),
     and `blocked` (`message`). Every row has a stable `key` and `selectable`. Empty
     query → open entries in order with `selectedIndex = defaultIndex`; otherwise ranked
     rows with `selectedIndex` = first selectable row (−1 when none).
   - `resolvePomodoroLinkCreateIntent(model, rawQuery)` → `{ kind: "new", name }`,
     `{ kind: "existing", entry }` (exact open name), or `{ kind: "none", reason }` —
     the `⇧↵` behavior.
   - `defaultPomodoroLinkChoice(model)` → the default entry's choice or null.
   - Choice shape (the picker's result everywhere):
     `{ kind: "existing", entryLine, entryText, title }` or `{ kind: "new", name }`.
   - `planExplicitPomodoroLinkInsertion(content, options)` with the same options as
     `planPomodoroLinkInsertion` plus `target` (a choice). Returns today's result shape
     plus
     `destination: { kind: "existing" | "created", entryLine, title, name, matchedExisting }`,
     or `{ error }` among `no-section`, `invalid-options`, `invalid-name`,
     `target-missing`, `target-closed`, `multiple-open-timed` (new target only),
     `overlap`, `verify-failed`. A created entry is re-parsed after planning and must be
     recognized at the expected line as an open placeholder with exactly the canonical
     name that owns the inserted link line (mirroring capture's
     `verify_created_named_pomodoro`), else `verify-failed`. Preserve CRLF
     (`lineEndingForInsertion`) and a missing final newline (`insertionEditAtLine`).
2. In `070-daily-and-activation-plans.js`, make `planPomodoroLinkInsertion` delegate to
   `planExplicitPomodoroLinkInsertion` when `options.target` is set; the target-less
   path stays byte-for-byte today's behavior.
3. Export every new function in `src/170-exports.js` `helpers`.
4. New `scripts/test-block-id-prompt-pomodoro-targets.cjs` (add to `package.json`): name
   canonicalization/validation vectors matching capture (`after tui fix` →
   `AFTER TUI FIX`; `bugs+2` valid; `+bugs`, `123`, `deep—work`, `café` invalid;
   whitespace collapse); slugs; entry parsing of canonical
   (`- [ ] (**0920-0950** [t:: 25m]) — CAPTURE`) and legacy forms; fenced, nested,
   completed, and cancelled lookalikes; running / next-up / default / ambiguous
   detection; link counts with struck links excluded; progress and overtime; row ranking
   (including `MEMORY WORK` before `MEMORY` with query `memory` → `MEMORY` first),
   create-row presence rules (exact match suppresses; prefix match keeps; completed-only
   → `againOf`; invalid; blocked); create intent; explicit planning for an existing
   target (shifted line, changed text, closed, ambiguous, idempotent, other-open
   cleanup) and a new target (each of the four anchors, indentation reuse, race →
   `matchedExisting`, multiple-open-timed refusal, CRLF, no final newline); the
   target-less path unchanged.
5. `npm run build`, `npm test`, `npm run validate`, `bob plugins sync`.

### Phase: Pomodoro link picker modal and styles

Pure UI in bob-plugins `plugins/block-id-prompt`; not yet wired to the command.

1. Add `src/085-pomodoro-link-picker-modal.js` (after
   `080-picker-ui-and-task-link-modal.js` in `fragments.json`) with
   `class PomodoroLinkPickerModal extends Modal`:
   - `constructor(app, request)` where
     `request = { model, task: { displayText, status }, needsBlockId, now, budget }` and
     `budget` is null or `{ current: meter | null, forNewName: (name) => meter | null }`
     (`meter = { themes: { count, cap, over }, links: { count, cap, over }, over }`).
     Budget callbacks are best effort: a throw or null hides the meter or badge.
   - `waitForChoice() → Promise<choice | null>` that settles exactly once (choice on
     commit, null on any close without a choice) and then closes. The modal touches no
     plugin state; the caller owns `promptOpen`.
   - Render the anatomy, rows, states, footer, and keys of the UX specification using
     the core phase's row builder, create-intent resolver, and `pomodoroRunProgress`.
     Reuse `applyIcon`, `appendHighlighted` (match highlighting), `isCtrlKey`, and
     `laneStatusName`. Handle `Ctrl+Shift+Enter` before the plain-Shift branch.
   - Accessibility: listbox/option roles, `aria-selected`, per-row `id` with the input's
     `aria-controls` and `aria-activedescendant`, row `aria-label`
     (`CAPTURE, running, 09:20–09:50, 3 tasks`), and `mousedown` preventDefault so
     clicks never steal input focus.
   - Export `PomodoroLinkPickerModal` in `helpers` (as nav exports its picker classes).
2. Styles in `styles.css` under a new `/* Pomodoro link picker. bid-ppk-* */` section,
   Obsidian variables and `color-mix` only (works in light and dark):
   `.modal.bid-ppk-modal` `width: min(92vw, 620px)`, `max-height: min(80vh, 640px)`,
   `padding: 0`, `border-radius: var(--radius-l)`; header, search, row selection, kbd,
   and empty-state treatments match the existing `bid-tlp-*` language (36px tinted icon
   tile, focus ring, 3px accent rail + 14% tint for selection); the running glyph is a
   22px ring drawn with
   `conic-gradient(var(--interactive-accent) calc(var(--bid-ppk-progress) * 1turn), color-mix(in srgb, var(--interactive-accent) 18%, transparent) 0)`
   around a center dot, with `--bid-ppk-progress` set inline per row; pills for
   `Running`, `Next up`, `New`, and the theme badge (`--color-red` tint when over); a
   dashed divider above the create row; the warning banner in an orange tint.
   Transitions and the 2.4s breathing dot only under
   `@media (prefers-reduced-motion: no-preference)`; a `max-width: 600px` breakpoint
   wraps badges below the text.
3. New `scripts/test-block-id-prompt-pomodoro-picker-view.cjs` (add to `package.json`)
   that loads the plugin with `modal-harness.cjs`'s `ModalStub`/`ElementStub` (follow
   `navigation-hotkeys-harness.cjs`'s loader pattern; leave
   `block-id-prompt-harness.cjs` on its stub Modal): header texts and lane chip per
   status; `+ block ID` chip; rows in document order with the default selected;
   Running/Next up pills; the progress custom property and `15m left` / `5m over` meta;
   filtering, ranking, and highlight; create row meta (`after …`, `becomes next up`,
   again) and theme badge (normal and over-cap red class); invalid/blocked rows skipped
   by navigation and inert on `↵`; banner; empty state; footer hints (`⇧↵ new NAME` only
   with an intent) and meter; `↵`, `⇧↵`, `Ctrl+Shift+Enter`, click, `Esc`, `Ctrl+[`
   results; settle-once.
4. `npm run build`, `npm test`, `npm run validate`, `bob plugins sync`.

### Phase: Wire the picker into Ctrl+Shift+Enter, notices, docs, release

1. **Make room in the full fragment.** `src/120-plugin-pomodoro-links.js` is near the
   1000-line cap. Move `gatePomodoroToggleInboxRoute`, `commitPomodoroToggleInboxRoute`,
   `finishPomodoroLinkWithInboxRoute`, and `finishPomodoroUnlinkWithInboxRoute` verbatim
   into a new `src/124-plugin-pomodoro-inbox-route.js` mixin
   (`BlockIdPromptPomodoroInboxRouteMixin`), registered in `fragments.json` and in
   `installBlockIdPromptMixins` (`160-install-methods.js`). No behavior change.
2. **New mixin** `src/122-plugin-pomodoro-link-picker.js`
   (`BlockIdPromptPomodoroLinkPickerMixin`, registered likewise):
   - `promptPomodoroLinkTarget(request)`: opens `PomodoroLinkPickerModal` and resolves
     its choice; never throws (log and resolve null). This is the test seam.
   - `choosePomodoroLinkTarget(source)`: the preflight (step 2 of the gesture flow),
     `buildPomodoroLinkPickerModel` with `this.now()`, the budget callbacks,
     `promptOpen` held true while the picker is open and released when it settles, then
     resolves the choice or null.
   - Budget: refactor the ledger-tools lookup inside `planBudgetNoticeSuffix`
     (`130-plugin-task-link-open-and-notices.js`) into a shared
     `readPlanBudgetMeter(content)` returning the meter shape or null, keep the suffix
     output identical, and build `forNewName(name)` by planning the new target on the
     picker snapshot (the task's block ID, or a placeholder ID when it has none, so the
     projection counts one new link) and metering the planned content.
3. **`openPomodoroTaskLink`** (120): on the link direction (an unlinked task with an ID,
   or any task without one), `await this.choosePomodoroLinkTarget` inside the existing
   `try`. A null result returns, so the existing `finally` settles the review origin
   with null; a choice is set as `source.pomodoroTarget` (and copied onto the block-ID
   `promptSource`) before the existing completion or prompt handoff.
4. **`applyPomodoroTaskLink`** (120): pass `target: source.pomodoroTarget || null` to
   `planPomodoroLinkInsertion`. Extend `pomodoroPlanErrorNotice` with
   `Task link blocked: Pomodoro <title> changed in <daily path>`,
   `Task link blocked: Pomodoro <title> is already closed`,
   `Task link blocked: <POMODORO_NAME_USAGE>`, and
   `Task link blocked: new Pomodoro <NAME> could not be verified`.
5. **Notices** (130): `formatPomodoroLinkOutcome` prefixes the destination when
   `pomodoroPlan.destination` exists (`Linked to <title>` / `Linked to new <NAME>`);
   target-less output stays `Linked · …`.
6. **Harness and existing tests**: in `scripts/block-id-prompt-harness.cjs`,
   `createTaskModeHarness` and `createTaskLinkHarness` install a default
   `promptPomodoroLinkTarget` stub that records each request and resolves
   `helpers.defaultPomodoroLinkChoice(request.model)` — the old implicit target — so
   existing semantics still hold. Update existing assertions that now include the
   destination (`Linked · stays Next` → `Linked to Current · stays Next` for the
   `- [ ] Current (10:00-10:25)` fixtures, including review-walk `continue` outcomes).
   Any other test that drives `openPomodoroTaskLink` into the link direction must stub
   the seam.
7. New `scripts/test-block-id-prompt-pomodoro-picker-runtime.cjs` (add to
   `package.json`): picker opens only on the link direction (not unlink, task-link mode,
   or Depends-On refusals); preflight refusals open nothing; picking a non-default entry
   links there; cancel writes nothing with no Notice and settles the walk null; a no-ID
   task gets picker → block-ID prompt with the target carried → link; inbox order is
   picker → route → write; a review-walk landing continues with `link-today` and the new
   text; an all-closed ledger creates `- [ ] () — DEEP WORK` after the last completed
   entry; several running entries allow an explicit pick; a target that changed or
   closed before commit refuses with no writes; the task note being the daily note uses
   one editor transaction; `promptOpen` blocks re-entry while the picker is open.
8. **Release**: bump `plugins/block-id-prompt/manifest.json` to `1.24.0` and append to
   its description that Ctrl+Shift+Enter asks which of today's Pomodoros to link into (↵
   = current/next, type to filter or name a new one). Update the Block ID Prompt row in
   the bob-plugins `README.md` (version and a concise picker sentence; keep the unlink
   and task-link-mode text).
9. **bob-cli docs** (primary repo): `docs/getting-started.md` walk sentence
   (Ctrl+Shift+Enter picks a Pomodoro, `↵` takes the current/next one);
   `docs/projects.md` Inbox routing gesture order (Pomodoro picker first, then block-ID
   prompt, then route picker; picker `Esc` cancels with nothing written); `docs/plan.md`
   "Obsidian Notices" row example (`Linked to CAPTURE · Next · plan 1/3 · 2/10`) plus a
   row for the picker's footer meter and `+1 theme` create badge; `docs/freshness.md`
   walk-answer mentions where they describe Ctrl+Shift+Enter linking.
10. `npm run build`, `npm test`, `npm run validate`, `bob plugins sync`, and `git diff`
    the bob-cli docs for accuracy against the shipped behavior.

## Out of scope (possible follow-ups)

- Relocating an already-linked task to another Pomodoro from the picker (capture's
  `@route+id#name` Ensure Next); the toggle still unlinks.
- A picker-free command or setting; `Ctrl+Shift+Enter` twice already gives the old
  behavior.
- Showing completed Pomodoros, renaming entries, or starting a session from the picker.
