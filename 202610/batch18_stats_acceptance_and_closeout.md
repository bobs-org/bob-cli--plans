---
tier: tale
title: Finish batch 18 stats acceptance and closeout
goal:
  Verify or reduce Apollo's actual stats index cost, finish the image provenance and
  rollout checks, and close batch 18 with evidence.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6
status: done
---

- **PARENT:**
  [202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.land.md)
- **COMMITS:**
  - [8c83a3a](https://github.com/bobs-org/bob-cli--plans/commit/8c83a3a81e02ddcb48fcf808a6642253b92631ed)
    — chore(plans): mark batch 18 and its stats closeout tale done

# Finish batch 18's stats acceptance and land the batch

Finish only the work left by the landing review of
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6`. This tale has no land agent; its coder
must finish the epic's closeout in the same turn as the final edits. Do not wait for
this tale's own commit SHA, push, or post-commit CI. Use the host finalizer for commits;
the current landing prompt supersedes the older plan's commit-before-close ordering.
Precommit snapshot CI is available when needed.

Read the epic and its landing notes, and use `sase artifact read` for
`plan:202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md`.
The final decisions remain `stats_perf = implement` and
`clippy_scope = gate_mechanical`; do not re-ask them.

## Verified baseline and scope

The lander audited every note on phases `.1` through `.4` and their commits:

- `b1c6be2c` removes `legacy_dependency_children` from hooks JSON with absence
  assertions, updated docs, `!`, and a `BREAKING CHANGE:` footer. Release v0.4.0
  contains the earlier pin commit `0f76d558`. No readers exist in bob-plugins,
  bob-mac-capture, or chezmoi.
- `a1dc0206` fixes and denies 48 named mechanical Clippy classes in both `just lint` and
  `just check`; judgment classes stay advisory. Its Linux check, macOS precommit gate,
  and landed CI passed. Keep this policy intact.
- `4a43170b` adds stats measurements and fixes the opt-in CLI parity test's explicit
  vault selection, but retains no performance change. Athena's index medians are
  178.73/190.89 ms. Apollo's full-command warm medians are still 333.67–345.16 ms, and
  its index stage was not measured. Identical baseline/final executable hashes prove the
  code was unchanged, not an optimization. The decision's fallback is evidence of
  irreducible I/O, not an assertion that a rewrite feels risky.
- bob-mac-capture `3dbbbf9` gives C/L rows tail truncation and an accent dot. The lander
  inspected light/dark renders from green run 40533608326 and both committed browse
  PNGs. They show the complete on-page counts, NOW, and `Now · BLOG Pomodoro`. Browse
  provenance is green master `ed8f3d3`, before the Copy fix; the exact approved sequence
  calls for a post-Copy green master import.

Integration reviewed through bob-cli `6dea0058` and bob-mac-capture `de57e2b`:
web-source commits `f8dc405b`, `4fda80b1`, and `6dea0058` preserve the mechanical
rewrites; docs-only `9613af85` has no consumer overlap. Mac `135667c` adds an alias
hotkey, `05b6f92` adds Shift-open in the link picker, and `de57e2b` widens source hints;
C/L preview semantics remain intact. Refresh this drift check before landing.

No child note contains `PROPOSED FOLLOW-UP:`. Follow-up triage is complete and recorded
on the epic: the unchanged-SHA rerun of Mac run 40538557703 passed its Test step after
attempt 1 timed out. New large flake task `bob-cli-d4` owns that unrelated residual
synchronization issue, with evidence `file:explicit:8f63ebf1b7cb1b1b8ecba607` and a
related link to the older, closed `bob-cli-ci`; do not duplicate it or mistake it for
Copy-card work. No separate advisory-Clippy task was proposed, and an initial smoke
invocation with Node 20 was corrected to the documented Node 22 prerequisite, not filed
as a bug. Carry every triage outcome into the close note. No epic-symbol entries existed
at review; no `symvision` recipe was available. Recheck both.

The lander closed `bob-cli-cq`, `bob-cli-v`, and `bob-cli-cw` with verification
evidence. `bob-cli-6n` and `bob-cli-a4` remain for this tale. Re-read their current
statuses; close each only after its specific acceptance has evidence. This tale creates
no new feature or task inventory. Baseline `just check` passed on `6dea0058` (ToolRun
`be95f4a4561839bf73cc3a367c943a7d`, 345985 ms), and current CLI CI 40547163812 passed
Linux, macOS, lint and package, including macOS-only tests.

## 1. Measure the actual stats index on Apollo, then remove demonstrated waste

Read
`sase memory read tailnet.md docs.md -r "Need stats measurement access and docs rules"`
and the Engine architecture of `plan:202610/daily_stats.md` through audited artifact
access. Build the current baseline in release mode under `$CARGO_TARGET_DIR`, and
preserve a temporary baseline executable. Read-only vault access is permitted; never
write the live vault or install into another host's checkout.

Use temporary copied executables on Apollo via the documented SSH alias. Profile Athena
and Apollo with the same clock (`BOB_NOW='2026-10-11 12:00:00'`), `NO_COLOR=1`, and
explicit vault path. Capture host load, source revision, binary hashes, command argv,
one initial run, and at least nine interleaved warm baseline/final runs. Report
index-only and full-command time separately. The original target is <=300 ms for the
single-pass index; do not declare it missed or met solely from wall time.

Use `bob stats -f json` for overview and `bob stats show -a -f json -r 4w` for all
metrics. `bob stats -f json -r 4w` is not the all-metrics command: correct labels in old
evidence if necessary. Temporary stage counters must be removed from production code
before the final checks.

If the actual Apollo index already meets the target, record that direct evidence; do not
force a parser rewrite. Otherwise retain a measured, behavior-preserving improvement. A
concrete first candidate is repeated daily-note scanning:

- `src/native/stats/index.rs::parse_daily_note` builds line vectors and calls
  `pomodoro_homes::session_links(contents, entry_index)` for every completed session.
- That wrapper reaches `src/native/capture_pomodoro_start.rs::list_session_links`, which
  rebuilds `line_spans`, a whole-note line vector, and a whole-note fenced-line set on
  every call. Share an immutable per-note prepared scan across those calls, keeping the
  existing wrapper for other callers. Preserve line numbers, indentation, struck-link
  inclusion, fenced-block exclusion, and marker handling exactly.
- Keep one source of truth for link parsing; do not add a simplified stats parser. The
  daily ledger stage measured 46.73–50.32 ms on Athena, enough to justify measuring this
  candidate first. Only add shared API that has a production caller.

If Tasks parsing is still the dominant avoidable stage, examine
`src/native/dataview/tasks/index.rs::StatsTaskParser::parse_file` and the canonical list
parser. Any prefilter must prove it cannot discard a supported task, including ordered,
quoted, nested, custom-status, and fenced/comment cases. Prefer avoiding allocations or
repeated work without changing grammar; do not introduce caches, parallel writes, or new
CLI options. A whole-file fast path must preserve the canonical parser's observable
output.

Document real before/after numbers in `docs/stats.md#performance`. If no safe
optimization is needed because both measured indexes meet the target, say so. The
approved irreducible-I/O fallback requires stage evidence on the remaining slow host.
CPU-heavy parsing or unmeasured Apollo indexing is not that fallback; keep the epic open
if the selected branch is still unfulfilled.

## 2. Verify output parity and complete the image provenance requirement

Preserve byte-identical JSON across baseline/final/baseline brackets for live overview,
worked comparison, all metrics over four weeks, and a stable fixture. Record hashes and
byte counts without publishing vault contents. Run the existing stats library and CLI
suites, goldens, and release opt-in real-vault parity. For a shared ledger-scan change,
also cover multiple sessions, nested links, struck links, fences, Unicode, and line
endings with meaningful regression cases and the existing capture/pomodoro-homes tests.
No wall-clock timing assertions in CI.

Open bob-mac-capture with `/sase_repo`. From a green master run containing `3dbbbf9` (at
review `de57e2b`, run 40545781911, passed every required step), run `just docs-media`.
Review its generated diff, keep the requested browse images and honest `docs/media.yml`
provenance, and inspect both themes for NOW and current why-here wording. Do not
redesign the card or change Swift. Run that repository's `just docs-check`. Use the
imported run's real SHA and timestamp, not the rerun time or the local checkout HEAD.
Preserve any unrelated generated changes according to the repository/finalizer rules
rather than silently discarding them.

## 3. Finish landing verification and preserve the verified plugin rollout

Run `just check` on the final bob-cli tree and `just docs-check`. Never run
`just check-full`. A check-full infrastructure failure is not unfinished epic work. Use
`sase tool run check` for bounded inline execution; if it escalates, join that same run
with `/sase_monitor`. Do not restart a still-running command.

Inspect the current main-branch diff since the review revisions in both repositories and
adapt overlapping callers/tests to this change. Keep the explicit Clippy gate on newly
landed web-source code. Use the packaged macOS precommit gate through `/sase_monitor` if
final code needs a fresh macOS verdict; inspect all required steps, including those
after a red lint/build. Exit 2 means unverified. Do not wait for a commit that this tale
can only receive after its turn ends.

The lander already completed the original batch's plugin rollout: fetched current
bob-plugins master `b13d634`, passed build/build:check, previewed then performed a
stamp-aware sync (0 copied, 0 skipped, 22 unchanged), and verified 8 synced plugins with
0 drift. The packaged post-sync smoke passed with Node 22: layout-ready 528 ms, peak
heap 51.7 MiB, relation ready, all four editor cases, zero exceptions/rejections/
crashes. Durable report: `file:explicit:5660f972d2e620e3bac9e32b`.

Recheck whether plugin source/deployed bytes drifted. If unchanged, carry forward that
evidence; do not repeat a successful rollout merely because this tale changes stats
code. If drift requires a new rollout, open bob-plugins with `/sase_repo`, read its
instructions/deployment docs, fetch/rebase current master, build and build-check, then
preview/sync with the stamp-aware bob and confirm zero drift. Never bypass a foreign
uncommitted-deploy refusal with `-F`. Run
`xvfb-run -a npm run smoke:obsidian -- --deployed-vault <vault> --report <file>` with
Node 22 actually selected in that checkout, and record the verdict. No arbitrary
live-vault edits, live publishes, credential handling, Cargo version edits, CHANGELOG
edits, or memory changes are in scope.

## 4. Close this epic and check its concrete parent

This is the tale's final implementation step. Closeout is part of the coder's work, not
a request for another land agent.

1. Re-read the epic and all phase notes, verify any new notes are addressed, carry
   forward the lander's follow-up triage outcomes (including declined proposals), and
   close the five target tasks normally with their verification evidence. Do not close
   `bob-cli-6n` until step 1 meets the selected decision. If a genuinely new unrelated
   issue was discovered while implementing, use `/sase_new_task` before filing, then
   record its deduplication/ownership outcome in the epic close note.
2. Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6`. Resolve
   every listed entry by wiring, privatizing, a justified non-test pragma, or deleting
   it per the Symvision policy. Re-key only when a specifically named, still-open later
   bead truly needs the exemption. Never leave entries keyed to this epic or its closing
   phases.
3. Close normally:
   `sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6 --note "<source and decision verification, actual Apollo index/optimization evidence and JSON parity, integration review, final check and macOS results, image provenance, plugin smoke verdict, and every follow-up outcome>"`.
   Run `just symvision` if available; otherwise record the missing recipe. Open the
   plans sidecar with `/sase_repo` and set `status: done` in
   `202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md`.
   If symbols or unfinished descendants reject close, finish them and retry. Never use
   `--force` to advance a successful nested landing.
4. Re-read the parent link and inspect the concrete direct parent plan
   `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6`, its previous landing notes, descendants
   and notes, linked plan
   `plan:202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md`,
   and post-child drift. At review it was already closed, but `bob-cli-co` and
   `bob-cli-cm` remained ready with incomplete live-vault repairs requiring gates and
   prerequisites. Do not equate a prior closed status with current readiness. Recheck
   those blockers. Under the current landing prompt, stop at the first incomplete or
   ambiguous parent, append a note explaining it, and report it. If fully complete,
   rerun descendant/linked-plan readiness checks, retire its epic symbols, close it
   normally (re-closing is a no-op), run symvision when available, mark its linked plan
   done, and repeat only through directly parented plan ancestors that remain fully
   complete. Never force an ancestor closed.
5. Re-enumerate unresolved tasks without a limit and report the actual counts:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Completing this batch does not complete the broader backlog campaign. Preserve its
   remaining tasks and their prerequisites; do not expand this repair tale into another
   batch or claim the campaign is complete. The current landing prompt's
   stop-at-incomplete-parent rule governs the ancestor handoff. If there is no such
   blocker and continuation is appropriate, the original plan's bounded successor
   contract still applies: reuse at most five existing actionable tasks, at most five
   phases with explicit dependencies, copy its continuation contract and working rules,
   and submit through `/sase_plan` after this closeout. Do not treat old suggestions or
   automatic memory declines as new feature authorization.

Commit obligations include the primary code/docs, any changed linked repository, and the
plans sidecar. Use `/sase_final` for the final declaration after all normal work and
verification; a successful SASE handoff is the documented exception.
