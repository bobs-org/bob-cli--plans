---
tier: epic
title: Choose parent tasks to move in with a `- ^^` capture bullet
goal: "In a project-note capture sealed with `@foo^bar@` or `@foo:bar@`, a first-level
  `- ^^` bullet lets Bryan sort the parent note after Return and before anything is
  written. A three-state Move card (Leave by default, `^` move, `:` move & link) or `bob
  capture -m` answers it. The picks land at the `^^` slot in parent order with exactly
  the vault result that authored `- ^id` / `- :id` bullets produce, in one guarded
  batch.

  "
decisions:
  memory_decision_record:
    ask:
      Add a decision record that `- ^^` is answered after Return, before any write, on
      `-m`, never `-p`?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
phases:
  - id: contract
    title: Contract docs, copy tables, vectors, and the decision record
    depends_on: []
    size: medium
    description: "contract: write the `- ^^` contract into docs/capture.md and the
      reference pages (grammar rows, copy and refusal tables, the `move_choice` JSON,
      `-m`, MQ vectors) and, if accepted, the decisions record; no Rust changes.

      "
  - id: cli-grammar
    title: Lexer arm, parsers, capture-parse spans, continuation key, grammar bump
    depends_on:
      - contract
    size: medium
    description: "cli-grammar: claim `- ^^` in the shared move lexer with its teaching
      errors, report it from capture-parse, advertise `^` as a continuation key on the
      lone `- ^` picker, bump the grammar, and refuse `^^` in bob capture until
      cli-report.

      "
  - id: cli-consequence
    title: One planner-true move consequence builder
    depends_on:
      - contract
    size: medium
    description: "cli-consequence: extract one consequence builder that the planner's
      `moved_tasks[].detail`, the future `move_choice` rows, and capture-complete all
      use, fed by the link guard, and fix the In Progress and Blocked copy.

      "
  - id: cli-report
    title: move_choice report, `-m/--move-in`, answer resolution, and refusals
    depends_on:
      - cli-grammar
      - cli-consequence
    size: medium
    description: "cli-report: build the `move_choice` report from the staged parent, add
      `-m/--move-in`, resolve answers to rows, emit every write-free refusal in order,
      and print the numbered terminal list; answered runs still refuse until
      cli-execute.

      "
  - id: cli-execute
    title: Mint, splice at the slot, offer exclusion, and output
    depends_on:
      - cli-report
    size: medium
    description: 'cli-execute: mint IDs for picked ID-less rows, splice picks into the
      move list at the `^^` slot in parent order, keep them out of successor offers,
      enforce the `#name` backstop, report `via: "choice"`, and pin MQ equivalence.

      '
  - id: mac-draft
    title: Typing `- ^^` in Bob Mac Capture
    depends_on:
      - cli-grammar
    size: small
    description: "mac-draft: honor Bob's `action_continuation_keys` in the Move picker
      so a second `^` leaves `- ^^`, show the footer hint, and colour the choice marker
      span.

      "
  - id: mac-card
    title: Move card state, view, and decoding
    depends_on:
      - cli-report
    size: medium
    description: "mac-card: decode `move_choice`, build the portable three-state card
      state machine and its tests, and draw the Move card (sigil wells, chips, counter,
      detail strip, primary label) without wiring it into submit.

      "
  - id: mac-flow
    title: Move card submit flow, chaining, refusals, preview hints, and docs
    depends_on:
      - cli-execute
      - mac-draft
      - mac-card
    size: medium
    description:
      "mac-flow: open the Move card on Return ahead of Pull-In, send `--move-in`, chain
      Pull-In and inbox routing, reopen on refusals with surviving marks, add the ghost
      hint, band line, move hint, old-bob gate, notification, and docs."
proposed_by: bbugyi200.apollo.research.37.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 08:50:21
status: wip
---

- **PROMPT:**
  [prompts/202611/move_choice.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/move_choice.md)

# Plan: Choose parent tasks to move in with a `- ^^` capture bullet

Research: `research:202611/parent_task_move_choice/parent_task_move_choice.md`. Bryan
accepted every requirement it recommends (A1–A12). This plan turns that report into a
contract and splits it into phases. §2.2 lists where this plan goes beyond the report.
It extends the shipped move-bullet work of epic `bob-cli-gl`
(`plan:202611/capture_move_bullets.md`); every rule of that contract still holds unless
this plan says otherwise.

## 1. The experience

You are carving a project for leaving Google out of `cash`, a 25-task area. You know one
task by name and want to see the rest:

```text
Finish the Google exit packet @cash^goog-exit@#admin
- Draft the resignation memo :draft-memo
- ^pay-cc
- ^^
- Book the lawyer
```

To get `- ^^`, type `- ^`; the Move picker opens as it does today. Type `^` again: the
picker closes and the line reads `- ^^`, drawn in the route accent. A dim ghost hint
follows it: `choose from cash.md's 25 open tasks on ↩`. The preview's moves band adds a
line, `↪ Asks about cash.md's 25 open tasks on ↩`. The preview shows the project as it
would be with no picks. That is honest, because Leave is the default.

Press Return. Nothing is written yet. The Move card opens at once, built from the
preview, with nothing marked:

```text
┌ ↪ Move into cash_goog_exit.md ──────────────────────── from cash.md · 25 open ┐
│ ⌕ Filter                               3 move in · 1 links today  TODAY 9/10 │
├ Tasks ────────────────────────────────────────────────────────────────────────┤
│    ^    ◐ Collect the equity paperwork        WORK 13:30      ^equity-docs   │
│    :    ○ Call Morgan Stanley about the 401k  → ADMIN         ^call-ms       │
│ ▸ ·[^]: ○ Renew the passport                                  ⊕ adds an ID   │
│    ·    ○ Ask Dana about the severance math                                  │
├ Admin ────────────────────────────────────────────────────────────────────────┤
│    ^    ○ Pay the credit card                 moving on line 3  ^pay-cc      │
│    ·    ○ File the expense report                             ^expenses      │
├───────────────────────────────────────────────────────────────────────────────┤
│ Renew the passport · Moves into cash_goog_exit.md · adds ^renew-the-passport │
│ Lands between “Pay the credit card” and “Book the lawyer”, in cash.md order  │
│ ^ move  : move & link  ␣ toggle  ⌘A all  ⎋ back  ⇧↩ none   ↩ Move 3 · link 1 │
└───────────────────────────────────────────────────────────────────────────────┘
```

**The card speaks the draft's own syntax.** Each row starts with a _state well_ holding
the sigil the row would have had if you had typed it, in the editor's own span colours:

- `·` (hairline): Leave;
- `^` (route accent): Move;
- `:` (explicit-toggle yellow): Move & link.

