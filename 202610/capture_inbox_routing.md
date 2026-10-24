---
tier: epic
title: Capture asks where inbox tasks go
goal: 'Every capture that changes the status of an existing inbox task makes Bob Mac
  Capture ask which area or open project the task belongs in, and bob applies the
  capture and the move in one atomic batch.

  '
phases:
- id: route-detect
  title: Detect inbox route candidates and refuse without a choice
  depends_on: []
  size: medium
  description: 'route-detect: bob capture finds existing inbox tasks whose checkbox
    the batch changes, reports them with route destinations in `inbox_route`, and
    refuses write-free under hidden `--require-inbox-route-choice`.'
- id: route-apply
  title: Apply inbox route answers inside the capture batch
  depends_on:
  - route-detect
  size: medium
  description: 'route-apply: `-i/--inbox-route` answers move candidates through the
    task-move engine inside the same capture batch, with freshness stamps, link repair,
    refusals, followed mirrors, and docs.'
- id: mac-route-client
  title: Decode inbox routes and send route answers in CaptureCore
  depends_on:
  - route-apply
  size: medium
  description: 'mac-route-client: CaptureCore decodes `inbox_route`, sends answers
    and the guard flag behind a per-identity capability, models the route choice state,
    and documents the protocol.'
- id: mac-route-card
  title: Route card in the Bob Mac Capture panel
  depends_on:
  - mac-route-client
  size: medium
  description: 'mac-route-card: the panel shows the route card on Return after any
    pull-in card, resubmits the answers, handles refusals, documents the card, and
    lands the memory updates.'
decisions:
  route_closes:
    ask: Should completing an inbox task by capture (`!note:id`, `x!N`) also ask where
      it goes?
    choices:
      route: Completions ask too; the Done task moves home before archive, unlike
        Obsidian
      skip: Completions never ask, as in Obsidian; Done tasks leave the inbox via
        `bob task archive`
    default: route
    why: You asked for a prompt when a capture changes an inbox task status "in any
      way"
    answer: route
  route_side_effects:
    ask: Should side-effect status changes (a Blocked inbox task unblocking) also
      ask?
    choices:
      all: Every inbox task whose checkbox the capture changes asks, recoveries included
      named: Only tasks the draft names or the pull-in card picks ask; recoveries
        stay silent
    default: all
    why: One before/after diff rule covers every capture path and matches "always"
    answer: all
  memory_inbox_file:
    ask: Update the glossary Inbox File entry to say captures ask where inbox tasks
      go?
    memory:
    - glossary:inbox-file
    default: true
    corrects: Ctrl+Shift+M also moves it, following it to the destination, and a closed
      task leaves through `bob task archive`.
    answer: true
  memory_decision_record:
    ask: Add a decisions record for capture inbox routing (bob detects and moves,
      the Mac asks)?
    memory:
    - decisions:capture-asks-where-inbox-tasks-go
    default: false
    answer: false
proposed_by: bbugyi200.apollo.a9
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 14:00:25
status: wip
bead_id: bob-cli-bk
---

- **PROMPT:** [prompts/202610/capture_inbox_routing.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_inbox_routing.md)
- **BEAD:** [bob-cli-bk](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bk/README.md)

# Plan: Bob Mac Capture asks where an inbox task goes when a capture changes its status

## Context

Inbox notes hold captured tasks waiting for triage. An inbox note is `inbox.md` itself,
or an area note whose frontmatter `parent` resolves to `inbox.md`. Today that means
`mac_inbox.md` and `gkeep_inbox.md`; see `glossary:inbox-file`.

Obsidian already asks where a task goes when it is triaged. On an open inbox task,
Ctrl+Shift+P and Ctrl+Shift+Enter open a route picker as the last step before they
write. The picker is titled `Route out of <inbox>` and lists areas and open projects,
never another inbox. `↵` applies the answer and then moves the task, and `⇧↵` applies it
in place. The design is `plan:202610/inbox_routing.md`; the docs are `docs/projects.md`
"Inbox routing". That contract currently lists `bob capture` and Bob Mac Capture under
"What does not route".

This epic closes that gap. Whenever a capture changes the checkbox status of an existing
task that lives in an inbox note, Bob Mac Capture asks which area note or project note
the task should move to. bob then applies the capture and the move as one atomic batch.

