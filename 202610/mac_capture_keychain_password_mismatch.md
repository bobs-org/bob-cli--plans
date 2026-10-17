---
tier: tale
title: Diagnose a login keychain whose password no longer matches the login password
goal:
  Bob Mac Capture installs name an out-of-sync login keychain password and its one-time
  fix instead of blaming SSH or tmux.
size: medium
decisions:
  out_of_sync_prompt:
    ask: Should the install offer to sync a mismatched login keychain password in place?
    choices:
      unlock:
        Explain the mismatch, offer a one-run unlock, and print the one-time sync
        command
      sync:
        Ask [y/N] to run security set-keychain-password, then fall back to the one-run
        unlock
    default: unlock
    why: Bryan syncs once himself; an installer should not change a keychain password
    answer: unlock
  log_confirmation:
    ask:
      Should the install read loginwindow's unified log to confirm the mismatch before
      naming it?
    choices:
      log:
        Confirm the mismatch from loginwindow's last keychain unlock; ~3 s, only after a
        failed probe
      skip: Never read the log; always word the mismatch as the likely cause
    default: log
    why:
      The last fix guessed wrong; the log line is definitive and costs nothing on
      success
    answer: log
proposed_by: bbugyi200.apollo.8r
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.8r](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.8r.md)
- **COMMITS:**
  - [4c91ca9](https://github.com/bobs-org/bob-mac-capture/commit/4c91ca9d928947d1fa167a22be25516daab056d6)
    — fix: diagnose login keychain password mismatch

# Plan: Diagnose a login keychain whose password no longer matches the Mac login password

## Diagnosis

Bryan sees `just install-all` stop at step 3/3 (Bob Mac Capture). It prints "The login
keychain is locked in this session (SSH, or a tmux server started over SSH) …" and asks
for the password to unlock `login.keychain-db`. He sees this on every run, from his
local tmux on the Mac. That prompt and its wording came from
`plan:202610/mac_capture_locked_keychain_signing.md` (bob-mac-capture `acb2563`, bob-cli
`b59af42`). That plan guessed the cause while the Mac was offline. Read-only checks over
`ssh mac` today show the guess was wrong.

### Evidence, Mac, 2026-10-17 (EDT)

1. **The tmux server is GUI-born.** The server (`tmux new-session -d -s sase`, started
   Oct 8 06:46) has `KITTY_PID`, `SECURITYSESSIONID=186b7`, and no `SSH_CONNECTION` in
   its global environment. It was started from a kitty tab, not over SSH.
2. **`launchctl managername` is a false signal.** It prints `Background` inside that
   GUI-born tmux too, because tmux on macOS moves its server to the per-user bootstrap
   namespace. securityd still serves the tmux server from the GUI security session.
   securityd logs show tmux clients using the same login-keychain record (`0x103184720`)
   as GUI apps. An SSH login gets a separate, locked record (`0x8d121a400`). So the
   `Background` vs `Aqua` test in the failure report and README › Signing says nothing
   about whether the keychain is usable.
3. **The keychain was not locked when the install failed.** At 16:22:43 securityd logged
   the login keychain as `is unlocked`. Yet the codesign probe failed with
   `CSSMERR_CSP_OPERATION_AUTH_DENIED`, caused by `CSSMERR_CSP_INVALID_DATA` ("error
   while checking integrity, denying access"), and surfaced as `errSecInternalComponent`
   (-2070). `security show-keychain-info` failed with
   `CSSMERR_DL_OPERATION_AUTH_DENIED`. It did not fail with "User interaction is not
   allowed" (exit 36), which is what an SSH session gets. `login_keychain_reachable`
   treats any failure as "locked", so the script printed the SSH/tmux story.
4. **The Mac login password changed today.**
   `dscl . -read /Users/bbugyi accountPolicyData` shows `passwordLastSetTime` =
   10:39:48. The unified log shows `sudo`, then `passwd` (pid 48691) changing the bbugyi
   record at 10:39:48. Bryan's zsh history has `sudo passwd bbugyi` at 10:39:28.
   `passwd`, like `dscl . -passwd` and `sysadminctl -resetPasswordFor`, changes only the
   account password. It does not change the login keychain password.
5. **Every screen unlock since then fails to unlock the login keychain.** loginwindow's
   `com.apple.securityd:dp_login` lines tell the story:
   - Up to 07:56 every unlock logs `_SecLookupIndirectUnlockKey returned 0` and then
     `unlocking with cached value succeeded`.
   - From 12:46 on, each unlock (12:46, 13:07, 13:52, 13:58, 15:00, 16:08) logs
     `_SecLookupIndirectUnlockKey returned -25308`, then
     `unlocking with password failed`, then `failed to unlock with handle`.
6. **Keychain access breaks for everything after each failed unlock.** Codesign fails,
   and so do the tmux status bar's `security` lookups, Highlights, and iCloud agents.
   Access works again only after a manual `security unlock-keychain`. Those manual
   unlocks fail on the first try and then succeed on a later try (13:21:21 ✗ → 13:21:35
   ✓; 15:03:34 ✗, 15:03:58 ✗ → 15:04:09 ✓; 16:24:01 ✓). So Bryan types the new login
   password first and the old one works. That is the password the keychain still uses.

**Root cause.** On Oct 17 at 10:39, `sudo passwd bbugyi` changed the Mac login password.
The login keychain stayed encrypted with the old password. After every screen lock,
loginwindow tries to unlock the login keychain with the new password and fails. The
keychain is then left reporting "unlocked" while denying every item access. The next
`just install-all` fails its codesign probe. `install.sh` mislabels this as an
SSH/tmux-locked keychain and asks for "the login password", which only works when Bryan
types the old one. That keeps the system working until the next screen lock, so the
prompt comes back on every run.

**The real fix** is Bryan's one-time keychain-password sync (see Bryan's steps). The
code change below makes the install name this cause and its fix. It also stops pointing
at SSH, tmux, and `launchctl managername`.

Rejected alternatives:

- **Auto-sync or store the password.** The script never sees a password. Apple's
  `security` tools read it from the terminal. The `sync` decision only lets Apple's tool
  prompt.
- **Keep the managername check.** Evidence 2 shows a GUI-born tmux reports `Background`
  while it has full keychain access.
- **Move the identity to its own keychain.** This was already rejected in the prior plan
  (it changes the identity, which resets the Accessibility grant). It would also hide
  this bug instead of fixing it.

## Work

Two repos change, both shell and Markdown only, with no Swift. One is the linked
`bob-mac-capture` repo: open it with `sase repo open bob-mac-capture -r "<why>"` and
read its `AGENTS.md`. The other is bob-cli, this repo. Commit both.

### 1. bob-mac-capture `Scripts/install.sh`

- **Classify the login keychain.** Replace `login_keychain_reachable` with
  `classify_login_keychain`.
  - Run `/usr/bin/security show-keychain-info "${keychain}"` inside an `if` so `set -e`
    cannot fire. Capture its combined output in a global `keychain_info_output`,
    initialized to `""` at top level for `set -u`.
  - Set a global `keychain_state`:
    - `reachable` on exit 0.
    - `locked` when the exit status is 36 or the output contains
      `User interaction is not allowed` (errSecInteractionNotAllowed). This session
      cannot show the unlock prompt: SSH, or a tmux server first started over SSH.
    - `refused` for any other failure. The login keychain reports unlocked but denies
      access. Today that showed as `CSSMERR_DL_OPERATION_AUTH_DENIED`.
  - Comment why the exit status, not "any failure", decides.
- **Gather mismatch evidence, `refused` only.** This is best-effort and must never fail
  the script or prompt.
  - **Password age.** Read `passwordLastSetTime` from
    `/usr/bin/dscl . -read "/Users/$(/usr/bin/id -un)" accountPolicyData`: skip the
    first line, pipe through `/usr/bin/plutil -extract passwordLastSetTime raw -`, and
    format with `/bin/date -r "${epoch%.*}" '+%Y-%m-%d %H:%M'`. If any step fails, leave
    it empty.

  > [!decision] log_confirmation = log
  >
  > Run
  > `/usr/bin/log show --last 1d --style compact --predicate 'process == "loginwindow" AND category == "dp_login" AND (eventMessage CONTAINS "unlocking with" OR eventMessage CONTAINS "failed to unlock")'`.
  > Use the absolute path, because zsh has a `log` builtin. Take the last line. If it
  > contains `failed`, record its first 19 characters (`YYYY-MM-DD HH:MM:SS`) as the
  > confirmed-mismatch time. If it contains `succeeded`, or `log` fails or finds nothing
  > (for example a non-admin user), record nothing. Comment that these are Apple's
  > private log strings, so a miss only drops the confirmation line.

  > [!decision] log_confirmation = skip
  >
  > Do not read the unified log. The report always words the mismatch as the likely
  > cause.

- **`ensure_signing_identity_usable`.** First run the probe. If it signs, return 0.
  Otherwise call `classify_login_keychain` and branch:
  - **`locked`, `[[ -t 0 ]]`:** Unchanged. Print the existing one-line notice, run
    `/usr/bin/security unlock-keychain "${keychain}" || true`, and re-probe.
  - **`refused`, `[[ -t 0 ]]`:**
    - Print a short notice to stderr:
      - The login keychain refused access although it is unlocked. Its password likely
        (or, when the log confirmed it, does) no longer match the Mac login password,
        which last changed at `<time>` when known.
      - macOS breaks access again after every screen lock until they match.
      - Unlock it for this run with the previous login password.
    - Run the same `security unlock-keychain … || true` and re-probe.
    - On success, print one reminder line with the one-time fix,
      `security set-keychain-password ~/Library/Keychains/login.keychain-db`, then
      return 0.

    > [!decision] out_of_sync_prompt = sync
    >
    > Before the unlock, ask
    > `Sync the login keychain password to your current login password now? [y/N]`,
    > reading from the terminal. On `y`, run
    > `/usr/bin/security set-keychain-password "${keychain}"`. Apple's tool prompts for
    > the old and new passwords. Then re-probe, and return 0 on success. When declined
    > or failed, fall through to the one-run unlock above.

  - **Otherwise** (no terminal, a failed re-probe, or `reachable`): print the failure
    report. Exit 78 for `refused`, and 77 for `locked` and `reachable`, as today.

- **Failure report.** Keep the codesign diagnostic block. Then add one block per state:
  - **`locked`:** Remove the `launchctl managername` line, here and wherever else it is
    used. Say the login keychain is locked in this session, which cannot show the macOS
    unlock prompt (SSH, or a tmux server first started over SSH). Keep "Nothing was
    built or installed." and the three existing fixes.
  - **`refused`:**
    - The `security show-keychain-info` message, indented four spaces.
    - "The login keychain is unlocked but refused access."
    - The confirmed line when present: "At the last screen unlock (`<time>`), macOS
      could not unlock it with your login password." Otherwise: "This usually means its
      password no longer matches your Mac login password."
    - The password-age line when known.
    - One line explaining that `passwd`, `dscl . -passwd`, and
      `sysadminctl -resetPasswordFor` change only the account password, and that macOS
      breaks keychain access again after every screen lock until the two match.
    - "Nothing was built or installed."
    - Fixes:
      1. Sync once:
         `security set-keychain-password ~/Library/Keychains/login.keychain-db` (old =
         the previous login password, new = the current one).
      2. For this run only:
         `security unlock-keychain ~/Library/Keychains/login.keychain-db` with the
         previous login password, then rerun.
      3. Install ad-hoc with `just install ~/Applications -`, which resets the
         Accessibility grant.
  - **`reachable`:** Unchanged (Always Allow, matching `find-identity` lines, duplicate
    hint).
- **`usage`.** Document that a refused login keychain exits 78. Keep the 77 wording for
  the rest.
- **Unchanged.** `--print-resolved-identity` never probes. Install-mode stdout stays the
  single install-path line. CI's `--identity "-"` installs never probe.

### 2. bob-mac-capture `Scripts/create-signing-identity.sh`

Keep the `show-keychain-info` gate and exit 75. Reword its message: the login keychain
is locked or refusing access. Run the script from a GUI Terminal on the Mac. If that
also fails, the login keychain password may differ from the login password (README ›
Signing).

### 3. bob-mac-capture `justfile`

Extend the `install` and `signing-check` comments by one clause each. They also explain
a login keychain whose password no longer matches the login password. In the `sync`
branch, say they offer to sync it.

### 4. bob-mac-capture README › Signing

Replace the paragraph that begins "Prefer a GUI Terminal on the Mac." with an accurate
version:

- Prefer a GUI Terminal or a tmux server started from one.
- Over SSH, or in a tmux server first started over SSH, the login keychain is locked.
- While the identity is missing, setup cannot run and the install falls back to ad-hoc.
- Once the identity exists, the install checks before building that codesign can use it:
  - **Locked session:** with a terminal, it offers `security unlock-keychain`. Otherwise
    it stops with exit 77.
  - **Password mismatch:** after the Mac login password changes through `passwd`,
    `dscl . -passwd`, or `sysadminctl`, the login keychain keeps the old password.
    loginwindow then fails to re-unlock it after every screen lock, and every app's
    keychain access fails. The install says so and offers the one-run unlock (or, for
    `sync`, the sync). Otherwise it stops with exit 78. The lasting fix is
    `security set-keychain-password ~/Library/Keychains/login.keychain-db`. Changing the
    password in System Settings › Users & Groups keeps the two in sync.
- `errSecInternalComponent` from codesign means it could not use the key, in either
  case.
- `just signing-check` runs the same check without building.
- Drop the `launchctl managername` sentence.
- Keep the Always Allow note and the `codesign -d -r-` verification.

### 5. bob-cli `scripts/install_all` and README

- **`mac_capture_step`.** Add an `elif (( install_status == 78 ))` failure branch with
  the detail
  `"$pull_detail · login keychain password differs from the Mac login password · run security set-keychain-password ~/Library/Keychains/login.keychain-db"`.
  Leave the 77 and 72 branches as they are.
- **README "Install everything".** Replace the sentence that begins "From a session
  whose login keychain is locked" with two short ones:
  - From a session whose login keychain is locked (SSH, or a tmux server started over
    SSH), the step asks to unlock it before building.
  - When the login keychain password no longer matches the Mac login password, it names
    that cause and offers a one-run unlock with the previous password. Name the sync
    instead in the `sync` branch. The lasting fix is `security set-keychain-password`.
    Without a terminal, either case fails before building with its fix.

## Verification (implementer)

- **bob-mac-capture:**
  - `bash -n Scripts/install.sh Scripts/create-signing-identity.sh`;
  - `shellcheck` on both scripts;
  - `git diff --check`.
- **bob-cli:**
  - `just check-scripts`;
  - `shellcheck scripts/install_all`;
  - `git diff --check`.
  - Do not run the Darwin-only Mac step here.
- **Mac, best effort.** Use `ssh -o BatchMode=yes -o ConnectTimeout=10 mac` with no `-t`
  and stdin from `/dev/null`. rsync a scratch copy of bob-mac-capture into a `mktemp -d`
  directory, leaving out `.build` and `.git`, and delete it afterwards.
  - Over SSH:
    - `Scripts/install.sh --check-signing` should exit 77 before any build output. The
      report should be the `locked` block, with no `managername` line.
    - `--check-signing --identity -` should print `-` and exit 0.
    - `--print-resolved-identity` should print `Bob Mac Capture Local`.
  - Inside Bryan's GUI-born tmux server, which uses the GUI security session: run
    `tmux -S "$HOME/tmp/tmux/tmux-$(id -u)/default" run-shell '<scratch>/Scripts/install.sh --check-signing > <tmpfile> 2>&1; echo "exit=$?" >> <tmpfile>'`,
    then read `<tmpfile>`.
    - This has no terminal, so it can never prompt.
    - Expect `Bob Mac Capture Local` and exit 0 while his keychain works.
    - Expect exit 78 with the `refused` block when he has locked the screen since his
      last manual unlock and has not yet synced. In the `log` branch it also shows the
      confirmed line.
    - Report which one you saw, plus the last loginwindow `dp_login` line from
      `/usr/bin/log show --last 1d …`.
  - Never run any of these on the Mac: `security unlock-keychain`,
    `security set-keychain-password`, `just install`, install mode, or anything in
    Bryan's checkout.
  - If the Mac is offline, report the macOS checks as unverified.
- Commit both repos.

## Bryan's steps

1. **Fix it now (one time).** In any terminal on the Mac, run
   `security set-keychain-password ~/Library/Keychains/login.keychain-db`.
   - Old Password: your login password from before 10:39 on Oct 17. It is the one that
     works at the unlock prompt.
   - New Password, twice: your current Mac login password.
2. **Confirm.** Lock the screen (⌃⌘Q) and unlock it. Then, in your tmux:
   - `security show-keychain-info ~/Library/Keychains/login.keychain-db` prints
     `no-timeout` instead of failing.
   - `just install-all` signs Bob Mac Capture without a password prompt.
   - Optionally,
     `/usr/bin/log show --last 10m --predicate 'process == "loginwindow" AND category == "dp_login"' | tail -3`
     shows `unlocking with cached value succeeded`.
3. **In future**, change the login password in System Settings › Users & Groups, not
   with `passwd`, so the login keychain follows it.
