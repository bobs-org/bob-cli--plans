---
tier: tale
title: Propagate Ctrl+Enter closure to embedded task transclusions
goal:
  Pressing <Ctrl+Enter> on an Obsidian task line whose body is an embedded block
  transclusion closes (and reopens) both the local checkbox and the transcluded source
  task, running the full close side-effects — reference retirement and Blocked-dependent
  recovery — against the source task's identity.
proposed_by: bbugyi200.athena.v0
create_time: 2026-08-07 16:18:46
status: wip
---

# Plan: Propagate `<Ctrl+Enter>` closure from a task line to its embedded transclusion

## Goal

Make the Task Status Cycler plugin's `<Ctrl+Enter>` open/done action, when it fires on
an Obsidian task line whose body is an embedded block transclusion — the shape used by
every task in `~/bob/sase_blog_blockers.md`:

```markdown
- [ ] #task ![[sase#^better-gate-inputs]] [created::2026-08-07]
```

— close **both** the local checkbox in the project note **and** the transcluded source
task (`sase.md` `^better-gate-inputs`), and run the full set of close side-effects
against the source task's identity: retiring/striking eligible embedded block links to
it (daily-note Pomodoro and task descendants) and recovering Blocked dependents that
named it.

Reopening must stay symmetric: `<Ctrl+Enter>` on `- [x] #task ![[...]]` reopens the
local line and the transcluded source together, restoring retired references.

## Findings

All line references are to `plugins/task-status-cycler/main.js` in the `bob-plugins`
linked repo (open it with `/sase_repo`; do not edit `~/bob/.obsidian/plugins/`
directly).

### Current behavior is local-only

`handleVimTaskToggleOpenDone` (`:4897`) dispatches in this order:

1. Open-Pomodoro task line → `completeActivePomodoroTask`.
2. Done-Pomodoro task line → `reopenActivePomodoroTask`.
3. Inside an **open Pomodoro's child range** (`:4933`): if the active line carries an
   embedded block link, delegate to `handleActiveTaskBlockLinkOpenDone`; otherwise
   complete the owning Pomodoro.
4. **`if (this.isOpenDoneTaskStatus(taskStatus))` (`:4963`)** →
   `toggleActiveCheckboxOpenDone`
   - `finalizeClosedTasks([localIdentity])`, then `return`.
5. Fallthrough (`:4985`) → `handleActiveTaskBlockLinkOpenDone`, the only path that
   resolves `![[note#^id]]` and writes the target.

A `sase_blog_blockers.md` line is itself a `- [ ]` task line (matches `TASK_LINE_RE`,
`:13`) and sits in a note with no `## Pomodoros` section, so branches 1–3 are all null
and **branch 4 consumes the keypress**. Branch 5 is never reached, so the transclusion
is never followed.

The `finalizeClosedTasks` call in branch 4 is additionally a no-op for these lines:
`closedTaskIdentity` (`:2384`) requires a trailing `^block-id` or a Tasks `[id:: ...]`,
and the blockers lines carry neither. So today nothing at all happens beyond flipping
one checkbox.

### The asymmetry is already visible today

Branch 3 shows the intended semantics already exist — a line with an embedded
transclusion _under an open Pomodoro_ does follow the embed, even when that line is
itself a `#task` checkbox. The bug is that the same line outside a Pomodoro child range
does not. The existing test
`"Ctrl+Enter behavior stays generic outside open Pomodoro child ranges"`
(`scripts/test-task-status-cycler.cjs:2878`) confirms a _non-task_ bullet
(`- [[#^target]]`) already closes its target outside a Pomodoro; only task-shaped lines
short-circuit.

### Pieces that already exist and should be reused, not rewritten

- `getTranscludedTaskTargetFromLine` (`:2011`) / `getActiveLineTranscludedTaskTarget`
  (`:7599`) — select an **embedded** (`![[...]]`) block transclusion from the active
  line, requiring exactly one candidate on the line or an unambiguous cursor hit
  (`getUnambiguousTranscludedBlockCandidate`, `:1996`).
