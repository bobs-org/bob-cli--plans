---
tier: epic
title: Complete labels — relation labels that complete their source task
goal: 'Committing a relation whose label is listed in `relations.complete_labels`
  (starting with `implemented by` and `verified by`) marks the relation''s source
  task Done in the same gesture, exactly as that surface''s managed Done gesture would.
  Every surface names the task before commit (✓ plus words, `↵ link & complete`).
  Bryan''s 14 `implements` links are reversed into `implemented by` by a new, previewed,
  revertable `bob relations rename --reverse` transaction.

  '
decisions:
  offer_successors:
    ask: Should a complete-label commit offer successors (the Done picker) like Ctrl+Enter
      and `!` do?
    default: true
    why: It is a managed Done; decisions:unblocked-tasks-are-offered covers every
      Done gesture
    answer: true
  memory_glossary_relation:
    ask: Should rollout correct glossary:task-relation to name both closing-label
      status exceptions?
    memory:
    - glossary:task-relation
    default: true
    corrects: The one status exception is committing a configured cancel label through
      capture or the Links stage, which cancels the source task at commit time.
    answer: true
  memory_decision_record:
    ask: Should rollout add the decision record closing-labels-close-their-source
      (cancel and complete labels)?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: complete-contract
  title: Complete-label contract (§18), reverse-rename contract, inverse pairs, and
    shared vectors
  depends_on: []
  size: medium
  description: 'complete-contract: write docs/task-relations.md §18 and RV10, amend
    §§1/2/8/10/13.2/14/15 and the capture, config, JSON, dependency, and project docs,
    add the two inverse pairs in Rust, and add tests/fixtures/relation_complete/vectors.json
    generated from the cycler''s Done bytes.'
- id: complete-capture
  title: relations.complete_labels config and capture ~ / ~~ completes the source
  depends_on:
  - complete-contract
  size: large
  description: 'complete-capture: parse complete_labels with the both-lists rule,
    add the complete verdict, extract the `!` engine path into one reusable existing-task
    completion, and stage complete-label commits in the capture batch with JSON, refusals,
    offers, and human rows.'
- id: complete-surfaces
  title: Completion, parse, relations list, rename warnings, and zsh show complete
    labels
  depends_on:
  - complete-capture
  size: medium
  description: 'complete-surfaces: add completes_source flags, the configured overlay,
    relation_complete verdicts and hints to capture-complete and capture-parse, the
    ✓ column and counts to bob relations list and zsh, and the complete-label rename
    warning.'
- id: reverse-rename
  title: bob relations rename --reverse moves each edge to its other end
  depends_on:
  - complete-surfaces
  size: large
  description: 'reverse-rename: add -r/--reverse to bob relations rename so S OLD
    T becomes T NEW S in one previewed transaction and one revertable commit, with
    mirrors, minting, merge, refusals, JSON, and beautiful human preview.'
- id: cycler-done-api
  title: task-status-cycler API v3 exposes its Done line rewrite and post-Done pass
  depends_on:
  - complete-contract
  size: medium
  description: 'cycler-done-api: expose completeTaskLineText and finalizeClosedTasks
    on the cycler api (version 3), refactor Ctrl+Enter onto them byte-identically,
    and pin the done vectors in bob-plugins.'
- id: nav-complete-writer
  title: Obsidian relation writer folds the Done into the source write
  depends_on:
  - cycler-done-api
  size: large
  description: 'nav-complete-writer: load closing labels with the both-lists rule,
    add planRelationCompleteEffects, fold the cycler''s Done line into the source
    write, gate stale commits, run the post-Done pass after the stage closes, compose
    one notice, and settle the walk once.'
- id: nav-complete-cue
  title: Obsidian Links stage ✓ indication, mixed batch confirm, hints, and inverse
    chips
  depends_on:
  - nav-complete-writer
  size: medium
  description: 'nav-complete-cue: render the ✓ completes row meta, live sentence,
    consequence strip, footer, mixed two-press banner, hints, green styles, and the
    new inverse pairs in nav and ledger-tools chips, then document it.'
- id: mac-complete-cue
  title: Bob Mac Capture renders bob's complete consequences and offers
  depends_on:
  - complete-surfaces
  size: medium
  description: 'mac-complete-cue: generalize the cancel cue into one close-effect
    presentation, decode the complete fields, render the ✓ capsule, sentence, slot,
    span tint, preview rows, and submit hint, and feed complete-label offers into
    the in-panel successor choice.'
- id: rollout
  title: Migrate implements, turn it on in chezmoi, record memory, and run acceptance
  depends_on:
  - reverse-rename
  - nav-complete-cue
  - mac-complete-cue
  size: medium
  description: 'rollout: reverse Bryan''s 14 implements links with the new rename,
    add complete_labels to chezmoi, make the accepted memory edits, sweep docs, and
    run end-to-end acceptance.'
proposed_by: bbugyi200.apollo.aw
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 14:49:51
status: done
bead_id: bob-cli-du
---

- **PROMPT:** [prompts/202610/relation_complete_labels.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/relation_complete_labels.md)
- **BEAD:** [bob-cli-du](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-du/README.md)

# Plan: Complete labels — relation labels that complete their source task

## Outcome

During the morning `]s` walk, Bryan lands on the reading task **Complete labels:
relation labels that complete their source task** in `bob.md`. He has read it and has
already captured the feature task that will build it. He presses Task Card `l`, picks
the feature task, and presses `⇧↵`. He types `impl` into the label field:

```text
🪢 Link with a relation
✓ Complete labels: relation labels that… — implemented by → Allow relation descriptions to configure auto-complete!
[ impl▏                                                              ]
▸ 🔗 implemented by                       used 14× · ✓ completes
  🔗 inspired by                          used 3×
✓ Completes “Complete labels: relation labels that…” · unblocks dependents · offers successors
↵ link & complete   ⇥ accept   ⌃D hide   esc back
```

One Return makes one write to `bob.md`: the LINKS line, `[x]`, and
`  [completion:: 2026-10-29]`. Ctrl+Z undoes all three. The task-status-cycler's normal
post-Done pass then runs, exactly as Ctrl+Enter's: it recovers the task's Blocked
dependents and opens the successor picker when there is something to offer. The walk
moves on once. One notice sums it up:
`🪢 Complete labels… — implemented by → Allow relation descriptions… · ✓ done`.

On the Mac, `Verify cancel labels on the Mac! ~~bob:cancel-relations|"verified by"`
shows a green `✓ completes` capsule on the relation-card row. The sentence reads
`✓ Add support for relation descriptions… — verified by → Verify cancel labels on the Mac!`.
The slot the `||` warning uses reads
`✓ Completes “Add support for relation descriptions…” · In Progress → Done`. The live
preview adds the same rows `!` would print (`✓ would complete …`, the ledger strike,
unblocked tasks), and the in-panel successor choice appears when the close has offers.

Typing the old habit, `implements`, never completes anything. Instead it shows a muted
hint: `“implements” never completes · to finish “{T}”, use ~~ with “implemented by”`. On
the feature task, the LINKED FROM chip still reads `implements ‹reading task›`, because
`implemented by` ⇄ `implements` joins the display-only inverse table.

Bryan's 14 existing `implements` links are reversed by one command, which shows the full
preview before it writes (counts below are the real vault's: 7 writable sources gain a
mirror and 5 active targets lose one):

