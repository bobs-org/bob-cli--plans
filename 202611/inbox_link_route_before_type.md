---
tier: tale
title: Route inbox tasks before picking their type on Ctrl+Shift+Enter
goal:
  On an inbox task, the Ctrl+Shift+Enter link asks for the destination note first, so
  Link to today suggests and retypes from the destination's type.
size: medium
decisions:
  route_chip:
    ask: Show the chosen destination as a `→ <dest>` chip in the Link to today header?
    default: true
    why:
      The picker now suggests from a note the task is not in yet; the chip explains it.
    answer: true
  glossary_inbox_file:
    ask:
      Update the Inbox File glossary strand to say a Ctrl+Shift+Enter link asks for the
      route first?
    memory:
      - glossary:inbox-file
    default: true
    corrects: the task line, and Alt+N ask where the task goes just before writing
    answer: true
proposed_by: bbugyi200.athena.16s
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 16:34:59
status: wip
---

# Ask where an inbox task goes before asking for its type (Ctrl+Shift+Enter)

## Context

The gesture is **`Ctrl+Shift+Enter`** (block-id-prompt `link-task-to-pomodoro`, "Link to
today"). Plain `Ctrl+Enter` is task-status-cycler's close. It never asks for a type or a
destination, so this plan does not touch it.

When the task sits in an inbox note (`inbox.md`, or an area whose `parent` is
`inbox.md`, such as `mac_inbox.md` or `gkeep_inbox.md`), the link direction runs today
in this order (bob-plugins `plugins/block-id-prompt/src/120-plugin-pomodoro-links.js`,
`openPomodoroTaskLink` and `applyPomodoroTaskLink`):

1. **Link to today picker** (`choosePomodoroLinkTarget`,
   `122-plugin-pomodoro-link-picker.js`). Its type suggestion and preselection come from
   `themesApi.suggestFor({ path: source.sourcePath, line, text })`, which uses the
   **inbox** note.
2. Block-ID prompt, when the task has no `^id`.
3. **Inbox route picker** (`gatePomodoroToggleInboxRoute` → nav
   `api.inboxRoute.prompt`).
4. Write the link and the `[type::]` retype, then `commitInboxRoute` moves the task.

Because the type is chosen before the destination, the suggestion reflects the inbox,
not the note the task is about to live in. The write-time retype
(`resolvePomodoroLinkThemeWrite` in `078-pomodoro-theme-write.js`) plans `planRetype`
against the inbox's inherited type too. It may write an explicit `[type:: X]` where the
destination would have supplied X anyway. After the move, nav's D8 relocation
(`planNoteMoveLedgerRelocation` in
`bob-navigation-hotkeys/src/655-plugin-inbox-route.js`) may then rename or move the
Pomodoro the user just picked.

**Goal.** For the link direction on an inbox task, the order becomes:

1. Route picker.
2. Link to today picker, which suggests the destination's type.
3. Block-ID prompt, when needed.
4. Write, then move.

`⇧↵` (stay) in the route picker keeps today's inbox-based suggestion. Unlinking (the
task already has a live link) keeps today's order: an optional Work summary prompt, then
the route picker right before the unlink write. Unlink never asks for a type.

### Why a one-line virtual note gives the right suggestion

The Ledger Tools cascade (`bob-ledger-tools/src/279-plugin-task-theme-api.js`
`taskTypeEffectiveTypeFor` → `279-plugin-task-theme-suggest.js`
`pomodoroThemeSuggestWithDiagnostics`) reads only these inputs:

- the request `path`, for the residence chain and the area/project `type` default via
  the metadata cache;
- the task's own line, for its `[type::]` field, the `#ref` / `^ref` reading rule, and a
  `[root::]` redirect.

The surrounding note text feeds only `pomodoroThemeTodayNames`, and only when the path
is today's daily note. Inbox route destinations are areas and open projects, never daily
notes.

