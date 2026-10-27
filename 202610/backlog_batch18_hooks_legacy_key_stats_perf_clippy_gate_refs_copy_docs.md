---
tier: epic
title: 'Backlog batch 18: the retired hooks JSON key, bob stats speed, a clippy gate,
  and the Bob Refs Copy card and browse images'
goal: Close five existing tasks by removing the released legacy_dependency_children
  key, bringing bob stats toward its 300 ms target, gating the mechanical clippy lints,
  fixing the Copy card C/L row previews, and refreshing the Bob Refs browse images.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6
phases:
- id: hooks-legacy-key
  title: Remove the pinned legacy_dependency_children key from hooks JSON
  size: small
  depends_on: []
  description: 'hooks-legacy-key: resolve bob-cli-cq by deleting the key that v0.4.0
    shipped pinned at 0, with a breaking-change commit and updated docs and tests.'
- id: stats-perf
  title: Shrink the live-vault bob stats index pass toward 300 ms
  size: large
  depends_on: []
  description: 'stats-perf: resolve bob-cli-6n per the stats_perf decision by profiling
    the single-pass stats index and cutting avoidable work without changing output.'
- id: clippy-gate
  title: Fix the mechanical clippy lint classes and gate them
  size: large
  depends_on: []
  description: 'clippy-gate: resolve bob-cli-v per the clippy_scope decision by applying
    behavior-preserving fixes for the mechanical lint classes and denying them in
    just lint.'
- id: mac-refs-polish
  title: Fix the Copy card C/L row previews and refresh the Bob Refs browse images
  size: medium
  depends_on: []
  description: 'mac-refs-polish: resolve bob-cli-cw (tail truncation and accent dot
    on the C/L rows) and bob-cli-a4 (browse images that predate the NOW section) in
    bob-mac-capture.'
decisions:
  stats_perf:
    ask: Should batch 18 profile and speed up the bob stats index pass on athena and
      apollo?
    choices:
      implement: Profile the release build and cut avoidable index work; output stays
        byte-identical
      decline: Close bob-cli-6n as canceled; the Mac already renders stats in about
        200 ms
    default: implement
    why: athena and apollo still take 340-600 ms against the plan's 300 ms target
    answer: implement
  clippy_scope:
    ask: Should the mechanical clippy lint classes be fixed everywhere and then denied
      in just lint?
    choices:
      gate_mechanical: Fix every mechanical-class site, deny those lints; judgment
        lints stay advisory
      decline: Close bob-cli-v as canceled; clippy warnings stay advisory and keep
        growing
    default: gate_mechanical
    why: Warning sites grew from 357 to 435 in two days; only a gate stops the growth
    answer: gate_mechanical
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-27 11:12:32
status: done
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.md)

# Backlog batch 18: the retired hooks JSON key, bob stats speed, a clippy gate, and the Bob Refs Copy card and browse images

This epic continues the bounded backlog campaign. Batch 17
(`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6`) closed bob-cli-by, bob-cli-b1, and
bob-cli-8k, and left bob-cli-co and bob-cli-cm open with lander notes:

- by: `scripts/dev/macos-ci-check` (bob-cli) and `Scripts/macos-ci-check`
  (bob-mac-capture) are byte-identical precommit gates. Each snapshots the working tree
  onto a throwaway `ci/precommit/<id>` branch, waits for that exact SHA's run, prints
  every macOS step, and always deletes the branch. Exit codes: 0 pass, 1 fail, 2 unrun.
  The lander fixed a settle race (b4a5850d / 3f70d49): a completed run whose macOS job
  still read `in_progress` had exited 2. Docs: `docs/develop/ci.md` in both repos.
- b1: af3a8fd4 turns dirty write targets into per-PDF `dirty_targets` holds
  (`retrying next scan`). Clean PDFs proceed, and an all-held scan exits 0. Bob Mac
  Capture f8a258f shows the hold as a footer.
