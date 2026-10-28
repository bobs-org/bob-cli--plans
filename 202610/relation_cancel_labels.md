---
tier: epic
title: Cancel labels — relation labels that cancel their source task
goal: 'Committing a relation whose label is listed in `relations.cancel_labels` (starting
  with `duplicate of` and `superseded by`, configured in chezmoi) cancels the relation''s
  source task in the same gesture, exactly as Task Card Cancel would: `[-]` plus `[cancelled::
  today]`, its links removed from today''s open Pomodoros, and blocked dependents
  recovered. Every surface names the task before you commit (⊘ plus words, `↵ link
  & cancel`), and nothing else ever cancels or reopens because of a label.

  '
decisions:
  cancel_stage_door:
    ask: Should the Task Card Cancel stage also get "⊘ Cancel as duplicate of… / superseded
      by…" rows?
    default: false
    why: About three uses a week; ship one door first and add rows later via the same
      writer
    answer: false
  memory_decision_record:
    ask: Should rollout add the decision record cancel-labels-cancel-their-source?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
  memory_glossary_relation:
    ask: Should rollout correct glossary:task-relation to name the one cancel-label
      status exception?
    memory:
    - glossary:task-relation
    default: true
    corrects: 'A relation is distinct from a Pomodoro [[task-link]]: it never blocks
      a task, counts as a Pomodoro Task Link, or changes its status, schedule, priority,
      or freshness.'
    answer: true
  memory_glossary_term:
    ask: Should the rollout phase add the glossary term "Cancel Label" (glossary:cancel-label)?
    memory:
    - glossary:cancel-label
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
phases:
- id: cancel-contract
  title: Cancel-label contract (§17), config reference, and shared cancel vectors
  depends_on: []
  size: medium
  description: 'cancel-contract: write docs/task-relations.md §17 Cancel labels (rule,
    config, source table, writes, verdicts, indication strings, JSON, refusal codes),
    amend §§2/8/13.2/14 and the hooks, capture, projects, configuration, json-outputs,
    and glossary docs, and commit tests/fixtures/task_cancel/vectors.json. The vectors''
    cancel and prune bytes are generated from nav''s existing Task Card Cancel planners,
    and the rule, key, and config sections are hand-authored.'
- id: cancel-engine
  title: relations.cancel_labels config and the Rust task_cancel engine
  depends_on:
  - cancel-contract
  size: large
  description: 'cancel-engine: parse relations.cancel_labels so that errors only ever
    reduce cancels, and add the pure src/native/task_cancel engine: verdict, line
    rewrite, Cancel Log prepend only when a log exists, the immediate-semantics prune
    of today''s open Pomodoros, and recovery-only dependent and ^prj recovery. It
    must pass every CX vector, and a test pins that reconcile never cancels.'
- id: capture-cancel
  title: Capture ~ / ~~ cancels the source in the same staged batch
  depends_on:
  - cancel-engine
  size: medium
  description: 'capture-cancel: on an Added or Relabeled relation with a cancel label,
    stage the source cancel, today''s prune, and recovery in the capture batch with
    dry-run parity. Adds the new-task, recurring, and conflict refusals, relation_cancels[]
    and relation_updates[].cancel_effect JSON, the relation_cancelled_source structured
    warning, the human ⊘ row, and the capture docs.'
- id: cancel-surfaces
  title: Completion, parse, relations list, rename, and zsh show cancel labels
  depends_on:
  - cancel-engine
  size: medium
  description: 'cancel-surfaces: capture-complete relation_label rows gain cancels_source,
    a configured zero-use overlay, and disabled_reason for refused sources, and the
    response gains relation_cancel {source, verdict, message, hint}. Also: capture-parse
    spans gain cancels_source; bob relations list shows ⊘ cancels source in human
    and JSON output; zsh descriptions; the relations rename warning (rename never
    cancels); config warnings; docs.'
- id: nav-cancel-writer
  title: Obsidian relation writer folds Task Card Cancel into the source write
  depends_on:
  - cancel-contract
  size: large
  description: 'nav-cancel-writer: mirror the vectors, add a cached cancel-label config
    loader and one pure planRelationCancelEffects planner, and fold planTaskCancelBatch
    into applyRelationEdit before stageWrite (one write, one undo). Extract a shared
    post-cancel helper (prune and recovery, no notice), show one composed notice,
    advance the walk once for a landed source, and re-plan at commit. Deploy with
    bob plugins sync.'
- id: nav-cancel-cue
  title: Obsidian Links stage ⊘ indication, batch confirm, and hints
  depends_on:
  - nav-cancel-writer
  size: medium
  description: 'nav-cancel-cue: add ⊘ cancels row meta and a configured-label overlay,
    a ⊘-prefixed dimmed source in the live sentence, the Task Card Cancel effects
    phrase in the consequence strip, ↵ link & cancel, and disabled recurring rows.
    Adds the second-Return banner for two or more cancels, the inverse, hand-edit,
    and relabel-away hints, light and dark styles, plugin docs, tests, and sync.'
- id: mac-cancel-cue
  title: Bob Mac Capture renders bob's cancel consequences
  depends_on:
  - capture-cancel
  - cancel-surfaces
  size: medium
  description: 'mac-cancel-cue: decode the new additive fields, then render the relation
    card ⊘ cancels capsule, the dimmed ⊘ source in the sentence, bob''s verdict line
    in the one-off warning slot, the ⊘ tint on the draft label span, and the preview
    and success rows. Restyles the relation_cancelled_source warning by code, adds
    fixtures from real bob output, tests, docs, and compiled Swift with green macOS
    CI.'
- id: rollout
  title: Turn it on in chezmoi, write the records, and run end-to-end acceptance
  depends_on:
  - capture-cancel
  - cancel-surfaces
  - nav-cancel-cue
  - mac-cancel-cue
  size: small
  description: 'rollout: add relations.cancel_labels [duplicate of, superseded by]
    to chezmoi home/dot_config/bob/config.yml and apply it per the chezmoi rules.
    Then write the accepted memory changes, drop every planned marker, check cross-repo
    doc links, run the end-to-end acceptance in a scratch vault, and report the host
    steps Bryan still has to run.'
proposed_by: bbugyi200.apollo.research.2r.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 08:15:24
status: done
bead_id: bob-cli-dl
---

- **PROMPT:** [prompts/202610/relation_cancel_labels.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/relation_cancel_labels.md)
- **BEAD:** [bob-cli-dl](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dl/README.md)

# Plan: Cancel labels — relation labels that cancel their source task

## Outcome

During the morning `]s` walk, Bryan lands on **Sell stocks!** in `cash.md` and realizes
it duplicates **Sell stocks now!**. He presses Task Card `l`, picks the newer task, and
presses `⇧↵`. He types `dup` into the label field:

```text
🪢 Link with a relation
⊘ Sell stocks! — duplicate of → Sell stocks now!
[ dup▏                                                              ]
▸ 🔗 duplicate of                       used 1× · ⊘ cancels
  🔗 implements                         used 10×
⊘ Cancels “Sell stocks!” · removes its links from today's open Pomodoros · unblocks dependents
↵ link & cancel   ⇥ accept   ⌃D hide   esc back
```