```text
$ bob relations rename implements "implemented by" --reverse
🪢 Rename “implements” → “implemented by” and reverse 14 links (7 in done/)
  bob.md:35  Add support for relation descriptions that always trigger cancelation!
     ⇄       now  Cancel labels: relation labels that cancel their source task  — implemented by →  Add support for relation…   bob.md:284
  …
  mirrors   7 LINKED FROM added · 5 removed     statuses   none change
  commit    bob relations rename: implements → implemented by (reversed) · undo: git revert <sha>
```

`bob relations` then shows `implemented by  14 links  ✓ completes source  ⇄ implements`.
Labels outside the config, hand edits, reconcile, renames (reversed or not), unlinks,
and sync never touch a status.

## Context

- **Prior art (shipped today, epic `bob-cli-dl`):** cancel labels.
  - The contract is `docs/task-relations.md` §17 (CL1–CL8 plus a vector index). Read it
    in full before any phase. This epic mirrors its shape on purpose, so the two
    features read as one design.
  - The cancel plan is `plan:202610/relation_cancel_labels.md`. Read it with
    `sase artifact read` for the reasoning behind each shared rule.
- **Words.** The request's "relation descriptions" are **relation labels** (RV1 bans
  "description"). The configured ones are **complete labels**. Cancel labels and
  complete labels together are **closing labels**. "Done" is the status; "complete" is
  the verb (`!`, `task_complete`).
- **Vault facts (read-only, verified 2026-10-28):**
  - There are 14 `implements` LINKS entries:
    - `bob.md` 5, `sase.md` 2, `done/bob_done.md` 6, `done/sase_done.md` 1;
    - 7 edges have both ends in `done/`, and 9 targets live in `done/`;
    - every source S already has a `^block-id`;
    - every target T is a Done `#ref` reading task. The pattern is "feature task
      `implements` research reading task".
  - Active targets carry generated mirrors such as
    `🪢 **LINKED FROM:** [[#^cancel-relations]] (implements)`.
  - There is one `verified by` edge: `sase.md` `^usage-tracking` `[x]` → `sase_pre_blog`
    `^verify-footprint` `[?]`. It needs no migration.
  - There are no `implemented by` or `verifies` LINKS entries yet, and no relation-label
    journal events for `implements`.
  - Prose such as "See the `sase-1oj` epic (implements this)!" is not a relation and
    stays.
  - The SASE artifact-relation slug `implements` (`sase/artifact_relations.json`) is an
    unrelated vocabulary and stays.
- **bob-cli code (verified):**
  - **Config:** `src/native/config/relations.rs` has
    `RelationsConfig {linked_from_lines, cancel_labels, warnings}`, with
    `parse_cancel_labels` and `is_cancel_label` / `cancel_label_for_inverse`.
  - **Inverse table:** `task_relations::label::inverse_phrase` has 5 display-only pairs,
    pinned by `tests/fixtures/relation_labels/vectors.json` catalog cases.
  - **Cancel composition:** `src/native/capture/relations_cancel.rs`
    (`plan_relation_cancels`) collects `Added`/`Relabeled` edges, refuses, dedupes by
    source, and stages through `CaptureBatchPlanner`.
  - **The Done engine:**
    - `src/native/capture/task_complete.rs` `plan_task_complete_item` (lines 13–614)
      resolves the task, validates status and recurrence, then calls `task_complete`'s
      `complete_task_tree`, `retire_completed_links`, and `plan_successors`, and stages
      every file.
    - `task_complete::is_completable_status` is `[ ]`, `[?]`, `[*]`, `[/]`.
    - `is_recurring_task_line` refuses `[repeat::`, `(repeat::`, and `🔁`.
    - Offers, `-p/--pull-in`, and hidden `--require-pull-in-choice` are
      `docs/task-dependencies.md` §12 / §12.14.
    - The `!` JSON object is documented in `docs/capture.md` "Completing tasks with
      '!'".
  - **Rename:** `src/native/relation_labels/rename.rs` (1227 lines, `run_rename`)
    rewrites label spans only (RV8, LR7).
  - **Relation edit planners:** `task_relations/plan.rs` `plan_links_edit`
    (add/remove/relabel) and `plan_linked_from_line`.
- **bob-plugins code (verified at `62ccc99`):**
  - **Nav has no Done writer.** The managed Done belongs to task-status-cycler:
    - Ctrl+Enter runs `toggleActiveCheckboxOpenDoneAndPropagate` (`160`).
    - The post-Done pass is `finalizeClosedTasks(identities, context)` (`130:52`). It
      plans successors, recovers dependents with `[unblocked::]`, opens nav's successor
      picker, retires `![[…]]` embeds, and shows the Unblocked card. Walk callers pass
      `successorNoticeTarget: "return"`.
    - Cross-note Done bytes come from the pure `rewriteTaskLineForLocalFallback`
      (`050:577`) plus `addOrReplaceCompletionField` (`050:562`), which write
      `  [completion:: YYYY-MM-DD]` before `^block`. The date comes from
      `getCompletionDateString` (`190:600`).
    - The cycler `api` (`110`, version 2) exposes only `recoverBlockedDependents` and
      `completeTaskAtCursor`.
  - **Cancel in nav:**
    - config `021` (157 lines);
    - planner `241` (323);
    - post-cancel `551` (87);
    - compose `596` (605), folded into `applyRelationEdit` (`594`, **991/1000 lines**);
    - commit gate `relationCancelCommitGate`;
    - cue in `426` (900), `428` (758), `304` (886), and `305` (227);
    - `styles.css` 1626–1651 (`--color-red`);
    - the vector copy is `plugins/bob-navigation-hotkeys/scripts/fixtures/task_cancel/`.
  - **Inverse twins:** nav `304:351`, ledger-tools `146-relation-chip-model.js:22` and
    `147-relation-label-model.js:11`.
  - The walk advances generically when a Task Card commit leaves the landed row closed
    (`reviewOutcomeResolves`, `538`).
- **Bob Mac Capture (verified at `e5f6604`):**
  - **Cancel cue:** `CaptureRelationCancelPresentation.swift` (179 lines), plus hooks in
    `CaptureModels.swift`, `CaptureRelationshipPresentation.swift`,
    `CompletionRowContent.swift` (`.relationCancelLabel`), `CaptureEditorPalette.swift`,
    `CapturePanelModel.swift`, and `CapturePanelView.swift` (`RelationLabelPromptCard`).
    Each hook is a cancel-only `Bool` or a two-case style.
  - **Done rendering:** `!` renders through `CaptureTaskCompletePresentation.swift` and
    `CaptureSuccessorPresentation.swift`, using `checkmark.circle(.fill)` in green.
  - **Offers:** the in-panel successor choice collects offers only from `taskComplete`
    and `pomodoroClose` (`makePullInPreviewSnapshot`).
  - **Fixtures:** recorded by `Scripts/generate-relation-cancel-fixtures.sh`.
- **Glyphs:** the vault's Done glyph is ✓ (`docs/reference/vault-format.md`), and
  Cancelled is ⊘.

## Design

`docs/task-relations.md` gains **§18 Complete labels** (CP1–CP8 below) and §16 gains
**RV10 Reversing a label**. `complete-contract` writes both. Until a rule's phase lands,
it carries `planned (phase <id>)`.

### CP1 The rule

