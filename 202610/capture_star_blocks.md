---
tier: epic
title: Capture `*` makes the new task block an existing task
goal: '`bob capture` and Bob Mac Capture accept `*note:block-id`, which makes the
  captured (or explicitly selected) task a prerequisite of an existing open task.
  The edge is written to the blocked task''s Depends-On line. That task''s live Task
  Links under today''s open Pomodoros are consumed by identity, with any notes under
  them kept. A Pomodoro entry is deleted only when this capture emptied it, and every
  removal is visible in the preview before submit.

  '
decisions:
  decision_record:
    ask: Record the accepted capture `*` design as a new strand in the decisions memory
      web?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
phases:
- id: contract
  title: Docs contract, grammar, wire names, and vectors
  depends_on: []
  size: medium
  description: 'contract: write the `*` grammar, consumption amendments (PDL8–PDL11),
    wire fields, human output, and ST/STP vectors into docs/capture.md, docs/task-dependencies.md,
    docs/completion.md, docs/freshness.md, and README.md before any code.'
- id: consume
  title: Shared capture consumption by identity, keeping notes, and reported
  depends_on:
  - contract
  size: medium
  description: 'consume: make every capture consumption path match Task Links by identity,
    keep authored notes under consumed link bullets, delete only entries this capture
    emptied, and report consumed_links[] plus removed pomodoro_blocks; resolves bob-cli-7l.'
- id: parse
  title: Two-direction lexer, Markdown guards, and capture-parse wire
  depends_on:
  - contract
  size: medium
  description: 'parse: generalize the `&` scanner to a Blocks direction with the `*`
    Markdown guards, carry blocked_tasks[] through the execution and editor models,
    add spans and the blocked_task need, mirror every `&` rejection, and fix the leading-run
    bug bob-cli-7m.'
- id: writer
  title: Path-addressed existing-dependent writer with star guards
  depends_on:
  - consume
  size: medium
  description: 'writer: address the existing-dependent writer by vault path plus block
    ID, add the `*` target resolver and A8 guards, the Next cap option, always-mint
    for a new blocker, and relocate-by-ID; `&` stays byte-identical.'
- id: complete
  title: Blocked-task completion pool and capture-task-id replacement
  depends_on:
  - parse
  size: medium
  description: 'complete: add the blocked_task completion context with an open-editable
    pool ranked today-linked first, already_blocked and disabled_reason rows, shell
    completion, and capture-task-id blocked_task_replacement.'
- id: execute
  title: Execute `*` captures and report them
  depends_on:
  - parse
  - writer
  size: medium
  description: 'execute: wire parsed blocked_tasks into new-task, sub-bullet, dependency-only,
    and forced --task captures in the link-then-consume order, and emit dependency_updates[],
    minted_blocker_block_id, and the human lines, with ST vector tests.'
- id: mac
  title: Bob Mac Capture direction-aware picker and blocks preview
  depends_on:
  - complete
  - execute
  size: medium
  description: 'mac: parameterize the dependency picker by direction for `*`, add
    span kinds, Backspace and Add-ID support, and render dependency_updates[], consumed_links[],
    and deleted sessions from real-bob fixtures.'
- id: publish
  title: Decision record and final docs sweep
  depends_on:
  - mac
  size: small
  description: 'publish: write the capture `*` decision record citing the epic''s
    commits, republish memory, and reconcile the docs'' JSON examples with real output
    from the final binary.'
proposed_by: bbugyi200.athena.research.4g.linker.w1
decided_by: auto
create_time: 2026-10-14 11:27:18
status: wip
bead_id: bob-cli-7n
---

- **PROMPT:** [prompts/202610/capture_star_blocks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_star_blocks.md)
- **BEAD:** [bob-cli-7n](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7n/README.md)

# Plan: Capture `*` makes the new task block an existing task

## Context

Bryan asked for a new `bob capture` / Bob Mac Capture syntax that works like `&`, but in
the opposite direction: the new task blocks the named task instead of depending on it.
When the blocked task has a Task Link in today's daily file, that link must be removed,
and so must its Pomodoro if the entry is left empty.

The research report
`research:202610/capture_star_blocks_modifier/capture_star_blocks_modifier.md` covers
this request. Bryan agreed with **all** of its requirements, A1–A10. He also took its
recommended answer to each of its four open questions:

1. A new blocker's lane is capped at Next `[*]`.
2. Capture keeps the authored notes under a consumed link. This amends PDL7 for capture
   only.
3. The sigil is `*`.
4. Existing owners (`*T @route+id`) are in v1.

None of these is reopened here.

Facts the plan builds on (bob-cli at `b4edb9b`):

- **Parser.** `src/native/capture_language/dependencies.rs` already has a sigil-generic
  scanner.
  - `scan_modifier_with_sigil` is shared by `&` and `!`.
  - `extract_line_dependencies` handles the leading and trailing runs.
  - The leading run breaks at the first non-modifier token and never skips
    `is_trailing_transparent` shapes. That is bug **bob-cli-7m**:
    `@cash &foo:bar Buy milk` captures literal text.
- **No `*` grammar exists yet.** Today no `*`-led token outside `=` tokens is special.
  `*foo:bar Buy milk`, `* Buy milk`, and a lone `*` all parse as plain task text.
- **Existing-dependent writer.** `plan_existing_task_dependencies`
  (`src/native/capture/dependencies.rs`) already does nearly everything `*` needs, with
  the roles swapped:
  - merges the managed Depends-On line, `[dependsOn::]`, and `[id::]`;
  - flips the dependent to `[?]` and stamps it fresh;
  - flags `^prj` residents;
  - transfers commitment and stages the BLOCKS mirror;
  - checks for cycles and runs its guards.

  It addresses the dependent as `format!("{route}.md")`. Routes are ASCII-lowercased,
  and `@route` cannot contain `/`.

