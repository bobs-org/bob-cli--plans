---
tier: epic
title: Close the running Pomodoro with =x in bob capture and Bob Mac Capture
goal: 'A capture item `=x` closes today''s running Pomodoro exactly the way Obsidian''s
  Pomodoro completion does, and also shortens a session stopped early. The forms `@route:block-id=x`,
  `^route:block-id=x`, and `<text> @route:block-id=x` first put that task into the
  running session. Every close is atomic. A missing, ambiguous, or malformed target
  gives an actionable diagnostic. Bob Mac Capture shows a rich, accurate preview of
  the session about to close.

  '
phases:
- id: close-ledger
  title: Pomodoro close engine, daily-note half
  depends_on: []
  size: medium
  description: 'close-ledger: add a pure module that finds the running Pomodoro and
    computes the early-stop auto-decrement. It ports Obsidian''s completion rewrite
    of the daily note: sub-bullet classification, tomato markers, deferred-link removal,
    and carrying links into a new placeholder. It exposes the planned task effects
    and Work Log note groups as data, with unit tests pinned to the worked example.'
- id: close-tasks
  title: Pomodoro close engine, linked-task effects and Work Log
  depends_on:
  - close-ledger
  size: medium
  description: 'close-tasks: share task-status-hooks'' vault link resolver. Port the
    linked-task side of completion: start bare-linked tasks `[/]`, close embedded
    targets recursively with a completion date, write dated Work Log entries, and
    retire closed embeds. Wrap both halves in one `plan_pomodoro_close` entry point
    over an injectable vault, with unit tests.'
- id: close-capture
  title: =x grammar, atomic capture transaction, and outputs
  depends_on:
  - close-tasks
  size: medium
  description: 'close-capture: recognize whole-item `=x` and the `=x` suffix on solo
    `@`/`^` links and body-bearing `:` captures, together with their near-miss and
    conflict errors. Wire the close through CaptureBatchPlanner, including the link-into-running
    step. Emit the `pomodoro_close` kind and object plus human output, point the start
    guards at `=x`, and add integration tests.'
- id: close-editor-contract
  title: Editor contract, help, and docs for =x
  depends_on:
  - close-capture
  size: medium
  description: 'close-editor-contract: add the capture-parse `pomodoro_close` mode,
    span, spec, incomplete state, and `invalid_pomodoro_close` diagnostics. Make capture-complete
    and capture-rewrite ignore the close suffix. Update the help for capture, capture-parse,
    and capture-complete, and update docs/capture.md, the task-status-hooks note,
    and the README.'
- id: mac-close-preview
  title: Bob Mac Capture close preview, footer, and notifications
  depends_on:
  - close-capture
  size: medium
  description: 'mac-close-preview: in bob-mac-capture, decode the additive `pomodoro_close`
    contract. Render the dedicated close card (session, timing chip, task rows with
    transitions and Work Log previews, next session), including the variants for link
    and new-task closes. Add the Close footer and notifications, fix stale preview
    state, and add fake-bob fixtures generated from real bob output, tests, README
    updates, and green macOS CI.'
proposed_by: bbugyi200.apollo.2i
create_time: 2026-09-28 06:24:48
status: done
bead_id: bob-cli-29
---

- **PROMPT:** [prompts/202609/capture_pomodoro_close.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/capture_pomodoro_close.md)
- **BEAD:** [bob-cli-29](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-29/README.md)

# Plan: Close the running Pomodoro with `=x`

## Why this exists

The user wants `bob capture` to accept `=x`, which closes (stops) the currently running
Pomodoro. `@file:id=x` and `^file:id=x` should first pull the existing task `^id` from
`file.md` into that Pomodoro. `=x` on its own should just close the session. When
nothing is running, the user should get a useful diagnostic. Bob Mac Capture should show
an excellent preview of the Pomodoro about to close.

The user's own vault task for this feature (`bob.md ^capture-stop`) reads: "Add support
for `=x` syntax to **auto-decrement (if necessary)** and stop pomodoro!" A session
closed before its planned end is therefore shortened to the stop time.

Today:

- `=x` becomes an ordinary inbox task, `- [ ] #task =x`.
- `@route:id=x` is a usage error, because `x` is not a valid `=<X>` start suffix.
- `^route:id=x` fails the same way.
- Nothing in bob-cli ever completes a Pomodoro. `docs/capture.md` says Bob "never
  overwrites, completes, or double-starts an entry".

### Which Obsidian keymap this ports (read this first)

The request points at the `<ctrl+shift+enter>` Obsidian keymap. In bob-plugins, that
chord is block-id-prompt's `link-task-to-pomodoro` command (`openPomodoroTaskLink`). It
is **task-level**:

- a Ready or Blocked task links into today's Pomodoro and becomes Next;
- a Next task becomes Open and loses its current/future links;
- an In Progress task is paused through a work-summary prompt.

It never acts on a Pomodoro entry line. On an entry it only shows "No open task under
cursor".

The action that actually **closes a Pomodoro** is task-status-cycler's completion: the
Vim `<C-CR>` (Ctrl+Enter) action on an open entry, through `handleVimTaskToggleOpenDone`
→ `completeActivePomodoroTask`. It is also the fallback when Ctrl+Enter lands on a
non-task child line. `=x` ports **that** flow, with byte parity, plus the
auto-decrement.

The Ctrl+Shift+Enter pause semantics are deliberately not part of `=x`. That flow sets
In Progress tasks back to Open and deletes their current-session links, which is the
opposite of logging the work.

Port from the source, not from memory. Open bob-plugins with `/sase_repo`: run
`sase repo open bob-plugins`, or, if that linked checkout is unavailable on the host,
`sase repo open gh:bobs-org/bob-plugins`. Then read
`plugins/task-status-cycler/main.js`:

| Function                                                                                                         | What it covers             |
| ---------------------------------------------------------------------------------------------------------------- | -------------------------- |
| `completeActivePomodoroTask`                                                                                     | Orchestration              |
| `buildPomodoroCompletionPlan`, `applyPomodoroCompletionPlan`                                                     | Ledger edits               |
| `classifyPomodoroSubBullets`                                                                                     | Sub-bullet buckets         |
| `getMoveOnlyPomodoroBlockLinkFromListItem`                                                                       | Deferred `#` links         |
| `rewritePomodoroMarkersInLine` with `completedPomodoroMarkerPolicy`                                              | 🍅 markers                 |
| `getSubBulletBlockRange`, `findNextPomodoroLine`, `parsePomodoroEntryLineParts`, `formatPomodoroPlaceholderLine` | Ranges, next entry, names  |
| `startPomodoroNonTranscludedTaskBullets`                                                                         | Starting bare-linked tasks |
| `completePomodoroTranscludedTaskBullets`                                                                         | Recursive embedded close   |
| `removeCompletionField` / `normalizeTaskMetadataSpacing`                                                         | Spacing rules              |
| `collectPomodoroWorkLogNoteGroups`, `getPomodoroWorkLogTaskLinkTarget`, `collectPomodoroWorkLogDescendantTree`   | Work Log note collection   |
| `buildWorkLogEntryLines`, `planPomodoroWorkLogGroupInsertion`, `childIndentUnitForIndent`                        | Work Log writing           |
| `getPomodoroWorkLogDateString`                                                                                   | Work Log date              |
| `finalizeClosedTasks`                                                                                            | Retirement                 |

