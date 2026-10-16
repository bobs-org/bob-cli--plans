---
tier: tale
title: Make just install-all keep Bob Mac Capture's Accessibility grant
goal:
  Bob Mac Capture installs with the stable signing identity from just install-all, so
  the Highlights Tabs Accessibility grant survives reinstalls and the permission card
  leads to a working grant.
size: small
decisions:
  identity_script:
    ask:
      Add a one-time Scripts/create-signing-identity.sh so Bryan can create the signing
      identity with one command?
    default: false
    why:
      the GUI steps take two minutes, and no agent can test a keychain-mutating script
    answer: false
proposed_by: bbugyi200.apollo.80
decided_by: auto
create_time: 2026-10-16 10:34:57
status: wip
---

# Plan: Make `just install-all` keep Bob Mac Capture's Accessibility grant

## Diagnosis

The Tabs permission card in the screenshot shows its ad-hoc caption ("Grants reset when
an ad-hoc build is reinstalled"). So the running app is ad-hoc signed, even though epic
bob-cli-84 shipped stable signing (phase 84.1, bob-mac-capture `1cd958e`).

Evidence gathered on the Mac (read-only, 2026-10-16 10:29):

- `codesign -dv ~/Applications/"Bob Mac Capture.app"` shows `Signature=adhoc`, and
  `codesign -d -r-` shows `designated => cdhash H"d41475a8…"`.
- `security find-identity -p codesigning` finds **0 identities**, and
  `security find-certificate -a -c "Bob Mac Capture"` finds nothing.
- The running app (started 10:23:36 with `--bob-mac-capture-install-restart`) was
  relaunched by the install helper through `/usr/bin/open`. LaunchServices therefore
  made it its own TCC responsible process, so a terminal's grant is not being checked
  instead.

**Root cause.** bob-cli's `scripts/install_all` installs the Mac app with:

```bash
just --justfile "$mac_capture_dir/justfile" install "$HOME/Applications" "${CODESIGN_IDENTITY:--}"
```

When `CODESIGN_IDENTITY` is unset, `${CODESIGN_IDENTITY:--}` becomes an explicit `-`.
bob-mac-capture's `Scripts/install.sh` treats an explicit identity as authoritative,
including `-` (rule 1 of the 84.1 resolver). So it never looks for the stable "Bob Mac
Capture Local" identity, and it also skips the loud ad-hoc warning, which only prints
when the identity was resolved from the keychain. That line predates the epic (bob-cli
`2af9fbe5`). Epic A11 said "no bob-cli change", so the 84.1 phase changed only the
bob-mac-capture `justfile` default and never touched this caller.

Each `just install-all` therefore signs a fresh ad-hoc build with a new `cdhash`. macOS
pins an ad-hoc app's Accessibility grant to the `cdhash` it was granted for. After every
reinstall, `AXIsProcessTrusted()` returns false while System Settings still shows the
switch on. That is exactly "I already granted it".

**Compounding cause.** The keychain holds no "Bob Mac Capture Local" identity: the
epic's manual step 1 was never done. Even with `install_all` fixed, `just install` would
fall back to ad-hoc until that identity exists. That part is a one-time manual step for
Bryan (see "Bryan's one-time steps").

**A recovery-path gap.** The card's **Open Accessibility Settings** button only calls
`AXIsProcessTrustedWithOptions(prompt: true)`. When Bob Mac Capture already has an entry
in the Accessibility list (a stale switch that looks on), macOS typically shows no
prompt, so the button does nothing visible. The signed-build card also gives no hint for
a stale entry. The first stable-signed install will hit exactly that case, because the
existing entry is pinned to an old ad-hoc `cdhash`.

Rejected alternative: ad-hoc signing with an explicit `designated => identifier "…"`
requirement. It is unreliable for TCC, and it would grant Accessibility to any ad-hoc
binary claiming the bundle id. The epic's research (E13/E14) already chose a stable
self-signed identity.

## Work

Two repos change: bob-cli (this repo) and the linked `bob-mac-capture` repo. Open the
latter with `/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`), read its
`AGENTS.md`, and commit in both repos.

