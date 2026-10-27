---
tier: epic
title: 'Backlog batch 17: a precommit macOS gate, the attach-source backfill skips,
  duplicate research references, scan-wide dirty refusals, and a Swift format config'
goal: Close five existing tasks by giving Linux-launched agents a bounded precommit
  macOS check, attaching saved sources to the skipped agent-report references, stopping
  the research hook from creating duplicate references, refusing dirty ref-scan targets
  per PDF, and pinning bob-mac-capture's Swift format.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6
phases:
- id: macos-precommit-gate
  title: Give Linux-launched agents a bounded precommit macOS CI check
  size: large
  depends_on: []
  description: 'macos-precommit-gate: resolve bob-cli-by per the macos_gate decision
    with a scratch-branch macOS CI check for bob-cli and bob-mac-capture that never
    moves the checkout and always deletes its branch.'
- id: attach-source-skips
  title: Attach saved sources to the ambiguous and title-mismatched backfill skips
  size: medium
  depends_on: []
  description: 'attach-source-skips: resolve bob-cli-co per the attach_disambiguation
    decision by verifying every candidate of an ambiguous stem and running the write
    pass only through a Bryan-approved gate.'
- id: research-hook-dedupe
  title: Stop the research-file hook from creating hash-suffixed duplicate references
  size: large
  depends_on: []
  description: 'research-hook-dedupe: resolve bob-cli-cm per the hook_dedupe decision
    by deriving stems without the artifact hash, choosing -f overwrite or -f reread,
    and proposing the fold of the five existing duplicates.'
- id: per-pdf-dirty-refusal
  title: Refuse dirty ref-scan targets per PDF instead of failing the whole scan
  size: large
  depends_on: []
  description: 'per-pdf-dirty-refusal: resolve bob-cli-b1 per the dirty_pdf decision
    so one autosaved Highlights PDF no longer refuses every other PDF in the scan.'
- id: swift-format-config
  title: Pin bob-mac-capture's Swift format and re-indent the minority files once
  size: small
  depends_on: []
  description: 'swift-format-config: resolve bob-cli-8k by adding a majority-style
    .swift-format, re-indenting the minority files in one whitespace-only commit,
    and compiling it on macOS CI before landing.'
