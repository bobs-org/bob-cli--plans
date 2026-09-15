---
tier: tale
title: Route human task-status retry progress to stdout
goal:
  Successful default task-status-hooks retries are captured by stdout-only cron logging
  without suppressing actionable stderr alerts or breaking JSON output.
size: small
proposed_by: bbugyi200.athena.0lc
create_time: 2026-09-15 12:23:21
status: wip
---

# Route human retry progress to stdout

## Goal

Make the routine timestamped retry decision and retry summary lines emitted by
`bob task-status-hooks` in its default human output mode go to stdout. This lets the Mac
cron entry shown in the report capture successful retry activity with its existing
`>> /var/tmp/bob_task_status_hooks.log` redirection instead of turning that activity
into local cron mail.

Preserve stderr for actionable diagnostics: command-line errors, human-mode terminal
failures, recovery-path failure details, and synchronization warnings must remain on
stderr. Preserve the machine-readable `--format json` contract as well: JSON mode must
continue to emit exactly one final JSON value on stdout, so its retry progress remains
on stderr rather than being interleaved with that value.

## Implementation

1. In `src/native/task_status_hooks.rs`, make the production retry logger aware of the
   selected `OutputFormat` when `run` constructs the `RetryEnv`. Configure its logging
   callback to print retry decisions and terminal retry summaries with `println!` for
   human output and retain `eprintln!` for JSON output. Do not change retry eligibility,
   timing, formatting, attempt counts, final result rendering, warning rendering, error
   rendering, or the injected logger used by unit tests.

2. Update the retry-focused integration coverage in `tests/cli.rs` to enforce the stream
   boundary without making timing more brittle:
   - Exercise a contended default/human run by reading its stdout while the process is
     live, release the held maintenance lock only after the first retry line appears,
     and verify stdout contains the decision, success summary, and final human report
     while stderr is empty.
   - Change the cron-style success test to use stdout-only append redirection
     (`>> logfile`, with no `2>&1`) in default/human mode. Verify the log receives the
     retry decision before lock release and ultimately contains the success summary and
     final result, while neither stream leaks back to the parent.
   - Retain or strengthen JSON retry assertions so the complete stdout remains parseable
     as the single final JSON object and retry progress remains on stderr. Keep
     fail-fast, exhausted-budget, dry-run, and non-retryable-error semantics covered.
   - Keep a failure-path assertion showing that stdout-only scheduler redirection does
     not swallow a real human-mode terminal error: it remains on stderr and the nonzero
     exit status is preserved.

3. Update `docs/task-status-hooks.md` to describe the format-sensitive stream contract:
   human retry progress and summaries use stdout; JSON retry progress uses stderr to
   protect the one-object stdout contract; warnings and human-mode terminal errors
   remain stderr. Explain that a default human cron invocation can append stdout alone,
   allowing genuine stderr diagnostics to continue triggering scheduler mail.

4. Update the `bob task-status-hooks` entry and surrounding explanation in
   `docs/vault-git-sync.md` to show `>> /var/tmp/bob_task_status_hooks.log` without
   `2>&1`. Leave the other scheduled commands' redirections unchanged, and explicitly
   distinguish routine task-status retry/result logging from stderr warnings and
   failures so the documented Mac setup matches the desired alerting behavior.

## Validation

1. Run the focused retry and cron integration tests, including the in-module
   retry-controller tests, with `cargo test task_status_hooks` and confirm the new
   stdout/stderr assertions pass.
2. Run `cargo fmt --check` and `cargo clippy --all-targets --all-features` to catch
   formatting or lint regressions in the stream-selection and test plumbing.
3. Run the complete `cargo test` suite to ensure the revised output contract does not
   regress other command behavior.
4. Re-read the two documentation sections and search for stale claims that
   `task-status-hooks` retry lines always use stderr or that its Mac cron entry
   redirects both streams.

## Acceptance criteria

- A default human `bob task-status-hooks` run that retries and later succeeds writes
  every retry line, its final retry summary, and its normal final report to stdout, with
  no stderr output when there are no warnings or errors.
- The reported Mac cron form using only `>> /var/tmp/bob_task_status_hooks.log` captures
  those successful-run lines and therefore does not generate cron mail merely because
  lock contention occurred.
- Human warnings and terminal failures still use stderr and retain their exit behavior,
  so stdout-only redirection does not hide actionable cron alerts.
- `--format json` continues to provide one parseable final JSON value on stdout; any
  retry progress for that mode remains on stderr.
- Retry policy, message contents, backoff behavior, vault writes, and dry-run behavior
  are otherwise unchanged.