- 8k: bob-mac-capture `.swift-format` pins 4-space indentation and 100-column lines. The
  lander re-indented f8a258f's 2-space lines (e6296bc) and added the AGENTS.md mention
  (ed8f3d3).
- co: the verify-each-candidate code shipped (69b7d962). The write pass waits for
  bob-cli-d2's landing and a Bryan-approved `/sase_gate`.
- cm: the stable research hook is deployed (chezmoi 10e9f1d2). Folding the five existing
  duplicates needs a gate, Mac scans, and Bryan's decision on the annotated row.

A fresh no-limit enumeration after batch 17 closed shows 114 unresolved tasks: 109 ready
and five snoozed (6q, aa, ah, cp, cq). `git tag --contains 0f76d558` now lists v0.4.0,
so the lander woke cq. Assign these five existing task beads, reused and never cloned:

- bob-cli-cq: v0.4.0 shipped `legacy_dependency_children` pinned at `0` for one release,
  as batch 14's `legacy_key = pin` decision required. The key can now go.
- bob-cli-6n: release `bob stats` on athena and apollo takes 340-370 ms, or 0.4-0.6 s
  under load, against the Daily Stats plan's 300 ms target. The Mac measures 190-200 ms.
- bob-cli-v: `cargo clippy --all-targets --all-features` reports 435 unique warning
  sites on master b4a5850d (357 at batch 10). The largest classes:
  - 124 `collapsible_if`/`collapsible_else_if`;
  - 48 `too_many_arguments`;
  - 29 `needless_borrow`;
  - 22 `needless_borrows_for_generic_args`;
  - 12 `large_enum_variant`.

  166 sites are under `src/native/highlights_ref`.

- bob-cli-cw: the Copy card's C and L rows middle-truncate away `● K on this page`, and
  draw the dot gray instead of in the accent color.
- bob-cli-a4: `docs/images/refs-browse{,-dark}.png` come from CI run 39330761089 (commit
  c81e00a, 2026-10-19), which predates ac09943's NOW section.

## Ranking and ownership

The hostile-process alert bob-cli-8r and the vault-sync halt alert bob-cli-9x still
outrank ordinary backlog work. Each needs its own large planned launch and must not be
folded into a backlog batch as a phase.

- co and cm (silent library damage) outrank this batch but are not actionable by phase
  agents. Each needs Bryan-approved live-vault gates, and co must also wait for
  bob-cli-d2 to land.
- cq is a deterministic contract cleanup whose release gate has just opened.
- 6n is the highest-ranked performance item.
- v is a growing lint debt that no gate stops.
- cw and a4 are user-visible Bob Refs polish, now verifiable from Linux through the
  batch-17 macOS gate and the CI render artifacts.

No active plan owns any of these five.

Deferred, with reasons:

- bob-cli-co: waits for bob-cli-d2's landing (its tale is changing the same verifier and
  backfilling the same vault), then a re-dry-run and a Bryan-approved write gate.
- bob-cli-cm: the duplicate fold needs Mac scans, a Bryan-approved gate, and Bryan's
  decision on the annotated `finalizer_completion_contracts-9c652d326895`.
- bob-cli-8n: a live-vault `AGENTS.md` edit that runs only through a Bryan-approved
  gate.
- bob-cli-cv, bob-cli-5m, and bob-cli-5n: SASE runner and tooling defects whose fixes
  live in the sase repository.
- bob-cli-8q and bob-cli-b9: completion flakes kept with epic bob-cli-3j, which is still
  in progress. Its land agent has not run since 2026-10-02; Bryan should relaunch it
  (`sase bead work bob-cli-3j`).
- bob-cli-58: waits for epic bob-cli-87's land. 87 is still in progress.
- bob-cli-b4 and bob-cli-ax: wait for Bryan's Multivisor and clipster decisions.
- bob-cli-ay, bob-cli-cx, and bob-cli-30: need Bryan's Mac session.
- bob-cli-1x (now +4, naming the macos-ci-check gate), bob-cli-d1, and the other memory
  tasks: each needs Bryan's authorization through `/sase_memory_write`.
