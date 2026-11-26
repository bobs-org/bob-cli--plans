---
tier: tale
title: Finish snoozed reader recovery and landing
goal: Restore direct snoozed recovery, align ledger membership and hooks forecasts,
  then close bob-cli-q5.2.1 and parent phase bob-cli-q5.2.
size: medium
proposed_by: bbugyi200.athena.bob-cli-q5.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-q5.2.1
status: done
---

- **PARENT:**
  [202611/snoozed_planned_today.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_planned_today.md)
- **BEAD:**
  [bob-cli-q5.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.2.1.md)

# Finish snoozed reader recovery and landing

Finish the remaining work of epic `bob-cli-q5.2.1`, whose approved plan is
`plan:202611/snoozed_planned_today.md`, then close that epic and its delegated parent
phase `bob-cli-q5.2`. Leave containing epic `bob-cli-q5` to its own lander. This is one
bounded Rust integration repair with tests and contract updates. This tale has no land
agent afterward: its coder must perform the closeout below in the same turn as the final
code, before the host commits that code. Do not make closeout depend on this work's
future commit SHA, push, release, or CI result.

## Verified starting point

The landing review read every note on the epic and all four children, both approved
plans, the governing decisions, actual source, and epic commits `b8678657`, `bbcae8e0`,
`4a812173`, and `f32e774c`. Baseline `f32e774c` matches fetched `origin/master`. No
commit for `bob-cli-q5.2.1.1` exists in reachable history. Its reported recovery
implementation did not land; phase .2 explicitly reported that loss and supplied only
the shared reader and review exclusions.

Present and reviewed: snapshot reader `snooze/read.rs`; ordinary, recurring, checklist,
placement, seed, and lane-walk exclusions; ledger-only Today and whole-lane plan counts;
clock-aware advisory forecast; schema-1 separate agenda rows and resolved wake-history
provenance. Keep this work. The epic's SP4 failure note is addressed by `bbcae8e0`
separating ignored state from reason, with full rebuilt vector verification recorded on
phase .2.

Three remaining gaps are concrete:

1. `task_status_hooks/sync.rs` computes `snoozed` near line 415, but only passes
   membership into freshness/unblocked-field placement near line 993. Neither
   `recovery_status` nor the daily-note `directly_recent` keep predicate uses it. Thus a
   daily Next task is cleared, and a Blocked task whose reasons disappeared recovers to
   Ready, despite its snoozed row. The current `task_transition` logic makes both
   failures explicit. An isolated installed-bob 0.6.0 smoke also reproduced
   `cleared: daily`, `unblocked recover to " "`, `kept_next: 0`; that smoke is
   corroboration, not exact-source build verification.
2. `plan_budget_for_sync` returns `None` at `!ledger.has_section` before the new
   enrichment. A Snoozed-only daily note therefore has a forecast in `bob plan` but none
   in hooks. The approved child plan required enrichment outside that guard for report
   paths.
3. `snooze/read.rs::ledger_identities` treats any unstruck block link in a direct ledger
   child as live, even `- Discuss [[tasks#^job]] next week`. Today correctly excludes
   this prose. It causes an otherwise live snoozed row to become
   `ignored/already_linked` and vanish from the new review/forecast/agenda views.
   `snooze/plan.rs::live_identities` duplicates an even broader scan including nested
   descendants. Align these classifiers with the existing dedicated-link extractor;
   preserve identity-map information needed by mutation planners. The installed-bob
   smoke confirmed ignored/already_linked alongside Today 0.

Post-start drift was reviewed from `19e7fe54` (before epic creation), as well as from
the first epic commit: q8 dependency/plain-link reconciliation and capture ties; ph
bold-drawer cutover; q4 note placement and settle acceptance; q7 caret documentation;
and q5.3 capture snooze grammar, atomic mutations, and completion (`1b386638`,
`ae78dbb2`, `712d69d9`). Keep the landed relation folding and settling. The important
new integration is actual capture-created snoozed rows flowing through reconcile and the
new readers, covered below. Recheck drift when starting.

## Binding decisions and scope

The final outer choices remain `uncommitted_rows=ignore`, `wake_notice=hud`,
`hold_in_meetings=yes`, `glossary_snoozed_task=yes`, `decision_record=no`. No memory
edits belong to this tale. No plugin, Mac, live-vault, release, version, or CHANGELOG
changes are required. Never seed freshness on the real vault.

Governing decisions:

- `decisions:today-is-read-from-the-ledger`: dedicated links under today's open
  Pomodoros define Today. Snoozed membership remains separate.
