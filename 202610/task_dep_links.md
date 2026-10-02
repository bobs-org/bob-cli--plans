---
tier: epic
title: 'Task dependency links: one Depends-On line, a vault-wide Ctrl+Shift+P picker,
  and live chips'
goal: 'A task''s prerequisites live as plain task dependency links on one managed
  `⛓️ **DEPENDS ON:**` first-child line. That line is the source of truth, and the
  `[dependsOn::]` / `[id::]` fields are derived from it. Ctrl+Shift+P adds and removes
  prerequisites by fuzzy-searching every open task in the vault. bob-ledger-tools
  draws each link as a live status chip. The vault no longer uses transcluded dependency
  bullets. The glossary, docs, and decision records describe the new contract.

  '
phases:
- id: contract
  title: Dependency-line contract doc and conformance vectors
  depends_on: []
  size: medium
  description: 'contract: write docs/task-dependencies.md (grammar, identity, R1-R10
    reconciliation, semantics, stage, chips, nav api v1, legacy window) with DP/DW/DR/DK/DC
    vectors, and pin DK ranking vectors against the capture ranker in Rust.'
- id: hooks-edges
  title: Rust dependency-line parser, promotion edges, and parser guards
  depends_on:
  - contract
  size: medium
  description: 'hooks-edges: add the bob-cli task_dependencies module (parse, canonical
    format, link form, shared id encoder, legacy children). Promotion edges come from
    Depends-On links plus legacy children, and #^ref embeds stop being edges. Guard
    the capture close, section, log, and placement parsers, and fix move-done-tasks
    pathless links.'
- id: hooks-reconcile
  title: R1-R10 reconciliation in bob task-status-hooks
  depends_on:
  - hooks-edges
  size: medium
  description: 'hooks-reconcile: before Blocked derivation, project Depends-On lines
    into the dependsOn/id fields and adopt, heal, canonicalize, and warn. Projection-only
    notes are written through the guarded pipeline and quiet interval. Add JSON and
    human output, docs, and tests, then verify with a read-only dry run against the
    vault.'
- id: chips
  title: bob-ledger-tools live dependency chips
  depends_on:
  - contract
  size: medium
  description: 'chips: render Depends-On lines as live status chips in Live Preview
    and Reading view, built from the Tasks cache memo with native click and hover,
    x and + actions through the nav api when present, a summary, accessible CSS on
    the task-status tokens, and tests.'
