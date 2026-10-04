---
tier: epic
title: PRE and POST checklist tiers around the ]s morning walk, with the freshness
  trial removed
goal: 'The ]s walk opens with every open, actionable-today #gtd #pre task (the gtd_daily.md
  chores) and closes with the #gtd #post Morning review. Bryan resolves each of those
  rows by completing it in the walk, and checking Morning review last certifies the
  review. No doc, vault note, decision record, or bead waits on a freshness trial
  any more.

  '
phases:
- id: prep
  title: Remove the freshness trial and tag the gtd_daily.md chores
  depends_on: []
  size: small
  description: 'prep: delete every trial gate, window, and keep rule from docs/freshness.md,
    rotten.md, the decision records (edited inline), and bead bob-cli-3h, then add
    the inert #gtd #pre / #gtd #post tags to the eight live gtd_daily.md chores.'
- id: cycler
  title: task-status-cycler completion API v2
  depends_on: []
  size: small
  description: 'cycler: add completeTaskAtCursor(editor) to the cycler''s cross-plugin
    API (v2). It closes the task through the Tasks command so recurrence fires, never
    stamps, refuses rather than writing [x] raw, and reports lineDelta.'
- id: rust
  title: Checklist tier contract and the Rust evaluator (schema 9)
  depends_on:
  - prep
  size: medium
  description: 'rust: land the checklist contract and CL vectors in docs/freshness.md,
    implement Tier::Pre/Post, checklist scope, [?] admission, counts, lints, and schema
    9 in bob freshness, and amend the review-walk decision and the freshness glossary
    strand inline.'
- id: ledger
  title: bob-ledger-tools checklist tiers (freshness namespace v7)
  depends_on:
  - rust
  size: medium
  description: 'ledger: mirror the checklist contract in the ledger-tools fragments
    (evaluate, row adapter, queue, counts, footer, status view, marks), publish checklistTiers
    on freshness namespace v7, and port the CL vectors.'
- id: nav
  title: Navigation walk support, complete-and-advance, and text-first cursor identity
  depends_on:
  - ledger
  - cycler
  size: medium
  description: 'nav: teach navigation-hotkeys the PRE/POST tiers and notices, route
    Alt+F / Alt+Shift+F on checklist rows to cycler completion, skip them in counted
    and Task Link batches, match the cursor by text first, and drop a previous-day
    walk anchor.'
- id: rollout
  title: Ritual rewrite, deploy, and live verification
  depends_on:
  - rust
  - ledger
  - cycler
  - nav
  size: small
  description: 'rollout: rewrite the Morning review chore as a closeout, update the
    ritual docs and rollout log, install bob, confirm the synced plugins, verify the
    live walk, and record any GUI checks that cannot run as a verification gate.'
proposed_by: bbugyi200.apollo.research.07.linker.w1
create_time: 2026-10-04 09:05:21
status: done
bead_id: bob-cli-48
---

- **PROMPT:** [prompts/202610/gtd_pre_post_review_tiers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/gtd_pre_post_review_tiers.md)
- **BEAD:** [bob-cli-48](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-48/README.md)

# Plan: PRE and POST checklist tiers around the `]s` morning walk

## Request, decisions, and what changed from the literal ask

Bryan wants two new groups in the GTD morning walk that `]s` drives in Obsidian:

- **PRE** is reviewed before every other group. It holds ready tasks tagged `#gtd` and
  `#pre`. Every daily-recurring chore in `~/bob/gtd_daily.md` except "Morning review"
  gets those tags.
- **POST** is reviewed after every other group. It holds ready tasks tagged `#gtd` and
  `#post`. "Morning review" gets those tags.

Bryan closes each recurring GTD chore as the walk reaches it. "Morning review" comes
last, so checking it certifies the whole review, except for any ROTTEN upkeep he could
not reach that day.

Bryan read the consolidated research report and agrees with almost all of its
recommendations. Read it with:

```bash
sase artifact read file:explicit:715e310d22ca404995bfa4a6 "PRE/POST checklist tier design"
```

This epic implements its R1–R8 adjustments and its recommended solution, with these
changes on Bryan's instructions:

1. **There is no trial.** Bryan rejected the report's §13 "don't change the ritual
   mid-trial" timing and its "ship now or hold until 10-19" question. Nothing waits for
   2026-10-19. The prep phase removes the trial from the docs, the vault, the decision
   records, and the one bead snoozed on it. After that, no plan, doc, or memory should
   suggest waiting for a trial again.
2. **No new decision files.** Bryan allowed inline edits to existing decision records
   for this change and forbade creating new decision memory files. So the report's R8
   successor record (`walk-is-framed-by-checklists`) becomes an inline amendment of
   `decisions/review-walk-is-tiered`. Trial references are deleted inline from the
   records that carry them. This overrides the decisions web's "immutable once accepted"
   rule for this change only. Each amended record says so in one line.

These report adjustments are adopted as written. Each is restated in the phase that owns
it.

- **R1, wider "ready".** A member is any open, actionable-today tagged task: status ` `,
  `*`, `/`, or `?`. The 15-minute hooks leave each new occurrence `[?]` until they run,
  and a strict `[ ]` rule would hide the chores exactly when they are due. For these
  rows only, the recurring, canonical-daily-note, and Today exclusions do not apply.
- **R2, membership is tags only.** "Recurs daily" is only the rule for the one-time
  vault tagging. A one-off or weekly task can join PRE or POST by tag, with no code
  change.
- **R3, exact tags.** Tags match exactly, as whole tokens, case-insensitively. Lints
  flag conflicting or incomplete tags, and a recurring member whose rule lacks
  `when done`.
- **R4, order.** PRE → NEW → PROJECTS → PENDING → NEXT → RETURNED → REFERENCES → ROTTEN
  → POST. PRE counts as a commitment. POST is a closing tier after ROTTEN, and `]S`
  reaches it in one key.
