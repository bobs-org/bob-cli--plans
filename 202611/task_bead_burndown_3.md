---
tier: epic
title:
  "Task-bead burn-down 3: signal-safe CI, executable stubs, capture block identity, and
  merchant names"
goal: "Close five verified task beads by repairing the clipboard CI signal harness,
  executable-stub races, two capture block-reference panics, and spaced merchant
  order-code cleaning; then triage and continue the task-bead burn-down.

  "
phases:
  - id: clipboard-signal
    title: Make the clipboard SIGINT test independent of inherited signal disposition
    size: small
    description:
      "clipboard-signal: fix the inherited-SIGINT test failure in chezmoi and close
      bob-cli-rd."
    depends_on: []
  - id: executable-stubs
    title: Adopt fork-safe writers for the remaining executable test stubs
    size: medium
    description:
      "executable-stubs: repair the Spotify and Gmail executable-stub races and close
      bob-cli-5l."
    depends_on:
      - clipboard-signal
  - id: idless-cancel
    title: Preserve task block identity when canceling an ID-less successor
    size: medium
    description:
      "idless-cancel: fix the capture -X stale task-block reference, verify JSON and
      written bytes, and close bob-cli-oh."
    depends_on:
      - executable-stubs
  - id: timed-entry-close
    title: Preserve Pomodoro block references around an existing timed entry
    size: medium
    description:
      "timed-entry-close: fix the x=build stale Pomodoro reference, preserve closed time
      and children, and close bob-cli-qa."
    depends_on:
      - idless-cancel
  - id: spaced-order-code
    title: Clean spaced Amazon order codes without erasing sub-merchants
    size: small
    description:
      "spaced-order-code: extend the conservative bank-name cleaner and close
      bob-cli-qx."
    depends_on:
      - timed-entry-close
proposed_by: bbugyi200.athena.bob-cli-r8.land
proposed_with: codex/gpt-6-astra
create_time: 2026-11-27 15:31:16
status: wip
---

- **PROMPT:**
  [prompts/202611/task_bead_burndown_3.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_bead_burndown_3.md)
- **PARENT:**
  [202611/task_bead_burndown_2.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_bead_burndown_2.md)

# Task-bead burn-down 3

## Scope and evidence

Continue the approved burn-down from `plan:202611/task_bead_burndown_2.md` (bob-cli-r8).
This is the next independent epic, not a child repair of r8. Five task beads, five
strictly sequential phases. No feature or memory opt-in is needed: all five repair
existing behavior and no memory is changed.

The r8 lander reviewed all phase notes and their actual commits across bob-cli,
bob-plugins, and chezmoi. It closed the omitted scratch task bob-cli-qt after verifying
its helper and the phase's two actual Home Cards retention runs. The other four r8 tasks
(id, or, pb, f8) were already closed. The r8 landing found no source integration
conflict with the concurrent prose, cabinet, tie-chord, or priority-bar work.

Triage on 2026-11-27 kept the previous triage's backlog by default. Since r8 began, the
new independent report was bob-cli-r9, a project-note type-menu refusal mismatch. Its
source condition still exists; keep it for a later batch. The lander also found and
filed bob-cli-rd after checking exact-SHA chezmoi CI. No other recently changed task had
evidence justifying a duplicate or obsolete close. Snoozed nc and ny still have elapsed
dates but remain snoozed; do not silently unsnooze them. The backlog remains large, so
this epic's lander must continue the chain.

| Task       | Priority and current evidence                                                                                                 |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------- |
| bob-cli-rd | Deterministic CI failure under inherited ignored SIGINT; exact-SHA CI and its predecessor both fail.                          |
| bob-cli-5l | 12 corroborations; Spotify still writes a runnable file in-process, and Gmail still copies one in-process.                    |
| bob-cli-oh | Debug capture panic and missing release-mode block output when canceling an ID-less dependent.                                |
| bob-cli-qa | Fresh r8 landing reproduction on cbbddc71 exits 101 at pomodoro_blocks.rs:685; release mode skips the same invalid reference. |
| bob-cli-qx | Distinct Amazon order codes survive cleaning, preventing merchant-name and category reuse.                                    |

