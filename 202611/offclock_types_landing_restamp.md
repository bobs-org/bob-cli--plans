---
tier: tale
size: medium
title:
  "offclock_types landing: re-stamp the clock after capture landing retypes, then close
  bob-cli-hc"
goal:
  A capture that links or starts a task and retypes it in the same write stamps the
  destination entry's [t::]/[off::] key from the post-retype type, and epic bob-cli-hc
  is closed out.
proposed_by: bbugyi200.apollo.bob-cli-hc.land
proposed_with: claude/opus
bead: bob-cli-hc
create_time: 2026-11-05 17:33:10
status: wip
---

- **PARENT:**
  [202611/offclock_types.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/offclock_types.md)
- **BEAD:**
  [bob-cli-hc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hc/README.md)

# Re-stamp the clock after capture landing retypes, then close bob-cli-hc

This is the last piece of epic **bob-cli-hc** ("offclock_types: off-clock by area and
task type", plan `plan:202611/offclock_types.md`). All six phases are closed, and the
land agent verified their work. This tale fixes the one remaining defect the epic
caused, then closes the epic. No other agent resumes this landing, so the closeout at
the end is part of this tale.

## Background

The epic made a task's **effective type** an input to its clock. An area or project
note's `offclock_types:` list sends only the listed types off the clock. The live vault
has `offclock_types: [MEET]` on `love.md` and `body.md`. Clock resolution lives in
`src/native/pomodoro_homes/clock.rs` (`effective_clock`). The start and lineup clock
writers live in `src/native/capture/meet_event_run.rs` (`finalize_session_clock`,
`apply_meet_rule`, `restamp_session_clock`). Read the contract table under "Which clock
a session is on" and "What counts" in `docs/capture.md`.

Before this epic, retyping a task could never change a session's clock. Now it can.
Phase bob-cli-hc.2 made `bob capture-pomodoro-name` and `#type` relocations re-stamp the
running entry after retyping (commit e6122f7a). One writer path was missed: the
**capture landing retype**.

## The defect

When a capture links or starts an existing or new task into a Pomodoro, the landing may
also retype the task in the same transaction:

- MEET destinations always take the retype path.
- Since bob-cli-hd's A10 rule (bfd7b005), a `#NAME` / `=name` destination that the same
  capture run created also retypes.

This happens in `CaptureTaskThemeContext::apply` → `apply_theme_change` in
`src/native/capture/task_theme.rs`. It runs _after_ the link/start planner has already
stamped the entry's clock. The written `[t::]` / `[off::]` key therefore reflects the
**pre-retype** type. That breaks the documented "start" and "running lineup edit" rows,
which say the key comes from the counted Task Links resolved against the staged vault.
The same capture's own `plan_budget.links.off_clock` already counts the link as
off-clock, so one capture reports contradictory clocks. Close later re-stamps correctly,
so closed history is fine. But the open row is wrong: the Now card, the agenda, home
chips, and the Mac menu-bar indicator show Worked for an off-clock session.

Reproduce it with a debug build of `bob`, using a scratch vault and config:

- Config file (`BOB_CONFIG_FILE`):

  ```yaml
  plan:
    reading_type: READ
  types:
    MEET: red
    CHORE: magenta
  ```

- `love.md`:

  ```markdown
  ---
  type: [[area]]
  offclock_types: [MEET]
  ---

  ## Tasks

  - [ ] #task Dinner with Kelly [type:: CHORE] ^dinner
  ```

- `2026/20261101.md` (also `BOB_DAY_FILE`): `## Pomodoros` followed by
  `- [ ] () — MEET`.
- Run
  `BOB_NOW='2026-11-01 09:02:00' bob capture -b <vault> -f json -- '^love:dinner=meet'`.

Observed, wrong: the task becomes `[type:: MEET]` and the entry becomes
`(**0905-0930** [t:: 25m]) — MEET`. `pomodoro_start.time_range` says `[t:: 25m]`, while
`plan_budget.links.off_clock` is 1. Expected: `[off:: 25m]` everywhere. The same thing
happens in these variants:

- the day has only `- [ ] () — LATER`, so `=meet` creates the entry;
- A10: `offclock_types: [CHORE]`, the task is `[type:: MEET]`, and the token is
  `^love:dinner=chore`;
- a running-lineup add: the day has `- [ ] (**0900-0925** [t:: 25m]) — MEET`,
  `BOB_NOW='2026-11-01 09:05:00'`, and the token is `^love:dinner#meet`. The task is
  retyped to MEET, but the key stays `[t:: 25m]`.

Closing with `x` at 09:31 then writes `[off:: 25m]`, which confirms that only the open
row is stale.

## Fix

1. **Re-stamp after a landing retype.** When the landing writes a type change
   (`LandOutcome::Set` / `Clear` through `apply_theme_change`) and the task's live link
   sits directly under an **open, timed** entry in the staged daily note, recompute that
   entry's key from the post-retype staged vault in the same transaction. Find the entry
   the same way `stage_merge_rename` does, through the `#^{block_id}]]` link line. Leave
   a placeholder `()` entry or a closed entry alone.
   - Use the same clock mode as the write that just ran. If this item **started** the
     entry (a link start, task start, or named or created start), re-run the start's
     rule with `use_event_run: true` and `SpanRule::Written`, exactly like
     `apply_link_start_clock` in `src/native/capture/pomodoro_link.rs`. A MEET with a
     non-empty event run then keeps following its run, and a MEET without one falls back
     to the counted links. Otherwise the item edited an already-running lineup: use
     `restamp_session_clock` (`use_event_run: false`), like the other lineup edits.
   - Add a small mode parameter to `restamp_session_clock` (or a sibling function)
     instead of copying the clock rule. Never write a second clock resolver.
   - `CaptureTaskThemeContext` stores `today` but not `now`. Thread `now` and the
     started/lineup mode in from the callers. The call sites are
     `pomodoro_link.rs::plan_existing_task_pomodoro_link_item` (it knows
     `link.pomodoro_start`), `plan.rs` (`apply_existing_task` around the A10
     `destination_is_new` block and `apply_new_task` in the routed-note path), and the
     three in `project_note.rs`. Add the arguments to `apply_new_task` /
     `apply_existing_task`, or store `now` on the context: pick whichever keeps clippy's
     `too_many_arguments` allowances from spreading. A Merge outcome writes no task
     field and needs no re-stamp, and neither does a landing that changes no type.
2. **Keep every report consistent with the written key.** After a re-stamp, update
   whatever this item already reported: `pomodoro_start` (`time_range`, `tracked`,
   `off_clock`, any clock note it carries), the human start line's off-clock suffix (for
   example `· off-clock via love`), and `pomodoro_blocks` line text, if those were
   captured before the re-stamp. Surface any `pomodoro_clock_mixed` warning the re-stamp
   raises, once. `--dry-run` must report the same post-image as the real write.
