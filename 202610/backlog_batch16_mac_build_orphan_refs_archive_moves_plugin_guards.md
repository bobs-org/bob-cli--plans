---
tier: epic
title: 'Backlog batch 16: the broken Mac build, orphaned reference sources, archive
  moves under open PDFs, unignored plugin folders, and a plugin startup smoke check'
goal: Close five existing tasks by repairing the bob-mac-capture master build, stopping
  bob ref create from leaving saved sources without a pointer, verifying and coordinating
  the annotated-overwrite archive move, warning when a deployed plugin folder is not
  git-ignored, and giving plugin rollouts a real Obsidian startup check.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6
phases:
- id: mac-build-repair
  title: Repair the bob-mac-capture master build broken by 906a7df
  size: small
  depends_on: []
  description: 'mac-build-repair: resolve bob-cli-cz by fixing the RefsCopyEntry scope
    and public-HUD compile errors, compiling before landing, and confirming master
    macOS CI also passes batch 15''s uncompiled 9db33a6.'
- id: ref-pointer-orphans
  title: Stop bob ref create from writing a saved source without its source_markdown
    pointer
  size: large
  depends_on: []
  description: 'ref-pointer-orphans: resolve bob-cli-cy by finding the create path
    that wrote the muse_commentary store but no pointer, making store and pointer
    land together, and routing the one live orphan''s repair through a gate.'
- id: archive-move-presenter
  title: Verify and coordinate the annotated-overwrite archive move under an open
    PDF
  size: large
  depends_on: []
  description: 'archive-move-presenter: resolve bob-cli-d0 by reproducing the archive
    move with the NSDocument presenter fixture on macOS CI, then sending the right
    move notifications or recording why merging-only access is correct.'
- id: plugin-ignore-guard
  title: Warn when a deployed custom plugin folder is not git-ignored or is tracked
  size: medium
  depends_on: []
  description: 'plugin-ignore-guard: resolve bob-cli-ch per the plugin_ignore decision,
    reusing batch 15''s deploy-stamp ignore checks to cover each plugin''s whole vault
    folder in sync, list, and JSON.'
- id: plugin-startup-smoke
  title: Package the Obsidian startup smoke check that plugin rollouts run
  size: large
  depends_on: []
  description: 'plugin-startup-smoke: resolve bob-cli-a6 per the startup_smoke decision
    by packaging the CDP Obsidian startup and editor-mount check as a reusable rollout
    script and wiring it into the deploy docs.'
decisions:
  plugin_ignore:
    ask: Should plugin sync and list check each deployed plugin folder against the
      vault's ignore rules?
    choices:
      warn: Detect and warn in sync, list, and JSON; never edit .gitignore or untrack
        files
      append: Also append a missing ignore line through vault-sync when no plugin
        file is tracked yet
      decline: Close bob-cli-ch as canceled; batch 15's stamp-ignore refusal is enough
    default: warn
    why: Warnings catch the next missed folder without bob writing the shared vault
      .gitignore on its own
    answer: warn
  startup_smoke:
    ask: Should plugin rollouts get a packaged real-Obsidian startup and editor-mount
      smoke check?
    choices:
      implement: Ship the script plus deploy-doc and lander wiring; it runs where
        Obsidian runs
      decline: Close bob-cli-a6 as canceled; rollouts keep relying on unit suites
    default: implement
    why: Two shipped regressions (the 1.60.0 OOM and 1.67.0 editor mount) passed every
      unit suite
    answer: implement
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-27 03:08:25
status: done
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md)
- **PARENT:** [202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.md)

# Backlog batch 16: the broken Mac build, orphaned reference sources, archive moves under open PDFs, unignored plugin folders, and a plugin startup smoke check

This epic continues the bounded backlog campaign. Batch 15 closed bob-cli-cj,
bob-cli-az, bob-cli-cu, bob-cli-cr, and bob-cli-cs. Its lander fixed two regressions the
epic had caused:

- bob-cli 0a5f0793: f00ae66e's macOS-only file-coordination tests never compiled, so
  bob-cli macOS CI failed its lint step on six consecutive master commits and the move
  tests never ran. They now run and pass natively.
- bob-cli 2f004d77: plugin deploy stamps labelled local time as UTC.

The lander also re-synced every plugin from bob-plugins master 0b02193 (8 synced, 0
drift, clean UTC stamps). It filed bob-cli-cz, bob-cli-d0, and bob-cli-d1.

A fresh no-limit task enumeration after batch 15 closed shows 122 unresolved tasks: 117
ready and five snoozed (6q, aa, ah, cp, cq). Assign these five existing task beads,
reused and never cloned:

- bob-cli-cz: bob-mac-capture master does not compile. 906a7df (agent 14k.w0, no bead)
  added Highlights code/link copy keys with three compile errors:
  - `RefsCopyEntry` is not in scope in `BobPanelCoordinator.swift` (no
    `import RefsCore`).
  - A `public` `RefsGlassToastController.show(... style: ...)` overload takes the
    internal `RefsGlassToastStyle`.
  - A knock-on `.card` inference error in `AppDelegate.swift`.

  Runs 40439969638 and 40457103786 fail at Build; 2764ec5's run 40407215560 was the last
  green one. The break also hides CI results for batch 15's uncompiled Mac commit
  9db33a6.

- bob-cli-cy: one `bob ref create` on apollo wrote a saved-source store
  (`.bob/ref-sources/chat/muse_commentary_missing_from_exec_stream--ed2f4a29e3e3`) but
  never the note's `source_markdown:` pointer. The reference loses code, link, and
  Markdown copy, and the same create path may orphan every future reference.
- bob-cli-d0: the annotated-overwrite archive step renames the open PDF with
  `rename_no_replace` inside merging-only access and sends no move notifications.
  `overwrite/execute.rs` and `overwrite/recovery.rs` `PlatformReverseCoordinator` do
  this. What Highlights' presenter then follows is unverified.
- bob-cli-ch: `bob plugins sync`/`list` never check that a custom plugin's vault folder
  is git-ignored or untracked. The bob-articles folder was tracked for about 10 days
  (bob-cli-cd).
- bob-cli-a6: no plugin rollout launches a real Obsidian before declaring success. The
  1.60.0 relation-index OOM and the 1.67.0 editor-mount crash both shipped past green
  unit suites.

## Ranking and ownership

The hostile-process alert bob-cli-8r and the vault-sync halt alert bob-cli-9x still
outrank ordinary backlog work. Each needs its own large planned launch and must not be
folded into a backlog batch as a phase.

- cz is a deterministic CI blocker on the Mac app's master and hides another epic's
  verification.
- cy is silent loss of copy features on references, possibly on every create from one
  path.
- d0 is possible loss or misplacement of unsaved Highlights annotations. It also extends
  batch 15's move coordination to the one remaining move.
- ch and a6 prevent the recurring plugin-deploy failures behind bob-cli-cd, the 1.60.0
  OOM, and the 1.67.0 editor crash.

No active plan owns any of these five. bob-cli-ct, which preceded 906a7df, and batch 15
are closed.

Deferred, with reasons:

- bob-cli-cv, bob-cli-5m, and bob-cli-5n: SASE runner and tooling defects whose fixes
  live in the sase repository.
- bob-cli-8q and bob-cli-b9: completion flakes kept with epic bob-cli-3j. 3j has had no
  open phases and no running land agent since 2026-10-02. Bryan should relaunch its
  lander (`sase bead work bob-cli-3j`). Re-check ownership again at the next landing.
- bob-cli-58: waits for epic bob-cli-87's land.
- bob-cli-b4 and bob-cli-ax: wait for Bryan's Multivisor and clipster decisions.
- bob-cli-cq: snoozed until a release tag contains 0f76d558. The latest tag, v0.3.1,
  does not.