- id: compat
  title: task-status-cycler and block-id-prompt compatibility
  depends_on:
  - contract
  size: medium
  description: 'compat: the cycler skips strike, restore, and tree close on Depends-On
    lines; Ctrl+Enter closes or reopens the target only; Alt+] and Alt+[ cycle the
    link under the cursor; the normalizer never edits the line. block-id-prompt''s
    Ctrl+Shift+Enter refuses the line. Tests, bumps, and deploy.'
- id: nav-model
  title: Navigation-hotkeys dependency model, single-transaction writer, and api v1
  depends_on:
  - contract
  - chips
  - compat
  size: medium
  description: 'nav-model: add the Depends-On grammar and pure planner, a writer that
    prepares targets and then commits the parent in one transaction (status effects,
    commitment transfer, immediate recovery on removal, legacy fold), switch the existing
    picker paths and recovery edges to it, and add api v1.'
- id: nav-stage
  title: Vault-wide Ctrl+Shift+P Depends on stage
  depends_on:
  - nav-model
  size: medium
  description: 'nav-stage: add the vault-wide candidate pool from the Tasks cache
    and open buffers, a port of the capture ranker, the CURRENT/RESULTS/BLOCKED layout,
    guards (self, cycle, unencodable, stale), the + id flow, every entry point including
    Task Link mode and the line itself, the summary pill, and notices.'
- id: nav-gestures
  title: Gesture cleanup, hand-edit mirror, and legacy writer removal
  depends_on:
  - nav-stage
  size: medium
  description: 'nav-gestures: ! becomes a pure transclusion toggle and is refused
    on the line; Ctrl+D removes the line and the field with recovery; add the editor
    hand-edit mirror; delete the embed-writing code, the consolidate command, and
    the old migration scripts; add Ctrl+Shift+M tests and update the README.'
- id: fleet-rollout
  title: Install bob and sync plugins on every machine
  depends_on:
  - hooks-reconcile
  - chips
  - compat
  - nav-stage
  - nav-gestures
  size: small
  description: 'fleet-rollout: install bob from master and sync the four plugins on
    this host, then on the MacBook (best effort), and record each machine''s bob commit,
    hooks capability, and plugin versions.'
- id: vault-migrate
  title: Migrate the vault to Depends-On lines
  depends_on:
  - fleet-rollout
  size: medium
  description: 'vault-migrate: preflight the fleet (ask Bryan only if the Mac is unverified),
    add a tested dry-run-first migration script, rehearse on a vault copy with before/after
    hooks checks, run it live, prove idempotence, update the CSS comment, and sync.'
- id: publish
  title: Publish glossary, decision record, and final docs
  depends_on:
  - vault-migrate
  size: small
  description: 'publish: add glossary:task-dependency-link, amend glossary:task-link,
    add the Depends-On decision record and mark task-status-is-derived, republish
    memory, sweep docs for stale transclusion wording, and record follow-ups and Bryan''s
    checklist.'
proposed_by: bbugyi200.athena.0vl
create_time: 2026-10-02 16:54:33
status: wip
bead_id: bob-cli-3n
---

- **PROMPT:** [prompts/202610/task_dep_links.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_dep_links.md)
- **BEAD:** [bob-cli-3n](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3n/README.md)

# Plan: Task dependency links on one Depends-On line

## Context

This epic implements
`research:202610/task_dep_link_depends_on_line/task_dep_link_depends_on_line.md` ("the
report"). Bryan agreed with **all** of its recommendations: ADJ-1…ADJ-10, the
Recommended solution, the Recommended design, the Delivery order, and the recommended
answers to Q1–Q6. **Every phase reads the report first** with `sase artifact read` and
treats it as the design reference. Where this plan refines the report (marked
**Refinement**), this plan wins.

What is wrong today (report, "How dependencies work today"):

- The Ctrl+Shift+P `dependsOn` stage only lists open tasks **in the current note** (F4).
  That is why 44 of 45 edges are same-note.
- Blocked comes from `[dependsOn::]` (F1), but promotion comes from _any_ sole
  transcluded child (F2), including 21 `#^ref` reading embeds.
- Closing an embedded ledger link recursively closes the prerequisites it transcludes
  (F3).
- Ctrl+D on `dependsOn` orphans the embeds. The consolidate command can delete
  cross-note embeds. Picker writes take several undo steps (F10).

Adopted answers:

- **Q1:** `#^ref` embeds stop promoting.
- **Q2:** `⛓️` plus `•`.
- **Q3:** adding a dependency transfers the dependent's Next/In Progress commitment to
  an open prerequisite.
- **Q4:** a reverse "Blocks…" stage comes later. It is not in this epic.
- **Q5:** the line is the authority. Edits made in the Tasks modal to a task that
  already has a line are dropped, with a warning. This is a documented cost.
- **Q6:** the "Edit task dependencies" command ships with no hotkey.

Surfaces:

| Surface            | Where                                                                                                 | How to open                                                        |
| ------------------ | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `bob` CLI and docs | this repo (`bob-cli`)                                                                                 | your workspace                                                     |
| Obsidian plugins   | `bob-plugins` (`bob-navigation-hotkeys`, `bob-ledger-tools`, `task-status-cycler`, `block-id-prompt`) | `sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md` |
| Vault              | `~/bob` (git-synced; custom `bob-*` plugins are gitignored, deployed per machine)                     | edit in place, following `~/bob/AGENTS.md`                         |
| MacBook            | `ssh mac` (user `bbugyi`; offline unless the lid is open; runs the hooks cron)                        | best effort only                                                   |

No Bob Mac Capture change is needed. Capture has no dependency grammar, and the line
shows up as an ordinary child in previews (`mac-capture-is-a-thin-client`).

## Design

The design is the report's "Recommended design" plus the refinements below. `contract`
turns it into `docs/task-dependencies.md`, which every implementation then cites.

### Vocabulary

- **Task Dependency Link** (aka **task dep link**, **dep link**): a plain, never
  transcluded Task Link to a prerequisite, placed on its dependent's Depends-On line.
- **Depends-On line**: the single managed first-child line that holds a task's dep
  links.
- **Dependent**: the task that owns the line. **Prerequisite**: a link target. "Blocked"
  stays the name of the derived `[?]` state.

### Grammar

```markdown
- [?] #task Make appt w/ Rahway Hospital for CT scan! [dependsOn:: body__hospital-swarm,
  sase_bug_bash__e2e-sase-8v] ^rahway
  - ⛓️ **DEPENDS ON:** [[#^hospital-swarm]] • [[sase_bug_bash#^e2e-sase-8v]]
  - 🗓️ **SCHEDULE LOG**
```

**Writer form**

- `⛓️` (U+26D3 U+FE0F), a space, `**DEPENDS ON:**`, a space, then links joined by `•`.
- **No aliases, embeds, or strikes.**
- Existing links keep their order. New links are appended, so removing a link and adding
  it again moves it to the end. Never re-sort.
- **Position:** the first direct child, or the second if a `❌ **CANCEL LOG**` child
  exists.
  - Reuse the task's existing child indent; otherwise use the parent's indent plus one
    tab.
  - Preserve the note's line endings and its final-newline state.
- **Empty list:** delete the line and remove the field. There is no placeholder.

**Reader tolerance** (writers canonicalise all of these):

- the emoji is missing, or is the legacy `🔗`; VS16 is optional;
- the label is `DEPENDENCIES` instead of `DEPENDS ON`;
- the separator is `·`, `,`, or whitespace instead of `•`;
- a link is aliased, struck, or `!`-embedded;
- the line is a direct child other than the first.

**Parse rules**

- Find the block links first, then check what surrounds them. Never split on separators,
  because an alias can contain one.
- Anything else on the line makes it **malformed**.
- Only a direct child of a `#task` line counts. Fenced code and nested lines never
  count.

**Recognisers that must reject the line:** dedicated Task Link and Pomodoro link
recognisers, section titles, and managed-log parsers in every repo.

### Identity and link form

- **Prerequisite id.** Use the target's existing valid `[id::]` if it has one. Otherwise
  use the canonical `dependency_id(path, blockId)` (`note__path__blockid`), which the
  writer adds to the target. **Refinement:** an existing `[id::]` is never rewritten. A
  target whose path can't be encoded (spaces, dots) is refused **only when it has no
  `[id::]` yet** (ADJ-9; defer the path codec).
- **Link form.** Use the shortest form that is unambiguous under the hooks' resolver:
  1. `[[#^id]]` for a target in the same note;
  2. `[[basename#^id]]` when the basename is unique in the vault (case-insensitive);
  3. otherwise `[[dir/note#^id]]`, the full vault-relative path without `.md`.
- **Field placement.** `[dependsOn:: a, b]` (comma-space) and `[id:: x]` go inside the
  Tasks suffix:
  - to the right of any `[fresh::]`, because the hooks' trailing-field parser stops at
    `fresh`;
  - before a trailing `^block-id`;
  - in Tasks key order (`id` before `dependsOn`).
- **Field order.** The field mirrors the line's order.

### Reconciliation (the hooks; report R1–R10 with refinements)

**Scope and timing**

- **Refinement:** only _open_ dependents are reconciled: status types Todo, In Progress,
  and On Hold, which includes `[?]`. Closed tasks are never touched.
- The step runs on the whole vault on every run, before edges and Blocked derivation,
  inside the existing guarded write.
- The previous daily note snapshot is never written.

**The dependency set** is the line's links followed by any **legacy children** (R8) not
already on the line, deduplicated by (path, block id).

| #   | Situation                                                                                                                                                                                                                            | Resolution                                                                                                                                                                                                                                                                                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | Well-formed line                                                                                                                                                                                                                     | `[dependsOn::]` := the ids of the set's resolved task targets, in set order. Targets without `[id::]` get one. Field ids not accounted for are dropped and reported (`dependency_field_ids_dropped`), except breadcrumbs (R4).                                                                        |
| R2  | No line, field present                                                                                                                                                                                                               | **Adopt** the field ids not covered by legacy children: write a canonical line linking to the tasks that carry each id and have a `^block-id`. Ids that can't be adopted stay in the field and are warned about (`unadoptable_dependency_id`).                                                        |
| R3  | A link doesn't resolve                                                                                                                                                                                                               | **Heal** it when exactly one scanned task has both the link's block id and one of the dependent's unaccounted field ids. Rewrite the link to that task's location in the shortest form, then apply R1.                                                                                                |
| R4  | A link doesn't resolve and can't be healed                                                                                                                                                                                           | Keep it verbatim and warn (`unresolved_dependency_link`). It never blocks. **Refinement:** while any unresolved link remains, keep the dependent's unaccounted field ids as **heal breadcrumbs**, so a task that is cut now and pasted later still heals.                                             |
| R5  | Resolves to a non-task block                                                                                                                                                                                                         | Keep it and warn (`non_task_dependency`). It is not projected.                                                                                                                                                                                                                                        |
| R6  | Points to its own task                                                                                                                                                                                                               | Not projected; warn (`self_dependency`).                                                                                                                                                                                                                                                              |
| R7  | Cycle                                                                                                                                                                                                                                | Keep it (every member stays Blocked) and warn (`dependency_cycle`) with the path.                                                                                                                                                                                                                     |
| R8  | **Legacy child**: a direct child bullet whose only content is one block link (plain, `![[…]]`, `~~[[…]]~~`, or `~~![[…]]~~`) and whose resolved target's id (its `[id::]`, canonical id, or same-note bare block id) is in the field | Counts as a dep link during the legacy window. The hooks never rewrite it; the migration converts it. **Refinement:** plain sole links count too, so un-embedding a legacy child with `!` doesn't silently drop the dependency. Any other sole embed (for example a `#^ref`) is content, not an edge. |
| R9  | Label but no links, and no legacy children                                                                                                                                                                                           | Delete the line and remove the field.                                                                                                                                                                                                                                                                 |
| R10 | Malformed line                                                                                                                                                                                                                       | No change for that dependent; warn (`malformed_dependency_line`). Projection outputs count as structural, so notes modified less than 2 s ago are deferred by the quiet interval.                                                                                                                     |

**Refinements beyond the table**

- **Canonicalise.** A well-formed but non-canonical line is rewritten to the writer form
  with the same targets: label and emoji variants, other separators, `!`, `~~`, and
  aliases.
- **Archive targets.** A link that resolves into `done/` through the existing archive
  catalog is a resolved, _closed_ prerequisite. Its id is kept, it is never warned
  about, and it never blocks.
- **Removals.** Never infer a removal from a _missing_ line (R2 adopts instead). Do
  infer removals from a _present_ line.
- **No moves.** The hooks never move or reorder lines.
- **Output.** Report every change:
  - `dependency_projection_updates`, entries of kind `dependent_field`, `target_id`, or
    `line_removed`;
  - `adopted_dependency_lines`;
  - `healed_dependency_links`;
  - `canonicalized_dependency_lines`;
  - `legacy_dependency_children` (a count, for deciding when to retire legacy reading);
  - `dependency_warnings`, entries of `{kind, path, line, detail}`.

### Dependency semantics

- **Blocked.** A task is `[?]` while any prerequisite is open (AND, finish-to-start) or
  while its `scheduled` date is in the future. Done and Cancelled prerequisites stop
  blocking.
- **Promotion.** Pomodoro roots promote their prerequisites to Next/In Progress along
  the reconciled set: line links plus R8 legacy children. The edge rules are unchanged:
  strongest rank wins, the walk is cycle-safe, recovery uses the same edges, and the
  lanes are sticky.
- **Not edges:** `#^ref` embeds and archived targets.
- **Closing.** Closing a dependent **never** closes its prerequisites. Embedded-tree
  closes in capture `=x` and the cycler ignore Depends-On lines.
- **History.** Closed prerequisites stay on the line as history and show as muted chips.
  Prerequisites never prevent closing a dependent by hand.
- **Adding a dependency** sets the dependent to `[?]` if the target is open. If the
  dependent was `[*]` or `[/]`, an open target rises to at least that lane (Q3,
  `getDependencyPromotionStatus`).
- **Removing a dependency** (ADJ-8) recovers the dependent to its derived rank
  _immediately_ when no open prerequisite and no future `scheduled` date remain.
  Otherwise it stays `[?]`. The hooks keep the final word.

### The Depends on stage (navigation-hotkeys)

**Entry points**

| Cursor                                                  | Gesture                                                                                  | Edits                                                                                       |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| A `#task` line                                          | Ctrl+Shift+P → **Depends on** row (pill `⛓ 2 · 1 open` / `⛓ none` replaces the raw ids)  | this task                                                                                   |
| Anywhere on a Depends-On line, including inside a link  | Ctrl+Shift+P (skips the property step)                                                   | the **owning** task, never the link under the cursor                                        |
| A dedicated Task Link                                   | Ctrl+Shift+P → Depends on (the row is no longer hidden)                                  | the **linked** task in its own note, named in the title                                     |
| A task line, counted                                    | `N<Ctrl+Shift+P>` → Depends on                                                           | this task plus the next N (existing add-to-all/remove-from-all, `k/n` badges, no Tab marks) |
| A chip                                                  | `＋` / hover `×`                                                                         | opens the stage / removes that prerequisite                                                 |
| Anywhere                                                | palette command **Edit task dependencies** (`edit-task-dependencies`, no default hotkey) | the task under the cursor                                                                   |
| Prose with several links, or a selection spanning tasks | any                                                                                      | refused with a short reason                                                                 |

**Layout.** It reuses the `bob-cnp` modal styling:

```text
╭─ ⛓ Depends on · Make appt w/ Rahway Hospital for CT scan! ────────────── body ─╮
│ ⌕ unemp▌                                                                        │
│ 1 prerequisite · 1 open · searching 844 open tasks                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│ CURRENT                                                                         │
│ ✓ [ ] Launch swarm to find hospital                 body · ^hospital-swarm   −  │
│ RESULTS                                                                         │
│   [ ] File for unemployment                         cash · ^unemployment     ＋ │
│   [*] Call unemployment office                      cash · ＋ id                │
│ ⟲ [?] Dispute North Face jacket                     cash   would create a cycle │
│ BLOCKED                                                                         │
│   [?] Rahway billing dispute                        cash   🔒 waits on 1        │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ↑↓ navigate · ⇥ mark · ↵ toggle · esc dismiss                                   │
╰─────────────────────────────────────────────────────────────────────────────────╯
```

**Pool**

- Source: the Tasks plugin cache (`getTasks()` when `getState()` is `"Warm"`). Tasks
  8.4.0 exposes `id` and `dependsOn` on each task.
- Open editor buffers override the cache for their notes, so unsaved edits count.
- If the cache isn't ready, fall back to a one-time vault scan when the stage opens.
- Never read from disk on a keystroke.

**RESULTS rows**

- Included: open tasks that pass the `#task` global filter, including `ref/`, inbox,
  Blocked, and `#hide`. `#hide` tasks are muted and ranked last.
- Excluded: daily notes (`YYYY/YYYYMMDD.md`), `done/`, `_templates`, `_generated`,
  `_conflicts` (matched as a path segment), dot-dirs, fenced code, and the dependent
  itself.

**CURRENT rows** resolve against everything, including closed, archived, hidden, and
missing targets, so any of them can still be removed.

**Ranking.** This ports `capture_link_tasks.rs::rank`:

- Every term must match. Each term scores its best tier across the cleaned description,
  `route:blockId`, block id, note route, and section heading: field prefix 3, word
  prefix 2, substring 1, in-order subsequence 0. Scores are summed.
- Ties keep canonical order: same note in document order, then In Progress, Next, Ready
  (by path, then line), then `#hide`.
- Blocked candidates render in their own BLOCKED section beneath RESULTS.
- An empty query shows CURRENT, then open tasks in the same note, then the In Progress
  and Next lanes.
- At most about 60 rows render. Typing reaches the rest.
- Matched characters are highlighted.
- The row badge shows the block id, never the path-encoded id.

**Keys**

| Key            | Action                                                                                                    |
| -------------- | --------------------------------------------------------------------------------------------------------- |
| type           | fuzzy search                                                                                              |
| ↑ / ↓, ^N / ^P | move                                                                                                      |
| ↵              | **toggle** the highlighted row (add if absent, remove if present) and close; with marks, apply every mark |
| ⇥              | mark or unmark the row (`＋ add` / `− remove` / `＋ id`) and move down                                    |
| Esc            | cancel; writes nothing and allocates no ids                                                               |

**Guards.** These rows are disabled and show their reason:

- the dependent itself;
- a cycle (`⟲`), checked on the graph _after_ the whole batch, with the path in a
  tooltip;
- a target with no `[id::]` whose path can't be encoded;
- a target or dependent that changed since the stage opened: refuse, then reopen fresh.

Removing a link is always allowed.

**Block ids.** A `＋ id` row opens the existing block-ID stage when the change is
applied. It is pre-filled from `suggestBlockIdFromTask`, with uniqueness checked against
the **target's** note, so ↵ accepts it. A batch prompts for one target at a time.
Highlighting a row never writes.

**One gesture writes**

1. **Prepare cross-note targets first.** Add the `^id` and `[id::]` through the open
   editor if the note is open, otherwise with a preimage-checked `vault.process`.
2. **Commit the dependent's note** in **one editor transaction** (one Ctrl+Z):
   - the line;
   - the field;
   - same-note target ids;
   - status effects;
   - folding of legacy children;
   - `[fresh:: today]` via `api.freshness.stampLine`.
3. **If preparation fails**, the dependent is untouched. An unused target id is
   acceptable; a link to an unprepared target is not.

**Notices**

- `⛓ Now waits on "File for unemployment" · Blocked`
- `⛓ No longer waits on "Launch swarm…" · Ready again`
- `⛓ 2 added · 1 removed · Blocked (1 open)`

### Chips (bob-ledger-tools)

```text
[?] Make appt w/ Rahway Hospital for CT scan!
    ⛓ depends on  ( ○ Launch swarm to find hospital )  ( ○ Run e2e on sase-8v ↗ sase_bug_bash )  ＋   waiting on 2
```

**Chip anatomy**

- a miniature checkbox showing the target's status symbol, coloured with the existing
  `--task-status-*` tokens (no new palette);
- the cleaned description, cut at about 40 characters;
- `↗ note` for a target in another note;
- a tooltip with the full text, note path, status name, and `scheduled` date.

**States**

| State                            | Look                                                      |
| -------------------------------- | --------------------------------------------------------- |
| Open, Next, In Progress, Blocked | coloured by status                                        |
| Done                             | ✓, dimmed, struck text; more than three collapse to `✓×N` |
| Cancelled                        | muted ✕, visibly different from Done                      |
| Broken                           | dashed, `⚠ ^id not found`                                 |
| Not a task                       | `⚠ not a task`                                            |

**Summary:** `waiting on N` or `✓ all clear`. It is never written to Markdown.

**Interaction**

- Click opens the target (Mod-click opens a new tab).
- Hover shows the native page preview.
- Hovering a chip shows a `×` that removes that prerequisite. A trailing `＋` opens the
  stage. Both go through nav api v1 and are hidden when it is absent.
- Putting the cursor on the line reveals the raw Markdown. Source mode stays raw.

**Implementation requirements**

- Chips never change line height.
- Only visible ranges are processed, after a cheap `DEPENDS ON` prefilter.
- Data comes only from the in-memory Tasks memo.
- `aria-label`s, visible focus, colour never the only signal, and reduced-motion
  support.
- Wraps between chips, never inside one, including at phone width.
- Without the plugin, the line still reads:
  `⛓️ DEPENDS ON: ^hospital-swarm • sase_bug_bash > ^e2e-sase-8v`.

### Other gestures

| Gesture                                                                         | After this epic                                                                                                                                                    |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `!` / `N!`                                                                      | Only toggles transclusion. Refused on a Depends-On line, with a notice pointing to Ctrl+Shift+P.                                                                   |
| Ctrl+Enter on a link on the line                                                | Closes or reopens the **target** only: root-only, no strike, no re-embed, no tree close. Dependents recover immediately.                                           |
| Cycler strike/restore when a target closes or reopens elsewhere                 | Skips Depends-On lines.                                                                                                                                            |
| Alt+] / Alt+[ on the line                                                       | Cycles the target of the link under the cursor. Never reformats the bullet.                                                                                        |
| Ctrl+Shift+Enter on the line                                                    | Refused, with a notice pointing to Ctrl+Shift+P.                                                                                                                   |
| Ctrl+D on the Depends on row                                                    | Deletes the line, the field, and any legacy children, with ADJ-8 recovery.                                                                                         |
| Hand edits in Obsidian                                                          | Once the cursor leaves the edited line (short debounce), nav applies R1/R9 to that task. Deleting the whole line clears the field. Malformed lines are left alone. |
| Ctrl+Shift+M, `move-done-tasks`                                                 | The line moves with its task. Same-note links inside a moved block whose target stayed behind gain the source note path.                                           |
| "Rewrite dependency navigation links" command, `migrate-dependency-bullets.mjs` | Deleted. They emit embeds.                                                                                                                                         |

### Plugin api v1 (navigation-hotkeys)

```js
app.plugins.plugins["bob-navigation-hotkeys"].api = Object.freeze({
  version: 1,
  openDependencyStage(ref),            // ref: { path, line } — any line of the task block or its Depends-On line
  removeDependency(parentRef, target), // target: { path, blockId }
});
```

- Both members return Promises resolving to `{ ok, reason? }` and never throw.
- `removeDependency` re-reads the dependent and refuses with a notice when it is stale.
- Plugins never import each other's `main.js`. bob-ledger-tools feature-detects
  `api?.version >= 1`.

### Legacy window and rollout order

- **Legacy readers.** Rust and nav read R8 legacy children for one release.
- **Touch migrates.** Any nav dependency write folds the dependent's legacy children
  into its line in the same transaction.
- **Chips and compat ship before any writer emits lines** (ADJ-3). That is why
  `nav-model` depends on `chips` and `compat`.
- **Fleet before migration.** `fleet-rollout` installs everywhere before
  `vault-migrate`.
- **The MacBook.** Its cron is the only hooks runner. Until it is updated, it keeps
  Blocked correct from the fields but loses dependency promotion for parents that
  already have a line.

## Cross-cutting rules for every phase

- **Read first:**
  - the report
    (`sase artifact read research:202610/task_dep_link_depends_on_line/task_dep_link_depends_on_line.md "<why>"`);
  - from `contract` on, `docs/task-dependencies.md` in bob-cli;
  - before changing behaviour a decision record governs,
    `decisions:task-status-is-derived` and `decisions:task-lanes-are-sticky`
    (`sase memory read … -r "<why>"`).
- **Vectors are the contract.** Each implementation copies the vector tables it needs
  from `docs/task-dependencies.md` into its tests, citing the section in a comment, as
  the freshness, Today, and Ready-cap vectors already do. If an implementation disagrees
  with a vector, fix the doc first in the same phase, and say so in the commit.
- **bob-cli:**
  - `just all` (fmt, lint, test) must pass.
  - Keep touched Rust files under about 1500 lines where practical.
  - Update `README.md` and `docs/*.md` in the phase that changes behaviour.
  - If a phase touches `--help` text, read `sase/memory/cli_rules.md` with
    `/sase_memory_read` first.
- **bob-plugins:**
  - `npm test` and `npm run validate` must pass.
  - Add every new test file to the `test` script in `package.json`.
  - Bump the touched plugin's `manifest.json` minor version and update its README row.
  - Deploy with `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>` (the
    default repo path may not exist on this host).
  - Phases that touch the same plugin, `package.json`, or README run in sequence. When
    rebasing, merge both sides of single-line conflicts.
- **Vault:**
  - Only `vault-migrate` edits it.
  - Follow `~/bob/AGENTS.md`: inspect `git status`, and commit only your own files.
  - Never touch daily-note Pomodoro history or `done/`.
  - Finish with `bob vault-sync run` and `bob vault-sync status`.
- **Hooks safety:** never run a live `bob task-status-hooks` pass from a phase except
  where `vault-migrate` says so. Dry runs (`--dry-run -f json`) are always fine.
- **MacBook** steps are best effort: `ssh -o ConnectTimeout=15 mac …`. Record exactly
  what is left when it is unreachable.
- **Memory:** only `publish` edits memory, through `/sase_memory_write`'s Edit And
  Republish path. This plan's approval is the authorization.
- **Follow-ups:** epic workers record `PROPOSED FOLLOW-UP:` notes on their own phase
  bead instead of creating beads.

## Phase: contract — dependency-line contract doc and conformance vectors

**Repo:** bob-cli.

1. **Write `docs/task-dependencies.md`**, the single contract the other phases cite. Its
   sections follow this plan's Design:
   - vocabulary;
   - grammar (writer form, reader tolerance, parse algorithm, recognisers that must
     reject the line);
   - identity and link form;
   - reconciliation (R1–R10 plus refinements, scope, output keys);
   - semantics;
   - the stage (entry points, pool, ranking, keys, guards, block ids, write order,
     notices);
   - chips;
   - other gestures;
   - nav api v1;
   - legacy window and rollout.

   Keep it a contract: precise and testable, with no implementation diaries.

2. **Add numbered vector tables** to the same doc:
   - **DP (parse, ≥ 20):**
     - same-note, unique-basename, and full-path links;
     - an alias containing `•`;
     - struck and `!` links;
     - legacy `🔗`, no emoji, `DEPENDENCIES`, missing VS16;
     - `·`, `,`, and whitespace separators;
     - a half-typed `[[#^a`, and trailing prose (both malformed);
     - label only (empty);
     - the line inside fenced code, nested deeper, or inside a Work Log entry (none of
       these is a line);
     - the line as a non-first direct child (accepted).
   - **DW (write, ≥ 15):**
     - create the line as first child, or after a Cancel Log;
     - append; remove the middle link; remove the last link (deletes the line and the
       field);
     - re-add moves the link to the end;
     - field order mirrors the line;
     - tab and space indents; CRLF and no-final-newline preserved;
     - link-form choice (same note, unique basename, ambiguous basename → full path);
     - an existing `[id::]` is preferred over the canonical id;
     - an unencodable path with no id is refused;
     - field placement to the right of `[fresh::]` and before `^id`;
     - folding legacy children;
     - canonicalising every reader-tolerated variant.
   - **DR (reconcile):**
     - one or more vectors per R1–R10;
     - a closed dependent stays untouched;
     - an archive target;
     - breadcrumb retention, then a later heal;
     - adoption that skips ids covered by legacy children;
     - a `#^ref` embed is not an edge;
     - idempotence (a second pass is a no-op).
   - **DK (ranking, ≥ 8):** over abstract candidates `{text, route, blockId, section}`,
     covering every tier, AND across terms, tie order, and an empty query.
   - **DC (chip model):**
     - open, Next, In Progress, Blocked, Done, Cancelled, broken, and non-task targets;
     - the cross-note `↗` label;
     - collapse of more than three Done;
     - both summaries;
     - 40-character truncation.
3. **Link the doc** from `docs/README.md` (or the docs index the README uses), from
   `README.md`, and with a one-line pointer from `docs/task-status-hooks.md`'s "Derived
   Blocked" section. The full hooks-doc rewrite belongs to the hooks phases.
4. **Pin DK in Rust.** Add a unit test next to the existing ranker tests in
   `src/native/capture_link_tasks.rs` (near
   `ranker_orders_by_tier_sum_then_canonical_order`) that feeds the DK vectors through
   `rank`, so the JS port provably matches capture `:`.
5. **Finish.** `just all`.

## Phase: hooks-edges — Rust parser, promotion edges, and parser guards

**Repo:** bob-cli. Sources: the report's F1–F3 and Sources section.

1. **New module `src/native/task_dependencies/`** (`mod.rs`, `parse.rs`, `format.rs`,
   `legacy.rs`, `tests.rs`; split as needed):
   - **Parser.** Implements the contract grammar.
     - Reuse the link scanner from `task_status_hooks/pomodoro.rs`
       (`block_link_occurrences`; it already handles aliases, `!`, `~~`, and
       `[[#^id]]`). Extract it into a shared helper rather than copying it, and make it
       ignore inline code spans.
     - Malformed lines return a typed error.
   - **Owning-line discovery.** A direct child of a `#task` line (reuse
     `nearest_parent_list_item`), skipping fenced code.
   - **Canonical formatter.**
   - **Link form.** `canonical_link(target, source, &NoteIndex)`.
   - **Id encoder.** Move `collect_done::transform::dependency_id` here as `pub(crate)`
     and import it back into `collect_done`.
   - **Legacy children (R8),** including plain sole links.
   - **Tests:** DP and DW vectors as unit tests.
2. **Promotion edges.** Rewrite `dependency_edges` (`task_status_hooks/references.rs`)
   so edges are the resolved line links plus R8 legacy children:
   - `#^ref` and other sole embeds that aren't managed by the field stop being edges;
   - links into `done/` are not edges;
   - keep the existing unresolved reporting for now;
   - update `docs/task-status-hooks.md`'s graph sections ("Post-rewrite graph", the
     transclusion edge rule, and recent-activity recovery) to describe dep-link edges.
3. **Guards:**
   - `capture_pomodoro_close/linked_tasks.rs::embedded_children` ignores Depends-On
     lines.
   - Tests pin that `is_section_title` and `parse_managed_task_log_marker` reject the
     line.
   - Capture's sub-bullet insertion (`capture/sub_bullet.rs`, around
     `first_direct_managed_log_start`) keeps an existing Depends-On line first. Add a
     test, and fix the placement if needed.
   - Audit every other caller of `sole_transcluded_block_reference`-style matching.
