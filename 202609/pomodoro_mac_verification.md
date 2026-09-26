---
tier: epic
title: Finish macOS verification of atomic Pomodoro capture
goal:
  Bob Mac Capture compiles and passes its tests with the additive Pomodoro start
  contract, and the panel presents the resolved session accessibly.
parent_bead: bob-cli-26
phases:
  - id: fix-swift-decoder
    title: Repair Pomodoro diagnostic range decoding
    depends_on: []
    size: small
    description:
      "fix-swift-decoder: make additive diagnostic ranges compile and decode safely on
      macOS."
  - id: verify-mac-capture
    title: Run Mac capture suite and panel checks
    depends_on:
      - fix-swift-decoder
    size: medium
    description:
      "verify-mac-capture: run Swift tests and check the new session preview and
      accessibility behavior."
proposed_by: bbugyi200.apollo.bob-cli-26.land
create_time: 2026-09-26 17:47:02
status: wip
---

- **PROMPT:**
  [prompts/202609/pomodoro_mac_verification.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/pomodoro_mac_verification.md)
- **PARENT:**
  [202609/capture_start_pomodoro.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/capture_start_pomodoro.md)

# Finish macOS verification of atomic Pomodoro capture

## Context

Epic `bob-cli-26` is awaiting landing. Its three phases are closed. The Rust
implementation and editor protocol were reviewed, and the seven focused
`capture_pomodoro_start` integration tests pass. The Mac phase added `pomodoro_start`
decoding, presentation, fixture coverage, and tests in the linked `bob-mac-capture`
repository, but its worker had no macOS runner.

The macOS host `mac` is reachable and has Apple Swift 6.3.2. Running `swift test`
against `bob-mac-capture` commit `7fae3fd` fails to compile
`Sources/CaptureCore/CaptureModels.swift` at lines 252 and 256: `try? decodeIfPresent`
already unwraps the optional in `if let`, so the second `let object` / `let pair`
bindings have nonoptional inputs. The errors are in the new diagnostic-range
compatibility decoder introduced by this epic. Swift's existing `FileManager` Sendable
warnings are unrelated.

The configured linked-repo open currently fails because its primary workspace directory
is absent. `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"` succeeds and
provides the audited checkout path; use that path for source reads and edits. A
temporary copy on `mac` was sufficient to reproduce the compile error. The other
proposed follow-up, repo-wide rustfmt drift from phase `bob-cli-26.2`, duplicates ready
task `bob-cli-24`; the land agent recorded an independent `+1` there.

## Phase: fix-swift-decoder

slug: fix-swift-decoder size: small depends_on: []

1. Open `bob-mac-capture` through `/sase_repo` and read its local instructions. Fix the
   diagnostic-range decoder with correct handling for object ranges, two-element array
   ranges, absent ranges, and malformed ranges. Preserve tolerance for older Bob
   responses and the version-one JSON contract. Keep the change in `CaptureCore`; do not
   add a Swift clock or vault writer.
2. Run a macOS Swift build or a focused test target to confirm the decoder compiles. Add
   a focused regression test only if the corrected branch is not already covered by the
   existing tests.

## Phase: verify-mac-capture

slug: verify-mac-capture size: medium depends_on: [fix-swift-decoder]

1. Run `swift test` on a macOS runner against the revised checkout. Fix any further
   failures caused by the atomic-start changes. Exercise the new parse, completion,
   process-client, panel, notification, and presentation tests.
2. Inspect the panel's session preview and accessibility label on macOS if a GUI session
   is available. Verify start/end, duration, selected or created destination, completion
   preservation around `=`, and actionable error display using the fake Bob fixture or a
   safe dry run. If a GUI session is unavailable, document the precise limitation and
   retain automated panel/a11y assertions.
3. Record test results and any remaining limitations on this phase bead. The parent
   `bob-cli-26` land agent will resume its closeout after this child lands.

## Done when

- `swift test` succeeds on macOS for the updated linked repository.
- The new `pomodoro_start` diagnostic decoder compiles and handles object, pair, absent,
  and malformed ranges without crashing or changing existing capture JSON behavior.
- The panel preview and accessibility checks have an observed result or a documented
  host limitation, with automated assertions passing.