### 1. bob-cli `scripts/install_all`: stop forcing ad-hoc (the fix)

In `mac_capture_step`, run the Mac install **without an identity argument**:

```bash
just --justfile "$mac_capture_dir/justfile" install "$HOME/Applications"
```

Update the matching `command_line` echo. The recipe's empty default identity makes
`install.sh` resolve the identity. `install.sh` itself still honors `CODESIGN_IDENTITY`
from the environment, so an explicit `CODESIGN_IDENTITY` (including `-`) still wins. Do
not re-add `${CODESIGN_IDENTITY:--}` anywhere.

### 2. bob-cli `scripts/install_all`: make an ad-hoc install a visible warning

After a successful install (status `ok`, or the exit-72 `warn` path), inspect the
installed bundle and warn when it is ad-hoc:

- Read `"$HOME/Applications/Bob Mac Capture.app"` with `/usr/bin/codesign -dv`. It
  reports on stderr even on success. Capture the output into a variable with
  `2>&1 || true` rather than piping into `grep -q`, because under `set -o pipefail` a
  `grep -q` early exit can SIGPIPE `codesign` and flip the status. Match
  `Signature=adhoc`.
- Warn only when `CODESIGN_IDENTITY` is unset or empty. An explicit `-` is a deliberate
  choice.
- On an ad-hoc result, bump `step_warning`, so the step lands as `warn` (⚠) rather than
  `ok`. Use a concise detail, for example
  `$pull_detail · installed ad-hoc · Accessibility grant resets each install · see bob-mac-capture README › Signing`.
  This keeps the existing "installed; restart it via…" wording on the 72 path and
  appends the ad-hoc note.
- Keep the check in a small function (for example `mac_capture_is_adhoc`), matching the
  script's helper style. A warning never fails the run or changes the exit status.

### 3. bob-cli README "Install everything"

Replace "with `just install`, honoring `CODESIGN_IDENTITY`" with what now happens:

- `just install-all` passes no identity, so Bob Mac Capture's installer signs with the
  stable "Bob Mac Capture Local" identity when the login keychain holds it;
- an explicit `CODESIGN_IDENTITY` (including `-`) still wins;
- an ad-hoc result is a ⚠ warning, because it resets the app's Accessibility grant, and
  the bob-mac-capture README › Signing explains how to create the identity.

Keep the paragraph's existing style.

### 4. bob-mac-capture: make "Open Accessibility Settings" always land somewhere

- In `Sources/BobMacCapture/AccessibilityTrust.swift`:
  - add a documented `static let settingsURL`, the Accessibility pane URL
    `x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility` (the
    same scheme `SettingsView` already uses for notifications);
  - add `static func openSettings()`, which opens that URL through
    `NSWorkspace.shared.open` (import AppKit).

  It stays the only home for trust calls.

- In `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift`, `requestTrust()` calls
  `AccessibilityTrust.requestPrompt()`. When that returns false, it also calls
  `AccessibilityTrust.openSettings()`. Update the doc comment. The
  `HighlightsTabsReading` protocol, the model's `.requestTabsPermission`, and
  `UnavailableHighlightsTabsReader` stay unchanged. Nothing new runs at launch.
- Tests: in `Tests/BobMacCaptureTests/AccessibilityTrustTests.swift`, assert the
  `settingsURL` string.

### 5. bob-mac-capture: a stale-grant hint on the permission card

- Add `AccessibilityTrust.permissionHint(isAdHocBuild:) -> String`:
  - ad-hoc keeps today's text exactly: "Grants reset when an ad-hoc build is
    reinstalled. Re-add the app, or run
    `tccutil reset Accessibility org.bobs.bob-mac-capture`.";
  - signed builds get "Already on in System Settings? Remove Bob Mac Capture there and
    add it again."
- In `Sources/BobMacCapture/Refs/RefsTabsViews.swift`, `RefsTabsPermissionCard` shows
  that hint for both variants, below the main line, and appends it to the accessibility
  label. This replaces the ad-hoc-only branch and its duplicated literal.