The constraints are fixed by accepted decisions:

- **`decisions:mac-capture-is-a-thin-client`.** bob is the only grammar, preview, and
  write authority. The app renders what bob returns and submits each draft as exactly
  one `bob capture` call. So bob must detect the affected tasks, list the valid
  destinations, and perform the moves. The app only asks and passes the answers back.
- **`decisions:unblocked-tasks-are-offered`.** It set the exact precedent with successor
  offers. `bob capture` never prompts. Callers make an explicit selection with a flag.
  The app learns about a needed choice from its live dry-run preview, or from a
  write-free refusal under a hidden guard flag (`--require-pull-in-choice` →
  `code: pull_in_choice_required`). It shows an in-panel card, then resubmits with
  `-p/--pull-in`. This epic mirrors that pattern end to end.
- **`decisions:bob-cli-releases-ride-a-daily-train`.** Agents never edit the
  `Cargo.toml` version or `CHANGELOG.md`. Every change here is additive (`feat(...)`,
  not a `!` break).

Repositories: the bob-cli phases work in this repo. The Mac phases work in the linked
`bob-mac-capture` repo; open it with `sase repo open bob-mac-capture` and read its
`AGENTS.md` first. bob-plugins needs no change.

## Design contract (all phases)

### Which captures prompt

A **route candidate** is found after the whole batch is planned. By then every item is
planned, the successor net post-pass has run, and the final `--pull-in` selection is
applied. bob then compares each staged file's original text with its current text. A
task is a candidate when all of these hold:

1. It existed in the original text of a note that is an inbox note. Classify the note on
   its original text with `task_complete::successors::is_inbox_note` and the batch's
   staged note index. Never cache the classification across runs. A project or untyped
   note filed under an inbox is not an inbox note.
2. Its checkbox symbol in the final text differs from the original. Only the status
   counts: edits to fields alone, such as `[fresh::]`, Depends-On, priority, or
   children, never make a candidate.
3. It is still in the same note in the final text.
4. It is not `^prj`.

More rules:

- **Nested tasks.** When the changed line is a nested task, its column-0 `#task`
  ancestor is the candidate, because the move carries the nested line. A changed nested
  task with no column-0 task ancestor is skipped.
- **De-duplication.** Candidates are de-duplicated per column-0 block.
- **New tasks** this batch creates are never candidates.

> [!decision] route_closes = skip Closing transitions (final status `[x]` or `[-]`) are
> not candidates. The move engine keeps refusing closed selections, so no allow-closed
> option is added in `route-apply`. Drop the completion vectors and say "closes never
> route, matching Obsidian" in the docs.

> [!decision] route_side_effects = named Also require the candidate's identity to be one
> the draft named, or one picked through `--pull-in`. The planner collects these
> identities from the parsed items. Automatic recoveries and promotions never prompt.
> Vectors cover both sides of the rule.

Under the defaults, these paths produce candidates when the task lives in an inbox:

- linking an existing task, which moves Ready or Blocked to Next: solo `@route:id` or
  `^route:id`, link starts, `@route+id` Ensure Next, and the link direction of
  `@route+id!`
- `@route:id=x…`
- `x` rows that set `[/]`
- `!note:id`, `x!N`, and other completions
- `&` and `*` making an existing task Blocked
- prerequisite promotion to Next
- automatic dependent recovery from Blocked to Ready
- picked successor offers

These never produce candidates: an unlink, `=` and `=~K` starts, sub-bullets, relations
(`~`), and a link to an already-Next task.

### `bob capture` contract

- **Report.** Every successful capture JSON (dry-run and real run alike) carries a new
  batch-level `inbox_route` object. Its fields are:
  - `supported: true`, which is the capability flag the app keys on
  - `candidate_count`, `routed_count`, `kept_count`
  - `candidates[]`
  - `destinations[]`, present only when `candidate_count > 0`, so ordinary previews pay
    nothing

  Each candidate has:
  - `key` and `n`
  - `note_path` and `inbox` (the inbox basename)
  - `block_id`, which is `""` when the task has none
  - `line`, the 1-based line in the inbox note's final pre-move text
  - `text`, the clean display text, using the same cleaning as `task_complete.text`
  - `previous_status_symbol` / `previous_status_name` and `status_symbol` /
    `status_name`
  - `answer`: `null`, `{"action":"keep"}`, or
    `{"action":"move","destination":"<relative_path>"}`
  - `outcome`: `pending`, `kept`, or `moved`
  - `moved` (when moved): `destination`, `line`, `block_id`, `previous_block_id`, and
    `fresh` (the stamped date, or `null`)

  The batch also carries `repaired_files[]` (`path`, `link_count`,
  `dependency_token_count`), `renamed_block_ids[]`, and `needs_reconcile` once a move is
  applied.

