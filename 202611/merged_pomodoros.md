---
tier: epic
title: 'Merged Pomodoros: merge by default, split and merge by type, honest close
  names'
goal: 'A Pomodoro that holds several task types is a first-class merged Pomodoro.
  Linking or moving a task into another type''s Pomodoro merges that type into the
  name instead of retyping the task. Capture `=*` and `=/` merge and split Pomodoros
  by type, and so does Obsidian Ctrl+S. `x` names the closed and carried sessions
  after the work that was actually done and is still planned. bob-cli, bob-plugins,
  and Bob Mac Capture all agree, through shared vectors.

  '
phases:
- id: routing-core
  title: Pure planners and shared vectors (bob-cli)
  depends_on: []
  size: medium
  description: 'routing-core: implement R1–R7 as pure Rust planners (land, merge_names,
    narrow_closed, shed, split_plan, select_by_type) with new shared vector sections
    and the task-types.md contract.'
- id: capture-link-merge
  title: Capture lands by merging (bob-cli)
  depends_on:
  - routing-core
  size: medium
  description: 'capture-link-merge: make named capture links, ref today, and #type
    markers merge the task''s type into the destination name; retire type_marker_conflict;
    report the merge in JSON and human output.'
- id: capture-split
  title: The =/ split syntax end to end (bob-cli)
  depends_on:
  - routing-core
  size: large
  description: 'capture-split: add the whole-item =/ token covering lexing, chaining,
    execution, capture-parse spans, completion contexts, output, and docs; bump the
    grammar to 5.'
- id: capture-merge
  title: The =* merge syntax end to end (bob-cli)
  depends_on:
  - capture-split
  size: medium
  description: 'capture-merge: repurpose =* as merge, keeping retired-close teaching
    for digit and marker forms; add execution, spans, completion, output, and docs.'
- id: capture-close-names
  title: x narrows the closed name and sheds the carried name (bob-cli)
  depends_on:
  - routing-core
  size: medium
  description: 'capture-close-names: apply R6 in the shared close planner using pre-close
    task types; split the closed-name and continuation-name roles; add the closed_name
    and shed output.'
- id: obsidian-link-merge
  title: Obsidian pickers merge by default (bob-plugins)
  depends_on:
  - routing-core
  size: medium
  description: 'obsidian-link-merge: copy the vectors, add the planLanding/mergeNames/shed
    mirror, flip Enter to merge and Alt+Enter to retype in the link and move pickers,
    ^^ links, successor retarget, and Ctrl+X merge naming; shed on move.'
- id: obsidian-ctrl-s
  title: Ctrl+S splits the selected Pomodoro (bob-plugins)
  depends_on:
  - obsidian-link-merge
  size: medium
  description: 'obsidian-ctrl-s: add the planSplit mirror, a contextual Ctrl+S dispatch,
    a palette command, the two-type auto split, and the multi-select split picker
    with refusals, docs, and tests.'
- id: obsidian-close-names
  title: Ctrl+Enter Pomodoro completion parity (bob-plugins)
  depends_on:
  - capture-close-names
  - obsidian-ctrl-s
  size: medium
  description: 'obsidian-close-names: mirror R6 narrowing and shedding in task-status-cycler''s
    Pomodoro completion so ledgers stay byte-identical with bob''s x.'
- id: mac-capture
  title: Present split, merge, merged links, and close names (Bob Mac Capture)
  depends_on:
  - capture-link-merge
  - capture-merge
  - capture-close-names
  size: medium
  description: 'mac-capture: decode and present the new span kinds, modes, needs,
    split/merge preview cards, merged destination rows, and close names; accept grammar
    5.'
- id: glossary
  title: Merged Pomodoro memory (bob-cli)
  depends_on:
  - capture-link-merge
  - capture-merge
  - capture-close-names
  - obsidian-ctrl-s
  - obsidian-close-names
  size: small
  description: 'glossary: add the Merged Pomodoro glossary strand and, if accepted,
    the links-merge decision record; regenerate memory.'
decisions:
  new_task_named_link:
    ask: When a capture creates a task inside a named Pomodoro of another type, should
      it merge rather than retype?
    choices:
      merge: The new task keeps its inherited type and the Pomodoro gains it, the
        same rule as every link
      retype: New tasks still take the Pomodoro's type; only existing tasks merge
    default: merge
    why: One landing rule everywhere; a free `#type` marker still types a new task
      explicitly
    answer: merge
  shed_on_move:
    ask: Should Ctrl+Shift+M drop a merged Pomodoro's type when the last Task Link
      of that type moves out?
    default: true
    why: A type travels with its tasks. Close uses the same shed rule, so names never
      go stale
    answer: true
  memory_merged_pomodoro:
    ask: Add a "Merged Pomodoro" glossary term?
    memory:
    - glossary:merged-pomodoro
    default: true
    requested: We should add an excellent new glossary term for "merged pomodoro".
    answer: true
  memory_links_merge_record:
    ask: Record a decision that linking merges types and never retypes tasks?
    memory:
    - decisions:links-merge-types
    default: false
    answer: false