The highlighted row expands its well into a three-segment control, `· ^ :`.
`Pay the credit card` is already moving through the explicit `- ^pay-cc` on line 3, so
its well shows that `^` dimmed and the row is locked. You press `^` on the first row and
`:` on the second. Each sigil sets that row's state and moves the highlight down, so you
can sweep a long parent one key per row. The counter, the TODAY badge, and the primary
button keep up as you go. You press Return.

One `bob capture --move-in` call then writes everything atomically:

- the new `cash_goog_exit.md` holds `^draft-memo`, `^pay-cc`, the three picks in
  `cash.md` order, and `Book the lawyer`, in that order;
- `Renew the passport` gains a minted `^renew-the-passport` in the same batch;
- `^call-ms` becomes Next and links under ADMIN;
- links to the moved tasks are repaired vault-wide;
- every moved task is stamped fresh.

The notification reads `Created cash_goog_exit` with the body
`moved 4 tasks from cash · 1 linked under ADMIN`.

From a terminal, the same draft previews as numbered rows, and a real run must answer:

```text
$ bob capture -d "$(cat draft.txt)"
[dry-run] ok would capture  cash_goog_exit.md
  parent  [[cash]]
  - [ ] #task #prj Finish the Google exit packet #hide ^prj
  3 tasks · 1 moved from cash.md
  would move  [ ] Pay the credit card  cash.md:21 → ^pay-cc
  - ^^ on line 4 asks which of cash.md's 25 open tasks move in:
     1  [/] Collect the equity paperwork        ^equity-docs · WORK link follows
     2  [ ] Call Morgan Stanley about the 401k  ^call-ms
     3  [ ] Renew the passport                  adds ^renew-the-passport
     …
     9  [ ] Pay the credit card                 moving on line 3
     …
  choose with -m 1,3 (move), -m :2 (move & link), or -m none

$ bob capture "$(cat draft.txt)"
error: `- ^^` on line 4 asks which of cash.md's 25 open tasks move in; answer with -m 1,3 (move), -m :2 (move & link), or -m none

$ bob capture -m 1,:2,3 "$(cat draft.txt)"
ok captured  cash_goog_exit.md
  6 tasks · 4 moved from cash.md
  moved  [ ] Pay the credit card  cash.md:21 → ^pay-cc
  moved  [/] Collect the equity paperwork  cash.md:7 → ^equity-docs · WORK link follows
  moved  [ ] → [*] Call Morgan Stanley about the 401k  cash.md:12 → ^call-ms · becomes Next · links under ADMIN
  moved  [ ] Renew the passport  cash.md:15 → ^renew-the-passport · added ID
  repaired  3 links in 2 notes
```

The `contract` phase pins the exact strings. The shapes above are the target.

## 2. Why this shape

### 2.1 The design in four rules

1. **`- ^` is recall; `- ^^` is recognition.** Use `- ^` when you know the task and want
   to name it. Use `- ^^` when you want to see the parent and sort it. One sentence in
   the docs, the move hint, and the picker footer teaches both.
2. **Decide before anything is written.** The card runs after Return and before any
   vault write (A1). Cancelling is free, the batch stays atomic, and the flow matches
   Pull-In and inbox routing.
3. **Picks behave exactly as if typed** (A3). Bob expands the answers server-side into
   the same move list that authored `- ^id` / `- :id` bullets produce, at the `^^` slot,
   in parent document order. The draft text is never rewritten.
4. **Focus never selects** (A4). The highlighted row is never a pick. Return with
   nothing marked does nothing and says why; ⇧↩ explicitly means "none".

`decisions:mac-capture-is-a-thin-client` holds throughout. bob parses, plans, words
every consequence, and writes; the Mac only presents what bob reports and sends one
answer.

### 2.2 Design calls this plan makes beyond the research

1. **Every enabled row offers both `^` and `:`, and `:` tells the truth.** The research
   dropped `:` under a future `s:N`. This plan keeps `:` wherever an authored `- :<id>`
   would be accepted, which keeps the "as if typed" equivalence exact. Each choice
   carries planner-true `consequence` copy plus `adds_link`. Counters and the TODAY
   badge count only choices that add a Task Link. So a `:` on a task that is already
   linked, held Blocked, or under a dated project reads, for example,
   `Moves in · stays unlinked until the project starts Nov 6`. The counters never
   overstate.
2. **One shared consequence builder** (phase `cli-consequence`). The same function
   produces the planner's `moved_tasks[].detail`, the card's per-choice copy, and the
   `- ^` / `- :` picker copy. It is fed by the same link guard
   (`capture/link_guard.rs::assess_link_target`) the planner runs. This fixes V11: the
   picker today says "becomes Next" for In Progress rows and promises a link to Blocked
   rows.
3. **Locked rows show their explicit sigil**, dimmed, with `moving on line N`. This is
   truer than a lock glyph and adds no new glyph. A row is locked when an explicit
   `- ^id` / `- :id` anywhere in the draft moves it from the same parent, in this item
   or another.
4. **Marks are remembered per parent note, not per exact draft.** After ⎋ you can fix a
   typo and press Return again without losing your sort. Marks are dropped when the
   capture succeeds, the draft is discarded, or the slot's parent changes. A stale
   reopen keeps every mark whose key still resolves.
5. **The filter hides rows but never reorders them.** The card is a top-to-bottom sort
   of the parent, so filtered rows keep document order and section headers. Matches are
   highlighted rather than ranked.
6. **`move_choice` is batch-level**, beside `pull_in` and `inbox_route`, with `slots[]`.
   v1 allows exactly one slot per draft (A7), so a later multi-slot version stays
   additive.
7. **Split the CLI work in three** (`cli-consequence`, `cli-report`, `cli-execute`) and
   the Mac work in three (`mac-draft`, `mac-card`, `mac-flow`), instead of the
   research's one large CLI phase and one large Mac phase. Each phase has a crisp
   contract, and the Mac can start the card while bob finishes execution.

### 2.3 Rejected alternatives (from the research)

- Writing the project first and prompting afterwards: two commits, a preview that lied,
  a crash window, and an awkward cancel.
- Multi-select inside the live `- ^` picker that writes N lines: ID-less rows would need
  N vault writes before the capture.
- Expanding `^^` at type time: the same ID problem, plus grammar that exists only to be
  erased.
- Always asking whenever the parent has tasks: implicit and nagging.
- Reusing `-p/--pull-in` with sigil prefixes: one flag with two meanings.
- Two checkboxes per row, or a cycling tri-state checkbox: an invalid "link without
  move" state, or overshooting into the stronger action.
- A one-key "move and link all": inflates today's plan in one keystroke. There is
  deliberately no `:all`.

## 3. Behavior contract

### 3.1 Grammar