One Return makes one write to `cash.md`: the LINKS line, `[-]`, and
`[cancelled:: 2026-10-28]`. Ctrl+Z undoes all three. The task's link under today's open
Pomodoro disappears, its Blocked dependents recover, the walk moves on, and one notice
sums it up:
`🪢 Sell stocks! — duplicate of → Sell stocks now! · ⊘ cancelled · removed 1 Pomodoro link`.

On the Mac, `Intro v1 ~~sase_blog:write-intro|"superseded by"` shows a `⊘ cancels`
capsule on the relation card row. The sentence dims the source as
`⊘ Write introduction`, and the slot the `||` warning uses reads
`⊘ Cancels “Write introduction” · Next → Cancelled`. The draft's label span takes the ⊘
tint. The live preview adds:

```text
⊘ would cancel “Write introduction” · Next → Cancelled · would remove 1 Pomodoro link · would unblock “Finish full v1 draft”
```

`bob relations` in the terminal shows `⊘ cancels source` next to both labels. Labels
outside the config, hand edits, reconcile, renames, unlinks, and sync never touch a
status.

## Context

- **Research (accepted in full by Bryan):**
  `research:202610/cancel_labels_for_task_relations/cancel_labels_for_task_relations.md`.
  Read it with `sase artifact read` before starting any phase.
  - Its requirement adjustments **A1–A11** are requirements of this epic.
  - Its design (the rule, which task, what one commit writes, the edge cases, the
    indication, and the architecture) is the baseline.
  - This plan settles its open questions and splits the work:
    1. **Cancel Log:** none by default; prepend only when a log already exists.
    2. **`merged into`:** not configured; it stays a commented-out line in chezmoi.
    3. **A new capture task as source:** refused, with a `~~` hint.
    4. **Task Card Cancel-stage door:** the `cancel_stage_door` decision.
- **Words.** The request's "relation descriptions" are **relation labels**. RV1 in
  `docs/task-relations.md` §16 bans "description" in code and contracts. The configured
  ones are **cancel labels**.
