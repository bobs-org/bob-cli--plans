---
tier: epic
title: Checklist stages and period-draft launches for the morning GTD walk
goal: 'PRE and POST checklist rows walk in a deliberate, configured order of named
  stages (terminal → periods → browser → everything else in PRE; periods → Morning
  review in POST) that every review surface shows the same way, and three recurring
  PRE rows remind Bryan, on each period-review day, to launch the agents that draft
  last week''s, quarter''s, and year''s period-note Reviews.

  '
decisions:
  decision_record:
    ask: Record the checklist-stages design as a new accepted decisions record?
    memory:
    - decisions
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    default: true
    answer: false
  glossary_term:
    ask: Add a "Checklist Stage" glossary term so agents share the new vocabulary?
    memory:
    - glossary
    default: false
    answer: false
phases:
- id: walk_order
  title: Checklist stages in the bob-cli evaluator, CLI, and contract
  depends_on: []
  size: medium
  description: 'walk_order: parse and validate freshness.pre_stages/post_stages, resolve
    each PRE/POST row''s stage, sort PRE/POST by stage rank then path and line, add
    three lints, schema-16 JSON, the human stage detail, the shared CS/SC fixture,
    and the freshness contract and guide docs.'
- id: period_ids
  title: Relative period ids (last-week, last-quarter, last-year) in bob periods
  depends_on: []
  size: small
  description: 'period_ids: accept last-week, last-quarter, and last-year wherever
    bob periods show, brief, and draft take a PERIOD, resolved against today, so the
    launch rows can carry one paste-ready agent prompt forever.'
- id: ledger_order
  title: Checklist stages in bob-ledger-tools (JS mirror, namespace v14, footer)
  depends_on:
  - walk_order
  size: medium
  description: 'ledger_order: mirror the stage config, resolver, and comparator in
    the JS evaluator against a byte-for-byte copy of the shared fixture, expose stage
    fields under freshness namespace v14 checklistStages, and show the stage in the
    review footer and landing toasts.'
- id: stage_labels
  title: Stage labels beside the sunrise and flag in the editor
  depends_on:
  - ledger_order
  size: medium
  description: 'stage_labels: specify and render a quiet small-caps stage label right
    after the PRE sunrise / POST flag in Live Preview and Reading view, only for a
    configured stage tag adjacent to the identity run, with tooltip, reveal, toggle,
    and light/dark checks.'
- id: nav_ticks
  title: Stage-end ticks and staged-advance regression tests in navigation
  depends_on:
  - ledger_order
  size: small
  description: 'nav_ticks: add the quiet STAGE ✓ tick to Ctrl+Enter done lines when
    a stage''s last due row is completed, and lock in queue-order advance when the
    next staged row sits above the cursor.'
- id: rollout
  title: Config, vault rows, end-to-end verification, and the decision record
  depends_on:
  - walk_order
  - period_ids
  - ledger_order
  - stage_labels
  - nav_ticks
  size: small
  description: 'rollout: add the stage lists to the chezmoi-managed bob config, tag
    the existing gtd_daily rows, add the three launch rows, reword the review rows,
    verify the Monday walk, and write the accepted memory records.'
proposed_by: bbugyi200.athena.193
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 18:44:49
status: wip
bead_id: bob-cli-ho
---

- **PROMPT:** [prompts/202611/gtd_checklist_stages.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gtd_checklist_stages.md)
- **BEAD:** [bob-cli-ho](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ho/README.md)

# Checklist stages and period-draft launches for the morning GTD walk

## Outcome

Today PRE and POST walk in `path ↑, line ↑` order. Because Obsidian Tasks inserts each
new occurrence above the completed line and the 15-minute status grouper moves blocks
around, that order changes almost every morning (SASE cleanup has wandered between
positions 1 and 7; the browser chores are never adjacent). On Mondays the Morning review
even closes the walk before the Weekly review.

After this epic, a Monday morning walks like this, every week, on every surface (`]s`,
Ctrl+Enter auto-advance, the footer, landing toasts, `bob freshness list`):

```text
PRE  · TERMINAL  ☀ terminal  Clean up SASE agents on athena and apollo …       (file order inside a stage)
                 ☀ terminal  Run `task reconcile` + `gcal sync` + `gkeep pull`
PRE  · PERIODS   ☀ periods   Start last week's review draft …   ← the draft agent runs from here on
PRE  · BROWSER   ☀ browser   Review next week in Google Calendar
                 ☀ browser   Check weather
                 ☀ browser   Read email
PRE              ☀ Brush teeth · ☀ Take fish oil … · ☀ Do morning stretches   (unstaged: last)
…  NEW → PROJECTS → PENDING → NEXT → RECURRING → TICKLER → ROTTEN  (unchanged)
POST · PERIODS   ⚑ periods   Weekly review: correct the draft PRE started, write the Reflection
POST             ⚑ Morning review …   ← always last; "Review closed" fires here
```

On 2027-01-04 the quarter launch row joins the PERIODS stage; on 2027-01-11 the year
launch row does. Quarterly and Yearly reviews stay ordinary RECURRING rows.

## Requirements, as adopted

Bryan asked for (1) recurring PRE rows that come due on each period-review day and
remind Bryan to launch agents that update the period notes, prioritized first in PRE;
(2) "Clean up SASE agents" next; (3) the browser chores (weather, email, calendar) next,
always together; (4) everything else after; and (5) support for ordering PRE and/or POST
rows. Bryan then **agreed with every requirement recommended** in
`research:202611/gtd_checklist_stages_and_period_prep/gtd_checklist_stages_and_period_prep.md`
(read it with `sase artifact read` before starting any phase). That research adjusts the
literal request; these adjustments are now requirements:

| ID  | Adopted requirement                                                                                                                                              |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | The feature is **checklist stages**, never "priority". `[priority::]` (the P1–P4 deferral and decay ladder) and every inline field stay untouched.               |
| A2  | **SASE cleanup walks before the launches** (`sase update` can break freshly launched agents). Stage order: `terminal → periods → browser`.                       |
| A3  | The `task reconcile` + `gcal sync` + `gkeep pull` row joins the `terminal` stage, so the agents brief a freshly synced vault.                                    |
| A4  | Only the Weekly review is POST. Quarterly/Yearly reviews stay RECURRING; their launch rows pair with them **by date**.                                           |
| A5  | Stages work in POST too: the Weekly review is `#periods`; the Morning review stays unstaged, so it is always last.                                               |
| A6  | A launch row is done once its agent is **running** (or the Review is already drafted/edited). Correcting the draft and the Reflection stay with the review rows. |
| A7  | Review rows are reworded: drafting moves to PRE; launching there is only the fallback.                                                                           |
| A8  | Stage membership is an explicit tag; nothing is inferred.                                                                                                        |
| A9  | Inside a stage, order stays file order.                                                                                                                          |
| A10 | No auto-launch.                                                                                                                                                  |

Design choices I made beyond the research (as design lead):

- **Relative period ids** (`last-week`, `last-quarter`, `last-year`) in `bob periods`.
  Tasks copies a row's description into every occurrence forever, so a launch row can
  never name a concrete period. With relative ids, each launch row carries one exact,
  paste-ready agent prompt instead of "today's note → week link → ‹" navigation and
  mental ISO-week math, and `bob periods` resolves the closed period deterministically.
- **Stage labels in the editor** ship in their own phase so the evaluator and footer can
  land and be verified first.
- **A stage-local position** (`TERMINAL 1/2`) appears wherever the tier position already
  does, and a quiet `STAGE ✓` tick marks a finished batch. Nothing else new is drawn.

## What exists today (read before implementing)

bob-cli (this repo):

- `src/native/freshness/state.rs`: `ChecklistKind`, `Tier`, `FreshnessRow` (no tags
  today; `checklist` is computed by `checklist_from_tags`), `checklist_in_scope`,
  `overlay_checklist`, `evaluate`, `QueueEntry`, and `queue()` whose comparator arm
  `Tier::Pre | Tier::Post | Tier::New => path, line` must be split.
- `src/native/freshness/scan.rs`: `RowCtx::freshness_row` (has `self.task.tags`, the
  `#`-prefixed tag list), `checklist_lints`, `collect_warnings`, `lint_message`.
- `src/native/freshness/cli.rs`: `SCHEMA_VERSION = 15`, `ListedRow`, `collect_list`
  (emits the config-level `freshness_stale_daily_budget_deprecated` diagnostic: the
  pattern for config-level stage diagnostics), `human_list`, `human_row` (PRE detail
  `Checklist · complete to resolve`, POST `Closeout · complete last`), `json_list`.
- `src/native/config/freshness.rs`: `FreshnessConfig`, `parse_freshness_config` (other
  invalid keys are hard config errors), tests including
  `parses_freshness_overrides_and_ignores_unknown_keys` and the fixture-driven
  `keeps_fixture_config_vectors_match_rust`
  (`tests/fixtures/freshness_keeps/vectors.json` is the shared-vector pattern to copy).
- `src/native/freshness/state_tests.rs`: CL1–CL12 checklist vectors.
- Docs: `docs/freshness.md` (§2 config, §4 evaluation and tier comparator table, lints,
  machine vocabulary, §7 `bob freshness`, §10 CL vectors, §13 rollout log),
  `docs/concepts/freshness-and-the-review-walk.md`, `docs/guides/run-morning-review.md`,
  `docs/reference/configuration.md`, `docs/reference/json-outputs.md`,
  `docs/task-tag-marks.md` ("GTD checklist tasks", TI5 "sunrise plus GTD name"),
  `docs/periods.md` ("Calendar and identity" period ids, "CLI so far", "How an agent
  updates a period note"). `src/native/periods/calendar.rs` has `PeriodId::parse` and
  `prev()`.

bob-plugins (linked repo; open it with `/sase_repo`, read its `AGENTS.md`):

- `plugins/bob-ledger-tools/src/090-freshness-config.js` (`coerceFreshnessConfig`: any
  invalid value falls back to the whole default block; stage lists must **not** do
  that), `100-freshness-evaluate.js` (`freshnessChecklistForTags`, rows carry `tags`),
  `110-freshness-queue.js` (`freshnessQueue` comparator, `tierRank`/`tierTotal`),
  `120-freshness-footer.js` (`freshnessReviewEntryView`: PRE detail `checklist`, POST
  `closeout`), `121-freshness-footer-paint.js`, `230-plugin-freshness-api.js` (config
  cache, snapshot lints incl. the deprecation lint), `170-plugin-lifecycle.js`
  (namespace `freshness` **version 13**; the fragment is at 996 of its 1000-line cap),
  `138-checklist-task-marks.js` and `138-task-type-identity.js` (identity run and type
  name), `styles.css`, `scripts/preview-review-footer.cjs`, the
  `scripts/test-ledger-tools-freshness-*.cjs` suites, and `scripts/fixtures/*/README.md`
  (byte-for-byte fixture copies pinned to a bob-cli commit).
- `plugins/bob-navigation-hotkeys/src/480-review-jump-and-nav-api.js` (landing toasts
  are built from ledger `reviewEntryView`: `Review r/N · TIER i/M · detail`),
  `470-keydown-and-freshness.js` (capability gates),
  `535-plugin-review-checklist-walk.js` (`reviewChecklistGroupScan` walks **queue
  order**, `formatReviewCtrlEnterDoneLine`, group-end and "Review closed" notices),
  `536-plugin-review-advance.js`, and `scripts/test-navigation-checklist*.cjs`.

Vault and config:

- `~/bob/gtd_daily.md`: open rows under `### Blocked` (all `[?]` with
  `[scheduled:: 2026-11-06]`), Weekly review `#gtd #post` (scheduled 2026-11-09),
  Quarterly and Yearly reviews `#gtd` only (2027-01-04, 2027-01-11). Done history below.
- `~/.config/bob/config.yml` is chezmoi-managed: source `home/dot_config/bob/config.yml`
  in the `chezmoi` linked repo, `freshness:` block near line 107. bob-ledger-tools reads
  this file directly, so the stage lists are host-wide once applied.
- Vault task `bob.md#^prioritize-pre-gtd` (`[/]`) tracks this work. Leave it for Bryan.

## Design (decided)

### Concept

> **A checklist stage is a named, ordered batch of PRE or POST rows.** A row joins a
> stage by carrying the stage's tag (written right after its checklist tag:
> `#task #gtd #pre #browser`). Config lists each tier's stages in walk order. Staged
> rows walk in that order, ahead of unstaged rows; inside a stage rows keep file order.
> Stages change **order only**: never membership, scope, status, dates, freshness,
> priority, tier order, counts, `by_tier`, `decide`, keeps, Ctrl+Enter semantics, or the
> PRE/POST boundary.