- **R5, completion only.** A checklist row is resolved only by completing it.
  - Alt+Shift+F completes the row and moves to the next one.
  - Alt+F completes it in place.
  - `]s` skips it.
  - Checklist rows never stamp and never touch keeps, decay, `upkeep_today`, or the
    budget.
- **R6, eight lines.** Tag only the eight active lines: seven chores get `#gtd #pre` and
  Morning review gets `#gtd #post`. The cancelled "Pick today" and "Weekly review"
  leftovers and the weekly "Weekly prune" stay untagged.
- **R7, rewrite Morning review.** Its text becomes a closeout checklist at deploy.
- **The F9 fix.** The cursor is matched by its text first, so a recurrence insert above
  the cursor cannot make `]s` skip a chore.

**Out of scope:**

- Dash sections or chips for PRE/POST.
- Changing Ctrl+Enter's `[?]` refusal.
- The optional chore edits: merging teeth, pills, and stretches; time-boxing email;
  moving the Keep import to the last line. Bryan can ask for these separately.
- Deleting the cancelled leftovers.
- The daily-note `^gtd` wrapper's meaning.
- Configurable tag lists, task-status-hooks, capture grammar, Bob Mac Capture, and any
  stored tier field.

## Context verified during planning (2026-10-04)

**bob-cli at `660c171`:**

- `docs/freshness.md` is the contract both evaluators cite:
  - §4: `walk_scope` at about lines 340–342; `tier(t)` at 363–372; the order paragraph
    and commitment sentence at 375–396; counts at 398–418; machine vocabulary and schema
    history at 523–561; lints at 577–586.
  - §6: the ritual.
  - §7: CLI/JSON, schema 8.
  - §§9–10a: vectors.
  - §13: "Two-week trial (2026-10-05 through 2026-10-18)".
  - §14: keep-streak rollout. It mentions the §13 trial once.
- `src/native/freshness/state.rs`:
  - `Tier` (72–95) derives `Ord`, so its declaration order is the walk order.
  - `FreshnessRow` (179–197) has no tags field.
  - In `evaluate`, `walk_scope` is at 308–312 and the tier chain at 435–470. Lane rows
    return early at 481–530. `match state { None => … tier: None }` at 532–545 nulls the
    tier of every out-of-scope row.
  - `queue` (703–770) drops rows whose `tier` or `lane` is None. `QueueEntry.lane` is a
    non-optional `Lane`.
  - `ByTier` is at 774–795 and `Counts` at 798–832.
- `scan.rs`:
  - The lane queries (`READY_QUERY`, `PENDING_QUERY`, `NEXT_QUERY`) exclude `[?]`,
    blocked, `#hide`, `_templates`, `_conflicts`, and future-scheduled tasks.
  - `^ref` tracker candidates come from `snapshot.all` through
    `tracker_candidate_visible` (316–342) into `Snapshot.trackers`.
  - `cli.rs` chains them into the queue input and the counts input at 345–357 and
    422–429.
- `RichTask.tags` (`src/native/dataview/tasks/mod.rs:105-123`) preserves case and keeps
  subtags whole. The `#task` global filter is dropped.
- `cli.rs`:
  - `SCHEMA_VERSION = 8` at line 54.
  - The human tiers array has 7 entries (556–564).
  - The "── commitments done above · upkeep below ──" divider is printed before ROTTEN.
- Stamping refuses recurring lines (`placement.rs` `stamp_inner`). `decide_for` fires
  only for Ready-lane ROTTEN/RETURNED rows.
- Tests:
  - `state_tests.rs`: today is 2026-10-08. There are full `FreshnessRow` literals at 70,
    344, and 464, plus one in `scan.rs:54`.
  - `tests/cli/freshness.rs` asserts `schema_version == 8` in eight places.
  - `just all` runs fmt, clippy, and test.
  - `just install` installs `bob`.
- Other docs that state the walk tier order:
  - `README.md` about lines 592–600
  - `docs/getting-started.md` 165–166
  - `docs/plan.md` 142–147
  - `docs/highlights-ref-sync.md` 727–729
  - `docs/projects.md` 96–100
- `sase/memory/glossary/task-freshness.md` still lists the five-tier order.

**bob-plugins at `5680659`.** Open it with `sase repo open bob-plugins -r "<reason>"`,
read its `AGENTS.md`, and use only the printed path.

- **bob-ledger-tools 1.28.1** is built from fragments. Edit `src/`, run `npm run build`,
  never hand-edit `main.js`, and keep each fragment at or below 1000 lines.
  - `src/100-freshness-evaluate.js`:
    - `freshnessLaneForRow` is at 32–43.
    - In `freshnessEvaluate` (210–522), `walkScope` is at 277–282 and the tier chain at
      366–386. Lane rows return early at 398–437. The null-state early return that nulls
      the tier is at 439–453.
    - `FRESHNESS_TIER_ORDER` is at 574–582 and `freshnessTierLabel` at 598.
    - A missing status symbol falls back to `"?"` at 235–236.
  - `src/160-widgets-and-row.js` `freshnessRowFromTask` (269–492) copies no tags. Its
    `laneVisible` is `planLaneVisible` (`060:343-368`), which excludes done tasks,
    blocked tasks, `#hide` (case-insensitive substring), `_templates`/`_conflicts`, and
    future-scheduled tasks.
  - `src/110-freshness-queue.js`:
    - The queue skips null-tier rows (19–21) and null-lane rows (22–24).
    - The per-tier sort is at 57–92. Unknown tiers fall to the ROTTEN comparator.
    - `freshnessCounts` (127–239) has seven `byTier` keys.
    - `freshnessStatusView` (284–401) hard-codes seven tiers.
  - `src/120-freshness-footer.js`:
    - `FRESHNESS_FOOTER_TIERS` and `FRESHNESS_FOOTER_COMMITMENT_TIERS` are at 1–18.
    - The `freshnessReviewMachineTier` whitelist is at 33–63.
    - `freshnessReviewEntryView` is at 176–235.
  - `src/230-*.js` has fallback counts with seven hard-coded keys (798–823). Its READY
    gate is `bucket !== null`.
  - `src/130-*.js` marks (440–447).
  - `src/170-plugin-lifecycle.js`: `api.version` 3 and `freshness` namespace
    `{ version: 6, trackerReview: true, referenceReview: true, … }` (228–253).