The scope is the bob-cli-gl move-bullet scope: a first-level `-`, `*`, or `+` bullet of
an item sealed with `@<route>^<id>@` or `@<route>:<id>@` (on its parent line or a
sealing child line). The shared lexer (`capture_language/move_bullet.rs`) gains one arm,
`MoveBulletHit::Choice { span }`. It claims the line from its raw shape, before trailing
IDs are stripped. `item.rs` and `editor_parse.rs` share it, so messages stay
byte-identical.

| First-level body (marker stripped, trimmed)                                             | Meaning                                           | Code                    |
| --------------------------------------------------------------------------------------- | ------------------------------------------------- | ----------------------- |
| `^^`                                                                                    | **Move choice**: asked on Return; picks land here | —                       |
| `^^ text`, `^^^`, `^^x`                                                                 | teaching error                                    | `move_choice_text`      |
| `[ ] ^^` (any checkbox)                                                                 | teaching error (reused)                           | `move_bullet_checkbox`  |
| `^^` with nested bullets under it                                                       | teaching error                                    | `move_choice_nested`    |
| a second `^^` anywhere in the draft                                                     | teaching error naming the first line              | `move_choice_duplicate` |
| `::`                                                                                    | reserved teaching error pointing at `^^`          | `move_choice_colon`     |
| `\^^`, `` `^^` ``, nested `  - ^^`, non-project items, `@file` lists, `@route+id` items | unchanged from today                              | —                       |

Messages. `<route>` comes from the seal. The `contract` phase may polish wording, but it
must keep each code and the information each message carries.

| Code                    | Message                                                                                   |
| ----------------------- | ----------------------------------------------------------------------------------------- |
| `move_choice_text`      | `` `- ^^` takes no other text; leave it alone on its line to choose tasks from cash.md `` |
| `move_choice_nested`    | ``notes under `- ^^` are not supported; add them after capture``                          |
| `move_choice_duplicate` | ``only one `- ^^` per capture; the first is on line 4``                                   |
| `move_choice_colon`     | `` `- ::` is not a picker; write `- ^^` and choose `^` or `:` for each task ``            |

Rules:

- A `- ^^` claims no ID and adds nothing to the item's ID namespace.
- For the "unused `#name`" rule, a `- ^^` counts as a _possible_ link, like an
  unfinished `- :`. The capture-time backstop is §3.7.
- Bump `GRAMMAR_REVISION` by one from the live constant (10 at planning time). The
  commit is `feat(capture)!:` with a BREAKING CHANGE footer: a `- ^^` bullet used to
  capture a literal task named `^^` (`decisions:bob-cli-releases-ride-a-daily-train`).

### 3.2 `capture-parse`

- A draft whose only open question is a `- ^^` stays **complete**: mode `project_note`
  (or `pomodoro_project_note` when an explicit `:` link exists), with **no `needs`**.
  The dry-run preview must keep running, because the card opens from it.
- A new span `project_task_move_choice_marker` covers both carets.
- A new additive array `sub_bullet_move_choices` runs parallel to `sub_bullet_moves`, at
  the top level and per item. It is present only when an entry is non-null. Each entry
  is `null` or `{range}`. The `^^` line's `sub_bullet_moves` entry stays `null`, so an
  older Mac never mistakes it for an unfinished `- ^`.
- The `^^` line's `sub_bullets` body is empty and its `sub_bullet_task_ids` entry is
  `null`.
- Human output shows `move choice ^^`.

### 3.3 `capture-complete`

- At a lone `- ^` with an empty query, the `project_task_move` picker descriptor adds
  `action_continuation_keys: ["^"]` (A5). The key is not offered for `- :`, for a
  non-empty query, or outside a sealed project-note item.
- At `- ^^` (cursor anywhere on it), there is no `project_task_move` context, so the
  picker does not reopen.
- The move-picker consequence copy comes from the shared builder (§3.5).

### 3.4 The `move_choice` report

Every capture JSON (success, and the refusals in §3.7) carries a top-level `move_choice`
object, the same way it carries `pull_in` and `inbox_route`. Clients use
`move_choice.supported` for capability detection. When the draft has a `- ^^`, the
object gains `slots`. The slot is built from the **staged** parent (earlier batch items
are visible), reading only the parent and today's daily note, plus the link-guard inputs
of §3.5:

```json
"move_choice": {
  "supported": true,
  "slots": [{
    "item": 1, "item_line": 4, "draft_range": [112, 114],
    "parent": "cash.md", "destination": "cash_goog_exit.md",
    "pomodoro_name": "ADMIN", "name_needs_link": false,
    "open_count": 25, "answered": false, "moved_count": 0, "linked_count": 0,
    "copy": {
      "title": "Move into cash_goog_exit.md",
      "subtitle": "from cash.md · 25 open",
      "slot": "Lands between “Pay the credit card” and “Book the lawyer”, in cash.md order",
      "leave": "Stays in cash.md",
      "ghost": "choose from cash.md's 25 open tasks on ↩",
      "band": "Asks about cash.md's 25 open tasks on ↩",
      "then": null
    },
    "rows": [{
      "key": "cash.md#^call-ms", "n": 2, "line": 12, "section": "Tasks",
      "block_id": "call-ms", "mint": null,
      "text": "Call Morgan Stanley about the 401k",
      "status_symbol": " ", "status_name": "Ready",
      "hidden": false, "scheduled": null, "pomodoro": null,
      "choices": [
        {"sigil": "^", "consequence": "Moves into cash_goog_exit.md", "adds_link": false},
        {"sigil": ":", "consequence": "Moves in · becomes Next · links under ADMIN", "adds_link": true}
      ],
      "locked": null, "disabled_reason": null, "answer": null
    }]
  }]
}
```

- **Rows** come from the shared `task_move::movable_roots` eligibility, in document
  order. They include ID-less rows and `#hide` rows. `^prj`, closed rows, nested rows,
  and the template placeholder are excluded. `section` is the nearest heading.
  `pomodoro` is today's link chip, in the shape the move picker rows already use.
- **`n`** is the 1-based document-order index over _all_ listed rows, so numbers stay
  put when a row is locked or disabled.
- **`key`** reuses the successor-offer KEYS rule (`docs/task-dependencies.md` §12.2,
  `note_tasks::stable_task_key`): `path#^id`, or `path@<sha8>[~k]` for ID-less rows.
- **`mint`** is Bob's collision-free ID suggestion for an ID-less row
  (`capture_block_ids::mint_block_id`). It avoids the parent's IDs, the new note's
  authored IDs, `prj`, and every other row's mint. It is `null` for identified rows.
- **`choices`** lists `^` and `:` for every enabled row (§2.2.1), each with
  `consequence` and `adds_link` from §3.5. It is empty for a disabled or locked row.
- **`locked`** is `{sigil, item_line}` when an explicit move anywhere in the draft
  claims the row from this parent.
- **`disabled_reason`** is Bob-authored. For example, under a future `s:N`, a non-`[?]`
  row reads `the project starts Nov 6, so only Blocked tasks can move in`.