One place to look in each direction: row → stage is the tag on the line; stage →
position is the config list; what you see is the stage name in the editor, footer,
toasts, and CLI.

### Config contract

```yaml
freshness:
  # PRE/POST checklist stages (bob-cli docs/freshness.md §4). A stage is a tag written
  # right after #pre/#post (`#task #gtd #pre #browser`). Stages walk in list order, ahead
  # of unstaged rows; inside a stage, rows keep file order. Names omit '#'.
  pre_stages: [terminal, periods, browser]
  post_stages: [periods]
```

Normalization and validation, identical in Rust and JS (vectors SC1–SC10):

- Absent or `null` → `[]`. `[]` → `[]`. `[]` means today's order for that tier.
- Each entry must be a string. One leading `#` is stripped (so a quoted `"#browser"`
  works), then the name is lowercased (Unicode lowercase on both sides).
- A normalized name must match the tag grammar `^[\p{L}\p{N}_/-]+$`, must not be `task`,
  `gtd`, `pre`, or `post`, and must be unique within its list.
- Any violation (or a non-list value) makes **that list alone** invalid: it is treated
  as `[]` and emits exactly one `freshness_stages_invalid` diagnostic naming the key and
  the first problem. The other list and every other freshness key still apply, the
  freshness block is **not** marked invalid (no exit 2, no JS default-block fallback). A
  typo can never break the walk.

### Evaluator contract (addition to `docs/freshness.md` §4)

```text
stages(k)      = normalized freshness.pre_stages (k = pre) | freshness.post_stages (k = post); [] when absent/invalid
stage(t)       = for a row in checklist_scope with kind k: the FIRST name in stages(k) that t carries
                 as an exact whole-token tag (case-insensitive, the same tag tokens as #pre/#post,
                 anywhere on the line); none otherwise. Always none outside PRE/POST.
stage_index(t) = index of stage(t) in stages(k); +∞ when none
Order (pre)    = stage_index ↑, path ↑, line ↑
Order (post)   = stage_index ↑, path ↑, line ↑
Order (new)    = path ↑, line ↑                      (unchanged: split out of the shared arm)
stage_rank(t)  = 1-based position of t among queue rows of the same tier and stage; none when unstaged
stage_total(t) = number of queue rows of the same tier and stage; none when unstaged
                 (both computed over the full queue, before --limit)
```

Lints (Rust `bob freshness` warnings; JS mirrors only the config-level one):

| Code                       | When                                                                                                          | Path / line        | Message (exact text in `lint_message`)                                    |
| -------------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------ | ------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------- |
| `freshness_stages_invalid` | A stage list is rejected                                                                                      | config path / none | `freshness.<key> <problem>; list ignored, <TIER> keeps file order`        |
| `checklist_stage_conflict` | An open checklist row carries ≥ 2 names from its kind's list                                                  | row                | `checklist row carries several stages; the earliest-listed (<name>) wins` |
| `checklist_stage_unused`   | A listed stage has no open (not done/cancelled) checklist row of that kind carrying its tag, scheduled or not | config path / none | `<pre                                                                     | post> stage '<name>' has no open #gtd #<pre | post> row; check the tag or the list` |

The conflict and unused lints run over every task (`snapshot.all`), like the existing
checklist lints. A future-scheduled `[?]` row still counts as a member, so the unused
lint is quiet on normal days and fires only for a real typo on either side.

### Machine output

- **bob-cli freshness JSON schema 16** (additive; no `!`): `config.pre_stages` and
  `config.post_stages` echo the effective normalized lists; every queue row gains
  `stage` (string or null), `stage_rank`, and `stage_total` (integer or null). The seed
  envelope advances with the shared constant; seed behavior is unchanged.
