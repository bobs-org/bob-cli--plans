---
tier: tale
title: Finish cycler relation successors and close bob-cli-fs.8.1
goal:
  Correct effective-state eligibility and result plumbing, verify and deploy the cycler,
  then close the child epic and parent phase bob-cli-fs.8.
size: medium
proposed_by: bbugyi200.athena.bob-cli-fs.8.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fs.8.1
create_time: 2026-11-02 09:22:28
status: wip
---

- **PARENT:**
  [202611/cycler_relation_successors.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/cycler_relation_successors.md)
- **BEAD:**
  [bob-cli-fs.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/bob-cli-fs.8.1.md)

# Finish the cycler relation-successor landing

Complete the remaining defects in **bob-cli-fs.8.1**, then close that epic and its
parent phase **bob-cli-fs.8** in this same coding turn. This tale has no separate land
agent. Do not wait for this turn's commit SHA, push, or CI: the host commits after the
turn ends. Leave the containing epic **bob-cli-fs** to its waiting land agent.

This is bounded repair of the existing cycler classifier, runtime adapter, and result
plumbing. One coding agent can finish it directly. Do not redo the shipped policy,
picker, copy library, or all four completed phases.

## Context and authority

Read `sase bead read bob-cli-fs.8.1 -r "Need landing evidence and triage"`, its four
closed children `.1` through `.4`, and parent phase `bob-cli-fs.8`. Read these plans
with `sase artifact read`:

- `plan:202611/cycler_relation_successors.md` — the exact child epic being finished.
- `plan:202611/starts_after_relation.md` — binding parent contract and decisions.

Use `/sase_repo` and
`sase repo open bob-plugins -r "Repair cycler relation successor landing defects"`; read
its `AGENTS.md`. All implementation paths below are relative to that opened checkout.
Follow `/sase_memory_read` for successor/recovery decisions and relevant glossary. Read
`docs.md` if documentation needs correction. **No memory edits** are needed or
authorized here: parent decisions authorize only the earlier three glossary changes and
explicitly reject a new decision record.

Limit production changes to task-status-cycler and necessary tests/generated build. The
public parked-event/unlink API belongs to later phase `bob-cli-fs.9`. Preserve its work
if it has since landed, but do not implement it as part of this repair.

## Verified baseline and evidence

At audit, bob-cli was `cb729a76` and bob-plugins was `2ead3a8`; both matched freshly
fetched origin/master. The four child commits are `4447723` (policy/pure rule/copy),
`65d25d9` (merge/picker), `f4550e2` (Done runtime), and `2ead3a8` (ship 1.46.0). All
child notes were reviewed; the epic initially had no notes of its own.

The 195 tests in the relation-successors, successor-wiring, successors,
successor-offers, successor-polish, keep-moving, project-pull-in, successor-theme-write,
and relations suites passed. Build `--check` passed. Those tests do not establish
completion: six additional behavior probes failed, while the 25 existing wiring tests
included in the same process passed.

Audited, durable evidence:

- `file:explicit:bcf92583e130ba24ca6dd32d` — executable regression probe. Read through
  `sase artifact read`; the script dynamically appends tests to the existing wiring test
  module without changing tracked files. Recreate it locally or port the cases into the
  permanent suite; invoke it with the opened plugin checkout as cwd.
- `file:explicit:f1a7a88d67cb76f8a82fbe6d` — 25 passes and six failures, including
  actual returned models and notices.
- `file:explicit:793fe12a35ddf65729ac6f00` — separate unrelated-test reproduction.

Six confirmed gaps:

1. Esc after a picker containing an eligible relation row and a Pending held row emits
   an unwanted `Already started` notice. `137-plugin-successor-project.js` puts
   `relation_picker_opened` on the internal plan but omits it from the returned model,
   so `135`/`138` cannot suppress held output.
2. A genuine `?` dependent recovered by completing its prerequisite links through
   hand-off but loses its relation second reason. `085d` marks recovery, while
   `089d::relationSourceTask`/`relationBlockedDetails` still classify the original `?`
   line. Several current SA14/SA15 tests supply already-effective statuses or
   `relationRecoveries`; they do not prove actual production recovery integration.
3. For a predecessor with no dependency `[id::]`, a Ready relation source whose own
   prerequisite is already Done is incorrectly held. `137` never requests the available
   Tasks snapshot just because relation candidates need dependency data.
4. A successful relation-only result has `daily_content: null` even after inserting its
   link. The final refresh condition still checks only unblocked/keep-moving.