- **Keys.** A key is `path#^id`, or `path@<sha8>[~ordinal]` for a task without a block
  ID. It is computed on the original line with the same digest rule as successor-offer
  keys (`offer_key_for` in `task_complete/successors.rs`). Factor that rule into a
  shared helper without changing offer keys. Clients treat keys as opaque. `n` numbers
  candidates from 1, ordered by note path and then original line.
- **Destinations.** Every Markdown note outside `done/`, `_templates/`, and
  `_conflicts/` whose kind passes `task_move`'s destination rule: an area, or a project
  with status `wip` or `waiting` (`classify_dest`). Inbox notes are removed, so the list
  never offers another inbox. The list is sorted by relative path, matching Obsidian's
  `collectTaskMoveDestinations`. Each row has:
  - `relative_path`
  - `route`: the relative path without `.md`
  - `label`: the basename without `.md`
  - `kind`: `area` or `project`
  - `status`: the project status label, or `null`
  - `parent`: the resolved parent label, or `null`
  - `blocked`: the same `^prj` Blocked test as `task_move`'s `DestBlocked`
- **Answers: `-i, --inbox-route ANSWER`.** The flag is public, repeatable, and
  alphabetical among the options. Each `ANSWER` is one of:
  - `<N|KEY>=<NOTE>`: move that candidate to `NOTE`.
  - `<N|KEY>=`: keep that candidate in its inbox.
  - `keep`: keep every candidate not otherwise answered.

  Split on the first `=`. `NOTE` resolves like `bob task move`'s `DEST` (stem or
  vault-relative path, with or without `.md`) and must be in the destination set.
  Conflicting answers for one candidate are a usage error (exit 2).

- **Guard: hidden `--require-inbox-route-choice`.** It is an internal subprocess flag:
  no short alias, `.hide(true)`, and absent from `--help`. When a candidate has no
  answer, capture writes nothing and fails with exit 1:

  ```json
  {"ok": false, "code": "inbox_route_choice_required",
   "error": "this capture changes 1 inbox task; pass --inbox-route",
   "inbox_route": {"supported": true, "candidate_count": 1, "candidates": [...], "destinations": [...]}}
  ```

  It is checked after the pull-in choice refusal, so a draft that needs both choices
  asks for successors first. Picks can change which tasks change status.

- **Other refusals.** Both write nothing and carry `inbox_route`:
  - `code: inbox_route_stale`, exit 2. An answer names an `N` or `KEY` that is not a
    current candidate. The error lists the current candidates, exactly as `-p` does for
    stale picks.
  - `code: inbox_route_refused`, exit 1. The destination does not resolve or is not in
    the set (missing, ambiguous, an inbox, under `done/`, a terminal project), or
    `task_move` refuses the plan (a project without `## Tasks`, a Blocked `^prj` for a
    non-Blocked task, a dependency on the destination `^prj`, an identity collision).
    The payload adds `refused: {key, n, destination, message}`.
- **Unanswered.** Without the guard flag, unanswered candidates stay where they are,
  like un-picked offers. The real run still succeeds and reports `outcome: pending`.
- **Human output.**
  - Each pending candidate prints an `inbox` row:
    `[ ] → [*] <text>  mac_inbox.md ^id · still in mac_inbox`.
  - One hint follows: `route with -i 1=<note> · -i keep`. Bob Mac Capture asks for it.
  - A moved candidate prints `routed  <text>  mac_inbox → health`, plus
    `· ^old renamed ^new` and `· N links repaired` when they apply.
  - A kept candidate prints `kept  <text> · still in mac_inbox`.
  - The `bob task reconcile` hint prints when `needs_reconcile` is true.
  - Dry-run output says `would route`.
  - Dry-run JSON equals real-run JSON with the same flags, except for `dry_run`.

### Writes: act, then move, in one batch

