---
tier: epic
title: Capture `&` and `*` relate existing `@route+id` tasks, first-class
goal: 'In `bob capture` and Bob Mac Capture, an existing task named with
  `@route+block-id` (or another existing-task spelling) can own `&` and `*`. A draft can
  be built from either end. Each one either writes exactly the stated relationship or
  refuses with a single accurate message, and the parse and execute steps agree on it.
  Both the terminal and the Mac preview say "<task> now depends on / now blocks <task>",
  with the status changes and no raw wire words or duplicated lines.

  '
phases:
  - id: contract
    title: Docs contract, wording, and RX/RXC vectors
    depends_on: []
    size: medium
    description:
      "contract: write the docs/capture.md section for relating existing tasks. It
      covers the owner rule, every spelling, the three authoring flows, the refusals and
      their exact messages, the CLI layout, the additive capture-complete wire fields,
      and the RX and RXC vectors. Also update grammar-table and README rows. No code."
  - id: grammar
    title:
      Owner grammar, `^route:id` alias, `+` selector with modifiers, agreeing refusals
    depends_on:
      - contract
    size: medium
    description:
      "grammar: in capture_language, accept bare `^route:id` as an owner alias in the
      leading and trailing positions. Classify `+`/`+query` beside modifiers as the
      incomplete parent selector, not a junk task. Drop the bogus Pomodoro-ledger-link
      diagnostic. Give parse and execute one shared relationship-only refusal for
      `p:`/`s:`/`%` and child lines. Update capture-parse help."
  - id: execute
    title: Reliable existing-owner execution and the relationship CLI layout
    depends_on:
      - grammar
    size: medium
    description:
      "execute: route inherited `@@route+id` relationship-only items through the
      dependency-only path, so no empty bullet is written. Refuse a Done or Canceled `*`
      owner instead of reopening it. Fix the cycle path text. Replace the existing-owner
      and `⛓` human output with the owner status row and one row per relationship. Fix
      sub-bullet row order. Add RX CLI tests."
  - id: complete
    title: Owner-aware completion wire and guarded owner pickers
    depends_on:
      - grammar
    size: medium
    description:
      "complete: add `text`, `note_path`, and status fields to capture-complete's
      `owner`. Add a `picker.relationship` descriptor to `+` and `@route+` pickers
      inside relationship drafts, with their `disabled_reason` rows (self, cycle,
      internal `^prj`). Shell completion skips disabled rows. Add RXC tests and update
      capture-complete help."
  - id: mac_pickers
    title: Bob Mac Capture owner-aware pickers and the calm owner hint
    depends_on:
      - complete
    size: medium
    description:
      'mac_pickers: give the `&`/`*` picker headers the owner''s name and direction, fix
      the nil new-task header, and add the "This note" group. Add an owner mode to the
      `+`/`@route+` picker with copy and disabled rows. Add a calm `dependency_target`
      hint in place of the red preview error. Fixtures, tests, and the README picker
      docs.'
  - id: mac_preview
    title: Bob Mac Capture relationship preview rows, owner header, and notifications
    depends_on:
      - execute
      - mac_pickers
    size: medium
    description:
      "mac_preview: give `task_dependency` items an owner header. Render one `⛓`/`🧱`
      row per relationship for every owner kind, in place of the headline and literal
      DEPENDS ON row. Stop printing the owner line twice, put the owner card first, and
      write relationship notification copy. Fixtures, tests, and README."
  - id: publish
    title: Final docs sweep, end-to-end verification, optional decision record
    depends_on:
      - execute
      - complete
      - mac_preview
    size: small
    description:
      "publish: check every RX/RXC vector against the shipped binaries. Reconcile
      docs/capture.md, README, and help text with what landed. Run a scratch-vault
      walkthrough of all three authoring flows. Write the decision strand only if the
      memory decision says yes."
decisions:
  decision_record:
    ask:
      Record the existing-owner relationship rules as a new strand in the decisions
      memory web?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.research.4g.linker.w1.w0
decided_by: auto
create_time: 2026-10-14 16:21:10
status: wip
---

- **PROMPT:**
  [prompts/202610/capture_relate_existing_tasks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_relate_existing_tasks.md)

# Plan: `&` and `*` on existing `@route+id` tasks, first-class

## 1. Where things stand

The bob-cli core already accepts an existing owner. The `&` epic (bob-cli-3u) and the
`*` epic (bob-cli-7n, closed today) made `&foo:bar @body+excercise`,
`@body+excercise *job:cleanup`, `Note @body+excercise &foo:bar`, and the narrow
`@route:id` alias parse and execute. What is missing is everything that makes the
feature feel supported. These are probes against a scratch vault with bob-cli at
`498e5ab`:

**Reliability defects (bob-cli)**

