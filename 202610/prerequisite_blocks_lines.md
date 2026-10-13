---
tier: epic
title: '🧱 BLOCKS lines: show on every prerequisite what waits on it'
goal: 'Every open task that other open tasks depend on carries one machine-owned `🧱
  **BLOCKS:**` child line that mirrors its dependents'' Depends-On links. The line
  is kept exact by `bob task reconcile`, updated immediately by every dependency writer
  (Ctrl+Shift+P, chip ×, hand edits and undo, capture `&`, capture dependency drops),
  rendered in Obsidian as chips that show what finishing the task would free, and
  never read back as dependency input.

  '
decisions:
  glyph:
    ask: Which glyph should the generated BLOCKS line use?
    choices:
      brick: '🧱 BLOCKS: a distinct mark that makes bottlenecks visible while scrolling'
      chain: '⛓️ BLOCKS: pairs with ⛓️ DEPENDS ON, but you must read the label for
        direction'
    default: brick
    why: One code point, unused by any plugin, and unmistakable beside the Depends-On
      line
    answer: brick
  memory_decision_record:
    ask: Should rollout add a decision record for BLOCKS lines as a never-read-back
      mirror?
    memory:
    - decisions:blocks-lines-mirror-depends-on
    default: false
    answer: false
  memory_glossary:
    ask: Should rollout add one sentence about the BLOCKS mirror to the Task Dependency
      Link term?
    memory:
    - glossary:task-dependency-link
    default: false
    answer: true
phases:
- id: contract
  title: The BLOCKS-line contract, managed-line registry, and vectors
  depends_on: []
  size: medium
  description: 'contract: write docs/task-dependencies.md §14 (grammar, membership,
    minting, order, position, authority, writers, gestures, chips, kill switch, outputs),
    grow §2.4 into the managed-child-line registry, add the BK/BR/BW/BC/MC vectors,
    and align projects.md, capture.md, and getting-started.md.'
- id: cli-core
  title: Rust grammar, shared planner, guards, projects-sync anchor, and config
  depends_on:
  - contract
  size: medium
  description: 'cli-core: add task_dependencies::blocks (parser, formatter, canonical
    sort, slot, and the one shared line planner), guard every Rust Task Link and tree-close
    scanner against both managed dependency lines, anchor Sub-projects after BLOCKS,
    and load dependencies.blocks_lines.'
- id: plugin-guards
  title: Recognisers and gesture guards in nav, cycler, and block-id-prompt
  depends_on:
  - contract
  size: medium
  description: 'plugin-guards: teach bob-navigation-hotkeys, task-status-cycler, and
    block-id-prompt to recognise and ignore BLOCKS lines (Pomodoro scanners, strike/restore,
    tree close, conversion, the hand-edit mirror) and route or refuse gestures on
    them, then deploy on this host.'
- id: chips
  title: bob-ledger-tools BLOCKS chips with the 🔑 last-blocker badge
  depends_on:
  - contract
  size: medium
  description: 'chips: render BLOCKS lines in Live Preview and Reading view from a
    live reverse index of the Tasks memo, with the 🔑 badge, `frees N` summaries, past-tense
    frozen rows, and stale-entry styling; guard the plan-budget link counter; deploy
    on this host.'
- id: cli-reconcile
  title: bob task reconcile owns the BLOCKS projection
  depends_on:
  - cli-core
  - plugin-guards
  - chips
  size: medium
  description: 'cli-reconcile: export the accepted explicit edges from R1–R10, add
    the BLOCKS pass (mint, insert, rewrite, canonicalise, deduplicate, delete, warn,
    never prune on uncertainty, honour the kill switch), report it additively, and
    dry-run it against the live vault.'
- id: cli-capture
  title: Capture `&` and dependency drops keep the mirror current
  depends_on:
  - cli-core
  - cli-reconcile
  size: medium
  description: 'cli-capture: stage each added prerequisite''s BLOCKS entry (and the
    dependent''s minted id) in the same capture batch, remove entries on dependency
    drops, report rows additively, and keep Bob Mac Capture working without Swift
    changes.'
- id: nav-writer
  title: Nav planner seam, minting, and cross-note follow-ups
  depends_on:
  - plugin-guards
  - cli-reconcile
  size: medium
  description: 'nav-writer: add the JS twin of the shared planner and the SB mint,
    make planDependencyEdit emit same-note BLOCKS edits plus cross-note follow-ups,
    add the preimage-checked follow-up writer, and wire the shared writer and the
    single same-note path.'
- id: nav-paths
  title: Every remaining nav writer, the hand-edit mirror, and undo
  depends_on:
  - nav-writer
  size: medium
  description: 'nav-paths: route counted, marked, and batch Depends-On writers, counted
    Ctrl+D, the hand-edit mirror (including undo), project conversion, and Ctrl+Shift+M
    through the BLOCKS seam, with a test for every path.'
- id: rollout
  title: Install, verify, reach the MacBook, record
  depends_on:
  - cli-capture
  - nav-paths
  size: small
  description: 'rollout: install bob on this host, review the live dry-run, verify
    the pilot script on a vault copy, sync plugins, update the MacBook on a best-effort
    basis, apply the accepted memory decisions, and record follow-ups.'
proposed_by: bbugyi200.athena.research.4d.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 11:09:48
status: wip
bead_id: bob-cli-7e
---

- **PROMPT:** [prompts/202610/prerequisite_blocks_lines.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/prerequisite_blocks_lines.md)
- **BEAD:** [bob-cli-7e](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7e/README.md)

# Plan: 🧱 BLOCKS lines, a generated mirror of Depends-On on every prerequisite

## Context

Bryan's request (2026-10-13):

> I would like to start automatically adding a new sub-bullet to tasks that other tasks
> depend on to indicate that the task blocks one or more other tasks. […] We already use
> a "Depends On" sub-bullet on the dependent tasks. This should happen automatically
> when dependencies are added to tasks (via the `<ctrl+shift+p>` keymap, for example).
> The `bob task reconcile` command should ensure the appropriate sub-bullets all exist
> and are consistent. […] I agree with all of the requirements recommended in that
> research file […]. I want you to lead the design on this one. Make sure you design
> this feature so it is intuitive, reliable, and (last but not least) beautiful!

**Research.** `research:202610/prerequisite_blocks_line/prerequisite_blocks_line.md`
(read it with `sase artifact read`) consolidates five independent reports. Bryan accepts
all of its requirement adjustments, RA1–RA12. This plan adopts them as written and
settles the report's four open questions:

1. **Glyph.** `🧱` by default, embedded as the `glyph` decision.
2. **Closed prerequisites.** Their lines freeze as history (RA3).
3. **v1 chip actions.** Navigation only (RA12).
4. **Phase-2 key.** Deferred to bob-cli-3p, the Reverse Blocks… stage.

The plan also covers one writer the research predates: capture's dependency drop
(bob-cli-7b, `docs/task-dependencies.md` §13.5.1). RA4 says every edge removal must
update the mirror, and a drop is an edge removal.

### What exists now

Verified at bob-cli `c80cbde` (origin/master) and bob-plugins `76924cc`. Line numbers
are approximate, so search by name.

**bob-cli (Rust)**

- **Depends-On grammar.** `src/native/task_dependencies/` contains:
  - `parse.rs`: `parse_dependency_line`, `is_dependency_line`, `dependency_child_of`;
  - `format.rs`: `format_dependency_line`, `canonical_link`, `child_indent_for_parent`,
    `note_line_ending`;
  - `mod.rs`: the shared link scanner (`block_link_spans`, `raw_wikilink_spans`) and
    `dependency_id`.
  - The Depends-On parser accepts only the labels `DEPENDS ON` / `DEPENDENCIES`, so it
    already returns not-a-line for any `**BLOCKS:**` line.
- **Reconcile.** `src/native/task_status_hooks/sync.rs::sync_task_statuses`:
  - `reconcile_dependencies` (`reconcile.rs`) runs R1–R10, then re-parses touched files
    (≈L430–443).
  - Next come `task_blocks`, `dependency_edges`, the status loop, `compose_outputs`
    (structural files plus `pre_reconcile` originals), consumption, and
    `apply_guarded_outputs` (quiet interval, preimages, recovery snapshots).
  - `ReconcileWorker::record_graph` holds the explicit per-dependent edge set, but:
    - it returns early for dependents without a `^block-id`;
    - `add_resident_implicit_edges` later mixes implicit resident → `^prj` edges into
      the same graph.
  - `dependency_edges` (`references.rs`) also skips id-less dependents and adds the
    implicit edges.
  - Neither one is the BLOCKS edge set.
- **Minting.** `capture_block_ids::mint_block_id(description, used)` is the SB policy
  (§11.7) used by successor offers.
- **Sub-projects.** `projects/sync.rs` has `subprojects_anchor_child`,
  `subprojects_subtree_end`, and `subprojects_marker_precedes_depends_on`. The marker is
  anchored after the Depends-On subtree, or after a first-child Cancel Log.
- **Guards that skip Depends-On lines today:**
  - `capture_completable_tasks.rs` (≈L329);
  - `task_complete/tree.rs` (≈L192, tree close);
  - `projects/sync.rs::is_depends_on_child`.
- **Scanners with no managed-line guard.**
  `task_status_hooks/pomodoro.rs::scan_pomodoros` takes every block link on every bullet
  under a Pomodoro entry (via `block_link_occurrences`) as a Task Link, which feeds
  promotion, consumption, and successor anchors. This is a latent bug for Depends-On
  lines too. Other Pomodoro-child link scanners use the shared scanner and need the same
  audit: `capture_pomodoro_close/*`, `capture_pomodoros_agenda.rs`,
  `capture_pomodoro_start.rs`, and `plan_budget`.
- **Capture `&`.** `capture/dependencies.rs` holds `plan_existing_task_dependencies`,
  `plan_new_task_dependencies`, and `plan_dependency_only_item`.
  - Prerequisites resolve first: `project_prerequisite_id` stamps a target's `[id::]`
    through `CaptureBatchPlanner::stage`, and each target is reported as a
    `dependency_target` task block through `TargetTouch`.
  - Bob Mac Capture's `&` preview reuses those final task-block cards, and its Swift
    decoders are keyed, so extra JSON keys are ignored.
- **Capture dependency drop.** `capture/dependency_drop.rs`: "prerequisite tasks are
  never edited" (§13.5.1).
- **Config.** `config/plan.rs` holds the `plan.link_unblocked` kill switch: default
  `true`, invalid values fall back to `true` and warn.

**bob-plugins**

- **bob-navigation-hotkeys** (2.32.0). Every Depends-On write goes through the pure
  planner `planDependencyEdit` (`080-dependency-identity.js`, 999 lines). It returns
  `nextContent`, same-note `targetEdits`, cross-note `preparations`, and `status`.
  - **Shared writer:** `applyDependencyEdit` (`580`). It is used by stage CURRENT-row
    removal, cross-note choose, api v3 `removeDependency`, the chip ×, Ctrl+D on the
    Depends on row, and the hand-edit mirror's `touch`.
  - **Paths that call the planner directly and run their own transaction:**
    `setLocalTaskDependencyLink` (`600`), counted paths (`420`, `570`/`270`), marked
    batches (`430`, `440`), counted Ctrl+D
    (`600 deleteCountedDependencyLinesAndFields`), and hand-edit clears
    (`600 applyDependencyHandEditClear`). Project conversion (`665`) also calls the
    planner directly.
  - **Cross-note target writes:** `prepareDependencyTargetNote` (`590`) only replaces
    one existing line, so it cannot insert or delete.
  - **Hand-edit mirror:** `600` and `070 planDependencyHandEditMirror`. It does not diff
    links; `touch` passes empty add/remove lists.
  - **Config:** nav reads `~/.config/bob/config.yml` only for `properties`
    (`020 loadBulletPropertyConfig`).
  - **Size limits:** fragments must stay at or below 1000 lines. Fragments 080 (999),
    590 (997), 570 (996), and 580 are full. Slots 081–089 and 591–599 are free.
  - **Notice builders:** `216`, including the `· Pomodoro links not removed` suffix
    pattern.
- **bob-ledger-tools** (1.49.1). The Depends-On chips are built from:
  - parser `140 parseDependencyLine`;
  - model `140 dependencyChipModel`;
  - element `150`;
  - widget `160`;
  - Live Preview `270 createDependencyChipExtension`;
  - Reading view `280 renderDependencyChipsIn`;
  - toggle `toggle-dependency-chips`;
  - CSS `styles.css` `bob-dep-*`;
  - data from `270 dependencyTasksIndex` on `freshnessEnsureMemo()`.

  The incoming-row copy, aria text, and × / ＋ handlers are hard-coded in 140, 150,
  and 280. There is no reverse index. `050 planBlockLinks` counts every block link on
  every indented line under an open entry, so it has no managed-line guard.

- **task-status-cycler** (1.37.0):
  - `050 isTaskDependencyLine`;
  - strike/restore `080 retireClosedTaskReferencesInText` / `restoreReopened…`;
  - tree close `050 collectEmbeddedTranscludedTaskTargetsInListItemBlock`;
  - Ctrl+Enter root-only `160` (`onDependencyLine`);
  - Alt+]/Alt+[ `190 isActiveTaskDependencyLine`;
  - unguarded `060 classifyPomodoroSubBullets`;
  - `086 mintBlockId`, the only JS port of the SB mint.
- **block-id-prompt** (1.30.0):
  - Ctrl+Shift+Enter guards in `120 startTaskLinkOpen` and `130 applyTaskLinkOpen`;
  - `060 isTaskDependencyLine`;
  - unguarded `060 planPomodoroLinkCleanupForRanges` and
    `075 collectPomodoroEntryBlockLinks`.

**Vault** (research, read-only query on 2026-10-13):

- 15 open tasks would get a line, covering 21 open→open edges (16 same-note, 5
  cross-note).
- Maximum fan-in is 4.
- 6 dependents need a minted `^block-id`.
- All 20 open dependents are `[?]`.
- One `^prj` is a dependent.

**Related work:**

- **bob-cli-3p** (Reverse Blocks… stage) is this feature's phase 2.
- **bob-cli-3q** (task-line ⛓ badge) and **bob-cli-3o** (retiring R8 legacy readers)
  touch the same model.
- **bob-cli-7c** (Successor Offers) is in progress in Rust close paths, capture,
  task-status-cycler, nav, and `docs/task-dependencies.md` §12. Rebase carefully and
  merge both sides of `package.json`, plugin `README.md` rows, and manifests.

## Design

### What Bryan sees

In the Markdown, readable in vim, git, and by agents:

```markdown
- [?] #task Did enrolling for unemployment work! [id:: cash__unemployment] [scheduled::
  2026-10-14] ^unemployment
  - 🧱 **BLOCKS:** [[#^audit-subscriptions]] • [[#^call-north-face]] •
    [[#^get-access-money]] • [[#^save-verizon-router-pwd]]
  - Kelly just needs 2 pay stubs and/or my W2!
```

A chain node reads upstream, then downstream:

```markdown
- [?] #task Make appt w/ Rahway Hospital for CT scan! ^rahway
  - ⛓️ **DEPENDS ON:** [[#^hospital-swarm]]
  - 🧱 **BLOCKS:** [[#^appt-7d-after]]
  - 🗓️ **SCHEDULE LOG**
```

In Obsidian, the line renders as chips from the same family as the Depends-On chips. A
🔑 marks each dependent for which this task is the last thing in the way:

```text
[?] Did enrolling for unemployment work!
    🧱 blocks  ( ? Audit monthly subscriptions 🔑 )  ( ? Dispute North Face… 🔑 )  ( ? Get access… 🔑 )  ( ? Save Verizon… 🔑 )   frees 4
[ ] The first SASE launch post: introduction
    🧱 blocks  ( ? Review outline and intro proposal )                                       frees 0 of 1
[x] (a closed prerequisite: the frozen line)
    🧱 was blocking  ( ✓ … )  ( ? … )                                                         ✓ freed 2
```

> [!decision] glyph = chain Every `🧱` in this plan becomes `⛓️` (U+26D3 U+FE0F) in
> writer output, chip labels, and vectors. The reader tolerance list is identical in
> both branches.

### Rule 1: one source of truth

`⛓️ **DEPENDS ON:**` on the dependent remains the only place a dependency is authored.
The BLOCKS line is a projection of the accepted explicit edges into each prerequisite.

- No engine reads a BLOCKS line as dependency input: not Blocked derivation, promotion,
  consumption, successor offers, fields, R2 adoption, or R8 legacy children.
- A hand edit of a BLOCKS line never adds or removes an edge. Reconcile restores the
  canonical line.
- **Key invariant (tested):** deleting every BLOCKS line in the vault changes nothing
  about blocking, promotion, scheduling, successor offers, fields, or authored edges.
  The next reconcile rebuilds identical bytes, and a second run makes zero changes.

### Rule 2: grammar

```text
<indent>- 🧱 **BLOCKS:** <link> • <link> • …
```

- **Writer form.** `🧱` (U+1F9F1, one code point, no VS16), one space, `**BLOCKS:**`,
  one space, then links joined by `•` (U+2022 with one space on each side).
  - No aliases, embeds, strikes, or duplicate entries.
  - The line is deleted when it would be empty; there is no placeholder.
- **Link form.** Same as §3: the shortest unambiguous form from the owner's note
  (`canonical_link` / nav `canonicalDependencyLink`), so `[[#^id]]`, then
  `[[basename#^id]]`, then `[[dir/note#^id]]`.
- **Reader tolerance.** All of these canonicalise:
  - the emoji is missing, or is `🚧`, `⛓️`, or `⛓`;
  - the label is `BLOCKING`;
  - the separator is `·`, `,`, or whitespace;
  - a link is aliased, struck, or embedded;
  - the order is not canonical;
  - an entry is duplicated;
  - the line is a direct child in any slot.
- **Not a line:**
  - a lowercase label;
  - the label `BLOCKED BY` (it names the opposite direction);
  - a fenced, nested, blockquoted, or marker-less line.
- **Parse algorithm.** Use the §2.3 algorithm, keyed on its own label: link scanner
  first, never split on separators, and only a direct child of a `#task` counts. The
  Depends-On and BLOCKS parsers each return not-a-line for the other's line.
- **Malformed** means prose after the links, a half-typed link, a bare note link, a
  heading link, or separators only. A malformed line is left alone and warns
  `malformed_blocks_line`. Writers insert no second line beside it.

### Rule 3: membership

```text
open(t)    = status type Todo | In Progress | On Hold (includes [?])            (§4.1)
edges      = the accepted explicit edges R1–R10 project: resolved scanned-task targets
             of the line, plus R8 legacy-child targets during the legacy window, plus
             R2-adopted links. Excluded: R4 unresolved, R5 non-task, R6 self, archive
             (done/) targets, previous-daily or unencodable targets kept verbatim, and
             ^prj internal or descendant targets. Implicit resident → ^prj edges and
             transitive chains are never edges here.
blocks(T)  = { D : (D → T) ∈ edges ∧ open(D) ∧ D not in the previous daily note }
T owns a line ⇔ open(T) ∧ blocks(T) ≠ ∅ ∧ T is writable (not the previous daily note)
closed T   → its line is never touched: frozen history
```

- A `^prj` dependent is listed once, as its own `^prj` line. Its residents are never
  listed.
- A cycle A ↔ B lists each task on the other's line. R7 already warns about the cycle.
- `#hide`, `ref/`, inbox, and today's daily-note tasks are in scope on both sides.

### Rule 4: identity and minting

- A dependent with no `^block-id` gets one appended, minted with
  `mint_block_id(description, used)`. This is the SB pipeline of §11.7: the body after
  the status box, cleaned with the `#task` global filter. `used` is every block id in
  the dependent's note plus this gesture's or run's reservations.
- The mint happens once per dependent. Existing ids stay stable.
- Minting never adds `[id::]`, because `[id::]` means "is a prerequisite".
- BLOCKS writes never stamp `[fresh::]`, never change a status, lane, tag, or field, and
  never touch the owner's task line. They write only the child line, plus the
  dependent's minted `^id`.
- **Never emit a link to an id that will not exist on disk after the write.**
  - nav mints inside the dependent's own transaction, before any cross-note follow-up.
  - capture mints inside its atomic staged batch.
  - Reconcile skips listing a dependent whose mint lands in a note that the quiet
    interval will defer this run (same predicate as the guarded write). The next run
    lists it.
- Dry-run and Escape mint nothing.

### Rule 5: order and position

- **Order.** Sort entries by canonical link text after ASCII case-folding, comparing by
  Unicode code point. Raw code-point order breaks ties.
  - Same-note `[[#^…]]` entries therefore come first, then cross-note entries grouped by
    note.
  - JS must compare code points, never UTF-16 units, so nav, ledger, and Rust emit
    identical bytes.
  - Because the order is a projection, it is independent of document order, which
    reconcile regroups every run.
- **Position.** Insert after the Depends-On subtree when there is one. Otherwise use the
  Depends-On slot: the first direct child, or the second after a first-child
  `❌ **CANCEL LOG**`.

  ```text
  - [ ] #task T ^t
    - ❌ **CANCEL LOG**        (if present; stays first)
    - ⛓️ **DEPENDS ON:** …     upstream: what T waits on
    - 🧱 **BLOCKS:** …         downstream: what waits on T
    - 🧩 **Sub-projects:** …   structure (^prj only)
    - user children, 🗓️ SCHEDULE LOG, Work Log …
  ```

  - Reuse the task's existing child indent; otherwise use the parent indent plus one
    tab.
  - Preserve CRLF and the final-newline state.
  - **Never move an existing BLOCKS line.** Rewrite it in place and delete duplicates
    after the first.
  - A task that later gains a Depends-On line gets it in the first slot, so the order
    still reads upstream, then downstream.

- **`bob projects sync`.** Sub-projects anchors after the Depends-On and BLOCKS
  subtrees. Its move rule becomes "a marker before a direct Depends-On or BLOCKS child
  moves after them".

### Rule 6: authority (the reconcile pass)

Reconcile is the authority. Gestures are early courtesy writes, and closes, vim edits,
agent edits, and moves are repaired only by reconcile.

1. **Export the edge set.** `ReconcileWorker` records each open dependent's accepted
   explicit targets (Rule 3) in `ReconcileOutcome`.
   - Record at the same points as `record_graph`, but before its block-id early return,
     so id-less dependents are included.
   - Record before `add_resident_implicit_edges`.
   - Also record the dependents whose Depends-On line is malformed (R10), as the
     uncertain set.
   - Key dependents by `(file_index, task_index)`. Reconcile never adds or removes task
     lines, so those keys survive its edits; debug-assert this after the re-parse.
2. **New pass `reconcile_blocks_lines`.** It runs right after the R1–R10 re-parse loop
   and before `task_blocks` / `dependency_edges`, over the re-parsed text. It is a
   separate pass, so a task that gains an R2 Depends-On line and a BLOCKS line in one
   run cannot have two inserts at one index flip their order.
3. **Owners to check.** These are every T in `blocks(T)`, plus every open, writable task
   that already has a BLOCKS child. To find the latter, byte-prefilter notes for
   `BLOCKS` / `BLOCKING`. Never walk every task's children.
4. **Steps for each owner.** Mint, then render against the post-mint state, then call
   the shared planner (`cli-core`) with the desired link set:

   | Existing line               | Result                                                |
   | --------------------------- | ----------------------------------------------------- |
   | missing                     | insert                                                |
   | different entries           | rewrite in place                                      |
   | same entries, non-canonical | canonicalise                                          |
   | not desired                 | delete                                                |
   | duplicates                  | keep the first, delete the rest                       |
   | any malformed BLOCKS child  | leave every BLOCKS child of that owner alone and warn |

5. **Never prune on uncertainty.** An existing entry that resolves to an uncertain
   dependent is kept, but an uncertain dependent is never newly added.
6. **Guarded output.** Edits are structural, so they join the reconciled set: the 2 s
   quiet interval, preimage checks, and recovery snapshots apply. Originals merge into
   `pre_reconcile`, keeping the earliest copy.
7. **Preconditions.** The pass inherits today's preconditions: the daily note exists and
   has a Pomodoros section. Live gestures do not depend on reconcile.

### Rule 7: immediate writers

| Writer                                                                                           | BLOCKS behaviour                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| nav Depends on stage (Task Card `b`, Ctrl+Shift+P on the line, counted, palette, marked batches) | **Same-note targets:** the owner's BLOCKS edit and any minted dependent id go into the dependent's one transaction, so one Ctrl+Z undoes both. **Cross-note targets:** after the commit, a best-effort follow-up per target note, for old ∪ new targets. It uses the open editor (its own undo entry), otherwise a preimage-checked `vault.process`. On failure the notice gains `· 🧱 BLOCKS not updated`; nothing rolls back. Preparation never writes a reverse entry. |
| nav removals (chip ×, api v3 `removeDependency`, Ctrl+D on the Depends on row, counted Ctrl+D)   | Remove the dependent from each removed target's line, deleting the line when it empties.                                                                                                                                                                                                                                                                                                                                                                                  |
| nav hand-edit mirror (cursor leaves an edited Depends-On line; undo re-arms it)                  | Diff the burst baseline's targets against the final targets and mirror the difference. Clearing the whole line removes every old target. This keeps cross-note undo honest.                                                                                                                                                                                                                                                                                               |
| capture `&` (and Bob Mac Capture through bob)                                                    | Stage each added target's BLOCKS entry and the dependent's minted `^id` in the same batch, using the shared Rust planner. The target's block is reported as a `dependency_target` task block.                                                                                                                                                                                                                                                                             |
| capture dependency drop (§13.5.1)                                                                | Stage the dependent's removal from each dropped prerequisite's line. Prerequisite task lines are still never edited.                                                                                                                                                                                                                                                                                                                                                      |
| Ctrl+Shift+M, `bob task archive`, project conversion                                             | The moved owner's line moves with it; generic link rebasing applies. Conversion never copies a BLOCKS line as a user child. Links into a moved or converted dependent are left to reconcile.                                                                                                                                                                                                                                                                              |
| Closes, cancels, reopens (capture `=x` / `!`, Ctrl+Enter, Alt+], Mac Capture, hand edits)        | **None.** The owner's line freezes; a closed dependent drops off at the next reconcile. Chips hide the lag.                                                                                                                                                                                                                                                                                                                                                               |

All writers share one planner, implemented as a Rust function with a JS twin and pinned
by the same vectors.

- A writer computes the desired entries and the planner turns them into edits:
  - reconcile computes the full projection;
  - gestures compute (existing entries resolved to identities) ∪ added − removed.
- Writers compare canonical output before writing, so no-ops write nothing and add no
  undo entries.
- Machine writes are annotated, so nav's listeners ignore them.

### Rule 8: gestures on a BLOCKS line

| Gesture                                                  | Behaviour                                                                                                                                                                |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Hand edit (typing, deleting an entry, deleting the line) | Never an edge change. Once per burst, nav shows `🧱 BLOCKS is generated from DEPENDS ON lines · change it on the dependent (Ctrl+Shift+P)`. Reconcile restores the line. |
| `!` / `N!`, Ctrl+Shift+Enter                             | Refused with the same notice.                                                                                                                                            |
| Ctrl+Enter on a link                                     | Closes or reopens the linked dependent: root-only, with no strike, no re-embed, and no tree close (as §8 for Depends-On links).                                          |
| Alt+] / Alt+[ on the line                                | Cycles the dependent under the cursor; never reformats the bullet.                                                                                                       |
| Ctrl+Shift+P on the line, palette Edit task dependencies | Acts on the owning task (its Task Card or Depends on stage), never on the chip under the cursor.                                                                         |

### Rule 9: chips (bob-ledger-tools)

- **Data.** A reverse index `id → open dependents`, built from the in-memory Tasks memo
  (`dependsOn` / `id`, the same source as the Depends-On chips).
  - It is shared infrastructure, so later features can reuse it: bob-cli-3p, bob-cli-3q,
    review cards, and the close preview.
  - Chips render the **live** set in the stored line's order. A live dependent missing
    from the text is appended in canonical order and counted.
  - A stored entry that is no longer live renders dashed, with the tooltip
    `No longer waiting on this task · refreshes on the next reconcile`, and is not
    counted.
- **Anatomy.** Reuse the `bob-dep-row` / `bob-dep-chip` family and the `--task-status-*`
  tokens:
  - miniature checkbox, a ~40-character description, `↗ note` for cross-note entries,
    hover preview, click / Mod-click;
  - a `^prj` chip shows 🧩 and the project note name;
  - `#hide` chips are muted.

  There is no new palette. Chips never change line height and wrap between chips, never
  inside one.

- **🔑 badge.** T is D's only remaining reason to be Blocked: no other open prerequisite
  (by D's `dependsOn` ids), no future `scheduled`, and no `#prj_is_blocked`.
  - The tooltip lists the other reasons:
    `also waits on "Outline" · 2026-11-02 · project sase`.
  - Each badge has `aria-label` text, and colour is never the only signal. Every live
    dependent is `[?]`, so status colour cannot tell chips apart; the badge carries that
    signal.
- **Label and summary.** The summary is never written to Markdown.

  | Owner                | Label                               | Summary                                                    |
  | -------------------- | ----------------------------------- | ---------------------------------------------------------- |
  | open                 | `🧱 blocks`                         | `frees N` when every live chip has 🔑, else `frees N of M` |
  | closed (frozen line) | `🧱 was blocking`, muted, no badges | `✓ freed M`                                                |

- **No × or ＋ in v1.** The row tooltip reads
  `Generated from DEPENDS ON lines · edit on the dependent`.
- **Toggle.** `toggle-dependency-chips` toggles Depends-On and BLOCKS rows together.
- **Without the plugin**, the line still reads as plain links.
- **Implementation.** Write new fragments with an explicit relation direction. Never
  reuse the incoming row's `waiting on` copy, action callbacks, or aria labels from 140,
  150, or 280.

### Rule 10: kill switch, config, and outputs

```yaml
dependencies:
  blocks_lines: true # write 🧱 BLOCKS lines on prerequisites (docs/task-dependencies.md §14)
```

- **Config.** The key lives in `~/.config/bob/config.yml` (`BOB_CONFIG_FILE` /
  `XDG_CONFIG_HOME` apply). Rust and nav read it.
  - An invalid value falls back to `true` and warns through the existing paths:
    reconcile stderr, and a capture warning string.
  - With `false`, reconcile's desired set is empty: it removes every BLOCKS line on open
    tasks, mints nothing, and leaves frozen lines alone. Gestures write no BLOCKS edits.
- **Reconcile JSON (additive).** These use the same line convention as
  `adopted_dependency_lines`:
  - `blocks_line_updates`: `{path, line, task, kind, added[], removed[]}`, where `kind`
    is one of `inserted`, `rewritten`, `canonicalized`, `removed`, or `deduplicated`;
  - `minted_dependent_block_ids`: `{path, line, block_id}`;
  - the new warning kind `malformed_blocks_line` in `dependency_warnings`.
  - Human output gets a `🧱` group in the existing dependency style.
  - `--dry-run` previews all of it.
- **Capture JSON (additive, omitted when empty).** Each item gets `blocks_line_updates`
  rows of the same shape, and `minted_dependent_block_id`. Human output stays quiet when
  the mirror works.
- **No new subcommand, flag, or Tasks field.**
  `bob query --tasks "(not done) AND (is blocking)"` already answers read-only
  questions.

### Rule 11: managed child lines are a registry

`docs/task-dependencies.md` §2.4 grows into one registry table of every machine-owned
child line:

- `❌ CANCEL LOG`
- `⛓️ DEPENDS ON`
- `🧱 BLOCKS`
- `🧩 Sub-projects`
- `🗓️ SCHEDULE LOG`
- Work Log entries

For each one, the registry gives its owner, its writer, its slot, and the recognisers
that must reject it. The MC vectors are copied into every engine's tests.

**Neither managed dependency line is ever a Task Link.** That covers Pomodoro promotion,
consumption, successor anchors, the plan budget (Rust and JS), completable sightings,
Pomodoro close / cleanup, and Task Link opening. Neither line is a tree-close or
strike/restore target.

## Vectors

`contract` copies these into the docs. Phases cite them by id in test comments.

### BK: parse

| #    | line                                                                          | context                                                                         | verdict                                                                                          |
| ---- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| BK1  | `🧱 **BLOCKS:** [[#^a]]`                                                      | direct child of a `#task`                                                       | accept(1), canonical                                                                             |
| BK2  | `🧱 **BLOCKS:** [[#^a]] • [[cash#^b]] • [[money/cash#^c]]`                    | direct child                                                                    | accept(3), canonical                                                                             |
| BK3  | `🧱 **BLOCKS:** [[#^a\|b • c]]`                                               | alias contains `•`                                                              | accept(1), canonicalise                                                                          |
| BK4  | `**BLOCKS:** [[#^a]]`                                                         | emoji missing                                                                   | accept(1), canonicalise                                                                          |
| BK5  | `🚧 **BLOCKS:** [[#^a]]`                                                      | WIP emoji                                                                       | accept(1), canonicalise                                                                          |
| BK6  | `⛓️ **BLOCKS:** [[#^a]]` and `⛓ **BLOCKS:** [[#^a]]`                          | chain emoji                                                                     | accept(1), canonicalise (under `glyph = chain` the `⛓️` form is canonical and BK1 canonicalises) |
| BK7  | `🧱 **BLOCKING:** [[#^a]]`                                                    | legacy label                                                                    | accept(1), canonicalise                                                                          |
| BK8  | `🧱 **BLOCKS:** [[#^a]] · [[#^b]]` / `[[#^a]], [[#^b]]` / `[[#^a]] [[#^b]]`   | separators                                                                      | accept(2), canonicalise                                                                          |
| BK9  | `🧱 **BLOCKS:** ~~[[#^a]]~~` / `![[#^a]]`                                     | struck / embedded                                                               | accept(1), canonicalise                                                                          |
| BK10 | `🧱 **BLOCKS:** [[cash#^b]] • [[#^a]]`                                        | out of order                                                                    | accept(2), canonicalise to `[[#^a]] • [[cash#^b]]`                                               |
| BK11 | `🧱 **BLOCKS:** [[#^a]] • [[#^a]]`                                            | duplicate entry                                                                 | accept(2), canonicalise to one entry                                                             |
| BK12 | `🧱 **BLOCKS:**`                                                              | label only                                                                      | empty                                                                                            |
| BK13 | `🧱 **BLOCKS:** [[#^a`                                                        | half-typed                                                                      | malformed                                                                                        |
| BK14 | `🧱 **BLOCKS:** [[#^a]] until Friday`                                         | trailing prose                                                                  | malformed                                                                                        |
| BK15 | `🧱 **BLOCKS:** [[note]]` / `[[note#Heading]]` / `• ,`                        | no block id / separators only                                                   | malformed                                                                                        |
| BK16 | `🧱 **blocks:** [[#^a]]`                                                      | lowercase                                                                       | not-a-line                                                                                       |
| BK17 | `🧱 **BLOCKED BY:** [[#^a]]`                                                  | inverse label                                                                   | not-a-line                                                                                       |
| BK18 | `🧱 **BLOCKS:** [[#^a]]`                                                      | fenced / grandchild / hangs off a Work Log entry / blockquoted / no list marker | not-a-line                                                                                       |
| BK19 | `🧱 **BLOCKS:** [[#^a]]`                                                      | third direct child after prose                                                  | accept(1) (no slot is enforced on read)                                                          |
| BK20 | BK1 through the Depends-On parser; DP1 through the BLOCKS parser              | cross-parse                                                                     | not-a-line both ways                                                                             |
| BK21 | entries `[[zeta#^e]]`, `[[élan#^d]]`, `[[cash#^c]]`, `[[Cash#^a]]`, `[[#^b]]` | sort                                                                            | `[[#^b]] • [[Cash#^a]] • [[cash#^c]] • [[zeta#^e]] • [[élan#^d]]`                                |

### BR: reconcile

| #    | situation                                                                                           | outcome                                                                                                  |
| ---- | --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| BR1  | open D `^d` depends on open T, same note                                                            | T gains `🧱 **BLOCKS:** [[#^d]]` (`inserted`)                                                            |
| BR2  | D in `cash.md` (unique basename), T elsewhere                                                       | entry `[[cash#^d]]`; ambiguous basename → `[[money/cash#^d]]`                                            |
| BR3  | three dependents, same-note and cross-note                                                          | one line, sorted per BK21                                                                                |
| BR4  | D has no `^block-id`                                                                                | D gains ` ^<mint>` (SB); no `[id::]` and no `[fresh::]`; T lists it; a `minted_dependent_block_ids` row  |
| BR5  | D's note is inside the quiet interval and D needs a mint                                            | D not listed this run; next run mints and lists it; then zero changes                                    |
| BR6  | T has Depends-On (with a nested child) and a Schedule Log                                           | BLOCKS inserted right after the Depends-On subtree                                                       |
| BR7  | T has a first-child Cancel Log and no Depends-On                                                    | BLOCKS inserted second                                                                                   |
| BR8  | T is `^prj` with Depends-On and Sub-projects                                                        | order Depends-On, BLOCKS, Sub-projects; `bob projects sync --dry-run` then changes nothing               |
| BR9  | T's only listed D closes                                                                            | entry dropped; line deleted (`removed`)                                                                  |
| BR10 | T closed, line lists open D                                                                         | untouched (frozen)                                                                                       |
| BR11 | T reopens                                                                                           | line re-projected                                                                                        |
| BR12 | T or D in the previous daily note                                                                   | T never written; D never listed or minted                                                                |
| BR13 | hand-added entry with no forward edge                                                               | removed (`rewritten`)                                                                                    |
| BR14 | any BK3–BK11 variant with the right entries                                                         | `canonicalized`                                                                                          |
| BR15 | two BLOCKS lines under T                                                                            | first rewritten, the rest deleted (`deduplicated`); copy-paste under a task with no dependents → deleted |
| BR16 | malformed BLOCKS line under T                                                                       | every BLOCKS child of T untouched; warn `malformed_blocks_line`; no second line                          |
| BR17 | D's Depends-On line malformed (R10)                                                                 | existing entries naming D on any T kept; D never newly added                                             |
| BR18 | D's link is R4 / R5 / R6, an archive target, or a previous-daily / unencodable target kept verbatim | no entry for that link                                                                                   |
| BR19 | `^prj` P depends on T; P has residents                                                              | T lists `[[p#^prj]]` only                                                                                |
| BR20 | `^prj` internal or descendant target                                                                | no entry                                                                                                 |
| BR21 | R8 legacy-child edge                                                                                | listed                                                                                                   |
| BR22 | cycle A ↔ B                                                                                         | each lists the other                                                                                     |
| BR23 | `#hide`, `ref/`, inbox, and today's daily-note tasks on either side                                 | listed normally                                                                                          |
| BR24 | `dependencies.blocks_lines: false`                                                                  | every BLOCKS line on an open task removed; frozen lines stay; no mints                                   |
| BR25 | invalid `dependencies.blocks_lines`                                                                 | behaves as `true`; stderr warning                                                                        |
| BR26 | delete every BLOCKS line in a reconciled vault                                                      | one run rebuilds identical bytes; fields, statuses, edges, promotion unchanged                           |
| BR27 | second run over a reconciled vault                                                                  | zero changes                                                                                             |
| BR28 | owner's note modified < 2 s ago                                                                     | edits deferred (structural)                                                                              |
| BR29 | `--dry-run -f json`                                                                                 | identical rows; nothing written; no mints persisted                                                      |

### BW: immediate writes (nav and capture)

| #    | gesture                                                                            | outcome                                                                                                 |
| ---- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| BW1  | stage adds same-note T to D                                                        | D's line and T's BLOCKS change in one transaction; one Ctrl+Z restores both                             |
| BW2  | stage adds cross-note T                                                            | D commits; then T's note gains or updates its line (open editor, else preimage-checked `vault.process`) |
| BW3  | the cross-note follow-up finds T changed or unreadable                             | D's write stands; notice suffix `· 🧱 BLOCKS not updated`; no rollback                                  |
| BW4  | D lacks `^block-id`                                                                | mint in D's own transaction, before any follow-up; no `[id::]`                                          |
| BW5  | removal via stage, chip ×, api `removeDependency`, or Ctrl+D on the Depends on row | D's entry leaves T's line; empty line deleted                                                           |
| BW6  | add to a T that already lists D                                                    | byte-identical; no write; no undo entry                                                                 |
| BW7  | T closed, or in the previous daily note                                            | T not written                                                                                           |
| BW8  | T's BLOCKS line malformed                                                          | left alone; D's write stands; suffix `· 🧱 BLOCKS not updated`                                          |
| BW9  | counted `N<Ctrl+Shift+P>` add-to-all on k dependents                               | T's single line lists all k, sorted                                                                     |
| BW10 | hand edit adds / removes a link on D's line (cursor leaves)                        | T lines updated from the diff                                                                           |
| BW11 | Ctrl+Z of a cross-note add in D's editor                                           | the mirror re-fires and removes D from T's line                                                         |
| BW12 | hand edit confined to a BLOCKS line                                                | no graph change; the generated-line notice once                                                         |
| BW13 | capture `&cash:t` on a new task or an explicit dependent                           | T's line updated in the same staged batch; row reported; T reported as a `dependency_target` task block |
| BW14 | capture dependency drop of T1 and T2                                               | D removed from both lines in the same batch                                                             |
| BW15 | kill switch off                                                                    | no writer writes BLOCKS or mints for it                                                                 |
| BW16 | capture `--dry-run`                                                                | rows reported; nothing written                                                                          |
| BW17 | the same input rendered by Rust and nav                                            | byte-identical line (BK21 included)                                                                     |
| BW18 | project conversion forward / reverse                                               | a BLOCKS line is never copied as a user child                                                           |
| BW19 | Ctrl+Shift+M of T                                                                  | T's line moves with rebased links; never two BLOCKS lines                                               |

### BC: chips

| #    | situation                                     | render                                                                                                                                                                       |
| ---- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BC1  | open T; one open D whose only reason is T     | `🧱 blocks ( ? D 🔑 ) frees 1`                                                                                                                                               |
| BC2  | D also waits on open U                        | no 🔑; `frees 0 of 1`; tooltip `also waits on "U"`                                                                                                                           |
| BC3  | D's only other reason is a future `scheduled` | no 🔑; the tooltip names the date                                                                                                                                            |
| BC4  | D carries `#prj_is_blocked`                   | no 🔑; the tooltip names the project                                                                                                                                         |
| BC5  | four dependents, all 🔑                       | `frees 4`                                                                                                                                                                    |
| BC6  | cross-note D                                  | `↗ note`                                                                                                                                                                     |
| BC7  | `^prj` D                                      | 🧩 plus the project note name                                                                                                                                                |
| BC8  | stored entry no longer live (open owner)      | dashed; the refresh tooltip; not counted; with no live chip left, the summary is omitted                                                                                     |
| BC9  | live D missing from the stored line           | appended in canonical order; counted                                                                                                                                         |
| BC10 | closed owner                                  | `🧱 was blocking`, muted, no 🔑, `✓ freed M` (M = stored entries); entries show their current status and are never dashed as stale; a missing target shows `⚠ ^id not found` |
| BC11 | D carries `#hide`                             | muted chip                                                                                                                                                                   |
| BC12 | interaction                                   | click opens D, Mod-click opens a new tab, hover previews; no × / ＋                                                                                                          |
| BC13 | 60-character description                      | cut at about 40; full text in the tooltip                                                                                                                                    |
| BC14 | `toggle-dependency-chips`                     | toggles Depends-On and BLOCKS rows together                                                                                                                                  |
| BC15 | malformed or not-a-line BLOCKS row            | no chips                                                                                                                                                                     |

### MC: managed child lines

| #   | situation                                                                  | outcome                                                                                                                                    |
| --- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| MC1 | a task nested under an open Pomodoro entry has a BLOCKS line               | no Task Link: no promotion, consumption, successor anchor, plan-budget link, completable sighting, or close / cleanup target (Rust and JS) |
| MC2 | the same with a Depends-On line                                            | same (fixes the latent miscount)                                                                                                           |
| MC3 | `🧱 **BLOCKS:** ![[#^d]]` under T; T closes by capture `=x` or Ctrl+Enter  | D is never closed (tree close skips both lines)                                                                                            |
| MC4 | T closes or reopens elsewhere                                              | cycler strike/restore never touches a BLOCKS line                                                                                          |
| MC5 | `!`, Ctrl+Shift+Enter on a BLOCKS line                                     | refused with the generated-line notice                                                                                                     |
| MC6 | R8 legacy-child recogniser on a BLOCKS line                                | not a legacy child                                                                                                                         |
| MC7 | section titles, Schedule / Cancel / Work Log, and Sub-projects recognisers | never claim a BLOCKS line; the BLOCKS parser never claims theirs                                                                           |
| MC8 | `bob projects sync` on `^prj` with Depends-On and BLOCKS                   | marker after both subtrees; a marker before BLOCKS moves after it                                                                          |
| MC9 | `bob task archive` of a closed T with a frozen `[[#^d]]`                   | link qualified to `[[note#^d]]`                                                                                                            |

## Cross-cutting rules for every phase

- **Read first:**
  - this plan's Design and Vectors;
  - `docs/task-dependencies.md` §§2–5, 6.6, 7, 13.5.1, and §14 once `contract` lands;
  - `sase artifact read research:202610/prerequisite_blocks_line/prerequisite_blocks_line.md "<why>"`;
  - `sase memory read decisions:task-deps-are-depends-on-links decisions:projects-block-from-prj-line decisions:capture-link-drops-blocking-deps glossary:task-dependency-link glossary:task-link -r "<why>"`.
- **Vectors are the contract.**
  - Copy every row you implement into tests, citing its id in a comment.
  - If code and a vector disagree, fix §14 first in the same change and say so in the
    commit.
- **Don't break anything.**
  - Every existing DP/DW/DR/DC/SL/SB/PD\* test keeps passing unchanged.
  - JSON changes are additive.
  - No reason kind, `trigger` value, or capture grammar changes. `capture-parse` does no
    new work.
- **Performance.**
  - Reconcile adds no file reads. Its owner discovery is byte-prefiltered.
  - Capture adds no vault scan: target notes are already loaded by the batch.
  - nav reads no note it did not already load, except a cross-note target's follow-up.
  - Record before/after timings for reconcile (dry-run on a vault copy) and for a
    capture `&` dry-run in your phase note.
- **Known failures.** Before calling a failure pre-existing, check
  `sase bead list -T flake -T ci` and reproduce it on clean origin/master.
- **bob-cli.**
  - Run `just check`.
  - Put new logic in new modules. `capture/dependencies.rs`, `capture/output.rs`,
    `task_status_hooks/reconcile.rs`, and `sync.rs` should grow by a few lines each.
  - Add no CLI options or subcommands. If you edit `--help` text, first read
    `cli_rules.md` with `/sase_memory_read`.
- **bob-plugins** (`sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md`):
  - Edit only `src/` fragments, each ≤ 1000 lines. Register new fragments in
    `src/fragments.json`, and export pure helpers for tests.
  - Run `npm run build`, `npm test`, and `npm run validate`. Append new test files to
    `package.json`'s `test` script.
  - Minor-bump each touched manifest from the latest master version, and update its root
    `README.md` row.
  - Deploy on this host only:
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`.
- **Safety.**
  - Never edit the vault by hand.
  - Never run a live `bob task reconcile`, `bob projects sync`, or non-dry-run
    `bob capture` against `~/bob`. Dry runs are fine.
  - Experiments use `BOB_DIR=<temporary copy of ~/bob>` (with `BOB_NOW` when the copy
    lacks today's daily note) and `BOB_CONFIG_FILE=<temp file>` for kill-switch tests.
    Delete the copies afterwards.
  - Never touch the MacBook outside `rollout`.
- **Memory.** Only `rollout` edits memory, and only within the accepted memory
  decisions.
- **Follow-ups.** Append `PROPOSED FOLLOW-UP:` notes to your own phase bead; never
  create beads.

## Phase: contract (the BLOCKS-line contract and vectors)

**Repo:** bob-cli, docs only.

1. **New `docs/task-dependencies.md` §14 "Blocks lines".** Subsections: Vocabulary
   (**Blocks line**, **owner**, **frozen line**), Grammar, Membership, Identity and
   minting, Order and position, Reconciliation, Immediate writers, Gestures, Chips, Kill
   switch, Outputs and warning kinds, Rollout, and the BK/BR/BW/BC vector tables.
   - Copy Rules 1–10 and the vectors faithfully, in the doc's voice.
   - Write the glyph per the `glyph` decision.
2. **§2.4 becomes "Managed child lines".** Turn it into the registry table (Rule 11) and
   add §11.8 with the MC vectors.
3. **Small edits elsewhere in the same file.**
   - Intro: one sentence on the mirror.
   - §1: a pointer to §14.
   - §2.1: the "On `^prj`…" bullet gains "a BLOCKS line, when present, sits between
     them".
   - §5: a "Visibility" bullet.
   - §7 and §8: pointers to §14.9 / §14.8.
   - §13.5.1: "prerequisite tasks are never edited" becomes "prerequisite task lines are
     never edited; the drop removes the dependent from each prerequisite's BLOCKS line
     (§14)".
4. **`docs/projects.md`.** Sub-projects anchoring and the move rule (Rule 5, MC8).
5. **`docs/capture.md`.** In the `&` and dependency-drop paragraphs: BLOCKS staging,
   minting, `blocks_line_updates`, and `minted_dependent_block_id`.
6. **`docs/getting-started.md`.** The config pointer list gains
   `dependencies.blocks_lines`.
7. **Finish.** Run Markdown and link checks, if the repo has them, and commit.

## Phase: cli-core (Rust grammar, shared planner, guards, anchor, config)

**Repo:** bob-cli.

1. **New `src/native/task_dependencies/blocks.rs`:**
   - Constants: emoji per `glyph`, label `BLOCKS`, and the shared `•` separator.
   - `BlocksLine` (`Accepted{links, canonical}` / `Empty` / `Malformed` / `NotALine`).
     `canonical` also requires sorted, duplicate-free entries.
   - `parse_blocks_line`, `is_blocks_line`, and `is_managed_dependency_line` (Depends-On
     or BLOCKS, in any verdict but not-a-line).
   - `blocks_children_of(lines, fenced, task_line)`: every direct-child BLOCKS line in
     order.
   - `sort_blocks_links` (BK21) and `format_blocks_line(indent, links)`.
   - `blocks_insert_slot`: after the Depends-On subtree, else the Depends-On slot.
   - **The shared planner**
     `plan_blocks_line(lines, fenced, owner_line, desired: &BTreeSet<String>) -> BlocksLinePlan`.
     It returns line edits (insert / replace / remove) plus a report (`kind`, `added`,
     `removed`) per the Rule 6 step-4 table, or `Malformed` with no edits.
   - **The delta helper** `blocks_desired_after_delta(existing, resolve, add, remove)`
     for gestures. It resolves the existing entries to identities through a caller
     callback, re-renders them canonically, and applies the delta.
   - Tests: BK1–BK21, and planner unit tests for every row of the step-4 table.
2. **Guards:**
   - `capture_completable_tasks.rs` and `task_complete/tree.rs` switch to
     `is_managed_dependency_line`.
   - `scan_pomodoros` skips bullets that are managed dependency lines (MC1/MC2).
   - Audit every Pomodoro-child link scanner: `capture_pomodoro_close/*`,
     `capture_pomodoros_agenda.rs`, `capture_pomodoro_start.rs`, and `plan_budget`.
     Guard each one that could count a labelled multi-link bullet, with an MC1/MC2 test
     per guarded scanner. Note the scanners already safe by shape in your phase note.
   - Add R8 (MC6) and recogniser (MC7) tests.
3. **`projects/sync.rs`** (MC8). `subprojects_anchor_child` anchors after the last
   direct Depends-On or BLOCKS child's subtree. The precedes check covers BLOCKS too.
4. **Config.** New `src/native/config/dependencies.rs`: `load_dependencies_config` →
   `DependenciesConfig { blocks_lines: bool }`, default `true`. An invalid value returns
   the same error type as plan config, so callers can fall back and warn. Tests mirror
   `link_unblocked`.
5. **Archive.** Add an MC9 test (existing behaviour; fix it if it fails).
6. **Scope.** No writer emits a BLOCKS line yet. Run `just check` and commit.

## Phase: plugin-guards (recognisers and gesture guards)

**Repo:** bob-plugins: `bob-navigation-hotkeys`, `task-status-cycler`,
`block-id-prompt`. Leave `bob-ledger-tools` to `chips`.

1. **Recognisers.**
   - **nav:** a new pure fragment `081-blocks-line-grammar.js` with `parseBlocksLine`,
     `isTaskBlocksLine`, `formatBlocksLine`, `sortBlocksLinks` (code-point compare),
     `findBlocksChildren`, and `blocksInsertLine`, exported for tests.
   - **cycler** (beside `050 isTaskDependencyLine`) and **block-id-prompt** (beside
     `060 isTaskDependencyLine`): an `isTaskBlocksLine` recogniser that is true for
     accept, empty, and malformed.
   - Copy BK1–BK21 into each plugin's tests (nav's sort and format rows too).
2. **nav guards:**
   - The hand-edit mirror ignores changes confined to a BLOCKS line and shows the Rule 8
     notice once per burst. Machine writes carry an annotation the listener skips.
   - `!` / `N!` refuse on a BLOCKS line.
   - Ctrl+Shift+P on a BLOCKS line opens the owning task's Task Card, and the palette
     command acts on the owner.
   - Project conversion (`splitProjectLifecycleChildren`,
     `collectProjectSourceDependencyBlock`) never copies a BLOCKS line as a user child;
     it drops it as it does Sub-projects. Replace the inline Sub-projects regex with
     named recognisers.
   - Audit nav's Pomodoro capture and reorder (`150`, `160`) and guard any multi-link
     Task Link reader.
3. **cycler guards:**
   - Strike/restore skip (MC4) and tree-close skip (MC3).
   - `onDependencyLine` and `isActiveTaskDependencyLine` also accept BLOCKS lines, so
     Ctrl+Enter is root-only and Alt+]/Alt+[ cycle the dependent under the cursor
     without reformatting.
   - `classifyPomodoroSubBullets` and `parseSuccessorLiveLink` skip managed lines
     (MC1/MC2).
4. **block-id-prompt guards:**
   - `startTaskLinkOpen` / `applyTaskLinkOpen` refuse with the Rule 8 notice (MC5).
   - `planPomodoroLinkCleanupForRanges` and `collectPomodoroEntryBlockLinks` skip
     managed lines.
5. **Tests.** Add tests for each guard (MC1–MC7 rows that apply). Bump, document, build,
   test, validate, deploy
   (`-p bob-navigation-hotkeys -p task-status-cycler -p block-id-prompt`), and commit.

## Phase: chips (bob-ledger-tools BLOCKS chips)

**Repo:** bob-plugins, `plugins/bob-ledger-tools`.

1. **New fragments** (≤ 1000 lines each; leave 140, 150, and 280 nearly untouched):
   - a pure model: `parseBlocksLine` (BK copy), ownership by reusing the owning-task
     helpers, `blocksDependentsIndex(memo)`,
     `blocksLastBlocker(dependent, owner, memo, today)`, and `blocksChipModel`;
   - an element builder: label, chips, 🔑 badge, summary, and tooltips per Rule 9;
   - Live Preview decorations: a `BLOCKS` / `BLOCKING` prefilter, visible ranges only,
     no line-height change;
   - the Reading-view post-processor, decorating existing anchors in place with the same
     owning-line and order-mapping rules as §7.4.
2. **Index.** Build the index once per memo generation, beside `dependencyTasksIndex`.
   Rebuild and refresh on the same events as the Depends-On chips.
3. **Wiring.** Register through `setupDependencyChips`, so one toggle and one refresh
   cover both rows (BC14).
4. **CSS.** Add the 🔑 badge, the muted frozen row, and the dashed stale chip, using
   existing tokens only. Support reduced motion and phone width.
5. **Plan budget.** `planBlockLinks` (`050`) skips managed dependency lines (MC1/MC2),
   matching Rust.
6. **Tests.** A new `scripts/test-ledger-tools-blocks-chips.cjs`: BK copy, BC1–BC15,
   Live Preview decorations, and Reading view. Add the budget guard test.
7. **Finish.** Bump, document, build, test, validate, deploy `-p bob-ledger-tools`, and
   commit.

## Phase: cli-reconcile (reconcile owns the projection)

**Repo:** bob-cli.

1. **Export** per Rule 6 step 1: `ReconcileOutcome.explicit_edges` and
   `uncertain_dependents`, recorded in `plan_accepted`, `plan_adoption`, and the
   legacy-only path. Add unit tests showing:
   - id-less dependents appear;
   - implicit resident edges and every BR18/BR20 exclusion do not.
2. **New module `src/native/task_status_hooks/blocks.rs`:**
   - `reconcile_blocks_lines(files, note_index, settings, edges, uncertain, previous_daily_file, quiet_deferred, enabled) -> BlocksOutcome`,
     where `BlocksOutcome` holds the touched files, first-touch originals, updates,
     mints, and warnings.
   - It applies Rules 3–6 using the `cli-core` planner and `mint_block_id`, then edits
     `files` contents in place and re-parses touched files.
3. **Wiring in `sync.rs`.**
   - Call the pass right after the reconcile re-parse loop.
   - Merge touched files into `structural_files` and originals into `pre_reconcile`.
   - Load `dependencies.blocks_lines` once per run; invalid values warn on stderr and
     fall back to `true`.
4. **Outputs.**
   - `SyncResult` gains `blocks_line_updates` and `minted_dependent_block_ids`;
     `malformed_blocks_line` joins `dependency_warnings`.
   - Add the human `🧱` group.
   - Update `docs/task-status-hooks.md`: the Dependency lines section, the JSON example,
     and the human output.
5. **Tests.** A new `tests/cli/task_status_hooks/blocks.rs` covering BR1–BR29,
   including:
   - BR26 (delete every line, rebuild identical bytes);
   - BR27 (idempotence);
   - BR5 (the quiet-interval mint skip);
   - BR8 with a `bob projects sync --dry-run` round trip.
6. **Live dry-run (read-only).**
   - Run the phase build's `bob task reconcile --dry-run -f json` against `~/bob`.
   - Summarise the inserts, mints, and warnings in your phase note. The research expects
     about 15 lines and 6 mints; explain any large difference.
   - Record timings.
7. **Finish.** Run `just check` and commit.

## Phase: cli-capture (capture keeps the mirror current)

**Repo:** bob-cli; Bob Mac Capture is read-only.

1. **New module `src/native/capture/blocks_mirror.rs`.** It is used by the `&` planners
   and `dependency_drop.rs`, and adds only a few lines to `dependencies.rs`.
   - **Add path.** After the dependent is staged, for each added target T that is open,
     writable, and not an archive or previous-daily task:
     - mint the dependent's `^id` in its staged note if missing, including new tasks,
       with batch reservations;
     - stage T's line through `blocks_desired_after_delta` and `plan_blocks_line`;
     - re-read staged text between writes, so same-note pairs compose.
   - **Drop path.** For each dropped prerequisite, stage the removal; an empty line is
     deleted.
   - **Malformed or skipped T.** Leave its line alone and push a capture warning. Never
     fail the capture.
   - **Kill switch off.** Do nothing.
2. **Reporting.**
   - Item-level `blocks_line_updates` and `minted_dependent_block_id` (omitted when
     empty).
   - A T whose block changed is reported as a `dependency_target` task block through
     `TargetTouch`, so the Bob Mac Capture preview card shows the new 🧱 line.
   - Drops report rows only, with no extra task blocks.
   - Human output is unchanged apart from warnings.
3. **Bob Mac Capture.** Confirm read-only that its decoders ignore the new keys and that
   target cards render any child line. Note this in the phase note; change no Swift
   code.
4. **Tests.** BW13–BW17 (BW17 against the `cli-core` planner), new-task minting,
   same-note composition, dry-run, and the existing `&` / drop suites unchanged.
5. **Finish.** Record timings (Cross-cutting rules), run `just check`, and commit.

## Phase: nav-writer (planner seam, minting, follow-ups)

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`.

1. **Extend `081`** with `planBlocksLineEdit(content, ownerLine, desiredLinks)` and
   `blocksDesiredAfterDelta(...)`, mirroring the Rust planner. Copy the BW17 and BK21
   render vectors.
2. **New fragment `082-blocks-mint.js`.** `mintDependentBlockId`, ported from the
   cycler's `086 mintBlockId`, with SB1–SB9 tests.
3. **Config.** Load `dependencies.blocks_lines` beside `loadBulletPropertyConfig`,
   stat-cached like the cycler's `loadLinkUnblocked`. Invalid values mean `true`.
4. **Seam.** `planDependencyEdit` calls a new `planBlocksForDependencyEdit` (new
   fragment `083`; move code out of `080` to stay ≤ 1000 lines). It does three things:
   - mints the dependent's `^id` into `nextContent` when needed;
   - applies same-note owners' BLOCKS edits into `nextContent`;
   - returns `blocksFollowUps: [{path, add: [link], remove: [identity]}]` for cross-note
     owners.

   It skips closed, previous-daily, archive, and missing owners silently.

5. **Follow-up writer.** New fragment `591-blocks-follow-ups.js`:
   `applyBlocksFollowUps(followUps)`.
   - It works per target note: through an open editor transaction (annotated) if the
     note is open, otherwise through a preimage-checked `vault.process`. It can insert
     and delete lines.
   - It compares canonical output, so no-ops write nothing.
   - It returns `{ok, failed: [path]}`, which drives the `· 🧱 BLOCKS not updated`
     suffix through the `216` notice builders.
6. **Wiring.**
   - The shared writer `applyDependencyEdit` covers stage removal, cross-note choose,
     api `removeDependency`, chip ×, and Ctrl+D on the Depends on row.
   - Also wire `setLocalTaskDependencyLink`.
7. **Tests.** BW1–BW8, BW12, BW15, and BW17 through the dependency-stage harnesses.
8. **Finish.** Bump, document, build, test, validate, deploy
   `-p bob-navigation-hotkeys`, and commit.

## Phase: nav-paths (every remaining writer, the mirror, undo)

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`.

1. **Remaining direct-planner paths.** These get same-note edits through the seam and
   call `applyBlocksFollowUps` once per gesture after their own commit:
   - counted remove-from-all (`420`);
   - vault counted add/remove (`420`);
   - counted same-note add (`570` / `270`);
   - marked batches (`430`, `440`);
   - counted Ctrl+D (`600`);
   - hand-edit clears (`600`).
2. **Hand-edit mirror.**
   - Diff the burst baseline's Depends-On targets against the final targets, parsed with
     reader tolerance and resolved, then feed the seam. Same-note edits go in the
     mirror's transaction; cross-note edits are follow-ups.
   - Undo re-fires the mirror (BW11), and canonical compare prevents loops and extra
     undo entries (BW6).
3. **Project conversion and move.**
   - Conversion forward and reverse (BW18): no BLOCKS copy, and owners naming the
     converted source are left to reconcile.
   - Ctrl+Shift+M (BW19): the moved owner's line is rebased by the existing move code;
     never two lines.
4. **Tests.** BW9–BW11, BW18, BW19, and one test per path in step 1.
5. **Finish.** Bump, document, build, test, validate, deploy, and commit.

## Phase: rollout (install, verify, reach the MacBook, record)

1. **This host.**
   - From up-to-date bob-cli master, run `just install` (never `--force`).
   - Run `bob task reconcile --dry-run -f json` against `~/bob`. Review every
     `blocks_line_updates` row and mint, and explain any warning.
   - Run `bob projects sync --dry-run`: no BLOCKS-caused change.
   - Pull the opened bob-plugins checkout to origin master, run
     `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`.
2. **The pilot script on a copy** (`BOB_DIR=<copy>`, `BOB_NOW` as needed):
   1. A live reconcile writes the lines.
   2. A second run makes zero changes (BR27).
   3. Delete every BLOCKS line; one run rebuilds identical bytes (BR26).
   4. With `BOB_CONFIG_FILE` setting `dependencies.blocks_lines: false`, lines on open
      tasks are removed; setting it back to `true` rebuilds them (BR24).
   5. `bob capture '&…'` updates the target's line, and a dependency-drop gesture
      removes the entry (BW13, BW14).
   6. Delete the copy.
3. **MacBook bob, best effort** (bob before plugins).
   - Retry `ssh -o ConnectTimeout=20 -o BatchMode=yes mac …` for up to about 30 minutes.
   - Install detached:
     `cd ~/projects/github/bobs-org/bob-cli && nohup zsh -lc 'git pull --ff-only && just install; echo INSTALL_EXIT=$?' > /tmp/bob-rollout-install-blocks.log 2>&1 < /dev/null &`.
     Poll the log for `INSTALL_EXIT=0`.
   - Verify the reconcile dry-run there.
4. **MacBook plugins, only after step 3 verifies.** Run `ssh mac 'bob plugins sync'` and
   `ssh mac 'bob plugins list'`. Otherwise record what is left.
5. **Memory.**

   > [!decision] memory_decision_record Use `/sase_memory_write` (Edit And Republish).
   > Add the decisions strand `blocks-lines-mirror-depends-on`, titled "BLOCKS Lines
   > Mirror Depends-On And Are Never Read Back", mirroring the existing strands'
   > frontmatter and sections:
   >
   > - **Applies to:** bob-cli, bob-plugins, vault.
   > - **Claim:** one machine-owned BLOCKS line per open prerequisite, listing its open
   >   direct dependents. It is a projection of Depends-On: reconcile is the authority,
   >   every forward-edge writer updates it, a closed owner's line freezes, it is never
   >   read back, and `dependencies.blocks_lines` removes it.
   > - **Why:** quote the request.
   > - **Rejected:** render-only chips; one bullet per dependent; editable lines that
   >   form a second source of truth; deleting lines on close; a `blockedBy` field; a
   >   new CLI.
   > - **Cost:** recogniser sprawl across five engines; reconcile-interval lag outside
   >   Obsidian; minted dependent ids.
   > - **Reopens when:** BLOCKS warnings or repair passes show up on most runs for two
   >   weeks, or fan-in turns lines into walls of links.
   > - **Evidence:** this plan's ref, the research ref, and the phase commits.
   > - **Links:** `[[decisions:task-deps-are-depends-on-links]]`.

   > [!decision] memory_decision_record = no Append a `PROPOSED FOLLOW-UP:` note
   > describing the record instead.

   > [!decision] memory_glossary Add one sentence to `glossary/task-dependency-link.md`:
   > each open prerequisite carries a generated `🧱 **BLOCKS:**` line listing its open
   > dependents, which Bob maintains from Depends-On lines and never reads back, so
   > dependencies are edited only on the dependent. Link the new record if it exists.

   > [!decision] memory_glossary = no Append a `PROPOSED FOLLOW-UP:` note instead.

   If anything changed, run `sase memory init`.

6. **Coordination notes** (as `PROPOSED FOLLOW-UP:` notes on this phase bead):
   - **bob-cli-3p:** the Reverse Blocks… stage should write dependents' Depends-On
     lines, add the chip ＋ / write-through ×, a Task Card "Blocks… (N)" row, and
     Ctrl+Shift+P on a BLOCKS line. It should reuse the planner seam and the ledger
     reverse index.
   - **bob-cli-3q:** reuse the reverse index.
   - **bob-cli-3o:** retiring R8 readers removes legacy edges from BLOCKS membership
     automatically.
7. **Record** in a phase note:
   - commits and plugin versions per machine;
   - the dry-run summary and pilot output;
   - what is left for Bryan: reload nav, ledger-tools, cycler, and block-id-prompt on
     the Mac, then pilot a same-note and a cross-note Ctrl+Shift+P add, undo each, chip
     ×, a close, and a hand edit of a BLOCKS line.

## Deliberately not doing

- **Editing from the prerequisite side.** That means chip × / ＋ on BLOCKS rows, a Task
  Card Blocks row, and a Blocks… stage; it is bob-cli-3p.
- **Writes on close, cancel, or reopen.** Closed owners freeze; closed dependents drop
  at the next reconcile.
- **Listing implicit or transitive edges**, or residents of a project that waits on T.
- **New interfaces.** No new CLI command or flag, no `[blockedBy::]` or other Tasks
  field, and no capture grammar change.
- **Bob Mac Capture code.** Its preview shows the new line through existing target
  cards.
- **Using BLOCKS counts or 🔑 for priority, freshness, schedules, lanes, or review
  order.** Those need separate decisions.

## Risks

| Risk                                                                           | Mitigation                                                                                                              |
| ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| Recogniser sprawl: a fifth grammar misread as Task Links or tree-close targets | The registry plus MC vectors in every engine; guards and chips ship before any writer (`cli-reconcile` depends on them) |
| Engines render different bytes                                                 | One planner with a JS twin, the BK21/BW17 vectors, and code-point sorting                                               |
| A link to an id that does not exist                                            | Minting rides with the listing (same transaction, same batch), and reconcile skips quiet-interval mints (BR5)           |
| Dropping real entries                                                          | Never prune on uncertainty (BR17); malformed lines are left alone (BR16)                                                |
| Cross-note undo drift                                                          | Follow-ups are best-effort, failures are visible (BW3), undo re-arms the mirror (BW11), and reconcile repairs the rest  |
| Lag between reconcile runs outside Obsidian                                    | Chips render live; documented; Mac cron interval                                                                        |
| Another epic installs master early                                             | Writers depend on guards and chips, and the kill switch `dependencies.blocks_lines: false` turns it off losslessly      |
| Merge conflicts with bob-cli-7c                                                | New modules and fragments; rebase onto master and merge both sides of `package.json` / README rows                      |
