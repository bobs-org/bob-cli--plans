---
tier: epic
title: 'Undirected task relations: a plain link has no direction, a label gives it
  one'
goal: 'An unlabelled task relation is one undirected link, shown everywhere as `🪢
  links` with no arrow. A labelled relation is a directed statement read from its
  source. A bare `~~` asks for a label. Every writer, reader, and client (bob-cli,
  bob-plugins, Bob Mac Capture) applies the same pair rules on the unchanged LINKS
  / LINKED FROM storage, so a plain re-add never erases a label and a pair is never
  listed twice.

  '
decisions:
  clear_scope:
    ask: What should Ctrl+D on the merged Task Card Links… row clear?
    choices:
      shown: Every relation the row shows whose record is writable; any read-only
        ones are kept and named
      own: Only plain links and this task's own statements; other tasks' statements
        stay
    default: shown
    why: The row shows one list of related tasks, so clearing it should clear that
      list
    answer: shown
  glossary_task_relation:
    ask: Should the Task Relation glossary strand be rewritten for undirected and
      directed relations?
    memory:
    - glossary:task-relation
    default: true
    requested: Any relation with a description would then become a directed relation
      (update the glossary term and add new ones for "directed" and "undirected" relations).
    answer: true
  glossary_undirected:
    ask: Should a new Undirected Relation glossary strand (aka plain link) be added?
    memory:
    - glossary:undirected-relation
    default: true
    requested: Any relation with a description would then become a directed relation
      (update the glossary term and add new ones for "directed" and "undirected" relations).
    answer: true
  glossary_directed:
    ask: Should a new Directed Relation glossary strand (aka labelled relation) be
      added?
    memory:
    - glossary:directed-relation
    default: true
    requested: Any relation with a description would then become a directed relation
      (update the glossary term and add new ones for "directed" and "undirected" relations).
    answer: true
  decision_record:
    ask: Should the decision record relations-undirected-unless-labelled be added?
    memory:
    - decisions:relations-undirected-unless-labelled
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
phases:
- id: contract
  title: Relation contract, glossary, and decision record
  depends_on: []
  size: medium
  description: 'contract: rewrite docs/task-relations.md around the pair model (vocabulary,
    pair rules, projection, writer table, chips, JSON, and every new or changed vector),
    update the vocabulary-level docs, and write the glossary strands and the decision
    record through /sase_memory_write.

    '
- id: pair-core
  title: Rust pair model, planner, and reconcile projection
  depends_on:
  - contract
  size: medium
  description: 'pair-core: add a pure pair-state and pair-write planner in task_relations,
    stop bare adds from stripping labels, teach reconcile to omit plain mirrors the
    target already names, and add the redundant_relation warning, all pinned by the
    LW and LR vectors.

    '
- id: capture-grammar
  title: Capture grammar, completion, and the label-required reverse link
  depends_on:
  - pair-core
  size: medium
  description: 'capture-grammar: make a bare `~~` an incomplete draft that submit
    refuses, add `kind` to parsed relations, make completion symmetric with existing_kind/existing_direction,
    end every `~~` replacement in `|`, add the related-label hint, rename the shell
    groups, and update the capture syntax docs (ships as `feat(capture)!:`).

    '
- id: capture-writes
  title: Pair-aware capture writers and sentence-shaped output
  depends_on:
  - pair-core
  - capture-grammar
  size: medium
  description: 'capture-writes: route `~` / `~~` writes through the pair planner in
    one all-or-nothing batch (no-op re-adds, label conversions, moves, both-endpoint
    mirror deltas), add `kind` and `moved_from` to relation_updates, rewrite the human
    output as sentences, and update the linking docs and guide.

    '
- id: nav-writer
  title: Obsidian pair planner twin, writers, and relations API v2
  depends_on:
  - pair-core
  size: medium
  description: 'nav-writer: port the pair planner to bob-navigation-hotkeys with shared
    vectors, make applyRelationEdit, removeLink, Ctrl+D, and the hand-edit mirror
    pair-aware with add-before-remove cross-note writes, and publish `api.relations`
    v2 (unlink, removeStatement, reverse openStage).

    '
- id: ledger-chips
  title: One aggregated links block per task in bob-ledger-tools
  depends_on:
  - contract
  size: medium
  description: 'ledger-chips: add the additive `related()` per-task view to `api.relationIndex`
    and render one `🪢 links` block per task (arrow-free plain chips, then statements)
    in Live Preview and Reading view, collapsing the second relation line, with v2-aware
    × and ＋ controls.

    '
- id: nav-stage
  title: One Links row and one Links stage with a live label sentence
  depends_on:
  - nav-writer
  - ledger-chips
  size: medium
  description: 'nav-stage: merge the Task Card rows and the two relation stages into
    one Links row and stage fed by `related()`, with plain ↵, a ⇧↵ label field showing
    a live sentence and ⇄ swap, ⇧L as the reverse-label shortcut, new copy, plugin
    docs, and a final `bob plugins sync`.

    '
- id: mac
  title: Bob Mac Capture pickers, label card, and previews
  depends_on:
  - capture-grammar
  - capture-writes
  size: medium
  description: 'mac: decode the new relation fields, route every `~~` pick into the
    label card, give the card a live plain/directed sentence with swap and a visible
    plain-link choice, and update picker, preview, notification copy, fixtures, and
    docs.'
proposed_by: bbugyi200.apollo.research.29.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-21 10:08:34
status: wip
bead_id: bob-cli-am
---

- **PROMPT:** [prompts/202610/undirected_task_relations.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/undirected_task_relations.md)
- **BEAD:** [bob-cli-am](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-am/README.md)

# Plan: Undirected task relations

## Context

Task relations shipped days ago as **directed** edges: `~` / `~~` in capture, `l` / `⇧L`
in the Task Card, and chips headed `🪢 links to →` / `🪢 linked from`. For an unlabelled
relation that direction means nothing, yet Bob makes you choose one and then shows it
everywhere. Bryan wants the rule **"a relation has no direction until you say what it
means"**: no label means undirected and shown as `links`; a label means directed; and
`~~` only makes sense with a label.

A five-way research swarm studied this and recommended a design. Bryan accepted **all**
of its requirement adjustments (A1–A11). Phase agents should read the report for
rationale, diagrams, and rejected alternatives:

```bash
sase artifact read file:explicit:8ab1c4f71f07d43d92b84ce4 "Rationale for the undirected relations epic"
```

(label `research:202610/undirected_task_relations/undirected_task_relations.md`). This
plan is the design of record. Where the plan and the report differ, **the plan wins**:
it settles the report's open questions, narrows some JSON, and fixes the phase split.

The research verified three defects that make this a **model change, not a copy
change**:

- A bare re-add silently strips a label: `plan_links_edit` treats "no label" as a
  different label (`src/native/task_relations/plan.rs`, the `entry.1 == label` branch).
- `already_linked` looks one way only (`src/native/capture_complete/candidates.rs`,
  `stored_link`), so `~T` against a task that already links here creates a reciprocal
  duplicate.
- Reciprocal plain records would show the same task twice.

Facts that keep this cheap:

- The vault holds exactly one relation (`~/bob/sase.md`), and it is plain. **Nothing
  needs migrating.**
- **Storage does not change.** Line names, grammar, slots, canonical sort, the kill
  switch, and LR11 all stay byte-for-byte.
- The relation-card truncation fix that had to land first (epic `bob-cli-aj`, phase
  `.2`) is already on bob-mac-capture master (`bf40f48`).

## The model (every phase implements this)

### Vocabulary

- **Label** is the one user-facing word for the relation text (A1). The words "relation
  description" and the nav field name "Relation" go away. The nav field says **Label**,
  and the Mac says "link with a label". JSON keeps `label`. "Description" already means
  task text in Bob.
- An **undirected relation** (aka **plain link**) is a relation with no label.
- A **directed relation** (aka **labelled relation**, or a **statement**) is a relation
  with a label, read from its source: "S ‹label› T".
- **Record**: one entry on one task's `🪢 **LINKS:**` line, written
  `(owner, other, label?)`, with at most one per ordered pair. This is today's storage
  unit.
- **Pair**: two distinct tasks {A, B}. Their meaning is derived from their (at most two)
  records. There is never a stored flag that could disagree with the records.
- Relations are still **task-to-task** (A11). "Linking another note" in the request
  means another task, including `^prj` and `^ref` tasks.

### Pair rules

| Records between A and B               | Meaning                                                                        | Shown on each task           |
| ------------------------------------- | ------------------------------------------------------------------------------ | ---------------------------- |
| none                                  | unrelated                                                                      | nothing                      |
| only plain (one, or one on each side) | one **undirected relation**                                                    | `(○ other)` under `🪢 links` |
| one or two labelled, plus any plain   | one **directed relation** per labelled record; plain records are **redundant** | the labelled entries only    |

- **Labelled subsumes plain.** A plain record never adds a chip to a pair that already
  has a statement.
- **A pair has at most one statement in each direction.** Two statements (A —x→ B and B
  —y→ A) can coexist, as LR19 allows today.
- **Nothing is erased by omission** (A7). A plain add never removes or changes a label.
  Clearing a label is always an explicit act.
- **Direction comes from the sentence, never from storage.** For a plain link, which
  task stores the record is provenance only. When a label is added, the record goes to
  the task the sentence names as its source.
- **Writers delete redundant plain records** whenever a write already changes the pair
  and the note is writable. Redundant records can still come from hand edits, sync
  races, or a failed plugin follow-up. They render nothing, reconcile warns about them,
  and the next writer that touches the pair cleans them up.
- **Inertness is unchanged.** Self-relations stay refused. Relations never block, imply
  transitive or reverse links, or change status, lane, freshness, schedule, Pomodoro
  sections, or the review walk.

### Writability

An endpoint is **read-only** when it is in `done/` or in the previous daily note. A
**`~~` source** must also pass today's LX7 refusal set: not a daily note, not `done/`,
not `_templates` / `_generated` / `_conflicts` / a dot-directory, and not fenced code.
Removing the _other_ task's redundant plain record follows mirror writability: any
active note, including today's daily note, is writable, but `done/` and the previous
daily note never are.

### The LINKED FROM projection (one new condition)

```text
mirror(T) = sort_canonical({ (S, label) : (S, T, label) ∈ edges
                             ∧ ¬(label = ∅ ∧ T's own LINKS names S) })
```

- A plain incoming entry is omitted when the target already names the source on its own
  LINKS line. Labelled incoming entries are always mirrored.
- **A malformed LINKS line on T names nobody** (never prune on uncertainty).
- Reconcile still **never edits LINKS**, and LR11 still holds.
- Reconcile gains warning kind **`redundant_relation`**. It fires once per pair that
  holds a redundant plain record, only when that record is writable, and is anchored on
  that record's LINKS line.
  - For two reciprocal plain records, anchor on the writable one whose
    `(path, block id)` sorts last.
  - A redundant record in `done/` or the previous daily note is never warned about.
  - Messages: `plain link to [[B]] is redundant: [[B]] already links here · remove one`,
    or `plain link to [[B]] is subsumed by ‘y’ from [[B]] · remove it`.

### The pair write planner

Rust is primary and the JS twin is pinned by the same vectors. The planner is **pure**.
Its inputs are the pair's two records, each `Option<label>` with a `writable` flag, plus
one request. It returns:

- the LINKS operations per side (`add(label)`, `relabel(label)`, `remove`);
- the LINKED FROM deltas per side, derived by evaluating the projection for this pair
  before and after;
- the `action`, the resulting `kind`, an optional `moved_from` side, and an optional
  refusal.

In the table, N is the owner (the captured task, or the Task Card's task) and T is the
other task.

| Request                                                            | Pair before                                    | Result                                                                                                                    | Reported                                                                                     |
| ------------------------------------------------------------------ | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Link** (plain: `~T`, stage ↵)                                    | unrelated                                      | add `[[T]]` to N                                                                                                          | `added`, `kind: undirected`                                                                  |
| Link                                                               | any relation                                   | nothing written; any label kept                                                                                           | `unchanged`, plus the existing kind and label                                                |
| **State** N —x→ T (`~T\|x`; `~~T\|x` is the same with T as source) | unrelated                                      | add `x [[T]]` to the source                                                                                               | `added`, `kind: directed`                                                                    |
| State                                                              | plain record on the source                     | relabel it in place; also remove the target's reciprocal plain record if writable                                         | `relabeled` (`previous_label: null`), plus `moved_from` when a reciprocal was removed        |
| State                                                              | plain record on the target only                | add `x [[T]]` to the source, **then** remove the target's record if writable (in `done/` it stays, subsumed and unwarned) | `relabeled`, plus `moved_from` when removed                                                  |
| State                                                              | the source already states y (y ≠ x) / states x | relabel in place / nothing                                                                                                | `relabeled` / `unchanged`                                                                    |
| State                                                              | only the reverse statement T —y→ N             | add N —x→ T; the reverse statement stays                                                                                  | `added`                                                                                      |
| **ClearLabel** N —x→ T                                             | no reverse statement                           | the entry stays where it is, without its label                                                                            | `relabeled`, `label: null`                                                                   |
| ClearLabel                                                         | plus T —y→ N                                   | remove N's record; the pair stays linked by T's statement                                                                 | `removed` (notice: `Removed ‘x’ · still linked by ‘y’`)                                      |
| **Swap** N —x→ T (T writable; refused while T —y→ N exists)        |                                                | add `x [[N]]` to T (relabelling T's plain record if one exists), then remove N's record                                   | `relabeled`, `moved_from: N`                                                                 |
| **RemoveStatement** S —x→ T                                        |                                                | remove that record; if no visible relation would remain, also remove every remaining writable record of the pair          | `removed`                                                                                    |
| **Unlink** {A, B}                                                  | any                                            | remove every writable record of the pair                                                                                  | `removed`; if a read-only record remains, the notice names it (`still linked from bob_done`) |

- **Refusal.** Refuse only when nothing writable can be changed. The message names the
  reason: `the link is stored on an archived task` or `… on the previous daily note`.
- **Mints.** The planner never mints. Writers mint a missing `^id` exactly as today, and
  also mint T when a move or swap needs to link to T.
- **Plain-link storage.** A new plain record goes on N. In Obsidian, when N is read-only
  and T is writable, it goes on T instead. Plain links have no direction, so this costs
  nothing. Capture's owner is always writable.
- **Add before remove.** A cross-note move writes the new labelled record first. If the
  removal then fails, the leftover plain record is subsumed, the display is already
  correct, reconcile warns, and the next touch cleans it up. Capture writes
  all-or-nothing anyway.
- **Line-level guard.** `plan_links_edit` (and its JS twin `planLinksEdit`) must treat
  an unlabelled `add` of an already-labelled entry as `unchanged`. The explicit
  `relabel` map stays the only way to clear a label.

### Capture grammar and JSON (schema_version stays 1; additive only)

| Draft                                 | Reading                                | Dry-run / preview row                        |
| ------------------------------------- | -------------------------------------- | -------------------------------------------- |
| `Fix it ~work:foo`                    | undirected {new, foo}                  | `🪢 links "Foo"   work.md ^foo`              |
| `Fix it ~work:foo\|"follow-up of"`    | new —follow-up of→ foo                 | `🪢 follow-up of → "Foo"   work.md ^foo`     |
| `Fix it ~~work:foo\|"next step"`      | foo —next step→ new                    | `🪢 "Foo" — next step → this   work.md ^foo` |
| `Fix it ~~work:foo`                   | **incomplete**: needs `relation_label` | refused on submit with the teaching error    |
| `@bob+fix ~work:foo`, already related | `unchanged`                            | `🪢 already linked "Foo" · follow-up of`     |

- **Bare `~~`** (A4) reports `needs: ["relation_label"]` and the diagnostic
  `relation_label_required`.
  - Diagnostic message: `~~ links need a label · use ~ for a plain link`.
  - `bob capture` refuses with
    `cannot relate ~~work:foo: ~~ needs a label, as in ~~work:foo|"next step" · use ~work:foo for a plain link`.
    Nothing is written or minted.
  - It is never silently rewritten to `~`.
- **`kind`.** `capture-parse` `relations[]` (top-level and per item) gains
  `kind: "undirected" | "directed"`. It is `directed` when the token has a `|` or the
  `~~` sigil, so it reports what the token becomes once complete. `capture-complete`
  `picker.relationship.relations[]` gains the same `kind`.
- **Completion rows.** `capture-complete` `relation_to` / `relation_from` rows change as
  follows:
  - `already_linked` becomes **symmetric**: the pair has any relation, either way.
  - New `existing_kind` (`undirected` | `directed`) and `existing_direction` (`to` |
    `from` | `both`, relative to the owner; directed pairs only).
  - `existing_label` is the label in the row's direction when present, otherwise the
    reverse statement's label.
  - Already-linked rows stay enabled, because labelling or adding a reverse statement is
    still legal.
- **Replacements.** Every `~~` replacement ends in `|`: the `relation_from` row
  replacement (`~~note:id|`), `capture-task-id`'s `relation_from_replacement`, and the
  shell rows. As a result, every client lands in the label step.
- **Related-label hint.** The `relation_label` context's `relation` object gains an
  optional `hint`. It is set when the typed label, case-insensitively and after
  normalization, is one of `related`, `related to`, `relates to`, `see`, or `see also`.
  The hint reads `‘see also’ makes this directed · leave it empty for a plain link`. It
  never refuses (A3).
- **Write output.** `capture` / `--dry-run` `relation_updates[]` gains `kind` and
  `moved_from: {path, block_id}`. `moved_from` is present when the write removed the
  other task's plain record for this pair.
  - The action enum stays `added | relabeled | unchanged`.
  - `direction` keeps meaning "which task authors the reported record": `to` = the
    owner, `from` = the other task. For added and relabeled rows this equals the sigil.
  - An `unchanged` row reports the existing record that subsumes the token. It prefers
    the owner-authored statement, then the reverse statement, then the owner's plain
    record, then the other task's plain record. Its `source`, `target`, `label`, `kind`,
    and `direction` describe that record.
  - Clients ignore `direction` when `kind` is `undirected`.
- **Reconcile JSON.** `bob task reconcile` `relation_warnings[].kind` gains
  `redundant_relation`.
- **Shell completion groups** become `links` (for `~`) and `reverse · needs a label`
  (for `~~`).
- **Within one capture item:**
  - a plain token on a pair that also has a labelled token is dropped before planning
    and produces no row (LX17);
  - `~T|a ~T|b` and `~~T|a ~~T|b` stay refused (LX14);
  - `~T|a ~~T|b` writes two statements;
  - an exact duplicate token stays an `unchanged` no-op.
- **Compatibility.** A new Mac with an old bob sees no `kind` and treats `label == null`
  as plain. An old Mac with a new bob shows "links to" copy for at most one train. Mac
  decodes `direction` / `action` as tolerant strings (verified), so no schema bump is
  needed.

### Copy across every surface

"This" is the owner, captured, or current task. X is the other task. Text previews use
an em dash for plain links and never `↔`. Colour carries status, never direction.

| Relation        | Chips and nav CURRENT                                             | CLI capture                                                                             | Mac preview row     | Notice / notification                                      |
| --------------- | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------- | ---------------------------------------------------------- |
| plain           | `(○ X)` under `🪢 links`                                          | `🪢 links "X"`                                                                          | `🪢 links “X”`      | `🪢 Linked “This” and “X”`                                 |
| this —x→ X      | `‹x› → (○ X)`                                                     | `🪢 x → "X"`                                                                            | `🪢 x → “X”`        | `🪢 “This” — x → “X”`                                      |
| X —x→ this      | `‹inverse› (○ X)`, or `(○ X) · ‹x› → this` when no inverse exists | `🪢 "X" — x → this`                                                                     | `🪢 “X” — x → this` | `🪢 “X” — x → “This”`                                      |
| already related | —                                                                 | `🪢 already linked "X"`, plus ` · x` when directed                                      | same                | —                                                          |
| relabelled      | —                                                                 | `🪢 relabeled x → "X"`, plus ` · was y`                                                 | same                | `🪢 “This” — x → “X”`                                      |
| removed         | —                                                                 | —                                                                                       | —                   | `🪢 Unlinked “X”` / `🪢 Removed ‘x’ · still linked by ‘y’` |
| dry run         | —                                                                 | `would link "X"` / `would add x → "X"` / `would add "X" — x → this` / `would relabel …` | —                   | —                                                          |

- The CLI locator suffix (`   work.md ^foo`) always names the other task.
  - This fixes today's `~~` quirk, where `human_relation_line` quoted the owner's own
    text.
- The inverse-phrase table (contract §10: `merged into` → `merged from`, …) stays
  **display-only** and unchanged.
  - The ledger and nav copies must agree, returning `null` for unknown labels. A shared
    vector file in bob-plugins pins them.
  - Rust needs no copy.
- Mac icons: plain rows keep `arrow.left.arrow.right`, and statements use `arrow.right`.

### Invariants every phase tests where it applies

1. A plain link created from A and then from B is **one** relation, and the second
   gesture is `unchanged`.
2. A plain add never changes or removes a label.
3. Each pair renders at most once as plain, and never as plain _and_ labelled.
4. A label means the same thing from either end, whichever task stores the record. Add
   an endpoint-swap property test for plain links in Rust.
5. After a failed cross-note follow-up, the display is already correct and reconcile
   reports `redundant_relation`.
6. Delete every LINKED FROM line and reconcile twice: the lines are rebuilt identically,
   and the second run is a no-op (LR11).
7. Every relation operation leaves statuses, lanes, fields, dependencies, Pomodoro
   sections, and the review walk byte-identical, apart from a minted `^id` (LR18, LX13,
   LG13).
8. Dry runs and live runs plan identical actions. Completion, preview, and Escape mint
   and write nothing.

### Vectors (the contract phase writes them into `docs/task-relations.md` §15; later phases copy them)

| #                     | Situation                                                                                                         | Outcome                                                                                                                                                  |
| --------------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| LW4 _(narrowed)_      | re-add T with a **new non-empty** label                                                                           | relabeled in place                                                                                                                                       |
| LW15                  | plain add; the pair already has a plain record on either side                                                     | `unchanged`; byte-identical                                                                                                                              |
| LW16                  | plain add; the pair has a statement either way (incl. `plan_links_edit` add-without-label on a labelled entry)    | `unchanged`; label kept                                                                                                                                  |
| LW17                  | N —x→ T requested; writable T holds plain `[[N]]`                                                                 | N gains `x [[T]]`; T's entry removed; T's mirror gains `[[N]] (x)`; N's mirror drops `[[T]]`; `moved_from` T                                             |
| LW18                  | as LW17, but T's record is in `done/`                                                                             | N gains `x [[T]]`; T's record stays (subsumed, unwarned); N's mirror drops `[[T]]`                                                                       |
| LW19                  | clear the only label                                                                                              | the entry stays where it is, without its label                                                                                                           |
| LW20                  | clear N —x→ T while T —y→ N exists                                                                                | N's record removed; T's statement kept                                                                                                                   |
| LW21                  | swap N —x→ T                                                                                                      | T gains `x [[N]]`; N's record removed; refused while T —y→ N exists                                                                                      |
| LW22                  | unlink a pair with plain records on both sides / remove the only statement while a redundant plain record remains | every writable record removed; pair unrelated                                                                                                            |
| LW23                  | N —x→ T requested; N holds plain `[[T]]` and T holds plain `[[N]]`                                                | N's entry relabeled in place; T's reciprocal removed (`moved_from` T)                                                                                    |
| LR19 _(changed)_      | plain A → B and plain B → A                                                                                       | no LINKED FROM entry for either; one `redundant_relation`                                                                                                |
| LR25                  | plain A → B and labelled B —y→ A                                                                                  | B's mirror omits A; A's mirror lists `[[B]] (y)`; `redundant_relation` on A's line                                                                       |
| LR26                  | labelled A —x→ B and labelled B —y→ A                                                                             | each mirror lists the other with its label (old LR19)                                                                                                    |
| LR27                  | plain A → B with A in `done/`, and B —y→ A                                                                        | B's mirror omits A; no warning; `done/` untouched                                                                                                        |
| LR28                  | B's LINKS line malformed; plain A → B; B's malformed line mentions A                                              | B's mirror keeps listing A                                                                                                                               |
| LX15 _(retired)_      | `N ~T ~~T`                                                                                                        | now an incomplete draft (LP2)                                                                                                                            |
| LX17                  | `N ~T ~~T\|x`                                                                                                     | one statement T —x→ N; the plain token produces no row                                                                                                   |
| LX18                  | `@N+n ~T` where T —y→ N                                                                                           | `unchanged`; row `kind: directed`, `direction: from`, `label: y`; human `🪢 already linked "T" · y`                                                      |
| LX19                  | `@N+n ~T\|x` where T holds plain `[[N]]`                                                                          | N gains `x [[T]]`; T's entry removed; mirrors updated; `relabeled` + `moved_from`; one batch                                                             |
| LX20                  | `@N+n ~T` where N holds `see [[T]]`                                                                               | `unchanged`; label `see` kept                                                                                                                            |
| LP2 _(changed)_       | `Fix it ~~work:foo`                                                                                               | relation `from`, `kind: directed`, need `relation_label`, diagnostic `relation_label_required`; submit refuses; zero writes and mints                    |
| LP19                  | `Fix it ~work:foo` / `~work:foo\|"follow-up of"` / `~work:foo\|`                                                  | `kind` `undirected` / `directed` / `directed` (needs label)                                                                                              |
| LP20                  | `relation_to` row for a task that links here plainly / that states y here                                         | `already_linked`, `existing_kind: undirected` / `directed`, `existing_direction: from`, `existing_label: y`                                              |
| LP21                  | `relation_from` row; `capture-task-id` for an id-less `~~` target                                                 | replacement `~~note:id\|`; `relation_from_replacement` `~~note:id\|`                                                                                     |
| LP22                  | label query `See also`                                                                                            | `relation.hint` present; the label is still accepted                                                                                                     |
| LC1 _(changed)_       | one unlabelled relation                                                                                           | `🪢 links (○ T)`, no arrow                                                                                                                               |
| LC15                  | plain with X (stored on both sides) and Y (stored on Y), `follow-up of → Z`, `merged into` from W                 | one block: `(○ X) (○ Y)`, then `‹follow-up of› → (Z)`, then `‹merged from› (W)`                                                                          |
| LC16                  | plain record stored on the other task                                                                             | the same `🪢 links (○ S)` as at the storing end                                                                                                          |
| LC17                  | a pair with a plain record and a statement                                                                        | only the statement renders                                                                                                                               |
| LC18                  | a task with both a LINKS and a LINKED FROM line                                                                   | one block on the first line; the second collapsed in Live Preview and hidden in Reading view; both raw lines revealed while the selection touches either |
| LG16                  | ⇧↵, a label, ⇄, ↵                                                                                                 | stored on the other task; a same-note write is one Ctrl+Z                                                                                                |
| LG17                  | × on a plain chip at the non-storing end                                                                          | the pair unlinked; notice `🪢 Unlinked “X”`                                                                                                              |
| LG18                  | ⇧L, pick a task, empty label, ↵                                                                                   | "Make it a plain link" offered; nothing written until chosen                                                                                             |
| LG19                  | Links stage RESULTS                                                                                               | tasks related in either direction are not listed (they are in CURRENT)                                                                                   |
| LG20                  | Task Card `Links…` row                                                                                            | `3 · Audit triage +2`, counting related **tasks** from the live index (loading state, never zero, while the index loads)                                 |
| LG7 / LG9 _(changed)_ | ⇧L with a label on S / Ctrl+Shift+L row Ctrl+D                                                                    | S's LINKS gains `label [[this]]` / per decision `clear_scope`                                                                                            |

## Cross-cutting rules

- **Repos.**
  - bob-cli phases run in this repo.
  - Plugin and Mac phases must open their repo with `/sase_repo`
    (`sase repo open bob-plugins` / `sase repo open bob-mac-capture`), read that repo's
    `AGENTS.md`, and work only in the printed path.
- **bob-cli rules.**
  - Read the `docs` and `cli_rules` reference memory before editing `docs/` or CLI help.
  - Regenerate goldens and generated pages with `just docs-gen`; never hand-edit
    `docs/reference/cli/*.md` or `docs/reference/glossary.md`.
  - Gate on `just check` and `just docs-check`.
  - Per `decisions:bob-cli-releases-ride-a-daily-train`, never touch the `Cargo.toml`
    version or `CHANGELOG.md`.
  - Only `capture-grammar` carries `!` in its commit subject.
- **bob-plugins rules.**
  - Edit `src/` fragments and keep each hand-edited fragment at or below 1000 lines. Add
    new fragments through `src/fragments.json` when a file would exceed that.
  - Run `npm run build`, then `npm test`. New test files must be added to the explicit
    list in `package.json`.
  - Run `npm run docs:gen` and `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check` when
    docs or manifests change.
  - Run **`bob plugins sync`** at the end of every plugin phase.
- **Bob Mac Capture rules.** Follow its `AGENTS.md` for where to build and test:
  portable targets on athena, and the `BobMacCapture*` targets on the Mac. Linux cannot
  validate the UI. Per `decisions:mac-capture-is-a-thin-client`, the app never parses
  grammar; it renders bob's fields and applies bob's replacements.
- **Contract first.** If an implementation disagrees with a vector, fix
  `docs/task-relations.md` first in the same change, as the contract already requires.

## Phase `contract`: relation contract, glossary, and decision record (bob-cli)

Docs and memory only; no code.

1. **Rewrite `docs/task-relations.md`** so it states the model above.
   - **Opening paragraph.** Undirected unless labelled; storage unchanged.
   - **§1 Vocabulary.** Undirected relation / plain link, directed relation / statement,
     label, record, pair. Retire "directed edge", and replace "two symmetric writers"
     with the single Links stage plus the ⇧L reverse-label shortcut.
   - **§3.2 Labels.** Keep "a label reads S ‹label› T", and add the related-label hint
     (A3).
   - **§4 order.** Narrow the relabel sentence to non-empty labels.
   - **§5.** Rename to "Identity, pair rules, and minting":
     - replace "A → B and B → A are independent edges" with the pair-rules table and
       principles;
     - add the writability rule;
     - keep the minting and link-form rules.
   - **§6.** Add the projection condition and the "malformed LINKS names nobody" rule.
   - **§7.** Add `redundant_relation`.
   - **§8.** Add the pair write planner table, add-before-remove for nav, and
     all-or-nothing for capture.
   - **§9.** Ctrl+Shift+P on either relation line opens the Links stage. Ctrl+Shift+L
     opens the Links stage, and Ctrl+Shift+Alt+L opens it in reverse-label mode. The
     generated-line notice ends `· change it with Ctrl+Shift+L`. Ctrl+D follows
     `clear_scope` (below).
   - **§10.** One `🪢 links` block per task: plain chips first (canonical order, no
     arrow), then outgoing statements in authored order, then incoming statements in
     canonical order. Also cover the mount and collapse rule, the controls (＋ "Link
     another task"; × `Unlink ‹name›` on plain chips and `Remove ‘‹label›’` on
     statements, shown on hover and focus), and `api.relations` v2. Keep the inverse
     table.
   - **§11.** Add `related(path, blockIdOrLine) → {plain, outgoing, incoming}`.
   - **§12.** Add every JSON addition from "Capture grammar and JSON" above, the
     `~~note:id|` replacements, and the new shell group names. Replace the `|see`
     example labels with `|"follow-up of"`.
   - **§15.** Add every vector from the table above, mark LX15 retired, and narrow LW4.

   > [!decision] clear_scope = shown §9 and LG9 say Ctrl+D on the Links row removes
   > every relation the row shows whose record is writable (plain links from either end,
   > this task's statements, and other tasks' statements about it). Any it cannot remove
   > (read-only notes) are kept and named in the notice.

   > [!decision] clear_scope = own §9 and LG9 say Ctrl+D on the Links row removes plain
   > links (every writable record of each plain pair) and this task's own statements.
   > Other tasks' statements about this task stay, and the notice says how many were
   > kept.

2. **Vocabulary-level docs.**
   - `docs/_meta/contracts.yml`: the task-relations summary becomes "Task relations:
     undirected plain links, labelled directed statements, the generated LINKED FROM
     mirror, and `~` / `~~` capture".
   - `docs/reference/vault-format.md`: the "directed relations" paragraph says storage
     is unchanged and that meaning comes from the label.
   - `docs/concepts/projects-and-dependencies.md`: "the live LINKS and LINKED FROM
     views" becomes "the live 🪢 links view".
   - Leave capture docs and the guide to the phases that change behaviour.
3. **Memory.** Use `/sase_memory_write` for each item, then run `sase memory init` and
   `just docs-gen` so `docs/reference/glossary.md` and `docs/develop/decisions.md`
   regenerate. Link strands with `[[...]]`, as the skill describes.

   > [!decision] glossary_task_relation Rewrite `glossary:task-relation` (keep its
   > aliases) to say:
   >
   > - An inert link between two tasks: an [[undirected-relation]] when it has no label,
   >   a [[directed-relation]] when it has one.
   > - It is authored once on one task's 🪢 **LINKS:** line and mirrored on the other's
   >   generated 🪢 **LINKED FROM:** line. Which task stores a plain link carries no
   >   meaning.
   > - Add one with Capture `~` (or `~~` with a label) or the Task Card `l` gesture.
   > - It is distinct from a Pomodoro [[task-link]] and a [[task-dependency-link]]: it
   >   never blocks a task, counts as a Pomodoro Task Link, or changes status, schedule,
   >   priority, or freshness.

   > [!decision] glossary_undirected Add `glossary:undirected-relation` (aliases: plain
   > link, undirected task relation):
   >
   > - A [[task-relation]] with no label. It says only that two tasks belong together,
   >   so it has no direction.
   > - Both tasks list each other under `🪢 links`, and linking them again from either
   >   end changes nothing.
   > - Adding a label turns it into a [[directed-relation]]. Two tasks that already
   >   share a directed relation need no plain link.

   > [!decision] glossary_directed Add `glossary:directed-relation` (aliases: labelled
   > relation, directed task relation):
   >
   > - A [[task-relation]] with a label, read from its source: "S ‹label› T", such as
   >   `follow-up of`.
   > - It is stored on S's LINKS line. T shows Bob's inverse phrase when one exists,
   >   otherwise "S · ‹label› → this".
   > - Capture `~T|label` makes the captured or owner task the source, and `~~T|label`
   >   makes T the source.
   > - Clearing the label turns it back into an [[undirected-relation]]. Two tasks carry
   >   at most one directed relation in each direction.

   > [!decision] decision_record Add `decisions:relations-undirected-unless-labelled`,
   > "Relations Are Undirected Unless Labelled", shaped like the other records (Applies
   > to / Claim / Why / Rejected alternatives / Cost / Reopens when / Evidence):
   >
   > - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture, and vault.
   > - **Claim:** an unlabelled relation has no direction and a label gives it one. A
   >   labelled record subsumes plain ones. Storage stays single-authored, a plain add
   >   never removes meaning, and `~~` requires a label.
   > - **Why:** vault idioms, the Jira, Linear, Zotero, and OWL precedents, and removing
   >   a choice that carries no information.
   > - **Rejected alternatives:** the research report's "Alternatives rejected" list
   >   (copy-only, both-ends writes, a canonical owner, a direction flag or third sigil,
   >   aliasing bare `~~`, a symmetric-label allowlist, renaming LINKED FROM, a schema
   >   v2).
   > - **Cost:** pair-aware writers, a required label for `~~`, and symmetric labels
   >   rendering directionally.
   > - **Reopens when:** symmetric labels become common, two labels per direction are
   >   needed, or the storage side starts to matter to users.
   > - **Evidence:** the research artifact above and this plan's archived ref.
   >
   > Keep the roster summary to one sentence.

4. Gate: `just docs-check`. Commit as `docs(relations): …`.

## Phase `pair-core`: Rust pair model, planner, and reconcile projection (bob-cli)

1. **New module `src/native/task_relations/pair.rs`**, registered in `mod.rs`. It is
   pure, with no I/O:
   - types for one pair's records (`here` = the record on A naming B, `there` = the
     record on B naming A, each `Option<{label: Option<String>, writable: bool}>`);
   - `PairState` (unrelated / undirected / directed with forward and backward labels,
     plus which records are redundant);
   - `mirror_entry(records, side)`, the projection restricted to this pair;
   - `plan_pair_write(records, request) -> PairPlan` for the requests **Link, State,
     ClearLabel, Swap, RemoveStatement, Unlink**, following the planner table exactly.
   - `PairPlan` carries the per-side LINKS operations, the per-side mirror deltas
     (`upsert(label)` / `remove`), `action` (`added | relabeled | unchanged | removed`),
     `kind`, `moved_from: Option<Side>`, and an optional refusal.
   - Capture uses Link and State. The others exist for the JS twin's parity and are
     pinned by the same vectors.
2. **`plan.rs` guard.** In `plan_links_edit`, an `add` with `None` on an existing
   labelled entry reports `unchanged` and keeps the label (LW16). Re-adding with a
   different non-empty label still relabels (LW4). Explicit `relabel` still clears.
3. **Reconcile** (`src/native/task_status_hooks/relations.rs`):
   - **Projection.** While collecting edges, record the set of resolved
     `(source identity → target identity)` edges. In `desired_entries`, skip a plain
     source S when T's own LINKS names S. Sources whose LINKS line is uncertain
     (malformed or duplicate id) keep today's never-prune behaviour, and a malformed T
     line names nobody.
   - **Warning.** Emit `redundant_relation` per the rules in "The LINKED FROM
     projection", through the existing `RelationWarning` / `warning_message` path, so
     JSON, human stderr, and `--dry-run` all carry it.
4. **Tests.**
   - Unit vectors LW4, LW15–LW23 in `src/native/task_relations/tests.rs`, plus an
     endpoint-swap property test for plain pairs.
   - Integration vectors LR19 (rewritten), LR25–LR28, and an LR11 re-run in
     `tests/cli/task_status_hooks/relations.rs`.
   - Existing LR/LW tests keep passing except where this plan changes them.
5. **Docs.** `docs/task-status-hooks.md`, where it lists relation warning kinds and
   behaviour: add `redundant_relation` and the omission rule.
6. Commit as `feat(relations): …`. This changes no CLI contract (the warning kind is
   additive).

## Phase `capture-grammar`: capture grammar, completion, and the label-required reverse link (bob-cli)

1. **Bare `~~`.**
   - In `src/native/capture_language/` (`editor_parse.rs` needs and diagnostics,
     `dependencies.rs` extraction, `item.rs` refusals), a `~~` token without `|` yields
     `Need::RelationLabel` and the diagnostic `relation_label_required`.
   - Submit refuses it with the teaching error from "Capture grammar and JSON", before
     any staging (zero writes and mints).
   - `rewrite.rs` `attach_relation_label` already handles `|`; keep it working for `~~`.
2. **`kind`.**
   - Add it to `ParseRelation` in `src/native/capture_parse.rs`, both top-level and per
     item.
   - Add it to `PickerRelationEntry` (`capture_complete/model.rs`, built in
     `candidates.rs` `apply_relationship_picker`).
3. **Completion** (`src/native/capture_complete/candidates.rs` `relation_candidates`):
   - `already_linked` = owner→candidate stored link **or** candidate→owner stored link,
     or typed in the draft;
   - add `existing_kind`, `existing_direction`, and the `existing_label` rule to
     `RelationCandidate`, using the pair module's `PairState` over the two catalogue
     entries (`task_links_catalog` already reads every task's outgoing entries);
   - the `~~` replacement appends `|`;
   - `render.rs` text rows show `· already linked · ‹label›`.
4. **`capture-task-id`** (`src/native/capture_task_id.rs`): `relation_from_replacement`
   becomes `~~note:id|`. Update its help and test.
5. **Label hint.** Add an optional `hint` on the `relation_label` context's `relation`
   object (`capture_complete/engine.rs` `CompletionRelation`, model and serialization),
   per the list and copy above.
6. **Shell** (`capture_complete/shell.rs`):
   - groups become `links` / `reverse · needs a label`;
   - `~~` rows end in `|` with no trailing space (make `ends_in_continuation` treat a
     trailing `|` as a continuation);
   - quote or escape `|` exactly as the existing `relation labels` rows do, so the shell
     never sees a pipe.
7. **CLI help.**
   - `capture_parse.rs` and `capture_complete/cli.rs`: change the `|see` example to
     `|"follow-up of"` or `|"next step"`, and describe `kind` and the `~~` label
     requirement.
   - `capture_task_id.rs` and `capture_rewrite.rs`: update where they mention `~~`.
8. **Docs.**
   - `docs/reference/capture-syntax.md` "Relation links" (`{#tilde-link}`,
     `{#tilde-tilde-link}`, `{#tilde-existing}`, the incomplete-input row, and the
     `|see` examples).
   - `docs/capture.md`: grammar rows (~110–114) and the `capture-parse`,
     `capture-complete`, and `capture-task-id` JSON sections.
   - Then run `just docs-gen` for the goldens.
9. **Tests.**
   - `tests/cli/capture/parse_relation.rs`: LP2 changed, plus LP19.
   - `src/native/capture_complete/tests/relations.rs`: LP20–LP22, shell groups, and the
     replacement `|`.
   - Retire the LX15 test in `tests/cli/capture/relations.rs`. Bare `~~` is now refused;
     assert that.
   - Update any `~~` fixtures and tests that relied on a bare `~~` being complete (also
     `tests/cli/capture/related.rs` if affected).
10. Commit as **`feat(capture)!: …`** (the bare-`~~` refusal is a CLI behaviour break).

## Phase `capture-writes`: pair-aware capture writers and sentence-shaped output (bob-cli)

1. **Writer rework** (`src/native/capture/relations.rs`):
   - **Read the pair, then plan it.** For each resolved token, read the pair's two
     records from **staged** text: the owner's LINKS entry naming the other task (extend
     `existing_links_entry`), and the other task's LINKS entry naming the owner. Call
     `plan_pair_write` with Link (`~T`), State with the owner as source (`~T|x`), or
     State with the other task as source (`~~T|x`).
   - **Apply the plan.** Apply its LINKS operations through `plan_links_edit` (add /
     relabel / remove) on each side. Apply its mirror deltas through the existing mirror
     path (`maybe_stage_mirror` / `stage_linked_from_add`), adding a remove counterpart
     on top of `linked_from_desired_after_delta`, which already supports removal. Keep
     today's skip reasons (`disabled`, `archived_target`, `previous_daily_target`,
     `malformed_line`).
   - **Mint and batch.** Mint as today (and mint T when a move needs it). Everything
     stays in the one staged batch, with rollback on any failure.
   - **Writability.** The other task's writability follows the "Writability" rules. `~~`
     keeps `refuse_from_target_path` unchanged.
   - **Normalize each item.** Before planning, drop plain tokens for a pair that also
     has a labelled token (LX17). Keep `refuse_duplicate_labels` (LX14), keyed so that
     `~T|a ~~T|b` is allowed and `~T|a ~T|b` is refused. Same-note S and T still compose
     against staged text (LX16).
2. **JSON** (`src/native/capture/output.rs`): add `kind` and `moved_from` to
   `RelationUpdateJson`, and apply the row rules from "Capture grammar and JSON",
   including the `unchanged`-row record choice. Both `plan_capture_item` and
   `plan_dependency_only_item` already route through `plan_relation_writes`.
3. **Human output.** Rewrite `human_relation_line` to the copy table (live and dry-run).
   The locator always names the other task, which also fixes the `~~` quirk.
4. **Docs.**
   - `docs/capture.md` "Linking tasks with '~' and '~~'" (Meaning, Targets, Human
     output, JSON).
   - `docs/guides/link-related-tasks.md`: replace "Choose a direction" with "Plain link
     or labelled relation", replace the `|"related to"` example with a plain link, and
     teach `~~` as "the other task says something about this one".
   - `docs/getting-started.md` only where a step became wrong.
5. **Tests** (`tests/cli/capture/relations.rs` and output tests):
   - LX3 (rerun `unchanged`; new label `relabeled`), LX17–LX20, and LX12 dry-run parity
     for each;
   - a bare-re-add-keeps-label case;
   - human-output assertions for every copy row;
   - an all-or-nothing case where a move's second note fails its preimage check and
     nothing is written.
6. Commit as `feat(capture): …`.

## Phase `nav-writer`: Obsidian pair planner twin, writers, and relations API v2 (bob-plugins / bob-navigation-hotkeys)

1. **JS twin.**
   - Add a new fragment next to `086-relation-line-plan.js`, for example
     `087-relation-pair-plan.js`. `086` is already 793 lines.
   - It exports `pairState`, `pairMirrorEntry`, and `planPairWrite`, mirroring Rust
     `pair.rs` and the planner table.
   - Fix `planLinksEdit` for LW16.
   - Put the vectors (LW4, LW15–LW23) in a shared `scripts/relation-pair-vectors.cjs`,
     copied from contract §15 and the Rust tests (LW14 engine agreement).
2. **Writers** (`594-plugin-relation-edit.js`, `595-relation-follow-ups.js`):
   - Rebuild `applyRelationEdit` around pair requests (link, state in either direction,
     clearLabel, swap, unlink, removeStatement). Keep an adapter for today's
     `{sources, add, remove, relabel, direction}` callers until `nav-stage` replaces
     them.
   - **Same note:** one transaction, one Ctrl+Z.
   - **Cross note:** the new labelled record first, then removals and LINKED FROM
     follow-ups, best-effort, under the existing machine-write annotation and preimage
     checks.
     - A failed removal of a redundant record adds no notice suffix, because the display
       is already correct.
     - A failed mirror follow-up keeps today's `· 🪢 LINKED FROM not updated`.
   - **Plain storage:** a plain link from a read-only current task goes on the writable
     other task.
   - **Mirrors:** `planLinkedFromLine` and `mirrorRelationHandEdit` apply the projection
     rule. A hand-added plain entry whose target already names this task produces no
     mirror entry, and it drops this task's own mirror entry for that target.
3. **`api.relations` v2** (`486-relations-nav-api.js`, mounted in
   `480-review-jump-and-nav-api.js`; fix the stale "v0" comment there):
   - `version: 2`;
   - `openStage(ref, {reverse?})` (v1's `direction: "from"` maps to `reverse: true`);
   - `unlink(refA, refB)` (Unlink);
   - `removeStatement(sourceRef, targetRef)` (RemoveStatement);
   - `removeLink(sourceRef, targetRef)` kept with v1 semantics as a documented legacy
     alias.
4. **Ctrl+D** (`removeAllRelationLinksByRef`): one batch per touched note, and a notice
   that counts what was removed and what was kept.

   > [!decision] clear_scope = shown Remove every relation shown for the task whose
   > record is writable: each plain pair's writable records, this task's statements, and
   > other tasks' statements about this task. Name any kept read-only ones in the notice
   > (`kept 1 in bob_done`).

   > [!decision] clear_scope = own Remove each plain pair's writable records and this
   > task's own statements only. Other tasks' statements about this task stay, and the
   > notice says `kept N from other tasks`.

5. **Notices** (`buildRelationEditNotice` in `086`): use the copy table. The
   generated-line notice (`TASK_LINKED_FROM_LINE_NOTICE` in
   `085-relation-line-grammar.js`, and its duplicate in
   `plugins/block-id-prompt/src/061-relation-line-grammar.js`) ends
   `· change it with Ctrl+Shift+L`. Update the matching assertions in
   `test-block-id-prompt-relations.cjs`.
6. **Tests.** `scripts/test-navigation-relation-writer.cjs`: LW vectors via the shared
   file; LG4, LG9 (per the decision), LG10, LG16, and LG17 at the writer and API level;
   the add-before-remove failure path (invariant 5); and v1 `removeLink` compatibility.
   Then run `npm test` and `bob plugins sync`.
7. This phase does not change the stage UI or the Task Card rows. Existing UI keeps
   working through the adapter.

## Phase `ledger-chips`: one aggregated links block per task (bob-plugins / bob-ledger-tools)

1. **Index** (`171-relation-index-lifecycle.js` `publishRelationIndex`):
   - Add `related(path, blockIdOrLine) → {plain, outgoing, incoming}` from the **same
     generation** as `outgoing` / `incoming`:
     - `plain`: one entry per plainly related task, deduped by pair, carrying
       `storedOn: "this" | "other" | "both"` plus the usual edge fields;
     - `outgoing` and `incoming`: the labelled edges;
     - redundant plain records are hidden.
   - `api.relationIndex` stays `version: 1` with `related` additive (callers
     feature-detect it).
   - Before the first scan finishes, callers can tell "loading" from "none".
   - `outgoing()` / `incoming()` behave exactly as before.
2. **Chip model** (`146-relation-chip-model.js`, `278-plugin-relation-model.js`):
   - Replace the per-raw-line, per-direction model with a **per-task** model built from
     `related()`:
     - heading `🪢 links`;
     - plain chips first, in one wrapping row, canonical order, **no arrow** (LC1 loses
       its `→`);
     - then outgoing statements `‹label› → (T)`, in the authored order of this task's
       LINKS line;
     - then incoming statements, `‹inverse› (S)` or `(S) · ‹label› → this`, in canonical
       order;
     - one statement per visual row (LC10).
   - Keep the stale rule (LC8: dashed stored mirror entries that are no longer live),
     broken entries (`⚠ ^id not found`), `✓×N` collapse, dimming, `↗ note`, hover
     preview, `#hide` muting, and the `toggle-dependency-chips` toggle.
   - Row tooltip: "Related tasks · live".
3. **Mounting** (`278` `buildRelationChipDecorations`, `153-relation-widget.js`):
   - Render the block on the task's **first** relation line, LINKS or LINKED FROM.
   - Collapse the other relation line in Live Preview. Use a block replace decoration,
     which must come from a `StateField` because CM6 forbids block decorations from view
     plugins.
   - When the selection touches either relation line, reveal **both** raw lines.
   - **Spike this first.** If collapsing proves fragile (cursor motion, undo, folding,
     or mobile), fall back to a continuation row: the second line renders the incoming
     statements under no heading, aligned with the first line's chips. Record which path
     shipped in the plugin docs.
   - A task with incoming relations only and mirrors off still has no anchor (today's
     limitation, kept).
4. **Reading view** (`282-plugin-relation-render.js`): the same per-task block on the
   first relation `li`, with the second relation `li` hidden.
5. **Controls** (`146` `relationAttachChipRemove` / `relationAttachAddControl`, `278`
   `relationRemoveLink` / `relationOpenStage`):
   - One trailing ＋ per block (aria "Link another task") calls
     `api.relations.openStage(ref)`.
   - × on a plain chip (aria `Unlink ‹name›`) calls v2 `unlink(this, other)`.
   - × on a statement (aria `Remove ‘‹label›’`) calls v2
     `removeStatement(source, target)`.
   - × is visible on focus as well as hover.
   - With nav v1 only, fall back to v1 `removeLink` on the storing side (today's swap
     logic).
   - Without nav, show no controls.
6. **Inverse table.** Keep `relationInversePhrase` (`null` for unknown). Add
   `scripts/relation-inverse-vectors.cjs` and use it from both the ledger and nav test
   suites.
7. **Styles** (`styles.css`): reuse the `.bob-relation-*` rules. Remove the
   `is-incoming` direction styling. Colour carries status only.
8. **Docs.** `docs/bob-ledger-tools.md` and `docs/develop/plugin-apis.md` (the
   `relationIndex.related` member).
9. **Tests.**
   - `scripts/test-ledger-tools-relation-chips.cjs`: LC1 changed, LC5–LC6 under the
     single heading, LC15–LC18, and LC11 controls against fake nav APIs v1 and v2.
   - `scripts/test-ledger-tools-relation-index.cjs`: `related()`, dedupe, redundant
     hiding, and generation agreement.
   - `scripts/test-relation-index-linked-from-stage.cjs` as affected.
   - Then run `npm test` and `bob plugins sync`.

## Phase `nav-stage`: one Links row and one Links stage with a live label sentence (bob-plugins / bob-navigation-hotkeys)

1. **Task Card** (`310-task-card-previews.js`, `312-task-card-relations.js`,
   `320-task-card-header.js`, `350-picker-task-card.js`):
   - One row, `Links…` (key `l`). Its detail is `N · ‹first related task› +K`, where N
     counts related **tasks** from `related()`. It shows a loading state while the index
     loads, falls back to stored lines when the ledger is absent, and is never zero
     while loading (LG20).
   - Remove the `linked-from` row. `⇧L` stays as a key that opens the same stage in
     reverse-label mode.
   - Ctrl+D on the row calls the `nav-writer` clear (decision `clear_scope`).
   - Update the row-order tests.
2. **Hotkeys.**
   - Ctrl+Shift+L opens the Links stage.
   - Ctrl+Shift+Alt+L opens it in reverse-label mode. Keep the command id
     `task-card-linked-from` so user bindings survive, and rename its display name to
     "Links: describe this task from another".
   - Ctrl+Shift+P on a LINKS **or** LINKED FROM line opens the Links stage (`300`,
     `550`, `590`).
   - Counted `N Ctrl+Shift+L` and Tab marking keep working, plain or labelled. Counted
     reverse stays refused, with the copy updated to "Count works with Ctrl+Shift+L".
3. **One stage** (`304-relation-stage-view.js`, `305-relation-stage-copy.js`,
   `426-picker-relation-stage.js`, `427-relation-stage-plugin.js`):
   - Fold the Linked-from stage into the Links stage: remove `showLinkedFromStage`,
     `planLinkedFromStageView`, `decorateLinkedFromVaultStage`, and
     `refreshVaultLinkedFromIncoming`, or turn them into thin wrappers.
   - Title `🪢 Links · ‹task›`. Subtitle
     `Links N · searching K tasks · ⇥ to mark several · does not block`.
   - **CURRENT** comes from `related()`, with stored lines as the fallback while
     loading. Plain rows come first, then statements, rendered like the chips.
   - **RESULTS** exclude tasks related in either direction (LG19). CLOSED stays as
     today.
   - Keys:
     - RESULTS ↵ → plain link;
     - ⇧↵ → label field;
     - CURRENT ↵ → `Unlink` (plain) / `Remove ‘x’` (statement);
     - CURRENT ⇧↵ → label field for that relation (add, change, clear, or swap).
   - Footer: `↵ link · ⇧↵ link with a label · ⇥ mark several`. The CURRENT strip reads
     `↵ unlink · ⇧↵ label…`.
4. **Label field** (`showRelationLabelStage`, `getRelationLabelSubtitle`,
   `getRelationLabelPreviewItems`):
   - Title `🪢 Link with a label`. Placeholder `Label (optional) — e.g. follow-up of`.
     "Relation" becomes "Label" everywhere.
   - It shows a **live sentence**:

     ```text
     🪢 Link with a label
       This task — Retry design                       plain link · no direction
       This task — [follow-up of▏] → Retry design     ⇄ swap (Alt+R)
       follow-up of · 3    merged into · 2    next step · 1
     ```

   - ⇄ swap (Alt+R) appears only when the label is non-empty, the other side is
     writable, and no reverse statement exists. It flips which task the sentence names
     as source (LG16).
   - Reverse-label mode (⇧L / Ctrl+Shift+Alt+L) starts swapped:
     `Retry design — [label] → This task`.
   - In reverse mode, an empty label + ↵ offers a single `Make it a plain link` row and
     writes nothing until it is chosen (LG18). In forward mode, an empty label + ↵ means
     plain link, or clear label on a CURRENT row.
   - Show the related-label hint under the field for the listed labels, as bob's hint
     does.
   - Suggestions still come from `relationIndex.labelSuggestions`.

5. **Inverse table.** Make nav's `inverseRelationPhrase` return `null` for unknown
   labels. Callers render `(S) · ‹label› → this` themselves. Pin it with
   `scripts/relation-inverse-vectors.cjs`.
6. **Docs and manifests.**
   - Plugin `docs/task-relations.md` (rewrite for the model), `docs/task-card.md`, and
     `docs/develop/plugin-apis.md` (document `api.relations` v2 and drop the stale v0
     text).
   - The relation sentences in the `manifest.json` description for nav and ledger.
   - Run `npm run docs:gen`, then `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`.
7. **Tests.**
   - `scripts/test-navigation-relation-links-stage.cjs`: LG1–LG8 updated, LG12,
     LG15–LG20, and the label field sentence, swap, and reverse mode.
   - Task Card model and view tests, and `test-navigation-hotkeys-vim-counts.cjs`.
   - Then run `npm test`.
8. **Finish with `bob plugins sync`** and confirm the vault's one relation (`sase.md`)
   renders as `🪢 links (○ …)`.

## Phase `mac`: Bob Mac Capture pickers, label card, and previews (bob-mac-capture)

Line numbers below are as of `bf40f48`; re-locate them before editing.

1. **Decoding** (`Sources/CaptureCore/CaptureModels.swift`), all optional and tolerant
   (`try?`):
   - `kind` on `CaptureParseRelationshipMarker`, `CaptureRelationUpdate`, and
     `CapturePickerRelationEntry`;
   - `movedFrom` on `CaptureRelationUpdate`;
   - `existingKind` / `existingDirection` on `CaptureCompletionCandidate`;
   - `hint` on `CaptureCompletionRelation`.
   - When `kind` is missing, treat `label == nil` as undirected. Keep `direction` /
     `action` tolerant.
2. **`~~` always reaches the card.** Bob now returns `~~note:id|`. Handle these paths:
   - `acceptPickerRow`: never append a trailing space after a replacement ending in `|`;
     open the card with the label range right after `|`. This applies to plain ↩ and to
     ⌘↩/⌃↩, which must not submit a draft whose relation needs a label.
   - `openRelationLabelPromptFromPickerRow`: compute the token range without the
     trailing `|`, so the label probe never produces `||`.
   - `completeRelationIDAssignment`: the ID-less path that splices
     `relation_from_replacement` opens the card.
   - The `|` typed-trigger lane from aj.2 keeps working for both sigils.
3. **Card** (`CapturePanelModel.swift` relation-card functions, `CapturePanelView.swift`
   card view):
   - Header `🪢 Label`.
   - Live sentence: with an empty label, `“A” — “B”` with the caption
     `plain link · no direction`; with a label, `“A” — follow-up of → “B”`.
   - ⇄ **Swap** (button plus ⌥R, routed by `CaptureKeyCommandRouter` before the text
     field) is enabled only with a non-empty label. It adds or removes one `~` at the
     token's sigil, the same byte edit `flipRelationDirection` makes today, so the app
     never re-lexes.
   - **Return with an empty label on a `~~` token** rewrites it to `~` (drops one `~`
     and the dangling `|`). The sentence has already previewed this as a plain link.
     This extends aj.2's `empty_label_drops_pipe` rule.
   - Show bob's `hint` under the field.
   - Replace the static hint "Tab cycles suggestions · esc back to picker" with one that
     is true for each entry path (`|` flow vs picker flow).
4. **Pickers** (`CapturePickerPresentation.swift`, `DependencyPickerPresentation.swift`,
   `CaptureRelationshipCopy.swift`, `CompletionRowContent.swift`):
   - **`~` picker:**
     - header `🪢 Link “Owner” with…`, caption `Links`, placeholder "Search tasks to
       link";
     - footer `↩ link · ⇧↩ link with a label · ~ reverse (needs a label)`.
   - **`~~` picker:**
     - header `🪢 Which task describes “Owner”?`, caption `Reverse · needs a label`;
     - ↩ inserts `~~note:id|` and opens the card;
     - Backspace on an empty filter flips back to `~`, as today.
   - **Already-linked rows** show `Already linked · ‹label›` from `existingKind`,
     `existingLabel`, and `existingDirection`.
     - In the `~` picker, ↩ changes nothing (as today) and ⇧↩ opens the card to label
       or relabel.
     - In the `~~` picker, ↩ opens the card.
   - Row hints: "Links this task and the owner" (plain) and "Adds a labelled statement"
     (with a label).
   - Rename "link with a relation" to "link with a label" everywhere, and "Type a
     relation label…" to "Type a label, or pick a suggestion".
   - Icons: `arrow.left.arrow.right` for plain rows, `arrow.right` for statements.
   - Remove the stale "Shift-Return is a copy/key-hint this phase; the relation card is
     a later phase" comment.
5. **Previews and notifications** (`CaptureRelationshipPresentation.swift`,
   `NotificationService.swift` call sites):
   - rows and titles follow the copy table, keyed on `kind`, `direction`, and `action`;
   - a `movedFrom` row reads like a relabel;
   - the mixed summary counts "links".
6. **Similar Tasks band** (`SimilarTasksBandEdits.swift`): the `~` insertion is
   unchanged. The announcement becomes `Linked with “title”`.
7. **Fixtures and tests.**
   - Update `Tests/Fixtures/relation-*.json` and the `relation-label-*` fixtures (bare
     `~~` now needs a label; `~~note:id|` replacements; `kind` fields), and the
     `fake-bob` routes.
   - Update the tests that assert `"New task ~~tasks:source "` and the
     `“Source” → “New task”` sentence.
   - Add tests for: the `~~` pick → card; empty-label Return → `~`; swap; already-linked
     ⇧↩; decoding with and without `kind`; and the preview copy.
   - Build and test per `AGENTS.md`: `just portable-check` on athena, and the full suite
     / `typecheck-tests` on the Mac.
8. **Docs.** `docs/relationship-pickers.md`, `docs/capture-previews.md`,
   `docs/protocol.md` (field list and compatibility table: `kind`, `moved_from`,
   `existing_*`, `hint`, `relation_label_required`, and the `~~note:id|` replacement),
   `docs/keyboard.md`, `docs/capture-panel.md`, and `docs/task-pickers.md`. Then run
   `just docs-check`.

## Rollout and risks

- **Order.** Contract first, then `pair-core`. Then two tracks run independently:
  bob-cli capture (`capture-grammar`, then `capture-writes`, then `mac`) and Obsidian
  (`nav-writer` and `ledger-chips`, then `nav-stage`).
  - Version skew affects copy only.
  - Until `nav-stage` lands, the old Task Card rows keep working through the
    `nav-writer` adapter.
- **Watch for:**
  - **Symmetric labels.** If symmetric labels (`see also`, `conflicts with`) become
    common, add self-inverse rows to the display-only inverse table, then reopen the
    decision.
  - **Recurring `redundant_relation` warnings** point to a writer bug or a sync race.
    Investigate them; reconcile never auto-deletes.
  - **CM6 collapse fragility.** Use the documented continuation-row fallback.