3. **Leave alone what the contract says to leave alone.** A `#type` retype that leaves
   the link in place without a landing (for example `@love+dinner #meet` while the link
   already sits under a running entry) still keeps the current key until close, as the
   "Other task type, root, or note-default changes" row says. Relocations already
   re-stamp through `restamp_retyped_timed_entries`. Do not change bob-plugins: Obsidian
   naming is tracked separately as task bob-cli-hk.

## Tests

Write the failing CLI tests first, in `tests/cli/capture/` (next to
`type_marker_existing.rs` or `pomodoro_clock.rs`), each covering the real write and
`--dry-run`:

- existing untimed MEET entry plus `^love:dinner=meet` on a CHORE love task under
  `offclock_types: [MEET]` → `[off:: 25m]`, `pomodoro_start.off_clock` true, and a
  `time_range` that matches the file;
- `=meet` creating its entry, same expectation;
- A10: `^love:dinner=chore` under `offclock_types: [CHORE]` on a MEET task → `[off::]`;
- a running-lineup add (`^love:dinner#meet` into a running MEET) → the key flips to
  `[off::]`;
- a guard case: the reverse retype (a MEET task landing in a created CHORE entry under
  `offclock_types: [MEET]`) flips a would-be `[off::]` to `[t::]`;
- a guard case: a MEET start with a non-empty event run keeps the run's clock (reuse an
  existing event-run fixture from `tests/cli/capture/`);
