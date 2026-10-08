---
tier: epic
title: RECURRING walk tier so due recurring tasks reach the ]s morning review
goal: "Every open, visible, non-checklist recurring task whose occurrence date has
  arrived walks in a new RECURRING commitment tier of the ]s review (CLI, ledger footer,
  and nav notices alike), is never stamped, and leaves the walk only when it is
  completed, rescheduled past today, linked to Today, or cancelled through Obsidian
  Tasks, with no change to stamps, buckets, chips, READY, the ready cap, or PRE/POST
  checklist rows.

  "
decisions:
  tier_position:
    ask: Where should the new RECURRING tier sit in the walk?
    choices:
      before_tickler:
        "… NEXT → RECURRING → TICKLER …: both date-driven arrival tiers together"
      after_pre:
        "PRE → RECURRING → NEW …: due recurring obligations right after the PRE chores"
    default: before_tickler
    why: Groups the date-driven arrivals; NEW stays the first tier after the PRE chores
    answer: before_tickler
  ctrl_alt_f:
    ask: What should Ctrl+Alt+F do on a landed RECURRING row?
    choices:
      refuse:
        Write nothing and stay; the notice names the real answers (Ctrl+Enter, link,
        reschedule)
      skip: Write nothing and advance to the next row, exactly like ]s
    default: refuse
    why:
      A due recurring obligation should get an explicit answer, never a reflexive keep
    answer: refuse
  memory_records:
    ask:
      Add a decision record, mark review-walk-is-tiered superseded in part, and edit the
      freshness glossary term?
    memory:
      - decisions
      - decisions:review-walk-is-tiered
      - glossary:task-freshness
    default: false
    answer: false
phases:
  - id: rust
    title: Contract, Rust evaluator, and bob freshness CLI
    depends_on: []
    size: medium
    description:
      "rust: write the RECURRING contract and RC vectors into docs/freshness.md, add
      due/start to the Rust rows, apply the recurring overlay, queue order, counts,
      schema 11 CLI output, the recurring_undated lint, and tests."
  - id: ledger
    title: bob-ledger-tools evaluator, footer, and freshness namespace v9
    depends_on:
      - rust
    size: medium
    description:
      "ledger: mirror the RECURRING overlay, ordering, and counts in bob-ledger-tools,
      add the RECUR footer group and entry view, publish freshness namespace v9 with
      recurringTier, and pin the RC vectors in JS tests."
  - id: nav
    title: Navigation Hotkeys tier handling and recurring answers
    depends_on:
      - ledger
    size: medium
    description:
      "nav: teach bob-navigation-hotkeys the recurring tier (labels, commitment set,
      notices), the Alt+F / Ctrl+Alt+F recurring answer, and which gestures resolve a
      recurring landing, with tests."
  - id: rollout
    title: Install, deploy, vault closeout text, memory, and live check
    depends_on:
      - rust
      - ledger
      - nav
    size: small
    description:
      "rollout: install bob, sync plugins, verify the live vault's overdue recurring
      rows walk in RECURRING, adjust the Morning review text, apply or defer the memory
      changes, and leave Bryan a checklist."
proposed_by: bbugyi200.athena.0yb
decided_by: auto
create_time: 2026-10-08 11:03:13
status: wip
---

- **PROMPT:**
  [prompts/202610/recurring_review_tier.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/recurring_review_tier.md)

# Plan: a RECURRING walk tier so due recurring tasks are never missed

## Why recurring tasks are missing from `]s` today

Bryan suspects the GTD morning walk (`]s` / `[s`) skips recurring tasks (Obsidian Tasks
lines with a `repeat` field) and that he has missed due tasks because of it. He is
right.

**What happens now.** `walk_scope` and `in_scope` in both evaluators require
`¬recurring` (`docs/freshness.md` §4; Rust `src/native/freshness/state.rs`
`evaluate_without_checklist`; JS `bob-ledger-tools` `src/100-freshness-evaluate.js`).
The only exception is `#gtd #pre` / `#gtd #post` checklist rows. So a recurring task is
in no tier: when its `scheduled` date arrives, the hooks flip it from `[?]` to `[ ]` and
the dash READY section shows it. But `]s` never lands on it, the footer never counts it,
and `bob freshness list` never prints it.

**Live vault on 2026-10-08.** Four open, non-checklist recurring occurrences are
overdue, and no review surface shows any of them:

