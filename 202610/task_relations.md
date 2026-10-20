---
tier: epic
title: 'Task relations: 🪢 LINKS, a generated 🪢 LINKED FROM mirror, and ~ / ~~ capture'
goal: 'Bryan can relate any task to any other task with an optional label, from Bob
  Mac Capture (`~` / `~~`) and from the Obsidian Task Card (`l` / `⇧L`). The relation
  is authored once on the source''s `🪢 **LINKS:**` line, and a generated `🪢 **LINKED
  FROM:**` line on the target always converges to it. Live chips render both ends
  correctly, and relations never block, promote, consume, schedule, stamp, or route
  anything.

  '
decisions:
  inverse_phrases:
    ask: How should a LINKED FROM chip show a known label from the target's side?
    choices:
      builtin: A fixed display-only table turns "merged into" into "merged from",
        and so on
      verbatim: Every label shows as written, followed by "→ this"
    default: builtin
    why: The chip then reads as a sentence from either end, and storage stays verbatim
    answer: builtin
  linked_from_remove:
    ask: Should live LINKED FROM chips get a hover × that removes the link from the
      source task?
    default: true
    why: Matches the BLOCKS chips' × (§15.8); the notice names the source note that
      changed
    answer: true
  memory_decision_record:
    ask: Add a decisions strand recording that task relations are inert, authored
      on the source, and mirrored?
    memory:
    - decisions
    default: false
    answer: false
  memory_glossary:
    ask: Add a "Task Relation" glossary term and a Task Link sentence saying LINKS
      entries are relations?
    memory:
    - glossary
    default: false
    answer: true
phases:
- id: contract
  title: Task-relation contract, capture grammar, wire names, and vectors
  depends_on: []
  size: medium
  description: 'contract: write docs/task-relations.md (grammar, slot, identity, projection,
    inertness, invariants, writers, gestures, chips, kill switch, outputs, and every
    vector table). Add the registry and MC rows in task-dependencies.md, the `~` /
    `~~` capture sections and wire names, capture-syntax rows, vault-format, task-status-hooks,
    and configuration. Docs only.'
- id: cli-core
  title: Rust relation grammar, planners, label rules, registry guards, and config
  depends_on:
  - contract
  size: medium
  description: 'cli-core: add src/native/task_relations/ (LINKS and LINKED FROM parsers,
    label normalisation, formatters, canonical sort, slot rules, and edit planners
    pinned by LK/LW). Add an is_managed_link_line registry predicate at every Task
    Link guard site, the relations.linked_from_lines config, and MC tests. No writer
    emits lines yet.'
- id: plugin-guards
  title: Plugin recognisers and gesture guards for relation lines
  depends_on:
  - contract
  size: medium
  description: 'plugin-guards: add LK recognisers to nav, the cycler, block-id-prompt,
    and the ledger plan budget. Keep relation lines out of Task Link, Pomodoro, tree-close,
    strike, `!`, and Ctrl+Shift+Enter paths. Give Ctrl+Enter and Alt+] parity with
    dependency lines. Add the LINKED FROM hand-edit notice and keep relation lines
    intact through project conversion. Deploy before any writer.'
- id: cli-reconcile
  title: Reconcile owns the LINKED FROM projection
  depends_on:
  - cli-core
  - plugin-guards
  size: medium
  description: 'cli-reconcile: collect relation edges from active notes, plus read-only
    previous daily and done/ sources. Add a reconcile_relation_lines pass that inserts,
    rewrites, deletes, and dedupes LINKED FROM lines, and mints source ids, with guarded
    output, never pruning on uncertainty, and the kill switch. Add JSON and human
    outputs, relation_warnings, LR tests, and a read-only live dry run.'
- id: capture-parse
  title: Capture `~` / `~~` lexer, guards, and capture-parse wire
  depends_on:
  - cli-core
  size: medium
  description: 'capture-parse: add a relation scanner in the `&` / `*` modifier runs
    with the digit, slash, strike, escape, and placement guards; labels after `|`;
    relation_* spans; relations[]; relation_target and relation_label needs; owner
    reuse; the relation_grammar signal; and a temporary execution refusal. Pinned
    by LP vectors.'
- id: capture-complete
  title: Relation completion pools, label suggestions, and task-id replacements
  depends_on:
  - capture-parse
  size: medium
  description: 'capture-complete: add contexts relation_to, relation_from, and relation_label;
    reuse the `&` catalog (open, closed, archived) with direction-specific eligibility,
    ranking, and badges. Label suggestions come from every LINKS line, and a typed-label
    row returns bob''s canonical spelling. Add the relation_to_replacement and relation_from_replacement
    task-id fields, the owner-picker relations[], and shell groups.'
- id: capture-execute
  title: Execute `~` / `~~` captures with a same-batch mirror
  depends_on:
  - capture-parse
  - plugin-guards
  size: medium
  description: 'capture-execute: plan relations for new and existing owners in the
    staged batch. Append or relabel the source''s LINKS line, mint the ids the edge
    needs, and stage the target''s LINKED FROM through the cli-core delta planner.
    Add refusals, inertness, relation_updates, linked_from_line_updates, task blocks,
    human output, and dry-run parity. Remove the parse-phase refusal. Pinned by LX
    vectors.'
- id: relation-index
  title: bob-ledger-tools live relation index and api.relationIndex
  depends_on:
  - plugin-guards
  size: medium
  description: 'relation-index: build a raw-text relation index in bob-ledger-tools.
    It does a prefiltered startup scan including done/, reparses per file on change,
    delete, and rename, overlays unsaved editor text, publishes forward and reverse
    maps together, and resolves targets by path plus block id. Publish it as api.relationIndex
    v1 with label suggestions. Measure cold and warm costs.'
- id: nav-writer
  title: nav relation grammar twin, planner, mint, writer, and api.relations
  depends_on:
  - plugin-guards
  size: medium
  description: 'nav-writer: add the JS twin of the LINKS / LINKED FROM planners (LW
    and LM bytes equal to Rust) and applyRelationEdit. It writes one same-note transaction,
    mints inside the source''s transaction, and sends cross-note mirror follow-ups
    with an honest failure suffix. Add the kill switch, notices, a machine-write annotation,
    and nav api.relations (removeLink now, openStage once the stages land, v1 after
    both). No UI.'
- id: chips
  title: bob-ledger-tools LINKS and LINKED FROM chips
  depends_on:
  - relation-index
  size: medium
  description: 'chips: render live chips for both relation rows in Live Preview and
    Reading view from the index, never the stored mirror. Add label pills, inverse
    phrases, ✓×N collapse, dashed stale and broken chips, and ＋ / × controls through
    nav api.relations. Reuse the bob-dep chip family with no new palette, and add
    LC tests.'
- id: nav-links-stage
  title: Task Card `l` Links to… and the Links stage with the relation field
  depends_on:
  - nav-writer
  - relation-index
  size: medium
  description: 'nav-links-stage: add the Task Card row `l` and Ctrl+Shift+L, the Links
    stage (open, closed, then archived pool; CURRENT, RESULTS, and CLOSED), ↵ link
    and ⇧↵ link with a relation, CURRENT unlink and relabel, Tab marks, ID prompts,
    the consequence strip, and Ctrl+Shift+P on a LINKS line opening the owner''s stage.'
- id: nav-linked-from
  title: Task Card `⇧L` Linked from…, counted gestures, hand-edit mirror, and Ctrl+D
  depends_on:
  - nav-links-stage
  size: medium
  description: 'nav-linked-from: add the reverse stage (⇧L, Ctrl+Shift+Alt+L, and
    Ctrl+Shift+P on a LINKED FROM line) that writes other tasks'' LINKS lines, counted
    N<Ctrl+Shift+L> sources, and the LINKS hand-edit mirror with undo re-arm. Ctrl+D
    on the Links to row removes every link, Ctrl+D on Linked from is refused, and
    LG vectors are tested.'
- id: mac-picker
  title: Bob Mac Capture `~` / `~~` pickers, live flip, and spans
  depends_on:
  - capture-complete
  size: medium
  description: 'mac-picker: give the direction-aware picker String sigils (& * ~ ~~),
    add the relation picker sources, the live flip when a second `~` is typed, relation
    span colours (a new knot accent), the Add-ID flow using bob''s relation replacements,
    and real-bob fixtures, with portable-check and a macOS typecheck.'
- id: mac-preview
  title: Bob Mac Capture relation card, preview rows, and capability warning
  depends_on:
  - mac-picker
  - capture-execute
  size: medium
  description: 'mac-preview: add the ⇧↩ relation card (field, suggestion chips with
    counts, live sentence, and bob''s inserted token), the 🪢 preview rows from relation_updates,
    managed-line suppression, the relation_grammar capability warning, docs, and fixtures.
    Verify on macOS CI.'
- id: rollout
  title: Install, pilot on a vault copy, reach the MacBook, guide, and memory
  depends_on:
  - cli-reconcile
  - capture-execute
  - chips
  - nav-linked-from
  - mac-preview
  size: medium
  description: 'rollout: install bob and the plugins on athena, then check that the
    live reconcile dry run shows zero relation changes. Run the pilot script on a
    vault copy, update the MacBook best-effort, write the user guide, add the memory
    records the accepted decisions allow, and record what is left for Bryan.'
proposed_by: bbugyi200.athena.research.53.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-20 09:08:19
status: done
bead_id: bob-cli-a2
---

- **PROMPT:** [prompts/202610/task_relations.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_relations.md)
- **BEAD:** [bob-cli-a2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a2/README.md)

# Plan: Task relations — 🪢 LINKS, a generated 🪢 LINKED FROM mirror, and `~` / `~~` capture

## Context

Bryan's request (2026-10-20):

> I would like to add support for directed links from one Obsidian task to another. […]
> display these links via a new `LINKS` task sub-bullet […] Tasks should also track
> their backlinks (via a new `BACKLINKS` sub-bullet?), which must always stay consistent
> and correct […] I need to be able to easily specify links when capturing new tasks in
> the bob-mac-capture app […] a new `~` syntax […] We also need a way to specify that
> another task should link to the current task in both Obsidian (via the
> `<ctrl+shift+p>` keymap?) and in the bob-mac-capture app (via a new `~~` syntax
> maybe?). The user should be able to specify an optional relation / link description
> […] I agree with all of the requirements recommended in that research file […] I want
> you to lead the design on this one. Make sure you design this feature so it is
> intuitive, reliable, and (last but not least) beautiful!

**Research.** `research:202610/task_relation_links/task_relation_links.md` (read it with
`sase artifact read research:202610/task_relation_links/task_relation_links.md "<why>"`)
consolidates five reports. Bryan accepts its requirement adjustments A1–A11, and this
plan adopts them:

- `BACKLINKS` becomes the generated `🪢 **LINKED FROM:**` line;
- "always consistent" becomes a tested guarantee (a live view plus a converging stored
  mirror);
- relations are **inert**;
- Ctrl+Shift+P stays the Task Card, with `l` / `⇧L` rows;
- labels are optional free text phrased from the source's side;
- any task with a block id can be a target, open or closed and archived included;
- `~` / `~~` are the capture sigils;
- there is no `[links::]` field and no new CLI command.

The plan settles the report's six open questions:

1. **Names and glyph:** `🪢 LINKS` / `🪢 LINKED FROM` (A1).
2. **Stored mirror:** yes, like BLOCKS (A2), with a lossless kill switch.
3. **Archived targets:** no stored mirror in `done/`. Chips show the live view.
4. **Cancel as duplicate of / merged into / superseded by:** deferred to a follow-up
   (see Deliberately not doing).
5. **Inverse wording:** a display-only table, embedded as the `inverse_phrases`
   decision.
6. **Adopting hand-written `Merged into` / `See [[…]]` / `RELATED:` bullets:** deferred.

**Where this plan deviates from the research, and why:**

