---
tier: tale
title: Sign or stop cleanly when the login keychain is locked
goal:
  just install either signs with the stable identity or stops before building with an
  actionable fix.
size: medium
decisions:
  locked_keychain:
    ask:
      Should the installer offer to unlock a locked login keychain when the stable
      identity exists?
    choices:
      unlock:
        Prompts via security unlock-keychain in a terminal, then retries; otherwise
        stops with exit 77
      fail: Never prompts; always stops before building with exit 77 and the fixes
      adhoc:
        Falls back to ad-hoc with a warning; the Accessibility grant resets on that
        install
    default: unlock
    why:
      SSH/tmux installs work after one password prompt and keep the stable signature and
      grant
    answer: unlock
proposed_by: bbugyi200.apollo.8n
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.8n](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.8n.md)
- **COMMITS:**
  - [acb2563](https://github.com/bobs-org/bob-mac-capture/commit/acb256321bc00d13996a6489964c3bc8e8824003)
    — fix: check signing identity access before install

# Plan: Make `just install` sign or stop cleanly when the login keychain is locked

## Diagnosis

Bryan's failing output is step 3/3 of bob-cli's `just install-all`. It runs
`just --justfile …/bob-mac-capture/justfile install ~/Applications`, which runs
`Scripts/install.sh --target ~/Applications --identity ""`.

1. **Identity resolution succeeded.** The output has no ad-hoc warning and no
   `create-signing-identity.sh` message. So `find_stored_signing_identity` found "Bob
   Mac Capture Local" in `security find-identity -p codesigning`. An earlier GUI install
   created that identity after bob-mac-capture `8e86b32` landed.
2. **codesign found the identity but could not use its private key.** In `bundle.sh`,
   the first `codesign --force --sign "Bob Mac Capture Local"` printed
   `replacing existing signature`, because SwiftPM's linker had already ad-hoc signed
   the binary. It then failed with `errSecInternalComponent`.
3. **It is not a trust or chain problem.** There is no
   `unable to build chain to self-signed root` warning, which is what an untrusted or
   incomplete chain prints. The probes in
   `plan:202610/mac_capture_stale_accessibility_grant.md` showed that this exact
   certificate recipe signs fine while untrusted (`CSSMERR_TP_NOT_TRUSTED`), provided
   its keychain is unlocked.
4. **Apple DTS names a locked keychain as the usual cause.** Quinn's "Resolving
   errSecInternalComponent errors during code signing" (Apple Developer Forums
   thread 712005) puts it this way. Over SSH, the login keychain stays locked: "Logging
   in via SSH does not unlock the keychain". The other cause is a key ACL that needs a
   confirmation nobody can click. The earlier plan already measured the SSH case:
   `security show-keychain-info ~/Library/Keychains/login.keychain-db` exits 36 with
   "User interaction is not allowed".
5. **tmux can inherit the same session.** A tmux server first started over SSH stays in
   that non-GUI launchd session even when a GUI terminal attaches to it later.
   `launchctl managername` prints `Background` there and `Aqua` in a GUI session.

**Root cause.** `install.sh` picks the stable identity whenever `security find-identity`
lists it. That command lists identities whether or not this session can unlock their
private keys. The signing design, and README › Signing, handled SSH only while the
identity was missing ("the install falls back to ad-hoc"). Once a GUI install created
the identity, every install from a session with a locked login keychain goes wrong. That
includes SSH sessions and tmux servers first started over SSH. The install resolves the
stable identity, spends the build, and then dies inside `bundle.sh` with an opaque
`errSecInternalComponent`. Nothing says why or how to fix it.

The Mac was offline while this plan was written, so the session type of Bryan's run is
inferred. Step 1 confirms it.

Rejected alternatives:

- **`security set-key-partition-list`, or passing the login password with `-p`.** Either
  one puts the password in a script. The import already trusts `/usr/bin/codesign` via
  `-T`, and the earlier probe showed codesign signs without a partition list once the
  keychain is unlocked. Unlocking is the missing piece.
- **A dedicated always-unlocked keychain, or the System keychain.** This would leave the
  key that the Accessibility grant trusts permanently usable by any process. Moving the
  identity would also change it, which resets the grant again.
- **A silent ad-hoc fallback.** This is the `adhoc` branch, not the default. It brings
  back the grant churn the stable identity exists to prevent.

## Work

Two repos change. One is the linked `bob-mac-capture` repo: open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<why>"`) and read its `AGENTS.md`. The other is
bob-cli, this repo. Commit both. No Swift changes.

### 1. Confirm the diagnosis on the Mac (best effort, read-only)

If `ssh -o BatchMode=yes -o ConnectTimeout=10 mac` connects, run the following. Use no
`-t` and take stdin from `/dev/null`, so nothing can prompt.

- `security find-identity -p codesigning`. Count the `"Bob Mac Capture Local"` lines
  under "Matching identities". Expect one.
- `security show-keychain-info ~/Library/Keychains/login.keychain-db; echo $?`. Expect
  `36`.
