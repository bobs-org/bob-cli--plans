---
tier: tale
title: Accept notification permission and always-focused Settings
goal:
  The Accept copy of Bob Mac Capture can request and receive notification permission,
  failures show in Settings, and Settings always opens in front with keyboard focus.
size: medium
proposed_by: bbugyi200.athena.1h9
proposed_with: claude/opus
create_time: 2026-11-27 12:30:41
status: wip
---

# Fix the Accept copy's notification permission and always focus Settings

All code changes are in the linked `bob-mac-capture` repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "..."`) and read its `AGENTS.md` before editing.
bob-cli itself does not change.

## Why (diagnosis, with evidence)

Bryan could not do question Q2 of the harness phase `bob-cli-q5.8.1.6.5.1.1`: open the
Accept copy's Settings, click Enable Notifications, Allow, then Send Test Notification.
His screencast and the Mac's logs show two separate faults.

### Fault 1: macOS refuses notification permission for the Accept copy (root cause: temp dir)

- `Scripts/snooze-accept.sh up` installs the scratch app at
  `/tmp/bob-wake-accept/Bob Mac Capture Accept.app` (realpath `/private/tmp/...`).
  LaunchServices registers that bundle with the `in-temp-dir` bundle flag
  (`lsregister -dump`: `bundle flags: has-display-name ui-element in-temp-dir`).
  Production in `~/Applications` has no such flag.
- At launch, usernoted rejects the client:
  `usernoted: Failed to find or validate client of identifier org.bobs.bob-mac-capture.accept with audit token ... PID: 21779`.
  Every Enable Notifications click then logs `Requesting authorization with options 6` →
  `Requested authorization [ didGrant: 0 hasError: 1 ]`. macOS shows no prompt, the
  status stays `Not requested`, and System Settings › Notifications never lists the
  Accept app. Bryan reported that missing entry earlier too (bead note #7). Send Test
  Notification fails with
  `UNErrorDomain Code=1 "Notifications are not allowed for this application"`.
- A controlled probe on the Mac (macOS 26.5) isolates the cause. It ran the same Swift
  binary, with the same ad-hoc signature and only the location changed.
  `/private/tmp/.../Probe.app` got the same `Failed to find or validate client` line.
  `~/.cache/.../Probe.app` got
  `Connection org.bobs.notify-probe.homeadhoc with path: ...`, meaning usernoted
  accepted it. So the ad-hoc signature is fine and the temp location is the cause.
  (Signing with the stable "Bob Mac Capture Local" identity over SSH fails with
  `errSecInternalComponent`, which is why the harness must stay ad-hoc.)
- The app made this invisible. `NotificationService.requestAuthorization()` runs
  `_ = try? await center.requestAuthorization(...)`, which swallows the error, so the
  button appears to do nothing.

### Fault 2: Settings opens behind the frontmost app

- `AppDelegate.openSettings()` → `SettingsPresentation.present()` calls the SwiftUI
  Settings scene's `openSettings()` and then `NSApplication.shared.activate()`. Bob Mac
  Capture is an `.accessory` app. In the screencast, Bryan picked Settings from the
  status menu at 12:10:35.7. WindowServer's frontmost process stayed Obsidian until he
  found the window through Mission Control and clicked it at 12:10:40.25. The activation
  request was declined.
- Probes on macOS 26.5 show that activation is strictly cooperative. `NSApp.activate()`,
  `activate(ignoringOtherApps: true)`, and `.regular` policy combined with either call
  were all declined; the frontmost app stayed kitty. A `.nonactivatingPanel` `NSPanel`
  calling `makeKeyAndOrderFront` became key without activation: WindowServer logged
  `advicePolicy: .keyThief` and routed real keyDown events to it. Capture and Refs
  already use this window type and take focus reliably from hotkeys and the status menu.
- So the only deterministic way to "always focus Settings" is to host Settings in an
  app-owned non-activating panel, not in the SwiftUI `Settings` scene.

## Governing decisions

- `decisions:q-and-ctrl-bracket-mirror-esc`: every Bob panel that Esc closes must also
  close with q and Ctrl+[. Settings does not close on Esc today, and this change must
  not add Esc-to-close. Settings keeps closing via its close button and Command-W only,
  so the decision is not triggered.
- `decisions:mac-capture-is-a-thin-client`: unaffected. This work is native window and
  harness plumbing only; no grammar, preview, or vault logic moves into the app.

## Guardrails (from the harness plan and bob-mac-capture AGENTS.md)

- Never touch production: `~/Applications/Bob Mac Capture.app`, the
  `org.bobs.bob-mac-capture` defaults domain, `~/bob`, `~/.local/state/bob-cli`, or
  Bryan's checkout. Never run `just install` / `just install-all` on the Mac.
- Never land Swift you have not compiled. On athena run `just portable-check`. For
  `Sources/BobMacCapture*` / `Tests/BobMacCapture*`, compile a scratch copy on the Mac
  (rsync without `.build`/`.git` to a fresh `/tmp` dir,
  `./Scripts/bundle.sh --identity -`, `just typecheck-tests`, then delete the copy). Run
  `Scripts/macos-ci-check` before landing. After pushing, watch the exact-SHA macOS CI
  run to green with `/sase_monitor`. Format only changed lines (`.swift-format`: 4
  spaces, 100 columns).
- Over SSH the Mac's login shell is zsh, where `log` is a builtin. Always call
  `/usr/bin/log`, or pipe scripts to `ssh mac 'bash -s'`.
- Do not edit `Cargo.toml` versions or `CHANGELOG.md` (bob-cli release train); bob-cli
  is untouched anyway.

## Changes

### 1. Harness: install the scratch app outside any temp dir (`Scripts/snooze-accept.sh`)

- Keep `ROOT=/tmp/bob-wake-accept` for the vault, state, evidence, and PID files. Later
  phases and bead notes cite those paths. Move only the app bundle:
  `SCRATCH_APP_DIR="$HOME/Library/Application Support/bob-wake-accept"`,
  `SCRATCH_APP="$SCRATCH_APP_DIR/$ACCEPT_NAME.app"`. The `find_scratch_pids` pattern
  `bob-wake-accept/$ACCEPT_NAME.app` still matches, and `find_prod_pid`
  (`Applications/Bob Mac Capture.app/...`) can never match it. Quote every use; the path
  contains spaces.
- Update the header comment to explain why the bundle lives there: usernoted refuses
  notification clients whose bundle LaunchServices flags `in-temp-dir` (`/private/tmp`,
  `/private/var/folders`).
- `up`: refuse when `$SCRATCH_APP_DIR` already exists, as it already does for `$ROOT`
  ("run down first"). After `launch_scratch`, add a `check_notification_client` step. It
  polls
  `/usr/bin/log show --last 2m --style compact --predicate 'process == "usernoted" AND eventMessage CONTAINS "<ACCEPT_ID>"'`
  for up to ~30 s:
  - `Failed to find or validate client of identifier $ACCEPT_ID` → `die` with a message
    naming the bundle path and the `in-temp-dir` cause.
  - `Connection $ACCEPT_ID with path:` → pass.
  - Neither line → warn on stderr, record `unknown`, and don't fail. Save the matching
    lines to `$EVIDENCE_DIR/usernoted-client.log`. Add
    `"notification_client":"accepted|unknown"` to the `up` JSON line.
- `guard`: add a cheap path check that fails if the realpath of `$SCRATCH_APP` is under
  `/private/tmp/` or `/private/var/folders/`.
- `status`: add `scratch_app` (the path) and `notification_client` (read back from the
  evidence file, or `unknown`).
- `down`: always remove the app dir. Stop by recorded PID,
  `lsregister -u "$SCRATCH_APP"`, then `rm -rf "$SCRATCH_APP_DIR"`, even when `$ROOT` is
  already gone. Today `down` returns early on a missing root and would leave the app
  behind. Also `lsregister -u` the legacy `$ROOT/$ACCEPT_NAME.app` if it exists, which
  is the case for the harness running today, before removing `$ROOT`. Report a leftover
  if the app dir still exists afterwards.
- `restart` relaunches the same bundle at the new path; no other change is needed.
- Keep `bash -n` (justfile `check-scripts`) and shellcheck clean.

### 2. App: surface notification-authorization failures (`NotificationService.swift`, `SettingsView.swift`)

- Stop swallowing the error. Give `NotificationService` a
  `@Published private(set) var authorizationRequestFailure: String?`. Clear it when a
  request starts and on a later success. Set it when `requestAuthorization` throws. Keep
  `refreshAuthorizationStatus()` afterwards.
- Add an injectable request seam for tests, for example an `init` parameter
  `requestAuthorizationAction: (UNAuthorizationOptions) async throws -> Bool` that
  defaults to the center. Do not change the existing `init` call sites' behavior.
- Add a pure, unit-tested formatter, for example
  `NotificationAuthorizationFailure.message(error:bundlePath:)`. It returns
  `Enable failed: <localizedDescription>`. When the bundle path resolves under a
  temporary directory (`/tmp/`, `/private/tmp/`, `/private/var/folders/`, or the
  resolved `NSTemporaryDirectory()`), it appends one sentence: macOS does not allow
  notifications for apps inside temporary folders, so install or move the app outside
  them.
- In Settings › Notifications, show the failure under the Authorization row as secondary
  text. Make it and the existing `testNotificationStatus` text wrap
  (`.fixedSize(horizontal: false, vertical: true)`) so the 620 pt form never clips them.
  The screencast showed the test-failure text clipped at the right edge.
- Install-time behavior stays unchanged: never prompt automatically. Denied or
  not-requested authorization on the other notify paths stays silent.

### 3. App: host Settings in a non-activating panel that always takes focus

- Add `Sources/BobMacCapture/SettingsWindowController.swift` (`@MainActor`). It owns one
  lazily created, retained `SettingsPanel: NSPanel` with:
  - styleMask `[.titled, .closable, .miniaturizable, .resizable, .nonactivatingPanel]`
  - `canBecomeKey` true
  - `isFloatingPanel = false`, `level = .normal`
  - `hidesOnDeactivate = false`, `isReleasedWhenClosed = false`,
    `becomesKeyOnlyIfNeeded = false`
  - `collectionBehavior` including `.moveToActiveSpace`, so the panel comes to the
    current Space instead of leaving the user on another one
  - title `"<CFBundleName> Settings"`, which keeps "Bob Mac Capture Settings" and "Bob
    Mac Capture Accept Settings"
  - an `NSHostingController` / `NSHostingView` content of the existing `SettingsView`
    with the same dependencies `BobSettingsScene` passes today (settings,
    notificationService, canceledDraftStash, savedTemplateShelf, and the three reset
    callbacks) Preserve today's geometry from `SettingsScrollContainer` /
    `SettingsScrollMetrics`: 620 pt wide, height between 420 pt and the screen, resizing
    vertically only. Center the panel on first show, then remember its frame
    (`setFrameAutosaveName`).
- `show()` must be idempotent and run on every open, whether the panel is new, already
  open and buried behind other apps, or minimized. It deminiaturizes if needed, then
  calls `makeKeyAndOrderFront(nil)` and `orderFrontRegardless()`. Do not call
  `NSApp.activate()`: the probes show it is declined, and the panel does not need it.
  The result is that the Settings panel is the key window, keyboard input reaches it,
  and Control-D / Control-U page at once with no click. `SettingsScrollKeyView`'s
  monitor is gated on its own window being key with no sheet, so it works unchanged.
- Rewire `SettingsPresentation` to call the controller: `present()` → `show()`, with the
  closure kept injectable for tests. Delete the SwiftUI `Settings` scene plumbing:
  `NSHostingSceneRepresentation`, `BobSettingsScene`, `settingsSceneRepresentation`, and
  the `representation.environment.openSettings()` call. Every entry point must route
  through `AppDelegate.openSettings()`: the status menu Settings item, the status-item
  issue row ("bob Not Resolved — Open Settings…"), and Bob Refs' `settingsPresenter`
  (Choose Highlights).
- Make the panel's interactions work in a non-activated app:
  - "Choose…" (Highlights app): replace `NSOpenPanel.runModal()` with
    `beginSheetModal(for:)` on the Settings panel.
  - The four `.confirmationDialog`s must still present and dismiss.
  - Text fields must accept typing and Command-X/C/V/A/Z (the main menu's Edit items
    already serve Capture this way).
  - Command-W must close Settings. Add a Close item (`performClose:`) to the app's main
    menu if `makeMainMenu` lacks one, or handle Command-W in `SettingsPanel`.
  - Do not add Esc/q/Ctrl+[ close behavior (`decisions:q-and-ctrl-bracket-mirror-esc`).
- Capture and Refs keep their floating levels and existing behavior.
- Update `testOpenSettingsMenuActionUsesRegisteredSettingsPresenterOnce` in
  `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` for the new presentation shape.
  Add `SettingsWindowControllerTests`. Every test window sets
  `isReleasedWhenClosed = false`; see the over-release crash fixed in 4053b19e. The
  tests assert:
  - one panel is created and reused across repeated `show()`
  - the style mask contains `.nonactivatingPanel`; `level == .normal`,
    `hidesOnDeactivate == false`, `isReleasedWhenClosed == false`
  - `show()` on a miniaturized or ordered-out panel orders it in
  - the title is derived from the bundle name
  - `canBecomeKey` is true Don't assert that the window is actually key, because CI runs
    headless.

### 4. Docs (bob-mac-capture `docs/`, mounted into Bob's site)

- `docs/settings.md` intro paragraph: Settings opens as a non-activating panel, like
  Capture and Refs. Opening it from the menu-bar item, the issue row, or Bob Refs brings
  it to the front on the current Space with keyboard focus, so Control-D works at once,
  even while another app stays frontmost. It never shows a Dock icon. Command-W closes
  it.
- `docs/permissions.md` › Notifications: a failed Enable Notifications request now shows
  `Enable failed: …` under Authorization. macOS refuses notification permission for an
  app bundle inside a temporary folder (`/tmp`, `/private/var/folders`), so install it
  outside one.
- `docs/develop/architecture.md`: one clause noting that Settings is an app-owned
  non-activating panel, not the SwiftUI Settings scene, because activation requests from
  the status menu are declined on macOS 26.
- Run `just docs-check` until it passes.

## Verification and landing

1. On athena: `just portable-check` and `just check-scripts` (or the recipe that runs
   `bash -n` over `Scripts/snooze-accept.sh`), plus
   `shellcheck Scripts/snooze-accept.sh`.
2. On the Mac (best effort), using a scratch copy in a fresh `/tmp` dir: `bundle.sh` and
   `just typecheck-tests`, then delete the copy. If the Mac is offline, record that the
   macOS targets are unverified locally and rely on CI.
3. `Scripts/macos-ci-check` green, then land on bob-mac-capture master and watch the
   exact-SHA macOS CI run to green with `/sase_monitor`. A Test step that dies at the
   watchdog is a hang; read the sampled stack. Known timing flakes (bob-cli-hu,
   bob-cli-jk family, CaptureTaskGotoPanelTests) get one `gh run rerun --failed`.
   Anything else is yours to fix.
4. Rebuild the Accept harness from the new green SHA, coordinating first (next section):
   `down` (record leftovers),
   `git archive <SHA> | ssh mac "rm -rf /tmp/harness-src && mkdir -p /tmp/harness-src && tar -x -C /tmp/harness-src"`,
   then run `up` detached so an SSH drop cannot kill it:
   `ssh mac "nohup bash /tmp/harness-src/Scripts/snooze-accept.sh up --sha <SHA> >/tmp/harness-up.log 2>&1 </dev/null &"`.
   Poll the log until the `status:"up"` JSON appears. Then run `guard` and `status`.
   Expect:
   - `notification_client: accepted`
   - the usernoted evidence line
     `Connection org.bobs.bob-mac-capture.accept with path: <home>/Library/Application Support/bob-wake-accept/...`
   - `lsregister -dump` showing the Accept bundle without `in-temp-dir` If production
     was reinstalled again since the last recorded PID (it was at about 12:09 EST on
     2026-11-27, after the last re-adopt), `up` re-records it from `find_prod_pid`.
     Never touch it.
5. Ask Bryan one `/sase_questions` round while he is at the Mac:
   - Is he at the Mac, awake and unlocked?
   - With another app frontmost, open Settings from the Accept app's menu-bar item. Does
     it come to the front with keyboard focus, so Control-D scrolls without a click? Do
     the same with Settings already open but buried behind another app's window.
   - In the Accept Settings, click Enable Notifications. Does the macOS Allow prompt
     appear? After Allow, does Authorization read `Authorized`?
   - Click Send Test Notification. Does a banner appear, and does System Settings ›
     Notifications now list "Bob Mac Capture Accept"? Write his verbatim answers as a
     note on the harness phase bead before anything else. If focus or the prompt still
     fails, diagnose from `/usr/bin/log` on the Mac (WindowServer `frontmostProcess` /
     `keyThief` lines; usernoted lines for the Accept ID) and fix. Do not finish with a
     known-failing check.

## Coordination with the running harness phase

- The harness phase `bob-cli-q5.8.1.6.5.1.1` owns the Accept harness and has a pending
  question round whose Q2 is exactly the blocked step. Before touching the harness, run
  `sase bead read bob-cli-q5.8.1.6.5.1.1 -r "..."` and `sase agent list`.
- If an agent of that phase is RUNNING, do not run `down`/`up`. Add a bead note with the
  fix SHA, its green CI run, and the rebuild steps above, then stop after step 3.
- Otherwise, do steps 4–5 yourself. Then add a NEXT ROUND note to that bead with the new
  harness SHA, PIDs, guard JSON, the `notification_client` evidence, and Bryan's
  answers. The harness successor can then skip the permission question and run its one
  `wake`.
- Never close or edit that phase bead's status. Notes only.

## Acceptance criteria

- `up` on the Mac places the Accept app outside temp dirs. usernoted accepts it
  (`Connection ... with path`), and `up` fails loudly if usernoted ever rejects it
  again. `down` leaves neither the root nor the app dir behind.
- Enable Notifications in the Accept Settings shows the macOS Allow prompt. After Allow,
  Authorization reads `Authorized`, and Send Test Notification shows a banner. If a
  request ever fails, Settings shows `Enable failed: …` (with the temp-folder hint when
  it applies) instead of doing nothing.
- Opening Settings from any entry point brings the Settings panel to the front on the
  current Space, as the key window with keyboard focus, even while another app is
  frontmost and even when it was already open and buried or minimized. No Dock icon
  appears.
- Settings keeps its 620 pt scrolling layout, Control-D / Control-U paging, its
  confirmation dialogs, the Choose… picker (now a sheet), text editing, and Command-W
  close.
- Docs updated; `just docs-check`, `portable-check`, script checks, `macos-ci-check`,
  and exact-SHA master CI are green.
- Bryan confirmed the focus and notification checks on the Mac, recorded verbatim on the
  harness bead.