| Research                                              | This plan                                                                                          | Why                                                                                                                                                                                                                                                         |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reader accepts `RELATED` / `SEE ALSO` as LINKS labels | Accept only `LINKS` / `LINKS TO`                                                                   | `sase_gate.md:11` already has a bold `- **RELATED:** [[sase_better_plans]]` under a `#task`. It is a note link with no block id, so it would warn `malformed_links_line` on every reconcile, forever. Adopting old idioms is an explicit, opt-in follow-up. |
| Quoted labels with `\"` / `\\` escapes                | Labels may not contain `"` or `\`, so a quoted label never needs an escape                         | One fewer escape grammar. Bob Mac Capture can build `\|"…"` probes without knowing escape rules.                                                                                                                                                            |
| Kill switch `links.linked_from_lines`                 | `relations.linked_from_lines`                                                                      | The concept is a _relation_ everywhere in docs and JSON. A `links:` section would read like Pomodoro Task Links.                                                                                                                                            |
| Reconcile owns "relation line updates"                | Reconcile writes only LINKED FROM lines and source mints; it never rewrites an authored LINKS line | Authored data changes only through gestures and hand edits. Writers canonicalise a LINKS line when they touch it.                                                                                                                                           |
| LINKED FROM chips get ＋ only                         | ＋, plus hover × under `linked_from_remove`                                                        | Parity with the shipped BLOCKS chip × (§15.8).                                                                                                                                                                                                              |
| "Make this a prerequisite" row action                 | Information badge only                                                                             | Converting an edge crosses into dependency semantics. That is a follow-up.                                                                                                                                                                                  |
| A new `relation_owner` need                           | Reuse `dependency_target`, mode `task_dependency`, and `relationship_needs_text`                   | Capture already calls `&` / `*` _relationship modifiers_, so `~` / `~~` join that umbrella. Old Mac builds keep working, and the docs say the names are historical.                                                                                         |
| An abstract capability flag                           | Top-level `relation_grammar: 1` in `capture-parse` whenever the draft contains `~`                 | Detected per response, as `docs/protocol.md` requires. Drafts without `~` stay byte-identical.                                                                                                                                                              |

### Words (use them exactly; they are not interchangeable)

- **Task relation** (or **relation**): a directed, optionally labelled, inert edge from
  a **source** task S to a **target** task T. It is stored as one entry on S's LINKS
  line.
- **Relationship modifiers**: capture's existing umbrella for task-to-task modifiers. It
  now covers `&` (depends on), `*` (blocks), `~` (links to), and `~~` (linked from).
  Existing JSON names such as `picker.relationship`, `dependency_target`,
  `relationship_needs_text`, and mode `task_dependency` keep their spelling and now
  cover all four.
- **Task Link** (glossary) keeps its meaning. Entries on a LINKS line are block links to
  tasks, but they are **never** Pomodoro Task Links.

### What exists now

Verified at bob-cli `d51c160e`, bob-plugins (nav 2.46.1, ledger-tools 1.59.1, cycler
1.42.1, block-id-prompt 1.33.2), and bob-mac-capture master. Line numbers are
approximate, so search by name.

**bob-cli**

- **Dependency and BLOCKS grammar** lives in `src/native/task_dependencies/`.
  - `blocks.rs` (581 lines) is the model to copy. It has `BlocksLine`,
    `parse_blocks_line`, `blocks_children_of`, `sort_blocks_links` (BK21), `format_*`,
    `blocks_insert_slot`, `blocks_desired_after_delta`, and `plan_blocks_line` →
    `BlocksLinePlan`, with `BlocksUpdateKind`.
  - `mod.rs` holds the shared scanners `block_link_spans`, `raw_wikilink_spans`,
    `strikethrough_spans`, and `inline_code_spans`, plus `is_managed_dependency_line`
    and `is_archive_path`.
  - `format.rs` holds `canonical_link`, `child_indent_for_parent`, and
    `note_line_ending`.
- **Managed-line guards** call `is_managed_dependency_line` from:
  - `capture_completable_tasks.rs:329`;
  - `task_complete/tree.rs:192`;
  - `plan_budget/mod.rs:550`;
  - `task_status_hooks/pomodoro.rs:63`;
  - `capture_pomodoro_close/ledger.rs:585`;
  - `projects/sync.rs:875`. This one is the Sub-projects anchor, which must stay
    dependency-only.
- **Reconcile.** `task_status_hooks/sync.rs::sync_task_statuses` runs the passes in this
  order:
  1. It reads every active note into `FileScan`s. `markdown_files` skips `done/`,
     `_generated/`, and dot-directories.
  2. It finds `previous_daily_path`.
  3. It runs R1–R10 and re-parses.
  4. It runs `reconcile_blocks_lines` (`task_status_hooks/blocks.rs`, 545 lines), which
     is wired at ≈L489–518 with `load_blocks_lines_enabled`, `quiet_deferred_files`, and
     the structural-files merge.
  5. It runs the status loop and `apply_guarded_outputs`.

  `archive_reference_catalog` (`references.rs`) loads only exact `done/…` reference
  targets, read-only. `docs/task-status-hooks.md` (≈L107, ≈L789) documents that narrow
  `done/` read.

- **Capture `&` / `*`.**
  - `capture_language/dependencies.rs` (1660 lines) has `scan_modifier_with_sigil`,
    `extract_line_dependencies` / `extract_line_relation`, `is_trailing_transparent`,
    the escape helpers, `operator_relation_error`, `resolve_execution_ownership`, and
    `relation_modifier_raws`. Today "relation" in these names means `&` / `*`.
  - `capture/dependencies.rs` and `capture/blocks_mirror.rs` (365 lines) are the
    same-batch mirror model: staged text is re-read between writes, mints use batch
    reservations, and target touches are recorded.
  - `capture_dependency_tasks.rs::discover` already catalogs **every** task (open,
    closed, hidden, and `done/` archives) with groups `in_this_note`, `in_progress`,
    `next`, `open`, and `completed`, plus `requires_block_id` and
    `block_id_suggestions`.
  - `capture_complete/{engine,candidates,render,shell}.rs` and `capture_task_id.rs`
    (`dependency_replacement` / `blocked_task_replacement`) round this out.
- **Config.** `config/dependencies.rs` holds `blocks_lines`, the kill-switch model.
- **Archive and move.** `collect_done/link_repair.rs` and `task_move/` rewrite inbound
  links on every line.

**bob-plugins** (open with `sase repo open bob-plugins -r "<why>"`; read its
`AGENTS.md`). Paths are relative to `plugins/<plugin>/src`. All fragments must stay at
or below 1000 lines, and there is no cross-plugin shared code.

- **nav Task Card.**
  - `310-task-card-previews.js` holds `planTaskCard`, `resolveTaskCardKey`,
    `TASK_CARD_ROW_ORDER` (schedule, depends-on "Blocked by…" `b`, blocks "Blocks…"
    `⇧B`, review-every `f`, lane Alt+N, cancel `x`), `TASK_CARD_ROW_SHORTCUTS`, and the
    `actionKey` regex `/^[0-9bfx]$/i`.
  - Rows are presented and opened by `320` (`presentTaskCardRow`) and `350`
    (`openTaskCardAction`, `dispatchInitialTaskCardRow`).
  - **`l` and `⇧L` are free on the card.** `⇧L` means "Less often" only on the separate
    decay card.
  - Chords are set in `490-plugin-lifecycle.js` (Ctrl+Shift+B ≈128, Ctrl+Shift+Alt+B
    ≈136). No plugin binds Ctrl+Shift+L or Ctrl+Shift+Alt+L. The counted-chord map is
    `610::countedBulletPropertyChord`, which honours Alt only for `KeyB`.
- **nav stages.**
  - The pool and ranking live in `290-local-task-items.js`. It is open-only (`290:294`)
    and excludes daily notes and `done/` (`isDependencyStageExcludedPath`).
  - The views are `300` and `302`; Blocks stage copy is in `303`.
  - The picker parts are `340`, `400` (`showBlockIdStage`), `410`, `425`
    (`showBlocksStage`), and `450`.
  - The reverse writer `applyBlocksEdit` is in `592`, the follow-ups in `591`, and the
    planners in `081` (`planBlocksLineEdit`), `082` (`mintDependentBlockId`), `083`, and
    `084`.
  - The hand-edit mirror is `600`, and its listener skips
    `BLOCKS_MACHINE_WRITE_ANNOTATION` (`600:623–634`).
  - Notices come from `216` (`withBlocksFollowUpNotice`), the api from `485` / `593`
    (`api.blocks`), and the kill switch from `020::loadBlocksLinesEnabled`.
  - Free slots: 085–089, 304–309, 312–319, 421–424, 426–429, 486–489, 594–599.
  - Full fragments: `590` (1000), `080` (998), `570` (996), `302` (941).
- **bob-ledger-tools.**
  - Depends-On chips: `140` / `150` / `160` / `270` / `280`.
  - BLOCKS chips: `141` / `152` / `272` / `281`. They are built from the Tasks memo,
    which **cannot see custom child lines**.
  - `setupDependencyChips` owns the `toggle-dependency-chips` command and the
    `bob-dep-chips` body class.
  - Per-file index patterns: `170-plugin-lifecycle.js:432–496`
    (`metadataCache.on("changed")`), `190::refreshNoteReadyForChangedFile` (per-path map
    plus generation, delete and rename), and `290::refreshTodayCacheForChangedFile` (raw
    text).
  - `050::planBlockLinks` is the plan-budget guard.
  - Free slots: 144–149, 151, 153–159, 161–169, 278, 279, 282–289.
  - `170` is at 958 lines, so add a new lifecycle fragment rather than growing it.
- **task-status-cycler** has its recognisers in `050` (`isTaskDependencyLine`,
  `isTaskBlocksLine`, `isManagedDependencyLine`). Its guard sites are:
  - `050:388`, tree close;
  - `080:675` / `080:756`, strike and restore;
  - `060:546`, Pomodoro sub-bullets;
  - `085:201`, successor live link;
  - `090:585` / `150` / `160:496` / `190`, Alt+] and Ctrl+Enter on a link.

  **Trap:** `isManagedDependencyLine` means both "never a Task Link" and "act on the
  link under the cursor".

- **block-id-prompt** has its recognisers in `060` (984 lines, nearly full). Its guard
  sites are `060:478`, `075:211`, `120:468–475`, and `130:22–28`. The Ctrl+Shift+Enter
  notices are in `010:70`.
- **Task move.** `nav/250::rewriteTaskMoveBlockLinks` already rewrites block links on
  every non-fenced line, relation lines included.

**bob-mac-capture** (open with `sase repo open bob-mac-capture -r "<why>"`; read
`AGENTS.md`).

- **Picker direction.** `Sources/CaptureCore/DependencyPickerPresentation.swift`'s
  `DependencyPickerDirection` has cases `.prerequisite` / `.blocks` and
  **`leadingSigil: Character`**, which cannot hold `~~`. The same one-character
  assumption appears in `FuzzyQuery`, `cursor > r.start + 1`
  (`handleDependencyCompletion`, `openPickerFromChip`), `triggerByte: UInt8`,
  `removePickerTrigger`, and `rememberRelationshipDisplayName` (`dropFirst()`).
- **Picker sources and copy.** `CapturePickerPresentation.swift` has
  `CapturePickerSource` (`.dependency` / `.blockedTask`) and `CapturePickerNeed`.
  `CaptureRelationshipCopy.swift` and `CaptureRelationshipPresentation.swift` own the ⛓
  / 🧱 rows.
- **Panel model.** `Sources/BobMacCapture/CapturePanelModel.swift` (≈8.3k lines) holds
  `pickerNeed`, `completionNeeds` / `completionSpanKinds`, `handleDependencyCompletion`,
  `continuePickerOperator`, and `pickerAutoOpenSuppressedStart`. The last one would
  block a same-start re-open on flip.
- **Panel view.** `CapturePanelView.swift` has `TaskIDPromptCard` and
  `PomodoroNamePromptCard`; the latter is the template for a free-text card. It also has
  `PreviewPane` with `suppressRelationshipLines` / `isManagedRelationshipLine`.
- **Spans and colours.** `CompletionRowContent.swift::captureSemanticCategory` maps span
  kinds to colours; unknown kinds render `.neutral`. `CaptureEditorPalette` has unused
  `.mint` and `.brown`.
- **Decoding.** Decoders tolerate unknown keys. New fields must use the
  `(try? decodeIfPresent) ?? default` form, never a throwing `decodeIfPresent`.
- **Verification.** `just portable-check` runs on athena (Swift 6.0.3, CaptureCore
  only). macOS targets need a scratch build on the Mac (`bundle.sh`,
  `just typecheck-tests`) and macOS CI. The template commit is `f2a5cb5` (the `*`
  picker).

**Vault (read-only, 2026-10-20).**

- 6,427 notes and 29 top-level `done/` archives, plus `done/2026/`.
- 660 non-bold `- LINKS:` lines in `ref/`, none under a `#task`.
- One bold `**RELATED:**` task child (`sase_gate.md:11`).
- No `🪢` relation lines yet.

## Design

### What Bryan sees

**In the Markdown** (vim, git, `rg`, and agents read it as English):

```markdown
- [ ] #task Fix flaky capture retries #bob ^fix-retries
  - 🪢 **LINKS:** follow-up of [[done/bob_done#^capture-star]] • [[sase#^retry-design]]

- [ ] #task Design a retry policy ^retry-design
  - 🪢 **LINKED FROM:** [[bob#^fix-retries]]

- [-] #task Old retry spike ^retry-spike
  - ❌ **CANCEL LOG**
  - 🪢 **LINKS:** merged into [[bob#^fix-retries]]
```

**In capture** (CLI or Bob Mac Capture):

```text
Fix flaky capture retries ~bob_done:capture-star|"follow-up of" ~sase:retry-design   # this → those
Write the outline ~~sase_blog:prj|"next step"                                       # that → this
@bob+fix-retries ~sase:retry-design|see                                             # two existing tasks
```

```text
✓ Fix flaky capture retries  mac_inbox.md ^fix-flaky-capture-retries   [ ] Ready
  🪢 links to "Capture * blocks existing tasks" · follow-up of   done/bob_done.md ^capture-star · archived
  🪢 links to "Design a retry policy"   sase.md ^retry-design
```

**In Bob Mac Capture**, typing `~` opens the task picker. A second `~` flips it, and ⇧↩
opens the relation card:

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ Fix flaky capture retries ~capture st▏                                   │
├──────────────────────────────────────────────────────────────────────────┤
│ 🪢 "Fix flaky capture retries" links to…       any task · ~ flips        │
│   ○  Capture star picker polish                   bob:star-polish        │
│ DONE                                                                     │
│   ✓  Capture * blocks existing tasks              bob_done:capture-star  │
│ ↩ link · ⇧↩ link with a relation · ~ flips direction                    │
└──────────────────────────────────────────────────────────────────────────┘
┌ 🪢 Relation ─────────────────────────────────────────────────────────────┐
│ [ follow-up of▏                ]   follow-up of 12 · see 7 · merged into 3 │
│ Fix flaky capture retries — follow-up of → Capture * blocks existing tasks │
│ Inserts ~bob_done:capture-star|"follow-up of"   ↩ insert · esc back        │
└──────────────────────────────────────────────────────────────────────────┘
```

**In Obsidian**, the Task Card (Ctrl+Shift+P) gains two rows after Blocks…:

```text
 Fix flaky capture retries · bob · Ready · P2
 ──────────────────────────────────────────────
 ↵   Schedule…
 b   Blocked by…
 ⇧B  Blocks…
 l   Links to…        2 · follow-up of Capture *… +1
 ⇧L  Linked from…     1 · Design a retry policy
 f   Review every…
```

Both rows also have direct chords: Ctrl+Shift+L and Ctrl+Shift+Alt+L. In both stages, ↵
links and ⇧↵ links with a relation. On a CURRENT row, ↵ unlinks and ⇧↵ relabels. The
chips read as sentences from either end:

```text
○ Fix flaky capture retries   #bob
   🪢 links to     follow-up of → ✓ Capture * blocks existing tasks ↗ bob_done
                   → ○ Design a retry policy ↗ sase                               ＋
○ Design a retry policy
   🪢 linked from  ○ Fix flaky capture retries ↗ bob                              ＋
○ Fix flaky capture retries
   🪢 linked from  merged from ✕ Old retry spike                                  ＋