proposed_by: bbugyi200.apollo.c4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 16:00:38
status: done
bead_id: bob-cli-fe
---

- **PROMPT:** [prompts/202611/merged_pomodoros.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/merged_pomodoros.md)
- **BEAD:** [bob-cli-fe](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fe/README.md)

# Plan: Merged Pomodoros

Today a Pomodoro name such as `BUILD + FIX + READ` already works:
`plan_budget::split_components` splits it, and each component gets a hue. Bob still
treats the composite as an accident to avoid, though:

- Linking a task into another type's Pomodoro **retypes the task** by default. That is
  D6 Enter in `src/native/pomodoro_themes/routing.rs::retype`. Merging is hidden behind
  Obsidian Alt+Enter.
- Splitting is possible only one bullet batch at a time, through the Ctrl+Shift+M
  `=NAME` peel.
- `x` copies the running name verbatim onto both the closed session and the carried one.

This epic makes the merged Pomodoro a first-class shape with one small set of rules
(R1–R7 below). Rust owns them as pure planners with shared vectors. Every surface then
applies the same rules:

- capture links,
- `=*` / `=/`,
- `x`,
- the Obsidian pickers, Ctrl+S, and Ctrl+Enter,
- Bob Mac Capture.

**Bryan's requests mapped to this plan**

| Request                                                                                                                                                 | Where                                                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| Ctrl+S splits the selected Pomodoro by type. Two types split the right one automatically; more types prompt a multi-select that excludes the first type | R4, phase `obsidian-ctrl-s`                                 |
| `=/<base>#<split>` capture syntax                                                                                                                       | R3, R4, phase `capture-split`                               |
| Linking or moving a task into a Pomodoro of a different type assumes a merge                                                                            | R2, phases `capture-link-merge` and `obsidian-link-merge`   |
| `=*<src>#<dest>[#<dest>…]` capture syntax                                                                                                               | R3, R5, phase `capture-merge`                               |
| Glossary term "merged pomodoro"                                                                                                                         | phase `glossary`                                            |
| `x` close names follow the real work                                                                                                                    | R6, phases `capture-close-names` and `obsidian-close-names` |

## The rules (shared contract)

### R1 — Vocabulary

- **Merged Pomodoro:** a ledger entry whose ` — NAME` has **two or more type
  components** (`routing::type_components`, with labels excluded).
  - `BUILD + FIX + READ` is merged.
  - `BOB + BUILD` is not merged when BOB is a label.
  - `LATER` is a container.
- **Base type:** the merged Pomodoro's _first_ type component, the one the others were
  merged into. A split never moves the base, and a merge keeps the survivor's base
  first.
- **Merged-in types:** every type component after the base, in name order.
- **Names follow links:** after any merge, split, move, or close, a name says which
  kinds of work its Task Links hold. Labels are never added, moved, or removed by these
  rules.
- **MEET stays MEET:** an entry whose name normalizes to `meet`
  (`capture/meet_event_run.rs::is_meet_name`) is owned by calendar sync. Sync finds it
  by its exact name (`docs/gcal.md`: "never … renamed"). No rule here ever renames,
  merges, or splits it.

### R2 — Landing: links merge, never retype

This rule replaces D6's default Enter path. When a gesture lands a task's live Task Link
in an open Pomodoro D, the first matching step wins:

1. **Container:** D has no type components. Keep the task's type; the name is unchanged.
2. **Consistent:** the task has no effective type, or its type key is already a
   component of D. Nothing changes.
3. **MEET destination:** take today's retype path, `Set MEET` or `Clear`. MEET names
   never grow.