- `launchctl managername`. Expect `Background`.
- Quinn's private-key test, which signs only a temp copy:
  `d="$(mktemp -d)"; cp /usr/bin/true "$d/probe"; /usr/bin/codesign --force --sign "Bob Mac Capture Local" --timestamp=none "$d/probe"; echo $?; rm -rf "$d"`.
  Expect `errSecInternalComponent`, the same error as Bryan's run.
- `codesign -d -r- ~/Applications/"Bob Mac Capture.app"`. This shows whether an earlier
  GUI install produced a stable `certificate leaf = H"…"` signature.

Put these results in the final report. The pre-build check below is correct whatever the
cause, so implement steps 2–5 in every case. If the probe signs over SSH, or more than
one identity is listed, say plainly in the final report that the diagnosis is
incomplete.

Over SSH, never run `security unlock-keychain`, `just install`, `install.sh` without
`--check-signing`/`--print-resolved-identity`, or `create-signing-identity.sh`. Never
touch anything in Bryan's checkout.

### 2. bob-mac-capture `Scripts/install.sh`: check the identity before building

- **Keychain path.** Add `keychain="${HOME}/Library/Keychains/login.keychain-db"` next
  to `stable_identity_name`, with a one-line comment.
- **`probe_signing_identity <identity>`.** This is Quinn's private-key test.
  - Make a `mktemp -d` dir, copy `/usr/bin/true` to `probe` inside it, and run
    `/usr/bin/codesign --force --sign "$1" --timestamp=none "<dir>/probe"`.
  - Capture codesign's combined stdout and stderr in a global `probe_output`. Initialize
    it to `""` at the top level for `set -u`.
  - Remove the dir and return codesign's status.
  - Call codesign inside an `if` so `set -e` cannot fire. Never pipe codesign into
    `grep -q`.
  - Add a comment explaining why: `find-identity` lists identities even when this
    session cannot use their private keys.
- **`login_keychain_reachable`.** This is
  `/usr/bin/security show-keychain-info "${keychain}" >/dev/null 2>&1`, the same check
  `create-signing-identity.sh` already makes.
- **`ensure_signing_identity_usable <identity>`.** Return 0 when the probe signs.
  Otherwise, handle the failure as follows.

> [!decision] locked_keychain = unlock
>
> If `! login_keychain_reachable` and `[[ -t 0 ]]`:
>
> 1. Print one stderr line: the login keychain is locked in this session (SSH, or a tmux
>    server started over SSH), so unlock it to sign with "<identity>"; macOS asks for
>    the login password.
> 2. Run `/usr/bin/security unlock-keychain "${keychain}" || true` with no `-p`. Apple's
>    tool reads the password from the terminal, and the script never sees or stores it.
> 3. Re-probe. Return 0 on success.
>
> If the probe still fails, or the shell has no terminal, print the failure report and
> `exit 77`.

> [!decision] locked_keychain = fail
>
> Never prompt or unlock. Print the failure report and `exit 77`. Drop the
> `security unlock-keychain` prompt; keep it only as the first fix the report lists.

> [!decision] locked_keychain = adhoc
>
> Never prompt or unlock.
>
> - **Stable identity resolved from the keychain** (`resolved_from_keychain=true`):
>   print the failure report's first part and a loud WARNING. The warning says the
>   install falls back to ad-hoc and the Accessibility grant resets on this install and
>   again on the next stable one. Then set `identity="-"` and continue.
> - **Explicit `--identity`/`CODESIGN_IDENTITY`**: still print the full report and
>   `exit 77`. An explicit identity is never replaced.

**Failure report.** Print to stderr in the README's voice, short lines, no backticks
needed:

- `codesign cannot use the "<identity>" signing identity from this session:` followed by
  `probe_output`, every line indented four spaces.
- **When the login keychain is unreachable**, say the following:
  - the keychain is locked here, giving the launchd session
    (`/bin/launchctl managername 2>/dev/null || printf 'unknown'`) and noting that a GUI
    Terminal reports `Aqua`;
  - this happens over SSH and in a tmux server first started over SSH;
  - nothing was built or installed.

  Then list the fixes:
  1. run `security unlock-keychain ~/Library/Keychains/login.keychain-db`, then rerun
     from this same shell;
  2. rerun from a GUI Terminal window that is not attached to an SSH-started tmux;
  3. install ad-hoc with `just install ~/Applications -`, which resets the Accessibility
     grant.

- **When the login keychain is reachable**, say the following:
  - codesign was refused the identity's private key, or matched more than one identity;
  - run the install from a GUI Terminal and answer codesign's keychain prompt with
    **Always Allow**;
  - list the `security find-identity -p codesigning` lines that contain `"<identity>"`
    (bash pattern matching on captured output, never `grep -q` on a pipe);
  - if more than one copy appears, delete the extras in Keychain Access › login › My
    Certificates and rerun.

**Wiring:**

- **Install path.** Run the check when `identity != "-"`, after the target-dir
  validation and before `previous_requirement` and `bundle.sh`. A failed check then
  builds nothing and touches no install path. Explicit identities get the same check.