This batch puts the newly isolated CI defect first, then the high-impact executable-stub
race and bounded correctness repairs. Keep bob-cli-mv open: r8 deliberately preserves
refusal when no single reachable commit contains the deployed bundle. Merely observing a
formerly dirty checkout now clean does not prove its deployed edits survived a reset or
rebase; mv needs a separate provenance design, not a permissive extension slipped into
this batch. Keep bob-cli-cd open too: the later bob-emoji recurrence needs a coordinated
vault untrack across hosts. Frequent flakes m1, ej, bo, gd, and pg remain eligible after
these correctness fixes.

## Rules and decisions

- Read each assigned task in full, including its typed fields, notes, and +1 evidence.
  Use `/sase_repo` before opening chezmoi or any other repository. Follow the returned
  checkout's AGENTS.md. Keep fixtures in scratch vaults.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit Cargo.toml versions or
  CHANGELOG.md. Preserve CLI/JSON schema and exit-code contracts; mark any unavoidable
  contract break with `!` and explain it.
- `decisions:unblocked-tasks-are-offered` and
  `decisions:task-deps-are-depends-on-links`: recovery and selection semantics stay
  unchanged; selected successor actions operate on the intended task, and unselected
  offers are not auto-linked.
- `decisions:consumption-keeps-notes-and-recorded-time`: preserve authored children and
  closed recorded time, including childless timed entries.
- `decisions:close-selections-list-explicitly` and
  `decisions:event-task-links-close-by-the-clock`: repair block tracking without
  changing close-selection or meeting-clock policy.
- The two capture phases fix stale identities at the writer/tracker boundary. Do not
  suppress debug assertions, swallow invalid references, or invent IDs for unrelated
  tasks to make a regression pass.
- `just check` is the bob-cli file-change gate. Never run `just check-full`. At r8
  landing, ToolRun 5787740ee74cb5105608f1e11247e2e2 failed only at known Clippy errors
  in prose/convert/inline.rs:760 (`filter_map_bool_then`) and :1357 (`useless_format`),
  introduced by eff5d986 and owned by active epic bob-cli-qj.5.1. Other known prose
  catalog/completion/root-help failures also belong there. Recheck ownership and report
  these if still present; do not fix unrelated active-epic code here. The bob-plugins
  snooze model vector `snooze-add-resnooze-replaces-history` belongs to bob-cli-q5.
- Run long checks and exact-SHA CI waits through `/sase_monitor` as required. Never end
  a turn with an unmonitored command still running.
- Phase workers record unrelated discoveries as `PROPOSED FOLLOW-UP:` notes, never
  create task beads. Close the assigned task only after its scope is verified, naming
  the code and commands/results. A finalizer SHA may be recorded by the lander; do not
  wait for your own future commit to close.
- Read docs.md before changing docs and cli_rules.md before changing CLI options/help.
  No docs or help change is expected except where an existing statement would become
  false.

## Phase clipboard-signal — bob-cli-rd

Open chezmoi.
`tests/bash/ssh_clip_doctor_test.sh::test_sigint_restores_once_and_stops_writes` expects
status 130 but gets 0 in CI. Run 46047534127 at 662569a34eb9209baa03e4cc50d32356ee2c7a08
and preceding run 45910998866 at d4cb694cc250b35f6b222ab8ad49f895d9c35463 fail the same
node. The r8 SHA's lint and repaired Poseidon assertions pass. This file and the
production doctor were unchanged by r8.

The lander reproduced locally by running bashunit after setting SIGINT to SIG_IGN in its
parent Python process; ordinary execution passes on the same tree. CI uses bashunit
0.53.0 while athena has 0.36.0. Confirm the precise CI launch path, but inherited
ignored SIGINT is already a demonstrated cause. `run_doctor` launches Bash directly, and
Bash cannot trap a signal ignored at entry. Fix the test boundary to start its child
with normal signal defaults (a small explicit Python exec wrapper is acceptable; Python
is installed in CI), leaving the surrounding runner's signal state alone.

