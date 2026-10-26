---
tier: epic
title: Finish unannotated overwrite - contract fixes and transaction hardening
goal:
  Close the contract gaps and transaction-safety gaps the bob-cli-c9.4.1 land review
  found in `bob ref create -f overwrite`, with behavior tests for each, so the
  interrupted landing of bob-cli-c9.4.1 can resume and close.
parent_bead: bob-cli-c9.4.1
phases:
  - id: selection
    title: Fix overwrite selection, refusals, and refusal coverage
    size: medium
    description:
      "selection: honor -N on URL and collection overwrite, refuse disabled ingest,
      archived reopen, and unstampable statuses before render, match the annotated
      refusal text, add the reread overwrite hint, surface verify_failed, and cover
      every refusal code with CLI tests."
    depends_on: []
  - id: output
    title: Preserve task alias, user fields, and audio; align output and docs
    size: medium
    description:
      "output: rewrite the task alias only on a real title change, report the final task
      state, align success output with the contract, keep the Play link, narration
      diagnostics, and audio extension correct, apply keep_user_fields, and correct the
      create and clip docs."
    depends_on:
      - selection
  - id: safety
    title: Harden the overwrite transaction and prove it with failpoints
    size: medium
    description:
      "safety: recheck selection after the locked pre-sync, refuse ignored or untracked
      files inside touched directories, tighten intent path validation, stop other
      pre-syncs from committing a crashed overwrite, drop blanket lint allows, add
      executor failpoint tests, and run the full gates."
    depends_on:
      - output
decisions:
  keep_user_fields:
    ask:
      Should overwrite carry the old aliases, topics, and research into the replacement
      when the capture omits them?
    choices:
      keep:
        Old aliases, topics, and research survive overwrite; alias links keep resolving
      drop:
        Rebuilt from the new capture only, as shipped; old aliases and topics are lost
    default: keep
    why: Overwrite replaces material, not progress; dropping aliases dangles alias links
    answer: keep
proposed_by: bbugyi200.athena.bob-cli-c9.4.1.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-26 03:39:11
status: wip
---

- **PROMPT:**
  [prompts/202610/overwrite_landing_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/overwrite_landing_fixes.md)
- **PARENT:**
  [202610/unannotated_reference_overwrite.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/unannotated_reference_overwrite.md)

# Finish unannotated overwrite: contract fixes and transaction hardening

The bob-cli-c9.4.1 land agent reviewed the combined tree at `94abe3d9` against
`plan:202610/unannotated_reference_overwrite.md` and the parent contract
`plan:202610/ref_create_force_recapture.md` ("The refusal", "Overwrite contract", and
"Phase `overwrite`"). `just check` passes, and every route runs through one staged
executor. The review still found contract deviations and untested safety paths. This
child epic fixes them.

Both parent plans remain the binding design. Their decisions apply unchanged:
`legacy_force = no`, `keep_created = yes`, no decision-record edits, and no memory edits
in this epic. The outer `annotated` phase bob-cli-c9.5 still owns annotated overwrite;
keep the interim annotated refusal.

Line numbers below are approximate and refer to `94abe3d9`. Re-locate them before
editing. Every fix needs a behavior test that fails before the fix and passes after it.
Prefer extending `tests/cli/highlights/overwrite*.rs` and the existing unit-test modules
over creating new harnesses.

## Phase `selection`: selection, refusals, and refusal coverage

1. **`-N` is ignored on URL and collection routes.** In
   `src/native/highlights_ref/recapture.rs`, `finish_selection` (around line 1146)
   validates `-N` but never sets `family.selected`. The overwrite then replaces the
   newest member and takes its defaults from it. Example: with `hello` and `hello__v2`
   filed, `-f overwrite -N hello URL` replaces `hello__v2`.
   - Fix: after `validate_reread_name`, set `family.selected` to the named member. Check
     that the no-force refusal's `-f overwrite` remedy then names the selected member.
   - The production path never calls `overwrite::select_url_member` or
     `recapture::select_overwrite_for_url`. Wire them in or delete them, and move their
     unit tests onto the live path.
   - Tighten `web_overwrite_selects_older_member_with_name` (`overwrite_web.rs`). It
     must assert that `hello` was replaced and that `hello__v2` is byte-identical (note
     and PDF). The current `contains("overwrite: blogs/hello")` also matches
     `hello__v2`. Add the same assertion for an exact-set collection with `-N`.