```

> [!decision] inverse_phrases = verbatim The last chip row reads
> `✕ Old retry spike · merged into → this`; no inverse table exists anywhere.

### Rule 1: inert — what relations never do

This list goes into the contract and the decision record. It is the main reliability
lever. A relation write, a relation line, or a relation gesture never:

- blocks, unblocks, promotes, releases, or consumes anything; never flips a checkbox to
  `[?]` or recovers one;
- counts as a Pomodoro Task Link, plan-budget item, completable sighting, successor
  anchor, or successor offer. Ctrl+Shift+Enter, `!`, and tree close never act on a
  relation line;
- changes a status, lane, `[fresh::]`, `[keeps::]`, `[unblocked::]`, schedule, priority,
  `[id::]`, or `[dependsOn::]`. The only task-line edit it may make is appending a
  minted `^block-id`;
- counts as a review answer, advances the `]s` walk, becomes a walk tier, or triggers
  inbox routing;
- disappears because either end closes, is cancelled, reopens, or is archived. Relations
  are history.

A link that needs to block is a dependency (`&`, `b`).

### Rule 2: grammar

```text
<indent>- 🪢 **LINKS:** <entry> • <entry> • …
<entry>        := [<label> ' ']<link>
<indent>- 🪢 **LINKED FROM:** <mirror-entry> • <mirror-entry> • …
<mirror-entry> := <link>[' (' <label> ')']
<link>         := [[#^id]] | [[basename#^id]] | [[dir/note#^id]]
```

- **Writer form.** It starts with `🪢` (U+1FAA2, one code point, no VS16), one space,
  the bold label, and one space. Entries are joined by `•`. There are no aliases,
  embeds, strikes, or duplicate targets. A line that would be empty is deleted, with no
  placeholder.
- **Labels.** A label is optional and reads "S ‹label› T". Writers trim it and collapse
  whitespace runs to one space, strip a trailing `:`, and require 1–40 characters.
  - A label may not contain `[`, `]`, `(`, `)`, `"`, `\`, `|`, `•`, `*`, `~`, or a
    control character.
  - It may not start with `·` or `,`.
  - Case is preserved. Suggestions dedupe case-insensitively.
  - Readers accept any length that obeys the character rules.
- **Parse algorithm.** Use the §2.3 approach of `docs/task-dependencies.md`: run
  `block_link_spans` first and never split on separators.
  - After each link, skip one separator (`•`, `·`, or `,`) and whitespace. The text up
    to the next link is that entry's label.
  - A struck (`~~[[…]]~~`) or embedded (`![[…]]`) link is accepted and canonicalised.
  - An aliased link's alias becomes the label only when no label precedes it.
  - On LINKED FROM, a `(label)` after a link is its label.
- **Reader tolerance.**
  - The glyph may be missing or be `🔗`.
  - The authored label may be `LINKS` or `LINKS TO`; the mirror label may be
    `LINKED FROM` or `BACKLINKS`. Labels must be bold and uppercase.
  - `·`, `,`, or whitespace-only separators are accepted.
  - The line may be in any direct-child slot.
- **Not a line:**
  - non-bold `LINKS:` (the 660 `ref/` lines);
  - lowercase labels, and `RELATED` / `SEE ALSO`;
  - anything that is not a direct child of a `#task`;
  - fenced, nested, blockquoted, or marker-less lines.
- **Malformed** means:
  - text after the last link;
  - a label that breaks the character rules;
  - a half-typed link, a bare note link, or a heading link;
  - separators only.

  A malformed line is left untouched and warned about (`malformed_links_line` /
  `malformed_linked_from_line`). Writers never insert a second line beside it. An
  authored write to a malformed LINKS line is refused with "its LINKS line is
  malformed".

- **Cross-parse.** The Depends-On, BLOCKS, LINKS, and LINKED FROM parsers each return
  not-a-line for the others' lines.

### Rule 3: slot, order, and duplicates

```text
- [ ] #task T ^t
  - ❌ **CANCEL LOG**        (first, if present)
  - ⛓️ **DEPENDS ON:** …     upstream
  - 🧱 **BLOCKS:** …         downstream
  - 🧩 **Sub-projects:** …   structure (^prj only)
  - 🪢 **LINKS:** …          what T relates to (authored)
  - 🪢 **LINKED FROM:** …    what relates to T (generated)
  - user children, 🗓️ SCHEDULE LOG, 🛠️ WORK LOG …
```

- **Slots.** LINKS goes after the last direct-child subtree of Cancel Log, Depends-On,
  BLOCKS, or Sub-projects. Otherwise it is the first direct child (the second after a
  first-child Cancel Log). LINKED FROM goes after the LINKS subtree when one exists,
  otherwise in the LINKS slot.
  - Reuse the task's existing child indent, otherwise use the parent indent plus one
    tab.
  - Preserve CRLF and the final-newline state.
- **Existing lines are never moved.** A Depends-On, BLOCKS, or Sub-projects line
  inserted later lands in its own earlier slot, so the order still reads upstream,
  downstream, structure, out, then in. `bob projects sync`'s anchor rule (after
  Depends-On and BLOCKS) is unchanged.
- **LINKS order is authored.** New entries are appended. Re-linking an existing target
  with a different label relabels it in place; with the same label it is a no-op.
- **LINKED FROM order is canonical.** Sort by canonical link text after ASCII
  case-folding, comparing Unicode code points, with raw code-point order as the
  tie-break: exactly BK21. Same-note entries therefore come first. JS compares code
  points, never UTF-16 units.
- **Duplicates.** A writer that touches a task with two LINKS lines merges the later
  line's entries into the first (appended, first label wins) and deletes the later one.
  On duplicate targets within one line, the first entry wins. Reconcile keeps the first
  LINKED FROM line and deletes the rest.

### Rule 4: identity and minting

- **Edge identity is (S, T),** each resolved by vault-relative path plus block id. A
  source holds at most one entry per target. A → B and B → A are independent edges, and
  cycles are allowed. Self-relations and non-task targets are refused.
- **The target needs a `^block-id`.** Pickers show `＋ id` and run the existing
  new-block-ID step (nav `showBlockIdStage`; capture `requires_block_id` /
  `block_id_suggestions` plus `capture-task-id`). Completion, dry-run, and Escape mint
  nothing. An id-less archived target is disabled ("archived tasks can't get an id").
- **The source needs a `^block-id` only for the mirror.** It is minted silently with
  `mint_block_id(description, used)` (SB, §11.7), once, inside the same transaction or
  batch, exactly like BLOCKS dependents (§14.4).
  - No `[id::]` is ever added.
  - When `relations.linked_from_lines` is false, no source is minted.
  - In `~~`, the new task is the _target_, so it is always minted.
- **Never emit a link to an id that will not be on disk after the write.** Reconcile
  skips listing a source whose mint would land in a note that is quiet-deferred this run
  (BR5).
- **Link form.** Use `canonical_link` from the owning note: `[[#^id]]`, then
  `[[basename#^id]]`, then `[[dir/note#^id]]`. A `done/` source or target always keeps
  the explicit `[[done/…#^id]]` form, the only form the hooks resolve into archives.
- **A broken link is never rebound** by title or by a reused id. Supported moves
  (`bob task move`, Ctrl+Shift+M, `bob task archive`, and Obsidian renames) already
  rewrite links on every line.

### Rule 5: the LINKED FROM projection

```text
sources   = #task lines with a LINKS line in: active notes (writable)
            ∪ the previous daily note (read-only) ∪ every .md under done/ (read-only)
edges     = { (S, T, label) : an entry on S's LINKS resolves to #task T ∧ T ≠ S }
            (unresolved, non-task, self, and malformed are warned and omitted; never pruned from LINKS)
mirror(T) = sort_canonical({ (S, label) : (S, T, label) ∈ edges })
T owns a LINKED FROM line ⇔ mirror(T) ≠ ∅ ∧ T is writable (not done/, not the previous daily note)
```

- **Open or closed T** gets a maintained mirror. Unlike BLOCKS, it never freezes.
- **Never prune on uncertainty.** If S's LINKS line is malformed, or S's note has
  duplicate block ids, existing LINKED FROM entries naming S are kept and S is never
  newly added.
- **The `done/` read is narrow.** Byte-prefilter every `done/**/*.md` for `LINKS`, parse
  only those files, and never write there. Document it beside the exact-`done/`-target
  exception in `docs/task-status-hooks.md`.

### Rule 6: authority (reconcile)

Reconcile is the authority for LINKED FROM. Gestures and capture make early courtesy
writes, and closes, vim edits, agent edits, moves, and failed follow-ups are repaired by
reconcile.

1. **New pass `reconcile_relation_lines`** in `task_status_hooks/relations.rs`. It runs
   right after `reconcile_blocks_lines`, over the re-parsed text, with the same
   structure.
2. **Owners to check:** every T with mirror(T) ≠ ∅, plus every writable task that
   already has a LINKED FROM child. Byte-prefilter for `LINKED FROM` / `BACKLINKS`;
   never walk every task's children.
3. **Steps for each owner:** mint missing source ids, render against the post-mint
   state, then call the cli-core mirror planner with the desired entries:

   | Existing line                   | Result                                                     |
   | ------------------------------- | ---------------------------------------------------------- |
   | missing                         | `inserted`                                                 |
   | different entries or labels     | `rewritten`                                                |
   | same entries, non-canonical     | `canonicalized`                                            |
   | not desired                     | `removed`                                                  |
   | duplicates                      | keep the first (`deduplicated`)                            |
   | any malformed LINKED FROM child | leave every LINKED FROM child of that owner alone and warn |

4. **Guarded output.** The edits are structural. They join the reconciled set, so the 2
   s quiet interval, preimage checks, recovery snapshots, and `pre_reconcile` originals
   all apply.
5. **Reconcile never edits a LINKS line.** It only reads, warns, and mints.

### Rule 7: immediate writers

| Writer                                                                  | Behaviour                                                                                                                                                                                                                                                                                               |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| capture `~` / `~~` (CLI and Bob Mac Capture through bob)                | S's LINKS edit, any minted ids, and T's LINKED FROM edit go into **one staged batch** via the cli-core delta planner. A skipped mirror (archived or previous-daily T, a malformed mirror, or the kill switch) is reported, never fatal.                                                                 |
| nav Links / Linked-from stages, chip ×, Ctrl+D, api `removeLink`        | Same note: one transaction, one Ctrl+Z. Cross note: S commits first, then a best-effort follow-up per touched T (open editor with the relations machine-write annotation, else a preimage-checked `vault.process`). On failure the notice gains `· 🪢 LINKED FROM not updated`, and nothing rolls back. |
| nav hand-edit mirror (cursor leaves an edited LINKS line; undo re-arms) | Diff the burst baseline's (target, label) set against the final set, then mirror the difference as follow-ups.                                                                                                                                                                                          |
| Hand edit of LINKED FROM                                                | Never an edge change. A notice appears once per burst, and reconcile restores the line.                                                                                                                                                                                                                 |
| Ctrl+Shift+M, `bob task move`, `bob task archive`, Obsidian rename      | The LINKS line moves with its task, and generic link rebasing applies. Reconcile re-sorts and repairs the mirrors.                                                                                                                                                                                      |
| Closes, cancels, reopens                                                | None.                                                                                                                                                                                                                                                                                                   |

All writers share one planner, implemented in Rust with a JS twin and pinned by the same
LW / LM vectors. Writers compare canonical output before writing, so a no-op writes
nothing and adds no undo entry. Writes stamp nothing.

### Rule 8: gestures on relation lines (Obsidian)

| Gesture                              | LINKS line                                                                                               | LINKED FROM line                                                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Ctrl+Shift+P                         | Opens the owner's Links stage                                                                            | Opens the owner's Linked-from stage                                                                          |
| Ctrl+Shift+L / Ctrl+Shift+Alt+L      | Act on the owning task, never the chip under the cursor                                                  | Same                                                                                                         |
| `!` / `N!`, Ctrl+Shift+Enter         | Refused: `🪢 LINKS entries are relations, not Task Links · edit them with Ctrl+Shift+L`                  | Refused with the generated-line notice                                                                       |
| Ctrl+Enter / Alt+] / Alt+[ on a link | Acts on the linked task root-only (no strike, re-embed, tree close, or reformat), as on dependency lines | Same                                                                                                         |
| Hand edit                            | Mirrored when the cursor leaves the line                                                                 | `🪢 LINKED FROM is generated from other tasks' LINKS lines · change it with Ctrl+Shift+Alt+L` once per burst |
| Ctrl+D on the Task Card row          | Links to: removes every outgoing link and its mirror entries                                             | Linked from: refused with the generated-line notice                                                          |

### Rule 9: chips (bob-ledger-tools)

- **Data.** A live raw-text relation index (Rule 10), never the stored mirror. Chips
  render the live set:
  - a stored LINKED FROM entry that is no longer live is dashed, with the tooltip
    `No longer links here · refreshes on the next reconcile`;
  - a live edge missing from the text is appended in canonical order;
  - a LINKS entry whose target cannot be resolved is a dashed `⚠ ^id not found`.
- **Anatomy.** Reuse the `bob-dep-row` / `bob-dep-chip` family: a mini status checkbox
  from `--task-status-*`, about 40 characters of description, `↗ note` for cross-note
  entries, a hover preview, and click / Mod-click.
  - The row accent stays neutral. **Colour carries status, never direction.**
  - Closed and archived targets are dimmed and never struck.
  - `#hide` tasks are muted.
  - There is no new palette, and no `frees` / `waiting on` badges.
- **Labels** are a muted small-caps pill on the arrow side.
  - LINKS: `‹label› → (chip)`.
  - LINKED FROM: `‹inverse› (chip)` for a known label (decision `inverse_phrases`),
    otherwise `(chip) · ‹label› → this`.
  - Rows with labels wrap one entry per visual row. Unlabelled rows wrap between chips.
    Chips never change line height.
- **Collapse.** Three or more closed chips on a row collapse to `✓×N`, which expands on
  click.
- **Controls** (only when nav `api.relations?.version >= 1`):
  - a trailing ＋ on LINKS (aria "Link this task to another") opens the Links stage;
  - a trailing ＋ on LINKED FROM (aria "Choose a task that links here") opens the
    Linked-from stage;
  - a hover × on each live LINKS chip (aria "Unlink <name>") removes the link.

  > [!decision] linked_from_remove Each live LINKED FROM chip also gets a hover × (aria
  > "Remove the link from <name>") that calls `removeLink(S, T)`. The notice names S's
  > note.

  > [!decision] linked_from_remove = no LINKED FROM rows get ＋ only. Their tooltip
  > reads `Generated from other tasks' LINKS lines · ＋ to add (Ctrl+Shift+Alt+L)`.

- **Toggle.** `toggle-dependency-chips` covers both relation rows too.
- **Without the plugin**, both lines still read as English.

**Inverse phrases** (display only; case-insensitive match; never stored):

| Stored label    | Shown on LINKED FROM |
| --------------- | -------------------- |
| `merged into`   | `merged from`        |
| `duplicate of`  | `duplicated by`      |
| `follow-up of`  | `followed up by`     |
| `superseded by` | `supersedes`         |
| `split from`    | `split into`         |

The nav Linked-from stage's CURRENT rows use the same table. Under
`inverse_phrases = verbatim`, the table does not exist.

### Rule 10: the live relation index (bob-ledger-tools, `api.relationIndex`)

- **Startup scan.** Run a chunked, generation-guarded scan with a cheap prefilter: skip
  files whose metadata cache has no `#^` block links, then text-prefilter for `🪢`,
  `**LINKS`, `**LINKED FROM`, or `**BACKLINKS`. The scan includes `done/`, as read-only
  sources.
- **Updates.** Reparse per file on `metadataCache.changed`, and handle vault `delete`
  and `rename`, following the `190` per-path map pattern. Overlay the active editor's
  unsaved text, so a just-typed entry shows at the target before save.
- **Publishing.** Forward (`S → [(T, label)]`) and reverse (`T → [(S, label)]`) maps are
  published together under one generation, so a count and its rows never disagree.
  Targets resolve by `getFirstLinkpathDest` plus the file's cached block ids. Status and
  text come from the Tasks memo, falling back to the cached line for notes the memo
  omits (`done/`).
- **API.** `api.relationIndex` v1 exposes `version`, `generation`, `ready()`,
  `outgoing(path, blockId|line)`, `incoming(path, blockId)`, `labelSuggestions(prefix?)`
  → `[{label, count}]` (case-insensitive dedupe, count-ranked), and `onChange(cb)`.
  - nav reads it for the Task Card details, the Linked-from stage CURRENT, and label
    suggestions.
  - Without it, nav falls back to a generation-guarded chunked vault scan when a stage
    opens, and omits the card details.
- **No disk cache.** The index is never needed to recover authored data.

### Rule 11: capture grammar (`~` / `~~`)

```text
~<note>:<block-id>[|<label>]     this item's task links to <note>:<block-id>
~~<note>:<block-id>[|<label>]    <note>:<block-id> links to this item's task
<note>  := as for & and *  (route, nested route, or "quoted name")
<label> := a run of non-space label characters, or "<label text>" (no escapes; Rule 2 characters)
```

`~` / `~~` is a directive only when **all** of these hold. The rules are modelled on the
shipped `*` rules (`capture_language/dependencies.rs`, `docs/capture.md`):

1. **Placement.** The token is whitespace-delimited and sits in the same leading run
   (parent line only) or trailing run as `&` / `*`. Both runs skip
   `is_trailing_transparent` shapes. A mid-line `~foo:bar` stays body text.
2. **Next byte.** The byte after the sigil is not whitespace or `/`, so `~/path` stays
   literal. It is not a digit when the token has no `:`, so `~5m`, `~2h`, and `~5pm`
   stay literal.
3. **Strike guard.** A token that starts _and_ ends with `~~` with text between is
   strikethrough body. A `~~` token contains no further `~` (`~~~x:y` is invalid).
   Inside session tokens, `x~~1` and `=~~2` are errors ("use one `~` list"), never
   relations. `=~2` and `x~1` are unchanged.
4. **Bare sigil.** A bare `~` or `~~` opens the picker only as the last token of a line
   with other text, or as the first token of a parent line (as `&` allows). An
   unresolved bare or partial sigil is refused on submit with:
   ``"`~` needs a task; choose one, or write `\~` for a literal tilde"``.
5. **Escapes and exclusions.** `\~` consumes only the backslash, and only where the
   token would otherwise be a directive. `[[…]]` and backtick spans are never scanned.
6. **Coexistence.** An item may carry several `~` / `~~` modifiers alongside `&` / `*`.
   A relation after `x` is refused (`operator_relation_error`). Each bulk-capture item
   is judged on its own.
7. **Owners.** Owners use exactly the `&` / `*` spellings: task text creates the owner,
   `@route+id` names an existing owner, `@@route+id` is inherited, and bare `@route:id`
   / `^route:id` alias without linking. A relation-only existing-owner item uses mode
   `task_dependency` and reports `relationship_needs_text` as `&` does.
8. **Targets.**
   - A `~` target is any task with a block id: open, closed, any active note, a daily
     note, or `done/`. A mirror is skipped for archived and previous-daily targets.
   - A `~~` target T has its **LINKS line written**, so it must be editable. The refusal
     set is the same as `*`'s: daily notes, `done/`, `_templates`, `_generated`,
     `_conflicts`, dot-directories, and fenced code.
   - The same target twice in one item is a no-op with the same label, and refused with
     a different label.

### Rule 12: wire contract (additive; every `schema_version` stays 1; no new subcommand or option)

- **`capture-parse`.**
  - Span kinds: `relation_sigil`, `relation_note`, `relation_block_id`,
    `relation_label_separator`, and `relation_label`.
  - Top-level and per-item
    `relations[] {raw, direction: "to"|"from", note, block_id, quoted, label, label_range, range}`,
    omitted when empty.
  - Needs: `relation_target` (a partial `~` / `~~`) and `relation_label` (after `|` or
    inside an open quote). An ownerless item reuses `dependency_target`.
  - Top-level `relation_grammar: 1` whenever the draft contains a `~` byte. Drafts
    without `~` are byte-identical to today.
  - Relations never appear in `dependencies[]` or `blocked_tasks[]`.
- **`capture-complete`.**
  - Contexts `relation_to` and `relation_from`. Their rows use the `DependencyCandidate`
    shape and groups, plus `already_linked`, `existing_label`, and `also_prerequisite`
    (an information badge; the row stays selectable) and `disabled_reason`:
    - `the owner task itself`;
    - the duplicate-block-ID reason;
    - `archived tasks can't get an id`;
    - for `relation_from` only: `read-only note`, `its LINKS line is malformed`, and the
      `*` refusal set.
  - Pools: `relation_to` uses the whole `discover` catalog. `relation_from` uses
    editable tasks only.
  - Empty-query ranking:
    - `relation_to`: the owner's note, In Progress, Next, open, closed in active notes,
      then `done/`.
    - `relation_from`: today's linked tasks (the running entry first), In Progress,
      Next, the owner's note, then the rest.

    A typed query ranks by match score first.

  - The replacement range covers the whole modifier, sigil included, so a typed second
    `~` switches the context in place.
  - Context `relation_label` returns `suggestions[] {label, count, replacement}` from
    every LINKS line, `done/` included, filtered by the typed prefix. When the typed
    label is non-empty, a first row `{label, typed: true, replacement}` carries bob's
    canonical `|label` / `|"multi word"` spelling, or a `disabled_reason` naming the
    broken rule.
  - `picker.relationship` gains `relations[]` (direction, text) for the owner picker.
  - Shell completion gets `links to` / `linked from` groups.

- **`capture-task-id`:** new fields `relation_to_replacement` (`~note:id`) and
  `relation_from_replacement` (`~~note:id`).
- **`capture` / `--dry-run`, per item** (omitted when empty):
  - `relation_updates[]`:
    `{direction, source{path,line,block_id,text}, target{path,line,block_id,text,archived?}, label, previous_label?, action: added|relabeled|unchanged}`;
  - `linked_from_line_updates[] {path,line,task,kind,added[],removed[]}`, using the
    `BlocksUpdateKind` vocabulary;
  - `minted_relation_block_ids[] {path,line,block_id,role: source|target}`;
  - `mirror_skipped[] {path, block_id, reason: archived_target|previous_daily_target|malformed_line|disabled}`.

  An existing task whose LINKS line changed is reported as a task block with role
  `relation_source`. Mirror-only changes report rows, not task blocks.

- **`bob task reconcile` JSON:**
  - `linked_from_line_updates`;
  - `minted_relation_source_block_ids`;
  - `relation_warnings[] {kind, path, line, message}`, where `kind` is one of
    `malformed_links_line`, `malformed_linked_from_line`, `unresolved_relation_link`,
    `relation_target_not_task`, `self_relation`, or `duplicate_block_id`.

  Human output gets a `🪢` group in the dependency style, and `--dry-run` previews all
  of it.

- **Config:**

  ```yaml
  relations:
    linked_from_lines: true # write 🪢 LINKED FROM mirrors (docs/task-relations.md)
  ```

  Rust and nav read it from `~/.config/bob/config.yml` (`BOB_CONFIG_FILE` /
  `XDG_CONFIG_HOME`). An invalid value falls back to `true` and warns.
  - With `false`, reconcile removes every LINKED FROM line on writable tasks, mints
    nothing, and leaves LINKS alone, so the switch is lossless.
  - Gestures and capture write no mirror and mint no source.

### Rule 13: guarantees (tested invariants)

1. **Single source (LR11).** Delete every LINKED FROM line and run reconcile once: the
   lines come back byte-identical, and a second run changes nothing.
2. **Inertness (LR18, LX13, LG13).** Adding, removing, or relabelling any relation
   leaves every status, lane, field, schedule, Pomodoro section, and Depends-On / BLOCKS
   line byte-identical, apart from a minted `^id`.
3. **Engine agreement (LW14, LK\*).** Rust and JS agree on every LK, LW, and LM vector,
   malformed cases included.
4. **Live view (LC8).** Chips render from the live index, never from the stored mirror.
5. **Convergence (LR3).** Canonical sorting makes two machines produce the same bytes,
   and the next reconcile heals any textual conflict.
6. **Read-only respect (LR7).** No relation write touches `done/` or the previous daily
   note.

## Vectors

`contract` copies these into `docs/task-relations.md`, and phases cite them by id in
test comments.

### LK — parse

| #    | Line (direct child of a `#task` unless stated)                                                                                | Verdict                                                                   |
| ---- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| LK1  | `🪢 **LINKS:** [[#^a]]`                                                                                                       | accept: 1 entry, no label, canonical                                      |
| LK2  | `🪢 **LINKS:** follow-up of [[done/bob_done#^cap]] • [[sase#^retry]]`                                                         | accept: labels `follow-up of`, none; canonical                            |
| LK3  | `🪢 **LINKS:** see [[#^a]] · merged into [[b#^c]]` and the `,` / whitespace-only separator forms                              | accept 2; canonicalise separators                                         |
| LK4  | `**LINKS:** [[#^a]]`, `🔗 **LINKS:** [[#^a]]`, `🪢 **LINKS TO:** [[#^a]]`                                                     | accept; canonicalise glyph and label                                      |
| LK5  | `🪢 **LINKS:** [[#^a\|merged into]]`                                                                                          | accept; alias adopted → `merged into [[#^a]]`                             |
| LK6  | `🪢 **LINKS:** see [[#^a\|x]]`                                                                                                | label `see`; alias dropped                                                |
| LK7  | `🪢 **LINKS:** ~~[[#^a]]~~`, `🪢 **LINKS:** ![[#^a]]`                                                                         | accept; canonicalise                                                      |
| LK8  | `🪢 **LINKS:** see [[#^a]] • see also [[#^a]]`                                                                                | accept; the canonical form keeps the first entry                          |
| LK9  | `🪢 **LINKS:** see:   also [[#^a]]`                                                                                           | label `see: also`; whitespace collapsed (`:` stripped only when trailing) |
| LK10 | `🪢 **LINKS:**`                                                                                                               | empty                                                                     |
| LK11 | `🪢 **LINKS:** [[#^a]] until Friday`                                                                                          | malformed (trailing prose)                                                |
| LK12 | `🪢 **LINKS:** [[#^a`, `[[note]]`, `[[note#Heading]]`, `• ,`                                                                  | malformed                                                                 |
| LK13 | `🪢 **LINKS:** see (old) [[#^a]]`, `"quoted" [[#^a]]`                                                                         | malformed (label characters)                                              |
| LK14 | `- LINKS: [[ref#^a]]`, `**links:** [[#^a]]`, `**RELATED:** [[#^a]]`, `**SEE ALSO:** [[#^a]]`                                  | not-a-line                                                                |
| LK15 | LK1 fenced, as a grandchild, blockquoted, without a list marker, or under a non-task                                          | not-a-line                                                                |
| LK16 | `🪢 **LINKED FROM:** [[bob#^fix]] (follow-up of) • [[#^b]]`                                                                   | accept 2 with labels                                                      |
| LK17 | `**BACKLINKS:** [[#^b]]`                                                                                                      | accept as LINKED FROM; canonicalise                                       |
| LK18 | `🪢 **LINKED FROM:** [[#^b]] extra`, `[[#^b]] (a) (b)`                                                                        | malformed                                                                 |
| LK19 | each relation line through the Depends-On, BLOCKS, and the other relation parser, and DP1 / BK1 through both relation parsers | not-a-line both ways                                                      |
| LK20 | mirror entries `[[zeta#^e]]`, `[[élan#^d]]`, `[[cash#^c]]`, `[[Cash#^a]]`, `[[#^b]]`                                          | sorted `[[#^b]] • [[Cash#^a]] • [[cash#^c]] • [[zeta#^e]] • [[élan#^d]]`  |

### LW — planner writes (LINKS and LINKED FROM)

| #    | Situation                                                              | Outcome                                                                |
| ---- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| LW1  | S has no LINKS; add T, unlabelled                                      | `🪢 **LINKS:** [[T]]` inserted in the Rule 3 slot                      |
| LW2  | add T with label `see` to an existing line                             | ` • see [[T]]` appended                                                |
| LW3  | re-add T with the same label                                           | no-op (`unchanged`); byte-identical                                    |
| LW4  | re-add T with a new label                                              | relabeled in place; order kept                                         |
| LW5  | remove the only entry                                                  | line deleted                                                           |
| LW6  | remove a middle entry                                                  | line rewritten; the others keep their order                            |
| LW7  | Cancel Log, Depends-On (nested child), BLOCKS, Sub-projects present    | LINKS after the Sub-projects subtree; LINKED FROM right after LINKS    |
| LW8  | no managed children; user child and Schedule Log present               | LINKS first; existing children untouched                               |
| LW9  | two LINKS lines                                                        | merged into the first (append, first label wins); the second deleted   |
| LW10 | malformed LINKS line                                                   | authored write refused: "its LINKS line is malformed"; nothing written |
| LW11 | label ` Follow-up  of:` / 41 characters / `see (old)`                  | `Follow-up of` / refused (too long) / refused (character)              |
| LW12 | CRLF note without a final newline; a task using a 4-space child indent | preserved; indent reused                                               |
| LW13 | target in a unique-basename note / an ambiguous basename / `done/`     | `[[note#^t]]` / `[[dir/note#^t]]` / `[[done/x_done#^t]]`               |
| LW14 | the same input rendered by Rust and nav                                | byte-identical lines                                                   |

### LR — reconcile (mirror)

| #    | Situation                                                 | Outcome                                                                                                    |
| ---- | --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| LR1  | S → T in the same note; T has no mirror                   | `🪢 **LINKED FROM:** [[#^s]]` (`inserted`)                                                                 |
| LR2  | labelled edge                                             | `[[#^s]] (follow-up of)`                                                                                   |
| LR3  | sources in the same note, another note, and `done/`       | one line sorted per LK20; the `done/` source as `[[done/…#^s]]`                                            |
| LR4  | S has no `^id`                                            | S gains a minted `^id` (no `[id::]`, no `[fresh::]`); T lists it; a `minted_relation_source_block_ids` row |
| LR5  | S needs a mint and S's note is quiet-deferred             | S not listed this run; the next run mints and lists it; then zero changes                                  |
| LR6  | T closed in an active note                                | mirror maintained                                                                                          |
| LR7  | T in `done/` or the previous daily note                   | never written                                                                                              |
| LR8  | S in `done/` / in the previous daily note                 | listed                                                                                                     |
| LR9  | hand-added mirror entry with no edge / hand-deleted entry | removed / restored (`rewritten`)                                                                           |
| LR10 | renamed note leaves the mirror unsorted                   | `canonicalized`                                                                                            |
| LR11 | delete every LINKED FROM line in a reconciled vault       | one run rebuilds identical bytes; a second run makes zero changes                                          |
| LR12 | malformed LINKED FROM under T                             | every LINKED FROM child of T untouched; warn                                                               |
| LR13 | S's LINKS malformed                                       | existing entries naming S kept; S never newly added; warn                                                  |
| LR14 | LINKS entry unresolved / non-task / self                  | no entry; matching warning; LINKS untouched                                                                |
| LR15 | two LINKED FROM lines                                     | first rewritten, the rest deleted (`deduplicated`)                                                         |
| LR16 | `relations.linked_from_lines: false`, then `true`         | every LINKED FROM on writable tasks removed, no mints; then rebuilt                                        |
| LR17 | invalid `relations.linked_from_lines`                     | behaves as `true`; stderr warning                                                                          |
| LR18 | relation added / removed / relabelled between runs        | only LINKED FROM lines and minted ids differ                                                               |
| LR19 | A → B and B → A                                           | each lists the other                                                                                       |
| LR20 | `--dry-run -f json`                                       | identical rows; nothing written; no mints persisted                                                        |
| LR21 | owner's note modified less than 2 s ago                   | edits deferred                                                                                             |
| LR22 | S archived by `bob task archive`                          | T's entry already rewritten to `[[done/…#^s]]` by link repair; reconcile keeps it unchanged                |
| LR23 | S's note has a duplicate `^s`                             | S not listed; `duplicate_block_id` warning                                                                 |
| LR24 | target is a `^prj` task                                   | the `^prj` line gets the mirror like any task                                                              |

### LX — capture execution

| #    | Draft or situation                                                         | Outcome                                                                                                                                               |
| ---- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| LX1  | `Fix retries ~bob_done:capture-star\|"follow-up of"`                       | new N with `🪢 **LINKS:** follow-up of [[done/bob_done#^capture-star]]`; `mirror_skipped` `archived_target`; N not minted for a mirror                |
| LX2  | `Write outline ~~sase_blog:prj\|"next step"`                               | `sase_blog`'s `^prj` LINKS gains `next step [[mac_inbox#^write-outline]]`; N minted; N born with `🪢 **LINKED FROM:** [[sase_blog#^prj]] (next step)` |
| LX3  | `@bob+fix-retries ~sase:retry-design\|see`, then a rerun, then a new label | mirror updated in the same batch; rerun `unchanged`; new label `relabeled`                                                                            |
| LX4  | `Ask plumber &job:parts ~job:cleanup`                                      | `&` blocks as today; `~` changes no status                                                                                                            |
| LX5  | `N ~T`, T in an active note (today's daily note included)                  | T's mirror written in the same batch; N minted                                                                                                        |
| LX6  | kill switch off                                                            | no mirror and no source mint; `~~` still mints N                                                                                                      |
| LX7  | `~~` target in a daily note, `done/`, `_templates`, or fenced code         | refused; vault byte-identical                                                                                                                         |
| LX8  | self target, non-task target, missing block id                             | refused with a teaching error                                                                                                                         |
| LX9  | S's LINKS line malformed                                                   | refused; vault byte-identical                                                                                                                         |
| LX10 | T's LINKED FROM malformed                                                  | capture succeeds; `mirror_skipped` `malformed_line` and a warning                                                                                     |
| LX11 | `x ~job:cleanup`                                                           | `operator_relation_error`                                                                                                                             |
| LX12 | any LX vector, dry run vs live                                             | identical JSON apart from `dry_run`                                                                                                                   |
| LX13 | any relation capture                                                       | no `[fresh::]`, lane, status, or Pomodoro change on S or T                                                                                            |
| LX14 | `N ~T1 ~T2 ~T1` / `N ~T1\|a ~T1\|b`                                        | T1 then T2, duplicate no-op / refused                                                                                                                 |
| LX15 | `N ~T ~~T`                                                                 | two independent edges                                                                                                                                 |
| LX16 | a same-note S and T                                                        | both edits composed against staged text; one write of the note                                                                                        |

### LP — capture parse

| #    | Draft                                                              | Reading                                                             |
| ---- | ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| LP1  | `Fix it ~work:foo` / `~work:foo Fix it` / `@cash ~work:foo Fix it` | relation `to` (trailing / leading / past `@cash`)                   |
| LP2  | `Fix it ~~work:foo`                                                | relation `from`                                                     |
| LP3  | `Fix ~work:foo now`                                                | literal (mid-line)                                                  |
| LP4  | `Plan trip ~5m`, `~2h`, `~5pm`, `Clean ~/Downloads`                | literal                                                             |
| LP5  | `Read ~~old~~ notes`, `Read ~~old~~`                               | literal (strike guard)                                              |
| LP6  | `Fix it ~`, `Fix it ~~`, `Fix it ~wo`, `Fix it ~work:`             | need `relation_target`                                              |
| LP7  | `Fix it ~work:foo\|see`, `Fix it ~work:foo\|"follow up of"`        | label spans and label                                               |
| LP8  | `Fix it ~work:foo\|`, `Fix it ~work:foo\|"follow up`               | need `relation_label`                                               |
| LP9  | `Fix it ~work:foo\|"a (b)"`                                        | invalid label with a teaching message                               |
| LP10 | `Fix it \~work:foo`                                                | literal `~work:foo`                                                 |
| LP11 | `=~2`, `x~1` / `x~~1`, `=~~2`                                      | unchanged drop lists / error "use one `~` list"                     |
| LP12 | `~work:foo` alone                                                  | need `dependency_target`                                            |
| LP13 | `x ~work:foo`                                                      | `operator_relation_error`                                           |
| LP14 | `Fix ~~~work:foo`                                                  | invalid                                                             |
| LP15 | `` Fix `~work:foo` ``, `Fix [[~work:foo]]`                         | never scanned                                                       |
| LP16 | `Meet ~noon`                                                       | need `relation_target`; submit refuses with the `\~` teaching error |
| LP17 | `@bob+fix ~sase:retry`                                             | mode `task_dependency`; no task text needed                         |
| LP18 | any draft without `~`                                              | byte-identical JSON (no `relation_grammar`)                         |

### LG — Obsidian gestures

| #    | Gesture                                                          | Outcome                                                                              |
| ---- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| LG1  | Links stage ↵ on a same-note T                                   | S's LINKS, T's mirror, and S's mint in one transaction; one Ctrl+Z undoes all        |
| LG2  | cross-note T / the follow-up finds T changed                     | S commits, then T's mirror follows / S stands, suffix `· 🪢 LINKED FROM not updated` |
| LG3  | ⇧↵ then a label / an empty label                                 | labelled entry / plain entry                                                         |
| LG4  | CURRENT ↵ / ⇧↵                                                   | unlink with the mirror entry removed / relabel in place                              |
| LG5  | id-less T                                                        | block-ID prompt; Escape writes and mints nothing                                     |
| LG6  | closed T / archived T                                            | allowed / allowed, no mirror, suffix `· archived: shown live only`                   |
| LG7  | Linked-from stage ↵ on S                                         | S's LINKS gains this task; this task's mirror updated                                |
| LG8  | Ctrl+Shift+P on a LINKS / LINKED FROM line                       | the owner's Links / Linked-from stage                                                |
| LG9  | Ctrl+D on the Links to / Linked from row                         | every link and its mirror entries removed / refused                                  |
| LG10 | hand edit adds / removes / relabels a LINKS entry, cursor leaves | mirrors follow; Ctrl+Z re-fires the mirror                                           |
| LG11 | hand edit confined to LINKED FROM                                | notice once; no edge change                                                          |
| LG12 | `3 Ctrl+Shift+L`, choose T with a label                          | four sources each gain one labelled entry                                            |
| LG13 | any relation gesture on an inbox task or a `]s` landing          | no stamp, lane change, inbox routing, or walk advance                                |
| LG14 | kill switch off                                                  | no mirror writes and no source mints                                                 |
| LG15 | Tab marks two targets, then ↵                                    | both linked in one gesture and one notice                                            |

### LC — chips

| #    | Situation                                | Render                                                               |
| ---- | ---------------------------------------- | -------------------------------------------------------------------- |
| LC1  | one unlabelled open target               | `🪢 links to → (○ T)`                                                |
| LC2  | labelled target                          | pill `follow-up of`, then `→ (○ T)`                                  |
| LC3  | closed / archived target                 | dimmed `✓` (never struck) / plus `↗ bob_done`                        |
| LC4  | unresolvable target                      | dashed `⚠ ^id not found`                                             |
| LC5  | LINKED FROM with label `merged into`     | `merged from (✕ S)` (under `verbatim`: `(✕ S) · merged into → this`) |
| LC6  | LINKED FROM with an unknown label `see`  | `(○ S) · see → this`                                                 |
| LC7  | a live edge missing from stored text     | appended in canonical order                                          |
| LC8  | a stored entry that is no longer live    | dashed; refresh tooltip; not counted                                 |
| LC9  | four closed sources                      | `✓×4`; click expands                                                 |
| LC10 | labels present                           | one entry per visual row; line height unchanged                      |
| LC11 | nav api present / absent                 | ＋ and × controls per Rule 9 / navigation only                       |
| LC12 | `toggle-dependency-chips`                | toggles both relation rows with the dependency rows                  |
| LC13 | malformed or not-a-line row              | no chips                                                             |
| LC14 | an entry typed in the editor before save | the target's chip row updates in the same session                    |

### MC — managed child lines (append to `docs/task-dependencies.md` §11.8)

| #    | Situation                                                                                                | Outcome                                                                                                                          |
| ---- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| MC10 | a task under an open Pomodoro entry has LINKS / LINKED FROM lines                                        | no Task Link effects: promotion, consumption, successor anchor, plan budget, completable sighting, close / cleanup (Rust and JS) |
| MC11 | `🪢 **LINKS:** ![[#^t]]` under S; S closes via capture `x` or Ctrl+Enter                                 | T never closed (tree close skips relation lines)                                                                                 |
| MC12 | T closes or reopens elsewhere                                                                            | strike / restore never touches relation lines                                                                                    |
| MC13 | `!` / Ctrl+Shift+Enter on either line                                                                    | refused with the Rule 8 notices                                                                                                  |
| MC14 | Ctrl+Enter / Alt+] on a link in either line                                                              | acts on the linked task root-only, without reformatting the bullet                                                               |
| MC15 | R8 and the section-title, Schedule / Cancel / Work Log, Sub-projects, Depends-On, and BLOCKS recognisers | never claim relation lines; relation parsers never claim theirs                                                                  |
| MC16 | `bob projects sync`; nav project conversion                                                              | never moves or anchors on relation lines; conversion keeps LINKS with its task and drops LINKED FROM                             |
| MC17 | `bob task archive`, `bob task move`, Ctrl+Shift+M                                                        | relation-line links rebased; LINKS moves with its task; never two LINKS lines                                                    |

## Cross-cutting rules for every phase

- **Read first:**
  - this plan's Design and Vectors;
  - `docs/task-relations.md` once `contract` lands, and `docs/task-dependencies.md` §§2,
    11.8, and 14 (BLOCKS, the closest precedent);
  - `sase artifact read research:202610/task_relation_links/task_relation_links.md "<why>"`;
  - `sase memory read decisions:task-deps-are-depends-on-links decisions:capture-star-blocks-existing-tasks decisions:mac-capture-is-a-thin-client glossary:task-link -r "<why>"`.
- **Vectors are the contract.**
  - Copy every row you implement into tests, citing its id in a comment.
  - If code and a vector disagree, fix `docs/task-relations.md` first in the same change
    and say so in the commit.
- **Words.** Use _relation_ for 🪢 edges and _relationship modifiers_ for the `&` `*`
  `~` `~~` umbrella. Never put a relation on `dependency_updates`, `dependencies[]`, or
  `blocked_tasks[]`.
- **Don't break anything.**
  - Every existing DP / DW / DR / BK / BR / BW / BC / RB / ST / MC test keeps passing
    unchanged.
  - JSON changes are additive. Drafts and vaults without relations produce
    byte-identical output.
  - Follow `cli_rules.md` (read it with `/sase_memory_read`): no new subcommand or
    option, and `capture-*` grows only additively.
- **Inertness is tested in every writer phase** (Rule 1, Rule 13 invariant 2).
- **Performance.**
  - Reconcile adds only the prefiltered `done/` reads.
  - Capture adds no vault scan beyond the catalog completion already uses.
  - nav reads no note it did not already load, except cross-note follow-ups and the
    no-index fallback.
  - Record before / after timings in your phase note.
- **Known failures.** Before calling a failure pre-existing, check
  `sase bead list -T flake -T ci` and reproduce it on clean origin/master.
- **bob-cli.**
  - Run `just check`, and `just docs-check` when `docs/` changes. Read `docs.md` with
    `/sase_memory_read` before editing docs.
  - Put new logic in new modules, so existing large files grow by a few lines each.
- **bob-plugins** (`sase repo open bob-plugins -r "<why>"`; read its `AGENTS.md`).
  - Edit only `src/` fragments, each ≤ 1000 lines. Register them in `src/fragments.json`
    and add new mixins to the installer fragments.
  - Export pure helpers for tests, and append new test files to `package.json`'s `test`
    script.
  - Run `npm run build`, `npm test`, and `npm run validate`.
  - Minor-bump each touched manifest from the latest master version and update its root
    `README.md` row and plugin docs.
  - Deploy on this host only: `bob plugins sync -n -r "<opened path>" -p <id>`.
  - Merge both sides of `package.json`, README rows, and manifests when parallel phases
    collide.
- **bob-mac-capture** (`sase repo open bob-mac-capture -r "<why>"`; read `AGENTS.md`).
  - The app stays a renderer (`decisions:mac-capture-is-a-thin-client`): it never
    decides grammar.
  - Run `just portable-check` on athena.
  - When the Mac is reachable (read `tailnet.md` with `/sase_memory_read`), compile a
    scratch copy there with `bundle.sh` and `just typecheck-tests`, then delete it.
  - After pushing, watch macOS CI through `/sase_monitor`. If the macOS targets were not
    verified, say so in the phase note.
- **Safety.**
  - Never edit the vault by hand. Never run a live `bob task reconcile`, `bob capture`,
    or `bob projects sync` against `~/bob`; dry runs are fine.
  - Experiment on `BOB_DIR=<temporary copy of ~/bob>`, with `BOB_NOW` / `BOB_DAY_FILE`
    as needed and `BOB_CONFIG_FILE=<temp>` for kill-switch tests. Delete the copies
    afterwards.
  - Touch the MacBook only in `rollout`, except for the Mac phases' scratch compile.
- **Memory.** Only `rollout` edits memory, and only within the accepted memory
  decisions.
- **Follow-ups.** Append `PROPOSED FOLLOW-UP:` notes to your own phase bead; never
  create beads.

## Phase: contract

**Repo:** bob-cli, docs only, one commit.

1. **New `docs/task-relations.md`**, a root behaviour contract, added to `mkdocs.yml`
   nav beside `task-dependencies.md`. Sections:
   - Vocabulary: the Words above, plus **source**, **target**, **LINKS line**, **LINKED
     FROM line**, and **relation label**.
   - What relations never do (Rule 1).
   - Grammar, Slot and order, Identity and minting, and the Projection (Rules 2–5).
   - Reconciliation (Rule 6), Immediate writers (Rule 7), and Gestures (Rule 8).
   - Chips and the live index (Rules 9–10), including the inverse table per the
     `inverse_phrases` decision and the × control per `linked_from_remove`.
   - Kill switch, Outputs, and warning kinds (Rule 12).
   - Guarantees (Rule 13).
   - Lifecycle: close, cancel, archive S, archive T, move, rename, T deleted, S deleted,
     duplicate ids, and raw cut-and-paste. Copy the research's Lifecycle section,
     corrected to this plan.
   - The LK, LW, LR, LX, LP, LG, and LC vector tables.
   - Write every rule in the doc's own voice; the plan's wording is not binding prose.
2. **`docs/task-dependencies.md`.** Add LINKS and LINKED FROM rows to the §2.4 registry
   (owner, writer, slot, and rejecting recognisers), plus one intro sentence pointing to
   `task-relations.md`. Add MC10–MC17 to §11.8.
3. **`docs/capture.md`.**
   - Grammar-table rows for every Rule 11 form, beside `&` and `*`.
   - A new section `### Linking tasks with '~' and '~~'` after
     `### Relating existing tasks with '&' and '*'`. It covers meaning, guards,
     examples, refusals, shell quoting, inertness, the human output shape, and a full
     JSON example.
   - Teach that `~` / `~~` are relationship modifiers.
   - Update the `capture-parse`, `capture-complete`, and `capture-task-id` sections, and
     the JSON output section, with every Rule 12 field.
4. **`docs/reference/capture-syntax.md`.** Add the rows, plus "Overloaded markers" rows:
   - `~` glued to `=` / `x` drops Task Links;
   - `~` as a task modifier links;
   - `~~word~~` is strikethrough;
   - `~~` as a task modifier links in reverse.

   Add the mnemonic **"`~` on its own ties two tasks; `~` inside `x` / `=` drops."**

5. **Other docs.**
   - `docs/reference/vault-format.md`: both lines.
   - `docs/task-status-hooks.md`: the relation pass, the narrow `done/` LINKS read, the
     JSON example, and the human output.
   - `docs/reference/configuration.md` and `docs/getting-started.md`:
     `relations.linked_from_lines`.
   - `docs/completion.md`: `~` / `~~` capture markers.
   - `docs/reference/glossary.md`: **Task relation**.
6. **Finish.** Run `just docs-check` and commit.

## Phase: cli-core

**Repo:** bob-cli.

1. **New module `src/native/task_relations/`.**
   - **`mod.rs`:** constants (`🪢`, the labels, `•`), the types, `is_links_line`,
     `is_linked_from_line`, and `is_relation_line`.
   - **`label.rs`:** `normalize_relation_label` (Rule 2), returning a typed reason on
     refusal, and `relation_label_quoted_form` (`|label` / `|"multi word"`) for capture.
   - **`parse.rs`:**
     - `parse_links_line` →
       `LinksLine::{Accepted{entries, canonical}, Empty, Malformed, NotALine}`, where
       `RelationEntry {target, block_id, label, embedded, struck, aliased}`;
     - `parse_linked_from_line`;
     - `links_children_of` / `linked_from_children_of`.

     Reuse `block_link_spans` and `blocks.rs`'s `has_task_tag`, `direct_child_lines`,
     and `subtree_end`. Extract shared helpers rather than copying them.

   - **`format.rs`:** `format_links_line` and `format_linked_from_line`, the sort that
     reuses `sort_blocks_links` (LK20), and `relation_insert_slot` (Rule 3, both lines).
   - **`plan.rs`:**
     - `plan_links_edit(lines, fenced, owner_line, edit: LinksEdit{add: [(link, label)], remove: [identity], relabel})`
       → edits plus a report (added / relabeled / unchanged / removed), or `Malformed`.
       It merges duplicate lines (LW9) and canonicalises the touched line.
     - `plan_linked_from_line(lines, fenced, owner_line, desired: BTreeSet<(link, label)>)`
       → `BlocksLinePlan`-style edits and report, using the Rule 6 table.
     - `linked_from_desired_after_delta(existing, resolve, add, remove)` for capture.
   - **Tests:** LK1–LK20, LW1–LW13, and a planner unit test per Rule 6 table row.

2. **Registry predicate.** Add `is_managed_link_line(line)` (dependency or relation,
   every verdict except not-a-line) beside `is_managed_dependency_line` in
   `task_dependencies/mod.rs`. Switch these sites to it:
   - `capture_completable_tasks.rs`;
   - `task_complete/tree.rs`;
   - `plan_budget/mod.rs`;
   - `task_status_hooks/pomodoro.rs`;
   - `capture_pomodoro_close/ledger.rs`.

   Leave `projects/sync.rs`'s anchor dependency-only, and add an MC16 test proving it
   never moves or anchors on relation lines. Audit every other block-link scanner under
   a Pomodoro (`capture_pomodoros_agenda.rs`, `capture_pomodoro_start.rs`, and successor
   anchors) with one MC10 test each. Record the scanners that are already safe by shape.

3. **Recogniser tests.** LK19 (cross-parse), MC15 (R8, section titles, and logs), and
   MC17 (`bob task archive` / `bob task move` rebasing a LINKS line). These cover
   existing behaviour; fix it if a test fails.
4. **Config.** New `src/native/config/relations.rs`: `load_relations_config` →
   `RelationsConfig { linked_from_lines: bool }`, default `true`. It mirrors
   `config/dependencies.rs`, including its error type and tests.
5. **Scope.** No writer emits a relation line yet. Run `just check` and commit.

## Phase: plugin-guards

**Repo:** bob-plugins: `bob-navigation-hotkeys`, `task-status-cycler`,
`block-id-prompt`, and `bob-ledger-tools`'s plan-budget guard only.

1. **Recognisers.**
   - **nav:** a new pure fragment (e.g. `085-relation-line-grammar.js`) with
     `parseLinksLine`, `parseLinkedFromLine`, `isTaskRelationLine`, and
     `normalizeRelationLabel`, exported for tests. Its formatters, sort, and slot
     helpers come later, in `nav-writer`.
   - **cycler** (new `051-…`): `isTaskRelationLine`, true for accept, empty, and
     malformed.
   - **block-id-prompt** (new `061-…`, since `060` is full): the same.
   - **ledger:** add the `050::planBlockLinks` guard (MC10).
   - Copy LK1–LK19 into each plugin's tests.
2. **Two predicates in the cycler.** This avoids the trap in What exists now.
   - "Never a Task Link" (`060` Pomodoro sub-bullets, `085` successor live link) skips
     every managed line, relations included.
   - "Act on the link under the cursor" (`090`, `150`, `160`, `190`) adds relation
     lines, so Ctrl+Enter and Alt+] are root-only and never reformat (MC14).
   - Tree close (`050:388`) and strike / restore (`080:675` / `080:756`) skip relation
     lines (MC11, MC12).
