---
tier: tale
size: small
title: Finish and land the obsidian-gestures epic (bob-cli-on.10.1)
goal:
  Close the two gaps the bob-cli-on.10.1 land agent found — the cycler's transcluded
  tree close still checks checkbox lines without `#task`, and the requirement writer
  twin's insert lacks bob-cli's new `added` count — then ship both plugins and close
  bob-cli-on.10.1 and its parent phase bob-cli-on.10.
proposed_by: bbugyi200.athena.bob-cli-on.10.1.land
proposed_with: claude/opus
bead: bob-cli-on.10.1
status: done
---

- **PARENT:**
  [202611/obsidian_req_gestures.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/obsidian_req_gestures.md)
- **BEAD:**
  [bob-cli-on.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.10.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-on.10.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-on.10.1.land.md)
- **COMMITS:**
  - [f74c45e](https://github.com/bobs-org/bob-plugins/commit/f74c45e1612c6320a4ef9fccae890b5b82696c78)
    — fix(plugins): finish req gesture landing

# Plan: Finish and land the obsidian-gestures epic

## Context

You are finishing the landing of epic **bob-cli-on.10.1** (plan
`plan:202611/obsidian_req_gestures.md`, the `obsidian-gestures` phase `bob-cli-on.10` of
epic `bob-cli-on`). Its land agent verified the six phases and their commits:

- bob-plugins: f292fff, e5cb895, 28d76fd, d3b80d7, d2877fe, and 7b96dce.
- bob-cli: e5ee4a8d.

That verification covered:

- Every new test passes: RQ/RL/RW vectors 144/144, twin copies 7/7, cycler req guards,
  drop, toggle, and typing, nav guards 19/19, Block ID Prompt guard 4/4, SV verdicts
  14/14, and the e2e self-test.
- The fixtures are byte-identical to bob-cli HEAD.
- No hand-edited fragment exceeds 1000 lines.
- All three plugins dry-run `up to date`.
- `sase bead epic-symbols bob-cli-on.10.1` lists nothing.

All child `PROPOSED FOLLOW-UP:` notes are already triaged and recorded on
bob-cli-on.10.1 (new beads bob-cli-p8 and bob-cli-p9; +1 on bob-cli-or, bob-cli-io,
bob-cli-m1, and bob-cli-np; corroboration notes on the active epics bob-cli-on and
bob-cli-os). Do not re-triage them and do not create beads for them.

Two pieces of work remain. Both are in **bob-plugins**. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`), read its `AGENTS.md`, and work only at the
printed path. Edit `src/` fragments, never `main.js`. Keep every hand-edited fragment at
or below 1000 lines (`170-plugin-pomodoro.js` is 839 lines and
`198-task-reqs-writers-twin.js` is 454). Rust is the authority. Do not edit bob-cli's
`Cargo.toml` version or `CHANGELOG.md`.

## 1. The transcluded tree close never checks a line without `#task`

The epic plan's `cycler-guards-and-drop` phase required this rule (step 2, "Link and
transclusion targets"): the recursive transcluded tree close
(`completeResolvedTranscludedTaskTargetTree`) never rewrites a checkbox on a line
without the global filter. This matches bob-cli's reader-hardening fix in
`src/native/task_complete/tree.rs` `finish_close_node` (commit 532df825):

```rust
if !task_has_global_filter(&line, &self.settings) {
    return Ok(false);
}
```

`task_has_global_filter` is `line.split_whitespace().any(|t| t == "#task")`. Phase 3
shipped only the requirement-line refusal (in `replaceResolvedTranscludedTaskLine`). A
non-`#task`, non-req checkbox reached through a Pomodoro transclusion therefore still
gets checked. Example: a top-level `- [ ] plain item ^abc` in another note, embedded as
`![[other#^abc]]`. The cycler resolves transcluded targets with `getTaskStatusForLine`,
which matches any checkbox, and `rewriteTaskLineForTranscludedSource` has a deliberate
non-`#task` branch.

- In `plugins/task-status-cycler/src/170-plugin-pomodoro.js`, change
  `completeResolvedTranscludedTaskTargetTree`. Keep the descendant loop exactly as it
  is: bob-cli still walks a non-`#task` node's children. Guard only the node's own
  write, the `isTranscludedCompletionClosableStatus(resolvedTarget.taskStatus)` block
  that calls `replaceResolvedTranscludedTaskLine(..., "x")`. Skip that write when the
  target line does not carry the whole-token global filter.
  - Use `taskReqCarriesGlobalFilter(lineText, TASKS_GLOBAL_FILTER)` from the RQ twin
    (`195-task-reqs-twin.js`), which is the exact JS mirror of `task_has_global_filter`.
  - The line text is `resolvedTarget.taskStatus.lineText`; fall back to
    `getLineTextFromSourceText(resolvedTarget.sourceText, resolvedTarget.line)`.
  - Leave the return shape (`{ visited, changed, closed }`) unchanged: a skipped node
    adds nothing to `closed`.
- Do not change other transcluded paths. That includes the non-recursive transclusion
  toggles, Alt+[ / Alt+] on transclusions, struck-link reopen, and
  `rewriteTaskLineForTranscludedSource` itself. bob-cli's rule is scoped to the tree
  close.
- Tests: add cases to `scripts/test-task-status-cycler-req-guards.cjs`, next to
  "transcluded writes refuse req targets", or to the Pomodoro completion test if its
  harness fits better.
  - A tree close through a Pomodoro sub-bullet transclusion leaves a non-`#task`
    checkbox target byte-identical.
  - A `#task` descendant transcluded beneath that plain node still closes, which shows
    that descendants are still traversed.
  - A `#task` root still closes with its `[completion::]` field.
- Then run the existing transcluded suites:
  `test-task-status-cycler-pomodoro-completion`, `-ctrl-enter`, `-dependencies`,
  `-relations`, `-reference-lifecycle`, `-task-links-api`, `-restore`, `-blocks-guards`,
  and `-bank-guard`. If a case asserted that the tree close checks a non-`#task`
  checkbox, update it to the bob-cli rule (cite 532df825 in a comment). List every
  changed case in your bead note. Never weaken a `#task` assertion.
- Docs: in `docs/task-status-cycler.md` § "Requirement lines", add one sentence saying
  that the transcluded tree close (Ctrl+Enter through a Pomodoro transclusion) never
  checks a line without `#task`, matching bob-cli, while still walking that line's
  embedded children.

## 2. The insert twin reports `added`

After the RW twin was ported, bob-cli 536455ac
(`fix(capture): count nested requirements added to existing tasks`) added
`InsertWrite.added` to `src/native/task_block_layout/writers.rs` `insert_reqs`. It
counts the requirements owned by the target task after the write minus those owned
before, at every depth:

```rust
let owned_before = task_reqs::note_req_forest(contents, settings)
    .iter().filter(|node| node.owner_line == task.line_index).count();
let owned_after = forest.iter()
    .filter(|node| node.owner_line == task.line_index).count();
let added = owned_after.saturating_sub(owned_before);
```

The JS mirror `taskReqInsert` in
`plugins/task-status-cycler/src/198-task-reqs-writers-twin.js` returns `firstLine`,
`lineCount`, `scooped`, and `notesCreated`, but no `added`.

- Add `added` to `taskReqInsert`'s success object, in Rust's field order (after
  `lineCount`). Compute `ownedBefore` from `taskReqForest(text, resolved)` and
  `ownedAfter` from the existing post-write `forest`, both filtered by
  `row.ownerIndex === task.lineIndex`, then return
  `Math.max(0, ownedAfter - ownedBefore)`. Touch nothing else.
- Copy the fragment byte for byte to
  `plugins/bob-navigation-hotkeys/src/698-task-reqs-writers-twin.js` with `cp`. Use no
  hand edits; `scripts/test-task-reqs-twin-copies.cjs` enforces byte identity.
- Tests: in `scripts/test-task-status-cycler-task-reqs-vectors.cjs`, add `added` cases
  that mirror bob-cli's:
  - Insert one head with two nested reqs into a task that owns no reqs: `added === 3`.
  - Insert one head into a task that already owns two reqs: `added === 1`.
  - Insert into a task whose loose notes get scooped: notes never count.

  Every existing RW vector must still pass. `writes.json` does not pin `added`, so do
  not edit the fixture.

## 3. Ship

1. Bump the manifest minor version, as each epic phase did: `task-status-cycler` 1.56.0
   → 1.57.0, and `bob-navigation-hotkeys` 2.103.0 → 2.104.0, or the next minor above
   whatever master holds when you start. Follow `docs/develop/deployment.md`.
2. Run `npm run build`, `npm run docs:gen`, `npm test` (includes `build:check` and the
   docs inventory check), and `npm run docs:check`. Known failures unrelated to this
   work, which you must not fix here:
   - Load-sensitive perf budgets: `test-ledger-tools-frees-marks` (bob-cli-io) and
     `test-navigation-dependencies-stage-performance` (bob-cli-m1). Rerun them alone to
     confirm they pass.
   - Inside `npm run docs:check`, bob-cli master's own failures: the contract-registry
     tests in `tools/docs/tests/test_contracts.py` (DISCOVERED ISSUE #1 on bob-cli-on)
     and the published-URLs break at `src/native/highlights_ref/podcast/feed.rs:70`
     (recorded on bob-cli-os).

   Report any other failure honestly.

3. Deploy with `bob plugins sync -p task-status-cycler` and
   `bob plugins sync -p bob-navigation-hotkeys`. Never pass `-F`. If sync refuses,
   follow bob-plugins `AGENTS.md` ("When sync refuses an overwrite").
4. Do not commit by hand: your `/sase_final` declaration commits the linked-repo
   changes.

## 4. Close out the epic (final step, same turn)

Do this after steps 1–3 pass. It needs no commit SHA, push, or CI result from this work.

1. Run `sase bead epic-symbols bob-cli-on.10.1` and
   `sase bead epic-symbols bob-cli-on.10`. Resolve each entry they list: wire the symbol
   up, privatize it, add a non-test pragma, or delete it, per the Symvision
   epic-whitelist policy. Re-key a Justfile line to another bead only when a still-open
   later bead needs the exemption. At planning time both commands listed nothing.
2. Close the epic:

   ```bash
   sase bead close bob-cli-on.10.1 --note "<verification>"
   ```

   The note states the following. The land agent verified six phase commits:
   - bob-plugins: f292fff, e5cb895, 28d76fd, d3b80d7, d2877fe, and 7b96dce.
   - bob-cli: e5ee4a8d.

   Those commits passed:
   - RQ/RL/RW vectors (144/144) and twin copies (7/7).
   - Cycler guard, drop, toggle, and typing tests, nav guards (19), and the BIP guard
     (4).
   - SV verdicts (14), including SV25.

   The fixtures are byte-identical to bob-cli. Fragments stay at or below 1000 lines.
   The docs §5 gesture table landed. Integration found two gaps, fixed in this tale
   (name both fixes and their test counts):
   - The transcluded tree close now mirrors `finish_close_node`'s `#task` rule.
   - The insert twin reports `added`, mirroring bob-cli 536455ac.

   Add the `npm test` and `docs:check` results with the known unrelated failures, and
   the synced versions. Follow-up triage is in the epic's notes: new beads bob-cli-p8
   and bob-cli-p9, +1 on or, io, m1, and np, and notes on bob-cli-on and bob-cli-os.
   Never pass `--force`.

3. From the bob-cli workspace, run `just symvision` and confirm it is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the EPIC PLAN
   path `sase bead read bob-cli-on.10.1 -r "Need the plan path"` prints, for
   `plan:202611/obsidian_req_gestures.md`.
5. The parent `bob-cli-on.10` is a **phase** bead of epic `bob-cli-on`. Confirm that the
   child plan delivered everything in its description:
   - recognizer and layout twins in the cycler and nav;
   - every task gesture guarded on req lines;
   - the Ctrl+] toggle with zone placement;
   - Alt+[ drop and Alt+] restore;
   - the owner Task Card;
   - the offer-close scope fix;
   - the `- [` + space expansion with atomic Backspace.

   Then close only that phase:

   ```bash
   sase bead close bob-cli-on.10 --note "<what you verified, plus the follow-up triage outcomes above>"
   ```

   Do not close `bob-cli-on`; its land agent owns it. Never use `--force`.