Preserve the actual INT/TERM injection, exit statuses, exactly-once restore, clipboard
contents, and no-later-writes assertions. Do not skip the SIGINT case or turn it into an
exit-code-only mock. Verify ordinary and inherited-ignored focused runs, the complete
clipboard test file, and `just test-bash`. Follow chezmoi's apply-after-commit rule. The
lander must inspect exact-SHA CI after finalization, using a monitor if still running;
this must not be a prerequisite for this phase's own close.

## Phase executable-stubs — bob-cli-5l

The original lib stub writers were repaired in f4f6a9ca, but later additions
reintroduced the same race. Current concrete sites:

- `src/native/highlights_ref/spotify.rs` fake-curl helper, around line 805: `fs::write`
  then chmod 0755.
- `tests/gmail_support/mod.rs::FakeGog::new`, around line 233:
  `fs::copy(FAKE_GOG_SRC, &path)` then `chmod_exec`. The task's correction note matters:
  this is a parent-process copy, not fs::write, but it exposes the same
  writable-descriptor inheritance window.

Use `src/native/test_stub.rs::write_executable_stub` for lib tests. For integration
tests, reuse or factor the established `tests/cli/support.rs::write_executable`
scratch-payload/child-copy pattern into an appropriate test-only shared module. Do not
make a cfg(test) lib helper public just for integration tests. Audit other executable
test writers and port same-cause sites; do not alter production writers or
non-executable fixture writes. Preserve contents, permissions, and cleanup.

Run the existing fork-stress test, Spotify redirect tests, and the Gmail transport tests
named in the latest +1. Verify repeatedly under the normal parallel harness and once
with high thread count; keep stress bounded and assert successful execution, not elapsed
time. Run `just check`, reporting only independently owned failures separately. Close
bob-cli-5l when no in-process writable descriptor remains on these executed stubs.

## Phase idless-cancel — bob-cli-oh

Start with the bead's scratch-vault reproduction: an open Pomodoro linking
`[[sase#^bar]]`, task Bar with `[id:: bar] ^bar`, and two ID-less Blocked dependents
Alpha/Beta in deps.md. With the configured Tasks statuses and 2026-10-05 clock, run
`bob capture -f json -X 1=why -- '!sase:bar'`. The debug assertion is now in
`TaskBlockTracker::finish` near `src/native/capture/task_blocks.rs:618`; the bead's
older line 587 is stale.

Trace the offer-close mutation and every insertion/reorder that moves the remaining
ID-less task. Repair the reference's forwarding through those edits or its origin
mapping so final `tracked.current` resolves the intended task. Preserve Cancel Log
placement and the unselected dependent's recovery. Do not use a broad text-only lookup
that conflates same-title tasks.

Add a CLI regression with the original fixture, a duplicate-title variant, and a
reordered-selection case. Exercise dry-run and real execution; assert task_blocks
target, line, roles, and complete child bytes against actual written notes, and prove
dry-run is mutation-free. Cover selection without `-j` (original failure) and its
jump-enabled consumer if it uses the same mapping. Run focused capture/offer-close tests
and `just check`, then close bob-cli-oh.

## Phase timed-entry-close — bob-cli-qa

Reproduce with the exact 2026-10-08 fixture in the bead: daily Pomodoros has an empty
open BUILD followed by a closed `0700-0800` WORK entry with `[t:: 60m]` and child
`[[jobs#^early]]`; jobs.md has the In Progress task `^early`. At 11:00 America/New_York,
`bob capture -f json -- x=build` hits `PomodoroBlockBefore::Resolve` with an after-line
absent from post_entries.

Trace the close planner's line reference before and after insertion, relocation, and
settle. Forward that reference against the actual staged text, using the existing
tracker interfaces. Keep the closed WORK entry, its recorded time, and its child link
intact. Avoid conflating duplicate entry headlines; use origin/identity where available.