- **bob-navigation-hotkeys 2.2.0** is a hand-edited `main.js` with no `src/`.
  - Tier whitelists:
    - `reviewEntryMachineTier` (~33473)
    - `reviewIsCommitmentTier` (~33523)
    - `reviewWalkRemaining` (~34011), which returns `{commitments, rotten}`
    - `buildReviewBoundaryNotice` (~34039), which fires only into `rotten`
    - the `buildReviewJumpNotice` fallback (~34485)
  - Walk functions:
    - `buildReviewAnchor` (~33973) has no day field.
    - `planReviewJump` (~34192) matches the cursor by line before text (~34235–34255).
    - `resolveReviewQueueLine` (~34390) relocates a landing by unique exact text.
    - `jumpToDueTask` is at ~37537 and `refreshTaskFreshness` at ~37647.
  - Alt+F paths:
    - Single and counted tasks: `refreshTaskFreshnessOnTasks` (~37794).
    - Task Link batches: `refreshTaskFreshnessOnLinks` (~38027).
    - Recurring refusal: `classifyFreshStampTarget` (~34596).
    - Exact queue match: `matchFreshStampExactEntry` (~34972).
  - Ledger gates:
    - `getReviewFreshnessApi` (~33428)
    - `reviewFreshnessSupportsTiers` (v4)
    - `reviewFreshnessSupportsTrackers`
  - `]s` / `[s` / `[S` / `]S` are mapped in the vault's `obsidian_vimrc.md` to
    `jump-to-{next,prev,first,last}-due-task`.
- **task-status-cycler 1.23.1** is fragment-built.
  - `OPEN_DONE_TASK_SYMBOLS` (`010-core.js:48`) excludes `?`.
  - Ctrl+Enter goes through `toggleActiveCheckboxOpenDoneAndPropagate`
    (`160-plugin-completion.js:124-159`) and `setActiveCheckboxStatus`
    (`180-plugin-editor-edits.js:352-404`). That path runs the Tasks command
    `obsidian-tasks-plugin:set-status-symbol-to-x`, then stamps freshness only when the
    line count is unchanged.
  - `finalizeClosedTasks` is at `130-plugin-references.js:52-86`.
  - The API is `Object.freeze({ version: 1, recoverBlockedDependents })`
    (`110-plugin-lifecycle.js:10-16`).
  - Nav consumes it with `Number(api.version) >= 1`.
  - A recurrence-insert-above mock exists at
    `scripts/test-task-status-cycler.cjs:4333-4370`.
- `npm test` runs a build check, then `node --test` over an explicit file list. A new
  test file must be added to that list.
- After changing bob-plugins, run `bob plugins sync` (repo rule).

**Vault (`gh:bobs-org/bob`, live at `~/bob`):**

- `gtd_daily.md` has seven open `[ ]` daily chores, all `#task` only and
  `[repeat:: every day when done]`:
  - Check weather
  - Brush teeth
  - Take fish oil pills + Take L-tyrosine pills
  - Review Calendar for today and tomorrow + Create @EVENT notes
  - Read Email
  - Import inbox tasks from Google Keep
  - Do morning stretches!
- It also has:
  - two cancelled `[-]` leftovers: "Pick today" (daily) and "Weekly review" (weekly);
  - the open daily "Morning review" chore;
  - the weekly `[?]` "Weekly prune" chore.
- No line has a block ID.
- `rotten.md` ends with a `## Trial tally (2026-10-05 through 2026-10-18)` section: an
  intro paragraph, a 2026-10-01 CROWDED bullet, and an empty tally table.
- `#pre` and `#post` are unused in the vault, and no open task carries `#gtd`, so the
  tags are inert until the code ships.

**Trial references found:**

- `docs/freshness.md` §13 (whole section) and §14 (one sentence).
- `~/bob/rotten.md` (the tally section).
- Decision records:
  - `review-walk-is-tiered` (Reopens)
  - `ready-is-freshness-gated` (one rejected alternative and Reopens)
  - `task-lanes-are-sticky` (one rejected alternative and Reopens)
  - `note-ready-cap-counts-the-lane` (Reopens)
- Bead `bob-cli-3h`, snoozed until 2026-10-19 with the reason "Freshness trial runs
  through 2026-10-18; ship notices after it and after the sase.md triage".
- No Rust or plugin source code reads a trial date any more: `fc438bc` and bob-plugins
  `f4b3562` already removed the decay date.
- Left alone on purpose:
  - `decisions/now-tag-is-user-owned` names the separate, already-failed `#now` trial
    and is superseded.
  - `rotten-keeps-use-priority-decay` and `decay-decisions-are-available-immediately`
    already supersede the decay date.
  - The nav test name at `scripts/test-navigation-task-card-view.cjs:1680` only asserts
    that no date gates the card.

## Shared checklist contract

Both evaluators implement exactly this. `today` is the vault's local date (`BOB_NOW` in
tests). Text in the same style lands in `docs/freshness.md` §4.