So `suggestFor({ path: <destination>, line: 1, text: <task line> })` returns exactly
what the task inherits once it lives in the destination. It matches what nav's
`effectiveTypeFor` sees after the move. No nav or Ledger Tools API change is needed.

### Why D8 relocation stays a harmless safety net

With the destination-aware effective type, `planRetype` returns one of five kinds:

- `consistent`: no field is written, and the picked entry agrees with the destination
  type.
- `set`: an explicit field is written, so relocation skips the task.
- `clear`: the task now inherits the destination type, and the entry agrees.
- `container`: `typeAgrees` treats type-less entries as agreeing.
- `merge`: the entry name contains the type.

In every case `planRelocation` returns `unchanged`, so the move no longer second-guesses
the pick. Keep D8 as-is for fallback paths (a missing themes API, a changed suggestion).

## Changes — bob-plugins (open with `/sase_repo`; read its `AGENTS.md`)

block-id-prompt uses the fragment build. Edit only `src/` fragments, then run
`npm run build`. Never hand-edit `main.js`. Keep every hand-edited fragment at or under
1000 lines.

### 1. Route first in `openPomodoroTaskLink` (`120-plugin-pomodoro-links.js`)

Insert a route step in the link direction. It goes after the link-presence check, which
decides link vs unlink, and before `choosePomodoroLinkTarget`:

- Hold `this.promptOpen = true` around the await, and release it in a `finally`. Then
  call `this.gatePomodoroToggleInboxRoute(source, "link to today", [])`. Reserved IDs
  are empty because no new ID exists yet; see step 5.
- On `cancel`, return. The existing `finally` settles the walk origin with `null` once.
  Nothing is written, and no picker or prompt opens.
- After a `stay` or `move`, check that the task is unchanged with
  `revalidatePomodoroToggleSourceAfterRoute(source, {})`. If the line drifted while the
  route picker was open, show `Task link blocked: selected task changed in <sourcePath>`
  and return.
- Store the outcome as `source.inboxRouteGate`, in the shape `{ kind: "stay" }` or
  `{ kind: "move", path, name }`. Non-inbox notes resolve to `stay` without prompting,
  because the gate already checks `isInboxNote`. Store that too, so later steps never
  prompt again.
- The `promptSource` spread (`{ ...source, ... }`) carries `inboxRouteGate` into the
  block-ID prompt.
- Update the "constant first step" comment to the new order:
  `(inbox route picker) → Pomodoro picker → (block-ID prompt) → write`.

### 2. A shared suggestion context

Add one helper, for example `pomodoroLinkThemeTaskContext(source, liveText)`, in
`078-pomodoro-theme-write.js`. That fragment loads before 122, and the fragments share
one script scope.

- When `source.inboxRouteGate` is `{ kind: "move", path }`, return
  `{ path: gate.path, line: 1, text: <the task's live line> }`. Read the line with
  `pomodoroThemeWriteLineText(liveText, source.line)`.
- Otherwise, return
  `{ path: source.sourcePath, line: source.line + 1, text: liveText }`.

Add a short comment that cites the cascade's path-plus-line inputs from the Context
section above.

Use this helper at both `suggestFor` call sites of this gesture:

- `choosePomodoroLinkTarget` (`122-plugin-pomodoro-link-picker.js`): build the
  `safeBlockPomodoroThemePickerSnapshot` task context from the helper. The footer's
  `lineText` stays the task's own line.
- `resolvePomodoroLinkThemeWrite` (`078`): compute `current` from the helper. The
  shown-vs-current identity check then compares like with like, and a routed link does
  not report `· type unchanged (suggestion changed)`. `planRetype` receives the
  destination's effective and inherited types.

Leave the `((` / `^^` path (`129-plugin-task-wikilink-pomodoro.js`) and the
`api.pomodoroLinkPicker.pick()` path (`126`) unchanged. They never route.

### 3. `applyPomodoroTaskLink` uses the chosen route

