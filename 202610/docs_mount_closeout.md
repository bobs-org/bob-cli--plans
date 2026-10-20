---
tier: tale
title: Finish mount validation and live mirroring, then land the docs toolchain
goal: Fix the four reproduced mount and serve defects and complete the verified nested
  toolchain landing.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-a0.2.1.7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-a0.2.1.7
status: done
---

- **PARENT:**
  [202610/docs_toolchain_landing_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/docs_toolchain_landing_fixes.md)
- **BEAD:**
  [bob-cli-a0.2.1.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.2.1.7.md)

# Finish mount validation and live mirroring, then land the docs toolchain

Finish the four reproduced defects below and complete the interrupted landing of
`bob-cli-a0.2.1.7`. This is one bounded coding assignment in the existing Python
package, not another phased epic. **This tale has no land agent: its coder must do the
closeout in the same turn as the final code.** Do not wait for this work's own commit
SHA, push, or CI result; the host commits after the turn ends.

## Governing scope and completed audit

Read these audited records before coding:

```bash
sase bead read bob-cli-a0.2.1.7 -r "Need landing findings and follow-up triage"
sase artifact read plan:202610/docs_toolchain_landing_fixes.md "Need the mount and serve contracts being completed"
sase bead read bob-cli-a0.2.1 -r "Need previous parent landing audit and readiness requirements"
sase artifact read plan:202610/docs_toolchain.md "Need governing toolchain contracts and parent closeout"
```

The land audit on `d5c98a2a` reviewed all three child phases and their notes, commits
`2c50df55`, `5cbfea1f`, and `d5c98a2a`, plus all six parent phases and notes. The
frontmatter, destination symlink/stale deletion, production, assembled lint, privacy,
nested fences, ratchets, URL/fragment/mention, and authoring changes are present. The
four defects below are the remaining reproduced work.

`just docs-check` passed all implemented gates and **168 tests**: 27 contracts, 32
pages, 365 mentions, one curated URL, two search cases, and offline lychee 3595 links
with zero errors. The live manifest returned the expected first-publish 404 warning.
`just docs-build-material` passed. The tracked tree stayed clean. `just check` remains
required after the fixes. Never run `just check-full`.

A fresh fetch showed `master = origin/master = d5c98a2a`; no non-epic commit landed
during this child epic. The only unrelated commit during the parent toolchain epic,
`24f43958`, updates `docs/task-move.md`, which remains registered and passes these
gates. The MIT license commit is compatible. Recheck drift before closing.

Inherited decisions remain MIT, search indexing enabled, public design decisions
enabled, and the existing `sase/memory/docs.md` addition only. **This tale changes no
SASE memory, Rust code, check recipe, public URL layout, or real mount table.**

## 1. Validate every discovered source repository

In `tools/docs/bob_docs/mounts.py`, `discover_repo` returns existing overrides,
siblings, and linked directories after only `is_dir()`. The promised invalid-source
tests were never added: the current precedence tests deliberately accept empty non-Git
directories.

Reproduced with disposable fixtures: an existing non-Git sibling, the same path as an
explicit override, and an initialized sibling whose origin is
`https://example.com/wrong.git` are all accepted against a different mount remote.

- Validate each selected local candidate before returning it. It must be a usable Git
  working tree rooted at that candidate, with a resolvable HEAD and an origin matching
  the mount remote. Merely running Git in a directory is insufficient: Git can discover
  an enclosing unrelated repository. Support worktree `.git` files, and compare resolved
  top-level paths.
- An existing invalid override, sibling, or linked candidate fails with its source,
  path, and reason; it must not fall through to a cache/network clone.
- Keep the established discovery precedence and one resolution per unique repo. Do not
  fetch or reset user-owned overrides/siblings/linked worktrees. Production continues to
  use its explicit pinned-source checks without discovery or fetching.
- Compare origins as repository identities so normal SSH/HTTPS spellings of the same
  GitHub repository work; normalize only known equivalent forms, never match by basename
  alone. Retain local fixture remotes (`file://` and local paths).
- Apply the working-tree identity check before mutating a cache clone too.
- Convert affected tests in `test_mounts.py` and assembled-mount lint fixtures to
  genuine temporary Git repositories with explicit matching origins. Add failures for
  existing non-Git candidates and wrong origins at every discovery tier, plus the
  nested-directory/enclosing-repo trap and equivalent-origin positives. Use local
  fixture remotes, never a real sibling or network source.

## 2. Validate each actual mount destination's ignored status

