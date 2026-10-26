---
tier: tale
title: Protect plugin deployments from a stale checkout
goal:
  Record plugin deployment provenance and refuse overwrites that would erase another
  checkout's uncommitted or unincorporated build, with explicit force and safe
  clean-landing recovery.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.3
create_time: 2026-10-26 16:07:28
status: wip
---

- **PARENT:**
  [202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.3.md)

# Protect plugin deployments from a stale checkout

Implement the `plugin-deploy-guard` phase assigned to
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.3` (called **the phase** below). The underlying
bug is `bob-cli-cu`; leave that task and every ancestor open for the batch lander. Close
only the phase when its implementation and verification are complete.

## Scope and binding decision

The parent design is
`plan:202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md`. Its final
decision is **deploy_guard = refuse**. Do not ask again or implement the warn-only
branch. Use the existing `-F/--force` to authorize a protected overwrite, while
preserving its existing permission to overwrite dirty vault files.

The phase has no earlier incomplete-work notes. This is one bounded change to the
existing Rust plugins subsystem, its tests, and its deployment documentation. No plugin
JavaScript, memory, release version, or changelog changes are required. Do not implement
the broader plugin-folder ignore enforcement in `bob-cli-ch` or the rollout smoke-check
feature in `bob-cli-a6`.

## Evidence and current behavior

At planning, bob-cli HEAD was `777df22f` and its tree was clean. Read the phase and
`bob-cli-cu` again before coding so concurrent completion is not duplicated.

The bug has recurred twice: one agent deploys an uncommitted fix, another deploys an
older build from an otherwise current checkout, and the fix disappears from the vault.
Rebasing cannot recover work that has not been committed yet.

- `src/native/plugins/sync.rs` discovers plugins and immediately copies each managed
  file in turn. It records no deployment provenance and takes no deploy lock.
- `git.rs::vault_file_is_dirty` protects only files reported by vault Git status.
  Ignored deployed bundles bypass that protection.
- `model.rs::MANAGED_FILES` is `manifest.json`, `main.js`, and optional `styles.css`.
  Runtime files, including `data.json`, are untouched. Keep those ownership rules.
- `cli.rs::run_sync` runs the foreign-checkout check before the optional pull.
  Dirty-vault skips currently exit 0; real failures produce exit 1 and JSON
  `{"ok":false,"error":"..."}`. The foreign-checkout refusal exits 2.
- `scan.rs` and `render.rs` expose byte-based sync state and enabled state. Keep those
  states and their counters intact when adding deployment information.
- Unit coverage is in `src/native/plugins/tests.rs`; command coverage is in
  `tests/cli/plugins.rs`. Existing fixtures often are not Git repositories.
- `sha2`, `fs2`, serde, and time support are already dependencies. The `fs2` nonblocking
  lock pattern in `src/native/gkeep/pull.rs` is reusable.
- `docs/develop/plugins/deployment.md` is a generated mount of bob-plugins
  `docs/develop/deployment.md`; edit the owning source and regenerate the mount.

## Implementation

### 1. Capture and publish deployment provenance

Add a focused `src/native/plugins/deploy.rs` module (split its tests into a child module
if helpful), keeping filesystem/Git probing separate from the guard policy. Wire it
through the existing facade, sync, scan, model, and render modules.

Use a versioned `.bob-deploy.json` beside each installed plugin. Record:

- canonical source checkout root and plugin ID;
- full source HEAD when available, and a tri-state dirty result rather than treating
  failed Git inspection as clean;
- host, deployment time in UTC, and an optional agent identifier only if already
  available in the environment without probing SASE internals;
- SHA-256 fingerprints of managed asset names, presence, and exact bytes, and a
  deterministic aggregate digest; exclude runtime files and the stamp itself;
- completion state sufficient to distinguish a finished deploy from interrupted or
  partial copies.

The dirty flag describes the source working tree, including staged, unstaged, and
untracked nonignored changes. Do not infer clean provenance just because generated
`main.js` matches HEAD while source fragments are dirty. Collect source metadata after
the existing pull, and use the captured asset bytes for both hashes and copies. If the
source changes during capture, refuse rather than attribute a mixed snapshot to one
clean revision. A non-Git source remains supported with explicit unknown provenance; do
not report it as clean or silently disable protection of an existing known deployment.
Do not add network fetches or reads of the stamped source path.

Before any actual copy, verify that the stamp can be kept out of vault Git. In a Git
vault, use Git's ignore evaluation for the exact stamp path and check that it is not
tracked. Refuse deployment if the stamp is tracked or unignored, naming the path and the
ignore rule the owner needs to add (for example `.obsidian/plugins/*/.bob-deploy.json`).
`--force` does not bypass this storage requirement. A non-Git vault needs no ignore
rule. Distinguish an actual non-Git vault from a Git inspection failure. Do not edit
`.gitignore`, the index, or live vault configuration automatically. This check is scoped
to the stamp, not all plugin files.

Serialize real syncs to the same canonical vault with a nonblocking advisory lock in
host-local state, keyed by canonical vault identity and independent of `--repo` and
`--backup-dir`. Hold it from the destination inspection through stamp publication; on
contention, fail promptly with a retry message. Do not create a lock inside the vault or
create lock/state files for list and dry-run. Resolve equivalent vault paths to the same
lock identity. This is a same-host guard, not distributed locking.

For each plugin, read its prior stamp, snapshot its managed files, and decide before
copying any file of that plugin. A blocked plugin must have no copied assets, stamp
changes, or backups. Unrelated plugins may retain the existing per-plugin progress
behavior, but the invocation must report failure if any deploy is refused.

For an allowed real copy, atomically publish a pending stamp before changing assets,
then publish the completed stamp only after all intended copies succeed and the
resulting managed bytes are verified. Use a sibling temporary file and atomic rename for
stamp writes. If copy/backup/final-stamp publication fails, leave detectable pending or
mismatched provenance and return an error; never claim a complete deploy. If the legacy
vault-file guard skips a file, preserve its warning/exit-0 behavior and do not label a
partially synced plugin as a completed deployment. Preserve backups before overwrites;
retain the prior stamp alongside the backed-up assets when a protected deployment is
explicitly replaced. Never overwrite the only backup from another sync in the same
second: make backup run identities collision-safe.

### 2. Apply the refusal policy and support clean landing

Evaluate the following rules against captured bytes, a validated stamp, and Git ancestry
in the incoming checkout. Never equate an unknown object, failing Git command, or
missing HEAD with a successful ancestry check.

| Situation                                                                                                   | Behavior without `--force`                                                                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| No prior stamp                                                                                              | Bootstrap using existing asset behavior and write provenance after successful sync; explicitly acknowledge that pre-stamp history cannot be protected retroactively.                                     |
| Complete stamp and incoming assets exactly equal the deployed bundle                                        | No content overwrite, so permit. Preserve provenance by default; a verifiably clean Git checkout with the exact full bundle may reconcile an old dirty/unknown stamp to its clean revision on real sync. |
| Dirty stamp from a different canonical checkout or host; incoming bytes differ                              | Refuse even if HEADs are equal or the stamped HEAD is an ancestor. HEAD ancestry cannot prove inclusion of uncommitted changes.                                                                          |
| Dirty stamp from the same canonical checkout and host                                                       | Permit iterative deployment only when the prior stamp still matches the vault and its known HEAD is an ancestor of the incoming HEAD. A reset/divergent/unknown history still requires force.            |
| Clean stamp; incoming bytes differ                                                                          | Permit when the stamped commit is a known ancestor of the incoming HEAD; otherwise refuse.                                                                                                               |
| Unknown provenance, malformed/unsupported/pending stamp, or stamp digest differs from actual deployed bytes | Refuse changing assets unless forced. Exact equality may reconcile only from a verifiably clean complete incoming bundle. Report the uncertainty explicitly.                                             |

For non-Git source directories, preserve unchanged no-op behavior, but differing assets
after their first stamped deployment require force because ancestry is unavailable. A
clean unpushed commit is protected when absent from incoming history; if incoming HEAD
includes it, those committed changes are already incorporated. Do not make protection
depend on a successful remote fetch or stale upstream status.

The equality exception must compare the complete managed bundle, including optional file
presence. Preserve the existing rule that absence of source `styles.css` does not delete
a destination file; that leftover cannot be ignored to falsely certify an exact clean
bundle. Byte-identical sync must not turn a clean stamp dirty merely because an
unrelated checkout is dirty. Once provenance is accurate, repeated unchanged real syncs
leave the stamp timestamp/content untouched.

The clean landing path is deliberate: after A's uncommitted build is committed and
incorporated by a clean checkout B, a real sync of identical bytes clears the old
uncommitted provenance without force. A later clean descendant can then update normally.
If B's build actually differs, retain the refusal and require review of the diff and an
explicit forced overwrite; never assume the fix landed from SHA alone.

Keep the original bare foreign-checkout guard before pull, including its refusal under
`--force`. Force overrides deployment-provenance and dirty-vault protection, not lock
contention, unsafe stamp storage, or I/O failures. A forced replacement records the new
provenance and names the deployment it replaces.

### 3. Report the provenance and document the contract

Keep existing list sync/vault values and counts. Add an optional structured deployment
object to list JSON and sync success records, with provenance and a clear status such as
clean, uncommitted, unknown, incomplete, or mismatched. Missing stamps are an ordinary
legacy state, not a reason the entire list fails. Provenance warnings should remain
visible for healthy rows even when one stamp is malformed.

In table output, print concise detail associated with each relevant plugin, including
the exact phrase `deployed from uncommitted build` for dirty stamps and the source
checkout, host, short SHA, and time identifying the deployment. Avoid widening the main
table until it becomes unreadable. A protection refusal must name the affected plugin
and earlier deployment, explain the cause, and give the preview/force recovery commands.
JSON remains one object without table or diff text on stdout; use the existing
`ok:false,error` envelope for refusals with an actionable error description.

Deploy-protection refusal exits 1 in normal and dry-run modes. A byte-identical or
otherwise safe dry-run still succeeds, previews stamp creation/reconciliation, and
writes no assets, stamps, temporary files, backups, or lock files. Keep existing default
pull semantics: `--no-pull` is necessary for a completely read-only source checkout
preview. `copied` continues to count managed assets, not metadata writes, so installer
restart detection does not change for stamp-only reconciliation.

Read `cli_rules.md` and `docs.md` through `/sase_memory_read` before help/doc changes.
Update `cli.rs` long help and the existing `--force` help; no new flag is necessary.
Update `docs/plugins.md`, relevant JSON reference text, and the deployment guide where
its existing instructions become inaccurate. Use `/sase_repo` to open bob-plugins, read
its `AGENTS.md`, and edit `docs/develop/deployment.md` there. Regenerate CLI goldens and
the mounted deployment page with `just docs-gen` using `BOB_DOCS_BOB_PLUGINS_DIR` set to
the path returned by `sase repo open`. Do not hand-edit mounted docs or accept unrelated
generated memory changes.

Explain the stamp, refusal/force behavior, source dirty versus vault dirty state,
non-Git limitations, bootstrap, ignore prerequisite, pending-deploy recovery, and the
commit/rebuild/exact-match landing workflow. Existing binaries that do not understand
stamps cannot enforce the new guard; the new behavior requires deploying the updated bob
binary to sync callers. Mark the CLI behavioral compatibility change with `!` in the
eventual change subject per the daily-release policy; do not change Cargo.toml's version
or CHANGELOG.md.

## Verification and delivery

Start with a failing regression using temporary local source checkouts A and B and an
isolated vault. Both start at H; A edits and deploys without committing; B at the same H
tries to overwrite it. No network, live vault, or sibling agent checkout is needed.
Assert nonzero refusal and byte-for-byte preservation of every asset, the stamp, and
backups. Repeat with B at a descendant of H that lacks A's dirty change.

Add meaningful unit/CLI coverage for:

- clean forward deployment, nonancestor/unknown-object refusal, same-checkout dirty
  iteration, and canonical path aliases;
- the clean exact-match landing reconciliation followed by a safe later update;
- forced replacement with preserved backups/provenance and the existing dirty-vault
  skip/force behavior (give those Git-vault fixtures a stamp ignore rule so the distinct
  guards are tested independently);
- stamp bootstrap, repeat no-op stability, malformed/unsupported stamp, digest mismatch,
  partial write/pending recovery, optional CSS presence, and preservation of
  `data.json`;
- Git ignored/unignored/tracked stamps, non-Git sources/vaults, and Git probe errors;
- real and dry-run refusal, dry-run making no files, `-p` isolation, list/table
  uncommitted detail, JSON shape, and unchanged `copied` metadata semantics;
- the lock rejecting a second writer and preventing stamp-check/copy races. Prefer
  deterministic synchronization or a held-lock test to timing-based sleeps.

Run focused plugin tests and help/reference coverage first, then the full `just check`
on the final implementation tree, never `just check-full`. Use the existing
`CARGO_TARGET_DIR` for builds. Run `just docs-check` with the opened bob-plugins path,
and full `npm test` plus `npm run docs:check` in bob-plugins as required by the epic.
Use `/sase_monitor` for long commands and wait for its handoff command itself to exit.

The linked repo change is documentation-only: no live plugin was changed by this phase.
The parent epic's specific docs-only deployment rule applies: fetch/rebase the linked
checkout onto current origin/master and rebuild before a sync preview, then run
`bob plugins sync --repo <opened-checkout> --no-pull --dry-run` and record its result
without deploying unrelated plugins. Test real sync/stamp behavior only against isolated
fixture vaults, explicitly setting `BOB_DIR`, `--bob-dir`, repo, and backup paths. Do
not repair or force-overwrite live vault content for a test.

Record baseline, changes, tests, and limitations on the phase. Any unrelated failure
that reproduces identically on the clean base gets a `PROPOSED FOLLOW-UP:` note on the
phase, citing an existing task if known; it does not keep this phase open. Do not create
task beads or expand into another backlog item. If rebasing changes the implementation,
rerun the relevant checks and required final gate before completion.

Before closure, run:

```bash
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.3
```

There were no entries during planning. Resolve any new ones or re-key remaining
temporary exceptions to a still-open owning bead before closing. Close **only** the
phase with `sase bead close ... --note "<verified behavior and checks>"`; leave
`bob-cli-cu`, the parent epic, and all ancestors to their land agents. Use the normal
`/sase_final` declaration for completed work in both repositories; do not manually
commit unless explicitly instructed to invoke `/sase_git_commit`.