When `source.inboxRouteGate` is present, use it as `routeGate` and do not prompt. Keep
the existing `revalidatePomodoroToggleSourceAfterRoute` call, the no-op guard, and
`finishPomodoroLinkWithInboxRoute` exactly as they are.

When `source.inboxRouteGate` is absent (a direct caller), keep today's gate call and the
`inboxRouteCancelled` flag as a fallback. The unlink path (`applyPomodoroTaskUnlink`,
`submitPomodoroWorkSummary`) does not change.

### 4. Destination chip (decision `route_chip`)

> [!decision] route_chip Pass `route: { name }` in the request that
> `choosePomodoroLinkTarget` builds when the gate is a move.
> `PomodoroLinkPickerModal.renderHeader` (`085-pomodoro-link-picker-modal.js`) adds a
> side chip `→ <name>` next to the lane and `+ block ID` chips, with the title
> `Moves to <name> after linking`. Add a muted `bid-ppk-route` style to
> `plugins/block-id-prompt/styles.css`, modeled on `bid-ppk-blockid`. A request without
> `route` renders exactly as today.

> [!decision] route_chip = no Skip this step. The header is unchanged.

### 5. Block-ID collision against the destination

The route preflight can no longer reserve the new ID. In
`submitPomodoroTaskLinkBlockId`, after the existing same-note duplicate check, add a
destination check that runs only when `source.inboxRouteGate` is a move:

1. Resolve the destination file with `this.resolveTaskFile(gate.path)`, or the
   equivalent vault lookup.
2. Read it with `this.readFileSnapshot(file, source)`.
3. If `blockTokenMatches(content, newId)` finds a match, show
   `Block ID '<id>' already exists in <gate.path>` and return `false`. The prompt stays
   open, as it does for a source duplicate.
4. If the file is unreadable or missing, show
   `Task link blocked: <gate.path> could not be read` and return `false`.

The move commit's own planner still refuses collisions as a backstop.

### 6. Version, docs, build

- Bump `plugins/block-id-prompt/manifest.json` from `1.34.0` to `1.35.0`. Update its
  description: on inbox notes, a `Ctrl+Shift+Enter` link asks where the task goes first,
  and Link to today then suggests the destination's type. An unlink asks right before
  writing.
- `docs/block-id-prompt.md`:
  - Rewrite the inbox paragraph under "Freshness, plan budget, and projects" for the new
    link order: the destination-based suggestion, the chip, `⇧↵` keeping the inbox
    suggestion, and the destination block-ID check. The unlink order stays as it is.
  - Replace "Inbox notes with no shown suggestion still retype from the live
    `suggestFor`" with the destination-aware statement.
- Run `npm run build`, then `npm run docs:gen` to regenerate the description inventory
  row.

### 7. Tests (`scripts/`; harness `scripts/block-id-prompt-harness.cjs`)

`test-block-id-prompt-pomodoro-picker-runtime.cjs`:

- Rename "inbox order is picker, then route, then write" to "route, then picker, then
  write", and assert `["route-prompt", "picker"]`.
- Add: a route cancel never opens the picker, and nothing is written.
- Add: after a `move`, the snapshot calls `suggestFor` with
  `{ path: <dest>, line: 1, text: <task line> }`.
- Add: after a `stay`, the snapshot uses the inbox path.
- Add (when `route_chip` is accepted): `request.route.name` is set after a move and
  absent otherwise.

`test-block-id-prompt-pomodoro-link-runtime.cjs` (inbox-route block):

- Rewrite "a new block ID goes through the ID prompt first, then the route, with the
  reserved ID passed". The new order is route (`reservedBlockIds: []`), then the picker,
  then the ID prompt. The submit does not prompt for the route again, and the commit
  `expected` carries the new ID.
- Add: a submitted ID that already exists in the destination keeps the prompt open. It
  shows the notice, writes nothing, and makes no commit call.
- Rework "block-ID route cancel returns sentinel" into: route cancel up front writes
  nothing, opens no ID prompt, and settles `null` once.
