---
tier: epic
title: Starts-after relations offer the next task when one is parked or done
goal: 'A task linked "S starts after T" is offered as a successor (never auto-linked,
  never blocked) when T is parked (left Pending with no live Task Link today) or done,
  on every surface, and every surface says plainly which relation and which event
  fired.

  '
decisions:
  memory_glossary:
    ask: Update the Task Relation, Task Successor, and Successor Offer glossary terms
      for starts-after?
    memory:
    - glossary:task-relation
    - glossary:task-successor
    - glossary:successor-offer
    corrects: A directed, optionally labelled, inert link from one task to another.
    default: true
    answer: true
  memory_decision_record:
    ask: Record this design as a new decision record, decisions:successor-labels-offer-when-parked-or-done?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: contract
  title: Contract, shared vectors, inverse pair, and glossary
  depends_on: []
  size: medium
  description: 'contract: write the shared design into docs/task-relations.md §20
    and docs/task-dependencies.md §12.18–§12.19, author tests/fixtures/relation_successor/vectors.json,
    add the starts after → starts before display inverse (Rust plus vectors), and
    apply the accepted memory decisions.

    '
- id: core
  title: Rust policy and relation-successor engine
  depends_on:
  - contract
  size: large
  description: 'core: add relations.successor_labels and the LabelEffect policy, build
    the pure relation-successor planner and copy builder, and integrate relation offers
    into plan_successors (dedup, order, keys, placement), all pinned by the shared
    vectors.

    '
- id: authoring
  title: Rust authoring indications and relations list
  depends_on:
  - core
  size: medium
  description: 'authoring: flag successor labels in capture-complete, capture-parse,
    capture commits, bob relations list, and rename warnings, with the offers strip,
    the already-released caveat, and the starts-before inverse hint.

    '
- id: capture
  title: Capture events, snapshot, reporting, and bob-authored group copy
  depends_on:
  - core
  size: large
  description: 'capture: raise parked events from x* closes and done events from every
    Rust managed Done, build the batch-scoped relation-sources snapshot, report relation
    offers in JSON and human output, add bob-authored copy to every pull_in group,
    extend the net pass, and measure the budget.

    '
- id: capture_unlink
  title: Capture unlink hand-offs (v1.1)
  depends_on:
  - capture
  size: medium
  description: 'capture_unlink: raise a parked event when capture''s @route+id! toggle
    unlinks a Pending task, offering its starts-after sources at the link''s former
    slot.

    '
- id: nav
  title: Obsidian picker v3 and notice v3 presentation
  depends_on:
  - contract
  size: medium
  description: 'nav: teach bob-navigation-hotkeys'' successor picker and Unblocked
    notice the relation group, its caption, tag, fold line, glyph, and title rules,
    behind api.successors and api.notice version 3.

    '
- id: nav_authoring
  title: Obsidian inverse pair and relation-stage indications
  depends_on:
  - contract
  size: medium
  description: 'nav_authoring: add the starts after → starts before pair to both JS
    inverse twins and show the offers row meta, consequence strip, caveat, and inverse
    hint in nav''s Links and Linked-from stages.

    '
- id: cycler
  title: Obsidian cycler relation-successor engine (Done path)
  depends_on:
  - contract
  - nav
  size: large
  description: 'cycler: port the relation-successor rule to task-status-cycler over
    api.relationIndex, wire it into every Obsidian managed Done gesture, and present
    it through nav picker v3 and notice v3.

    '
- id: obsidian_unlink
  title: Obsidian unlink hand-offs (v1.1)
  depends_on:
  - cycler
  size: medium
  description: 'obsidian_unlink: expose a cycler API for parked events and call it
    from block-id-prompt once the Unlink task prompt settles on a Pending task with
    no live link left.

    '
- id: mac
  title: Bob Mac Capture generic group copy and relation presentation
  depends_on:
  - authoring
  - capture
  - capture_unlink
  size: large
  description: 'mac: render every Pull-In group from bob''s copy, decode and present
    relation offers, held rows, block-diff badges, notifications, and the authoring
    capsule, with fixtures from a real bob.

    '
- id: enable
  title: Enable starts after in chezmoi and smoke it
  depends_on:
  - authoring
  - capture
  - nav_authoring
  - cycler
  - mac
  size: small
  description: 'enable: add relations.successor_labels [starts after] to the chezmoi
    bob config and smoke-test the feature end to end on a scratch vault.'
proposed_by: bbugyi200.athena.research.5m.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 05:57:10
status: done
bead_id: bob-cli-fs
---

- **PROMPT:** [prompts/202611/starts_after_relation.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/starts_after_relation.md)
- **BEAD:** [bob-cli-fs](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/README.md)

# Plan: Starts-after relations

## Context

Bryan wants a new relation label, **`starts after`**. Its LINKED FROM mirror should read
**`starts before`**. When a task T is marked Pending and taken off today's plan, which
`x*` does today, Bob should offer every task S that "starts after" T as a successor. It
should also offer S when T is closed, the way ordinary successors work.

The typical case: the tasks Bryan parks with `x*` are usually agent dispatches. Some
follow-up prompts should only be thought through and sent after another dispatch has
gone out. Because Bob now has several kinds of successors, every surface must make it
obvious _why_ this offer appeared.

Bryan accepted every requirement in
`research:202611/starts_after_relation/starts_after_relation.md`. This plan adopts that
research's design and adjustments A1–A11. It decides the copy, contracts, and phase
split, and it records three refinements where the research left room:

- **The event is called `parked`.** It means T is Pending with no live link left today,
  which matches the `x*` row `Parked · not carried`.
- **The why caption names only the event.** The title names the relation, and the tag
  says "not dependents".
- **Bob authors a `copy.fold` line for each group.** The Mac renders it as given.

### Verified facts that shape the work

- **Relations.** `docs/task-relations.md` is the contract. §1 and §2 say relations are
  inert, with two commit-time exceptions:
  - §17, cancel labels;
  - §18, complete labels.

  The policy lives in `src/native/config/relations.rs`:
  - `RelationsConfig { cancel_labels, complete_labels, … }`;
  - `CloseEffect { Cancel, Complete }`, `close_effect(key)`, and
    `close_label_for_inverse`;
  - one-effect overlap handling.

  `parse_effect_labels` is a parser that never raises a `ConfigError`.

- **Inverse phrases** are display-only:
  - Rust: `src/native/task_relations/inverse.rs`, the `NAMED_PAIRS` table with four
    pairs.
  - JS: two byte-identical twins, bob-ledger-tools `src/145a-relation-inverse-phrase.js`
    and bob-navigation-hotkeys `src/303a-relation-inverse-phrase.js`.
    `scripts/test-ledger-tools-relation-chips.cjs` asserts their byte equality and
    hard-codes the four pairs.
  - Vectors: `tests/fixtures/relation_labels/vectors.json`, where the `inverse` array
    uses `expected: null` for "no inverse". bob-plugins copies it to
    `plugins/bob-ledger-tools/scripts/fixtures/relation_labels/`.

  `starts after` matches neither the named pairs nor the `‹participle› by` rule, so it
  needs a named pair.

- **Successor offers.** The contract is `docs/task-dependencies.md` §12, specifically
  §12.1–§12.17. The accepted decision is `decisions:unblocked-tasks-are-offered`.
  - **Engine** (`src/native/task_complete/`):
    - `successors.rs` holds
      `OfferGroup { HandOff, Project, ProjectFinished, KeepMoving }`, `SuccessorInput`,
      `SuccessorPlan`, `plan_successors` → `plan_successors_inner`, the early gate
      return, DayView live links, anchors, `route_successor_offers`, keys, numbering,
      and selection.
    - `keep_moving.rs` is the newest group. Its pre-gate, classifier, and
      `attach_offer(…, OfferGroup::KeepMoving, …)` wiring are the template for this
      feature.
  - **Callers:**
    - `src/native/capture/pomodoro_close.rs::plan_close_successors` builds C only from
      embedded and subtask rows that moved to `x`. Parked links never reach it.
    - The `!` path and complete-label commits run through `capture/task_complete.rs`.
  - **Batch-scoped snapshot pattern:** `DependentsSnapshot` with
    `read_prefiltered_parallel` and `snapshot_complete(overlay)`, in
    `capture/dependencies.rs`.
  - **Rest of capture:**
    - Pull-in assembly is in `capture/pull_in.rs`. `PullInGroupJson` is
      `{group, project, folded}` and exists today only for project and keep-moving
      groups.
    - The net pass is `capture/keep_moving_net.rs`.
    - Output is `capture/output.rs` (`all_offer_rows`, the `keep_moving` arrays).
- **`x*` parks.** `docs/capture.md`, near "Parking records normal In Progress work":
  - A parked link reports `outcome == "parked"`.
  - It gets normal 🍅 / Work Log / `[/]` effects and is **not carried**.
  - "Parking is a close outcome, not a new task status."
  - §12.1 and KM20 list `x*` parks as never offering.
- **The `@route+id!` toggle** removes every matching link under every open Pomodoro and
  leaves the status unchanged (`docs/capture.md` "Task Link toggle"). The item result
  carries `toggle_direction` at item level.