Add this fixture to the existing pomodoro-close CLI tests. Use
`assert_pomodoro_blocks_cover_changes` and explicit target/line/text checks so release
behavior cannot silently omit a block. Verify dry-run/real parity, closed timed-entry
preservation, a reordered closed entry, and the childless-timed preservation rule.
Re-run the preceding ID-less regression because both phases touch staged-line tracking.
Run focused tests and `just check`, then close bob-cli-qa.

## Phase spaced-order-code — bob-cli-qx

`src/native/bank/naming.rs::strip_processor_order_code` only examines each whitespace
token's suffix after `*`. Thus `RETA* BV5NH` survives, although the existing
`is_short_digit_code` already recognizes BV5NH.

Use a peekable token walk (or equivalent) to recognize a word ending in `*` followed by
one short digit-bearing code. Remove the star and that one code, preserving the merchant
prefix and every later word. Reuse the existing code predicate, including its
masked-digit behavior. Preserve word-only sub-merchants such as `DELIVERY *SUSHI`, glued
forms, trailing lone stars, and the cleaner's never-empty fallback.

Add fixtures for the bead's full Wells Fargo string and two distinct Amazon codes
producing the same cleaned name/merchant key; cover whitespace, oversized/non-code
suffixes, and existing processor cases. Verify learned naming can reuse the resulting
merchant key with synthetic transactions. Do not migrate the live journal or rewrite
historical payees. Run naming unit tests, bank import regressions, and `just check`;
close bob-cli-qx.

## Landing and the next epic

The land agent verifies every phase, note, commit, decision, and integration with
changes landed since this epic started. It inspects the chezmoi CI run on the exact SHA
pushed by clipboard-signal. Watch a running job with `/sase_monitor` rather than waiting
inline. Do not claim green unless it is green; triage a genuinely unrelated failure
separately and record that limit.

1. **Confirm this epic's beads.** bob-cli-rd, bob-cli-5l, bob-cli-oh, bob-cli-qa, and
   bob-cli-qx must each close with evidence, or the land note must explain why a task
   stays open. Collect all PROPOSED FOLLOW-UP notes; use `/sase_new_task` for distinct
   unrelated work and record every outcome.
2. **Triage what is left.** List open task beads (`sase bead list -t task -n 0`; JSON
   includes updated_at and plus_one_count). Focus on additions/updates after the
   2026-11-27 r8 landing and on fixes that landed meanwhile. Keep by default. Close
   duplicates or obsolete tasks only with checkable evidence: a fix commit and source,
   passing check, existing memory, governing decision, or explicit reviewer refusal of a
   memory proposal. Age and low +1 counts are not reasons. For large batches, read-only
   helpers may review; the lander performs every close.
3. **Plan the next epic** with `/sase_plan` if any eligible task is neither snoozed nor
   owned by an active epic. Select at most five tasks and five phases, strictly
   sequential. Prioritize deterministic gate failures, correctness/data-loss bugs,
   frequent flakes by +1 count, docs gates, other bugs, false memory, then features/new
   memory. Read the decisions roster and cite governing records. Each feature, memory,
   or previously opt-in task needs a reviewer decision; its no branch closes it
   canceled. Use `/sase_memory_write` before proposing memory edits. Do not re-propose
   records already declined. Copy this landing-and-next-epic section into the new plan
   with its task IDs and current evidence, continuing the chain.
4. **Close this epic normally.** Retire/re-key all entries from
   `sase bead epic-symbols <this-epic>` before close, run `just symvision` if available
   after close, and mark this plan `status: done`. Do not force a successful landing.
   Follow the land prompt's parent rules when applicable.
5. **Stop the chain** only if remaining tasks are snoozed or owned by active epics. List
   their wake times or owning epics in the landing note. Otherwise validate with
   --explain, revalidate, propose the next epic, and finish via `/sase_final` so the
   host can adopt it.