- Keep "source drift during route prompt refuses without writes" passing under the new
  order. It now fails before the picker opens.
- Update the stay, move, and no-op tests so they assert exactly one route prompt per
  gesture.

`test-block-id-prompt-pomodoro-theme-write.cjs`:

- Rework "an inbox task with no shown suggestion retypes from the live suggestion" to go
  through a routed source. Use `source.inboxRouteGate` = move. Assert that `suggestFor`
  received the destination context.
- Add: the inbox and the destination inherit different types, and the user picks the
  destination's suggested Pomodoro. The result is `consistent`: no `[type::]` field is
  written, and there is no "suggestion changed" suffix.
- Add: picking a different type writes `[type:: X]`.

`test-block-id-prompt-pomodoro-unlink-runtime.cjs`: no change expected. It must still
pass.

Run `npm test`, `npm run validate`, and `npm run docs:check`. Then run
`bob plugins sync` to deploy to the vault, as bob-plugins' `AGENTS.md` requires.

## Changes — bob-cli (this repo)

- `docs/projects.md`, "Inbox routing":
  - The opening sentence says all three gestures ask "as the last step before they
    write". Make that true only for `Ctrl+Shift+P`, `Alt+N`, and the `Ctrl+Shift+Enter`
    unlink.
  - In **Gesture order**, describe the `Ctrl+Shift+Enter` link as: route picker, then
    the Pomodoro picker suggesting the destination's type (with the `→ <dest>` chip when
    `route_chip` is accepted), then the block-ID prompt, which refuses an ID that
    already exists in the destination, then the write and the move.
  - `Esc` in either picker cancels the whole toggle with nothing written or moved.
  - `⇧↵` keeps the task in the inbox, and the inbox-based suggestion still applies.
  - The unlink keeps its Work summary prompt first, then the route picker right before
    the write.
- `docs/getting-started.md` (around line 231): split the sentence so Task Card answers
  still ask as the last step before writing, while `Ctrl+Shift+Enter` asks first, so its
  Pomodoro picker suggests the destination's type.
- Leave the dated history entry in `docs/freshness.md` untouched.

> [!decision] glossary_inbox_file In `sase/memory/glossary/inbox-file.md`, reword only
> the routing sentence. Ctrl+Shift+P answers and Alt+N still ask just before writing. A
> Ctrl+Shift+Enter link on the task line asks first, so the Pomodoro picker suggests the
> destination's type. Its unlink asks just before writing. Use `/sase_memory_write`
> before editing. Then run `sase memory init`, and regenerate
> `docs/reference/glossary.md` with `uv run --project tools/docs bob-docs gen` (or
> `just docs-gen`).

Run `just docs-check`. Do not edit `CHANGELOG.md` or the `Cargo.toml` version. No Rust
code changes.

## Out of scope

- Task Card answers and `Alt+N` keep routing at the last step. Their type, if any, is
  chosen inside the card.
- The project-blocked or prerequisite-blocked refusal (`consumeBlockedTaskPomodoroLink`)
  still runs at write time, after the pickers, as it does today.
- Bob Mac Capture's route card does not change. It has no type prompt.

## Verification

- Every test in step 7 passes, along with `npm test` and `npm run validate` in
  bob-plugins and `just docs-check` in bob-cli. `npm run build:check` shows the
  generated `main.js` is current.
- Manual check after `bob plugins sync`: in `mac_inbox.md`, run `Ctrl+Shift+Enter` on an
  unlinked task with no `^id`.
  - The route picker opens first.
  - Pick an area whose `type` default differs from the inbox's. Link to today preselects
    that area's type, labeled `Suggested · from ● <area>`, and shows the `→ <area>`
    chip.
  - `↵` opens the block-ID prompt. An ID that already exists in the area is refused.
  - A fresh ID links the task under the suggested Pomodoro with no `[type::]` field, and
    the notice ends `· moved to <area>`.
  - `Esc` in the route picker writes nothing. `⇧↵` gives today's inbox-based suggestion.