The tests in `scripts/test-task-status-cycler.cjs` (the Pomodoro completion and Work Log
groups) are the executable spec. Mirror their cases.

## The contract (shared by every phase)

### Terms

- **Ledger:** today's daily note, resolved as everywhere else. That is `BOB_DAY_FILE`,
  else `<bob-dir>/YYYY/YYYYMMDD.md` from `BOB_NOW`/`DATE`/the local clock. Its
  `## Pomodoros` section (heading suffixes allowed) is the ledger.
- **Running Pomodoro _R_:** the single open (`[ ]`), column-0, timed entry in the
  ledger. This is the same selection `+N`/`-N` uses (`capture_pomodoros::scan`,
  filtering `Open && time_range.is_some()`).
- **Clock:** `now` is `bob_env::current_datetime()`, and only its hour and minute are
  used.
- **Work Log date _D_:** the daily note's own date when its file name parses with
  `pomodoro::parse_day_file_date`, else the clock's date.
- **Completion date:** the clock's date.

### Grammar

| Item                             | Meaning                                                                          |
| -------------------------------- | -------------------------------------------------------------------------------- |
| `=x`                             | Close _R_                                                                        |
| `@route:block-id=x` (whole item) | Put the existing `^block-id` task of `route.md` into _R_, then close _R_         |
| `^route:block-id=x`              | Identical execution; `^` is the active-task spelling                             |
| `<text> @route:block-id=x`       | Create the new Pomodoro-linked task in _R_ (today's `:` capture), then close _R_ |

- `x` is case-insensitive (`=X` is accepted). `raw` preserves what was typed. Docs and
  help show `=x`.
- **Whole-item `=x`.** The item's trimmed text is exactly `=x`, on a single physical
  line. Leading and trailing whitespace is fine.
  - If the first token of an item's first line is exactly `=x` but the item has anything
    else, the item fails with an `invalid_pomodoro_close` error and never becomes task
    text. "Anything else" means more text, a marker, `s:`/`p:`/`%`, or an authored child
    line. The error reads: "`=x` must be the whole capture item; to log a task while
    closing, use `@route:block-id=x`". This follows the `+N` near-miss precedent.
  - A whole item that is only `=` is an **incomplete** editing state. Strict capture
    fails with "`=` is incomplete: type `=x` to close the running Pomodoro".
  - Other first tokens that start with `=` (`=3`, `=xx`, `=x!`, `==`) and a mid-body or
    trailing `=x` (`Plan =x`) stay ordinary prose, exactly as today.
- **The `=x` suffix.** It takes the place of the `=<X>` start suffix wherever that
  suffix is legal: the solo `@`/`^` link item and the body-bearing `:` capture.
  - The shared post-sigil parser (`parse_colon_link_tail`) returns a session suffix of
    either `Start(PomodoroStartSpec)` or `Close(PomodoroCloseSpec { raw })`.
  - Project-note `+` forms reject it, as they reject `=<X>`.
  - Ensure Next (`@route+id`) and `!` never accept `=`, as today.
- **Conflicts.** Each of these gives a specific error, and nothing is written:
  - `#pomodoro` together with `=x`: "`=x` always closes the running Pomodoro; remove
    `#name` (drop `=x` to link under a named Pomodoro instead)";
  - `=x` with `s:<N>` or `p:<N>`: a scheduled task starts Blocked and cannot be worked
    in the closing session;
  - forced flags (`--route`, `--section`, `--task`, `--task-ref`, `--task-section`,
    `--clip`, `%`) on a whole-item `=x`, exactly as for adjustments;
  - the existing solo-link conflicts (children, clip, `s:`/`p:`) keep their messages.
- **`@@`.** A `@@route` declaration never applies to `=x` items and never turns one into
  a task, matching `+N` and `^`.

### Selecting _R_, and diagnostics

All of the following fail before anything is written. The quoted text is the required
substance; tests assert on the stable leading phrase.

- **Missing day file:** "no running Pomodoro to close: today's daily note
  `2026/20260928.md` does not exist".
- **Missing Pomodoros section:** reuse the existing phrasing.
- **No open timed entry:** "no running Pomodoro to close: today's ledger
  (`2026/20260928.md`) has no open timed entry". When an open placeholder exists,
  continue with "; next up is CAPTURE at line 13". Link forms add "(to start a session
  with this task instead, use `…=`)", naming the item's own spelling.
- **Several open timed entries:** "multiple open timed Pomodoros (CAPTURE at line 5,
  SASE at line 14); finish all but one before closing".
- **Unparseable range:** a timed entry whose range cannot be parsed by the adjustment
  parser still closes, but auto-decrement is skipped. A warning says the time range was
  left as written.

The existing start guards keep their text and append a pointer to the new syntax.
Existing tests keep passing because the phrase "finish the current Pomodoro first" is
unchanged.

| Guard                                    | New text                                                                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Generic start guard                      | "Bob daily note has an active timed Pomodoro; finish the current Pomodoro first (close it with `=x`)"                    |
| Task already queued in the running entry | "Pomodoro NAME is already running; finish the current Pomodoro first (close it with `=x`) or use `+N`/`-N` to adjust it" |

### The close transaction

The order of operations mirrors `completeActivePomodoroTask`:

1. snapshot, classify, and collect Work Log notes;
2. rewrite the ledger;
3. apply the effects on linked tasks;
4. write Work Logs;
5. retire closed embeds.

Everything is staged, then committed atomically.

1. **Auto-decrement.** This is the bob-only addition. Parse _R_'s range with the
   adjustment parser (`parse_adjustment_range` and `adjustment_duration_minutes`).
   - Compute `remaining = end − now`, normalized into (−720, 720] minutes.
   - If `remaining ≥ 5`:
     - `units = floor(remaining / 5)`
     - `new_duration = max(0, duration − 5·units)`
     - `new_end = (start + new_duration) mod 1440`
     - Rewrite the range with `format_adjusted_range`, the same canonical
       `(**HHMM-HHMM** [t:: Nm])` form that `+N`/`-N` writes, preserving other metadata.
   - Otherwise leave the range bytes untouched. Bob never extends a session.
   - The new end is therefore the earliest five-minute step at or after `now`. For
     example, 0920–0950 closed at 09:37 becomes 0920–0940 (20m), while closed at 09:49
     or later it is unchanged.
   - A session whose start is still in the future clamps to 0m.