4. **move-done-tasks.** In `collect_done/link_repair.rs`, a pathless `[[#^x]]` inside an
   archived block whose target stays behind is rewritten to
   `[[<source path without .md>#^x]]`. Add a test for both a Depends-On line and a
   legacy child.
5. **Tests:**
   - unit tests for edges: same-note, cross-note, legacy plain/embed/struck, `#^ref` not
     an edge, archive target not an edge;
   - a CLI fixture under `tests/cli/task_status_hooks/` showing a Pomodoro-linked
     dependent promoting its prerequisite through a Depends-On line.
6. **Finish.** Update the README's hooks section, then `just all`.

## Phase: hooks-reconcile — R1–R10 reconciliation in `bob task-status-hooks`

**Repo:** bob-cli. Builds on `hooks-edges`.

1. **Reconcile step.**
   - **Placement:** in `sync_task_statuses` (`task_status_hooks/sync.rs`), right after
     `NoteIndex` and `task_blocks` are rebuilt following the daily-note normalization,
     and before `dependency_edges`.
   - **Edits:** for each open dependent, compute R1–R10 and the refinements, producing
     text edits to the dependent (line and field) and to the targets (`[id::]`).
   - **Re-parse** every touched `FileScan`: tasks, status byte offsets, `depends_on`,
     `task_id`. Edges, `task_dependency_states`, and transitions in the **same run**
     then see the reconciled fields.
   - **Never write** the previous-daily input.
