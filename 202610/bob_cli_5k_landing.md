---
tier: tale
title: Finish and land epic bob-cli-5k
goal:
  "Finish the epic work bob-cli-5k's phases reported done but left incomplete: the
  bob-plugins sync rule in AGENTS.md, the Tasks sandbox test and error gaps, a clippy
  ban that actually fails the gate, a correct nested TestEnvGuard restore, exact
  bob-plugins origin matching, and a complete artifact-link cutover. Then verify on
  athena and close epic bob-cli-5k."
size: medium
bead: bob-cli-5k
proposed_by: bbugyi200.athena.bob-cli-5k.land
create_time: 2026-10-07 21:43:08
status: wip
---

- **PARENT:**
  [202610/close_top_ten_impact_beads.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/close_top_ten_impact_beads.md)
- **BEAD:**
  [bob-cli-5k](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5k/README.md)

# Plan: Finish and land epic bob-cli-5k

## 1. Context

Epic bob-cli-5k (plan `plan:202610/close_top_ten_impact_beads.md`) closed the ten
highest-impact bob-cli task beads. All seven phases and all thirteen audited beads are
closed. Its land agent re-verified the final master tree (`74c2afc`, equal to
`origin/master`) on athena; see epic bead note #3 (`sase bead read bob-cli-5k -r ...`).

- `just check` passed twice in a row: 1945 lib tests, 1180 CLI tests, and every other
  binary.
- The 4j, 4u and 5i exact tests pass.
- `cargo test --lib` passed 5 runs in a row.
- The live read paths are fast: `bob freshness list -f json -l 1` takes ~1.3 s and
  `bob plan -f json` ~0.48 s.
- `bob ref doctor` reports `annotations: ok` and `coverage: ok`.
- Follow-up triage is finished and recorded in epic note #2. Do not re-triage it.

The audit also found the six gaps below. Each was caused by this epic: the phase that
owned it reported done, but the work is incomplete. This tale finishes them,
re-verifies, and closes the epic. Do not run `just check-full`. `just check` is the
file-change gate.

## 2. Land the bob-plugins AGENTS.md sync rule (phase bob-cli-5k.6 gap)

Phase bob-cli-5k.6 committed the AGENTS.md rule to bob-plugins as `856afc9`, but that
commit never reached bob-plugins `origin/master`.

- Open the linked repo with `/sase_repo`:
  `sase repo open bob-plugins -r "Land the bare-sync AGENTS.md rule for bob-cli-5k"`.
  Read its `AGENTS.md`.
- Replace the final paragraph with the text below. It is the exact text of `856afc9`.
  Then commit it in that repo, with message
  `docs: bare bob plugins sync refuses from a SASE worktree`.

  ```text
  After any changes to files in this repo, you MUST run `bob plugins sync` to sync the
  repo changes to the vault. From a SASE worktree (or any checkout other than the
  resolved plugins repo), a bare `bob plugins sync` refuses before any pull or copy:
  pass the worktree explicitly with `bob plugins sync --repo <worktree>`, adding
  `-p <id>` to move one plugin (for example
  `bob plugins sync --repo <worktree> -p bob-ledger-tools`).
  ```

- This change touches docs only, so no plugin needs deploying. Do not run a bare
  `bob plugins sync`.

## 3. Tasks JS sandbox gaps (phase bob-cli-5k.5)

Files: `src/native/dataview/tasks/js.rs` and `src/native/dataview/tasks/mod.rs`. Plan
§7.2–7.3 of the epic required three things that commit `577866d` did not deliver.

The current shape:

- `EXPRESSION_TIMEOUT` (2 s), `INIT_BASE_TIMEOUT`, `INIT_PER_TASK`, `INIT_MAX_TIMEOUT`,
  and `init_timeout(task_count)` are constants and a function near the top of `js.rs`.
- `JsSandbox::new(tasks, query_context, now)` arms one shared
  `Deadline = Arc<Mutex<Instant>>` for `init_timeout(tasks.len())`. It runs the
  initialization units through
  `eval_init_unit(context, deadline, source, action, budget)`, then calls
  `arm_deadline`.