2. **Entry line.** Change only the checkbox, from `[ ]` to `[x]`: no completion date, no
   other edits. The only exception is the range rewrite from step 1.
3. **Sub-bullet range.** The contiguous lines after _R_ that match an indented list item
   (`^[ \t]+(?:[-+*]|\d+[.)])(?:[ \t]+|$)`). The range stops at the first blank line or
   non-list line, and at the section end.
4. **Classification.** Strip 🍅 markers from each line first, then put the line in the
   first bucket that matches:

   | Bucket    | Rule                                                                                          | Effect                                                                       |
   | --------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
   | fenced    | inside a code fence                                                                           | note, untouched                                                              |
   | embedded  | at least one valid `![[…#^id]]`                                                               | not carried; targets closed (step 7)                                         |
   | deferred  | the body is exactly one plain block link immediately followed by `#` (trailing space allowed) | line **removed** from _R_; carried with that `#` dropped; target not started |
   | worked-on | at least one plain block link that is not inside `~~…~~`                                      | line kept, gets 🍅, carried                                                  |
   | startable | a worked-on line whose body is exactly one bare link (alias allowed)                          | target started (step 6)                                                      |
   | note      | everything else                                                                               | stays; not carried                                                           |

   "Everything else" in the note bucket includes prose, struck-only links, and
   `[[x]] #`. The deferred bucket does **not** include `![[x]]#`, `~~[[x]]~~#`,
   `[[x]] #`, `[[x]]##`, `[[x]] #tag`, or `prose [[x]]#`.

5. **Markers.** Every unfenced line of the range that is not removed is rewritten with
   the completed-marker policy, applied per block-link token:
   - embedded: drop any 🍅;
   - exactly struck `~~[[…]]~~`: a single `🍅 ` if it had one or more, otherwise bare;
   - plain: exactly one `🍅 ` (U+1F345 followed by a space).
6. **Carry forward and the next Pomodoro.**
   - The carried lines are all worked-on lines in source order, then all deferred lines
     in source order. Each keeps its original indentation with markers stripped; this
     includes nested lines.
   - A new placeholder is created **iff** something is carried **or** no later top-level
     entry of any status follows _R_ in the section.
   - The placeholder text is `- [ ] ()`, or `- [ ] () — NAME` with _R_'s name and a
     single space either side of U+2014.
   - It is inserted directly after _R_'s sub-bullet range and followed by the carried
     lines. When nothing is carried, it is followed by the stub line `\t- `.
   - Existing later entries are pushed down, never merged.
7. **Linked-task effects.** Resolve each link with the shared vault link resolver (see
   the close-tasks phase). Tasks are deduplicated by resolved path and block ID.
   - **Startable targets:** a `#task` checkbox line with Ready `[ ]` or Next `[*]`
     becomes In Progress `[/]`. The same edit trims trailing whitespace and normalizes
     the space before a trailing `^id` to exactly one (`removeCompletionField` parity).
     All other statuses are unchanged.
   - **Embedded targets:** Ready, Next, or In Progress becomes `[x]`. A `#task` line
     also gets `  [completion:: YYYY-MM-DD]` (two spaces, the completion date), placed
     before a trailing `^id` and replacing any existing completion field.
     - This recurses through sole-content `![[…]]` children of the target's list block,
       at most 25 levels deep and 250 targets.
     - Resolution does not require `#task`.
8. **Work Log.**
   - **Qualifying links.** Only **direct** children of _R_, unfenced, qualify. After
     stripping 🍅, dropping one trailing `#`, and unwrapping one `~~…~~`, the body must
     be exactly one block link, embedded or plain, alias allowed. So worked-on,
     deferred, struck, and embedded links all qualify.
   - **Descendants.** Each qualifying link needs at least one non-blank descendant.
     Descendants are the following lines until the first non-blank line indented no
     deeper than the link, and they **do** cross blank lines. They are collected from
     the pre-close snapshot, and nesting comes from indent widths.
   - **Entry text.**
     - A depth-1 node becomes `<indent>- *D* — <body>`.
     - Deeper nodes keep their marker and are re-indented one unit per level.
     - An empty node is dropped along with its subtree.
     - The indent unit is the parent indent itself when it is non-empty and tab-free,
       otherwise `\t`.
   - **Placement.**
     - **Existing marker:** a direct-child marker recognized by
       `parse_managed_task_log_marker` as the Work kind. Filter by kind: do not use
       `first_direct_managed_log_start`, which also matches Schedule Logs. Entries are
       inserted directly after the marker line, so the newest close is on top. The
       indent and bullet come from the marker's first direct child, else the marker
       indent plus one unit and `-`.
     - **No marker:** append `<P><M> 🛠️ **WORK LOG**` after the task's whole child
       block, which includes any SCHEDULE LOG. `P`/`M` come from the task's first direct
       child, else the task indent plus one unit and `-`. The entries go at `P + unit`,
       with `-`.
     - **Same task linked twice in one close:** the second group goes directly after the
       first, so source order is kept.
   - **Notes stay in the ledger.** They are copied, not moved.
9. **Retirement inside _R_.** Each `![[p#^id]]` in _R_'s range whose target is Done
   after step 7 becomes `~~[[p#^id]]~~`, dropping `!` and any 🍅 before it.

**Also match these plugin quirks.**

- A deferred line's own children stay behind in _R_, orphaned under the preceding line,
  and their notes still reach that task's Work Log.
- Carried nested lines keep their original indentation.
- Only the last of two 🍅 written without a space between them (`🍅🍅 [[x]]`) is
  recognized as a marker.

**Deliberate divergences.** Each keeps writes safe and deterministic.

- **Ambiguous targets are skipped.** An unresolvable, ambiguous (duplicate basename), or
  duplicate-block-ID target, or a non-task line, gets a warning and is left unchanged.
  The plugin writes to the first match.
- **Newlines are preserved.** Line endings follow the file: CRLF when the anchor line or
  the file uses CRLF. A file with no final newline keeps none. The plugin adds one at
  EOF.
- **No vault-wide cleanup.** Blocked-dependent recovery and completed-reference
  retirement outside _R_ (`finalizeClosedTasks`) are left to `bob task-status-hooks`,
  which already owns both. Say so in the docs.
