---
tier: epic
title:
  Finish `!note:block-id` completion — one tree traversal, contract-exact output, and
  Mac picker polish
goal: "Close the gaps the bob-cli-4i landing review found between the shipped
  `!note:block-id` completion and its approved plan (plan:202610/bang_task_complete.md).
  The `=x` close and `!` share one embedded-tree traversal. `!` only differs from `=x`
  at the root. The Complete picker never offers a task capture refuses. `bob capture`
  prints the contract's ledger lines and reports the entry names behind them. Docs and
  tests match the shipped behavior. Bob Mac Capture shows the filtered Today and All
  open tasks headers, the full detail strip, the Done capsule, and Bob's clean task text
  and ledger names.

  "
parent_bead: bob-cli-4i
phases:
  - id: close_unify
    title: Route the `=x` embedded-tree close through `complete_task_tree`
    depends_on: []
    size: medium
    description: "close_unify: in bob-cli, make `ClosePlanner::apply_embedded_tree`
      delegate its traversal to `task_complete::complete_task_tree` with `CloseLink` so
      one traversal remains. Keep every close test unchanged and `=x!N` output
      byte-identical. Make `Explicit` differ from `CloseLink` only at the root and in
      reporting recurring descendants. Stop reporting Canceled or Done descendants as
      left open. Remove unused `task_complete` re-exports.

      "
  - id: picker_parse
    title: Picker status filter, today-section sinking, and parse/claim consistency
    depends_on: []
    size: small
    description: "picker_parse: in bob-cli, limit the completable-task catalog to `
      `/`?`/`*`/`/` tasks, sink hidden and recurring rows inside each today entry,
      mirror the `task_complete` object at the top level of single-item capture-parse
      JSON, and claim `!` before dependency scanning in execution. Fill the
      grammar/picker test gaps. Fix the stale capture-parse and capture-complete
      reference lists in docs/capture.md.

      "
  - id: ledger_output
    title:
      Contract-exact ledger lines, clean task text, one vault walk, and execute tests
    depends_on:
      - close_unify
      - picker_parse
    size: medium
    description: "ledger_output: in bob-cli, report `struck_in` and `dropped` ledger
      entries plus a clean `task_complete.text`, print the contract's ledger lines, use
      the configured global filter, build the recovery vault walk once per batch, drop
      the dead preimage rechecks, fix new clippy warnings in the epic's files, fill the
      execute CLI test gaps with exact human output, and rewrite the docs/capture.md `!`
      execution docs.

      "
  - id: mac_followups
    title:
      Mac Complete picker headers, detail strip, Done capsule, and Bob's text and ledger
      names
    depends_on:
      - ledger_output
      - picker_parse
    size: medium
    description: "mac_followups: in bob-mac-capture, show the filtered Today / All open
      tasks headers, use `TaskCompletePickerIndex.actionLine` in the detail strip, add
      the Done capsule, render Bob's `task_complete.text` and `struck_in`/`dropped`
      ledger names, show Bob's locator in the ID prompt, refresh real-bob fixtures, fill
      the panel test gaps, then commit and get macOS CI green.

      "
proposed_by: bbugyi200.apollo.bob-cli-4i.land
create_time: 2026-10-05 18:10:30
status: wip
---

- **PROMPT:**
  [prompts/202610/bang_task_complete_finish.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bang_task_complete_finish.md)
- **PARENT:**
  [202610/bang_task_complete.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bang_task_complete.md)

# Context

Epic bob-cli-4i shipped whole-item `!note:block-id` completion in bob-cli (commits
7c8d854, 1b6f8bc, 40e561f, fbc4f43) and Bob Mac Capture (c7003c3, 547dbc8, 57220a2,
6940c93, 6cc8552). The parent plan is plan:202610/bang_task_complete.md. Its Design
section stays the contract for everything below unless this plan says otherwise.

The land review confirmed that the feature works end to end. `cargo test` is green apart
from the unrelated bob-cli-4j kinds test. `just lint` is red only on the pre-existing
`|| true` deny that bob-cli-28 owns. The bob-mac-capture macOS CI run for 6cc8552 is
green. The review also found these deviations from the approved plan, all introduced by
bob-cli-4i:

1. **Two traversals.** `ClosePlanner::apply_embedded_tree`
   (`src/native/capture_pomodoro_close/linked_tasks.rs`) still runs its own recursion,
   with hardcoded `25`/`250` caps. It shares only the gate predicates and line helpers
   with `task_complete::complete_task_tree` (`src/native/task_complete/tree.rs`). The
   plan said to rewire the close through the engine.
2. **`Explicit` descends through Blocked descendants.** `TreeWork::close_node` passes
   `self.policy` to `close_traversal_gate` for descendants too, so `!R` descends through
   a Blocked `[?]` or `X` descendant and closes that descendant's open children. The
   plan says only the root may differ from the existing close rules.
3. **Canceled descendants are reported as left open.** A Canceled `[-]` subtask lands in
   `subtasks_left_open` with reason `unknown_status`, and the human output prints
   `left Canceled [-] …`.
4. **The picker offers tasks that capture refuses.**
   `src/native/capture_completable_tasks.rs` filters on `task.open`. Custom statuses
   such as `[>]` default to an open type, so they are offered, and execution then
   refuses them.
5. **Human ledger lines miss the contract.** `bob capture` prints a bare
   `Task Link struck` and `dropped N duplicate(s) already linked at the destination`
   (`src/native/capture/output.rs`). The contract wants `Task Link struck in CAPTURE`,
   `Task Link struck in PLAN (completed)`, and
   `Task Link already in PLAN; dropped the SASE copy`. The JSON has no entry names for
   in-place strikes or dedupes, so Bob Mac Capture cannot name them either.
6. **`#task` is hardcoded.** `task_complete_display_text` strips a literal `#task`
   instead of the configured global filter. Bob Mac Capture shows `#task` in the preview
   and notification (`Completed: #task Fix …`) because the JSON has no clean task text.