- `arm_deadline` always uses `EXPRESSION_TIMEOUT`, and `eval_unit` re-arms it for every
  user expression.
- `mod.rs` builds sandboxes through `maybe_sandbox` (`query.uses_javascript()`). Three
  entry points use it: `query_matching_descriptions`, `query_rich_tasks` and `run`.
  `run_note` builds one sandbox for the first block that needs JavaScript.

Changes:

1. **Injectable budgets.**
   - Add a small budgets value (init and expression `Duration`s).
   - `JsSandbox::new` keeps its signature and production defaults:
     `init_timeout(tasks.len())` and `EXPRESSION_TIMEOUT`.
   - It delegates to one constructor that takes the budgets. Expose that constructor to
     tests (`#[cfg(test)]` or `pub(super)`).
   - Store the expression budget on the sandbox, and make `arm_deadline` / `eval_unit`
     use it instead of the constant.
2. **Init timeouts say they timed out.** Today a genuine init timeout surfaces as
   `JavaScript error while initializing the Tasks JavaScript sandbox: Error: interrupted`.
   - When an init unit fails after its deadline has passed, return a message that says
     it timed out. It should name the budget and the task count, for example
     `timed out after 10.4s initializing the Tasks JavaScript sandbox (1,280 tasks)`.
   - Keep the generic message for real JavaScript errors.
   - If `eval_unit` can share the same check cheaply, give user-expression timeouts the
     same treatment (`timed out after 2s while evaluating ...`). Keep the 2 s budget
     itself.
3. **Construction counter.**
   - Add a `#[cfg(test)]` thread-local counter that the sandbox constructor increments,
     plus a test-only reader.
   - It must be thread-local so parallel tests cannot see each other's constructions.
     Confirm construction happens on the calling thread.
4. **Regression tests** (in `js.rs` / `mod.rs` test modules):
   - **Zero sandboxes without JavaScript.** Each of the four entry points builds zero
     sandboxes for a query with no `by function`, for example `not done`. Use the
     fixture vault `tests/fixtures/tasks_parity/vault`. For `run_note`, write a temp
     vault note with only plain `tasks` fences. A `filter by function ...` query builds
     exactly one sandbox. A `run_note` note with one plain block and one by-function
     block also builds exactly one.
   - **Tiny expression budget.** Build a sandbox over the fixture tasks with an
     expression budget of `Duration::ZERO` and the normal init budget. Construction must
     succeed, which proves the budgets are separate. A busy-loop function filter
     evaluated through it (for example
     `filter by function (() => { while (true) {} })()`) must then fail with the
     expression timeout. Do not rely on a short expression: QuickJS only polls the
     interrupt handler periodically.
   - **Tiny init budget.** An init budget of `Duration::ZERO` makes construction fail
     with the new timed-out message, which names initializing.
   - Keep the existing `init_timeout` scaling tests and
     `non_function_queries_skip_the_sandbox_on_a_real_vault`.

## 4. Make the set_var / remove_var ban fail the gate (phase bob-cli-5k.2 gap)

`clippy.toml` lists `std::env::set_var` and `std::env::remove_var` as
`disallowed-methods`. That lint is warn-level, though, and `just check` runs
`cargo clippy --all-targets --all-features` without `-D`. So a new `set_var` passes the
gate as a warning among dozens.

- Add this to `Cargo.toml`, so every clippy run (`just lint`, `just check`, editors)
  denies the lint:

  ```toml
  [lints.clippy]
  disallowed_methods = "deny"
  ```

- The one allowed write, the `TZ` pin in `pin_tz_utc0_for_test` in `src/native/env.rs`,
  keeps its `#[allow(clippy::disallowed_methods)]`.