- **`--check-signing` flag.** Add it, and document it and exit status 77 in `usage`.
  - It never creates the identity and never builds or installs.
  - After resolution and the `--print-resolved-identity` early exit, it runs the check
    when `identity != "-"`, prints the resolved identity (or `-`) on stdout, and
    exits 0. A failed check exits 77 (or, for the `adhoc` branch's resolved case, prints
    the warning and `-` and exits 0).
  - When the stable identity is missing, it prints one stderr note that `just install`
    from a GUI Terminal creates it.
- **Unchanged behavior:**
  - `--print-resolved-identity` stays read-only, with no probe;
  - install-mode stdout stays the single install-path line, so every new message goes to
    stderr;
  - CI's `--identity "-"` installs never probe.

### 3. bob-mac-capture `justfile`

- Add a recipe:

  ```just
  # Checks, without building or installing, that codesign can use the install
  # signing identity from this session.
  signing-check identity="":
      ./Scripts/install.sh --check-signing --identity "{{identity}}"
  ```

  In the `unlock` branch, also say the recipe offers to unlock a locked login keychain.

- Extend the comment above `install` with one clause: the install checks that the
  identity is usable before building, and the comment says what it does when the
  identity is locked, following the chosen branch.

### 4. bob-mac-capture README › Signing

Replace the paragraph "Always run `just install` from a GUI Terminal session on the Mac.
Over SSH the login keychain is locked, …" with text that is accurate for the chosen
branch.

- Prefer a GUI Terminal on the Mac.
- Over SSH, or in a tmux server first started over SSH, the login keychain is locked.
  `launchctl managername` prints `Background` there instead of `Aqua`.
- While the identity is missing, setup cannot run and the install falls back to ad-hoc.
- Once the identity exists, the install checks before building that codesign can use it.
  - `unlock`: in a terminal it asks to unlock the login keychain through
    `security unlock-keychain`; otherwise it stops with exit 77 and the fixes.
  - `fail`: it stops with exit 77 and the fixes.
  - `adhoc`: it falls back to ad-hoc with a warning.
- `just signing-check` runs the same check without building.
- Mention that `errSecInternalComponent` from codesign means this locked-keychain case.

Keep the Always Allow note and the `codesign -d -r-` verification.

### 5. bob-cli `scripts/install_all` and README

- **Exit 77 in `mac_capture_step`.** In the install failure branch, add an
  `elif (( install_status == 77 ))` before the generic failure. It stays a failure, with
  the detail
  `"$pull_detail · signing identity locked in this session · run security unlock-keychain or install from a GUI Terminal"`.
- **`adhoc` branch wording.** The existing ad-hoc warning detail can now also fire when
  the identity exists but is locked. Reword its tail to
  `· rerun from a GUI Terminal or unlock the login keychain to sign with the stable identity`.
- **README "Install everything".** Add one sentence matching the branch. For `unlock`:
  from a session whose login keychain is locked (SSH, or a tmux server started over
  SSH), the Bob Mac Capture step asks to unlock it before building, and otherwise fails
  with that hint.

## Verification (implementer)

- **bob-mac-capture shell:**
  - `bash -n Scripts/install.sh`;
  - `shellcheck Scripts/install.sh`, on athena if this host lacks shellcheck;
  - `git diff --check`.
- **Mac, best effort.** Over `ssh mac`, with no `-t` and stdin from `/dev/null`, so the
  unlock prompt never runs. rsync a scratch copy into a temp dir, leaving out `.build`
  and `.git`. In the copy:
  - `Scripts/install.sh --check-signing`: expect exit 77 before any build output, and a
    report naming the locked keychain and `Background`. In the `adhoc` branch, expect
    the warning and `-` with exit 0.
  - `Scripts/install.sh --check-signing --identity -`: prints `-`, exits 0.
  - `Scripts/install.sh --print-resolved-identity`: prints `Bob Mac Capture Local`,
    unchanged.

  Delete the copy afterwards. Never run install mode on the Mac, never run
  `security unlock-keychain` there, and never touch Bryan's checkout. If the Mac is
  offline, report the macOS checks as unverified.

- **bob-cli:**
  - `just check-scripts`;
  - `shellcheck scripts/install_all`, on athena if needed;
  - `git diff --check`.

  The Mac step of `install_all` runs only on Darwin, so do not run it.

- Commit both repos via `/sase_git_commit`.

## Bryan's steps

1. **Unblock now, before the fix lands.** Run `launchctl managername` in the shell where
   `just install-all` failed.
   - **Not `Aqua`** (SSH, or a tmux server started over SSH): run
     `security unlock-keychain ~/Library/Keychains/login.keychain-db` and rerun
     `just install-all` in that same shell. Or rerun it from a fresh GUI Terminal window
     outside that tmux server.
   - **`Aqua`**: run
     `cp /usr/bin/true /tmp/probe && codesign -f -s "Bob Mac Capture Local" /tmp/probe`
     and answer any keychain prompt with **Always Allow**. If it fails without a prompt,
     check `security find-identity -p codesigning` for duplicate "Bob Mac Capture Local"
     entries.
2. **After the fix.** An install from SSH or that tmux server asks to unlock the login
   keychain (in the `unlock` branch) or stops before building with the fixes.
   `just signing-check` in bob-mac-capture checks a session without building.