> When an explicit relation gesture **adds** a complete label to an open task, or
> **relabels** one of its links **into** a complete label, the same gesture marks that
> task (the relation's **source**) Done exactly as that surface's managed Done gesture
> would. Nothing else completes because of a label, and nothing ever reopens because of
> one.

- **Explicit gestures** are the same as CL1:
  - capture `~` / `~~` (CLI and Mac);
  - the nav Links-to and Linked-from stages;
  - counted or marked batches in those stages.
- **Never** fires on: reconcile, `bob relations rename` (including `--reverse`), hand
  edits of LINKS, agent edits, unlink, relabel away, mirrors, vault sync, config
  changes, or an `Unchanged` re-add.
- **Fires once, at commit.** It is never enforced. A reopened source stays open.
- **The source closes, always.** Closing labels are phrased from the task that closes:
  - "S duplicate of T" cancels S;
  - "S implemented by T" and "S verified by T" complete S.

  This is why the vault's `implements` links are reversed rather than configured.

### CP2 Config

```yaml
# Task relations (docs/task-relations.md in bob-cli).
relations:
  # linked_from_lines: true   # write 🪢 LINKED FROM mirrors (default)
  cancel_labels:
    - duplicate of
    - superseded by
    # - merged into
  # Committing a link with one of these labels also completes (marks Done) the task
  # the label is phrased from ("S implemented by T" completes S), exactly as
  # Ctrl+Enter or capture `!` would. Matched like cancel_labels. Empty completes
  # nothing.
  complete_labels:
    - implemented by
    - verified by
```

- **Parsing is CL2's, verbatim:** key matching, fail toward fewer, skip bad entries,
  collapse duplicate keys, and never a `ConfigError`. Only the noun changes:
  `invalid relations.complete_labels in <path>: expected a list of relation labels; completing nothing`.
- **One effect per label.** A key listed in both lists closes nothing. bob drops it from
  **both** lists and warns once:
  `relations: “{label}” is in both cancel_labels and complete_labels in <path>; it closes nothing`.
  - This rule also governs cancel parsing, in Rust and JS.
  - It is the only change to shipped cancel behavior, and it is reachable only through a
    misconfiguration.
- **Hide is not disable.** The catalog overlay works as in CL2: zero-use configured rows
  are marked `configured`. Labels are host-wide YAML, never stored in the journal.

### CP3 Which task completes

| Gesture                                       | Source S (completed)                      | Target T (unchanged)                    |
| --------------------------------------------- | ----------------------------------------- | --------------------------------------- |
| Task Card `l` → `⇧↵`                          | task under the cursor                     | picked task                             |
| Task Card `⇧L` → `⇧↵`                         | **picked task**, possibly in another note | task under the cursor                   |
| Capture `@note+id ~t:id\|"implemented by"`    | routed existing task                      | picked task                             |
| Capture `New task ~~ref:id\|"implemented by"` | **picked existing task**                  | the new task                            |
| Capture `New task ~old:id\|"verified by"`     | — refused `relation_completes_new_task`   | —                                       |
| Counted `N Ctrl+Shift+L` / marked rows        | each source                               | one target (second Return if ≥2 closes) |

### CP4 What one commit writes

There is nothing new about Done. Each surface composes its **existing** managed Done
with the relation write.

- **Capture (CLI and Mac)** runs exactly what `!note:id` runs, against the same staged
  batch as the LINKS edit:
  1. tree close: `[x]` plus `  [completion:: today]` on S, and embedded `![[…]]`
     subtasks per `!`'s rules;
  2. scoped Task Link retirement in today's daily note;
  3. dependent recovery with `[unblocked::]`;
  4. > [!decision] offer_successors successor offers. They join the batch's single offer
     > set: numbered, picked with `-p/--pull-in`, and gated by
     > `--require-pull-in-choice`.

  The LINKS edit, T's mirror, and all of the above are one all-or-nothing batch. `!`'s
  other rules hold verbatim: Blocked roots complete, embedded descendants follow `!`,
  and status-group moves stay with reconcile.

- **Obsidian (nav)** runs exactly what Ctrl+Enter on the task line runs:
  1. S's note, one write: the LINKS add or relabel plus the cycler's own Done line
     rewrite. One Ctrl+Z restores both.
  2. T's LINKED FROM mirror, as the usual best-effort follow-up.
  3. After the stage closes, the cycler's `finalizeClosedTasks([S], context)`: recovery,
     embed retirement, the Unblocked notice, and

     > [!decision] offer_successors the successor picker.

     Ctrl+Enter on a task line strikes no plain Pomodoro Task Link immediately, so this
     path strikes none either; reconcile retires it as it does after Ctrl+Enter today.

- **No Work Log, no Cancel Log, no freshness stamp.** Done never writes them, so neither
  does this.

> [!decision] offer_successors = no Complete-label commits recover dependents (stamping
> `[unblocked::]`) and retire references, but make **no** successor offers, exactly like
> cancel. Capture reports recovered rows only and adds nothing to the batch offer set.
> Nav calls the cycler's `recoverBlockedDependents` plus embed retirement instead of the
> full finalize. `docs/task-dependencies.md` §12.1 lists complete-label commits under
> "never prompts". The strip reads `✓ Completes “{S}” · unblocks dependents`.

### CP5 Verdicts and edge cases

There is one verdict function per engine: Rust `task_complete::verdicts` (new) and JS
`planRelationCompleteEffects`. Shared `rule` vectors pin both.

| Source state                                                           | Verdict               | Effect                                                           |
| ---------------------------------------------------------------------- | --------------------- | ---------------------------------------------------------------- |
| Completable: `[ ]` Ready, `[*]` Next, `[/]` In Progress, `[?]` Blocked | `completes`           | link and complete (Blocked included, as `!` allows)              |
| `^prj` task, completable                                               | `completes` (project) | as `!` / Ctrl+Enter; copy adds "its N open tasks stay open"      |
| Done, Cancelled, or any other status                                   | `already_closed`      | link or relabel only; never Cancelled → Done; keep original date |
| Recurring (`is_recurring_task_line`)                                   | `recurring`           | refused before writing; complete-label rows disabled             |
| `~` source is the task this capture item creates                       | `new_task`            | refused before writing, with a `~~` hint                         |
| Item also starts, Pomodoro-links, or adds `&` / `*` to S               | `conflict`            | refused `relation_complete_conflicts_with_item_action`           |
| One commit both cancels and completes the same S                       | `conflict`            | refused `relation_close_effects_conflict`                        |

More edge cases:

- **`Unchanged` re-add:** no-op. Retries are safe.
- **Relabel between two complete labels** on a Done S: relabel only.
- **Relabel away, or unlink:** S stays Done. Nav says `stays Done · reopen with Alt+[`
  (verify the chord against the cycler keymap).
- **T is Cancelled or archived:** allowed, with the amber caveat `“{T}” is Cancelled`. T
  is never changed.
- **T is open:** normal, and no caveat. `verified by` usually points at an open
  verification task.
- **Inverse typed** (`implements`, `verifies`): no completion, plus a muted flip hint.
- **Midnight or config change between cue and commit:** re-plan at commit. Nav writes
  nothing on a changed verdict and re-renders. Capture reports what it did.
- **Old cycler** (nav finds `api.version < 3`): complete-label rows are disabled with
  `Update Task Status Cycler to complete from a link`, and nothing completes.

### CP6 The indication

Name the task, change the verb, use ✓, stay calm.

- **✓ plus words**, in the Done-status green, mark the consequence; meaning never
  depends on color.
- **Red** stays for errors; **amber** stays for caveats.
- **No strikethrough** before the write.
- **Mac:** `checkmark.circle` + green, matching the picker's Done glyph and `!`'s seal.

Canonical strings (`{S}`, `{T}` task texts; plurals follow counts):

| Where                                                   | Text                                                                                                                                                                                   |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Row meta (nav) / capsule (Mac) / `relations list` / zsh | `✓ completes` / `checkmark.circle completes` / `✓ completes source` / `N links · completes source`                                                                                     |
| Live sentence (nav, Mac)                                | `✓ {S} — {label} → {T}`, source dimmed; remote: `✓ {S} ({note}) — …`                                                                                                                   |
| Verdict `completes` (bob `relation_complete.message`)   | `✓ Completes “{S}” · {Status} → Done`                                                                                                                                                  |
| Consequence strip (nav)                                 | `✓ Completes “{S}” · unblocks dependents · offers successors`                                                                                                                          |
| `^prj` suffix                                           | `· its {N} open tasks stay open`                                                                                                                                                       |
| Verdict `already_closed` (muted)                        | `Already Done · link only` / `Stays Cancelled · link only` / `Stays {Status} · link only`                                                                                              |
| Verdict `recurring` (disabled row)                      | `“{S}” repeats — complete it in Obsidian so Tasks writes the next one`                                                                                                                 |
| Verdict `new_task` (disabled row, refusal)              | `“{S}” is new — it would be born done · to finish “{T}”, use ~~`                                                                                                                       |
| Inverse hint (muted)                                    | `“{typed}” never completes · to finish “{T}”, use ~~ with “{label}”` (`~` from `~~`; nav says `⇧L` / `l`)                                                                              |
| Return hint (nav)                                       | `↵ link & complete`                                                                                                                                                                    |
| Batch banner (nav)                                      | `✓ Completes {N} tasks — ↵ again to confirm`; mixed: `Closes {N} tasks · ✓ {a} · ⊘ {b} — ↵ again to confirm`                                                                           |
| Notice (nav)                                            | `🪢 {S} — {label} → {T} · ✓ done · unblocked {k}`, plus the cycler's returned successor text                                                                                           |
| Capture row / preview (bob-authored)                    | `✓ completed “{S}” · {Status} → Done · struck {n} Pomodoro link · unblocked “{D}”`, then `!`'s ledger and offer rows; dry run says `would complete` / `would strike` / `would unblock` |
| Structured warning `relation_completed_source`          | `✓ completed “{S}” ({label} “{T}”)`; dry run `✓ would complete …`                                                                                                                      |
| Both-effects refusal                                    | `“{S}” can't be both cancelled and completed in one commit`                                                                                                                            |
| Hand-edit hint (nav, once per burst)                    | `✓ “{label}” completes only from the Links stage or capture · this task stays open`                                                                                                    |
| Rename warning                                          | `“{label}” is a complete label in config.yml · a rename never completes anything — check relations.complete_labels`                                                                    |
| Mac submit hint                                         | `Capture & complete {N} task(s)`; mixed with cancels: `Capture & close {N} tasks`                                                                                                      |

### CP7 Contracts

Additive only. Every `schema_version` stays 1, and fields are omitted when they do not
apply, so output with no `complete_labels` stays byte-identical. Names are final.

- **`capture-complete` `relation_label`:**
  - rows gain `completes_source: true`, and the shared `configured: true` on overlay
    rows;
  - `disabled_reason` covers `recurring` / `new_task`;
  - the response gains
    `relation_complete: {source: {path, block_id?, text, status_symbol, status_name, project_open_tasks?}, verdict: "completes"|"already_closed"|"recurring"|"new_task", message, hint?}`.
    It is present only when at least one complete label is configured, and it never
    scans the vault. It sits beside `relation_cancel` when both kinds are configured.
- **`capture-parse`:** relation-label spans gain `completes_source: true`.
- **`capture` (dry run and commit), per item:**
  - `relation_completes[]: {source: {path, line, block_id, text}, target: {path, block_id, text}, labels[], action: "added"|"relabeled", previous_status, previous_status_name, project_open_tasks?, task_complete, message}`.
    `task_complete` is exactly the object `!note:id` reports for S, with `raw` set to
    the relation token. It is serialized only for sources that completed.
  - `relation_updates[]` rows gain `complete_effect: "completed"|"already_closed"`.
  - `warnings[]` gains `{code: "relation_completed_source", label, message}`. The human
    renderer prints the ✓ row once.
  - > [!decision] offer_successors Offers join top-level `pull_in` and its numbering.
  - Refusal codes: `relation_completes_new_task`, `relation_completes_recurring_task`,
    `relation_complete_conflicts_with_item_action`, `relation_close_effects_conflict`.
- **`bob relations list`:**
  - JSON rows gain `completes_source`, and the top level gains `complete_labels?` (a
    count);
  - the human header gains `· N complete`;
  - rows read `✓ completes source`.

### CP8 Architecture

- **One Done per engine, never a new one.**
  - Rust: complete-label commits call the same function `!` calls. `complete-capture`
    extracts it from `plan_task_complete_item`, so `!` output stays byte-identical.
  - JS: the cycler owns Done. Nav calls the cycler `api` v3:
    - `completeTaskLineText(lineText, {date})` returns the exact fallback bytes;
    - `finalizeClosedTasks(identities, context)` runs the post-Done pass.

    Plugins never import each other's `main.js`.

- **Policy layer shared with cancel:**
  - Rust: `RelationsConfig::close_effect(key) -> Option<CloseEffect>`, where
    `enum CloseEffect { Cancel, Complete }`, and one parser for both lists.
  - JS: one loader, `loadRelationsCloseLabels()` → `{cancel, complete, warnings}`, of
    which `loadRelationsCancelLabels()` becomes a view.
- **Shared vectors** are `tests/fixtures/relation_complete/vectors.json`. bob-cli is the
  source, and bob-plugins copies it byte-for-byte.
  - `done` is generated from the cycler's `rewriteTaskLineForLocalFallback` +
    `addOrReplaceCompletionField`, and a Rust test proves `complete_task_tree`'s root
    rewrite writes the same bytes.
  - `rule`, `key`, and `config` are hand-authored from §18.
- **Atomicity, stated honestly:**
  - Capture is one staged, preimage-checked batch.
  - Nav writes S's note in one transaction with one undo. The mirror and the post-Done
    pass are follow-ups, the same contract Ctrl+Enter and cancel labels ship with.
- **Mac decides nothing** (`mac-capture-is-a-thin-client`).

### RV10 Reversing a label (`bob relations rename OLD NEW -r/--reverse`)

The migration tool, built to be reused.

- **Meaning.** Every accepted LINKS entry `S —OLD→ T` becomes `T —NEW→ S`:
  1. Remove the entry from S's LINKS line. Delete the line when it empties, and keep the
     order of the remaining entries.
  2. Append `NEW [[canonical link to S]]` to T's LINKS line, creating the line in its §4
     slot. The link uses §5's `canonical_link`, so `done/` keeps `[[done/…#^id]]`.
  3. Rewrite the LINKED FROM mirrors of S and T to the §6 projection. Only writable
     notes get mirrors. `linked_from_lines: false` writes none.
- **Scope and transaction.** These are RV8's:
  - every note in the RV2 scope, `done/` and the previous daily note included;
  - pre-sync, the lock, the `vault_mismatch` check, `stale_note`, and verify by rescan;
  - one `relation_label.rename` journal event (unchanged v1 schema; hidden state moves);
  - one commit, `bob relations rename: OLD → NEW (reversed)`, undone by `git revert`.
- **LR7 grows one clause:** a reversed rename may move an edge's entry between the two
  ends' LINKS lines, including in `done/`. It still never changes a status, a field, or
  another byte.
- **NEW may equal OLD** under `--reverse`, which is a pure reversal of a symmetric
  label. NEW's existing key still needs `-m/--merge`.
- **Collisions:**
  - If T already links S with NEW's key, merge: drop S's entry and report `merged`.
  - If T already links S with another label, refuse `reverse_conflict` and list each
    pair.
- **Refusals added:**
  - `unresolved_target`: an OLD entry does not resolve to a `#task`.
  - `source_needs_block_id`: S has no id and lives in `done/` or the previous daily
    note. A writable S is minted silently in the same transaction, per §5.
- **Never completes or cancels**, even when NEW is a closing label. When NEW is
  configured, the preview adds a muted
  `stay open  {k} new sources keep their status — a rename never closes` row listing
  them.
- **Output:**
  - The human preview groups moves by source note, as in the Outcome block, with mint,
    mirror, merge, and status counts.
  - JSON adds `reversed: true`,
    `moves[]: {source: {path, line, block_id, text}, target: {path, line, block_id, text}, from_label, to_label, action: "moved"|"merged"}`,
    and `minted_block_ids[]`.

### Inverse pairs (display only)

§10 and every twin gain `implemented by` ⇄ `implements` and `verified by` ⇄ `verifies`:

- Rust `inverse_phrase`, with `relation_labels` vector cases;
- nav `304`;
- ledger-tools `146` and `147`.

Mac renders bob's `inverse` field.

### Rejected alternatives

| Alternative                                       | Why not                                                                                                     |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Configure `implements` to complete its **target** | Breaks "closing labels close their source"; two directions of effect would make every cue ambiguous         |
| A one-off agent script for the migration          | 14 edges across `done/` deserve a preview, a verify rescan, and one revertable commit; the tool is reusable |
| Auto-complete open sources during the migration   | Renames never close (CP1); every current target is already Done anyway                                      |
| A nav-local Done writer                           | A second JS Done engine would drift from Ctrl+Enter; the cycler owns Done                                   |
| Waiting for T to be Done before completing S      | That is a dependency (`&`); a relation is the statement that S is finished                                  |
| A generic `relations.label_effects` map           | Two parallel lists read better in YAML and keep the shipped `cancel_labels` contract untouched              |
| A label in both lists means "cancel wins"         | Any guess closes a task the wrong way; fail toward fewer closes                                             |

## Cross-cutting rules for every phase

- **Contracts first.** §18, RV10, and the vectors land before any client.
  - JSON grows additively.
  - Clients decode with `decodeIfPresent` or feature detection.
  - Read §17 and §18 in full before starting.
- **Shared vectors.** `tests/fixtures/relation_complete/vectors.json` is the single
  source.
  - Its sections are `done`, `rule`, `key`, and `config`. Rust-only cases are marked
    `rust_only`.
  - bob-plugins copies it byte-for-byte, with a README that records the bob-cli commit.
    Never hand-edit the copy.
  - The same rule applies to the re-synced `relation_labels/vectors.json`.
- **CLI and docs rules.** Before touching commands, help, JSON, or docs, read
  `cli_rules.md` and `docs.md` with `/sase_memory_read`.
  - Run `just docs-gen` when help changes, then `just docs-check`.
  - Plugins use `npm run docs:check`. The Mac repo uses `just docs-check`.
- **Release train.** Never edit `Cargo.toml` versions or `CHANGELOG.md`.
  - Use `feat(relations): …`, `feat(capture): …`, and `feat(nav): …` commits.
  - Nothing here is a contract break: the default is `[]`, JSON is additive, and
    `--reverse` is a new flag. So there is no `!`.
- **Cross-repo work.**
  - Open bob-plugins, bob-mac-capture, chezmoi, and the plans sidecar only through
    `/sase_repo`, and read each repo's `AGENTS.md`.
  - Plugin fragments stay ≤1000 lines. `594` has 9 lines of headroom, so new logic goes
    in new fragments registered in `src/fragments.json`.
  - Run `npm run build` and `npm test`, then `bob plugins sync`.
- **Mac builds** follow its `AGENTS.md`.
  - Run `just portable-check`, on athena if Swift is missing locally.
  - Make a best-effort scratch typecheck on the mac.
  - Close only on green macOS CI, and say so if the mac is unreachable.
- **Vault safety.**
  - Tests use scratch vaults (`BOB_DIR`, `BOB_DAY_FILE`, `BOB_NOW`, `BOB_CONFIG_FILE`).
  - `~/bob` is read-only in every phase except `rollout`, which changes it only through
    the reverse rename.
  - The live config changes only in `rollout`, through chezmoi.
- **Inert until rollout.** With `relations.complete_labels` absent, every phase leaves
  behavior byte-identical. Prove it with the touched suites' existing relation, cancel,
  and `!` tests, unchanged.
- **Baselines and follow-ups.**
  - Compare failures with a clean-base run.
  - Phase workers create no beads. They record `PROPOSED FOLLOW-UP:` notes on their
    phase bead.

## Phases

### complete-contract

**Repo:** bob-cli. bob-plugins is read-only here, used only to generate bytes.

1. **`docs/task-relations.md` §18 Complete labels:** CP1–CP8 normatively, the canonical
   strings, the JSON names, the refusal codes, and a CP vector index. Mark each rule
   `planned (phase <id>)`.
2. **§16 RV10:** the reverse rename, its refusals and output, and the amended LR7 (§6).
   Mark it planned for `reverse-rename`.
3. **Amend, don't delete.**
   - §1: add the terms **Complete label** and **Closing label**.
   - §2: the exception bullet names §17 and §18.
   - §8: add the row "Complete-label commit → §18".
   - §10: the two inverse pairs.
   - §13.2: carve out §18 like §17.
   - §14: relabel-away or unlink leaves S Done.
   - §15: add the vector
     `reconcile never completes an open source carrying a complete label`.
   - §17 CL2: the both-lists rule, cross-referenced from CP2.
4. **Other bob-cli docs:**
   - **`docs/reference/configuration.md`:** `relations.complete_labels` and the
     both-lists rule.
   - **`docs/capture.md`:** the `~` / `~~` overview, Inertness, and a "Complete-label
     commits" paragraph beside the cancel one.
   - **`docs/task-dependencies.md` §12.1:** add the complete-label commit to the managed
     Done trigger table.
     > [!decision] offer_successors = no Under "never prompts" instead.
   - **`docs/projects.md`:** a complete-label commit on `^prj` closes it as `!` does.
   - **`docs/reference/json-outputs.md`:** stub the CP7 fields as planned.
   - **`docs/reference/glossary.md`:** add **Complete label**.
5. **Inverse pairs in Rust.**
   - Add both pairs to `inverse_phrase`.
   - Add catalog cases for them to `tests/fixtures/relation_labels/vectors.json`, plus
     the case that `implements` / `verifies` resolve case-insensitively.
   - Run the relation-label tests.
6. **Vectors** `tests/fixtures/relation_complete/vectors.json` plus a `README.md`.
   - **`done` (generated):** source-line preimage plus date, mapped to the postimage.
     - Generate it with a throwaway Node harness that loads bob-plugins' built
       task-status-cycler `main.js` and calls `rewriteTaskLineForLocalFallback` +
       `addOrReplaceCompletionField`, the way the cancel vectors loaded nav.
     - Cover each completable status; fields present or absent (`fresh`, `created`,
       `priority`, `scheduled`, `id`, `dependsOn`, `unblocked`, `keeps`, `^block-id`);
       `#prj_is_blocked`; a `^prj` line; an existing `[completion::]`; and CRLF.
     - Record already Done and Cancelled as unchanged, and recurring as refused.
   - **`rule` (hand-authored):**
     - inputs: configured cancel and complete labels, label, previous label, action,
       direction, source state, whether the source is new, and item modifiers;
     - outputs: verdict, effect (`complete`/`cancel`/`none`), and refusal code.
     - Cover every CP5 row, every CP1 never path, the CP3 table, and both-effects
       conflicts.
   - **`key`:** matching cases, including that inverses and `implement` never match.
   - **`config`:** YAML mapped to the effective cancel and complete keys and the warning
     count. Cover missing, `null`, `[]`, a scalar, a mapping, a non-string entry, an
     illegal label, duplicates, a key in both lists (dropped from both), and
     `cancel_labels` alone (unchanged).
   - **README:** provenance, regeneration, and the rule that hand-edited sections change
     only with §18.
   - **Hand review:** check every generated postimage against `!`'s documented bytes. If
     cycler and Rust bytes disagree (spacing, field order, tag handling), stop and
     record it on the bead rather than encoding a surprise.
7. **Acceptance:**
   - `just docs-check` is green;
   - a tiny Rust test asserts the vector sections exist;
   - a Rust test asserts that `complete_task_tree`'s root rewrite equals every
     non-refused `done` case.

### complete-capture

**Repo:** bob-cli.

1. **Config** (`src/native/config/relations.rs`):
   - Add `complete_labels` and generalize the parser so both lists share it.
   - Apply the both-lists drop.
   - Add `CloseEffect`, `close_effect(key)`, and `close_label_for_inverse(key)`.
   - Keep `is_cancel_label` behavior.
   - Pass the `config` vectors.
2. **Verdict:**
   `task_complete::verdicts::complete_verdict(source_line, settings, is_new_task)`
   returns `Completes{previous_status, project}`, `AlreadyClosed{status}`, `Recurring`,
   or `NewTask`. Pass the `rule` vectors' verdict part.
3. **Extract, don't fork.**
   - Move `plan_task_complete_item`'s body after token resolution into
     `complete_existing_task(...)`: status validation, recurrence, tree close,
     retirement, successors, staging, and the summary JSON.
   - `!` and complete-label commits both call it.
   - Every existing `!`, `x!`, and successor test stays green unchanged.
4. **Compose** (new `src/native/capture/relations_complete.rs`, beside
   `relations_cancel.rs`):
   - Collect `Added`/`Relabeled` edges with complete keys.
   - Refuse (coded usage errors, nothing written): `new_task`, `recurring`, conflicts
     with the item's `=` / `:` / `@route:id` / `&` / `*` on S, and a source that one
     item or batch would both cancel and complete.
   - Dedupe by source.
   - Run `complete_existing_task` on the staged snapshot **after** the relation writes,
     so S's note is one staged file.
   - `already_closed` links only and sets `complete_effect`.
   - Factor the shared collection and conflict code with `relations_cancel.rs`, not copy
     it.
5. **Offers** (> [!decision] offer_successors): the item's close joins the batch offer
   set exactly as a `!` item's would, so numbering, `-p`, and `--require-pull-in-choice`
   need no new rules. Test a mixed draft with a `!` item and a complete-label item.
6. **Cross-item:** a later item sees the completion through staged overlays. Linking,
   starting, or completing it again is refused, or no-ops, exactly as after a `!`.
7. **Output:**
   - per-item `relation_completes[]` embedding `task_complete`;
   - `complete_effect`;
   - the `relation_completed_source` warning;
   - the human `✓ completed …` row followed by `!`'s own renderer for ledger, unblocked,
     and offer rows;
   - `would` phrasing on dry runs.
8. **Docs:** `docs/capture.md` and `docs/reference/json-outputs.md`. Drop the matching
   §18 markers.
9. **Tests** (LX style, scratch vault):
   - `~` from an existing owner;
   - `~~` finishing a `#ref` reading task from a new feature task;
   - each refusal, including both-effects;
   - already Done, and Cancelled;
   - `Unchanged` re-run no-op;
   - relabel into a complete label;
   - two complete edges on one source;
   - S in today's daily note;
   - today's note absent;
   - `linked_from_lines: false`;
   - hidden and `||` complete labels still complete;
   - a Blocked source;
   - a `^prj` source;
   - embedded subtasks;
   - dependents recovered and offered;
   - New York midnight;
   - dry run equals commit;
   - the reconcile pin;
   - with no config, output is byte-identical.

### complete-surfaces

**Repo:** bob-cli.

1. **`capture-complete` `relation_label`:**
   - `completes_source` on rows, and the shared overlay that respects hidden;
   - `relation_complete {source, verdict, message, hint?}`;
   - `disabled_reason` for `recurring` / `new_task`;
   - the inverse hint for `implements` / `verifies`, phrased per direction.

   Resolve the source by path and block id, never by vault scan. Stay inside the
   relation-label latency budget and record the numbers.

2. **`capture-parse`:** `completes_source` on relation-label spans.
3. **`bob relations list`:**
   - human `✓ completes source` and `· N complete`;
   - JSON `completes_source`, `configured`, and `complete_labels`;
   - `-a` overlay;
   - zsh `N links · completes source`.
4. **`bob relations rename`:** the CP6 warning when OLD or NEW is a complete label. A
   test proves rename never changes a status.
5. **Config warnings** (including both-lists) print once per command, and appear in JSON
   `warnings[]` where present.
6. **Docs:** §16 RV7, §18, `json-outputs.md`, and the CLI reference through
   `just docs-gen`. Drop the markers.
7. **Tests:**
   - every verdict through `capture-complete` for `~` / `~~` with new and existing
     owners;
   - both kinds configured at once (`relation_cancel` beside `relation_complete`);
   - overlay and hidden;
   - the hints;
   - spans;
   - list output, human and JSON;
   - zsh;
   - the rename warning;
   - with no config, every output is byte-identical.

### reverse-rename

**Repo:** bob-cli.

1. **CLI:** `-r/--reverse` on `bob relations rename`, alphabetical after `-o/--offline`.
   - Help explains the reversal in one line with an example.
   - Allow NEW == OLD only with `--reverse`.
   - Follow `cli_rules.md`.
2. **Planner:** a new module (e.g. `src/native/relation_labels/reverse.rs`). Keep
   `rename.rs` from growing past readability. Build on `plan_links_edit` (remove on S,
   add on T) and the §6 projection for mirrors. Plan every move before writing anything.
   Cover:
   - same-note and cross-note edges, and `done/` ends;
   - a LINKS line that empties, and multi-entry lines (the `sase.md:395` shape);
   - T that already has a LINKS line, or a LINKED FROM line;
   - merge, `reverse_conflict`, `unresolved_target`, minting, and
     `source_needs_block_id`.
3. **Transaction:** reuse RV8's sync, lock, stale, verify, journal, and commit
   machinery. The commit subject reads `… (reversed)`. A verify mismatch restores every
   before-image.
4. **Output:**
   - the human preview (Outcome block), mint-relation accent on a TTY, readable with
     `NO_COLOR`;
   - JSON `reversed`, `moves[]`, and `minted_block_ids[]`;
   - the `stay open` row when NEW is a closing label.
5. **Docs:** RV10, LR7, `docs/task-status-hooks.md` (the `done/` exception paragraph),
   and the CLI reference via `just docs-gen`. Drop the markers.
6. **Tests:**
   - a scratch-vault copy of the real shape: 5 active, 2 active-to-`done/`, and 7
     all-`done/` edges, one multi-entry line, and mirrors on active ends;
   - preview equals apply;
   - `git revert` restores bytes exactly;
   - a second run refuses `unknown_label`;
   - reconcile is a no-op afterwards;
   - every refusal;
   - `linked_from_lines: false`;
   - CRLF;
   - offline mode;
   - NEW == OLD;
   - statuses are byte-identical even when NEW is a configured complete label.

### cycler-done-api

**Repo:** bob-plugins (`plugins/task-status-cycler`).

1. **API v3** in `110`, keeping v2 members unchanged:
   - **`completeTaskLineText(lineText, {date})`** is pure. It returns `{ok, lineText}`,
     or `{ok: false, reason: "recurring"|"not_completable"}`, using the exact fallback
     bytes.
   - **`finalizeClosedTasks(identities, context)`** wraps `finalizeClosedTasks`
     (`130:52`).
     - It accepts `{activePath, editor?, successorNoticeTarget?}`.
     - It returns `{successorNotice, recovered, offered, picked}`.
     - It queues on `enqueueTaskReferenceMutation` like every other caller.
     - It never opens a picker while another picker is active. Document this in the
       header.
2. **One path.** Ctrl+Enter's fallback, cross-note paths, and walk paths call the same
   functions. Existing cycler tests stay green unchanged.
3. **Vectors.** Copy `tests/fixtures/relation_complete/vectors.json` byte-for-byte, with
   a README naming the bob-cli commit. Add a test script, wired into `npm test`, that
   runs `completeTaskLineText` over `done`.
4. **Docs:** plugin `docs/task-status-cycler.md`, API v3. Then `npm run build`,
   `npm test`, and `bob plugins sync`. This phase is inert, because nothing calls the
   new API yet.

### nav-complete-writer

**Repo:** bob-plugins (`plugins/bob-navigation-hotkeys`).

1. **Loader:** `loadRelationsCloseLabels()` in a new fragment.
   - It applies the mtime/size cache, fail toward fewer, warn once, and the both-lists
     drop.
   - `loadRelationsCancelLabels()` becomes a view over it.
   - Pass the `config` and `key` vectors from a copy of
     `relation_complete/vectors.json`. Its `done` section is skipped here, because the
     cycler owns it.
2. **Planner:** `planRelationCompleteEffects(...)` (new fragment) is pure and returns,
   per source, the verdict, `completes`, and refusals. It refuses `cycler_api` when the
   cycler is below v3. Pass the `rule` vectors. Share edge derivation with `241` rather
   than copying it.
3. **Writer** (new compose fragment, beside `596`):
   - After the planner reports an `added`/`relabeled` complete edge on a `completes`
     source, fold `cycler.api.completeTaskLineText` into that source's staged content
     **before** `stageWrite`, giving one write and one undo.
   - Generalize the commit gate to closing effects: any change since the cue means write
     nothing and return `{stale: true}`.
   - A source that would be both cancelled and completed refuses before writing.
4. **Post-Done:** after the stage closes, call
   `cycler.api.finalizeClosedTasks(identities, {activePath, successorNoticeTarget})`
   once per gesture, with all completed identities.
   - On a walk landing, pass `successorNoticeTarget: "return"`.
   - Settle the walk **once, after** the post-Done pass (and any successor picker)
     resolves, so the walk advances exactly like a Ctrl+Enter landing answer
     (`decisions:answering-advances-the-walk`).
   - A remote S (`⇧L`) leaves the walk where it is.
   - > [!decision] offer_successors = no Call `recoverBlockedDependents` plus embed
     > retirement instead.
5. **One composed notice** (CP6), with the existing failure chips. A mixed gesture's
   notice lists ✓ and ⊘ in authored order.
6. **Header comments:** amend `594`'s header (LG13): the writer changes statuses only
   through §17 and §18.
7. **Tests:**
   - same-note and cross-note sources;
   - S is today's daily note;
   - `⇧L` with a remote S, open and closed;
   - `Unchanged` no-op;
   - relabel into a complete label;
   - already closed;
   - recurring;
   - Blocked;
   - `^prj`;
   - one undo restores LINKS and status;
   - stale re-plan writes nothing;
   - successor picker sequencing;
   - walk advance vs. stay;
   - a mixed cancel-and-complete batch;
   - both-effects refusal;
   - old cycler disables;
   - hand edits never complete;
   - with no config, writes are byte-identical.

   Then `npm run build`, `npm test`, `bob plugins sync`.

### nav-complete-cue

**Repo:** bob-plugins (bob-navigation-hotkeys, bob-ledger-tools).

1. **Rows:**
   - `✓ completes` meta on configured rows, typed rows included;
   - the shared configured overlay;
   - disabled rows for `recurring` and `cycler_api`.
2. **Live sentence:** a ✓ prefix and a dimmed source, with the note stem when S is
   remote.
3. **Consequence strip (`305`):**
   - the CP6 text, with the `^prj` suffix;
   - the muted `already_closed` line;
   - the amber `“{T}” is Cancelled` caveat.
4. **Footer** `↵ link & complete`. A mixed selection reads `↵ link & close`.
5. **Batch.** Generalize the two-press confirm to closing effects: arm at 2 or more
   closes, use the CP6 banners, and disarm on Esc or any change.
6. **Hints:**
   - the inverse flip hint;
   - `stays Done · reopen with Alt+[` on relabel-away or unlink;
   - the once-per-burst hand-edit hint.
7. **Inverse pairs:**
   - add both pairs to nav `304`, ledger-tools `146`, and ledger-tools `147`;
   - re-sync `relation_labels/vectors.json` byte-for-byte;
   - the LINKED FROM chip on a feature task reads `implements ‹reading task›`.
8. **Styles:** `.bob-cnp-relation-complete-*` tokens in `var(--color-green)`, mirroring
   the cancel tokens. No strikethrough. Check light and dark.
9. **Docs:**
   - `docs/task-card.md`: the heading becomes "Links stage and closing labels";
   - `docs/task-relations.md`: amend "The one status exception" into the two exceptions,
     and link bob-cli §18;
   - `docs/bob-navigation-hotkeys.md` relation-label stage.

   Then run `npm run docs:check`.

10. **Tests:**
    - copy per verdict;
    - footer swaps;
    - mixed banner arm, disarm, and Esc;
    - overlay and hidden;
    - each hint;
    - chips for the new inverses;
    - with no config, the copy is unchanged.

    Then `npm run build`, `npm test`, `bob plugins sync`.

### mac-complete-cue

**Repo:** bob-mac-capture.

1. **Generalize first.** Replace the cancel-only hooks with one `RelationSourceEffect`
   (`cancel`, `complete`), which carries the SF Symbol, tint, verb, and accessibility
   phrase:
   - `usesCancelledTint`;
   - `CaptureRelationCancelSlot.Style.cancels`;
   - `CapturePreviewWarningRow.Style.cancelled`;
   - `.relationCancelLabel`;
   - the cancel `submitHint`.

   The cancel UI stays pixel- and string-identical, and `CaptureRelationCancelTests`
   passes unchanged.

2. **Decode** with `decodeIfPresent`:
   - `completes_source` (rows and spans);
   - `relation_complete`;
   - `relation_completes[]`, whose `task_complete` decodes with the existing
     `CaptureTaskComplete`;
   - `complete_effect`;
   - the `relation_completed_source` code.

   Swift computes no verdict.

3. **Card:**
   - a `checkmark.circle` green `completes` capsule;
   - a ✓ dimmed source;
   - bob's message in the slot;
   - the muted `already_closed`, hint, and disabled rows;
   - VoiceOver: "Completes {S}".
4. **Draft:** a green palette role for `completes_source` spans, distinct from the `!`
   sigil's role.
5. **Preview and success:**
   - per `relation_completes[]`, bob's message plus the existing `!` rows (transition,
     ledger, unblocked) through `CaptureTaskCompletePresentation`, under the 🪢 row;
   - no duplicate warning row;
   - submit hints `Capture & complete N task(s)` / `Capture & close N tasks`.