- Update the `clippy.toml` comment to say the ban is denied through `Cargo.toml`.
- Prove it, without committing the probe:
  1. Add a temporary `std::env::set_var` call in a lib test.
  2. Confirm `cargo clippy --all-targets --all-features` exits non-zero naming
     `disallowed_methods`.
  3. Remove the probe and confirm clippy exits 0.

## 5. Fix TestEnvGuard nested restore (phase bob-cli-5k.2 latent bug)

In `src/native/env.rs`, `TestEnvGuard::set` saves
`overrides.get(key).cloned().unwrap_or_else(|| env::var_os(key))` as an
`Option<OsString>`. That loses the difference between "override = unset" and "no
override".

- **Inner guard over an outer unset.** Suppose an outer guard unsets `K` and an inner
  guard then sets it. When the inner guard drops, it removes the override entirely, so
  the real process value of `K` leaks back in.
- **Outer guard with no prior override.** When it drops, it inserts the real value as a
  stale override instead of removing its entry, and `inherit_overrides` /
  `snapshot_overrides` then forward that entry.

Fix:

- Save the previous map entry exactly, as `Option<Option<OsString>>`, where `None` means
  there was no override.
- On drop, restore in reverse order: insert the previous entry when there was one,
  otherwise remove the key.
- Add unit tests. Use a variable that is always set in the test process (`PATH`):
  - With an outer unset and an inner set, `var_os` is `None` again after the inner drop.
  - After the outer drop, the real value is back and `snapshot_overrides()` has no entry
    for the key.
  - A lone guard leaves no entry behind.

## 6. Match the bob-plugins origin exactly (phase bob-cli-5k.6)

`origin_points_at_bob_plugins` in `src/native/plugins/guard.rs` uses
`stdout.contains("bobs-org/bob-plugins")`. It therefore also treats checkouts like
`bobs-org/bob-plugins--research` or `bob-plugins-fork` as bob-plugins. A bare sync run
from one of those is refused, with a `--repo` suggestion that names the wrong repo.

- Add a matcher for the remote URL:
  - trim it, strip one trailing `/`, and strip a `.git` suffix;
  - split on `/` and `:`;
  - require the last two segments to be `bobs-org` and `bob-plugins`, compared
    ASCII-case-insensitively.
- It must accept `git@github.com:bobs-org/bob-plugins.git`,
  `https://github.com/bobs-org/bob-plugins`,
  `https://github.com/bobs-org/bob-plugins.git/`,
  `ssh://git@github.com/bobs-org/bob-plugins.git`, and a local path ending
  `/bobs-org/bob-plugins`.
- It must reject `git@github.com:bobs-org/bob-plugins--research.git`,
  `https://github.com/bobs-org/bob-plugins-fork` and
  `https://github.com/someone/bob-plugins`.
- Unit-test it in `guard.rs`. Keep the `plugins/` + `package.json` marker fallback
  unchanged.
- In `tests/cli/plugins.rs`:
  - Tighten `plugins_sync_bare_from_foreign_checkout_refuses_before_pull_or_copy` so it
    asserts the exact suggested command,
    `bob plugins sync --repo <canonical foreign toplevel> [-p <id>]`, rather than only
    `--repo`.
  - Add a test: from a checkout whose origin is
    `git@github.com:bobs-org/bob-plugins--research.git`, and which lacks the marker, a
    bare sync is not refused. Follow `plugins_sync_bare_from_unrelated_cwd_is_allowed`,
    using a temp vault and temp repos only.
- Never run a bare sync against the live vault.

## 7. Complete the artifact-link cutover (phase bob-cli-5k.4, bob-cli-21 criterion)

`sase artifact doctor` exits 1 with
`artifact-link cutover is incomplete; resume with sase artifact link import-indexes --apply fleet-capable-65bfe92269a7. Affected roles: plans, research. artifact-link cutover markers are partial`.
Every other row is clean:

- no link event validation or reduction errors;
- no dangling refs or orphaned indexes;
- the pending gap is down from +9505 to +504.

Phase bob-cli-5k.4 had applied this same cutover, yet its markers are partial again.
Typed link writes do work: `sase artifact link add` succeeded five times during the
landing.