- **bob-ledger-tools `api.freshness` namespace v14** advertising
  `checklistStages: true`. Queue entries gain `stage`, `stageIndex` (0-based index into
  the tier's list), `stageRank`, and `stageTotal` (null when unstaged);
  `api.freshness.config()` gains `preStages` and `postStages`. A consumer without the
  capability keeps today's behavior; nav already walks whatever order the queue has.

### Surfaces

The rule: every place you already look tells you which batch you are in. No new keys,
chips, counters, tiers, colors, or progress bars. The one hue stays the PRE/POST teal;
the stage name is written in small capitals, and its stage-local count appears only when
the stage has two or more due rows.

**`bob freshness list` (human).** Staged rows replace the generic first word of the
detail with the stage; unstaged rows are unchanged. The tier heading names the
configured order when that tier has rows and a non-empty list. The stage word is
visually distinct from the dim remainder (an existing `Styler` emphasis, NO_COLOR-safe).

```text
  PRE 9 · terminal → periods → browser
    gtd_daily.md:21   Clean up SASE agents on athena …    TERMINAL 1/2 · complete to resolve
    gtd_daily.md:22   Run `task reconcile` + …            TERMINAL 2/2 · complete to resolve
    gtd_daily.md:27   Start last week's review draft …    PERIODS · complete to resolve
    gtd_daily.md:17   Review next week in Google …        BROWSER 1/3 · complete to resolve
    gtd_daily.md:19   Brush teeth                         Checklist · complete to resolve

  ── review closeout ──

  POST 2 · periods
    gtd_daily.md:24   Weekly review: …                    PERIODS · complete to resolve
    gtd_daily.md:15   Morning review: …                   Closeout · complete last
```

**Review footer and landing toasts.** `freshnessReviewEntryView` returns the stage as
the detail (`TERMINAL 1/2`, or `PERIODS` when the stage has one row) and compact form
(`TERMINAL`). Landing toasts in nav are already built from this view, so they follow
with no nav change. Staged POST rows use the PRE action hint
(`Ctrl+Enter done · ]s skip`) because they do not close the review; unstaged POST rows
keep `closeout · … closes the review`.

```text
⟳ Review 1/38 · PRE 1/9 · TERMINAL 1/2 · Ctrl+Enter done · ]s skip
⟳ Review 3/38 · PRE 3/9 · PERIODS · Ctrl+Enter done · ]s skip
⟳ Review 7/38 · PRE 7/9 · checklist · Ctrl+Enter done · ]s skip          (unstaged)
⟳ Review 37/38 · POST 1/2 · PERIODS · Ctrl+Enter done · ]s skip
⟳ Review 38/38 · POST 2/2 · closeout · Ctrl+Enter done · closes the review
```

**Stage-end tick (nav).** When Ctrl+Enter (or Ctrl+Alt+F) completes the last live due
row of its stage, the done line gains ` · <STAGE> ✓` right after the task text, and the
rest of the line is unchanged:
`✓ Done · Read email · BROWSER ✓ · Ctrl+Enter → next PRE`. The walk never pauses at a
stage boundary; the PRE/POST boundary rule stays the only checklist stop.

**Editor stage label.** A configured stage tag written immediately after the checklist
identity run renders as a quiet small-caps label after the sunrise or flag (after a type
name when one is shown, TI5), in the identity teal, slightly quieter than the glyph:

```text
before: - [?] ☀ #terminal Clean up SASE agents on athena and apollo: …
after:  - [?] ☀ terminal  Clean up SASE agents on athena and apollo: …   (small caps)
        - [?] ⚑ periods   Weekly review: …
        - [?] ☀ #brower   Check weather                                   (typo: stays a pill)
```

- Eligible only when the existing PRE/POST identity run is recognized, followed by one
  whitespace run and an exact whole-token tag equal to `stage(t)` as the evaluator
  resolves it from the configured list. Anything else (unknown or mistyped tag, a tag
  elsewhere on the line, a conflict whose adjacent tag is not the winner, a name from
  the other tier's list, no config) keeps the native pill, and that plain look is the
  typo signal.
- Tooltip: line 1 `#terminal · PRE stage 1 of 3`; line 2 the tier's order with the
  current stage capitalized: `TERMINAL → periods → browser → unstaged`. Accessible label
  `PRE stage 1 of 3: terminal`. Never contains `::`.
- Live Preview: the existing run widget extends over the stage tag; touching any part
  reveals all raw tags. Reading view: hide the stage anchor in place (it stays in the
  DOM for copy and Dataview round trips) and render the label. Tasks query rows keep the
  native pill (CSS cannot know the config; no JS touches Tasks' DOM). The existing
  "Toggle task tag marks" command restores the raw pill. Closed rows fade exactly like
  the glyph. The label refreshes when the cached freshness config changes; no reload is
  required beyond what the config cache already needs.

### Relative period ids

`bob periods show|brief|draft` accept `last-week`, `last-quarter`, and `last-year`
(case-insensitive; `_` accepted for `-`, mirroring `all_time`/`all-time`), resolved
against today (`BOB_NOW`-aware) to the period of that kind immediately before the one
containing today: the previous ISO week, the previous 13-week quarter, the previous ISO
week-year. Every output names the canonical id (`2026-W45`); human `show` output must
name the vault-relative note path near the top (add it if missing). Refusals
(`refused_open`, `refused_state`) and all other behavior are exactly those of the
canonical id. Unknown relative words (`last-month`, `next-week`) are usage errors
(exit 2) whose message lists the accepted forms including the relative ones. No new
subcommands or flags.

### Vault rows (exact)

Edit only open (first) occurrences in `~/bob/gtd_daily.md`; leave Done history alone.
Tags go right after the checklist tag.

| Row                                                              | Change                                                                                                                                                                                                                                                                                                           |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Clean up SASE agents on athena and apollo                        | `#gtd #pre` → `#gtd #pre #terminal`                                                                                                                                                                                                                                                                              |
| Run `task reconcile` + `gcal sync` + `gkeep pull`                | add `#terminal`                                                                                                                                                                                                                                                                                                  |
| Review next week in Google Calendar · Check weather · Read email | add `#browser`                                                                                                                                                                                                                                                                                                   |
| Brush teeth · fish oil + L-tyrosine · morning stretches          | unchanged (unstaged)                                                                                                                                                                                                                                                                                             |
| Morning review (`#post`)                                         | unchanged (unstaged, so always last)                                                                                                                                                                                                                                                                             |
| Weekly review (`#post`)                                          | add `#periods`; replace `no Review yet? launch an agent with "Update YYYY/YYYY-Www.md", correct its draft, and write the Reflection` with ``correct the draft PRE started (still empty? launch "Update the last-week period note (`bob periods show last-week` prints its path)" now) and write the Reflection`` |
| Quarterly review                                                 | replace `launch an agent with "Update YYYY/YYYY-Qn.md", correct its draft, write the Reflection` with ``correct the draft PRE started (still empty? launch "Update the last-quarter period note (`bob periods show last-quarter` prints its path)" now), write the Reflection``; no tags                         |
| Yearly review                                                    | replace `launch an agent with "Update YYYY.md", correct its draft, write the Reflection` with ``correct the draft PRE started (still empty? launch "Update the last-year period note (`bob periods show last-year` prints its path)" now), write the Reflection``; no tags                                       |

Three new rows, appended as `[ ]` beside their siblings in the open list (the
task-status hooks derive `[?]` while their date is in the future). `<today>` is the edit
date; each `scheduled` copies its paired review row's **current open** `scheduled` date
(today 2026-11-09, 2027-01-04, 2027-01-11):

```markdown
- [ ] #task #gtd #pre #periods **Start last week's review draft**: launch an agent with
      "Update the last-week period note (`bob periods show last-week` prints its path)"
      · done once the agent runs, or when last week's Review is already drafted or
      edited. [repeat:: every week on Monday when done] [created:: <today>] [scheduled::
      <Weekly review's scheduled>]
- [ ] #task #gtd #pre #periods **Start last quarter's review draft**: launch an agent
      with "Update the last-quarter period note (`bob periods show last-quarter` prints
      its path)"; when last week's draft is due too, one agent can do both, week first ·
      done once the agent runs, or when the Review is already drafted or edited.
      [repeat:: every 3 months on the 1st Monday when done] [created:: <today>]
      [scheduled:: <Quarterly review's scheduled>]
- [ ] #task #gtd #pre #periods **Start last year's review draft**: finish the Quarterly
      review first if it is still open (the year brief reads the Q4 Review), then launch
      an agent with "Update the last-year period note (`bob periods show last-year`
      prints its path)" · done once the agent runs, or when the Review is already
      drafted or edited. [repeat:: every January on the 2nd Monday when done] [created::
      <today>] [scheduled:: <Yearly review's scheduled>]
```

Notes that the docs must not contradict:

- `when done` keeps the rows lint-clean (`checklist_repeat_not_when_done`), and Tasks
  re-anchors at the completion day, so completing a row in its due month yields the
  review row's next date. The pair share a calendar and seed date but stay two
  independent series; nothing promises lockstep.
- The rows never name a concrete period; `last-*` resolves it, and `bob periods draft`
  refusing open periods and edited Reviews stays the safety net.
- Stage tags are stripped from habit identity (`chore_identity`), so tagging keeps every
  chore's history. Rewording the three review rows starts new habit series for them;
  accepted.

## Conformance vectors

All three vector families live in **one shared fixture**,
`tests/fixtures/freshness_stages/vectors.json` in bob-cli (the Rust suite is the source
of truth; every case lists its `runners`, and lint-only cases are `rust` only).
bob-ledger-tools copies it byte-for-byte to
`plugins/bob-ledger-tools/scripts/fixtures/freshness_stages/` with a README pinning the
bob-cli commit, like the existing fixture READMEs. Unless stated, `path` is
`gtd_daily.md`, today is 2026-11-09, `pre_stages: [terminal, periods, browser]`,
`post_stages: [periods]`, rows are open `[?]` recurring `when done` rows scheduled on or
before today.

Config (SC):

| #    | YAML                                                     | Expect                                                              |
| ---- | -------------------------------------------------------- | ------------------------------------------------------------------- |
| SC1  | no `freshness` block                                     | `pre [] · post []`, no diagnostic                                   |
| SC2  | `pre_stages: null`                                       | `[]`, no diagnostic                                                 |
| SC3  | `pre_stages: []`                                         | `[]`, no diagnostic                                                 |
| SC4  | `pre_stages: [Terminal, "#periods", browser]`            | `[terminal, periods, browser]`                                      |
| SC5  | `pre_stages: browser`                                    | pre invalid → `[]`, one `freshness_stages_invalid`; post unaffected |
| SC6  | `pre_stages: [gtd]` (also `task`, `pre`, `post`, `#PRE`) | invalid, reserved                                                   |
| SC7  | `pre_stages: [browser, Browser]`                         | invalid, duplicate                                                  |
| SC8  | `["web browser"]`, `[""]`, `["#"]`, `[3]`, `[{a: 1}]`    | invalid (grammar / empty / non-string)                              |
| SC9  | `pre_stages: [gtd]` + `post_stages: [periods]`           | pre `[]` + one diagnostic; post `[periods]`                         |
| SC10 | invalid `pre_stages` + `interval: 3`                     | interval 3 kept; freshness block not invalid (JS `invalid: false`)  |

Queue (CS):

| #    | Setup                                                                                      | Expect                                                                                        |
| ---- | ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| CS1  | L1 `#gtd #pre Brush`; L9 `#gtd #pre #browser Weather`                                      | L9, L1                                                                                        |
| CS2  | L1 `#browser`; L2 `#terminal`                                                              | L2, L1 (config beats line)                                                                    |
| CS3  | `#gtd #pre #browser #terminal X`                                                           | stage `terminal` (earliest-listed, not first-written); lint `checklist_stage_conflict` (rust) |
| CS4  | CS2 rows, no stage config                                                                  | L1, L2; every `stage` null (regression)                                                       |
| CS5  | `#Browser` / `#browsers` / `#browser/x` / `#web/browser` / `#brower`                       | `browser` / none / none / none / none                                                         |
| CS6  | PRE `#periods`, POST `#periods`, POST `#terminal`                                          | PRE periods 1/1; POST periods 1/1; POST `#terminal` unstaged (not in post list)               |
| CS7  | `- [ ] #task #browser Ready thing` (no fresh)                                              | tier `new`, stage null, no lint; identical to the row without the tag                         |
| CS8  | `pre_stages: [gtd]`, rows as CS2                                                           | PRE in path/line order; one diagnostic; POST stages still apply                               |
| CS9  | `#gtd #pre #terminal` scheduled 2026-11-10                                                 | absent from the queue; still counts as a member (no `checklist_stage_unused` for terminal)    |
| CS10 | NEW rows in `a.md`/`b.md` beside staged PRE rows                                           | NEW order byte-identical to the old comparator                                                |
| CS11 | L2 periods, L3 terminal, L5 terminal, L4 and L6 unstaged                                   | L3 terminal 1/2, L5 terminal 2/2, L2 periods 1/1, L4, L6 (rank/total null)                    |
| CS12 | `a.md:1` unstaged PRE; `b.md:9` `#browser` PRE                                             | `b.md:9` first (stage beats path)                                                             |
| CS13 | L15 `#gtd #post Morning review`; L24 `#gtd #post #periods Weekly review`                   | L24, L15                                                                                      |
| CS14 | `pre_stages: [terminal, periods, brower]`, no open row carries `#brower` (a done row does) | one `checklist_stage_unused` for pre `brower` (rust)                                          |
| CS15 | staged `[ ]`, `[*]`, and `[?]` PRE rows                                                    | sorted by stage like each other; lanes unchanged                                              |
| CS16 | `#task #gtd #pre Weather #browser` (tag at the end)                                        | stage `browser` in the walk (labels are a display rule; see SL4)                              |

Period ids (PI, bob-cli only):

| #   | `BOB_NOW`                        | Expect                                                                         |
| --- | -------------------------------- | ------------------------------------------------------------------------------ |
| PI1 | 2026-11-09 (Mon, 2026-W46)       | `last-week` 2026-W45 · `last-quarter` 2026-Q3 · `last-year` 2025               |
| PI2 | 2027-01-04 (Mon, 2027-W01)       | `last-week` 2026-W53 · `last-quarter` 2026-Q4                                  |
| PI3 | 2027-01-11                       | `last-week` 2027-W01 · `last-year` 2026                                        |
| PI4 | 2027-01-03 (Sun, still 2026-W53) | `last-week` 2026-W52 · `last-quarter` 2026-Q3 · `last-year` 2025               |
| PI5 | any                              | `LAST-WEEK` and `last_week` accepted                                           |
| PI6 | any                              | `last-month`, `last-weeks`, `next-week` → exit 2, message lists accepted forms |
| PI7 | 2026-11-09                       | `draft last-week` behaves exactly as `draft 2026-W45` (incl. `refused_state`)  |

Footer (FT, ledger-tools) and labels (SL, ledger-tools; recorded in
`docs/task-tag-marks.md`) and nav (NT) vectors are listed in their phases.

## Phase `walk_order` — checklist stages in the bob-cli evaluator, CLI, and contract

1. Config: add `pre_stages`/`post_stages` to `FreshnessConfig` (normalized
   `Vec<String>`) plus the per-key invalid reasons; parse per the config contract
   without ever returning `ConfigError` for them. Keep the existing unknown-keys test
   green and add SC1–SC10 via the shared fixture
   (`tests/fixtures/freshness_stages/vectors.json`, schema of your choosing, documented
   in a short `README.md` beside it; rows describe tags, status, path, line, dates, and
   recurrence neutrally so both runners can build their inputs).
2. Evaluator: carry the row's lowercased tags (or the resolved stage) into the
   evaluator, resolve `stage(t)` only for checklist-scope rows, and add `stage`,
   `stage_index`, `stage_rank`, `stage_total` to `QueueEntry`. Split `Tier::New` out of
   the PRE/POST comparator arm; PRE/POST sort by `stage_index` (none last), path, line.
   Compute ranks and totals after sorting, before any `--limit`. Run CS1–CS16 from the
   fixture in `state_tests.rs` (or a sibling test module), and keep CL1–CL12 untouched
   and green.
3. Lints: `checklist_stage_conflict` per row in `collect_warnings`;
   `freshness_stages_invalid` and `checklist_stage_unused` once each per key/stage in
   `collect_list`, beside the deprecation diagnostic; exact messages in `lint_message`.
4. Output: `SCHEMA_VERSION = 16` (update the schema history comment), JSON fields per
   "Machine output", the human detail and tier-heading suffix per "Surfaces". Update any
   tests or fixtures pinned to schema 15.
5. Docs (contract first, then only what became incorrect): `docs/freshness.md` §2
   (keys), §4 (the contract block, the comparator table rows for pre/post, lints,
   machine vocabulary schema 16 and schema history), §7, §10 (a "Checklist stages
   (CS1–CS16, SC1–SC10)" vector list), §13 rollout log line;
   `docs/concepts/freshness-and-the-review-walk.md` (PRE/POST rows walk in stage order);
   `docs/guides/run-morning-review.md` (a short "Order PRE and POST with stages" step:
   tag a row, list the stage, reorder by editing the list);
   `docs/reference/configuration.md`; `docs/reference/json-outputs.md`. Run
   `just docs-check` and the full Rust test/lint gates the repo uses.
6. Commit as `feat(freshness): …` (additive; no `!`). Never edit `Cargo.toml`'s version
   or `CHANGELOG.md` (release train).

## Phase `period_ids` — relative period ids in `bob periods`

1. Add a resolver used by `show`, `brief`, and `draft` (the CLI layer knows today via
   the existing `BOB_NOW`-aware clock) that maps `last-week|last-quarter|last-year`
   (case- and `_`/`-`-insensitive) to the previous period of that kind via
   `PeriodId::prev()` of the period containing today, and otherwise defers to
   `PeriodId::parse`. Extend the invalid-id usage message to list the relative forms.
2. Ensure human `bob periods show` names the canonical id and vault-relative note path
   at the top (add the path if it is missing); JSON already carries ids/paths, so verify
   rather than change.
3. Tests: PI1–PI7. Docs: `docs/periods.md` "Period ids" (relative forms and their
   resolution), "CLI so far" mentions, and one sentence in "How an agent updates a
   period note" (a relative id resolves to the canonical one; `bob periods show <id>`
   prints the note's path). Update the PERIOD argument help text so `-h` shows the
   accepted forms, run `just docs-gen` and `just docs-check` (read
   `sase memory read cli_rules.md` and `docs.md` first).
4. Commit as `feat(periods): …`.

## Phase `ledger_order` — checklist stages in bob-ledger-tools

Work in the bob-plugins linked repo (`/sase_repo`), from its latest master.

1. Copy `tests/fixtures/freshness_stages/vectors.json` byte-for-byte from bob-cli master
   to `plugins/bob-ledger-tools/scripts/fixtures/freshness_stages/vectors.json`, with a
   README naming the source commit (pattern:
   `scripts/fixtures/pomodoro_themes/README.md`).
2. Config (`090`): add `preStages`/`postStages` normalized per the config contract; an
   invalid list yields `[]` plus a `stagesInvalid` record, and never sets the
   block-level `invalid` or triggers the default-block fallback. Snapshot lints (`230`)
   emit `freshness_stages_invalid` the way they emit the deprecation lint.
3. Evaluator/queue (`100`, `110`): a shared pure helper
   `freshnessChecklistStage(tags, kind, config)` → `{ stage, stageIndex }`; the PRE/POST
   comparator becomes `stageIndex ↑ (none last), path ↑, line ↑`, NEW unchanged; entries
   gain `stage`, `stageIndex`, `stageRank`, `stageTotal` (null when unstaged). Run every
   fixture case whose `runners` include `js` in a new
   `scripts/test-ledger-tools-freshness-stages.cjs` (add it to the `npm test` list).
4. Namespace (`170`): `freshness` version 14 with `checklistStages: true`; update the
   namespace comment tersely; the fragment must stay ≤ 1000 lines (move comment text to
   `230` if needed). `api.freshness.config()` exposes `preStages`/`postStages`. Update
   the namespace test.
5. Footer (`120`/`121`): implement FT1–FT6 below in `freshnessReviewEntryView` and its
   tests; paint the stage text in the tier's existing color (no new hue). Render
   `scripts/preview-review-footer.cjs` with a staged Monday queue and inspect it in
   light and dark.

   | #   | Entry                                          | Footer                                                               |
   | --- | ---------------------------------------------- | -------------------------------------------------------------------- |
   | FT1 | PRE, stage terminal, stageRank 1, stageTotal 2 | `⟳ Review 1/38 · PRE 1/9 · TERMINAL 1/2 · Ctrl+Enter done · ]s skip` |
   | FT2 | PRE, stage periods, total 1                    | `… · PERIODS · Ctrl+Enter done · ]s skip`                            |
   | FT3 | PRE unstaged                                   | `… · checklist · Ctrl+Enter done · ]s skip` (unchanged)              |
   | FT4 | POST, stage periods                            | `… · POST 1/2 · PERIODS · Ctrl+Enter done · ]s skip`                 |
   | FT5 | POST unstaged                                  | `… · closeout · Ctrl+Enter done · closes the review` (unchanged)     |
   | FT6 | Narrow footer, staged row                      | compact `TERMINAL` (count dropped first)                             |

6. Docs: bob-plugins `docs/bob-ledger-tools.md` (footer section and the namespace), the
   plugin manifest description if it lists footer behavior, `npm run docs:gen` /
   `docs:check`; bob-cli `docs/freshness.md` §13 rollout line and the "Bob-ledger-tools
   keeps top-level api v3 …" paragraph (namespace v14, `checklistStages`; fix its stale
   "version to 12" while there). Bump the plugin version per repo convention,
   `npm run build`, `npm test`, then `bob plugins sync`.

## Phase `stage_labels` — stage labels beside the sunrise and flag

1. Contract first: add "Checklist stage labels" under "GTD checklist tasks" in bob-cli
   `docs/task-tag-marks.md` with the label rules from "Editor stage label" and these
   vectors (config as in the Conformance section):

   | #    | Line                                            | Expected                                                                                                                                    |
   | ---- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
   | SL1  | `- [?] #task #gtd #pre #terminal Clean up`      | ☀ + `terminal` label; tooltip `#terminal · PRE stage 1 of 3` / `TERMINAL → periods → browser → unstaged`; aria `PRE stage 1 of 3: terminal` |
   | SL2  | `- [?] #task #gtd #post #periods Weekly review` | ⚑ + `periods`; `POST stage 1 of 1`; `PERIODS → unstaged`                                                                                    |
   | SL3  | `- [?] #task #gtd #pre #brower Weather`         | ☀ + native `#brower` pill                                                                                                                   |
   | SL4  | `- [?] #task #gtd #pre Weather #browser`        | ☀ only; native pill at the end                                                                                                              |
   | SL5  | `- [?] #task #gtd #pre #browser #terminal X`    | ☀ only; both pills native (adjacent tag is not the winner)                                                                                  |
   | SL6  | SL1 with no stage config                        | ☀ + native `#terminal` pill                                                                                                                 |
   | SL7  | SL1 plus `[type:: GTD]`                         | ☀ + `GTD` type name + `terminal` label                                                                                                      |
   | SL8  | "Toggle task tag marks" off                     | raw `#task #gtd #pre #terminal` pills                                                                                                       |
   | SL9  | SL1 as `[x]`                                    | label fades with the glyph                                                                                                                  |
   | SL10 | Live Preview cursor touches the run             | all four raw tags revealed                                                                                                                  |
   | SL11 | SL1 inside a Tasks query result                 | native `#terminal` pill                                                                                                                     |
   | SL12 | `#task #gtd #post #terminal`                    | ⚑ + native pill (terminal is a PRE stage only)                                                                                              |

2. Implement in bob-ledger-tools (`138-checklist-task-marks.js` and its Live Preview /
   Reading view hosts; reuse `freshnessChecklistStage` and the cached freshness config),
   with unit tests for SL1–SL12 in `scripts/test-ledger-tools-task-tag-marks.cjs` (or a
   sibling suite).
3. Style in `styles.css`: small caps (`font-variant-caps: all-small-caps`), the identity
   teal at reduced emphasis, gentle letter-spacing, a stable left edge so stacked staged
   rows align; legible on the red Blocked `[?]` tint and in light and dark themes
   (assert contrast if an existing contrast test pattern fits). Verify visually with the
   existing isolated real-Obsidian e2e harness pattern (`scripts/obsidian-*-e2e*.cjs`)
   or a preview script, light and dark, and record what you checked in the contract.
4. bob-plugins docs, version bump, `npm run build`, `npm test`, `bob plugins sync`;
   bob-cli commit for the contract (`docs(marks): …`).

## Phase `nav_ticks` — stage-end ticks and staged-advance regression tests

1. In bob-navigation-hotkeys, gate on `checklistStages` entries carrying `stage`. When a
   Ctrl+Enter or Ctrl+Alt+F completion on a PRE/POST landing leaves no live due row of
   the same tier and stage (`liveGroup.after ∪ liveGroup.before`), add ` · <STAGE> ✓`
   after the task text in the done line (every PRE/POST done-line formatter: `535`, and
   `536` if it formats checklist completions). Group-end, "N POST left", and "Review
   closed" notices are otherwise unchanged.
2. Vectors in `scripts/test-navigation-checklist-ctrl-enter.cjs` (or a sibling):

   | #   | Case                                                    | Expect                                                                    |
   | --- | ------------------------------------------------------- | ------------------------------------------------------------------------- |
   | NT1 | last live BROWSER row completed                         | `✓ Done · Read email · BROWSER ✓ · Ctrl+Enter → next PRE`                 |
   | NT2 | another live BROWSER row remains                        | no tick                                                                   |
   | NT3 | next staged row sits above the cursor in the file       | advance lands on the queue successor above, not the next line             |
   | NT4 | namespace v13 entries (no `stage`)                      | today's done lines exactly                                                |
   | NT5 | staged Weekly review completed with Morning review left | `✓ Done · Weekly review · PERIODS ✓`, then the existing POST flow         |
   | NT6 | `]s` landing on a staged row                            | toast head `Review 1/38 · PRE 1/9 · TERMINAL 1/2` (via `reviewEntryView`) |

3. bob-plugins docs (`docs/bob-navigation-hotkeys.md` toast section), version bump,
   build, test, `bob plugins sync`; bob-cli `docs/freshness.md` §6 one sentence on the
   tick and §13 rollout line.

## Phase `rollout` — config, vault rows, verification, and memory

1. Config: in the `chezmoi` linked repo (`/sase_repo`), add the commented
   `pre_stages`/`post_stages` lines from "Config contract" to the `freshness:` block of
   `home/dot_config/bob/config.yml`, then commit and push there per its conventions. On
   this host first, then on the other hosts (read `sase memory read tailnet.md` first),
   pull the chezmoi source without forcing and apply **only** the
   `~/.config/bob/config.yml` target (`chezmoi apply ~/.config/bob/config.yml`). Never
   overwrite local changes; report any host you could not update.
2. Vault: make the edits in "Vault rows (exact)" following the vault's `AGENTS.md`
   commit rules. Confirm the new rows become `[?]` after the next task-status hooks run
   (or run the hooks per `docs/task-status-hooks.md`).
3. Verify from a bob-cli checkout at master (the installed `bob` lags until the daily
   release): `BOB_NOW=2026-11-09T07:00:00 cargo run --quiet -- freshness list` and its
   `-f json` form show PRE as terminal (cleanup, reconcile; file order) → periods (week
   launch) → browser (calendar, weather, email; file order) → unstaged (teeth, pills;
   stretches absent until 11-11), POST as Weekly review → Morning review, `stage` fields
   set, and no stage lints. Repeat with `BOB_NOW=2027-01-04T07:00:00` (week + quarter
   launches in PERIODS) and `2027-01-11` (week + year). Run
   `cargo run --quiet -- periods show last-week` and check the path. In Obsidian on this
   host, confirm the footer and labels on the open rows if a session is available;
   otherwise say so. Do not complete any `gtd_daily.md` row and leave
   `bob.md#^prioritize-pre-gtd` for Bryan, whose first staged walk is the acceptance.

> [!decision] decision_record Use `/sase_memory_write`, then add a new `decisions`
> record, e.g. `checklist-stages-order-pre-and-post` ("Checklist Stages Order PRE And
> POST"). Roster rule: tagged checklist stages (`freshness.pre_stages`/`post_stages`)
> order PRE and POST rows by stage rank, then path and line, unstaged last, without
> changing membership, counts, or the Ctrl+Enter boundary; never `[priority::]` or
> inline fields. Body: applies to bob-cli, bob-plugins, vault, config; the claim
> (including SASE-first, period-draft launch rows paired by date, Quarterly/Yearly
> staying RECURRING, relative period ids); rejected alternatives (`[priority::]`,
> `[review_order::]`/`[band::]` fields and the F4 parser hazard, reordering the file,
> sub-tiers or a new walk tier, `dependsOn` chains, launch-first, auto-launch); cost;
> reopen triggers from "Reopen triggers" below; evidence (the research ref, this plan,
> the phase commits). Link `[[decisions/review-walk-is-tiered]]` and
> `[[decisions/recurring-occurrences-walk-when-due]]`; nothing is superseded (their
> comparators are by reference to `docs/freshness.md`). Run `sase memory init` and any
> docs regeneration `just docs-check` then requires.

> [!decision] glossary_term If accepted: add a `glossary` strand "Checklist Stage" (a
> named, configured batch of PRE or POST rows set by a tag right after `#pre`/`#post`;
> order only; shown as a small-caps label, footer/toast detail, and CLI detail; not a
> priority, tier, or lane), linked from nothing else; `sase memory init`, then
> `just docs-check`. If declined, file nothing.

## Cross-cutting rules for every phase

- Start each phase from the latest master of every repo it touches; other epics may be
  in flight in the same files. Regenerate generated files (`main.js`, docs inventory)
  rather than hand-merging them.
- bob-cli ships through the daily release train: never edit the `Cargo.toml` version or
  `CHANGELOG.md`; these changes are additive, so no `!`.
- bob-plugins: edit `src/` fragments only, keep each fragment ≤ 1000 lines,
  `npm run build`, `npm test`, `npm run docs:check`, and always `bob plugins sync` after
  changes.
- Rust and JS must agree on every shared fixture case; when they disagree, the bob-cli
  contract wins and the fixture copy is re-synced, never edited.

## What not to build (restraint)

No new walk tier or sub-tier, no inline field, no `[priority::]` use, no per-stage
colors, chips, counters, or progress bars, no stage support outside PRE/POST, no
within-stage tie-break, no pause at stage boundaries, no auto-launch of agents, no stage
shown in Tasks query rows, no change to Bob Mac Capture.

## Costs

- One more config concept and a shared fixture kept green across two evaluators.
- A mistyped stage tag drops its row to the end of its tier; the missing label and
  `checklist_stage_unused` are the signals.
- Order inside a stage still follows the file and can swap day to day (A9).
- Launch rows add due chores on review Mondays; launching moves a few minutes later than
  first requested (A2); rewording the review rows restarts their habit series.

## Reopen triggers

- The launch rows become rubber stamps (say 8 of 10 review Mondays): automate behind
  SASE launch approval.
- Within-stage reshuffling starts to bother Bryan: add a normalized-text tie-break in
  both evaluators, or make the status grouper append moved blocks.
- SASE cleanup routinely starves the drafts: split "update SASE on all machines" into
  its own first `#terminal` row.
- A stage list grows past about five entries, or stages are wanted in other tiers.
- A whole stage is `]s`-skipped on most mornings.
- A new unstaged POST row starts walking after the Morning review: allow a "rest" slot
  in `post_stages`.