3. **block-id-prompt.** `startTaskLinkOpen` / `applyTaskLinkOpen` refuse on both lines
   with the Rule 8 notices (MC13). Pomodoro cleanup (`060:478`) and
   `collectPomodoroEntryBlockLinks` (`075:211`) skip relation lines (MC10).
4. **nav guards.**
   - `!` / `N!` are refused on both lines (`180:418`, `500`).
   - The hand-edit listener (`600`) shows the LINKED FROM generated-line notice once per
     burst, and ignores bursts confined to a LINKS line for now. `nav-linked-from`
     mirrors them later.
   - Project conversion (`212::splitProjectLifecycleChildren`) keeps LINKS with its task
     and drops LINKED FROM (MC16).
   - Check that inbound links on other notes' LINKS lines survive conversion, as task
     move's external-role rewrite does. Reconcile never rebinds a broken entry. If they
     do not survive, fix conversion with the existing rewrite, or record a
     `PROPOSED FOLLOW-UP:` when the fix is out of scope.
   - Pomodoro cleanup (`160:592`) skips relation lines.
5. **Tests.** Add one test per guard (the MC10–MC16 rows that apply). Bump, document,
   build, test, validate, and deploy
   (`-p bob-navigation-hotkeys -p task-status-cycler -p block-id-prompt -p bob-ledger-tools`),
   then commit.