- **The Obsidian unlink.** In block-id-prompt (`docs/block-id-prompt.md`),
  Ctrl+Shift+Enter on a linked task, or on a selected Task Link, removes today's
  open-Pomodoro links. For an In Progress target it opens the optional `Unlink task`
  Work Log prompt (`src/090-prompt-modals.js`).
- **bob-plugins** (keep-moving landed: `99333a5`, `524ffc8`, `053a82c`).
  - **Cycler entry points.** The cycler routes Done through `finalizeClosedTasks`
    (`130`) → `planAndApplySuccessorsNow` (`137`, with `quietGate`).
    - `planSuccessors` (`085`, 993 lines) calls `applyContext`, which runs
      `planSuccessorKeepMoving` (`089c`).
    - `applySuccessorVerdicts` (`089b`) holds `priorityOf` and `appendRow`.
    - `buildSuccessorPickerModel` (`085b`) authors the group header, caption, and tag.
    - `successorNoticeText` and the config loaders are in `087` (987 lines).
    - The receipt is `136`.
  - **Cycler inputs.** The cycler reads no relations config and does not use
    `api.relationIndex`.
  - **Nav picker.** `normalizeSuccessorPickerModel` (`206a:251`) allow-lists four groups
    and drops any other. The picker API is v2 (`206:594`). The notice is
    `285-unblocked-notice.js`, also API v2.
  - **Nav relation authoring.** Row meta lives in `426` (`⊘ cancels`, `✓ completes`),
    copy in `305`, and inverse hints in `428`. Config loading is
    `loadRelationsCloseLabels` (`022`) through `parseYaml`.
  - **`api.relationIndex` v2.** It lives in ledger-tools `171`, offers
    `incoming(path, blockId)` / `outgoing` / `ready()` / `generation`, and overlays
    unsaved editor text.
- **Bob Mac Capture** (keep-moving phase `bob-cli-fb.5` **not landed**):
  - `CapturePullInChoice.swift` and `CapturePullInChoiceView.swift` switch on group
    keys. An unknown group such as `relation` renders its raw key as the title, a
    `lock.open` glyph, and an "Unblocked by …" context line. Its triggers render as
    `trigger: …`. This is exactly the confusion Bryan wants gone.
  - Offers are gathered in `CapturePanelModel.makePullInPreviewSnapshot`.
  - Decoding is tolerant (`try?` per field).
  - Fixture scripts need `BOB_BIN` and a scratch vault.
- **In flight:** epic `bob-cli-fb` (keep-moving).
  - `bob-cli-fb.2` (Rust; repair tale `bob-cli-fb.2.1` pending) edits `successors.rs`,
    `keep_moving.rs`, `pomodoro_close.rs`, `task_complete.rs`, `output.rs`,
    `pull_in.rs`, and `keep_moving_net.rs`, and it settles CLI rows on lane names.
  - `bob-cli-fb.5` (Mac) edits the Pull-In files.
- **Vault.** No `starts after` edges exist yet, so nothing needs migrating. Labels that
  carry effects are the ones Bryan actually uses (`implemented by` 25, `superseded by`
  13).

## Shared design (binding for every phase)

### Vocabulary

- **Successor label.** A relation label listed in `relations.successor_labels`. bob's
  default is `[]`; chezmoi enables `starts after`. An edge `S ‹label› T` stored on S's
  LINKS line is a **standing offer**:
  - it never blocks or changes anything;
  - it is read only when a gesture _parks_ or _completes_ T;
  - its only effect is to add S to that gesture's successor offers.
- **Parked.** T is parked when an explicit hand-off gesture leaves T `[/]` Pending with
  **no live Task Link left** in today's open Pomodoros, where T had one before the
  gesture. Like "park", this describes the outcome of a gesture, never a stored status.
- **Event.** `(T, parked)` or `(T, done)`. Events are computed at the gesture and never
  stored.
- **Released.** X is released when X is Done or Cancelled, or X is `[/]` with no live
  link on the post-gesture day text. This is computed from current state.
- **Relation successor.** An offer row in the new group **`relation`**, with row trigger
  `"relation"`.
- **Live link and baseline.** These keep their §12 meanings. The baseline is today's day
  text before this gesture strikes, moves, or drops anything, and after any explicit
  link step of the same item.

### Storage and display

```markdown
- [ ] #task Prepare evaluation prompt ^evaluate
  - 🪢 **LINKS:** starts after [[agents#^implement]]

- [/] #task Send implementation prompt ^implement
  - 🪢 **LINKED FROM:** [[agents#^evaluate]] (starts after)
```

- The stored mirror stays `[[S]] (starts after)`.
- `starts before` is a **display inverse**: one new named pair,
  `starts after → starts before`, added to the Rust table, both JS twins, and the
  vectors. Chips, the nav Linked-from stage, and `bob relations list ⇄` then render
  `starts before`.
- Typing `starts before` stores an inert label (see the inverse hint).
- Hand-edited LINKS entries also arm the offer. Unlike closing labels, nothing fires at
  commit, so there is nothing to bypass.

### Config and policy

```yaml
relations:
  cancel_labels: [duplicate of, superseded by]
  complete_labels: [implemented by, verified by]
  # Linking S with one of these labels to T offers S as a successor when T is
  # parked (left Pending, off today's plan) or done. Never blocks; S is never
  # auto-linked. Matched like cancel_labels. Empty offers nothing.
  successor_labels:
    - starts after
```

- **Parsing** follows the CL2/CP2 rules exactly: `relation_label_key` matching, and
  warn-and-skip on invalid entries. A parse problem is never a `ConfigError`.
- **Policy type.** It becomes
  `RelationsConfig::label_effect(key) -> Option<LabelEffect>` with
  `enum LabelEffect { Cancel, Complete, Successor }`. `close_effect` stays a view that
  never returns `Successor`.
- **One effect per label.** A key listed in two or more lists has no effect and is
  dropped from every list.
  - The existing cancel+complete overlap warning stays byte-identical.
  - Any overlap that involves `successor_labels` warns once:
    `relations: “{label}” is in more than one of cancel_labels, complete_labels, successor_labels in <path>; it has no effect`.
- **Inverse lookup** becomes `label_for_inverse(key) -> Option<(&str, LabelEffect)>`.
- **Catalog overlay.** A configured successor label appears in suggestions and in
  `bob relations list -a` at zero uses, marked `configured`.
  `decisions:relation-label-hide-is-suggestions-only` holds: hiding is not disabling.
- **Kill switch.** Either an empty list or `plan.link_unblocked: false` turns the
  feature off. Output then carries no relation field and no scan runs (SA24). The
  bob-authored `pull_in` group copy below is independent of this switch.
  `plan.keep_moving` is unrelated.

### Events (when it fires)

| Gesture on T                                                                                                                                                                                                  | Event    | Phase                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | --------------------------------- |
| Capture close with an effective **parked** outcome for T (`x*`, `x*N`, `x1*`, `x*!M`, `=x*…`), after which T is `[/]` with no live link on the post-close day. This includes a T already `[/]`                | `parked` | capture                           |
| Every §12.1 managed Done:<br>• `!note:id`<br>• `x` with embeds, `x!M`, `x!`, `^route:id=x…`<br>• complete-label commits<br>• Ctrl+Enter (every form)<br>• Alt+] into Done<br>• the walk's PRE/POST Ctrl+Enter | `done`   | capture (Rust), cycler (Obsidian) |
| Capture `@route+id!` unlinking a `[/]` T                                                                                                                                                                      | `parked` | capture_unlink                    |
| block-id-prompt Ctrl+Shift+Enter that removes a `[/]` T's links, after the `Unlink task` prompt settles                                                                                                       | `parked` | obsidian_unlink                   |

Nothing else ever fires. That includes:

- a park while T keeps another live link today (SA2);
- cancel of any kind (a cancelled T still counts as _released_ for R6);
- an `x~` drop;
- a plain `x` carry or defer;
- Alt+N release;
- reopen;
- a lane toggle not into Done;
- a raw checkbox click;
- reconcile and hooks;
- hand or agent edits;
- vault sync;
- config changes;
- adding, relabelling, or renaming a relation. These only arm future events: authoring
  never replays history.

### Offer rule (both engines)