Answered moves run as a new stage inside capture planning, on the batch planner:

1. It runs after every item, the successor net post-pass, and candidate detection.
2. It runs before the task and Pomodoro block trackers finish and before
   `into_staged_files`.

For each source → destination group, in candidate order, the stage:

1. Builds a full Markdown snapshot. It uses the same file set `bob task move` repairs
   (`collect_done::transform::link_repair_markdown_files`, including `done/`), overlaid
   with every staged text (`staged_overlay_map`).
2. Plans the move with `task_move::plan_task_move`.
3. Stages every postimage through the planner. A file is loaded first, so its preimage
   is the text the snapshot saw and the commit's preimage guard covers every rewritten
   file.
4. Stamps each moved task's root line with `freshness::stamp_fresh` (dated today in the
   configured timezone) as the line's last transformation, when the task is still open
   and in freshness scope. This matches "routed moves stamp like Ctrl+Shift+M" in
   `docs/freshness.md` §5. Closed tasks are never stamped.

The move engine already repairs everything this needs:

- the Task Link the same capture just wrote into today's daily note, which becomes
  `[[health#^id]]`
- `[id::]` and `[dependsOn::]` tokens
- block-ID collisions, by renaming with the next suffix

Two crate-internal additions to `task_move` (the CLI `bob task move` stays
byte-identical):

- an option that accepts a closed selection, used only by capture routing
- an exact-line selector for candidates without a block ID; never select by fuzzy text

These outputs describe the final, post-route text:

- `task_blocks` and `pomodoro_blocks`: a routed task's entry follows it to the
  destination path and its final lines and block ID.
- `open_notes`: an entry that would land on a routed task opens the destination at the
  moved line instead.

Capture still takes no maintenance lock and makes no git commit. That is deliberate: the
preimage guard and the all-or-nothing commit cover the extra files. If any file changed
since planning, the whole batch is rolled back with the existing `NoteChanged` failure.

### Bob Mac Capture route card

**Capability.** The app remembers per bob executable identity (`identityToken`) that any
decoded capture JSON carried `inbox_route.supported == true`. Once it is remembered,
every real submit sends `--require-inbox-route-choice`. That is what makes "always"
hold, even when Return beats the live preview. An older bob never advertises the
capability, so it never receives either flag and behaves as it does today. The app also
adds `--require-inbox-route-choice` to the guard-flag retry for an `unexpected argument`
rejection (`unsupportedGuardFlag`).

**Preview.** When the matching live preview has candidates, the preview shows an inbox
summary row, in the style of `pullInPreviewSummary`:

- `Inbox · <text> · Ready → Next · ↩ choose where it goes`
- `<N> inbox tasks · ↩ choose where they go` for several candidates

**Order on Return.** The pull-in card comes first when it would open.

- **Opened locally.** With no pull-in step, if the matching preview snapshot has
  candidates, the route card opens locally without a spawn.
- **Opened by refusal.** Otherwise the `inbox_route_choice_required` refusal opens it,
  with the refusal's `candidates` and `destinations`. After a pull-in choice, the route
  card always opens from that refusal, because picks can change the candidates.

**Card.** The card is hosted exactly like `CapturePullInChoiceCard`: the first branch of
the auxiliary region, the editor input locked, and a fixed height chosen when the card
opens. It shows one candidate at a time:

- **Header.** A capsule with `tray.and.arrow.up` and `Route out of <inbox>`. A
  `CapturePickerFilterField` with the placeholder
  `Where does this go? Filter areas and open projects`. A count, `N destinations` or
  `k of N`.
- **Context line.** `<text> · <previous_status_name> → <status_name>`, plus
  `Task k of M` when there are several. It turns into a red refusal message after
  `inbox_route_refused`.
- **Rows.** bob's destinations, filtered locally with `FuzzyMatcher` over label, route,
  and relative path. Each row has:
  - a kind icon: area `square.stack`, project `folder`
  - the label
  - the relative path and parent as secondary text
  - a `wip`/`waiting` badge
  - a lock `Blocked` chip when `blocked` is set
- **Empty state.** `No matching destinations`.
- **Initial highlight.** The destination answered for the previous candidate in this
  card. Otherwise, the last destination this Mac routed to, kept in UserDefaults per
  vault directory. Otherwise, the first row.
