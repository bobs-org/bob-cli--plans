---
tier: tale
title: Turn bob-cli-ix's red Mac CI green and close the epic
goal:
  Bob Mac Capture's sized-close tests pass on macOS CI because they edit fixtures
  structurally and expect uppercase X to stay off Bob, and epic bob-cli-ix is closed.
size: small
proposed_by: bbugyi200.apollo.bob-cli-ix.land
proposed_with: claude/opus
bead: bob-cli-ix
status: done
---

- **BEAD:**
  [bob-cli-ix](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ix/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ix.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ix.land.md)
- **COMMITS:**
  - [90fa59d](https://github.com/bobs-org/bob-mac-capture/commit/90fa59dfa38374fc2ea95c6166c9b61337122a7c)
    — test(mac-capture): edit sized-close fixtures structurally and keep uppercase X off
    Bob

# Turn bob-cli-ix's red Mac CI green and close the epic

## Why

Epic `bob-cli-ix` (Nx: close the running Pomodoro at a stated size) landed every bob-cli
commit, and bob-cli's tests pass. But the Bob Mac Capture landing commit `b81302b`
("feat(mac-capture): present sized closes with one chip and docs") is **red** on its
exact-SHA macOS CI run (GitHub Actions run 42640876363, job "macOS 26 SwiftPM"). All 10
failures are in tests that this epic added. The product code under
`Sources/CaptureCore/CapturePomodoroClosePresentation.swift` and
`Sources/BobMacCapture/CapturePanelController.swift` is correct. The tests are wrong.
The epic was reopened because of this failure. This tale fixes the tests and closes the
epic.

Every Mac change below is in the linked `bob-mac-capture` repo. Open it with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "..."`). Read its `AGENTS.md` and
work only in the path that command prints. bob-cli itself needs no code change.

## Root causes (verified against the CI log)

1. `Tests/CaptureCoreTests/CapturePomodoroClosePresentationTests.swift` builds three
   variants by doing **compact-JSON string replacements** on
   `Tests/Fixtures/pomodoro-close-sized-short.json`. That fixture is **pretty-printed**
   (`"closed_at": "0910"` with a space, and `"size": {` on its own line), so none of the
   replacements match and each variant decodes as the unmodified 2x/0900-0910 fixture:
   - `testSizedCloseEqualSizeKeepsDurationWithoutArrow`: the nine
     `.replacingOccurrences` calls are all no-ops. Result: "0900-0925 → 0900-0910 · 10m"
     instead of "0900-0925 · 25m", and the chip reads "2x · as of 09:10".
   - `testSizedCloseRoundUpChipReadsClosedAt`: the `"closed_at":"0910"` and
     `"remaining_minutes":0` replacements are no-ops, so the chip reads 09:10, not
     09:08.
   - `testUnsizedAndOldBobCloseStillRenders`: the hand-rolled brace scanner searches for
     `,"size":{`, never finds it, and the old-bob payload keeps its `size`.
2. `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`
   `testSizedCloseAliasLeavesProseLiteral` has a `("2X", "x", "2Xx", <record>)` case.
   `CapturePanelController.closeAliasPrefilterAllows` correctly refuses to forward when
   the character before the caret is uppercase `X`, so bob is never spawned and the
   record file is never created. `try String(contentsOf: recordURL)` then throws
   NSCocoaErrorDomain 260 at line 2155. The prefilter unit assertions
   (`closeAliasPrefilterAllows(text: "X", ...)` is false) and the uppercase block in
   `testCloseAliasDeclinesRepeatsModifiersUppercaseAndIME` already pin this intended
   behaviour.

## Changes (bob-mac-capture, tests and fixtures only)

### 1. `Tests/CaptureCoreTests/CapturePomodoroClosePresentationTests.swift`

- Add a private helper next to `decodeFixture` / `fixtureText` / `decodeSuccess`. It
  edits the fixture structurally, so variants no longer depend on its whitespace. Give
  it a distinct name so it never makes an existing `decodeFixture(_:)` call ambiguous:

  ```swift
  /// Decodes a fixture after editing its `pomodoro_close` object as JSON, so a variant
  /// never depends on the fixture's whitespace or key order.
  private func decodeFixtureVariant(
      _ name: String,
      settingClose overrides: [String: Any] = [:],
      removingClose removals: [String] = []
  ) throws -> CaptureCommandSuccess {
      let data = try XCTUnwrap(fixtureText(name).data(using: .utf8))
      var root = try XCTUnwrap(JSONSerialization.jsonObject(with: data) as? [String: Any])
      var close = try XCTUnwrap(root["pomodoro_close"] as? [String: Any])
      for (key, value) in overrides { close[key] = value }
      for key in removals { close.removeValue(forKey: key) }
      root["pomodoro_close"] = close
      let edited = try JSONSerialization.data(withJSONObject: root)
      return try decodeSuccess(String(decoding: edited, as: UTF8.self))
  }
  ```

- `testSizedCloseEqualSizeKeepsDurationWithoutArrow`: drop the string-replacement chain.
  Decode the real bob output `pomodoro-close-sized-equal.json` with
  `decodeFixture("pomodoro-close-sized-equal.json")` (5x, 0900-0925, delta 0) and keep
  all five assertions unchanged (sessionText "0900-0925 · 25m", nil sessionChangeText,
  sessionPlannedText "0900-0925 · 25m", chip "5x · as of 09:25", the "as planned"
  accessibility sentence). `testSizedEqualFixtureIsSameSizeSuccess` then becomes a
  strict subset of this test, so delete it.
- `testSizedCloseRoundUpChipReadsClosedAt`: build the payload with
  `decodeFixtureVariant("pomodoro-close-sized-short.json", settingClose: ["closed_at": "0908", "remaining_minutes": 2])`.
  Keep the three assertions.
- `testUnsizedAndOldBobCloseStillRenders`: replace the `var oldPayload` brace scanner
  with
  `decodeFixtureVariant("pomodoro-close-sized-short.json", removingClose: ["size"])`.
  Keep the `worked` half and both old-bob assertions.
- Keep `fixtureText`; `decodeFixtureVariant` still uses it.

### 2. `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`

In `testSizedCloseAliasLeavesProseLiteral`, keep the `Plan 2x` and `x made 2x` cases
(they really reach bob through the `(^|[ \t=])[0-9]{1,2}x` / leading-`x` prefilter paths
and need the record). Move the `2X` case out of the loop into its own block that matches
the existing uppercase pattern:

```swift
// Uppercase `X` is not a close opener: the prefilter never spawns Bob.
do {
    let recordURL = FileManager.default.temporaryDirectory.appendingPathComponent(UUID().uuidString)
    let model = try makeSupportedModel(recordURL: recordURL)
    let textView = makeTextView("2X")
    wireNativeApply(model, textView: textView)
    typeCloseAlias("x", into: textView, model: model)
    XCTAssertFalse(model.rewriteInFlightForTesting)
    XCTAssertEqual(model.plainDraft, "2Xx")
    XCTAssertEqual(textView.string, "2Xx")
    let logged = (try? String(contentsOf: recordURL)) ?? ""
    XCTAssertFalse(logged.contains("capture-rewrite"), "uppercase X must not reach Bob: \(logged)")
}
```

The remaining loop has only two tuples. Keep its explicit
`as [(String, String, String, String)]` annotation.

### 3. Remove the now-unreachable `2Xx` fake-bob path

The app never asks bob about `2Xx`, so remove:

- the `elif [[ "${draft}" == "2Xx" ]]` branch in `Tests/Fixtures/fake-bob`;
- `Tests/Fixtures/capture-rewrite-sigil-alias-close-sized-prose-caps.json`;
- the matching header-comment line and `"$bob_bin" capture-rewrite --cursor 3 ... "2Xx"`
  line in `Scripts/generate-close-alias-fixtures.sh`.

Leave `docs/capture-previews.md:232` ("`2Xx` stay[s] literal prose") as is. It is still
true.

Format only the lines you change (`.swift-format`: 4-space indent, 100 columns). Wrap
the long `decodeFixtureVariant(...)` calls and the `recordURL` line if they exceed 100
columns.

## Verification

Never land Swift you have not compiled (bob-mac-capture `AGENTS.md`):

1. `just portable-check` on a tailnet host with Swift (athena; apollo's Linux Swift 6
   hits unrelated Sendable strictness). It lints, builds, and runs `CaptureCoreTests`,
   including the three repaired presentation tests.
2. On the Mac, scratch-copy the repo (rsync without `.build`/`.git`, never Bryan's
   checkout, never `just install`), run `./Scripts/bundle.sh` and
   `just typecheck-tests`, then delete the copy. This type-checks
   `CaptureSeparatorToggleTests`. If the Mac is offline, say so in the close note.
3. Run `Scripts/macos-ci-check` from the bob-mac-capture checkout in the foreground with
   a long explicit timeout (it pushes a throwaway `ci/precommit/*` branch and waits for
   the real macOS run, ~20 minutes). Every macOS step, especially **Test**, must be
   green, with `CapturePomodoroClosePresentationTests` and
   `CaptureSeparatorToggleTests.testSizedCloseAliasLeavesProseLiteral` passing. If an
   unrelated, pre-existing flake fails, rerun once. If it fails on an unchanged tree,
   file it with `/sase_new_task` (type `flake`) instead of fixing it here.

## Closeout (final step, same turn as the code)

Do not wait for the post-push exact-SHA CI run before closing. The green
`Scripts/macos-ci-check` run on this exact tree is the landing gate.

1. Run `sase bead epic-symbols bob-cli-ix`. It listed no entries at audit time. If any
   appear, resolve each one (wire it up, privatize it, add a non-test pragma, or delete
   it per the Symvision epic-whitelist policy). Re-key a Justfile line only to a
   still-open later bead that still needs it.
2. Close the epic:
   `sase bead close bob-cli-ix --note "<what you verified: the four test fixes, portable-check, Mac typecheck-tests, macos-ci-check run URL/verdict; bob-cli side and follow-up triage were already verified in notes #1-#7>"`.
   Never use `--force` merely to make the close succeed.
3. Run `just symvision` in bob-cli if the recipe exists, and confirm the whitelist is
   clean.
4. The epic's plan file `plan:202611/sized_pomodoro_close.md` already has
   `status: done`. Confirm it still does, and set it if anything reverted it.
5. `bob-cli-ix` has no `parent_bead`, so nothing further to close.
6. Finish through `/sase_final`, declaring a commit for the bob-mac-capture repository
   obligation (message along the lines of
   `test(mac-capture): edit sized-close fixtures structurally and keep uppercase X off Bob`
   carrying the `bob-cli-ix` bead).
