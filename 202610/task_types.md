---
tier: epic
title: 'Task types: Pomodoro themes become per-task types that always agree with their
  Task Links'
goal: 'Every open task has one computed task type (field, reading rule, note default,
  then plan.default_type, CHORE for Bryan). Every live Task Link agrees with its task''s
  type: type edits move links, link moves and named captures retype tasks, and Bob
  places new links by type. Explicit types render as a colored #TYPE in place of the
  #task glyph. The words "theme" and "Pomodoro theme" become "task type" everywhere
  a person reads them.

  '
decisions:
  running_retype:
    ask: When a Task Card retype hits a link in the running session, should that session
      absorb the new type?
    choices:
      absorb: Task stays in the running session; a sole link renames it, otherwise
        it gains + TYPE
      pull_out: Link leaves the running session for a planned TYPE entry, like any
        planned link
    default: absorb
    why: Pulling the task you are working on out of the session you are in would surprise
      you
    answer: absorb
  memory_glossary:
    ask: Rename the pomodoro-theme glossary strand to task-type and repoint the three
      strands that link it?
    memory:
    - glossary:pomodoro-theme
    - glossary:day-strip
    - glossary:stats-dash
    - glossary:root-parent
    default: true
    requested: We should rename the "pomodoro theme" glossary entry to "task type"
      and update it accordingly.
    answer: true
  memory_decision_records:
    ask: Add two decision records, one for the type/link agreement invariant and one
      for the computed type?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
phases:
- id: vocab
  title: Task-type words, docs/task-types.md, and the bob types command
  depends_on: []
  size: medium
  description: 'vocab: move the themes contract to docs/task-types.md with a redirect,
    make bob types canonical with bob themes as a permanent hidden alias, and switch
    user-facing CLI copy from theme to type.'
- id: storage
  title: '[type::] task fields and task_type: note defaults'
  depends_on:
  - vocab
  size: medium
  description: 'storage: read [type::] and task_type: as canonical and the theme spellings
    as legacy, emit [type::] from every writer, and extend rename, split, preservation
    tests, and the task_field vectors.'
- id: migrate
  title: bob types migrate
  depends_on:
  - storage
  size: small
  description: 'migrate: add a previewed, single-commit vault rewrite that moves open-task
    [theme::] fields and area/project theme: defaults to the canonical type spellings.'
- id: default-type
  title: plan.default_type and canonical types config
  depends_on:
  - storage
  size: medium
  description: 'default-type: add plan.default_type as the last cascade step, accept
    types: and plan.reading_type as canonical config keys, resolve new captures from
    the staged line, and set CHORE in chezmoi.'
- id: routing-core
  title: Rust routing planners and the type_mismatch lint
  depends_on:
  - default-type
  size: medium
  description: 'routing-core: add the pure agreement, retype, placement, and relocation
    planners with shared vectors, plus a read-only type_mismatch lint in bob plan.'
- id: capture-retype
  title: Named captures keep the task type in agreement
  depends_on:
  - routing-core
  size: medium
  description: 'capture-retype: replace the old theme write rule with the retype planner
    for every #NAME and =name capture form, including project-note bullets and exempt
    types such as PLAN.'
- id: capture-placement
  title: Bare captures place links by task type
  depends_on:
  - capture-retype
  size: medium
  description: 'capture-placement: route every bare link-creating capture form to
    the Pomodoro chosen by the task''s effective type, report it additively in JSON
    and previews, and refuse under plan.strict.'
- id: js-mirror
  title: Ledger Tools type model and routing mirror
  depends_on:
  - routing-core
  size: medium
  description: 'js-mirror: mirror storage, config, the cascade, and the four routing
    planners in Ledger Tools JS against the copied vectors, extend api.themes additively,
    and update every plugin-local name helper.'
- id: type-tag-rendering
  title: '#TYPE identity, ledger agreement marks, and Ledger Tools words'
  depends_on:
  - js-mirror
  size: medium
  description: 'type-tag-rendering: render explicit types as a colored #TYPE in the
    #task slot, retire the trailing pill, mark disagreeing ledger links, and rename
    Ledger Tools copy to type.'
- id: move-retype
  title: Ctrl+Shift+M Pomodoro bullet moves retype tasks
  depends_on:
  - js-mirror
  size: medium
  description: 'move-retype: make the Pomodoro bullet move picker apply the retype
    rule with Enter, Alt+Enter keep-type merges, and Tab component cycling, with consequence
    footers and checked cross-note writes.'
- id: link-picker-retype
  title: Link to today retypes and Alt+Enter keeps the type
  depends_on:
  - js-mirror
  size: medium
  description: 'link-picker-retype: replace the block-id-prompt theme write guards
    with the retype rule, turn link once into the keep-type merge, and update footers,
    toasts, and docs.'
- id: ref-today-successors
  title: bob ref today and successor offers place by type
  depends_on:
  - capture-placement
  - js-mirror
  size: medium
  description: 'ref-today-successors: place reading-task links and successor links
    by their own type in Rust and task-status-cycler, and retype on explicit ref-today
    targets.'
- id: task-card-relocate
  title: Task Card type edits relocate the live Task Link
  depends_on:
  - move-retype
  size: medium
  description: 'task-card-relocate: rename the Task Card stage to Type, allow every
    type including PLAN, and relocate the task''s live link with a consequence line,
    strict refusal, and one gesture.'
- id: gesture-retype
  title: Note moves and Pomodoro renames keep agreement
  depends_on:
  - task-card-relocate
  - capture-placement
  - type-tag-rendering
  size: medium
  description: 'gesture-retype: relocate the live link when a note move changes an
    inherited type, and retype linked tasks when naming or renaming an entry gives
    it a different type.'
- id: mac-copy
  title: Bob Mac Capture type copy
  depends_on:
  - capture-placement
  size: small
  description: 'mac-copy: render bob''s type words and the additive type placement
    label in Mac capture previews, completions, and toasts without adding any capture
    logic.'
- id: closeout
  title: Memory, config spelling, vectors cleanup, and acceptance
  depends_on:
  - migrate
  - type-tag-rendering
  - link-picker-retype
  - ref-today-successors
  - gesture-retype
  - mac-copy
  size: small
  description: 'closeout: rename the glossary strand, add the two decision records,
    switch chezmoi to types:, drop the retired write_rule vectors, sweep docs, and
    run the acceptance cases.'
proposed_by: bbugyi200.apollo.research.2t.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 16:16:32
status: wip
bead_id: bob-cli-e6
---

- **PROMPT:** [prompts/202610/task_types.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_types.md)
- **BEAD:** [bob-cli-e6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-e6/README.md)

# Plan: Task types

Pomodoro themes are task types. Bryan's vocabulary (BUILD, CHORE, FIX, MEET, PLAN, READ,
WRITE) is a set of work kinds. Tasks already carry `[theme:: X]` overrides, area and
project notes already declare `theme:` defaults, and Rust already resolves one suggested
theme per task. This epic names the concept, gives every open task a type, and adds the
missing rule: **a task's type and its live Task Link always agree.**

Context:
`research:202610/task_types_from_pomodoro_themes/task_types_from_pomodoro_themes.md`
(read it with `sase artifact read`). Bryan accepted every requirement adjustment it
recommends (R1–R14). This plan turns those into one shared contract (the **D** sections
below) and sixteen phases. Every phase implements its part of the contract and does not
re-decide it.

## Shared design contract

### D1. Words

- **Task type** (_type_) is the kind of work a task is: one ALL-CAPS name.
- **Pomodoro name** is a ledger entry's whole ` — NAME` tail.
- **Component** is one `+`-separated part of a name, keyed by `normalize_component`.
  This is unchanged (`decisions:pomodoro-theme-is-one-component`).
- **Type component** is a component that names a type. **Label** is any other component
  (D4).