1. Run `sase artifact link import-indexes`. The preview writes nothing. Confirm it names
   roles `plans` and `research`, and note its attestation token. On 2026-10-07 it
   printed `fleet-capable-65bfe92269a7`, 59 unique rows and fleet machines apollo and
   athena.
2. Run `sase artifact link import-indexes --apply <token>` with that token.
3. Rerun `sase artifact doctor`. Record its exit status and every non-`none` row,
   excluding the informational missing-source list.
4. Confirm `sase artifact link list bead:bob-cli-5l` still lists its related links to
   bob-cli-o and bob-cli-5k.2.
5. If the doctor still exits 1:
   - If the cutover markers are partial again, or the remaining pending backlog does not
     drain, use `/sase_new_task` to file or corroborate it as a sase issue. Its details
     must identify the proposing bead: bob-cli-5k.4 note #3 for the backlog.
   - Record the outcome in the close note.

   The epic still closes in this case. The store accepting writes is the epic goal, and
   plan §6.2.4 asks only that the gap be "reduced or explained".

6. Never run `prune --apply`, `reclaim --apply`, or a trash purge, and never hand-edit
   sidecar objects.

## 8. Verify on athena

- `cargo fmt --check` is clean.
- The clippy probe from §4 behaves as described, and the final
  `cargo clippy --all-targets --all-features` exits 0.
- `rg 'env::(set_var|remove_var)' src` matches only the `TZ` pin in `src/native/env.rs`
  (plus its doc comment).
- The targeted suites pass:
  - `cargo test --lib native::dataview::tasks`
  - `cargo test --lib native::env`
  - `cargo test --lib native::plugins`
  - `cargo test --test tasks_parity`
  - `cargo test --test cli plugins::`
- `just check` passes twice in a row at default parallelism. Record the test counts.
  - Some known load flakes are tracked elsewhere: the bob-cli-3j completion timing and
    bash readline tests, bob-cli-5o
    `capture_with_kick_returns_before_the_clip_finishes`, and bob-cli-5l lib ETXTBSY.
  - If one of those fails, confirm it passes in isolation and corroborate it through
    `/sase_new_task`. Then keep running until two consecutive `just check` passes.
  - Any other failure is yours to fix.

## 9. Close out epic bob-cli-5k (final step)

1. Run `sase bead epic-symbols bob-cli-5k`. The landing audit found no entries.
   - For any entry now listed, resolve the symbol: wire it up, privatize it, add a
     non-test pragma, or delete it per the Symvision epic-whitelist policy.
   - Re-key its Justfile line only if a still-open later bead needs the exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-5k --note "<verification>"
   ```

   The note must cover:
   - the landing audit facts from epic note #3: 13 beads closed; just check 2x green; 5
     lib runs; live timings `bob freshness` ~1.3 s and `bob plan` ~0.48 s; ref doctor
     annotations ok and coverage ok; walk-identity npm tests pass; integration clean;
   - the follow-up triage outcomes from epic note #2: bob-cli-5l, 5m, 5n and 5o filed;
     bob-cli-3j note #8; +1 on bob-cli-3w; the DISPLAY scrub and 3cbef27 declined as
     resolved; the outbox residue declined;
   - what this tale fixed, with the commit;
   - the bob-plugins AGENTS.md commit;
   - the final `sase artifact doctor` status;
   - the two `just check` runs, with counts.

   Never use `--force`. If the close is rejected, fix the cause and close again.

3. Run `just symvision` if `just --list` shows that recipe. It did not exist at the
   landing audit; say so in your final response if it still does not.
4. Set `status: done` in the frontmatter of the epic's plan file, currently
   `status: wip`. That is the PLAN path shown by `sase bead read bob-cli-5k -r ...`
   (`plan:202610/close_top_ten_impact_beads.md` in the plans sidecar).
5. bob-cli-5k has no `parent_bead`, so there is no parent to close. Finish normally.