```text
tags'(t)           = t's tags, lowercased, compared as whole tokens (#gtd/pre ≠ #gtd, #pressed_juice ≠ #pre)
checklist(t)       = pre   if #gtd ∈ tags' ∧ #pre ∈ tags'
                   | post  if #gtd ∈ tags' ∧ #post ∈ tags' ∧ #pre ∉ tags'
                   | none
checklist_scope(t) = checklist(t) ≠ none
                     ∧ status symbol (as written on the line) ∈ { ' ', '*', '/', '?' }
                     ∧ ¬dependency-blocked ∧ ¬(scheduled > today)
                     ∧ ¬hidden (the existing #hide predicate, case-insensitive substring, unchanged)
                     ∧ path has no _templates / _conflicts segment (existing predicate)
                     (recurring, canonical daily note, and Today-linked are NOT exclusions here)
tier(t)            = checklist(t)             if checklist_scope(t)   — beats every other tier, incl. ^prj/^ref and lanes
                   | existing tier chain      otherwise
order              = pre · new · projects · pending · next · returned · references · rotten · post
within pre / post  = path ↑, line ↑   (file order is the checklist order)
lane               = existing lane(t), or null (e.g. a [?] row); a null lane never drops a checklist row
due                = always, while in checklist scope; due_on = null, days_overdue = null
interval fields    = whatever the existing interval resolution reports (informational only)
state, bucket      = existing evaluation, unchanged (recurring / [?] / lane rows → null; a one-off
                     Ready member keeps its own state and bucket, e.g. NEW)
decide             = false (decide_for already requires Ready lane + ROTTEN/RETURNED)
counts             = by_tier += pre, post; pre_due = by_tier.pre; post_due = by_tier.post;
                     walk = Σ by_tier; due, new, resurfaced, rotten, fresh, decide,
                     refreshed_today, upkeep_today, budget, budget_met formulas unchanged
commitment tiers   = pre, new, projects, pending, next, returned, references   (post is a closing tier)
resolution         = completion through Tasks only; checklist rows are never stamped by the walk
```

**Traps.** Both must be handled in each evaluator.

- **The null-state early return** (Rust `state.rs:532-545`, JS `100:439-453`) forces
  `tier: null`. Lane rows also return early, before it. Compute the checklist branch
  first and carry its tier through every return path. Otherwise every recurring member
  silently loses its tier.
- **The queue drops null-lane rows** (Rust `queue`, JS `110:22-24`). Exempt checklist
  tiers. In Rust, `QueueEntry.lane` becomes `Option<Lane>`.
- **JS falls back to `"?"`** when a row has no status symbol (`100:235-236`). The
  checklist status test must read the real symbol on the line, so a symbol-less row
  never counts as `[?]`.

**Conformance vectors CL1–CL12.** Today is 2026-10-08. The prefix is `CL`, because
`C1–C13` already name config vectors.

- **CL1.**
  `- [ ] #task #gtd #pre Brush teeth [repeat:: every day when done] [scheduled:: 2026-10-01]`
  → tier `pre`, lane `ready`, state and bucket null, due, `due_on` null.
- **CL2.**
  `- [?] #task #gtd #pre Check weather [repeat:: every day when done] [scheduled:: 2026-10-08]`
  → tier `pre`, lane null, in the queue.
- **CL3.** Each of these has no tier and is absent from the queue: CL2 with an open
  dependency; CL1 with `[scheduled:: 2026-10-09]`; CL1 plus `#hide`; CL1 under
  `_templates/`.
- **CL4.** CL1 linked under today's open Pomodoro (Today), and CL1 in canonical daily
  note `2026/20261008.md` → both still `pre`.
- **CL5.**
  - `#pre` without `#gtd` → none, lint `checklist_tag_incomplete`.
  - `#gtd #pressed_juice` → none, no lint.
  - `#gtd/pre` → none, no lint.
  - `#GTD #Pre` → `pre`.
- **CL6.** `#gtd #pre #post` → `pre`, lint `checklist_tag_conflict`.
- **CL7.** One-off, unstamped `- [ ] #task #gtd #post Write retro` → tier `post`, state
  `new`, bucket `new`. It counts in `new` and `post_due`.
- **CL8.** One row per tier → queue order pre, new, projects, pending, next, returned,
  references, rotten, post. `walk = Σ by_tier`, and `due`/`new`/`rotten` equal the same
  vault without the checklist rows (except a CL7-style one-off member's own state).
- **CL9.** A recurring checklist row refuses a stamp (Rust `stamp`, JS
  `stampLine`/`keepLine`). `keeps`, `upkeep_today`, and `refreshed_today` are unchanged,
  and `decide` is false.
- **CL10.** After completion with the next occurrence inserted above
  (`- [ ] … [scheduled:: 2026-10-09]`, then `- [x] … [completion:: 2026-10-08]`),
  neither line is in the queue.
- **CL11.** `[*]` with `#gtd #pre` → `pre` (lane `next`). `#gtd #post … ^ref` → `post`.
  Checklist beats lane tiers and trackers.
- **CL12.** A `#gtd #pre` row with `[repeat:: every day]` and no `when done` → `pre`,
  lint `checklist_repeat_not_when_done`.

Lints are Rust-only (`bob freshness list` LINTS). They apply only to open tasks, not to
done or cancelled lines in archives.

## Phase prep

Remove the trial and tag the vault. Read the decision records with `/sase_memory_read`
and use `/sase_memory_write` before editing memory. Plan approval authorizes these
memory edits.

1. **`docs/freshness.md` §13.** Retitle it `## 13. Rollout log` so section numbers and
   existing `§13` citations stay stable. Replace the whole body with:
   - one sentence: "There is no freshness trial: no ritual change, release, or tuning
     waits on a trial window or keep rule; tune intervals or the budget whenever the
     walk needs it.";
   - the two dated entries, reworded without the trial:
     - "2026-10-01: per-note CROWDED surfaces (`bob ready`, the dash chip, crowded.md,
       the heading chips) went live; gesture notices are a separate follow-up."
     - "2026-10-01: the tiered morning review walk went live (schema 3, lane intervals,
       tier notices, walk anchor)."

   Delete the tally instructions and the keep rule.

2. **§14.** Delete the sentence "The accepted Ready trial in §13 ran 2026-10-05 through
   2026-10-18 independently of this decay gate."
3. **Vault.**
   1. Run `sase repo open gh:bobs-org/bob -r "<reason>"`, then check `git status` and
      preserve unrelated work.
   2. In `rotten.md`, delete the whole `## Trial tally (2026-10-05 through 2026-10-18)`
      section: heading, paragraph, bullet, and table.
   3. In `gtd_daily.md`, insert `#gtd #pre` right after `#task` on the seven open daily
      chores, for example `- [ ] #task #gtd #pre Check weather  [repeat:: …`. Insert
      `#gtd #post` right after `#task` on "Morning review". Change nothing else on any
      line: the text, fields, spacing, and order stay as they are. Leave the two `[-]`
      leftovers and "Weekly prune" untagged.
   4. Commit only these two files. Get them to `~/bob` through the documented
      `bob vault-sync` path, then verify the destination content.
   5. Confirm the tags are inert: `bob freshness list -f json` counts match a snapshot
      taken before the edit.