- `resolveTranscludedBlockTarget` (`:7663`) — resolves the link to a file + line via the
  metadata cache with a source-text rescan fallback, filtered by a status predicate.
- `replaceResolvedTranscludedTaskLine` (`:7800`) — writes through the live editor when
  the target is the active note, otherwise through the vault; **revalidates the line
  text and block ID before writing**, so a stale line number degrades to a no-op rather
  than corrupting an unrelated line. Accepts a `forcedNextSymbol`.
- `finalizeClosedTasks` (`:4407`) — serializes Blocked-dependent recovery
  (`recoverBlockedDependentsNow`) then reference retirement
  (`retireClosedTaskReferencesNow`) on the shared `referenceMutationQueue`, and accepts
  a **batch** of identities.
- `restoreReopenedTaskReferences` (`:4388`) — the reopen counterpart.
- `setActiveCheckboxStatus` (`:7149`) — the local write. For `#task` lines it prefers
  the Obsidian Tasks command (`obsidian-tasks-plugin:set-status-symbol-to-x` /
  `...-to-space`), falling back to a metadata-aware local rewrite. This path must be
  preserved exactly so Tasks keeps owning `[completion:: ...]` and recurrence.

### Retirement will not disturb the project note itself

`hasEligibleRetirementAncestor` (`:2256`) rejects any reference line at indentation 0.
Every `## Tasks` line in `sase_blog_blockers.md` is at indentation 0, so closing the
source task will **not** strike out the blockers note's own `![[sase#^...]]` embed. Only
indented references under a `#task` ancestor or a top-level Pomodoro task get retired —
i.e. exactly the daily-note strikethrough behavior the user expects. This is the desired
outcome and should be asserted in tests so a future change to the retirement ancestry
rule cannot silently start mangling project notes.

### Test harness

`scripts/test-task-status-cycler.cjs` already provides `createInMemoryObsidianApp`,
`createTextEditor`, `attachActiveMarkdownView`, `registerTaskToggleVimAction`, and
`flushAsyncActions`, and constructs real `new TaskStatusCyclerPlugin()` instances
against an in-memory multi-file vault. New coverage needs no new scaffolding. The full
suite (`npm test`) is currently green at 346 tests.

## Design decisions

1. **Local status drives intent; the target is forced to match.** Deriving the target's
   transition from its _own_ status (as `handleActiveTaskBlockLinkOpenDone` does) can
   diverge: a local `[x]` over a target `[ ]` would reopen the local line while
   _closing_ the target. Instead:
   - local status closable (`[ ]`, `[*]`, `[/]` — `CLOSABLE_TASK_SYMBOLS`, `:49`) →
     local goes to `[x]` and the target is forced to `x` when it is closable, no-op when
     already `[x]`;
   - local `[x]` → local goes to `[ ]` and the target is forced to `" "` when it is
     `[x]`, no-op otherwise. This keeps the two lines convergent and makes repeated
     presses idempotent per direction.

2. **Embedded transclusions only.** Use the `![[...]]` selector
   (`getActiveLineTranscludedTaskTarget`), not the broader
   `getActiveLineTaskBlockLinkTarget` (`:7622`) which also accepts plain `[[note#^id]]`.
   A `#task` line that merely _links to_ another task is a different relationship from
   one that _is_ a view of it; widening to plain links would change behavior for
   existing daily notes. The blockers-note shape is always an embed.

3. **Non-recursive, single target.** Match the established outside-a-Pomodoro block-link
   behavior (`handleActiveTaskBlockLinkOpenDone` only recurses when
   `getActivePomodoroTranscludedTaskLineTarget` is non-null). Project-note task lines
   are not Pomodoro sub-bullets, so do not walk the target's descendants.