- Every person-facing surface says "type": docs, CLI help and human output, warnings and
  lint messages, menus, picker footers, toasts, Task Card, Day strip, Stats labels, and
  Mac copy. "Theme" survives only as an alias in the glossary, the CLI, and config keys.
- **Rename the words, not the wire.** These keep their historical names: JSON keys and
  codes (`task_theme`, `theme_suggestion`, `ThemeRef`, `theme_not_configured`,
  `hue_source`, Stats `theme` parts), Rust/JS/Swift/CSS identifiers (`pomodoro_themes`,
  `api.themes`, `bob-pomo-theme-*`), the `.bob/themes/<host>.jsonl` journal and its
  event names, and the `PT1`–`PT13` section ids in the contract doc. JSON `message`
  strings may change wording.
- Never use "type" alone where it could mean a SASE bead `task_type`, a note's `type:`
  frontmatter (`[[area]]`/`[[project]]`), or a Dataview value type.

### D2. Storage and config

- **Task field.**
  - Canonical: `[type:: X]` / `(type:: X)`. Legacy: `[theme:: X]` / `(theme:: X)`, still
    read.
  - Any two type-ish fields on one line (either key, same or different value) are the
    `duplicate` repair state. One invalid field is `invalid`. Both fall through to
    inheritance (unchanged semantics).
  - Every gesture writer emits `[type:: NAME]` and removes every `type` and `theme`
    field. Placement is unchanged (before the trailing inline-field/tag/`^id` run). A
    semantic no-op never rewrites bytes.
  - `bob types rename` rewrites values only and keeps each field's key spelling;
    spelling changes are `bob types migrate`'s job.
- **Note default.**
  - Canonical: scalar `task_type: X` on area/project notes. Legacy: `theme: X`, still
    read. Never `type:`, which already names the note kind.
  - When both keys are present, `task_type:` wins. A different legacy value adds one
    `type_default_conflict` warning, through the same channel as
    `theme_default_invalid`.
  - `off` stops the walk (either key). The cascade then continues to `plan.default_type`
    (D3).
- **Config** (`~/.config/bob/config.yml`; Bob reads it and never writes it):
  - `plan.default_type`: a single valid name, `off`, or null. Unset, null, and `off` all
    mean no global default, which is today's behavior. A name outside a configured
    vocabulary still applies; `bob plan` reports it with a `default_type_not_configured`
    lint.
  - `types:` is the canonical vocabulary block. `themes:` is a permanent legacy alias
    with the same shape and validation. Both present is an invalid block (the existing
    non-fatal "drop the block and warn" path), and the message names both keys.
  - `plan.reading_type` is canonical. `plan.reading_theme` is a legacy alias. Both
    present is a plan config error naming both keys.
  - `plan.max_themes`, `plan.exempt`, `plan.inventory_labels`, and `plan.strict` are
    unchanged.
- **Closed tasks** keep whatever field they have as history. Nothing backfills or stamps
  a default onto any line.
- **Dataview caveat.** Dataview merges list-item inline fields into page fields, so page
  `type` on a note with typed tasks can become a list. Bob reads note kinds from
  frontmatter and Bases read `note.type`, so neither is affected. No vault Dataview
  query filters on page `type` today (verified 2026-10-29). Raw Dataview
  `task.type`/`task.theme` is not an effective-type API; point readers at
  `bob types suggest`.

### D3. Effective type

One cascade, first match wins. Rust is the authority; the Ledger Tools JS mirror serves
unsaved editor buffers; shared vectors keep them equal.

1. **Task field** — source `task`, label `this task`.
2. **Reading task** — `plan.reading_type` (default READ; `off` skips this step). Source
   `reading`, label `reading task`.
3. **Nearest note default** on the residence chain. A set `[root:: X]` field starts the
   walk at X. Daily notes have no walk of their own. Source `note`, label `from <stem>`.
4. **`plan.default_type`** — source `default` (additive enum value), label
   `default type`.
5. None. This happens only when no default is configured.

- Closed tasks have no suggestion (unchanged).
- New captures resolve from the **staged** task line after parsing. A new `#ref` task is
  therefore READ, and an authored `[type:: FIX]` is honored.
- Choosing a task's inherited type anywhere clears its field, so fields hold only
  exceptions.
- **Exempt** (`plan.exempt`, default PLAN) is a budget attribute only. PLAN is an
  ordinary type in fields, defaults, menus, placement, moves, and the Task Card. It is
  not counted by the budget, and its hue is muted.

### D4. Type components and labels

`type_components(name)` classifies each component of a Pomodoro name:

- **Configured vocabulary present** (`types:` with at least one entry): a component is a
  type component iff its key is in the vocabulary. Everything else is a label:
  `plan.inventory_labels`, ad-hoc names such as SNACK, and project names used as session
  names.
- **No vocabulary**: every component except inventory labels is a type component.
- Unnamed entries and `(unnamed)` have no components.
- An entry with no type components is a **container**. It holds any task without typing
  it.
- Ad-hoc names stay fully supported as names, with the existing warning reworded:
  `SNACK isn't one of your types; tasks here keep their own types`.

### D5. Agreement invariant

> Every live Task Link sits under a Pomodoro whose name includes its task's type, or
> under a container. Whichever side a deliberate gesture changes, Bob updates the other
> side in the same write.

- A _live_ link is a dedicated Task Link under an open Pomodoro in today's daily note.
  Capture already treats more than one movable link as a write-free invariant error, so
  each task has at most one.
- `agrees(task_type, entry)` is true when any of these holds:
  - the entry is a container;
  - the task has no effective type;
  - the task type's key equals **any** component key of the entry (labels included, so a
    SNACK-typed task under `— SNACK` agrees).
- Drift that no gesture caused (hand edits, frontmatter or config edits, sync merges) is
  **flagged and never silently repaired**: a `type_mismatch` lint in `bob plan`, plus a
  ledger mark in Obsidian.
- Closed Pomodoros are history. No rule edits them, and Stats keeps attributing past
  sessions to the names they had.
- Type-only edits never stamp freshness, change lanes, consume dependency guards, or
  advance the review walk.
- Existing link guards keep their meaning: Blocked tasks stay unlinked,
  `capture-link-drops-blocking-deps` applies, and successor links still need explicit
  selection.

### D6. Retype rule: a link lands in Pomodoro D

Pure `retype(field, effective, inherited, D, selector, keep_type)`:

1. D is a container → **no change** (`keeps BUILD`).
2. The task's type is any component of D → **consistent**, nothing written.
3. `keep_type` (Alt+Enter in Obsidian pickers only) → **merge**. D's name gains
   ` + TYPE` as a new last component. No field change.
4. Otherwise retype to one type component X of D. X is chosen as follows:
   - the component the authored selector names (capture `#fix` into `BUILD + FIX` → FIX,
     by slug/prefix against components);
   - else the Obsidian Tab-cycled choice;
   - else the first non-exempt type component;
   - else the first type component.

   Then: if X equals the inherited key, **clear** the field; otherwise **set**
   `[type:: X]`, catalog-spelled.

- There is no "suggestion shown" guard and no "single non-exempt component" guard, so
  PLAN is a legal result.
- Capture `#NAME` and `=name` always take the Enter path; capture has no Alt+Enter.
- The retired "link once" behavior was exactly the inconsistent option, so it is gone.

### D7. Placement rule: Bob picks the Pomodoro for type T

`place(T)` extends `preselect::choose`:

- An already-queued task (it has a live link) keeps its queue for bare link forms. A
  mismatching queue is the lint's job. Ensure Next `@route+id` is the one exception: it
  relocates its link to `place(T)` (see `capture-placement`).
- Otherwise take the first of these that exists:
  1. a running exact entry;
  2. the first open exact entry in ledger order (exact means the components, after
     removing exempt keys other than T, are exactly {T});
  3. the first open entry containing T;
  4. a new `— T` placeholder at the existing named-creation insertion point.
- Unlike menus, placement may create entries for unknown or unconfigured names, with the
  `theme_not_configured` warning.