4. **Decision records.** Edit these inline. Each edited record gets one line at the end
   of its body: "Amended in place 2026-10-04 at Bryan's request: the freshness trial was
   removed; nothing waits on it."
   - `decisions/review-walk-is-tiered`, Reopens: delete ", or the 2026-10-05 through
     2026-10-18 trial fails under its keep rule". It ends "…, or long-interval tasks
     starve past the weekly prune."
   - `decisions/ready-is-freshness-gated`:
     - In the dim-not-hide rejected alternative, replace "dimming is reconsidered only
       if the trial fails, after interval/budget tuning" with "dimming is reconsidered
       only if gating hides needed work after interval/budget tuning".
     - Reopens becomes: "A required plugin-free surface needs gating, or gating hides
       needed work (a lost-needed-task case) after interval/budget tuning."
   - `decisions/task-lanes-are-sticky`:
     - Replace "**Age-based Next decay.** Only if the trial fails, never for Pending."
       with "**Age-based Next decay.** Not adopted, and never for Pending."
     - Reopens becomes: "Lanes must survive Blocked (then Blocked becomes an overlay)."
   - `decisions/note-ready-cap-counts-the-lane`, Reopens becomes: "Bryan finds the cap
     unhelpful after tuning N and the notice scope."

   Leave `now-tag-is-user-owned`, `rotten-keeps-use-priority-decay`, and
   `decay-decisions-are-available-immediately` unchanged. Then run `sase memory init`.

5. **Bead `bob-cli-3h`.**
   1. Wake it: `sase bead snooze bob-cli-3h --cancel`.
   2. Append:
      `sase bead note bob-cli-3h "Bryan (2026-10-04): there is no freshness trial; nothing waits on it. The only remaining precondition from the original snooze is the sase.md triage session (vault task bob_gtd#^prj-task-count-warn)."`
   3. Don't change its description, size, or status otherwise.
6. **Verify.**
   - `grep -rniE "trial|2026-10-18|2026-10-19|10-19"` over bob-cli `docs/`, `README.md`,
     `src/`, `tests/`, and `sase/memory/`, plus vault `rotten.md` and `gtd_daily.md`,
     shows only: the deliberate §13 "There is no freshness trial" sentence, the
     untouched records named above, the amendment lines, and unrelated test dates.
   - `just all` passes.

## Phase cycler

Work in the opened bob-plugins checkout, in `plugins/task-status-cycler/src/` fragments.
Build with `npm run build`.

1. **`completeTaskAtCursor(editor)`.** Add it next to `recoverBlockedDependents`. Bump
   the API to `{ version: 2, recoverBlockedDependents, completeTaskAtCursor }`, and
   update the "bump on new method" comment. Behavior:
   - **Refusals.** Each returns `{ ok: false, reason }` and writes nothing:
     - the cursor line is not a `#task` checkbox line: `not-task`;
     - the status symbol is not ` `, `*`, `/`, or `?`: `not-open`;
     - the Tasks `set-status-symbol-to-x` command is unavailable:
       `tasks-command-missing`.

     Never fall back to writing `[x]` raw, because that would skip recurrence.

   - **Close.**
     1. Before writing, capture the closed task's identity (`closedTaskIdentity`) and
        the line count, as the Ctrl+Enter path already does.
     2. Run the Tasks command on the cursor line.
     3. Verify that the original line now reads `[x]`, at the cursor line or offset by
        the insert. If it doesn't: `{ ok: false, reason: "not-closed" }`.
   - **No stamp.** Never add a `[fresh::]` stamp, even when the line count is unchanged.
     Skip the post-close stamp the Ctrl+Enter path applies.
   - **Finalize.** Run `finalizeClosedTasks` for the captured identity, exactly as a
     Ctrl+Enter close does.
   - **Result.** Return `{ ok: true, lineDelta }`, where `lineDelta` is the line count
     after minus before (1 when Tasks inserts the next occurrence). The method may be
     async; callers await it.
   - **Reuse.** Factor the shared close internals rather than duplicating them.
     Ctrl+Enter behavior stays byte-for-byte unchanged, including its `[?]` refusal and
     its stamp.

2. **Tests** in `scripts/test-task-status-cycler.cjs`:
   - each starting symbol ` `, `*`, `/`, `?` closes through the Tasks command mock;
   - the insert-above mock yields `lineDelta: 1`, and no line gains `[fresh::]`;
   - `[x]` and `[-]` refuse;
   - a non-task line refuses;
   - a missing Tasks command refuses with no write;
   - `finalizeClosedTasks` runs once with the right identity;
   - the API object is frozen at version 2;
   - the existing Ctrl+Enter tests stay green.
3. **Ship.**
   - Bump the cycler `manifest.json` minor to 1.24.0.
   - Update the root `README.md` cycler API paragraph (~line 151).
   - Run `npm test` and `npm run validate`.
   - Commit, then run `bob plugins sync`.

## Phase rust

Work in bob-cli. Land the contract docs first, then the code.

