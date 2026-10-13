---
tier: tale
title: Make bob-mac-capture's just install compile again
goal: bob-mac-capture builds, tests, lints, and installs again after the Go To and
  link-guard commits.
size: medium
decisions:
  mac_verify:
    ask: Should the coder compile a scratch copy of the fix on the MacBook over Tailscale
      SSH before landing?
    choices:
      mac_scratch: Builds app, helper, and tests in a temp copy on `mac`; your checkout
        and installed app are untouched
      linux_only: Lands on Linux checks alone; macOS targets stay unverified until
        you rerun `just install`
    default: mac_scratch
    why: Linux cannot type-check BobMacCapture, which is the target that broke `just
      install`
    answer: mac_scratch
  agents_gate:
    ask: Should the fix also add a `just portable-check` recipe and an AGENTS.md rule
      to compile Swift before landing?
    choices:
      add: Adds one justfile recipe, two README sentences, and an AGENTS.md that `sase
        repo open` shows
      skip: Lands only the compile fixes; the next agent without Swift can ship uncompiled
        code again
    default: add
    why: The last four red CI runs all came from commits whose agents never compiled
      the Swift they changed
    answer: add
proposed_by: bbugyi200.athena.101
decided_by: auto
status: done
---

# Plan: Make bob-mac-capture's `just install` compile again

## Context

Bryan's `just install` on the MacBook fails while building the `BobMacCapture` release
product. The build log shows one error:

```
Sources/BobMacCapture/CapturePanelModel.swift:2097:45: error: value of optional type
'String?' must be unwrapped to a value of type 'String'
    expectedContext: completion.context
```

Everything else in that log is a warning from code these commits did not change. Leave
those warnings alone.

### Root cause

- `6345673` ("feat(capture): open Obsidian from the Go To picker") changed
  `handleCompletionResponse` in `Sources/BobMacCapture/CapturePanelModel.swift`. It now
  passes `completion.context` to `handleTaskLinkCompletion(..., expectedContext:)`.
  `CaptureCompletionResponse.context` is `String?`
  (`Sources/CaptureCore/CaptureModels.swift`), but `expectedContext` is a non-optional
  `String = "task_link"`.
- Nobody saw this before the install because CI's first real step, **Lint Swift
  formatting**, has failed on Swift parse errors since `8f89423`. That skipped Build,
  Test, Bundle, and the install gates on `8f89423`, `a86eddb`, `6345673`, and `7b87f79`
  (runs 38408350959, 38414162582, 38417330907, and 38419021175). The last green run was
  `eed300d` (run 38140692096).
- `7b87f79` ("fix(capture): compile link-guard presentation on Swift") ran on a host
  with no Swift toolchain, and its plan said not to compile. It fixed the `guard`
  identifier. It then replaced three illegal `""",` multi-line literals in
  `Tests/CaptureCoreTests/CaptureTogglePresentationTests.swift` with `#"` … `"#`. Those
  are single-line raw strings, so they still cannot span lines. CI run 38419021175 and a
  local Linux build both report unterminated string literals at lines 512, 576, and 588.
  `just install` doesn't build tests, so it hides this, but `just test`,
  `just format-lint`, and CI all fail on it.

### Evidence from planning (scratch copies under /tmp, Linux Swift 6.0.3)

- At `eed300d`, `swift test` passes all 1088 tests.
- At HEAD (`7b87f79`), with only the test-literal fix from step 3 applied:
  `swift build --build-tests` is clean, `swift test` passes all 1106 tests, and
  `swift format lint --recursive Package.swift Sources Tests` exits 0 with warnings
  only.
- Static audit of the macOS-only diff `eed300d..HEAD`:
  - Every exhaustive `switch` over `CapturePickerSource`, `CapturePickerIndex`, and
    `CapturePickerNeed` in `Sources/BobMacCapture` already handles `.taskGoto`.
  - The CaptureCore APIs the new Mac code calls exist with matching signatures:
    `ObsidianOpenURL.taskGotoURL(path:line:ref:blockID:)`,
    `CapturePickerRow.taskGotoDestination`, the `CaptureLinkGuardPresentation` fields,
    and `CapturePomodoroClosePresentation.droppedDependenciesChip`.
  - `CaptureKeyRoutingContext` memberwise call sites pass arguments in declaration
    order.
- Release builds type-check the whole module, so line 2097 is probably the only type
  error in `BobMacCapture`. Two gaps remain:
  - SIL-stage diagnostics (definite initialization, missing returns, and so on) run only
    after type checking passes.
  - `Tests/BobMacCaptureTests` has never compiled since `eed300d`. That includes the new
    `CaptureTaskGotoPanelTests.swift` (492 lines) and one new test in
    `CapturePickerMarkerHighlightTests.swift`.