- **Keys.**

  | Key                            | Effect                                                                                                                                                                  |
  | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `↩`                           | Answer "move to the highlighted destination", then go to the next candidate, or submit after the last                                                                   |
  | `⌃↩`                          | The same, with open-after-capture intent                                                                                                                                |
  | `⇧↩`                          | Keep this candidate in its inbox, then go to the next candidate or submit                                                                                               |
  | `esc` / `⌃[`                   | Go back one candidate. From the first, return to the pull-in card with its marks intact when one preceded, else to the editor with the draft intact. Nothing is written |
  | `↑↓` / `⌃P ⌃N`                 | Move the highlight                                                                                                                                                      |
  | key repeat on `↩`, `⌃T`, `⌃\` | Consumed, as on the pull-in card                                                                                                                                        |

- **Footer.**
  - Buttons: `Keep in inbox ⇧↩`, and `Move ↩`, which is disabled with no highlighted
    row.
  - Hints: `⌃N ⌃P Move · esc Back · ⌃↩ Move & Open`.
  - It has a `ViewThatFits` fallback like the pull-in footer.

**Submit.** After the last candidate, the app dispatches one `bob capture` with:

- an `--inbox-route KEY=<relative_path>` or `KEY=` answer for every candidate
- the preceding `--pull-in` selection, if any
- the same priority-roll seed
- the same override rule
- `--require-inbox-route-choice`, still set

**Responses.**

- `inbox_route_refused` reopens the card on the refused candidate. Other answers are
  kept and the message is shown.
- `inbox_route_choice_required`, sent because new candidates appeared, reopens the card.
  Answers are kept for keys that are still present.
- `inbox_route_stale` closes the card, keeps the draft, and says
  `Inbox tasks changed since the preview; press Return to choose again`.
- Success shows the usual notice, plus `· moved to <label>`, `· N tasks routed`, or
  `· kept in <inbox>` built from `inbox_route`.

The card is invalidated by the same events that invalidate the pull-in card: a draft
edit, a process-client swap, a new analysis, or dismissal.

### Out of scope

- Obsidian and its plugins, `bob task move`'s CLI behavior, and the review walk.
- Prompts for gestures that do not change status.
- A batch "send all to one destination" key.

### Rejected alternatives

- **The app runs `bob task move` after `bob capture`.** That is two writes for one
  draft, with partial-failure states. It breaks the one-call rule.
- **Route answers as capture grammar** (a draft marker). The successor-offer decision
  rejected selection grammar: the UI owns the choice, and bob takes flags.
- **Move first, then act.** Every capture writer resolves its task in the source note,
  and the move engine already plans from any snapshot. Act-then-move changes no writer
  and matches Obsidian.
- **Moving through `capture-targets`.** It lists root notes only, marks only `mac_inbox`
  as `inbox`, and includes project statuses that the move refuses. The route list must
  come from the move rule.

## Phase route-detect: Detect inbox route candidates and refuse without a choice

Work in bob-cli.

1. **Destination lister.** In `src/native/task_move/`, add a crate-internal lister for
   the destination set defined in the contract. It reuses `classify_dest` and the
   `DestBlocked` Blocked test, so the list and the move can never disagree. Add unit
   tests for:
   - a nested area
   - a terminal project
   - an `Other` status
   - `done/` and `_templates/`
   - the exclusion of inbox notes
   - the `blocked` flag
2. **Detection module.** Add `src/native/capture/inbox_route.rs`. It does the post-plan
   comparison per staged file with `note_tasks::scan` on both texts. It pairs tasks by
   block ID, and otherwise by `block_diff::line_maps`, as `TaskBlockTracker::finish`
   already does. It also covers the inbox classification, the column-0 ancestor rule,
   keys (through the factored-out shared digest helper, with offer keys unchanged), `n`
   numbering, and the destination list, which is built only when candidates exist.
3. **Wiring.** Call the module from `plan_capture_batch_with_config_path` at the point
   the contract names. Store an `InboxRouteReport` on `PlannedCaptureBatch`. Emit
   `inbox_route` on every success. Print the human `inbox` rows and the hint (the `-i`
   hint text lands in this phase, and the flag in the next).
4. **Guard flag.** Add hidden `--require-inbox-route-choice` in `cli.rs` using the clap
   builder pattern of `require-pull-in-choice`, kept alphabetical, with
   `CaptureRequest.require_inbox_route_choice`. Register it in
   `capture_flag_guard::capture_spec`, where the parity test
   `guard_specs_match_clap_declarations` must pass. Add the refusal
   `CaptureError::inbox_route_choice_required`, exit 1, with `code` and the
   `inbox_route` payload through `extra`, checked after the pull-in refusal.
5. **Detection vectors.** Write them as an `IR` table in `docs/capture.md`, and cite
   each vector id in its Rust test. Use a sandbox vault with `inbox.md`, a
   `mac_inbox.md` and a `gkeep_inbox.md` area under it, a project under `mac_inbox`, and
   ordinary areas and projects. Cover:
   - `@mac_inbox:id` Ready→Next (candidate)
   - an already-Next link (none)
   - `^mac_inbox:id`
   - `@mac_inbox+id` Ensure Next
   - `@mac_inbox+id!` link (candidate) and unlink (none)
   - `@mac_inbox:id=x1`
   - `x` setting `[/]` on an inbox Task Link
   - `x!1` and `!mac_inbox:id` (completion)
   - `^mac_inbox:id &dev:x` and `Ask *mac_inbox:id` (Blocked)
   - a new Next task with `&mac_inbox:id` that promotes the inbox prerequisite
   - a Blocked inbox dependent recovering on `!dev:x`
   - a `gkeep_inbox.md` task
   - a task in the project under `mac_inbox` (none)
   - a new `mac_inbox` task (none)
   - a nested changed task (candidate is the column-0 parent)
   - an id-less key
   - `--require-inbox-route-choice` refusal JSON with no write
   - the pull-in refusal winning when both apply
   - dry-run JSON equal to real-run JSON
6. **Docs.** In `docs/capture.md`, add a "Routing inbox tasks out" section with a
   Contents entry, place it next to the successor-offer material, and add the
   `inbox_route` fields to "Input, stdin, and JSON output". Also add a
   Grammar-at-a-glance row only if the docs check requires one (the feature is a flag,
   not a marker). Read the `docs.md` and `cli_rules.md` reference memory first.
7. **Checks.** Run `just docs-gen`, `just docs-check`, and `just check`.

## Phase route-apply: Apply inbox route answers inside the capture batch

Work in bob-cli.

1. **`task_move` additions.** Add the two crate-internal additions: the closed-selection
   option (under `route_closes = route`) and the exact-line selector. Add tests proving
   that `bob task move` CLI refusals and output are unchanged.
2. **The flag.** Add public `-i, --inbox-route ANSWER`, repeatable
   (`ArgAction::Append`), with excellent `--help` text and alphabetical placement. Add
   `i` to `short_values` in `capture_flag_guard::capture_spec`. Parse answers into
   `CaptureRequest.inbox_routes` following the contract: `N` or `KEY`, `=NOTE` or `=`,
   `keep`, and conflict errors.
3. **Apply stage.** Implement the stage described under "Writes": snapshot overlay,
   per-group `plan_task_move`, staging through the planner, the freshness stamp, and the
   recorded `moved`, `repaired_files`, `renamed_block_ids`, and `needs_reconcile`.
   Implement the `inbox_route_stale` and `inbox_route_refused` refusals with their
   payloads. Make `task_blocks`, `pomodoro_blocks`, and `open_notes` follow routed
   tasks.
4. **Human output.** Print the `routed` and `kept` rows, the dry-run `would route`
   wording, and the reconcile hint.
5. **Vectors.** Extend the `IR` table and its tests:
   - a linked Ready→Next inbox task moved to an area, with its new Task Link repaired in
     the daily note and `[fresh:: today]` stamped (clearing `[unblocked::]` and
     `[keeps::]`)
   - a completed task moved with no stamp
   - an id-less recovered dependent moved by line
   - a block-ID collision rename
   - two candidates moved to two destinations in one bulk draft
   - `keep` and `N=`
   - a conflicting duplicate answer
   - a stale key
   - refusals for an inbox destination, a project without `## Tasks`, and a Blocked
     `^prj` for a Next task
   - `--pull-in` plus `--inbox-route` together
   - dry-run equal to real-run with the same flags
   - a concurrent edit to a link-repaired file aborting the whole batch byte-identically
   - `open_notes` landing on the destination line