- With `plan.strict: true`, a creation that adds a type past `plan.max_themes` is
  **refused before any write**. The teaching error names the type, the cap, and the
  `#NAME` escape. With `strict: false`, Bob warns and proceeds.
- No effective type (no default configured) → today's implicit current/next selection,
  unchanged.
- Timed ambiguity behaves exactly as for named creation: an existing match resolves, and
  creation with more than one open timed entry is the existing invariant error.

### D8. Relocation rule: a task's type becomes X

Pure `relocate(entries, live_link, X, budget)`:

1. No live link, or the link's entry already agrees with X → **ledger unchanged**. A
   backlog task never gains a link, so no new Today membership is created
   (`decisions:today-is-read-from-the-ledger`).
2. The link's entry is running:

> [!decision] running_retype = absorb **Running keeps it.** If the task is the entry's
> sole Task Link, replace the entry's type components with X (labels kept, X in the
> first type component's position): `— FIX` → `— BUILD`, `— BOB + FIX` →
> `— BOB + BUILD`. Otherwise the entry gains ` + X`. The timespan never changes.

> [!decision] running_retype = pull_out **Running lets it go.** Treat the running entry
> like a planned one (step 3 onward). The running entry is never deleted, even when
> emptied, because timed entries are never deleted.

3. An existing planned (open, not running) entry that `place(X)`'s steps 1–3 would
   choose, skipping running entries → **move** the link's whole subtree (work-log
   children included) there.
4. The task is the sole link of its planned entry → **rename that placeholder** in
   place: type components replaced by X, labels kept.
5. Otherwise → **create a `— X` placeholder directly below the source entry's block**
   and move the subtree into it.

- An emptied _placeholder_ is deleted by the existing cleanup rule; a timed entry never
  is.
- With `plan.strict: true`, a creation past the cap refuses the whole edit: neither the
  field nor the link changes.
- Full duplicate-type Pomodoro management is out of scope. Among duplicates, ledger
  order decides.

### D9. Who follows whom

| Gesture                                               | Changes                         | Bob restores agreement by                          | Phase                                |
| ----------------------------------------------------- | ------------------------------- | -------------------------------------------------- | ------------------------------------ |
| Task Card Type stage, Use default, Ctrl+D on Type     | the type                        | relocating the live link (D8)                      | `task-card-relocate`                 |
| Task move to another note (Obsidian, `bob task move`) | the inherited type, if no field | relocating the live link (D8)                      | `gesture-retype`                     |
| Ctrl+Shift+M on a Pomodoro sub-bullet                 | the link's entry                | retyping each moved task (D6)                      | `move-retype`                        |
| Link to today (Ctrl+Shift+Enter, `]s` landings)       | the link's entry                | retyping (D6)                                      | `link-picker-retype`                 |
| Capture `#NAME` / `=name` forms (zsh and Mac)         | the link's entry                | retyping (D6)                                      | `capture-retype`                     |
| Bare capture forms, project `:id` bullets             | Bob chooses the entry           | placing by type (D7)                               | `capture-placement`                  |
| `bob ref today`, successor picks                      | Bob chooses the entry           | placing by type (D7); explicit targets retype (D6) | `ref-today-successors`               |
| Naming or renaming an entry that holds links          | the entry's types               | retyping its tasks, count previewed (D6)           | `gesture-retype`                     |
| Ctrl+X merge, Alt+Enter keep type                     | adds types                      | nothing to do                                      | —                                    |
| Close, roll, rollover, reconcile, lanes, freshness    | neither side                    | nothing                                            | —                                    |
| Hand edits, frontmatter/config edits, sync            | either side                     | `type_mismatch` lint and ledger mark only          | `routing-core`, `type-tag-rendering` |

### D10. Pinned copy

Consequences are always stated before Enter, and silence is a bug (Bryan's closed
`^bug-task-card-themes`). Use these strings verbatim. Adapt only counts and names.

- **Picker footers** (move and link). `⇥ …` appears only for a composite with two or
  more candidate type components:
  - `→ FIX · task becomes FIX · dev suggests BUILD · ⌥↵ keep BUILD (FIX + BUILD) · ⇥ BUILD`
  - `→ FIX · 2 tasks become FIX · ⌥↵ keep types (FIX + BUILD + WRITE)`
  - `→ BUILD · type cleared · from dev`
  - `→ BUILD · keeps BUILD`
  - `→ LATER · keeps BUILD`
- **Move and link toasts:**
  - `Moved 2 bullets to FIX · 1 task now FIX`
  - `Linked under FIX · type FIX`
  - `Linked under BUILD · type cleared`
  - `Moved to FIX + BUILD · keeps BUILD`
- **Task Card consequence line** (under the selected row):
  - `link moves FIX → BUILD`
  - `running FIX becomes FIX + BUILD` / `running FIX becomes BUILD`
  - `FIX placeholder becomes BUILD`
  - `new BUILD Pomodoro below FIX`
  - `no live link · ledger unchanged`
  - `already under BUILD`
  - strict: `plan.strict: BUILD would be type 4 of 3 · nothing written`
- **Task Card toasts:**
  - `Type BUILD · link moved to BUILD`
  - `Type cleared · BUILD from dev · link moved to BUILD`
  - partial: `Type BUILD set · link not moved — Ctrl+Shift+M to move it`
- **Capture human rows:**
  - type row: `type   → FIX · dev suggests BUILD` /
    `type   cleared · dev suggests BUILD`
  - the destination row gains ` · <label>`: `→ under BUILD (running) · from dev`,
    `→ new CHORE Pomodoro · default type`
- **Mac** previews mirror these: `type → FIX · …`, `would set type → FIX · …`,
  `type cleared · …`, `would clear type · …`.
- **Lint:**
  `BUILD task under FIX · sase#^fix-x · Ctrl+Shift+M to move or Ctrl+Shift+P t to retype`.
- **Labels:** `SNACK isn't one of your types; tasks here keep their own types` and
  `LATER is a label · tasks keep their own types`.
- Never put `::` in tooltip or aria text, because Dataview rescans rendered text.

### D11. Rendering (`#TYPE`)

The stored line keeps `#task` and its optional `[type:: X]`. Ledger Tools draws the
identity slot as **glyph plus type** only when the task has an explicit, valid field.

```text
- [ ] ⌗ Restart athena!                        inherited BUILD: hash only; tooltip "BUILD · from dev"
- [/] #FIX Significantly speed up sase commits explicit FIX in FIX ink; [type:: FIX] folded away
- [*] #PLAN Break the epic into phases         exempt or auto-muted: muted ink
- [ ] 📖 Harness Engineering                    reading task, no field: book unchanged
- [ ] 📖BUILD Port the harness idea             reading task with explicit BUILD: book plus name
- [ ] #SNACK Buy oat milk                       unconfigured: neutral ink, dotted underline
- [ ] #? Fix the gate                           repair state: warning-orange mark; raw fields visible
```

- **Ink.**
  `--bob-type-ink-<slot>: color-mix(in srgb, var(--bob-hue-<slot>) 55%, var(--text-normal))`
  for the eight palette slots. This clears 4.5:1 for every slot in both themes; today's
  72% teal ratio fails for yellow, magenta, and aqua. Add a contrast test over 8 slots ×
  {plain, Blocked tint, In Progress tint, Next tint} × {light, dark}, all ≥ 4.5:1.
- **Typography.** Keep the hash geometry. The name is set in `var(--font-interface)`,
  about 0.8em, weight 600, letter-spacing 0.02em, on the text baseline. No resting
  background (Bryan asked to remove the chip). Hover and focus show the existing faint
  capsule.
- **States.**
  - Closed tasks use the existing fade.
  - Exempt and `auto`-muted hues use muted ink.
  - Unconfigured names use neutral ink with a dotted underline.
  - Repair uses `#?`.
  - `#gtd #pre`/`#post` glyphs keep priority and are followed by the name, like the
    book.