7. **Recovery walks the whole vault once per `!` item.** `staged_snapshot_for_recovery`
   runs per item, although the plan said to share one walk per batch. Two "changed while
   planning" rechecks in `src/native/capture/task_complete.rs` compare a value with
   itself and can never fire.
8. **Execute tests and docs are thin.**
   - There is no test of a batch with two completions plus `=x`.
   - The forced-flag, ambiguous-note, and duplicate-ID tests never assert an exit code
     or an untouched vault.
   - Human output is checked with `contains`.
   - The docs/capture.md worked example has no dry-run JSON excerpt, no after-ledger,
     and no ledger line.
   - The docs/capture.md error list gives no messages, and it says refusals exit 1,
     although usage errors exit 2.
9. **Stale reference lists in docs/capture.md** omit `task_complete`. This covers the
   capture-parse mode, needs, and span-kind lists, the capture-complete `context` list,
   the "`query` is present only for …" sentence, and the picker-descriptor sentence that
   says the descriptor is omitted elsewhere. The mode and needs lists also lack the
   earlier `task_dependency` values.
10. **Parse inconsistencies.**
    - Single-item capture-parse drafts carry no `task_complete` object, because it lives
      only on `items[]`.
    - Execution runs `scan_item_dependencies` before the `!` claim, so `!sase:x &foo`
      gets a dependency error from `bob capture` but `invalid_task_complete` from the
      editor.
    - Hidden and recurring rows only sink on exact ties inside today sections
      (`canonical_key`).
11. **Bob Mac Capture gaps.**
    - The filtered `Today` / `All open tasks` sections use kind `.matches`, whose header
      the view suppresses (`CapturePickerView.swift`).
    - The live detail strip says `Inserts !x — completes it`. It does not call the
      tested `TaskCompletePickerIndex.actionLine`, so it lacks the `[*] → [x]`
      transition and the Blocked variant.
    - Completed Pomodoro sections have no `Done` capsule.
    - The Add block ID prompt previews `!sase.md:id` instead of Bob's locator.
    - Panel tests do not cover the ID prompt splicing `complete_replacement`, typing `!`
      to auto-open the picker, or Shift-Return reopening the picker. The refusal fixture
      is unused, and the worked-ordering test has only one worked Pomodoro.

All contracts stay additive: `schema_version` 1, and fields and enum values are only
added. Decision `mac-capture-is-a-thin-client` applies: Bob owns every string the app
inserts and every fact it previews.

# close_unify

Work in bob-cli.