6. **Docs.**
   - Finish the `docs/capture.md` section: the flag, answers, refusals, the write order,
     and output.
   - Add the option to the "Command-line options" list.
   - Update `docs/projects.md` "What does not route": capture now routes through
     `--inbox-route` and Bob Mac Capture's route card, with a link. Keep the Obsidian
     rules unchanged.
   - Add "each open task an `--inbox-route` answer moves" to the `bob capture` row of
     `docs/freshness.md` §5.
   - Add one sentence to `docs/task-move.md` noting that capture routing reuses the
     engine.
7. **Checks.** Run `just docs-gen`, `just docs-check`, and `just check`.

## Phase mac-route-client: Decode inbox routes and send route answers in CaptureCore

Work in the linked `bob-mac-capture` repo. CaptureCore is checkable on Linux with
`just portable-check`.

1. **Models.** In `Sources/CaptureCore/CaptureModels.swift`, add `CaptureInboxRoute`,
   `CaptureInboxRouteCandidate` (with answer, outcome, and moved), and
   `CaptureInboxRouteDestination`.
   - Decode them on `CaptureCommandSuccess` (`inbox_route`) and on
     `CaptureCommandFailure` (`inbox_route`, `refused`).
   - Recognize the codes `inbox_route_choice_required`, `inbox_route_refused`, and
     `inbox_route_stale`.
   - Every field uses `decodeIfPresent`, and the payload uses `try?`, so a malformed
     value degrades to nil.
   - Keys stay opaque.