- Snoozed 6q, aa, ah, and cp: their date, toolchain, publication, or usage gates have
  not arrived.

## Phase details

### hooks-legacy-key

Read bob-cli-cq, then `docs/task-status-hooks.md` and §10 ("Compatibility output") of
`docs/task-dependencies.md`. Confirm that `git tag --contains 0f76d558` lists a release
before editing.

1. Delete `legacy_dependency_children` from the hooks model and JSON output
   (`src/native/task_status_hooks/model.rs`, `sync.rs`, `retry.rs`), and remove any
   summary text that still mentions legacy children.
2. Update both docs and the CLI tests under `tests/cli/task_status_hooks/` that assert
   the key. Add one test that the JSON no longer contains it.
3. Search bob-plugins, bob-mac-capture, and chezmoi for readers of the key (open each
   with `/sase_repo`). The batch-17 lander found none; record the search either way.

This is a CLI/JSON contract break. Mark the commit with `!` and a `BREAKING CHANGE:`
footer, per `decisions:bob-cli-releases-ride-a-daily-train`.

### stats-perf

Read bob-cli-6n, the "Engine architecture" section of `plan:202610/daily_stats.md`,
`docs/stats.md#performance`, and `src/native/stats/index.rs`.

> [!decision] stats_perf = implement Measure first: time a release build on athena
> against the live vault, and profile the index pass (file walk, per-file Tasks parse,
> `done/` residence mapping, daily-note ledger and field scans). Use
> `BOB_STATS_REAL_VAULT_PARITY=1 BOB_NOW='2026-10-11 12:00:00' cargo test --release --lib real_vault_parity_is_opt_in`
> to keep parity. Then cut avoidable work, for example by prefiltering lines before the
> Tasks parser or skipping ledger scans on non-daily notes. Output must stay
> byte-identical: run the existing parity and golden tests, and compare `-f json` before
> and after on the live vault. Do not add timing assertions to CI. Record before and
> after numbers for athena (and apollo when reachable) in `docs/stats.md#performance`.
> If profiling shows the remaining cost is irreducible I/O, record that evidence and
> close bob-cli-6n with it rather than forcing a risky rewrite. Under `decline`, close
> bob-cli-6n as canceled with the decision recorded and change no code.

Read only the live vault; never write to it.

### clippy-gate

Read bob-cli-v and its batch-10 scope note. Measure again on current master with
`cargo clippy --all-targets --all-features --message-format short`. Count unique
`file:line` sites per lint class, and record the per-class counts on the phase.

> [!decision] clippy_scope = gate_mechanical Fix every site of the mechanical,
> machine-applicable classes, such as `collapsible_if`, `collapsible_else_if`,
> `needless_borrow`, `needless_borrows_for_generic_args`, `explicit_auto_deref`,
> `useless_conversion`, `needless_return`, and `unnecessary_lazy_evaluations`. Prefer
> `cargo clippy --fix` passes reviewed per module over hand edits. Then add exactly
> those lints as `-D clippy::<lint>` to the `just lint` / `just check` clippy
> invocation, so they cannot return. Judgment lints stay advisory: `too_many_arguments`,
> `large_enum_variant`, `type_complexity`, and any class whose fix would change behavior
> or public signatures. List them in the developer docs as advisory. Under `decline`,
> close bob-cli-v as canceled with the decision recorded and change no code.

Constraints for this phase:

- Fixes must not change behavior. Run `just check` on the committed tree.
- The new deny list also applies to macOS-only code. Run the `aarch64-apple-darwin`
  clippy gate described below with the same deny list before landing, and watch the
  macOS CI job.
- Concurrent agents edit `src/native/highlights_ref` and `src/native/ref_library`.
  Rebase immediately before committing. On a conflict, rerun `cargo clippy --fix` on the
  rebased file rather than merging hand edits. Split commits by module where that keeps
  them reviewable.
- Do not touch generated files or vendored code. Do not raise any lint's level beyond
  the agreed list.