## Phase: cli-reconcile

**Repo:** bob-cli.

1. **Edge collection** (`task_status_hooks/relations.rs`).
   - Parse LINKS lines from the active `files`, prefiltered by bytes. The previous daily
     note is read-only.
   - Load every `done/**/*.md` read-only, byte-prefiltered for `LINKS`. Record each
     snapshot in `inputs`, as `archive_reference_catalog` does.
   - Resolve each entry: active targets through the `NoteIndex` basename rules, and
     explicit `done/` targets through the loaded archives.
   - Build `edges`, the uncertain sources (malformed LINKS lines and duplicate-id
     notes), and the warnings (Rule 5).
2. **The pass
   `reconcile_relation_lines(files, note_index, archives, previous_daily_file, quiet_deferred, enabled)`
   → `RelationsOutcome`.** It holds the touched files, first-touch originals, updates,
   mints, and warnings. It applies Rules 4–6 with the cli-core planner and
   `mint_block_id`, edits `files` in place, and re-parses the touched files.
3. **Wiring in `sync.rs`.**
   - Call the pass right after `reconcile_blocks_lines`.
   - Merge touched files into the structural set and originals into `pre_reconcile`.
   - Load `relations.linked_from_lines` once per run. An invalid value warns on stderr
     and falls back to `true`.