- **`name_needs_link`** is true when the item has a `#name`, no other `:` link exists,
  and the form needs one (the Create form; the Offer form defers the check as it does
  today).
- **`copy`**: `title`, `subtitle`, `slot` (the neighbouring task roots, or "first" /
  "last"), `leave`, `ghost`, `band`, and `then`. Under a `:` seal, `then` is
  `"Then: choose today's links for N new tasks"` when offers remain; otherwise null.
- **After an answer**: `answered: true`, `moved_count`, `linked_count`, and each row's
  `answer` (`"^"`, `":"`, or `null`). Dry-run JSON equals real-run JSON apart from
  `dry_run`.

### 3.5 Consequence copy (planner-true)

One builder (new `capture/move_consequence.rs`, or wherever `cli-consequence` finds it
fits best) maps a movable root plus its context to the copy and `adds_link` for each
sigil. Its inputs are:

- status;
- today's link (from the ledger);
- the item's `#name`;
- the destination's block state (a future `s:N`, or an inherited parent block);
- the link-guard verdict for linking the row after the move.

The planner's `moved_tasks[].detail` is built by the same function from the _actual_
outcome. A conformance test asserts that the prediction equals the outcome for every row
state.

Each consequence is a head plus optional `·`-joined suffixes. The head is
`Moves into <destination>` for `^` and `Moves in` for `:`, exactly as the picker copy
pinned by MC4 reads today. In this table, `Moves into …` abbreviates
`Moves into cash_goog_exit.md`.

| Row state                                  | `^` consequence                            | `:` consequence                                                                 |
| ------------------------------------------ | ------------------------------------------ | ------------------------------------------------------------------------------- |
| Ready, unlinked                            | `Moves into …`                             | `Moves in · becomes Next · links under ADMIN`                                   |
| Next, unlinked                             | `Moves into …`                             | `Moves in · links under ADMIN`                                                  |
| In Progress, unlinked                      | `Moves into …`                             | `Moves in · links under ADMIN` (lanes are sticky)                               |
| linked today under WORK                    | `Moves into … · its WORK link follows`     | `Moves in · keeps its WORK link` (`adds_link: false`)                           |
| Blocked only by its own open prerequisites | `Moves into … · stays Blocked`             | `Moves in · drops 1 prerequisite · becomes Next · links under ADMIN`            |
| Blocked only by its own future date        | `Moves into … · stays Blocked until Nov 9` | `Moves in · pulls Nov 9 forward · becomes Next · links under ADMIN`             |
| Blocked otherwise (guard consumes)         | `Moves into … · stays Blocked`             | `Moves in · stays unlinked: waits on ^lawyer-call` (`adds_link: false`)         |
| into a dated (`s:N`) project, `[?]`        | `Moves into … · stays Blocked`             | `Moves in · stays unlinked until the project starts Nov 6` (`adds_link: false`) |
| ID-less (suffix on either)                 | `… · adds ^renew-the-passport`             | `… · adds ^renew-the-passport`                                                  |

The suffixes after the head are exactly the moved task's `detail`. An empty suffix list
equals a `null` `detail`. Without `#name`, `links under ADMIN` reads
`links into today's plan`, as today. The capture-complete picker keeps its no-vault-scan
rule. It uses the builder in status-only mode: In Progress copy is corrected there, and
for a `[?]` row its `:` copy says `Moves in · links only if Bob can unblock it` instead
of promising a link. Its ID-less prefix, `Needs an ID first · ⊕ ^…`, is unchanged,
because that picker asks for the ID first.

### 3.6 `-m, --move-in <PICKS>`

This is a new repeatable `bob capture` option, sorted between `--inbox-route` and
`--no-clip`. Its help reads:
`Answer a - ^^ move choice: none, all, or a comma list of row numbers and keys; a leading : also links; repeatable`.

| Pick                       | Meaning                                                                |
| -------------------------- | ---------------------------------------------------------------------- |
| `3` / `cash.md#^call-ms`   | Move that row                                                          |
| `:3` / `:cash.md@1a2b3c4d` | Move it, then link it                                                  |
| `^3`                       | Synonym for `3` (documented with the zsh `EXTENDED_GLOB` quoting note) |
| `all`                      | Every enabled, unlocked row, as a plain move                           |
| `none`                     | Move nothing; the `- ^^` contributes nothing                           |

Bare numbers mean _move_. There is no `:all`. Values from repeated flags are unioned. A
token that is not `none`, `all`, a number, or a key (keys contain `#^` or `@`), with an
optional leading `:` / `^`, is a usage error. `none` or `all` mixed with other picks is
`move_choice_conflict`. Numbers resolve against the rows rebuilt at execution time. Keys
are stable and are what the Mac sends.

### 3.7 Refusals and their order

Every refusal writes nothing, in a dry run or a real run. JSON refusals carry
`move_choice` so a client can (re)open the card from them. Exit codes mirror the pull-in
and inbox-route equivalents. These checks run after the existing earlier refusals
(Pomodoro override, lexical errors, move-bullet refusals), in this order, and **before**
`pull_in_choice_required` and `inbox_route_choice_required`:

| Code                           | When                                                                                                                       |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `move_choice_unasked`          | `-m` given, but the draft has no `- ^^`                                                                                    |
| `move_choice_empty`            | No selectable rows (empty parent, or every row locked/disabled); a dry run reports it as a warning with `ok: true` instead |
| `move_choice_required`         | A real run with a `- ^^` and no `-m` (always; there is no opt-in flag)                                                     |
| `move_choice_conflict`         | `3,:3`, or `none`/`all` mixed with other picks                                                                             |
| `move_choice_stale`            | A key or number no longer resolves; carries fresh rows                                                                     |
| `move_choice_refused`          | A pick of a locked or disabled row                                                                                         |
| `unused_project_note_pomodoro` | A `#name` that the answers leave unused (`name_needs_link` and no `:` pick); existing code and message                     |

A dry run with no `-m` is `ok`. It plans the `- ^^` as `none` and reports the rows.

Interim: from `cli-grammar` until `cli-report`, `bob capture` refuses any `- ^^` with
`move_choice_unavailable`. From `cli-report` until `cli-execute`, it refuses any
_answered_ `- ^^` the same way:
`choosing parent tasks with - ^^ is not available in this build yet`. Each interim
refusal has a test that the next phase deletes. All interim states are write-free.

### 3.8 Execution (`plan_project_note_item`)

1. **Resolve** the answers against rows rebuilt from the staged parent (§3.4, §3.7).
2. **Mint** IDs for picked ID-less rows into the staged parent. Use `mint` when it is
   still free; otherwise take the next free suffix, avoiding the parent's IDs, the new
   note's authored IDs, `prj`, descendants of every moved block, and other mints. A
   collision with an imported descendant still refuses `move_task_id_collision`, as
   today.