4. **Ambiguity and fenced code fall back to today's behavior.** Two embeds on one line
   with the cursor outside both → no target → plain local toggle, exactly as now. Note
   that `getActiveLineTranscludedTaskTarget` (`:7599`), unlike
   `getActiveLineTaskBlockLinkTarget` (`:7622`), does **not** consult
   `getFencedLineNumbers`; the new branch must apply the fenced-line guard itself so a
   task line inside a fenced block never triggers a cross-file write.

5. **Unresolvable targets stay silent and still toggle locally.** A missing file,
   missing block ID, or non-task target must not swallow the keypress or abort the local
   write — the user still gets the local close. This matches every other transclusion
   path, which treats resolution failure as best-effort.

6. **Order: local write first, then resolve and write the target.** The Tasks-plugin
   command owns the local line and may insert a new line above it for recurring tasks;
   resolving the target _after_ the local write means the target's line number is read
   from post-write state. Cross-file targets (the normal case) are unaffected either
   way; same-file `![[#^id]]` embeds are protected by
   `replaceResolvedTranscludedTaskLine`'s revalidation.

7. **One batched finalize.** Collect the local identity (often null) and the target
   identity into a single `finalizeClosedTasks` / `restoreReopenedTaskReferences` call
   so dependent recovery sees both closures in one post-close vault snapshot rather than
   racing two queued passes.

## Implementation plan

1. **Add a shared helper on the plugin class**, e.g.
   `async closeOrReopenTranscludedTaskLine(editor, activeFile, taskStatus, closing)`,
   placed near `toggleActiveTranscludedTaskOpenDone` (`:6232`). It should:
   - return `null` when the active line is fenced or yields no unambiguous embedded
     transclusion candidate (per decisions 2 and 4);
   - build the standard `{ editor, activePath, originPath: activePath }` context;
   - `resolveTranscludedBlockTarget` with `taskStatusPredicate: isOpenDoneTaskStatus`,
     swallowing throws;
   - when `closing`, write only if
     `isTranscludedCompletionClosableStatus(target.taskStatus)` with
     `forcedNextSymbol: "x"`; when reopening, write only if
     `isTranscludedReopenableStatus(target.taskStatus)` with `forcedNextSymbol: " "` and
     `{ forcedStatusPredicate: isTranscludedReopenableStatus }`, mirroring
     `reopenResolvedTranscludedTaskTarget` (`:6360`);
   - return the target identity —
     `closedTaskIdentity(target.file.path, target.taskStatus.lineText) || { path, blockId }`
     — when a write actually landed, `null` otherwise.

2. **Rewrite branch 4 of `handleVimTaskToggleOpenDone` (`:4963`)** to keep the existing
   local write and then fold in the target. Sketch:
   - capture the candidate before the local write (the embed survives the rewrite, but
     capturing first keeps the selection independent of Tasks-command timing);
   - `const wrote = this.toggleActiveCheckboxOpenDone(...)` — unchanged;
   - `const closing = isTranscludedCompletionClosableStatus(taskStatus)`;
   - if `wrote`,
     `void (async () => { const targetIdentity = await this.closeOrReopenTranscludedTaskLine(...); const identities = [localIdentity, targetIdentity].filter(Boolean); if (identities.length) { closing ? await this.finalizeClosedTasks(identities, context) : await this.restoreReopenedTaskReferences(identities, context); } })().catch(() => {})`;
   - `return` as before, so the keypress stays consumed.

   Do **not** touch branches 1–3 or the fallthrough at `:4985`.

3. **Apply the same change to `handleToggleOpenDoneCommand` (`:6151`)**, the non-Vim
   command spelling. It duplicates branch 4's logic verbatim today; factoring the shared
   "toggle local + propagate + finalize" body into one method called by both is
   preferable to copying it a second time. Existing tests
   `"non-Vim open/done command finalizes the closed task identity"` (`:2148`) and
   `"...restores the reopened task identity"` (`:2169`) must keep passing unchanged.