- **Interaction.** Moving the cursor into the mark reveals `#task`; the folded field
  reveals `[type:: X]`. Clicking never writes. Tooltip:
  `FIX · set on this task · dev suggests BUILD · Ctrl+Shift+P t to change`. Inherited
  types appear in the tooltip of the plain glyph as `BUILD · from dev`. Aria label:
  `#task · type FIX`. In rendered views, put `data-bob-type` on the existing `a.tag` and
  draw the name with `::after`, so the post-processor creates no nodes. Widget `eq()`
  keys include type key, hue, and state.
- **Retired.** The trailing explicit-field pill on task lines retires. Pills stay where
  names are _session_ names: the ledger, pickers, Task Card, Day strip, and Stats. Hide
  Dataview's `type` inline-field span exactly as the `theme` span is hidden today.
- **Toggle.** Add a `Toggle inherited task types` command, off by default. It renders
  inherited types as `#BUILD` in a quieter ink (about 60% of explicit weight). Tasks
  query rows keep hiding the hash.
- **Ledger marks.**
  - A live Task Link that disagrees (D5) shows `#BUILD` before the link with a dashed
    warning-orange underline. Tooltip:
    `BUILD task under FIX · Ctrl+Shift+M to move · Ctrl+Shift+P t to retype`.
  - In an entry with two or more type components, agreeing links show a quiet,
    underline-free `#BUILD` prefix, so a merged `BUILD + FIX` project session reads at a
    glance.
- **Performance.** Use cached resolver results with batched invalidation. Never spawn
  `bob` per row. With stale data, show the generic glyph rather than a confidently wrong
  type.

### D12. Write safety

- **Rust capture** commits the field, task line, and ledger edits in its existing single
  staged transaction. A batch sees earlier staged creations, so two captures that both
  need CHORE create one placeholder.
- **Obsidian** gestures use the existing cross-note pattern:
  `readDeferredPomodoroSnapshot` / `writeDeferredPomodoroCleanup` /
  `prepareDependencyTargetNote`.
  - Open files go through the Editor API, with one undo step per open editor. Closed
    files go through `vault.process` with a preimage check.
  - Preimages are rechecked immediately before each write. A changed buffer means replan
    or refuse; never overwrite.
  - The gesture's primary side is written first: the ledger for moves, the field for
    Task Card edits. Then the other side is written as a checked step.
  - A partial failure shows the pinned partial toast and leaves the lint and mark to
    show the drift.
  - A repeated gesture whose post-state already holds is a no-op.

### Out of scope

