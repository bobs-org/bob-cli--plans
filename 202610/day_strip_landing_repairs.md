---
tier: tale
title: Repair Day strip visibility and rollover, integrate docs, and land bob-cli-bz
goal:
  The redesigned Day strip honors its lifecycle contract, the integration branch
  documents it, and bob-cli-bz is verified and closed.
size: medium
proposed_by: bbugyi200.athena.bob-cli-bz.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-bz
create_time: 2026-10-25 09:03:34
status: wip
---

- **PARENT:**
  [202610/day_strip_under_the_badges.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/day_strip_under_the_badges.md)
- **BEAD:**
  [bob-cli-bz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bz/README.md)

# Finish the Day strip lifecycle and land bob-cli-bz

Repair the remaining acceptance and integration gaps in epic `bob-cli-bz`, then complete
its landing in this coding turn. This is one bounded implementation, not another phased
redesign. This tale has no separate land agent.

## Scope and verified context

Read `sase bead read bob-cli-bz -r "Need landing audit and remaining repairs"` and
`sase artifact read plan:202610/day_strip_under_the_badges.md "Need the accepted Day strip contract"`.
The epic's three phases (`bob-cli-bz.1`, `.2`, `.3`) are closed. There are no DECISIONS
overrides and no parent bead (`parent_id: null`, confirmed by the lander). The original
plan is `plan:202610/day_strip_under_the_badges.md`, at
`202610/day_strip_under_the_badges.md` inside the plans repository. Preserve its design:
five report-owned LABEL-value facts, area mix, no buttons/history/collapse, under
daily-note plan badges in both rendered views, never writing note bytes.

Use `/sase_repo` to open `bob-plugins` before reading or editing it, and read its
`AGENTS.md`. Paths below are relative to the stated repository. Edit plugin source
fragments, regenerate `main.js` with `npm run build`, keep hand-edited fragments at most
1000 lines, and run `bob plugins sync` after changes. The plan hooks file is already 989
lines; keep lifecycle logic in the day-strip fragments.

The lander verified plugin commits `d77503c` (look) and `8388ef2` (plan host), read the
model/painter/CSS/registry/host source and tests, and reran all 47 Day strip tests,
`npm run build:check`, and the Chromium preview (84 cells, zero failures/overflow). The
540px light and 410px dark screenshots were inspected. The CodeMirror mount is deleted
and the plan repaint preserves the same host node. Do not redo this work.

Two isolated lifecycle probes found gaps absent from those passing tests:

- A cold automatic plan host fetches before its first visibility callback. In
  `239-plugin-day-strip-plan-host.js`, `dayStripSyncPlanHost` initializes
  `visible: true`, observes, paints, then immediately checks. Using the existing
  plan-host harness with a stubbed Stats spawn and no observer callback produced
  `{spawns: 1, visible: true}`; delivering the first offscreen callback then produced
  `{spawns: 1, visible: false}`. The new code-block host must uphold the existing
  visible-only fetch contract, including offscreen Reading-view blocks.
- `8388ef2` added an early return in `dayStripHandleRollover` after refreshing plan
  hosts. This skips synchronous repaint of explicit fence records. A visible fence
  seeded with today's report, advanced to tomorrow with a generation bump and a deferred
  refetch, retained `bob-day-strip-state-ready bob-day-strip-provisional` and `so far`
  while the request was pending. An offscreen fence also stayed unchanged. The pure
  model correctly returns `skeleton` for that stale report.

The docs phase claims commit `38ad9e88`, but repeated `git fetch origin` left the
integration branch and `origin/master` at `f4f6a9ca`, with that object absent.
`docs/stats.md` still describes the old bottom mount, history marks, chevron and fence
stand-down. Recover the actual docs change if it becomes available; otherwise reapply
that bounded phase on the current integration branch.

Integration review from epic creation found only unrelated test infrastructure:
`bob-cli` commit `f4f6a9ca` routes executable test stubs through a fork-safe writer;
`bob-plugins` commit `ab71502` stabilizes a ranker timing test. Neither needs Day strip
adaptation. Check for newer drift when implementing and integrate any actual feature
overlap.