6. **Offers** (> [!decision] offer_successors): `makePullInPreviewSnapshot` also
   collects `relationCompletes[].taskComplete.unblocked`, so the in-panel successor
   choice and `--pull-in` work exactly as for `!`.
   - Test that a complete-label draft with offers is never stuck behind
     `pull_in_choice_required`.
   - Record what the **current** shipped app does in that case, for the rollout's
     ordering note.
7. **Fixtures and tests:**
   - Add a generator script beside the cancel one, recording `capture-complete`,
     `capture-parse`, `--dry-run`, and a commit (with offers) from real bob.
   - Test decoding old bob and new bob.
   - Test the capsule, sentence, and slot per verdict.
   - Test the span tint.
   - Test the preview rows.
   - Test the offers wiring.
   - Test the mixed submit hint.
8. **Docs:**
   - `docs/relationship-pickers.md`;
   - `docs/capture-previews.md`;
   - a `docs/protocol.md` compatibility row (older bob: no cue; upgrade bob).

   Then run `just docs-check`.

9. **Build** per `AGENTS.md`, with green macOS CI. Attach light and dark screenshots if
   the mac is reachable.

### rollout

**Repos:** the Bob vault (through bob only), chezmoi, and bob-cli memory and docs.

1. **Build the migration binary.** Build bob from bob-cli `master` at or after
   `reverse-rename`, for example with `cargo build --release`, and use that binary
   explicitly. The installed bob may predate the release train. Confirm that `--help`
   lists `-r/--reverse`.
