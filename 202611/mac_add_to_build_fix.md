---
tier: tale
title: Fix the bob-mac-capture install break from Record Notes' .addTo action
goal:
  bob-mac-capture builds and installs again, and epic bob-cli-ir records the cause, fix,
  and proof.
size: medium
decisions:
  record_row_gaps:
    ask:
      Also fix the two latent record-row Add to it defects (native-editor range, + key)
      found here?
    choices:
      note:
        Mac fix stays three compile lines; both defects go on bob-cli-ir for its live
        lander
      fix:
        Also fix the native Add to it range and the + key, with CI-only Mac regression
        tests
    default: note
    why:
      Keeps the unblock fast and conflict-free; the live Record Notes lander owns the
      feature
    answer: note
  install_excerpt:
    ask: Make install-all's failure excerpt start at the first real compiler error?
    choices:
      fix:
        Change live.py _failure_excerpt plus unit tests, verified by just check-scripts
      task: No bob-cli change now; file a bug task bead through /sase_new_task
    default: fix
    why:
      Small, precise fix for why the reported output showed only warnings, not the
      errors
    answer: fix
proposed_by: bbugyi200.athena.1al
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-09 14:23:18
status: wip
---

# Fix the bob-mac-capture install break from Record Notes' `.addTo` action

## Outcome

`just install` / `just install-all` builds bob-mac-capture again, master's exact-SHA
macOS CI gets past Build, and the epic that caused the break (`bob-cli-ir`, Record
Notes) carries a note recording the root cause, the fix, and its verification.

This is a single-agent fix. It does not close any bead.

## Diagnosis (established; do not re-derive)

- **Real failure.** The Mac install log
  (`~/.local/state/bob-cli/install-all/logs/20261109-135435-7872-bob-mac-capture.log` on
  the `mac` host, 838 lines) contains exactly three `error:` lines, all
  `switch must be exhaustive` with `note: add missing case: '.addTo'`:
  - `Sources/BobMacCapture/CapturePanelModel.swift`,
    `completeSimilarTaskIDAssignment(prompt:success:action:itemIndex:)`, the
    `switch action` that picks a task-ID replacement.
  - `Sources/BobMacCapture/CapturePanelView.swift`, `linkInsertsLine(for:)`, the inner
    `switch action` under `case .similarTask(...)`.
  - `Sources/BobMacCapture/CapturePanelView.swift`,
    `similarTaskInsertsText(route:typed:action:)`.
- **The Sendable/actor-isolation lines in the reported output are warnings only** (Swift
  5 language mode). They predate this break and are out of scope.
- **Cause.** Mac commit `a5152aa` ("feat(mac-capture): show record updates,
  existing-record rows, and Similar records", phase `bob-cli-ir.12` of epic
  `bob-cli-ir`) added `case addTo` to `CaptureRelatedAction` in
  `Sources/CaptureCore/CaptureRelatedModels.swift`. It did not adapt the three
  exhaustive switches in the `BobMacCapture` target.
- **Why it shipped.** `bob-cli-ir.12` note #1 says "swift build + swift test green" and
  "Mac typecheck/macos-ci not run from Linux". `just portable-check` cannot type-check
  `Sources/BobMacCapture*`. The phase closed (18:12Z) without the scratch Mac compile or
  `Scripts/macos-ci-check` that bob-mac-capture's `AGENTS.md` requires. Its exact-SHA CI
  run 42672189863 went red at Build. Child commit `ef5dc4a` (bob-cli-j6.2) inherited the
  break: run 42673185238 is also red at Build, so its Test, Bundle, and install gates
  never ran.
- **Why the reported output hid the errors.** bob-cli's
  `scripts/lib/install_ui/live.py::_failure_excerpt` starts its 40-line excerpt at the
  first line whose lowercase text contains `error`. Every Swift 6 warning ends with
  "this is an error in the Swift 6 language mode", so the excerpt began at the first
  Sendable warning (log line 7). The real errors at log lines 496–558 were never shown.