```text
E(g)        = {(T, done) : T ∈ C(g)}                                      (§12.2's C)
            ∪ {(T, parked) : g is a hand-off gesture above, T live-linked on BASELINE,
                             T is [/] after g, no live link to T on the post-g day}
EDGES(T)    = {(S, label) : an accepted LINKS entry on S resolves to T, S ≠ T,
               relation_label_key(label) ∈ successor_labels}
              — staged text overlays disk; LINKED FROM is never read; resolution uses
                the reconcile target rules (same-note [[#^id]], basename, path)
RELEASED(X) ⇔ X is Done or Cancelled
              ∨ (X is [/] ∧ no live link to X on the post-gesture day text)

FOR EACH S reached from some T ∈ E(g), first match wins:
  R1  S is not an open #task, lives under done/, or lives in the previous daily note
                                                              → skip, unreported
  R2  S parked by g, or S live-linked on BASELINE            → held already_planned (JSON only)
  R3  S is [/]                                                → held already_started
  R4  S is [?], has an own open prerequisite, is scheduled after today, or is
      project-blocked (read after this gesture's recoveries)  → held blocked{waits_on|scheduled|project}
      (S carries [dependsOn::] and the dependents snapshot is unavailable → held blocked; KM31 parity)
  R5  S has whole-token #hide                                 → held hidden
  R6  some other successor-label target T′ of S resolves and is not RELEASED
      (unresolved targets are ignored; every event of g counts as released)
                                                              → held waits_for[T′…]
  R7  otherwise (S is [ ] Ready or [*] Next)                  → OFFER in group relation(T, label)
```

- **Groups.** There is one `relation` group per `(T, label key)` with at least one
  offered row. A LINKS line never repeats a target, so every row in a group shares its
  label. An S reached from several Ts in one gesture appears once, under the first group
  in ORDER. That row carries `after “T1” and “T2”` (or `after “T1” and k more`).
- **DEDUP** (one row per task identity; the surviving row keeps every reason). The
  priority is `hand_off > relation > project > project_finished > keep_moving`.
  - An S that is also a direct dependent of a completed T stays in `hand_off`, carrying
    a `relation` second reason (SA14).
  - An S that a project, pull-in, or keep-moving reason would also offer moves to its
    relation group. It keeps every recovery those rules perform (SA15).
  - A project group left with no selectable rows keeps its existing empty-group
    behavior.
- **ORDER.**
  - Groups go in the DEDUP order. Within that, relation groups sort by T's anchor ledger
    position, then T's note path and line.
  - Rows within a relation group sort by lane pull order (Next → Ready), then note path,
    then line.
- **KEYS, numbering, SELECTION.** These follow §12.2 unchanged. `-p` numbers and keys
  select relation rows like any other offer.
- **Placement.** The §12.3 anchors over the triggering T decide where a picked row goes;
  D7 type routing then applies:
  - **`done`:**
    - a slot right after T's struck bullet;
    - the closing anchor for a Pomodoro close;
    - `inherit`;
    - otherwise the implicit destination.
  - **`parked` by `x*`:** the closing anchor (the continuation, else the next open
    same-name entry, else a created `- [ ] () — NAME` placeholder). S takes T's place in
    the next session.
  - **`parked` by an unlink:** a slot at the position of T's first removed link. When
    the unlink's cleanup deleted that entry, use the implicit destination.

  A placeholder is never created only to _show_ an offer.

- **PICK.** Identical to a picked `hand_off`:
  - `' '` → `'*'`, and `'*'` stays;
  - mint a missing `^block-id`;
  - stamp `[fresh:: today]`, clearing `[unblocked::]` and `[keeps::]`;
  - insert the plain Task Link.

  **Unpicked rows write nothing:** no recovery, no `[unblocked::]`, no tickler, no mint.
  S was never blocked by this edge. **Stateless:** skipping S at a park means it is
  offered again when T is done (SA16).

- **Revalidation.** The Obsidian re-plan with picks re-runs the rule on fresh text. A
  picked row whose edge vanished, whose S is no longer eligible, or whose T event no
  longer holds is refused per row (SO30 style) and never substituted. Capture plans the
  selection in its single pass.
- **Batches (SL20).** A relation added by an _earlier_ item of the same draft is visible
  through the staged overlay (SA19). In the same draft:
  - An edge added _after_ the park never fires retroactively. Its authoring item warns
    `relation_successor_already_released` (SA20).
  - A later item that links, completes, cancels, drops, starts, or re-blocks S removes
    S's row.
  - A later item that live-links a parked T again drops that T's `parked` groups (SA21).
- **Uncertainty is visible, never silent.** If the relation-sources snapshot is
  incomplete (Rust), or `api.relationIndex` is missing or not `ready()` (JS):
  - the gesture still succeeds;
  - no relation rows are produced;
  - the JSON reports `relation_check: "unavailable"`;
  - one muted line names the gap (copy table).

### Reporting (additive; every `schema_version` stays 1; no `!`)