### mac-refs-polish

Read bob-cli-cw (evidence: `file:explicit:322b1400187c912a478f0c18`), bob-cli-a4,
bob-mac-capture's `AGENTS.md`, `plan:202610/ref_code_block_copy.md`, and
`plan:202610/ref_link_copy.md`.

1. **cw**: give the C and L Copy card rows `.tail` truncation, or drop the site list
   before the page information. Their previews must keep `N … · ● K on this page`.
   Render the leading `●` in `Color.accentColor`, as the picker rows already do. The
   code lives in `Sources/RefsCore/RefsCopy.swift` (`RefsCopyRow`, `codeRow`,
   `linkPickerContentsRow`) and `Sources/BobMacCapture/Refs/RefsCopyCardView.swift`. Add
   RefsCore tests for the truncation and dot ranges, and update the design fixtures.
   Follow `.swift-format`: format only the lines you change.
2. Verify on macOS with `Scripts/macos-ci-check` before landing (the Mac may be
   offline). Then inspect the light and dark Copy card renders in that run's
   `render-fixtures` artifact.
3. **a4**: after the cw commit lands and master CI is green, run `just docs-media` to
   import `refs-browse.png` and `refs-browse-dark.png` from that green master run's
   render fixtures. Confirm both images show the NOW section and the current why-here
   wording. Commit them with the updated `docs/media.yml` provenance. Run
   bob-mac-capture's `just docs-check` before committing.

Do not edit any `AGENTS.md`.

## Acceptance

- bob-cli-cq, bob-cli-6n, bob-cli-v, bob-cli-cw, and bob-cli-a4 are closed normally with
  verification evidence. If Bryan declines one, close it as canceled with his decision.
- The batch-18 plan is validated and proposed through sase plan propose. Its reference
  and ownership are recorded on the batch-17 epic bead.
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
   - `bob-cli-co` once bob-cli-d2 has landed (re-dry-run, then a gated write pass);
   - `bob-cli-cm`'s gated fold and `bob-cli-8n`'s gated vault `AGENTS.md` fix, once
     Bryan can approve gates;
   - Athena `bob-cli-b4` and `bob-cli-ax`;
   - `bob-cli-58` once active epic `bob-cli-87` has landed;
   - flakes `bob-cli-b9` and `bob-cli-8q`;
   - `bob-cli-ay`, `bob-cli-cx`, and `bob-cli-30` once Bryan's Mac session is available;
   - the advisory clippy classes if clippy-gate leaves a reviewable remainder.

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
   and `bob-cli-cp` are not obsolete simply because their date, toolchain, publication,
   or usage gate has not arrived. Verify real prerequisites, not just wake timestamps.
   Work other actionable tasks first. If only gated tasks remain, the next bounded epic
   must own the readiness check and supported `/sase_monitor` or `/sase_questions`
   handoff needed to resume; do not abandon the chain, spin on fresh identical epics, or
   close a valid task merely to reach zero. Do not interfere with another worker's
   claim.
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
- A type-check is not a runtime verdict. When the Mac is offline, run new macOS-only
  tests on macOS CI before landing.
- A native NSDocument presenter test that edits after a coordinated move must first wait
  for the presenter's post-move re-read (`wait_for_reloaded_document` in
  `src/native/file_coordination/macos_tests.rs`).

Batch 18 adds two:

- Use the packaged precommit gate for macOS verification from Linux:
  `scripts/dev/macos-ci-check` in bob-cli or `Scripts/macos-ci-check` in
  bob-mac-capture, run through `/sase_monitor` (it takes about 15-20 minutes). Exit 0 is
  a pass, exit 1 is a real CI failure, and exit 2 means unverified, never a pass. Draft
  PRs and hand-pushed `ci/` branches are no longer needed.
- Swift in bob-mac-capture follows the pinned `.swift-format` (4-space indentation, 100
  columns). Write new lines in that style and never re-indent whole files. The batch-17
  lander had to re-indent a phase's 2-space additions after the fact.