`validate_mount_targets` checks only a synthetic
`docs/<target>/.__bob_docs_mount_probe.md`. Reproduction: add
`!docs/reference/plugins/guide.md` after the normal target ignore, then assemble a
fixture containing `guide.md`. Assembly succeeds and writes the file even though
`git check-ignore -q --no-index docs/reference/plugins/guide.md` exits 1.

- Keep the directory/probe check, but also require every planned destination to be
  gitignored, using the shared helper and its actual repository-relative path. The
  committed `index.md` remains the landing-page exception; it is not copied.
- Check all outputs before any write or stale deletion, alongside the existing
  tracked-file and symlink checks. Emit the rejected concrete path.
- Add a fixture with a negated ignore for one page in a later mount, an earlier mount
  whose source changed, and a stale output. Rejected assembly must leave previous
  outputs byte-identical, retain the stale output, and write no new file. Cover an asset
  as well as Markdown, including nested paths. Preserve the current production
  cleanliness and normal rerun/stale-removal behavior.

## 3. Make cache fallback reflect genuine network unavailability

`_refresh_cached_clone` treats every nonzero fetch as offline. After cloning a local
fixture's `main`, asking for branch `missing` succeeds, keeps the old bytes, and prints
`offline ... fatal: couldn't find remote ref missing`. This masks a reachable remote's
configuration error, violating the accepted contract.