- **One traversal.** `ClosePlanner::apply_embedded_tree` must stop recursing on its own
  and drive the embedded-tree close through
  `task_complete::complete_task_tree(…, RootPolicy::CloseLink)`, or through one shared
  traversal core that both call. Choose the mechanism. For example, add an observer or
  hook parameter so the close can run its existing bookkeeping (`register_reference`,
  task rows, `refresh_task_row`, and warnings keyed to the ledger line) at each visited
  node. Or have the engine return an ordered visit log that the close replays.
  - Hard constraints:
    - every existing close test passes **unchanged**;
    - `=x!N` human output, JSON, warnings text, and vault writes stay byte-identical;
    - the `25`/`250` caps live only in `tree.rs` (`MAX_TREE_DEPTH`/`MAX_TREE_TARGETS`);
    - no second copy of the gate, cap, or Depends-On rules remains.
  - Warning wording differs today. The close says `below line {ledger_line}` and the
    engine says `below line {root line}`, so parameterize the label rather than changing
    either output.
- **`Explicit` differs only at the root.**
  - Under both policies, descendants use the `CloseLink` traversal gate: never descend
    through a Blocked `[?]` or `X` descendant.
  - A Blocked descendant is reported as left open with reason `blocked`, and its own
    embedded children are not visited.
  - `Explicit` keeps exactly two differences:
    - a Blocked root closes;
    - recurring lines are left open with reason `recurring` instead of closing. `=x`
      keeps closing recurring descendants, as it does today.
- **Left-open reporting.** A descendant is reported in `left_open` only when its status
  type is open but not closable: Blocked, recurring under `Explicit`, a custom open
  status (`unknown_status`), or `cap`. Done-type and Canceled-type descendants (resolve
  the type through the Tasks status settings, so `-` and any configured CANCELLED symbol
  count) are neither closed nor reported.
- **Re-exports.** Remove every `task_complete` re-export that is unused after the
  rewire, so the module adds no `unused_imports` warnings. Keep everything `pub(crate)`.
- **Tests.**
  - `src/native/task_complete/tests/tree_tests.rs`:
    - Under `Explicit`, a Blocked descendant's open child stays open, and the Blocked
      descendant is reported `blocked`.
    - A Canceled descendant is not reported.
    - Under `Explicit`, an `X` descendant is not descended.
    - Update the existing Canceled `unknown_status` assertion.
  - `tests/cli/capture/task_complete.rs`: extend the subtasks case with a Blocked
    subtask that embeds an open task, and assert that the open task stays `[ ]`.
  - The full close suite (`tests/cli/capture/*close*`,
    `src/native/capture_pomodoro_close/*tests*`) passes unchanged.
- **Docs.** In docs/capture.md's `!` execution semantics, add one sentence: a Blocked
  descendant and everything below it stay open. Add nothing else; `ledger_output`
  rewrites that section.
- **Verification.**
  - `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test`
    pass.
  - Failures already tracked elsewhere are expected: the `|| true` clippy deny in
    `tests/cli/capture/pomodoro_name.rs` (bob-cli-28) and
    `completion::kinds::every_value_arg_has_a_decision` (bob-cli-4j). Record them, and
    do not fix them here.
  - Confirm that clippy reports no new warnings in `src/native/task_complete/` or
    `linked_tasks.rs`.

# picker_parse

Work in bob-cli.

- **Status filter.** In `src/native/capture_completable_tasks.rs`, offer only tasks
  whose status symbol is ` `, `?`, `*`, or `/`, the same set `bob capture` accepts
  (`src/native/capture/task_complete.rs`). Share one predicate, for example
  `task_complete::is_completable_status(symbol)`, between the catalog and execution.
  - Catalog unit test: a `[>]` custom-open task and a Canceled task are absent.
  - CLI test in `tests/cli/capture/complete_task_complete.rs`: a `[>]` task does not
    appear for `!`.
- **Today sinking.** In `canonical_key`, put `recurring` and then `hidden` right after
  the today role and entry keys, before the link line. Within each today Pomodoro entry
  (and within the `noted` group), visible rows then come first and recurring rows come
  last. Add a catalog unit test covering a recurring today row linked before a normal
  row in the same entry.
- **Single-item capture-parse.** When the draft has one item and that item is
  `task_complete`, emit the same `task_complete` object at the top level of the
  capture-parse result, mirroring how top-level `dependencies` is emitted. Use
  `skip_serializing_if`, so other drafts are unchanged. Test it in
  `tests/cli/capture/task_complete_parse.rs` with a lone `!sase:fix-flaky` and a lone
  quoted `!"Shopping List":milk`.
