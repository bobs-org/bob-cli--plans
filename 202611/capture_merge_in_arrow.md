---
tier: epic
title: 'Capture `<`: merge existing tasks into the task you are creating'
goal: 'A capture line that creates one open task can end with `< @route+id …` to fold
  one or more existing tasks into that new task through the explicit merge engine,
  in one all-or-nothing capture whose vault bytes equal today''s two-item `>` draft.
  Misuse teaches the right spelling with one-key rewrites, and Bob Mac Capture previews
  the merge as a calm, concise purple band inside the ordinary new-task card.

  '
decisions:
  pull_in_title_count:
    ask: Should the merge-only Pull-In title count only sources that actually lost
      a live Task Link?
    default: true
    why: The title names the slot the new task takes; an unlinked source vacated no
      slot.
    answer: true
  memory_task_merge_glossary:
    ask: Update glossary:task-merge so its list of merge gestures includes capture
      `<`?
    memory:
    - glossary:task-merge
    default: true
    corrects: note section through `bob task merge`, capture's `>`, or the configured
      `merged into`
    answer: true
  memory_merge_in_decision:
    ask: Record a new decision that capture `<` is an explicit merge into the new
      task, kept a task item?
    memory:
    - decisions:capture-merge-in-arrow
    default: false
    answer: false
phases:
- id: contract
  title: Contract and docs
  depends_on: []
  size: medium
  description: 'contract: write the `<` contract into docs/capture.md, the capture-syntax
    reference, task-merge.md, and task-relations.md, with MI vectors and search-smoke
    rows, marked contract-only.'
- id: grammar
  title: Grammar, editor parse, and teaching refusals
  depends_on:
  - contract
  size: large
  description: 'grammar: add the shared merge-in scanner, claim it in execution and
    editor parsing before fan-out, gate it on a single open new task, ship structured
    rewrites and grammar 15, and refuse execution until the next phase lands.'
- id: execute
  title: One-transaction execution, output, and copy fixes
  depends_on:
  - grammar
  size: large
  description: 'execute: stage the new task, then run one explicit merge into it through
    a helper lifted from the `>` path; report merges, offers, origins, and pending
    context; fix bob-cli-hi merge copy; add the shell guard.'
- id: assist
  title: Pickers, rewrite toggle, and Similar tasks
  depends_on:
  - grammar
  size: medium
  description: 'assist: give `<` source slots a merge-only picker with ID-less rows
    and comma multi-pick, the `+` separator toggle, and capture-related source exclusion
    with a marker anchor before the arrow.'
- id: mac
  title: Bob Mac Capture merge-in band and pickers
  depends_on:
  - execute
  - assist
  size: large
  description: 'mac: render the Merging-in band inside the new-task card, pending
    and refusal states with rewrite buttons, merge-in pickers, folded landing groups,
    submit/status/notification/VoiceOver copy, and raise the grammar gate.'
- id: memory
  title: Memory notes
  depends_on:
  - execute
  size: small
  description: 'memory: apply the accepted memory decisions for the Task Merge glossary
    strand and the optional new decision record.'
proposed_by: bbugyi200.athena.190
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-05 18:05:38
status: wip
bead_id: bob-cli-hl
---

- **PROMPT:** [prompts/202611/capture_merge_in_arrow.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_merge_in_arrow.md)
- **BEAD:** [bob-cli-hl](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hl/README.md)

# Plan: Capture `<` — merge existing tasks into the task you are creating

## Context

Bryan asked for a `< @foo+f` capture syntax that complements `> @foo+f`: valid only
where a capture creates a new task, merging every listed task into the new task with the
existing "merged into" relation label, with an excellent but concise Bob Mac Capture
preview. He agreed with every requirement in the research report
`research:202611/capture_merge_in_arrow/capture_merge_in_arrow.md` (read it with
`sase artifact read research:202611/capture_merge_in_arrow/capture_merge_in_arrow.md "<why>"`).
This plan adopts that report's design (R1–R14, the claim rule, codes, preview, and MI
vectors) and pins every detail it left open.

What changed since the research snapshot, verified on master `7c71241f`:

- **Grammar 14 is taken** (`86554bf6`, past-Pomodoro segments). `<` ships as **grammar
  15**, or the next free revision if epic `bob-cli-hj` bumps first. Every "15" below
  means "the revision the grammar phase ships".
- **`bob-cli-hi`** (READY bug) records three merge copy defects in the renderer this
  epic touches: G4 "1 notes moved" (`src/native/task_merge/plan.rs:656`), G2 label
  merges print no merge rows (`src/native/capture/output.rs:3677` gates
  `print_human_transfer_merges` on `kind == "task_merge"`), and G5 the Pull-In title
  counts unlinked sources (`src/native/capture/pull_in_copy.rs:611–618`). The execute
  phase absorbs them.
- **Concurrent work:** epic `bob-cli-hj` (phases hj.4 and hj.6) is changing capture
  grammar, `capture-rewrite`, and the Mac editor. Mac macOS CI has pre-existing red
  beads (`bob-cli-h7`, `bob-cli-fu`, `bob-cli-fp`), and the Mac `docs/protocol.md` is
  already over its 6000-word mounted budget (`bob-cli-h9`). Rebase often; never "fix"
  those failures inside this epic, but never add new ones.
