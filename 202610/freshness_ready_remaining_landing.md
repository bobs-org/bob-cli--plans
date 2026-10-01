---
tier: tale
title: Finish freshness READY integration and vault rollout, then land bob-cli-3b
goal:
  Repair the verified snapshot and daily READY integration defects, deliver the
  NEW/ROTTEN vault cutover with the latest compatible plugins, account for live
  verification honestly, and close epic bob-cli-3b in this coding turn.
size: medium
proposed_by: bbugyi200.athena.bob-cli-3b.land
bead: bob-cli-3b
status: done
---

- **PARENT:**
  [202610/freshness_gated_ready.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/freshness_gated_ready.md)
- **BEAD:**
  [bob-cli-3b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3b/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-3b.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-3b.land.md)
- **COMMITS:**
  - [8d9a229](https://github.com/bobs-org/bob-cli/commit/8d9a2292f2f2c82e6b3a8a953780e8b31daf8d84)
    — docs(plan): correct daily bob-plan block to the four-chip contract

# Finish the remaining work and land bob-cli-3b

This is bounded remaining work for one coding agent. You must finish the epic's landing
yourself: this tale has no separate land agent. Do not wait for this turn's own commit
SHA, push, or CI result before closing completed beads. The host commits the final
source after the coding turn ends.

## Audit already completed

The lander read the epic, every child, every child note, the approved parent plan
`plan:202610/freshness_gated_ready.md`, the actual Rust/plugin implementation, and the
commits. Phases bob-cli-3b.1/.2/.3 are CLOSED/done. Their primary commits are 6d54e39,
286ff35, and e77bbe3; plugin commits are 570f40d, d5c1281, and 3cb3016. The Rust bucket
mapping, schema-2 envelope, rotten vocabulary, canonical budget key precedence, and
legacy budget warning are implemented. Existing tests passed 56 Rust unit tests and 25
CLI tests (`cargo test freshness -- --test-threads=1`), 1,078 plugin tests (`npm test`),
six manifest validations, and `cargo fmt --check`. These passes do not cover the defects
below.

The reported vault commit f7d7a2c9 is absent from the opened remote checkout at
4f72dbef. That checkout still has `freshness.md`, a REVIEW chip, ungated READY, the old
review chores, and no `rotten.md`. Publishing an external clone alone never satisfied
the live cutover requirement. Recover that existing rollout if available through audited
repository/artifact access; otherwise restore its approved edits against the current
vault, preserving newer notes and changes.

All five phase `PROPOSED FOLLOW-UP:` entries have been triaged on bob-cli-3b:

- .1 note #1, .2 note #1, .3 note #1: the existing BOB_DAY_FILE process-env race;
  corroborated ready bug bob-cli-2e (+1, now +3). Duplicate flake tasks declined.
- .2 note #2 and .3 note #2: incomplete deployment/verification belongs to this epic and
  this tale, not separate tasks. The future trial only needs to be ready to run; do not
  wait two weeks or claim its results.
- Separately discovered missing `just check` recipe is ready task bob-cli-3c. It
  predates this epic. Do not expand this tale to fix unrelated infrastructure.

There is currently no parent_bead on bob-cli-3b. No `--epic-symbol` entries were
reported for the epic or phases. Recheck both at closeout.

## 1. Reopen source and integrate newer changes

Read bob-cli-3b and its new notes plus the approved plan through audited commands. Use
`/sase_repo` to open `bob-plugins` and `gh:bobs-org/bob`, reading their AGENTS and
statuses. Use only returned checkout paths. Read artifacts with `sase artifact read`; do
not read canonical sidecar artifacts directly or reuse another agent's checkout path.
Use `/sase_memory_read` for `sase_beads.md`, `obsidian.md`, `glossary:freshness`, and
`decisions:ready-is-freshness-gated`.

Fetch and inspect changes since the epic's first commit, excluding its own commits;
inspect the remote base branch as well as the local checkout. The lander identified
bob-cli cd7a8cf and bob-plugins beb634f, associated with
`plan:202610/daily_ready_badge_dialect.md`. Preserve the daily single-tone READY
styling, dashboard two-tone styling, READY-only aggregate-over feedback, and plugin-only
`ready_cap_exceeded` lint. PLAN status/enforcement is unchanged. Ledger-tools 1.13.1
includes these changes; nav-hotkeys 1.49.0 includes namespace-v3 consumers. Integrate
further drift if present. No phase replay or evaluator redesign.

## 2. Repair the verified snapshot and daily-renderer defects

Work in `plugins/bob-ledger-tools/main.js` and its focused test suites.

1. **Identity collisions.** `freshnessBuildMemo` fills `evaluatedByKey` by unconditional
   `Map.set`, while `freshnessEvaluatedFor` prefers that map even for an exact task
   object already in `indexByTask`. Last-writer overwrite breaks the partition.
   Reproducer at pinned local date 2026-10-08: two visible TODO tasks in `notes/a.md`,
   lineNumber 0 and 1, both blockId `^duplicate`; first originalMarkdown
   `- [ ] #task New`, second `- [ ] #task Fresh [fresh:: 2026-10-08]`. Independent
   evaluation gives NEW/FRESH, but both API buckets are null, counts.new=1 and gated
   READY=2 for only two tasks. Serve each resolvable source row's own cached result and
   prevent an ambiguous/missing identity from borrowing another row's classification.
   Preserve the existing neutral policy for genuinely unresolved identity. Cover
   distinct objects, duplicate block IDs, line identity, and cloned Tasks query objects.
   Assert actual membership plus badge/model equality, not totals alone. No task-line
   writes or persisted classifications.
2. **Warm interval lookup.** `apiFreshnessIntervalFor` ensures a memo, calls
   `apiFreshnessRowFor(task)` without it (ensuring again), and reparses the line. Use
   the already cached intervalDays/intervalSource from the same evaluated result used
   for state/bucket. On a miss use the already acquired memo. A warm-call counter
   currently reports two `freshnessEnsureMemo` calls; require one acquisition and no
   warm row reparse/re-evaluation/config read.
3. **Actual daily READY tooltip.** `planBlockModel` creates the correct
   `readyModel.tooltip`, but `paintPlanBlock` passes only count/cap/over into
   `paintReadyElement`, dropping lane pressure. With NEW=1, ROTTEN=0, READY=1, the model
   says `READY 1/100 · lane 2 = 1 new + 0 rotten + 1 ready`, while the rendered badge
   uses the legacy tooltip. Carry the same snapshot's lane data to the actual shared
   renderer. Assert rendered title/aria, counts, and lifecycle refresh agree with
   dashboard READY, including unavailable, old daily notes, and legacy/throwing
   freshness fallbacks. Preserve beb634f's styling and lint; do not introduce another
   renderer or per-chip polling loop.
4. **Correct the surface docs.** bob-cli `docs/plan.md` daily Surfaces row and
   bob-plugins README table claim daily NEW/ROTTEN chips, but `paintPlanBlock` renders
   TODAY/PENDING/NEXT/READY. Correct those claims to the four-chip daily contract. Keep
   the dashboard's seven-chip order and five sections intact. The approved epic required
   shared gated daily READY, not new daily chips.

Bump ledger-tools over the latest source version and match its README version. Preserve
freshness namespace v3, top-level API v3, and CLI schema 2.

## 3. Finish the approved vault cutover and coordinated deployment

Use the original plan's dash-gating specification; restore only missing changes:

- `dash.md` section order TODAY → NEW → PENDING → NEXT → READY. NEW is unlimited; READY
  keeps the old visible TODO/Today predicate and excludes new/rotten buckets.
- Chips NEW/PENDING/NEXT/READY/BLOCKED/ROTTEN/TODAY, lifecycle-owned live NEW/ROTTEN
  models, shared READY, guarded tiny legacy fallback, no REVIEW-only code.
- Rename `freshness.md` to `rotten.md`, preserving its parent and Review/Freshness
  review aliases plus Rotten Tasks. Source tasks stay in their notes. Always show
  RETURNED and ROTTEN groups, ranked from the shared queue, with source-line review
  instructions and unavailable `–` feedback.
- Update `gtd_daily.md` morning/weekly chores and `blocked.md` TOMORROW copy; migrate
  current navigation targets to rotten, including Markdown/wikilinks.
- Include the 2026-10-05 through 2026-10-18 trial tally: NEW, RETURNED, expired ROTTEN,
  confirmed FRESH, READY, red-chip flag. Keep if red on at most three mornings, about 30
  confirmed tasks on most mornings, and no lost-needed-task case. If late, record actual
  dates for a full 14 days.

Deploy through the normal supported vault workflow (`bob vault-sync`) so the cutover
reaches the live vault, and deploy plugins from the opened source using
`bob plugins sync --no-pull --repo <opened-bob-plugins> --plugin bob-ledger-tools` and
nav-hotkeys (dry-run first). Never force dirty-file guards or patch deployed plugin
copies. Record destination, sync result, and deployed versions. Preserve all newer vault
content. Do not make landing depend on this tale's future commit: deploy the verified
source in this turn through the supported workflow, leaving source commits to the host
finalizer. If access/publication is genuinely blocked, record the precise blocker and
leave the epic open; do not call an edited clone a successful live deployment.

Where remote deployment is needed, read `tailnet.md` first and use the documented
athena/apollo/Mac path. Plugins are gitignored and require separate per-machine
deployment. Verify deployed `bob freshness list -f json` is schema 2, rotten
counts/rows/config names, and a single diagnostic for a valid legacy budget key. Do not
change machine clocks, restamp real tasks, reseed, or bulk review tasks.

## 4. Validate the fixes, partition, and rollout

Run file-change verification with `just check`. Never run `just check-full`. At audit
time `check` does not exist (bob-cli-3c); if still absent, record that specific
infrastructure failure and use the existing focused Rust commands as additional
evidence, without calling it a `just check` pass. Read/parse/config counters and
representative fixture size/timing must demonstrate warm lookup behavior; avoid brittle
wall-clock thresholds. Run focused freshness, READY, Today, plan-budget, mark, and
navigation suites, then plugin `npm test` and `npm run validate`, plus
`git diff --check` and Rust formatting/freshness tests. Report independently confirmed
baseline failures accurately; the environment race belongs to bob-cli-2e. A passing
`just check` with a failing `check-full` would be an infrastructure defect, not
remaining feature work.

Execute the actual final vault Tasks predicate strings with a stubbed API against one
representative pinned-date fixture covering NEW/RETURNED/ROTTEN/READY and exemptions.
Assert pairwise-disjoint union of B, group/count/chip equality,
Today/Next/Pending/Blocked/hidden/template/conflict exclusions, and the actual fallback
sections/counts with missing/old/throwing APIs. Cross-check the same vault/config/date
against CLI schema 2. Cover effective-interval escalation, config
creation/deletion/invalid recovery/legacy null-precedence, same-array Tasks updates,
source stamp/frontmatter edits, Today membership, and simulated midnight without hooks
or task writes. Check exact destinations and text spans.

Compare hooks `--dry-run` before/after: no feature-induced rewrites. Review the vault
diff for unintended stamps, lane/status changes, copied tasks, or tags. Only intended
chore edits and explicitly verified completion of the two named vault tasks are
authorized task-content changes. After corresponding behavior passes, complete
`bob_gtd.md#^hide-rotten-tasks` and `#^scheduled-are-stale` using the existing
completion convention, retaining IDs/children; S11 must pass before the latter. Leave
`^wip-next-refresh` and unrelated tasks alone.

Record actual Obsidian evidence where GUI access is available: chip text,
click/hover/keyboard destinations, marks in Reading/Live Preview, Alt+F and NEW review
into READY, updates after note edits/Tasks reload/Today changes, and daily READY
typography plus ready_cap_exceeded boundaries. Measure actual rerender timing when
possible. The approved epic expressly permits unavailable GUI/platform checks to remain
a concrete gate: report exact machine, steps, expected results, and still-unverified
task completion instead of fabricating a pass. It does not permit omitting the actual
live vault cutover. Trial outcome is future human work; ready instructions satisfy this
epic.

## 5. Close the epic in this turn

After implementation/deployment and the applicable checks are complete, reread
bob-cli-3b, all three children and their notes, and its linked plan. Recheck descendant
readiness and any new drift. Include all note outcomes in the close note: bob-cli-2e
corroboration, declined duplicate flake proposals, rollout proposals fulfilled here,
bob-cli-3c infrastructure tracking, repaired epic defects, newer commits integrated,
verification results, and precise GUI gates.

1. Run `sase bead epic-symbols bob-cli-3b`. Resolve every exemption by wiring,
   privatizing, adding an appropriate non-test pragma, or deleting the symbol. Re-key
   only to a still-open later bead that actually needs it. Do not leave exemptions keyed
   to this epic or its completed phases.
2. If this tale has its own assigned descendant bead, finish and close that completed
   implementation bead normally before closing its ancestor. Do not wait for its source
   commit. Confirm all named phases are genuinely done.
3. Close normally:
   `sase bead close bob-cli-3b --note "<the verification and outcomes above>"`. If
   symbols reject the close, clean them and retry. If unfinished phases reject it,
   finish/reopen them; use cancellation/supersession with `--force --reason` only for an
   intentional abandoned scope. Never force a successful nested landing or use force
   just to make the command succeed.
4. Run `just symvision` after closing when available. It was absent from this repo's
   justfile during the audit; explicitly record absence if still absent.
5. Set `status: done` in the frontmatter of the original epic plan
   `plan:202610/freshness_gated_ready.md`, using the PLAN path returned by
   `sase bead read`. Open the plans repo with `/sase_repo` before editing it and include
   every changed repository in `/sase_final`'s host commit declaration.
6. Re-read bob-cli-3b for its parent_bead. None existed at audit time. If still absent,
   finish normally. If a parent phase was attached, verify its required work and close
   only that phase normally, leaving its containing epic to its waiting lander. If a
   parent plan was attached, review its previous landing note, all descendants/notes,
   linked plan, and drift; recheck readiness, retire its epic symbols, close it
   normally, run symvision if available, and mark its plan done. Repeat through directly
   parented complete plan ancestors only. Stop at the first incomplete/ambiguous parent,
   note its blocker, and report it.

Submit `/sase_final` as the last action before a normal final response. Commit
declarations describe the actual changed repos; don't manually commit just to obtain a
SHA for this landing.