- **Claim order.** In `parse_capture_item` (`src/native/capture_language/item.rs`),
  evaluate the `!` claim before `scan_item_dependencies`. A claimed item then gets the
  `!` teaching error rather than a dependency error, matching the editor. Add a grammar
  test showing that `!sase:x &foo` is claimed invalid with `remove `&foo``.
- **Test gaps.**
  - Add an `@@` test: a `@@` declaration followed by a `!sase:x` item leaves the `!`
    item a `TaskComplete` with no inherited route or tags (execution parse), and the
    editor parse reports `task_complete` mode.
  - Add editor-mode tests for the prose rows: `! foo`, `![[sase#^x]]`, `![alt](url)`,
    `!!!`, `!fix` plus a child line, and `!"Shopping List"` (a query).
  - Make the claimed-invalid refusal CLI test assert the exact full message.
  - In the shell-completion test, assert that today rows come first.
- **Docs (docs/capture.md reference lists only).**
  - Add `task_complete` to the capture-parse mode list, the needs list, and the
    span-kind list (`task_complete_sigil`, `task_complete_note`,
    `task_complete_block_id`).
  - Add the missing `task_dependency` mode and need values introduced by the earlier `&`
    work.
  - Add `task_complete` to the capture-complete `context` list.
  - Correct the "`query` is present only for …" sentence and the picker-descriptor
    sentence, so both include `task_complete` and its `kind`.
  - Document the top-level single-item `task_complete` object.
- **Verification.** Same commands and expected failures as `close_unify`.

# ledger_output

Work in bob-cli. This phase builds on `close_unify`'s left-open semantics and edits the
same docs/capture.md file as `picker_parse`.

- **Engine data.**
  - Extend `task_complete::LedgerRetirement` (`src/native/task_complete/retirement.rs`)
    with the entries where a completed link was struck in place (line, context, open),
    deduplicated per entry.
  - Record each deduplicated bullet's source and destination entries. The existing
    `deduplicated` list may already carry them; extend it if not.
  - Reconcile output stays byte-identical.
- **Additive JSON** (`src/native/capture/output.rs`, `TaskCompleteLedgerJson` and the
  `task_complete` object):
  - `task_complete.text`: the root task's clean display text, the same string the human
    output prints. Strip the **configured** global filter, inline fields, and the
    trailing block ID. Present on `completed` and `already_done`.
  - `ledger.struck_in`: `[{"line", "name", "status"}]`, one per entry with an in-place
    strike. `status` takes the existing endpoint values (`running`, `queued`,
    `completed`).
  - `ledger.dropped`:
    `[{"from": {"line","name","status"}, "to": {"line","name","status"}}]`, one per
    deduplicated bullet.
  - Both arrays are always present (possibly empty) whenever `ledger` is present.
  - `struck` and `deduplicated` keep their count meaning.
  - When an entry has no name, `name` is `""` and the human output uses `line N`.
- **Human ledger line.** Build the `  ledger  …` parts in this order, joined with `·`,
  and then end with the day file:
  1. each `struck_in` entry: `Task Link struck in <NAME>`, with ` (completed)` appended
     when the entry is completed;
  2. each move: `Task Link moved <FROM> → <TO> (struck)`;
  3. each dropped copy: `Task Link already in <TO>; dropped the <FROM> copy`;
  4. each removed placeholder: `removed empty <NAME>`.
- **Global filter.** `task_complete_display_text` uses
  `note_tasks::read_settings(bob_dir).global_filter` (or the settings already in scope),
  never a literal `#task`.
- **Empty block IDs.** When an unblocked dependent or left-open subtask has no block ID,
  the human locator prints only the note path with no trailing ` ^`. JSON keeps
  `block_id: ""`.
- **One vault walk per batch.**
  - Build the on-disk markdown snapshot that recovery needs at most once per
    `bob capture` batch. Build it lazily, only when the batch has a `!` item, and keep
    it on the batch context next to `DependencyContext`.
  - Overlay the current staged files per item.
  - Behavior is unchanged: recovery still sees earlier items' staged edits.
- **Dead rechecks.** Remove the two "changed while planning" comparisons in
  `src/native/capture/task_complete.rs` that compare freshly re-read staged text with
  itself. Keep the error string only if a real check still uses it. The batch writer's
  commit-time preimage check remains the guard against external edits; confirm that it
  exists and cite it in a code comment.