4. **Extend `scripts/test-task-status-cycler.cjs`** with a new test group modeled on
   `"Ctrl+Enter behavior stays generic outside open Pomodoro child ranges"` (`:2878`),
   using a two-file in-memory vault shaped like the real one (a `Blockers.md` project
   note, a `Source.md` with `- [/] #task ... ^target`, and a `Daily.md` with an indented
   `![[Source#^target]]` under a Pomodoro). Cover:
   - closing `- [ ] #task ![[Source#^target]] [created::...]` sets the local line to
     `[x]` **and** `Source.md`'s task to `[x]`;
   - the daily note's indented reference becomes `~~[[Source#^target]]~~` (retirement
     fired against the _target_ identity);
   - the blockers note's own indentation-0 embed is left untouched (decision / finding
     above);
   - a Blocked `[?]` dependent naming the target recovers to `[ ]`;
   - `[x] → [ ]` reopen restores both lines and un-retires the daily-note reference;
   - a local `[x]` over an already-`[ ]` target reopens locally and leaves the target
     `[ ]` (no accidental close — decision 1);
   - an already-`[x]` target with an open local line closes the local line without a
     second write or a duplicate identity;
   - an unresolvable embed (`![[Missing#^nope]]`) still closes the local line and raises
     no error;
   - a two-embed task line with the cursor outside both embeds toggles locally only;
   - a task line inside a fenced code block is not treated as a transclusion source
     (decision 4);
   - the Tasks-plugin command path is still used for the local write on a `#task` line
     (assert `executeCommandById` receives `set-status-symbol-to-x`), reusing the
     pattern in
     `"Vim Ctrl+Enter dispatches task transitions and restores a reopened identity"`
     (`:2947`).

5. **Bump and document.** In `bob-plugins`: raise
   `plugins/task-status-cycler/manifest.json` to `1.8.0` (new user-visible behavior, no
   breaking change) and update its `description`; update the Task Status Cycler row of
   the plugin table in `README.md` to mention that `<Ctrl+Enter>` on a task line
   wrapping an embedded transclusion closes/reopens the transcluded source too, and that
   retirement still only touches indented references. No change is needed in the
   `bob-cli` repo: its `README.md` and `docs/task-status-hooks.md` describe the
   Ctrl+Enter recovery _contract_ ("after that keypress actually changes one or more
   tasks to Done…"), which this work widens the input set of but does not alter.

6. **Validate**, from the `bob-plugins` workspace checkout:

   ```bash
   npm test
   npm run validate
   ```

7. **Deploy to the vault** (the linked-repo workflow; `bob plugins sync` defaults to a
   source path that does not exist in a SASE workspace, so point it at the checkout):

   ```bash
   bob plugins sync -p task-status-cycler -r "$PWD" --dry-run
   bob plugins sync -p task-status-cycler -r "$PWD"
   ```

   If sync reports the file as "dirty in vault", verify the on-disk vault copy matches
   `git show HEAD:plugins/task-status-cycler/main.js` in the vault repo before
   considering `-F/--force`. Tell the user they must reload the plugin in Obsidian for
   the new `main.js` to take effect.

8. **Manual check** in the vault after reload: put the cursor on
   `- [ ] #task ![[sase#^better-gate-inputs]] [created::2026-08-07]` in
   `sase_blog_blockers.md`, press `<Ctrl+Enter>`, and confirm the blockers line becomes
   `[x]`, `sase.md` line 128 becomes `[x]`, and any indented
   `![[sase#^better-gate-inputs]]` reference in today's daily note is struck through.
   Press `<Ctrl+Enter>` again on the same line to confirm both reopen.

## Expected result

`<Ctrl+Enter>` on any `#task` line whose body is a single embedded block transclusion
closes the local checkbox and the transcluded source task together, then runs
Blocked-dependent recovery and reference retirement against the _source_ task's identity
— so block links to it in the daily note get struck out exactly as they do when the
source task is closed directly. Pressing it again reopens both and restores the retired
references. Pomodoro completion, Pomodoro sub-bullet recursion, plain non-task
block-link bullets, canceled and custom statuses, and Tasks-plugin metadata ownership
all behave exactly as before.