4. **Outputs.**
   - `SyncResult` gains `linked_from_line_updates`, `minted_relation_source_block_ids`,
     and `relation_warnings`.
   - Add the human `🪢` group.
   - Update `docs/task-status-hooks.md` examples to match real output.
5. **Tests.** A new `tests/cli/task_status_hooks/relations.rs` covering LR1–LR24,
   including:
   - LR11 (delete every line, rebuild identical bytes);
   - idempotence;
   - LR5 (the quiet-interval mint skip);
   - LR16 / LR17 with `BOB_CONFIG_FILE`;
   - LR18 (byte-identical statuses and fields).
6. **Live dry run (read-only).** Run the phase build's
   `bob task reconcile --dry-run -f json` against `~/bob`. Expect zero relation updates,
   zero mints, and zero `relation_warnings`, because no relation lines exist yet.
   Explain any warning. Record the timing delta from the `done/` read.
7. **Finish.** Run `just check` and commit.

## Phase: capture-parse

**Repo:** bob-cli.

1. **Scanner.** Add a new `src/native/capture_language/relations.rs` and wire it into
   `extract_line_dependencies`' leading and trailing runs. The `~` / `~~` modifiers are
   extracted in the same runs as `&` / `*`.
   - Apply rules 1–5 of Rule 11.
   - Labels after `|` are validated with `normalize_relation_label`.
   - Quoted notes reuse `scan_quoted_modifier`.
   - Escapes extend the unescape helpers.
   - Session-token `x~~` / `=~~` produce the "use one `~` list" error.