- **Clippy.** Fix the warnings that bob-cli-4i introduced in its own files:
  - `useless_format` in `src/native/capture/output.rs`;
  - `result_large_err` in `src/native/capture/task_complete.rs`, or justify it with a
    local `#[allow]` that matches the module's existing pattern;
  - any other warning that `cargo clippy --all-targets --all-features` attributes to
    `src/native/capture/task_complete.rs` or `src/native/task_complete/`.

  Do not touch pre-existing warnings elsewhere (bob-cli-v).

- **CLI tests** (`tests/cli/capture/task_complete.rs`):
  - One draft with two `!` completions plus a `=x` close item, separated by blank lines:
    everything commits, and each item's JSON is asserted.
  - Each forced flag (`--route`, `--section`, `--task`, `--task-section`, `--clip`)
    exits 2 with the `invalid_task_complete` message and leaves the vault
    byte-identical.
  - The ambiguous-note and duplicate-ID refusals assert their exit code, their exact
    message, and an untouched vault.
  - **Exact** human output (with `NO_COLOR=1`), compared as full stdout and not with
    `contains`, for real and dry-run runs of the following scenarios:
    - a running-session strike (`Task Link struck in CAPTURE`);
    - a strike under a completed entry (`… in PLAN (completed)`);
    - a placeholder move with removal;
    - a dedupe (`Task Link already in PLAN; dropped the SASE copy`);
    - subtasks plus left open;
    - an unblocked dependent;
    - already done.
  - JSON assertions for `text`, `struck_in`, and `dropped` in the strike, dedupe, and
    move cases.
  - A vault whose Tasks global filter is not `#task` gets clean `text` and human output.
  - Keep `capture_json_dry_run_matches_real` green.
- **Docs (docs/capture.md `!` execution section and result contract).**
  - Rewrite the worked example. Show the note and ledger before and after, a dry-run
    JSON excerpt that includes `task_complete` with `text`, `ledger.struck_in`, and
    `ledger.dropped`, and the exact human output including its `ledger` line.
  - List every refusal with its exact message and exit code. Usage errors exit 2; check
    every other refusal's real exit code against the code and state it.
  - Document the new fields and the ledger-line variants.
  - Remove the incorrect "exit 1" claim.
- **Verification.** Same commands and expected failures as `close_unify`. Then run a
  sandbox check against a copy of the fixture vault, covering a strike and a dedupe, and
  paste the human output into this phase's bead notes.

# mac_followups

Work in bob-mac-capture. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<reason>"`). If the linked checkout is unavailable
on the host, use `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"`. Use only
the printed path, and read its `AGENTS.md` first. bob-cli master must already contain
`ledger_output`. Build `bob` from bob-cli master (`cargo build`, then use
`target/debug/bob` or `just install`) for fixtures.

- **Filtered headers.** The filtered Complete picker must show its `Today` and
  `All open tasks` headers. Give those sections a kind whose header renders, for example
  a new task-complete filtered kind or a per-section flag, without changing how other
  pickers' `.matches` sections render
  (`Sources/CaptureCore/TaskCompletePickerPresentation.swift`,
  `Sources/BobMacCapture/CapturePickerView.swift`).
- **Detail strip.** For `.taskComplete`, the detail strip renders
  `TaskCompletePickerIndex.actionLine(for:)`:
  - `Inserts !sase:fix-flaky — completes it [*] → [x]`;
  - the Blocked variant;
  - the disabled reason.

  Do not keep a second inline copy of that wording in the view.

- **Done capsule.** Completed Pomodoro section headers show a `Done` capsule next to 🍅
  and the time range, per the parent plan.
- **Bob's task text.**
  - Decode `task_complete.text` (`decodeIfPresent`).
  - `CaptureTaskCompletePresentation` uses it for `previewText`, `transitionText`,
    `notificationTitle`, and the accessibility summary when it is non-empty.
  - Otherwise it falls back to the current `taskPreviewText`, so older bob still works.
  - Never strip tags in Swift.
- **Bob's ledger names.**
  - Decode `ledger.struck_in` and `ledger.dropped` (`decodeIfPresent`, defaulting to
    empty).
  - Build `ledgerText` from them, tense-aware and mirroring bob's human line:
    - `Strikes its Task Link in CAPTURE` / `Struck its Task Link in CAPTURE`, with
      ` (completed)` for completed entries;
    - `Moves its Task Link SASE → CAPTURE, struck` / `Moved …`;
    - `Task Link already in PLAN; drops the SASE copy` / `… dropped the SASE copy`;
    - `removes empty SASE` / `removed empty SASE`.

    Join the parts with `·`.

  - Keep the count-based wording as the fallback when the arrays are absent.
  - Omit `^` from unblocked and left-open locators when `block_id` is empty.