3. **Splice** the picks into the item's sub-bullets at the `^^` position as synthetic
   move bullets `{sigil, block_id, range: <^^ range>, via: Choice}`, in parent document
   order whatever order they were picked in. With no picks, the `^^` line contributes
   nothing (the template placeholder appears only if nothing else does). From here the
   bob-cli-gl pipeline runs unchanged:
   - render slots, where blocks before the same line keep selector order;
   - resolve against the staged parent;
   - the Blocked-at-birth guard;
   - one `capture::batch_move` call with collisions refused;
   - freshness stamps;
   - the `:` link step (link guard, dependency drop, Ready → Next, Task Link placement);
   - destination-before-source writes and rollback.
4. **Offers** (A6). Under a `:` seal, rows that came from the Move choice are never
   offered as successors, whatever their sigil. A `^^` answer is final. Authored `- ^a`
   rows are still offered (MV5 unchanged), as are the note's new tasks.
5. **`#name` backstop**: `unused_project_note_pomodoro` when the answers leave a needed
   `#name` unused.
6. **Report**: `moved_tasks[]` entries gain `via: "choice"` (`"bullet"` otherwise,
   additive), and their `draft_range` is the `^^` range. A minted entry's `detail` ends
   `added ID`. `move_choice` flips to answered (§3.4).

The crash property holds: an interruption can duplicate a task but never lose one.

### 3.9 Output

- JSON: §3.4, §3.8, plus `move_choice` on every success. `schema_version` is unchanged,
  and every field is additive.
- Human (dry run, unanswered): one
  `- ^^ on line N asks which of P's K open tasks move in:` header, then one numbered row
  per listed row. A row shows its status, title, and `^id` or `adds ^mint`, plus
  `· <^ consequence suffix>`, or the lock/disabled reason. A final `choose with …` line
  follows. More than 20 rows are printed in full; the list is the only way to answer
  from a terminal.
- Human (answered): the existing `would move` / `moved` lines, in draft order.
- `move_hint` copy (MV17) gains `^^`. CLI:
  `hint: '- ^' moves one of cash.md's 12 open tasks into this project; '- ^^' lets you choose several`.
- `bob capture` long help gains one `- ^^` paragraph and one example. Run
  `just docs-gen` and update `tests/fixtures/help/*`.

### 3.10 Bob Mac Capture

**Typing (`mac-draft`).**

- The Move picker honors Bob's `action_continuation_keys`. Today
  `ProjectTaskMovePickerContext` and `CapturePickerDescriptor.projectTaskMoveContext`
  drop the keys, and `pickerOperatorContinuationKeys` / `continuePickerOperator` gate
  them to the `+` and `!` pickers.
- With an empty filter, `^` closes the picker, inserts itself, and suppresses reopening.
- When `^` is listed, the footer adds `^ again · choose several on ↩`.
- `project_task_move_choice_marker` maps to `.route` in
  `captureSemanticCategory(forSpanKind:)`.

**The Move card (`mac-card`).** A new portable `CaptureMoveChoiceState` in CaptureCore
holds:

- the rows in document order;
- `answers: [key: sigil]`;
- the highlight and the filter;
- `name_needs_link`;
- a `reseed(rows:)` that keeps surviving keys.

A new `CaptureMoveChoiceView` draws the §1 card:

- the state well, expanding to a `· ^ :` segmented control on the highlighted row;
- the status glyph, and the title with match highlights;
- chips: Pomodoro, schedule, `→ ADMIN` while `:` is marked, and `^id` in block-ID indigo
  or `⊕ adds an ID` while an ID-less row is marked;
- disabled rows dimmed with their reason; `#hide` rows muted; locked rows showing their
  sigil dimmed;
- the header, from Bob `copy`;
- a live counter (`3 move in · 1 links today`, with `· 2 marked outside this filter`
  when relevant);
- the plan-budget badge the Pull-In card shows when the preview reports `plan_budget`,
  projected by the `adds_link` marks;
- a two-line detail strip: the highlighted row's current consequence (`copy.leave` for
  Leave), then `copy.slot`, plus `copy.then` when present;
- the key-hint footer.

The primary button spells out the action (`Move 3 · link 1`). With no marks it reads
`Mark tasks with ^ or :` and is inert. When `name_needs_link` holds and nothing is
marked `:`, both ↵ and ⇧↩ are inert with `#admin needs a : pick, or remove #admin`. The
card adds no new glyphs or hex colours (task-dependencies §12.6). VoiceOver exposes each
well as an adjustable element with the values "Stays in cash", "Moves in", and "Moves in
and links".

| Key                | Action                                                                                                                                                         |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `^` / `:`          | **Set** the highlighted row to that state (idempotent), then move to the next enabled row. On a locked or disabled row, announce Bob's reason and do not move. |
| Space              | Toggle in place: Leave ↔ `^`, and `:` → Leave. It never escalates to `:`. It types a space while the filter has text, as on Pull-In.                          |
| ↑ ↓ ⌃P ⌃N          | Move the highlight                                                                                                                                             |
| ⌘A                 | Mark every visible, enabled Leave row `^`; a second ⌘A clears the marks that ⌘A set. Respects the filter.                                                      |
| other printable, ⌫ | Edit the filter (`^` and `:` are never filter characters)                                                                                                      |
| ↵                  | Apply the marks; inert with a hint at zero marks; auto-repeat ignored                                                                                          |
| ⇧↩                | Create without moves (`--move-in none`)                                                                                                                        |
| ⌃↩                | Apply, capture, and open Bob's reported notes                                                                                                                  |
| ⎋                  | Back to the unchanged draft; marks are kept (§2.2.4)                                                                                                           |
| mouse              | Click a row to highlight it; click a well segment to set that state                                                                                            |

**Flow (`mac-flow`).**

- `submit()` checks `move_choice` after the relation gate and **before** the Pull-In
  branch. If the current preview has an unanswered slot, the card opens from it with no
  bob call.
- Confirming carries the answers forward. Under a `:` seal with offers, the existing
  Pull-In card opens next from the same preview. Otherwise inbox routing runs as today.
  Otherwise the app dispatches.
- Every capture call after the card carries `--move-in` (one comma list of keys, `:`
  prefixed, or `none`), placed before `--pull-in`. `CaptureInboxRouteSubmitContext`
  carries it too.
- ⎋ on a Pull-In card that followed the Move card returns to the Move card with marks
  intact.
- A `move_choice_required` refusal (no matching preview) or a `move_choice_stale`
  refusal reopens the card from the refusal's rows, keeping marks whose keys survive.
  The strip then reads `cash.md changed · 1 marked task is gone`.
- `move_choice_refused`, `move_choice_conflict`, and `move_choice_empty` show Bob's
  message in place.