- **`.addTo` can never reach the Add block ID prompt.** bob-cli's
  `src/native/capture_related/records.rs` emits record rows with
  `requires_block_id: None`, which decodes to `requiresBlockID == false`.
  `CapturePanelModel.similarApply` and the pending-action path present the similar-task
  ID prompt only when `row.match.requiresBlockID` is true. The three new cases are
  therefore pure compile adaptations. They must not synthesize a task-ID replacement for
  a record splice.
- **Already recorded.** `bob-cli-ir` note #6 (by `bob-cli-j6.land`) records this exact
  CI failure as a DISCOVERED ISSUE. This tale resolves it.

## Coordination with live agents (check first)

Two agents may touch the same lines while this tale runs:

- The `bob-cli-j6` closeout tale (`plan:202611/modifier_alias_landing.md`, §3) is
  verifying an **uncommitted** minimal `.addTo` adaptation with exactly the text in step
  1 below. It will commit it with its other Mac alias fixes. Its plan says to
  "incorporate an existing Record Notes fix ... if present".
- `bob-cli-ir.land`, the Record Notes lander, is running its landing audit and may
  propose a closeout tale that covers note #6.

Before editing, open bob-mac-capture through `/sase_repo` (it fast-forwards to
`origin/master`) and read its `AGENTS.md`. Then:

1. If `origin/master` already handles `.addTo` in all three switches, skip step 1 and go
   to verification. Record which commit fixed it, then watch that SHA's CI to green with
   `/sase_monitor` (or cite its completed run).
2. Re-read `sase bead read bob-cli-ir -r "<why>"` and
   `sase bead read bob-cli-j6 -r "<why>"` for notes newer than bob-cli-ir #6 and
   bob-cli-j6 #2. Honour any newer ownership claim. Under `record_row_gaps = note`, do
   not write a DISCOVERED ISSUE that a newer note already records; add independent
   evidence instead.

## 1. Compile fix in bob-mac-capture (always, unless already landed)

Make exactly these three edits. They match the j6 closeout's pending edit byte-for-byte,
so its later commit rebases cleanly. Keep 4-space indentation and change only these
lines:

- `CapturePanelModel.completeSimilarTaskIDAssignment`: change
  `        case .open, .discard, .unknown:` to
  `        case .addTo, .open, .discard, .unknown:`. The existing `replacement = nil`
  branch then reports the error and never splices.
- `CapturePanelView.linkInsertsLine(for:)`, inside `case .similarTask(...)`: insert
  these lines immediately before `            case .open, .discard, .unknown:`

  ```swift
              case .addTo:
                  return "Updates record text"
  ```

- `CapturePanelView.similarTaskInsertsText(route:typed:action:)`: insert these lines
  immediately before `        case .open, .discard, .unknown:`

  ```swift
          case .addTo:
              return Text("Updates record text")
  ```

Then grep the `BobMacCapture` and `BobMacCaptureTests` targets for any other exhaustive
`switch` over `CaptureRelatedAction` without a `default`. Diagnosis found none, but
confirm. Do not touch `SimilarTasksBandEdits` or the CaptureCore record-splice behavior.

## 2. Record-row "Add to it" defects (decision `record_row_gaps`)

Both defects are source-verified, not reproduced. Neither affects the build.

- **D1, native-editor range.** In `CapturePanelModel.applySimilarActionFresh`,
  `editReplacementStart`, `editReplacementEnd`, and `nativeReplacementText` fall through
  to their `default` branches for `.addTo`. The `CaptureNativeRewriteEdit` therefore
  targets a zero-length range at `item.markerAt` with an "inserted slice" guessed from
  the length difference. Bob's `add_to` instead replaces the explicit byte range
  `match.addTo.start..<match.addTo.end`. Example from
  `CaptureRecordRevisitsTests.testAddToSplicesBobsExplicitRange`: `Ninja Sushi @eat/`
  with `add_to {0,17,"@eat/ninja_sushi_bar"}` makes the native bridge insert `bar` at
  the end. That fails the `resultingDraft` check after the `NSTextView` was already
  mutated, so the model falls back to `setPlainDraft`. The final text is right, but the
  editor's undo stack holds a bogus step.
