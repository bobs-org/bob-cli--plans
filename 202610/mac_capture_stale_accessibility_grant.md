---
tier: tale
title: Make Bob Mac Capture's Accessibility grant stick
goal:
  Bob Refs Tabs works once Bryan grants Accessibility, and the grant survives later
  installs.
size: medium
decisions:
  identity_setup:
    ask:
      Should `just install` create the "Bob Mac Capture Local" signing identity itself
      when the login keychain lacks it?
    choices:
      auto:
        The first GUI-Terminal install creates it unattended, so install-all needs no
        manual setup
      command:
        Installs never touch the keychain; Bryan runs `just signing-identity` once by
        hand
    default: auto
    why:
      the manual Keychain Access step never happened, and every ad-hoc install breaks
      Tabs again
    answer: auto
proposed_by: bbugyi200.apollo.80.f0
decided_by: auto
create_time: 2026-10-16 11:03:21
status: wip
---

# Plan: Make Bob Mac Capture's Accessibility grant actually stick

## Diagnosis

The screenshot shows Bob Mac Capture switched **on** in Privacy & Security ›
Accessibility, yet Bob Refs › Tabs still shows the permission card. The Mac's own logs
explain why. Evidence was gathered read-only over `ssh mac` on 2026-10-16 between 10:55
and 11:05.

- **The app is still ad-hoc.** `codesign -dv ~/Applications/"Bob Mac Capture.app"` shows
  `Signature=adhoc` with `designated => cdhash H"96fb6281…"`.
  `security find-identity -p codesigning` finds 0 identities, so no "Bob Mac Capture
  Local" identity exists. The previous fix (bob-cli `bc27d528`, bob-mac-capture
  `3300128`) works as designed: install-all now resolves the identity and prints the ⚠
  "installed ad-hoc" warning visible in the screenshot's terminal. But the identity it
  resolves to was never created. That is the manual Keychain Access step the epic left
  to Bryan.
- **The switch Bryan turned on is pinned to an old build.** tccd logs
  `Failed to match existing code requirement for subject org.bobs.bob-mac-capture and service kTCCServiceAccessibility`,
  with stored `cdhash H"c5dfd928…"` against running `cdhash H"96fb6281…"`. The row was
  last granted at 09:32–09:33 for build `c5dfd928`. Every check since 10:23 fails the
  match.
- **Toggling never re-pins it.** Bryan's switch flips at 10:24:51, 10:24:53, 10:50:57,
  10:50:59, 10:51:52 and 10:52:10 each publish a `Modify` event for that same row. A
  toggle changes the row's allowed bit, but it keeps the stale code requirement, so
  `AXIsProcessTrusted()` stays false while the switch looks on.
- **The card's button cannot escape the trap.** At 10:52:14 **Open Accessibility
  Settings** made two `TCCAccessRequest` calls, and both failed the same match. macOS
  shows no prompt while a row exists, so the button only reopens the pane showing the
  misleading switch.

**Root cause.** Every install is ad-hoc with a fresh `cdhash`, because the stable
identity depends on a manual step nobody performed. Each install leaves the
Accessibility row pinned to an older `cdhash`, and macOS keeps showing that dead row as
"on". Only removing and re-adding the row (or `tccutil reset`) re-pins it. Neither the
installer nor the app does that, and the app's hint still asks Bryan to do it by hand.

**Probes that de-risk the fix** (run on the Mac in a throwaway keychain and a scratch
binary; the keychain search list was restored afterwards):

- A scripted identity works:
  1. `/usr/bin/openssl` (LibreSSL 3.3.6) `req -x509` creates the certificate with
     `basicConstraints=critical,CA:false`, `keyUsage=critical,digitalSignature`, and
     `extendedKeyUsage=critical,codeSigning`.
  2. `openssl pkcs12 -export` bundles it with LibreSSL's default ciphers.
  3. `security import … -T /usr/bin/codesign` imports it.
  4. `security find-identity -p codesigning` (no `-v`) then lists it as
     `"…" (CSSMERR_TP_NOT_TRUSTED)`, which install.sh's existing matcher accepts.
  5. `codesign -f -s "<name>"` signs with it, with no trust setting needed, and produces
     `designated => identifier "…" and certificate leaf = H"<sha1>"`, which stays stable
     across rebuilds.