2. **Field writes.**
   - Upsert or remove `[dependsOn::]` and `[id::]` following the contract's placement
     rule.
   - Build on `freshness::placement::tasks_suffix_start`, `task_fields::inline_fields`,
     and the `projects/edits.rs` upsert helpers (`upsert_task_scheduled`,
     `remove_all_inline_fields`, `task_metadata_insertion_offset`). Promote them to
     `pub(crate)` where needed rather than copying them.
3. **`compose_outputs`** (`compose.rs`) today only emits notes with status or grouping
   changes, comparing against the original contents.
   - Generalise the daily `normalized_daily_contents` pattern into a per-file projected
     content layer.
   - Status byte edits apply on the projected contents.
   - Preflight still compares against the original bytes.
   - Projection outputs set `structural_regrouping = true`, so the 2 s quiet interval
     (`task_status_hooks_write.rs::wait_quiet_period`) covers them (R10).
4. **Output.**
   - Add the keys from Design ("Reconciliation") to `SyncResult` (`model.rs`), its
     constructor in `sync.rs`, and `retry.rs::empty_sync_result`.
   - Count projection changes in `change_count`.
   - Add human sections, plus warnings in `print_warnings`.
   - Append the new counts to the **end** of the `Summary:` line; existing prefix
     assertions must keep passing.