- **Today, verified in code:**
  - **Relations are contractually inert:** `docs/task-relations.md` §§2, 8 ("Closes,
    cancels, reopens → None"), 13.2 (LR18/LX13/LG13), and 14. The nav stage copy says
    "Links only · never blocks, schedules, or changes status"
    (`305-relation-stage-copy.js`).
  - **Config:** `src/native/config/relations.rs` parses only `linked_from_lines`. Other
    `relations.*` keys are ignored, so an older bob safely ignores `cancel_labels`. The
    nav twin is `loadRelationsLinkedFromLinesEnabled` in `020-config-load-and-tasks.js`.
  - **Label identity:** `relation_label_key` (`src/native/relation_labels/mod.rs`),
    pinned for JS by `tests/fixtures/relation_labels/vectors.json`. The display-only
    inverse table is `task_relations::label::inverse_phrase` (`duplicate of` ⇄
    `duplicated by`, `superseded by` ⇄ `supersedes`).
  - **Capture:** `src/native/capture/relations.rs` (`plan_relation_writes`,
    `apply_links_to`, `apply_linked_from`) already classifies each edge as
    `RelationAction::{Added, Relabeled, Unchanged}`. Coded refusals use
    `CaptureError::usage_code`. Structured warnings use `CaptureWarning::Structured`
    (precedent: `hides_existing_relation_label`).
  - **bob-cli has no general cancel writer.** `[cancelled::]` is written only by
    reference-task code.
    - Capture completion (`capture/task_complete.rs`) is the template for a staged
      close.
    - Its pieces live in `src/native/task_complete/`: `retirement.rs` (scoped ledger
      retirement on the reconcile structural planner), `successors.rs`
      (`plan_successors`, which does recovery plus offers), and `project_close.rs`.
    - `recovery.rs` is a test-only parity oracle.
  - **Task Card Cancel in nav:**
    - `planTaskCancelBatch` (`240-cancel-and-lanes.js`) writes `[-]` and upserts
      `[cancelled:: date]`. It skips closed tasks and refuses recurring ones. With an
      empty reason it writes no Cancel Log, unless a log exists, in which case it writes
      `fallbackReason`.
    - `planDeferredPomodoroLinkCleanup` (`160-pomodoro-reorder.js`) is the immediate
      prune: a dedicated link bullet goes with its subtree, mixed prose loses only the
      token, and struck links, closed Pomodoros, and other days stay.
    - Orchestration lives in `540-plugin-decay-picker-and-cancel.js`
      (`applyTaskCancelFromEditor`) and `550-plugin-cancel-and-counted-property.js`
      (`finishTaskCancelNotice`). They write the source in one transaction, prune
      today's note afterwards (chip `Pomodoro links not removed` on failure), then call
      cycler `recoverBlockedDependents`, which makes no successor offers.
    - The effects phrase comes from `380-picker-cancel-and-lane.js`: "Removes its links
      from today's open Pomodoros · unblocks dependents".
  - **Nav relation writer:** `applyRelationEdit` in `594-plugin-relation-edit.js`
    composes each source note's LINKS edit and mints ids, then calls `stageWrite`;
    LINKED FROM mirrors are best-effort follow-ups (`595-relation-follow-ups.js`). The
    stage lives in `426-picker-relation-stage.js`, `427-relation-stage-plugin.js`,
    `428-picker-relation-label-vocabulary.js`, and `304`/`305`.
  - **Fragment budget:** each hand-edited nav fragment must stay at or below 1000 lines.
    `550` is at 984, `020` at 923, `240` at 912, `540` at 885, and `594` at 847. New
    code goes in **new fragments** registered in `src/fragments.json`.
  - **Mac:** the relation card (`CapturePanelView.swift` around the `prompt.oneOff`
    sentence, and `CapturePanelModel.swift`) inserts the label into the draft and never
    commits. Older apps already honor bob's `disabled_reason` on `relation_label` rows,
    and the success card renders every bob-authored warning message.
  - **⊘** is the vault's Cancelled status glyph (`docs/reference/vault-format.md`).
- **Vault facts (read-only):** `superseded by` has 2 links and `duplicate of` has 1. All
  three were cancelled by hand within seconds, with no Cancel Log; two of those cancels
  pruned a link from today's Pomodoro. No open task carries either label today, so
  turning the config on needs no migration.

## Design

`docs/task-relations.md` gains **§17 Cancel labels**, the normative form of CL1–CL8
below. `cancel-contract` writes it. Until its phase lands, a rule carries
`planned (phase <id>)`.

### CL1 The rule

> When an explicit relation gesture **adds** a cancel label to an open task, or
> **relabels** one of its links **into** a cancel label, the same gesture cancels that
> task (the relation's **source**) exactly as Task Card Cancel would. Nothing else
> cancels because of a label, and nothing ever reopens because of one.

- **Explicit gestures** are capture `~` / `~~` (CLI and Mac), the nav Links-to and
  Linked-from stages (`l` / `⇧L` → `⇧↵`), and counted or marked batches in those stages.
- **Never** fires on: reconcile, `bob relations rename`, hand edits of LINKS, agent
  edits, unlink, relabel _away_ from a cancel label, mirrors, vault sync, config
  changes, or an `Unchanged` re-add.
- **Fires once, at commit.** It is never a state that reconcile enforces. A reopened
  source stays open. Only a _new_ explicit add or relabel can cancel it again.

### CL2 Config

The config lives in chezmoi `home/dot_config/bob/config.yml`. bob's built-in default is
`[]`.

```yaml
# Task relations (docs/task-relations.md in bob-cli).
relations:
  # linked_from_lines: true   # write 🪢 LINKED FROM mirrors (default)
  # Committing a link with one of these labels also cancels the task the label
  # is phrased from ("S duplicate of T" cancels S). Matched like relation
  # labels (case, spacing, trailing colon ignored). Empty cancels nothing.
  cancel_labels:
    - duplicate of
    - superseded by
    # - merged into
```

- **Matching** uses `relation_label_key`. `Duplicate  Of:` matches; `duplicated by`,
  `supersedes`, and `dup of` do not. There is no substring or fuzzy matching.
- **Parsing fails toward fewer cancels.**
  - A missing key, `null`, or `[]` is a valid empty policy.
  - A non-sequence value warns
    `invalid relations.cancel_labels in <path>: expected a list of relation labels; cancelling nothing`
    and yields an empty set.
  - An entry that is not a string, or not a legal relation label (the writer's length
    and character rules), warns and is skipped.
  - Entries that share a key collapse.
  - A `cancel_labels` problem never becomes a `ConfigError`, and never changes
    `linked_from_lines`.
- **Hide is not disable.** A hidden or `||` one-off cancel label still cancels when
  typed (`relation-label-hide-is-suggestions-only`). To turn the effect off, change the
  config.
- **Catalog overlay.** A configured cancel label appears in suggestions and in
  `bob relations list -a` even with zero uses. It is marked `configured`, ranks as a
  zero-use row, and is never boosted. Hidden still wins.
- **Host-wide YAML**, like `linked_from_lines`. It never goes in the per-host label
  journal, and bob never edits `config.yml`.

### CL3 Which task is cancelled

The source is always the task the label is phrased from: "S duplicate of T" cancels S.

| Gesture                                      | Source S (cancelled)                      | Target T (survives)                      |
| -------------------------------------------- | ----------------------------------------- | ---------------------------------------- |
| Task Card `l` → `⇧↵`                         | task under the cursor                     | picked task                              |
| Task Card `⇧L` → `⇧↵`                        | **picked task**, possibly in another note | task under the cursor                    |
| Capture `@note+id ~t:id\|"duplicate of"`     | routed existing task                      | picked task                              |
| Capture `New task ~~old:id\|"superseded by"` | **picked existing task**                  | the new task                             |
| Capture `New task ~old:id\|"duplicate of"`   | — refused `relation_cancels_new_task`     | —                                        |
| Counted `N Ctrl+Shift+L` / marked rows       | each source                               | one target (second Return if ≥2 cancels) |

### CL4 What one commit writes

Every step below is existing cancel behavior. The only new part is composing them.

1. **S's note, one write.** The LINKS add or relabel, then `[-]` and an upserted
   `[cancelled:: today]` on S's task line. "Today" is bob's configured day timezone in
   Rust, and nav's existing Task Card Cancel date in JS.
   - There is no new Cancel Log.
   - If S already keeps a `❌ CANCEL LOG`, prepend `*date* — 🪢 duplicate of [[T]]`,
     using the same link text the LINKS entry uses. This is nav's `fallbackReason`. Two
     cancel edges on one source in one commit join in authored order, separated by `·`.
2. **T's LINKED FROM mirror.** The usual best-effort follow-up; reconcile stays the
   authority. `linked_from_lines: false` skips only this step.
3. **Today's daily note.** Prune S's links under **open** Pomodoros with Task Card
   Cancel's immediate semantics:
   - a dedicated link bullet goes with its subtree;
   - in mixed prose, only the token goes;
   - struck links, closed Pomodoros, top-level entry lines, fenced examples, text
     outside `## Pomodoros`, and other days stay.

   Never create the note: an absent note means 0 links, and an unreadable note is an
   error.

4. **Recovery.** S's Blocked dependents recover through the existing engines, stamping
   `[unblocked:: today]` (`unblocked-tasks-resurface-as-ticklers`). For a `^prj`, this
   includes residents blocked only by the project. There are **no successor offers** and
   no dependency transfer.
5. **Feedback.** One composed result names the cancel, the prune count, and how many
   tasks were unblocked.

For the `cash.md` case, the result is byte-identical to the hand-made cancel:

```markdown
- [-] #task Sell stocks! [fresh:: 2026-10-16] [created::2026-10-12] [priority::high]
  [scheduled:: 2026-10-22] [cancelled:: 2026-10-22] ^sell-stocks
  - 🪢 **LINKS:** duplicate of [[dev_plex_hack_patch#^sell-stocks-now]]
```

### CL5 Verdicts and edge cases

The cue and the writer use one verdict function per engine: Rust in `task_cancel`, JS
`planRelationCancelEffects`. The shared `rule` vectors pin both, so the cue can never
disagree with the write.

| Source state                                                                                    | Verdict             | Effect                                                               |
| ----------------------------------------------------------------------------------------------- | ------------------- | -------------------------------------------------------------------- |
| Open: `[ ]` Ready, `[*]` Next, `[/]` In Progress, `[?]` Blocked (the Task Card Cancel open set) | `cancels`           | link and cancel                                                      |
| `^prj` task, open                                                                               | `cancels` (project) | as Task Card `x`; copy adds "its N open tasks stay open"             |
| Done, Cancelled, or any status outside the open set                                             | `already_closed`    | link or relabel only; keep the original date; never Done → Cancelled |
| Recurring (`is_recurring_task_line`)                                                            | `recurring`         | refused before writing; cancel-label rows disabled                   |
| `~` source is the task this capture item creates                                                | `new_task`          | refused before writing, with a `~~` hint                             |
| Item also completes, starts, Pomodoro-links, or adds `&` / `*` to S                             | `conflict`          | refused `relation_cancel_conflicts_with_item_action`                 |

More edge cases:

- **Same edge, same label key (`Unchanged`):** no cancel and no log. Retries are safe.
- **Relabel from one cancel label to another** on an already-cancelled S: relabel only.
- **Relabel away, or unlink:** S stays Cancelled. The nav notice says
  `stays Cancelled · reopen with Alt+]` (verify the chord against the cycler keymap).
- **T is Cancelled or archived:** allowed. Amber caveat `“T” is Cancelled too`. T is
  never changed.
- **Inverse label typed** (`supersedes`, `duplicated by`): no cancel, plus a muted flip
  hint (CL6).
- **`bob relations rename` into or out of a cancel label:** never cancels. The preview
  warns that the config needs updating.
- **Midnight rollover or a config change between cue and commit:** re-plan at commit. If
  the consequence changed, nav writes nothing and re-renders for a fresh Return. Capture
  always plans at commit, so its result reports what happened.

### CL6 The indication

Name the task, change the verb, use ⊘, stay calm.

- **⊘ plus words**, in the existing Cancelled-status tint, marks the intended
  consequence. Meaning never depends on color alone.
- **Amber** stays reserved for caveats (T already cancelled, the `||` hide warning).
- **Red** stays reserved for errors and refusals.
- **No strikethrough** on a task that has not been cancelled yet. Bob already uses
  strikethrough for closed ledger children.

Canonical strings. `{S}` and `{T}` are task texts. `{note}` is the note stem, shown only
when S is not in the active note (nav), or for `~~` (capture). Plurals follow the count.

| Where                                                   | Text                                                                                                                                                   |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Row meta (nav) / capsule (Mac) / `relations list` / zsh | `⊘ cancels` / `circle.slash cancels` / `⊘ cancels source` / `N links · cancels source`                                                                 |
| Live sentence (nav, Mac)                                | `⊘ {S} — {label} → {T}`, source dimmed; remote: `⊘ {S} ({note}) — …`                                                                                   |
| Verdict `cancels` (bob `relation_cancel.message`)       | `⊘ Cancels “{S}” · {Status} → Cancelled`                                                                                                               |
| Consequence strip (nav)                                 | `⊘ Cancels “{S}” · removes its links from today's open Pomodoros · unblocks dependents` (no Pomodoro clause when S has no block id, as Task Card does) |
| `^prj` suffix                                           | `· its {N} open tasks stay open`                                                                                                                       |
| Verdict `already_closed` (muted)                        | `Already Cancelled · link only` / `Stays Done · link only`                                                                                             |
| Verdict `recurring` (disabled row)                      | `“{S}” repeats — cancel it with Obsidian Tasks first`                                                                                                  |
| Verdict `new_task` (disabled row, refusal)              | `“{S}” is new — it would be born cancelled · to retire “{T}”, use ~~`                                                                                  |
| Inverse hint (muted)                                    | `“supersedes” never cancels · to retire “{T}”, use ~~ with “superseded by”` (nav says `⇧L` instead of `~~`)                                            |
| Return hint (nav)                                       | `↵ link & cancel`                                                                                                                                      |
| Batch banner (nav)                                      | `⊘ Cancels {N} tasks — ↵ again to confirm`                                                                                                             |
| Notice (nav)                                            | `🪢 {S} — {label} → {T} · ⊘ cancelled · removed {n} Pomodoro link · unblocked {k}`                                                                     |
| Capture row / preview (bob-authored)                    | `⊘ cancelled “{S}” · {Status} → Cancelled · removed {n} Pomodoro link · unblocked “{D}”`; dry run uses `would cancel`, `would remove`, `would unblock` |
| Structured warning `relation_cancelled_source`          | `⊘ cancelled “{S}” ({label} “{T}”)`; dry run `⊘ would cancel …`                                                                                        |
| Hand-edit hint (nav, once per burst)                    | `⊘ “{label}” cancels only from the Links stage or capture · this task stays open`                                                                      |
| Rename warning                                          | `“{label}” is a cancel label in config.yml — update relations.cancel_labels`                                                                           |

### CL7 Contracts

All additions are additive, every `schema_version` stays 1, and every new field is
omitted when it does not apply, so older output stays byte-identical. Names below are
final.

- **`capture-complete`, `relation_label` context:**
  - Rows gain `cancels_source: true` when the row's key is configured, and
    `configured: true` on overlay rows.
  - A cancel-label row gets the existing `disabled_reason` when the verdict is
    `recurring` or `new_task`.
  - The response gains
    `relation_cancel: {source: {path, block_id?, text, status_symbol, status_name, project_open_tasks?}, verdict: "cancels"|"already_closed"|"recurring"|"new_task", message, hint?}`.
    It is present only when at least one cancel label is configured. It resolves only
    the token's source: the item owner for `~`, the `~~` token's task for `~~`. It never
    scans the vault.
- **`capture-parse`:** relation-label spans gain `cancels_source: true`.
- **`capture` (dry run and commit), per item:**
  - `relation_cancels[]: {source: {path, line, block_id, text}, target: {path, block_id, text}, labels[], action: "added"|"relabeled", previous_status, previous_status_name, project_open_tasks?, cancel_log: "none"|"prepended", pomodoro_links_removed, recovered[]: {path, line, block_id, text}, message}`.
  - `relation_updates[]` rows gain `cancel_effect: "cancelled"|"already_closed"`.
  - `warnings[]` gains `{code: "relation_cancelled_source", label, message}`. The human
    renderer prints the ⊘ row once and does not repeat the warning.
  - Refusal codes: `relation_cancels_new_task`, `relation_cancels_recurring_task`,
    `relation_cancel_conflicts_with_item_action`.
- **`bob relations list`:** JSON rows gain `cancels_source` and `configured`. The human
  header gains `· N cancel`.

### CL8 Architecture

Three layers, with one authority per surface:

1. **Policy:** label key → cancels or not, plus config validity.
2. **Pure effect plan:** edge delta plus snapshots → source transition, optional log
   line, daily prune, recovery, refusals, and summary.
3. **Existing writers:** apply the planned contents through today's guarded paths.

- bob owns capture (CLI and Mac). Swift renders bob's flags and strings and decides
  nothing (`mac-capture-is-a-thin-client`).
- nav keeps its JS planner and editor transaction: unsaved buffers, same-note
  transactions, and undo stay local. Obsidian never goes through `bob capture`.
- Never call the relation writer and then the cancel UI handler. Compose the plans
  before the write.
- **Shared vectors** `tests/fixtures/task_cancel/vectors.json` (bob-cli is the source of
  truth; bob-plugins copies it byte-for-byte) keep the two engines writing identical
  bytes:
  - `cancel` and `prune` bytes are **generated from nav's current Task Card Cancel
    planners**, so "just like other cancels" is checked, not assumed;
  - `rule`, `key`, and `config` are hand-authored from §17.
- **Atomicity, stated honestly:**
  - Capture: S's LINKS, status, and log, T's mirror, today's prune, and recovery share
    one staged, preimage-checked batch.
  - Obsidian, when S's note is the daily note: one transaction and one undo.
  - Obsidian, otherwise: S's note commits as one undo step. The prune, mirror, and
    recovery are follow-ups that show a chip on failure, and reconcile heals them. This
    is the contract Task Card Cancel already ships with.

### Rejected alternatives

| Alternative                                    | Why not                                                                                                                        |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Reconcile enforcement, or reopen-on-unlink     | Silent cancels on hand, agent, and sync edits; would mass-apply when the config lands; would fight a deliberate reopen forever |
| Hard-coded labels                              | Bryan asked for config, and `[]` protects other vaults                                                                         |
| A Cancel reason prompt between link and cancel | The label is the reason; a second prompt makes the fused gesture ugly                                                          |
| Transferring dependents to T                   | Would have created a cycle in the real vault (`^v1-draft`)                                                                     |
| Labels that cancel the target (`supersedes`)   | Breaks "cancel labels cancel their source"; the flip hint teaches the alternative                                              |
| Plugin-only MVP                                | Mac would silently not cancel                                                                                                  |
| Required ack token for effectful captures      | Slows every CLI capture to guard a skew window that rollout order closes                                                       |
| Generic relation-effects engine                | Far more surface than two labels need                                                                                          |

## Cross-cutting rules for every phase

- **Contracts first.** bob-cli §17 and the vectors land before the clients. JSON grows
  additively. Clients decode with `decodeIfPresent` or feature detection.
- **Shared vectors.** `tests/fixtures/task_cancel/vectors.json` is the single source.
  - Its sections are `cancel`, `prune`, `rule`, `key`, and `config`. Rust-only cases are
    marked `rust_only`.
  - bob-plugins copies it byte-for-byte, with a README that records the bob-cli base
    commit.
  - A behavior change edits bob-cli first; never hand-edit the copy.
- **CLI and docs rules.** Before touching commands, help, JSON, or docs, read
  `cli_rules.md` and `docs.md` with `/sase_memory_read`. Run `just docs-gen` when help
  changes, then `just docs-check`. Plugin docs use `npm run docs:check`; Mac docs use
  `just docs-check`.
- **Release train.** Never edit `Cargo.toml` versions or `CHANGELOG.md`.
  - Use `feat(relations): …`, `feat(capture): …`, and `feat(nav): …` commits.
  - Nothing here is a contract break. The default is `[]` and all JSON is additive, so
    there is no `!`.
- **Cross-repo work.**
  - Open bob-plugins, bob-mac-capture, chezmoi, and the research sidecar only through
    `/sase_repo`, and read each repo's `AGENTS.md`.
  - Plugin phases: edit `src/` fragments (each ≤1000 lines, new fragments registered in
    `fragments.json`), run `npm run build` and `npm test`, then `bob plugins sync`.
- **Mac builds.** Follow bob-mac-capture's `AGENTS.md`.
  - Never land uncompiled Swift. Run `just portable-check`, using athena if the local
    host lacks Swift.
  - Make a best-effort scratch compile on the mac (`bundle.sh` plus
    `just typecheck-tests`).
  - A Mac phase closes only on green macOS CI. If the mac is unreachable, say so.
- **Vault safety.**
  - Tests use scratch vaults with `BOB_DIR`, `BOB_DAY_FILE`, `BOB_NOW`, and
    `BOB_CONFIG_FILE`.
  - `~/bob` is read-only for every phase.
  - The live config changes only in `rollout`, through chezmoi.
- **Inert until rollout.** Every phase before `rollout` must leave behavior unchanged
  while `relations.cancel_labels` is absent. Prove it: run each touched suite's existing
  relation tests unchanged.
- **Baselines and follow-ups.**
  - Compare test failures against a clean-base run.
  - Phase workers create no beads. They record discoveries as `PROPOSED FOLLOW-UP:`
    notes on their phase bead.

## Phases

### cancel-contract

**Repo:** bob-cli (docs and fixtures). bob-plugins is read-only, used only to generate
bytes.

1. **`docs/task-relations.md` §17 Cancel labels.** Write CL1–CL8 normatively:
   - the rule;
   - the config;
   - the source table;
   - the writes;
   - the verdict and edge tables;
   - the canonical strings;
   - the JSON fields and refusal codes;
   - a CL vector index.

   Mark each rule `planned (phase <id>)`.

2. **Amend, don't delete.**
   - §2: keep every "never" and add one bullet, "except the commit-time cancel of §17".
   - §8: add the row "Cancel-label commit → §17". Keep "Closes, cancels, reopens → None"
     for lifecycle events.
   - §13.2: carve §17 out of LR18/LX13/LG13.
   - §14: relabel-away or unlink leaves S Cancelled.
   - §15: add the vector
     `reconcile never cancels an open source carrying a cancel label`.
3. **Other bob-cli docs:**
   - **`docs/task-status-hooks.md` "Canceled Task References":** fix the claim that the
     Cancel row and reconcile "re-derive the same cleanup". Reconcile also removes
     struck links and mixed-prose items that the immediate prune keeps. Name the
     cancel-label commit as another immediate-prune caller.
   - **`docs/reference/configuration.md` `relations`:** `relations.cancel_labels`
     (default `[]`, key matching, fail toward fewer, hide is not disable).
   - **`docs/capture.md`:** amend the Inertness paragraph and the `~` / `~~` relation
     overview to point at §17.
   - **`docs/projects.md`:** a cancel-label commit on a `^prj` cancels it as Task Card
     `x` does, and residents stay open.
   - **`docs/reference/json-outputs.md`:** stub the CL7 fields as planned.
   - **`docs/reference/glossary.md`:** add **Cancel label**.
4. **Vectors** `tests/fixtures/task_cancel/vectors.json` plus a `README.md`.
   - **`cancel` section:** source-note preimage, target line, date, and
     `fallbackReason`, mapped to the postimage. Generate it with a throwaway Node
     harness that loads bob-plugins' built nav `main.js` the way
     `scripts/test-navigation-relation-writer.cjs` does, and runs `planTaskCancelBatch`
     with `reason: ""`. Cover:
     - each open status;
     - fields present or absent (`fresh`, `created`, `priority`, `scheduled`, `id`,
       `dependsOn`, `^block-id`);
     - an existing `❌ CANCEL LOG` (prepend) and no log (none);
     - already Done or Cancelled (unchanged);
     - recurring (refused);
     - a `^prj` line;
     - the `cash.md` example from CL4.
   - **`prune` section:** daily preimage and block ids, mapped to the postimage and a
     removed count. Generate it from `planDeferredPomodoroLinkCleanup`. Cover:
     - a dedicated bullet with children;
     - a mixed-prose token;
     - a struck link (kept);
     - a closed Pomodoro (kept);
     - a top-level entry line (kept);
     - a fenced example (kept);
     - a link outside `## Pomodoros` (kept);
     - several links;
     - an empty daily note.
   - **`rule` section (hand-authored):**
     - inputs: configured labels, label, previous label, action
       (`added`/`relabeled`/`unchanged`), direction, source state, whether the source is
       new, and item modifiers;
     - outputs: verdict and `cancels: bool`.

     Cover every CL5 row, every CL1 "never" path, and the CL3 table.

   - **`key` section:** cancel-label matching cases from CL2.
   - **`config` section:** YAML text mapped to the effective keys and the warning count.
     Cover missing, `null`, `[]`, a scalar, a mapping, a non-string entry, an illegal
     label, duplicate keys, and an invalid `linked_from_lines` beside a valid
     `cancel_labels`.
   - **README:** provenance (the bob-plugins commit), how to regenerate the generated
     sections, and the rule that hand-edited sections change only with §17.
   - Review every generated postimage by hand against CL4. If nav's bytes disagree with
     CL4, stop and record it on the bead rather than encoding a surprise.

5. **Acceptance:**
   - `just docs-check` is green.
   - The vectors file parses.
   - A tiny Rust test loads the file and asserts that its sections exist, so the fixture
     cannot silently rot before `cancel-engine`.

### cancel-engine

**Repo:** bob-cli. This phase changes no user-visible behavior.

1. **Config** (`src/native/config/relations.rs`):
   - Add `cancel_labels` to `RelationsConfig`: an ordered set of keys plus display
     names.
   - Add `warnings: Vec<String>`.
   - Implement CL2 parsing. Validate labels with the writer rules in
     `task_relations::label`.
   - Add helpers `is_cancel_label(key)` and
     `cancel_label_for_inverse(key) -> Option<&str>`, which use `inverse_phrase`.
   - Pass the `config` vectors.
2. **New module `src/native/task_cancel/`**, beside `task_complete/`. Everything is
   `pub(crate)` and pure over snapshots, with no IO.
   - **`verdict`:** `cancel_verdict(source_line, tasks_settings, is_new_task)` returns
     `Cancels{previous_status, project}`, `AlreadyClosed{status}`, `Recurring`, or
     `NewTask`. It uses the Task Card Cancel open set and `is_recurring_task_line`.
   - **`line`:** status flip plus the `[cancelled:: date]` upsert, byte-identical to the
     `cancel` vectors.
   - **`log`:** prepend the fallback entry only under an existing `❌ CANCEL LOG` (reuse
     `is_cancel_log_line` / `is_cancel_log`).
   - **`prune`:** the immediate-semantics prune scoped to open Pomodoros in today's
     daily note. Build it on bob's Pomodoro scanning
     (`task_status_hooks::scan_pomodoros` and friends), not on reconcile's subtree rule,
     and never run reconcile. It must be byte-identical to the `prune` vectors.
   - **`recovery`:** reuse the successor planner's recovery path with **offers
     suppressed**. That covers dependents, `^prj` residents blocked only by the project,
     and `[unblocked:: today]`. If `plan_successors` cannot suppress offers cleanly, add
     a narrow recovery-only mode. Never duplicate the open-dependency rule.
   - **Entry point:**
     `plan_task_cancel(input) -> TaskCancelPlan { source_text, daily_text: Option, cancel_log, previous_status, pomodoro_links_removed, recovered, writes }`,
     where `writes` are the recovery files' contents.
3. **Tests:**
   - every `cancel`, `prune`, `rule` (verdict part), `key`, and `config` vector;
   - recovery parity: cancelling S recovers exactly the dependents that completing S
     recovers, with no offers;
   - a `^prj` source recovers residents blocked only by the project;
   - an absent daily note gives 0, and an unreadable one gives an error;
   - New York midnight via `BOB_NOW`;
   - the reconcile pin: with `cancel_labels` configured, `bob task-status-hooks`
     reconcile leaves an open task whose LINKS carry `duplicate of` open, and is
     byte-stable.
4. **Docs:** drop the `planned` markers for the config and engine rules in §17. Update
   `docs/reference/configuration.md` if the warning text differs from the plan.

### capture-cancel

**Repo:** bob-cli.

1. **Collect.** In `capture/relations.rs`, after an item's relation writes are planned,
   take each `relation_updates` row with `Added` or `Relabeled` whose label key is a
   cancel label.
   - The source is the item owner for `~` (`apply_links_to`) and the picked task for
     `~~` (`apply_linked_from`).
   - Dedupe by source identity: one cancel per source, with the triggering labels in
     authored order.
2. **Refuse** with coded usage errors (`CaptureError::usage_code`). Capture is
   all-or-nothing, so nothing is written.
   - `relation_cancels_new_task`: a `~` cancel label on the task this item creates.
   - `relation_cancels_recurring_task`.
   - `relation_cancel_conflicts_with_item_action`: the same item also completes (`!`),
     starts (`=`), Pomodoro-links (`:`/`@route:id`), or adds `&` / `*` on S.
   - Messages are the CL6 strings.
3. **Stage** the `task_cancel` plan through the `CaptureBatchPlanner` overlays:
   - S's note, which is the same staged file as its LINKS edit;
   - today's daily note prune, in the same batch and never created;
   - the recovery writes.

   `already_closed` sources get the link only, with
   `relation_updates[].cancel_effect: "already_closed"`. Dry run plans the same bytes.

4. **Cross-item.** A later item sees an earlier cancel through the staged overlays. Test
   that an item which links, starts, or completes a task an earlier item cancelled is
   refused. If the existing open-task checks do not refuse it, add a coded refusal.
5. **Output** (CL7): per-item `relation_cancels[]`, `cancel_effect`, the
   `relation_cancelled_source` structured warning (`would` phrasing on dry run), and the
   human `⊘ cancelled …` row printed once.
6. **Docs:**
   - `docs/capture.md`: the `~` / `~~` cancel-label behavior, JSON, and refusals;
   - `docs/reference/json-outputs.md`;
   - drop the capture markers in §17.
7. **Tests**, in LX style, in a scratch vault:
   - `~` from an existing owner;
   - `~~` retiring an old task;
   - each refusal;
   - already Done, and already Cancelled;
   - `Unchanged`: re-running the same draft is a no-op the second time;
   - relabel into a cancel label;
   - two cancel edges on one source (one cancel, one log line);
   - S living in today's daily note (one staged file);
   - today's note absent;
   - `linked_from_lines: false`;
   - hidden and `||` cancel labels still cancel;
   - a `^prj` source;
   - a dependent that recovers vs. one still blocked by another prerequisite;
   - New York midnight;
   - dry-run output equals the commit plan;
   - with no config, output is byte-identical to today's.

### cancel-surfaces

**Repo:** bob-cli.

1. **`capture-complete` `relation_label`:**
   - `cancels_source` and `configured` on rows;
   - the zero-use overlay of configured labels, which respects hidden;
   - `relation_cancel {source, verdict, message, hint?}`;
   - `disabled_reason` on cancel-label rows for `recurring` and `new_task`;
   - `hint` when the typed key is the inverse of a cancel label.

   Resolve the source by note path and block id from the token or owner, never with a
   vault scan. Stay inside the existing relation-label latency budget and record the
   numbers.

2. **`capture-parse`:** `cancels_source` on relation-label spans.
3. **`bob relations list`:**
   - human output shows the `⊘ cancels source` column and the header count;
   - JSON gains `cancels_source` and `configured`;
   - `-a` includes zero-use configured labels;
   - zsh descriptions read `N links · cancels source` (`src/native/completion`).
4. **`bob relations rename`:** the CL6 warning in preview and apply when OLD or NEW is a
   cancel label. A test proves rename never changes a status.
5. **Config warnings** print once per command on stderr and appear in JSON `warnings[]`
   where the command has one.
6. **Docs:**
   - `docs/task-relations.md` §16 RV7 and §17;
   - `docs/reference/json-outputs.md`;
   - the CLI reference through `just docs-gen`;
   - drop the matching markers.
7. **Tests:**
   - each verdict through `capture-complete` for `~` and `~~`, for new and existing
     owners;
   - overlay and hidden interplay;
   - the inverse hint;
   - parse spans;
   - list output, human and JSON;
   - the zsh description;
   - the rename warning;
   - with no config, every output stays byte-identical.

### nav-cancel-writer

**Repo:** bob-plugins (`plugins/bob-navigation-hotkeys`).

1. **Vectors.** Copy `tests/fixtures/task_cancel/vectors.json` byte-for-byte, with a
   README recording the bob-cli commit. Add a test script, wired into `npm test`, that
   runs:
   - `planTaskCancelBatch` and `planDeferredPomodoroLinkCleanup` over `cancel` and
     `prune`;
   - the new config loader over `config`;
   - the JS relation-label key over `key`;
   - `planRelationCancelEffects` over `rule`.
2. **Config loader** `loadRelationsCancelLabels()`, in a new fragment. Clone
   `loadRelationsLinkedFromLinesEnabled`:
   - cache by mtime and size, so an edit takes effect on the next gesture;
   - fail toward fewer;
   - warn once on the console.
3. **Planner** `planRelationCancelEffects({report, sources, cancelLabels, ...})`. It is
   pure and returns, per source: verdict, `cancels`, the fallback log reason
   (`🪢 {label} [[T]]`, joined in authored order), and refusals. It is the only verdict
   function in nav; `nav-cancel-cue` reads it.
4. **Writer.** In `applyRelationEdit`, after the planner reports an edge `added` or
   `relabeled` with a configured key on a source whose verdict is `cancels`:
   - Fold
     `planTaskCancelBatch(content, {targets: [source]}, {date, reason: "", fallbackReason})`
     into that source's content **before** `stageWrite`. This gives one write and one
     undo group for LINKS, status, and log.
   - When S's note is today's daily note, fold the prune into the same content.
   - A `recurring` verdict refuses before writing, as a guard.
   - Re-plan at commit. If the verdict differs from the one the stage displayed, write
     nothing and return `{stale: true}` so the stage re-renders.
5. **Shared post-cancel helper.** Extract the post-write half of
   `applyTaskCancelFromEditor` and `finishTaskCancelNotice` into a new fragment:
   `runPostCancelEffects(...)`.
   - It runs the deferred prune when the daily note is not S's note, then cycler
     `recoverBlockedDependents` (no offers).
   - It returns `{pomodoroLinksRemoved, pruneFailed, recovered}` and never closes
     pickers or shows notices.
   - Task Card Cancel calls it and stays byte-for-byte identical. Existing cancel tests
     stay green unchanged.
6. **One composed notice** (CL6), with the existing failure chips
   (`Pomodoro links not removed`, `🪢 LINKED FROM not updated`). Never show a relation
   toast followed by a cancel toast.
7. **Walk (A10).** When the cancelled S is the row `]s` landed on, advance once through
   `reviewWalk` (`decisions:answering-advances-the-walk`). A remote S (`⇧L`) leaves the
   walk where it is.
8. **Header comment.** Amend the `594` header (LG13): the writer touches Pomodoros only
   through the §17 cancel.
9. **Tests:**
   - same-note and cross-note sources;
   - S is today's daily note;
   - `⇧L` with a remote S in an open editor and in a closed file;
   - `Unchanged` is a no-op;
   - relabel into a cancel label;
   - already closed;
   - recurring refusal;
   - `^prj`;
   - one undo step restores LINKS and status;
   - a stale re-plan writes nothing;
   - walk advance vs. stay;
   - hand-edit mirror and unlink never cancel;
   - with no config, writes are byte-identical.
10. `npm run build`, `npm test`, `bob plugins sync`. The feature stays inert because the
    config is absent.

### nav-cancel-cue

**Repo:** bob-plugins.

1. **Rows:**
   - a trailing `⊘ cancels` meta on rows whose key is configured, typed rows included;
   - a configured zero-use overlay, ranked as zero-use and respecting hidden;
   - a `recurring` verdict disables cancel-label rows with the CL6 reason.
2. **Live sentence** (`relationStageLiveSentence`): a ⊘ prefix and a dimmed source for
   `cancels`, with the note stem when S is remote.
3. **Consequence strip** (`305`): for a cancel selection, the CL6 effects phrase,
   reusing the `380` wording and its `hasBlockId` rule.
   - Add the `^prj` suffix when S is a project task.
   - Show the muted `already_closed` line when S is already closed.
   - Show the amber `“{T}” is Cancelled too` caveat when T is cancelled.
   - The subtitle's "does not block" stays.
4. **Footer:** `↵ link` becomes `↵ link & cancel` for a cancel selection.
5. **Batch.** When a commit would cancel two or more tasks:
   - the first Return shows `⊘ Cancels {N} tasks — ↵ again to confirm`, reusing the
     decay-cancel two-press banner pattern;
   - the second Return commits;
   - Esc writes nothing;
   - any change of selection or query disarms it.
6. **Hints:**
   - the inverse flip hint;
   - the relabel-away/unlink notice `stays Cancelled · reopen with Alt+]`;
   - the once-per-burst hand-edit hint when a hand edit adds a cancel label to an open
     task's LINKS. The hand-edit path still never cancels.
7. **Styles** (`styles.css`): a ⊘ token in the existing Cancelled tint, amber only for
   caveats, red only for errors, and no strikethrough. Check light and dark.
8. **Docs:** plugin `docs/task-card.md` (Links stage, footer, two-press) and
   `docs/task-relations.md` (cancel labels, with a link to bob-cli §17). Run
   `npm run docs:check`.

> [!decision] cancel_stage_door Also add `⊘ Cancel as duplicate of…` and
> `⊘ Cancel as superseded by…` rows to the Task Card Cancel stage. Each opens the
> Links-to picker with its label preselected, so the stage shows the full cue, and
> commits through the same writer. Pressing `⊘ Cancel as…` never cancels without a
> target. Document the rows in `docs/task-card.md`.

9. **Tests:**
   - copy for every verdict;
   - footer swap;
   - disabled recurring rows;
   - the two-press arm, disarm, and Esc;
   - the overlay and hidden interplay;
   - each hint;
   - with no config, the copy is unchanged.

   Then `npm run build`, `npm test`, `bob plugins sync`.

### mac-cancel-cue

**Repo:** bob-mac-capture.

1. **Decode** with `decodeIfPresent`:
   - `cancels_source` and `configured` on `relation_label` rows;
   - `relation_cancel` on completion responses;
   - `cancels_source` on parse spans;
   - `relation_cancels[]` and `relation_updates[].cancel_effect` on capture and dry-run
     results;
   - the `relation_cancelled_source` warning code.

   Swift computes no verdict.

2. **Relation card:**
   - a `circle.slash` + `cancels` capsule on `cancels_source` rows;
   - when the selected or typed label cancels and the verdict is `cancels`, the sentence
     source gets ⊘ and dims;
   - bob's `relation_cancel.message` shows in the slot the `||` warning uses: ⊘ tint for
     `cancels`, muted for `already_closed`, and bob's `disabled_reason` rows render
     disabled as today;
   - `hint` renders muted;
   - VoiceOver says "Cancels Sell stocks!".
3. **Draft:** a new `CaptureEditorPalette` role tints `cancels_source` label spans.
4. **Preview and success:**
   - one row per `relation_cancels[]` under the 🪢 row, using bob's message;
   - the `relation_cancelled_source` warning renders in the ⊘ style (matched by code),
     never as an orange duplicate of that row;
   - if the panel shows a submit hint, it reads `capture & cancel 1 task` while the
     latest dry run has cancels.
5. **Fixtures and tests:**
   - Record fixtures from real bob output: `capture-complete`, `capture-parse`,
     `capture --dry-run`, and a commit, in a scratch vault with `BOB_CONFIG_FILE`.
   - Test decoding with and without the fields (old bob);
   - capsule, sentence, and verdict slot per verdict;
   - the span tint;
   - the preview and success rows;
   - warning restyling with no duplicate.
6. **Docs:**
   - `docs/relationship-pickers.md`;
   - `docs/capture-previews.md`;
   - a `docs/protocol.md` compatibility-table row (older bob: no cue; upgrade bob);
   - `just docs-check`.
7. **Build** per `AGENTS.md`, with green macOS CI. If the mac is reachable, attach light
   and dark screenshots of the card and the preview to the phase note.

### rollout

**Repos:** chezmoi, and bob-cli memory and docs.

1. **Chezmoi.** Add the CL2 `relations:` block to `home/dot_config/bob/config.yml`:
   - `linked_from_lines` stays commented as the default;
   - `merged into` stays commented out.

   Commit, then run `chezmoi update -a --force` as chezmoi's `AGENTS.md` requires. This
   is the switch for the host where it runs.

2. **Check the installed bob** on that host: `bob relations list -a -f json` must report
   `cancels_source`. If the installed bob predates `capture-cancel`, still land: an
   older bob ignores the key, which means fewer cancels, never more. Say so in the
   report.
3. **Memory.** Make the accepted memory changes through `/sase_memory_write`, written
   from what shipped, then run `sase memory init`.

> [!decision] memory_decision_record Add
> `decisions/cancel-labels-cancel-their-source.md`. Applies to: capture `~` / `~~`, the
> nav Links and Linked-from stages, and `relations.cancel_labels`.
>
> - **Claim:** CL1, including "fires once at commit, never enforced, never reopens".
> - **Why:** the three hand cancels, industry precedent, and inertness everywhere else.
> - **Rejected alternatives:** the table above.
> - **Cost:** cross-note undo covers S only, and a false unblock is possible.
> - **What would reopen it:** a request for a second relation effect, or for target-side
>   cancels.
> - **Evidence:** this plan, the research ref, and the landed commits.

> [!decision] memory_glossary_relation Rewrite the last sentence of
> `glossary:task-relation`: a relation never blocks a task, counts as a Pomodoro Task
> Link, or changes its schedule, priority, or freshness. The one status exception is
> committing a [[cancel-label]] through capture or the Links stage, which cancels the
> source at commit time ([[decisions:cancel-labels-cancel-their-source]]).

> [!decision] memory_glossary_term Add `glossary/cancel-label.md` ("Cancel Label"). It
> is a relation label listed in `relations.cancel_labels` (chezmoi
> `home/dot_config/bob/config.yml`), matched by `relation_label_key`. Committing one
> through capture `~` / `~~` or the Links stage cancels the relation's source exactly as
> Task Card Cancel would. Hand edits, reconcile, and rename never do. Link
> [[task-relation]].

4. **Docs sweep:**
   - drop every remaining `planned (phase …)` marker;
   - check cross-links between bob-cli §17, `docs/capture.md`, the reference pages, the
     plugin docs, and the Mac docs;
   - `just docs-check`.
5. **End-to-end acceptance** (below), in a scratch vault, with the results recorded in
   the phase note.
6. **Report** what Bryan still has to do:
   - install the new Bob Mac Capture build (agents never `just install`);
   - run `chezmoi update` on the other hosts;
   - `bob plugins sync` has already run.

   Record the Task Card door (if declined) and a Mac Similar Tasks "Replace it" action
   (`~~old|"superseded by"`) as `PROPOSED FOLLOW-UP:` notes.

## End-to-end acceptance

1. **Capture, forward.** In a scratch vault with the CL2 config,
   `@cash+sell-stocks ~cash:sell-stocks-now|"duplicate of"` produces the CL4 bytes, and
   today's open-Pomodoro link is removed. Re-running it is a no-op.
2. **Capture, reverse.** `Intro v1 ~~sase_blog:write-intro|"superseded by"` cancels the
   old task and recovers `^v1-draft` with `[unblocked:: today]`. `--dry-run` reports the
   same plan with `would` phrasing.
3. **Refusals.** `New task ~old|"duplicate of"`, a recurring source, and a conflicting
   `!` on S each refuse with their code, and nothing is written.
4. **Never paths.** A hand edit, reconcile, `bob relations rename`, unlink, and relabel
   away each leave every status byte-identical.
5. **Agreement.** Rust and JS pass every shared vector. `bob relations list -f json`,
   `capture-complete`, and the nav stage agree on which labels cancel.
6. **Inert without config.** With `relations.cancel_labels` absent, every changed
   command's output and every nav write is byte-identical to before.
7. **Indication.** The nav copy tests and the Mac fixtures show the CL6 strings for
   every verdict. Mac screenshots are attached if the mac is reachable.

## Risks

- **Semantic creep.** There is one carve-out, at commit time only. It is recorded as a
  decision. §2 still lists everything that never cancels, and a reconcile vector pins
  the boundary.
- **Engine drift (Rust ↔ JS).** The cancel and prune bytes are generated from nav's
  current planners and pinned on both sides. Verdicts share one `rule` table.
- **Surprise or remote cancel.** Every surface names S, and its note when S is remote,
  before commit. Return's verb changes, and two or more cancels need a second Return.
- **False unblock** ("A superseded by B" releases A's dependents while B is open). This
  is standard cancel behavior. The cue says "unblocks dependents", and released tasks
  resurface as TICKLER.
- **Old Mac app with new bob.** Rollout ships last. Older apps already honor
  `disabled_reason`, and they show the `relation_cancelled_source` warning text in
  previews and on the success card.
- **Fragment and model size.** New nav fragments keep each file at or below 1000 lines.
  Mac changes stay scoped to the relation card and the preview rows.
- **Undo honesty.** Ctrl+Z restores S's note (LINKS and status together), not a
  cross-note prune, which matches Task Card Cancel. A guarded cross-note undo would help
  every cancel and belongs to a separate feature.

## Out of scope

- Reconcile enforcement, reopen-on-unlink, dependency transfer, target-side cancel
  labels, a generic relation-effects engine, and configuring `merged into`.
- The Mac Similar Tasks "Replace it" action, and the Task Card Cancel-stage door unless
  `cancel_stage_door` is accepted. Both are recorded as follow-ups by `rollout`.