- Tests: add both variants to `AccessibilityTrustTests`. The existing
  `RefsPanelDesignTests` permission renders (ad-hoc and signed) must still pass
  unchanged.

### 6. bob-mac-capture README

- **Development › Signing:** add one sentence saying bob-cli's `just install-all`
  installs the same way (no identity, so the same resolution).
- **Bob Refs › Tabs › Troubleshooting:** note that Open Accessibility Settings now opens
  the Accessibility pane, and that a switch that already looks on after an identity
  change must be removed and re-added (or `tccutil reset …`).

Keep edits small and in the README's voice.

> [!decision] identity_script Add `Scripts/create-signing-identity.sh`, which Bryan runs
> once in a GUI Terminal.
>
> - It refuses when `security find-certificate -c "Bob Mac Capture Local"` already finds
>   the identity.
> - It generates a key and a certificate with `openssl`: CN `Bob Mac Capture Local`,
>   `extendedKeyUsage=codeSigning`, `basicConstraints=critical,CA:false`, and a ten-year
>   validity.
> - It exports a PKCS#12, adding `-legacy` when OpenSSL 3 needs it, and imports it into
>   the login keychain with `security import … -T /usr/bin/codesign`.
> - It deletes its temp files through a `trap`.
>
> The README › Signing documents it first, with the Keychain Access steps as a fallback.
> Agents may run `shellcheck` and `bash -n` on it, but must never execute it.

## Verification (implementer)

- bob-cli: run `just check-scripts` (`bash -n`). Run `shellcheck scripts/install_all` on
  athena, which has it; this host may not. Run `git diff --check`. `install_all`'s Mac
  step only runs on Darwin, and agents must never install on the Mac, so do not run it
  there.
- bob-mac-capture: run `just portable-check` on athena (lint first, then build and
  test). The Swift changes are in `Sources/BobMacCapture*`/`Tests/BobMacCapture*`, so
  compile a scratch copy on the Mac (`ssh mac`, best effort):
  1. rsync without `.build`/`.git` into a temp directory;
  2. run `Scripts/bundle.sh` (ad-hoc default) and `Scripts/typecheck-tests.sh`;
  3. delete the copy.

  Never use Bryan's checkout or `just install`, and never call trust or prompt APIs over
  SSH. The Mac is often offline (it was by the end of diagnosis). If it is, say in the
  final report that the macOS targets are unverified. CI runs the XCTest suites.

## Bryan's one-time steps (after this lands; GUI Mac only)

1. Create the "Bob Mac Capture Local" identity per bob-mac-capture README › Development
   › Signing (Keychain Access › Certificate Assistant: Self-Signed Root, Code Signing,
   login keychain), or with the script when `identity_script` is accepted.
2. From a Terminal on the Mac (tmux is fine if its server started in the GUI session,
   not over SSH), run `just install-all` in bob-cli. Choose **Always Allow** at
   codesign's keychain prompt. The summary's bob-mac-capture line should be ✓, with no
   ad-hoc warning.
3. `codesign -d -r- ~/Applications/"Bob Mac Capture.app"` shows
   `certificate leaf = H"…"`, not a `cdhash`.
4. Drop the stale ad-hoc grant with
   `tccutil reset Accessibility org.bobs.bob-mac-capture`.
5. In Highlights press ⇧⌘O, then click **Open Accessibility Settings**. Switch Bob Mac
   Capture on, then press ⌘R (or reopen the panel): tabs list. Settings › References ›
   Highlights tabs reads **Allowed**.
6. Run `just install-all` again. Highlights tabs still reads **Allowed**, with no new
   grant.

## Out of scope

- The 84.1 phase's install.sh resolver and the epic's other behaviour stay as they are.
- bob-mac-capture's README says an SSH install "falls back to ad-hoc". Once an identity
  exists, `find-identity` may still list it over SSH while `codesign` fails with
  `errSecInternalComponent`, because the keychain is locked. That fails safely before
  the installed copy is touched. If Bryan's verification hits it, file a follow-up
  rather than widening this tale.