5. With a warm snapshot, a Ready source with an open own prerequisite is incorrectly
   offered. `089d` sees snapshot availability but receives no computed prerequisite
   count unless another recovery path happened to supply one.
6. A Ready resident of a project whose `^prj` is still Blocked is incorrectly offered.
   R4 depends on sparse recovery metadata rather than checking the candidate's effective
   project state. The probe uses a future-scheduled `^prj` with a resident whose
   checkbox has not yet been reconciled.

Adjacent source-confirmed omissions to fix with the same result plumbing: `130` counts
reopened rows only in `unblocked`, so recovered residents moved into
`relation_successors` must be included; held target copy can be empty because
`successorDisplayText(found)` is given a parser row with `lineText`, not `rawLine`. Held
`waits_for` text must name the target task, not merely its block ID.

## Repair and verify

1. **Make R4 consume real effective state.** In `085-successor-plan.js`,
   `085d-successor-relation-merge.js`, `089d-successor-relation.js`, and the `137`/`138`
   adapter, derive prerequisite and project holds for each actual candidate using the
   existing dependency/project helpers and fresh note overlay. Request/reuse dependency
   data lazily after a positive incoming lookup only when candidate eligibility needs
   it, including an idless predecessor. An unavailable snapshot remains distinct from a
   known empty pool; conservatively hold affected candidates. Preserve zero added reads
   for disabled and edge-free gestures.

   Classify genuine recoveries against their post-recovery status, dependency counts,
   and project state, retaining exact original lines for guarded writes and receipts. A
   relation never causes recovery by itself. Preserve dedup priority
   `hand_off > relation > project > project_finished > keep_moving`, all triggering
   reasons, and independent recovery when the picker is dismissed. Verify actual blocked
   hand-off and project-resident recovery through `planSuccessors` and
   `finalizeClosedTasks`, without injecting expected recovery verdicts.

2. **Complete result and presentation plumbing.** Carry picker-opened state through the
   returned internal model to plain notices, nav v3 notices, and the walk's returned
   toast; keep internal metadata out of the nav payload. An opened picker suppresses
   held notices even on Esc. Held-only gestures still show their correct shared copy,
   and already-planned rows stay silent. Preserve real failures and mixed
   dependency/relation counts. Resolve held target descriptions from fresh parsed task
   text and use the existing 48-scalar copy helper.

   Include relation rows in daily-content refresh and reopened accounting, using the
   existing unified row helper where appropriate. Audit the other row-array consumers
   for the same omission. Keep single-use, same-day, untouched-only receipt ownership,
   including multi-predecessor conservatism.

3. **Turn the probes into permanent behavioral tests and finish the required runtime
   coverage.** Keep all six reproductions and add real project recovery with picks/Esc
   and truthful reopened counts. Exercise the post-picker pass when an own prerequisite
   or project becomes blocked, a new live link appears, the event target ceases to be
   Done, config/index readiness changes, and a new relevant outgoing target is added.
   Refresh only relevant notes, refuse stale picks individually, never substitute
   another offer or open a second picker. Preserve the original gesture baseline for
   R2/anchors separately from current live state; verify placement after a closing-entry
   relocation.

   Cover at least one real managed Done gesture and the review-walk pending picker
   lock/one advance, in addition to calls directly into the finalizer. Check the
   existing routes (task, Task Link, transclusion/tree, Pomodoro embedded close, Alt+]
   into Done, complete-label finalize API, PRE/POST) still reach the same adapter.
   Retain v2 picker disablement, v2 notice fallback, empty-label and link_unblocked
   gates, cancel no-query, idless tasks, edge-free read count, and
   untouched/edited/next-day receipts. Do not claim missing coverage from a passing
   generic successor test alone.