2. **Models.**
   - `ParsedRelation` on `CaptureItem` / `ItemDependencySet`.
   - `relations` on `EditorParse` / `EditorItemParse`.
   - `Need::RelationTarget` / `Need::RelationLabel`, and the five `SpanKind::Relation*`
     kinds.
   - `dependency_target`, mode `TaskDependency`, and `relationship_needs_text` are
     computed when **any** relationship list is non-empty.
3. **Rejections.** Every `&` / `*` rejection site gets the `~` twin, with messages
   naming the typed sigil:
   - operators (`operator_relation_error`);
   - session operators and section bullets;
   - `!` completions and suffixed owners;
   - `resolve_execution_ownership` and `@@` inheritance;
   - rewrite absorption (`capture_language/rewrite.rs`).
4. **`capture-parse` output.** Add `relations[]` (top-level and per item),
   `relation_grammar`, the needs, the spans, and the help text beside the `*` help.
5. **Temporary guard.** `bob capture` refuses any item with relations with "`~` task
   links are not supported yet", so a half-landed epic never writes `~` as text.
   `capture-execute` removes this guard.
6. **Tests.** Inline lexer tests for LP1–LP18, and a new
   `tests/cli/capture/parse_relation.rs` registered in `tests/cli/capture/mod.rs`. All
   existing `=~` / `x~` and `*` parse tests must stay unchanged.
7. **Finish.** Run `just check` and commit.

## Phase: capture-complete

**Repo:** bob-cli.

1. **Detection.** Add `CompletionContext::RelationTo` / `RelationFrom` /
   `RelationLabel`, with field-at helpers mirroring `blocked_task_completion_field_at`.
   Strip `~` modifiers before marker completion, as `&` / `*` do.
2. **Candidates** (`capture_complete/`, new `relations.rs`).
   - Both pools come from `capture_dependency_tasks::discover`.
   - Apply the Rule 12 eligibility, ranking, badges, and disabled reasons. Today's
     linked tasks come from `capture_active_tasks::read_ledger`.
   - Set `already_linked` / `existing_label` from the owner's LINKS line or earlier
     modifiers in the item. Set `also_prerequisite` from the discovered Depends-On
     edges.
   - Build replacements with `replacement_for_sigil`, generalised to a `&str` sigil.
3. **Labels.** `relation_label` suggestions scan LINKS lines across the catalog's notes,
   `done/` included, prefiltered by bytes. The typed row uses
   `relation_label_quoted_form`.
4. **Surfaces.**
   - `capture_task_id.rs`: `relation_to_replacement` and `relation_from_replacement`,
     with help text.
   - `picker.relationship.relations[]`.
   - Shell groups and the `render.rs` labels.
5. **Tests.** A new `tests/cli/capture/complete_relation.rs` covering:
   - `X ~`, `X ~~`, `X ~jo`, and `X ~job:`;
   - the flip (same replacement start, a context change);
   - open, closed, and archived ordering;
   - `relation_from` exclusions;
   - each badge and disabled reason;
   - label suggestions and the typed row, including invalid labels;
   - the task-id replacements.
6. **Finish.** Run `just check` and commit.

## Phase: capture-execute

**Repo:** bob-cli.

1. **New `src/native/capture/relations.rs`** (planner) and **`relations_mirror.rs`**
   (same-batch mirror, modelled on `blocks_mirror.rs`).
   - Resolve every relation target (Rule 11.8 refusals, dedupe), and refuse before
     anything is staged.
   - Plan after the item's `&` / `*` passes. Re-locate the owner by block id.
   - **`~`:** stage S's LINKS edit with `plan_links_edit`. Then, if T is writable and
     the mirror is enabled, mint S's id (batch reservations) and stage T's LINKED FROM
     with `linked_from_desired_after_delta`.
   - **`~~`:** always mint the item's task id, stage T's LINKS edit, and then stage the
     item task's LINKED FROM.
   - Re-read staged text between writes, so same-note pairs compose (LX16).
   - A mirror problem pushes `mirror_skipped` plus a warning, and never fails the
     capture.
2. **Owners.**
   - New-task items go through `plan_capture_item`.
   - Existing owners go through `plan_dependency_only_item` and the sub-bullet and
     forced `--task` / `--route` paths, exactly as `*` does.
3. **Outputs.**
   - Per item: `relation_updates[]`, `linked_from_line_updates[]`,
     `minted_relation_block_ids[]`, and `mirror_skipped[]`.
   - A `relation_source` task block for an existing task whose LINKS line changed.
   - The human `🪢` lines (What Bryan sees), with `would …` in dry runs.
4. **Remove** `capture-parse`'s temporary guard.
5. **Tests.** A new `tests/cli/capture/relations.rs` covering LX1–LX16. Assert:
   - exact bytes for every touched note and the day file, which must be untouched;
   - JSON;
   - human output;
   - dry run equal to live;
   - a second live run reported as `unchanged`.
6. **Finish.** Make `docs/capture.md`'s JSON and human examples match real output. Run
   `just check` and commit.

## Phase: relation-index

**Repo:** bob-plugins, `plugins/bob-ledger-tools`.

1. **New fragments** (e.g. `144-relation-grammar.js`, `145-relation-index.js`, and a new
   lifecycle fragment, since `170` is at 958 lines):
   - an LK copy of the parser;
   - the index per Rule 10;
   - per-path maps plus a generation;
   - the editor overlay;
   - target resolution;
   - `labelSuggestions`.
2. **API.** Publish `api.relationIndex` (v1) beside the existing ledger api. Document it
   in the plugins repo's `docs/develop/plugin-apis.md`.
3. **Measure.** A Node harness runs the cold scan over a temporary copy of `~/bob`, and
   the warm per-file update over a large daily note. Record both in the phase note and
   in a fragment comment. If the cold scan exceeds about 1 s on athena, keep it chunked
   on idle and say so.
4. **Tests.** A new `scripts/test-ledger-tools-relation-index.cjs` covering:
   - LK parity;
   - forward and reverse maps published together;
   - change, delete, and rename handling;
   - the overlay (LC14);
   - `done/` sources;
   - label suggestions (case-insensitive dedupe, count order).
5. **Finish.** Bump, document, build, test, validate, deploy `-p bob-ledger-tools`, and
   commit.

## Phase: nav-writer

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`.

1. **Grammar.** Extend the relation grammar fragment (or add
   `086-relation-line-plan.js`) with:
   - `formatLinksLine` and `formatLinkedFromLine`, plus the code-point sort;
   - `relationInsertLine`;
   - `planLinksEdit`, `planLinkedFromLine`, and `linkedFromDesiredAfterDelta`;
   - `normalizeRelationLabel`.

   Copy LW1–LW14, LK20, and the planner tables. LW14 compares against bytes rendered by
   the `cli-core` Rust tests.

2. **Writer.** A new `594-plugin-relation-edit.js`:
   `applyRelationEdit({ sources, add: [{target, label}], remove: [target], relabel, direction })`
   → `{ ok, reason?, notice, committed[], failed[], mirrorFailed }`.
   1. **Preflight.** Read the touched notes live and refuse with "changed — reopen" if
      anything differs from the stage's snapshot.
   2. **Plan.** Same-note: S's LINKS edit, S's mint (`mintDependentBlockId`), and T's
      mirror in one transaction (LG1).
   3. **Commit.** Write source notes first: an open editor uses one transaction with a
      new `RELATIONS_MACHINE_WRITE_ANNOTATION` (add it to `600`'s skip), and a closed
      note uses a preimage-checked `vault.process`.
   4. **Follow-ups.** For each cross-note T, a mirror follow-up through a new
      `595-relation-follow-ups.js`, modelled on `591`.
   5. **Notice.** Build it (Rule 8 copy, `216`-style suffixes) and return it. The writer
      never shows it.

   It never stamps, never routes inbox tasks, and never touches Pomodoros (LG13). The
   kill switch is loaded beside `loadBlocksLinesEnabled` (LG14).

3. **API.** A new `486-relations-nav-api.js` with `createRelationsNavApi`: frozen
   `{version, openStage({path, line, direction}), removeLink(sourceRef, targetRef)}`,
   whose calls settle and never throw. Mount it as `relations:` in
   `createDependencyNavApi`. The shape is pinned in `docs/task-relations.md`.
   - It ships with `version: 0`: `removeLink` works, and `openStage` settles with
     `{ok: false}`.
   - `nav-linked-from` raises it to `version: 1` once both stage directions open, so
     ledger chips stay navigation-only until every control works.
4. **Tests.** LG1, LG2, LG4 (writer level), LG6, LG13, LG14, and LW14 through the
   harness.
5. **Finish.** Bump, document, build, test, validate, deploy
   `-p bob-navigation-hotkeys`, and commit.

## Phase: chips

**Repo:** bob-plugins, `plugins/bob-ledger-tools`.

1. **New fragments** (e.g. `146-relation-chip-model.js`, `153-relation-widget.js`,
   `278-plugin-relation-model.js`, `282-plugin-relation-render.js`):
   - a pure model (Rule 9: live merge, stale and broken states, inverse table,
     collapse);
   - an element builder that reuses the `bob-dep-*` DOM;
   - Live Preview decorations (a `🪢` / `LINKS` / `LINKED FROM` prefilter, visible
     ranges only);
   - the Reading-view post-processor, decorating existing anchors in place.
2. **Wiring.** Register through `setupDependencyChips`, so one toggle and one refresh
   cover every row (LC12). Refresh on `api.relationIndex.onChange`.
3. **Controls.** Use nav `api.relations` (Rule 9 and the `linked_from_remove` decision),
   feature-detected on `version >= 1`. Test against a fake v1 api, because the real one
   reaches v1 only in `nav-linked-from`; `rollout` verifies the real wiring.
4. **CSS.** Add the label pill (muted small caps), the arrow, `✓×N`, and dashed stale
   and broken chips, using existing tokens only. Support reduced motion and phone width.
5. **Tests.** A new `scripts/test-ledger-tools-relation-chips.cjs` covering LC1–LC14,
   the decision branches, Live Preview, and Reading view.
6. **Finish.** Bump, document (the plugins repo's relation docs page, linked from
   `blocks-mirrors.md`), build, test, validate, deploy `-p bob-ledger-tools`, and
   commit.

## Phase: nav-links-stage

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`.

1. **Task Card.** Add row `links-to`, "Links to…", with keycap `l`, after `blocks`, to
   `TASK_CARD_ROW_ORDER`, `TASK_CARD_ROW_SHORTCUTS`, `resolveTaskCardKey`, the
   `actionKey` regex, `presentTaskCardRow`, `planTaskCard`, and `openTaskCardAction`.
   Put row logic in a new `312-task-card-relations.js`.
   - The detail is `N · <first label> <first target, cut> +K` from the owner's own LINKS
     line.
   - The row is available on open and closed tasks. It is unavailable for a non-task
     ("Links require a #task checkbox"), and the existing read-only refusal applies in
     `done/` and the previous daily note.
2. **Chord and palette.** Ctrl+Shift+L (`task-card-links`,
   `initialTaskCardRow: "links-to"`) in `490`. First confirm that Obsidian core binds
   nothing to it, and record the evidence.
3. **Stage.** New fragments: `304-relation-stage-view.js`, `305-relation-stage-copy.js`,
   and `426-picker-relation-stage.js`.
   - **Pool:** the dependency-stage pool plus closed tasks and `done/` archives (last).
     Keep the other exclusions. Get warm data from the Tasks cache, falling back to the
     chunked vault scan.
   - **Sections:** CURRENT (outgoing, with label pills, in authored order), RESULTS
     (open), and CLOSED (closed, then archived).
   - **Badges:** label, `⛓ prerequisite` (information only), `＋ id`, `🍅`, and hidden
     (muted).
   - **Keys:** ↵ links, or unlinks on CURRENT. ⇧↵ opens the relation field, or relabels
     on CURRENT. Tab marks; Escape writes nothing. Back and empty Backspace return to
     the card.
   - **The relation field:** one input, suggestion chips from
     `api.relationIndex.labelSuggestions`, a live sentence `S — ‹label› → T`, validation
     by `normalizeRelationLabel` with the reason inline, ↵ to commit, and Escape back to
     the list.
   - **Consequence strip** (fixed height, aria-live):
     - RESULTS: `Links only · never blocks, schedules, or changes status`;
     - archived: `Archived · its LINKED FROM shows live only`;
     - CURRENT: `↵ unlinks · ⇧↵ relabels`.
   - **Title:** `🪢 Links · <S text>`. Use the lucide `link-2` icon if `knot` is
     missing.
   - **Subtitle:**
     `Links to K · searching P tasks · ⇥ to mark several · does not block`.
   - **Writes** go through `applyRelationEdit`. ID-less targets run `showBlockIdStage`
     once on apply (LG5).
4. **Redirect.** Ctrl+Shift+P or Ctrl+Shift+L on a LINKS line opens the owner's Links
   stage (`550::openBulletPropertyPicker`, `300::resolveDependencyStageEntry` line
   kinds) (LG8).
5. **Docs.** Add the rows and stage to the plugins repo's `docs/task-card.md` and the
   relations page.
6. **Tests.** LG1–LG6, LG8 (LINKS), LG15, and the Task Card model and view tests.
7. **Finish.** Bump, document, build, test, validate, deploy, and commit.