- bob-cli-d1 and the other memory tasks: each needs Bryan's authorization through
  `/sase_memory_write`.
- Next candidates: bob-cli-by (reliable precommit macOS verification, the root cause of
  cz-type breaks), a resized bob-cli-v (macOS CI clippy lists 57 warnings in the `cli`
  test target alone), bob-cli-8k, bob-cli-cw, bob-cli-6n, bob-cli-co, and bob-cli-ay
  (after the Mac acceptance run).

## Phase details

### mac-build-repair

Read bob-mac-capture `AGENTS.md` first: never land Swift you have not compiled. The fix
should be two lines:

- add `import RefsCore` to `Sources/BobMacCapture/BobPanelCoordinator.swift`;
- make the new `show(title:value:detail:style:centeredOn:)` overload in
  `Sources/BobMacCapture/Refs/RefsCopyHUD.swift` non-public, or make
  `RefsGlassToastStyle` public if a public caller needs it.

Recheck the CI log for any further errors once these clear.

Compile before landing:

1. Run `just portable-check`.
2. Then compile on the Mac: a scratch rsync without `.build`/`.git`, then
   `Scripts/bundle.sh` and `just typecheck-tests`. Delete the copy afterwards, and never
   use Bryan's checkout or `just install`.
3. If `ssh mac` times out (it did all through batch 15), get the compile from CI before
   landing instead. The workflow runs on `pull_request`: push a scratch branch, open a
   draft PR, and wait for the macOS job. Land only when Build passes, then close the PR
   and delete the branch.

After landing, confirm master macOS CI on the landed SHA passes every step after Build:
Test, bundle, and install gates. The run must include
`RefsPanelModelTests.testRetryableCoordinationRenameErrorsRetainDraftAndDoNotAnnounceSuccess`
from batch 15's 9db33a6. If 9db33a6 itself fails there, that is now in scope here
because it was never compiled. Record a `PROPOSED FOLLOW-UP` only for failures from
other commits.

### ref-pointer-orphans

Reproduce from the vault history named on bob-cli-cy. Vault commit 6beb2dad7 auto-synced
the store from apollo with `renderer.bob = 0.3.1` (a dev build), and two seconds later
d679d5151 committed the note without a pointer. Determine which `bob ref create` path,
or which bob revision, writes the source store without `source_markdown`. Candidates:

- the SASE research-file hook or another route that bypasses the pointer writer;
- a dev build between the store-writer commit 009060f9 and its pointer wiring;
- an early return between the two writes in `src/native/highlights_ref/create.rs`.

Then make store and pointer land in one scoped transaction, or have create refuse or
self-repair when the pointer is missing. Add a regression test for the path you found.
If `bob ref doctor` only reports orphans, decide whether a doctor finding should name
the repair command. Do not change the write-once store or content-addressed layout
(`decisions:reference-sources-are-preserved-write-once`).

The single live orphan may be repaired only as a command of a Bryan-approved
`/sase_gate` turn, for example `bob ref attach-source` or a pointer repair. Otherwise
record it as a limitation for the lander. Check `bob-cli-co` (attach-source backfill
skips) for overlap and do not take over its scope.

### archive-move-presenter

Read `decisions:open-pdf-writes-are-coordinated` and batch 15's move primitive first:
`coordinate_moving_write` (ForMerging|ForMoving, `move_forward`/`move_back` with
`willMoveToURL`/`didMoveToURL`) in `src/native/file_coordination.rs` and
`file_coordination/macos.rs`. Its NSDocument presenter fixture
(`tests/fixtures/file_coordination/presenter.swift`, driven from
`file_coordination/macos_tests.rs`) now runs natively on bob-cli macOS CI.