- `docs-check` enforces capture token coverage: every token-index row in
  `docs/reference/capture-syntax.md` needs a literal query in
  `docs/_meta/search_smoke.yml` (see `bob-cli-fo`).

Today's behavior that this epic replaces (installed parser):
`Some great new task! @foo^great < @foo+f @bar+b` files the note
`Some great new task! @foo^great <` under **both** `^f` and `^b`; `Remind me < @work`
creates the task `Remind me <` in `work.md`. That is why this is a grammar break shipped
with `!` and a raised Mac grammar gate.

## Design

This section is the contract. The `contract` phase writes it into the docs; later phases
implement exactly it. Governing decisions: `decisions:mac-capture-is-a-thin-client` (Bob
owns grammar, copy, and edits; the Mac renders),
`decisions:capture-sigils-say-where-what-how`,
`decisions:bob-cli-releases-ride-a-daily-train` (`!` on contract breaks; never touch the
Cargo version or `CHANGELOG.md`), and `decisions:similar-tasks-are-a-settled-heads-up`.

### Meaning

> `T < S₁ … Sₙ` creates **T** exactly as the left side says, then runs the existing
> **explicit** task merge (`MergeGesture::Explicit`) of `S₁ … Sₙ` into T, in typed
> order, inside the same capture transaction.

- It is a real merge, not a link: each source's notes move under T after T's own
  authored children (MG20), each source is cancelled with `🪢 LINKS: merged into [[T]]`,
  T's LINKED FROM gains the inverse entry, open dependents re-point to T, today's live
  Task Links to the sources are pruned, and T is offered once in their place.
- The label is the first configured `relations.merge_labels` entry, falling back to
  `merged into` — the same rule as `>`. Empty `merge_labels` never disables `<`.
- T owns its fields. Nothing is inherited from sources (status, type, priority,
  schedule, prerequisites); lost prerequisites are warned (`prerequisites_left`).
- **Equivalence property:** `T @n^t < @a+x @b+y` writes the same vault bytes as the
  two-item draft `T @n^t` ⏎⏎ `@a+x @b+y > @n+t`.
- All or nothing: a bad source, a refusal, or a later failing item in a bulk draft
  leaves no T and no retired source. `--dry-run` equals the commit except `dry_run`, and
  `bob task reconcile` afterwards is a no-op.
- T gets a block ID automatically when none is typed (existing minting), shown in the
  preview; a dry run persists nothing.
- T's route is never inferred from the sources: it routes like any new task.

### Grammar and the claim rule

```text
<ordinary new-task line> < <source> [<source> …]        source := @route+block-id
```

A new scanner, `scan_merge_in_line` in `src/native/capture_language/merge_in.rs`, runs
on parent lines and task-list bullet lines **after** `scan_transfer_line` leaves the
line unclaimed and **before** target-run fan-out. Both `draft.rs` (execution) and
`editor_parse.rs` (editor) call it, so highlighting can never disagree with execution.
It never runs on authored child lines, under forced `-r`/`--task`/`--drawer`
destinations (which keep every `@` literal, as for `>`), or inside protected markup
(code, wikilinks, autolinks).

A **standalone arrow** is a token that is exactly `<` (whitespace on both sides).
Scanning left to right, the first standalone `<` whose following token is claimable
claims the line:

| Draft                                                                       | Result                                                                                                                           |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `New outcome @foo^great < @foo+f @bar+b`                                    | **Valid**: create one task, merge two sources in typed order                                                                     |
| `New outcome < @foo+f`                                                      | **Valid**: T routes normally (inbox) and gets a minted ID                                                                        |
| `New outcome @foo^great <`                                                  | **Incomplete** (need `merge_in_source`): calm hint “type + to pick tasks to merge in”; submit refused with `incomplete_merge_in` |
| `New outcome < +fla`, `< +`, `< @foo+`, `< @foo+f @bar+`                    | **Incomplete**: vault picker or note-scoped picker; complete sources stay listed                                                 |
| `New outcome < @foo`                                                        | **Incomplete**: a note is never a source; type `+` to pick a task from it                                                        |
| `New outcome < @foo:f`, `< @foo^f`, `< @foo+f#d`, `< @foo+f!`               | **Invalid**: reused `transfer_source_marker` / `transfer_source_suffix`                                                          |
| `New outcome < @foo+f @foo+f`                                               | **Invalid**: `duplicate_capture_target` on the second occurrence                                                                 |
| `New outcome < @foo+f s:3`, a second `<`                                    | **Invalid** `merge_in_extra_text`, rewrite moves the extra tokens before `<` (a stray `<` is dropped)                            |
| `text<@a+x`, `text <@a+x`, `text< @a+x`                                     | **Invalid** `merge_in_arrow_spacing`, rewrite adds the spaces                                                                    |
| `@foo+t < @bar+b`                                                           | **Invalid** `merge_in_needs_new_task`, rewrite to `@bar+b > @foo+t`                                                              |
| `< @a+x` (nothing before the arrow)                                         | **Invalid** `merge_in_needs_new_task`: “name the new task before <”                                                              |
| `Latency < 50ms @foo`, `a < b`, `<=`, `<<`, `<https://…>`, `` `a < @b+x` `` | **Unclaimed**: unchanged prose                                                                                                   |
| `Escaped \< @foo+f @work`                                                   | **Escaped**: see below                                                                                                           |