| Draft                                                                | Today                                                                                                                                                                                                                          | Must be                                              |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- |
| `@@body+excercise` + `&foo:bar` (inherited owner, relationship-only) | Writes the edge, plus an empty `- ` bullet under Exercise. The parse says `task_dependency`, but execution takes the sub-bullet path (`plan.rs` only routes `TaskToggle`/`PomodoroLink` kinds to `plan_dependency_only_item`). | Edge only, no bullet                                 |
| `@foo+old *job:cleanup` (owner is `[x]` Done)                        | Reopens the owner to `[*]`, keeping `[completion::]`. T gets an edge to it but stays Next. `plan_dependency_only_item` never checks whether a `*`-only owner is open.                                                          | Refuse; vault byte-identical                         |
| `+ &foo:bar`, `+ex &foo:bar`                                         | Creates the inbox tasks `+` and `+ex`                                                                                                                                                                                          | Incomplete parent selector (`task_parent`)           |
| `&foo:bar ^body:excercise`                                           | Creates the inbox task `^body:excercise`                                                                                                                                                                                       | `^route:id` owner alias                              |
| `^body:excercise &foo:bar`                                           | Refused: "cannot attach to a Pomodoro operator capture". Parse also reports `invalid_pomodoro_link`                                                                                                                            | `^route:id` owner alias                              |
| `&foo:bar +`, `*job:cleanup +`                                       | Correct `task_parent` need, plus an error diagnostic: "cannot attach to a Pomodoro-ledger-link capture"                                                                                                                        | `task_parent` need, no diagnostic                    |
| `@body+excercise &foo:bar p:2` (also `s:3`, `%`)                     | Parse: OK, mode `sub_bullet`, empty body. Execute: "mixes a Pomodoro action"                                                                                                                                                   | One accurate refusal, identical in parse and execute |
| `@body+excercise &foo:bar` + `- note`                                | Parse: OK. Execute: "task text is required; pass TEXT or pipe it on stdin"                                                                                                                                                     | One accurate refusal, identical in parse and execute |
| `@body+excercise &foo:bar *foo:bar`                                  | "cycle: body.md^excercise -> foo.md^bar -> foo.md^bar" (wrong path)                                                                                                                                                            | `foo.md^bar -> body.md^excercise -> foo.md^bar`      |
| `&foo:bar +` and `*job:cleanup +` pickers                            | Offer `@foo+bar` and `@job+cleanup` themselves as enabled rows; picking one makes a self-edge                                                                                                                                  | Disabled, with a reason                              |
| `^body:excercise &` picker                                           | No `owner`; the owner's own row is enabled                                                                                                                                                                                     | Owner reported; own row disabled                     |

**Presentation gaps (bob-cli human output and Bob Mac Capture)**

- **CLI headers.**
  - The existing-owner headers are `Add dependency to "Exercise"` and
    `Add blocker from "Exercise"`. Their capitalization and phrasing break the
    `✓ captured` / `toggled` / `ensured` pattern.
  - The `⛓` trailer names prerequisites as wikilinks with counts, while `🧱` rows name
    text. The owner's own status change only shows inside the raw task line.
- **Mac preview header.** A `task_dependency` item shows
  `body.md  updated  task_dependency` (the raw wire word), and then the raw owner
  `task_line` (with `[id::]`/`[dependsOn::]`) a second time above its own task card.
- **Mac `⛓` rows.** These print `⛓️ **DEPENDS ON:** …` with the Markdown asterisks
  visible. The headline ignores `role`, so a `*`-only existing owner is never named.
- **Mac picker headers.**
  - The `&`/`*` picker header names an existing owner by its raw token
    (`For: @body+excercise`).
  - A new-task owner gets the ownerless prompt even when the text is typed: the header
    passes `dependentText: nil`.
  - Bob's first group, `in_this_note`, falls into Open.