- **New fields** on `task_complete` and `pomodoro_close` objects, nested
  `relation_completes[].task_complete`, and the capture item itself for toggles. Every
  new relation field is **omitted when it does not apply**, so a disabled or edge-free
  gesture reports no relation field at all:
  - `relation_successors[]` uses the exact `unblocked[]` row shape, with:
    - `trigger: "relation"` and `offer.group: "relation"`;
    - `offer.after: "<note_path>#^<block_id>"` (T's key, which joins rows to groups);
    - `unblocked_by` = the triggering Ts (keep-moving parity);
    - `relation: {label, key, event: "parked"|"done", after: [{note_path, block_id, text}]}`.

    Rows never appear in `unblocked[]` or `keep_moving[]`.

  - Rows of other groups gain the same `relation` object when the relation is a second
    reason (SA14).
  - `relation_held[]` rows carry
    `{note_path, block_id, line, text, status_symbol, status_name, reason: "already_planned"|"already_started"|"blocked"|"hidden"|"waits_for", waits_on?, scheduled?, project?, waits_for?: [{note_path, block_id, text}], relation}`.
  - `relation_check: "checked"|"unavailable"` is present only when the relation gate
    reached the lookup.

- **`pull_in`:**
  - `offer_count`, `picked_count`, and the `pull_in_choice_required` refusal (its flat
    `offers` too) include relation offers.
  - `groups[]` now has **one entry per offered group of every kind**. Each entry gains
    bob-authored `copy: {title, why?, tag?, fold?, glyph}`, where `glyph` is one of
    `hand_off|project|project_created|project_finished|keep_moving|relation`.
    - Existing project and keep-moving entries keep their current keys, plus `copy`.
    - New hand_off and project_finished entries carry no `folded`.
    - Relation entries carry
      `{group: "relation", after: {key, note_path, block_id, text}, label, event, folded: {hidden, blocked, already_started, waits_for}, copy}`.
    - `copy` strings are exactly the cycler picker's strings for that group (§12.6 and
      §12.15 copy for existing groups; this plan's table for relation groups). No new
      copy is invented for existing groups.
  - `pull_in.copy: {title, capsule, glyph}` follows the picker-title rule.
- **Diffs and blocks.** The Pomodoro block-diff reason is `"relation"` for a surviving
  relation pick. `TaskBlockRole` gains `RelationSuccessor` (`"relation_successor"`).
- **Dry run.** Dry-run JSON equals real-run JSON with the same flags.
- **Authoring contracts** are in the authoring phase. Obsidian picker and notice API v3
  are in the nav phase.

### Copy and visual system (identical on every surface)

‹T›, ‹S›, and ‹T′› are cleaned task text, truncated to 48 characters with `…`. ‹label›
is the configured label as written (`starts after`); ‹Label› is the same with its first
letter uppercased.

- **One glyph:**
  - 🪢 on text surfaces;
  - lucide `link-2` in Obsidian, which nav's relation stage already uses;
  - SF Symbol `link` on the Mac.
- **One accent: neutral.** Relations never carry status colour (task-relations §10).
- No 🔓, no "unblocked", no new hex values.
- Rows show lane names, never raw status symbols.
- Every relation surface answers three questions, in this order:
  1. **which relation** (the title);
  2. **what just happened** (the why caption);
  3. **what these are not** (the tag).

| Element                                     | Copy                                                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Group title                                 | `‹Label› “‹T›”`                                                                                                                                                   |
| Why caption, parked                         | `“‹T›” is parked · Pending, off today's plan`                                                                                                                     |
| Why caption, done                           | `“‹T›” is done`                                                                                                                                                   |
| Contrast tag                                | `your links · not dependents`                                                                                                                                     |
| Fold line (omit zero parts; one named wait) | `not offered · “‹S›” waits for “‹T′›” · 1 already started · 1 blocked · 1 hidden`; several waits add `+N more waiting`                                            |
| Row suffix, several triggers                | `after “‹T1›” and “‹T2›”` / `after “‹T1›” and k more`                                                                                                             |
| Second reason on another group's row        | `· ‹label› “‹T›”`                                                                                                                                                 |
| Picker / card title, only relation groups   | `‹Label› “‹T›”` / `‹Label› k tasks`; mixed pickers keep the existing title                                                                                        |
| Mac capsule, every offer relational         | `‹Label›`                                                                                                                                                         |
| Notice, one pick                            | `🪢 Next in ‹DEST›: ‹S› · ‹label› “‹T›”`                                                                                                                          |
| Notice, several picks                       | `🪢 n linked → ‹DEST› · ‹label› “‹T›”`                                                                                                                            |
| Notice, mixed with dependency rows          | the existing 🔓 forms; relation picks count as linked                                                                                                             |
| Held notice (only when no picker opened)    | `🪢 “‹S›” ‹label› “‹T›” · not offered: waits for “‹T′›”` / `already started` / `blocked (waits on n)` / `scheduled ‹Mon D›` / `blocked by its project` / `hidden` |
| Uncertainty line (muted)                    | `🪢 Couldn't check ‹label› links · nothing offered` (several labels: `relation links`)                                                                            |
| CLI group header                            | `‹Label› “‹T›” · it is parked (your 🪢 links, not dependents)` / `· it is done (…)`                                                                               |
| CLI fold row                                | `not offered  “‹S›” waits for “‹T′›” · 1 already started`                                                                                                         |
| CLI linked suffix                           | `· ‹label› “‹T›”`                                                                                                                                                 |
| Mac notification line                       | `🪢 Pulled in n · ‹label› “‹T›”`                                                                                                                                  |
| Mac block-diff badge                        | SF `link`, `.help("Offered by your relation link")`                                                                                                               |

Already-planned held rows stay silent everywhere except JSON. A notice that would only
say "nothing picked" is silence.

**`bob capture 'x*' --dry-run`** (only the last four lines are new; the close rows keep
their existing format and the lane-name style that `bob-cli-fb.2` settles):

```text
would close BUILD 1145-1220 (35m)
  Next → Pending  Send implementation prompt  agents.md ^implement · Parked · not carried
  Starts after “Send implementation prompt” · it is parked (your 🪢 links, not dependents)
  offer 1  Ready  Prepare evaluation prompt  agents.md ^evaluate → BUILD
  not offered  “Write rollout notes” waits for “Ship schema change”
  -p 1 · -p all
```

**Bob Mac Capture Pull-In card** (every word comes from bob's `copy`):

```text
 🔗 Starts after                                                       (capsule)
 🔗 Starts after “Send implementation prompt”          your links · not dependents
    “Send implementation prompt” is parked · Pending, off today's plan
    ☐ 1  Prepare evaluation prompt                         Ready   → BUILD
    not offered · “Write rollout notes” waits for “Ship schema change”
  ⇧↩ none                                                       ↩ pull in 1
```

**Obsidian picker** (nav v3; the Done path, mixed with hand-offs):

```text
┌──────────────────────────────────────────────────────────────────────┐
│ ⑂  Pull in successors of Send implementation prompt                   │
│ [ Filter tasks, notes, and groups                                  ]  │
│   Hand-offs                                                           │
│   □ Ship the schema change            sase · Ready · → BUILD          │
│ │ 🔗 Starts after “Send implementation prompt”  your links · not dependents │
│ │ “Send implementation prompt” is done                                │
│ │ □ Prepare evaluation prompt          agents · Next · → BUILD        │
│ │   not offered · 1 already started                                   │
│  Tab toggle and move · ↵ pull in · Esc none                           │
└──────────────────────────────────────────────────────────────────────┘
```

The mixed picker keeps today's title and `git-pull-request` header icon. The `│` rule is
a neutral left accent. When every group is relational, the modal title is
`Starts after “Send implementation prompt”` with the `link-2` header icon.

### Authoring indications

These are additive and never block a write.

| Where                                                                        | Copy                                                                                                      |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Label row meta (nav) / capsule (Mac) / `relations list` row / zsh            | `🪢 offers` / SF `link` `offers` / `🪢 offers source` / `N links · offers source`                         |
| `relations list` header                                                      | `· N offer`                                                                                               |
| Verdict `offers` (bob `relation_successor.message`; nav consequence strip)   | `🪢 Offers “‹S›” when “‹T›” is parked or done · never blocks`                                             |
| Verdict `already_parked` (amber caveat)                                      | `“‹T›” is already parked — “‹S›” will be offered when it is done`                                         |
| Verdict `already_closed` (amber caveat)                                      | `“‹T›” is already closed — nothing left to wait for`                                                      |
| Inverse hint (muted) on the inverse spelling of a configured successor label | `“‹typed›” never offers anything · to offer “‹X›” after this task, use ~~ with “‹label›”` (nav says `⇧L`) |
| Structured warning `relation_successor_already_released`                     | the matching caveat text                                                                                  |
| Rename warning                                                               | `“‹label›” is a successor label in config.yml — update relations.successor_labels`                        |

A brand-new task may be a source, because nothing fires at link time.

### Gates and performance

- **Rust gates, cheapest first:**
  - G0: `successor_labels` is non-empty. This is one batch-cached config read, shared
    with the close policy.
  - G1: `plan.link_unblocked`.
  - G2: E(g) is non-empty. Parked events need only the held post-close day text.
  - G3: some event T has a block ID.

  Only then is the **batch-scoped relation-sources snapshot** built:
  - one parallel byte pass over the catalog's paths outside `done/`;
  - prefiltered on `**LINKS`, then per gesture on `#^<id>` for some event T;
  - parsed with the shared LINKS parser;
  - staged files overlay disk;
  - **shared with the dependents snapshot's byte pass** when both run.

- **Rust lookups never use:**
  - the 60 s relation-label scan cache;
  - LINKED FROM;
  - any persistent index (§12.7 priced the stat walk at the same cost).
- **Rust released checks** read only the T′ notes involved, through the staged planner.
- **Targets**, measured on apollo with a release build against a temporary **copy** of
  `~/bob`:
  - `hello` and a plain `x` that parks and completes nothing stay ≤ 40 ms, with zero
    added reads;
  - an enabled `x*` that parks a linked task stays ≤ 70 ms;
  - a `!` / `x!` stays within today's ≤ 70 ms, with no more than +5 ms when the
    dependents pass also runs.

  Record the measurements in §12.7.

- **JS:** `api.relationIndex.incoming(path, blockId)` filtered by label key, then S is
  re-read from staged text. Closes whose task has no successor-label incoming row add
  zero reads (wiring test).

### Walk, undo, plan budget

- **Walk.** The picker holds the review-walk gesture lock. After it settles, Esc
  included, the walk advances once.
- **Undo.**
  - Alt+N on a picked link releases it.
  - A same-day Obsidian reopen of a done T takes back untouched relation picks through
    the receipt (KM28 parity).
  - Rust has no reopen take-back.
- **Plan budget.** It warns and never refuses.

### Interplay with accepted decisions

- **`decisions:unblocked-tasks-are-offered`** is extended:
  - one picker per gesture;
  - nothing preselected or auto-linked;
  - no cap.
- **`decisions:task-lanes-are-sticky`.** T's lane is never touched. A pick raises S to
  Next.
- **`decisions:today-is-read-from-the-ledger`.** Events and released are read from live
  links at the gesture and never stored.
- **`decisions:projects-block-from-prj-line`.** A Blocked S is held, never linked.
- **`decisions:mac-capture-is-a-thin-client`.** bob computes, writes, and authors every
  string; the Mac renders.
- **`decisions:relation-label-hide-is-suggestions-only` and
  `decisions:capture-sigils-say-where-what-how`.** There is no new sigil, field, or tag.
  Authoring is `~…|starts after`, `l` / `⇧L`, or a hand edit.
- **`decisions:bob-cli-releases-ride-a-daily-train`.**
  - No phase edits `Cargo.toml`'s version or `CHANGELOG.md`.
  - Commits are additive `feat(…)` with no `!`.

### Why this shape (rejected alternatives)

- **A start-to-start dependency type.** It makes S Blocked and strips its links. It
  stamps `[unblocked::]`, touches hooks, the walk, and project blocking, and it never
  fires on a park.
- **Planning S in a later placeholder, or using milestone tasks.** These work only when
  the order is known today, or they cost one extra task per dispatch. Milestones remain
  the advice for genuinely gated work: needs T's _result_ → `&`; needs T _out of your
  hands_ → `starts after`.
- **Auto-linking S into T's slot.** Rejected by `decisions:unblocked-tasks-are-offered`.
- **A new sigil, a `[startsAfter::]` field, or a tag.** Sigils are scarce, and a field
  or tag duplicates relations.
- **A watcher on transitions into `[/]`.** It fires mid-work, misses parks of tasks
  already Pending, and needs stored "already offered" state.
- **Overloading `follow-up of`.** That label is narrative history.
- **Storing `(starts before)`, or treating `starts before` as a trigger.** Either would
  state the reverse order and require a two-sided lookup.
- **OR semantics across targets.** "S starts after T1 and T2" reads as both.
- **Offering a Pending S.** It would re-offer work that was already dispatched.
- **A persistent edge cache or the 60 s label cache.** It costs as much as a fresh scan,
  or it is unsafe for execution.

## Conformance vectors

The contract phase copies this table into `docs/task-dependencies.md` §12.19. It
expresses every engine-reachable row as data in
`tests/fixtures/relation_successor/vectors.json` (bob-cli is the source; bob-plugins
copies it byte-for-byte).

**Base fixture.**

- S = `agents.md ^evaluate`, Ready, with
  `- 🪢 **LINKS:** starts after [[agents#^implement]]`.
- T = `agents.md ^implement`, Next, live-linked in the running `BUILD` session.
- Config: `successor_labels: [starts after]`, `link_unblocked: true`.

| #    | Situation                                                                                      | Outcome                                                                                                                                                                                                   |
| ---- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SA1  | `x*` parks T; T has no other live link                                                         | group `Starts after “Send implementation prompt”`, `event: parked`, S offered at the closing anchor. Nothing is written without `-p`. A pick gives `[*]` + `[fresh:: today]` + a link in the continuation |
| SA2  | the same, but T is also linked in a later open placeholder                                     | silence                                                                                                                                                                                                   |
| SA3  | the same, with T already `[/]` before the park                                                 | offered (the park is the event)                                                                                                                                                                           |
| SA4  | `!agents:implement`                                                                            | `event: done`; slot right after T's struck bullet                                                                                                                                                         |
| SA5  | `x` with T embedded, or `x!1` completing T                                                     | `event: done`; closing anchor                                                                                                                                                                             |
| SA6  | cancel T; `x~` drop; Alt+N release; plain `x` carry or defer; raw checkbox; reconcile          | silence                                                                                                                                                                                                   |
| SA7  | S is `[/]`                                                                                     | held `already_started`; muted held notice; no picker                                                                                                                                                      |
| SA8  | S is `[?]` / has an own open prerequisite / is scheduled tomorrow / is project-blocked         | held `blocked` with `waits_on` / `waits_on` / `scheduled` / `project`                                                                                                                                     |
| SA9  | S has `#hide`                                                                                  | held `hidden`                                                                                                                                                                                             |
| SA10 | S is already live-linked today                                                                 | held `already_planned`, JSON only                                                                                                                                                                         |
| SA11 | S also starts after T2 (Ready, unplanned)                                                      | held `waits_for [T2]`. A later `x*` parking T2 (T still parked) offers S under `Starts after “T2”`                                                                                                        |
| SA12 | one `x*` parks T and T2; S starts after both                                                   | one row, under T's group (the earlier link), suffixed `after “…” and “…”`                                                                                                                                 |
| SA13 | S's other target does not resolve                                                              | ignored; S offered                                                                                                                                                                                        |
| SA14 | S also depends on T; T done                                                                    | one row in `hand_off`, with `relation` as a second reason. Parking T instead leaves S held `blocked`                                                                                                      |
| SA15 | S is a resident of a project whose `^prj` T's Done recovered                                   | one row, in the relation group; S is still recovered                                                                                                                                                      |
| SA16 | skip at the park (Esc, `-p none`), then complete T later                                       | offered again                                                                                                                                                                                             |
| SA17 | S picked at the park, then T done later                                                        | held `already_planned` (silent)                                                                                                                                                                           |
| SA18 | `x*!2`: completes A (which has dependents) and parks T                                         | one picker: `Hand-offs`, then `Starts after “T”`                                                                                                                                                          |
| SA19 | draft: item 1 `Prepare evaluation prompt ~agents:implement\|starts after`, item 2 `x*`         | S offered (staged edge)                                                                                                                                                                                   |
| SA20 | draft: item 1 `x*`, item 2 adds the edge                                                       | no offer; warning `relation_successor_already_released`                                                                                                                                                   |
| SA21 | draft: item 1 `x*` offers S, item 2 links S; or item 2 re-links T                              | S's row dropped; or T's group dropped                                                                                                                                                                     |
| SA22 | the label is typed `Starts  After:`; `starts before` stored on T's LINKS                       | matches; `starts before` is inert, and authoring shows the inverse hint                                                                                                                                   |
| SA23 | a key in both `successor_labels` and `complete_labels`                                         | no effect from either list; one warning                                                                                                                                                                   |
| SA24 | `successor_labels: []` or `plan.link_unblocked: false`                                         | no relation field in the output; no scan                                                                                                                                                                  |
| SA25 | the edge was added in Obsidian 5 s before `x*`                                                 | found (fresh scan, no TTL)                                                                                                                                                                                |
| SA26 | neither task has `[id::]`; `relations.linked_from_lines: false`                                | still found; no dependency fields written                                                                                                                                                                 |
| SA27 | the relation-sources snapshot is incomplete (Rust) / `relationIndex` is not ready (JS)         | no rows; `relation_check: "unavailable"`; muted line; the gesture succeeds                                                                                                                                |
| SA28 | S is closed, lives under `done/`, or lives in the previous daily note                          | skipped, unreported                                                                                                                                                                                       |
| SA29 | a picked S lost its edge, or became Blocked or linked before apply (Obsidian re-plan)          | that row is refused; nothing substituted                                                                                                                                                                  |
| SA30 | authoring an edge to a T that is already parked / already closed                               | caveat `already parked — … offered when it is done` / `already closed — nothing left to wait for`; no offer at commit                                                                                     |
| SA31 | `@agents+implement!` unlinks a `[/]` T                                                         | `event: parked`; S offered at T's former slot                                                                                                                                                             |
| SA32 | `@agents+implement!` unlinks a `[*]` T                                                         | silence (T is not Pending)                                                                                                                                                                                |
| SA33 | Obsidian Ctrl+Shift+Enter on a `[/]` T's link, after the `Unlink task` prompt settles          | relation picker, `event: parked`. Esc on that prompt (nothing unlinked) gives silence                                                                                                                     |
| SA34 | nav `api.successors.version < 3`                                                               | the cycler plans no relation group; the close succeeds                                                                                                                                                    |
| SA35 | same-day Obsidian reopen of T after picking S                                                  | S's untouched link is taken back                                                                                                                                                                          |
| SA36 | a plain `x` that parks and completes nothing; a close of a T with no successor-label edge (JS) | zero added reads                                                                                                                                                                                          |

**`tests/fixtures/relation_successor/vectors.json` schema** (sections; every vector has
`name` and its SA/SU id):

- **`config`.** YAML in, then the effective cancel / complete / successor keys and the
  warning count. Cases:
  - missing, `null`, `[]`, scalar, mapping;
  - non-string and illegal entries;
  - duplicates;
  - every pairwise overlap and the three-way overlap.
- **`key`.** Label in, then the effect and the inverse lookup. Cases:
  - case, spacing, and colon variants;
  - `starts before` → `(starts after, successor)` for hints only;
  - substrings such as `starts`, which never match.
- **`rule`.** `notes: [{path, body}]`, `day_before`, `day_after`, `events` /
  `completed`, and `config` in; out come the expected `groups` (`after`, `label`,
  `event`, ordered rows `{key, lane, suffix?}`) and `held` (`{key, reason, …}`). Every
  engine-expressible SA row must be present.
- **`copy`.** Inputs in, then every string in the two copy tables, including
  pluralisation and 48-character truncation.

Rust and the cycler both run every `rule` and `copy` vector; nav runs `copy` and `key`.

## Launch gate and sequencing

- **`contract`** may start immediately. It touches docs, fixtures, the inverse table,
  and memory only.
- **`core`, `authoring`, `capture`, and `capture_unlink`** start only after bead
  `bob-cli-fb.2` is CLOSED. Its keep-moving repair rewrites the same planner, output,
  net-pass, and lane-name code.
- **`mac`** starts only after `bob-cli-fb.5` is CLOSED.
- **First step of every gated phase:** run
  `sase bead read bob-cli-fb.2 -r "starts-after launch gate"` (`bob-cli-fb.5` for
  `mac`). If the bead is still open:
  - make no changes;
  - add a note to your phase bead: `BLOCKED: waiting on <bead>`;
  - end the turn.
- **Repositories.** Open every repository other than bob-cli with `/sase_repo`
  (`bob-plugins`, `bob-mac-capture`, `chezmoi`), and read its `AGENTS.md`. Never edit
  the deployed copies of the plugins in `~/bob`.
- **Phases may not edit memory** except where this plan's accepted memory decisions say
  so; those edits are made in `contract`.

## Phase: contract — Contract, shared vectors, inverse pair, and glossary

This phase covers bob-cli only. Read reference memory `docs.md` first (through
`/sase_memory_read`). Write the shared design above into the contract, verbatim where
this plan gives copy.

1. **`docs/task-relations.md`.**
   - **Intro and §2.** Name the third exception (§20): a successor label never changes
     anything at commit. It only adds offers when its target is parked or done.
   - **§1.** Add the **Successor label** and **Label effect** vocabulary.
   - **§10.** Add the `starts after` → `starts before` row.
   - **New `## 20. Successor labels`.** Add these subsections:
     - SU1 The rule;
     - SU2 Config (the YAML above, the one-effect rule, the overlay,
       hide-is-not-disable);
     - SU3 Source and target;
     - SU4 Events, as a summary that links §12.18;
     - SU5 The indication (the authoring table);
     - SU6 Contracts (the authoring JSON in the authoring phase below);
     - SU7 Architecture (policy layer, engines, shared vectors);
     - an SU vector index.
2. **`docs/task-dependencies.md`.**
   - **§12 vocabulary.** Add the `relation` group, the Offer (D) clause, and "a
     predecessor may also leave the plan parked (§12.18)".
   - **§12.1.** Add the parked rows. Change the `x*` exclusion to "`x*` parks offer only
     relation successors (§12.18)". Amend KM20 and any SL row that says parks never
     offer.
   - **§12.2.** Update DEDUP and ORDER.
   - **§12.5.** Add the reporting section above.
   - **§12.6.** Add the 🪢 glyph rule and the copy rules.
   - **§12.7.** Add the relation-sources layer and the targets.
   - **§12.8.** Add the kill switch.
   - **§12.13.** Document picker **API v3** and notice **API v3**, specified in the nav
     phase.
   - **New sections.** Add **§12.18 Relation successors (starts after)**: events, rule,
     dedup, placement, picks, batches, uncertainty, the copy table, and the three mocks.
     Then add **§12.19 SA — relation successor vectors**.
3. **`docs/capture.md`.**
   - Link §12.18 from the parking paragraph and from the `@route+id!` toggle table, with
     the note that toggle offers ship in v1.1.
   - Add `relation_successors` / `relation_held` / `relation_check` and the human header
     to the successor-output passage.
4. **Fixtures.**
   - Create `tests/fixtures/relation_successor/vectors.json` per the schema above, with
     a short `README.md` beside it stating the byte-copy rule.
   - Add `inverse` rows to `tests/fixtures/relation_labels/vectors.json`:
     - `starts after` → `starts before`;
     - `Starts  After:` → `starts before`;
     - `starts before` → `null`.
   - Add `("starts after", "starts before")` to Rust `NAMED_PAIRS`, so the fixture stays
     green in the same commit.
5. **Memory.** Make each change only as its decision allows, through
   `/sase_memory_write`, then run `sase memory init`.

   > [!decision] memory_glossary Make three glossary edits. Each corrects text that this
   > epic makes incomplete or misleading.
   >
   > **Task Relation.** In `glossary/task-relation.md`, change "inert link" to "link,
   > inert unless its label carries an effect,". After the cancel/complete exception
   > sentence, add: "A relation whose label is a configured successor label (such as
   > `starts after`) also offers its source as a [[successor-offer]] when its target is
   > parked (left Pending with no live Task Link today) or done; it still never blocks,
   > and its mirror reads `starts before`."
   >
   > **Task Successor.** In `glossary/task-successor.md`, after the keep-moving clause,
   > add: "A predecessor also leaves the plan when a gesture parks it (Pending, no live
   > Task Link left today); parking or completing it offers every task linked to it with
   > a successor label such as `starts after`, which need not depend on it."
   >
   > **Successor Offer.** In `glossary/successor-offer.md`, after the keep-moving
   > clause, add: "Parking or completing a task also offers the tasks linked to it with
   > a configured successor label (`starts after`) in their own relation group, which
   > says which relation and event fired and that they are not dependents."

   > [!decision] memory_decision_record Create
   > `sase/memory/decisions/successor-labels-offer-when-parked-or-done.md` in the format
   > of `decisions/unblocked-tasks-are-offered.md`:
   >
   > - `metadata.status: accepted`;
   > - title "Successor Labels Offer Their Source When The Target Is Parked Or Done";
   > - a one-sentence roster summary;
   > - the sections Applies to, Claim, Why (the rejected alternatives above), Cost (the
   >   first relation read back after it is written, a fifth group, the `x*` scan), and
   >   Reopens when (repeat offers become noise; S keeps starting before T is handed
   >   off; the inverse spelling should author edges).
   >
   > Its evidence is Bryan's request of 2026-11-01, the research report, and this epic.
   > Link [[decisions:unblocked-tasks-are-offered]] and
   > [[decisions:mac-capture-is-a-thin-client]]. It supersedes nothing.