Claimable means: nothing (terminal `<`), a source-like token (`@…` containing `+`, `:`,
or `^`), a bare route `@route`, or a `+`/`+query` parent-task selector. Anything else (a
word, number, operator) leaves that `<` as prose and the scan continues. After the first
source token, every following token must be a complete source, except that the final
token may be a partial (`+query`, `@route`, `@route+`); anything else is
`merge_in_extra_text`. Glued spacing errors are claimed only when the glued `<` touches
a source-like token, so `<=`, `=>`, `<3`, and autolinks stay prose.

**Escape.** `\<` at the arrow position escapes the whole clause: the backslash is
dropped and everything from `<` to the end of the parent line stays literal task text,
so nothing after it routes or fans out. Everything before it parses exactly as it would
alone. `Escaped \< @foo+f @work` creates the inbox task `Escaped < @foo+f @work`. This
follows the research, which Bryan accepted; it deliberately differs from `\>`, whose
following tokens still parse, because a literal `<` followed by task markers should
never silently file notes under those tasks. Pin it with a test.

### Where `<` is valid (the context gate)

The left side is parsed as an ordinary item from a **masked view** of the draft (the
clause bytes from the arrow to the end of the parent line replaced by spaces, so every
byte offset is preserved — the same view mechanism the transfer escape path uses). The
clause is valid only when that left parse yields **exactly one** item (no target-run
fan-out) that creates one task still open at commit:

| Left side                                                                                          | `<`                                                                                                                   |
| -------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Plain text, `@route text`, `@@`-inherited route, `@route^id text`, `%c` clip, `%t` row as expanded | ✅ (`CaptureKind::Task` / `TaskWithBlockId`)                                                                          |
| `text @route:id` (Pomodoro-linked new task)                                                        | ✅ (`CaptureKind::Pomodoro` with no start and no close); MG10 then suppresses the offer                               |
| First-level bullets under a lone `@route` task-list header                                         | ✅ per bullet (unlike `>`, which refuses there with `list_bullet_marker`)                                             |
| T with authored child bullets or drawers, `&`, `*`, `~`, `s:N`, `$N`, `#type`                      | ✅ T's authored notes land first, merged groups after                                                                 |
| `:id=` (start) or `=x` (born closed)                                                               | ❌ `merge_in_needs_new_task`: “the new task must stay open to take in other tasks; start or close it in a later item” |
| Project-note creation `@r^id@` / `@r:id@` and its bullets                                          | ❌ `merge_in_needs_new_task`: “a new project keeps tasks as tasks: list them as - ^id bullets to move them in”        |
| Existing-task lines (`@c+id` toggles, sub-bullets, target runs, dependency-only items)             | ❌ `merge_in_needs_new_task`, plus the `>` rewrite when the left is exactly one valid `>` destination token           |
| Note/section bullets, URLs/refs, Pomodoro operators, everything else                               | ❌ `merge_in_needs_new_task`: “this line creates no task”                                                             |

The `>` rewrite applies whenever the whole left side is one token that `transfer.rs`'s
destination parser accepts (`@c`, `@c+id`, `@c+id#drawer`, `@c#section`):
`@foo+t < @a+x @b+y` → `@a+x @b+y > @foo+t`, and `@db < @a+x` → `@a+x > @db`.

### Codes, copy, and structured rewrites

Bob authors every string. Each refusal carries the item number and the offending UTF-8
byte span.

| Code                          | When                                                                                                                                                                                                                                            | Copy                                                                                                                                               |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `merge_in_needs_new_task`     | left names an existing task, creates no task, is empty, starts/closes a session, or creates a project                                                                                                                                           | “< merges into the task this line creates; ^t already exists. Write @bar+b > @foo+t” / the context-table copy above                                |
| `merge_in_extra_text`         | tokens after the sources, or a second `<`                                                                                                                                                                                                       | “everything about the new task goes before <; only @note+id tasks follow it”                                                                       |
| `merge_in_arrow_spacing`      | glued arrow next to a source                                                                                                                                                                                                                    | “put spaces around <: New task < @a+x”                                                                                                             |
| `incomplete_merge_in`         | submit with a terminal `<`, a bare `@route`, or an unresolved picker                                                                                                                                                                            | “pick tasks to merge in after <, or write \< to keep it”                                                                                           |
| `merge_in_source_also_marked` | `&`/`*`/`~`/`~~` on the item names a source, or the item carries any merge-label `~~`                                                                                                                                                           | ““S” merges into the new task; it can't also be its {prerequisite \| dependent \| relation}” / “use < for every task you merge into this one”      |
| `merge_in_unavailable`        | transitional only (grammar phase → execute phase)                                                                                                                                                                                               | “merging into a new task with < needs a newer bob”                                                                                                 |
| reused                        | `transfer_source_suffix`, `transfer_source_marker`, `duplicate_capture_target`, `transfer_overlapping_sources`, `merge_source_closed`, `merge_source_recurring`, `merge_source_project`, `merge_source_open_subtasks`, `merge_dependency_cycle` | unchanged, except `merge_source_closed` names an existing merge: “Fix the flaky login test” is already merged into “Some great new task!” (foo.md) |

**Structured rewrites.** `capture-parse` diagnostics gain one additive, optional key,
omitted when absent:
`"rewrite": {"start": N, "end": M, "replacement": "…", "label": "…"}` — a half-open
UTF-8 byte range in the exact draft, Bob's replacement text, and Bob's button label.
This epic emits it for `merge_in_needs_new_task` (label `Use >`, only for the
one-destination-token case), `merge_in_extra_text` (label `Move before <`), and
`merge_in_arrow_spacing` (label `Add spaces`). Clients apply it only as an explicit
draft edit; Return never changes. The mechanism is general so later grammar can reuse
it.