- Distinguish recognized connectivity failures from missing branches, authorization
  failures, missing repositories, and local Git/configuration errors. Only genuine
  connectivity failure may retain a previously validated cache with an explicit warning.
  Unknown failures fail closed with the fetch diagnostic. Use deterministic diagnostics
  (for example Git's C locale) if classification examines stderr.
- After successful shallow fetch, reset to the exact fetched requested branch head,
  including when the configured branch differs from the branch used for the original
  shallow clone. Do not rely on a stale or absent `origin/<branch>` ref; use an explicit
  fetch refspec or the just-fetched verified ref.
- Keep origin mismatch fatal before fetching. Preserve cache SHA provenance in
  `build.json`; production must never enter this path.
- Test an actual local-remote missing branch (must fail), a branch switch/update (bytes
  and SHA must match), and mocked genuine offline and authentication failures
  (respectively warn/retain and fail). The existing test deletes a local remote and
  calls that offline; replace that assumption with a real connectivity-failure fixture.
  No live network dependency in the tests.

## 4. Mirror file/directory replacements without getting stuck

`mirror_docs_once` copies before deleting stale paths. Replacing authored file `section`
with directory `section/page.md` raises `FileExistsError`; replacing a directory with a
file raises `IsADirectoryError`. The serve loop catches either and retries the same
failure forever, leaving stale served content.

- Validate the complete authored tree using the existing symlink rules before modifying
  staged content. Safely remove conflicting staged path types and obsolete descendants
  in an order that allows both file-to-directory and directory-to-file replacement, then
  copy the current files and prune empty obsolete directories.
- Preserve assembly-owned `build.json`, including its bytes, and retain current
  additions/edits/deletions and server-thread shutdown behavior. Do not introduce a
  second build or a real-server unit test.
- Add regression tests for both replacement directions, a second idempotent mirror pass,
  and rejecting an escaping authored symlink before any staged mutation. Verify all
  expected staged bytes, stale removals, and manifest preservation.

## 5. Verify the combined tree and integration

- Run focused regression tests while implementing, then `just docs-check` and
  `just docs-build-material` sequentially: they share `target/docs/`. Inspect both
  builder manifests and confirm the correct builder and source provenance.
- Run `just check` (the named `sase tool run check` wrapper is appropriate). Do not run
  `just check-full`. If the named check escalates, join its exact running ID via
  `/sase_monitor` and give the successor this plan's remaining closeout; do not rerun or
  abandon a live command. A long command needs `/sase_monitor`, not a normal final
  response promising to resume.
- Check `git diff --check`, generated drift, and source status. Only intentional
  source/test/documentation changes may remain. Update existing toolchain README wording
  only where needed to explain the corrected validation/cache behavior.
- Fetch/review commits that landed after `d5c98a2a` and integrate any conflicting
  consumers. The approved scope stays the four fixes, affected fixtures/docs, and
  closeout; newly found epic-caused blockers must be completed before closing.

## 6. Finish this landing and its eligible ancestors in this turn

Follow-up triage is complete and recorded on `bob-cli-a0.2.1.7`:

- Its three phases have no `PROPOSED FOLLOW-UP:` entries.
- Parent phase `bob-cli-a0.2.1.1` note #1, the PDFKit public citation, was rechecked
  with `/sase_new_task` (all-status searches, one-week sweep, active epics). It is
  already owned by `bob-cli-a0` as a `DISCOVERED ISSUE`, now corroborated. No task
  created; `bob-cli-8p` is a related memory decision, and `bob-cli-85` is an unrelated
  rendering defect. Leave the public citation to that outer epic.
- Parent phase `bob-cli-a0.2.1.4` note #1, HTTP-only fragment checking, is declined
  because its premise was false. Offline fragments already pass with
  `--include-fragments --index-files index.html`; no task is warranted.
- The four defects in this tale are epic work, not deferred follow-ups. Record any
  genuinely new unrelated discoveries through `/sase_new_task` before closeout.

After the final code passes verification, complete this sequence without waiting for
this tale's commit, push, or CI:

1. If SASE assigned this tale its own descendant bead, close that completed tale bead
   normally with verification evidence before closing its parent. Mark the tale's own
   archived plan done if required. Use the assigned ID supplied at launch; do not invent
   one. Re-read `bob-cli-a0.2.1.7` and every descendant to confirm all required children
   are complete. The existing direct phases are `bob-cli-a0.2.1.7.1`, `.2`, and `.3`;
   review any new descendants and notes too.
2. Run `sase bead epic-symbols bob-cli-a0.2.1.7`. Resolve every listed entry by wiring,
   privatizing, adding an allowed non-test pragma, or deleting the symbol; re-key only
   when a still-open later bead actually needs the exemption. Then:

   ```bash
   sase bead close bob-cli-a0.2.1.7 --note "Verified all phase notes and implementation, completed source identity validation, per-output ignore checks, genuine offline cache handling, and file/directory live mirroring; docs-check, Material, and just check passed; post-start drift integrated; follow-up triage retained on this bead; epic symbols resolved."
   ```

   Use the actual verification outcome, not an unsupported stock claim. Run
   `just symvision` if available; the audited tree has no such recipe, so otherwise
   record that fact and the empty epic-symbol result. Open the plans repository with
   `/sase_repo` (`sase repo open plans -r "Mark the verified landing plans done"`), read
   plan artifacts through `sase artifact read`, and set `status: done` in the
   frontmatter of **`plan:202610/docs_toolchain_landing_fixes.md`**, at the PLAN path
   from the bead read inside the returned repository. Do not hand-edit beads.

3. Re-read the closed epic with
   `sase bead read bob-cli-a0.2.1.7 -r "Need the parent link"`. Its parent is plan
   **`bob-cli-a0.2.1`**. Read that parent's two previous landing notes, all descendants
   and notes (including direct phases `.1` through `.6`), and
   `plan:202610/docs_toolchain.md`. Recheck descendant closure, every linked plan's
   readiness, and post-child source drift. Existing phase implementation is covered by
   the recorded audit plus this combined verification; inspect any changed code. If
   still complete, retire all entries from `sase bead epic-symbols bob-cli-a0.2.1`, then
   close normally:

   ```bash
   sase bead close bob-cli-a0.2.1 --note "Rechecked the previous landing audit, all descendant notes and linked plans, completed landing fixes, post-child drift, both strict builders, docs gates, and unchanged Rust check; all required descendants are complete and follow-up outcomes are recorded."
   ```

   Again use actual evidence. Confirm with `just symvision` if available and set
   `status: done` in **`plan:202610/docs_toolchain.md`**.

4. That plan's parent is phase **`bob-cli-a0.2`**. Read it and verify this completed
   child toolchain satisfies its scope (27 current contracts, despite the older
   description's 26). Resolve any entries from `sase bead epic-symbols bob-cli-a0.2`,
   then close only this phase normally:

   ```bash
   sase bead close bob-cli-a0.2 --note "Verified the completed documentation toolchain child plan, current 27 contracts in place, strict Zensical and Material builds, docs gates and negative fixtures, authorized docs memory, and unchanged Rust check recipe; both nested toolchain plans are complete."
   ```

   Leave **`bob-cli-a0`** open for its waiting land agent, and leave
   `plan:202610/bob_public_docs_site.md` in progress. Do not mark an outer epic plan
   done just because its toolchain phase is closed.

If symbol entries reject a close, clean them and retry. Never use `--force` merely to
make a close succeed or to advance this successful nested landing. If an ancestor is
incomplete or ambiguous, stop at it, append a precise blocker note, and report it; do
not sweep unfinished descendants closed. Before the normal final response, use
`/sase_final` for every changed repository, including the plans sidecar. The host's
commit happens afterward and must not block the closeout above.