1. **`docs/freshness.md`.**
   - **Intro and §4.**
     - Add the shared checklist contract above in the §4 pseudo-spec style.
     - Rewrite the `walk_scope` and `tier(t)` blocks so checklist scope comes first.
     - Update the order paragraph and per-tier table to nine tiers. PRE and POST sort by
       path, then line.
     - Add PRE to the commitment sentence and call POST the closing tier.
     - Add `by_tier.pre/post` and `pre_due`/`post_due` to the counts bullets.
     - Add the three lints to the lint list.
   - **§7.**
     - Schema 9 history entry: "9: PRE/POST checklist tiers: `pre`/`post` in `tier` and
       `by_tier`, `pre_due`/`post_due`, `lane` may be null on checklist rows."
     - Update the JSON tier enum and the `lane` type (`ready|pending|next|null`).
     - Update the human example with a PRE section first and a POST section last.
   - **§8.** Update the surfaces table's tier list.
   - **§10.** Add a "Checklist tiers (CL1–CL12)" vector block after the R/B vectors.
   - Leave §6 to the rollout phase.
   - Update the tier-order sentence in `README.md` "Task freshness", plus the schema
     number, `docs/getting-started.md`, `docs/plan.md`, `docs/highlights-ref-sync.md`,
     and `docs/projects.md`, wherever they state the full walk order.
2. **`state.rs`.**
   - Add `ChecklistKind { Pre, Post }`, `Tier::Pre` declared first and `Tier::Post`
     last, with `as_str` `"pre"`/`"post"`.
   - Add `FreshnessRow.checklist: Option<ChecklistKind>`, and update the four full
     literals.
   - In `evaluate`, compute checklist scope from the row's real status symbol,
     `lane_visible`, and `checklist`. When in scope, the tier is the checklist kind on
     every return path. State, bucket, keeps, and interval follow the existing logic.
     `due_on` and `days_overdue` are None, and `decide` is false.
   - `queue`: admit checklist tiers with a None lane, with
     `QueueEntry.lane: Option<Lane>`. The PRE/POST comparator is path, then line.
   - `ByTier` gains `pre` and `post`, and `sum()` includes them. `Counts` gains
     `pre_due` and `post_due`.
   - Leave `upkeep`, `refreshed_today`, `budget_met`, and `decide_for` unchanged.
3. **`scan.rs`.**
   - Build `checklist` in `freshness_row` from `RichTask.tags`, as a lowercase
     whole-token match.
   - Add `Snapshot.checklist` candidates from `snapshot.all`, modeled on the `^ref`
     tracker candidates. Admit open tasks whose `checklist` is Some and whose status is
     in the set, and which pass a `checklist_candidate_visible` predicate:
     - not `is_blocked`;
     - no tag containing `#hide` (case-insensitive, matching the engine's
       `tags do not include`);
     - no `_templates`/`_conflicts` in the path (as `tracker_candidate_visible`);
     - `scheduled` None or ≤ today.
   - Dedupe the candidates by `path:line` against the lane and tracker rows. Lane-query
     rows that are checklist members already carry `checklist`.
   - `is_daily_note` and `is_today` come from the existing context closure.
   - Emit the three lints from `collect_warnings` and `lint_message`, for open tasks
     only.
4. **`cli.rs`.**
   - `SCHEMA_VERSION = 9` plus a history comment line.
   - Long-about tier text.
   - Header:
     `REVIEW {walk} due · {pre} pre · {new} new · … · {rotten} rotten · {post} post · ✓…`.
   - The tiers array grows to 9. PRE prints first. The existing divider stays before
     ROTTEN, and a `── review closeout ──` divider precedes POST.
   - `human_row` arms:
     - `pre`: "Checklist · complete to resolve"
     - `post`: "Closeout · complete last"
   - JSON: the `by_tier` keys, `pre_due`/`post_due`, and a nullable `lane`.
   - Chain `snapshot.checklist` into both the queue input and the counts input, as
     `trackers` is chained.
5. **Tests.**
   - `state_tests.rs`: CL1–CL12, plus a nine-tier order test that extends
     `seven_tier_order_with_references`. Keep every existing S/Q/L/R/B/D and tracker
     test green.
   - `tests/cli/freshness.rs`:
     - Move the schema assertions to 9.
     - Add a `gtd_daily`-style fixture vault: past-scheduled `[ ]` chores, a `[?]` chore
       scheduled today, a future-scheduled `[?]` chore, a Today-linked chore, the
       `#gtd #post` Morning review, a `#pre`-only line, and a triple-tagged line.
     - Assert the JSON tiers, order, null lane, counts, and lints, and the human
       PRE-first / POST-last sections with the closeout divider.
   - `just all` passes.
6. **Memory (inline amendment, no new files).** Use `/sase_memory_write`.
   - **`decisions/review-walk-is-tiered`:**
     - **Summary:** "The ]s walk visits one shared queue in explicit tiers PRE → NEW →
       PROJECTS → PENDING → NEXT → RETURNED → REFERENCES → ROTTEN → POST; #gtd
       #pre/#post checklist rows resolve only by completion; Pending and Next come due
       daily under pending_interval / next_interval (default 1, false walks that lane
       off); tiers never feed buckets or chips; upkeep outside the lanes counts the
       budget; stamps stay and the seed never re-runs."
     - **Claim:** add a paragraph covering checklist membership by exact tags, the scope
       (recurring, daily-note, and Today rows are admitted; `[?]` is allowed),
       completion-only resolution, and PRE as a commitment with POST as the closing tier
       that `]S` reaches. Qualify "…Today-linked tasks are in no tier" with "except as
       PRE/POST checklist rows".
     - **Rejected alternatives:** add these:
       - a second keymap or gtd-only walk;
       - membership by file path, block ID, inline field, or nested tag;
       - lifting the recurring exclusion for every task;
       - dropping `repeat`;
       - POST before ROTTEN, to be reopened if `]S` is pressed at the boundary with zero
         ROTTEN done on most mornings;
       - auto-completing Morning review.
     - **Evidence:** the research ref above, this plan, and schema 9.
     - **Reopens:** add "PRE rows are skipped with `]s` on most mornings (then the
       habits leave `#task`)".
     - **Amendment line:** "Amended in place 2026-10-04 at Bryan's request: PRE/POST
       checklist tiers."
   - **`glossary:task-freshness`:** replace the five-tier walk sentence with the
     nine-tier order. Add that PRE/POST `#gtd` checklist rows (often recurring) resolve
     by completion, never by a stamp.
   - Run `sase memory init`.
7. Do not install `bob` in this phase; the rollout phase installs it from merged
   `master`.