- Full duplicate-type Pomodoro management (Bryan's `^same-type-pom-dups` vault task).
- Counting open Pomodoros instead of themes (`^count-open-poms-not-themes`). Do it next,
  because type placement makes "over budget" routine.
- Stats by task type (time through links), AI classification, per-project type
  registries, per-link work-mode overrides, and any wire or identifier rename.

## Rollout rules for every phase

- **bob-cli** ships only through the daily release train
  (`decisions:bob-cli-releases-ride-a-daily-train`). Never edit the `Cargo.toml` version
  or `CHANGELOG.md`. Use `!` only where a phase below says it breaks a contract.
- Read `cli_rules.md` and `docs.md` memory (`/sase_memory_read`) before touching CLI or
  docs. Run `just check` and, for docs, `just docs-gen` plus `just docs-check`.
- **bob-plugins** phases follow that repo's `AGENTS.md`: edit `src/` fragments, keep
  each at or under 1000 lines, `npm run build`, `npm test` (register new test files in
  `package.json`), and finish with the `bob plugins sync` deploy it prescribes.
  - Phases that touch the same plugin are serialized, because each regenerates that
    plugin's `main.js`.
  - Deployed plugins may run ahead of the installed `bob` until the next release. This
    is safe: old `bob` preserves `[type::]` bytes as an unknown field.
- **Shared vectors.** `tests/fixtures/pomodoro_themes/vectors.json` stays the single
  source. A phase that adds cases after `js-mirror` copies the file byte-for-byte into
  both bob-plugins locations
  (`plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/` and
  `scripts/fixtures/pomodoro_themes/`), updates the README pin, and runs the JS vector
  tests in the same phase.
- **bob-mac-capture** follows `decisions:mac-capture-is-a-thin-client` and runs its
  `just` checks (`just portable-check` without Xcode).
- **Never write Bryan's live vault or run `chezmoi apply`.** Use temp vaults and
  fixtures. Edits to the chezmoi repo are committed for Bryan to apply.

## Phase `vocab`: Task-type words, docs/task-types.md, and the bob types command

Repo: bob-cli.

- **Docs move.**
  - `git mv docs/pomodoro-themes.md docs/task-types.md`. Add
    `redirect_maps: {pomodoro-themes.md: task-types.md}` to the empty `redirects` plugin
    in `mkdocs.yml`; `tools/docs/bob_docs/lint.py` already reads `redirect_maps`.
  - Update the nav entry ("Task types"), the llmstxt list, and every inbound link under
    `docs/`.
  - Retitle the page "Task types". Rewrite its intro and PT1 to define D1 and D4's
    words.
  - Elsewhere, rename the concept word in place: "task type"/"type" for the per-task
    concept and for type components, "Pomodoro name" for the whole tail. Keep the
    `PT<n>` ids and add one line: "PT numbering is historical."
  - Behavior text stays accurate to what ships today. Later phases rewrite their own
    sections.
- **CLI.**
  - Register `bob types` as the canonical group with the same members, in alphabetical
    help order, in the Daily workflow section after `stats`.
  - Make `bob themes` a permanent hidden alias with byte-identical behavior and no
    deprecation output. The root `ALIASES` table and its "alias points at a leaf" test
    (`runner.rs`) need a group-alias form, or one alias per member. Choose whichever
    keeps `bob themes <member> …` and bare `bob themes` byte-identical, and prove it
    with tests.
  - Rename `COMMAND_NAME` constants, help, `after_help`, completion groups, and command
    catalog rows to `bob types …`. Vault-rewrite commit subjects become
    `bob types rename: OLD → NEW` and `bob types split: …`. Update the undo runbooks.
- **Human copy.**
  - `bob types list` header: `TYPES  7 configured · 81 known · 4 today · …`.
  - Meta labels: `muted · not one of your types` for unconfigured rows.
  - Hints and rename/split previews say "type".
  - The `theme_not_configured` message becomes `SNACK isn't one of your types` (plus the
    existing `; did you mean CHORE?`).
  - zsh completion descriptions: `your type · never used`,
    `new · not one of your types`.
  - Plan lint and capture warning text say "type".
  - JSON keys and codes are unchanged (D1).
- **Generated docs.** Run `just docs-gen`. It should produce
  `docs/reference/cli/types.md`; redirect `reference/cli/themes.md` to it.
- **Tests.**
  - Alias parity: identical stdout, stderr, and exit for `themes list -f json` vs
    `types list -f json`, for `themes rename … -d`, and for bare `themes`.
  - Help snapshots and completion tests.
- **Commit.** `feat(types): rename bob themes to bob types and adopt task-type words`.

## Phase `storage`: `[type::]` task fields and `task_type:` note defaults

Repo: bob-cli. Implements D2's storage rules.

- **`src/native/task_theme.rs`.**
  - `parse_task_theme` reads `type` and `theme` (`inline_fields` for both keys). Two or
    more fields across both keys means `Duplicate`.
  - `set_task_theme` writes `[type:: NAME]` after removing both keys; `clear_task_theme`
    removes both.
  - Keep the function names (wire-and-identifier rule) and update the module docs.
- **`src/native/pomodoro_homes/facts.rs`.** `parse_note_theme` reads `task_type:` first,
  then legacy `theme:`. Expose the conflict so `suggest.rs` emits
  `type_default_conflict` once per declaring note. Check the other `NoteFacts.theme`
  consumers (`defaults.rs`, `coverage.rs`) for both keys.
- **Rename and split.**
  - In `coverage.rs`, rename rewrites `[type::]` and `[theme::]` field values, keeping
    each key's spelling, and rewrites both `task_type:` and `theme:` default values. It
    keeps the `frontmatter_theme_value_span` quote rules.
  - Split's `split_leaves_task_themes` leftover counts include both spellings.
  - Rename's verify step ("differs only in the value span") still holds.
- **Preservation.** Extend each test that keeps a distinctive `[theme::]` byte-identical
  with `[type::]` and mixed cases. The list:
  - `task_move/tests.rs`, `freshness/placement_tests.rs`, `collect_done/tests/plan.rs`
  - `task_complete/tests/tree_tests.rs`, `task_cancel/tests.rs`
  - `task_status_hooks/reconcile/fields.rs`, `ref_tasks/edit.rs`
  - `highlights_ref/rename/links.rs`, `highlights_ref/tests/reading_execute.rs`
  - `task_root.rs`, plus the CLI `themes_rename`, `themes_split`, `themes`, and
    `ref_library/today` tests.

  Freshness stamps still land after the type field.

- **Vectors.** Add `task_field` cases to `tests/fixtures/pomodoro_themes/vectors.json`:
  type set, legacy set, mixed duplicate, paren form, set replaces legacy, clear removes
  both.
- **Docs.** In `docs/task-types.md`, update PT13's note defaults, task fields,
  placement, preservation, rename coverage, and the Dataview caveat (D2). Also update
  `docs/vault-format.md` and the PT10 scope wording.
- **Commit.**
  `feat(types): store task types as [type::] and task_type: with legacy theme reads`.

## Phase `migrate`: bob types migrate

Repo: bob-cli.

- **Command.**
  `bob types migrate [-b/--bob-dir DIR] [-d/--dry-run] [-f/--format human|json] [-o/--offline]`,
  a new `bob types` member in alphabetical help order.
- **What it rewrites.**
  - Every **open** task's single valid legacy field becomes `[type:: X]`, or
    `(type:: X)` for paren form, with the value spelling kept.
  - Every area/project legacy `theme:` default becomes `task_type:`, keeping value bytes
    and quote style. When `task_type:` already exists, the legacy line is removed only
    if its key matches; otherwise the note is reported as `conflict` and left alone.
  - Closed tasks, `done/`, repair-state lines, prose, fenced examples, `.bob/`, and
    `_conflicts/` are never touched. Repair lines are listed as `needs_repair`.
- **Transaction.** Reuse `vault_rewrite::run` with the same lock, pre-sync,
  `vault_mismatch`, stale-note, verify, single commit, and post-sync flow as rename. The
  commit subject is `bob types migrate: N fields, M defaults`. Verify that every changed
  line differs only in the key span. When nothing is legacy, refuse with `unchanged`
  (exit 1, no commit). Undo is `git revert`.
- **Output.** The preview lists per-file counts. JSON is
  `{schema_version: 1, ok, fields, defaults, files: [{path, fields, defaults}], needs_repair, conflicts}`.
- **Tests.** Temp-vault CLI tests: dry-run byte-identity, write plus verify, idempotence
  (a second run is `unchanged`), conflict and repair reporting, refusals.
- **Docs.** A new section in `docs/task-types.md` with the runbook ("run `-d`, then the
  write; undo with `git revert` and `bob vault-sync`"), plus `just docs-gen`.
- **Commit.** `feat(types): add bob types migrate`.

## Phase `default-type`: plan.default_type and canonical types config

Repos: bob-cli, chezmoi.

- **Config parsing.**
  - In `src/native/config/plan.rs`, add `default_type`, following the
    `parse_reading_theme` pattern (name, `off`, or null).
  - Accept `reading_type` with `reading_theme` as the legacy alias; both present is a
    `ConfigError` naming both keys.
  - In `config/mod.rs` `RawConfig` and `config/themes.rs`, accept `types:` with
    `themes:` as the legacy alias. Both present goes through the existing invalid-block
    warning path.
  - Update the parse and error tests.
- **Cascade.**
  - In `pomodoro_themes/suggest.rs`, add `SuggestionSource::Default` (`default`, label
    `default type`) as D3 step 4. It applies to `resolve_without_task_field`, to
    `note_default`'s `off` and no-default exits, to daily-note residences, and to new
    tasks.
  - `inherited` for a set field now includes the default.
- **New tasks.** `suggest_for_new_task` and capture's `apply_new_task` path resolve from
  the staged new line, so reading tasks and authored fields count. Keep
  `suggest_for_new_task` for bare-route explanations in `bob types suggest`.
- **`bob types suggest`.** Human output prints `■ CHORE  suggested · default type`. Task
  overrides print `without it: ■ CHORE default type`. JSON is additive.
- **Lint.** `bob plan` reports `default_type_not_configured` when the default is outside
  a configured vocabulary.
- **Vectors.** Add `suggest` cases: default fallback, `off` falls to the default,
  daily-note task, reading task resolved from the staged line, and the no-default
  regression (none).
- **Docs.** `docs/reference/configuration.md` (`plan.default_type`, `plan.reading_type`,
  `types:`), and `docs/task-types.md` PT12/PT13 precedence and copy.
- **chezmoi.** In `home/dot_config/bob/config.yml`, add
  `default_type: CHORE # tasks with no field, reading rule, or note default are CHORE`
  under `plan:`.
  - Unknown plan keys are ignored by the current Rust parser; confirm with the existing
    `ignores_unknown_keys` test.
  - Before committing, also confirm that the deployed Ledger Tools `coercePlanCaps`
    (`bob-plugins/plugins/bob-ledger-tools/src/050-plan-budget.js`) ignores unknown
    keys. If it does not, leave chezmoi to `js-mirror` and record a
    `PROPOSED FOLLOW-UP:` on this phase's bead.
  - Keep `themes:` there; `closeout` switches the spelling.
- **Commit.** bob-cli: `feat(types): add plan.default_type and canonical types: config`.
  chezmoi: `bob: default untyped tasks to CHORE`.

## Phase `routing-core`: Rust routing planners and the type_mismatch lint

Repo: bob-cli. Pure planners only; no capture behavior changes yet.

- **Module.** Add `src/native/pomodoro_themes/routing.rs`, next to `preselect.rs`, with:
  - `type_components` (D4), over `config::themes::ThemeSources` (vocabulary, exempt,
    inventory);
  - `agrees` (D5);
  - `retype` → `Container | Consistent | Merge { name } | Set { name, key } | Clear`
    (D6), including selector-to-component matching by slug or prefix;
  - `place` → an entry, a create, or a strict refusal (D7), wrapping `preselect::choose`
    with placement's create-for-unknown and strict semantics;
  - `relocate` →
    `Unchanged | RenameEntry | AppendComponent | MoveTo | RenamePlaceholder | CreateBelow | Refused`
    (D8), implementing the accepted `running_retype` branch.

  Leave `plan_theme_change` in place for now; `capture-retype` replaces its caller.

- **Vectors.** Add `agree`, `retype`, `place`, and `relocate` sections, each case as
  `{name, input, expected}` over plain data (entry names, open/running flags, link
  counts, vocabulary, exempt, inventory, cap, strict). These sections are the JS
  mirror's contract. Cover:
  - containers, labels, and composites;
  - exempt PLAN as a target;
  - the selector pick and the clear-on-inherited case;
  - the running sole/non-sole cases and placeholder rename;
  - create-below and the strict refusal;
  - unknown-type creation;
  - no-vocabulary mode.
- **Lint.** In `bob plan`, compute effective types where `plan_budget/today.rs` resolves
  today's linked tasks. Emit one `type_mismatch` lint per disagreeing live link:
  `{code, message, link: {route, block_id}, entry: {line, name}, task_type: ThemeRef, entry_types: [ThemeRef]}`.
  - The human line uses D10's lint copy.
  - It is read-only, with nothing repaired.
  - Keep the extra resolver cost inside `bob plan`'s budget by reusing one
    `HomeResolver` per run.
- **Docs.** In `docs/task-types.md`, add sections for agreement, retype, placement, and
  relocation (D4–D8 in contract form, including D10 copy). Add the lint to
  `docs/plan.md` and to `docs/reference/json-outputs.md`.
- **Commit.** `feat(types): add task-type routing planners and type_mismatch lint`.

## Phase `capture-retype`: Named captures keep the task type in agreement

Repo: bob-cli. This is a capture behavior break.

- **Write rule.** In `src/native/capture/task_theme.rs`, `apply` calls `routing::retype`
  in place of `plan_theme_change`, passing the resolved destination name and the
  authored selector text. Explicit `#NAME`, `#"NAME"`, and `=name` are always
  deliberate. Delete `plan_theme_change` and `single_non_exempt_component` once nothing
  calls them; keep the `write_rule` vectors until `closeout`.
- **Forms covered.**
  - new task `<text> @route:id#NAME`
  - whole-item `@route:id#NAME` / `^route:id#NAME`
  - Ensure Next `@route+id#NAME`
  - the `=name` shorthands
  - project-note bullets with a name: `capture/project_note.rs` currently hardcodes
    `task_theme: None` and now applies retype per linked task.

  Link guards still win: a guarded (Blocked) target writes no field.

- **Labels.** A `#SNACK` or `#LATER` destination keeps the task's type and prints D4's
  label copy as a capture warning.
- **Output.** The `task_theme` JSON shape is unchanged (`set`/`cleared`, omitted on no
  change). The human type row uses D10 (`format_task_theme_row` in `capture/output.rs`).
  Dry-run reports the same post-image.
- **Tests.** CLI capture tests for these acceptance cases:
  - inherited-BUILD task plus `#fix` writes `[type:: FIX]`;
  - `#build` on a task with a FIX field clears it;
  - `#plan` writes `[type:: PLAN]`;
  - `#later` keeps the type;
  - `#fix` into an existing `BUILD + FIX` keeps a FIX task, and retypes a WRITE task to
    FIX;
  - batch staging;
  - dry-run parity.
- **Docs.** Update `docs/capture.md` (the explicit-`#NAME` paragraphs in the
  named-target section, the existing-task link section, and the `task_theme` JSON
  section) and replace the write-rule section in `docs/task-types.md`.
- **Commit.** `feat(capture)!: named Pomodoro captures keep the task type in agreement`.

## Phase `capture-placement`: Bare captures place links by task type

Repo: bob-cli. This is a capture behavior break.

- **Shared selection.** Add one helper that every link-creating path calls instead of
  `select_implicit_open_entry` (`capture_task_toggle/links.rs`). It resolves the task's
  effective type (D3, from the staged line) and runs `routing::place`. When there is no
  type, it falls back to the legacy implicit selection. Creation reuses the
  named-creation insertion (`new_pomodoro_insertion_index` / `insert_named_placeholder`
  / `plan_created_named_entry`). Call sites:
  - `links.rs` `plan_link_insertion`
  - `capture_task_toggle/ledger.rs` (the no-name branch)
  - `relocation.rs` `resolve_relocation_destination`
  - `capture/pomodoro_insert.rs` (`PomodoroSelection::CurrentOrFuture`)
  - `capture/project_note.rs`
- **Forms.**
  - New task `<text> @route:id` and `- text :id` project bullets: place by type.
  - Whole-item `@route:id` / `^route:id`: a queued task keeps its queue; otherwise place
    by type.
  - Ensure Next `@route+id` relocates to `place(T)`, which is a no-op when already
    there. It prefers a running exact match, so "bring it to now" still works within the
    type.
  - Toggle `@route+id!`: place by type.
  - Unnamed `=` starts with no queue start the first untimed open entry `place(T)` would
    choose, else create a started `— T` entry instead of an unnamed one. Queued tasks
    still start their queue. The "finish the current Pomodoro first" guard is unchanged.
- **Budget.** Strict refusal per D7 (`capture/budget.rs`), with code `strict_cap` and a
  teaching message. Non-strict creation appends the existing budget warnings.
- **Output.**
  - Add an additive `type` object to `PomodoroLinkEndpoint` destinations when Bob placed
    by type: `{theme: ThemeRef, source, label}`. The existing `role` values are
    unchanged.
  - The human destination row appends ` · <label>` (D10).
  - `capture-parse`/dry-run previews carry the same data, so the Mac shows it without
    logic.
- **Acceptance tests (CLI).**
  - Inbox task, no note default, BUILD running → a CHORE placeholder is created; BUILD's
    timer is untouched; the preview says `→ new CHORE Pomodoro · default type`.
  - A `dev` task (BUILD) with BUILD running → links into the running BUILD.
  - A new `#ref` task → READ.
  - With no default configured → legacy behavior, byte-identical.
  - Two batch items needing CHORE → one placeholder.
  - `strict: true` refusal.
  - Timed-ambiguity errors are unchanged.
- **Docs.** In `docs/capture.md`, update the bare-target paragraphs (current/next →
  type-placed), the forms table rows, and the existing-task and Ensure Next sections.
  Update `docs/reference/mac-capture/protocol.md` for the additive `type` object, and
  the placement section in `docs/task-types.md`.
- **Commit.** `feat(capture)!: bare captures place Task Links by task type`.

## Phase `js-mirror`: Ledger Tools type model and routing mirror

Repo: bob-plugins (Ledger Tools, plus read-only helper updates in navigation-hotkeys,
block-id-prompt, and task-status-cycler).

- **Model.**
  - In `279-plugin-task-theme-model.js`, `pomodoroThemeTaskFieldState` and
    `pomodoroThemeRemoveTaskFields` accept both keys (D2 duplicate rule), and
    `pomodoroThemeSetTaskTheme` emits `[type:: X]`.
  - In `279-plugin-task-theme-suggest.js`, `pomodoroThemeNoteDefault` reads `task_type`,
    then `theme`, with the conflict warning. Add the default step (source `default`)
    after the note walk. `off` falls through to the default.
  - In `050-plan-budget.js` `coercePlanCaps`, parse `default_type` and `reading_type`
    (with the legacy alias and the both-present error) without resetting the block for
    unknown keys.
- **Routing mirror.** Add `src/279-plugin-task-type-routing.js` with `typeComponents`,
  `typeAgrees`, `planRetype`, `planPlacement`, and `planRelocation`, matching the Rust
  sections. Vocabulary, exempt, and inventory come from the existing
  `pomodoroThemesConfiguredRows` snapshot and plan caps.
- **`api.themes` v1, additive.** Add `effectiveTypeFor` (the cascade including the
  default), `typeComponentsOf`, `typeAgrees`, `planRetype`, `planPlacement`, and
  `planRelocation`. Keep `planThemeChange` as a deprecated wrapper over `planRetype`
  with `keep_type = linkOnce` until its consumers migrate. Document all of this in
  `docs/develop/plugin-apis.md`.
- **Vectors.** Copy bob-cli's `vectors.json` byte-for-byte to both locations, update the
  README pin, and run the `task_field`, `suggest`, `agree`, `retype`, `place`, and
  `relocate` sections in `scripts/test-ledger-tools-pomodoro-themes.cjs`, or a new test
  file registered in `package.json`.
- **Local helpers.** Update the plugin-local name and field helpers to read both keys:
  - `bob-navigation-hotkeys/src/315-task-card-theme-stage.js`
  - `bob-navigation-hotkeys/src/165-pomodoro-theme-picker.js`
  - `block-id-prompt/src/076-pomodoro-theme-picker.js`
  - `task-status-cycler/src/085c-successor-theme-write.js`

  No behavior change in those plugins yet.

- **chezmoi**, only if `default-type` deferred it: add `plan.default_type: CHORE` as
  specified there.
- **Commit.**
  `feat(ledger-tools): mirror task types, default type, and routing planners`. Deploy.

## Phase `type-tag-rendering`: `#TYPE` identity, ledger agreement marks, and Ledger Tools words

Repos: bob-plugins (Ledger Tools) and bob-cli (`docs/task-tag-marks.md`, which owns the
display contract and vectors).

- **Task-line identity** (D11).
  - The core goes in `src/138-task-tag-marks.js`: ranges and model gain `type`, `hue`,
    and `state`; widgets' `eq()` includes them. Live Preview and reading view go in
    `src/264-plugin-task-tag-marks.js` (`data-bob-type` plus `::after`).
  - Fold `[type:: X]` in Live Preview like today's theme field. Hide Dataview's `type`
    span (`data-dv-norm-key="type"`) as `taskThemeIsThemeInlineField` does for `theme`.
  - Provenance tooltips use `effectiveTypeFor`. Add the inherited-types toggle command
    (off by default).
- **Pill retirement.** Retire the trailing task-line pill paths in
  `279-plugin-task-theme-marks.js`, `-editor.js`, `-reading.js`, and `-tasks.js`. The
  existing toggle command id stays and is relabeled "Toggle task type marks". Keep the
  session-name pills.
- **CSS.** In `styles.css`, add `--bob-type-ink-<slot>` for light and dark, the
  typography, the states, and the repair `#?`. Reuse the existing `--bob-task-tag-color`
  hook.
- **Contrast.** Add a node test computing WCAG contrast for D11's 8 × 4 × 2 matrix from
  the CSS palette values; it fails below 4.5:1.
- **Ledger marks.** In today's ledger mirror, add the disagreement mark and the
  composite quiet prefix (D11) through `typeAgrees` and cached `effectiveTypeFor`.
- **Ledger Tools words.** The themes manager modal becomes "Types", the Day strip lane
  "Types", the Stats dimension label "type" ("by type"), and Pomodoro-name menu toasts
  read `Type hidden from Recent`. JSON and wire are unchanged.
- **Docs.** In bob-cli's `docs/task-tag-marks.md`, update the contract and conformance
  vectors, including the carve-out that ink now encodes type. Update bob-plugins'
  `docs/bob-ledger-tools.md`.
- **Commits.** bob-plugins:
  `feat(ledger-tools): render explicit task types as colored #TYPE`. bob-cli:
  `docs(task-tag-marks): #TYPE identity contract`. Deploy.

## Phase `move-retype`: Ctrl+Shift+M Pomodoro bullet moves retype tasks

Repo: bob-plugins (bob-navigation-hotkeys).

- **Planner.** `planPomodoroBulletMove` (`150-pomodoro-capture.js`) stays the subtree
  primitive. A new planning step computes `planRetype` for each moved dedicated Task
  Link whose open target task resolves, against the destination entry, or against the
  new `=NAME` entry.
- **Picker** (`PomodoroBulletMovePickerModal`, `165-pomodoro-theme-picker.js`):
  - Enter applies retypes.
  - Alt+Enter is new here. It keeps the types and merges every missing moved type into
    the destination's name, in move order.
  - Tab cycles the chosen component for composite destinations.
  - The footer shows the D10 consequence line for the highlighted row and updates live.
- **Commit** (`commitPomodoroBulletMoveSession`, `650-plugin-move-commit.js`). The
  ledger transaction comes first, then the per-task field writes via D12. The toast
  comes from `buildPomodoroBulletMoveNotice` (D10 copy).
- **Pomodoro-name menu copy** in this plugin moves to "type" words.
- **Tests.**
  - Node tests for single and multi-bullet moves into exact, composite, label, unnamed,
    PLAN, and new-entry destinations, including Enter vs Alt+Enter vs Tab.
  - A buffer changed between preview and commit → replan or refuse.
  - A closed-note field write uses `vault.process`.
- **Docs.** `docs/bob-navigation-hotkeys.md` (bob-plugins).
- **Commit.**
  `feat(navigation-hotkeys): Pomodoro bullet moves keep task types in agreement`.
  Deploy.

## Phase `link-picker-retype`: Link to today retypes and Alt+Enter keeps the type

Repo: bob-plugins (block-id-prompt).

- **Write path.** In `078-pomodoro-theme-write.js`, `resolvePomodoroLinkThemeWrite` uses
  `planRetype`. It drops the shown-suggestion and single-non-exempt guards and keeps the
  fresh revalidation. The edit still rides `planTargetTaskUpdate`'s `themeEdit`.
- **Alt+Enter.** `commitLinkOnce` (`085-pomodoro-link-picker-modal.js`) becomes the
  keep-type commit: link plus the D6 merge, and no field write. When the destination is
  a container or already agrees, Alt+Enter equals Enter.
- **Footer** (`describePomodoroThemeFooterPreview`, `077-pomodoro-picker-intent.js`):
  D10 copy. Remove `⌥↵ link once`. Add Tab component cycling for composites.
- **Walk behavior.** `]s` landings and the walk are unchanged:
  `decisions:answering-advances-the-walk` still applies to Ctrl+Shift+Enter.
- **Menu copy.** Pomodoro-name menu copy in `076-pomodoro-theme-picker.js` moves to
  "type" words.
- **Tests.** Node tests: picker footers, Enter/Alt+Enter/Tab outcomes, PLAN target,
  label target, and the clear-on-inherited case.
- **Docs.** `docs/block-id-prompt.md` and the `themeIntent` section in
  `docs/develop/plugin-apis.md` (bob-plugins).
- **Commit.** `feat(block-id-prompt)!: Link to today keeps task types in agreement`.
  Deploy.

## Phase `ref-today-successors`: bob ref today and successor offers place by type

Repos: bob-cli, bob-plugins (task-status-cycler).

- **`bob ref today`.**
  - With no target, the reading task takes `capture-placement`'s typed placement through
    `link_existing_task`. Verify it, keep `already_queued`, and align `theme_suggestion`
    with D7, including `create: true` meaning the placement will create.
  - With explicit targets (`-e`, `-P`, `#name`), the pick is deliberate, so apply
    `routing::retype` and report the additive `task_theme` object. The docs line "This
    command never writes a theme field" goes away.
- **Rust successors** (`task_complete/successors.rs`): each successor's default
  destination keeps the closing entry's slot when the closing entry agrees with the
  successor's type; otherwise it is `place(T_successor)`. Explicit successor selection
  remains required (`decisions:unblocked-tasks-are-offered`).
- **JS successors.** `085c-successor-theme-write.js` and `089b-successor-verdicts.js`
  get the same default via `planPlacement`. A retargeted pick applies `planRetype`
  (Enter path). Chips use D10 copy.
- **Tests.** Rust CLI tests for ref-today default and explicit targets and for successor
  defaults. Node tests for successor rows.
- **Docs.** `docs/ref.md` (ref today), the successor sections of `docs/capture.md`, the
  ref-today paragraph in `docs/task-types.md`, and bob-plugins
  `docs/develop/plugin-apis.md` (the successor section).
- **Commits.** bob-cli: `feat(ref)!: bob ref today places reading tasks by type`.
  bob-plugins: `feat(task-status-cycler): successor offers place by task type`. Deploy.

## Phase `task-card-relocate`: Task Card type edits relocate the live Task Link

Repo: bob-plugins (bob-navigation-hotkeys).

- **Words and choices.** The row and stage become "Type" (the `t` key stays). Any type
  is choosable, including PLAN and other exempt types: remove the single-non-exempt rule
  from `315-task-card-theme-stage.js`. "Use default" always clears, and its label names
  the inherited source (`Use default · BUILD from dev`, or `· CHORE default type`).
- **Relocation.** On apply (`applyTaskCardThemeChoice`,
  `352-picker-task-card-theme-stage.js`):
  - compute the field edit, then `planRelocation` (D8, accepted `running_retype` branch)
    from today's ledger snapshot;
  - write the field first, then the ledger as D12's checked step;
  - the ledger edit reuses the bullet-move subtree primitive from `move-retype` and the
    existing placeholder-cleanup rule.

  Ctrl+D (`clear-theme` intent) follows the same path.

- **Consequence line.** Show it under the selected stage row before Enter (D10),
  updating as the selection moves. The strict refusal shows in the stage and writes
  nothing.
- **Toasts.** Replace `taskCardThemeSavedNotice` and `taskCardThemeUnchangedNotice` with
  the D10 copy. The edit never advances the walk (`reviewOrigin` stays cleared) and
  never stamps freshness or changes lanes.
- **Tests.**
  - Node tests: unlinked backlog retype (field only); planned link moving into an
    existing planned BUILD; placeholder rename; create-below; running sole and non-sole
    per the accepted branch; strict refusal; buffer race; one undo per open editor.
  - The linked-tasks path (`prepareDependencyTargetNote`).
- **Docs.** bob-plugins `docs/task-card.md`: replace "No status, Task Link, ID, route, …
  change is caused by a theme-only edit" with the relocation contract. Also
  `docs/bob-navigation-hotkeys.md`.
- **Commit.** `feat(navigation-hotkeys): Task Card type edits move the live Task Link`.
  Deploy.

## Phase `gesture-retype`: Note moves and Pomodoro renames keep agreement

Repos: bob-plugins (bob-navigation-hotkeys task move, plus wherever entry rename and
Name-it gestures live), bob-cli (`bob task move`, `bob capture-pomodoro-name`).

- **Note moves.** Moving a task without a field to a note whose chain yields a different
  inherited type relocates its live link (D8). A task with an explicit field keeps its
  type, so no relocation happens. This applies to:
  - Obsidian task move (`640-plugin-notes-and-task-move.js` and `writeTaskMoveChange`,
    with its existing rollback);
  - `bob task move` (`src/native/task_move`), as an additive JSON `relocated` object and
    a human row.

  The toast and human row read `Moved to dev · link moved to BUILD`.

- **Entry naming and renames.** Naming or renaming an open entry that holds Task Links
  applies D6's Enter path to each linked task whose type the new name does not include.
  A label or container result retypes nothing.
  - The preview shows the count: `3 tasks become BUILD`.
  - It applies to `bob capture-pomodoro-name` (additive JSON
    `retyped: [{route, block_id, action, theme}]`; the protocol only grows) and to the
    Obsidian rename and Name-it gestures. Find them via `formatPomodoroEntryLine`
    callers and `165-pomodoro-theme-picker.js`.
  - Ctrl+X merges only add types, so they never retype.
- **Out of scope.** Frontmatter `parent:`, `task_type:`, and config edits are not
  gestures; the lint covers them.
- **Tests.** Rust tests for `bob task move` relocation and `capture-pomodoro-name`
  retypes. Node tests for Obsidian task move and rename retypes, plus the no-field and
  field cases.
- **Docs.** `docs/task-move.md`, `docs/capture.md` (Name-it), `docs/task-types.md` (D9
  rows), and bob-plugins hotkeys docs.
- **Commits.** bob-cli:
  `feat(types): note moves and Pomodoro renames keep task types in agreement`.
  bob-plugins:
  `feat(navigation-hotkeys): note moves and renames keep task types in agreement`.
  Deploy.

## Phase `mac-copy`: Bob Mac Capture type copy

Repo: bob-mac-capture. Copy and rendering only; Bob computes everything.

- **Copy.**
  - `Sources/CaptureCore/CaptureTaskThemePresentation.swift` uses D10's Mac strings.
  - `CompletionRowContent.swift` says `Type hidden from Recent…` and
    `your type · never used`.
  - `CaptureUnconfiguredThemePresentation.swift` uses the label copy (D4).
- **Destination label.** `CapturePomodoroLinkPresentation.swift` renders the additive
  destination `type` label (`→ BUILD · from dev`,
  `→ new CHORE Pomodoro · default type`). It decodes the field optionally, so older bob
  JSON still renders.
- **Tests.** Update `CaptureTaskThemeTests.swift` and
  `CaptureUnconfiguredThemePresentationTests.swift`, and add decoding and presentation
  tests for the `type` object. Run `just portable-check`, or the full `just` checks with
  Xcode.
- **Docs.** Update the Mac docs mounted into the bob site, if they quote the old
  strings.
- **Commit.** `feat(capture): show task-type placement and type copy`.

## Phase `closeout`: Memory, config spelling, vectors cleanup, and acceptance

Repos: bob-cli (memory, docs, vectors), bob-plugins (vectors copies), chezmoi.

> [!decision] memory_glossary Use `/sase_memory_write`.
>
> - Rename `sase/memory/glossary/pomodoro-theme.md` to `glossary/task-type.md`, keyword
>   **Task Type**, aliases `type`, `theme`, `pomodoro theme`, `pomodoro-theme`.
> - Rewrite its body to define D1–D5 in glossary form:
>   - the cascade with `plan.default_type`;
>   - fields holding only exceptions;
>   - type components vs labels;
>   - the agreement rule, with drift flagged;
>   - exempt as a budget attribute only;
>   - `bob types` (alias `bob themes`);
>   - unchanged hue priority and Recent-only hiding;
>   - the "not a SASE bead `task_type`, note `type:`, Pomodoro home, or appearance
>     theme" disambiguation;
>   - `docs/task-types.md`.
> - Repoint `[[pomodoro-theme]]` to `[[task-type]]` in `glossary/day-strip.md`,
>   `glossary/stats-dash.md`, and `glossary/root-parent.md`.
> - Run `sase memory init` and confirm `glossary:theme` and `glossary:pomodoro-theme`
>   still resolve.

> [!decision] memory_decision_records Add two accepted records to the `decisions` memory
> web. Follow the existing record format: applies-to, claim, why with rejected
> alternatives, cost, reopens-when, and evidence citing the research ref and this epic.
>
> 1. **Task Types And Live Task Links Always Agree**
>    (`task-types-and-live-links-agree`): D5 through D9. Rejected alternatives:
>    one-direction "type is master", silent drift repair, free disagreement, and
>    deriving Pomodoro names from tasks.
> 2. **Every Open Task Has A Computed Type** (`every-open-task-has-a-computed-type`):
>    D2, D3, and D11's explicit-only rendering. Rejected alternatives: stamping
>    defaults, tag storage (`#fix`), frontmatter `type:`, and keeping two nouns.
>
> Do not edit or supersede existing records: `pomodoro-theme-is-one-component` and the
> theme hue and preference records stay true. Run `sase memory init`.

- **chezmoi.** Confirm that the installed `bob` reads `types:`: run
  `bob types list -f json` against a temp config containing `types:` and check for
  `configured_themes`. If it does, rename `themes:` to `types:` in
  `home/dot_config/bob/config.yml` and commit `bob: spell the type vocabulary types:`.
  If not, record a `PROPOSED FOLLOW-UP:` on this phase's bead and leave `themes:`.
- **Vectors.** Delete the retired `write_rule` section from bob-cli's vectors and both
  bob-plugins copies, once `rg` shows no consumer. Remove the deprecated
  `api.themes.planThemeChange` wrapper only if no plugin calls it; otherwise keep it and
  note why. Deploy if plugins changed.
- **Docs sweep.** Remove leftover user-facing "theme" words for the task-type concept
  under `docs/` and bob-plugins `docs/` (wire names excepted). Then run `just docs-gen`
  and `just docs-check`.
- **Acceptance.**
  - Run D7, D8, and the phase acceptance lists end to end in a temp vault through the
    CLI.
  - Run the plugin suites with `npm test`.
  - Run `bob types migrate -d` against Bryan's live vault (read-only) and report the
    preview. Bryan runs the write himself.
- **Commit.** bob-cli: `chore(types): close out the task-types epic`, plus the memory
  commit.