4. **Explicit retype:** the gesture asked to retype (the Obsidian pickers' Alt+Enter).
   Take today's Set/Clear path, choosing the selector, then the Tab pick, then the first
   non-exempt type, then the first type.
5. **Merge (the default):** D's name gains ` + T`, where T is the task's effective type
   in catalog spelling, appended after the existing components. No task field is
   written. Several links landing at once append each missing type once, in link order,
   deduped by key.

Further rules:

- **`#<type>` marker:** a capture `#<type>` marker sets X first, then lands with the new
  type. A marker that disagrees with a named destination therefore **merges**. The
  `type_marker_conflict` refusal is retired and never emitted again.
- **Writers that do not change:**
  - Naming or renaming an entry still retypes its linked tasks (D6 rename). Naming
    states what an entry _is_.
  - D8 relocation after a deliberate type change is unchanged.
  - D7 bare placement is unchanged (placing by type cannot disagree).
  - The Task Card and inbox route are unchanged.
- **Rust API:**
  - `routing::retype` becomes `routing::land`, with `LandInput` and `LandOutcome`.
    `RetypeInput.keep_type` becomes `retype: bool`, and false is the default.
  - The `retype` vector section becomes `land`. Each case states `retype` instead of
    `keep_type`, and default cases now expect `merge`.

> [!decision] new_task_named_link = merge A new task created by `<text> @route:id#NAME`,
> `@route:id=name`, or a project-note ` :<id>` bullet under `#NAME` lands like any other
> link. It keeps its inherited type, and D gains ` + T` when T is missing. Typing it
> explicitly is `#fix`.

> [!decision] new_task_named_link = retype Capture's _new-task_ paths keep today's
> retype outcome by passing `retype: true`. Existing-task links, ref-today, and the
> Obsidian pickers still merge.

### R3 — Type selectors name Pomodoros

A **type selector** names an open Pomodoro by the type work would land in. It is written
as a bare name (`build`), as `#name`, or as a literal `#"SASE V18"` (a literal matches a
whole component key exactly and never by prefix).

Resolution reuses placement (`routing::place`) without ever creating an entry, over
today's _open_ entries:

1. the running entry that is exact for S (`routing::is_exact`),
2. else the first open exact entry,
3. else the first open entry containing S.

When no whole-key match exists anywhere, the same three steps run with key-prefix
matching. Ledger order breaks ties. Closed entries never match.

The `=/` **base** selector is narrower. It matches the _base_ of open merged Pomodoros
(whole key, else prefix), preferring the running entry, then ledger order.

**Current Pomodoro:** a selector may be omitted. It then means the running Pomodoro, or,
when nothing runs, the next planned entry a bare `=` would start.

### R4 — Split

Inputs are a target E (an open merged Pomodoro) and a split set K (non-empty, a subset
of E's merged-in types).

- **What moves:** each top-level child of E whose body is exactly a Task Link moves to
  its type's new entry, together with its whole subtree, when its task's effective type
  key is in K. Task Link bodies are `[[T]]`, `[[T]]#`, `![[T]]`, or `![[T]]#`, including
  same-note `[[#^id]]`. Use the forms that
  `capture_pomodoro_close/selection.rs::classify_numbered_line` recognizes.
- **What stays:** prose, notes, mentions, and every other link (base type, unsplit
  types, untyped, or drifted).
- **E's name:** E loses exactly the K components. Everything else keeps its spelling and
  order.
- **E's entry line:** E keeps its checkbox, timespan, and position. A running entry
  stays running.
- **New entries:** one new planned entry `- [ ] () — T` for each T in K. They are
  inserted directly below E's block, in E's name order, with T spelled as in E's name.
  - An entry is created even when T had no links (it becomes a planned T slot).
  - Separator and indentation follow `insert_named_placeholder` /
    `format_named_placeholder_line`.
- **Guards** (each one a precise, teaching refusal):
  - E is closed: closed Pomodoros are history.
  - E has fewer than two type components (`BUILD has one type — nothing to split`).
  - E is MEET.
  - A K member is the base (`BUILD is the base of BUILD + FIX + READ and stays`).
  - A K member is not a component of E.
  - A K member is named twice.
  - With **exactly two** types, the split type is implicit, so an explicit `#type` is
    refused with code `split_type_not_needed`. The refusal teaches `=/build`; this
    follows Bryan's "`#<split_type>` can be provided iff there are more than 2 types".
- **No K given:** with no K, every merged-in type splits. With exactly two types that is
  the right-hand one.

### R5 — Merge

Inputs are a survivor S (open) and absorbed entries A1…An (open, _untimed_ planned,
distinct from each other and from S).

- **S's name:** S's components, then each Ai's components that are not already present.
  Dedupe by `normalize_component` (the first spelling wins), keep labels, and follow
  selector order.
- **Children:** each Ai's child lines append to the end of S's block in selector order,
  re-indented to S's child indentation.
- **Duplicate bullets:** a top-level Task Link bullet with no children whose trimmed
  text already appears as a top-level child of S (or of an earlier Ai) is dropped as an
  exact duplicate. This matches the existing Obsidian rule that exact duplicate bullets
  merge away.
- **Absorbed entry lines:** each Ai's entry line is removed.
- **Guards** (teaching refusals):
  - An Ai is running or timed. Its timespan would be lost, so the refusal teaches the
    swapped form `=*fix#build`.
  - S or an Ai is MEET.
  - An Ai equals S, or is named twice.
  - A selector matches nothing.
  - A selector resolves to a type S already contains through the same entry
    (`FIX is already part of BUILD + FIX`).

Obsidian's Ctrl+X entry merge (`planPomodoroEntryMerge`) adopts the same name rule,
deduping by key, in phase `obsidian-link-merge`. Its 48-character cap stays an
Obsidian-only guard.

### R6 — Close names: narrow the closed session, shed the carried one

At close, every numbered Task Link of the running entry E has two attributes:

- its pre-close effective type key, if any, read from the _pre-close_ staged task line
  (closed tasks stop suggesting);
- its outcome: `in_progress`, `parked`, `complete`, `deferred`, or `dropped`.

**Worked** = in progress, parked, or complete. **Carried** = in progress or deferred
(today's carry set).

- **Closed name (narrow):**
  - When at least one worked link's type is a type component of E, the closed entry's
    name becomes E's labels plus exactly the type components some worked link has, in
    E's order and spelling.
  - Otherwise the name is unchanged: nothing typed was worked, or only drift was worked.
  - Bryan's example: only a FIX task was worked in `BUILD + FIX`, so the closed entry
    reads `FIX`.
- **Carried name (shed),** only when the close _creates_ the next entry:
  - Drop each type component T when E had at least one link of type T and no carried
    link has type T.
  - If that would drop every type component, keep the base.
  - Components that never had a link stay. Labels stay.
  - Bryan's example: the last FIX task was completed in `BUILD + FIX`, so the new entry
    reads `BUILD`.
  - The rule generalises consistently: a FIX link that was parked or dropped is not
    carried, so FIX is shed as well.
- **When the rule does nothing:**
  - Single-type entries never change.
  - MEET entries are skipped explicitly.
  - The `x0` reset keeps its name.
  - A close that points `next_pomodoro` at an existing later entry never renames that
    entry.

### R7 — Shedding on move

> [!decision] shed_on_move When Ctrl+Shift+M moves Task Links out of an open merged
> Pomodoro (by Enter or Alt+Enter), the source sheds by R6's shed rule:
>
> - Each type T with at least one link before the move and none after it is dropped.
> - If every type would go, the base stays.
>
> When the move empties the source, the existing delete rule still wins. Answering
> **no** leaves source names untouched.

## Phase `routing-core` — pure planners and shared vectors (bob-cli)

Implement R1–R6 (and R7's shed rule) as pure functions over plain data. They live in
`src/native/pomodoro_themes/routing.rs`, plus a sibling such as `merged.rs` if
`routing.rs` grows past comfort.

- `land` (R2): `LandInput { …, retype: bool }` and
  `LandOutcome::{Container, Consistent, Merge{name}, Set{name,key}, Clear}`. Apply the
  MEET rule. Update every caller to the new names:
  - `capture/task_theme.rs`,
  - `capture_pomodoro_name.rs`,
  - the vector test harness.

  Callers keep their current behaviour until their own phase (`capture_pomodoro_name`
  passes `retype: true`). This phase changes semantics only in the pure function and its
  vectors.

- `merge_names(base, others) -> String` (R5 dedupe).
- `narrow_closed(name, worked_keys, sources)`.
- `shed(name, before_keys, after_keys, sources)` (R6 / R7).
- `split_plan(name, link_keys, split_selectors?, sources)`. It returns the remaining
  name, the ordered new entries `[{name, link_indices}]`, and the guard error codes from
  R4.
- `select_by_type(entries, selector, filter)` (R3), including the base-selector variant.
  It reuses `place` / `is_exact` without creation.
- `tests/fixtures/pomodoro_themes/vectors.json`:
  - Rename `retype` to `land` and flip the default-case expectations to `merge`.
  - Add the sections `merge`, `split`, `narrow`, `shed`, and `select`.
  - The cases must cover labels, exempt PLAN, MEET, prefix vs whole-key, literals, the
    two-type implicit split, duplicates, and the "keep the base" fallback.

  Both JS copies in bob-plugins are refreshed in phase `obsidian-link-merge`. Later
  bob-cli phases must not change these sections; a needed change goes back through this
  contract, with a re-copy.

- Rust tests consume every new section.
- Docs, `docs/task-types.md`:
  - Rewrite the "Routing: agreement, retype, placement, relocation" bullets as _landing_
    (R2).
  - Add a **Merged Pomodoros** subsection that defines R1, R3, R4, R5, and R6 as the
    shared contract.
  - Update the vectors list under "Shared vectors".
  - Read the `docs.md` reference memory first.

## Phase `capture-link-merge` — capture lands by merging (bob-cli)

Wire R2 into every capture form that lands a link in a _named_ or explicitly chosen
Pomodoro:

- `<text> @route:id#NAME` (subject to `new_task_named_link`),
- whole-item `@route:id#NAME` / `^route:id#NAME`,
- the `=name` link-start shorthands (`@route:id=name`),
- project-note ` :<id>` bullets and picked offers under `#NAME`,
- `bob ref today -p/-P` (it goes through `capture/pomodoro_link.rs`).

The entry points are `CaptureTaskThemeContext::apply` (`capture/task_theme.rs`, which
today returns `Ok(None)` on `Merge`) and its callers in `capture/plan.rs`,
`capture/pomodoro_link.rs`, and the project-note path.

- **On `Merge`:** stage the destination entry's rename through the batch planner, in the
  same transaction as the link. Use `relocation_apply::rename_entry`, or the shared tail
  helper, rather than a fourth private copy. Later items in a batch must see the merged
  name (`BUGS + BUILD`), so two captures landing BUILD in BUGS append it once.
- **`#<type>` marker:** set X, then land. Delete `validate_type_marker_agreement`'s
  refusal; a disagreement becomes a merge. Remove the `type_marker_conflict` rows from
  docs and tests.
- **Output:**
  - `PomodoroLinkEndpoint` gains an additive
    `merged: {before, after, added: [ThemeRef]}`.
  - The human destination row appends ` · merged FIX`
    (`→ under BUGS + FIX · merged FIX`).
  - Dry-run reports the same data, so Mac previews it.
  - `task_theme` stays omitted when no field changes.
- **Docs:**
  - `docs/capture.md`: the grammar table rows for `@route:id#pomodoro`,
    `@route:id=name`, and `#<type>`; the "Pomodoro-linked tasks" and "Linking and
    starting existing tasks" sections; the Pomodoro blocks JSON.
  - `docs/task-types.md`: "Named captures retype the task" becomes "Named captures merge
    types".
  - `docs/ref.md` (around line 853).
- **Commit:** this is a behaviour change on the release train, so commit it as
  `feat(capture)!:`. Never edit `Cargo.toml` or `CHANGELOG.md`.

## Phase `capture-split` — `=/` split syntax, end to end (bob-cli)

Add a whole-item session token. The grammar is:

```text
=/                      split the current Pomodoro (running, else the next planned); all merged-in types
=/<base>                split the open merged Pomodoro whose base matches <base>; all merged-in types
=/<base>#<t>[#<t>…]     split only the listed types (target must have ≥3 types)
=/#<t>[#<t>…]           same, on the current Pomodoro
```

- **Lexing:**
  - Extend `session_equals_token` / `EqualsToken` (`capture_language/item.rs`) with a
    `Split` variant. Today `=/x` falls through to an ordinary task.
  - Base names use `bare_start_name_len`. Type parts use the named-start `#name` /
    `#"Literal"` lexer.
  - Add the token to `is_session_chain_token` so `x =/` and `=/ =` chain, running left
    to right against staged edits.
  - Whole-item only: extra text, child lines, or route and section flags are each
    refused with a precise error. A mid-body `=/` stays prose.
- **Execution:**
  - A new `CaptureKind::PomodoroSplit` is planned through `CaptureBatchPlanner`. It
    resolves with R3, partitions with `split_plan`, and resolves link types with
    `CaptureTaskThemeContext::placement_for_task` against staged lines.
  - Subtrees move with the existing helpers (`move_subtree_to_entry`,
    `insert_named_placeholder`).
  - All of R4's guards apply, with stable codes such as `split_not_merged`,
    `split_base_type`, `split_type_unknown`, `split_type_repeated`,
    `split_type_not_needed`, `split_meet`, and `split_target_missing`.
  - A missing target teaches with today's merged Pomodoros, for example
    `BUILD + FIX + READ → =/build#fix`.
- **`capture-parse`:**
  - `EditorMode::PomodoroSplit`.
  - Span `pomodoro_split` over `=/`.
  - `pomodoro_name` spans over the base and each `#type`, so `attach_theme_refs` adds
    pills. Check `is_theme_name_span`.
  - Need `pomodoro_split_type` for a trailing `#`.
- **`capture-complete`:**
  - New contexts: `pomodoro_split_base` offers today's open merged Pomodoros, each
    labelled with its full name, plus "current". `pomodoro_split_type` offers the
    target's merged-in types not yet listed, each with its link count.
  - These contexts must run before the "session item is an action" early return
    (`capture_language/completion.rs`, around line 1081).
- **Output:**
  - JSON
    `pomodoro_split: {path, entry_line, before, after, base: ThemeRef, created: [{type: ThemeRef, line, name, moved_links}], stayed_links}`.
  - Human, for example:

    ```text
    split  BUILD + FIX + READ → BUILD · 20261101.md line 14
      ↓ FIX   new Pomodoro · 2 links
      ↓ READ  new Pomodoro · no links
    ```

  - `pomodoro_blocks` roles pick up the created entries.

- **Grammar:** bump `GRAMMAR_REVISION` to 5. Bob Mac Capture's gate is `>=`, so this is
  safe. Update help text and `docs/reference/json-outputs.md`.
- **Docs, `docs/capture.md`:**
  - grammar-at-a-glance rows and the "You typed" examples,
  - a new "Splitting a merged Pomodoro" section next to "Starting a named Pomodoro",
  - chain rules,
  - the Pomodoro blocks JSON.
- **Tests:**
  - capture_language unit tests (grammar, chain, editor spans, completion),
  - `tests/cli/capture/` integration tests with fixtures under
    `tests/fixtures/capture_pomodoros/`,
  - dry-run round trips.
- **Commit:** `feat(capture)!:` (grammar 5).

## Phase `capture-merge` — `=*` merge syntax, end to end (bob-cli)

The grammar is:

```text
=*<src>#<dest>[#<dest>…]   merge the planned <dest> Pomodoros into <src>
=*#<dest>[#<dest>…]        merge them into the current Pomodoro
```

- **Collision with retired closes.** `=*…` is a retired close spelling today
  (`close_selection.rs::close_after_eq_parts`, `retired_whole_item_close_rewrite`; tests
  in `capture_language/tests/retired_close.rs`, `tests/chain.rs`, `tests/bare_close.rs`,
  `tests/cli/capture/pomodoro_close_retired.rs`; docs in `docs/capture.md` "Retired
  close spellings"). Partition it cleanly:
  - `=*` followed by a letter, `#`, or `"` is **merge**.
  - `=*` followed by a digit, `!`, `~`, `@`, `*`, or `,` (`=*1`, `=*!2`, `=**`)
    **stays** a retired-close teaching error.
  - A bare `=*` becomes an _incomplete merge_. The need is `pomodoro_merge_dest`, and
    the message adds "to close parking every link, write `x*`", so the retired hint
    survives.
  - `=*<src>` with no `#dest` is incomplete with the same need.
  - `==*` keeps its "not a close" error.
  - Update the retired-close table and tests to match.
- **Execution:**
  - A new `CaptureKind::PomodoroMerge` resolves the src with R3 (or uses the current
    Pomodoro) and each dest with R3 restricted to untimed planned entries other than the
    src.
  - It applies R5, reusing `rename_entry` and the subtree-move helpers from
    `capture-split`.
  - Guards use stable codes such as `merge_timed_pomodoro` (which teaches the swap),
    `merge_meet`, `merge_same_pomodoro`, `merge_dest_repeated`, `merge_target_missing`,
    and `merge_already_part`.
  - Chains via `is_session_chain_token`.
- **`capture-parse`:** `EditorMode::PomodoroMerge`, span `pomodoro_merge` over `=*`,
  `pomodoro_name` spans on src and dests, and need `pomodoro_merge_dest`.
- **`capture-complete`:** contexts `pomodoro_merge_src` and `pomodoro_merge_dest`. Rows
  are type selectors, each annotated with the entry it resolves to
  (`FIX → FIX (planned · 2 links)`). Dest rows exclude the src and timed entries.
- **Output:**
  - JSON
    `pomodoro_merge: {path, entry_line, before, after, absorbed: [{name, from_line, moved_lines, dropped_duplicates}]}`.
  - Human, for example:
    `merged  FIX, READ → BUILD + FIX + READ · 20261101.md line 14 · 3 links moved`.
- **Docs:** `docs/capture.md` gets grammar rows, a "Merging Pomodoros" section, and the
  retired-close table. Commit as `feat(capture)!:`, since a retired spelling is
  repurposed.

## Phase `capture-close-names` — `x` narrows and sheds (bob-cli)

Apply R6 inside the shared close planner, so whole-item `x`, link `=x`, and task closes
all agree. The planner is `capture_pomodoro_close/linked_tasks.rs::plan_pomodoro_close`,
and its ledger edits are in `capture_pomodoro_close/ledger.rs`.

- **Resolving types:** resolve each numbered link's type from pre-close lines through
  the staged snapshot (`SnapshotCloseVault` + `lookup_task`). Thread
  `CaptureTaskThemeContext::placement_for_task` from `capture/plan.rs` into
  `plan_pomodoro_close_item` and its siblings. Read the types before `stage_close_plan`
  runs.
- **Ledger edits:**
  - `close_entry_line` writes the narrowed name.
  - The created placeholder (`ledger.rs`, around lines 380–399) uses the shed name.
- **Two names, two jobs:** `ClosingContext.closed_name` both finds the closed entry and
  names the continuation (`task_complete/successors.rs` around 1339–1376). Split those
  jobs: locate by the pre-close name, name and reuse the continuation by the shed name.
- **JSON:**
  - Additive `pomodoro_close.closed_name`, present when the entry is named.
  - `next_pomodoro.name` carries the shed name.
  - Additive `next_pomodoro.shed: [ThemeRef]`.
- **Human:** `closed FIX 0920-0950 · from BUILD + FIX` and
  `next: BUILD (created) · carries 1 link · FIX done`.
- **Docs, `docs/capture.md`:** update "Closing the running Pomodoro" and "Choosing each
  Task Link's outcome". Restate the Obsidian parity promise ("byte-identical to
  Ctrl+Enter") and note that it now includes names (delivered by
  `obsidian-close-names`).
- **Tests:**
  - Extend the close unit and CLI tests with merged entries, labels, MEET, and a FIX
    link under each outcome.
  - Existing name-pinning tests (`tests/cli/capture/pomodoro_close.rs`,
    `capture_pomodoro_close/tests.rs`, `keep_moving_tests.rs`,
    `tests/fixtures/event_task_close/vectors.json`) must still pass unchanged, because
    single-type and untyped entries never rename.

## Phase `obsidian-link-merge` — pickers merge by default (bob-plugins)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Edit `src/` fragments only,
then run `npm run build`, `npm run docs:gen`, `npm test`, and `npm run validate`, and
finish with `bob plugins sync`.

- **Vectors:** copy bob-cli's `tests/fixtures/pomodoro_themes/vectors.json` byte for
  byte into _both_ copies (`plugins/bob-ledger-tools/scripts/fixtures/…` and
  `scripts/fixtures/…`) and update the README's source commit.
- **Ledger Tools mirror** (`279-plugin-task-type-routing.js`,
  `279-plugin-task-theme-api.js`):
  - Add `api.themes.planLanding` with the R2 semantics and an explicit `retype` option.
  - Keep `planRetype` as a v1 alias of `planLanding`.
  - Add `mergeNames` and `shed`.
  - Consume the `land`, `merge`, and `shed` vectors.
- **Ctrl+Shift+Enter link picker** (block-id-prompt `085-pomodoro-link-picker-modal.js`,
  `077-pomodoro-picker-intent.js`, `078-pomodoro-theme-write.js`):
  - **Enter** links and merges. The footer reads
    `→ BUILD + FIX · keeps FIX · ⌥↵ retype BUILD`, and the notice reads
    `Linked under BUILD + FIX · merged FIX`.
  - **Alt+Enter** retypes, using the Tab-cycled component.
  - `themeIntent` gains `retype` (true only for Alt+Enter). `linkOnce` is retired.
  - Update every in-repo consumer, including task-status-cycler's successor pick intent
    (`085b`), and `docs/develop/plugin-apis.md`.
- **Ctrl+Shift+M bullet move** (bob-navigation-hotkeys
  `200a-pomodoro-bullet-type-move.js`, `651-pomodoro-bullet-type-move-commit.js`,
  `165-pomodoro-theme-picker.js`):
  - **Enter** moves and merges missing types into the destination
    (`mergePomodoroBulletMoveTaskTypes`).
  - **Alt+Enter** retypes, with Tab cycling.
  - Consequence copy and toasts flip to match.
  - When `shed_on_move` is accepted, the source sheds via `shed`.
- **Other surfaces:**
  - The `^^` task wikilink in a Pomodoro sub-bullet (block-id-prompt
    `129-plugin-task-wikilink-pomodoro.js`) and the successor retarget
    (task-status-cycler `085c-successor-theme-write.js`) land by merging.
  - The Ctrl+X entry merge (`150-pomodoro-capture.js::planPomodoroEntryMerge`) dedupes
    names via `mergeNames`.
  - Pomodoro rename keeps retyping its linked tasks.
- **Docs:** `docs/block-id-prompt.md`, the "Pomodoro gestures" section of
  `docs/bob-navigation-hotkeys.md`, and `docs/develop/plugin-apis.md`.
- **Versions:** bump minor versions on every touched plugin's `manifest.json`.
- **Tests:** update the existing picker, move, and theme-write suites (Alt+Enter merge
  at `test-navigation-hotkeys-pomodoro-move-pickers.cjs`, and the consequence copy), and
  add cases for the default merge, Alt+Enter retype, MEET retype, and shedding.

## Phase `obsidian-ctrl-s` — Ctrl+S splits the selected Pomodoro (bob-plugins)

Bob Navigation Hotkeys, with a pure split mirror in Ledger Tools.

- **Pure planner:**
  - `api.themes.planSplit` mirrors `split_plan` and passes the `split` vectors.
  - A NAV planner (next to `planPomodoroBulletSplit` in `150-pomodoro-capture.js`) turns
    it into one undoable `applyEditorContentTransaction`. It moves top-level Task Link
    subtrees and inserts the `- [ ] () — T` entries below the source, in name order.
- **Link types:** resolve them with the same resolver the Ctrl+Shift+M preview uses
  (`resolvePomodoroBulletMoveTypePreview`). Unresolvable links stay with the base.
- **Selection:** the cursor on an entry line (`findPomodoroEntryContext`) or anywhere in
  its child block (`findPomodoroBulletContext`, mapped to its owning entry).
- **Key dispatch:**
  - Exactly Ctrl+S, with no Shift, Alt, or Meta, inside a Pomodoro block of the first
    `## Pomodoros` section, splits.
  - Everywhere else Ctrl+S is untouched: Obsidian's Save on Linux, nothing on macOS.
  - Implement it as a capture-phase keydown handler (precedent:
    `615-plugin-counted-task-move.js`) that stops propagation only when it acts.
  - If the harness shows Obsidian's built-in `Mod+S` cannot be pre-empted that way, fall
    back to wrapping `editor:save-file`'s callback (the `bob-articles` `030-plugin.js`
    pattern), acting only for a Ctrl+S keydown in a Pomodoro block on non-macOS. Restore
    it only if the callback is still this plugin's wrapper.
  - Also register the palette command **Bob: Split Pomodoro by type**
    (`split-pomodoro-by-type`). It has no registered default hotkey, and its
    `editorCheckCallback` holds only inside a Pomodoro block.
- **Behaviour:**
  - **Refusals** are notices with no edit: closed (`Closed Pomodoros are history`),
    MEET, one type (`BUILD has one type — nothing to split`), and container
    (`LATER holds any type — nothing to split`).
  - **Two types:** split the right one immediately. The notice reads
    `Split FIX out of BUILD + FIX · 2 tasks → FIX below`.
  - **Three or more types:** open a picker titled `Split BUILD + FIX + READ`, with a
    `BUILD stays` header.
    - Rows: a preselected `All · FIX, READ` row (the same as `=/`), then one row per
      merged-in type, with its hue swatch and link count (`FIX · 2 tasks`,
      `READ · no tasks`). The base is never offered.
    - **Tab** marks or unmarks type rows, following `SuccessorOfferPickerModal`'s
      `markedKeys` pattern. **Enter** applies the marked rows, else the highlighted row.
      **Esc** cancels.
    - The footer previews the resulting lineup, `BUILD + READ · FIX ↓`, and keys
      `↵ split · ⇥ mark · esc`.
  - **Cursor:** stays on its own line, and follows a moved bullet.
- **Docs:** the "Pomodoro gestures" prose and the "Editor and Vim handlers" layer in
  `docs/bob-navigation-hotkeys.md`, plus the manifest description. Bump minor versions.
- **Tests:** a new harness suite (added to `package.json`'s explicit `test` list) for
  two-type, three-type, all, marked, refusals, labels, nested subtrees, and key
  dispatch, including Ctrl+S outside a block passing through.

## Phase `obsidian-close-names` — Ctrl+Enter parity (bob-plugins)

Mirror R6 in task-status-cycler's Pomodoro completion (`completeActivePomodoroTask`,
`buildPomodoroCompletionPlan` in `060-pomodoro.js` / `170-plugin-pomodoro.js`):

- the closed entry's name is narrowed;
- the created placeholder (`formatPomodoroPlaceholderLine`) is shed;
- MEET entries are skipped.

Implementation:

- Resolve link types through ledger-tools `api.themes`.
- Use the `narrow` and `shed` vectors through a Ledger Tools `api.themes.narrowClosed` /
  `shed` mirror.
- Map the cycler's link classification to R6 outcomes.

Ledger results must stay byte-identical to bob-cli's `x` for the shared cases. Add
parity cases to the close suites, update the cycler docs, bump the version, and run
`bob plugins sync`.

## Phase `mac-capture` — present the new shapes (Bob Mac Capture)

Open the repo with `/sase_repo` and read its `AGENTS.md`. The app stays a thin client:
it decodes and presents what bob reports and never parses grammar.

- **Span kinds:** map `pomodoro_split` and `pomodoro_merge` to the Pomodoro-session
  highlight in `CompletionRowContent.swift`. Type parts are already `pomodoro_name`
  spans, with pills.
- **Modes and needs:** add the editor modes and the needs `pomodoro_split_type` and
  `pomodoro_merge_dest`. The picker is driven by bob candidates.
- **Preview cards:**
  - Split and merge cards are built from the `pomodoro_split` / `pomodoro_merge` result
    objects (before → after, created or absorbed rows with type pills and link counts).
  - The footer verbs are **Split** and **Merge**.
  - The link previews show the destination `merged` row (`merged FIX into BUGS + FIX`).
  - The close card shows `closed_name` and the `shed` types.
- **Compatibility:** decode with `decodeIfPresent`, so an older bob still works. Accept
  grammar 5.
- **Tests and CI:** add JSON fixtures and tests. macOS CI gates this phase, since agent
  hosts have no Swift toolchain.

## Phase `glossary` — memory (bob-cli)

> [!decision] memory_merged_pomodoro Use `/sase_memory_write`, then add the new
> `glossary:merged-pomodoro` strand (keyword **Merged Pomodoro**, aliases
> `merged pomodoro`, `composite pomodoro`, `base type`). It should cover:
>
> - the R1 definition and the base type;
> - that landing merges instead of retyping (Alt+Enter retypes);
> - `=*` / `=/` / Ctrl+S;
> - close narrowing and shedding;
> - MEET staying MEET;
> - what it is not: not a container, not a Pomodoro home, not D6 rename.
>
> Link it to `[[pomodoro]]`, `[[task-type]]`, `[[task-link]]`, and `[[capture]]`. Then
> run `sase memory init`.

> [!decision] memory_links_merge_record Add the new `decisions:links-merge-types` record
> (status accepted), titled "Linking Merges Types, Never Retypes". It covers:
>
> - **Claim:** R2.
> - **Why:** retype-on-link silently rewrote deliberate task classifications, and merged
>   names now carry the information.
> - **Rejected alternatives:** keeping Enter as retype, and prompting on every mismatch.
> - **Cost:** names can accumulate types, which close shedding mitigates.
> - **Reopens when:** Bryan finds himself pressing Alt+Enter on most links.
> - **Evidence:** this plan.
>
> Then run `sase memory init`. Answering **no** writes nothing.

## Verification

- bob-cli: `just` check targets (fmt, clippy, tests) green. Shared vectors pass in Rust.
  Dry-run demos:

  ```text
  printf -- '- [ ] (**0920-0950** [t:: 30m]) — BUILD + FIX + READ\n  - [[a#^b1]]\n  - [[a#^f1]]\n  - [[a#^r1]]\n'
  ```

  Seed that ledger in a temporary vault, then run `bob capture --dry-run '=/'`,
  `'=/#fix'`, `'=*build#fix'`, `'x!2'`, and a named link of a FIX task under BUILD. Each
  one prints the documented output.

- bob-plugins: `npm test` and `npm run validate` green, and the vectors copy is
  byte-identical. Then `bob plugins sync`.
- Bob Mac Capture: macOS CI green.

## Out of scope (follow-ups, not built here)

Shedding when a link leaves through unlink toggles, cancels, D8 relocation, or start
drops. A merged-Pomodoro budget view in `bob plan`. `bob types split` remains the
history-rewriting tool and is unrelated to `=/`.