## Phase ledger

Work in the opened bob-plugins checkout, in `plugins/bob-ledger-tools/src/`. The
contract is bob-cli `docs/freshness.md` as the rust phase landed it. Cite it; never
re-derive it.

1. **Row adapter (`160`).**
   - Add `checklist: "pre" | "post" | null`, from `task.tags` lowercased with exact
     token equality. Write a small helper; do not reuse the `#hide` substring helper.
   - Add the real status symbol from the line, never the `"?"` fallback.
2. **Evaluate (`100`).**
   - `FRESHNESS_TIER_ORDER` becomes
     `{pre:0, new:1, projects:2, pending:3, next:4, returned:5, references:6, rotten:7, post:8}`.
   - Add tier labels `PRE` and `POST`.
   - Add the checklist-scope branch with the tier carried through every return. Reuse
     `laneVisible` as the visibility half.
3. **Queue and counts (`110`).**
   - Exempt checklist tiers from the null-lane skip.
   - PRE/POST comparator: path, then line. They must not fall to the ROTTEN comparator.
   - `byTier.pre/post`, `preDue`/`postDue`, and `walk` summed over all nine.
   - Update `freshnessStatusView` text, tooltip, and mode for nine tiers.
   - Update the hard-coded fallback counts in `230`.
4. **Footer (`120`).**
   - Add `pre`/`post` to `FRESHNESS_FOOTER_TIERS` and the machine-tier whitelist.
   - Add `pre` to `FRESHNESS_FOOTER_COMMITMENT_TIERS`.
   - Render POST's group last.
   - `freshnessReviewEntryView`:
     - PRE: detail `checklist`, hint `Alt+Shift+F done → next · ]s skip`.
     - POST: detail `closeout`, hint `Alt+F done · closes the review`.
5. **Marks (`130`).** A PRE/POST row must never advertise a stamp (`Alt+F to confirm`).
   If the existing model would, show the tier hint instead. Otherwise leave the marks
   unchanged.
6. **Lifecycle (`170`).**
   - The `freshness` namespace goes to `version: 7` with `checklistTiers: true`. Update
     the explanatory comment.
   - `tier()`, `isDue()`, and `rank()` report checklist rows.
   - `stampLine`/`keepLine` keep refusing recurring lines (CL9).
7. **Tests.**
   - Port CL1–CL12 (minus the Rust-only lints) and the nine-tier order into
     `scripts/test-ledger-tools-freshness.cjs`.
   - Assert namespace v7 and `checklistTiers`.
   - In `test-ledger-tools-freshness-footer.cjs`, cover the PRE/POST groups, the
     commitments including PRE, and both entry views.
   - Add a mark test for a checklist row.
   - Keep the READY dashboard-parity test green: recurring members have a null bucket.
8. **Ship.**
   - `npm run build`, `npm test`, `npm run validate`. Every fragment stays at or below
     1000 lines.
   - Bump `manifest.json` to 1.29.0.
   - Update `README.md`: the ledger row, the namespace paragraph (~118–125), and the
     tier order.
   - Commit, then run `bob plugins sync`.

## Phase nav

Work in the opened bob-plugins checkout, in the hand-edited
`plugins/bob-navigation-hotkeys/main.js`.

1. **Gates.**
   - **Checklist support:** ledger `freshness.version >= 7` and
     `checklistTiers === true`.
   - **Completion:** additionally the cycler API `version >= 2` with a
     `completeTaskAtCursor` function.
   - **Without the gates:** PRE/POST rows still walk with the generic tier notice. Alt+F
     on one shows `Checklist rows close by completion — update task-status-cycler` (or
     the ledger, as applicable) and writes nothing.
2. **Tier recognition.**
   - Add `pre`/`post` to `reviewEntryMachineTier`, and `pre` to
     `reviewIsCommitmentTier`.
   - `reviewWalkRemaining` returns `{commitments, rotten, post}`.
   - Give the fallback notices PRE/POST text.
3. **Boundary notices.**
   - Into `rotten`: keep both existing lines, and append ` · ]S closes the review` when
     POST remains non-empty.
   - A forward step from a commitment tier straight into `post` (ROTTEN empty) shows the
     same `Commitments done — 0 ROTTEN left` line.
   - Landing on POST appends ` · {c} commitments due · {r} ROTTEN left` to its landing
     notice, for example
     `POST 1/1 · Morning review · 0 commitments due · 70 ROTTEN left`.
4. **Text-first cursor identity in `planReviewJump`.**
   - Among unhandled entries in the cursor's file, match
     `originalMarkdown === cursorText` first. If several match, prefer the one on the
     cursor line, then the nearest.
   - Fall back to the line number only when no text matches.
   - Keep the endpoint (`[S`/`]S`) and anchor branches as they are.
5. **Day-scoped anchor.**
   - `buildReviewAnchor` records the local day.
   - `planReviewJump` and the stamp/complete paths ignore an anchor from another day, so
     yesterday's handled `gtd_daily.md:N` keys cannot hide today's occurrences.
6. **Complete gesture on a checklist row.** This is the single-target path in
   `refreshTaskFreshnessOnTasks`: no Vim count, and the cursor line exactly matches a
   queue entry whose tier is `pre`/`post`.
   1. Identify the entry by text first, as above.
   2. If advancing and the tier is not `post`, compute the next target before writing:
      `planReviewJump` forward, with handled keys = anchor keys plus this entry's key.
   3. Await `completeTaskAtCursor(editor)`. On `ok: false`, show
      `Not completed — {reason}` and stop.
   4. Rebuild `reviewAnchor` from the pre-write queue position, with this key handled.
   5. Report the outcome:
      - **Alt+Shift+F on PRE (or any non-POST checklist row):** land on the precomputed
        target through `landOnReviewQueueEntry` / `resolveReviewQueueLine`, which
        re-finds a shifted line by text. Show `✓ Done · {task text}`, then the boundary
        line (if crossing) and the landing notice.
      - **Alt+F on PRE:** stay, with `✓ Done · {k} PRE left`.
      - **Either key on POST:** stay, and show
        `Review closed — {r} ROTTEN left for later`. Prepend
        `{c} commitments still due · ` when c > 0.

   Never stamp, open a decay card, count a keep, or show the Pending Work Log prompt for
   checklist rows. A tagged line that is not an exact PRE/POST queue match (for example,
   scheduled tomorrow) falls through to today's behavior, including the
   `recurring · not reviewed` refusal.

