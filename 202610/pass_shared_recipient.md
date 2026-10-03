---
tier: tale
title: Fix cross-machine password-store recipient selection
goal:
  Make ordinary pass inserts on the Mac and Apollo decryptable on both machines using
  their existing shared GPG key.
size: small
proposed_by: bbugyi200.apollo.4f
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4f](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4f.md)
- **COMMITS:**
  - [16a0c61](https://github.com/bbugyi200/password-store/commit/16a0c61849145c481633cfea6c23da528438c551)
    — fix: pin password-store recipient to shared primary fingerprint

# Make Mac and Apollo password-store inserts use the same GPG key

## Outcome and scope

Make passwords inserted with ordinary `pass insert` on the Mac readable on Apollo, and
preserve the reverse direction. One implementation agent can complete this focused
configuration and operational repair; no epic or application-code change is needed.
Change the password-store recipient policy, restore one missing local trust assignment
on Apollo, deploy the policy to both machines, and verify the actual pass workflow. Do
not implement anything before this plan is approved.

## Confirmed diagnosis

Read-only investigation on 2026-10-03 established:

- Both live stores use `git@github.com:bbugyi200/password-store.git`, branch `master`,
  and had clean worktrees at `eb56b845872c128b4d105935b14670e8c148e344`.
- The repository has one `.gpg-id`, at its root, containing only
  `bryanbugyi34@gmail.com`. No `PASSWORD_STORE_KEY`, `PASSWORD_STORE_DIR`,
  `PASSWORD_STORE_GPG_OPTS`, or `GNUPGHOME` override was reported by the inspected
  sessions, including the Mac's interactive login shell.
- That email matches multiple keys on the Mac. Encrypting harmless input to it selects
  encryption subkey `087B0C5FE6E15749` on the Mac and `6DE4381D2A123F70` on Apollo. Thus
  identical recipient text does not select the same cryptographic key on the two
  machines.
- The token added in commit `7e58a24` was encrypted to Mac-only subkey
  `087B0C5FE6E15749`. Decrypting those historical ciphertext bytes reproduces
  `public key decryption failed: No secret key` on Apollo and succeeds on the Mac.
  Password output was discarded.
- The token was removed and re-added in `eb56b84`, this time to shared subkey
  `6DE4381D2A123F70`. The current live `pass show sase_listen_feed_token` succeeds on
  both machines. This replacement repaired that entry but left future inserts exposed to
  the same ambiguous-recipient problem.

Use this existing shared identity:

| Purpose                                   | Fingerprint / ID                           |
| ----------------------------------------- | ------------------------------------------ |
| Primary fingerprint to store in `.gpg-id` | `AED14B3D56DEEEC2365194F3ADAF90AC1B9BCD7A` |
| Its encryption-subkey fingerprint         | `DF4ECEB30FF3A83CE22D04296DE4381D2A123F70` |
| Expected ciphertext recipient ID          | `6DE4381D2A123F70`                         |

Both machines already have usable private material for this encryption subkey. Apollo's
primary secret key is absent (`sec#`), but its encryption secret subkey is present and
demonstrably decrypts; the absent primary secret key is not the cause. There is no need
to export, copy, or generate private keys, or add Apollo's separate SASE identity to the
store's recipients.

A second, distinct issue matters for reverse-direction inserts: the shared key has
ultimate ownertrust on the Mac (`<primary fingerprint>:6:`), while Apollo has no
ownertrust assignment for it. Apollo's batch encryption with normal trust checking fails
with `There is no assurance this key belongs to the named user` / `Unusable public key`.
Harmless fingerprint-pinned round trips succeeded in both directions when the diagnostic
encryption used `--trust-model always`; final acceptance must work without that bypass.

The current repository contains 267 ciphertext files: 266 target the shared Athena
subkey, and the old `cas.rutgers.edu.gpg` targets `EFCDE1FA9439D210`. That legacy
exception dates to the initial imported history and is outside this repair. Preserve it
byte-for-byte. There are currently no Mac-only ciphertexts in the current tree. The
history also records a fingerprint-based `.gpg-id` in `32bfe95`, reverted to the email
in `c14fbb1`; do not infer the reason for that revert. Today's reproduction and
successful shared-key tests are the evidence for the proposed policy.

## Implementation

1. **Recheck state and prepare a narrow rollback.** Use `/sase_repo` to open
   `gh:bbugyi200/password-store` and use only its returned checkout for repository
   exploration and edits. Re-read any applicable instructions. Use the operational
   `pass git` interface to inspect and synchronize the live stores. Read the tailnet
   reference through `/sase_memory_read` before remote access. Connect to `mac` as
   configured, with bounded SSH timeouts and host-key verification; Homebrew's
   executables are under `/opt/homebrew/bin` and absent from its noninteractive default
   PATH. Verify both live stores and the working checkout still agree, check for
   concurrent/uncommitted changes, and recheck recipient overrides and the shared
   fingerprints. Record starting revisions, ciphertext hashes, and Apollo's current
   ownertrust metadata in a private operational backup. Do not reset, stash, or
   overwrite someone else's changes. If the Mac is unavailable, use the appropriate SASE
   handoff for a retry and do not claim the cross-machine repair is complete.

2. **Pin the recipient in the password-store repository.** Replace the email in root
   `.gpg-id` with exactly the full primary fingerprint above, followed by a newline. Do
   not append `!` to the primary fingerprint: the primary is a signing/certifying key,
   and GPG must select its encryption subkey. Keep this in the shared repository rather
   than a machine-specific environment override. Add a short `README.md` explaining the
   shared identity, why email matching failed, the expected encryption subkey, a safe
   metadata/decryption check, and the need to provision a matching private encryption
   subkey and appropriate local trust when adding a machine. Explain that Git
   synchronization carries ciphertext and recipient policy, not private keys or trust.
   No chezmoi, bob-cli, shell-wrapper, or memory changes are required.

3. **Correct trust for the verified owned key on Apollo.** Reconfirm that the full
   fingerprint matches the Mac's trusted personal key and that the corresponding private
   encryption subkey is usable. If Apollo still lacks this assignment, import only this
   line via `gpg --import-ownertrust`:

   ```text
   AED14B3D56DEEEC2365194F3ADAF90AC1B9BCD7A:6:
   ```

   Refresh/check the trust database and verify ordinary encryption now succeeds.
   Preserve all other trust assignments and the existing keys. Do not set global
   `trust-model always`, suppress trust checking in pass, change pinentry, or copy the
   Mac's entire trust database. If unexpected identity or trust state appears, resolve
   that discrepancy before altering it.

4. **Publish and activate the scoped configuration change.** Use the authorized SASE
   commit/publication workflow for `.gpg-id` and the short README, never raw
   `git commit` or a force push. Coordinate commit and activation ordering so the live
   verification below happens after publication; do not equate an edited workspace clone
   with a deployed fix. Update the Mac and Apollo through `pass git pull --ff-only` once
   clean-state/upstream checks pass. Verify both live stores actually have the pinned
   policy and the expected revision. Keep rollback scoped to this configuration change.
   Do not run broad `pass init` on the live repository: the installed implementation
   rewrites recipient policy, traverses all ciphertexts for reencryption, and creates
   commits, which is unnecessary here.

5. **Repair current affected ciphertexts only if the refreshed inventory requires it.**
   Expect no ciphertext changes: the reported token is already repaired. If new Mac-only
   entries appeared after planning, enumerate precisely those current files, confirm the
   Mac can decrypt each, and reencrypt only them to the shared fingerprint. Keep
   plaintext entirely in a pipe/process memory, preserve its exact bytes including
   multiline content, and check both decrypt and encrypt exit statuses before atomic
   replacement. Retain original ciphertext for rollback until verification succeeds. Use
   the SASE-opened checkout and normal publication/sync flow; do not edit historical
   commits or touch the preexisting legacy exception. Never print real passwords,
   plaintext hashes, private keys, or passphrases. Avoid Git textconv when inspecting
   encrypted diffs (`*.gpg diff=gpg` is configured).

## Verification and acceptance

- Confirm that both active stores use the pinned primary fingerprint and have no
  conflicting recipient environment override. Inspect ciphertext recipients with
  `gpg --batch --list-only --decrypt --status-fd=1`; this does not decrypt the payload.
- On each machine create a private temporary, non-Git password store using the deployed
  `.gpg-id` policy. Insert a known non-secret probe with actual
  `pass insert --multiline`, using normal trust checking and no `PASSWORD_STORE_KEY` or
  `--trust-model always` override. Check that the ciphertext recipient is
  `6DE4381D2A123F70`. Transfer only that probe ciphertext into the other machine's
  temporary test store and verify `pass show` returns exactly the original probe.
  Perform both Mac-to-Apollo and Apollo-to-Mac tests, check subprocess statuses, and
  remove only the temporary stores created by this run. Do not add probe entries to the
  real store or Git history.
- Verify live `pass show sase_listen_feed_token` exits zero on both machines with
  password output sent to `/dev/null`; use batch/noninteractive failure behavior rather
  than an unattended pinentry prompt. If an unlock is required, route that user
  interaction through SASE and never solicit the passphrase in chat.
- Compare before/after ciphertext hashes and inventories. In the expected case all 267
  files remain unchanged. If targeted recovery was needed, require plaintext equality
  checks in memory and successful destination decryption for precisely those files;
  unrelated files and the legacy exception must remain unchanged.
- Verify only the intended policy/documentation changes and explicitly needed recovery
  files were published, the live stores are synchronized and clean, and no global trust
  bypass was introduced. Report the original cause, the already-repaired token's status,
  the policy/trust fix, and both actual pass round-trip results. Do not claim that this
  repairs the legacy entry or all historical ciphertext versions.

## Rollback and limits

Save the original policy and trust metadata before implementation. If the new policy
cannot pass the probe tests, avoid publishing/activating it until the discrepancy is
resolved. If rollback is needed after deployment, publish a narrow policy revert and
sync it normally; restore only this key's prior trust assignment using GPG's supported
trust interface, preserving other assignments. Restore only ciphertexts this run changed
from their encrypted backups. Returning to the email policy reintroduces the known
ambiguity and must be described as rollback, not a successful repair.

## Supporting documentation

- [pass documentation](https://www.passwordstore.org/): stores ciphertext in Git and
  allows recipient selection through its initialization policy.
- [GnuPG key selectors](https://www.gnupg.org/documentation/manuals/gnupg/Specify-a-User-ID.html):
  recommends fingerprints for unambiguous automated key selection and documents the
  special meaning of `!`.
- [GnuPG list-only behavior](https://www.gnupg.org/documentation/manuals/gnupg/GPG-Esoteric-Options.html):
  permits recipient inspection while skipping actual decryption.
