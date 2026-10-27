---
tier: tale
title: Verify the aliases completion deadline fix on Linux and macOS
goal:
  Complete bob-cli-da.1 with loaded aliases verification and macOS CI evidence while
  preserving bob-cli-b9's close gate.
size: medium
proposed_by: bbugyi200.athena.bob-cli-da.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-da.1
create_time: 2026-10-27 16:40:15
status: wip
---

- **PARENT:**
  [202610/backlog_batch_b2.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b2.md)
- **BEAD:**
  [bob-cli-da.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-da/bob-cli-da.1.md)

# Verify the aliases completion deadline fix on Linux and macOS

Complete assigned phase `bob-cli-da.1` of `plan:202610/backlog_batch_b2.md`. The
remaining work fits one coding agent: the test deadline fix has landed, and this phase
needs bounded verification, CI evidence, and an accurate handoff to the epic lander. No
previous implementation attempt or remaining-work note existed on this phase when
planning began.

## Scope and fixed decisions

The epic's final `b9_close_gate = ci_green` decision applies. A Linux pass alone does
not resolve the macOS flake. Preserve production completion deadlines and protocol
behavior. No memory, documentation, release-version, or other epic-phase changes belong
here.

The launch instruction says to close only `bob-cli-da.1`. Accordingly, report whether
`bob-cli-b9` meets its close gate, leave its closure to the parent epic lander, and
never close `bob-cli-da` or an ancestor plan bead. Do not create task beads or set
assigned-bead status by hand. Record discoveries as `PROPOSED FOLLOW-UP:` notes on
`bob-cli-da.1` for the lander to triage.

## Verified starting evidence

- The planning checkout was clean at `f00854be21c20a101dbe4404f98d178fef0f6740`. Its
  change to `tests/cli/aliases.rs::complete()` replaces `bob_command()` with
  `bob_completion_command()`.
- `complete_values()` calls `complete()`, asserts exit success, removes protocol
  directive lines, and collects candidate values. Empty stdout therefore becomes `[]`
  despite exit success. `tests/cli/support.rs` sets the shared test-only
  `BOB_COMPLETE_DEADLINE_MS=10000` explicitly on the child command.
- `src/native/completion/mod.rs::run_complete_inner()` waits for a worker via
  `recv_timeout`. Its production default is 150 ms, and expiration returns empty stdout
  with exit 0. The deadline covers tree construction and completion, not just vault
  providers. Worker panics and upstream errors can also produce empty results, so the
  historic failure is consistent with timeout but is not a proven macOS timeout without
  diagnostic evidence. `BOB_COMPLETE_DEBUG` already logs request, response, elapsed
  time, and timeout errors when configured.
- Audited `bob-cli-b9` note #1 and `bob-cli-d7.3` notes report two Linux `just check`
  passes and five loaded completion-module passes (580 test cases). Those are prior
  evidence, not fresh aliases-module measurements or macOS proof.
- GitHub CI run **40572791226**, attempt 1, checks exactly `f00854be`:
  <https://github.com/bobs-org/bob-cli/actions/runs/40572791226>. macOS job
  **122433733173** ran `cargo test --locked --no-fail-fast` and logged
  `test aliases::completion_offers_group_members_status_flags_and_alias_options ... ok`
  at `2026-10-27T20:05:20Z`. Its CLI suite finished with 2301 passed and 1 failed. The
  sole failure was `completion::zsh_adapter::real_zsh_first_tab_completes`, with
  `SETUP OK`, `FIRST-TAB OK`, `DASH-TAB OK`, then a readiness timeout,
  `FORMAT-TAB MISS`, and `PROMPT-READ-FAILED`. Thus the aliases node passed under the
  full macOS suite, but the macOS job and overall CI were red. Linux, lint, and package
  jobs passed.
- `bob-cli-8q` is closed and already covers historical bash/zsh PTY flakes;
  `bob-cli-d7.3` implemented the relevant changes. The observed macOS failure happened
  after the code fix but before 8q's recorded close. Do not describe it as a fresh
  post-close reproduction or automatically reopen 8q.
- `sase bead epic-symbols bob-cli-da.1` found no entries during planning.

## Work

1. Re-read `bob-cli-da.1` and `bob-cli-b9` with audited `sase bead read`, and read the
   parent design via `sase artifact read` if it is not in the invocation. Check the
   current SHA, working tree, notes, and the three completion helper functions above.
   Preserve the landed pin. A verification-only result is expected; do not manufacture
   another deadline edit. If the aliases test still fails, collect the original
   subprocess output and a fixture-local `BOB_COMPLETE_DEBUG` log before deciding
   whether a narrowly scoped test-helper repair is warranted. Do not add retries, skip
   assertions, or raise the production deadline to make failures disappear.