7. **Batches.**
   - Counted Alt+F sessions and Task Link Alt+F batches skip PRE/POST targets.
   - Add a `· skipped {n} checklist` tail through the existing skip-tail helpers.
8. **Tests.**
   - **`scripts/test-navigation-freshness.cjs`:**
     - PRE/POST tier notices and gates;
     - text-first cursor matching in a stale queue after an above-insert: the cursor on
       "Brush teeth", now on Pills' old line, must not skip Pills;
     - the day-scoped anchor rollover;
     - boundary suffix and the into-POST line;
     - `]S` reaching POST with 77 ROTTEN rows and a null budget.
   - **`scripts/test-navigation-keep-counting.cjs`** (or a new file added to the
     `npm test` list):
     - complete all seven PRE rows in one file with Alt+Shift+F, with
       `recurrenceOnNextLine` both false and true, from a stale cache and from a
       refreshed cache;
     - manual Ctrl+Enter then `]s` from both caches;
     - POST stays put with the closing notice;
     - counted and link batches skip with the tail;
     - missing-gate refusals write nothing.
9. **Ship.**
   - Bump `manifest.json` to 2.3.0.
   - Update the `README.md` nav row (~16): PRE first, POST last, the completion keys,
     `]S` closes, and the gates.
   - `npm test`, `npm run validate`.
   - Commit, then run `bob plugins sync`.

## Phase rollout

1. **Binary.**
   - Run `just install` from merged bob-cli `master`.
   - Verify that `bob freshness list -f json` reports `schema_version: 9`, with `pre` =
     7 (or the number of chores still open and due today) and `post` = 1, in file order.
   - Verify the human output shows PRE first and POST last.
2. **Plugins.**
   - Confirm `bob plugins sync` deployed ledger 1.29.0, nav 2.3.0, and cycler 1.24.0
     byte-identical from the opened source.
   - Re-run the sync if not. Never patch `~/bob/.obsidian/plugins` by hand.
3. **Vault ritual (R7).** In the opened `gh:bobs-org/bob` clone, after checking
   `git status`, replace the Morning review text between `#post` and its first inline
   field with a closeout. Keep every inline field (`repeat`, `created`, `scheduled`)
   exactly as it is.

   ```markdown
   - [ ] #task #gtd #post Morning review: PRE chores done or knowingly skipped · walked
         to Commitments done (NEW never capped or skipped; lanes: keep Alt+Shift+F,
         today Ctrl+Shift+Enter, release Alt+N; returned not-now is a priority roll, not
         Alt+F) · [[crowded|CROWDED]] at 0 (split, sequence, defer, drop) · highlight
         chosen (≤3 themes, highlight first) · [[rotten|ROTTEN]] upkeep until 0, budget,
         or you stop — then ]S and complete this [repeat:: every day when done]
         [created:: …] [scheduled:: …]
   ```

   This drops `bob gkeep pull` (now the Keep-import PRE chore), the inner `]s` recipe,
   and "(≈10 min)". Commit only `gtd_daily.md`, sync it through `bob vault-sync`, and
   verify `~/bob`.

4. **Docs.**
   - **`docs/freshness.md` §6.** Rewrite the ritual:
     1. `[S`, or `]s` from outside the queue, starts at PRE.
     2. Complete each chore with Alt+Shift+F, or skip it with `]s`.
     3. Walk the commitments to "Commitments done".
     4. ROTTEN is optional upkeep.
     5. `]S` jumps to POST; complete Morning review last.
   - **§13 Rollout log.** Add a dated entry: "PRE/POST checklist tiers went live (schema
     9, ledger 1.29.0 / namespace v7, nav 2.3.0, cycler API v2)."
   - **`docs/getting-started.md`.** Update the morning walk lines.
5. **Live verification.** Use Obsidian on apollo or athena, and the Mac if available.
   Read `tailnet.md` with `/sase_memory_read` before any remote access.
   - From a non-queue cursor, `]s` lands on the first PRE chore with the PRE notice.
   - Alt+Shift+F completes it through Tasks: the next occurrence appears, there is no
     `[fresh::]`, and the walk lands on the next chore. Repeat through all seven,
     including one still `[?]` before the hooks run.
   - After PRE, the walk reaches NEW.
   - The commitments → ROTTEN boundary shows `· ]S closes the review`.
   - `]S` lands on Morning review with its counts.
   - Alt+F closes it with `Review closed — …`.
   - `bob freshness list -f json` and `api.freshness.queue()` agree on the first 20 keys
     and the tier counts.
   - The NEW/ROTTEN chips and READY counts match a pre-deploy snapshot.
   - `bob task-status-hooks --dry-run` shows no feature-induced rewrites.

   If a GUI or machine is unavailable, record the exact remaining checks as a
   verification gate on the phase bead rather than claiming a pass.

### Epic completion and rollback

**Completion.** The epic is complete when all of these hold:

- both evaluators pass CL1–CL12 and every existing vector;
- `bob` schema 9 is installed;
- ledger 1.29.0, nav 2.3.0, and cycler 1.24.0 are deployed;
- the vault tags, Morning review closeout, trial removal, decision and glossary
  amendments, and the `bob-cli-3h` wake are landed;
- live verification has passed, or its remaining checks are recorded as a gate.

**Fast rollback:** remove the `#pre`/`#post` tags from `gtd_daily.md`. The rows leave
the walk with no code change.

**Full rollback:** redeploy the previous binary and the three plugins together, and
restore the Morning review text.

Never roll back by stamping, restamping, or moving tasks. Do not bring back a trial.
