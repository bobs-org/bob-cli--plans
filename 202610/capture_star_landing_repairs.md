---
tier: tale
size: medium
title: Finish the capture `*` epic (bob-cli-7n) landing repairs and close it out
goal:
  Fix the epic-caused bugs and contract deviations that landing verification found in
  capture `*`, add the missing vector tests, correct the docs examples against the real
  binary, then close epic bob-cli-7n.
decisions:
  fix_star_decision_evidence_sha:
    ask:
      Correct the writer commit SHA that the capture-star decision record cites as
      evidence?
    memory:
      - decisions:capture-star-blocks-existing-tasks
    default: true
    corrects: "`1a074c6` (consumption), `8774e89` (parser), `158f6de` (writer),"
    answer: true
proposed_by: bbugyi200.athena.bob-cli-7n.land
bead: bob-cli-7n
decided_by: auto
status: done
---

- **PARENT:**
  [202610/capture_star_blocks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_star_blocks.md)
- **BEAD:**
  [bob-cli-7n](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7n/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-7n.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-7n.land.md)
- **COMMITS:**
  - [498e5ab](https://github.com/bobs-org/bob-cli/commit/498e5ab27f07e5dab7675a2cfae095a36e44dbae)
    — fix(capture): land star-blocks repairs and close the epic

# Plan: finish the capture `*` epic (bob-cli-7n) landing repairs and close it out

## Context

Epic **bob-cli-7n** (`plan:202610/capture_star_blocks.md`; read it with
`sase artifact read plan:202610/capture_star_blocks.md "<why>"`) added the capture `*`
sigil: `N *T` makes the captured task N a prerequisite of an existing open task T. All 8
phases are closed. Their commits are:

| Phase    | Commit                    |
| -------- | ------------------------- |
| contract | `04a6e84`                 |
| parse    | `8774e89`                 |
| consume  | `1a074c6`                 |
| complete | `78f1a69`                 |
| writer   | `158d6de`                 |
| execute  | `1b9d35f`                 |
| publish  | `908b3ef`                 |
| mac      | bob-mac-capture `f2a5cb5` |

The land agent verified the phases against the plan and the code. Most of the work is
correct:

- `just check` is green apart from the known ETXTBSY flake bob-cli-5l.
- The absorbed bugs bob-cli-7l and bob-cli-7m are verified fixed and closed.
- Follow-up triage is recorded as a note on bob-cli-7n:
  - duplicates bob-cli-7o, bob-cli-5l, and bob-cli-7h got a +1;
  - new beads bob-cli-7p and bob-cli-7q were filed;
  - the plan §8 items were declined.

Verification also found the gaps below. **The epic caused all of them, so they are epic
work.** This tale fixes them and then performs the epic's closeout.

Every repro below was run against a throwaway vault. Build one with the Tasks settings
from `tests/cli/support.rs` `blocked_tasks_settings_json`. Set these env vars:
`BOB_NOW`, `BOB_DAY_FILE`, and `BOB_CONFIG_FILE` (pointing at a nonexistent file).

## Rules

- **Never run a live `bob capture`, `bob task reconcile`, or `bob projects sync` against
  `~/bob`.** Dry runs are fine. Tests and repros use temporary vaults.
- **The docs own the behavior.** `docs/capture.md` and `docs/task-dependencies.md` are
  the contract. Where this tale picks a new wording, update the docs in the same change.
- **Do not edit bob-mac-capture.** Every change here is a bug fix or additive, and the
  Mac app only renders bob's output (`decisions:mac-capture-is-a-thin-client`).
- **`&` must not regress.** The only intended `&` changes are fixes 2, 3, and 8.

## Fixes (each gets a regression test)

### 1. A body-bearing close with `*` silently drops the `*`

- **Repro.**
  - `cash.md` is `- [ ] #task Fund ^fund`.
  - The day file has a running `- [ ] (1010-1035) — JOB` entry with one child.
  - `bob capture -- 'New thing @mac_inbox:new-thing=x *cash:fund'` exits 0. It closes
    the Pomodoro and creates N, but `cash.md` is unchanged and there is no warning.
  - `capture-parse` shows `mode: pomodoro_task`, one `blocked_tasks` entry, and no
    diagnostics.
- **Cause.** `src/native/capture/plan.rs` (~1229, the `if let Some(close_spec)` branch
  of `plan_capture_item`) returns early into `plan_pomodoro_close_task_item`, which
  ignores `blocked_tasks`. The execute phase removed the blanket "not supported yet"
  guard that used to catch this.
- **Contract.** Plan §2 rejects `@route:id=x… *T`, because closing and blocking in one
  item is incoherent. Only the bare form is refused today, with an
  `invalid_dependency_target` diagnostic: "relationship-only '…' must be bare …" in
  `editor_parse.rs` ~636 and `dependencies.rs` ~1179.
- **Fix.**
  - Refuse every close item that carries `blocked_tasks`, both at parse time (an editor
    diagnostic plus a `capture-parse` diagnostic) and at execution time. The message
    must name `*` and teach the two-item spelling.
  - Audit `plan_capture_item` for any other early return that ignores `blocked_tasks`.
    The `=` start forms already write the edge; keep them.
  - The `capture-parse` help says "a draft with blocked-task modifiers never captures
    without them", and that must hold afterwards.
  - Add the body-bearing form to the docs/capture.md §2 rejections list.

### 2. `task_blocks` rows are wrong when N and T share a note

- **Repro.**
  - `job.md` holds two lines: `- [ ] #task Cleanup ^cleanup` and
    `- [ ] #task Other ^other`.
  - `bob capture --dry-run -f json -- 'Ask Kelly @job *job:cleanup'` reports a
    `task_blocks` row for `other` with role `dependency_target`, though nothing touched
    it.
  - With two `*` targets, N's row appears twice.
  - `&` has the same defect when a same-note prerequisite sits below the dependent:
    `&job:third @job+cleanup`.
- **Cause.** The writer records `TargetTouch.line`
  (`src/native/capture/dependencies.rs`, struct ~870; pushes
  ~1032/~1674/~1923/~2060/~2534) before it inserts T's Depends-On line. That makes the
  line stale in the item's post-state text, which `TaskBlockRef.line` requires
  (`task_blocks.rs` ~25). `plan_blocked_task_writes`
  (`src/native/capture/blocked_tasks.rs` ~245) also discards its relocation result
  (`let _ = locate_task_by_block_id(..)`).
- **Fix.** Before building each `TaskBlockRef` (`target_block_ref`, `dependencies.rs`
  ~1576, and N's own `dependency_target` ref), relocate it by block ID in the item's
  final post-state text. `locate_task_by_block_id` already exists for this. Then
  `TaskBlockTracker::merge_or_start` merges rows correctly. Vault bytes must not change.

### 3. `dependency_updates[]` is missing on `&` close/reset items

- **Repro.** `'New thing @mac_inbox:new-thing=x &cash:fund'` emits a singular
  `dependency_update` (role `dependent`) but no `dependency_updates`.
- **Contract (§5).** The array is present whenever any update exists, and the singular
  field equals `[0]`.
- **Cause.** `src/native/capture/pomodoro_close.rs` (~1251, ~2607) hard-codes
  `dependency_updates: Vec::new()`.
- **Fix.** Enforce the invariant in one place, for example where the per-item JSON is
  built: when the singular field is set and the array is empty, emit `[singular]`. Audit
  the other `dependency_updates: Vec::new()` sites in `pomodoro_close.rs`,
  `pomodoro_adjust.rs`, `task_complete.rs`, and `plan.rs`.

### 4. Kept-notes text keeps tags (PDL8)

- **Repro.** In ST8, put an authored child note under a resident's Task Link. The
  consumed bullet becomes `- Work #prj_is_blocked`, so a managed tag leaks into the
  daily note.
- **Existing test.** `tests/cli/capture/pomodoro_close.rs` asserts the leaked
  `Ship it #prj #hide`.
- **Contract.** `docs/capture.md` "Notes kept (PDL8)" and `docs/task-dependencies.md`
  §13.5 "Removal (capture)" both say the text loses its checkbox, tags, inline fields,
  and `^id`, and only `[[…#^…]]` links are flattened.
- **Cause.** `src/native/capture/project_deps.rs` (~355-370) builds the text with
  `task_complete_display_text`, which strips only the global-filter tag, and
  `flatten_block_wikilinks` also flattens plain `[[note]]` links.
- **Fix.**
  - Strip every `#tag`, `[k:: v]` field, and `^id`, and collapse whitespace.
  - Flatten only block links.
  - Update the `pomodoro_close.rs` assertion.

### 5. Duplicate `*` targets are not deduped by identity, and a repeat reads as a block

- **Repro 1.** `'Ask Kelly *job:cleanup *Job:cleanup'` runs two writer passes and prints
  a second `🧱 would block "Cleanup"  job.md ^cleanup  [?] → [?] Blocked`.
- **Repro 2.** A repeat on an existing owner already on T's line,
  `*job:cleanup @body+work`, prints `🧱 ^work would now block "Cleanup" … [?] → [?]`.
- **Contract (§4 step 1, ST12, ST13).** Dedupe by resolved identity. A repeat is a no-op
  reported as `already_present`.
- **Fix.**
  - In `plan_blocked_task_writes`, resolve every target first. Skip duplicates by
    `(vault-relative path, block id)`, so a typed duplicate adds no `dependency_updates`
    entry.
  - When an update adds nothing (`added == 0`, `already_present > 0`), human output
    prints an "already" line instead of a status transition, for example
    `🧱 ^work already blocks "Cleanup"  job.md ^cleanup`, the same in dry runs.
  - Document the wording under the docs/capture.md human-output variants.

### 6. A fenced-code target is refused even when the real task is outside the fence

- **Cause.** `src/native/capture/blocked_tasks.rs` `block_id_in_fenced_code` (~71) is
  checked before the task lookup (~102). It refuses T whenever any fenced line carries
  the same `^id`.
- **Fix.** Refuse only when the resolved task itself is fenced, meaning no unfenced task
  with that block ID exists.
- **Test.** A note with a fenced example line `- [ ] #task Example ^cleanup` and a real
  `^cleanup` task outside the fence. `*` must accept it, as `&` already does.

### 7. Human-output variants (`src/native/capture/output.rs`)

- **Dry-run residents.** A dry run prints `· blocked N project tasks`. It must print
  `· would block N project tasks`.
- **Sub-bullet owners.** `Note text @body+work *job:other` prints
  `🧱 would block "Other"`. An existing owner must use the documented existing-owner
  form: `🧱 ^work would now block …` in a dry run and `🧱 ^work now blocks …` live. The
  check is keyed on `kind == "task_dependency"` and misses this path.
- **`*`-only existing-owner header.** `*job:cleanup @body+work` prints
  `✓ Add dependency to "Cleanup"  body.md`, pairing T's text with the owner's file.
  - Make the header name the owner consistently: the owner's text with the owner's file,
    or wording that names T with T's file.
  - Pick one wording and document it.
  - Keep `&` headers unchanged.

### 8. Forced-target rejections do not name the sigil

`src/native/capture/plan.rs` (~566-585) produces two errors that say "task dependencies"
even for `*`:

- "task dependencies with --task need --route for the dependent note"
- "task dependencies cannot attach to a --task-ref picker selection"

Name the sigil(s) actually typed: `&`, `*`, or both. Plan §2 says error text names the
sigil the user typed.

### 9. The `*` picker warns about a missing day file

- **Problem.** `src/native/capture_complete/candidates.rs` (~703, the blocked-task pool)
  returns `capture_active_tasks::read_ledger` warnings. A missing day file therefore
  surfaces `Bob daily note does not exist` in the `*` picker, which the `&` picker does
  not.
- **Fix.** Ranking is best-effort, so treat a missing day file as an empty ledger with
  no warning. Discard the ledger warnings the way the ~200 call site does.
- **Test.** Add one in `tests/cli/capture/complete_blocked_task.rs`.

## Docs corrections

Regenerate every literal from the real binary against a temp vault. Find each spot with
grep, because line numbers drift.

- **`docs/capture.md`:**
  - **`capture-parse` example `'Ask Kelly *job:cleanup *job:send-batch'`.** The real
    output has `route: null` and `dependency_target` `{"kind":"new_task"}`; the docs
    claim `mac_inbox`. The real ranges are 10–22 and 23–38; the docs say 10–23 and
    24–39.
  - **`*job:cleanup @bob+ask-kelly` example.** The range is 0–12, not 0–13.
  - **`capture-complete` example.** `-c 12` on `'Ask Kelly *jo'` yields query `"j"`, so
    use `-c 13`.
  - **ST3 JSON example.** Every `ask-kelly-for-references` (`block_id`, the
    prerequisite's `block_id`/`link`, `minted_blocker_block_id`, the `task_blocks`
    entry) must be `ask-kelly-references`, matching its own `task_line` and the binary.
  - **Leftover sentence.** Delete "The `publish` phase reconciles example literals
    against the shipped binary."
  - **Needs order.** List it as `…task_dependency, blocked_task, dependency_target…`,
    matching the binary and the help text.
  - **New wordings.** Document fixes 1, 5, and 7.
- **`docs/completion.md`:**
  - `&` rows are grouped `prerequisite tasks` (`capture_complete/shell.rs` ~320), not
    `tasks`, and the group list must include `prerequisite tasks`.
  - Drop or correct the claim that values ending in `*` or `&` get no trailing space.
    `ends_in_continuation` (`shell.rs` ~569) never checks those characters.
  - Say that a leading-run `*query` also opens the picker, not only a trailing one.

## Tests

- **`tests/cli/capture/parse_blocked_task.rs`:**
  - Parse-level tests for:
    - `@cash *job:cleanup Ask Kelly`
    - `@bob:ask-kelly *job:cleanup`
    - `X *"Release Plan":publish`
    - `!sase:x *foo` (keeps the bang teaching error)
    - `Ask Kelly @mac_inbox:ask-kelly *job:cleanup`
    - `Order parts *projects/house/kitchen:cabinets`
  - `*` twins of the `&` rejection tests, each asserting the message names `*`:
    - session operators
    - `+N`/`-N`
    - section bullet
    - project-note construction
    - `--task-ref`
    - suffixed owners
    - `@r:id=x *T`
    - fix 1's body-bearing close
  - Strengthen `=x*1`, `=*`, and `==*` to assert mode and body, not only
    `blocked_tasks: null`.
- **`tests/cli/capture/blocked_tasks.rs`:**
  - **ST3.** The "entry deleted" check looks for `(1010-1035) JOB`, but the file has
    `(1010-1035) — JOB`, so it can never fail. Assert exact day-file bytes.
  - **ST1–ST4 and ST8.** Assert exact bytes for T, N, and the day file (plan §6 /
    execute step 4).
  - **ST7.** Link T under an entry beside an unrelated childless placeholder entry, so
    consumption really runs. Assert the placeholder is untouched.
  - **ST8.** Assert `blocked_residents` equals the real count, and that each resident is
    `[?]` with `#prj_is_blocked`.
  - **ST9.** Add the new-N-lands-in-the-project form and the new-N-in-a-descendant form.
  - **ST17.** Add a Next T, and compare the day file too.
  - **ST19.** Add `@route:id *T`.
  - **Human variants.** Cover:
    - the kept-notes line
    - the residents line
    - every `would …` dry-run form, including residents
    - both existing-owner forms
    - the "already" line
  - **Regression tests.** One for each of fixes 1–9.
- **`&` coverage.** Existing `&`, link-guard, and BLOCKS tests stay green. Fixes 2, 3,
  and 8 each add an `&` regression test.

## Verification

1. Run `cargo fmt`, then `just check`. It must be green.
   - If only `native::highlights_ref::fetch::tests::*` fails with `Text file busy`, that
     is the known flake bob-cli-5l. Rerun it and cite it.
   - Do not run `just check-full`.
2. Run the built binary against temp vaults for ST1, ST3, ST4, ST8, and each repro
   above.
   - Confirm dry-run JSON equals live JSON.
   - Confirm a second live run is a no-op.

> [!decision] fix_star_decision_evidence_sha
>
> Run `/sase_memory_write` first. In
> `sase/memory/decisions/capture-star-blocks-existing-tasks.md`, change the Evidence
> entry `` `158f6de` (writer) `` to `` `158d6de` (writer) ``. Change nothing else; the
> record is otherwise accepted as written. Run `sase memory init` to regenerate
> `AGENTS.md`, the provider shims, and the memory README.

> [!decision] fix_star_decision_evidence_sha = no
>
> Leave the record unchanged and file the correction as a `memory` task bead through
> `/sase_new_task`.

## Final step: close out epic bob-cli-7n

This tale is the epic's landing. Nothing resumes it afterwards, so finish all of these
in the same turn as the code:

1. **Epic symbols.** Run `sase bead epic-symbols bob-cli-7n`.
   - For every listed `--epic-symbol` entry, resolve the symbol: wire it up, privatize
     it, add a non-test pragma, or delete it per the Symvision epic-whitelist policy.
   - Re-key a Justfile line only when a still-open later bead needs the exemption.
   - The list was empty at landing review; re-check it after the code changes.
2. **Close.** Run `sase bead close bob-cli-7n --note "<verification>"`. The note must
   summarize all of the following:
   - all 8 phases were verified against the plan and their commits;
   - bob-cli-7l and bob-cli-7m were verified and closed by the land agent;
   - the fixes this tale shipped (1–9, the docs, and the tests) and the `just check`
     result;
   - **integration:** since the epic started, the only non-epic bob-cli commit is
     `0ee788d` (`docs/ref.md`, unrelated), bob-plugins has no commits, and the
     bob-mac-capture Refs commits are unrelated;
   - **Mac:** `just portable-check` and the scratch typecheck from bob-cli-7n.7 passed.
     Mac CI is red only on Refs tests owned by bob-cli-7h and bob-cli-7p;
   - follow-up triage is already recorded on the epic.
   - Never use `--force`. If the close is rejected because `--epic-symbol` entries
     remain, finish that cleanup and close again.
3. **Symvision.** Run `just symvision` if the recipe exists. bob-cli's `justfile`
   currently has none; in that case skip it and say so in the final report.
4. **Plan status.** Set `status: done` in the frontmatter of the epic's plan file, the
   PLAN path shown by `sase bead read bob-cli-7n -r "<why>"`
   (`plan:202610/capture_star_blocks.md`).
5. **Parent.** bob-cli-7n has no `parent_bead`, so the landing ends here.