- the unchanged-behavior case: `@love+dinner #meet` with the link already in place keeps
  `[t::]`.

Do not edit `tests/fixtures/pomodoro_clock/vectors.json`. Its copies in bob-plugins must
stay byte-identical, and these cases are capture-writer behavior rather than resolver
contract.

## Docs

`docs/capture.md`, in the "What counts" table and the landing paragraph near "When the
same capture run created the destination entry, the task instead takes the retype path
(A10)": state that a landing retype is part of the start or add, so the key is computed
from the post-retype links. Keep the "Other task type …" row for retypes outside a
landing.

## Gates

Run `just check` and `just docs-check`. The machine can be heavily loaded: if a test
command or the docs tutorial build times out, rerun it with a larger explicit timeout.
These failures are known, unrelated, and already tracked, so do not fix them here:

- `highlights::ingest::create_keeps_intake_for_flag_config_env_and_dry_run`
  (minute-boundary flake, bob-cli-hf);
- `capture::related::draft_related_stays_under_the_debug_perf_gate` (load flake,
  bob-cli-bo).

If `just docs-check` reports generated-docs drift in files this tale did not touch (for
example `docs/reference/cli/capture-parse.md` from commit 6e816775), do not commit that
regeneration. Report it instead.

## Epic closeout (final step)

When the fix is green:

1. Run `sase bead epic-symbols bob-cli-hc`. At planning time it listed no entries. For
   any entry that appears, resolve the symbol (wire it up, privatize it, add a non-test
   pragma, or delete it per the Symvision epic-whitelist policy), or re-key its Justfile
   line to a still-open bead that needs it.
2. Close the epic. Do not use `--force`; if the close is refused, fix the cause.

   ```bash
   sase bead close bob-cli-hc --note "<verification>"
   ```

   The note must summarize:
   - all six phases verified against the plan and its DECISIONS;
   - the land agent's integration edits: the strand
     `decisions:off-clock-note-defaults-filter-by-task-type` gained `keyword:` and
     accepted metadata; the gcal clock unit test now injects `TypeRules`;
   - bob-cli-hd's fmt, clippy, and two CLI-test regressions were bisected to 3c2d9622 /
     cccdf94e and fixed by hd's lander 5235d292, not by this epic;
   - this tale's landing-retype re-stamp fix and its tests;
   - follow-up triage: the tutorial `=#focus` failure was already fixed by 2e91d45b; the
     perf flake was corroborated on bob-cli-bo; the Obsidian naming re-stamp was filed
     as bob-cli-hk;
   - the vault cutover gate (see below).

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read bob-cli-hc -r "..."` shows (`plan:202611/offclock_types.md` in the
   plans repo); change only the `status:` line.
5. bob-cli-hc has no `parent_bead`, so no parent needs handling. Confirm this with
   `sase bead read bob-cli-hc -r "Need the parent link"`. If a parent does show up,
   handle it the way the land instructions describe for phase and plan parents.

## Vault cutover gate

The epic's `cutover = gate` decision staged gate `custom-74557157`. It removes
`offclock: true` from `~/bob/love.md` and `~/bob/body.md`. Bryan confirmed it on
2026-11-05, but its guard aborted with `aborted_binary_too_old`: the installed `bob` on
apollo predates `offclock_types`. Both vault notes still carry the dual keys, which is
the correct interim state. Do not run `just install` or `just install-all`, and do not
edit the vault. In the final response, tell Bryan to run `just install-all` on athena,
apollo, and the Mac, ideally after this fix ships, and then restart the gate.