### Editor contracts

**`capture-parse`** keeps the item's own `mode` (`task` / `pomodoro_task`) whenever the
clause is valid or pending; this keeps highlighting and the Similar-tasks gate working.
A pending clause adds need `merge_in_source` plus the usual `interactive_placeholder`
span on the partial token and **no error diagnostic** (pending never flashes red). New
span kinds `merge_in_arrow` and `merge_in_source` (one span per whole source token; do
not reuse `transfer_*`, which the Mac keys transfer-only picker logic on). One additive
item object, omitted when the line has no clause:

```json
"merge_in": {
  "arrow":   {"start": 32, "end": 33},
  "sources": [
    {"start": 34, "end": 40, "marker": "@foo+f", "route": "foo", "block_id": "f"},
    {"start": 41, "end": 47, "marker": "@bar+b", "route": "bar", "block_id": "b"}
  ]
}
```

`grammar` reports 15; `schema_version` stays 1.

**`capture-complete`** in a `<` source slot:

- `< +` / `< +query` opens the vault-wide `task_parent` picker in a new merge-in source
  role; `< @route+` uses the note-scoped task context; `< @` completes routes.
- Rows are **mergeable only** (open, not `^prj`, not recurring, no open subtasks — the
  existing `mergeable_source_ids` rule), excluding already-typed sources and aliases.
- ID-less mergeable rows stay, with `requires_block_id: true` and their `route`, so the
  client's existing Add block ID flow (`bob capture-task-id --route`) returns
  `parent_replacement` (`@route+id`). Apply the same ID-less rule to `>` source slots
  (movable or mergeable rows), per research R9.
- Accepted rows return `continuation: " +"`, so `,` opens the picker for the next
  source.
- Each row carries a Bob-authored detail `⤵ merge in · 2 notes` (`· title only` when no
  notes would move).

**`capture-rewrite`:** in a `<` source slot, `+` typed right after `@route:id` or
`@route^id` switches to `@route+id`, exactly as on a `>` line; `:`/`^` after a source
`+id` stays literal.

**`capture-related`:** every task named in the item's `<` clause appears under `linked`
with `link: "merge_in"` and never in `matches`; the similarity query uses the left
side's body only; `marker_at` sits before the arrow (so Bob-verified `relate`/`needs`/
`blocks` splices land on the left side and stay valid).

### CLI output and JSON

- The item keeps `kind: "task"` / `"pomodoro_task"` and gains the existing flattened
  `merges[]`, `merge_offers`, and `repaired_files[]`; offers join the draft-level
  `pull_in` and its numbering.
- `merges[].target.created` is amended to mean “the container did not exist before this
  capture”: a drawer or section the merge created, **or a task created by this item or
  an earlier item of the same draft** (so the two-item `>` equivalent reports it too).
- `task_blocks`: T appears with roles `["merge_target"]` and `created: true`; each
  source appears with `merge_source`. Roles come from `merges[]`, never from `kind`.
- New additive `origin` on `task_blocks[].lines[]` rows: on every row a merge in this
  capture landed under its target (each group's title bullet and moved lines), the
  source identity `"<relative_path>#^<block_id>"` matching `merges[].source`; omitted on
  every other row. Applies to `>` and `<` merges alike.
- New additive `merge_in_pending` on dry-run JSON failures whose code is
  `incomplete_merge_in`, mirroring `transfer_pending`: `item` (1-based), `need`, and
  `sources[]` (each typed complete source with resolved `path`, `line`, `title`, task
  status, and whether it is mergeable, or `error: {code, message}`).
- Human output prints the merge block whenever `merges[]` is non-empty (this also fixes
  `~~` merge labels, bob-cli-hi G2). For a capture-born target the header reads
  `⤵ Merged 2 tasks into new “T” · foo.md` (`⤵ Would merge … into new “T”` on dry run).

```text
✓ captured  foo.md
  - [ ] #task Some great new task! [type:: FIX] [created:: 2026-11-05] ^great
⤵ Would merge 2 tasks into new “Some great new task!” · foo.md
  ⊘ Fix the flaky login test         foo.md:1 · Next → Cancelled · 2 notes moved
  ⛓ “Waits on f” now waits on “Some great new task!”
  ◌ BUILD link removed · offer 1: link “Some great new task!” under BUILD (-p 1)
  ⊘ Login test timeout is too short  bar.md:4 · Ready → Cancelled · 1 note moved
```

- Shell guard: when draft arguments are present and stdin is a regular file whose
  basename starts with `@`, refuse with `shell_ate_merge_in_arrow`: “your shell ate “<”:
  quote the draft (bob capture 'New task < @a+x')”. Mirror the stdout guard in
  `src/native/capture/shell_guard.rs` (Linux `/proc/self/fd/0`, macOS `F_GETPATH`).

### The Bob Mac Capture preview

Principles: the card reads like what you typed (the new task stays the card's subject
and header); reuse, don't fork (`CaptureTransferCardView`'s source-row,
status-transition, consequence-row, and Pull-In subviews and merge-purple tint; h4's
`Merge target` / `Merged` capsules and tombstone folds); concise by default, complete on
demand.