2. **Process client.** `BobProcessClient.capture(...)` gains:
   - inbox-route answers, emitted as repeated `--inbox-route KEY=<relative_path>` or
     `KEY=` before `--`
   - `requireInboxRouteChoice`
   - an `inboxRouteCapabilityAdvertised` gate. Answers without the capability throw
     `BobClientError.unsupportedOption("--inbox-route")`.

   Add the per-identity capability memory described in the contract. Add
   `--require-inbox-route-choice` to `unsupportedGuardFlag`'s retry list.

3. **Choice state.** Add a pure route-choice state type and its presentation helpers in
   CaptureCore. It holds:
   - the candidates, the destinations, and the cursor
   - answers by key
   - filter text and fuzzy filtering
   - the initial-highlight rule (previous answer, then the last-used destination passed
     in, then the first row)
   - back and forward
   - `answersForSubmit`
   - reopening at a refused key with its message
   - re-seeding from a new candidate set while keeping answers for surviving keys

   The presentation helpers cover the context line, row text, the preview summary text,
   and the success-notice suffix.

4. **Fixtures.** Generate `Tests/Fixtures/inbox-route-*.json` from a real `bob capture`
   built from bob-cli master after route-apply, run against a sandbox vault. Record the
   recipe in the test file header, as `CaptureTaskCompletePresentationTests.swift` does.
   Generate:
   - a preview with one candidate
   - a preview with two candidates
   - the choice-required refusal
   - the refused refusal
   - the stale error
   - a moved success
   - a kept success
   - a pull-in plus route draft

   Add matching knobs or draft matches to `Tests/Fixtures/fake-bob`, including `capture`
   record lines for the new argv.

5. **Tests.** Add tests in `Tests/CaptureCoreTests/`:
   - decoding, including an old bob with no `inbox_route` and malformed payloads
   - argv placement and the capability gate
   - guard retry and caching per identity
   - no retry for structured refusals
   - the state type's navigation, filtering, highlight, refusal reopen, and re-seed
     rules
6. **Docs.** In `docs/protocol.md`, cover:
   - the capability requirement
   - the hidden guard under "Timeouts and guards". While there, correct the existing
     claim that `--require-pull-in-choice` is retried; it has been gated on preview
     support since `8400bd3`.
   - the optional `inbox_route` fields
   - a Compatibility row: Inbox routing | Route card before submit | Captures without
     asking | Upgrade bob
7. **Checks.** Run `just portable-check` and `just docs-check`.

## Phase mac-route-card: Route card in the Bob Mac Capture panel

Work in the linked `bob-mac-capture` repo, plus the bob-cli memory notes.

