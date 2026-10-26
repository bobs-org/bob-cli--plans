---
tier: tale
title: Correct queued code-copy XCTest expectations
goal:
  Verify queued Return copies block one without a page and the YAML block with page
  seven, preserving exact payload and confirmation assertions.
size: small
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4
create_time: 2026-10-26 16:04:43
status: wip
---

- **PARENT:**
  [202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4.md)

# Correct the queued code-copy XCTest to match page-aware selection

Implement assigned phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4` (`mac-code-key-ci`)
from `plan:202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md`. The tracked
CI defect is `bob-cli-cr`; leave that task open for the parent epic's lander to verify
on the landed SHA. Close only the assigned phase after its work and local validation are
complete.

## Scope and established cause

This is one bounded test repair in the linked `bob-mac-capture` repository, with no
earlier unfinished implementation to divide into a child epic. At planning, the linked
checkout is clean at `591e4fe` and the assigned phase has no previous work notes beyond
this investigation. Re-read the phase before implementation to account for new evidence.
Use `/sase_repo` to open `bob-mac-capture` and read its `AGENTS.md` before accessing its
files.

The root cause was recorded on the assigned phase before any source edit:

- The approved `plan:202610/ref_code_block_copy.md` says under Initial selection: with a
  Highlights page, select the first containing block (then the following or preceding
  fallback); **without a page, select block 1**.
- `Sources/RefsCore/RefsCode.swift`, `RefsCodePicker.initialSelection`, implements that
  rule. `Tests/RefsCoreTests/RefsCodeTests.swift` already asserts index zero for a nil
  page. Commit `c7a5f37` did not change this rule.
- `c7a5f37` introduced both the failing integration test and
  `Tests/Fixtures/refs-code.json`. The fixture starts with makefile block 1 on page 2,
  whose exact text is `build:\n\tswift build\n\tswift test`; YAML block 2 is on page 7
  and has 14 lines.
- `testQueuedCodeKeyCopiesWhenCatalogReturns` in
  `Tests/BobMacCaptureTests/RefsPanelModelTests.swift` opens an ordinary library Copy
  card without supplying a Highlights Tabs page. It queues C and Return while an 80 ms
  catalog request is pending, but expects the page-7 YAML block.
- `Sources/RefsCore/RefsCopy.swift`, `RefsCopyCard.codeLoaded`, resolves the pending
  picker and replays queued Return against that picker's initial selection. The
  resulting makefile payload and `Copied makefile · 3 lines` confirmation therefore
  match the approved behavior.

The reported failures are deterministic across master CI runs `40365325905`,
`40376353485`, `40383389151`, and `40384861340`. On `591e4fe` this is the only failing
test in the 3147-test suite. The correction belongs in test expectations and scenario
setup. Preserve production selection behavior and fixture order.

## Implementation

1. Reconfirm the relevant source, fixture, and current phase notes. If the tree has
   advanced, inspect the current version before applying the same fix. Keep the recorded
   policy evidence with the phase; update it if new facts matter.
2. Correct the existing no-page queued-Return test to assert that the open card's
   `highlightedPage` is nil and that C enters `.codePicker` pending state. Keep Return
   before the response arrives, the existing bounded wait, and the 80 ms delay. Assert
   the exact makefile string, including literal tabs and no added final newline, plus
   the exact `Copied makefile · 3 lines` confirmation. Assert one pasteboard write and
   one completion; retain card closure verification. A clearer test name may explicitly
   identify the no-page case.
3. Preserve the original YAML intent in a separate queued-copy scenario with an explicit
   page-7 Highlights tab. Reuse the setup from
   `testCopyCardFreezesHighlightedPageFromCurrentTab`: `FakeTabsReader`,
   `tabsSnapshot(..., pageNumber: 7)`,
   `prepareForPresentation(source: .highlightsTabs)`, and the bounded wait for `.loaded`
   Tabs state. Open the card, assert its frozen page is 7, then queue C and Return while
   code is still loading. Retain the exact 14-line YAML payload and
   `Copied yaml · 14 lines` assertions from the original failing test, and assert one
   pasteboard write, one completion, and card closure. Keep each scenario's harness
   isolated and its waits bounded.

Do not replace payload checks with nonempty/contains assertions or compute the expected
selection through the production picker. Do not widen timeouts to mask the semantic
mismatch. The existing shared fixture intentionally tests both selection contexts and
should remain unchanged. No production Swift, CLI, documentation, release-version,
changelog, or memory change is needed.

## Validation

Follow the linked repository's compile rules: never land uncompiled Swift.

1. Run the full `just portable-check` in `bob-mac-capture` on the edited tree. This
   lint-checks every Swift file and builds/runs the portable targets; it does not
   type-check or execute `BobMacCaptureTests` on Linux.
2. Read `tailnet.md` through `/sase_memory_read` before remote work. Best effort, rsync
   a scratch copy of the opened checkout to SSH alias `mac`, excluding `.build` and
   `.git`; run `./Scripts/bundle.sh` followed by `just typecheck-tests`, then remove
   only that scratch copy. Never build in Bryan's checkout and never run `just install`.
   The Mac has Command Line Tools but no XCTest: the stub typecheck validates
   compilation, not runtime assertions. If the Mac is offline, record the attempted
   check and explicitly report macOS targets as unverified.
3. Run the full `just check` in the primary bob-cli checkout as required by the parent
   epic; never `just check-full`. Keep any builds under the configured
   `CARGO_TARGET_DIR`. Run required checks on the final tree, not just focused suites.
   Use `/sase_monitor` for long commands and preserve the phase and this plan's context
   through the monitor handoff; do not abandon running commands at the end of a provider
   turn.
4. Investigate any failure caused by this change. For an unrelated failure, reproduce on
   the clean base and record exact evidence plus any already tracking task as a
   `PROPOSED FOLLOW-UP:` note on this phase; an identical base failure does not keep the
   phase open. Do not create task beads.

The real runtime verdict for these macOS tests is master CI after the final commit
lands. The parent epic lander must inspect the run for the exact landed SHA (including
later steps after lint), verify these queued-Return cases and the full suite, and only
then close `bob-cli-cr`. Record this pending obligation explicitly; local portable tests
and stub typechecking do not prove it green.

## Completion and ownership

Record the baseline, test correction, test results, linked repository revision, and
Mac/CI limitations on `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4`. Discoveries outside
this repair belong only in `sase bead note <assigned-phase> 'PROPOSED FOLLOW-UP: ...'`
for the lander to triage. The parent's final `deploy_guard = refuse` decision is
unchanged and does not affect this test-only phase. No memory edits are authorized or
needed.

Before closing, run:

```bash
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4
```

No entries existed at planning. Resolve any new entries or re-key the relevant Justfile
line to a still-open parent/later phase before closing. Then close only this phase with
a verification note:

```bash
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.4 --note "<verified changes, check results, and pending landed-SHA macOS CI>"
```

Do not change status by hand, close `bob-cli-cr`, or close a parent/ancestor. Finalize
changed repositories through `/sase_final` as required by the host; the linked
repository is part of the final declaration. Do not use raw Git commits. Any manual
commit explicitly required by the host uses `/sase_git_commit`. Leave the parent lander
the final CI and task-closure evidence it needs.