## Phase: nav-linked-from

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`.

1. **Linked from… row and stage.** Add row `linked-from`, keycap `⇧L`, chord
   Ctrl+Shift+Alt+L (`task-card-linked-from`), and a palette command.
   - The detail is `N · <first source>`, from `api.relationIndex.incoming`; it is
     omitted without the index.
   - **Stage:** CURRENT is the sources that link here, from the index, or from a
     generation-guarded vault scan when the index is absent. Never read it from the
     stored mirror. RESULTS is editable tasks, open first.
   - ↵ toggles adding or removing _this_ task on the chosen source's LINKS line, through
     `applyRelationEdit` with `direction: "from"`. ⇧↵ asks for a label phrased from the
     other side: `S — ‹label› → this`.
   - CURRENT rows show the inverse phrase per `inverse_phrases`.
   - Notices: `🪢 “S” now links here · ‹label›` / `🪢 “S” no longer links here`.
   - Ctrl+Shift+P or Ctrl+Shift+Alt+L on a LINKED FROM line opens the owner's
     Linked-from stage (LG7, LG8).
   - Make `api.relations.openStage` open both directions, and raise `api.relations` to
     `version: 1`.
2. **Counted gestures.**
   - `countedBulletPropertyChord` learns `KeyL`, with Alt for the reverse chord.
   - `N<Ctrl+Shift+L>` gives counted sources, and each gains the chosen target with one
     label (LG12).
   - Counted Linked-from is refused with "Count works with Links to (Ctrl+Shift+L)".
3. **Hand-edit mirror.** When the cursor leaves an edited LINKS line, diff the burst
   baseline's (target, label) set against the final set and apply mirror follow-ups. A
   same-note mirror goes in one annotated transaction. Ctrl+Z re-fires the mirror, and
   canonical compare prevents loops (LG10).
4. **Ctrl+D.** On the Links to row, remove every outgoing link and its mirror entries,
   with notice `🪢 Removed N links`. On the Linked from row, refuse with the
   generated-line notice (LG9).
5. **Docs.** Update the plugins repo docs.
6. **Tests.** LG7–LG12, LG14, plus undo and counted paths.
7. **Finish.** Bump, document, build, test, validate, deploy, and commit.

## Phase: mac-picker

**Repo:** bob-mac-capture. Build `bob` from a bob-cli checkout that contains
`capture-complete`, and use it to capture fixtures against a temporary vault.

1. **Direction model.** Grow `DependencyPickerDirection` into a relationship-direction
   enum (`.prerequisite` `&`, `.blocks` `*`, `.linksTo` `~`, `.linkedFrom` `~~`), with
   `leadingSigil: String`. Fix every one-character assumption listed in What exists now:
   - `FuzzyQuery` strips the whole sigil;
   - `cursor > r.start + sigil.utf8.count`;
   - `removePickerTrigger` removes the whole sigil;
   - `rememberRelationshipDisplayName` drops the sigil prefix.
2. **Sources and needs.**
   - Add `CapturePickerSource` case(s) for relations, choosing the shape that keeps the
     exhaustive switches simplest.
   - Add needs `relation_target` / `relation_label` to `CapturePickerNeed`,
     `completionNeeds`, and `pickerNeed`, and the five `relation_*` spans to
     `completionSpanKinds`.
   - Route contexts `relation_to` / `relation_from` through
     `handleDependencyCompletion`.
3. **Live flip.** When a completion response at the same replacement start changes
   context (`relation_to` ↔ `relation_from`), re-present the picker in place: keep the
   filter, swap the header and ranking, and never set `pickerAutoOpenSuppressedStart`.
   Backspace on `~~`'s empty filter flips back.
4. **Copy.**
   - Headers: `🪢 "<owner>" links to…` and
     `🪢 Choose a task that will link to "<owner>"`. The subtitle is
     `any task · ~ flips direction`.
   - Action line: `↩ link · ⇧↩ link with a relation · ~ flips direction`.
   - Badges: `Already linked · <label>`, `Prerequisite too`, `Done`, `Archived` (by
     locator), and Bob's `disabledReason` verbatim.
   - Owner-picker copy uses `picker.relationship.relations[]`.
5. **Spans and palette.**
   - A new `CaptureSemanticCategory.relationSigil` with a knot accent (`.mint`). It must
     be visually distinct from the grey Pomodoro-drop `~`.
   - Note: route accent; block id: `.blockID`; label: muted italic; separator: muted.
6. **Add-ID.** A `CaptureTaskIDPromptPurpose` for relations, using
   `relation_to_replacement` / `relation_from_replacement`. When the field is missing,
   show `This Bob is too old for ~ links. Update Bob first.`
7. **Tests and fixtures.**
   - `relation-*.json` fixtures from real bob, wired into `Tests/Fixtures/fake-bob`.
   - Extend `DependencyPickerPresentationTests`, `DependencyDecodingTests`, and
     `CaptureDependencyPanelTests` to cover the flip, Backspace, Add-ID, and multi-byte
     sigils.
8. **Verify** per the cross-cutting rules, update `docs/relationship-pickers.md`,
   `pickers.md`, and `keyboard.md`, and commit.

## Phase: mac-preview

**Repo:** bob-mac-capture. Build `bob` from a checkout containing `capture-execute`.

1. **Relation card.** ⇧↩ on a relation picker row, or typing `|` right after an accepted
   relation token, opens a card modelled on `PomodoroNamePromptCard`:
   - one field;
   - suggestion chips with counts, from the `relation_label` completion of a probe draft
     that splices `|"<typed>` after the token;
   - a live sentence (`S — ‹label› → T`, or `T — ‹label› → S` for `~~`);
   - `Inserts <bob's typed-row replacement>`.

   ↩ inserts bob's replacement, a chip click inserts that suggestion, Tab cycles the
   chips, and Escape returns to the picker. A `disabled_reason` shows inline and blocks
   ↩.

2. **Preview.**
   - Decode `relation_updates[]`, `linked_from_line_updates[]`,
     `minted_relation_block_ids[]`, and `mirror_skipped[]` tolerantly (`try?`).
   - Render 🪢 rows in `CaptureRelationshipPresentation`:
     - `🪢 links to "T" · ‹label›`;
     - `🪢 "T" now links here · ‹label›`;
     - `relabeled` and `already linked` variants;
     - a muted `· archived: LINKED FROM shows live only`.
   - Extend `isManagedRelationshipLine` to `**LINKS:**` / `**LINKED FROM:**` when
     relation rows are present.
   - Relations never render red, consumption, or status arrows.
   - Notifications count relations separately.
3. **Capability warning.** When the draft contains `~` and the parse response lacks
   `relation_grammar`, show a calm warning:
   `This bob predates ~ links; ~ text would be captured literally · update bob`. The
   first ⌘↩ shows it without submitting, and a second ⌘↩ submits.
4. **Docs.** Update `docs/capture-previews.md` and add `protocol.md` compatibility rows
   for `~` / `~~`.
5. **Tests and fixtures.**
   - Preview fixtures for LX1, LX2, LX3 (relabel), LX10, and a mixed `&` + `~` draft.
   - Card tests for suggestions, the typed row, an invalid label, and Escape.
   - Capability-warning tests with an old-bob fixture.
6. **Verify** per the cross-cutting rules, record a manual check for Bryan, and commit.

## Phase: rollout

1. **This host.**
   - From up-to-date bob-cli master, run `just install` (never `--force`).
   - Run `bob task reconcile --dry-run -f json` against `~/bob`. It must show zero
     relation updates, mints, and warnings.
   - Run a `bob capture --dry-run -f json 'Probe ~<real note>:<real id>|see'` and review
     it.
   - Pull the opened bob-plugins checkout, run
     `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`.
2. **The pilot script on a copy** (`BOB_DIR=<copy>`, `BOB_NOW` as needed):
   1. A same-note `~` capture, then a labelled cross-note `~` and a `~~`. Check the
      bytes.
   2. A live reconcile changes nothing, and a second run changes nothing.
   3. Delete every LINKED FROM line; one run rebuilds identical bytes (LR11).
   4. Hand-delete one entry and reconcile; it is restored.
   5. Mix `~` with `&`; only `&` blocks (LX4).
   6. `~~old~~`, `=~2`, and `Clean ~/Downloads` are unchanged.
   7. Kill switch false, then true (LR16).
   8. Delete the copy.
3. **MacBook, best effort.** Install bob before the plugins.
   - Retry `ssh -o ConnectTimeout=20 -o BatchMode=yes mac …` for up to about 30 minutes.
   - Install bob detached, as the BLOCKS rollout did
     (`git pull --ff-only && just install`, with `nohup` and a log), then poll for
     `INSTALL_EXIT=0`.
   - Run the reconcile dry run there.
   - Then run `ssh mac 'bob plugins sync'` and `ssh mac 'bob plugins list'`.
   - Update Bob Mac Capture only through an agent-safe path its docs define. Otherwise
     list it as left for Bryan.
4. **Guide.** Write a new `docs/guides/link-related-tasks.md`: capture `~` / `~~`, the
   Task Card `l` / `⇧L`, labels, and what relations never do. Add one sentence and a
   link in `docs/concepts/projects-and-dependencies.md`, then run `just docs-check`.
5. **Memory** (`/sase_memory_write`, Edit And Republish; run `sase memory init` if
   anything changed).

   > [!decision] memory_decision_record Add decisions strand `task-relations-are-inert`,
   > titled "Task Relations Are Inert, Authored On The Source, And Mirrored On The
   > Target", in the existing record format:
   >
   > - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture, vault.
   > - **Claim:** the Rule 1 list. One authored `🪢 **LINKS:**` line per source. A
   >   generated `🪢 **LINKED FROM:**` mirror that reconcile owns, nothing reads back,
   >   and `relations.linked_from_lines` removes losslessly. `~` / `~~` and `l` / `⇧L`
   >   as the gestures.
   > - **Why:** quote the request.
   > - **Rejected:** authoring both lines, render-only as the primary design, soft
   >   dependencies inside DEPENDS ON, one edge table replacing DEPENDS ON, one bullet
   >   per link, aliases as labels, a typed ontology, `~>` / `~^` / no reverse sigil, a
   >   `[links::]` field, auto-relating from mentions, `bob task links`.
   > - **Cost:** recogniser sprawl across five engines; reconcile-interval lag outside
   >   Obsidian; the narrow `done/` read; minted source ids.
   > - **Reopens when:** relation warnings appear on most runs for two weeks; any LINKED
   >   FROM routinely exceeds about 8 entries; or a concrete need for a relation that
   >   _does_ something appears (a new decision, never an amendment).
   > - **Evidence:** this plan's ref, the research ref, and the phase commits.
   > - **Links:** `[[decisions:task-deps-are-depends-on-links]]` and
   >   `[[decisions:capture-star-blocks-existing-tasks]]`.

   > [!decision] memory_decision_record = no Append a `PROPOSED FOLLOW-UP:` note
   > describing the record instead.

   > [!decision] memory_glossary Make two glossary changes:
   >
   > - Add glossary strand `task-relation` ("Task Relation"; aka relation, LINKS line,
   >   LINKED FROM). It is a directed, optionally labelled, inert link from one task to
   >   another, authored on the source's `🪢 **LINKS:**` line and mirrored on the
   >   target's generated `🪢 **LINKED FROM:**` line. Add it with Capture `~` / `~~` and
   >   Task Card `l` / `⇧L`. It never blocks, counts as a Pomodoro Task Link, or changes
   >   status. Link the record if it exists.
   > - Append to `glossary/task-link.md`: "Entries on a task's `🪢 **LINKS:**` /
   >   `🪢 **LINKED FROM:**` lines are [[task-relation]]s, never Pomodoro Task Links."

   > [!decision] memory_glossary = no Append a `PROPOSED FOLLOW-UP:` note describing
   > both glossary changes instead.

6. **Record** in a phase note:
   - commits and plugin versions per machine;
   - the dry-run and pilot output;
   - the coordination follow-ups (below);
   - what is left for Bryan: reload nav, ledger-tools, the cycler, and block-id-prompt
     on the Mac; update Bob Mac Capture if not done; then pilot the research's six-step
     script, especially "remove a relation from the incoming side and predict which note
     changes". Also try chip ＋ / × on both rows, which exercises the real
     `api.relations` v1 wiring.

## Deliberately not doing (recorded as `PROPOSED FOLLOW-UP:` notes by `rollout`)

- **Cancel as duplicate of / merged into / superseded by…** in the Task Card `x` stage.
  It would write the CANCEL LOG reason and a labelled relation in one gesture, replacing
  hand-kept Merged-into / Merged-from pairs.
- **Opt-in, dry-run adoption** of existing `Merged into [[…]]`, `See [[…]]`, and
  `**RELATED:**` bullets.
- **"Make this a prerequisite"**, which would convert a relation into a Depends-On edge.
- **A label near-duplicate lint and a one-shot relabel.** Reopen if there are more than
  about 30 distinct labels after a month.
- **A configurable inverse-phrase table.**
- **A read-only `bob relations` report.** It would be top-level per `cli_rules`.
- **Writes on close, cancel, or reopen.** Relations are history; nothing changes.

## Risks

| Risk                                                                | Mitigation                                                                                                  |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| An engine claims relation lines as Task Links or tree-close targets | Registry predicate plus MC10–MC17 in every engine; `plugin-guards` ships before every writer                |
| Engines render different bytes                                      | One planner with a JS twin; LW14 and LK20; code-point sorting                                               |
| A link to an id that does not exist                                 | Minting rides with the write (same transaction or batch); reconcile skips quiet-deferred mints (LR5)        |
| Dropping authored data                                              | Reconcile never edits LINKS; it never prunes on uncertainty (LR13); malformed lines are left alone          |
| Cross-note undo drift                                               | Follow-ups are best-effort and visible; undo re-arms the hand-edit mirror; reconcile repairs the rest       |
| The live index is slow on a large vault                             | Measured in `relation-index`; prefilters; chunked idle scan; nav falls back to a scan                       |
| `~` prose collisions                                                | Digit, slash, strike, and escape guards and a teaching error; reopen by requiring `:` before a token counts |
| An old bob behind a new Mac app                                     | `relation_grammar` signal plus a submit-twice warning                                                       |
| "relation" / "relationship" confusion                               | The Words section, docs, and the glossary keep them distinct; no relation ever rides on dependency JSON     |
| Parallel bob-plugins phases collide                                 | Separate plugins per parallel pair; merge both sides of `package.json` and README rows                      |