- **Atomic.** The plugin's writes are best-effort. The close is all-or-nothing: day
  file, task notes, and every batch item commit together through `CaptureBatchPlanner`.
  Files whose bytes are unchanged are not staged. `--dry-run` plans the identical
  result.

### Link forms (`@route:id=x`, `^route:id=x`, `<text> @route:id=x`)

- _R_ must exist, as described above. The implicit current/next choice and "respect the
  queue" do **not** apply here: the destination is always _R_.
- **Existing task, solo forms.**
  - Resolve and gate the task exactly like the solo link: missing, duplicate, and
    non-task IDs give the existing errors, and Done or canceled tasks fail.
  - Apply `plan_task_link`: Ready/Blocked become Next, with future-schedule retirement
    and the pull-forward log entry. Next and In Progress are kept.
  - Then place the Task Link. Let _L_ be the task's movable dedicated links
    (`find_movable_task_links`); more than one is the existing invariant error.

    | Where _L_ is              | Action                                                                                           | Report                   |
    | ------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------ |
    | in _R_                    | nothing moves                                                                                    | `already_current`        |
    | in another open entry _Q_ | move _L_'s whole subtree to the end of _R_'s sub-bullet range                                    | `moved`, with source _Q_ |
    | absent                    | append `[[route#^block-id]]` at the end of _R_'s sub-bullet range, using _R_'s child indentation | `linked`                 |

    "End of _R_'s sub-bullet range" is always step 3's contiguous range, so the close
    sees the link.

- **New task, body-bearing form.** It is today's `:` new-task capture: a Next task with
  `^block-id`. Its link goes to _R_, appended the same way, and never to another entry.
- **Then close _R_** on the staged ledger. The link is a bare dedicated link, so it is
  startable: the task ends In Progress `[/]` (Ready → Next → In Progress) or stays In
  Progress. Its notes, including a moved subtree's children, reach its Work Log. The
  link is carried into the new placeholder.
- The route note must not be the day file (existing check). A task whose note is also
  touched by the close is composed through the planner's staged contents, so every step
  reads the latest staged bytes and re-finds tasks by block ID.

### Worked example (use as the shared test fixture)

`BOB_NOW=2026-09-28 09:37:00`. `BOB_DAY_FILE` is `2026/20260928.md` in the vault, with
exactly these lines (TAB indentation):

```markdown
## Pomodoros

- [x] (**0830-0855** [t:: 25m]) — PLAN
  - 🍅 [[bob#^capture-stop]]
- [ ] (**0920-0950** [t:: 30m]) — CAPTURE
  - [[bob#^capture-stop]]
    - Designed the `=x` grammar
      - chose `x` for done
    - Wrote the plan
  - [[bob#^web-capture]]#
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
- [ ] () — SASE
  - [[sase#^recovery-panel]]
```

`bob.md`:

```markdown
## Tasks

- [*] #task Add support for `=x` syntax! [created::2026-09-26] ^capture-stop
- [*] #task Add capture support for web URLs! [created::2026-09-21] ^web-capture
- [ ] #task Plain ready task [created::2026-09-20] ^ready
```

`sase.md`:

```markdown
## Tasks

- [x] #task Restart axe [created::2026-09-27] [completion:: 2026-09-28] ^axe-restart
  - 🛠️ **WORK LOG**
    - _2026-09-27_ — Diagnosed the hang
- [ ] #task Recovery panel [created::2026-09-25] ^recovery-panel
```

**`bob capture =x`**

The daily note becomes:

```markdown
## Pomodoros

- [x] (**0830-0855** [t:: 25m]) — PLAN
  - 🍅 [[bob#^capture-stop]]
- [x] (**0920-0940** [t:: 20m]) — CAPTURE
  - 🍅 [[bob#^capture-stop]]
    - Designed the `=x` grammar
      - chose `x` for done
    - Wrote the plan
  - ~~[[sase#^axe-restart]]~~
    - Restarted axe
  - quick note
- [ ] () — CAPTURE
  - [[bob#^capture-stop]]
  - [[bob#^web-capture]]
- [ ] () — SASE
  - [[sase#^recovery-panel]]
```

`bob.md` becomes as follows. `^capture-stop` is started and gets a new Work Log;
`^web-capture` was deferred, so it stays Next.

```markdown
- [/] #task Add support for `=x` syntax! [created::2026-09-26] ^capture-stop
  - 🛠️ **WORK LOG**
    - _2026-09-28_ — Designed the `=x` grammar
      - chose `x` for done
    - _2026-09-28_ — Wrote the plan
- [*] #task Add capture support for web URLs! [created::2026-09-21] ^web-capture
- [ ] #task Plain ready task [created::2026-09-20] ^ready
```

In `sase.md`, the struck link qualifies for the Work Log, and the new entry is prepended
under the existing marker:

```markdown
    - 🛠️ **WORK LOG**
    	- *2026-09-28* — Restarted axe
    	- *2026-09-27* — Diagnosed the hang
```

**Other rows on the same fixture**