6. Run `just docs-gen`, then `just docs-check`. Then run the Rust inverse tests
   (`cargo test inverse`) and the relation-label vector tests.

**Done when:**

- §20, §12.18, and §12.19 read as one coherent extension, and every SA row is present.
- The vectors JSON parses, and each SA row is either covered or marked UI-only.
- The docs, inverse, and vector gates are green.

## Phase: core — Rust policy and relation-successor engine

Respect the launch gate. Implement the engine half of §12.18 and SU2. Make no capture
wiring changes; capture output stays byte-identical in this phase.

1. **Policy (`src/native/config/relations.rs`).**
   - Add `successor_labels`, `LabelEffect`, `label_effect`, and `label_for_inverse`.
   - Apply the three-list one-effect rule and the warning text above.
   - Keep `close_effect` and `close_label_for_inverse` as views, so every existing
     cancel/complete test stays green.
   - Drive the tests from the `config` and `key` vectors.
2. **Pure engine `src/native/task_complete/relation_successors.rs`.** Keep it out of the
   2,870-line `successors.rs`, and re-export it from `task_complete/mod.rs`. It
   contains:
   - edge extraction from source texts, using `task_relations::parse` and the reconcile
     target-resolution rules (share them; do not fork them);
   - an `Event {target, kind: Parked|Done, anchor}` input. The anchor is optional, so
     `capture_unlink` can pass an explicit slot;
   - `released(...)` over target statuses and the post-gesture `DayView`;
   - R1–R7 classification, held rows, grouping by `(T, label key)`, row suffixes, and
     ORDER;
   - a pure **copy builder** that produces every string of both copy tables from typed
     inputs, driven by the `copy` vectors. Capture output, authoring, and tests all call
     it, and no other code formats these strings.