2. Run a fresh isolated aliases module and bounded loaded repetitions on Linux. Build
   the CLI test target once with
   `cargo test --locked --test cli --no-run --message-format=json`, selecting the
   executable from Cargo's `compiler-artifact` record for test target `cli`. Run
   `cargo test --locked --test cli aliases::` once, then five waves of four simultaneous
   invocations of that same compiled harness, each filtering `aliases::` with
   `--test-threads=8`. Constrain those stress children to two CPUs from the host's
   allowed affinity set where supported, so the load is reproducible without saturating
   the whole shared host. Keep each process's output and exit status separate and join
   every child. This exercises 20 instances of the affected test and, on the planning
   tree's 12-test aliases module, 240 total test cases under load. Recount from actual
   harness summaries if the module changes. Record SHA, affinity, concurrency, failures,
   and timings. Use `/sase_monitor` before starting commands expected to outlive the
   provider turn; preserve enough continuation context to finish this same plan.

3. Re-check GitHub evidence using `gh`, matching the tested SHA and run attempt. Start
   with run 40572791226 and macOS job 122433733173:

   ```sh
   gh run view 40572791226 --repo bobs-org/bob-cli --json headSha,conclusion,jobs,url
   gh run view 40572791226 --repo bobs-org/bob-cli --attempt 1 --job 122433733173 --log
   ```

   Look for a newer completed full-suite macOS job that includes the landed fix. Verify
   inclusion from local Git history; if another repository is required, open it through
   `/sase_repo` first. Check the exact aliases node, the suite invocation, and the macOS
   job conclusion, not just the workflow summary. A passing aliases node in a red job is
   useful evidence but does not satisfy this plan's conservative reading of `ci_green`.

   If no qualifying green macOS job exists, request one bounded rerun of the failed
   macOS job using
   `gh run rerun 40572791226 --repo bobs-org/bob-cli --job 122433733173`, provided it
   still tests the relevant unchanged helper code. If an equivalent run is already
   pending, use it instead. Follow the resulting attempt through `/sase_monitor` for a
   long CI wait. Do not rerun repeatedly until green or edit CI to omit the failing PTY
   test. Keep the original failure and the new attempt's result in the evidence. If
   source changes prove necessary, evidence must cover the new revision before calling
   the gate met; an old run cannot validate untested changes.

4. Report the outcomes precisely. If the loaded aliases runs pass and a full-suite macOS
   job containing the fix is green with the exact node passing, record that `bob-cli-b9`
   is ready for lander closure. Otherwise append a coordination note to b9 naming the
   missing gate, Linux evidence, exact passing/failing macOS nodes, SHA, attempt, URLs,
   and the run still needed. Leave b9 open in either case because this worker is
   authorized to close only its assigned phase.

   Record the already observed separate zsh failure on `bob-cli-da.1` as
   `PROPOSED FOLLOW-UP: macOS zsh PTY readiness failure persists after f00854be — run 40572791226 attempt 1, job 122433733173 failed completion::zsh_adapter::real_zsh_first_tab_completes at FORMAT-TAB while the aliases node passed; related closed bead bob-cli-8q and phase bob-cli-d7.3; this observation predates 8q closure.`
   Add any fresh rerun evidence accurately. Do not expand this phase into the separate
   PTY repair. Do not create beads.

5. If any tracked source changes were required, run `just check` on the final tree. For
   a verification-only outcome, the fresh targeted stress results plus the inspected
   full-suite CI evidence are sufficient; do not repeat unrelated local full-suite work
   just to create more results. If a check fails outside this change, compare against
   the clean base using permitted local checkout isolation and record identical failures
   as `PROPOSED FOLLOW-UP:` notes, citing an existing task when known. Such a
   base-identical failure does not leave this assigned phase open.

6. Re-run `sase bead epic-symbols bob-cli-da.1` immediately before closing. Resolve any
   entries or re-key them to a still-open owner. Then close only `bob-cli-da.1` with
   `sase bead close bob-cli-da.1 --note "<verified results>"`. The close note must
   distinguish the already-landed fix, fresh Linux evidence, macOS job status, whether
   b9's gate is met, and follow-up ownership. A missing macOS gate is an explicit
   coordination outcome allowed by the parent design: finish this verification phase
   with the evidence and leave b9 for the lander.

## Acceptance

- The existing 10000 ms test-only pin is verified, and the silent empty-output path is
  explained without asserting an unproven historic root cause.
- Fresh isolated and loaded aliases results are recorded with actual counts.
- macOS evidence names the SHA, attempt, full-suite command, exact aliases node, and job
  conclusion. At most one new rerun is requested; missing evidence is stated honestly
  and b9's gate is never waived.
- The separate PTY failure is preserved for lander triage. Only `bob-cli-da.1` closes,
  after the required epic-symbol check.
