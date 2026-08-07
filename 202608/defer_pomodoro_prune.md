---
tier: tale
title: Deferring a task with Ctrl+Shift+P prunes its links from today's open Pomodoros
goal:
  When the `Ctrl+Shift+P` bullet-property picker gives a task a strictly future
  `scheduled` date, every live block link to that task under an open Pomodoro entry in
  today's daily note is removed, so the deferred task stops seeding `bob
  task-status-hooks`' promotion graph and the ledger visibly reflects that the task is
  not today's work.
proposed_by: bbugyi200.athena.uw
create_time: 2026-08-07 13:51:40
status: wip
---

# Plan: prune deferred tasks out of today's open Pomodoros

## Repos touched

- **`bob-plugins`** (linked repo — open with `/sase_repo` first, do **not** guess the
  path): all code, tests, README, and manifest changes. Everything lives under
  `plugins/bob-navigation-hotkeys/` and `scripts/test-navigation-hotkeys.cjs`.
- **`bob-cli`** (your own workspace checkout): user-facing docs only —
  `docs/projects.md`, plus one cross-reference line in `docs/task-status-hooks.md`. **No
  Rust changes.**

Never edit plugin files under `~/bob/`; `bob plugins sync` overwrites them.

## Background

### What the picker does today

`plugins/bob-navigation-hotkeys/main.js` implements `Ctrl+Shift+P` as
`BulletPropertyPickerModal`. Choosing a value fans out through `applySelectedValue` into
exactly three writers:

| Target                                     | Writer                                                                                               | Application style                                   |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| ordinary task, inline `[scheduled:: ...]`  | `setBulletPropertyValue` / `setBulletPriorityValue` → `setInlineBulletPropertyValues`                | `replaceEditorLine` on one line                     |
| `^prj` lifecycle task, project frontmatter | `setProjectNoteScheduledValue` → `planProjectScheduledUpdate`                                        | `applyEditorContentTransaction` over `plan.content` |
| counted session (`N<Ctrl+Shift+P>`)        | `setCountedBulletPropertyValue` / `setCountedBulletPriorityValue` → `planCountedBulletPropertyBatch` | `applyEditorContentTransaction` over `plan.content` |

All three already recognize a strictly future date and mark the task Blocked:

- `setInlineBulletPropertyValues` (~line 15897) — `isFutureInlineScheduledValue(...)` →
  `blockObsidianTaskCheckboxStatus`.
- `planProjectTaskSchedules` (~line 10242) — same predicate per propagated task, counted
  in `blockedTaskCount`.
- `planCountedBulletPropertyBatch` — same, via its `shouldBlockInlineTasks` path.

Those three branches are the exact trigger points for this feature. Everything else
about the picker — guards, schedule log, notices, `Ctrl+D` — stays as it is.

### Why leaving the Pomodoro link behind is wrong

`bob task-status-hooks` (`src/native/task_status_hooks.rs`) reads block links from
indented bullets beneath **open** top-level Pomodoro entries in the current daily note
and feeds them to `desired_statuses` as roots. Read that function (~line 2486): a root's
desired status is `strongest_current_status(...).unwrap_or(Next).max(Next)`. A `[?]`
Blocked task has no ranked status, so it seeds **`Next`**, and every transcluded
`![[#^dep]]` dependency beneath it inherits that rank. So a deferred task left under an
open Pomodoro:

1. keeps promoting its whole dependency chain to `[*]`/`[/]`;
2. keeps counting as "recent activity", which preserves stale `[/]` In Progress in
   area/project notes; and
3. for a `^prj` lifecycle task specifically, gets marked `[*]` **itself** — `^prj` keeps
   its schedule in frontmatter, never inline, so the derived-Blocked rule never fires on
   it.

Plus the plain-English reason: an open Pomodoro is a statement about today's work, and a
task scheduled for next month does not belong in it.

### The nearest precedent — read this before writing anything

`plugins/block-id-prompt/main.js` already solves the structurally identical problem (it
prunes duplicate links from future open Pomodoros when a link is created). Reuse its
**shape**, not its code — the plugins deliberately do not import from each other (see
the "Duplicated here on purpose" comment at `bob-navigation-hotkeys/main.js:34`):

| block-id-prompt function          | Line | What to mirror                                                   |
| --------------------------------- | ---: | ---------------------------------------------------------------- |
| `pomodoroEntryEndLine`            | 1901 | entry block ends at the next column-0 non-blank line             |
| `collectFutureOpenPomodoroRanges` | 1916 | walk entries, skip closed ones, emit child ranges                |
| `referenceRemovalRange`           | 1950 | widen a link span over a leading `!` and a wrapping `~~ ~~`      |
| `isDedicatedLinkBullet`           | 2001 | the link is the bullet's entire body                             |
| `listItemSubtreeEdit`             | 2014 | delete the bullet plus every deeper-indented line under it       |
| `planFuturePomodoroLinkCleanup`   | 2039 | plan → subtree edits first, then token edits not already covered |

The other reusable precedent, already inside `bob-navigation-hotkeys/main.js`:

| Existing helper                             |  Line | Reuse for                                                      |
| ------------------------------------------- | ----: | -------------------------------------------------------------- |
| `scheduledRecoveryDailyPaths(files, today)` |  4888 | resolving today's `YYYY/YYYYMMDD.md` (`.current`)              |
| `createScheduledRecoveryNoteIndex(files)`   |  4745 | path + unique-basename index                                   |
| `resolveScheduledRecoveryNote(index, ...)`  |  4766 | `[[Tasks#^x]]` / `[[Areas/Tasks#^x]]` / `[[#^x]]` → vault path |
| `getOpenMarkdownBufferContents(app)`        |  5199 | read unsaved buffers, with its own ambiguity flag              |
| `getOpenMarkdownEditorForPath(path)`        | 14654 | find an open editor for the daily note                         |
| `writeTaskMoveChange` (preimage guard)      | 18571 | the guarded cross-file write pattern                           |
| `getTrailingBlockId(line)`                  |   941 | the task's `^block-id`                                         |
| `splitMarkdownContent(content)`             |  8421 | CRLF-preserving line split                                     |
| `getMarkdownLineContexts(content)`          |  8508 | frontmatter / fenced-code masking                              |

## Design

### Definitions this feature commits to

**Open Pomodoro entry.** A column-0 list item `- [c] <text>` inside the daily note's
`## Pomodoros` section whose checkbox `c` is **not** `x`, `X`, or `-`. This is
`pomodoro::open_ledger_task` in `bob-cli` (`src/native/pomodoro.rs:334`) — the exact
rule `bob task-status-hooks` uses, which is the contract that motivates the feature. Add
a dedicated constant and predicate with a comment naming that Rust function.

Do **not** reuse either existing predicate; both are wrong here and the difference is
silent:

- `POMODORO_NAVIGATION_STATUSES` (`main.js:43`) includes `x`/`X` — it would prune
  completed Pomodoros.
- `block-id-prompt`'s `isPomodoroEntryLine` additionally requires a `()` placeholder or
  a time range — it would skip a plain `- [ ] Future work` entry that
  `task-status-hooks` treats as open.

So `[ ]`, `[/]`, `[*]`, and `[?]` entries are all in scope; `[x]`, `[X]`, and `[-]` are
not. "Current or future" in the request maps onto exactly this, since the current
Pomodoro is just the open entry that has a time range.

**Pomodoro sub-bullet.** Any indented list item between an open entry line and the end
of its block (the next column-0 non-blank line, or the end of the section). Nesting
depth does not matter.

**Matched link.** A wiki block link `[[<target>#^<id>]]` or embed `![[<target>#^<id>]]`,
optional `|alias`, that is **not** inside a `~~...~~` span. Struck links are retired
records of work already done: `task-status-hooks` skips them when building
`raw_references`, and deleting them would destroy history. Reuse
`recoveryStrikethroughSpans` (`main.js:4791`) for the span test — its containment rule
already matches the Rust one.

**Identity.** Resolved vault-relative path + block ID, via
`createScheduledRecoveryNoteIndex` + `resolveScheduledRecoveryNote`: exact path first,
then unique case-insensitive basename, empty target = the daily note itself. An
ambiguous basename resolves to `null`; that occurrence is skipped and counted as
unresolved, never guessed.

**Removal unit.**

- The link (including a leading `!` and an optional leading `🍅 ` marker) is the
  bullet's **entire body** → delete the whole list-item subtree: that line and every
  deeper-indented line beneath it, bounded by the entry block.
- Otherwise → delete only the link token and its `!`/`🍅 ` prefix, then collapse the
  resulting double space and strip trailing whitespace. Authored prose and unrelated
  links on that bullet survive.

This is `block-id-prompt`'s split, chosen there for the same reason. It is deliberately
narrower than `task-status-hooks`' canceled-reference rule, which always deletes the
subtree.

### Scope: which writes trigger a prune

Trigger when, and only when, a picker write leaves a task carrying a **strictly future**
task-level schedule relative to the writer's own `today`/`baseDate`:

| Gesture                                                   | Pruned targets                                                                                              |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `scheduled` → future date (typed, preset, or pinned roll) | the one task                                                                                                |
| `priority` → P1–P4 (rolls a future date)                  | the one task                                                                                                |
| `N<Ctrl+Shift+P>` → either of the above                   | every counted target whose own resulting date is future                                                     |
| `^prj` project `scheduled` → future date                  | every ordinary task that `planProjectTaskSchedules` gave a future schedule, **plus the `^prj` task itself** |

Including `^prj` in its own prune is a deliberate call, and the reviewer should veto it
here if unwanted: `^prj` never receives an inline schedule, so it is the one case where
`task-status-hooks` really would mark the deferred task itself `[*]` from a lingering
Pomodoro link.

A target with no trailing `^block-id` cannot be linked from a Pomodoro, so it
contributes nothing.

**Not triggered by:** `Ctrl+D` (removing `scheduled` or `priority`), a today/past date,
an unchanged counted target, a task whose status the picker leaves alone, or any
non-`scheduled` property. Removal is never undone — re-adding a link is a manual
`Ctrl+Shift+O`/`^^` gesture.

### Where the daily note is, and how it is written

Today's ledger is
`scheduledRecoveryDailyPaths(vault.getMarkdownFiles(), today).current`. No such file →
clean no-op.

Two cases, and the difference matters:

1. **The edited note _is_ today's daily note.** Fold the prune into the primary write's
   own content so it lands in one editor transaction. This is real: tasks captured
   through `bob capture`'s Pomodoro routes live in the daily note.
2. **Separate files** (the normal case). Apply the primary write first, then prune the
   daily note as a second, guarded write.

For case 2, follow `writeTaskMoveChange` exactly: if the daily note is open in a
Markdown view, use its editor with `applyEditorContentTransaction` after checking the
preimage; otherwise `vault.process(file, ...)` with the same preimage check inside the
callback. Read the preimage from the open buffer when there is one
(`getOpenMarkdownBufferContents` already reports its own ambiguity), else
`vault.cachedRead`.

**Ordering and failure policy.** The schedule is the user's intent and it is already
durable when the prune runs; a prune failure must never roll the schedule back, and
there is no retry. Report it in the notice and stop. Conversely the async snapshot read
happens _before_ the primary write, inside the writers' existing "read async, then
re-guard" window (the one `buildTargetScheduledRecoveryByLine` already uses at
`main.js:15856`, `15050`, `15585`) — reuse that re-guard rather than adding a second,
weaker one.

## Implementation

### 1. Pure helpers (module scope, next to the existing Pomodoro predicates ~line 5319)

Add and export:

- `POMODORO_LEDGER_CLOSED_STATUSES = new Set(["x", "X", "-"])` and
  `isOpenPomodoroLedgerEntryLine(lineText)` — column-0 list item, checkbox present,
  status not closed. Comment it as mirroring `bob-cli`'s `pomodoro::open_ledger_task`.
- `findPomodorosSectionRange(content)` → `{ startLine, endLine } | null`, using
  `getMarkdownLineContexts` so frontmatter and fenced blocks are excluded and the
  section ends at the next unfenced `## ` heading.
- `collectOpenPomodoroRanges(lines, contexts, section)` →
  `[{ entryLine, startLine, endLine }]`.
- `collectPomodoroBlockLinkOccurrences(lineText)` →
  `[{ start, end, target, blockId, struck, embedded, markerStart }]`.
- `planDeferredPomodoroLinkCleanup(dailyContent, targets, options)` →
  `{ content, changed, removedBulletCount, removedLinkCount, removedTargets, unresolvedCount }`,
  where `targets` is an iterable of `{ path, blockId }` and `options` carries
  `{ dailyPath, noteIndex }`. Build subtree edits first, drop token edits already
  covered by one (`editContainsReference`), sort, apply back-to-front, and rejoin with
  `splitMarkdownContent`'s `lineEnding`.
- `deferredPomodoroTargetsFromLines(sourcePath, lines, lineNumbers)` → frozen
  `{ path, blockId }` list, skipping lines with no trailing block ID.

### 2. Snapshot + write plumbing (plugin methods)

- `async readDeferredPomodoroSnapshot(app, { sourcePath, sourceContent, today })` →
  `{ dailyPath, file, editor, content, noteIndex, sameFile } | null`. Returns `null`
  (clean no-op) when the vault API is missing, buffers are ambiguous, there is no daily
  note, or the read throws. When `sameFile`, `content` is the caller's post-write source
  content and `file`/`editor` are unused.
- `async writeDeferredPomodoroCleanup(snapshot, plan)` → `true`/`false`, using the
  `writeTaskMoveChange` guarded pattern. Never throws to the caller; a `false` becomes a
  notice chip.

### 3. Writer integration

For each of the three writers, in the block that already computes the
future-schedule/Blocked outcome:

- `setInlineBulletPropertyValues` — after the existing
  `blocked = blockedLine !== nextLine` branch. Take the snapshot in the same async
  pre-write window as `buildTargetScheduledRecoveryByLine`; note this branch currently
  only reads asynchronously in the _recovery_ case, so extend the existing
  `shouldRecover` guard block to also cover the future case, keeping one re-guard for
  both.
- `planCountedBulletPropertyBatch` — return a new `futureScheduledTaskLines` array
  (original line numbers) alongside `blockedTaskCount`; both
  `setCountedBulletPropertyValue` and `setCountedBulletPriorityValue` consume it.
- `planProjectTaskSchedules` — return a new `futureScheduledTaskLines` array covering
  the propagated ordinary tasks, and include the `^prj` line when `future` is true;
  surface it through `planProjectScheduledUpdate` to `setProjectNoteScheduledValue`.

### 4. Notices

Thread `removedPomodoroLinkCount`, `removedPomodoroBulletCount`, and
`pomodoroPruneFailed` into the outcome objects, and extend
`getPriorityNoticeOutcomeParts` (~11604), `getPriorityNoticeChipTone` (~11655), and
`getPriorityNoticeChipText` (~11671):

| Condition             | Part text                    | Chip          | Tone   |
| --------------------- | ---------------------------- | ------------- | ------ |
| `removed > 0`         | `removed N Pomodoro link`    | `N removed`   | `info` |
| `pomodoroPruneFailed` | `Pomodoro links not removed` | `not removed` | `warn` |

The plain-`Notice` fallbacks in `setInlineBulletPropertyValues`,
`setProjectNoteScheduledValue`, and `setCountedBulletPropertyValue` get the same
suffixes, matching how `scheduleLogSuffix` is already appended there.

No `styles.css` change: both chips reuse existing `info`/`warn` tones.

No config surface. `scheduled` is hard-scoped in code, consistent with the other
`scheduled` special cases in `main.js`.

## Edge cases the implementation must handle

1. **No daily note today** — clean no-op, no notice chip.
2. **Daily note has no `## Pomodoros` section** — no-op.
3. **Every Pomodoro entry is closed** — no-op; `[x]`, `[X]`, and `[-]` blocks are never
   touched.
4. **Struck link under an open entry** (`~~[[Tasks#^x]]~~`) — left in place.
5. **Embed** (`![[Tasks#^x]]`) — removed; embeds are `block_link_occurrences` in Rust
   and do promote.
6. **Alias** (`[[Tasks#^x|do the thing]]`) — removed; the alias sits after the block ID.
7. **Same-note link** (`[[#^x]]` in the daily note) — resolves to the daily note itself.
8. **Explicit path vs unique basename** — `[[Areas/Tasks#^x]]` and `[[Tasks#^x]]` both
   match the same task.
9. **Ambiguous basename** — skipped, counted as unresolved, never guessed.
10. **Mixed-content bullet** (`Review [[a#^x]] and [[b#^y]]`) — only the matched token
    goes; spacing is normalized; the bullet and the other link survive.
11. **Dedicated bullet with nested children** — the whole subtree goes.
12. **Two matched links on one dedicated bullet** — one subtree deletion,
    `removedLinkCount` still counts both.
13. **Stray `🍅 ` marker under an open entry** — consumed with the link, never left
    orphaned.
14. **Fenced code inside the section** — untouched.
15. **Bullets outside `## Pomodoros`, and the entry lines themselves** — untouched.
16. **Task with no trailing `^block-id`** — contributes no target.
17. **Source note is today's daily note** — one transaction; the cursor still lands on
    the edited task line.
18. **Daily note open in another tab with unsaved edits** — the buffer is the preimage
    and the editor is the write path.
19. **Daily note changed between snapshot and write** — preimage mismatch; prune
    skipped, schedule kept, `not removed` chip.
20. **CRLF daily note** — line endings preserved.
21. **Counted session where only some targets get a future date** — only those targets
    prune.
22. **Re-running the same gesture** — idempotent; the second run removes nothing and
    shows no chip.

## Testing

All tests go in `scripts/test-navigation-hotkeys.cjs`, in its existing style
(`node:test`, `node:assert/strict`, string fixtures, helpers pulled from
`module.exports`, `TransactionEditor` and the `plugin.app` vault stub around lines
2307–2440 and 3228 as models).

Pure-planner coverage — one test per numbered edge case above where it is a content
question (1–16, 20), asserting exact resulting content plus `removedBulletCount` /
`removedLinkCount` / `unresolvedCount`. Also cover `isOpenPomodoroLedgerEntryLine`
directly against `[ ]`, `[/]`, `[*]`, `[?]`, `[x]`, `[X]`, `[-]`, an indented child, and
a non-checkbox line, and `findPomodorosSectionRange` against frontmatter, a fenced
`## Pomodoros`, and a following `## ` heading.

Planner-level coverage of the batch planners: `planCountedBulletPropertyBatch` and
`planProjectTaskSchedules` return the expected `futureScheduledTaskLines` for a mixed
batch (future / today / unchanged / no-block-ID / `^prj`).

Runtime coverage through the plugin methods, with a stubbed vault:

- future `scheduled` on a single task prunes the daily note and reports
  `removed 1 Pomodoro link`;
- a `priority` P2 roll prunes the same way;
- a today's-date and a past-date write prune nothing;
- a counted session prunes exactly its future targets;
- an `^prj` project schedule prunes the propagated tasks and the `^prj` link;
- no daily note → the schedule still writes, no chip;
- daily note open in an editor → written by editor transaction, one undo group;
- daily preimage changed under the snapshot → schedule kept, `not removed` chip, daily
  note byte-identical;
- `vault.process` throwing → same outcome, no exception escapes;
- source note is the daily note → single transaction, cursor preserved;
- running the gesture twice is idempotent.

Run from the `bob-plugins` repo root:

```bash
npm test
node scripts/validate-manifests.mjs
```

## Docs and release

1. **`plugins/bob-navigation-hotkeys/manifest.json`** — bump `version` `1.20.0` →
   `1.21.0`.
2. **`bob-plugins/README.md`** — update the version column and extend the Bob Navigation
   Hotkeys description row to mention that a future `scheduled` date removes the task's
   live links from today's open Pomodoro entries.
3. **`bob-cli/docs/projects.md`** — add a subsection after _"Schedule-log reason
   prompt"_ documenting: the trigger set (all four gestures in the scope table), the
   open-entry definition and its tie to `bob task-status-hooks`, that struck links and
   closed/cancelled entries are never touched, the dedicated-bullet-vs-token removal
   split, that only today's daily note is touched, that `Ctrl+D` does not restore
   anything, and the notice chips. Include a worked before/after Markdown example.
4. **`bob-cli/docs/task-status-hooks.md`** — one cross-reference sentence in _"Sync
   Rules"_ noting that the picker removes deferred tasks' links from open Pomodoros
   before this command runs, so the command sees a smaller root set.
5. **Deploy**, after committing in the `bob-plugins` checkout — `-r` is required because
   the default repo path does not exist in a SASE workspace:

   ```bash
   bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run
   bob plugins sync -p bob-navigation-hotkeys -r "$PWD"
   ```

   Then tell the user to reload the plugin in Obsidian.

## Out of scope (file follow-up task beads if wanted)

- Any change to `bob task-status-hooks` or other Rust code.
- A `bob capture <text> p:<N>` / `s:<N>` CLI equivalent that prunes the ledger.
- Restoring pruned bullets when `Ctrl+D` removes the schedule, or when a schedule is
  moved back to today.
- Pruning links from the _previous_ daily note, or from any note other than today's
  ledger.
- Pruning on the `!` dependency-sync or `Ctrl+Shift+M` task-move gestures.