- `decisions:task-lanes-are-sticky`, `decisions:projects-block-from-prj-line`,
  `decisions:subprojects-inherit-project-blocking`, and
  `decisions:unblocked-tasks-resurface-as-ticklers`: retain blocking precedence, daily
  clearing exceptions, legitimate recovery ranks, and freshness WRITE skips.
- `decisions:consumption-keeps-notes-and-recorded-time`: retain ledger history and
  authored notes; reconcile never consumes or edits Snoozed rows.
- `decisions:ready-is-freshness-gated`, `decisions:review-walk-is-tiered`,
  `decisions:recurring-occurrences-walk-when-due`,
  `decisions:review-attention-follows-the-walk`, and
  `decisions:w-count-reads-lane-excess`: keep existing buckets, overlays, ordering, and
  derived attention/counts while correcting membership.
- `decisions:off-clock-follows-tag-then-note-default` and
  `decisions:off-clock-note-defaults-filter-by-task-type`: continue using
  LinkClockContext for forecasts; no parallel clock policy.
- `decisions:idle-capture-shows-ledger-agenda` and
  `decisions:mac-capture-is-a-thin-client`: preserve agenda schema 1 and existing roles,
  numbers, and rendering; bob owns the facts.

## Implementation

1. Reuse the existing snapshot-backed planned set in `sync.rs`. Compute one direct
   membership boolean per task. Merge `Some(RankedStatus::Next)` into that task's
   recovery rank only, keeping the strongest legitimate ledger/activity rank. Include
   direct snoozed membership in the daily Next keep predicate. Do not add identities to
   `desired`, `direct_desired`, `recent_activity_roots`, dependency traversal, or
   Pomodoro references. Derived blocking still wins; Ready remains unchanged. Keep all
   source reads in the existing guarded scan. Retain the existing membership input to
   freshness placement.
2. Make shared snooze live-ledger membership use `plan_budget::today::today_links` (the
   pure extractor backed by `capture_pomodoro_start::list_queued_links`), or a shared
   lower-level equivalent, resolving each link through the existing index/identity map.
   Use this rule in both `read::ledger_identities` and `plan::live_identities` so
   list/read and wake agree. Preserve all spelling-to-identity mappings needed by
   add/drop/wake and capture's staged identity-map adapter. Do not casually narrow
   consumption's broader note-preserving mixed-link cleanup; the change here is
   membership.
3. Assemble and enrich hooks' report before deciding whether it has useful data. A
   Snoozed-only note with planned rows must publish its snoozed meter/forecast, with
   Today and current links zero. Preserve `plan_budget: null` for invalid config and for
   an absent Pomodoros section with no planned snoozed rows, so empty-day compatibility
   stays bounded. Keep normal schema-3 reports, budget status, current cap enforcement,
   and whole-lane meters unchanged. Update the helper comment and documented null
   condition to match.
4. Read `docs.md` with `/sase_memory_read` before editing documentation. Complete the
   missing recovery contract in hooks long help, `docs/task-dependencies.md`, and
   `docs/snooze.md`; update hooks' Snoozed-only report rule and dedicated-link
   definition where their exact contracts live. Regenerate owned help with
   `just docs-gen`. Do not restore phase-relative comments about missing readers.

## Verification

Add meaningful binary-level regressions using the existing isolated Tasks settings
helper and fixed Bob-zone date/clock. Pin both dry-run reports and committed bytes:

- Daily-note Next kept, absent-row daily Next still cleared, and a previous-day row
  unable to keep today's task.
- Snoozed Blocked task stays blocked while its own/project/inherited reasons remain;
  after those reasons disappear it recovers to Next without a ledger link. A
  prerequisite not directly snoozed receives no recovery from the snoozed dependent.
- Stronger existing recovery rank wins; Ready never promotes; ignored, terminal, Event,
  ambiguous, duplicate-loser, and history rows supply no extra reach.
- Snoozed section bytes (including nested notes, duplicate rows and struck history)
  survive actual reconcile with LF and CRLF; a second run is idempotent. Existing
  relation-fold and guarded-snapshot behavior still works.
- Membership parity: plain/embedded/aliased/same-note dedicated links win; prose
  mentions, nested notes, fenced links, and struck links do not. Verify list, wake,
  freshness/walks, plan, and agenda agree, with Today still ledger-derived.
- Hooks and `bob plan` report the same snoozed forecast for a Snoozed-only note, plus
  ordinary no-row/missing-section and blocked/off-clock cases. Preserve status and
  current links even when the forecast exceeds cap.