1. Add a macOS test that reproduces the overwrite archive with the presenter holding the
   original path. It should:
   - make unsaved presenter edits;
   - archive-rename inside merging access exactly as `overwrite/execute.rs` does;
   - install the replacement at the original path;
   - observe which URL the presenter tracks and where a later save lands.
2. Do the same for the reverse path in `overwrite/recovery.rs`.
3. Decide from that evidence which destination the open document should follow for the
   annotated-overwrite design. Then either route the archive and reverse moves through
   the move scope with explicit notifications, or record in code and docs why
   merging-only access is already correct. Never fall back to an uncoordinated write.

macOS-only Rust cannot compile on athena by default. Type-check it before landing with
the `aarch64-apple-darwin` target, which is installed on athena, and a stub C compiler
for the bundled C dependencies:

1. Write a script that creates whatever path follows `-o` and exits 0. Use it as
   `CC_aarch64_apple_darwin`, and use a matching stub that writes `!<arch>` as
   `AR_aarch64_apple_darwin`.
2. Set `CRATE_CC_NO_DEFAULTS=1` and run
   `cargo clippy --target aarch64-apple-darwin --all-targets --all-features -- -D unused -A unused_extern_crates -A unused_macro_rules -D private_interfaces`.

Batch 15's lander used this gate to reproduce and fix CI's exact errors. The runtime
verdict comes from macOS CI on the landed SHA; the lander must check it. Real-Highlights
behavior needs Bryan's Mac session: extend
`scripts/dev/highlights_coordination_acceptance` (or its documented steps) to cover the
overwrite archive, and record the unrun acceptance as a limitation. Do not edit memory.
If the decision record's scope becomes wrong, add the evidence to bob-cli-d1 as a note
rather than filing a new memory task.

### plugin-ignore-guard

Read `sase/memory/cli_rules.md` and `docs.md` through `/sase_memory_read` first. Batch
15 (115a3610) already checks, with `git check-ignore`/`ls-files` in
`src/native/plugins/deploy.rs`, that each `.bob-deploy.json` stamp is ignored and
untracked. Reuse those helpers to check every managed file and the plugin folder itself.
Report the result in:

- sync output;
- `bob plugins list` and its drift view;
- both JSON shapes.

Never run `git rm --cached`: untracking on one host deletes the files on the others at
their next pull. Cover a non-Git vault, a tracked file, an unignored folder, and the
dry-run path. Update the plugins docs and the bob-plugins deployment page.

> [!decision] plugin_ignore = warn Sync and list warn and JSON carries the finding;
> nothing writes `.gitignore`. Under `append`, sync also adds a missing
> `.obsidian/plugins/<id>/` line through vault-sync when no plugin file is tracked yet,
> with a dry-run preview and a test that a tracked file blocks the append. Under
> `decline`, close bob-cli-ch as canceled with the decision recorded and change no code.

### plugin-startup-smoke

Read bob-cli-a6 and its +1 evidence first. Then:

1. Decide where the check lives (a bob-plugins script or a bob-cli dev script) and which
   host can launch Obsidian. Find out where the ad-hoc 1.60.0 check ran, and never use
   Bryan's live session.
2. Package the CDP check. Launch Obsidian with `--remote-debugging-port` on a disposable
   copy of a fixture or scratch vault, never the live vault, and assert:
   - layoutReady within 30 s;
   - the deployed plugin versions;
   - index ready, with zero non-Markdown index entries;
   - heap under about 1.5 GB;
   - no new renderer crash reports;
   - one canonical daily note and one ordinary note mount their editors without an
     exception in both Live Preview and Source mode.
3. Wire the script into the bob-plugins deployment docs and the lander re-sync step as a
   rollout gate.

If no reachable host can run Obsidian during the phase, ship the script with a self-test
of its assertion logic and record the unrun live check as a limitation.

> [!decision] startup_smoke = implement Ship the script, its docs, and the lander
> wiring. Under `decline`, close bob-cli-a6 as canceled with the decision recorded and
> change no code.