| Item                                      | Result                                                                                                                                                              |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `=x` at `09:49:00`                        | Same, but the range stays `0920-0950 [t:: 30m]`: less than 5 minutes remain                                                                                         |
| `^bob:ready=x`                            | `[ ]` → `[/]`; `[[bob#^ready]]` appended after `quick note`, closed as `🍅 [[bob#^ready]]`, carried second (capture-stop, ready, then web-capture); action `linked` |
| `^sase:recovery-panel=x`                  | The subtree moves from SASE into CAPTURE (action `moved`, source SASE, using the existing move helper's handling of the emptied source); `[ ]` → `[/]`; carried     |
| `^bob:capture-stop=x`                     | Link already in _R_ (`already_current`); identical to plain `=x`                                                                                                    |
| `Draft docs @bob:draft-docs=x`            | New `- [/] #task Draft docs [created::2026-09-28] ^draft-docs` in `bob.md`; link appended in CAPTURE, then closed and carried                                       |
| `-2`, blank line, `=x`                    | The adjust shrinks 0950 to 0940, then the close finds 3m remaining: no further decrement                                                                            |
| `=x`, blank line, `^sase:recovery-panel=` | "Switch tasks": close CAPTURE, then start SASE through the existing start rules                                                                                     |
| `^bob:ready#capture=x`                    | Error: remove `#capture`                                                                                                                                            |
| `=x` run a second time                    | Error "no running Pomodoro to close… next up is CAPTURE at line 13"                                                                                                 |
| `=x more`, or `=x` with a child line      | `invalid_pomodoro_close` error; nothing written                                                                                                                     |

### Output contract

**JSON** (`bob capture --format json`). Schema version 1, strictly additive.

- **A whole-item `=x`** reports:
  - `kind: "pomodoro_close"`, `routed: false`, `route_label: ""`;
  - `relative_target`/`target` = the day file, and `day_file`;
  - `text` = the raw token (`"=x"`), `created: false`, `scheduled: null`;
  - `placement: "closed"`, a new `Placement` variant;
  - `task_line` = the post-image closed entry line;
  - `pomodoro_name` = _R_'s name.
- **Link forms** keep `kind: "pomodoro_link"`, and the body-bearing form keeps
  `kind: "pomodoro_task"`, with every existing key.
  - The status and `task_line` fields report the **final** post-image, for example
    `previous_status_symbol: " "` and `status_symbol: "/"`.
  - `pomodoro_link_destination` is _R_ in the final post-image.
- **Every form with `=x`** adds this object:

```json
"pomodoro_close": {
  "raw": "=x",
  "pomodoro_line": 5,
  "pomodoro_name": "CAPTURE",
  "entry_line": "- [x] (**0920-0940** [t:: 20m]) — CAPTURE",
  "planned": {"start": "0920", "end": "0950", "duration_minutes": 30, "time_range": "0920-0950"},
  "closed":  {"start": "0920", "end": "0940", "duration_minutes": 20, "time_range": "0920-0940"},
  "closed_at": "0937",
  "remaining_minutes": 13,
  "decremented_minutes": 10,
  "tasks": [
    {"role": "worked", "block_link": "[[bob#^capture-stop]]", "ledger_line": 6,
     "resolved": true, "relative_target": "bob.md", "block_id": "capture-stop",
     "text": "Add support for `=x` syntax!",
     "previous_status_symbol": "*", "previous_status_name": "Next",
     "status_symbol": "/", "status_name": "In Progress", "status_changed": true,
     "carried": true,
     "work_log": ["*2026-09-28* — Designed the `=x` grammar", "*2026-09-28* — Wrote the plan"],
     "work_log_created": true, "warning": null},
    {"role": "deferred", "block_link": "[[bob#^web-capture]]", "ledger_line": 10, "...": "..."},
    {"role": "struck", "block_link": "[[sase#^axe-restart]]", "ledger_line": 11, "...": "..."}
  ],
  "carried": [{"kind": "worked", "text": "[[bob#^capture-stop]]"},
              {"kind": "deferred", "text": "[[bob#^web-capture]]"}],
  "notes": ["quick note"],
  "next_pomodoro": {"line": 13, "name": "CAPTURE", "time_range": null, "created": true}
}
```

Field rules:

- **Timing.**
  - `pomodoro_line` is the pre-image line of _R_; `next_pomodoro.line` is post-image.
  - `planned` is the range before the close; `closed` is the range written.
  - `closed_at` is the clock time as `HHMM`.
  - `remaining_minutes` = planned end − `closed_at`, normalized into (−720, 720].
    Positive means the session closed early; negative means it overran.
  - `decremented_minutes` = planned − closed duration, never negative.
- **`tasks`.** One entry per distinct target, in first-appearance order: every
  classified block link in _R_, plus the `subtask` entries closed through embeds.
  - `role` comes from the target's first link line and is one of `worked` (startable),
    `mentioned` (worked-on but not startable), `deferred`, `struck`, `embedded`, or
    `subtask` (closed recursively).
  - `work_log` lists the dated depth-1 entry texts written for that task; nested lines
    are omitted.
  - An unresolved target reports `resolved: false`, null status fields, and a `warning`.
    That warning also goes to the top-level `warnings`.
- **`notes`.** The bodies of _R_'s direct-child non-link bullets, in source order.
- **`next_pomodoro`.** `null` only when nothing was created and no later open entry
  exists.

The top-level `warnings` array also carries the skipped-decrement warning.

**Human output.** Styled with `Styler` (colour on a TTY, plain otherwise). It follows
the adjust and link families: `✓`/`[dry-run] ok`, with verbs `closed`/`would close`.
Required content, in order:

1. A header naming the session, the range change, the file, and the line, e.g.
   `closed CAPTURE 0920-0950 → 0920-0940 (20m, −10m) · 2026/20260928.md line 5`. With no
   decrement, show a single range. Append `(ran 7m over)` when `remaining_minutes < 0`.
2. For link forms, the existing link status and ledger lines.
3. One line per task: a styled status transition (`[*] → [/]`, `[*] deferred`, `[x]`,
   `[x] closed`), the text, `note ^id`, and `+N Work Log` when entries were written.
   Unresolved targets are listed as warnings.
4. `next: CAPTURE (created) at line 13 · carries 2 links`, or `next: SASE at line 14`.

### Editor contract

**`bob capture-parse`**

- A whole-item `=x` reports mode `pomodoro_close`, a `pomodoro_close` span over the
  token, an additive `pomodoro_close: {"raw": "=x"}` spec on the item and at top level,
  and `needs: []`.
- A whole-item `=` reports `incomplete` with no spans, like a lone `+`.
- On link items the `=x` span kind is `pomodoro_close` instead of `pomodoro_start`. The
  mode stays `pomodoro_link` or `pomodoro_task`, and the item carries the spec.
- `invalid_pomodoro_close` is an error diagnostic with a precise range:
  - the extra text or child line for a whole-item near miss;
  - the `#name` component for `#name=x`;
  - the conflicting marker for `s:`/`p:`/`%`.
- `@@` skips `=x` items, as it does adjustments.
- Human output prints `close: =x`.

**`bob capture-complete`**

- It returns an empty success for a whole-item `=x`/`=`, and whenever the cursor is
  inside a `=x` suffix.
- `^route:` and `@route:` completion before the `=` is unchanged, and replacements still
  stop before `#`/`=`.

**`bob capture-rewrite`**

- `pomodoro_close` is non-absorbable, and `=x` items are never rewritten.

### Deliberate choices

1. **Port completion, not the pause.** `=x` ports Obsidian's Pomodoro completion, the
   Ctrl+Enter flow, byte for byte. That flow is what closes a Pomodoro; Ctrl+Shift+Enter
   is the task-level pause. The ledger looks the same whichever tool closed it.
2. **Auto-decrement only.** A session is shortened to the stop time, as the user's
   `^capture-stop` task asks, but never extended. An overrun is reported
   (`ran 7m over`), and `+N` before `=x` in the same draft extends first.
3. **`=` is the session operator.** `=<X>` starts, `+N`/`-N` adjust, and `=x` closes,
   with `x` as in a done checkbox. `=x` works wherever `=<X>` works. `#name` is rejected
   because the running session is the only valid target.
4. **Link forms target _R_, never the queue.** "Log this task in the session I'm
   closing" must put the link where the work happened. A queued link moves rather than
   duplicating, because task-status-hooks would delete a duplicate anyway.
5. **A new kind for the solo form.** It is `pomodoro_close`, not an overloaded kind, so
   older clients degrade to a neutral preview. Link and task forms stay additive, with
   the `pomodoro_close` object on their existing kinds.
6. **Safe writes.** Unresolved or ambiguous targets warn instead of guessing. Vault-wide
   dependency recovery stays in task-status-hooks.

**Non-goals**

- Changing Obsidian plugins, task-status-hooks behavior, Ensure Next, `!`, or project
  notes.
- A standalone start (`=3` with no task).
- Auto-extending overruns.
- New CLI subcommands or options.

## Phase: close-ledger

Add `src/native/capture_pomodoro_close.rs` and register it in `src/native.rs`. It is a
pure module over strings, with no IO, containing:

- **`find_running_pomodoro(contents)`**, which returns _R_ or a typed error: no section,
  no open timed entry (carrying the next open placeholder's name and line), or multiple
  open timed entries (carrying their names and lines). The diagnostics above are built
  from these.
- **`close_timing(range, now)`**, the auto-decrement math with checked arithmetic.
  - Reuse `parse_adjustment_range`, `adjustment_duration_minutes`,
    `format_adjusted_range`, and `normalize_minutes` from `capture.rs`: raise their
    visibility to `pub(crate)`, or move them into a small shared helper. Do not
    duplicate them.
  - The `+N`/`-N` tests must pass unchanged.
- **`plan_ledger_close(contents, entry, now)`**, which returns:
  - the new ledger contents;
  - the timing summary;
  - the classified links, each with its pre-image line, role, raw target, block ID, and
    `carried`;
  - the carried lines;
  - the notes;
  - the next-Pomodoro endpoint;
  - the startable and embedded target lists;
  - the Work Log note groups, as trees from the snapshot;
  - _R_'s sub-bullet range, which close-capture's link step appends into.
- **`sub_bullet_range(lines, entry_index)`**, exposed separately for the same reason.

Implement steps 1–6 of the close transaction exactly, including the classification
table, the lookalikes, the marker policy, carry order, placeholder creation, the stub,
and the quirks. Preserve CRLF and a missing final newline. Reuse the existing helpers
(`markdown::fenced_lines`, `capture_pomodoros::parse_name_tail`/`scan`, and the
line-span utilities) rather than writing new ones.

**Unit tests** must assert, not comment:

- The worked example's ledger post-image, byte for byte.
- The 09:49 no-decrement case, a start-in-the-future clamp to 0m, and a
  midnight-crossing range.
- An unnamed _R_ with no children and no later entry, which creates `- [ ] ()` plus
  `\t- `.
- Nothing carried with a later entry present, which creates nothing.
- Every lookalike in the deferred table.
- Struck links with and without 🍅, `🍅🍅` collapse, and an embedded marker drop.
- Nested worked-on links carried at their own indent.
- Fenced lines untouched.
- A range cut at a blank line.
- A deferred line's orphaned children.
- CRLF.
- No final newline.
- Multiple open timed entries, and no open timed entry with a next placeholder named.

Run `cargo clippy --all-targets --all-features` and `cargo test`. Format only the code
you touched.

## Phase: close-tasks

**Shared link resolver.** Extract task-status-hooks' private `NoteIndex` (with
`markdown_basename` and `target_to_markdown_path`) into a shared `pub(crate)` module
such as `src/native/vault_links.rs`. task-status-hooks must use it with **no behavior
change**, and its existing tests must pass.

- Resolution rules: an exact vault-relative `<target>.md` first; else, only for targets
  without `/`, a unique case-insensitive basename. `[[#^id]]` means the day file itself.
  Aliases (`|…`) and `<…>`/URL-encoded targets are unwrapped the way task-status-hooks
  does.
- Build the index lazily. The common case, an exact root-level `route.md`, must not walk
  the vault. A walk skips hidden directories.

**`CloseVault`.** A small trait over "resolve this target from the day file" and "read
the latest staged bytes of this path". close-capture implements it over
`CaptureBatchPlanner`; the tests use in-memory maps.

**Task effects** (steps 7–9):

- Locate tasks with `note_tasks::read_settings`/`scan`/`by_block_id`. `NotATask`,
  `Duplicate`, and `Missing` give a warning and a skip.
- Starting a task (`[ ]`/`[*]` → `[/]`) applies the spacing normalization. A
  `set_task_line_status`-style helper can be generalized to `pub(crate)`.
- The embedded close is recursive, with a `[completion:: date]` field and the depth and
  target caps.
- Retirement edits only _R_'s range.

**Work Log writer.** Add `src/native/capture_work_log.rs`. Given a note's contents, a
task line, the note trees, and the date _D_, it returns the new contents and the dated
entry texts written. It implements the qualifying rules, the tree rendering, the
indent-unit rule, marker detection by **Work** kind, prepending under an existing marker
versus appending a new marker after the child block (after the SCHEDULE LOG),
same-task-twice ordering, CRLF, and final-newline preservation. It must be reusable by
future Work Log features.

**Entry point.** Expose `plan_pomodoro_close(day_path, day_contents, now, vault)`. It
runs close-ledger, then the task effects, then the Work Logs, then retirement, and
returns:

- the set of changed files and their new contents, where the day file may also be a task
  note;
- the `PomodoroCloseSummary` data needed for the JSON object;
- the warnings.

It writes nothing.

**Unit tests** must assert:

- The worked example's `bob.md` and `sase.md` post-images, byte for byte.
- A task linked twice (source order in its Work Log).
- An existing marker with and without entries; space-indented notes (the indent-unit
  rule).
- A Work Log appended after a SCHEDULE LOG, and not placed under it.
- Recursive embedded closure with the caps and a completion field placed before `^id`,
  plus embeds retired to `~~[[…]]~~`.
- Blocked, Done, and In Progress bare targets left unchanged.
- Unresolved, ambiguous-basename, duplicate-ID, and non-task targets warned and skipped.
- A task note that is the day file itself.
- The resolver's exact versus basename behavior, and the lazy-walk behavior.
- CRLF.

Run `cargo clippy --all-targets --all-features` and `cargo test`.

## Phase: close-capture

**Grammar** (`src/native/capture_language.rs`)

- Add `CaptureKind::PomodoroClose { spec }` and an item-level pre-parse next to
  `parse_pomodoro_adjust_item`. It implements whole-item `=x`, the incomplete `=`, the
  near misses, and the prose lookalikes.
- Change `parse_colon_link_tail` to return a session-suffix enum, `Start(spec)` or
  `Close(spec)`. Thread it through `CaptureKind::Pomodoro` and `PomodoroLink` in place
  of the bare `start` option. Every existing `=<X>` behavior and message must be
  unchanged.
- Reject `#name=x`, `=x` with `s:`/`p:`, project-note forms with `=x`, and forced flags,
  with the messages above.
- Make `@@` inheritance skip `PomodoroClose`.
- Update every exhaustive match, including `capture_kind_label` ("Pomodoro close"),
  `plan_capture_item`'s arms, `classify_local_marker`, `non_absorbable_marker_notice`,
  and the parity test `editor_agrees_with_execution_for_resolved_captures`.
- The editor side only has to compile and must not report `=x` as a start. The final
  mode, spans, and diagnostics belong to close-editor-contract.

**Transaction** (`src/native/capture.rs`)

- `plan_pomodoro_close_item` handles the whole-item form: resolve the day file through
  the planner, apply the diagnostics, call `plan_pomodoro_close` with a `CloseVault`
  over `CaptureBatchPlanner`, and stage only the changed files.
- The solo link with a close suffix reuses `plan_pomodoro_link_capture`'s lookup, status
  gate, `plan_task_link`, and `dependsOn` warning, and then runs the link step into _R_
  as described in "Link forms". `find_movable_task_links` and `move_subtree_to_entry`
  place the link at the end of _R_'s sub-bullet range. The close runs after that.
- The body-bearing path forces the link destination to _R_, erroring when there is none,
  and then closes.
- Batches compose through the planner. A later item sees earlier staged edits, and a
  late failure rolls everything back.

**Guard text.** Append the `=x` pointer to the start guards, as described in "Selecting
_R_, and diagnostics".

**Output**

- Add `Placement::Closed`, the `pomodoro_close` field on `CaptureItemResult` at every
  construction site, and `PomodoroCloseSummary` (plus its nested structs), serialized
  exactly as in the output contract.
- Add `print_human_pomodoro_close_success`, and wire it into `print_human_item_success`
  for all three forms. Reuse `style_task_status_marker`.

**Integration tests** (`tests/cli.rs`) go in a new group modelled on the solo-link
group, with a `close_vault()` fixture that is the worked example. Every case must be
asserted; none may be left as a comment or with `|| true`.

- Every worked-example row: JSON and file post-images.
- Human output, plain and dry-run.
- Every diagnostic: missing day file, missing section, none running (with and without a
  next placeholder), multiple timed entries, the unparseable-range warning path, near
  misses, `#name=x`, `s:`/`p:`, forced flags, project-note `=x`, `@@` not applying,
  `=3`/`=xx`/`Plan =x` staying prose, and `=X` accepted.
- Link actions `linked`, `moved`, and `already_current`; Ready, Blocked, Next, and In
  Progress transitions; a Done task rejected; the missing-ID hint.
- Batches: `-2` then `=x`; `=x` then `^…=` (switch tasks); a second `=x` failing and
  rolling back the whole batch.
- CRLF and no final newline in both the day file and a task note.
- `--dry-run` writing nothing while printing JSON identical to a real run.
- The existing start and adjust tests unchanged, apart from the added guard hint.

Run `cargo clippy --all-targets --all-features` and `cargo test`. Format only what you
touched; repo-wide rustfmt drift is not this epic's concern.

## Phase: close-editor-contract

**`capture-parse`** (`capture_language.rs`, `capture_parse.rs`)

- Add `EditorMode::PomodoroClose` (`pomodoro_close`), `SpanKind::PomodoroClose`
  (`pomodoro_close`), the additive `pomodoro_close` spec on `CaptureParseResult` and
  `CaptureParseItem`, the incomplete `=`, and `invalid_pomodoro_close` diagnostics with
  precise ranges.
- The `=x` suffix spans are `pomodoro_close` on `@`/`^` links and body-bearing items.
- Pin these inputs with unit and protocol tests: `=x`, `=X`, `=`, `=x more`, `=x` with a
  child, `@r:id=x`, `^r:id=x`, `^r:id#n=x`, `Text @r:id=x`, `Text @r:id=x s:2`, `=3`,
  `Plan =x`, and a multi-item draft mixing `+5`, `=x`, and a task.
- Extend `editor_agrees_with_execution_for_resolved_captures`.

**`capture-complete` and `capture-rewrite`.** Give them the behavior described in the
editor contract, with tests: an empty success inside `=x`, `^`/`@` completion before `=`
unchanged, and `=x` never absorbed or rewritten.

**Help** (`build_cli` long_about and after_help; keep option lists sorted)

- `bob capture --help` gets a concise `=x` paragraph and examples: `bob capture =x`,
  `bob capture '^bob:capture-stop=x'`, and `printf -- '-2\n\n=x\n' | bob capture`.
- `bob capture-parse --help` lists the mode, the span, and the diagnostic.
- `bob capture-complete --help` notes that `=x` is never a completion field.

**Docs**

- **`docs/capture.md`:**
  - Contents; grammar table rows; the `#`/`=` disambiguation rows.
  - A new "Closing the running Pomodoro" section after "Adjusting the current Pomodoro",
    covering:
    - the keymap note (Ctrl+Enter completion, not Ctrl+Shift+Enter);
    - auto-decrement;
    - the classification and carry rules;
    - Work Log rules;
    - link forms;
    - the worked example;
    - diagnostics;
    - JSON and human output;
    - the "switch tasks" batch idiom.
  - Fix the "never … completes" sentence.
  - The kind list (`pomodoro_close`), the capture-parse modes, spans, and diagnostics,
    the capture-complete notes, and Interactive editor markers.
- **`docs/task-status-hooks.md`:** `=x` leaves vault-wide Blocked recovery and
  completed-reference retirement to task-status-hooks.
- **`README.md`:** the capture summary, grammar rows, and examples.

Run `cargo clippy --all-targets --all-features` and `cargo test`.

## Phase: mac-close-preview

**Setup.** Open the repo with `/sase_repo`: `sase repo open bob-mac-capture`, or
`sase repo open gh:bobs-org/bob-mac-capture` when the linked checkout is unavailable.
Read its `AGENTS.md` if present. Keep Swift free of grammar, clock, and ledger logic:
everything comes from Bob's JSON. The live preview is already a real
`capture --dry-run --no-clip --format json` of the draft, so the close card shows
exactly what Return will do.

**Decode** (`Sources/CaptureCore/CaptureModels.swift`)

- `PomodoroCloseSpec { raw }` on `CaptureParseResponse` and `CaptureParseItem`.
- `PomodoroCloseSummary` with nested `timing`, `task`, `carried`, and `next` structs,
  and `CaptureCommandSuccess.pomodoroClose`.
- Everything is `decodeIfPresent`, with arrays `?? []`. Older Bob output must still
  decode, and unknown `role`/`kind` strings must degrade to a neutral row.
- `placement: "closed"` stays a plain string.

**Presentation** (new `Sources/CaptureCore/CapturePomodoroClosePresentation.swift`) is
pure and unit-tested:

- `title`: "Close CAPTURE", or "Close session" when unnamed.
- `sessionText`: "0920-0950 → 0920-0940 · 20m", or "0920-0950 · 30m" when not
  decremented.
- `timingText` and `timingTone`: "13m early" (early), "on time", or "7m over" (over).
- `destinationText`: "2026/20260928.md · line 5".
- `taskRows`:
  - a glyph kind per role;
  - a transition such as `[*] → [/]`, `[*] deferred`, `[x] closed`, or `[x]`;
  - the task text;
  - the locator `bob.md · ^capture-stop`;
  - the Work Log count and up to two entry previews with the date prefix stripped;
  - an unresolved warning.
- `nextText`: "Next: CAPTURE · new · carries 2 links", or "Next: SASE · line 14".
- `emptyText`: "No Task Links — the session simply closes".
- `statusText`: "Would close CAPTURE · 1 started · 3 Work Log entries" or "Closed …".
- `primaryActionTitle`: "Close".
- `notificationTitle` ("Closed CAPTURE") and `notificationBody` (session and timing
  line, tasks and Work Log line, next line).
- `batchSuffix`: " (closed CAPTURE)".
- `accessibilitySummary`.

**The card** (`Sources/BobMacCapture/CapturePanelView.swift`). Add a dedicated
`closePreviewItem`, chosen in `previewItem` whenever `pomodoroClose` is present, ahead
of the link, toggle, and standard renderers. Use the existing visual language:

- **Header:**
  - SF Symbol `stop.circle.fill`, tinted with the Pomodoro-session palette colour;
  - a bold title;
  - a trailing monospaced session text, with the decrement in the session tint;
  - underneath, the destination in secondary, plus a capsule timing chip: orange when
    early, green when on time, secondary when over.
- **Rows**, up to 6 and then "+N more":
  - a leading glyph:

    | Role                | SF Symbol                            |
    | ------------------- | ------------------------------------ |
    | worked              | the monospaced transition text       |
    | deferred            | `arrow.uturn.forward`                |
    | struck              | `checkmark.circle`                   |
    | embedded or subtask | `checkmark.circle.fill`              |
    | unresolved          | `exclamationmark.triangle` in yellow |

  - the task text, with struck rows shown with strikethrough;
  - a trailing secondary locator;
  - Work Log previews underneath in secondary callout style, prefixed with
    `square.and.pencil`, one line each, truncated at the tail.
  - `notes` appear as one dimmed `text.alignleft` row, "1 note stays".

- **Footer row:** `arrow.turn.down.right` with `nextText`.
- **Link and new-task forms:** prepend a "via" row. Use `link` for "Linked bob.md ·
  ^ready into CAPTURE" or "Moved from SASE", and `plus.circle` for "New task Draft docs
  → bob.md · ^draft-docs". The matching task row gets a subtle "linked"/"new" tag.
- It must look right in light and dark mode, and at narrow panel widths, where the
  locator truncates first.
- Add the card's accessibility label to `previewAccessibilityLabel`.

**Model and footer** (`CapturePanelModel.swift`)

- `primaryActionTitle` is "Close" whenever the preview has a `pomodoroClose`.
- The live-preview, preview, and submit `statusText` come from the presentation.
- `captureWroteDayFile` is true.
- Keep the `pomodoro_close` span **out of** completion gating.
- Fix two staleness gaps that would make a close preview lie:
  - clear `previewResult`/`previewResults` when a live dry run fails, so a stale card
    never sits next to the red error;
  - re-run analysis when the panel is re-presented with a retained non-empty draft, so
    `closed_at`/timing are fresh.

**Palette and notifications**

- Map the `pomodoro_close` span kind to the existing Pomodoro-session category in
  `CompletionRowContent.swift`, the same pink family as `pomodoro_start` and
  `pomodoro_adjust`.
- In `NotificationService.swift`:
  - add a close branch for single captures, which also covers the link and task forms;
  - add the batch suffix;
  - map `friendlyKindLabel` `pomodoro_close` to "Close";
  - make the day file the Open Note target.

**Fixtures from real Bob.** Build bob-cli at or after the landed close-capture commit
(`cargo build`; the globally installed `bob` may be stale). Generate the fake-bob JSON
by running `target/debug/bob capture --dry-run --format json` and `capture-parse` on the
worked-example vault. Cases:

- `=x` (dry run and submit);
- `=` (incomplete parse, dry-run failure);
- the "no running Pomodoro" failure;
- `^bob:ready=x` and `^sase:recovery-panel=x` (moved);
- `Draft docs @bob:draft-docs=x`;
- `^bob:ready#capture=x` (diagnostic);
- the mixed `-2`/`=x` batch.

Paste real outputs, adjusting only `dry_run` interpolation and paths.

**Tests**

- Model decoding, including older Bob.
- `CapturePomodoroClosePresentationTests`: every role, the timing tones, empty,
  decrement versus none, unnamed, and truncation.
- `CompletionRowContentTests` span mapping.
- `CapturePanelModelTests`: preview and submit argv, footer "Close", status text, the
  stale-results fix, and re-presentation refresh.
- `NotificationServiceTests`: single, batch, link-close, and task-close. Keep helper
  argument order, with `relativeTarget` last.

**README.** Update the runtime contract, preview, grammar, and notifications sections.

**Verification.** Run the CaptureCore tests where possible. The app target needs macOS,
so push and confirm that the `macOS 26 SwiftPM` GitHub Actions run passes format lint,
build, and the full `swift test`. Report the run ID, and fix any failure before
finishing.

## Acceptance

- Every worked-example row behaves as specified from the CLI. Bob Mac Capture's preview
  of `=x` on that fixture shows the session, the decrement, the timing chip, three task
  rows with transitions and Work Log previews, and the new CAPTURE placeholder. Submit
  writes exactly what the preview showed.
- Closing through `=x` leaves the ledger and task notes byte-identical to Obsidian's
  Ctrl+Enter completion on the same fixture, except for the documented auto-decrement
  and deliberate divergences.
- Typing `=x` with nothing running gives an actionable message that names the next
  placeholder, both in the CLI and in the Mac preview.
- `capture-parse` and `capture` agree on every `=x`, near-miss, and prose input.
- Failed items write nothing, and batches stay atomic.
- `cargo clippy`, `cargo test`, and macOS CI pass on the landed revisions.