- Exercise an actual `bob capture -- "@tasks:job z:15:00"` and a daily-note target
  through live reconcile, plan, freshness, walks, and agenda, then wake and repeat wake.
  Use bold drawer fixtures and the already-landed settle pipeline. Upgrade the existing
  agenda lifecycle test where useful: it currently checks only dry-run reconcile on a
  non-daily Next task, which cannot detect missing recovery.

Run focused snooze, hooks recovery/blocking/consumption, freshness/seed/placement, walks
SP vectors, plan_snoozed, and pomodoros_agenda suites. Then run `just check` and
`just docs-check` on the final tree. Never run `just check-full` for this task. Use
`/sase_monitor` before long commands when a handoff is required; its successor must
retain this tale's explicit closeout responsibility. Fix introduced failures. Known
unrelated failures are not permission to ignore new or changed diagnostics. Compare any
retained red nodes against an untouched base and record exact evidence.

## Follow-up triage already completed by the lander

The lander recorded every proposal and disposition on `bob-cli-q5.2.1` before proposing
this tale. Do not create duplicates:

- Phase .1 #1 docs inventory: resolved; untouched bob-plugins `3af803f9` passes
  `npm run docs:inventory:check` and is clean.
- Phase .1 #2 manual_contains: resolved by `71f7b670`; current assertion uses
  `ids.contains(&"dup")`.
- Phase .2 #1 missing recovery: this tale's epic work, not a standalone task.
- Phase .2 #5, .3 #1, .4 #1 baseline reports: 80 drawer nodes routed with proposing bead
  attribution to active `bob-cli-ph`; 17 settlement nodes routed to active `bob-cli-q4`;
  WISPr duplicate corroborated on `bob-cli-q9` (+4). The comprehensive phase .2
  comparison is `file:explicit:0bc82d5d4b7f7164deb53e4c`.
- Phase .3 #2 hooks grouping failure is one of those 17 settlement nodes, already in
  q4's existing repair; no separate hooks task.
- Epic note #1 SP4 ignored-state assertion: fixed by `bbcae8e0`, retained tests must
  continue to pass.

No new task beads were needed. The main epic remains open because its recovery and
integration behavior is incomplete. Verification in this tale must be against the
then-current source, not the installed-bob corroborating smoke.

## Final step: finish this landing in the coding turn

After the fixes and verification, re-read `bob-cli-q5.2.1`, all four child scopes and
notes, and linked plan `plan:202611/snoozed_planned_today.md` using audited reads.
Confirm every phase requirement and note is addressed; include this tale's fixes,
post-start integration, decision compliance, test evidence, and all follow-up outcomes
in the close note. No later commit or CI event is a prerequisite.

1. Run `sase bead epic-symbols bob-cli-q5.2.1`. It was empty during landing review;
   recheck. Resolve each entry by wiring, privatizing, deleting, or a justified non-test
   pragma; re-key only if a named still-open later bead needs it. Never leave entries
   keyed to a bead about to close.
2. Close normally with
   `sase bead close bob-cli-q5.2.1 --note "<verification and complete follow-up dispositions>"`.
   If symbols reject the close, finish their cleanup and retry. If named phases are
   incomplete, finish or reopen them. Never force a successful nested landing or force
   merely to make the command succeed; canceled/superseded work requires a deliberate
   reason and that resolution.
3. Run `just symvision` if available (there is no recipe at review baseline). Open the
   plans repository using `/sase_repo`; use its printed path only. Set `status: done` in
   the frontmatter of `202611/snoozed_planned_today.md` (the epic's PLAN path),
   preserving all other metadata. Do not edit the containing outer snoozed_tasks plan's
   status.
4. Run `sase bead read bob-cli-q5.2.1 -r "Need the parent link"`. Its direct parent is
   phase `bob-cli-q5.2`. Re-read that phase and verify this completed child plan
   satisfies its entire planned-today scope. Run `sase bead epic-symbols bob-cli-q5.2`
   and resolve/re-key any entries, then close only that phase normally with
   `sase bead close bob-cli-q5.2 --note "<verified direct recovery, byte preservation, exclusions, forecast, schema-1 agenda, and integrated tests>"`.
   Leave containing epic `bob-cli-q5` open for its already-waiting land agent. If new
   evidence makes that parent incomplete or ambiguous, leave it open and append a
   concrete blocker note instead of claiming completion.
5. Finish through `/sase_final`, declaring every changed repository for commit. Report
   exactly what closed and the gate results. No source changes after the final
   declaration.