decisions:
  macos_gate:
    ask: Should agents get a script that runs their change on GitHub macOS CI from
      a throwaway ci/ branch?
    choices:
      scratch_ci: Ship check scripts for both repos, add a ci/** trigger to bob-mac-capture,
        document it
      decline: Close bob-cli-by as canceled; macOS breaks keep surfacing only after
        they reach master
    default: scratch_ci
    why: Three batches in a row landed macOS code that failed only once it was on
      master
    answer: scratch_ci
  attach_disambiguation:
    ask: Should attach-source verify each candidate of an ambiguous stem and attach
      the one that passes?
    choices:
      verify: Attach only a single verified candidate, pair mismatches only by review,
        write via a gate
      decline: Close bob-cli-co as canceled; the 82 references stay without saved
        sources
    default: verify
    why: These references lack code, link, and Markdown copy, the loss bob-cli-cy
      fixed for new ones
    answer: verify
  hook_dedupe:
    ask: Should the research hook drop the artifact hash from stems and pick -f overwrite
      or -f reread?
    choices:
      implement: Fix the stem and recapture mode, and fold the existing duplicates
        only through a gate
      decline: Close bob-cli-cm as canceled; every republished report keeps adding
        a duplicate
    default: implement
    why: Five duplicate references already exist and each republish adds another
    answer: implement
  dirty_pdf:
    ask: Should ref scan refuse only the PDFs whose own write targets are dirty and
      keep scanning the rest?
    choices:
      hold: Report each dirty PDF as a hold that retries next scan; clean PDFs proceed
      fail: Report each dirty PDF as a per-PDF failure; clean PDFs proceed
      decline: Keep the scan-wide dirty_targets refusal and close bob-cli-b1 as canceled
    default: hold
    why: An autosaved PDF clears once vault-sync commits it, which is what a hold
      means
    answer: hold
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-27 06:23:57
status: wip
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md)
- **PARENT:** [202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.md)

# Backlog batch 17: a precommit macOS gate, the attach-source backfill skips, duplicate research references, scan-wide dirty refusals, and a Swift format config

This epic continues the bounded backlog campaign. Batch 16 closed bob-cli-cz,
bob-cli-cy, bob-cli-d0, bob-cli-ch, and bob-cli-a6:

- cz: bob-mac-capture 0e7eba4 (PR #8) restored the master Build. Master CI 40469875355
  passed every step, including batch 15's
  `RefsPanelModelTests.testRetryableCoordinationRenameErrorsRetainDraftAndDoNotAnnounceSuccess`.
- cy: bob-cli 017b6927 carries the planned `source_markdown:` pointer through both sync
  executors and makes create intake refuse a missing pointer. A Bryan-approved gate
  repaired the one live orphan.
- d0: bob-cli e557b021 runs the annotated-overwrite archive move and its recovery inside
  bounded ForMerging|ForMoving scopes with move notifications.
- ch: bob-cli 49f6131d warns, in sync, list, and JSON, when a plugin's vault folder is
  not ignored or holds tracked files.
- a6: bob-plugins 88f0cdd packages the isolated real-Obsidian startup smoke check.

The batch-16 lander fixed one regression the epic had caused. e557b021's native rollback
presenter test failed deterministically on macOS CI on two master SHAs. NSDocument
applies move notifications and re-reads the file about half a second after the request
ends, and the test edited before that re-read. bob-cli c2b69089 makes the test wait for
the re-read, and macOS CI passed it. Scratch-branch diagnostics showed the coordinated
rollback itself was correct.

The lander re-synced bob-plugins master 88f0cdd: 8 synced, 0 drift, no vault-ignore
findings. The packaged smoke gate passed against the deployment. That evidence was
routed onto bob-cli-d1 and bob-cli-ay, and no new task was filed.

A fresh no-limit enumeration after batch 16 closed shows 117 unresolved tasks: 112 ready
and five snoozed (6q, aa, ah, cp, cq). Assign these five existing task beads, reused and
never cloned:

- bob-cli-by: reliable precommit macOS verification for Linux-launched agents. The Mac
  stayed offline (`ssh mac` timed out) through batches 15 and 16. Each batch still
  shipped macOS code that failed only once it reached master:
  - batch 15's f00ae66e tests never compiled;
  - 906a7df broke the Swift build (bob-cli-cz);
  - batch 16's e557b021 presenter test went red.
- bob-cli-co: 82 agent-report references that the bob-cli-cc.6 backfill skipped as
  ambiguous (47) or title-mismatched (35) still have no saved source. So they lack code,
  link, and Markdown copy.
- bob-cli-cm: the SASE research-file hook derives stems that include the artifact hash.
  A republished report dodges `bob ref create` dedupe and files a duplicate reference.
  Five `ref/chat/*-<12 hex>.md` duplicates exist.
- bob-cli-b1: `ensure_safe_to_write` refuses the whole `bob ref scan` with
  `dirty_targets` when any one PDF's targets are dirty. With
  `highlights.coordinate_pdf_writes` on in Bryan's managed config, one autosaved
  Highlights PDF can block the 15-minute Mac scan and Bob Refs ⌘S for every other PDF.
- bob-cli-8k: bob-mac-capture has no `.swift-format`. Agents that run the formatter
  re-indent whole files between 2 and 4 spaces. cb4c01e is 11,560/10,192 lines but
  1,532/164 with `-w`.

## Ranking and ownership

The hostile-process alert bob-cli-8r and the vault-sync halt alert bob-cli-9x still
outrank ordinary backlog work. Each needs its own large planned launch and must not be
folded into a backlog batch as a phase.

- co and cm are silent library damage: references missing their saved sources, and
  duplicate references created on every republish.
- by is the root cause of the recurring red macOS masters behind cz and batch 16's
  regression.
- b1 is a deterministic intake stall whenever one PDF is mid-annotation.
- 8k is recurring review and merge churn on every bob-mac-capture formatting run.

No active plan owns any of these five.

Deferred, with reasons:

- bob-cli-cv, bob-cli-5m, and bob-cli-5n: SASE runner and tooling defects whose fixes
  live in the sase repository.
- bob-cli-8q and bob-cli-b9: completion flakes kept with epic bob-cli-3j. Its phases and
  child epic are all closed, but its land agent has not run since 2026-10-02. Bryan
  should relaunch it (`sase bead work bob-cli-3j`). Re-check ownership at the next
  landing.
- bob-cli-58: waits for epic bob-cli-87's land. All 87 phases and its child epic are
  closed, but the epic is still in progress.
- bob-cli-b4 and bob-cli-ax: wait for Bryan's Multivisor and clipster decisions.
- bob-cli-ay, bob-cli-cx, and bob-cli-30: need Bryan's Mac session.
- bob-cli-cq: snoozed until a release tag contains 0f76d558. The latest tag, v0.3.1,
  does not.
- bob-cli-d1, bob-cli-1x, and the other memory tasks: each needs Bryan's authorization
  through `/sase_memory_write`.
- Next candidates: a resized bob-cli-v (357 clippy warning sites, seven +1s),
  bob-cli-cw, bob-cli-6n, bob-cli-8n, and bob-cli-a4.

## Phase details

### macos-precommit-gate

Read bob-cli-by and its corroborating notes first. Then read both CI workflows:

- bob-cli `.github/workflows/ci.yml` already runs the full suite on any push to `ci/**`.
- bob-mac-capture `.github/workflows/ci.yml` runs only on pushes to master and on pull
  requests.

Batch 16's mac-build-repair used a draft PR, and its lander used a hand-pushed
`ci/b16-presenter-diag` branch.

> [!decision] macos_gate = scratch_ci Ship one check script per repository (for example
> `scripts/dev/macos-ci-check` in bob-cli and a `Scripts/` peer in bob-mac-capture). It
> must:
>
> - snapshot the working tree into a throwaway commit object through a temporary index
>   (`GIT_INDEX_FILE`), never moving HEAD, a local branch, or the checkout's index;
> - push it only to `refs/heads/ci/precommit/<id>`, never master;
> - find the run for that exact SHA and wait with bounded first-run and total deadlines;
> - print each macOS step's verdict and the failing test names;
> - always delete the remote branch, including on failure, timeout, and interrupt;
> - exit 0 on pass, 1 on fail, and 2 when unrun (no gh auth, no run, or deadline
>   passed), so callers cannot mistake unrun for pass.
>
> Add a `ci/**` push trigger to bob-mac-capture so the check needs no PR. Ship an
> offline self-test with stubbed `git`/`gh` and wire it into each repo's script checks.
> Document the gate for agents in bob-cli docs and the bob-mac-capture docs. Do not edit
> any `AGENTS.md`; record a `PROPOSED FOLLOW-UP` against memory task bob-cli-1x instead.
> Prove the gate with one real run per repository, and confirm each scratch branch was
> deleted. Under `decline`, close bob-cli-by as canceled with the decision recorded and
> change no code.

The throwaway commit object is verification-only. It never lands, and it is not a commit
to the repository's history, so `/sase_git_commit` remains the only way to land work.
Never depend on the Mac: `ssh mac` stays best-effort, and nothing is provisioned.

### attach-source-skips

Read bob-cli-co, the attach-source section of `docs/highlights-ref-sync.md`, and the
candidate selection in `src/native/ref_library/attach.rs`. Ambiguous stems currently
return `Err("ambiguous")`. Read `decisions:reference-sources-are-preserved-write-once`
before changing anything; the store layout and write-once rule must not change.

> [!decision] attach_disambiguation = verify
>
> - For an ambiguous stem, run the existing block-match verification on every candidate
>   against the library PDF text. Attach only when exactly one passes; keep skipping
>   when zero or several pass.
> - Dry-run and JSON output name every candidate with its verdict.
> - Never loosen the block-match threshold. Title mismatches attach only through
>   explicit `bob ref attach-source REF -s FILE` pairs that a human reviewed.
> - Produce the pairing list from dry-run output for Bryan.
> - Add tests for a one-pass ambiguous stem, a zero-pass stem, a two-pass stem, and an
>   explicit pair.
>
> Then run a read-only dry run on athena with every research sidecar as `-s` and record
> the counts. The write pass runs only as commands of a Bryan-approved `/sase_gate`
> turn, in batches, each one revertable commit. Without that approval, record the
> dry-run counts and the exact gate commands as a limitation for the lander. Under
> `decline`, close bob-cli-co as canceled with the decision recorded and change no code.

### research-hook-dedupe

Read bob-cli-cm first. Then read the recapture-modes research report with
`sase artifact read research:202610/ref_create_recapture_modes/ref_create_recapture_modes.md "<reason>"`
and `plan:202610/ref_create_force_recapture.md`. The hook lives in the chezmoi
repository; open it with `/sase_repo`. Determine whether the hash-suffixed stem comes
from the hook script or from bob's stem derivation.

> [!decision] hook_dedupe = implement
>
> - Derive the stem without the artifact id hash.
> - Recapture a republished report with `bob ref create <md> -f overwrite` while the
>   earlier capture is still ready with 0 highlights. Otherwise use bare `-f`, which
>   adds a `<root>__vN` reading.
> - Add tests for the derivation and the mode choice in whichever repository owns them.
> - Before running `chezmoi apply` on the hook target, run `chezmoi diff` on it and
>   confirm it shows only this change; otherwise leave applying to Bryan and record it.
> - For the five existing duplicates, write a fold proposal: which family each joins
>   through `bob ref rename`, or which to archive. Execute it only as commands of a
>   Bryan-approved `/sase_gate` turn; otherwise record it for the lander.
>
> Under `decline`, close bob-cli-cm as canceled with the decision recorded and change no
> code.

Commit the chezmoi change through `/sase_git_commit` in that repository.

### per-pdf-dirty-refusal

Read bob-cli-b1. Then read these before planning:

- `ensure_safe_to_write` and `dirty_entry_allowed_for_plans` in
  `src/native/highlights_ref/guard.rs`;
- the hold vocabulary in `marker_write.rs`;
- the scan receipt;
- the `bob ref scan` JSON contract.

Check whether Bob Mac Capture decodes scan JSON. If it does, the new per-PDF row must
not break that decoder (`decisions:mac-capture-is-a-thin-client`); coordinate any
contract change across both repositories, and mark a JSON break with `!`.

> [!decision] dirty_pdf = hold Partition plans by their own dirty targets and keep the
> rest of the scan running. Report each dirty PDF as a hold (`retrying next scan`) in
> human output, JSON, and the scan receipt. Under `fail`, report the same PDFs as
> per-PDF failures instead. Under `decline`, close bob-cli-b1 as canceled with the
> decision recorded and change no code.

Under `hold` or `fail`:

- Never write a dirty target, and never add a bypass.
- Define and document the exit status when every PDF is dirty.
- Test one dirty PDF among clean ones, every PDF dirty, the JSON shape, the receipt, and
  the Bob Refs ⌘S path.

### swift-format-config

Read bob-cli-8k and bob-mac-capture's `AGENTS.md` first: never land Swift you have not
compiled. Then:

1. Measure the indentation split again.
2. Add `.swift-format` with the majority style: 4-space indentation, and a `lineLength`
   that matches existing code.
3. Re-indent the minority files in one dedicated formatting-only commit, and show that
   `git diff -w` for it is empty.
4. Check that multi-line string literals keep their content.

Compile before landing:

- through the macos-precommit-gate script if it has landed;
- otherwise through a draft PR, landing only when Build and Test pass, then closing the
  PR and deleting the branch.

Do not edit `AGENTS.md`. If it should mention the config, record a `PROPOSED FOLLOW-UP`.

## Acceptance

- bob-cli-by, bob-cli-co, bob-cli-cm, bob-cli-b1, and bob-cli-8k are closed normally
  with verification evidence. If Bryan declines one, close it as canceled with his
  decision.
- The batch-17 plan is validated and proposed through sase plan propose. Its reference
  and ownership are recorded on the batch-16 epic bead.
- Commit every changed repository, including the plans sidecar, through
  /sase_git_commit.
- At landing, the lander re-syncs bob-plugins from master with a stamp-aware bob and
  confirms that `bob plugins list` reports 0 drift. Then it runs the packaged rollout
  smoke gate
  (`xvfb-run -a npm run smoke:obsidian -- --deployed-vault <vault> --report <file>`) and
  records the verdict.
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
   - a resized `bob-cli-v` (357 warning sites; split by lint class or module, or narrow
     it to the gate-relevant subset);
   - `bob-cli-cw`, `bob-cli-6n`, `bob-cli-8n`, and `bob-cli-a4`;
   - Athena `bob-cli-b4` and `bob-cli-ax`;
   - `bob-cli-58` once active epic `bob-cli-87` has landed;
   - flakes `bob-cli-b9` and `bob-cli-8q`;
   - `bob-cli-ay`, `bob-cli-cx`, and `bob-cli-30` once Bryan's Mac session is available.

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
- macOS-only Rust (`#[cfg(target_os = "macos")]`) must pass the `aarch64-apple-darwin`
  clippy gate before it lands. Use stub `CC_aarch64_apple_darwin` and
  `AR_aarch64_apple_darwin` scripts and `CRATE_CC_NO_DEFAULTS=1`, then run
  `cargo clippy --target aarch64-apple-darwin --all-targets --all-features -- -D unused -A unused_extern_crates -A unused_macro_rules -D private_interfaces`.
- Linux `just check` cannot see macOS-only code. After every push, the lander checks the
  macOS CI job of both bob-cli and bob-mac-capture for the landed SHA, including the
  steps after a red lint or build.

Batch 17 adds two:

- A type-check is not a runtime verdict. When the Mac is offline, run new macOS-only
  tests on macOS CI before landing:
  - bob-cli: push a throwaway `ci/**` branch, which runs CI on push.
  - bob-mac-capture: open a draft PR until macos-precommit-gate adds a `ci/**` trigger.

  Use the packaged check once macos-precommit-gate lands. Delete every scratch branch
  afterwards.

- A native NSDocument presenter test that edits after a coordinated move must first wait
  for the presenter's post-move re-read (`wait_for_reloaded_document` in
  `src/native/file_coordination/macos_tests.rs`). The URL and text alone do not show
  that the presenter has applied the move.