5. **Docs.** In `docs/task-status-hooks.md`:
   - a new "Dependency lines" section that summarises R1–R10 and points to the contract;
   - "Derived Blocked" updated;
   - the "Guarded writes" quiet interval now also covers projection writes;
   - "Output" field semantics.

   Also update the README hooks section and the command's `long_about`
   (`task_status_hooks/mod.rs`), reading `cli_rules.md` first.

6. **Tests:** DR vectors as unit tests, plus CLI tests (`tests/cli/task_status_hooks/`)
   for:
   - Blocked via a line;
   - editing the line to remove the last open prerequisite unblocks it in the same run;
   - adoption;
   - heal, and breadcrumb retention followed by a later heal;
   - canonicalisation;
   - a malformed line is a no-op with a warning;
   - quiet-interval deferral;
   - closed dependents untouched;
   - cross-note target `[id::]` write;
   - archive targets;
   - legacy window (line plus legacy children);
   - idempotence (a second run reports zero changes);
   - `--dry-run` writes nothing.
7. **Verification (read-only).** Build, then run both the installed `bob` and the new
   build with `BOB_DIR=~/bob … task-status-hooks --dry-run -f json`. Paste into the
   phase notes:
   - every new count and the warnings by kind;
   - the list of adoptions;
   - the `marked_blocked` / `unblocked` diff.

   Expectations:
   - no Blocked flips;
   - rank changes only where the report's F2 `#^ref` embeds lose their edge;
   - `legacy_dependency_children` near the report's 38 live plus 5 struck.

   Explain any deviation before finishing.

## Phase: chips — bob-ledger-tools live dependency chips

**Repo:** bob-plugins, `plugins/bob-ledger-tools`. The patterns to copy are the
freshness mark (`createFreshnessMarkExtension`, `buildFreshnessMarkDecorations`,
`FreshnessMarkWidget`, `renderFreshnessMarksIn`, `setupFreshnessMarks`) and the badge
link helpers (`makeNoteReadyHeadingAnchor`).

1. **Pure model.**
   - `parseDependencyLine`: a copy of the contract grammar, tested against DP.
   - `dependencyChipModel(lineText, sourcePath, lookup)` returns
     `{label, chips[], doneCollapsed, summary}`. Each chip has
     `{state, symbol, text, noteLabel, linktext, tooltip, ariaLabel}`. Tested against
     DC.
2. **Lookup.**
   - Index the Tasks memo by `(path, blockId)`, extending `freshnessEnsureMemo`'s
     lifecycle rather than adding a second cache.
   - Resolve link paths with
     `app.metadataCache.getFirstLinkpathDest(linkpath, sourcePath)`; a same-note link
     uses `sourcePath`.
   - No disk reads during render.
3. **Live Preview.** A CM6 extension (`createDependencyChipExtension`), registered in
   `setupDependencyChips()` from `onload`:
   - only `view.visibleRanges`, after a cheap `DEPENDS ON` / `DEPENDENCIES` prefilter;
   - skip code (`freshnessMarkPosInCode`);
   - show the raw source whenever any selection touches the line;
   - replace the line body after the list marker with one widget whose `eq()` compares
     the serialised model, so unchanged chips don't flicker;
   - create the refresh `StateEffect` **lazily** (the surfaces test asserts exactly one
     eager `define`);
   - fan refreshes out through `scheduleLiveWidgetRefresh()`;
   - a mousedown on non-interactive parts places the cursor on the line, as freshness
     marks do.
4. **Reading view.** `registerMarkdownPostProcessor` finds list items whose text starts
   with the label:
   - turn each `a.internal-link` into a chip, keeping the anchor so native click and
     hover still work;
   - hide the separators, and add the label and the summary;
   - be idempotent, skip `code`/`pre`, and work in embeds and hover previews.
5. **Interaction.**
   - Click → `workspace.openLinkText(linktext, sourcePath, mod)`.
   - Hover → `workspace.trigger("hover-link", { source: "bob-dependency-chips", … })`.
   - Hover `×` → nav `api.removeDependency`; trailing `＋` → `api.openDependencyStage`.
     Both are hidden unless
     `app.plugins.plugins["bob-navigation-hotkeys"]?.api?.version >= 1`.
   - Chips are focusable, and Enter opens one.
6. **CSS (`styles.css`).**
   - Classes: `.bob-dep-row`, `.bob-dep-label`, `.bob-dep-chip` with state modifiers,
     `.bob-dep-chip-box[data-task]`, `.bob-dep-chip-note`, `.bob-dep-chip-remove`,
     `.bob-dep-add`, `.bob-dep-summary`.
   - Colours: `var(--task-status-next|in-progress|blocked|cancelled, <fallback>)`, with
     Obsidian surface and border variables for neutrals.
   - Size: inline-flex pills at `font-size: .8em; line-height: 1.1`, with the same
     vertical-align trick as `.bob-fresh-mark`, so the line box never grows.
   - Text: `nowrap` inside a chip; wrap between chips; ellipsis at about 40ch.
   - States: hover lift, `focus-visible` ring, `prefers-reduced-motion`.
   - Check light and dark themes and phone width.