- **D2, `+` key.** `CaptureKeyCommandRouter` maps a focused-band `+` to
  `.similarApply(.note)`. Record rows offer only `[.addTo, .open, .discard]`, so
  `similarApply(.note)` silently returns. Yet the strip (`SimilarTasksBandView`,
  `SimilarTasksBandEdits.sigil(for: .addTo) == "+"`) and the docs table in
  `docs/capture-previews.md` advertise `+` for Add to it. Return still works through
  `defaultAction`.

> [!decision] record_row_gaps = note Make no change for D1/D2. In step 4, append one
> `DISCOVERED ISSUE:` note on `bob-cli-ir` that describes D1 and D2 with the
> file/function names, the example above, and the fix sketch from the `fix` branch. Say
> they are source-verified only.

> [!decision] record_row_gaps = fix
>
> - D1: in `CapturePanelModel`, give `editReplacementStart` and `editReplacementEnd` a
>   `case .addTo` that returns `match.addTo?.start` / `match.addTo?.end` (falling back
>   to the current marker behavior when nil). Give `nativeReplacementText` the `match`
>   (private, one caller) and return `match.addTo?.replacement ?? ""` for `.addTo`.
> - D2: in `similarApply(_:)`, before the `available.contains(action)` guard, remap
>   `.note` to `.addTo` when the focused row offers `.addTo` and not `.note`. Store the
>   remapped action in `similarPendingAction` too. Leave the router context-free.
> - Add `BobMacCaptureTests` regressions next to the existing `SimilarTasksActionsTests`
>   patterns. Use a record row with the example above. Assert the captured
>   `CaptureNativeRewriteEdit` has UTF-16 range `{0, 17}` and replacement text
>   `@eat/ninja_sushi_bar`. Assert `similarApply(.note)` on a record row yields
>   `@eat/ninja_sushi_bar`, and that `.note` on a task row is unchanged.
> - No docs change: `docs/capture-previews.md` already documents `+` for Add to it, and
>   the mounted Mac pages are over their word budgets (bob-cli-h9).
> - Record D1/D2 as fixed in the step-4 note instead of a DISCOVERED ISSUE.

## 3. install-all failure excerpt in bob-cli (decision `install_excerpt`)

> [!decision] install_excerpt = fix In
> `scripts/lib/install_ui/live.py::_failure_excerpt`, first look for the first line
> carrying a real diagnostic error label. Use a case-sensitive regex such as
> `(?:^|[\s:])error(?:\[[^\]]*\])?:`, which matches swiftc `file:1:2: error: ...`, rustc
> `error[E0308]: ...`, cargo/just `error: ...`, and Swift's caret line `|- error: ...`.
> It must not match `... this is an error in the Swift 6 language mode`. Start the
> 40-line excerpt there. Otherwise keep today's fallbacks unchanged: the
> case-insensitive `error` substring, then the last 20 lines. Add unit tests in
> `scripts/lib/install_ui/tests/test_live.py` (or a sibling test module):
>
> - A log with several Swift 6 "is an error in the Swift 6 language mode" warnings
>   followed by `X.swift:15612:9: error: switch must be exhaustive` yields that error
>   line first.
> - A log with only a capitalized `Error:` still uses the substring fallback.
> - A log with no error falls back to the tail.
>
> Leave `scripts/install_all`'s plugin-sync `grep -i 'error'` excerpt alone (different
> tool output, not implicated). Run `just check-scripts`. No docs or CHANGELOG edits;
> the release train owns versions.