- **Consumption.** `consume_live_links` (`src/native/capture/project_deps.rs`) removes
  the literal text `[[route#^id]]` through `capture_task_toggle::plan_link_removal`.
  - A link-only bullet goes with its whole subtree (PDL7).
  - It then deletes **every** childless entry (`plan_empty_pomodoro_removals`).
  - It reports nothing structured: only the warning
    `removed N live Pomodoro link(s) to newly Blocked task(s)`.
  - Its `PomodoroBlockRef::removed` refs never show the deleted entry in
    `pomodoro_blocks`.

  That last point is bug **bob-cli-7l**. Reconcile instead matches by identity
  (`task_status_hooks::apply_blocked_consumption` with resolved references), and
  `capture::pomodoro_close::SnapshotCloseVault` is a staged-aware resolver that capture
  can use.

- **New tasks.** A new task gets a minted block ID only for the BLOCKS mirror
  (`mint_staged_dependent_id`). `locate_new_task_ref` finds the new task by exact line
  text, which goes stale after later edits to that line.
- **Bob Mac Capture hard-codes `&`.**
  - `DependencyPickerPresentation.swift` uses `leadingSigil: "&"` and
    `insertionPrefix: "&"`.
  - `CapturePanelModel.removePickerTrigger` checks byte 38.
  - The Add-ID flow reads only `dependency_replacement`.
  - A line-initial lone `*` on line 2 or later is a bullet placeholder
    (`CapturePanelController.isAuthoredBulletRow`).

Governing records: `task-deps-are-depends-on-links`, `capture-link-drops-blocking-deps`,
`projects-block-from-prj-line`, `subprojects-inherit-project-blocking`,
`task-lanes-are-sticky`, `unblocked-tasks-are-offered`, and
`mac-capture-is-a-thin-client`. Read each with
`sase memory read decisions:<slug> -r "<why>"` before changing behavior it governs.

Beads:

- **Absorbed:** bob-cli-7l (phase `consume` closes it) and bob-cli-7m (phase `parse`
  closes it).
- **Related, left open:**
  - bob-cli-73: Mac link_guard rendering.
  - bob-cli-3p: the reverse Blocks… stage in Obsidian. It can share the new §5 bullet.
  - bob-cli-2u: unreported toggle moves.

## Design

### 1. Meaning

Let N be the item's task (new, or an existing owner) and T a task named with `*`. Arrows
mean "depends on".

| Draft                          | Stored edge                          | Line that changes       |
| ------------------------------ | ------------------------------------ | ----------------------- |
| `N &P` (today)                 | N → P                                | N's Depends-On          |
| `N *T`                         | T → N                                | T's Depends-On          |
| `N &P *T`                      | T → N → P                            | both ("insert between") |
| `*T @route+id`, `@route:id *T` | T → `route#^id`                      | T's Depends-On          |
| `Note text @route+id *T`       | T → `route#^id`, plus the sub-bullet | T's Depends-On          |

The edge lives only on T's `⛓️ **DEPENDS ON:**` line, and N's `🧱 **BLOCKS:**` line is
the usual generated mirror. No new data model, field, or tag is added.

`*` is the dual of `&`. The two share ownership rules (`resolve_execution_ownership`,
`@@` inheritance, forced `--task`/`--route`), child-line trailing runs, and rejections.

### 2. Grammar contract

`*` is a directive only when **all** of these hold:

1. **Placement.** It starts a whitespace-delimited token in the parent line's leading
   run or in any line's trailing run, exactly where `&` is allowed. A mid-line
   `*foo:bar` stays body text. Both runs skip the `is_trailing_transparent` marker
   shapes (`@…`, `#…`, `s:`, `p:`, schedule, priority, and clip tokens). Adding that
   skip to the leading run fixes bob-cli-7m for `&` too.
2. **Next byte.** The byte after `*` is neither whitespace nor `*`. This keeps
   `**bold**`, `* bullet`, and `5 * 3` literal.
3. **Rest of the token.** The rest of the token contains no `*`, which keeps `*urgent*`
   literal. `validate_note` rejects `*` in the note component for both sigils, since
   Obsidian file names cannot contain it.
4. **Bare `*`.** A bare `*` is a directive (it opens the picker) only as the **last
   token of a line that has other text before it**. A line-initial `*` is always literal
   or a list marker. This differs from `&`, whose bare form also works as the first
   parent token.
5. **Escape.** `\*` escapes it, but only where an unescaped token would have been a
   directive under rules 1–3. Elsewhere `\*` stays verbatim Markdown.

Rejections:

- `=x*…`, `=*`, and `==*` stay park operators; they live inside `=` tokens.
- Session operators, `+N`/`-N`, section bullets, project-note construction, `!`
  completions, `--task-ref` pickers, and suffixed owners reject `*` exactly as they
  reject `&`.
- `@route:id=x… *T` is also rejected: closing and blocking in one item is incoherent.
- Error text names the sigil the user typed.

| Draft                                                                                 | Reading                                                           |
| ------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `Ask Kelly *job:cleanup`                                                              | New inbox task N; `job#^cleanup` depends on N                     |
| `*job:cleanup Ask Kelly`, `@cash *job:cleanup Ask Kelly`                              | Same (leading run, past `@cash`)                                  |
| `Ask Kelly @mac_inbox:ask-kelly *job:cleanup`                                         | Same, and N is linked first so it takes T's place in the session  |
| `Ask Kelly *job:cleanup *job:send-batch`                                              | N blocks both, in typed order                                     |
| `Call registrar &school:transcript *job:apply-x`                                      | Insert between; cycle-checked                                     |
| `*job:cleanup @bob+ask-kelly`, `@bob:ask-kelly *job:cleanup`                          | Existing task blocks T (mode `task_dependency`)                   |
| `Order parts *projects/house/kitchen:cabinets`, `X *"Release Plan":publish`           | Nested and quoted targets                                         |
| `*job:cleanup` with no other text                                                     | Incomplete: needs `dependency_target`                             |
| `X *`, `X *jo`, `X *job:`                                                             | Incomplete: needs `blocked_task` (picker open)                    |
| `Fix *urgent* bug`, `Read **Dune**`, `* Buy milk`, `5 * 3`, `R*D`, `Fix *foo:bar now` | Literal                                                           |
| `X \*job:cleanup`                                                                     | Literal `*job:cleanup`                                            |
| `Support *nix`                                                                        | Partial; submit fails with a teaching error that suggests `\*nix` |
| `N *T &T`                                                                             | Direct cycle; refused, vault byte-identical                       |