7. **Toggle command.** "Toggle dependency chips": session-only, on by default, using a
   body class, like the freshness-mark toggle.
8. **Tests.** New `scripts/test-ledger-tools-dependency-chips.cjs`, modelled on
   `test-ledger-tools-freshness-mark-surfaces.cjs`. It covers:
   - the DP and DC vectors;
   - the decoration builder: reveal on the cursor line, visible ranges only, code
     skipped, `eq()` stability;
   - the post-processor on a stub DOM;
   - api feature detection with a stubbed nav api.
9. **Finish.** Bump the manifest minor version; update the README row and the api
   paragraph if anything is exported; then `npm test`, `npm run validate`, and deploy.

## Phase: compat — task-status-cycler and block-id-prompt

**Repo:** bob-plugins. Each plugin copies the small Depends-On recogniser (DP subset)
rather than importing it.

**task-status-cycler**

1. `retireClosedTaskReferencesInText` and `restoreReopenedTaskReferencesInText` skip
   Depends-On lines. This removes the hazard where an exactly struck plain link is
   restored as `![[…]]`.
2. Embedded-tree closes (`collectEmbeddedTranscludedTaskTargetsInListItemBlock`, used by
   `completeResolvedTranscludedTaskTargetTree`) ignore Depends-On lines.
3. Ctrl+Enter on a link on the line (`handleActiveTaskBlockLinkOpenDone`):
   - closes or reopens the **target** root-only;
   - runs the Blocked-dependent recovery from `finalizeClosedTasks`;
   - never strikes, restores, or embeds anything on the line.
4. Alt+] / Alt+[ (single and counted) on the line cycle the target of the link under the
   cursor; this is the plain-link analogue of `cycleResolvedTranscludedTaskLink`.
   - `getPlainBulletFormatToggle` must never apply to the line.
   - With the cursor not on a link, show the notice
     `⛓ Put the cursor on a dependency link to cycle it`.
5. Pin with tests that the dependency-id normaliser
   (`normalizeActiveEditorDependencyBlockIds`, `normalizeVaultFileDependencyBlockIds`,
   `reconcileRenamedDependencyIds`) never edits a Depends-On line, and that its
   target-id rewrites leave line-owning dependents consistent: R1 recomputes the same
   field.

**block-id-prompt**

6. Ctrl+Shift+Enter with the cursor on any link on the line (`startTaskLinkOpen`,
   `applyTaskLinkOpen`) is refused with a new `TASK_DEPENDENCY_LINE_NOTICE`
   (`⛓ Dependency link — edit it with Ctrl+Shift+P`) and never deletes a token. The
   legacy transclusion refusal stays.
7. Tests that `isDedicatedTaskLinkBullet`, `isSoleContentLinkBullet`, and
   `isDedicatedLinkBullet` reject the line.

**Finish.** Tests in `scripts/test-task-status-cycler.cjs` and
`scripts/test-block-id-prompt.cjs`; bump both manifests; update both README rows
(including the Task Status Cycler api paragraph if needed); deploy both. The known Alt+]
`finalizeClosedTasks` gap (bead `bob-cli-3k`) stays out of scope.