| Note       | Task                                                          | `scheduled` | Overdue |
| ---------- | ------------------------------------------------------------- | ----------- | ------: |
| `cash.md`  | Apply for unemployment every Sunday! (`every week on Sunday`) | 2026-09-13  |    25 d |
| `recur.md` | Email #person/jenika for new meds! (`every 28 days`)          | 2026-09-25  |    13 d |
| `recur.md` | Update [[cash#^balance]] … (`every 30 days when done`)        | 2026-09-30  |     8 d |
| `recur.md` | Pay [[kelly]] salary … (`every month on the 1st`)             | 2026-10-01  |     7 d |

The only other open recurring rows are the nine `#gtd` checklist rows in `gtd_daily.md`,
and those already walk in PRE/POST. No open task in the vault has a `due::` date. Seven
open tasks have `start::`, and none of those recur.

**Why it was built this way.** The exclusion was deliberate, but its premise did not
survive the move to a due-only walk:

1. **Freshness (bob-cli-31, `plan:202609/task_freshness_review.md`, research R13)**
   excluded recurring tasks for two reasons:
   - "Their recurrence already resurfaces them": at the time Bryan still walked the
     whole dash READY list, so each new occurrence showed up there.
   - "Tasks would copy `fresh` into the next occurrence." `[fresh::]` sits before the
     Tasks suffix, so Tasks treats it as description text and carries it forward.

   Stamping a recurring line has been refused ever since: Rust `placement.rs`, JS
   `freshnessStampLine` / `keepLine` / `setRefreshLine`, and nav
   `classifyFreshStampTarget` all refuse it.

2. **The tiered walk** (`decisions:review-walk-is-tiered`, research ADJ-10) kept them
   out of every tier because "`stampLine` refuses recurring lines". Every walk tier at
   that time resolved by a stamp, so an unstampable row could never leave.
3. **PRE/POST** rejected "lift the recurring exclusion for every task", because the bank
   and medication chores would land in NEW and then ROTTEN and stall there, unstampable.
   It admitted recurrence only for tagged checklist rows, which resolve by completion.

No step ever built the "recurrence resurfaces it" half. Once the freshness walk replaced
the full READY walk, a recurring occurrence that arrived became invisible to the review.

## The fix: a RECURRING tier gated by the occurrence date, not by a stamp

A recurring task has a natural due signal that ordinary tasks lack: its occurrence date.
The fix adds one walk tier, RECURRING, keyed on that date.

- **Membership.** A recurring row walks in RECURRING exactly when its occurrence has
  arrived. It must also be open in a lane, visible, not in a daily note, not
  Today-linked, and not a checklist row.
- **No writes.** It is never stamped, so the copy-into-next-occurrence problem cannot
  arise, and no stamp refusal changes anywhere.
- **Resolution.** It leaves the walk by itself when the occurrence stops being "arrived
  and visible":
  - completed: Tasks writes the next occurrence, and if that one is future-scheduled it
    is hidden;
  - its `scheduled` date is moved past today;
  - it is linked to Today;
  - it is cancelled through Obsidian Tasks;
  - it gains an open dependency.

This is the "recurrence resurfaces it" behavior the original design assumed. It is not
the rejected "lift the exclusion": recurring rows never enter NEW, ROTTEN, or any
stamp-resolved tier.

**Unchanged on purpose:**

- `state`, `bucket`, the `B = NEW ∪ TICKLER ∪ ROTTEN ∪ READY` partition, the dash
  sections and chips, `rotten.md`, the note ready cap and its `↻` recurring count, and
  `counts.due/new/resurfaced/rotten/fresh`;
- `upkeep_today`, the budget, keeps, decay, and `decide`;
- every stamp refusal, the seed, PRE/POST checklist behavior (checklist always wins),
  the Task Card's cancel and roll refusals for recurring tasks, and the counted / Task
  Link Alt+F whole-batch recurring refusal;
- Bob Mac Capture. It does not read `bob freshness` JSON; verified, it has no freshness
  references.

**Rejected variants** (recorded in the decision record if `memory_records` is approved):

- Stamping recurring rows: Tasks copies `[fresh::]`/`[keeps::]` forward, and an overdue
  fixed-cadence completion would create an already-arrived occurrence that looks fresh
  and hides.
- Folding the rows into TICKLER, which resolves by stamp.
- Walking undated recurring rows daily: they have no occurrence; they get a lint
  instead.
- Admitting `[?]` like checklist rows: the 15-minute hooks restore `[ ]` on their first
  run after the date arrives, the row stays due the next morning, and TICKLER already
  accepts the same lag. Reopen if an arrival-morning miss is ever observed.

## Shared contract (both evaluators implement exactly this)

`today` is the vault's local date (`BOB_NOW` in tests). Add this to `docs/freshness.md`
§4 in the existing pseudo-spec style:

```text
recurring(t)       = t carries a Tasks recurrence (`repeat` field) — the existing predicate on each side
occurs_on(t)       = the earliest of t's valid `scheduled`, `due`, and `start` dates (Tasks' "happens" date);
                     none when it has none of them
recurring_scope(t) = recurring(t) ∧ checklist(t) = none
                     ∧ lane(t) ≠ none                       (status ` `/TODO type, `*`, or `/`; `[?]` and closed rows stay out)
                     ∧ lane-visible                         (the existing predicate: not dependency-blocked, no #hide,
                                                             not under _templates/_conflicts, no scheduled > today)
                     ∧ ¬canonical daily note ∧ ¬Today(t)
recurring_due(t)   = recurring_scope(t) ∧ occurs_on(t) exists ∧ occurs_on(t) ≤ today
tier(t)            = checklist(t)   if checklist_scope(t)              (unchanged; beats everything)
                   | recurring      if recurring_due(t)                 (beats every other rule; trackers included)
                   | the existing rules otherwise (recurring rows never reach them: in_scope/walk_scope keep ¬recurring)
due_on(t)          = occurs_on(t) for recurring rows; days_overdue = today − occurs_on(t) (0 on the day)
state, bucket      = unchanged: null for every recurring row (they stay out of in_scope)
decide             = false for recurring rows; lane and interval fields are reported as today
```

Implement the tier as an **overlay** applied after the checklist overlay, so that no
early return in the evaluator can drop it (the same trap the PRE/POST research flagged).
In Rust: `evaluate` =
`overlay_recurring(overlay_checklist(evaluate_without_checklist(…)))`. The recurring
overlay is a no-op whenever the row is a checklist member.

**Order within RECURRING:** `due_on ↑`, then path ↑, then line ↑ (the oldest occurrence
first).

**Walk position** (decision `tier_position`):

> [!decision] tier_position = before_tickler PRE → NEW → PROJECTS → PENDING → NEXT →
> **RECURRING** → TICKLER → REFERENCES → ROTTEN → POST. In Rust, declare
> `Tier::Recurring` between `Next` and `Tickler` (`Tier` derives `Ord`, so declaration
> order is walk order). In JS, set `FRESHNESS_TIER_ORDER` to
> `pre 0, new 1, projects 2, pending 3, next 4, recurring 5, tickler 6, references 7, rotten 8, post 9`.

> [!decision] tier_position = after_pre PRE → **RECURRING** → NEW → PROJECTS → PENDING →
> NEXT → TICKLER → REFERENCES → ROTTEN → POST. In Rust, declare `Tier::Recurring`
> between `Pre` and `New`. In JS, set `FRESHNESS_TIER_ORDER` to
> `pre 0, recurring 1, new 2, …, post 9`.

**Tier properties:**

- RECURRING is a **commitment** tier, before the "Commitments done" boundary.
- Machine name `recurring`. Human/CLI/notice label `RECURRING`. Footer short label
  `RECUR`, with legend `RECUR = RECURRING` (shown only when visible, in walk order with
  the other abbreviations).

**Counts:**

- `by_tier` gains `recurring`, making ten keys.
- `walk = sum(by_tier)` includes it.
- New `counts.recurring_due` (JS `recurringDue`) equals the tier count.
- Nothing else changes.

**Lint (Rust scan-side only, like the checklist lints):** `recurring_undated`. It fires
for an open task that meets all of these:

- it is recurring;
- `checklist_from_tags` is none;
- its status is a lane status (` `/TODO type, `*`, `/`);
- it is outside `_templates`/`_conflicts`;
- it has no valid `scheduled`, `due`, or `start`.

Message: "recurring task has no scheduled, due, or start date; the review walk cannot
tell when it comes due".

**Schema:**

- `bob freshness list` JSON moves to schema 11: "11: RECURRING walk tier: `recurring` in
  `tier` and `by_tier`, `counts.recurring_due`; recurring rows carry `due_on` = the
  occurrence date".
- bob-ledger-tools publishes freshness namespace v9 with the explicit capability
  `recurringTier: true`. Top-level api stays v3.

### RC vectors (copy verbatim into `docs/freshness.md` §10 as "Recurring tier (RC1–RC12)")

Today `2026-10-08`, default config (interval 7, lanes 1), path `a.md` and Ready `[ ]`
unless noted. "→ none" means no tier and `state` null.

- **RC1 arrived:**
  `- [ ] #task Pay salary [repeat:: every month on the 1st] [scheduled:: 2026-10-01]` →
  tier `recurring`, lane `ready`, `state`/`bucket` null, `due_on` 2026-10-01,
  `days_overdue` 7, `decide` false.
- **RC2 arrives today:** the same with `[scheduled:: 2026-10-08]` → `recurring`,
  `due_on` 2026-10-08, `days_overdue` 0.
- **RC3 not yet:** `[scheduled:: 2026-10-09]` → none (not lane-visible).
- **RC4 due only, past:** `[repeat:: every year] [due:: 2026-10-05]` with no `scheduled`
  → `recurring`, `due_on` 2026-10-05, `days_overdue` 3.
- **RC5 due only, future:** `[due:: 2026-10-20]` with no `scheduled` → none.
- **RC6 earliest date wins:** `[start:: 2026-10-02] [due:: 2026-10-20]` → `recurring`,
  `due_on` 2026-10-02, `days_overdue` 6.
- **RC7 undated:** `- [ ] #task Water plants [repeat:: every week]` → none. Rust
  `bob freshness list` reports warning `recurring_undated` for it. This is also why L3
  and S13 (whose recurring rows carry no dates) stay unchanged.
- **RC8 lanes:** a recurring `[*]` and a recurring `[/]`, each with
  `[scheduled:: 2026-10-01]` → `recurring`, keeping lane `next` / `pending` and null
  state. Neither is ever `next` or `pending` tier.
- **RC9 exclusions:** a recurring row with `[scheduled:: 2026-10-01]` that is any of the
  following → none:
  - Today-linked;
  - in `2026/20261008.md`;
  - `#hide`;
  - dependency-blocked;
  - under `_templates/`;
  - `[?]` (not a lane).
- **RC10 checklist wins:**
  `- [ ] #task #gtd #pre Brush teeth [repeat:: every day when done] [scheduled:: 2026-10-01]`
  → `pre`, never `recurring`. CL1–CL12 are unchanged.
- **RC11 a stray stamp is ignored:** `[fresh:: 2026-10-08] [scheduled:: 2026-10-01]` on
  a recurring row → still `recurring`, with null `state`. Stamping that line is still
  refused.
- **RC12 order and counts:** the RECURRING rows are:
  - `b.md:3` scheduled 10-05
  - `a.md:9` scheduled 10-05
  - `a.md:2` scheduled 10-01

  Add NEW `w.md:1`, a `[*]` `n.md:1` stamped 10-07, and TICKLER `t.md:1` (fresh 10-05,
  scheduled 10-07). The RECURRING order is `a.md:2`, `a.md:9`, `b.md:3`. Counts:
  - `by_tier.recurring` 3, `recurring_due` 3, `walk` 6;
  - `due` 2 (NEW + resurfaced; recurring rows add nothing);
  - `upkeep_today` unchanged.

  The full walk order depends on `tier_position`:

> [!decision] tier_position = before_tickler RC12 full order: `w.md:1` (new), `n.md:1`
> (next), `a.md:2`, `a.md:9`, `b.md:3` (recurring), `t.md:1` (tickler).

> [!decision] tier_position = after_pre RC12 full order: `a.md:2`, `a.md:9`, `b.md:3`
> (recurring), `w.md:1` (new), `n.md:1` (next), `t.md:1` (tickler).

### Answers on a RECURRING landing (nav)

| Gesture on the landed RECURRING row | Effect                                                                                                                            | Walk                                             |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Ctrl+Enter                          | completes through Tasks (recurrence fires); existing cycler path                                                                  | advances once (`complete` resolves)              |
| Ctrl+Shift+Enter                    | links to Today (Today-linked rows leave the walk)                                                                                 | advances once (`link-today` resolves)            |
| Task Card commit (Ctrl+Shift+P)     | resolves when closed, `scheduled` > today, a new `dependsOn` id appears, or the earliest inline `scheduled`/`due`/`start` > today | advances once only when it resolves              |
| Alt+N                               | lane change only; the row is still due                                                                                            | stays (`lane` does not resolve for `recurring`)  |
| block-id-prompt route               | the row is still due wherever it lives                                                                                            | stays (`route` does not resolve for `recurring`) |
| Alt+F                               | writes nothing; recurring-tier notice                                                                                             | stays                                            |
| Ctrl+Alt+F                          | see decision `ctrl_alt_f`                                                                                                         | see decision                                     |
| `]s` / Ctrl+Shift+M                 | unchanged (skip / move never advances)                                                                                            | unchanged                                        |

The recurring-tier notice reads
`RECURRING · never stamped — Ctrl+Enter done · Ctrl+Shift+Enter today · Ctrl+Shift+P reschedule · ]s skip`.
Name only answers that actually work on recurring lines; see the nav phase's
verification step. Recurring rows that are not walk landings keep today's
`recurring · not reviewed`.

> [!decision] ctrl_alt_f = refuse Ctrl+Alt+F on a landed RECURRING row behaves like
> Alt+F: it writes nothing, shows the recurring-tier notice, and stays on the row. This
> matches today's behavior, where both chords already refuse recurring lines; only the
> notice text changes.

> [!decision] ctrl_alt_f = skip Ctrl+Alt+F on a landed RECURRING row writes nothing and
> advances to the next remaining review item, with the preamble `RECURRING left due`. It
> uses the same advance tail as the keep path (`buildReviewAnchor` plus
> `jumpToDueTask(1, { fromAdvance: true, preamble })`). Alt+F still refuses in place.

## Phase rust

Work in bob-cli. Land the contract docs first, then the code.

1. **`docs/freshness.md`.**
   - **Intro:** after the "visible, non-recurring Ready task" sentence, add: "Recurring
     tasks are never stamped; once a recurring occurrence's date arrives it walks in the
     RECURRING tier until it is completed, rescheduled, linked to Today, or cancelled."
     Add RECURRING to the tier-order sentence.
   - **§3 refusals:** reword the recurring bullet to: "its occurrence walks in RECURRING
     once it arrives (§4), and Tasks would copy the stamp into the next occurrence".
   - **§4:**
     - Add the shared contract above.
     - Update the `tier(t)` block, the order paragraph, and the per-tier table (add a
       `recurring` row).
     - Add RECURRING to the commitment sentence and to the counts bullets (ten `by_tier`
       keys, `recurring_due`).
     - Update the tracking-review tier-order sentence.
     - Machine vocabulary: add a schema 11 entry and the namespace v9 / `recurringTier`
       sentence.
     - Footer: add the `RECUR` label and legend to the short-label paragraph and its
       example.
     - Lints: add `recurring_undated` to the lint list.
   - **§6 ritual:**
     - Add RECURRING to the commitments list in step 3, with one sentence: "In RECURRING
       the occurrence date has arrived: complete it (Ctrl+Enter), do it today
       (Ctrl+Shift+Enter), or move its scheduled date past today (Ctrl+Shift+P);
       recurring rows are never stamped and cancel only through Obsidian Tasks."
     - Add the Ctrl+Alt+F behavior chosen by `ctrl_alt_f`.
     - Add a recurring entry to the "Review outcomes" paragraph.
   - **§7:**
     - Schema 11 history line.
     - JSON `tier` enum.
     - `by_tier` and `recurring_due` in the JSON example.
     - A RECURRING section in the human example, at its walk position, above the
       commitments divider.
   - **§8:** update the surfaces table's tier list if it states one.
   - **§10:**
     - Amend L3 to read "an undated recurring `[*]`".
     - Note under S13 that its recurring row is undated.
     - Add the RC1–RC12 block verbatim.
   - **Other files that state the full walk order:** update them all. Find them with
     `grep -rn "NEXT → TICKLER"`; this finds `README.md` "Task freshness" plus code
     comments in `state.rs`, `state_tests.rs`, and `cli.rs`. Also bump the schema number
     wherever it is stated.
2. **`src/native/dataview/tasks/mod.rs`.**
   - `RichTask` gains `due` and `start` (`Option<NaiveDate>` via
     `TaskDate::valid_date`), filled in `rich_task`.
   - Update every other `RichTask { … }` literal (grep for them).
3. **`state.rs`.**
   - Add `Tier::Recurring` at the decided position, with `as_str` `"recurring"`.
   - `FreshnessRow` gains `due` and `start`. Update the `row`/`lane_row`/`ready_row`
     helpers and every full literal.
   - Add `occurs_on(row)` (the minimum of the present dates).
   - Add `overlay_recurring` per the contract, wired into `evaluate` after
     `overlay_checklist`. It sets `tier`, `due_on`, `days_overdue`, and
     `decide = false`, and leaves `state`, `lane`, `fresh`, interval, keeps, and lints
     alone.
   - In `queue`, add the RECURRING comparator arm. Recurring rows always carry a lane,
     so the null-lane guard is unchanged.
   - `ByTier.recurring` and `sum()`; `Counts.recurring_due`.
   - Fix any exhaustive `Tier` match elsewhere that the compiler flags.
4. **`scan.rs`.**
   - `freshness_row` passes `due`/`start`.
   - No new candidate scan is needed: recurring lane rows already arrive through
     `READY_QUERY` / `PENDING_QUERY` / `NEXT_QUERY`.
   - Add `recurring_undated` beside `checklist_lints` in `collect_warnings`, plus its
     `lint_message`.
5. **`cli.rs`.**
   - `SCHEMA_VERSION = 11`, plus a history comment.
   - Add RECURRING to the long-about tier text.
   - Header: add `{recurring} recurring` at its walk position.
   - Grow the `tiers` array to 10 entries. RECURRING counts toward `commitment_rows`.
   - Add a `human_row` `"recurring"` arm:
     - `Recurring · due today`, or
     - `Recurring · {N}d overdue · since {due_on}`.
   - JSON: add `by_tier.recurring` and `counts.recurring_due`.
6. **Tests.**
   - `state_tests.rs`:
     - Add RC1–RC12 as tests.
     - Extend the existing nine-tier order test to ten tiers.
     - Keep every S/Q/L/R/B/D/PR/RF/CL/K test green.
   - `tests/cli/freshness.rs`:
     - Move the schema assertions to 11.
     - Add a `recur.md`-style fixture vault containing:
       - a monthly `[ ]` scheduled in the past;
       - a due-only past row;
       - a future-scheduled `[?]`;
       - an undated recurring row (expect the warning);
       - a Today-linked recurring row;
       - a `#gtd #pre` recurring chore.
     - Assert the JSON tiers, order, `due_on`/`days_overdue`, counts, and warning, plus
       the human RECURRING section's position relative to the commitments divider.
   - `just all` passes.
7. Do not install `bob` and do not edit memory in this phase.

## Phase ledger

Work in bob-plugins. Open it with `sase repo open bob-plugins -r "<reason>"`, read its
`AGENTS.md`, and use only the printed path. bob-ledger-tools is fragment-built:

- Edit `plugins/bob-ledger-tools/src/`.
- Run `npm run build`; never hand-edit `main.js`.
- Keep every fragment at or below 1000 lines. `120-freshness-footer.js` is about 923,
  `130-freshness-marks.js` about 955, and `230-plugin-freshness-api.js` about 920. Split
  into a new fragment registered in `src/fragments.json` if needed.

Copy the RC vectors from bob-cli `docs/freshness.md` §10 as merged by the rust phase.

1. **`160-widgets-and-row.js` `freshnessRowFromTask`.**
   - Add `due` and `start` (`YYYY-MM-DD` or null) to the main return object and to both
     fallback objects.
   - Source them from Tasks `dueDate` / `startDate`, falling back to inline `[due::]` /
     `[start::]` exactly as `scheduled` already falls back.
2. **`100-freshness-evaluate.js`.**
   - Add `freshnessOccursOn(row)`.
   - Apply the recurring overlay at a single exit after the checklist logic, so the
     lane-row and null-state early returns cannot drop the tier.
   - Update `FRESHNESS_TIER_ORDER` per the decision.
   - `freshnessTierLabel` → `RECURRING`; `freshnessTierFooterLabel` → `RECUR`.
   - Update the header comments.
   - `walkScope` and `inScope` keep `!safe.recurring`.
3. **`110-freshness-queue.js`.**
   - RECURRING comparator: `dueOn ↑`, path ↑, line ↑. It must not fall through to the
     rotten comparator.
   - `freshnessCounts`: `byTier.recurring`, `walk` sum, `recurringDue`.
   - `freshnessStatusView`: tier count, walk fallback, text, tooltip, and RECURRING in
     the commitment mode list.
4. **`120-freshness-footer.js`.**
   - Add the tier to `FRESHNESS_FOOTER_TIERS` and `FRESHNESS_FOOTER_COMMITMENT_TIERS`
     (walk order).
   - Add `recurring` to the `freshnessReviewMachineTier` whitelist.
   - `freshnessReviewEntryView` recurring branch:
     - detail `recurring · due today` / `recurring · {N}d overdue`;
     - compact equivalent;
     - `actionHint`
       `Ctrl+Enter done · Ctrl+Shift+Enter today · Ctrl+Shift+P reschedule · ]s skip`.
   - `freshnessFooterReadTiers`.
   - Legend `RECUR = RECURRING`.
5. **`130-freshness-marks.js`.** If tone or tooltip logic keys on tier, treat
   `recurring` as a due commitment tier, with the tooltip line "complete or reschedule
   to resolve". Change nothing else.
6. **`230-plugin-freshness-api.js`.** Add `recurringDue` and `byTier.recurring` to the
   fallback counts, and update the header comment.
7. **`170-plugin-lifecycle.js`.** Freshness namespace `version: 9` with
   `recurringTier: true`; keep every existing capability.
8. **Versioning and docs.**
   - Bump `plugins/bob-ledger-tools/manifest.json` to the next minor version.
   - In the root `README.md`, update the ledger row (version, tier order, namespace v9 /
     `recurringTier`), the `freshness` object shape, and the footer tier/legend text.
9. **Tests.**
   - Add RC1–RC12 (RC7 without the lint) in a new
     `scripts/test-ledger-tools-freshness-recurring.cjs`, registered in the root
     `package.json` `test` script. Give the harness row builders (`sRow`, `laneRow`,
     `checklistTaskRow`) `due`/`start` overrides.
   - Update the assertions the new tier legitimately changes:
     - namespace version / capability;
     - tier order;
     - footer label map;
     - legend;
     - `counts.walk`.
   - Leave unrelated exact strings alone. Empty tiers stay hidden, so most status-bar
     strings do not change.
   - Run `npm test` and `npm run validate`, which must pass.
10. Commit in bob-plugins, then run `bob plugins sync` (repo rule). Until the nav phase
    lands, the older nav treats `recurring` entries generically: `]s` visits them, and
    both Alt+F chords still refuse with `recurring · not reviewed`.

## Phase nav

Work in bob-plugins (`sase repo open bob-plugins -r "<reason>"`; read `AGENTS.md`).
bob-navigation-hotkeys is now fragment-built from `src/` (`src/fragments.json`), so edit
fragments, run `npm run build`, and respect the 1000-line cap. Line budgets:
`536-plugin-review-advance.js` about 961, `480-review-jump-and-nav-api.js` about 943,
`470-keydown-and-freshness.js` about 900. Add a new fragment if needed. Refresh-and-
advance is **Ctrl+Alt+F**; Alt+Shift+F is retired and must stay unmatched.

1. **`470-keydown-and-freshness.js`.**
   - Add `reviewFreshnessSupportsRecurringTier(api)`: namespace `version >= 9` and
     `recurringTier === true`.
   - Add `recurring` to `reviewEntryMachineTier` and `reviewIsCommitmentTier`, and to
     any tier label map.
   - `reviewWalkRemaining` counts recurring rows as commitments.
   - The boundary notice keeps firing only into ROTTEN/POST.
   - Add a `matchReviewRecurringCursor` helper modeled on `matchReviewChecklistCursor`
     (text-first match against the live queue).
2. **`480-review-jump-and-nav-api.js`.**
   - Add a recurring detail and hint to the `buildReviewJumpNotice` per-tier fallback.
   - Add the recurring-tier notice constant.
   - `classifyFreshStampTarget` / `freshStampRefusalNotice` are unchanged for
     non-landing rows.
3. **`520-plugin-lane-links-and-review.js` `refreshTaskFreshnessOnTasks`.** For a
   single, uncounted target whose cursor row matches a live `recurring` queue entry, and
   when the capability is present, apply the Alt+F / Ctrl+Alt+F behavior from the table
   and decision above before any stamp classification. It writes nothing. Counted and
   Task Link batches are unchanged.
4. **`538-review-walk-identity.js` `reviewOutcomeResolves`.** Add the `recurring` rules
   from the table:
   - `complete` and `link-today` resolve; `lane` and `route` do not.
   - `card` resolves when the row is closed, `scheduled` > today, a new `dependsOn` id
     appears, or the earliest inline `scheduled`/`due`/`start` > today. It never
     resolves via `fresh`.
   - Pre/post behavior is unchanged.
5. **Ctrl+Enter.** No new claim is needed: the cycler's existing path completes through
   Tasks and calls `continue({ kind: "complete" })`. Add a test for an overdue
   fixed-cadence row (`every week on Sunday`, `scheduled` 3 weeks back) whose next
   occurrence is inserted above and is still arrived. Completing the landed row must not
   land on the `[x]` line and must not skip another due row, and the new occurrence must
   stay in RECURRING.
6. **Verify the reschedule answer** on a recurring line:
   - The Task Card Schedule date pick must write a new `scheduled`. Only the
     cancel/decay-cancel paths refuse recurring today; `040-priority-roll.js`
     `kind: "unavailable"` covers the cancel step only.
   - If any Schedule path refuses recurring lines, drop `Ctrl+Shift+P reschedule` from
     the notice and from the ledger `actionHint`, and say "edit its scheduled date"
     instead.
7. **Versioning and docs.**
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` to the next minor version.
   - Update the root `README.md` nav row: version, tier order, and one sentence on
     RECURRING answers.
   - Update any manifest description that lists the tiers.
8. **Tests.** Extend the following, adding a new test file to the root `package.json`
   list if you create one:
   - `test-navigation-freshness.cjs`: machine tier, commitment set, capability gate,
     boundary, recurring landing notice for both chords, and the Alt+Shift+F retirement
     still holding;
   - `test-navigation-review-advance.cjs`: the `reviewOutcomeResolves` table rows for
     `recurring`;
   - the Ctrl+Enter recurrence-insert case.

   Run `npm test` and `npm run validate`, which must pass. Commit, then run
   `bob plugins sync`.

## Phase rollout

1. **Install.** From bob-cli `master`, run `just install`.
2. **Live check.**
   - `bob freshness list -f json` reports `schema_version` 11.
   - Every open, arrived, non-checklist recurring row in the live vault (on 2026-10-08:
     the four rows in the table at the top, unless Bryan has handled them) is in tier
     `recurring`, with the right `due_on` / `days_overdue`.
   - The `gtd_daily.md` `#gtd` rows are still `pre`/`post`.
   - `counts.due` / `new` / `rotten` match a pre-install capture taken just before
     `just install`.
   - Report any `recurring_undated` warnings.
3. **Deploy plugins.**
   - Run `bob plugins sync` from bob-plugins `master`.
   - Confirm the deployed `main.js` files match the repo (`cmp`) for bob-ledger-tools
     and bob-navigation-hotkeys.
   - Confirm the deployed manifest versions.
4. **Vault.** In `~/bob/gtd_daily.md`, edit the `#gtd #post` Morning review line in
   place. Change only its text, and keep its fields:
   - Right after "tickler not-now is a priority roll, not Alt+F", add
     `; recurring: Ctrl+Enter done, Ctrl+Shift+Enter today, or reschedule — never Alt+F`.
   - Never commit the vault by hand. Finish with `bob vault-sync run` and
     `bob vault-sync status`.
5. **Memory** (decision `memory_records`):

> [!decision] memory_records Use `/sase_memory_write`, then run `sase memory init`.
>
> - **New decision record** `decisions/recurring-occurrences-walk-when-due`, titled "A
>   Recurring Task Walks Once Its Occurrence Arrives". Follow the frontmatter of its
>   sibling records.
>   - **Roster summary:** "A non-checklist recurring task walks in the RECURRING
>     commitment tier once its occurrence date (earliest scheduled/due/start) arrives;
>     it is never stamped and leaves only when completed, rescheduled past today, linked
>     to Today, or cancelled through Tasks."
>   - **Applies to:** bob-cli, bob-plugins, and the vault.
>   - **Claim:** the shared contract, as decided (including the `tier_position` and
>     `ctrl_alt_f` answers).
>   - **Rejected alternatives:** the four "Rejected variants" above, plus "Ctrl+Alt+F
>     completes, as on PRE/POST: a reflexive key must never mark Pay salary done".
>   - **Evidence:** this plan, bob-cli schema 11, ledger freshness namespace v9
>     `recurringTier`, the nav version, and the 2026-10-08 census of four overdue rows.
>   - **Cost:**
>     - a due recurring row asks every morning until answered;
>     - an overdue fixed-cadence task needs one answer per missed occurrence, or one
>       reschedule;
>     - an arrival-morning `[?]` row appears only after the first hooks run.
>   - **Reopens when:** an arrival-morning miss is traced to a stale `[?]`, or RECURRING
>     rows are routinely skipped with `]s` (then they need a snooze).
>   - **Links:** `[[decisions/review-walk-is-tiered]]` and
>     `[[decisions/answering-advances-the-walk]]`.
> - **Mark `decisions/review-walk-is-tiered`** with
>   `metadata.status: superseded-in-part` and `superseded_by` the new record. Add one
>   back-link line stating what is retired:
>   - its tier list;
>   - "recurring … tasks are in no tier except as PRE/POST checklist rows";
>   - the rejected alternative "Lifting the recurring exclusion for every task", now
>     narrowed: recurring rows still never enter NEW/ROTTEN.
>
>   Do not otherwise edit its body.
>
> - **Glossary `glossary:task-freshness`:** update the walk-order sentence to the ten
>   tiers, and add: "A non-checklist recurring task is never stamped; once its
>   occurrence date (earliest of scheduled, due, start) arrives it walks in RECURRING
>   until completed, rescheduled, linked to Today, or cancelled."

> [!decision] memory_records = no Do not edit memory. File one `memory` task bead
> through `/sase_new_task`. It names `decisions/review-walk-is-tiered` (it now
> contradicts the code: "recurring … tasks are in no tier") and
> `glossary/task-freshness` (its tier order and "non-recurring" wording), and it
> proposes the record and edits described in the `memory_records` branch.

6. **Bryan's checklist** (record it as a note on the epic bead):
   1. Reload Obsidian.
   2. `[S`, then walk.
   3. At RECURRING, expect the overdue rows and a `RECUR n` footer group.
   4. Answer each row. "Apply for unemployment every Sunday!" is about four occurrences
      behind: completing it creates the 2026-09-20 occurrence, which is still overdue.
      Either reschedule it to the next Sunday or cancel it through Obsidian Tasks if it
      no longer applies.
   5. Confirm that Alt+F and Ctrl+Alt+F on a RECURRING row write nothing.
   6. Confirm that Ctrl+Enter completes and moves on.
   7. Confirm that "Commitments done" appears only after RECURRING is clear or skipped.

## Epic completion and rollback

**Done means:**

- All four phases' tests pass (`just all`; bob-plugins `npm test` and
  `npm run validate`).
- The deployed plugins match `master`.
- The live check in rollout step 2 holds.

**Rollback** writes nothing to restore, because RECURRING never writes a task line:

1. Revert the bob-cli commits and run `just install`.
2. Revert the bob-plugins commits and run `bob plugins sync`.
3. Revert the one-line vault edit.
4. Revert the memory edits if they were applied, and run `sase memory init`.