3. **Integration in `plan_successors_inner`.**
   - Add `SuccessorInput.relation: Option<RelationSuccessorInput>`, carrying labels,
     parked events, source texts, a target-note reader, and `snapshot_complete`.
   - Done events derive from `completed`.
   - The early `SuccessorPlan::default()` return must not fire when relation events
     exist. A parked-only gesture has an empty C.
   - Add `OfferGroup::Relation` (`"relation"`).
   - Apply DEDUP (`hand_off > relation > project > project_finished > keep_moving`,
     moving project, pull-in, and keep-moving rows into relation groups while keeping
     their recoveries) and ORDER.
   - Reuse `attach_offer`, `route_successor_offers`, `assign_offer_keys`,
     `apply_selection`, `number_offers`, and placement. Parked events from closes use
     the closing context.
   - Add to `SuccessorPlan`: `relation_successors`, `relation_held`, `relation_groups`
     (with `copy`), and `relation_check`.
   - `plan_recovery_only` (cancel) never runs the relation pass.
4. **Tests** in `src/native/task_complete/tests/relation_successor_tests.rs`:
   - every `rule` and `copy` vector;
   - DEDUP with each other group;
   - the unchanged SO / PP / KM / LP suites.

   Cite vector ids in the tests.

5. **Verify.** Run the `just` lint, test, and docs gates from `AGENTS.md`. Do not edit
   `Cargo.toml`'s version or `CHANGELOG.md`.

**Done when:**

- The engine passes every `rule` and `copy` vector.
- Existing successor suites are unchanged.
- No capture JSON changed.

## Phase: authoring — Rust authoring indications and relations list

Respect the launch gate. Every field below is additive and omitted when no successor
label is configured, so output stays byte-identical.

1. **`capture-complete`, `relation_label` context**
   (`src/native/capture_complete/{candidates,engine,complete,render,shell}.rs`):
   - Rows gain `offers_source: true` for configured keys. The zero-use overlay rows of
     configured successor labels carry `configured: true`.
   - The response gains
     `relation_successor: {source: {path, block_id?, text}, target: {path, block_id?, text, status_symbol, status_name}, verdict: "offers"|"already_parked"|"already_closed", message, hint?}`.
     It is present only when a successor label is configured and the token is a
     successor-label (or inverse-spelling) relation.
   - It resolves only the token's two tasks (`~`: S = the owner; `~~`: S = the token's
     task) plus today's day text, and it never scans the vault.
   - `hint` is the inverse hint, set when the typed label is the inverse of a configured
     successor label.
   - zsh rows read `N links · offers source`.
2. **`capture-parse`.** Relation-label spans gain `offers_source: true`.
3. **`capture` commits and dry runs.**
   - `relation_updates[]` rows gain
     `offer_effect: "offers"|"already_parked"|"already_closed"`.
   - `warnings[]` gains `{code: "relation_successor_already_released", label, message}`
     when the target is already released at commit. This includes SA20's later-in-draft
     edge.
   - Nothing fires at commit.
4. **`bob relations list`** (`src/native/relation_labels/cli.rs`):
   - JSON rows gain `offers_source`, plus a top-level `successor_labels` count.
   - The human header gains `· N offer`.
   - Rows read `🪢 offers source`.
   - `⇄ starts before` comes from the inverse pair.
5. **Rename warnings** (`relation_labels/rename.rs`) add the successor-label line.
6. **Tests** cover capture-complete (`~`, `~~`, overlay, hidden still flagged, every
   verdict, the inverse hint), capture-parse spans, commit warnings (SA20, SA30),
   relations list human and JSON output, and rename warnings. The vectors drive the
   strings.
7. **Docs.** Fill in task-relations SU6 if the contract left placeholders. Then run the
   `just` gates.

**Done when:**

- `bob capture-complete` on a scratch vault with `successor_labels: [starts after]`
  reports `offers_source` and each verdict.
- Output is byte-identical with the list empty.

## Phase: capture — Capture events, snapshot, reporting, and bob-authored group copy