On the shell, `*` must be quoted (`bob capture 'Ask Kelly *job:cleanup'`), because zsh
globs it. The docs say so next to the existing `&` note.

### 3. Capture consumption (one path for `&`, `*`, and the link guards)

Every capture path that consumes live links changes in the same way. These are
`plan_existing_task_dependencies`, `plan_resident_block` (`^prj` residents), and the
link-guard `Consume` branches in `pomodoro_link.rs`, `task_toggle.rs`, `ensure_next.rs`,
and `pomodoro_close.rs`.

- **Scope (A1).** Only live (unstruck) Task Links under **today's open** Pomodoros
  (running and planned), every occurrence. Closed entries, `🍅` history, other days,
  prose outside `## Pomodoros`, and dependency lines are never touched. A missing day
  file means there is nothing to consume.
- **Identity (A2).** Match by resolved `(vault-relative path, block id)`, not by text.
  Aliases (`|x`), `[[#^local]]`, basename, path, and case spellings all match. Resolve
  through `SnapshotCloseVault::from_planner` (staged-aware); `task_complete.rs`
  `compute_link_statuses` shows the pattern. `consume_live_links` takes a set of
  identities instead of `(route, block_ids)`.
- **Notes kept (A3, new PDL8).** A consumed **link-only bullet with children** is not
  deleted. Its link token is replaced by the task's clean display text: the same `text`
  capture reports, with checkbox, tags, inline fields, and `^id` stripped, and any
  `[[…#^…]]` inside it flattened to plain text. The marker, indentation, and children
  stay unchanged, and the bullet carries no Task Link. A childless link-only bullet is
  deleted. A prose bullet loses only the token (PDL6, unchanged).
- **Only entries this capture emptied (A4, new PDL9).** "Pomodoro" means the column-0
  entry, never the `## Pomodoros` heading. An entry is deleted only if it had a child
  list item before this consumption and has none after it, judged after all of the
  item's edits. That includes a running timed entry. Entries that were already childless
  stay.
- **Reported (A4, new PDL11).** Each consumed link becomes a `consumed_links[]` row.
  Each deleted entry appears in `pomodoro_blocks` with `removed: true`. This covers
  every `PomodoroBlockRef::removed` producer, not just consumption. Human output names
  the session.
- **Reconcile and plugins are unchanged (PDL7 stays theirs).** Implement capture's
  behavior as an options struct on the shared planner, for example
  `apply_blocked_consumption_with(.., ConsumptionOptions { keep_notes, only_emptied })`.
  Reconcile keeps passing the current behavior. Parity is a follow-up (§8).

### 4. Executing `*`

The whole draft plans before anything is written. For an item with `blocked_tasks`:

1. **Resolve and check every target (A8).**
   - Dedupe by resolved identity.
   - The note resolves vault-wide exactly like `&note`: path or basename, nested,
     quoted, plus staged notes.
   - T must be an open, editable task. Refuse targets in:
     - daily notes;
     - `done/`;
     - `_templates`, `_generated`, `_conflicts`;
     - dot-directories;
     - fenced code.
   - Also refuse: a self target; a T with several or malformed Depends-On lines; and a
     `^prj` T when N lives in that project's note or a descendant project (a guaranteed
     deadlock).
   - A repeat is a no-op reported as `already_present`.
2. **Give N a block ID (A9).**
   - Use the typed `^id` or `:id`. Otherwise mint one with this batch's reservations,
     even when `dependencies.blocks_lines: false`.
   - When the ID was minted, report it as `minted_blocker_block_id`. Never reuse
     `minted_dependent_block_id`, which means the opposite.
3. **Plan N as today.**
   - N's own `&` prerequisites come first, so N may be born `[?]`.
   - Insert N and stage its Task Link if `:id`/`#pomodoro` asked for one and the
     existing link guard allows it. This is the "link N, then consume T" order (A5):
     `N @mac_inbox:id *T` keeps the session alive.
4. **Run one writer pass per T, in typed order.**
   - Each pass is the existing-dependent writer with dependent = T (path plus block ID)
     and prerequisites = [N]. It merges the line and fields, stamps T fresh, flips T to
     `[?]`, blocks and tags `^prj` residents, consumes links (§3), and stages N's BLOCKS
     line.
   - **Commitment transfer (A6).** An open N rises to T's pre-capture lane (for a `^prj`
     T, the strongest resident lane), **capped at Next `[*]` when N is new in this
     capture**. The cap applies to the whole promotion walk that this pass seeds. An
     existing owner gets the normal §5 transfer and may become `[/]`. A `^prj` N is
     never raised.
   - Re-locate N by block ID after every pass; never by exact line text.
5. **Existing owners (A7)** go through the dependency-only path beside
   `plan_dependency_only_item`. The sub-bullet path and forced `--task`/`--route` mirror
   `&`.
6. **Cycles.** Direct, transitive, cross-item, and `&`+`*` cycles are refused by the
   writer's `reject_new_cycles`, which sees N's staged edges. The vault stays
   byte-identical.
7. **Later link gestures.** A later `@T:id` in the same batch drops N from T's line
   under `decisions:capture-link-drops-blocking-deps`. That is intended, and the final
   preview must report it accurately.