> [!decision] install_excerpt = task Make no bob-cli change. Use `/sase_new_task` to
> file a `bug` task bead (size `small`, root cause as described in the Diagnosis) unless
> an equivalent task already exists, in which case `+1` it with this log as evidence.

## 4. Verify, then leave notes

Verification (Mac):

- Run `just portable-check` from the linked bob-mac-capture checkout on athena. It
  parse-lints every Swift file and runs the portable targets.
- Best effort, per bob-mac-capture `AGENTS.md` and the tailnet reference memory: rsync a
  scratch copy (excluding `.build` and `.git`) to the `mac` host. There, run
  `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
  Bryan's checkout and never run `just install` there. The Mac was offline at the end of
  diagnosis. If it still is, say so in the final report and rely on CI.
- Run `Scripts/macos-ci-check` against the final working tree through `/sase_monitor`.
  Never wait in-turn. Required: Lint, Build, Test, Bundle, and the install gates green.
  This is the first CI run to execute `ef5dc4a`'s tests. If Test fails only in
  bob-cli-j6-owned alias suites (`CaptureModifierAliasTests`,
  `CaptureRelationEqualsAliasTests`, or fake-bob modifier delays, which the bob-cli-j6
  land audit already lists as open gaps), still land this fix: Build and the scratch
  bundle prove it. Record the failing tests on `bob-cli-j6` with `sase bead note`.
  Investigate any other red step. Fix it here only if this tale caused it; otherwise
  record it on its owning epic.
- If `install_excerpt = fix`, `just check-scripts` must pass in bob-cli.

Notes (after verification succeeds, before `/sase_final`; the commit SHA is not known
yet, so cite the commit subject and the precommit snapshot run):

- On `bob-cli-ir`, append a note starting `RESOLVED (note #6):`. Cover the root cause
  and why it shipped (the ir.12 note #1 quote and the skipped Mac compile and
  `macos-ci-check`). Cover the impact: master red at Build since `a5152aa` (runs
  42672189863 and 42673185238), and Bryan's `just install-all` failing on the Mac
  2026-11-09 13:54 EST. Say that this commit adds the no-task-ID `.addTo` cases in the
  three switches, which are unreachable because record rows never require a block ID,
  and list the verification results. Mention the `install_excerpt` outcome and why the
  reported output hid the errors.
- On `bob-cli-ir`, add the D1/D2 note required by the `record_row_gaps` branch.
- On `bob-cli-j6`, append one short coordination note. Say that the `.addTo` compile
  adaptation landed through this Record Notes fix with text identical to its pending
  edit, so its closeout incorporates it rather than reapplying. Include any j6-owned
  test failures observed in macos-ci-check.

Do not close or reopen any bead. Never close `bob-cli-ir` or `bob-cli-j6`; their land
agents do that.

## 5. Finish

Finish through `/sase_final`, declaring the linked bob-mac-capture checkout. Also
declare bob-cli if `install_excerpt = fix`. Mac commit subject:
`fix(capture): handle record Add to it in Add block ID switches`. The body must cite
`a5152aa` and `bob-cli-ir`. bob-cli commit subject (if applicable):
`fix(install): start failure excerpts at the first real compiler error`.

After the push, watch the exact-SHA macOS CI run through `/sase_monitor` as
bob-mac-capture's `AGENTS.md` requires. Do not make the notes wait on it. If that watch
finds a regression this tale caused, fix it. In any case, append a one-line `bob-cli-ir`
note with the landed SHA and run verdict.

The final report tells Bryan to rerun `just install-all` (or `just install` in his
bob-mac-capture checkout) on the Mac. The tale itself never runs `just install` there.

## Non-goals

- The Swift 6 Sendable / actor-isolation warnings in the build log.
- bob-cli-j6's remaining alias, fixture, and docs gaps (its closeout tale owns them).
- The mounted Mac docs word-budget overages (bob-cli-h9 and bob-cli-ir note #5).
- Any memory edits.