- **ID prompt locator.** The Add block ID prompt for `.taskComplete` previews
  `!<locator>:<id>` using Bob's candidate `locator` (for example `sase`), never the file
  name. After assignment it still splices Bob's `complete_replacement` verbatim.
- **Fixtures.**
  - Regenerate the task-complete real-bob fixtures with the new bob: running strike,
    placeholder move plus removal, dedupe, subtasks plus left open, unblocked, already
    done, two-item batch, and refusal.
  - Update the generating commands in the test header.
  - Add a fixture for a strike under a completed entry.
- **Tests.**
  - Update `CaptureModelTests` for the new fields, both absent and present.
  - In `CaptureTaskCompletePresentationTests`, cover:
    - every ledger variant in both tenses;
    - clean text from Bob;
    - the fallback with no `text`;
    - empty-block-ID locators.
  - In `TaskCompletePickerPresentationTests`, cover:
    - filtered header visibility;
    - two worked Pomodoros ordered most recent first;
    - the Done capsule data.
  - Panel tests:
    - typing `!` into an empty draft auto-opens the Complete picker (through the
      fake-bob `capture-complete` route, not by installing the picker directly);
    - Shift-Return inserts, appends `\n\n!`, and reopens the picker with the first pick
      marked already selected;
    - the ID prompt splices Bob's `complete_replacement`;
    - the refusal fixture surfaces Bob's error.
  - Update the `CapturePickerDesignTests` render case for the headers and capsule.
- **README.** Update the Complete Picker and "Completing tasks with `!`" sections:
  - the filtered headers;
  - the detail strip;
  - the Done capsule;
  - the ledger wording.
- **Commit, then CI.** This plan explicitly instructs you to commit the bob-mac-capture
  changes with `/sase_git_commit` before waiting on CI. macOS CI only runs on the pushed
  commit, and agent hosts have no Apple Swift toolchain for the app target.
  - Use subject `fix(capture): finish the Complete picker and completion preview`. If
    `$SASE_BEAD_ID` is set, pass `-B keep`.
  - Wait with `/sase_monitor` on the `CI` workflow run for that SHA
    (`gh run list --commit <sha> --limit 1`, then `gh run watch <id>`), with a timeout
    of at least 20 minutes.
  - If the run is red, read `gh run view <id> --log-failed`, fix forward, commit, and
    watch again.
  - The phase is done only when the latest run for its commit is green.

# Non-goals

- No change to reconcile output, `=x!N` output, or the `!` grammar beyond the
  claim-order fix.
- No fixes for the pre-existing failures owned elsewhere: bob-cli-28 (`|| true` clippy
  deny), bob-cli-4j (kinds test), bob-cli-40 and bob-cli-2e (capture_pomodoros flake),
  bob-cli-v (old clippy warnings), and bob-cli-2l (enabling dedupe for reconcile).
- No new CLI subcommands or options, and no SASE memory changes.

# Acceptance

- `ClosePlanner` has no embedded-tree recursion of its own, and the close suite passes
  unchanged.
- `!R`, where R embeds a Blocked B that embeds an open C, leaves C open.
- A Canceled subtask is not reported as left open.
- `!` never lists a `[>]` task.
- `bob capture '!sase:fix-flaky'` prints `Task Link struck in CAPTURE`. A dedupe prints
  `Task Link already in PLAN; dropped the SASE copy`. JSON carries `text`,
  `ledger.struck_in`, and `ledger.dropped`.
- A batch with N `!` items walks the vault once.
- docs/capture.md lists `task_complete` in every capture-parse and capture-complete
  enumeration. Its worked example matches real output.
- Bob Mac Capture shows the filtered Today / All open tasks headers, the full detail
  strip, Done capsules, Bob's clean task text, and named ledger lines. macOS CI is green
  for the landed revision.
- bob-cli `cargo fmt --check` and `cargo test` pass, apart from bob-cli-4j.
  `cargo clippy` reports no deny other than bob-cli-28's and no new warnings in this
  epic's files.