## Phase: nav-model — dependency model, single-transaction writer, and api v1

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/main.js`. Use `grep -a`; the file
is about 37k lines. Start from the dependency map in the report's F4–F10.

1. **Grammar.**
   - Replace the writer constants and regexes (`DEPENDENCY_NAVIGATION_LABEL`, `_EMOJI`,
     `_SEPARATOR`, `DEPENDENCY_NAVIGATION_BULLET_RE`, `DEPENDENCY_NAVIGATION_LINK_RE`)
     with the contract grammar: writer form, reader tolerance, and cross-note links. The
     old regex only matched same-note `[[#^id]]`.
   - Keep the legacy-child reader (`parseDependencyTransclusionBulletDetails` plus plain
     and struck sole links) for R8.
   - Export the new helpers on `module.exports.helpers` for tests and the migration
     script.
2. **Task-block helpers:**
   - find the owning task from any line in its block or its Depends-On line;
   - find the task's Depends-On line among its direct children;
   - the insertion point (first child, or after `❌ **CANCEL LOG**`);
   - the child indent (`getDependencyChildIndent`).
3. **Identity and link form.**
   - `canonicalDependencyLink(target, sourcePath, markdownFiles)` picks the shortest
     unambiguous form.
   - `dependencyTargetId(targetLine, path, blockId)` returns the existing valid
     `[id::]`, else `tryDependencyId`, else null.
4. **Pure planner.**
   `planDependencyEdit({ content, parentLine, parentPath, add, remove, files })`
   returns:
   - the dependent note's edits: create, append, remove, or delete the line; fold legacy
     children; the field per R1; status;
   - same-note target edits (`^id`, `[id::]`);
   - cross-note target preparations;
   - a summary for notices.

   Test it against the DW vectors.

5. **Writer.** `applyDependencyEdit(...)` follows the contract write order.
   - **Cross-note preparation:** the open editor via `applyEditorContentTransaction`,
     otherwise `vault.process` with a preimage check, as `writeLinkPickerNoteChange`
     does.
   - **Commit:** one `applyEditorContentTransaction` for the dependent's note.
   - **Add:**
     - `blockObsidianTaskCheckboxStatus` when any target is open;
     - commitment transfer through `getDependencyPromotionStatus` and
       `promoteObsidianTaskCheckboxStatus`.
   - **Remove (ADJ-8):**
     - compute recovery on the **post-edit** content: `buildScheduledRecoveryIndex` with
       the edited buffer overriding, then `getScheduledRecoveryMetadata` and
       `reconcileBlockedScheduledTaskLine`;
     - the dependent stays `[?]` while it has open prerequisites or a future schedule.
   - **Freshness:** stamp via `api.freshness.stampLine`.
   - **Notices:** per the contract.
6. **Rewire every existing dependency writer** to the planner and writer:
   - `chooseTaskDependency`;
   - `commitMarkedDependencies`, `executeDependencyBatch`,
     `reconcileDependencyNavigationBullets`;
   - `chooseCountedTaskDependency`, `applyCountedLocalTaskDependency`,
     `planCountedLocalTaskDependency`;
   - `confirmSingleBlockId`, `confirmBatchBlockId`, `confirmCountedDependencyBlockId`;
   - `setLocalTaskDependency`.

   Candidates stay current-note until `nav-stage`, but every write now produces the
   line, and nothing writes embeds.

7. **Recovery and rank edges.** `buildScheduledRecoveryIndex` and
   `parseRecoveryTransclusion` use line links plus R8 legacy children, the same as Rust.
   `#^ref` embeds are not edges.
8. **api v1.** Create the frozen `this.api` per the Design:
   - `openDependencyStage` opens today's stage for the owning task; `nav-stage` swaps in
     the vault-wide stage.
   - `removeDependency` uses the writer.
   - Neither ever throws.
9. **Tests.**
   - New `scripts/test-navigation-dependencies.cjs` covers:
     - the DP and DW vectors and the planner;
     - one undo group (`TransactionEditor.undoGroups`);
     - a failed cross-note preparation leaves the dependent untouched;
     - status effects: add blocks; commitment transfer; remove recovers, or stays
       Blocked with another open prerequisite or a future schedule;
     - legacy fold;
     - the api.
   - Update the existing grammar tests in `scripts/test-navigation-hotkeys.cjs` that
     assert embed output.
10. **Finish.** Bump the manifest, update the README row, and deploy. chips and compat
    are already deployed, so lines render and behave correctly from the first write.

## Phase: nav-stage — the vault-wide Ctrl+Shift+P Depends on stage

**Repo:** bob-plugins, navigation-hotkeys. Builds on `nav-model`.

1. **Pool.**
   - Source: `app.plugins.plugins["obsidian-tasks-plugin"].getTasks()` when `getState()`
     is `"Warm"`. Otherwise fall back to one scan per stage open, reusing
     `buildInteractiveScheduledRecoverySnapshot`'s reader.
   - Open buffers (`getOpenMarkdownBufferContents`) override the cache for their notes.
   - Apply the RESULTS eligibility and exclusions from the Design.
   - CURRENT resolves against everything.
   - Never read from disk per keystroke.
2. **Ranker.**
   - Port `capture_link_tasks.rs::rank` and pass the DK vectors.
   - Apply the canonical tie order, the BLOCKED section, the empty-query order, the cap
     of about 60 rendered rows ("type to search N more"), and match highlighting.
3. **Layout.**
   - Extend `BulletPropertyPickerModal`'s value stage, or add a sibling modal on
     `FilteredPickerModal`, reusing the `bob-cnp` classes plus new `bob-cnp-dep-*`
     classes in `styles.css`.
   - Show the header, subtitle, sections, and row anatomy from the Design.
   - Badges: `−`, `＋`, `＋ id`, `🔒 waits on N`, and disabled reasons
     (`would create a cycle`, `this task`, `path can't be an id`, `changed — reopen`).
   - Mark states: `＋ add` and `− remove`.
   - Footer hints: `↑↓ navigate · ⇥ mark · ↵ toggle · esc dismiss`, with ↵ reading
     `apply N` when there are marks.
   - Use status colours from the `--task-status-*` tokens.
4. **Keys and guards** per the Design.
   - The cycle check uses `task.id` / `task.dependsOn` from the pool (with buffer
     overrides) on the post-batch graph, and puts the path in a tooltip.
   - A stale target or dependent refuses and reopens fresh.
5. **`＋ id` flow.** The existing block-ID stage, pre-filled with
   `suggestBlockIdFromTask(text, <target note content>)`. Today's
   `blockIdExistsInContent` checks only the dependent's note; check the target's. A
   batch prompts one target at a time.
6. **Entry points.** Every row of the Design's entry-point table:
   - **Property row.** The `getBulletPropertyCurrentLabel` pill for `dependsOn` becomes
     `⛓ N · K open` / `⛓ none`.
   - **The line.** Route `openBulletPropertyPicker` for a Depends-On line _before_ the
     `parseLinkPickerTaskLink` link-mode check.
   - **Task Link mode.** Remove the `dependsOn` filter in
     `createLinkPickerPropertyItems` and the "Dependencies cannot be set through a Task
     Link" refusal; edit the linked task in its own note.
   - **Counted.** Counted sessions use the vault-wide pool.
   - **Palette command.** `edit-task-dependencies`, with no default hotkey.
   - **api.** `api.openDependencyStage` opens this stage.
   - **Refusals** for ambiguous cursors.
7. **Notices** per the Design.
8. **Tests.** Extend `scripts/test-navigation-dependencies.cjs`, or add
   `test-navigation-dependency-stage.cjs`. Use `createBulletPropertyPickerHarness` and
   `createLinkPickerHarness`, with a stubbed Tasks plugin. Cover:
   - eligibility and exclusions;
   - an unsaved buffer in another note;
   - DK parity, empty-query order, and sections;
   - guards: self, a cycle created by a batch, unencodable, stale;
   - `＋ id`;
   - Esc changes no bytes;
   - one undo group;
   - Task Link mode, line entry, counted;
   - the command is registered with no hotkey;
   - a performance check: filtering 1,000 synthetic tasks takes under 16 ms per
     keystroke.
9. **Finish.** Bump the manifest, update the README row, and deploy.

## Phase: nav-gestures — gesture cleanup, hand-edit mirror, and legacy writer removal

**Repo:** bob-plugins, navigation-hotkeys. Builds on `nav-stage`.

1. **`!` / `N!`.** Remove the dependency coupling in
   `applyDependencyAwareTransclusionChanges`:
   - no `dependsOn` edits and no status changes;
   - `!` only toggles transclusion;
   - on a Depends-On line, refuse with
     `⛓ Dependencies use plain links — edit them with Ctrl+Shift+P`.

   Toggling a legacy child stays a plain toggle; R8's plain-link rule keeps the
   dependency.

2. **Ctrl+D on the Depends on row** (`deleteBulletPropertyValue`,
   `deleteCountedBulletPropertyValue`) removes the line, the field, and any legacy
   children in one transaction, with ADJ-8 recovery.
3. **Hand-edit mirror.** A CM6 update listener tracks Depends-On lines touched by a doc
   change, including a line deleted outright.
   - **When it runs:** once the cursor has left that line and about 400 ms have passed.
     Never during IME composition or while a modal is open.
   - **What it does:** applies R1/R9 to the owning task in one transaction: the field,
     same-note target `[id::]`, and the same status effects as the stage except
     commitment transfer.
   - **What it leaves alone:** cross-note targets with no `[id::]` (the hooks handle
     them) and malformed lines. It does not stamp freshness.
   - **Tests** cover a half-typed `[[`, deleting the whole line, adding by hand, and
     removing the last open prerequisite.
4. **Delete the legacy writers and tools:**
   - the `consolidate-dependency-navigation-links` command and
     `consolidateDependencyNavigationLinks`;
   - `formatDependencyNavigationBulletWithMarker` / `formatDependencyNavigationBullet`;
   - `planDependencyNavigationBulletSync`, `applyDependencyNavigationBulletSyncPlan`,
     `applyDependencyNavigationPlanToLines`, `transformDependencyBulletsInContent`;
   - the dead helpers the survey found (`normalizeDependencyNavigationBlockIds`,
     `formatDependencyNavigationBulletFromDetails`, the
     `planDependencyNavigationBullet*` insertion/removal and label-normalisation
     helpers);
   - `scripts/migrate-dependency-bullets.mjs`,
     `scripts/migrate-task-dependency-identities.mjs`, and
     `scripts/test-task-dependency-identity-migration.cjs` (one-shot migrations that
     emit embeds), with their `package.json` and README entries.

   Keep the legacy **readers**.

5. **Ctrl+Shift+M.** Add tests that a moved task keeps its Depends-On line, and that
   `rewriteTaskMoveBlockLinks` gives same-note links to tasks left behind the source
   path and rewrites links to moved tasks. Fix any gap.
6. **README.** Rewrite the navigation-hotkeys row to cover:
   - the Depends on stage and its entry points;
   - the line;
   - the mirror;
   - `!` as a pure toggle;
   - Ctrl+D;
   - api v1.

   Remove the "Dependency identity migration" section.

7. **Finish.** Bump the manifest, run the tests, and deploy.

## Phase: fleet-rollout — install bob and sync plugins on every machine

1. **This host.**
   - From an up-to-date bob-cli master, run `cargo install --path . --locked --force`.
   - Pull the opened bob-plugins checkout to origin master, then run
     `bob plugins sync -n -r "<opened bob-plugins path>"`.
   - Check with `bob plugins list`.
2. **MacBook** (best effort; `ssh -o ConnectTimeout=15 mac`, retrying for about 10
   minutes).
   - Find how `bob` was installed (`~/.cargo/.crates.toml` / `.crates2.json`):
     - **a clean `path+file://` checkout on `master`:** `git pull --ff-only`, then
       `cargo install --path . --locked --force`;
     - **a `git+` install:** reinstall from the same source with `--locked --force`;
     - **anything else:** stop and record the exact steps.
   - Sync the plugins on the Mac the same way, from its bob-plugins checkout.
   - Verify with
     `ssh mac '~/.cargo/bin/bob task-status-hooks --dry-run -f json' | jq 'has("dependency_projection_updates")'`.
3. **Other hosts.** apollo or athena, whichever this isn't: install `bob` the same way
   if it is installed there. They run no hooks.
4. **Record** in the phase notes, per machine: bob commit, hooks capability true/false,
   and plugin versions. List what is left for Bryan, including reloading the plugins in
   each running Obsidian.

## Phase: vault-migrate — migrate the vault to Depends-On lines

**Repo:** bob-plugins (the script), then the vault.

1. **Preflight.**
   - Re-verify the Mac's hooks capability over ssh.
   - If it can't be verified, ask Bryan with `/sase_questions` whether to migrate now or
     wait. If Bryan chooses to migrate now, the Mac's old hooks lose dependency
     promotion for migrated parents until it updates; Blocked is unaffected.
   - This host must pass the capability check.
2. **Script `scripts/migrate-dependency-lines.mjs`.** Model it on the retired
   identity-migration script, using nav's exported `helpers` for the grammar, formatter,
   and planner.
   - **Exports:** `parseArgs`, a pure `planMigration(files)`, and `runMigration`.
   - **CLI:** `--vault DIR` (default `~/bob`) and `--write`. Dry-run is the default.
   - **Exclusions:** the same as the hooks (`.git`, `.obsidian`, `_conflicts`,
     `_generated`, `_templates`, `done/`, dot-dirs).
   - **Convert,** for every `#task` dependent, open or closed:
     - R8 legacy children (live embeds, struck, plain) and any `🔗` / `DEPENDENCIES` row
       become one canonical line: existing row links first, then legacy children in
       document order;
     - for open dependents, the field is set per R1, with R2 adoption for field-only
       dependents.
   - **Leave alone,** reported:
     - `#^ref` and other unmanaged embeds;
     - Pomodoro ledger embeds;
     - Work Logs and fenced examples;
     - any legacy child that has its own sub-bullets (a conflict, never flattened).
   - **Preserve** indentation, CRLF, and final-newline state.
   - **Refuse `--write`** on preflight conflicts.
   - **Tests:** `scripts/test-dependency-line-migration.cjs` (in-memory files) covers
     the cases above, plus idempotence (re-running on `nextContent` changes nothing).
   - Commit and push the script before running it.
3. **Rehearse on a copy.**
   - Copy the vault to a temp directory, excluding `.git`.
   - Snapshot `bob task-status-hooks --dry-run -f json` with `BOB_DIR` set to the copy.
   - Run the script `--write` on the copy, then the dry run again. Require:
     - zero Blocked flips;
     - `dependency_projection_updates` and `adopted_dependency_lines` empty, or each one
       explained;
     - `legacy_dependency_children` = 0;
     - rank edges that differ only for `#^ref` embeds;
     - a second script run changes nothing.
   - Paste the counts and the changed-file list (the report expects about 18 notes).
4. **Live run.**
   - Inspect `~/bob` `git status`.
   - Run the script `--write` on `~/bob`, then repeat the step 3 checks on the real
     vault with dry runs.
   - Update the comment in `.obsidian/snippets/dataview-properties.css`: the fields
     derive from "the block id and the task's Depends-On line", not transcluded links.
   - Commit only the migration's files, following `~/bob/AGENTS.md`.
   - Run `bob vault-sync run` and `bob vault-sync status`.
5. **Spot-check.** Open two migrated notes in Obsidian or render them with
   `bob query --tasks` to confirm the fields and Blocked state. Paste one before/after
   excerpt.

## Phase: publish — glossary, decision record, and final docs

Use `/sase_memory_write` (Edit And Republish), then `sase memory init`. Mirror the
existing strands' frontmatter and section shape.

1. **New glossary strand** `sase/memory/glossary/task-dependency-link.md`.
   - **Keyword:** `Task Dependency Link`.
   - **Aliases:** `task dep link`, `dep link`.
   - **Body:** the report's draft, adjusted to what shipped. For example:

     > A plain (never transcluded) [[task-link]] to a prerequisite of another Obsidian
     > task. A task's dep links all sit on its Depends-On line, its first direct child:
     > `⛓️ **DEPENDS ON:** [[#^a]] • [[note#^b]]`. That line is the source of truth: Bob
     > derives the task's `[dependsOn::]` field and each target's `[id::]` from it, and
     > the task stays Blocked `[?]` while any target is open. Linking the dependent
     > under today's Pomodoros raises its open prerequisites to Next. Add or remove dep
     > links with Ctrl+Shift+P → Depends on, which fuzzy-searches open tasks across the
     > vault, or by editing the line. A dep link is a prerequisite, not a sub-task:
     > closing the dependent never closes its targets, and dep links never count as
     > Pomodoro Task Links.

   - Syntax details stay in `docs/task-dependencies.md`.

2. **Amend `glossary/task-link.md`.** Replace the "transcluded task links … sub-tasks
   (i.e. dependencies)" sentence with "Task Links on a task's Depends-On line are
   [[task-dependency-link]]s." Leave the Pomodoro sentence as it is.
3. **New decision record** `sase/memory/decisions/task-deps-are-depends-on-links.md`:
   "Task Dependencies Are Links On One Depends-On Line".
   - **Frontmatter:** aliases (`task dependency links`, `depends-on line`, `dep links`);
     a one-sentence summary; `metadata.status: accepted`; `decided: <today>`.
   - **Applies to:** bob-cli, bob-plugins, vault.
   - **Claim:** the grammar, authority, and semantics in one paragraph.
   - **Why:** the report's "Is this a good idea".
   - **Rejected:**
     - links only;
     - fields only with a virtual line;
     - a field-authoritative generated line;
     - one bullet per dependency;
     - links on the task line;
     - the Tasks modal as the editor;
     - `!` as the gesture;
     - stored aliases and strikes.
   - **Cost:**
     - four parsers kept in sync by vectors;
     - the projection exists at all;
     - deleting the whole line outside Obsidian is re-adopted;
     - Tasks-modal dependency edits to a task that has a line are dropped;
     - `#^ref` embeds no longer promote.
   - **Reopens when:**
     - R1/R2 warnings appear on most hooks runs for two weeks;
     - the Tasks modal becomes a real editing surface;
     - Tasks' `is blocked` stops being load-bearing.
   - **Evidence:** the report, this epic's plan ref (from your bead), and the key
     commits.
4. **Mark `decisions/task-status-is-derived`.**
   - Keep `metadata.status: superseded-in-part`.
   - Make `superseded_by` a list that adds `decisions/task-deps-are-depends-on-links`.
   - Add a back-link sentence: the transcluded-dependency path and the `!` removal
     rationale are retired by [[decisions/task-deps-are-depends-on-links]]; the Blocked
     rule stands.
   - Don't otherwise edit the accepted body.
5. **Republish.** Run `sase memory init`. Confirm that the CLAUDE.md glossary roster
   lists "Task Dependency Link (task dep link; dep link)" and that the decisions roster
   shows the new record.
6. **Docs sweep.**
   - In bob-cli, run `rg -n -i "transclu"` over `docs/` and `README.md` and fix any
     stale _dependency_ wording (Pomodoro embeds are still valid).
   - Check `docs/plan.md`, `docs/capture.md`, `docs/projects.md`, and
     `docs/freshness.md`.
   - Check the bob-plugins README dependency text.
7. **`PROPOSED FOLLOW-UP:` notes on your bead:**
   - remove R8 legacy-child reading in Rust and nav once the hooks report
     `legacy_dependency_children: 0` across a week of runs;
   - a reverse "Blocks…" stage (Q4);
   - a `⛓ 1/2` task-line mini-badge;
   - a v2 path codec when a note needs one (ADJ-9).
8. **Bryan's checklist** in your final response:
   - reload the four plugins in each running Obsidian;
   - pilot: add prerequisites from two projects, remove a completed one, follow a chip,
     and edit while another note has unsaved changes;
   - optionally bind a chord to **Edit task dependencies**.

## Deliberately not doing

- **Stored data:** written aliases or `~~strikes~~` on the line; removing the
  `[dependsOn::]` / `[id::]` fields (ADJ-1).
- **Planning features:** OR-groups, lags, or a graph canvas.
- **Deferred editing surfaces:** a reverse "Blocks…" stage (Q4); two-way Tasks-modal
  editing (Q5).
- **Deferred display and encoding:** the task-line mini-badge; the path codec (ADJ-9).
- **Capture:** a capture or Bob Mac Capture dependency grammar. Any future one lands in
  bob-cli first.
- **Bugs left alone:** the cycler's Alt+] `finalizeClosedTasks` gap (`bob-cli-3k`).

## Risks

| Risk                                                                             | Mitigation                                                                                                                                 |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Four parsers drift (Rust, nav, ledger-tools, cycler/block-id-prompt recognisers) | One contract doc and shared numbered vectors, copied into every test suite                                                                 |
| The projection is a second representation                                        | One authority; writers update both in one transaction; an idempotent reconciler with reported counts; the decision record's reopen trigger |
| The hooks now write `[id::]` and lines in other notes every 15 minutes           | The guarded snapshot-and-retry pipeline, quiet interval, recovery copies, and a read-only dry-run verification before rollout              |
| Hand edits flip Blocked mid-typing                                               | The mirror waits for the cursor to leave the line; malformed lines are skipped by every reader; the hooks' quiet interval                  |
| A third format swing back to embeds                                              | Chips ship before any writer emits lines, and Ctrl+Enter / Alt+] act on targets in place                                                   |
| Chips destabilise the editor (F11's class of bugs)                               | Fixed-height inline widgets, visible ranges only, raw text on the cursor line, Vim `j`/`k` tests                                           |
| A mixed fleet (the Mac's old hooks)                                              | Legacy readers for a release; fields keep Blocked correct; `fleet-rollout` and the migration preflight gate on the Mac                     |