2. **Disabled ingest must refuse.** `create.rs` (around line 245) documents that
   disabled ingest does not refuse overwrite. That contradicts both plans ("Intake that
   cannot run (ingest disabled, …) refuses. Overwrite never queues").
   - When `intake::ingest_on_create()` reports disabled (config or
     `BOB_HIGHLIGHTS_INGEST_ON_CREATE`), filed overwrite refuses with code
     `intake_unavailable` and exit 1 on every route. The message names the reason and
     the hint says how to enable ingest on this host.
   - The refusal happens before fetch, adapters, pandoc, or listen, including on `-d`.
   - Update the comment and the docs.
3. **Annotated refusal text.** The message must contain exactly
   `overwriting an annotated reference is not available yet; -f starts a new reading`
   (contract wording). Keep the `annotated_reference` code and the evidence list (it may
   follow on its own line). Add CLI tests for each evidence kind:
   - a sidecar highlight;
   - a PDF highlight annotation;
   - an inbound `#^h-` fragment link;
   - an inbound follow-up link;
   - plus one test showing that PDF `Link` (navigation and audio) annotations alone do
     not refuse.

   Today only `.assets/` is covered.

4. **Archived task with an explicit open `-s`.**
   - First reproduce with a test: a reference whose reading task is closed in `done/`,
     overwritten with `-s ready`. The reviewer traced a late `verify_failed` rollback:
     `open_ref_task_in_done` in `sync.rs` fires only after fetch, render, and listen.
   - Overwrite keeps the task address and must not change scan's archived-task policy.
     So an explicit open status on an archived task refuses in preflight, before any
     render, with exit 2 and a hint toward reopening the task where it lives.
   - The success hint for a kept archived task must not suggest a command that would
     then fail.
   - Confirm that `-s ready` on a closed task outside `done/` still reopens it.
5. **Unstampable statuses are refused late.** Clap accepts `-s legacy`, but
   `validate_staged_metadata` refuses it only after render and listen. Refuse any status
   the overwrite cannot stamp before fetch or render.
6. **The reread overwrite hint is missing.** The parent plan requires
   `hint: <stem> is still ready with 0 highlights; -f overwrite would replace it instead`
   on a bare reread (`-f`) whose newest reading is `ready` with 0 highlights. Print it
   on the real run and the dry run of every reread route, and test it.
7. **`verify_failed` is invisible to humans.** The executor attaches it only as a
   machine code. Make human output for verification failures carry the `verify_failed`
   code, the same way `OverwriteRefusal::command_error` prefixes its code.
8. **Refusal coverage.** Add CLI tests for every remaining refusal. Each must exit with
   the documented code and leave the vault (`git status`, HEAD, file bytes) and the
   host-local intent directory unchanged. Cover:
   - `-k`, a differing `-t`, and a differing `-P` (hint names `bob task move`);
   - a matching explicit `-P` and `-t` being accepted;
   - `legacy_reference`, `intake_reference`, `not_in_sync`, `dirty_paths`,
     `open_note_tasks`, and `intake_unavailable`;
   - `nothing to overwrite` for a fresh local file;
   - a free `-N` on a title route;
   - the `-f overwrite` remedy row in the no-force refusal (actual selected stem, and
     `ref_history/`).

## Phase `output`: task alias, user fields, audio, output, and docs

1. **The task alias changes with no title change.** In `overwrite/execute.rs`,
   `rewrite_task_subtree` always calls `rewrite_task_alias` with `member.reading`. So
   overwriting the root of a two-reading family with `-N` turns `Title` into
   `Title (v1)`, and any custom alias is lost.
   - Rewrite the alias only when the staged title differs from the planned title.
   - When rewriting, keep the old alias's ` (vN)` suffix if it had one; add none
     otherwise.
   - With an unchanged title and no `-s`, the task subtree must stay byte-identical.
     Test the root, `__v2`, and custom-alias cases.
2. **The `task:` line reports the old state.** `overwrite/local.rs::print_task_line`
   prints the pre-overwrite mark and status, even after `-s`. Print the final mark and
   status (`OverwriteResult::task_mark`, currently unused). Align the success lines with
   the contract and the collections-phase spec:
   - `overwrite: <type>/<stem> (<status> · <N> highlights)`;
   - `task: kept <path> ^<id> [<mark>] <status>`, with ` · pass -s ready to reopen`
     appended for a kept closed, non-archived task;
   - `audio: kept <path> (narrates the replaced capture; add -L to re-narrate)`;
   - the commit result in the same `sync: committed <sha7> · …` style as ordinary
     create. Keep the exact push-pending text
     `overwritten; push pending (run bob vault-sync)`.
   - The `title:` line may stay. Dry runs print the same plan and end with
     `writes: none`. Update every test that pins the old lines.
3. **Markdown and vault-note overwrites lose the PDF Play link.** The overwrite renders
   pass no listen URI (`create.rs` around lines 2095 and 2234). Ordinary create passes
   `play_uri`. When the overwrite keeps or replaces companion audio, render the same
   Play link that ordinary create would.
4. **A finished narration is dropped on later failure.** When listen produced audio and
   a later step (staging validation or the executor) fails, keep the scratch audio and
   print `kept: <path>` through `listen::post_listen_error`, as ordinary create does.
5. **`-a` with a different extension.** `overwrite_audio_destination` reuses the
   existing companion path, so `-a x.m4a` lands in `report.mp3`.
   - Publish the replacement at the library stem with its own extension.
   - Delete the old companion in the same transaction (in the manifest, preimage
     checked), so exactly one companion remains and the note embed points at it.
6. **`keep_user_fields`** (decision above).

   > [!decision] keep_user_fields = keep Carry `aliases`, `topics`, and `research` from
   > the selected reference's current projection into the replacement marker and live
   > note whenever the new capture does not supply them. An explicit capture-supplied
   > value wins. Do not carry pipeline or annotation metadata. Test with a reference
   > holding an alias and topics.

   > [!decision] keep_user_fields = drop Keep the shipped behavior. Document under
   > Recapture that overwrite rebuilds `aliases`, `topics`, and `research` from the new
   > capture.

7. **Docs.** Read `sase/memory/docs.md` (with `/sase_memory_read`) before editing docs.
   - `docs/highlights-clip.md` (around lines 61–64 and 166–169) still says a filed
     reference "refuses even with `--force`" and that library PDFs are "never
     overwritten … never remade". Correct both for `-f overwrite`.
   - Fix the same claim in `docs/highlights-create.md` (around line 606).
   - In `docs/highlights-create.md`, document the overwrite success and dry-run lines,
     the no-archive `earlier:` line, push-pending reporting, and the refusal codes:
     `legacy_reference`, `intake_reference`, `not_in_sync`, `dirty_paths`,
     `open_note_tasks`, `intake_unavailable`, `annotated_reference`, the incompatible
     flags, and `verify_failed`.
   - Update clap help only where wording is now false.
   - Run `just docs-gen` and `just docs-check`.

## Phase `safety`: transaction hardening and failpoint proof

1. **Selection is not rechecked under the locks.** `execute_with_sync` re-runs `prepare`
   for the originally chosen member after the pre-sync. It never re-runs selection. If
   the pre-sync pulls a newer reading, a default (newest) overwrite would replace what
   is now an older reading.
   - After the locked pre-sync, re-derive selection from the same inputs. A default
     selection must still be the newest live member (no `has_later`), and an explicit
     `-N` member must still exist in the same family.
   - Collections must still match the exact URL set.
   - Otherwise abort with `preimage_changed` and write nothing.
2. **Ignored files inside a sidecar directory.**
   - `collect_path_preimages` (in `overwrite.rs`) hashes every file under a
     `.textbundle`, including gitignored ones.
   - The dirty check uses `git status --untracked-files=all`, which never reports
     ignored files.
   - `remove_old_sidecar` then deletes them for good. A rollback's
     `git checkout <base> -- …` fails on any path absent from base, so recovery wedges
     with the intent kept.
   - Fix both sides:
     - Preflight (and the locked recheck) refuses `dirty_paths` and lists every ignored
       or untracked file under a touched directory.
     - Rollback and recovery restore only paths tracked in the base commit, remove only
       recorded creations, and never fail wholesale because one path is absent from
       base.
   - Test with a gitignored file inside a `.textbundle`.
3. **Intent path validation.** `recovery.rs::validated_paths` accepts `.` and leading
   `CurDir` components. An intent naming `.` would run `git checkout <base> -- .` across
   the whole vault. Reject empty paths, any `CurDir`, and anything that does not name a
   path below the vault root. Add unit tests.
4. **Other pre-syncs can commit a crashed overwrite.** This is the epic note
   `DISCOVERED ISSUE`.
   - These callers pre-sync through `vault_sync` without checking for a pending
     host-local overwrite intent:
     - `ref rename`
     - `task move`
     - `randomize`
     - `vault_rewrite`
     - `migrate_tasks`
     - `migrate_zorg`
     - the standalone `bob vault-sync` cycle
   - After a crash they can commit the half-applied files. Recovery then refuses with
     "vault moved", and every later create or scan fails until someone removes the
     intent by hand.
   - `ref rename` already holds `bob_sync.lock → scan.lock`: call the existing
     `recovery::recover_with_locks_held` before its pre-sync.
   - The standalone `bob vault-sync` command takes `bob_sync.lock` itself. When a
     pending intent exists for its vault, it also takes `scan.lock` (in that order,
     using the existing timeout helper) and runs the same recovery before staging. If
     the lock is busy or recovery fails, it refuses the cycle without staging.
   - Every other `run_cycle_*` entry refuses to stage or commit while an intent for that
     vault is pending. The error names the stem and says to run `bob ref scan` on this
     host.
   - Never take a lock the caller already holds. Executor, scan, create, and intake
     recover before their pre-sync, so their own syncs must still succeed.
   - Test: a planted intent with a half-applied tree, then `bob vault-sync` (recovers)
     and `bob task move` (refuses). Document the behavior in
     `docs/highlights-ref-sync.md` next to recovery.
5. **Failpoint proof.** The epic required executor failpoints. None exist, and the
   `vault_sync` closure seam in `execute_with_sync` has no test.
   - Add a test-only seam (`#[cfg(test)]`, never a CLI flag or env var read in release)
     at these points:
     - after staging validation;
     - after intent persistence;
     - after each apply step (frozen note, sidecar removal, audio, PDF, note, task edit,
       sync write);
     - a forced verification failure;
     - before commit;
     - a commit that succeeds but is reported as an error;
     - a post-sync failure.
   - Each test builds a Git fixture vault and asserts exactly one outcome: the intact
     old state (bytes, index, HEAD, intent cleared), or the verified committed new
     state.
     - The commit-reported-as-error case keeps the commit and never advises a rerun.
     - The post-sync case prints the push-pending text.
   - Simulate a crash (return without rollback) and assert that the next recovery
     restores the old state and prints
     `recovered: rolled back an interrupted overwrite of <stem>`.
   - Use the pre-sync closure to mutate the vault and prove `preimage_changed` and the
     item-1 selection recheck.
6. **Lint hygiene.**
   - Remove `#[allow(dead_code)]` from `mod overwrite` (`highlights_ref/mod.rs`) and
     from `overwrite/recovery.rs`, and delete or properly scope what becomes unused (for
     example `load_intent`).
   - Refresh stale comments that still describe the interim "filed overwrite
     unavailable" message or later phases.
   - Add no new blanket suppressions.
7. **Gates.** Run `just check`, `just docs-gen`, and `just docs-check` on the combined
   tree, using `/sase_monitor` for long runs.
   - Record results on this phase.
   - `src/native/capture_related/tests.rs` timing flakes are tracked by `bob-cli-ca`.
     Rerun them in isolation, and note them rather than treating them as phase work.
   - Run `sase bead epic-symbols <phase-id>` before closing; this child epic adds no
     `--epic-symbol` entries.

## Hygiene for every phase

- Workers close only their own phase.
- Discovered work becomes a `PROPOSED FOLLOW-UP:` note on the phase, never a new bead.
- No memory edits, linked-repo edits, live-vault operations, Cargo version bumps, or
  CHANGELOG edits.
- Keep every commit buildable and the shipped command truthful.
- Mark contract-visible CLI output changes in commit messages per the release-train
  rules.
- This child epic's land agent verifies the combined tree and closes this child epic.
  Its `parent_bead` link then resumes the interrupted landing of bob-cli-c9.4.1, whose
  land agent closes bob-cli-c9.4.1 and then the phase bob-cli-c9.4. No phase here closes
  either bead.