Respect the launch gate. This is where the feature becomes real for `bob capture` and
Bob Mac Capture.

1. **Relation-sources snapshot** (`capture/dependencies.rs`, or a sibling module):
   - Make it batch-scoped and lazy, behind G0–G3.
   - Read the catalog walk in one parallel byte pass. When both snapshots are needed,
     share that pass with `DependentsSnapshot`: generalise `read_prefiltered_parallel`
     to several predicates rather than reading twice.
   - Overlay staged files. Implement `snapshot_complete(overlay)` semantics.
   - A walk failure or an unexcused read failure means `relation_check: "unavailable"`.
2. **Events.**
   - **`pomodoro_close.rs::plan_close_successors`:**
     - Parked events come from `summary.task_links` rows with `outcome == "parked"`
       whose task resolves, is `[/]` in staged text, and has no live link on `post_day`.
     - Done events come from C.
     - When relation events exist, bypass the "nothing to do" return. Never add a parked
       task to C, because that would unblock dependents and trigger project rules.
   - **The `!` path and complete-label commits** (`capture/task_complete.rs`): done
     events from C.
3. **Output** (`capture/output.rs`, `pull_in.rs`):
   - Add `relation_successors`, `relation_held`, and `relation_check` to every object
     listed in Reporting.
   - Extend `all_offer_rows` and the refusal payload.
   - Give `pull_in.groups[]` one entry per offered group of every kind, each with `copy`
     from the core copy builder. Port the existing hand_off, project, project_finished,
     and keep_moving strings verbatim from §12.6 and §12.15.
   - Add the `pull_in.copy` top-level object.
   - Add block-diff `reason: "relation"` and `TaskBlockRole::RelationSuccessor`.
   - Set `day_file` when a relation pick links.
4. **Human output.** Add the group header, offer rows (lane names), fold row, held
   notices (only when no group is offered), the uncertainty line, and the linked suffix,
   all from the copy builder.
5. **Net pass.** Generalise `keep_moving_net.rs` (or add a sibling) to cover the Batches
   rules, SA19–SA21.
6. **Tests:**
   - CLI tests in `tests/cli/capture/relation_successors.rs` covering SA1–SA30
     (capture-expressible rows): JSON shape, numbering across groups, `-p` numbers and
     keys, the refusal, dry-run/real-run equality, human output, and the kill switch
     (SA24: no relation field and no snapshot read, as a wiring test);
   - a wiring test proving zero added reads for SA36;
   - existing goldens and SO / PP / KM suites stay green. Update existing goldens only
     where `pull_in.groups[].copy` is the sole addition, and say so in the commit body.
7. **Measure** the Gates and performance targets on apollo (release build, a temporary
   copy of `~/bob`, never the live vault). Record the numbers in §12.7.
8. Run the `just` gates. Do not edit `Cargo.toml`'s version or `CHANGELOG.md`.

**Done when:**

- `bob capture 'x*' --dry-run --format json` on a scratch vault with the base fixture
  reports SA1's group.
- `!` reports SA4.
- Every gate is green and the budget numbers are recorded.

## Phase: capture_unlink — Capture unlink hand-offs (v1.1)

Respect the launch gate.

- When the `@route+id!` toggle **unlinks** a task that is `[/]` before and after (the
  toggle never changes status), raise `(T, parked)`. Its explicit slot anchor is the
  open entry and position of T's first removed link.
- Run the relation-only engine with no C.
- Report `relation_successors` / `relation_held` / `relation_check` on the **capture
  item** (toggles have no sub-object). Include them in `all_offer_rows`, `pull_in`,
  `-p`, the refusal, and human output. Extend the net pass.
- Unlinking a Next or Ready T stays silent (SA32).
- Add CLI tests for SA31 and SA32, then update `docs/capture.md`'s toggle table, §12.1,
  and §12.18 (v1.1 shipped). Run the `just` gates.

**Done when** SA31 and SA32 pass, and dry-run and real-run output match.

## Phase: nav — Obsidian picker v3 and notice v3 presentation

This phase covers bob-plugins `plugins/bob-navigation-hotkeys` only. Read the repo's
`AGENTS.md`; it covers fragments, the 1000-line cap, and `npm run build`.

1. **Picker API v3.** Bump `api.successors.version` to 3.
   - `normalizeSuccessorPickerModel` (`206a`) accepts group `relation` with:
     - `header`, `caption`, `tag`, and `foldText`;
     - `after: {notePath, blockId, text}`;
     - `event: "parked"|"done"`;
     - `label`;
     - folded `alreadyStarted` and `waitsFor`.
   - Row `suffix` (several triggers) and `reason` (second-reason text) are optional on
     every group.
   - Unknown groups are still dropped. v1 and v2 models render exactly as before.
2. **Rendering** (`206`, `styles.css`):
   - `successorPickerGroupTitle` uses `header` for relation groups.
   - A group heading shows its caption and a right-aligned tag.
   - The fold line uses `foldText` when present.
   - The header icon is `link-2` and the modal title is the model title when every group
     is relational.
   - Add `.bob-successor-group.is-relation` with a **neutral** left accent and no new
     hex values.
   - Show row suffixes and second reasons as muted meta.
3. **Notice API v3** (`285`). Bump `api.notice.version` to 3.
   - `buildUnblockedNoticeModel` reads an optional `relation_successors` array.
   - When every row is relational, the title is `‹Label› “‹T›”` with 🪢 in the glyph
     slot and modifier class `is-relation`.
   - Mixed cards get a `‹label› “‹T›”` chip.
   - The receipt reads `✓ ‹T›` for done and `‹T› parked` for parked.
   - Add muted lines for held notices and for uncertainty.
4. **Tests.** Extend `scripts/test-navigation-successor-picker.cjs` and
   `scripts/test-navigation-hotkeys-unblocked-notice.cjs` to cover:
   - v1/v2 parity;
   - the v3 relation group, caption, tag, fold, title, and icon;
   - mixed ordering.

   Copy strings come from the copied `relation_successor` `copy` vectors under
   `plugins/bob-navigation-hotkeys/scripts/fixtures/relation_successor/`. Export new
   pure helpers from `700-exports.js`.

5. **Ship.**
   - Bump the nav manifest (minor) and its description.
   - Update `docs/develop/plugin-apis.md` (successors v3, notice v3).
   - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.

**Done when** a hand-built v3 model renders the Obsidian mock, older models are
unchanged, and the plugins are synced.

## Phase: nav_authoring — Obsidian inverse pair and relation-stage indications

This phase covers bob-plugins only (bob-ledger-tools and bob-navigation-hotkeys).

1. **Inverse pair.**
   - Add `"starts after": "starts before"` to `RELATION_INVERSE_NAMED_PAIRS` in both
     `145a` and `303a`, keeping them byte-identical.
   - Re-copy `relation_labels/vectors.json` from bob-cli byte-for-byte.
   - Update the hard-coded pairs assertion in
     `scripts/test-ledger-tools-relation-chips.cjs`.

   Chips and the Linked-from stage then render `starts before` with no other change.

2. **Config.**
   - Extend `loadRelationsCloseLabels` (`022`) to return
     `{cancel, complete, successor, warnings}` with the three-list one-effect rule.
   - Copy `relation_successor/vectors.json` and run its `config` and `key` sections.
   - Keep `loadRelationsCancelLabels` and every closing behavior unchanged.
3. **Links / Linked-from stages** (`305`, `426`, `428`):
   - The configured successor label shows `🪢 offers` row meta, plus a zero-use overlay
     marked configured.
   - The consequence strip shows
     `🪢 Offers “S” when “T” is parked or done · never blocks`.
   - Show the amber caveat when T is already released. Compute it from
     `api.relationIndex` / the Tasks memo statuses and today's daily-note live links.
     When unknown, show no caveat.
   - Show the muted inverse hint for `starts before`.
   - Hand edits need no hint, because hand-edited edges also arm offers.
4. **Tests.** Extend the relation-stage, relation-chips, and label-vocabulary scripts.
   Cite SA22, SA23, and SA30.
5. **Ship.**
   - Bump both manifests (minor).
   - Update `docs/task-relations.md` in bob-plugins.
   - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.

**Done when** chips read `starts before`, the relation stage shows all four indications
from the vectors, and the plugins are synced.

## Phase: cycler — Obsidian cycler relation-successor engine (Done path)

This phase covers bob-plugins `plugins/task-status-cycler` only. It depends on nav v3.

1. **New fragment `089d-successor-relation.js`.** `085` and `087` are at the cap.
   Register it in `src/fragments.json`. It contains:
   - `loadRelationSuccessorLabels()`, which parses `relations.successor_labels` with the
     shared one-effect rule (the `config` vectors). Cache it by stat, like
     `loadKeepMoving`.
   - The pure planner `planSuccessorRelations(options)` (R1–R7, released, grouping,
     ORDER, suffixes) and the copy builder, both passing every `rule` and `copy` vector.
     That includes parked events, so `obsidian_unlink` only wires.
   - An adapter over ledger-tools `api.relationIndex` v2:
     - feature-detect it;
     - `incoming(path, blockId)` filtered by label key;
     - `outgoing(S)` for R6, plus statuses;
     - not-ready means uncertainty, never silence.
