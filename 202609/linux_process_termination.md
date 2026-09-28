---
tier: tale
title: Restore Linux process termination test coverage
goal:
  Cancellation and timeout terminate Bob's child process, and the full Linux SwiftPM
  suite passes with meaningful lifecycle assertions.
size: medium
proposed_by: bbugyi200.athena.bob-cli-1u
bead: bob-cli-1u
create_time: 2026-09-28 06:38:42
status: wip
---

- **BEAD:**
  [bob-cli-1u](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-1u/README.md)

# Restore Linux process termination coverage in bob-mac-capture

Task bead: `bob-cli-1u`.

## Context

The linked `bob-mac-capture` repository exposes `CaptureCore` to Linux SwiftPM tests. On
current master (`9e61a77`),
`swift test --filter 'BobProcessClientTests/testCancellationTerminatesProcess|BobProcessClientTests/testRunTerminatesAndThrowsTimedOutWhenProcessOutlivesTheTimeout'`
fails both assertions that expect `Tests/Fixtures/fake-bob` to write its `TERM` marker.
The same tests pass in macOS CI. `BobProcessClient.run` calls `terminateIfRunning` on
cancellation and timeout; a Linux `strace` of the tests confirms
`kill(..., SIGTERM) = 0`. A separate direct Foundation `Process` experiment with the
same fixture writes the marker. The failure therefore needs investigation in the test
runner context before choosing a client or fixture change.

## Work

1. Reproduce the two failures and determine whether the child remains alive, exits
   without running the shell trap, or the marker check races with termination. Inspect
   Linux signal delivery and the fixture's `sleep & wait` behavior as needed. Preserve
   the distinction between an actual client failure and a fixture/platform assumption.
2. Apply the smallest sound fix in `bob-mac-capture`: fix the client if cancellation or
   timeout leaves its direct child running; otherwise make the fixture or assertions
   accurately observe termination on both Linux and macOS. Keep both tests active on
   Linux and verify timeout still throws `BobClientError.timedOut` and cancellation
   actually ends the process. Avoid a platform skip that would restore a green suite
   without process coverage.
3. Run the targeted lifecycle tests, then the full Linux `swift test` suite. Check the
   final diff for unrelated changes. If macOS execution is unavailable, report that
   limit and rely on the existing macOS CI behavior plus cross-platform code review; do
   not claim a fresh macOS run.
4. Close `bob-cli-1u` with
   `sase bead close bob-cli-1u --note "<what was fixed and verified>"` after successful
   verification. If investigation uncovers genuinely distinct out-of-scope work, follow
   `/sase_new_task` and identify `bob-cli-1u` in its evidence.

## Done when

Both lifecycle tests and the full Linux SwiftPM suite pass; the tests still establish
that cancellation and timeout terminate the direct child; the bead close note records
the observed result and any macOS verification limit.