- Over SSH the login keychain is unreachable:
  `security show-keychain-info ~/Library/Keychains/login.keychain-db` exits 36 with
  "User interaction is not allowed".
- `/usr/bin/tccutil` carries `com.apple.private.tcc.manager.access.delete`
  (`kTCCServiceAll`), so a non-root `tccutil reset Accessibility <bundle-id>` works. It
  exits 64 for a bundle id that LaunchServices does not know.

Rejected alternatives:

- Ad-hoc signing with an explicit `identifier "…"` designated requirement. It is
  unreliable for TCC, and it would trust any ad-hoc binary that claims the bundle id.
  The epic research (E13/E14) already rejected it.
- Writing the TCC database directly. It needs Full Disk Access and SIP-protected paths.

## Work

Two repos change: the linked `bob-mac-capture` repo and bob-cli (this repo). Open the
former with `/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`), read its
`AGENTS.md`, and commit in both repos.

### 1. bob-mac-capture: `Scripts/create-signing-identity.sh` (new) and `just signing-identity`

Add a standalone, idempotent script (`set -euo pipefail`, `-h` usage) that creates the
stable identity in the login keychain. Mirror the probe recipe above exactly.

- `name="Bob Mac Capture Local"` and
  `keychain="${HOME}/Library/Keychains/login.keychain-db"`.