1. **Model.** In `Sources/BobMacCapture/CapturePanelModel.swift`, add the route choice
   to the submit state machine next to the pull-in choice:
   - Open it locally from the matching preview snapshot (extend
     `CapturePullInPreviewSnapshot`, or add a sibling, with `inboxRoute`), or from the
     `inbox_route_choice_required` refusal in `completeSubmit`.
   - Sequence it after the pull-in choice. Stash the pull-in state so Esc can return to
     it.
   - Implement confirm (move), keep, and back.
   - Resubmit through `dispatchCapture` with the answers, the pull-in selection, the
     seed, and the override rule.
   - Handle `inbox_route_refused` and `inbox_route_stale`, and the invalidation events.
   - Count it in `inlinePromptVisible`, lock the editor input, and save the last-used
     destination per vault directory.
   - Add the success-notice suffix.
2. **Views.** Add `CaptureInboxRouteChoiceView.swift` with the card, its height policy,
   and its footer, per the contract. Host it in `CapturePanelView.auxiliaryRegion`, and
   add the footer swap and the preview summary row.
   - Keys go in `CaptureKeyCommandRouter`: a branch next to the pull-in branch, with the
     key table from the contract.
   - Wire the controller's perform cases and focus targets, including the filter field
     identifier and focus repair.
   - Add VoiceOver announcements on open, highlight, candidate change, and refusal.
3. **Tests.**
   - Model tests with fake-bob that assert the argv: the preview opens the card and Esc
     writes nothing; `↩` sends `--inbox-route KEY=dest --require-inbox-route-choice`;
     `⇧↩` sends `KEY=`; two candidates are answered in sequence; pull-in comes before
     the route and Esc returns to the pull-in card; the refusal path opens the card;
     `refused` reopens at the right candidate; `stale` shows its message; a draft edit
     invalidates the card; an old bob never sees either flag; the same seed is reused.
   - Router key-contract tests.
   - Height-policy tests.
   - Design render fixtures in light and dark at both panel widths, in the style of
     `CapturePullInChoiceDesignTests`.
4. **Docs.**
   - In `docs/capture-previews.md`, add an "Inbox routing" section next to "Successor
     offers".
   - In `docs/keyboard.md`, add an "Inbox routing" section and the consumed-key note.
   - In `docs/develop/architecture.md`, add the route card to the modal owners.
   - Add render media through `just docs-media` if the page shows the card.
   - Run `just docs-check`.
5. **Compile.** Compile per `AGENTS.md`; never skip compiling. Run
   `just portable-check`. Then rsync a scratch copy without `.build` or `.git` to the
   mac, run `bundle.sh` and `just typecheck-tests`, and delete the copy. Never use
   Bryan's checkout or `just install` there. If the mac is offline, say in the final
   report that the macOS targets are unverified.
6. **Memory.** Apply the accepted memory decisions in bob-cli's `sase/memory/`, then run
   `sase memory init`. Use the `/sase_memory_write` skill first.

> [!decision] memory_inbox_file In `sase/memory/glossary/inbox-file.md`, replace the
> sentence "Ctrl+Shift+M also moves it, following it to the destination, and a closed
> task leaves through `bob task archive`." with two sentences. First: Ctrl+Shift+M also
> moves it, following it to the destination. Second: a capture that changes an existing
> inbox task's status asks the same question in Bob Mac Capture's route card, and bob
> moves the task in that capture's batch (`bob capture -i/--inbox-route`); otherwise a
> closed task leaves through `bob task archive`. Under `route_closes = skip`, add
> "non-closing" before "capture". Change nothing else. Keep the strand one paragraph.

> [!decision] memory_decision_record Add a new strand to the decisions web,
> `decisions:capture-asks-where-inbox-tasks-go`. Follow an existing record's frontmatter
> and section shape (read one with `sase memory read decisions:<keyword>`). It covers:
>
> - **Applies to:** bob-cli and Bob Mac Capture.
> - **Claim:** bob detects the candidates, lists destinations, and moves them in the
>   capture batch, while the Mac only asks.
> - **Why:** the rejected alternatives from this plan.
> - **Cost:** the extra full-vault scan on routed submits.
> - **Reopens when:** a second client cannot keep the answer contract.
> - **Evidence:** this plan and the phase commits.
>
> Link `[[decisions:mac-capture-is-a-thin-client]]` and
> `[[decisions:unblocked-tasks-are-offered]]`.