### Non-goals

- Do not change Go To, link-guard, or dropped-dependency behavior.
- Do not fix the pre-existing warnings: `await` with no async operation, `var` that
  could be `let`, Sendable conversions, and the unused `openSelected` result.
- Do not edit bob-cli. This plan touches only the linked `bob-mac-capture` repo.

## 1. Open the linked repo

```sh
sase repo open bob-mac-capture -r "Fix the Swift compile errors that break bob-mac-capture just install"
```

Use only the path it prints. Confirm HEAD is `7b87f79` or a descendant. If newer commits
touch `handleCompletionResponse` or the `CaptureTogglePresentationTests` fixtures,
re-read those spots before you edit.

## 2. Unwrap the completion context in `handleCompletionResponse`

In `Sources/BobMacCapture/CapturePanelModel.swift`, bind the context once so the branch,
the source choice, and `expectedContext` all use the same non-optional value. Replace:

```swift
if completion.context == "task_link" || completion.context == "task_goto" {
    await handleTaskLinkCompletion(
        completion,
        draft: draft,
        cursor: cursor,
        generation: generation,
        processClient: processClient,
        trigger: trigger,
        source: completion.context == "task_goto" ? .taskGoto : .taskLink,
        expectedContext: completion.context
    )
    return
}
```

with:

```swift
if let context = completion.context, context == "task_link" || context == "task_goto" {
    await handleTaskLinkCompletion(
        completion,
        draft: draft,
        cursor: cursor,
        generation: generation,
        processClient: processClient,
        trigger: trigger,
        source: context == "task_goto" ? .taskGoto : .taskLink,
        expectedContext: context
    )
    return
}
```

Behavior does not change. Do not force-unwrap, and do not coalesce to `"task_link"`.

## 3. Repair the three dropped-dependency test literals

In `Tests/CaptureCoreTests/CaptureTogglePresentationTests.swift`, three fragments use
`#"` … `"#`:

- `nextDirectionJSON`'s `droppedField` (about line 512)
- `ensureNextJSON`'s `case 1` arm (about line 576)
- `ensureNextJSON`'s `default` arm (about line 588)

Rewrite each one as a standard multi-line string literal:

- The opening `"""` ends its line.
- The closing `"""` sits alone on its own line.
- Every content line is indented at least as far as the closing delimiter.

The literal drops the newline after the opening delimiter and the newline before the
closing one. The fragment therefore still starts with `,` and splices cleanly after
`"pomodoro_selector_unused": false` or after the `pomodoro_link_destination` object. For
`nextDirectionJSON`:

```swift
        let droppedField = droppedDependencies
            ? """
            , "dropped_dependencies": [
              {
                "note_path": "job.md",
                "block_id": "prep",
                "link": "[[job#^prep]]",
                "text": "Prep work",
                "status_symbol": " ",
                "status_name": "Ready"
              }
            ]
            """
            : ""
```

Apply the same shape to both `ensureNextJSON` arms:

- `case 1`: the single `job.md` / `prep` object.
- `default`: the `job.md` / `prep` object, then the `ops.md` / `budget` /
  `"Budget work"` object.

Keep every key and value, the ternary/switch shape, and the
`\(pomodoroNameField)\(droppedField)` interpolation exactly as they are. Afterwards,
`grep -n '#"$' Tests/CaptureCoreTests/CaptureTogglePresentationTests.swift` and
`grep -rn '""",' Sources Tests` must both print nothing.

## 4. Audit the macOS-only code nobody has compiled

Review `git diff eed300d..HEAD -- Sources/BobMacCapture Tests/BobMacCaptureTests`. Most
of that code is in `Tests/BobMacCaptureTests/CaptureTaskGotoPanelTests.swift` and the
new `testMarkerHighlightRangeForTaskGotoKeepsSigilRange`. For every symbol a new or
changed line references, open its declaration and confirm:

- the name, argument labels and order, types, and optionality
- that it is visible to `@testable import` (internal is fine, `private` is not)
- that its `@MainActor` isolation fits the call site

Fix only genuine compile errors, in the smallest way. Do not restyle code or rename
tests.

## 5. Verify on Linux (required)

Run these from the linked checkout with the `swift` on PATH (athena has Swift 6.0.3). If
the current host has no `swift`, run the same commands on a tailnet host that has one,
against a scratch copy. Never land unverified Swift.

```sh
swift format lint --recursive Package.swift Sources Tests   # exit 0; warnings are fine
swift build --build-tests                                   # no errors
swift test                                                  # 0 failures, at least 1106 tests
```