4. **Integrate and validate the repaired build.** Recheck commits on both checked-out
   branches and origin/master since the audit. During this epic, plugin changes
   `33318e0`/`75e7875` changed Stats, `66741d4` changed off-clock chips, and
   `89dae58`/`257255e`/`5f1c23c` added blocked-mark rendering. Their new test
   registrations are retained and they introduce no new successor writer; do not
   duplicate their classifiers or change rendering from the cycler. Main-repo
   `311b1514`, `b483c93b`, and `fa1118c1` changed blocked-mark docs, capture section
   selectors, and calendar ordering. The sibling Rust engine work remains separate.

   Canonical bob-cli and cycler relation vectors currently share SHA256
   `08ffccedd68489dbfc06c73faf7a1be65df7b49cb48fdee1804f98d748c11cc0`. Navigation
   SA23/SU-P22 parity and Unicode truncation are already owned by `bob-cli-fs.2.1`'s
   repair tale; recheck that ownership without competing edits.

   Run the focused suites above and all required affected gesture/walk tests, then
   `npm run build`, `npm run validate`, `npm test`, and `npm run docs:check` in the
   plugin checkout. Keep every source fragment at most 1000 lines; extract helpers if
   needed and register fragments before regenerating main.js. Run `just check` in
   bob-cli for file-change verification; **do not run `just check-full`**. Use
   `/sase_monitor` if a command requires a handoff, and never end with a live foreground
   command. Repair regressions caused here; compare new unrelated failures against the
   clean base and keep the exact evidence. A `just check` pass with a `just check-full`
   failure is test infrastructure work.

   The plugin is already 1.46.0; do not repeat the feature minor bump. Update its
   documentation only if the repaired contract needs clarification. Run
   `bob plugins sync --repo <opened-plugin-checkout> --no-pull -p task-status-cycler`
   and verify deployment succeeded. Use a scratch Obsidian vault for a live
   Ctrl+Enter/pick/Esc smoke if available; otherwise record the availability limit
   explicitly. Never test by changing Bryan's live tasks.

5. **Finish the landing in this same turn.** Re-read the epic and child readiness,
   verify the repair and intervening integration, and use the already-completed
   follow-up triage below in the close note. Resolve or legitimately re-key every entry
   from `sase bead epic-symbols bob-cli-fs.8.1` to a still-open later bead that
   genuinely needs it; audit currently finds none. Then run
   `sase bead close bob-cli-fs.8.1 --note "<source/commit audit, six repairs, checks, integration, deployment, smoke limit, and all follow-up outcomes>"`
   normally. If symbol cleanup is refused, finish it and retry. Never use `--force`
   merely to succeed or to advance this successful nested landing. An unfinished named
   phase requires finishing/reopening it; deliberate canceled/superseded scope is the
   only case for the user's explicit `--force --reason ... --resolution ...`
   alternative.

   After close, run `just symvision` if available (no recipe existed at audit). Use
   `/sase_repo` to open `plans` and set only `status: done` in the frontmatter of
   **`202611/cycler_relation_successors.md`**, the PLAN file for
   `plan:202611/cycler_relation_successors.md`; consume its content using audited
   artifact reads. Include this changed sidecar in finalization.

   Inspect the parent link with
   `sase bead read bob-cli-fs.8.1 -r "Need the parent link"`. The concrete parent is
   phase **bob-cli-fs.8**. Verify the child has completed its Done-engine scope, run
   `sase bead epic-symbols bob-cli-fs.8`, retire any entries, and close only that phase
   with
   `sase bead close bob-cli-fs.8 --note "<engine and all managed Done route verification, test results, deployed version, known baseline failures, and live-smoke status>"`.
   Leave **bob-cli-fs** open and do not mark `202611/starts_after_relation.md` done. If
   an actual blocker remains, keep the affected bead open and record precise evidence
   rather than force-closing it. Submit `/sase_final` for all repositories changed by
   this tale.

## Follow-up triage already completed by the lander

Do not refile these proposals. Copy their outcomes into the epic close note:

- `bob-cli-fs.8.1.4` note #1: independent clean-HEAD reproductions corroborated
  **bob-cli-et** (freshness due-key date), **bob-cli-f7** (UTC/local time),
  **bob-cli-eh** (both startup fakes), and **bob-cli-ff** (expired future date), each
  with `+1` and `file:explicit:793fe12a35ddf65729ac6f00`.
- Note #2: corroborated **bob-cli-f8** with a fresh warm-snapshot budget miss of
  72.3ms >50ms. Ship's cold-chunk observation (19.4ms >16ms, isolated passes) remains in
  that existing investigation; no claim of an identical root cause.
- Note #3: independently confirmed navigation fixture drift and corroborated the
  causally responsible active **bob-cli-fs.2.1** epic, whose repair tale already owns
  it. No separate task and no parallel navigation edit.
- New tasks were deliberately declined for all proposals because existing tasks or an
  active epic own the work. No other child proposed follow-ups exist. The six defects
  above were discovered in this landing and remain this epic's work, never external task
  beads. These outcomes are recorded on both `bob-cli-fs.8.1` and `bob-cli-fs.8`.