8. **Commit** through the existing guarded writer, with its preimage checks and
   rollback. Do not promise cross-process or crash atomicity.

### 5. Wire contract

Every change is additive. All four `schema_version`s stay 1, and no subcommand or option
is added (`cli_rules.md`).

- **`capture-parse`.**
  - New span kinds `blocked_task_sigil`, `blocked_task_note`, and
    `blocked_task_block_id`.
  - A top-level and per-item `blocked_tasks[{raw, note, block_id, quoted, range}]`, the
    same shape as `dependencies[]`.
  - A new need `blocked_task` for a partial `*`.
  - Reused: need `dependency_target` (no owner) and mode `task_dependency` (existing
    owner only).
  - Never put a direction flag on `dependencies[]`.
- **`capture-complete`.**
  - Context `blocked_task`, with the same `query`, `owner`, and replacement-range
    semantics as `task_dependency`.
  - Rows reuse the `DependencyCandidate` shape, plus `already_blocked` (omitted when
    false) and `disabled_reason`:
    - `the owner task itself`;
    - `would create a dependency cycle`;
    - `a project cannot depend on its own resident`;
    - the existing duplicate-block-ID reason.
  - The pool is open, editable tasks only (A8).
  - With an empty query, ranking is:
    1. today's linked tasks (the running entry's first);
    2. Pending `[/]`;
    3. Next `[*]`;
    4. the owner's note;
    5. everything else.

    With a non-empty query, ranking is match score first, then the same order.

  - Replacements use `*` (`replacement_for_sigil`). Shell completion adds a
    `blocked tasks` group.

- **`capture-task-id`.** A new `blocked_task_replacement` field (`*note:id`, quoted when
  needed) beside `dependency_replacement`.
- **`bob capture` JSON (per item).**
  - `dependency_updates[]`:
    - Every dependency update the item made: the item's own `&` update first, then one
      per `*` target in typed order.
    - Present whenever at least one update exists, so `&`-only items carry it too.
    - The singular `dependency_update` stays and equals `dependency_updates[0]`. For
      `N *T`, it is therefore T's update (dependent T, prerequisites [N]), which older
      Mac builds already render truthfully.
    - Each entry gains:
      - `role`: `dependent` for the item's own `&` update, `blocked_task` for a `*`
        target;
      - `previous_status` and `previous_status_name`;
      - `blocked_residents`, the count of `^prj` residents newly blocked (omitted when
        0).
  - `consumed_links[]`:
    - Fields:
      `{day_file, line, link, note, block_id, text, action, pomodoro: {line, name?, time_range?, status, removed}}`.
    - `action` is `removed`, `token_removed`, or `notes_kept`.
    - `status` is `running` or `queued`.
    - `line` values are 1-based in the item's pre-state day text.
  - `minted_blocker_block_id {path, line, block_id}`, only when N's ID was minted for
    `*`.
  - Batch `pomodoro_blocks` entries gain `removed: true` (omitted when false) and the
    role `removed`. A deleted entry's lines all carry `change: removed`.
  - The legacy `warnings[]` consumption string stays for older clients. Human output
    replaces it with the structured lines below, so the same fact is never printed
    twice.
- **Human output** (the example shows the intended lines; match the existing new-task
  line format):

  ```text
  ✓ Ask Kelly for references!  mac_inbox.md ^ask-kelly-for-references   [*] Next
    🧱 blocks "Do cleanup tasks!"  job.md ^cleanup  [/] → [?] Blocked
    − removed its Task Link from JOB 10:10–10:35 (running) · deleted the empty Pomodoro
  ```

  - **Variants:**
    - `− kept its notes under JOB 10:10–10:35 as plain text`;
    - `· blocked 4 project tasks` for a `^prj` T;
    - `would …` in dry runs.
  - **`&` consumption** prints the same `−` lines.
  - **Existing-owner items** print `🧱 ^ask-kelly now blocks "…"`.

### 6. Vectors (the `contract` phase writes them; later phases test them)

Execution vectors, **ST**. Add them to `docs/task-dependencies.md` beside §13.5; the
phase picks the exact section number.

| #    | Situation                                                                                                       | Outcome                                                                                    |
| ---- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| ST1  | `N *T`, T Ready and unlinked                                                                                    | N `[ ]` with a minted `^id` and a BLOCKS line; T's Depends-On gains N; T `[?]` and fresh   |
| ST2  | T `[*]`, linked under the running entry, which has other children                                               | link removed; entry kept; N `[*]`                                                          |
| ST3  | T `[/]`, the running entry's only child                                                                         | link removed; entry deleted and reported (`removed: true`, `(running)`); N capped at `[*]` |
| ST4  | T's link bullet has authored child notes                                                                        | bullet becomes T's plain text; children and entry kept (`notes_kept`)                      |
| ST5  | nested T; the ledger link is a basename or alias spelling                                                       | consumed by identity                                                                       |
| ST6  | `N @mac_inbox:id *T`                                                                                            | N linked into the session first; T's link consumed; session kept                           |
| ST7  | an unrelated childless placeholder entry                                                                        | untouched by capture                                                                       |
| ST8  | `^prj` T                                                                                                        | residents `[?]` plus `#prj_is_blocked`; their links consumed; `blocked_residents` reported |
| ST9  | `^prj` T while N lands in that project or a descendant                                                          | refused; vault byte-identical                                                              |
| ST10 | `N *T &T`; an existing owner that already depends on T; transitive and cross-item cycles                        | refused; vault byte-identical                                                              |
| ST11 | T closed, in a daily note, in `done/`/`_templates`/fenced code, self, or with two or malformed Depends-On lines | refused with a teaching error                                                              |
| ST12 | repeat `*T` on an existing owner already on T's line; a second live run                                         | `already_present`; no-op                                                                   |
| ST13 | `N *T1 *T2 *T1`                                                                                                 | T1 then T2; the duplicate is a no-op                                                       |
| ST14 | `N &P *T`                                                                                                       | T → N → P; N `[?]`; T `[?]`                                                                |
| ST15 | `dependencies.blocks_lines: false`                                                                              | N still gets an ID; no BLOCKS line                                                         |
| ST16 | `N *T` then a later item `@T-route:T-id`                                                                        | N dropped from T's line (capture-link-drops); final preview accurate                       |
| ST17 | `N *T` vs the two-item `N @mac_inbox^id` + `&mac_inbox:id @T-route+T-id`, T Ready or Next                       | identical vault diffs apart from the ID                                                    |
| ST18 | any ST vector, dry run vs live run                                                                              | identical JSON apart from `dry_run`                                                        |
| ST19 | `*T @route+id`, and `Note text @route+id *T`                                                                    | owner gets the normal §5 transfer (may become `[/]`); sub-bullet added                     |