2. **Wiring.**
   - Add the relation pre-gate to `quietGate` (`137`). The relationIndex lookups are in
     memory.
   - Add the planner to `applyContext`.
   - In `089b`, update `priorityOf` to the DEDUP order and route rows to
     `relation_successors` in `appendRow`.
   - Picks reuse the hand_off write path. Re-plan refusal follows SA29.
   - `closeKind: "cancelled"` never runs it.
   - Plan relation groups **only when nav `api.successors.version >= 3`** (SA34).
3. **Picker model** (`085b`). Build relation groups with `header`, `caption`, `tag`,
   `foldText`, `after`, `event`, `label`, row `lane` / `suffix` / `reason`, and the
   title rule.
4. **Notice.**
   - `successorNoticeText` gains the 🪢 forms.
   - Pass `relation_successors` to nav only when `api.notice.version >= 3`; otherwise
     use the plain-text fallback.
   - Add the held notice and the uncertainty line.
5. **Receipt.** `rememberSuccessorPass` records relation picks, so a same-day reopen
   takes them back (SA35).
6. **Tests.** Add `scripts/test-task-status-cycler-relation-successors.cjs`, registered
   in `package.json`:
   - every `rule` and `copy` vector, using a fake relationIndex built from the vector
     notes;
   - wiring through the fake nav picker: v3 offers, v2 suppression, Esc, the
     walk-landing hold, re-plan refusal, the receipt, and the zero-read path (SA36).
7. **Ship.**
   - Bump the cycler manifest (minor).
   - Update `docs/task-status-cycler.md`.
   - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.
   - If a **scratch copy** of the vault can be opened in Obsidian, verify Ctrl+Enter on
     T live. Otherwise report that live verification is left for Bryan.

**Done when:**

- Every `rule` and `copy` vector passes in JS, with keys, order, and destinations
  matching Rust.
- Every Obsidian Done gesture offers relation successors.
- The plugins are synced.

## Phase: obsidian_unlink — Obsidian unlink hand-offs (v1.1)

This phase covers bob-plugins (task-status-cycler and block-id-prompt).

1. **Cycler API.** Bump the cycler api version and add
   `offerParkedSuccessors(identities, {slot})`. It:
   - checks that each target is `[/]` with no live link on the fresh daily note;
   - plans relation-only offers with `event: parked`;
   - presents the nav picker, applies picks, and shows the notice;
   - never throws, and resolves `{offered, picked}`.
2. **block-id-prompt.** Hook every Ctrl+Shift+Enter path that removes a `[/]` task's
   open-Pomodoro links.
   - Act only after the `Unlink task` prompt settles **and** the unlink write commits.
     Esc that cancels the unlink offers nothing (SA33).
   - Feature-detect the cycler API, and pass the removed link's entry and position as
     `slot`.
   - Older cyclers mean silence.
3. **Tests.** Cover SA33 (prompt settled, Esc, non-Pending target, a target still linked
   elsewhere today), and confirm that block-id-prompt's own suites stay green.
4. **Ship.**
   - Bump both manifests.
   - Update `docs/block-id-prompt.md` and `docs/task-status-cycler.md`.
   - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.

**Done when** unlinking a Pending task in Obsidian offers its starts-after sources, and
every other unlink stays silent.

## Phase: mac — Bob Mac Capture generic group copy and relation presentation

Respect the launch gate (`bob-cli-fb.5`). This phase covers bob-mac-capture only. Read
its `AGENTS.md`: Swift must compile, `just portable-check` runs, and the Mac scratch
typecheck applies.

1. **Decode** with `decodeIfPresent ?? []` throughout:
   - `relation_successors`, `relation_held`, and `relation_check` on
     `CaptureTaskComplete`, `PomodoroCloseSummary`, nested relation completes, and the
     capture item (toggles);
   - `offer.after`, row `relation`, `pull_in.copy`, `pull_in.groups[].copy` / `after` /
     `label` / `event`, and the new folded keys;
   - block-diff `reason == "relation"`;
   - `relation_successor` in `capture-complete`, and `offers_source` on spans and
     candidates.
2. **Generic group presentation.**
   - Sections key on `(group, project, after)` and follow the order of bob's
     `pull_in.groups`.
   - Titles, why lines, tags, fold lines, glyph keys, the capsule, and the context line
     all come from bob's `copy`.
   - The Mac maps glyph keys to SF Symbols. Unknown keys get a neutral fallback, never
     `lock.open`:
     - hand_off → `arrow.turn.down.right`;
     - project → `folder`;
     - project_created → `folder.badge.plus`;
     - project_finished → `checkmark.circle`;
     - keep_moving → `forward.end.fill`;
     - relation → `link`.
   - The Swift per-group switches stay only as the fallback for older bobs that send no
     `copy`.
   - "Unblocked by …" and `trigger: …` never appear for non-dependency groups.
   - Second-reason and suffix meta render muted.
3. **Gather** relation offers in `makePullInPreviewSnapshot` from every location listed
   above, so Return opens the choice step and `--pull-in` keys round-trip.
   - Preview cards (`pullInPreviewSummary`) show the relation section with
     `↩ choose what to pull in`.
   - Held notices render muted when no section exists.
4. **Notification and badges.**
   - `notificationLine` / `pulledInSummary` count relation picks and render
     `🪢 Pulled in n · ‹label› “‹T›”`.
   - Block-diff relation lines get the `link` badge.
5. **Authoring.** Extend `CaptureRelationCloseEffect` / `RelationSourceEffect` with an
   `offers` effect:
   - SF `link` capsule `offers`;
   - the slot shows bob's `relation_successor.message`;
   - caveats are amber and inverse hints are muted.

   Never compute the verdict in Swift.

6. **Fixtures.** Add `Scripts/generate-relation-successor-fixtures.sh`, modelled on
   `generate-project-note-offer-fixtures.sh`.
   - Use a `BOB_BIN` built from bob-cli master (open it with `/sase_repo`, a scratch
     build, no version edits).
   - Generate `x*` dry-run, `!` dry-run, toggle-unlink, held-only, mixed `x*!2`, and
     capture-complete authoring fixtures.
7. **Tests.** Add them to `CapturePullInDecodingTests`, `CapturePullInChoiceTests`,
   `CapturePullInChoiceDesignTests`, `CaptureSuccessorPresentationTests`,
   `NotificationServiceTests`, and the relation-effect tests. They cover:
   - absent, empty, and populated arrays;
   - generic copy for every existing group, with keep-moving matching fb.5's look;
   - older payloads without `copy` rendering exactly as before.
8. **Docs and verify.**
   - Update `docs/protocol.md` and `docs/capture-previews.md`, then run
     `just docs-check`.
   - Run `just portable-check` (on athena if `swift` is missing), plus the Mac scratch
     `bundle.sh` and `just typecheck-tests`.
   - If the Mac is offline, say the macOS targets are unverified.

**Done when** fixture-driven tests show the Pull-In card mock, older payloads are
unchanged, and portable-check passes.

## Phase: enable — Enable starts after in chezmoi and smoke it

1. Open `chezmoi` with `/sase_repo` and read its `AGENTS.md`.
2. In `home/dot_config/bob/config.yml`, add `successor_labels: [starts after]` under
   `relations:`, with the comment from Config and policy.
3. Apply only that target the way chezmoi's `AGENTS.md` prescribes. If it says Bryan
   applies, leave it to him and say so.
4. **Smoke test** on a scratch vault (never the live vault) with the base fixture:
   - `bob capture 'x*' --dry-run` shows SA1;
   - `-p 1` links S in the continuation;
   - `!` shows SA4;
   - `bob relations list -a` shows `🪢 offers source  ⇄ starts before`.
5. Report which surfaces were verified live (CLI) and which are left for Bryan
   (Obsidian, the Mac).

**Done when** the config is committed in chezmoi and the CLI smoke matches SA1, SA4, and
the relations-list row.

## Verification (epic level)

- **One rule, two engines.** Rust and the cycler pass every `rule` and `copy` vector,
  with identical keys, order, destinations, and strings.
- **Every surface:**
  - `bob capture`: `x*` (parked), `!` / `x!` / complete labels (done), and `@route+id!`
    (v1.1);
  - Bob Mac Capture, through bob;
  - Obsidian: Ctrl+Enter, Alt+], the walk, and Links-stage complete labels (done), plus
    block-id-prompt unlink (v1.1).

  Cancel never offers.

- **Clarity.**
  - Every relation group reads `‹Label› “‹T›”`, an event caption, and
    `your links · not dependents`, with the 🪢 / `link-2` / `link` glyph.
  - No relation surface says "unblocked" or shows 🔓.
  - LINKED FROM chips read `starts before`.
- **Safety.**
  - An empty list or `link_unblocked: false` gives no relation field and no scan.
  - Older nav, cycler, and Mac versions never mislabel the rows.
  - The zero-read and timing budgets hold, and the numbers are recorded in §12.7.