**Never route a `kind: task` item through `CaptureTransferPresentation`**: that card
replaces the whole new-task card, including the title header, type, schedule, and the
Similar-tasks band (`Sources/BobMacCapture/CapturePanelView.swift` `previewItem`).

State 1 — arrow typed, nothing picked (`Some great new task! @foo^great <`):

```text
  ○ Some great new task!                 foo.md · ^great  new
    ⤵ Merge tasks into this one — type + to pick           (calm, no error red)
```

State 2 — picker open (`… < +fla`): picker header `⤵ Merge into “Some great new task!”`,
rows with status, note, and Bob's `⤵ merge in · 2 notes` detail; ID-less rows show
`Add block ID`; footer `↩ pick · , pick another`.

State 3 — complete (`Some great new task! @foo^great #fix < @foo+f @bar+b`). Titles and
counts below are illustrative:

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ Some great new task!                         foo.md · ^great   ┆new┆   │  standard header
│   Ready · #fix                                                           │
│ ┃ ⤵ Merging in 2 tasks                                                   │  the band
│ ┃ ⊘ Fix the flaky login test        foo.md:1  Next → Cancelled · 2 notes │
│ ┃ ⊘ Login test timeout is too short bar.md:4  Ready → Cancelled · 1 note │
│ ┃   ⛓ “Waits on f” now waits on this task                                │
│ ┃   ⚠ “Login test timeout…” waited on “Upgrade CI runners”; this won't   │
│ ┃   [Pull in] Link this task under BUILD in place of “Fix the flaky…”    │
│   Similar tasks (only non-source matches that clear Bob's bar)           │
└──────────────────────────────────────────────────────────────────────────┘
  foo.md · ^great   [New] [Merge target]
    - [ ] #task Some great new task! [type:: FIX] … ^great
      - 🪢 LINKED FROM: [[#^f]] (merged into) • [[bar#^b]] (merged into)
      ⤵ Fix the flaky login test · 2 notes ▸
      ⤵ Login test timeout is too short · 1 note ▸
  foo.md:1  [Merged]  - [-] Fix the flaky login test …  ⤵ 2 lines moved
  bar.md:4  [Merged]  - [-] Login test timeout …        ⤵ 1 line moved
                                                     Capture & merge 2   ⌘↩
```

Band rules:

- Built by a new `CaptureMergeInPresentation` (in `CaptureCore`) from `merges[]` when
  the item's kind is `task` or `pomodoro_task`; rendered inside `standardPreviewItem`
  under the header and above the Similar-tasks band. Because it keys on `merges[]`,
  `T ~~old:a|"merged into"` gets the same band.
- One row per source: title first, `note:line` secondary, a visible text transition
  (`Next → Cancelled`, not tint alone), and the moved-note count (`title only` at 0).
  Show all rows for 1–3 sources; for more, three plus a keyboard-accessible
  `Show all N`. Never hide a blocking error, the aggregate count, or a meaning-changing
  warning.
- Consequences are Bob's `consequences[]` text verbatim, de-duplicated by `(kind, keys)`
  so a dependent of two sources appears once. `prerequisites_left` is the ⚠ row.
- The Pull-In offer is explicit and never pre-selected. If T is born Blocked (`s:N`,
  `&`) there is no offer and the band says the source's link leaves.
- T's exact landing block appears once: when the item has a created `merge_target` task
  block, omit the item's verbatim `previewBlockLines` (keep the header). Each run of
  lines sharing an `origin` folds to one disclosure row `⤵ <source title> · N notes ▸`;
  apply the same fold to `>` merge targets for parity. Tombstone cards keep h4's shipped
  `tombstones = cards` behavior.
- Pending: built from the parse (`mode` task + need `merge_in_source` + `merge_in`) and
  Bob's `merge_in_pending`; never red; complete sources stay listed; a source that
  resolves but cannot merge shows Bob's reason inline and Submit stays unavailable.
- Refusal: Bob's message plus one button per diagnostic `rewrite` (label from Bob).
- Responses stay tied to the draft revision; a stale response is never shown as current.

Copy:

| Surface         | Text                                                                                                                                                                                          |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Footer / Return | `Capture & merge N` for a single-item draft whose item merges (ML7's `Capture & merge {N} task(s)`); bulk drafts keep `Capture`                                                               |
| Status line     | `Would create “Some great new task!” and merge 2 tasks into it · foo.md`                                                                                                                      |
| Notification    | `Created “Some great new task!” · merged 2 tasks`; body one line per source `Fix the flaky login test · Next → Cancelled · 2 notes`                                                           |
| VoiceOver       | “New task Some great new task!, foo.md, new. Merging in 2 tasks: Fix the flaky login test, Next to Cancelled, 2 notes; …” — announced when settled, never per keystroke, without moving focus |
| Grammar gate    | `CaptureGrammarGate.minimumRevision` = 15, blocked copy `Update bob to merge tasks with <`                                                                                                    |

### Compatibility and rollout

| App / bob         | Behavior                                                                                                                    |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------- |
| New app / new bob | Full band, pickers, rewrite buttons, one composite commit                                                                   |
| New app / old bob | Blocked by the raised grammar gate with the update copy: a `<` draft must never reach a parser that files sub-bullets       |
| Old app / new bob | Correct writes; the preview shows the standard new-task card plus `Merge target` / `Merged` landing cards, without the band |
| CLI               | No skew; the grammar phase's `feat(capture)!:` commit body says plainly that `X < @a+x` used to file a note                 |

Ship order: bob phases land on master first; the Mac phase raises the gate last.

### Acceptance vectors

The `contract` phase records these in `docs/task-merge.md` (an MI index beside the MG
index, with test names filled in by the implementing phases).

| ID   | Covers                                                                                                                                                                                                                        | Phase                            |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| MI1  | `T @n^t < @a+x`: T created; x's title bullet and notes land under T; x is `[-]` with `merged into [[n#^t]]`; T has LINKED FROM                                                                                                | execute                          |
| MI2  | Several sources keep typed order after T's authored children and drawers                                                                                                                                                      | execute                          |
| MI3  | Equivalence: `<` item ≡ two-item `>` draft, vault bytes identical                                                                                                                                                             | execute                          |
| MI4  | Same-note T and source stage one postimage; same-note descendant-anchor collisions                                                                                                                                            | execute                          |
| MI5  | Dependents re-point to T; a cycle through T's own `&` refuses; shared dependents counted once                                                                                                                                 | execute                          |
| MI6  | Closed, recurring, `^prj`, open-subtask, duplicate, alias, overlapping, and missing sources refuse; vault byte-identical                                                                                                      | execute                          |
| MI7  | Every ❌ row of the context table refuses with its code; `@n+t < @a+x` and `@db < @a+x` carry the `>` rewrite                                                                                                                 | grammar                          |
| MI8  | Lexical: prose `<`, `<=`, `<<`, autolinks, code, glued (rewrite), `\<` whole-clause escape, terminal `<`, `< +q`, `< @r`, `< @r+`, extra text (rewrite), second `<`, Unicode and multi-line spans, editor/execution agreement | grammar                          |
| MI9  | Offers: plain T offered at the earliest vacated slot; `:id` T gets no offer (MG10); `s:N`/`&` T gets no offer while the prune lands                                                                                           | execute                          |
| MI10 | `merge_labels: []` still merges and writes `merged into`                                                                                                                                                                      | execute                          |
| MI11 | Re-submit: explicit-ID collision refuses; auto-ID retry refuses with “already merged into …”; zero writes                                                                                                                     | execute                          |
| MI12 | Atomicity: a bad second source or later bulk item leaves no T and no retired source; injected guarded-write failure rolls back                                                                                                | execute                          |
| MI13 | Task-list header with `- T < @a+x` bullets works per bullet                                                                                                                                                                   | grammar (parse), execute (write) |
| MI14 | `&`/`*`/`~`/merge-label `~~` naming a source, or any merge-label `~~`, refuses `merge_in_source_also_marked` (lexical and resolved aliases)                                                                                   | grammar, execute                 |
| MI15 | ID-less T mints an ID; dry run persists nothing; preview and `merges[]` show it                                                                                                                                               | execute                          |
| MI16 | Dry run ≡ commit under a fixed clock and snapshot; `bob task reconcile` is a no-op afterwards                                                                                                                                 | execute                          |
| MI17 | JSON: kind stays `task`; `target.created: true` (also for the two-item `>` equivalent); task-block roles and `origin`; `merge_in_pending`                                                                                     | execute                          |
| MI18 | Pickers: merge-only rows, ID-less rows, exclusions, continuation, `+` toggle, capture-related exclusion and `marker_at`                                                                                                       | assist                           |

## Phase: contract

Docs only; no Rust. Read `sase memory read docs.md -r "<why>"` first.

- `docs/capture.md`: a new section **Merging tasks into a new task with `<`** right
  after “Moving and merging tasks with `>`”, covering Meaning, the claim table, escape,
  the context table, codes and copy, the structured `rewrite` key, editor contracts,
  JSON, human output, and the shell guard. Add rows to the “Grammar at a glance” marker
  table. Cross-link `>` and `<` both ways. Add the `merge_in` object, span kinds, need,
  and `rewrite` to the `capture-parse` section; the merge-in role, ID-less rows, and
  continuation to `capture-complete`; the toggle to `capture-rewrite`; the `merge_in`
  link kind and `marker_at` rule to `capture-related`; `origin`, `merge_in_pending`, and
  the amended `target.created` to the JSON sections (task blocks, transfer section).
- `docs/reference/capture-syntax.md`: a token-index row (`T < @a+x @b+y`) and a
  `{#merge-in}` section; add matching literal queries to `docs/_meta/search_smoke.yml`.
- `docs/reference/json-outputs.md`: extend the `bob capture`, `capture-parse`, and
  `capture-complete` rows with the additive keys above (`merges[]` on task items,
  `origin`, `merge_in_pending`, `merge_in`, span kinds, need, `rewrite`, the merge-in
  picker role).
- `docs/task-merge.md`: list `<` among the gestures in the intro and §8; amend
  `target.created` in §9; add §10 copy rows (new-target header, `merge_in_*` codes,
  improved `merge_source_closed`, singular `1 note moved`); add the MI index after MG.
- `docs/task-relations.md` ML6: add “`Umbrella < @old+a` ≡
  `Umbrella ~~old:a|"merged into"` without config, with explicit-merge refusals” and
  note `<` is the capture spelling of merge-in; ML3's table gains a `<` row.
- Mark every section that describes unshipped behavior `*Status: contract only.*`, the
  convention `docs/task-relations.md` already uses; the execute and assist phases remove
  those marks when their behavior lands. Do not change the documented grammar revision
  yet (the grammar phase does).
- Gate: `just docs-check` green.

## Phase: grammar

Read `src/native/capture_language/transfer.rs`, `draft.rs` (`parse_transfer_claim`,
`parse_capture_item_members`), `editor_parse.rs` (`parse_editor_item_with`), and
`editor_model.rs` first.

1. **Shared source helpers.** Move `is_source_like`, `parse_complete_source_text`,
   `parse_source_token`, and `find_lexical_duplicate` out of `transfer.rs` into a shared
   module (for example `capture_language/task_sources.rs`); `transfer.rs` re-uses them
   and every existing transfer test stays green byte-for-byte.
2. **Scanner.** `capture_language/merge_in.rs`:
   `scan_merge_in_line(parent, source, has_children) -> MergeInScan` with `Unclaimed`,
   `Escaped`, `Incomplete`, `Valid {arrow, sources}`, and
   `Invalid {code, message, span, rewrite}`, implementing the claim table exactly.
3. **Execution claim** (`draft.rs`): after `parse_transfer_claim` returns `None`, run
   the merge-in claim (task-list bullets included). On `Valid`, parse the masked view
   through the ordinary member parser, apply the context gate, and attach the clause to
   the single `ParsedCaptureText` (add an `Option<MergeInClause>` field, `None`
   everywhere else). `Escaped` keeps the clause as literal body text. `Incomplete` →
   `incomplete_merge_in`; `Invalid` and gate failures → their coded errors.
4. **Editor claim** (`editor_parse.rs`): the same claim and gate; left-side spans from
   the masked parse; `merge_in_arrow` / `merge_in_source` spans; the `merge_in` object;
   need `merge_in_source` with the placeholder span on a pending partial; diagnostics
   with `rewrite`. Add `SpanKind::{MergeInArrow, MergeInSource}`, `Need::MergeInSource`,
   and the optional `Diagnostic.rewrite` (serialized only when present).
5. **Lexical conflicts:** `&`/`*`/`~`/`~~` on the item whose note and ID match a source
   (lower-cased route) → `merge_in_source_also_marked`.
6. **Transitional refusal:** the planner (`plan_capture_item`) refuses any item carrying
   a clause with `merge_in_unavailable`, so no master build ever creates T while
   silently skipping the merge. The execute phase removes it.
7. Audit every `scan_transfer_line` call site (`draft.rs`, `editor_parse.rs`,
   `completion.rs`, `rewrite.rs`) and make the merge-in parity decision explicit. The
   completion and rewrite behavior itself belongs to the assist phase; here, only make
   sure a claimed `<` line never falls through to target-run completions or rewrites
   that would corrupt it.
8. Bump `GRAMMAR_REVISION` (15, or next free), update `docs/reference/compatibility.md`,
   the revision sentence in `docs/capture.md`, and the `grammar: N` mentions in
   `docs/reference/json-outputs.md`, and remove contract-only marks for the grammar and
   `capture-parse` parts.
9. Tests in `src/native/capture_language/tests/merge_in.rs`: MI7, MI8, MI13 (parse),
   MI14 (lexical), and an editor-versus-execution agreement test over the claim table.
10. Commit as `feat(capture)!: …`; the body names the break (`X < @a+x` and `X < @route`
    used to write text or notes).

## Phase: execute

Read `src/native/capture/transfer.rs` (`plan_transfer_merge`,
`transfer_task_block_refs`, helpers), `relations_merge.rs`, `relations.rs`
(`ensure_owner_block_id`, `locate_relation_owner`), `relations_mirror.rs`
(`mint_and_append_block_id`), `plan.rs` (`plan_capture_item`, the relations block),
`output.rs`, `pull_in_copy.rs`, `transfer_pending.rs`, `shell_guard.rs`, and
`src/native/task_merge/`.

1. **Lift one helper.** Extract
   `plan_explicit_merge(request, planner, ctx, snapshot, sources, target: MergeContainer, warnings)`
   from `plan_transfer_merge` (context construction, label rule, `plan_task_merge`,
   `stage_merge_plan`, offer rows, warnings). `>` calls it unchanged; all transfer and
   MG tests stay green.
2. **Stage the merge** in `plan_capture_item` after T, its dependencies, `*`, and
   relations are staged: settle T's block ID (typed `^id`/`:id`, an ID already minted by
   `&`/`*`/`~`, else mint with the shared helper) and report it through the item's
   existing `block_id`, `block_link`, and `task_line`; reload the staged snapshot;
   resolve sources (`resolve_transfer_note`), refuse resolved duplicates, aliases,
   overlaps, and `merge_in_source_also_marked` against the item's resolved `&`/`*`/`~`
   targets and any merge-label edge; then call the helper with
   `MergeContainer::Task(T)`. Remove the transitional `merge_in_unavailable` refusal.
3. **Results:** `merges[]`, `merge_offers`, `repaired_files[]`, draft-level `pull_in`;
   task-block refs (T `MergeTarget`, created; each source `MergeSource`); amend
   `target.created` per the Design for capture-born targets (both arrows); add the
   `origin` line key; `merge_in_pending` on dry-run JSON failures (mirror
   `transfer_pending.rs`).
4. **Copy:** human merge block whenever `merges[]` is non-empty, with the new-target
   header (bob-cli-hi G2); `1 note moved` singular in `task_merge/plan.rs` and its tests
   (G4); `merge_source_closed` names an existing merge target.
   > [!decision] pull_in_title_count When on, the merge-only Pull-In title counts only
   > records that pruned a live link (`links_pruned > 0`), so `Link “T” in place of “S”`
   > names the one source that held a slot (bob-cli-hi G5). When off, keep today's count
   > of every merge record.
5. **Shell guard** `shell_ate_merge_in_arrow` with unit tests for the fd-0 path probe.
6. **Tests** in `src/native/capture/tests/` (and `task_merge/tests.rs` where the engine
   changes): MI1–MI6, MI9–MI12, MI13 (write), MI14 (resolved), MI15–MI17; MI3 as a
   property-style test over several fixtures; MI16 runs production reconcile.
7. Docs: remove contract-only marks for execution, JSON, and output; fill MI test names.
8. When G2, G4, and G5 are all resolved, close `bob-cli-hi` with
   `sase bead close bob-cli-hi --note "<commit and tests>"`; if G5 was declined, add a
   `sase bead note` saying G2/G4 shipped and G5 stays open instead.

## Phase: assist

Read `src/native/capture_language/completion.rs` (`TransferCompletionRole`,
`transfer_completion_field_at`), `src/native/capture_complete/transfer_assist.rs`,
`capture_complete/cli.rs` (continuation), `capture_language/rewrite.rs`, and
`src/native/capture_related.rs`.

- Add a merge-in source completion role covering every `<` source slot position, routed
  through the existing `TaskParent`, `Task`, and `Route` contexts; scope rows
  mergeable-only with exclusions, keep ID-less rows with `requires_block_id` and
  `route`, return `continuation: " +"`, and add the `⤵ merge in · N notes` detail.
- Let `>` source slots keep ID-less movable-or-mergeable rows the same way (R9).
- `capture-rewrite`: the `+` toggle in `<` source slots.
- `capture-related`: `linked` rows with `link: "merge_in"`, exclusion from `matches`,
  left-only query, and `marker_at` before the arrow; verify a `relate` action on a `<`
  draft re-parses as valid.
- Tests for MI18; docs: remove the matching contract-only marks.

## Phase: mac

Work in the `bob-mac-capture` linked repo (open it with `/sase_repo`; read its
`AGENTS.md`). Compile everything you touch: `just portable-check`, a best-effort Mac
scratch compile of `Sources/BobMacCapture*` per `AGENTS.md`, `Scripts/macos-ci-check`
before landing, and after every push watch the exact-SHA macOS run with `/sase_monitor`.
Pre-existing red tests tracked by `bob-cli-h7`/`bob-cli-fu` are not yours; prove you
added no new failures and say so. If the Mac is offline, say the macOS targets are
unverified.

- Decode the new parse fields (`merge_in`, span kinds, need, `Diagnostic.rewrite`), task
  block `origin`, `merge_in_pending`, and the `merge_in` related link kind with
  `decodeIfPresent`/lossy decoding.
- Editor colors: `merge_in_arrow` merge purple, `merge_in_source` route family.
- Pickers for need `merge_in_source`: the vault and note-scoped task pickers with the
  `⤵ Merge into “T”` header, Bob's row detail, Add block ID splicing
  `parent_replacement`, and comma continuation.
- `CaptureMergeInPresentation` and the band in `standardPreviewItem`, the pending and
  refusal states (one button per `rewrite`), single display of T's block, `origin` folds
  for both arrows, and the copy table (footer, status, notification, VoiceOver).
- Raise `CaptureGrammarGate.minimumRevision` to the shipped revision with the new copy.
- Tests and fixtures: one, two, and five sources; long and duplicate titles; no notes; a
  ⚠ warning; zero, one, and many dependents; Blocked T; a pending second source; stale
  responses; mixed bulk drafts; `~~` merge-label band; light and dark; narrow width;
  keyboard disclosure; VoiceOver strings; gate blocking.
- Docs: update the Mac preview docs for the band without growing `docs/protocol.md`
  (already over budget, `bob-cli-h9`): put new prose in the page that owns the preview
  cards, then run `just docs-check` in both repos.

## Phase: memory

Use `/sase_memory_write` and edit only what the accepted memory decisions cover.

> [!decision] memory_task_merge_glossary Rewrite the Task Merge strand's gesture list to
> name `bob task merge`, capture's `>` and `<`, and the configured `merged into`
> relation.

> [!decision] memory_merge_in_decision Add `decisions:capture-merge-in-arrow`: `<` is
> the capture spelling of an explicit merge into the task the line creates; the item
> stays a task item with a merge band; it never runs the label path, never infers T's
> route, and never inherits source fields. Cite this plan, the research report, and the
> grammar and execute commits; name the rejected alternatives (`task_merge` kind, `~~`
> lowering, creatable `>` destination, keyword form).

Run `sase memory init` after editing.

## Cross-cutting rules

- Conventional Commits; only the grammar phase carries `!`. Never edit the Cargo.toml
  version or `CHANGELOG.md`.
- Gates per bob-cli phase: `just check` and `just docs-check`. Keep `schema_version` 1;
  every JSON addition is additive and omitted when empty.
- Rebase before landing; epic `bob-cli-hj` is changing nearby grammar, rewrite, and Mac
  files. If it takes revision 15 first, use the next free number everywhere.
- Never test against Bryan's vault; use fixture vaults.