- Preview: the ghost hint after `- ^^` uses `copy.ghost` with `draft_range`, reusing
  gl.7's ghost overlay rules (shown only while the preview's draft equals the current
  draft and the bytes at the range are `^^`). Ghost titles skip `via: "choice"` entries.
- The moves band shows `copy.band`. The move hint becomes
  `Type - ^ to move a task from cash.md, or - ^^ to choose several (12 open)` when
  `move_choice.supported`.
- Notification: the existing title. The body becomes `moved N tasks from cash`, plus
  `· K linked under ADMIN` when links were made.

**Compatibility.**

| Mac × bob         | Behavior                                                                                                                                                                                                                                                                                 |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| new × new         | Full flow                                                                                                                                                                                                                                                                                |
| old Mac × new bob | The app sends no `--move-in`, so bob refuses `move_choice_required` with its message. Nothing is written.                                                                                                                                                                                |
| new Mac × old bob | No `move_choice.supported`. Reuse the `~` relation-gate pattern: the first ⌘↩ on a draft with a whole-line `- ^^` warns `This bob predates - ^^; it would capture a task named ^^ · update bob`, and a second ⌘↩ submits. `--move-in` is only sent after a preview advertised support. |
| old × old         | Unchanged (a literal `^^` task)                                                                                                                                                                                                                                                          |

Do not raise `CaptureGrammarGate.minimumRevision`.

### 3.11 Performance

- A draft without `- ^^` keeps today's cost exactly.
- An unanswered `- ^^` dry run reads the staged parent, today's daily note, and the
  link-guard graph. The graph is built at most once per report, and only when some row
  has dependency inputs.
- `cli-report` records the warm dry-run time on a vault copy. Measure the §1 draft
  against a 76-root parent (the size of `sase.md`), and also the same draft with `- ^^`
  removed. The difference goes in the phase notes. If it exceeds 150 ms, report why on
  the phase bead.
- `capture-complete` never scans the vault.

### 3.12 Non-goals and follow-ups

These are out of scope:

- several `- ^^` per draft;
- `- ^^query` or suffix forms;
- a merged Move + Pull-In card;
- an "Undo move-in" notification action;
- persisting marks across panel sessions or stashes (the docs say so plainly);
- `- ^^` under `@route+id`, note lists, or non-project items.

## 4. Vectors

The `contract` phase writes these into `docs/capture.md` as "Move-choice conformance
vectors". The implementing phases cite them in tests (`tests/cli/` style, temp vaults,
`BOB_NOW`).

| ID   | Draft / state                                                                                                                                         | Expected                                                                                                                                                     |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| MQ1  | `@cash^x@` / `@cash:x@` + `- ^^` (and `*`, `+` markers)                                                                                               | complete `project_note`, no `needs`; `project_task_move_choice_marker` over `^^`; `sub_bullet_move_choices` `{range}`; `sub_bullet_moves` entry null         |
| MQ2  | `- ^^ tax`, `- ^^^`, `- ^^x`, `- [ ] ^^`, `- ^^` + nested, two `- ^^`, `- ::`                                                                         | matching §3.1 code; capture-parse and `bob capture` messages byte-identical                                                                                  |
| MQ3  | `- \^^`, `` - `^^` ``, `  - ^^`, `- ^^` in a non-project item, `@file` list, or `@route+id` item                                                      | unchanged from today                                                                                                                                         |
| MQ4  | capture-complete at lone `- ^`; at `- ^d`; at `- :`; at `- ^^`                                                                                        | `action_continuation_keys: ["^"]` only on the first; no `project_task_move` context at `- ^^`                                                                |
| MQ5  | unanswered dry run                                                                                                                                    | `ok`; planned as none; rows in document order with `n`, keys, mints, `choices`; human numbered list                                                          |
| MQ6  | unanswered real run (also with `--require-pull-in-choice` under a `:` seal, and with inbox candidates)                                                | `move_choice_required` first, carrying `move_choice`; vault byte-identical                                                                                   |
| MQ7  | `-m 1,:2` vs authoring `- ^a` / `- :b` at the slot (`^` seal)                                                                                         | every file byte-identical; `moved_tasks` equal apart from `via` and `draft_range`; `-m 3,1` = `-m 1,3`; numbers = keys                                       |
| MQ8  | `- New` / `- ^^` / `- Other`, picks rows 4 then 1                                                                                                     | `## Tasks`: New, row 1, row 4, Other                                                                                                                         |
| MQ9  | pick an ID-less row; its `mint` collides with an authored ID / another mint; descendant collision                                                     | mints `mint` or the next free suffix in parent and new note; `detail` ends `added ID`; dry run = real; descendant collision refuses `move_task_id_collision` |
| MQ10 | `@cash:x@` + `- ^^` + `- New`: `-m 1`; `-m 1 -p all`; `-m :1 -p all`                                                                                  | row 1 never offered; `New` offered; the second leaves row 1 unlinked; the third links both                                                                   |
| MQ11 | `- ^a` + `- ^^`; item 2 also moves `^b` from `cash`                                                                                                   | rows a and b `locked`; `-m <a>` refuses `move_choice_refused`; `all` skips locked rows                                                                       |
| MQ12 | `s:3` + `- ^^`                                                                                                                                        | Ready rows disabled with reason; `[?]` rows offer `^` and `:` (`:` `adds_link: false`); `-m <ready>` refused                                                 |
| MQ13 | row edited, closed, or moved between dry run and run; out-of-range number                                                                             | `move_choice_stale` with fresh rows; nothing written                                                                                                         |
| MQ14 | `-m 3,:3`; `-m none,1`; `-m 1` without `- ^^`; `-m foo`                                                                                               | `move_choice_conflict` ×2; `move_choice_unasked`; usage error                                                                                                |
| MQ15 | parent has no movable roots; every root locked                                                                                                        | dry run `ok` with a `move_choice_empty` warning; real run refuses `move_choice_empty`, even with `-m none`                                                   |
| MQ16 | `@cash^x@#admin` + `- ^^`, no other link: parse; `-m 1`; `-m :1`                                                                                      | parse ok and `name_needs_link: true`; `unused_project_note_pomodoro`; ok                                                                                     |
| MQ17 | each §3.5 row state, through `^^` and through an authored bullet                                                                                      | each predicted `consequence` and `adds_link` equals the outcome (`detail`, `link`) of the authored run                                                       |
| MQ18 | write order, injected failure, `NoteChanged` during planning                                                                                          | destination before source; full rollback; nothing written                                                                                                    |
| MQ19 | project item with no moves and no `- ^^`; parent has 12 open roots                                                                                    | `move_hint` unchanged in JSON; human hint mentions `- ^^`                                                                                                    |
| MM1  | Mac typing: `- ^` then `^`; `- :` then `:`; `- ^d` then `^`                                                                                           | `- ^^`, picker closed, not reopened; the others filter or type as today                                                                                      |
| MM2  | Mac card keys: sigil sweep, skip on locked/disabled, Space, ⌘A on a filter, ↵ at zero, ⇧↩, `name_needs_link` gating, ⎋ keeps marks                   | behaves as §3.10                                                                                                                                             |
| MM3  | Mac flow: Pull-In follow-on and ⎋ back, inbox-route chain, stale reopen keeps surviving marks, required-refusal reopen, old-bob warning, notification | behaves as §3.10                                                                                                                                             |
| MM4  | Mac design: light and dark, long titles, 76 rows, keyboard-only, VoiceOver                                                                            | legible, nothing clipped, every state reachable and announced                                                                                                |