- If `security find-identity -p codesigning` already lists `"${name}"` (the same quoted
  match as `install.sh`'s `find_stored_signing_identity`), print that it exists and
  exit 0.
- If `security show-keychain-info "${keychain}"` fails, print that the login keychain is
  locked or unreachable and to run the script from a GUI Terminal on the Mac, then
  exit 75. Do this check up front so the script never hangs on an SSH session.
- Work in `mktemp -d` and remove it via `trap … EXIT`.
- Use `/usr/bin/openssl` explicitly, never a `PATH` lookup. LibreSSL's default PKCS#12
  ciphers import cleanly, while Homebrew's OpenSSL (4.x on this Mac) needs `-legacy`
  handling. Use a small config file rather than `-addext`:
  - CN `Bob Mac Capture Local`;
  - `basicConstraints = critical,CA:false`;
  - `keyUsage = critical,digitalSignature`;
  - `extendedKeyUsage = critical,codeSigning`;
  - `subjectKeyIdentifier = hash`;
  - `rsa:2048`, `-nodes`, `-days 3650`.
- `pkcs12 -export -name "${name}"`, with a random throwaway password from
  `/usr/bin/openssl rand -hex 16`.
- `security import "$p12" -k "${keychain}" -f pkcs12 -P "$pass" -T /usr/bin/codesign`.
- Re-run the `find-identity` match. Fail with exit 1 if the identity is still missing.
  Otherwise print its SHA-1, and say that the first signed install may show a `codesign`
  keychain prompt where Bryan should choose **Always Allow**.
- Never add trust settings, never call `set-key-partition-list`, and never ask for the
  login password.

Add a `justfile` recipe `signing-identity: ./Scripts/create-signing-identity.sh` with a
one-line comment. Agents may run `bash -n` and `shellcheck` on the script, but must
never execute it on the Mac.

### 2. bob-mac-capture `Scripts/install.sh`: identity bootstrap

> [!decision] identity_setup = auto
>
> After the `--print-resolved-identity` early exit, which must stay read-only and never
> create anything, handle the case where the identity was resolved from the keychain
> (`resolved_from_keychain=true`) and came back `-`. In that case, run
> `Scripts/create-signing-identity.sh` with stdout sent to stderr, the way install.sh
> sends its other subcommand output. If it succeeds, re-resolve with
> `find_stored_signing_identity`.
>
> `package_root` is currently defined later in the script, so compute it (or the script
> directory) before this point. A creation failure (exit 75 over SSH, or any other)
> never fails the install; it falls through to the ad-hoc warning.
>
> An explicit `--identity`/`CODESIGN_IDENTITY` (including `-`, which CI uses) never
> triggers creation.

> [!decision] identity_setup = command
>
> install.sh never creates the identity. Only the warning text changes, as below.

In both branches, keep the existing ad-hoc warning, but make it actionable. It should
say that the "Bob Mac Capture Local" identity is missing, that the Accessibility grant
resets on this install, and how to fix that:

- for `auto`: install again from a GUI Terminal on the Mac, which creates the identity;
- for `command`: run `just signing-identity` once from a GUI Terminal, then install
  again.

Replace the "Created once by Bryan in Keychain Access; never by this script" comment
with the truth for the chosen branch. Update the `justfile` comment above `install` the
same way.

### 3. bob-mac-capture `Scripts/install.sh`: clear a dead Accessibility row when the signature changes

This keeps System Settings from showing a switch that cannot work.

- Add a helper `designated_requirement <path>`. It captures
  `/usr/bin/codesign -d -r- "<path>" 2>&1 || true` into a variable, then prints the text
  after `designated => ` on the matching line. Ad-hoc output prefixes that line with
  `# `, so strip that prefix. Match with bash patterns or `sed`. Do not pipe `codesign`
  into `grep -q`, because under `pipefail` an early `grep` exit can SIGPIPE it.
- Capture `previous_requirement` from the existing `${install_path}` before it is moved
  to the backup path. Leave it empty when nothing is installed.
- After the post-install verification succeeds and the backup is removed, and **before**
  the restart handoff, compute the new requirement. When both are non-empty and differ:
  - run `/usr/bin/tccutil reset Accessibility org.bobs.bob-mac-capture`, sending its
    output to stderr;
  - on success, print one stderr line saying the Accessibility grant was cleared because
    the code signature changed, and to grant it again from Bob Refs › Tabs › Open
    Accessibility Settings;
  - on failure, print one stderr line with the manual `tccutil` command.

  The reset is best effort. It must never trip `set -e` or change the exit status, and
  stdout stays the single install-path line.

- Net effect:
  - ad-hoc reinstalls clear the dead row, which was already dead;
  - the first stable-signed install clears today's stale `c5dfd928` row;
  - later stable installs keep the same `certificate leaf` requirement, so the grant
    survives.
- CI installs with `--identity "-"` into a temp `HOME`. The reset may run there, and it
  must stay harmless and non-fatal. Do not change `.github/workflows/ci.yml` unless a
  gate actually breaks.

### 4. bob-mac-capture app: the permission button repairs a stale grant

- `Sources/BobMacCapture/AccessibilityTrust.swift` (still the only home for trust
  calls):
  - add `static let bundleIdentifier = "org.bobs.bob-mac-capture"`;
  - add `static let resetArguments = ["reset", "Accessibility", bundleIdentifier]`;
  - add `@discardableResult static func resetGrant() -> Bool`. It runs
    `/usr/bin/tccutil` with `resetArguments` through `Process`, discards its output,
    waits, and returns `terminationStatus == 0`. Model it on
    `BundleSigningInspector.currentBundleState()`. Document it as gesture-only, called
    only while untrusted.
  - Update the type doc comment, which currently lists only `isTrusted`/`requestPrompt`.
- `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift` `requestTrust()`:
  1. return early if `AccessibilityTrust.isTrusted()`;
  2. otherwise call `AccessibilityTrust.resetGrant()`, then `requestPrompt()`;
  3. call `openSettings()` when that returns false.

  With the dead row gone, macOS shows its real "would like to control this computer"
  prompt and adds a fresh row pinned to the running build. Update the doc comment. The
  `HighlightsTabsReading` protocol, the model's `.requestTabsPermission`, and the fakes
  stay unchanged. Nothing new runs at launch.

- `AccessibilityTrust.permissionHint(isAdHocBuild:)` now describes the button instead of
  manual steps, with no backticks (`Text(String)` renders them literally):
  - ad-hoc: "Grants reset when an ad-hoc build is reinstalled. Open Accessibility
    Settings clears the stale grant; then switch Bob Mac Capture on."
  - signed: "Open Accessibility Settings clears any stale grant; then switch Bob Mac
    Capture on."
- Tests in `Tests/BobMacCaptureTests/AccessibilityTrustTests.swift`:
  - assert `resetArguments` equals
    `["reset", "Accessibility", "org.bobs.bob-mac-capture"]`;
  - update both hint assertions.

  The `RefsPanelDesignTests` permission renders (ad-hoc and signed) must still pass.

### 5. bob-mac-capture README

- **Development › Signing:**
  - for `auto`, the first `just install` (or bob-cli `just install-all`) from a GUI
    Terminal creates the identity automatically;
  - for `command`, run `just signing-identity` once from a GUI Terminal;
  - keep the Keychain Access steps as a manual fallback;
  - say that an install whose signature changed clears the stale Accessibility row, so
    Bryan grants once more from Tabs;
  - keep the Always Allow, SSH-locked-keychain, and `codesign -d -r-` verification
    notes.
- **Bob Refs › Tabs › Troubleshooting:**
  - **Open Accessibility Settings** now clears a stale grant and brings up the system
    prompt;
  - a switch that looks on but does nothing is a row pinned to an older build;
  - the button, or `tccutil reset Accessibility org.bobs.bob-mac-capture`, fixes it.

Keep edits small and in the README's voice.

### 6. bob-cli `scripts/install_all` and README

- In `mac_capture_step`, keep the `mac_capture_is_adhoc` check. Change the ad-hoc detail
  so its fix matches the decision branch while staying one concise line:
  - `auto`:
    `… · installed ad-hoc · Accessibility grant resets each install · rerun from a GUI Terminal to create the signing identity`
  - `command`:
    `… · installed ad-hoc · Accessibility grant resets each install · run just signing-identity in bob-mac-capture once`
- In the README "Install everything" paragraph, replace the "see the bob-mac-capture
  README › Signing for identity setup" clause with the branch's behavior. Also mention
  that an install whose signature changed clears Bob Mac Capture's stale Accessibility
  grant.

## Verification (implementer)

- bob-cli:
  - `just check-scripts`;
  - `shellcheck scripts/install_all` (on athena if this host lacks it);
  - `git diff --check`.

  `install_all`'s Mac step runs only on Darwin; do not run it.

- bob-mac-capture shell:
  - `bash -n` and `shellcheck` on `Scripts/install.sh` and
    `Scripts/create-signing-identity.sh`;
  - exercise `designated_requirement` parsing against both sample shapes:
    `# designated => cdhash H"…"` and
    `designated => identifier "org.bobs.bob-mac-capture" and certificate leaf = H"…"`.
- bob-mac-capture Swift:
  - `just portable-check` on athena;
  - then, best effort over `ssh mac`, a scratch copy (rsync without `.build`/`.git` into
    a temp dir) running `Scripts/bundle.sh` (ad-hoc default) and
    `Scripts/typecheck-tests.sh`; delete the copy afterwards.
- Never touch Bryan's checkout, never run `just install` or `create-signing-identity.sh`
  on the Mac, and never call `tccutil`, trust, or prompt APIs over SSH. If the Mac is
  offline, report the macOS targets as unverified.
- Commit both repos via `/sase_git_commit`.

## Bryan's steps after this lands (GUI Mac only)

1. Immediate unblock, even before the fix lands:
   1. run `tccutil reset Accessibility org.bobs.bob-mac-capture` in a Mac Terminal;
   2. reopen Bob Refs › Tabs and press **Open Accessibility Settings**;
   3. allow the system prompt and switch Bob Mac Capture on.

   This holds until the next ad-hoc install.

2. From a GUI Terminal on the Mac, run `just install-all` in bob-cli:
   - with `auto`, it creates the identity;
   - with `command`, first run `just signing-identity` in bob-mac-capture.

   Choose **Always Allow** at `codesign`'s keychain prompt. The bob-mac-capture summary
   line should be ✓, with no ad-hoc warning.

3. The installer clears the old row. Press **Open Accessibility Settings** in Tabs once,
   allow the prompt, and switch the app on.
   `codesign -d -r- ~/Applications/"Bob Mac Capture.app"` should show
   `certificate leaf = H"…"`. Later installs keep the grant.