Parse vectors, **STP**, cover every row of the §2 table. They also cover `=x*1`, `=*`,
and `==*` unchanged; `&no*te:id` and `*no*te:id` invalid; `@cash &foo:bar Buy milk` now
a dependency; and `!sase:x *foo` keeping the bang teaching error.

Ledger vectors (amend `docs/task-dependencies.md` §13.14):

- **PDL7** becomes "reconcile and plugin consumption: whole subtree removed".
- **PDL8** is capture keeping notes under a plain-text bullet.
- **PDL9** is capture deleting only the entries it emptied.
- **PDL10** is capture's identity matching.
- **PDL11** is capture's `consumed_links[]` and `removed` block reporting.

### 7. Bob Mac Capture

The app stays a renderer (`decisions:mac-capture-is-a-thin-client`). It has:

- one dependency picker parameterized by direction;
- the `blocked_task_*` span kinds;
- `*` support in Backspace-on-empty-filter and in the Add-ID flow;
- preview rendering of `dependency_updates[]`, `consumed_links[]`, and deleted entries.

A line-initial `*` stays a bullet placeholder. bob never makes a line-initial `*` a
directive, so the two cannot collide.

### 8. Deliberately not doing

Phases record these as `PROPOSED FOLLOW-UP:` notes where relevant:

- **Reconcile and bob-plugins parity.** Keeping notes, identity matching, and
  only-emptied deletion are capture-only in v1. Reconcile keeps PDL7, and plugin
  Depends-On consumption is unchanged.
- **`task_complete` retirement.** Its empty-entry pass still deletes every childless
  entry.
- **Auto-linking N into T's slot.** The explicit `N @mac_inbox:id *T` idiom covers it
  (`decisions:unblocked-tasks-are-offered`).
- **A picker key that flips `&`↔`*` through `capture-rewrite`.**
- **The soft-block drop on later link gestures.** It stays as designed.
- **The Obsidian reverse Blocks… stage.** That is bob-cli-3p.

## Cross-cutting rules for every phase

- **Contract first.** The docs written by `contract` own the behavior. If an
  implementation must differ, fix the doc in the same commit and say so in the commit
  message.
- **No live runs on Bryan's vault.** Never run live `bob capture`, `bob task reconcile`,
  or `bob projects sync` against `~/bob`. Dry runs are fine; tests use temporary vaults
  (`tests/cli/support.rs` helpers, `BOB_NOW`, `BOB_DAY_FILE`).
- **Verification.** Every bob-cli phase that touches code finishes green on `just check`
  (fmt, clippy, every test binary). If a failure looks unrelated, rerun it on a clean
  base, cite it, and finish. Dry-run JSON must equal live JSON
  (`capture_json_dry_run_matches_real` pattern).
- **`&` must not regress.** Existing `&`, link-guard, and BLOCKS tests stay green. The
  only expected `&` changes are §3's consumption improvements and the additive
  `dependency_updates[]`.
- **Thin client.** The Mac phase never decides grammar or semantics.
- **Follow-ups.** Phase workers record `PROPOSED FOLLOW-UP:` notes on their own phase
  bead and never create beads.
- **Memory.** Only `publish` edits memory, and only under accepted memory decisions.

## Phase: contract — docs contract, grammar, wire names, and vectors

**Repo:** bob-cli. Docs only, in one commit.

1. **`docs/capture.md`:**
   - Grammar-table rows for every §2 form, next to the `&` rows (around line 88).
   - A new section `### Blocking existing tasks with '*'` right after
     `### Adding prerequisites with '&'`, covering:
     - the meaning (§1) and the §2 guards, examples table, and rejections;
     - shell quoting;
     - the "take its place" idiom `N @mac_inbox:id *T`;
     - the soft-block note that a later link gesture drops the edge;
     - the Next cap;
     - consumption effects (§3) and human output (§5);
     - a full JSON example.
   - Update the `&` section's consumption paragraph and the JSON output section with
     every §5 field: `dependency_updates[]` and its new entry fields,
     `consumed_links[]`, `minted_blocker_block_id`, and the `removed` Pomodoro blocks.
   - Update the `capture-parse` section (`blocked_tasks[]`, spans, need `blocked_task`),
     the `capture-complete` section (context, rows, ranking, disabled reasons), and
     `## bob capture-task-id` (`blocked_task_replacement`).
2. **`docs/task-dependencies.md`:**
   - §5: a new bullet, **Adding a dependency from the prerequisite side** (capture `*`).
     Word it so bob-cli-3p can share it. Also amend the Consumption bullet for capture's
     PDL8–PDL11 behavior.
   - §13.5 Removal: split it into reconcile/plugins (unchanged) and capture (§3).
   - §13.14: the PDL7–PDL11 amendments (§6).
   - §14.7 immediate writers: capture `*` stages N's BLOCKS line on T's add.
   - A new subsection for capture `*` holding the ST vectors (§6).
