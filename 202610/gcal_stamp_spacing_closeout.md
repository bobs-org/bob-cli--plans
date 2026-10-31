---
tier: tale
title: Repair Events stamp spacing and finish landing bob-cli-es
goal: Keep the Events stamp separate from prose without accumulating blank lines,
  verify the repair, and close bob-cli-es.
size: small
proposed_by: bbugyi200.apollo.bob-cli-es.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-es
status: done
---

- **PARENT:**
  [202610/gcal_events_sync_stamp.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/gcal_events_sync_stamp.md)
- **BEAD:**
  [bob-cli-es](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-es/README.md)

# Repair Events stamp spacing and finish landing bob-cli-es

## Outcome

Finish the two remaining stamp-spacing defects in epic `bob-cli-es`, verify the result,
and close the epic in this same coding turn. One coding agent can do this bounded Rust
repair directly; no further implementation phases or land agent are needed. Do not wait
for this work's commit SHA, push, or CI: the host commits only after the coding turn
ends.

The existing approved decisions are final: `stamp_refresh=on_change`,
`account_guard=yes`, and `decision_record=no`. Preserve the account guard, the
last-change meaning, and the plugin's always-fresh caption. Do not edit memory,
`CHANGELOG.md`, or the Cargo package version, and add no options or configuration.

## Audit already completed

The lander read `bob-cli-es`, both children (`bob-cli-es.1`, `bob-cli-es.2`), every
child note, and `plan:202610/gcal_events_sync_stamp.md`. The implementation commits are
bob-cli `4f29a824` and bob-plugins `4e31220`. The approved behavior is implemented apart
from the spacing gaps below. Rust source and tests cover canonical rendering, escaped
accounts, same-day and dated times, structural refreshes, no idle churn, account refusal
before reconciliation and on retry, additive JSON, human reporting, and MEET line
offsets. Plugin source paints both editor and Reading view, scopes the caption to
Events, and disables stale state under `on_change`.

Concurrent and later changes reviewed: CLI `4a95e5b7` and `e08f859f`; plugins `a8f33a2`,
`458907b`, `38da67d`, `c391a0d`, and `4bb8821`. These change capture type markers, task
wikilinks, Command Mode sequences, and task-type ink. They leave the plain Events
caption independent; the sync decoder accepts the additive `stamp` field. No integration
patch was needed for those commits. The plugin checkout was fast-forwarded to `4bb8821`,
where `npm run build:check` and all eight Calendar Events tests passed. Recheck only
drift since these reviewed commits.

The lander finished all `PROPOSED FOLLOW-UP:` triage through `/sase_new_task` and
recorded the outcomes on `bob-cli-es`:

- `bob-cli-es.2` note #1, the two startup harness failures: independently reproduced and
  corroborated existing task `bob-cli-eh` with a +1. The failures are the note-ready
  startup-storm scan count `1 !== 2` and the relation-index layout-ready timeout. They
  predate the stamp work.
- The same note's distinct task-wikilink 12 ms ranking failure: reproduced at 13.1 ms,
  then the five-test performance file passed unchanged in isolation. Routed as a
  `DISCOVERED ISSUE:` note to active causal epic `bob-cli-eq`, whose phase `.3`
  introduced that node. No new task was created. Reopening `bob-cli-3w` was declined
  because that closed task covers the different, repaired dependencyStageRank 16 ms
  node.
- Evidence: `file:explicit:78360ee1c9c0ed50f7648b3a` (focused plugin results) and
  `file:explicit:ce587d56b66df46f0b04012a` (unchanged isolated ranker pass).

Do not repeat that triage or let those unrelated failures expand this repair.
`sase bead epic-symbols bob-cli-es` was empty, no `symvision` recipe was available, and
the epic had no `parent_bead` at audit time. Recheck these at closeout.

## Remaining defects and repair

Read the current `src/native/gcal/stamp.rs`, `src/native/gcal/plan.rs`, and the existing
stamp tests before editing. Confirm both cases with regression tests against the current
code, then fix them.

1. **Repair a missing separator even on an idle run.**

   In `stamp::is_due`, a line can match its canonical rendering and slot even when the
   very next line is ordinary prose. It returns false without checking that boundary.
   With an otherwise unchanged note, `plan_stamp` then preserves:

   ```markdown
   ## Events

   _↻ synced 12:00 ·
   [you@example.com](https://calendar.google.com/calendar/r/day/2026/10/30?authuser=you%40example.com)_
   Personal notes for the day.

   ## Pomodoros
   ```

   Those two lines render as one paragraph, allowing caption styling to include the
   user's prose. The approved canonical-slot contract requires a blank line between the
   stamp and a following paragraph. Detect that missing boundary as structural repair,
   preserve the prose, and refresh the stamp with the current minute. Keep the existing
   no-extra-gap behavior for event list rows and headings.

