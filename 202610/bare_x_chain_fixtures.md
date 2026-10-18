---
tier: tale
title: Correct bare-x Mac chain fixtures and finish bob-cli-9b landing
goal:
  Match Mac chain fixtures and completion ranges to current Bob, verify the correction,
  and close bob-cli-9b with its plan marked done.
size: small
proposed_by: bbugyi200.athena.bob-cli-9b.land--1
proposed_with: codex/gpt-6-astra
bead: bob-cli-9b
status: done
---

- **BEAD:**
  [bob-cli-9b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9b/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-9b.land--1--code](https://github.com/bobs-org/bob-cli--agents/blob/main/agents/bbugyi200.athena.bob-cli-9b.land--1--code/README.md)
- **COMMITS:**
  - [0f12e2b](https://github.com/bobs-org/bob-cli--plans/commit/0f12e2bc8aeb408fc5c1b060b98847cf9f5bd6a0)
    — docs(plan): mark bare-x closeout complete

# Correct bare-x Mac chain fixtures and finish bob-cli-9b landing

## Scope and decisions

Finish the remaining work for epic **bob-cli-9b**, then close that epic in this same
coding turn. This is a single-agent correction and closeout, not another multi-phase
epic. The original plan is `plan:202610/bare_x_close.md`.

The epic's final decisions remain binding: only lowercase bare `x` closes; whole-item
`=x`/`=X`/`=*`/`=!` are retired teaching errors; link/task `=x` suffixes remain
supported; only the two already-updated Capture/Bulk Capture glossary strands were
authorized for memory edits; no decision records may change. This correction requires no
memory edits and no grammar/API changes.

The land audit read all four closed phases and their notes, reviewed bob-cli commits
`c166ed2`, `6bc432b`, `0f47802` and Mac commit `2c007c7`, and reviewed intervening
reference-intake/Reading Board/Newest commits. Latest refreshed bob-cli base commits
`0511bee` and `2a11d05` change reference documentation, comments, and test data; they
need no bare-x integration edits. Mac origin/master was `2c007c7`. Recheck new drift
before landing; do not redo completed work without new evidence.

## Confirmed defect and evidence

Mac commit `2c007c7` respelled three parser fixtures' text without moving their byte
ranges. Comparing their complete JSON against current `bob capture-parse` found:

| Fixture under `Tests/Fixtures/`               | Draft      | Correct close span/item range | Correct following spans/item range        |
| --------------------------------------------- | ---------- | ----------------------------- | ----------------------------------------- |
| `pomodoro-start-drop-parse-chain.json`        | `x =~2`    | `[0,1]`                       | start `[2,3]`, drop `[3,5]`, item `[2,5]` |
| `pomodoro-start-named-parse-chain-query.json` | `x =#de`   | `[0,1]`                       | start `[2,3]`, name `[4,6]`, item `[2,6]` |
| `pomodoro-start-named-parse-chain.json`       | `x =#bugs` | `[0,1]`                       | start `[2,3]`, name `[4,8]`, item `[2,8]` |

The current fixtures still use `[0,2]` for the close and subsequent offsets one byte too
large. This is not a vault-context difference. The other **33** changed parser fixtures
exactly matched the current CLI.

The related `pomodoro-start-named-complete-chain-query.json` completion fixture still
has replacement `[5,7]` for `x =#de`; real
`bob capture-complete --cursor 6 -f json -- 'x =#de'` returns `[4,6]`. Two migrated
tests incorrectly preserve `[5,7]`:

- `Tests/CaptureCoreTests/BobProcessClientTests.swift`,
  `testCaptureCompleteStartNameChainDraftScopesToToken`.
- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`,
  `testNamedStartChainTokenRequestsStartNameCompletion`.

`Tests/Fixtures/fake-bob` serves these four fixtures for the corresponding drafts. The
existing portable tests pass because they assert the stale fixture values or only
inspect modes; the wrong completion range can hide a bad editor replacement.

## Implementation

1. Use `/sase_memory_read` for required bead/artifact memory, then read
   `sase bead read bob-cli-9b -r "Need landing evidence and final decisions"` and
   `sase artifact read plan:202610/bare_x_close.md "Need original epic closeout scope"`.
   Open `bob-mac-capture` and `plans` with `/sase_repo`; use only the printed paths, and
   read the linked Mac `AGENTS.md`. Do not assume the previous workspace path. Read any
   newly appended child/epic notes and review post-audit source drift.
2. Build the actual current bob binary. An inherited `CARGO_TARGET_DIR` can leave
   `target/debug/bob` stale: either explicitly build with
   `cargo build --bin bob --target-dir target` or resolve the real Cargo output path. Do
   not regenerate fixtures using the installed or stale binary.
3. Regenerate the three parser fixtures above from live `capture-parse -f json` output
   using each fixture's own input. Use a scratch vault and missing-config override so
   user configuration cannot alter the results. Preserve existing compact JSON
   formatting and newline style. Compare the full objects, not just `raw`.
4. Correct the completion fixture's replacement range using current `capture-complete`
   output. Preserve its intentional DEEP WORK/unnamed candidate context, or recreate the
   original fixture vault to regenerate the full response; do not replace those
   candidates with unrelated scratch-vault rows. Update both named Swift test
   expectations to `[4,6]` while keeping cursor 6. Inspect nearby chain mocks for the
   same prefix-width mistake, fixing only directly related mismatches.
5. Strengthen the existing relevant fixture tests so they verify that the close range
   selects exactly `x` and the subsequent ranges select the intended start/name/drop
   text. Extend the completion-chain test to apply the reported replacement to `x =#de`
   and verify `x =#deep-work` if existing helpers make this straightforward. Keep this
   bounded to these regressions; do not build a general fixture framework.

## Verification and known unrelated issue

The prior verification monitor `bdtdhceyyt3f` passed `just check` and Mac
`just portable-check` (1,369 tests). A fresh explicit bob build then passed:

- scratch-vault `capture --dry-run --no-clip -f json -- 'x1!2'`, asserting raw and
  selection, and `capture-parse -f json -- x2`, asserting schema 1 and one-byte span;
- all 17 retired/near-miss table messages from `docs/capture.md`, exactly against both
  execution and editor output;
- 16 uppercase/prose cases remaining tasks, preserved link/task `=x` suffixes, and
  byte-for-byte unchanged scratch vault after dry-runs.

After this correction:

1. Compare all **36** parser fixtures changed/added by `2c007c7` that contain an `input`
   field against current Bob, including the three start-chain filenames above, not only
   `pomodoro-close-parse-*`. Require full JSON equality unless a concrete intentional
   context difference is documented. Check the completion range against the live CLI as
   well.
2. Run `just portable-check` in the opened Mac repository. Run the host's file-change
   gate `just check` as required; never run `just check-full`. If the named SASE tool
   `check` is still unavailable, the prior audited fallback was
   `SASE_TOOL_BYPASS="check is absent from this project tool catalog" just check`. Use
   `/sase_monitor` for long commands, with continuation instructions explicitly
   preserving this tale's responsibility to finish epic closeout.
3. Because `Tests/BobMacCaptureTests` changes, follow the Mac `AGENTS.md`: best-effort
   compile a scratch copy on the mac with `bundle.sh` and `just typecheck-tests`, never
   Bryan's checkout or an install. Use tailnet memory before connecting. Record exactly
   what compiled or was unavailable; do not claim macOS XCTest ran. The original phase
   recorded a successful scratch bundle/typecheck, and CI on `2c007c7` passed lint/build
   but Test compilation failed on `RefsNewestTests.zone` conflicting with
   `NSObject.zone()`. Blame identifies `5a0bff5`/`bob-cli-9d.2`, an unrelated active
   epic. This has already been triaged through `/sase_new_task` and recorded on
   **bob-cli-9d**; do not fix it here or file a duplicate. If it still blocks broad Mac
   checks, report that existing limitation while verifying the changed code as far as
   possible.
4. Review diffs, ensure fixture/test changes are limited to the verified regression, and
   confirm all expected checks are complete before closing.

## Final step: close bob-cli-9b and finish its plan

This tale has no land agent and nothing resumes the original landing automatically. The
coder must perform these closeout actions in its own turn after verification, without
waiting for its own commit SHA, push, or CI run. The host commits only after the final
declaration. No successful close requires `--force`.

1. Recheck `bob-cli-9b` descendants, every new note, its linked plan readiness, and
   post-audit drift. If the generated tale has an open descendant bead, finish and close
   that verified work normally before closing the containing epic. Do not close
   unfinished descendants, sweep them, or use force to make the command pass.
2. Run **`sase bead epic-symbols bob-cli-9b`**. The last audit found none. For every
   entry now returned, resolve it (wire, privatize, legitimate non-test pragma, or
   delete under policy), or re-key only to a still-open later bead that truly needs it.
   Resolve all entries keyed to this epic or its phases before closing.
3. Close normally with **`sase bead close bob-cli-9b --note "<verification>"`**. The
   note must include the original source/commit and integration audit, new fixture
   corrections and final verification outcomes, macOS validation limits, and EVERY
   follow-up outcome: **9b.2 note #1** exact-error docs proposal was fulfilled by
   **9b.3**, confirmed against all 17 real table messages, so a separate task was
   declined as already completed in-epic handoff work; no other child proposals;
   unrelated macOS CI **39164361288** / `RefsNewestTests.zone` was attached via
   `/sase_new_task` to active **bob-cli-9d**, no new duplicate task. This fixture
   correction was epic-caused remaining work, now completed, not a follow-up task. If
   any unfinished-work rejection occurs, finish the named work instead of forcing.
4. After close, run **`just symvision` if available**; the last bob-cli justfile had no
   such recipe. Record unavailability rather than claim a pass. Update
   **`status: done`** in the YAML frontmatter of **`plan:202610/bare_x_close.md`**, at
   `202610/bare_x_close.md` under the opened plans repository. Do not rewrite the
   original approved plan content.
5. Run **`sase bead read bob-cli-9b -r "Need the parent link"`** after closing. It
   currently has **no parent_bead**, so no ancestor close is expected. If that link
   changed meanwhile, inspect it: a phase parent closes only after its scope is verified
   and its containing epic remains with its land agent; a plan parent needs its previous
   landing note, all descendant notes, linked plan readiness and drift rechecked before
   symbols cleanup, normal close, symvision, and plan done. Repeat directly parented
   complete plan ancestors only. On ambiguity or incompleteness, record the blocker on
   that parent and report it; never force a successful landing.
6. Use `/sase_final` as the last action before the final response, declaring commits for
   every changed repository, including the opened Mac and plans repos. Report the
   completed epic and any remaining unrelated Mac test limitation accurately.