3. **`docs/completion.md` "Capture markers":** add the missing `&` bullet and a `*`
   bullet.
4. **`docs/freshness.md` §5 `bob capture` row:** an `*` edit to T stamps T. N, a new
   task, is not stamped, and neither are an existing owner's target-ID and lane edits.
5. **`README.md`:** grammar rows beside the `&` rows (around lines 358–359), plus the
   `*` picker sentence near the `&` picker paragraph.
6. No code. Leave `just check` for the code phases.

## Phase: consume — shared capture consumption by identity, keeping notes, and reported

**Repo:** bob-cli. Implements §3 and PDL8–PDL11, and resolves bob-cli-7l.

1. **Shared planner.** In `src/native/task_status_hooks/structure.rs`:
   - Add an options-taking variant of `apply_blocked_consumption` and
     `plan_blocked_link_removals`: `keep_notes` (link-only bullet with children →
     plain-text bullet, children kept) and `only_emptied` (filter the empty-entry plan
     to entries that had a child before and none after).
   - Return per-link actions and the deleted entries.
   - Reconcile keeps calling it with today's behavior.
     `tests/cli/task_status_hooks/consume.rs` must stay green unchanged.
2. **Capture adapter.** Rewrite `consume_live_links`
   (`src/native/capture/project_deps.rs`) to:
   - take a set of `(vault-relative path, block id)` identities;
   - resolve the day file's references with `SnapshotCloseVault::from_planner`;
   - call the planner with both options on.

   Update every caller to pass identities: `plan_existing_task_dependencies`,
   `plan_resident_block`, and the guard paths in `pomodoro_link.rs`, `task_toggle.rs`,
   `ensure_next.rs`, and `pomodoro_close.rs`.

3. **Reporting.**
   - Add `ConsumedLinkJson` and per-item `consumed_links[]` (§5), skipped when empty.
   - Detect a running entry with the existing running-Pomodoro helpers
     (`capture_pomodoro_close::find_running_pomodoro` or the scan model).
   - Teach `PomodoroBlockTracker` to report every `PomodoroBlockBefore::Removed` entry
     as a block with `removed: true`, role `removed`, its pre-state status, and all
     lines `change: removed`.
   - Fix the removed-ref index so it is relative to the item's pre-state day text even
     when the same item already edited the day file above it (for example, N linked
     earlier in the item).
   - Keep the legacy warning string in `warnings[]`. Human output prints the §5 `−`
     lines for every consumption and suppresses that one warning line.
4. **Tests.**
   - Unit tests in `structure.rs`/`project_deps.rs`:
     - keep-notes conversion, including a task text holding a block link;
     - only-emptied with an unrelated childless entry;
     - alias, basename, path, case, and `[[#^local]]` matching;
     - struck links untouched;
     - closed entries byte-identical.
   - CLI tests (`tests/cli/capture/parse_dependency.rs` and the guard-path test files)
     for an `&` add consuming:
     - a running entry's only link, which is deleted and reported;
     - a link with notes, which keeps the notes and the entry;
     - an unrelated placeholder, which is untouched;
     - an index-shifted removed ref.
   - Update `capture_dependency_consumes_links_when_newly_blocked` and any
     retirement-path tests whose `pomodoro_blocks` now include removed entries.
5. Close bob-cli-7l with
   `sase bead close bob-cli-7l --note "<commit + what was verified>"`.

## Phase: parse — two-direction lexer, Markdown guards, and capture-parse wire

**Repo:** bob-cli. Implements §2 and the parse half of §5, and resolves bob-cli-7m.

1. **Scanner** (`src/native/capture_language/dependencies.rs`):
   - Generalize the line scan to a relation `DependsOn` (`&`) or `Blocks` (`*`), reusing
     `scan_modifier_with_sigil`.
   - Add the `*` guards (rules 2–5).
   - Extend `validate_note` to reject `*` for both sigils.
   - Bare-`*` placement is last-token-only and never line-initial.
   - Extend the escape scan and the two unescape helpers to `\*` within rule 5's scope.
2. **Runs.**
   - In `extract_line_dependencies`, the leading run skips `is_trailing_transparent`
     tokens like the trailing run does. This fixes bob-cli-7m for both sigils.
   - The trailing run also treats `\*…` tokens as transparent.
   - `LineDependencies` and `ItemDependencySet` carry a separate blocked list.
3. **Models.**
   - Add `ParsedCaptureText.blocked_tasks: Vec<ParsedDependency>` (`model.rs`), and
     `blocked_tasks` on `EditorParse` / `EditorItemParse`.
   - Add `Need::BlockedTask` (label `blocked_task`) and the three
     `SpanKind::BlockedTask*` kinds (`editor_model.rs`).
   - `dependency_target` and mode `TaskDependency` are computed when **either** list is
     non-empty (`editor_parse.rs` ~2085–2190, `item.rs` ~415–930).
4. **Rejections.** Every `&` rejection site listed in §2 gets the `*` twin, including
   `item.rs` operator, owner, and section checks, `resolve_execution_ownership`, editor
   codes `dependency_unsupported_target` / `invalid_dependency_target` /
   `invalid_dependency`, `@@` inheritance (`finish_inherited_dependency_target`), and
   rewrite absorption (`capture_language/rewrite.rs` ~426, ~488). Messages name the
   typed sigil. The `!` claim keeps precedence.
5. **`capture-parse`.** Add `blocked_tasks[]` (top-level and per item), the help text
   beside the `&` help (~279–299), and the modes/needs `after_help`.
6. **Tests.**
   - Inline lexer tests for every STP vector.
   - CLI parse tests in a new `tests/cli/capture/parse_blocked_task.rs`, registered in
     `tests/cli/capture/mod.rs`, mirroring `parse_dependency.rs`.
   - A leading-`@route` test for `&` in `parse_dependency.rs`.
   - `=x*`, `=*`, and `==*` regression tests.