## Acceptance

- bob-cli-cz, bob-cli-cy, bob-cli-d0, bob-cli-ch, and bob-cli-a6 are closed normally
  with verification evidence. If Bryan declines one, close it as canceled with his
  decision.
- The batch-16 plan is validated and proposed through sase plan propose. Its reference
  and ownership are recorded on the batch-15 epic bead.
- Commit every changed repository, including the plans sidecar, through
  /sase_git_commit.
- At landing, the lander re-syncs bob-plugins from master with a stamp-aware bob and
  confirms that `bob plugins list` reports 0 drift.
- The lander checks bob-cli and bob-mac-capture macOS CI on every landed SHA, including
  steps after a red lint or build.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include:
   - reliable precommit macOS verification `bob-cli-by`;
   - a resized `bob-cli-v`, `bob-cli-8k`, and `bob-cli-cw`;
   - Athena `bob-cli-b4` and `bob-cli-ax`;
   - `bob-cli-58` once active epic `bob-cli-87` has landed;
   - flakes `bob-cli-b9` and `bob-cli-8q`;
   - `bob-cli-6n`, `bob-cli-co`, and `bob-cli-ay` once the Mac acceptance run passes.

   Security `bob-cli-8r` and content-loss alerting `bob-cli-9x` outrank these. Each is
   large and should get its own planned launch rather than a backlog-batch phase.
   `bob-cli-8q` belongs to active epic `bob-cli-3j` while that epic owns the PTY
   completion drivers. This is guidance, not a preassigned batch, and concurrent work
   may obsolete it.

4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, `bob-cli-ah`,
   `bob-cli-cp`, and `bob-cli-cq` are not obsolete simply because their
   date/evidence/publication/release gate has not arrived. `bob-cli-cq` is actionable
   once a release tag contains bob-cli 0f76d558 (`git tag --contains 0f76d558`). Verify
   real prerequisites, not just wake timestamps. Work other actionable tasks first. If
   only gated tasks remain, the next bounded epic must own the readiness check and
   supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.

The successor epic must also carry the working rules batch 9 used:

- Phase workers leave target tasks open for the lander and record baseline, fix, tests,
  and limitations on their phase.
- Use `/sase_repo` before touching another repository, and `sase artifact read` for
  artifacts.
- No live vault edits, live feed publishes, or credential handling. There are two
  exceptions:
  - bob-plugins phases deploy only through `bob plugins sync`.
  - A live-vault repair runs only as a command of a Bryan-approved `/sase_gate` turn.
- Build bob through `$CARGO_TARGET_DIR`.
- Follow the daily release train: no version or `CHANGELOG.md` edits.
- Run `just check`, never `just check-full`.
- The lander must commit every changed repository, including the plans sidecar status
  edit, with `/sase_git_commit` before proposing its own successor. This plan explicitly
  authorizes the skill for those obligations.

Earlier batches added these deploy and verification rules:

- Before any `bob plugins sync`, a bob-plugins phase fetches and rebases its checkout
  onto current origin/master and rebuilds.
- Run `bob plugins sync --dry-run` first and deploy only the plugins the phase changed
  (`-p <id>`). Since batch 15, sync refuses to overwrite another checkout's uncommitted
  deploy. Never pass `-F` to get past that refusal without first checking that master
  contains the earlier deploy.
- Phase workers and landers run the full `just check`, and the full `npm test` for
  bob-plugins changes, on the tree they commit, not only focused suites.

Batch 16 adds two:

- macOS-only Rust (`#[cfg(target_os = "macos")]`) must pass the `aarch64-apple-darwin`
  clippy gate described under archive-move-presenter before it lands.
- Linux `just check` cannot see macOS-only code. After every push, the lander checks the
  macOS CI job of both bob-cli and bob-mac-capture for the landed SHA, including the
  steps after a red lint or build. In batch 15, a red macOS lint went unnoticed across
  six master commits.