These commands cover CaptureCore, RefsCore, refs-rank, and their tests, plus a parse of
every Swift file. They cannot type-check the `BobMacCapture*` targets.

## 6. Verify the macOS targets

> [!decision] mac_verify = mac_scratch
>
> The MacBook (`ssh mac`, user `bbugyi`) is best effort and is offline when its lid is
> closed. Probe it with `ssh -o ConnectTimeout=10 mac true`. If it answers, work only in
> a scratch copy:
>
> ```sh
> rsync -a --delete --exclude .build --exclude .git ./ mac:/tmp/bob-mac-capture-verify/
> ssh mac 'cd /tmp/bob-mac-capture-verify \
>   && ./Scripts/bundle.sh --identity - \
>   && ./Scripts/xcode-swift.sh build --configuration release --product BobMacCaptureInstallHelper \
>   && ./Scripts/xcode-swift.sh build --build-tests \
>   && ./Scripts/xcode-swift.sh test --filter "CaptureTaskGotoPanelTests|CapturePickerMarkerHighlightTests"'
> ssh mac 'rm -rf /tmp/bob-mac-capture-verify'
> ```
>
> `bundle.sh` builds the same release product `just install` builds and assembles the
> app under the scratch `.build/`.
>
> - Never touch `~/projects/github/bobs-org/bob-mac-capture` on the mac. Never run
>   `just install` or `Scripts/install.sh`. Never touch `~/Applications`.
> - Fix any compile error this reveals, then re-run steps 5 and 6.
> - If a test in the two filtered classes fails at runtime, fix it when the cause is
>   clearly these commits. Otherwise report it with its output.
> - If the mac does not answer, go ahead on step 5 alone, and state in the final report
>   that the macOS targets are unverified.

> [!decision] mac_verify = linux_only
>
> Skip the MacBook. Rely on steps 4 and 5, and state in the final report that the macOS
> targets are unverified until Bryan reruns `just install` or CI runs on master.

## 7. Guard against recurrence

> [!decision] agents_gate = add
>
> - **`justfile`:** add a recipe after `test`. Hosts without Xcode cannot use
>   `xcode-swift.sh`, so the recipe uses the PATH toolchain, and its lint line mirrors
>   `format-lint`'s `swift-format` fallback:
>
>   ```just
>   # Gate for hosts without Xcode (Linux): parse-lint every Swift file, then build and
>   # test the portable CaptureCore/RefsCore targets with the PATH Swift toolchain.
>   portable-check:
>       if command -v swift-format >/dev/null 2>&1; then \
>         swift-format lint --recursive Package.swift Sources Tests; \
>       else \
>         swift format lint --recursive Package.swift Sources Tests; \
>       fi
>       swift build --build-tests
>       swift test
>   ```
>
>   Run `just portable-check` on Linux and confirm it passes.
>
> - **`README.md`, `## Development`:** add one sentence after the command block. On a
>   host without Xcode, only CaptureCore, RefsCore, and refs-rank build;
>   `just portable-check` lints every Swift file and builds and tests those targets, but
>   cannot type-check the `BobMacCapture*` targets.
> - **`README.md`, `## CI`:** add one sentence. Formatting runs first, and a Swift parse
>   error there skips Build, Test, Bundle, and the install gates, so a red lint step can
>   hide compile errors further down.
> - **New `AGENTS.md`** at the repo root, short and imperative:
>   - Never land Swift you have not compiled, and never write a plan that tells an agent
>     to skip compiling.
>   - On a host without Xcode, run `just portable-check`. If `swift` is missing, run it
>     on a tailnet host that has Swift (athena).
>   - `portable-check` cannot type-check `Sources/BobMacCapture*` or
>     `Tests/BobMacCapture*`. When you change those, compile a scratch copy on the mac
>     (best effort). Use the step 6 recipe: rsync without `.build`/`.git`, then
>     `bundle.sh` and `xcode-swift.sh build --build-tests`, then delete the copy. Never
>     use Bryan's checkout or `just install` there.
>   - If the mac is offline, say in the final report that the macOS targets are
>     unverified.
>   - After a red **Lint Swift formatting** step, check the later CI steps too.

> [!decision] agents_gate = skip
>
> Change nothing beyond steps 2–4.

## 8. Land

Before landing, re-run step 5. Commit the bob-mac-capture changes through the normal
finalizer flow, as one commit or as two:

1. `fix(capture): unwrap Go To completion context and repair test literals`
2. Only if `agents_gate = add`: `chore(agents): require compiled Swift before landing`

Do not wait for CI.

The final report must cover:

- which checks ran where, with test counts
- whether the macOS targets compiled on the mac or are still unverified
- that Bryan should rerun `just install` on the mac to pick up the fix
