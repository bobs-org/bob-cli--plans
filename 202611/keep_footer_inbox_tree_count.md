---
tier: tale
title: Count the whole inbox tree in the Keep footer's inbox segment
goal:
  The Obsidian footer's inbox tray counts open tasks in inbox.md and every area note
  below it.
size: medium
decisions:
  inbox_click:
    ask:
      When several inbox notes hold open tasks, should clicking the inbox count show a
      menu of those notes?
    choices:
      first:
        Opens the note with the most open tasks at its first one; click again for the
        next
      menu: A menu lists each note and its count; a single note still opens directly
    default: first
    why:
      Triage drains one note at a time; the tooltip already names every note and count
    answer: first
proposed_by: bbugyi200.apollo.bs.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 08:24:59
status: wip
---

# Plan: Count the whole inbox tree in the Keep footer's inbox segment

## Context

The `bob-keep-inbox` status-bar item (bob-plugins, plugin `bob-ledger-tools`) has two
segments: the Google Keep bulb pill and an inbox-tray count. Plan
`plan:202611/keep_footer_icons_hide_zero.md` (commit `ba0a6ee` in bob-plugins) made each
segment show only while its count is above zero. Today the inbox segment counts open
top-level tasks in **one note**: the Keep pull target `report.inbox.path` from
`bob gkeep status`, which is `gkeep_inbox.md`.

Bryan wants the inbox segment to count open tasks in **every area note that descends
from the `inbox` area note**: an area note whose `parent` is `[[inbox]]`, or whose
parent chain reaches `inbox` at any depth.

### Today's vault tree (for orientation, not hard-coded)

| Note                           | `type`    | `parent`             | Counted after this plan?                     |
| ------------------------------ | --------- | -------------------- | -------------------------------------------- |
| `inbox.md`                     | area      | `[[gtd]]`            | Yes, as the tree root (holds no tasks today) |
| `mac_inbox.md`                 | area      | `"[[inbox]]"`        | Yes                                          |
| `gkeep_inbox.md`               | area      | `"[[inbox]]"`        | Yes                                          |
| `inbox_overflow.md`            | (none)    | `[[inbox]]` unquoted | No: not an area note                         |
| `gkeep_gdocs_inbox_dump.md`    | project   | `"[[mac_inbox]]"`    | No: a project note, not an area note         |
| `done/*_done.md`, `ref/chat/…` | done, ref | an inbox note        | No: not area notes                           |

Include `inbox.md` itself. A task placed there awaits triage just like one in
`mac_inbox.md`, the glossary's Inbox File definition already treats `inbox.md` as an
inbox, and it holds no checkbox tasks today, so the count does not change.

### Code as it stands (bob-plugins `plugins/bob-ledger-tools/src/`)

- `292-keep-inbox-model.js` (895 lines, pure):
  - `keepInboxOpenCount(tasks, path)` counts open top-level Tasks-plugin tasks in one
    note. Open means any symbol except `x`, `X`, or `-`.
  - `keepInboxInboxPart(opts, targetPath, nowMs, inboxDoc)` builds the segment. It hides
    when `inboxDoc.exists === false`. When Tasks is ready it uses `opts.inboxCount`;
    otherwise it falls back to bob's `inbox.open`. Its tooltip reads
    `<path> · N open tasks to triage\nClick to open the inbox`, and its aria reads
    `N open tasks in <path>`.
- `294-keep-inbox-indicator.js` (699 lines):
  - `keepInboxRecountNow` calls `planBlockTasks(this.app)`, then
    `keepInboxOpenCount(tasks, this.keepInboxTargetPath())`.
  - `openKeepInboxNote()` opens the Keep target and puts the cursor on its first open
    top-level task (`keepInboxFirstOpenLineFromTasks`, then the
    `keepInboxFirstOpenLineFromText` fallback). The inbox button's left click and the
    context menu's **Open gkeep_inbox.md** both call it.
- `142-pomodoro-home-chips.js` already has the parent-chain primitives this plan reuses.
  All fragments share one script scope.
  - `pomodoroHomeParentTarget(frontmatter.parent)` handles both `"[[x]]"` and the
    unquoted `[[x]]` array form.
  - `pomodoroHomeResolveFile(target, sourcePath, dailyPath, metadataCache, vault)`
  - `pomodoroHomeReadFrontmatter(file, metadataCache, vault)`
  - `pomodoroHomeType(frontmatter)` returns `"area"`, `"project"`, or `"other"`.
  - `POMODORO_HOME_MAX_DEPTH = 16`