## 5. Phases

Every bob-cli phase runs `just check`, plus `just docs-check` when it touches `docs/`,
help text, or `mkdocs.yml`. A phase that changes clap help runs `just docs-gen` and
updates `tests/fixtures/help/*`. Never edit `Cargo.toml`'s version or `CHANGELOG.md`
(`decisions:bob-cli-releases-ride-a-daily-train`). A phase that adds or changes CLI
options reads `cli_rules.md`. A phase that edits docs reads `docs.md`. Use
`/sase_memory_read` for both.

Mac phases open `bob-mac-capture` with `/sase_repo` and follow its `AGENTS.md`:

- the portable check on a host with a Swift toolchain;
- app-target changes typechecked on a scratch copy on the Mac host;
- CI runs the app tests;
- 100-column lines, and format only changed lines.

Never land uncompiled Swift. If the Mac is offline, the phase note must say that the
macOS targets are unverified. Real-bob fixtures need `BOB_BIN` pointing at a bob built
from current bob-cli master.

### 5.1 `contract`: docs, copy, vectors, and the decision record

- **`docs/capture.md`:**
  - Add a grammar-at-a-glance row for `- ^^`.
  - Under "Moving parent tasks in", add a new subsection
    `##### Choosing several parent tasks (- ^^) {#choosing-several-parent-tasks}`. It
    covers the §1 example and the §2.1 rules in prose, and the §3.1, §3.5, and §3.7
    tables.
  - Document the `-m` reference (§3.6), the `capture-parse` and `capture-complete`
    additions (§3.2–3.3), the `move_choice` JSON (§3.4), execution (§3.8), and output
    (§3.9).
  - Add the §4 vectors (MQ and MM).
  - Add `- ^^` to the `#pomodoro` "unused name" rule.
  - In "Offer a new project's tasks", state A6 (Move-choice rows are never re-offered;
    authored `- ^a` rows still are).
- **`docs/reference/json-outputs.md`:** `move_choice`, and `moved_tasks[].via`.
- **`docs/reference/compatibility.md`:** the grammar bump, `-m`,
  `move_choice.supported`, and the §3.10 matrix.
- **`docs/task-move.md` and `docs/freshness.md` §5 "Who stamps":** one line each: `^^`
  picks move and stamp exactly like move bullets, and ID-less picks are minted in the
  batch.
- **Gate:** `just docs-check` passes. No Rust behavior changes in this phase.

> [!decision] memory_decision_record

- **Memory.** Implement `memory_decision_record` only if accepted, through
  `/sase_memory_write`. Write one new `decisions` strand, for example
  `move-choice-asks-before-write`, with:
  - **Applies to:** bob-cli and Bob Mac Capture.
  - **Claim:** `- ^^` asks after Return and before any write; a real run never moves
    unanswered; focus never selects (↵ is inert at zero marks, ⇧↩ means none); answers
    ride `-m`, never `-p`; rows answered there are never re-offered; picks land at the
    slot in parent order with in-batch IDs.
  - **Why** (the §2.3 rejected alternatives), **cost**, and **reopens when**.
  - **Evidence:** the research ref and this plan.

  Then run `sase memory init`.

### 5.2 `cli-grammar`: lexer, parsers, capture-parse, continuation key

- **Lexer.** Add `MoveBulletHit::Choice { span }` and §3.1's invalid shapes to
  `move_bullet.rs`, with shared error constructors and `code_for_message` entries. Today
  `- ^^` returns `None` at the "first word is not alphanumeric" check. Claim it there,
  before that check, and keep `- ^` / `- ^id` behavior byte-identical.
- **Draft-wide duplicate.** Both parsers detect a second `- ^^` across items and name
  the first line.
- **Model.** Add `AuthoredSubBullet.move_choice: Option<MoveChoiceBullet { range }>`.
  `MoveBullet` gains `via: MoveVia::{Bullet, Choice}` (always `Bullet` here).
- **Parsers.**
  - `item.rs`: claim the line before `strip_trailing_task_id`.
  - `editor_parse.rs`: diagnostics, the span, `sub_bullet_move_choices`, the `#name`
    possible-link rule, and no need.
  - Add the span kind to `editor_model.rs`.
- **capture-parse** human output, plus the `capture-parse` help paragraph.
- **capture-complete.** Add `action_continuation_keys: ["^"]` (§3.3) in
  `capture_complete/engine.rs` `picker_descriptor`. Make sure `- ^^` yields no
  `project_task_move` context.
- **Grammar bump** (§3.1). Count it from the live constant at commit time.
- **Interim refusal** `move_choice_unavailable` (§3.7) in `plan_project_note_item`, with
  a test that `cli-report` deletes.
- **Tests:** MQ1–MQ4, plus parser-agreement tests (execution message equals diagnostic
  message).
- Run `just docs-gen` if help changed.

### 5.3 `cli-consequence`: one planner-true consequence builder

- Extract the copy now split between
  `capture_complete/candidates.rs::project_move_consequence` and the inline `detail`
  strings in `capture/project_note.rs` (the `:` link step and
  `execute_project_note_moves`). Put it in one builder with two entry points:
  - `predict(root, sigil, ctx)` returns `{consequence, adds_link}`;
  - `describe(outcome)` returns the `detail` suffix from an actual moved-task outcome.
- **Guard input.** Feed `predict` the link-guard verdict for linking that root _after_
  the move: own reasons from the staged graph, plus the destination's birth or inherited
  block. Build the graph at most once per call site.
- **capture-complete** uses status-only mode (§3.5). Fix V11: In Progress gets no
  "becomes Next", and Blocked rows get no link promise.
- **No change** to any `moved_tasks[].detail` string that MV1–MV17 pin. If honesty
  requires a new suffix (for example the dependency-drop or pull-forward wording),
  update the docs table and state it in the commit body.
- **Tests:** MQ17 driven through _authored_ `- ^a` / `- :a` drafts (no `^^` is needed
  yet). For each §3.5 row state, the suffixes of `predict` (after its head) equal
  `describe(outcome)`, which is the run's `detail`. An empty suffix list equals a `null`
  `detail`. `adds_link` equals "the run added a new `task_links[]` entry for that task".
  Update the MC4 picker copy tests.

### 5.4 `cli-report`: `move_choice`, `-m`, resolution, refusals