2. **Migrate.**
   - Run `rename implements "implemented by" --reverse --dry-run` against `~/bob`.
   - Check the preview against the Context's 14 edges: 9 `done/` targets, 7 all-`done/`
     edges, the `sase.md:395` multi-entry line, and no status changes. Paste it into the
     phase note.
   - Then run it online. Record the commit SHA and the undo command.
   - Verify:
     - `bob relations list -a` shows `implemented by 14 links` and no `implements` row;
     - a reconcile dry run (`bob task-status-hooks` / `bob task reconcile` preview)
       plans no relation edits;
     - the `verified by` edge is untouched.
3. **Chezmoi.** Add the CP2 `complete_labels` block to `home/dot_config/bob/config.yml`,
   commit, and run `chezmoi update -a --force` as chezmoi's `AGENTS.md` requires. Check
   `bob relations list -a -f json` with the built binary, and with the installed one. An
   older bob ignores the key, so it closes fewer tasks, never more. Report it.
4. **Memory.** Make the accepted memory edits through `/sase_memory_write`, written from
   what shipped, then run `sase memory init`.

> [!decision] memory_glossary_relation Rewrite the last sentence of
> `glossary:task-relation`. A relation never blocks a task, counts as a Pomodoro Task
> Link, or changes its schedule, priority, or freshness. The two status exceptions are
> both commit-time and both close the source: committing a configured cancel label
> cancels it, and committing a configured complete label (such as `implemented by`)
> marks it Done, through capture or the Links stages.