Follow-up triage is already recorded on the epic: all four child notes were reviewed,
there are zero `PROPOSED FOLLOW-UP:` entries, and no separate tasks were accepted,
declined, or created. Existing memory tasks `bob-cli-bs` and `bob-cli-bt` both carry the
required supersession notes from `.3`. Do not duplicate them or edit memory. The repairs
below belong to this epic. Preserve this triage in the closing note.

## 1. Gate new plan-host fetches on established visibility

In `bob-plugins`, fix the plan host and its observer integration so an automatic host
can paint immediately from cache, but cannot start a fetch until its first intersection
notification establishes that it is visible and eligible.

Keep detached code-block processing safe: recheck hover-popover, markdown-embed, and
canvas-node ancestors after attachment and before any fetch. Detach excluded hosts,
disconnect their observers, and remove the spacing class. Do not make the production
path depend on a test observer's `trigger` method or a `closest()` call without a
selector. Simplify the current observer wrapper as needed and exercise the actual
callback path in tests.

Preserve a deliberate working fallback where IntersectionObserver is unavailable;
connected, eligible, shown hosts must eventually fetch, while disconnected or excluded
hosts must not. Preserve the shared per-key queue, hidden-tab checks, today/past age
limits, retry behavior, and cached first paint. Do not redesign the Stats page or widen
this into general cache work.

Add meaningful regression cases in `scripts/test-ledger-tools-day-strip-plan-host.cjs`:

- Cold host: zero spawns before the observer callback; first false callback remains
  zero; first true callback fetches once; subsequent clean callbacks do not refetch.
- Cache hit: synchronous ready paint before a callback, zero spawn.
- Detached host inserted into each excluded context: no fetch and proper release. Model
  connectivity through parent nodes correctly; a child of a detached parent must not
  pretend to be connected.
- No-observer fallback and unload cleanup.
- Existing repeated plan repaint tests still preserve the host and focused fact, without
  strip repaint or fetch. Keep source/Reading behavior and controls intact.

## 2. Restore rollover repaint for explicit fences

In `plugins/bob-ledger-tools/src/239-plugin-day-strip-registry.js`, make rollover
refresh automatic plan hosts for future-to-today attachment and also synchronously
invalidate/repaint every registered strip, including `kind: "fence"`. A report from
before local midnight must become a skeleton immediately, before any request settles.
Network checks remain visibility/dirty/age gated; offscreen repaint is free. Avoid
duplicate fetches or repeated unnecessary plan-host paints.

Extend the hosts/plan-host tests with a visible explicit fence whose rollover fetch is
deliberately held pending, and an offscreen explicit fence. Both must immediately stop
presenting yesterday's ready/provisional frame; the offscreen case must not spawn.
Resolve the visible request with a report for the new local day and verify the original
day renders as past with no `so far`. Also exercise an existing automatic today host
becoming past alongside the already-covered future attachment. Use a controlled clock or
the existing local-day seam, and distinguish immediate DOM assertions from assertions
after the request resolves.

## 3. Integrate the missing shipped documentation

In `bob-cli`, use `/sase_memory_read` to read `docs.md`. Recheck current history for the
`.3` docs change and integrate it if available; otherwise rewrite only the
`### Day strip` section of `docs/stats.md` to match the accepted contract:

- Under `bob-plan` badges on canonical daily notes, in Live Preview and Reading; daily
  notes without that block have no automatic strip.
- Five report-owned LABEL-value facts, then area time mix and today's fetch-time
  `so far · HH:MM`; no history marks, collapse, or separate Stats link.
- Honest loading, unavailable, warning, zero-work, stale and midnight states; metric
  hiding and redefined worked behavior remain report-driven.
- Explicit `bob-day-stats` fences and their optional `day:` key coexist with the
  automatic host; no fence stand-down.