- **`-m/--move-in`** in `capture/cli.rs`: parsing into `MoveInPicks`, sorted placement,
  help, `just docs-gen`, and the help fixtures.
- **Report builder.** `move_choice` with `supported: true` on every capture result and
  on the §3.7 refusal payloads, plus `slots` per §3.4:
  - rows via `movable_roots` on the staged parent;
  - keys via `stable_task_key`;
  - mints via `mint_block_id`;
  - pomodoro chips from today's ledger;
  - draft-wide locks, computed from all parsed items before planning;
  - disabled reasons;
  - `choices` via `cli-consequence`;
  - `copy`, `name_needs_link`, and `open_count`.
- **Resolution.** Turn answers into resolved picks with the §3.7 checks in order, before
  `plan_with_pull_in`'s choice-required refusal and the inbox-route refusal. Then hand
  an answered `- ^^` to the interim refusal (§3.7) until `cli-execute` replaces it.
  Delete `cli-grammar`'s interim test.
- **Human output.** The §3.9 unanswered list, the `choose with` line, the refusal
  messages, and the `move_hint` copy.
- **Tests:** MQ5, MQ6, MQ11–MQ16 (refusal halves), MQ19, plus JSON golden updates for
  the new top-level field.
- **Timing.** Record §3.11 in the phase notes.

### 5.5 `cli-execute`: mint, splice, offers, output

- §3.8 steps 2–6 in `capture/project_note.rs` and `capture_project_note.rs`:
  - in-batch minting into the staged parent;
  - synthetic `via: Choice` move bullets at the `^^` index, in parent order;
  - `ResolvedProjectMove` and `ProjectNoteMovedTask` carry `via`;
  - offer exclusion keyed on `via: Choice`;
  - the `#name` backstop;
  - `move_choice` answered fields.
- Remove the interim refusal and its test.
- **Human output** for answered runs (`added ID` detail).
- **`bob capture` long help:** add the `- ^^` paragraph and example, then run
  `just docs-gen` and update the fixtures.
- **Tests:** MQ7–MQ10, MQ12 (execution half), MQ16 (`-m :1` succeeds), MQ17 through
  `^^`, and MQ18.
- Reconcile `docs/capture.md` wherever implementation forced a deviation, and say so in
  the commit body.

### 5.6 `mac-draft`: typing `- ^^`

- §3.10 "Typing":
  - thread `action_continuation_keys` through `ProjectTaskMovePickerContext`,
    `CapturePickerDescriptor.projectTaskMoveContext`, `presentProjectTaskMovePicker`,
    and the previews installer;
  - admit `.projectTaskMove` in `pickerOperatorContinuationKeys` and
    `continuePickerOperator`;
  - add the footer hint;
  - add the span mapping.
- Regenerate the move-complete fixtures (`Scripts/generate-move-fixtures.sh`) from a bob
  with `cli-grammar`.
- **Tests:** MM1, and the existing `CaptureProjectTaskMovePanelTests` stay green.
- **Docs:** the "Move Picker" section of `docs/task-pickers.md`.

### 5.7 `mac-card`: Move card state, view, and decoding

- **Decoding** (CaptureCore, all `decodeIfPresent`): `CaptureMoveChoice`, its slot, row,
  choice, lock, and copy types; `moved_tasks[].via`.
- **State.** `CaptureMoveChoiceState` in CaptureCore (portable and unit-tested):
  - set-and-advance;
  - skip with reason;
  - Space toggle;
  - ⌘A set and clear, respecting the filter;
  - filter without reordering, and the outside-filter count;
  - counters from `adds_link`;
  - primary-label and inertness rules, including `name_needs_link`;
  - the answer serialization (keys in parent order, `:`-prefixed, or `none`);
  - `reseed(rows:)` keeping surviving keys and reporting the lost count.
- **View.** `CaptureMoveChoiceView` (BobMacCapture), the §1 card, reusing the Pull-In
  card's section header, height policy, action button style, and TODAY badge components
  rather than copying them. Add a previews installer like
  `installPullInChoiceForPreviews`.
- **Routing.** Add the card's key commands to `CaptureKeyCommandRouter` behind a new
  focus target, unreachable until `mac-flow` mounts it.
- **Fixtures:** real-bob `move-choice-*.json` dry-run fixtures for MQ5, MQ11, MQ12, and
  MQ16, from a new `Scripts/generate-move-choice-fixtures.sh`.
- **Tests:** MM2 (state), plus design tests for MM4.

### 5.8 `mac-flow`: submit flow, chaining, refusals, preview, docs

- §3.10 "Flow":
  - the `submit()` order;
  - `BobProcessClient.capture` gains `moveInSelection` and emits `--move-in` before
    `--pull-in`;
  - Pull-In and inbox-route contexts carry the answers;
  - ⎋ back from Pull-In;
  - `completeSubmit` branches for the move-choice refusals;
  - mark memory keyed by parent;
  - the old-bob gate.
- **Preview:** the ghost hint, the band line, the move hint, and the notification body.
- **`fake-bob`:** handlers for the required, stale, and refused refusals and for
  answered success (env toggles in the style of `FAKE_BOB_STALE_PULL_IN`).
- **Fixtures:** real-bob answered-success fixtures for MQ7 and MQ10.
- **Tests:** MM3 and MM4 in BobMacCaptureTests.
- **Docs:**
  - `docs/keyboard.md`: a new "Move choices" section beside "Successor choices";
  - `docs/capture-previews.md`: "Project-note moves" gains the `- ^^` ghost, band, and
    card;
  - `docs/task-pickers.md`: the Move card, linked from Move Picker;
  - `docs/protocol.md`: `--move-in`, `move_choice`, and the compatibility rows.

  Then run `just docs-check`.

- **Before closing**, record in the phase note the three walkthroughs the research asks
  for on the real card:
  - move three tasks without linking;
  - mix two moves and one link;
  - cancel after marking an ID-less task.

  Bryan must be able to predict every task's resulting note and today's links from the
  card alone. If no one can drive the Mac UI, say so and leave the walkthrough to Bryan.

## 6. Sequencing with in-flight work

Epic `bob-cli-go` (transfer `>`, phases go.10–go.13) is still in progress. It rewrites
the same lexer, `editor_parse.rs`, `capture_complete`, `batch_move`, the grammar
constant, and (go.12) the Mac editor spans. Before starting, every phase reads
`bob-cli-go` with `sase bead read` and rebases onto whatever has landed. Transfer-line
precedence and target-run marker precedence stay as shipped: a `- ^^` is claimed only in
a sealed project-note item's first-level bullets. Compute the grammar bump from the live
constant at commit time. If go.11 or go.12 is still open when `cli-grammar` or
`mac-draft` starts, keep the change surgical, and record any overlap on the phase bead
rather than editing go's code paths.