7. **Execution until `execute` lands.** `bob capture` must refuse any item with
   `blocked_tasks` with a clear "`*` blocked tasks are not supported yet" error, so a
   half-landed epic never writes `*` as text. `execute` removes this guard.
8. Close bob-cli-7m with `sase bead close bob-cli-7m --note "<commit + vectors>"`.

## Phase: writer — path-addressed existing-dependent writer with star guards

**Repo:** bob-cli (`src/native/capture/dependencies.rs`, plus a new
`src/native/capture/blocked_tasks.rs` if it reads better). No grammar wiring. `&` vault
bytes stay identical; its JSON gains only the additive fields in step 6.

1. **Path addressing.** Change `plan_existing_task_dependencies` to take a dependent
   reference `{relative: PathBuf, block_id}` instead of a lowercased route. Existing `&`
   callers build it exactly as today (`{route}.md`).
2. **Options.** Add an options struct with `transfer_cap: Option<lane>`; the `*` path
   passes Next for a new N. Apply it to `transfer_lane` before `plan_promotions`.
3. **Target resolver.** Add `resolve_blocked_task_target`, which resolves the typed
   `note:block-id` like `&` (`resolve_prerequisite_note`, staged fallback) and enforces
   every A8 refusal from §4 step 1 with teaching errors.
   - Confirm the existing `^prj` same-note/descendant check (~2144–2167) refuses a
     staged new N inside T's project or a descendant.
   - Extend it if it doesn't.
4. **Always-mint.** Generalize `mint_staged_dependent_id` into a reservation helper that
   mints N's ID for `*` regardless of `blocks_lines`, using this batch's reservations.
5. **Relocate by ID.** Add `locate_task_by_block_id` (staged-aware) so callers can
   re-find N after every pass.
6. **Report fields.** Extend `DependencyUpdateJson` with `role`, `previous_status`,
   `previous_status_name`, and `blocked_residents` (wired from `plan_resident_block`).
   Existing `&` JSON gains only `role: "dependent"` and the previous-status fields.
7. **Unit tests.** Cover path addressing for a nested dependent, every A8 refusal, the
   Next cap with a `[/]` and with a `^prj` T, always-mint with `blocks_lines` off,
   relocate after `[id::]` stamping and lane promotion, and the `^prj` internal refusal.

## Phase: complete — blocked-task completion pool and capture-task-id replacement

**Repo:** bob-cli. Implements the completion half of §5.

1. **Detection.** Add `CompletionContext::BlockedTask` and a
   `blocked_task_completion_field_at` that mirrors `dependency_completion_field_at`
   (`capture_language/completion.rs` ~171) with the `*` guards. Wire it next to the `&`
   call (~678). Strip `*` modifiers before marker completion, like `&` (~696).
2. **Candidates** (`capture_complete/engine.rs`, `candidates.rs`,
   `capture_dependency_tasks.rs`):
   - A `blocked_task_candidates` pool built from `capture_dependency_tasks::discover`,
     filtered to open, editable tasks (A8).
   - Ranking per §5, using `capture_active_tasks::read_ledger` for today's linked tasks
     and the running entry.
   - `already_blocked` is set when the owner already sits on T's Depends-On line, or T
     was typed earlier in the item.
   - `disabled_reason` cases:
     - **self**: the owner task itself;
     - **would-cycle**: the owner, or the item's typed `&` prerequisites, already reach
       T. Use edges from the discovered Depends-On lines; the writer remains
       authoritative.
     - **internal `^prj`**: T is a `^prj` whose project contains the owner's destination
       note;
     - **duplicate ID**: the existing reason.
   - Replacements go through `replacement_for_sigil(.., b'*')`.
3. **Surfaces.**
   - `capture_complete/shell.rs`: a `blocked tasks` group.
   - `render.rs`: a human context label.
   - `capture_task_id.rs`: `blocked_task_replacement` plus its help text (~91).
4. **Tests.**
   - A new `tests/cli/capture/complete_blocked_task.rs` (registered in `mod.rs`)
     covering `X *`, `X *jo`, and `X *job:`, the owner shapes, ranking with a running
     entry, closed and excluded tasks absent, and each `disabled_reason`.
   - Task-ID tests beside `complete_dependency.rs` ~362–553.

## Phase: execute — execute `*` captures and report them

**Repo:** bob-cli. Implements §4 and the remaining §5 fields, and removes `parse`'s
temporary refusal.

1. **New-task items** (`plan.rs` `plan_capture_item`):
   - Reserve N's ID before the capture line is built (A9).
   - Plan N's own `&` dependencies as today, then insert and link N.
   - In the dependency-reporting block (~1303), run one writer pass per resolved T with
     prerequisites = [N] and the Next cap, re-locating N by ID after each pass.
   - Collect `dependency_updates[]`, `consumed_links[]`, `task_blocks` refs (T as
     `dependency`, N as `dependency_target`), and `pomodoro_refs`.
2. **Existing owners.**
   - Extend `plan_dependency_only_item` for `*`-only and `&`+`*` owners. Apply the `&`
     update first, then the `*` passes, with no Next cap.
   - Extend the sub-bullet existing-target path (`Note text @route+id *T`) and the
     forced `--task`/`--route` path.
3. **Outputs.**
   - Emit `dependency_updates[]` with the singular field equal to its first entry, and
     `minted_blocker_block_id`.
   - Print the §5 human lines (`🧱 blocks …`, the existing-owner variant, residents, dry
     run) in `output.rs`.
4. **Tests.** A new `tests/cli/capture/blocked_tasks.rs`, registered in `mod.rs`,
   covering ST1–ST19. Assert:
   - exact bytes for the T and N notes and the day file;
   - JSON fields;
   - human output;
   - dry run equal to live;
   - a second live run is a no-op.

   ST17 compares the vault diff of the one-item and two-item spellings.

