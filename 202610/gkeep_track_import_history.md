---
tier: tale
title: Track Google Keep import history through the vault Git allowlist
goal:
  Restore marker-free GKeep pulls on the Mac with synced receipts, actionable Git
  diagnostics, and commit-safe retries.
size: medium
proposed_by: bbugyi200.apollo.6a.f0.f1
create_time: 2026-10-10 10:34:05
status: wip
---

# Make marker-free Google Keep imports work with the vault's Git allowlist

## Outcome and scope

Restore ordinary `bob gkeep pull` on Bryan's Mac by allowing the new import receipts in
the vault's tracked data. Keep the marker-free task rendering, linked lightbulb, receipt
location and schema, and commit-before-archive contract. Make future ignore failures
explain the actual rule, and close the directly related retry path that can currently
skip that contract.

This is a medium tale for one implementation agent: a narrow configuration change in
`bobs-org/bob`, plus bounded GKeep Git checks, recovery routing, tests, and docs in
`bobs-org/bob-cli`. No new CLI flags, storage migration, global Git configuration
changes, memory changes, or plugin work are needed. Authoring this tale is large work
under the SASE size guidance; its implementation is medium.

## Diagnosis and evidence

The marker-free implementation at bob-cli commit `bc303b1` introduced versioned
`.bob/gkeep/imports/<transaction-id>.json` receipts. Its preflight correctly refuses to
proceed when Git cannot track that import evidence.

The vault repository at `68901affb695c99f50ff2b3328817a8e4a107c2d` uses an allowlist:
root `.gitignore` line 3 is `*`, line 10 is `!*/`, selected note and attachment
extensions are allowed, and JSON is allowed only under `.obsidian`. There is no
allowance for the new receipt directory. Reading this repository through
`sase repo open gh:bobs-org/bob` reproduced the reported path exactly:

```text
$ git check-ignore -v -- .bob/gkeep/imports/09b30c84e9c330f2.json
.gitignore:3:*    .bob/gkeep/imports/09b30c84e9c330f2.json
```

This is an integration omission between the new receipt format and the vault allowlist,
reproducible on Linux as well as consistent with the Mac report. It is not evidence of a
Google authentication or filesystem-permissions failure. The shared chezmoi global
ignore file has no rule explaining this path. The live Mac's vault copy was not opened
or changed during planning; confirm its matching rule when verifying rollout rather than
claiming a live-machine fix from a clone.

Relevant current behavior:

- `src/native/gkeep/imports.rs::preflight_trackable` uses quiet `git check-ignore`,
  hides the matching rule and stderr, and incorrectly treats every nonzero status as
  success. Git distinguishes status 1 (not ignored) from fatal errors.
- `pull.rs` already checks prospective target/receipt paths before persisting a new
  batch; preserve that protection and the commit-time recheck. The reported error alone
  does not prove any task or receipt was written.
- `migrate.rs` currently checks trackability only after writing receipts and removing
  markers. Move the check before those mutations.
- `pull.rs` returns directly to `finish_with_archive` when no fresh task is needed and
  `imports::unfinished` is empty. That helper counts only prepared entries.
  Verified-but-uncommitted receipts therefore bypass the later branch intended to finish
  their scoped commit. A failed commit, including an ignore change between the two
  checks, must not turn into archive-without-commit on retry.
- Current Git integration fixtures initialize an unrestricted repository; they do not
  reproduce the production allowlist. Existing recovery tests also lack a Git-backed
  failure-after-receipt retry.

