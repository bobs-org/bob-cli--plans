---
tier: tale
size: medium
title:
  "Finish backlog batch 9: keep unrelated panic messages under the EPIPE guard, keep
  usage-error exit codes, then close the epic and propose batch 10"
goal:
  Repair the two defects batch 9's EPIPE phase introduced in src/runner.rs, close
  bob-cli-bl and epic bob-cli-an.6.6.6.5.6.5.5.6, mark its plan done, and submit the
  mandatory successor backlog epic.
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.land
proposed_with: claude/opus
bead: bob-cli-an.6.6.6.5.6.5.5.6
status: done
---

- **PARENT:**
  [202610/backlog_batch9_epipe_listen_publish_refresh_flakes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch9_epipe_listen_publish_refresh_flakes.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.land.md)
- **COMMITS:**
  - [af3b30c](https://github.com/bobs-org/bob-cli--plans/commit/af3b30c5c2d6b941b95a4f684e4420d2c813db23)
    — chore(plans): mark batch 9 plan done after EPIPE hook closeout

# Finish backlog batch 9: EPIPE guard repairs, epic closeout, and batch 10

You are finishing the landing of epic `bob-cli-an.6.6.6.5.6.5.5.6` ("Backlog batch 9").
Its lander verified everything except the EPIPE phase and handed the remaining work to
you. Nothing resumes the landing after you, so this tale ends with the epic closeout and
the successor-epic proposal. Read the epic first:

```bash
sase bead read bob-cli-an.6.6.6.5.6.5.5.6 -r "Need the epic scope, decisions, lander notes, and plan path"
sase bead read bob-cli-an.6.6.6.5.6.5.5.6.1 -r "Need the EPIPE phase notes"
sase bead read bob-cli-bl -r "Need the EPIPE task report"
```

The epic's DECISIONS are final: `epipe_status = zero`. A closed stdout or stderr pipe
ends bob with status 0 and no output. Do not restore `SIGPIPE` to `SIG_DFL`.

## Already done by the lander (do not redo)

- Verified and closed `bob-cli-c1`, `bob-cli-c0`, `bob-cli-bo`, and `bob-cli-c2` with
  evidence (sase-listen `9d69f50`; bob-mac-capture `461f46c` with macOS CI 3087/0;
  bob-cli `0e75ca56` and `f255819a`; loaded `cargo test --test cli` and ref_library
  reruns all green). bob-cli CI on `6292c35b` passed.
- Follow-up triage is finished and recorded in the epic's lander note. No phase recorded
  a `PROPOSED FOLLOW-UP`. Two unfixed source follow-ups from post-start drift commit
  `1194779f` became `bob-cli-c3` (URL dedupe keys drop explicit non-default ports) and
  `bob-cli-c4` (capture TEXT help overstates the unknown-flag guard). The phase-2
  auto-publish warning wording was declined as an accepted limitation.
- `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6` reported no entries.
- The parent bead `bob-cli-an.6.6.6.5.6.5.5` (batch 8) is already closed. Its lander
  closed it before proposing batch 9.

## Step 1: Fix the two epic-caused defects in `src/runner.rs`

Phase `.1` (commit `6292c35b`) wrapped `run_bob_inner` in a panic hook plus
`catch_unwind`. Two behaviors are wrong.

### Defect A: every panic is now silent

The hook installed in `run_bob` has an empty body for non-EPIPE panics, so it prints
nothing for any panic. `run_bob` then rethrows non-EPIPE payloads with
`std::panic::resume_unwind`, and `resume_unwind` never invokes a panic hook, so the
restored hook never runs either. Result: an unrelated panic exits 101 with **no message
at all**. A panic in any thread spawned during the command is also swallowed. The lander
reproduced this with a standalone program of the same shape (take_hook, silent set_hook,
catch_unwind, set_hook(previous), resume_unwind): a deliberate index panic gave exit 101
and empty stderr. This breaks the phase acceptance "Unrelated panics still print and
exit 101."

Fix: the hook must delegate every non-EPIPE panic to the previous hook at panic time and
stay silent only for `is_epipe_print_panic` payloads. A natural shape is to factor the
guard into a small private helper that `run_bob` calls with `run_bob_inner`:

```rust
fn run_with_quiet_epipe(run: impl FnOnce() -> i32) -> i32 {
    let previous_hook = std::panic::take_hook();
    std::panic::set_hook(Box::new(move |info| {
        if !is_epipe_print_panic(info.payload()) {
            previous_hook(info);
        }
    }));
    match std::panic::catch_unwind(std::panic::AssertUnwindSafe(run)) {
        Ok(exit_code) => exit_code,
        Err(payload) if is_epipe_print_panic(payload.as_ref()) => 0,
        // The hook already printed this panic once; resume_unwind does not
        // run it again, and the runtime turns the unwind into exit 101.
        Err(payload) => std::panic::resume_unwind(payload),
    }
}
```

The hook does not have to be restored after `catch_unwind`, because the process exits
right after `run_bob` returns. Restore it (for example through an `Arc`) only if you
prefer. `src/native/completion/mod.rs` takes and restores whatever hook is current, so
it keeps working either way. Rewrite the misleading comments in `run_bob` and the hook
so they describe the real behavior.

### Defect B: usage errors exit 0 when the reader is gone

In `route_help` (the `invalid_help_path` branch) and `run_bob_with_args` (the
`try_get_matches_from` error branch), a `BrokenPipe` from `error.print()` now returns
exit 0. When the reader of bob's output has closed, a real usage error therefore reports
success. The lander reproduced this: `( sleep 0.3; bob bogus-subcommand 2>&1 ) | true`
and `( sleep 0.3; bob stats --bogus-flag 2>&1 ) | true` both exited 0, while the same
commands normally exit 2.

The `zero` decision covers a reader that stopped reading output it asked for. It does
not turn a failed invocation into success. Fix: on `BrokenPipe`, skip the
`failed to print command-line error` message and return clap's own `exit_code`
(`Err(exit_code)` in `route_help`). Help and version renders keep exit 0, because clap's
`DisplayHelp`/`DisplayVersion` exit code is already 0, so `bob --help | head -c 1` stays
0 and quiet. Usage errors keep 2.

Leave the `capture --expand` `BrokenPipe` → 0 mapping in `src/native/capture/mod.rs`
unchanged. That command succeeded, and only its output was cut short.

### Tests

Keep the existing classifier unit tests and `tests/cli/epipe.rs` unchanged and green.
`epipe.rs` proves the in-hook classification still silences a real EPIPE print panic.

Add lib unit tests in `src/runner.rs` for the guard helper. A panic hook is
process-global and lib tests run in parallel, so do not install the hook inside the
shared test process. Use the re-exec pattern instead: the test spawns
`std::env::current_exe()` with
`--exact <its own test path> --nocapture --test-threads=1` and a probe environment
variable. In the child, when the variable is set, run the probe case under the helper,
wrap that in an outer `catch_unwind` so a rethrown panic maps to 101, and end the child
with `std::process::exit(code)`. Assert at least these cases:

1. Unrelated panic (for example `panic!("runner-guard-probe boom")`): exit 101, and
   stderr contains `panicked` and the message exactly once.
2. Panic in a spawned thread that the closure joins and then returns 0: exit 0, and
   stderr contains the thread's panic message.
3. EPIPE-shaped payload
   (`std::panic::panic_any(String::from("failed printing to stdout: Broken pipe (os error 32)"))`):
   exit 0 and empty stderr.

Add a CLI regression to `tests/cli/epipe.rs` for defect B. Spawn bob with an invalid
subcommand and piped stderr, close the read end before bob writes, and assert exit 2
with no `panicked` and no `Broken pipe`. You can close the read end by dropping
`child.stderr` immediately after spawn, or by passing in the write end of a pipe whose
read end is already closed. Add a matching help case that asserts exit 0. If dropping
the handle cannot make the write race deterministic, make it deterministic another way,
for example with a large help output or a reader that closes before bob writes. Do not
`sleep` and hope.

Run the focused tests (`cargo test --lib runner::`, `cargo test --test cli epipe`),
rebuild bob through `$CARGO_TARGET_DIR`, and recheck by hand:

- `BOB_DIR=tests/fixtures/stats/vault BOB_NOW="2026-10-11 12:00:00" NO_COLOR=1 bob stats -r 2026-01-01..2026-10-11 -f json | head -c 10`
  under `set -o pipefail` gives status 0 and empty stderr.
- `bob --help | head -c 1` gives 0 and empty stderr.
- The two sleep-then-closed-pipe usage-error commands above give 2.

Then run `just check`, not `just check-full`, as one long foreground command with a
generous timeout. No docs exit-status contract exists (phase `.1` checked), so no docs
change is expected. Do not edit `Cargo.toml`'s version or `CHANGELOG.md`.

## Step 2: Close `bob-cli-bl`

Once step 1 is green, close the task:
`sase bead close bob-cli-bl --note "<baseline, both fixes, tests, and the hand checks above>"`.
Name commit `6292c35b` plus your fix and the observed exit codes.

## Step 3: Close epic `bob-cli-an.6.6.6.5.6.5.5.6`

1. Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6`. For each listed
   `--epic-symbol` entry, resolve the symbol (wire it up, privatize it, add a non-test
   pragma, or delete it per the Symvision epic-whitelist policy). Re-key a Justfile line
   to a still-open bead only if that bead still needs the exemption. The lander saw no
   entries.
2. Close the epic normally, never with `--force`:
   `sase bead close bob-cli-an.6.6.6.5.6.5.5.6 --note "<verification>"`. Summarize the
   lander's note (four targets verified and closed, CI evidence, follow-ups c3/c4
   created, auto-publish wording declined, no epic symbols) and add your step 1–2
   evidence. If the close is rejected, fix the cause and close again.
3. Run `just symvision` if the recipe exists (`just --list | grep -i symvision`). At
   batch-8 landing bob-cli had none; if it is still absent, record that in your notes.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202610/backlog_batch9_epipe_listen_publish_refresh_flakes.md`. Use the PLAN
   path printed by `sase bead read bob-cli-an.6.6.6.5.6.5.5.6`; it lives in the plans
   sidecar.
5. Parent handling: the epic's `parent_bead` is the plan bead
   `bob-cli-an.6.6.6.5.6.5.5`. Re-read it with
   `sase bead read bob-cli-an.6.6.6.5.6.5.5 -r "Need the parent link"`. It was already
   closed (done, 2026-10-25T12:34:58Z) by its lander before batch 9 was proposed, so
   there is nothing to close. If it is unexpectedly open, follow the standard
   plan-parent readiness traversal instead: recheck its descendants and plan, retire its
   epic symbols, and close it normally only if it is complete. Stop at the first
   incomplete ancestor and record a blocker note on it.

## Step 4: Commit, then propose the successor epic (mandatory continuation)

`sase plan propose` terminates the runner without the commit finalizer. Before
proposing, commit every repository you changed with `/sase_git_commit`: bob-cli for the
runner fix and tests, and the plans sidecar for the `status: done` edit. This approved
plan explicitly authorizes `/sase_git_commit` for exactly these repository obligations,
as batch 9's own plan does for its lander.

Then carry out the continuation contract below as the effective lander of batch 9.
Author the next backlog epic ("Backlog batch 10") through `/sase_plan` with
`parent_bead: bob-cli-an.6.6.6.5.6.5.5.6`. Copy the contract verbatim into it, including
its own lander instructions, refreshing only the candidate guidance in item 3. Before
proposing, record the successor's reference and ownership with
`sase bead note bob-cli-an.6.6.6.5.6.5.5.6 "CONTINUATION: ..."`, including the fresh
unresolved count.

Lander's count after closing c0, c1, bo, and c2: **127 unresolved tasks, 123 ready and
four snoozed** (`3o`, `6q`, `aa`, `ah`). Closing `bl` makes it 126. Enumerate again
anyway. Candidate notes beyond the contract text below: `bob-cli-c3` (small, reproduced
port-dedupe identity bug; it can merge distinct references, so rank it above flakes) and
`bob-cli-c4` (xsmall help-text accuracy). `bob-cli-4k` passed again on bob-mac-capture
macOS CI run `40195734261` (`461f46c`), the fourth consecutive pass, so recheck whether
to close it as not reproducing with that evidence.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include flakes `bob-cli-4k` (recheck recurrence), `bob-cli-5o`,
   `bob-cli-b9`, `bob-cli-67`, and `bob-cli-2m`; `bob-cli-b8` for live identity repair
   (needs an implement/decline decision); `bob-cli-9i` for the pdftotext-absent Mermaid
   tests; small deterministic plugin bugs such as `bob-cli-4e`, `bob-cli-8z`,
   `bob-cli-9q`, and `bob-cli-bn`; and Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`,
   and `bob-cli-9x`. `bob-cli-8q` belongs to active epic `bob-cli-3j` while that epic
   owns the PTY completion drivers. This is guidance, not a preassigned batch, and
   concurrent work may obsolete it.
4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes **and** live hooks report `legacy_dependency_children: 0`.
   Verify real prerequisites, not just wake timestamps. Work other actionable tasks
   first. If only gated tasks remain, the next bounded epic must own the readiness check
   and supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.

The successor epic must also carry the working rules batch 9 used. Phase workers leave
target tasks open for the lander and record baseline, fix, tests, and limitations on
their phase. Use `/sase_repo` before touching another repository, and
`sase artifact read` for artifacts. No live vault edits, live feed publishes, or
credential handling. Build bob through `$CARGO_TARGET_DIR`. Follow the daily release
train: no version or `CHANGELOG.md` edits. Run `just check`, never `just check-full`.
The lander must commit every changed repository, including the plans sidecar status
edit, with `/sase_git_commit` before proposing its own successor, and that plan
explicitly authorizes the skill for those obligations.

## Acceptance

- An unrelated panic under `run_bob` prints its normal message once and exits 101.
  Thread panics print. A closed-pipe print panic exits 0 with empty stderr. SIGPIPE
  disposition is unchanged.
- A usage error whose output reader is gone exits 2 quietly, and help into a closed pipe
  exits 0 quietly.
- New and existing tests pass, and `just check` exits 0.
- `bob-cli-bl` and epic `bob-cli-an.6.6.6.5.6.5.5.6` are closed normally with evidence,
  epic symbols are clean, the batch-9 plan file says `status: done`, and both
  repositories are committed.
- Batch 10 passes validation and is proposed through `sase plan propose`, with its
  reference recorded on the batch-9 epic bead.