> [!decision] memory_decision_record Add
> `decisions/closing-labels-close-their-source.md`.
>
> - **Applies to:** capture `~` / `~~`, the nav Links stages, and
>   `relations.cancel_labels` / `relations.complete_labels`.
> - **Claim:** CP1. Closing labels are phrased from the task that closes, fire once at
>   commit, are never enforced, and never reopen. A label carries at most one effect.
> - **Rejected alternatives:** this plan's tables plus the cancel plan's.
> - **Cost:** cross-note undo covers S only, and a false unblock is possible.
> - **What would reopen it:** target-side effects, or a third effect.
> - **Evidence:** both plans, the landed commits, and the migration commit.

5. **Docs sweep:**
   - drop every remaining `planned (phase …)` marker;
   - check cross-links between bob-cli §17 and §18, RV10, `docs/capture.md`, the plugin
     docs, and the Mac docs;
   - `just docs-check`.
6. **End-to-end acceptance** (below), in a scratch vault, recorded in the phase note.
7. **Report** what Bryan still has to do:
   - install the new Bob Mac Capture build **before** running `chezmoi update` on the
     MacBook (see Risks);
   - run `chezmoi update` on the other hosts;
   - say whether `bob plugins sync` ran cleanly;
   - give the migration's `git revert` command.