2. **Do not move the separator into the heading gap on refresh.**

   `plan_stamp` currently deletes recognized stamps before calling `canonical_slot`.
   When a stamp already has a blank line below it, that blank becomes contiguous with
   the heading gap. The recalculated slot skips it. Above prose, `need_blank_after` then
   creates another blank below the new stamp, growing the heading gap by one on every
   refresh.

   Preserve the insertion anchor and heading-gap boundary while removing old stamps, so
   a normal refresh replaces the stamp in place and reuses its existing separator.
   Preserve the user's existing blank lines rather than treating a below-stamp separator
   as part of the heading gap. Continue normalizing missing, malformed, duplicate, and
   misplaced stamps to one first section line. Keep the MEET entry's 1-based reported
   line correct for insertions and removals, including a hand-written Events section
   below Pomodoros.

Keep this repair focused on structural spacing. Age alone must never rewrite a valid
stamp, and the plugin needs no behavior change.

## Verification

Extend `src/native/gcal/plan/tests.rs` (and focused `stamp.rs` tests as useful):

- A canonical 12:00 stamp directly followed by prose is repaired at 12:20, with exactly
  one separating blank and unchanged prose. A subsequent idle plan at a later time is
  byte-identical and reports `stamp_refreshed=false`.
- A stamp above prose that already has a separator, with one and with multiple heading
  blanks, keeps the same layout across repeated forced structural refreshes. Use
  future/edited timestamps or actual planner changes to trigger refreshes; do not make
  age a refresh trigger.
- A stamp with a blank below it before a list preserves that boundary on refresh.
- Duplicate/misplaced normalization and MEET offsets still hold above and below
  Pomodoros. Extend existing tests rather than duplicating unrelated coverage.

Add an isolated fake-gog regression in `tests/gcal_sync.rs` for the missing paragraph
gap: a dry run proposes the same repair without writing; a real run reports the
refreshed stamp; the next idle run is unchanged. Assert that the journal does not gain
records for a stamp-only repair.

Run focused gcal tests, then file-change verification `just check` (through
`sase tool run check` when available). Follow `/sase_monitor` if the tool escalates and
ensure its continuation finishes this closeout. Do not run `just check-full`. If later
integration requires plugin edits, use `/sase_repo`, follow its `AGENTS.md`, rebuild
generated code, run relevant tests, and deploy the changed plugin with
`bob plugins sync`; otherwise the already-passing plugin checks above are sufficient.
Check for new base-branch drift since the audited SHAs before closing and address any
actual conflict with the stamp.

## Final step: close bob-cli-es in this coding turn

After the repair and verification, complete all of the following without waiting for
this turn's commit or a future lander:

1. Re-read `sase bead read bob-cli-es -r "Need final readiness and parent link"` and its
   children, confirming both original phases remain done and no new note or descendant
   introduces unfinished work. Read the linked plan with
   `sase artifact read plan:202610/gcal_events_sync_stamp.md "Confirm final landing scope"`.
2. Run `sase bead epic-symbols bob-cli-es`. Resolve every listed exemption by wiring,
   privatizing, adding a justified non-test pragma, or deleting it. Re-key only when an
   identified still-open later bead genuinely needs it. Do not leave an entry keyed to
   this epic or its completed phases.
3. Close normally with a concrete verification note, for example:

   ```sh
   sase bead close bob-cli-es --note "Verified both phase notes and implementation commits 4f29a824/4e31220 against on_change, account_guard=yes, and no memory edits. Audited concurrent capture, task-wikilink, Command Mode, and task-ink changes through plugin 4bb8821 plus subsequent drift. Repaired missing paragraph separation and separator migration during stamp refresh; focused gcal tests and just check passed. Plugin build check and eight Calendar tests passed. es.2 follow-up outcomes: existing bob-cli-eh corroborated for both startup harness failures; the distinct 12 ms ranking flake routed to causal active epic bob-cli-eq; bob-cli-3w reopening declined because it covers another fixed node. All epic-symbol entries resolved."
   ```

   Adjust the note to the actual checks and any newer drift. If close refuses for
   leftover symbols, resolve them and retry. If it names unfinished phases, finish or
   reopen them. Never use `--force` merely to succeed or to advance a successful nested
   landing; cancellation/supersession requires a deliberate non-done resolution with a
   reason.

4. After closing, run `just symvision` if the recipe is available and require it to
   pass. Otherwise record that this checkout has no such recipe.
5. Open the plans repo using `/sase_repo` and set `status: done` in the frontmatter of
   `202610/gcal_events_sync_stamp.md` under the path printed by
   `sase repo open plans -r "Mark the completed bob-cli-es plan done"`. Preserve all
   other metadata and content. This is the original epic's linked plan, not merely this
   repair tale.
6. Run `sase bead read bob-cli-es -r "Need the parent link"` after close. There was no
   parent at audit, so finish normally if still absent. If a parent has appeared, follow
   the user's parent rules: for a phase, verify and close only that phase; for a plan,
   re-audit its landing note, all descendants and linked plans, and post-child drift
   before normal closure, symbol cleanup, symvision, and marking its plan done. Repeat
   only through fully complete directly parented plans; record and report the first
   incomplete or ambiguous parent's blocker.

Finish with `/sase_final` as required. Include the plans-repo status change in the final
declaration, and report the epic's actual close state and checks.