- Existing shared-report/range selection and visible/dirty/aged refresh budget;
  automatic-only config/session controls and read-only behavior.

Check the daily-note counterpart link in `docs/dashboard.md` and the Day strip sentence
in `docs/reference/json-outputs.md`; edit only if now false. They were consistent at
audit time. The plugin docs and minor manifest bump already landed with `.2`; do not
repeat a feature bump or change `Cargo.toml`/`CHANGELOG.md`.

## 4. Verify and deploy the repairs

Run the focused Day strip suites first, then the required plugin gates: `npm run build`,
`npm test`, `npm run validate`, `npm run docs:check`, and
`node scripts/preview-day-strip.cjs --out <temporary-directory>`. Inspect the matrix
summary and representative light/dark screenshots; the existing 84-cell geometry matrix
must remain green. Run `bob plugins sync` from the prepared plugin checkout and record
its result.

In `bob-cli`, run `just docs-check` and file-change verification `just check`. Never run
`just check-full`: it is not authorized. If a gate is long, use `/sase_monitor` with a
continuation that explicitly retains all remaining closeout steps; do not end a turn
leaving an unmanaged command running. Handle any unrelated test-infrastructure issue
through the required task-triage workflow, without misclassifying it as missing Day
strip functionality.

The Mac smoke check remains Bryan's manual acceptance: sync plugins and reload Obsidian,
inspect today/past notes in both views, hover/click and return to vim, type through plan
repaints, check toggles and popover exclusion, and inspect narrow split/light/dark/200%
zoom. Report that this was not executed on the Mac; do not claim the Linux harness
verifies the actual Mac application.

## 5. Finish the epic landing in this same coding turn

Do not wait for this tale's commit SHA, push, or CI: the host commits after the turn
ends. Closing after verified working-tree changes is the normal landing. This tale must
perform closeout itself; no land agent resumes after it.

1. Reread `bob-cli-bz` and all descendants/notes. Confirm the above repairs and newer
   integration drift are resolved. If SASE created a tracked descendant bead for this
   tale, close that completed descendant normally before closing the epic. Preserve the
   existing phase completions and the recorded follow-up triage.
2. Run `sase bead epic-symbols bob-cli-bz`. Resolve each entry by wiring, privatizing,
   applying a justified non-test pragma, or deleting it per policy; re-key only if a
   still-open later bead truly needs the exemption. There were no entries at audit time.
   Do not leave stale exemptions to another agent.
3. Close normally with `sase bead close bob-cli-bz --note "<verification>"`. The note
   must state: all child notes and design constraints reviewed; plugin commits verified;
   automatic-host visibility and fence rollover repaired with regression tests; missing
   docs integrated; post-start changes checked; actual gate results and sync outcome; no
   proposed follow-ups/no new tasks and existing bs/bt supersession notes retained;
   symbol cleanup outcome. Never force merely to succeed or to advance a successful
   nested landing. If rejected for symbols or incomplete phases, resolve that concrete
   blocker and retry normally.
4. Run `just symvision` if that recipe is available; it was absent in the audited
   bob-cli justfile. Explicitly record absence if still unavailable, and confirm there
   are no epic symbol exemptions.
5. Open `plans` with `/sase_repo`, using only the path it prints. The original epic plan
   is `202610/day_strip_under_the_badges.md` within that checkout. Use
   `sase artifact read plan:202610/day_strip_under_the_badges.md "Need the epic plan for final closeout"`
   for audited context and set `status: done` in that file's frontmatter.
6. Run `sase bead read bob-cli-bz -r "Need the parent link"`. There is no parent as of
   the audit, so finish normally. If the link has changed, follow the user's
   phase-versus-plan ancestor closeout rules and stop with a recorded blocker at the
   first incomplete ancestor; never force a successful landing.
7. Use `/sase_final` as the last action before the final response, declaring every
   modified primary, plugin, and plans repository for host commit. Report the epic
   closure and verification results, plus the remaining manual Mac smoke check.