5. Update the `capture --help`/`capture-parse` text if it still calls `*` unsupported,
   and fix any doc example that disagrees with real output.

## Phase: mac — Bob Mac Capture direction-aware picker and blocks preview

**Repo:** bob-mac-capture. Open it with `sase repo open bob-mac-capture -r "<why>"`;
read its `AGENTS.md` and `README.md` first. Build `bob` from a bob-cli checkout that
contains the `complete` and `execute` commits, and use it to capture fixtures.

1. **Picker.**
   - Parameterize `DependencyPickerPresentation` by direction (prerequisite or blocks):
     the fuzzy `leadingSigil`, the `insertionPrefix`, and all copy.
   - Add a `CapturePickerSource` case for blocked tasks, or a direction payload on
     `.dependency`, whichever keeps the exhaustive switches simplest. Trigger byte
     is 42.
   - Blocks copy:
     - header `Blocks · Choose a task to block` (or `For: …` with an owner);
     - summary `Becomes Blocked until this is done`;
     - action line `↩ inserts *job:cleanup · space adds another *`;
     - badge `Already blocked`;
     - the disabled reasons verbatim from bob.
   - Route context `blocked_task` and need `blocked_task` through the shared handler
     (`handleCompletionResponse`, `pickerNeed`, `shouldRequestCompletion`,
     `dependencyPickerQuery`).
   - The `*` path must never use the hard-coded `&` prefix.
2. **Editor.**
   - Add the three `blocked_task_*` span kinds to the completion trigger sets and to
     `captureSemanticCategory`. Give the sigil its own color, as `task_complete_sigil`
     has; the note and ID match their `&` counterparts.
   - `removePickerTrigger` accepts `*` (42) for the blocked-task source.
   - Confirm Tab and Shift-Tab precedence: a line-initial `*` stays a bullet
     placeholder. `closePendingTrim` and the close park rendering are unaffected.
3. **Add-ID.**
   - Decode `blocked_task_replacement` (`CaptureTaskIDSuccess`) and add a purpose or
     direction to `CaptureTaskIDPromptPurpose`.
   - The insert line reads `Inserts *note:id`.
   - Missing field → `This Bob is too old for blocked-task picks. Update Bob first.`
4. **Preview.**
   - Decode tolerantly (`try?`): `dependency_updates[]` (new `role`, previous status,
     residents), `consumed_links[]`, `minted_blocker_block_id`, and the Pomodoro block
     `removed`. When the array is absent, fall back to the singular `dependency_update`.
   - Render per update: `role: blocked_task` → `🧱 Blocks "T" · [/] → [?] Blocked`, plus
     `· blocks N project tasks`. Render consumed links with the §5 wording.
   - A deleted running session is red. `removed` lines already render red.
5. **Fixtures and tests.**
   - Real-bob fixtures from a temporary vault: `blocked-task-parse-*.json`,
     `blocked-task-complete-*.json`, `blocked-task-task-id-success.json`, and
     `blocked-task-preview-*.json` (new N; running-session deletion; notes kept;
     `^prj`). Wire them into `Tests/Fixtures/fake-bob`.
   - Extend `DependencyPickerPresentationTests` (blocks direction),
     `DependencyDecodingTests`, and `CaptureDependencyPanelTests` (the `*` flow,
     Backspace, Add-ID).
   - Update `README.md`.
6. **Verification** per `AGENTS.md`:
   - `just portable-check` on athena (Linux Swift 6.0.3 covers CaptureCore). If the Mac
     is reachable, also compile a scratch copy there: rsync without `.build`/`.git`, run
     `bundle.sh` and `just typecheck-tests`, then delete the copy.
   - After pushing, watch macOS CI with `gh run list -R bobs-org/bob-mac-capture`
     through `/sase_monitor`, and fix failures. A red lint step skips the later steps.
   - If the Mac targets cannot be verified, say so in the phase's final report.
   - Record a manual check for Bryan: `Ask Kelly *<note>:<id>` on a task linked in the
     running session.

## Phase: publish — decision record and final docs sweep

**Repo:** bob-cli.

> [!decision] decision_record
>
> Use `/sase_memory_write`, then add a decisions-web strand (suggested slug
> `capture-star-blocks-existing-tasks`) in the existing record format: Applies to,
> Claim, Why, Cost, Reopens when, and Evidence.
>
> - **Claim.** The `*` sigil and its Markdown guards. "N blocks T" is written only on
>   T's Depends-On line. Capture consumption is one shared path (identity, notes kept,
>   only-emptied, reported). A new blocker is capped at Next. Link-then-consume
>   ordering. Existing owners are allowed. No auto-link into T's slot.
> - **Why.** The rejected alternatives come from the research's alternatives table: the
>   two-item draft, a Task Card follow-up, directional `&`, a `blocks:` keyword, a
>   Mac-only toggle, authoring BLOCKS, auto-linking, and other sigils.
> - **Cost.** Capture now edits notes the user didn't touch. Capture and
>   reconcile/plugins consumption diverge on PDL7. The block is soft. Two release
>   trains.
> - **Evidence.** The research ref, this plan's `plan:` ref, and every phase commit in
>   both repos.
>
> Link it to `[[decisions:task-deps-are-depends-on-links]]` and
> `[[decisions:capture-link-drops-blocking-deps]]`. Mark no older record superseded:
> none is contradicted. Run `sase memory init`.

> [!decision] decision_record = no
>
> Skip the record.

In both cases:

1. Run the final binary against a temporary vault for ST1, ST3, ST4, and ST8. Make
   `docs/capture.md`'s JSON and human examples match that real output.
2. Confirm that README and `docs/completion.md` agree with the shipped wire names.
3. Run `just check` only if code changed.