- bob-navigation-hotkeys' `classifyInboxNote` (route picker, direct children only) is a
  different concept, the routing "inbox note". Do not change it or reuse its name. Call
  the new set the **inbox tree**.

## Target behavior

- **Inbox tree member:** a note is a member when either:
  - its path is `inbox.md` (the tree root, at the vault root, matching
    bob-navigation-hotkeys' `INBOX_NOTE_PATH`), or
  - its own frontmatter type is area (`pomodoroHomeType(fm) === "area"`) and walking
    `parent` links upward reaches `inbox.md`.

  Intermediate notes of any type are walked through. An area note under a project under
  `inbox` counts, as the request words it ("one of its parent's ancestors").
  Unresolvable parents, missing frontmatter, cycles, and chains deeper than
  `POMODORO_HOME_MAX_DEPTH` mean the note is not a member. Membership reads live
  frontmatter on every recount and is memoized only within one recount.

- **Count:** keep the per-task predicate unchanged (top-level, open, from the Tasks
  plugin cache). Sum it over all member notes. The segment still shows only when the
  total is above 0.
- **Tasks not ready:** the segment hides. Drop the bob `inbox.open` fallback. That
  number counts only `gkeep_inbox.md`, so showing it would flash a different quantity
  and then jump when Tasks loads.
- **Keep target existence:** `inbox.exists === false` (a missing `gkeep_inbox.md`) no
  longer hides the inbox segment.
- **Tooltip:** sort the per-note breakdown by open count descending, then path
  ascending. List at most 4 notes, then `+K more`. Labels are note stems:

  ```text
  Inbox · 6 open tasks to triage
  gkeep_inbox 4 · mac_inbox 2
  Click to open gkeep_inbox
  ```

  Use singular `1 open task to triage`. The aria reads `6 open tasks in the inbox`
  (`1 open task in the inbox`). Without a note list (legacy callers and tests), omit the
  breakdown line and end with `Click to open the inbox`.

- **Left click on the inbox segment:** follows the `inbox_click` decision.
- **These do not change:**
  - Item-level gates: session toggle, no read yet, unconfigured, and unsupported with a
    missing default target.
  - The Keep pill.
  - The context menu, including **Open gkeep_inbox.md** (the Keep pull target).
  - Vault-event handling. It still watches only the Keep target, because it exists to
    re-read bob after terminal pulls. Tasks `cache-update` already recounts the tree on
    any task edit.
  - The bob-cli `bob gkeep status` contract.

> [!decision] inbox_click = first Left click opens the first note in the tooltip order
> (most open tasks, then path) at its first open top-level task. The tooltip's last line
> names that note: `Click to open <stem>`.

> [!decision] inbox_click = menu With exactly one member note holding open tasks, the
> left click opens it directly, as under `first`. With two or more, it shows a
> `KeepInboxActionMenu` (the same captured `obsidian.Menu`) at the click. It has one
> item per note, titled `<stem> · N`, with icon `inbox`, in tooltip order. Each item
> opens that note at its first open task. The tooltip's last line reads
> `Click to choose an inbox note`. If `Menu` is unavailable, behave like `first`.

## Implementation

**Repo:** `bob-plugins`, plugin `plugins/bob-ledger-tools`. Open it with `/sase_repo`
(`sase repo open bob-plugins -r …`) and read its `AGENTS.md`. Edit `src/` fragments
only, run `npm run build`, never hand-edit `main.js`, and keep every fragment at or
below 1000 lines.

### 1. New fragment `src/292a-keep-inbox-tree.js`

Add it to `src/fragments.json` immediately after `292-keep-inbox-model.js`. It may call
the `142-pomodoro-home-chips.js` helpers, which load earlier. Every function never
throws.

- `const KEEP_INBOX_TREE_ROOT = "inbox.md";`
- `keepInboxTreeMember(path, metadataCache, vault, memo)` returns a boolean.
  - Normalize the path (`\` to `/`, strip a leading `./`) and use `memo`, a `Map`, when
    given.
  - The root path returns `true`.
  - Resolve the file (`vault.getAbstractFileByPath`) and read its frontmatter with
    `pomodoroHomeReadFrontmatter`. If the type is not `area`, return `false`.
  - Walk upward. Get `pomodoroHomeParentTarget(fm.parent)`, then resolve it with
    `pomodoroHomeResolveFile(target, current.path, current.path, metadataCache, vault)`.
  - A resolved parent at `inbox.md` returns `true`. A missing target, an unresolved
    file, a revisited path (case-insensitive `Set`), or depth ≥
    `POMODORO_HOME_MAX_DEPTH` returns `false`.
  - Memoize the answer for the starting path.
- `keepInboxTreeCounts(tasks, isMember)` is pure, given a `(path) => boolean`.
  - It returns `{ total, notes }`, where `notes` is `[{ path, label, open }]` for
    members with `open > 0`, sorted by `open` descending, then `path` ascending. `label`
    is `pomodoroHomeStem(path)`.
  - Reuse `keepInboxIsTopLevelTask`, `keepInboxTaskPath`, and `keepInboxTaskSymbol` so
    the per-task predicate stays identical to `keepInboxOpenCount`.
  - Call `isMember` once per distinct path.
  - A non-array `tasks` returns `{ total: 0, notes: [] }`.
- Export the new helpers and the constant in `src/350-exports.js` beside the other
  `keepInbox*` helpers.

### 2. Model (`src/292-keep-inbox-model.js`)

- Rework `keepInboxInboxPart` for the tree.
  - Inputs: `opts.tasksReady`, `opts.inboxCount` (total), and the new optional
    `opts.inboxNotes` (the `notes` array above).
  - Delete the `inboxDoc.exists` check and the bob `inbox.open` fallback. Not ready, or
    a total of 0, returns `keepInboxHiddenInboxPart()`.
  - Build the tooltip and aria from Target behavior, with the decision's last tooltip
    line.
  - Drop the now-unused `targetPath`, `nowMs`, and `inboxDoc` parameters. Simplify the
    callers (`keepInboxUnsupportedView`, `keepInboxInboxOnlyView`, `keepInboxFreshView`,
    `keepInboxStaleView`, `keepInboxSigninView`) and remove the inbox-doc lookups that
    only fed it.
  - Keep `keepInboxViewModel`'s use of `report.inbox` only where something else still
    needs it. Nothing in the inbox segment should.
- Update the `keepInboxViewModel` header comment. `inboxCount` is now the inbox-tree
  total, the new `inboxNotes` is the per-note breakdown, and bob's `inbox.open` is no
  longer read.
- Keep `keepInboxOpenCount` for parity with bob's single-note count (its test stays).
  Update its comment to say the footer now sums the same predicate over the inbox tree.

### 3. Indicator (`src/294-keep-inbox-indicator.js`)

- `keepInboxRecountNow`:
  - Build a fresh `Map` memo.
  - Compute
    `keepInboxTreeCounts(tasks, (path) => keepInboxTreeMember(path, this.app.metadataCache, this.app.vault, memo))`.
  - Store `this.keepInboxCount = total` and `this.keepInboxTreeNotes = notes`.
  - It no longer needs `keepInboxTargetPath()`.
- Initialize `this.keepInboxTreeNotes = []` in `setupKeepInboxIndicator`. Pass
  `inboxNotes: this.keepInboxTreeNotes` into `keepInboxViewModel` in
  `keepInboxUpdateView`.
- Split the open logic:
  - `openKeepInboxNote(path)` opens the given path, defaulting to
    `keepInboxTargetPath()` so the context menu's **Open gkeep_inbox.md** keeps working.
    The not-found Notice and first-open-line placement stay as they are.
  - Add `openKeepInboxTreeNote(event)` for the inbox button's left click. It implements
    `inbox_click`, taking its notes from `this.keepInboxTreeNotes`.
    - With an empty list (only reachable in a race), open `KEEP_INBOX_TREE_ROOT`.
    - Under `menu`, it builds the menu in this fragment or in
      `295-keep-inbox-actions.js`, wherever `KeepInboxActionMenu` and
      `keepInboxActionAddMenuItem` are reachable. Both are top-level in the shared
      scope.
  - Rewire `onInboxClick` to call it.
- The header comment changes from "the inbox count is recomputed from the Tasks cache"
  to "the inbox-tree count …".

### 4. Tests (`scripts/test-ledger-tools-keep-inbox.cjs`)

Add a small fake vault helper. Each fake note is `{ path, frontmatter }`, with
`metadataCache.getFileCache(file)` returning `{ frontmatter }`,
`getFirstLinkpathDest(name)` resolving by stem, and `vault.getAbstractFileByPath`.

- **Membership.** Members:
  - `inbox.md`
  - `mac_inbox` and `gkeep_inbox` (area, `parent: "[[inbox]]"`)
  - a grandchild area (`parent: "[[mac_inbox]]"`)
  - an area under a project under `inbox`

  Non-members:
  - a project under `mac_inbox` (models `gkeep_gdocs_inbox_dump`)
  - an untyped note with the unquoted array form `[["inbox"]]` (models `inbox_overflow`)
  - an unrelated area (`parent: "[[area]]"`)
  - an unresolvable parent
  - a two-note `parent` cycle, which must return without hanging

- **Counts.** `keepInboxTreeCounts` skips nested, done, cancelled, and non-member tasks,
  sorts by count then path, and calls `isMember` once per path (spy).
- **View model.** These replace `tasks-not-ready falls back to bob's inbox.open…`:
  - Tasks not ready with `inbox.open: 1` hides the inbox segment.
  - `inbox.exists: false` with a positive tree total still shows it.
  - Tooltip and aria use the tree copy with the breakdown, the `+K more` cap, and the
    decision's last line.
  - Without `inboxNotes`, the copy is generic.
  - Update the existing tooltip/aria test's expected strings.
- **Recount integration.** Build a plugin with a fake app: Tasks plugin
  `obsidian-tasks-plugin` exposing `getTasks()`, plus the fake vault. Assert that
  `keepInboxRecountNow` stores the tree total and notes.
- **Click.**
  - `first`: the inbox click opens the top note (stub `workspace.getLeaf().openFile` and
    assert the file path).
  - `menu`: with two notes the click shows a `TestMenu` with `<stem> · N` items, and
    with one note it opens directly.
  - The context menu's **Open gkeep_inbox.md** still opens `gkeep_inbox.md`.
- Keep the parity-fixture test for `keepInboxOpenCount` unchanged.

### 5. Docs

- bob-plugins `docs/bob-ledger-tools.md`, "Keep inbox indicator":
  - The inbox tray counts open top-level tasks in the inbox tree: `inbox.md` and every
    area note below it at any depth via `parent`, such as `mac_inbox.md` and
    `gkeep_inbox.md`. Project and untyped notes filed under it do not count.
  - Replace "falling back to bob's cached `inbox.open` before Tasks is ready" with "the
    inbox tray stays hidden until Tasks is ready".
  - Hovering lists each note's count. Rewrite "Clicking the inbox opens the target
    note…" per the decision. The context menu's **Open `gkeep_inbox.md`** is unchanged.
  - Run `npm run docs:check`.
- bob-cli `docs/guides/drain-google-keep.md`, "See the count in Obsidian": say the inbox
  tray shows the open-task count across the inbox area notes (`inbox.md` and every area
  note under it), not only `gkeep_inbox.md`.
- bob-cli `docs/gkeep.md`, Status: change "(the Obsidian footer reads it)" to "(the
  Obsidian footer reads its Keep count)". The status contract itself does not change.
  Run `just docs-check`.

### 6. Build, verify, deploy, commit

- In bob-plugins, run:
  - `npm run build`
  - `npm run build:check`
  - `node --test scripts/test-ledger-tools-keep-inbox.cjs`
  - `npm test`
- Before blaming this change for an `npm test` failure, check whether it also fails on
  unchanged `master`. Known unrelated failures: Chromium-less e2e (bead `bob-cli-f1`),
  `bob-cli-et`, and `bob-cli-eh`.
- Confirm that `292`, `292a`, `294`, and `295` are each ≤ 1000 lines.
- Run `bob plugins sync`.
- Commit in bob-plugins as
  `feat(ledger-tools): count the whole inbox tree in the Keep footer`.
- Commit in bob-cli as `docs(gkeep): footer inbox tray counts the inbox tree`. Do not
  touch `Cargo.toml` or `CHANGELOG.md`.

## Out of scope

- No bob-cli code or JSON contract change. `bob gkeep status`'s `inbox` document still
  describes the Keep pull target.
- Project notes under the inbox, such as `gkeep_gdocs_inbox_dump.md`, are not counted:
  the request names area notes.
- No change to which hosts show the item. It still hides when Keep is unconfigured.
- No change to bob-navigation-hotkeys' routing `classifyInboxNote` or to the glossary.
  "Inbox File" (direct children, used by routing) stays accurate. This plan's "inbox
  tree" is a separate, footer-only set, and no memory note describes the footer's count.

## Live verification (Bryan)

After `bob plugins sync` and an Obsidian reload of bob-ledger-tools:

- The inbox tray shows the sum of the open top-level tasks in `mac_inbox.md` and
  `gkeep_inbox.md`.
- Hovering lists both notes with their counts.
- Clicking follows the chosen `inbox_click` behavior.
- Moving a task into a new area note filed under `mac_inbox` raises the count after the
  next Tasks cache update.