## End-to-end acceptance

1. **Capture, reverse direction.** `Build complete labels ~~bob:ref-x|"implemented by"`
   completes the reading task with `!`'s bytes, retires its Pomodoro link, and recovers
   a dependent. `--dry-run` reports the same plan with `would` phrasing. Re-running is a
   no-op.
2. **Capture, forward direction.**
   `@sase+usage-tracking ~sase_pre_blog:verify|"verified by"` completes the routed
   owner.
3. **Offers.** A source with two Blocked dependents reports two offers. `-p 1` links
   one, and Mac fixtures show the in-panel choice.
4. **Refusals.** `new_task`, recurring, a conflict, and a both-effects draft each refuse
   with their code, and nothing is written.
5. **Never paths.** A hand edit, reconcile, `bob relations rename` (plain and
   `--reverse`), unlink, and relabel away leave every status byte-identical.
6. **Agreement.**
   - Rust and JS pass every shared vector.
   - `bob relations list -f json`, `capture-complete`, and the nav stage agree on which
     labels complete.
   - The cycler's bytes equal `!`'s bytes.
7. **Migration.** The live reverse rename matches its preview, reconcile is a no-op
   afterwards, and `git revert` of the commit (tested on a scratch clone) restores the
   prior bytes.
8. **Inert without config.** With `relations.complete_labels` absent, every changed
   command's output and every nav write is byte-identical to before.