- **Mac ownerless draft.** `&foo:bar ` runs a dry run that fails, and the panel shows a
  red "Preview failed". That is exactly the moment the user needs a hint ("now name the
  task").
- **Mac `+` picker.** Inside a relationship draft it says "Choose a task to append to",
  and no rows are disabled.
- **Mac notification.** It says "Task Dependency captured".

So the work is not new grammar. It makes an existing-owner relationship a first-class,
trustworthy gesture end to end.

## 2. Design

### 2.1 One owner rule

> `&` and `*` hang on the item's task. Task text means a new task. An existing-task
> marker with no text means that task.

Existing-task spellings that can own `&`/`*`:

| Spelling     | Status            | Notes                                                                                                    |
| ------------ | ----------------- | -------------------------------------------------------------------------------------------------------- |
| `@route+id`  | **Canonical**     | Leading or trailing, before or after the modifiers                                                       |
| `@@route+id` | Inherited         | Any line; an ownerless relationship item uses it                                                         |
| `@route:id`  | Alias (unchanged) | Bare only; it never links                                                                                |
| `^route:id`  | **New alias**     | Bare only; it never links. Leading (`^body:excercise &foo:bar`) or trailing (`&foo:bar ^body:excercise`) |

`^route:id` becomes an alias because the `^` picker is how Bryan names "the task I am
working on", and "my current task now waits on X" is the most common thing to say. An
alias never links: with `&`/`*` present, `@route:id` and `^route:id` only name the
owner, exactly as the frozen `@route:id` narrow-alias contract from the `&` epic
(bob-cli-3u), reused by `*` in bob-cli-7n, says.

Keeping a session alive when an existing task takes over a blocked task's slot stays a
documented two-item idiom. The first item links the blocker into the session, so the
second item's consumption cannot empty it:

```text
@bob:ask-kelly

@bob+ask-kelly *job:cleanup
```

Suffixed owners (`#name`, `=…`, `!`) keep today's "must be bare … mixes a Pomodoro
action" refusal. Those really are Pomodoro actions.

Owners must live where `@route` can write: the inbox, area notes, and non-terminal
project notes. A prerequisite or blocked task can live anywhere. To relate two tasks
when only one is routable, flip the direction:
`@cash+budget *projects/house/kitchen:cabinets` instead of
`@projects/…+cabinets &cash:budget`. The docs teach this.

### 2.2 Three ways to author it

1. **Owner first.** Type `@body+` (scoped picker) or `+` (vault picker) and choose
   Exercise. Then type ` &` or ` *`. The picker header now reads **⛓ Exercise depends
   on…** or **🧱 Exercise blocks…**, with `body.md ^excercise · In Progress` beneath.
   Rows that would be self-edges or cycles are disabled, with Bob's reason.
2. **Relationship first.** Type `&` and choose Buy shoes. The draft `&foo:bar ` now
   needs `dependency_target`. Instead of a red preview error the panel shows a calm
   hint: **⛓ What depends on “Buy shoes”? Type a new task, or + for an existing one.**
   Typing ` +` opens the vault picker in owner mode, headed **⛓ Which task depends on
   “Buy shoes”?**. The pick inserts `@body+excercise` and the preview turns into the
   relationship rows.
3. **Current task.** Type `^`, pick the active task, and continue with ` &` or ` *`. It
   works because of the new alias.

The hint copy for the other shapes:

- `*T`: **🧱 What blocks “Do cleanup tasks”? Type a new task, or + for an existing one**
- `&P *T`: **Type the task between “Buy shoes” and “Do cleanup tasks”, or + for an
  existing one**

The `+`-picker owner-mode headers:

- **⛓ Which task depends on “Buy shoes”?**
- **🧱 Which task blocks “Do cleanup tasks”?**
- **Which task depends on “Buy shoes” and blocks “Do cleanup tasks”?**

When there are several targets, the first is named and the rest are counted: “Buy
shoes” + 1 more.

### 2.3 What a relationship-only item refuses

A bare-owner item (an existing owner and no task text) only edits Depends-On and BLOCKS
lines. Anything that needs text gets one shared message from `capture_language`. The
`capture-parse` diagnostic (new code `relationship_needs_text`, error severity, ranged
on the offending token) and the `bob capture` error use the same function, so the Mac
preview and the CLI always agree:

```text
`p:2` needs task text: `@body+excercise` with no text only adds task relationships;
add text to write a sub-bullet (`Note @body+excercise &foo:bar p:2`)
```

The same message covers `s:<N>`, `%…`, and child bullets. For child bullets the token is
`child bullets`, and the example moves the note onto line 1. A `*` owner that is Done or
Canceled is refused, both in the dependency-only path and the sub-bullet path:

```text
task ^old in foo.md is Done; a finished task cannot block another task (reopen it first)
```

`&` already refuses a closed owner, and that message stays as it is.

### 2.4 The look: terminal

Existing-owner items (`task_dependency`) adopt the sibling header pattern, an owner
status row, and one row per relationship. Each relationship row ends with the status of
the task on its far side:

```text
✓ updated  body.md
  [/] → [?]  Exercise  ^excercise
  ⛓ now depends on "Buy shoes"  foo.md ^bar  [ ] Ready
  🧱 now blocks "Do cleanup tasks"  job.md ^cleanup  [*] → [?] Blocked
  − removed its Task Link from BODY 10:10–10:35 (running) · deleted the empty Pomodoro
```

- **Dry run:** `[dry-run] ok would update  body.md`, `⛓ would now depend on …`,
  `🧱 would now block …`, `− would remove …`.
- **Unchanged owner:** `  [ ] stays Ready  Exercise  ^excercise`, the toggle output's
  phrasing.
- **Repeats:** `⛓ already depends on "…"  foo.md ^bar  [ ] Ready` and the existing
  `🧱 already blocks "…"` (no arrow).
- **Residents:** the existing `· blocked 4 project tasks` chip stays on `🧱` rows.

The shared `⛓` trailer becomes per-prerequisite rows for every owner kind:

- **New task:** `⛓ depends on "Buy shoes"  foo.md ^bar  [ ] Ready`. The new task's `[?]`
  is already visible on its task line.
- **Sub-bullet:** `⛓ now depends on …`. The `under` row shows the parent's final status
  as `under [ ] → [?] Exercise  ^excercise` when a relationship changed it. Relationship
  rows print after the sub-bullet and its authored children, never between them.

`bob capture -f json` keeps its shape; only the fixed behaviors change its values.

### 2.5 The look: Bob Mac Capture

The app renders only what Bob returns. The one piece of state it remembers is the
display text of `&`/`*` rows the user accepted (keyed by the inserted replacement), used
for hint copy. It falls back to the raw locator `foo:bar` when a token was typed by
hand.

- **`&`/`*` picker header**
  - Existing owner: **⛓ Exercise depends on…** / **🧱 Exercise blocks…**, with the
    subtitle `body.md ^excercise · In Progress`. If an older Bob sends no owner text,
    fall back to `@body+excercise`.
  - New-task owner: **⛓ “Buy Groceries!” depends on…**, else **New task depends on…**.
  - Ownerless: the title is unchanged. The subtitle becomes "Then type the task, or +
    for an existing one".
  - No word appears twice in the header.
  - Bob's `in_this_note` group renders first, titled **This note**.
- **`+`/`@route+` picker in owner mode** (present when Bob sends `picker.relationship`):
  - The §2.2 headers.
  - The status line and reopen chip read "Choose the task to relate".
  - Disabled rows look and act like disabled dependency-picker rows: dimmed, Bob's
    reason as the badge, and accepting only announces the reason.
- **Calm owner hint** for need `dependency_target` with no open picker:
  - The §2.2 copy, in secondary (not red) style.
  - No dry run is attempted while the need is outstanding.
- **Relationship rows**, used for every owner kind and in Bob's `dependency_updates[]`
  order:
  - `⛓ depends on · Buy shoes · foo.md ^bar · Ready`
  - `🧱 blocks · Do cleanup tasks · job.md ^cleanup · Next → Blocked`
  - Rows use the "already …" wording when `added == 0`. They show no arrow when the
    status is unchanged, keep the residents chip, and each has an accessibility label.
  - These rows replace the old headline, the literal `⛓️ **DEPENDS ON:**` row, and the
    waiting caption.
- **Owner header** for `task_dependency`:
  - The owner text in semibold, then `body.md ^excercise`, then `In Progress → Blocked`
    (just the status when unchanged).
  - This replaces `body.md updated task_dependency`.
  - When `task_blocks` cover the owner, the verbatim `previewBlockLines` stack is
    skipped and the owner's card renders first.
- **Notification** after submitting a `task_dependency` item:
  - Title: **⛓ Exercise now depends on Buy shoes**, **🧱 Exercise now blocks Do cleanup
    tasks**, `Exercise now depends on 2 tasks`, or mixed
    `Exercise · depends on 1 · blocks 1`.
  - Subtitle: the route label.
  - Body: the consumed-link summary, when there is one.

### 2.6 Wire contract (additive, schema version 1)

**`capture-parse`**

- Bare `^route:id` beside modifiers reports `dependency_target`
  `{kind: existing_task, route, block_id}` and mode `task_dependency`, or `sub_bullet`
  when there is text.
- A `+`/`+query` token that is the item's whole remaining body beside modifiers reports
  mode `incomplete` and needs `[task_parent]`, keeps `dependencies`/`blocked_tasks`, and
  has no diagnostics. The exact lone `+` with nothing else stays the Pomodoro
  adjustment.
- New diagnostic code: `relationship_needs_text`.

**`capture-complete`**

- **`owner`** gains optional fields:
  - `text`: the owner's task text, or the draft body for `new_task` when non-empty.
  - `note_path`: the vault-relative path, with extension.
  - `status_symbol` and `status_name`.

  An existing owner whose task cannot be resolved keeps today's three fields.

- **`picker.relationship`** is added on `task_parent` and scoped `task`/`parent_task`
  responses when the cursor's item has complete `&`/`*` modifiers:

  ```json
  "relationship": {
    "dependencies":  [{"raw": "&foo:bar", "note_path": "foo.md", "block_id": "bar", "text": "Buy shoes"}],
    "blocked_tasks": [{"raw": "*job:cleanup", "note_path": "job.md", "block_id": "cleanup", "text": "Do cleanup tasks"}]
  }
  ```

  Unresolvable entries keep `raw`, `note_path` (the locator), and `block_id`, and drop
  `text`.

- **Parent-row `disabled_reason`** when `relationship` is present:
  - `the prerequisite itself`
  - `the blocked task itself`
  - `would create a dependency cycle` (using the discovered Depends-On edges plus the
    item's typed modifiers; the writer stays authoritative)
  - `a project cannot depend on its own resident`

  Disabled rows carry no insertable replacement. Shell completion omits them.

### 2.7 Vectors

Execution and parse vectors **RX**. Each one asserts the parse result, the exact vault
bytes, the JSON, the human output, dry run equal to live, and a second live run being a
no-op where one applies.

| #    | Draft / state                                                                                  | Outcome                                                                                                                                                        |
| ---- | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RX1  | `&foo:bar @body+excercise`, `@body+excercise &foo:bar`                                         | Exercise → Buy shoes; Exercise `[?]`; no bullet; mode `task_dependency`                                                                                        |
| RX2  | `@body+excercise *job:cleanup`                                                                 | cleanup → Exercise; cleanup `[?]`; Exercise gets the normal commitment transfer (task-dependencies.md §5)                                                      |
| RX3  | `@body+excercise &foo:bar *job:cleanup`                                                        | cleanup → Exercise → Buy shoes                                                                                                                                 |
| RX4  | `^body:excercise &foo:bar`, `&foo:bar ^body:excercise`, `^body:excercise *job:cleanup`         | Same as RX1 / RX2; parse owner `existing_task`                                                                                                                 |
| RX5  | `@body:excercise &foo:bar`                                                                     | Unchanged alias behavior (regression)                                                                                                                          |
| RX6  | `@@body+excercise` + newline + `&foo:bar`; the same with a blank line between                  | Edge only; **no empty bullet**; `inherited: true`                                                                                                              |
| RX7  | `@foo+old *job:cleanup`, `Note @foo+old *job:cleanup` (owner `[x]`/`[-]`)                      | Refused with the §2.3 message; vault byte-identical                                                                                                            |
| RX8  | `+ &foo:bar`, `+ex &foo:bar`, `&foo:bar +`, `*job:cleanup +ex`                                 | Incomplete `task_parent`; no diagnostics; execute refuses with "`+` needs a task selection; choose the task to relate before capture"; never a task named `+…` |
| RX9  | `@body+excercise &foo:bar p:2` / `s:3` / `%`                                                   | `relationship_needs_text` in parse and the identical execute error                                                                                             |
| RX10 | `@body+excercise &foo:bar` + `- note`                                                          | Same, with the token `child bullets`                                                                                                                           |
| RX11 | `@body+excercise &foo:bar *foo:bar`                                                            | Cycle refused, with the path `foo.md^bar -> body.md^excercise -> foo.md^bar`                                                                                   |
| RX12 | `Waiting @body+excercise &foo:bar` + `- detail`                                                | Human output: the sub-bullet and its child, then the `⛓` row; `under` shows `[ ] → [?]`                                                                        |
| RX13 | `@body+excercise#bugs &foo:bar`, `@body+excercise! *job:cleanup`, `^body:excercise=x &foo:bar` | Today's "must be bare" refusals, unchanged                                                                                                                     |
| RX14 | Owner `[/]` linked in the running Pomodoro, `@body+excercise &foo:bar`                         | Today's consumption (`−` row, entry deleted when emptied); new layout                                                                                          |
| RX15 | Repeat any RX1–RX4                                                                             | `already depends on` / `already blocks`; no write                                                                                                              |
| RX16 | New task `Buy milk &foo:bar`                                                                   | Per-prerequisite `⛓ depends on "Buy shoes"  foo.md ^bar  [ ] Ready` row                                                                                        |

Completion vectors **RXC**:

| #    | Draft (cursor at end)                    | Outcome                                                                                                                                               |
| ---- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| RXC1 | `@body+excercise &`                      | `owner` has `text: "Exercise"`, `note_path: "body.md"`, status `/` In Progress                                                                        |
| RXC2 | `^body:excercise &`, `^body:excercise *` | Same owner; the Exercise row is disabled (`the dependent task itself` / `the owner task itself`)                                                      |
| RXC3 | `Buy milk &`                             | `owner: {kind: new_task, text: "Buy milk"}`                                                                                                           |
| RXC4 | `&foo:bar +`                             | `task_parent`; `picker.relationship.dependencies[0].text == "Buy shoes"`; `@foo+bar` disabled (`the prerequisite itself`); a would-cycle row disabled |
| RXC5 | `*job:cleanup +`                         | `relationship.blocked_tasks`; `@job+cleanup` disabled (`the blocked task itself`)                                                                     |
| RXC6 | `&foo:bar @body+`                        | Note-scoped picker with the same `relationship` and guards                                                                                            |
| RXC7 | `+ &foo:bar`, cursor after `+`           | `task_parent`; the replacement covers only the `+` token                                                                                              |
| RXC8 | Any draft without modifiers              | Byte-identical to today (no `relationship`, no new owner fields)                                                                                      |

### 2.8 Out of scope

- Linking an existing blocker through `@route:id *T`. Aliases stay non-linking, per the
  bob-cli-3u/7n contract and `decisions:capture-star-blocks-existing-tasks`; the
  two-item idiom above covers the session case.
- Owners in unroutable notes (the §2.1 flip covers it).
- `p:`/`s:` on an existing task.
- Child bullets under a bare `@route+id` with no relationships.
- Plugin or reconcile behavior.
- The `^` picker opening after a modifier (`&foo:bar ^`): `+` is the one owner picker.

Coordinate with these in-flight beads and do not duplicate them:

- bob-cli-7q (consumption parity)
- bob-cli-73 (Mac `link_guard` rendering)
- bob-cli-3p (reverse Blocks… stage)

## Phase: contract — docs contract, wording, and RX/RXC vectors

**Repo:** bob-cli. No code.

1. **New section in `docs/capture.md`:** "Relating existing tasks with `&` and `*`",
   placed after "Blocking existing tasks with '\*'". Add it to Contents. It holds:
   - the §2.1 owner rule and spelling table;
   - the alias-never-links note and the two-item session idiom;
   - the owner-placement flip;
   - the §2.2 flows;
   - the §2.3 refusals with exact messages;
   - the §2.4 CLI layout, with real-shaped examples;
   - the §2.6 wire additions, with frozen JSON examples for RXC1, RXC4, and RX8's parse;
   - the RX and RXC tables.
2. **Other parts of `docs/capture.md`:**
   - Grammar at a glance: add `^route:id &note:id` / `*note:id` rows; reword the
     `&note:block-id` / `*note:block-id` "no other text" rows to point at `+`; add a
     `+query &note:id` row.
   - The `&` and `*` section tables: add the `^` alias rows.
   - The `capture-complete` section: the `owner` fields and `picker.relationship`.
   - The `capture-parse` section: the needs/codes list (`relationship_needs_text`).
   - Human-output examples elsewhere in the file that show the old `⛓ … ([[…]] …)`
     trailer or the `Add dependency to` / `Add blocker from` headers.
3. **`README.md`:** the grammar rows near the `&`/`*` rows, including `^route:id` and
   the `+` flow.
4. **`docs/task-dependencies.md`:** one pointer from §13.5.2 (ST vectors) to the RX
   table. Do not copy it.

Wording is a contract for later phases: they test these strings verbatim. If a later
phase must change a string, it updates this section in the same commit.

## Phase: grammar — owner grammar, `^route:id` alias, `+` selector, agreeing refusals

**Repo:** bob-cli, `src/native/capture_language/`. Implements §2.1, §2.3 (parse side),
and the §2.6 `capture-parse` items. Vectors: the parse half of RX4, RX5, RX8–RX10, and
RX13.

1. **`^route:id` alias.**
   - `resolve_execution_ownership` (`dependencies.rs`): accept
     `PomodoroLinkSpelling::Caret` exactly like `At` when bare (no `#name`, `=`, close,
     or creation modifiers).
   - Classify the caret link on the modifier-stripped body, so both positions work.
     Today the leading form trips `invalid_pomodoro_link` and the trailing form becomes
     body text.
   - Mirror this in the editor ownership path (`editor_parse.rs`; see
     `pending_colon_dependency_target`), so `capture-parse` and `editor_item_at` report
     the owner.
2. **`+` selector beside modifiers.**
   - `parent_task_selector_tokens` (`tokens.rs`): treat a `+`/`+query` token as the
     selector when it is the item's whole remaining body after the relationship
     modifiers (and transparent markers) are removed, not only when it is the line's
     last token.
   - The exact lone `+` item with no modifiers stays the dual-use adjustment.
   - Execution refuses with the RX8 message. The editor outcome keeps the item's
     `dependencies`/`blocked_tasks`.
3. **No bogus diagnostic.** The incomplete parent-selector outcome never receives
   `dependency_unsupported_target`. Fix `dependency_owner_mode_word`'s
   `_ => "Pomodoro-ledger-link"` fallback, so no mode is mislabeled. A real ledger-link
   rejection keeps its wording.
4. **`relationship_needs_text`.**
   - One message builder in `capture_language` for a bare-owner item that carries `p:`,
     `s:`, `%…`, or child bullets.
   - The editor emits it as a ranged error diagnostic.
   - `resolve_execution_ownership` / item resolution returns the same string instead of
     `invalid_relation_target_error` (for `has_creation_modifiers`) or the downstream
     "task text is required" error.
   - `#name`/`=`/`!` keep `invalid_relation_target_error`.
5. **Help.** Update the `capture-parse` help text and examples (`capture_parse.rs`):
   `^r:id &foo:bar`, `&foo:bar +`.
6. **Tests.** Inline lexer and editor tests for every vector above, plus the STP
   regression rows. Update `tests/cli/capture/parse_dependency.rs` and the star parse
   tests where they pinned the old diagnostics.
7. Run `just check` (fmt, clippy, full `cargo test`).

## Phase: execute — reliable existing-owner execution and the relationship CLI layout

**Repo:** bob-cli, `src/native/capture/`. Implements §2.3 (execution side) and §2.4.
Vectors: RX1–RX7 and RX9–RX16.

1. **Inherited owner (RX6).** In `plan_capture_item` (`plan.rs`), also route
   `CaptureKind::SubBullet { target: BlockId }` with an empty body, no authored
   children, and an `ExistingTask` dependency target to `plan_dependency_only_item`. The
   root cause is that only `TaskToggle`/`PomodoroLink` reach it today. The `inherited`
   owner then writes the edge only.
2. **Closed `*` owner (RX7).** Before any `*` pass, reject an owner whose status is Done
   or Canceled with the §2.3 message. Do this in `plan_dependency_only_item` and in the
   sub-bullet and forced `--task`/`--route` `*` paths, through one shared guard. `&`
   keeps its existing check.
3. **Cycle text (RX11).** In `reject_new_cycles` (`dependencies.rs` near the
   `task dependency cycle:` format), render the cycle once, from the new edge's source:
   `source -> target -> … -> source`, with no duplicated final hop. Check `reaches`'
   path semantics before you change the join.
4. **Human output (`output.rs`).**
   - Replace `print_human_task_dependency_success`'s `Add dependency to` /
     `Add blocker from` header with `updated` / `would update  <note>`.
   - Add the owner status row (`prev → next` or `stays <status>`, the toggle phrasing).
   - Replace `print_human_dependency_line`'s wikilink trailer with per-prerequisite rows
     for every owner kind: new task (`depends on`), existing (`now depends on` /
     `would now depend on` / `already depends on`). Each row ends with the
     prerequisite's status marker and name.
   - Keep `print_human_blocked_task_line` aligned with the same column rhythm.
   - Sub-bullet items print relationship rows after the bullet and its children. Their
     `under` row shows the parent's final status, with an arrow when it changed.
   - Keep the `−` consumption rows as they are.
5. **Help.** `bob capture --help`: describe the owner rule, the `^route:id` alias, and
   the `+` flow in the `&`/`*` paragraphs.
6. **Tests.**
   - A new `tests/cli/capture/relate_existing.rs`, registered in `mod.rs`, covering
     RX1–RX7 and RX9–RX16. Assert exact vault bytes, JSON, and human output, that dry
     run equals live, and that a repeat is a no-op.
   - Update the existing `&`/`*` human-output assertions (`blocked_tasks.rs`,
     `parse_dependency.rs`, and the sub-bullet dependency tests) to the new layout.
7. Run `just check`. The `docs/capture.md` examples must match the shipped output
   byte-for-byte. Fix the docs in this commit if a string had to change.

## Phase: complete — owner-aware completion wire and guarded owner pickers

**Repo:** bob-cli, `src/native/capture_complete/` and `capture_language/completion.rs`.
Implements the §2.6 `capture-complete` items. Vectors: RXC1–RXC8.

1. **Owner fields.**
   - Where the engine builds `owner` (`engine.rs`, the `(query, owner)` block), resolve
     an existing owner through the same vault lookup the candidate builders use, and
     fill `text`, `note_path`, `status_symbol`, and `status_name`.
   - For `new_task`, set `text` from the editor item's body.
   - Extend the model in `model.rs` additively: fields are omitted when absent.
2. **`picker.relationship`.** In `picker_descriptor` for `task_parent` and the scoped
   `task` picker, when `editor_item_at` reports complete modifiers, resolve each one's
   `note_path` and `text`. The resolver is vault-wide, like `&note`.
3. **Disabled parent rows.**
   - Compute the four §2.6 reasons in `task_parent_candidates` / the scoped task
     candidates, reusing the cycle and internal-`^prj` helpers that
     `blocked_task_candidates` already has (`candidates.rs` around the
     `the owner task itself` branch).
   - Disabled rows get no replacement.
   - `^route:id` owners (from the grammar phase) feed the existing `&`/`*` self and
     cycle guards (RXC2).
4. **Shell completion.** `shell.rs` skips disabled parent rows.
5. **Help.** `capture-complete` help text for the new fields.
6. **Tests.** A new `tests/cli/capture/complete_relationship_owner.rs`, registered in
   `mod.rs`, covering RXC1–RXC8, including the byte-identical regression for drafts
   without modifiers.
7. Run `just check`. Freeze the RXC JSON examples in `docs/capture.md` to the real
   output.

## Phase: mac_pickers — owner-aware pickers and the calm owner hint

**Repo:** bob-mac-capture. Read its `AGENTS.md` first. Implements the §2.5 picker items
and the hint. The app stays a renderer (`decisions:mac-capture-is-a-thin-client`).

1. **Decode** (additive, `decodeIfPresent`):
   - `DependencyOwner` gains `text`, `notePath`, `statusSymbol`, `statusName`
     (`CaptureModels.swift`, near `DependencyOwner`).
   - The parent picker descriptor gains `relationship`.
   - Parent candidates decode `disabled_reason`.
   - Per-item parse `dependencies` / `blocked_tasks` (raw, note, block_id) and
     `dependency_target`, used for the hint.
2. **`&`/`*` header** (`DependencyPickerPresentation.headerTitle`/`headerSubtitle`, call
   site `CapturePickerView.swift` around 258–297):
   - Implement the §2.5 header copy, with direction-aware glyphs.
   - Fix the always-nil `dependentText` path by using `owner.text`.
   - Add a `.thisNote` group for wire `in_this_note`, rendered first and titled **This
     note**.
3. **Owner-mode parent picker** (`ParentTaskPickerPresentation.swift`,
   `CapturePickerPresentation.swift` near the "Choose a task to append to" and "Select a
   parent task" / "Append to a task" strings):
   - When `relationship` is present, use the §2.2 headers and "Choose the task to
     relate".
   - Render disabled rows the way `DependencyPickerPresentation` does: dimmed, with the
     badge. Accepting one only announces the reason.
4. **Calm hint.**
   - Add a `CapturePickerNeed` case for `dependency_target`. Handle it in `pickerNeed`
     (`CapturePanelModel.swift` around 4547–4594), so no dry run runs and no red error
     shows.
   - Show the §2.2 hint copy. Names come from a `[replacement: text]` map filled when an
     `&`/`*` row is accepted and cleared on draft reset; fall back to `note:block-id`.
5. **README:** the Dependency Picker, Blocks Picker, and parent-picker paragraphs
   (around lines 565–630) describe the new headers, owner mode, and the hint.
6. **Fixtures and tests.**
   - fake-bob routes and JSON fixtures for:
     - `@body+excercise &` and `^body:excercise *`, with owner text;
     - `Buy milk &`, a new-task owner with text;
     - `&foo:bar +` and `*job:cleanup +`, with relationship and disabled rows;
     - the `&foo:bar ` parse with `dependency_target`.
   - Presentation tests for every header string, the hint variants, the disabled-row
     announce, and `This note` ordering.
   - Repair the mismatched `CaptureDependencyPanelTests` existing-owner case ("Ask Kelly
     \*").
7. **Verify.**
   - Run `just portable-check` on a host with Swift.
   - Then compile a scratch copy on the Mac exactly as `AGENTS.md` describes: rsync
     without `.build`/`.git`, run `bundle.sh` and `just typecheck-tests`, then delete
     the copy.
   - After the commit lands, check that commit's macOS CI run. Known unrelated reds:
     bob-cli-7h and bob-cli-7p.
   - If the Mac is offline, say in the final report that the macOS targets are
     unverified.

## Phase: mac_preview — relationship preview rows, owner header, and notifications

**Repo:** bob-mac-capture. Implements the §2.5 preview and notification items. The
inputs come from the execute phase's real dry-run JSON; record fixtures from the shipped
bob, never by hand.

1. **Relationship rows** (`CapturePanelView.swift`, the dependency section around
   3156–3199; `CaptureModels.swift`, the dependency headline around 3179):
   - One row per prerequisite and per blocked task, for every owner kind, following
     §2.5.
   - Remove the headline, the literal `⛓️ **DEPENDS ON:**` row, and the waiting caption.
   - Add "already …" handling, no arrow when the status is unchanged, the residents
     chip, and an accessibility label on every row (the `*` branch has none today).
2. **Owner header** for `task_dependency`:
   - A `CaptureRelationshipPresentation` beside `CaptureSubBulletPresentation`, giving
     the owner text, `note ^id`, and the status transition.
   - Use it in place of `standardHeader`'s raw `kind`.
3. **No duplicate owner line.**
   - Extend `CaptureTaskBlockPresentation.covers` to `task_dependency`, so the verbatim
     `previewBlockLines` stack and the trailing `relativeTarget` line are skipped when
     the owner's card is present.
   - Order cards owner first, then the relationship-row order.
4. **Notifications** (`NotificationService.swift` around 499–508 and 690–724):
   `task_dependency` titles, subtitle, and body per §2.5.
5. **README:** the preview and notification paragraphs.
6. **Fixtures and tests.**
   - Dry-run fixtures recorded from the shipped bob for RX1, RX2, RX3, RX4, RX14 (with
     consumption), RX16 (new task), and a sub-bullet with `&`.
   - fake-bob routes for each.
   - Presentation tests for row text, the arrow and "already" logic, header text,
     covered-stack suppression, card order, and notification titles.
7. **Verify** exactly as in `mac_pickers` step 7.

## Phase: publish — final docs sweep, end-to-end verification, optional decision record

**Repo:** bob-cli (docs, plus memory only if decided).

1. **Check every RX/RXC vector** against the landed bob binary in a scratch vault with
   the Tasks plugin status settings. Fix any doc string that drifted.
2. **Reconcile everything with what shipped:** `docs/capture.md` (examples are
   byte-exact), `README.md`, the three `--help` texts, and the Mac README.
3. **Walk the three §2.2 flows** through `capture-complete` / `capture-parse` /
   `capture --dry-run`, the way the Mac drives them, and note the transcript in the
   phase bead.
4. **Decision record.**

> [!decision] decision_record Write a new strand in the `decisions` web, for example
> `existing-task-owners-relate`, recording:
>
> - **Claim.** The owner rule and spelling table (aliases never link). Bare-owner items
>   refuse text-bearing extras with one shared message. Closed owners can never block.
> - **Why.** The defects in §1, with these rejected alternatives: linking through
>   `@route:id *T`, applying `p:`/`s:` to the existing owner, and a separate owner
>   sigil.
> - **Cost.** Aliases differ from the new-task `N @route:id *T` linking idiom.
> - **Reopens when.** Bryan wants alias linking or owners in unroutable notes.
>
> Cite the phase commits. Link `[[decisions:capture-star-blocks-existing-tasks]]` and
> `[[decisions:mac-capture-is-a-thin-client]]`. Run `sase memory init`. When the answer
> is no, record no memory change.

5. Run `just check`.