Git's documented ignore precedence, directory traversal, verbose output, and exit
statuses inform the implementation: [gitignore](https://git-scm.com/docs/gitignore) and
[git-check-ignore](https://git-scm.com/docs/git-check-ignore). In particular, verbose
output can show an inclusion pattern beginning with `!`; output alone is not proof that
a path is ignored.

## Implementation

### 1. Add the missing vault allowance

Open the external repository with
`sase repo open gh:bobs-org/bob -r "Allow durable GKeep import history in vault Git sync"`.
Use only its returned checkout; read its `AGENTS.md` and inspect `git status` before
editing. Recheck the current ignore file in case the rule was fixed since planning. Edit
only its root `.gitignore`, adding this commented allowance after the existing
directory-traversal allowance:

```gitignore
# Durable Google Keep import history; sync with the imported tasks.
!/.bob/gkeep/imports/*.json
```

The existing `!*/` supplies traversal. Do not allow all JSON or all `.bob` data.
Ordinary `git add` must accept receipts; atomic temporary files such as
`.bob/gkeep/imports/.abc.json.tmp.123`, unrelated `.bob` files, and the existing
excluded plugin/PDF/cache trees must remain ignored. Preserve unrelated changes.

This reviewed repository edit is the actual configuration fix. The bob executable must
continue to respect ignore rules, without force-adding files or editing user ignore
files automatically. Keep the receipt path stable so existing prepared and verified
evidence remains usable. Include the external repository change in the SASE final
declaration as well as the bob-cli change; use the prescribed commit workflow, not raw
manual Git commits.

### 2. Make trackability checks accurate and actionable

Keep the shared GKeep check in `imports.rs`, using `ob::git_command` and its child
environment. Distinguish three outcomes from the quiet check: 0 means ignored, 1 means
trackable, and any other status, signal, or spawn failure is an error with the Git
diagnostic preserved. Do not silently downgrade failed repository detection to a non-Git
vault when a Git ancestor is present.

When ignored, obtain the winning source file, line, and pattern using verbose
`git check-ignore` output. Prefer its NUL-delimited format so spaces, tabs, and colons
in paths do not break parsing. Do not use `--no-index` for the operational check:
tracked paths remain trackable even when an ignore pattern would match an untracked
counterpart. Keep diagnostic parsing separate from deciding whether the path is ignored,
so a negated inclusion is not misclassified. If the diagnostic lookup fails or the rule
changes between queries, remain failed with the original path and a usable inspection
command; never convert it into a successful check.

Replace the suggestion to move the vault with a concise repair explanation. For the
reproduced case, convey this information through the existing error envelope:

```text
cannot track GKeep import history: .bob/gkeep/imports/<id>.json
ignored by .gitignore:3 (pattern: *)
Allow /.bob/gkeep/imports/*.json in the vault's .gitignore, then retry.
```

Explain in the hint/docs that the history must sync with its tasks. For other rules,
name the actual source and give a correctly quoted `git -C ... check-ignore -v -- ...`
diagnostic command. An ignored target note should be called a target note, not metadata,
and receive a target-specific repair. A parent exclusion such as `.bob/` needs ancestor
traversal exceptions; do not promise that a leaf exception alone fixes every rule. Keep
exit status 1 and the existing JSON error schema. Avoid exposing task bodies or
credential data in diagnostics.

### 3. Preserve the transaction boundary in pull, migration, and retry

Retain pull's pre-write check and its final pre-commit check. Integrate the improved
helper with migration before any receipt is persisted or corresponding marker is
removed. Plan the affected paths under the existing maintenance lock, allocate
prospective receipt names, and preflight the candidate notes and both new and reused
receipt paths before applying cleanup. On an ignore or Git-inspection failure, preserve
note bytes and existing evidence. Retain the commit-time recheck for rules that change
during execution. Dry runs remain free of writes.

For normal Git-backed pulls, ensure an archive-only action supported by a vault receipt
passes through locked receipt/commit reconciliation. The shortcut must not treat
`Verified` as proof of a Git commit. Select evidence by the planned Keep id/fingerprint
and receipt's recorded destination, rather than assuming that every receipt belongs to
the currently configured inbox; migration can create receipts for other notes. Re-read
evidence under the vault lock. Use the existing scoped commit path for outstanding
receipt batches and their relevant destinations before archival, and fail without
archiving when trackability or commit fails. Do not stage the whole vault or commit
unrelated receipts or user-staged files.

Already committed receipts continue to provide durable history after a task is edited,
completed, moved, or its original note disappears. Detect that a receipt is present
unchanged in HEAD before demanding an old destination for recovery; the index alone is
not proof of a commit. A successful repeat pull must not create an empty commit. For an
outstanding receipt whose needed destination is missing or cannot be safely established,
retain evidence and report the recovery problem; do not recreate notes or infer success
from a source URL. Preserve existing prepared-transaction recovery and
exact-block/multiplicity verification.

Keep pure legacy-marker and clipped-reference archive paths working without a spurious
metadata requirement. Preserve explicit `--no-commit` and non-Git semantics: they still
require durable receipts but do not require Git tracking. They are not the recommended
workaround for a Git-synced vault. This step is limited to receipt-backed commit/ignore
recovery, not a rewrite of capture, clipping, or the transaction format.

### 4. Document repair and rollout

Update `docs/gkeep.md` with the allowlist requirement, the exact allowance for Bryan's
existing `*` / `!*/` setup, ancestor-exclusion caveats, diagnostic command, and safe
retry instructions. Update the relevant section of `docs/vault-git-sync.md` to identify
receipts as tracked shared history, including on the Mac. State that moving/deleting
receipts or disabling commits is not needed.

The vault allowance must be published and reach the Mac through the documented vault Git
sync workflow; a bob-cli binary update alone cannot change vault rules. After the
approved vault change is available remotely, verify sync status and run the documented
`bob vault-sync` workflow on the affected host as appropriate. For any repository
access, follow `sase_repo` and use its returned path; do not bypass that rule by
SSH-reading another checkout. If live verification cannot be performed under those
access rules or the Mac is offline, report that precise rollout limit and give the short
verification sequence for the user rather than claiming success.

Verify on the affected host with quiet `git check-ignore` against a prospective receipt
path: exit 1 is the expected unignored result. `bob gkeep pull --dry-run` can confirm
preview behavior but does not prove the commit step. Prove that step with the isolated
integration tests below; leave any real Keep import/archive smoke operation explicit in
the completion report. Normal `bob gkeep pull` is the user's next action once the fix is
deployed. Preserve any receipts left by prior attempts so the improved retry can use
them.

## Regression tests and verification

Use the existing fake adapter and temporary Git vault harness. Keep tests offline and
isolate global/system Git settings so a developer's ignore rules cannot make them pass
or fail accidentally. Add only tests that exercise the observed rule or the affected
transaction boundaries:

1. Reproduce a production-style allowlist (`*`, `!.gitignore`, `!*/`, `!*.md`, and an
   Obsidian JSON allowance). A new pull fails naming `.gitignore`, the matching line and
   `*`; the target remains byte-identical, no receipt appears, and no archive call
   occurs. Include JSON error-envelope coverage.
2. Add only `!/.bob/gkeep/imports/*.json` to that fixture and retry. One task is
   written, its receipt and target land in the same scoped commit, then archive is
   called. A second pull duplicates nothing. Assert unrelated staged/dirty files remain
   outside that commit. Check receipt eligibility and continued exclusion of temporary
   files, other `.bob` JSON, and existing excluded vault directories.
3. Exercise a parent-directory exclusion, a custom excludes file, and an ignored target.
   Verify truthful diagnostics. Cover a tracked receipt despite an ignore pattern, an
   explicit negated allowance, and a Git fatal exit such as 128; fatal inspection errors
   must prevent writes/archives. Test unusual path characters if parsing verbose output
   rather than delegating quoting to Git.
4. Migration under the restrictive allowlist leaves markers and preexisting evidence
   untouched. With the allowance it removes markers and commits the notes with receipts.
   Include reuse of an existing uncommitted receipt so a retry cannot omit the metadata
   path just because it creates no new receipt.
5. In a Git vault, use the existing failure-after-receipt hook or a failing commit hook
   to leave verified but uncommitted evidence. Make its receipt ignored. Retry must fail
   with zero archive calls and no duplicate write. Restore trackability/commit
   availability and retry: commit the existing task/receipt before the fake archive
   call. Observe HEAD from the adapter when needed to establish ordering rather than
   merely checking eventual state.
6. Check a receipt with a recorded destination different from the current inbox, and an
   already committed receipt whose task was subsequently moved/completed or whose
   original note was removed. Recovery must respect recorded evidence without requiring
   nonexistent old files for committed history. Existing non-Git, `--no-commit`,
   dry-run, reference-only, and prepared-recovery tests remain green; add narrowly
   focused cases where their coverage is absent.

Run the affected suites first:

```sh
cargo test --lib native::gkeep
cargo test --test gkeep_pull --test gkeep_pull_recovery --test gkeep_migrate --test gkeep_list --test gkeep_cli
```

Then run the canonical `just check` through the repository's SASE tool/monitor workflow
as required. Prior continuation reported nine failures in
`native::highlights_ref::return_links`; this is historical evidence, not a waiver.
Report the actual current check result and substantiate any unrelated failure. Do not
modify that subsystem as part of this fix.

Inspect both repository diffs and statuses before finalization. For the vault change,
demonstrate an untracked prospective receipt is eligible without `-f` and the excluded
controls remain ignored. Report which changes are published, which are merely prepared
for host finalization, and whether the Mac's synced ignore rule was actually verified.
Completion requires both repository changes and the targeted checks; do not present
improved error wording alone as the fix.