9. **Indication.** The nav copy tests and Mac fixtures show the CP6 strings for every
   verdict, including mixed cancel-and-complete batches.

## Risks

- **Semantic creep.** There are now two carve-outs. Both are commit-time, both close the
  source, and one label carries at most one effect. §2 still lists every "never", and a
  reconcile vector pins each boundary.
- **Engine drift.**
  - Done is never re-implemented. Capture shares `!`'s function and nav calls the
    cycler's.
  - The `done` vectors pin the cycler against Rust.
- **Old Mac app, new bob, and offers.** With complete labels configured, an old app's
  submit can hit `pull_in_choice_required` for offers it cannot show.
  - Mitigations: rollout order (install the new app before the MacBook's
    `chezmoi update`); the `mac-complete-cue` record of the old app's behavior; and the
    rarity of dependents on reading tasks.
- **Live migration.**
  - It is previewed, verified by rescan, one commit, and revertable.
  - It never changes a status.
  - It refuses rather than under-covering.
- **False completion** ("S implemented by T" while T is unfinished). This is deliberate:
  the relation is the statement that S is finished. A dependency (`&`) is the tool for
  waiting, and the copy names S before every commit.
- **Fragment budgets.** `594` is at 991 lines. All new nav logic goes in new fragments.

## Out of scope

- Target-side effects; a generic label-effects map; reconcile enforcement;
  reopen-on-unlink.
- An Obsidian or Mac UI for `--reverse`. Bob's rename UIs keep plain rename.
- Extra configured labels beyond `implemented by` and `verified by`.
- Changing `!`, Ctrl+Enter, or Task Card Cancel behavior.
- Re-wording the shipped cancel rename warning.
