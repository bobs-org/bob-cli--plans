---
tier: tale
title: Unify reference-task freshness and remove the REFERENCES review tier
goal:
  Ref tasks use ordinary lane freshness and review groups, with the reference interval
  removed across CLI, plugins, and config.
size: medium
decisions:
  memory_reference_policy:
    ask: Record the ordinary reference-task freshness policy in a new decision note?
    default: false
    memory:
      - decisions:reference-tasks-use-ordinary-freshness
    answer: false
  memory_walk_supersession:
    ask:
      Mark the review-walk decision partly superseded for REFERENCES, linking the new
      policy?
    default: false
    memory:
      - decisions:review-walk-is-tiered
    answer: false
  memory_freshness_glossary:
    ask:
      Update the task-freshness glossary for ordinary ref freshness and the resulting
      review order?
    default: false
    memory:
      - glossary:task-freshness
    answer: false
proposed_by: bbugyi200.apollo.65
decided_by: auto
create_time: 2026-10-10 06:21:05
status: wip
---

# Unify reference-task freshness and remove the REFERENCES review tier

## Outcome and scope

Bryan requests removal of `freshness.reference_interval` and the REF group from the
morning GTD review started with `]s`. Ref tasks must follow the same freshness and
review rules as other tasks in their lane. Implement this as one coordinated change
across bob-cli, linked bob-plugins, and the managed Bob config in chezmoi. This is a
medium tale: the behavior is bounded and one coding agent can implement and verify both
evaluators together without a phase handoff.

The group is named `references` in machine output, `REFERENCES` in CLI/navigation, and
`REFS` in the footer. Remove that review group, including its counters and interval
source. References remain valid tasks and library objects; the separate dashboard
REFERENCES collection, reference links/tags, import/sync lifecycle, and reference task
placement are outside this change.

## Findings and implementation entry points

The repositories were inspected read-only before writing this plan. At that time:

- `src/native/config/freshness.rs` defines and parses the optional reference interval;
  unknown config keys are already ignored.
- `src/native/freshness/state.rs::evaluate_without_checklist` treats only **Ready** refs
  as special trackers. Pending/Next refs already use ordinary lane intervals. Ready refs
  divert NEW/RESURFACED/ROTTEN states into `Tier::References`, excluding them from
  ordinary keep/decay eligibility. `scan.rs` assigns tracker identity.
- `src/native/freshness/cli.rs` emits schema 12, including `config.reference_interval`,
  `counts.references_due`, `by_tier.references`, and interval source `reference`. The
  seed envelope shares the schema constant.
- The bob-ledger-tools JavaScript mirror lives in source fragments
  `090-freshness-config.js`, `100-freshness-evaluate.js`, `110-freshness-queue.js`,
  `120-freshness-footer.js`, `130-freshness-marks.js`, `170-plugin-lifecycle.js`, and
  `230-plugin-freshness-api.js`. `api.freshness` is version 10; the enclosing API is
  version 3.
- bob-navigation-hotkeys uses the shared queue and entry presentation. Its
  `470-keydown-and-freshness.js` and `480-review-jump-and-nav-api.js` also recognize the
  references tier for notices and commitment boundaries.
- chezmoi's `home/dot_config/bob/config.yml` contains `reference_interval: 7` next to
  the ordinary interval and lane settings.
- Existing tests cover tracker precedence, ref tag/anchor identity, hidden refs, tier
  ordering/counts, recurring overlays, footer groups, and keep eligibility. Extend those
  suites with ref/ordinary equivalence tests, rather than only changing old expected
  strings.

Open `bob-plugins` and `chezmoi` with `/sase_repo` in the implementing agent's own
session and use only the paths it returns. Read their `AGENTS.md` instructions. All
plugin paths below are relative to the opened bob-plugins checkout. Plugins with
`src/fragments.json` require source edits and `npm run build`; never edit generated
`main.js` directly. Source fragments must remain at most 1000 lines.

## Behavior contract

### Intervals and config removal

1. For any ordinary Ready task, including `#ref` and legacy `^ref` tasks, keep
   precedence `[refresh:: N]` → note `task_refresh` → `freshness.interval` → built-in 7
   days. The request to use `freshness.interval` means the same global default as
   ordinary tasks, retaining their existing local overrides.
2. Pending refs use `freshness.pending_interval`; Next refs use
   `freshness.next_interval`. Enabled lane intervals override the Ready chain;
   absent/null still defaults to 1. A lane configured `false` is not walked and follows
   the existing ordinary-task fallback metadata rules, without turning the ref into a
   Ready task.
3. Remove the reference field from Rust config, JavaScript normalized/default/
   fallback/public config, active documentation, and the managed personal config. Stop
   recognizing both the snake_case key and the tolerated JavaScript `referenceInterval`
   spelling. A leftover key in a syntactically valid config is ignored under the
   existing unknown-key rule, even if its value would previously have been invalid. It
   never overrides an interval or triggers the plugin's invalid-config fallback. Do not
   introduce an alias, migration write, replacement knob, or special deprecation parser
   for it.
4. Preserve `project_interval`, exact `^prj` precedence, PROJECTS review in all lanes,
   and project visibility rules. A row with `^prj` and `#ref` still follows the existing
   project identity precedence.

### Review grouping and consequences

For visible, eligible non-checklist/non-recurring refs:

| Ref task condition                                | Review group | Freshness state/bucket |
| ------------------------------------------------- | ------------ | ---------------------- |
| Ready with no valid confirmation                  | NEW          | new / new              |
| Ready with an arrived schedule after confirmation | TICKLER      | resurfaced / rotten    |
| Ready at or beyond its effective interval         | ROTTEN       | rotten / rotten        |
| Ready still fresh                                 | none         | fresh / null           |
| Due Pending                                       | PENDING      | null / null            |
| Due Next                                          | NEXT         | null / null            |

The complete order becomes **PRE → NEW → PROJECTS → PENDING → NEXT → RECURRING → TICKLER
→ ROTTEN → POST**, with the existing comparator for each surviving tier. Refs receive no
extra sorting key or subgroup. Existing PRE/POST tag precedence and recurring-occurrence
rules apply to refs too. Preserve hidden, blocked, Today-linked, daily-note,
template/conflict, and future-scheduled exclusions and the existing checklist
exceptions. An ordinary hidden `^ref` remains hidden.

Due Ready refs in TICKLER/ROTTEN now have ordinary keep counting and `decide`
eligibility. At the keep limit an explicit review gesture opens the existing decision
card; nothing decays or changes priority/schedule automatically. NEW, Pending/Next,
PROJECTS, recurring, and checklist rows keep their existing eligibility rules. Reuse the
existing helpers and card paths.

ROTTEN refs are upkeep, while NEW/TICKLER and the other commitment tiers count as
commitments. Update boundary calculations accordingly. Whole-queue counts and per-tier
ranks must remain correct before `--limit`; no ref can disappear or count twice. Keep
state counts separate from tier counts and preserve the existing upkeep meter and budget
formula.

This change is evaluated at read time. Preserve existing stamps, refresh overrides,
keeps, statuses, task bodies, and links. There is no vault-wide task rewrite and the
freshness seed is not rerun.

## Implementation sequence

### 1. Change the Rust evaluator and machine contract

- Remove `reference_interval` from `FreshnessConfig`, defaults, parsing, and
  constructors. Retain validation for project and ordinary interval settings.
- Make project identity the only freshness tracker exception. Delete reference
  override/tier logic in `state.rs`; simplify `TrackerKind`, identity helpers, and
  `scan.rs` where the now-unused ref distinction can be removed. Keep unrelated
  reference-library identity handling intact.
- Remove `Tier::References`, `IntervalSource::Reference`, reference histogram and
  convenience-count fields, their sort arms, and all active human render branches. Route
  refs through the ordinary evaluator, including `decide_for`.
- Remove reference config/count/source fields from JSON and the human REVIEW summary,
  group headings, and help. `by_tier` must contain exactly the nine surviving keys,
  including zero values.
- Advance freshness JSON schema **12 → 13** for this breaking removal and
  redistribution. Update list and shared seed-envelope assertions; do not alter seed
  behavior. If the baseline has advanced before implementation, use its next schema
  version and document the reason. No CLI subcommands/options are added.

### 2. Mirror the contract in Obsidian and remove the group presentation

- Apply the same config, interval, scope, classification, comparator, counts, and
  keep-decision changes in bob-ledger-tools's source fragments listed above. Cover
  `freshnessIntervalForLine` as well as the queue evaluator: the Task Card refresh row
  and its Ready fallback must show the ordinary interval/source. Remove unused
  reference-only helpers/exports in `350-exports.js` as appropriate.
- Remove `references` from current tier order and commitment arrays, public and fallback
  histograms, status text, group labels, tooltip legends, footer compact modes, entry
  presentation, and reference-specific freshness mark/source text. Never use a
  zero-valued zombie tier to keep old tests passing.
- Advance freshness namespace **10 → 11**, preserving top-level API version 3. Retire
  `referenceReview` and the obsolete ref-specific `refTagIdentity` capability
  advertisement. Preserve `trackerReview` for projects, `checklistTiers`,
  `recurringTier`, and the synchronous/nonthrowing API contract. Public config/count
  fallbacks must have the same new shape as warm snapshots.
- Update navigation's presentation and commitment logic for the surviving groups. Keep
  the shared evaluator authoritative; navigation must not filter out refs or
  independently reclassify current queue rows. Preserve `]s`/`[s`, `]S` closeout, walk
  anchors, jump history, and review-answer advancement semantics.
- If old-provider compatibility still needs recognition of a `references` entry, isolate
  it as explicitly legacy behavior for freshness namespaces <=10 and test it. New
  producer APIs and their consumers must never generate a REF/REFS/ REFERENCES review
  group. Do not drop rows from an older loaded plugin silently; a matching plugin reload
  is the cutover.
- Bump the manifests of changed plugins following repo conventions, update their README
  version/API description, and regenerate committed bundles with `npm run build` before
  checks and deployment.

### 3. Remove the personal setting and update active documentation

- Delete only `freshness.reference_interval` and its attached comment from chezmoi's
  `home/dot_config/bob/config.yml`. Leave Bryan's other cadence settings intact. Confirm
  its YAML remains valid and the ordinary config values survive.
- Update bob-cli `docs/freshness.md` (configuration, pseudocode, tracker section, tier
  comparators, counts, schemas/API, examples, marks/footer, and conformance vectors),
  `docs/plan.md`'s lane review paragraph, `docs/highlights-ref-sync.md`'s obsolete
  reference-tracker paragraph, and the freshness section of `README.md`. Update the
  corresponding bob-plugins README contract. Clearly distinguish historical schema
  descriptions from current behavior if retaining history.
- Describe the keep/decay and commitments/upkeep implications explicitly. Keep separate
  dashboard/reference-library terminology and behavior intact; a blanket replacement of
  the word `references` would damage unrelated features.

### 4. Verify behavior across both implementations

Use fixed dates and temporary vaults for behavioral tests. Cover both tag-only refs
(including `#REF`) and legacy `^ref` tasks, as well as an ordinary control row differing
only in ref identity. Test matching evaluated interval/source, state, bucket, tier, due
date, overdue days, keeps, and decision eligibility.

- Ready: absent stamp, fresh, exactly-due and overdue boundaries; global default and
  configured values; task-over-note-over-config precedence; malformed local overrides
  use the same lint/fallback behavior; resurfacing beats age expiry.
- Pending/Next: absent/null/default, non-default cadence, fresh and due states, and
  `false` disabling each lane despite task/note overrides and old reference config. Lane
  rows retain null state/bucket and correct lane due metadata.
- Obsolete config: numeric, null, boolean, string, and object old-key values are ignored
  without invalidating otherwise good config. Compare absent vs present old key.
  JavaScript also covers camelCase. Project config still validates.
- Scope/overlays: hidden refs, blocked and future rows, Today-linked/daily rows,
  recurring arrived/undated/future occurrences, and PRE/POST refs retain ordinary
  behavior. Preserve `^prj`, including `^prj` combined with `#ref`.
- Mixed queue: refs interleave with ordinary peers by each surviving comparator; exact
  nine-tier histogram, stable ties/ranks, correct total/counts with limit=1, and no
  current reference key/source/group. NEW/TICKLER refs keep commitments outstanding; a
  remaining ROTTEN ref allows commitments-done/upkeep mode.
- Keeps: a due Ready ref increments through the exact existing helper; NEW and non-Ready
  refs do not. At the limit, the explicit gesture opens the card without a premature
  write; `decay: false` preserves ordinary counting-only behavior.
- UI/API: public config, invalid/missing/mobile fallbacks, footer full/compact/ tooltip
  modes, freshness marks, intervalForLine, reviewModel, and navigation
  notices/advancement agree; ordinary READY gating and separate dashboard REFERENCES
  collections remain correct. Verify old-provider fallback explicitly if legacy
  reference handling is retained.

Extend `src/native/config/freshness.rs` unit tests,
`src/native/freshness/state_tests.rs`, and `tests/cli/freshness.rs`; include Ready
consumer tests when the shared evaluator change affects them. In bob-plugins, update the
existing freshness tracking/states/recurring/namespace/review-model/ footer tests,
navigation freshness/refresh-modal/keep-counting/decision-card coverage, and their
shared harnesses. Keep Rust and JS conformance examples in the canonical freshness doc
aligned. Add any new test file to `npm test`.

Run focused tests while implementing, then the repository gates: **`just check`** in
bob-cli; **`npm run build`, `npm test`, and `npm run validate`** in bob-plugins. Use
`/sase_monitor` for long commands as the session instructions require. Do not claim
checks passed until their final exit/results are known. Audit remaining
reference-specific symbols with `rg`; residual hits should be historical notes,
negative/legacy tests, or unrelated reference-library/dashboard features.

### 5. Deploy and smoke-check the resulting behavior

After successful checks, deploy the built changed plugins using
`bob plugins sync --repo <path returned by sase repo open bob-plugins> --no-pull`
(preview with `--dry-run`; scope with `-p` to changed plugins as appropriate). Use the
explicit opened checkout so the sync deploys these changes. Observe normal sync
backup/local-edit guards. Verify the deployed bundles and manifests match the tested
files, and reload the plugins through the available supported Obsidian path if the
running application is accessible.

Follow chezmoi's repository instruction to run `chezmoi update -a --force` after its
change is committed, including when committed by SASE finalization; arrange this through
the supported finalization/follow-up mechanism. Do not mistake applying an older
canonical source tree for deployment of the opened checkout. Verify the effective Bob
config no longer contains the retired key after apply.

Use the newly built Bob binary for a read-only freshness JSON/human smoke check. If live
Obsidian is available, inspect `]s` and the footer on refs in different lanes and verify
no dedicated REF group. Otherwise report live keymap/reload verification as unperformed,
with automated tests and bundle sync results separately stated. No test should stamp or
reschedule real tasks.

## Optional memory decisions

Use `/sase_memory_read` and `/sase_memory_write` before implementing accepted memory
edits. The three frontmatter decisions authorize these notes individually:

> [!decision] memory_reference_policy Create the decision strand
> `decisions:reference-tasks-use-ordinary-freshness` with the user's request as its
> stated rationale and the approved plan/landed code as evidence. Record ordinary lane
> freshness, ref redistribution, and ordinary keep policy.

> [!decision] memory_walk_supersession Mark only the REFERENCES claim in
> `sase/memory/decisions/review-walk-is-tiered.md` as superseded-in-part using
> `metadata.status`, `superseded_by`, and a body back-link. Preserve other supersession
> targets and the accepted body. This requires the new record to exist (an accepted
> `memory_reference_policy` decision); otherwise defer this edit as a follow-up instead
> of creating a dangling link or an unauthorized record.

> [!decision] memory_freshness_glossary Update the interval explanation and remove
> REFERENCES from the current tier order in `sase/memory/glossary/task-freshness.md`,
> preserving existing recurring behavior.

Run `sase memory init` after accepted edits; never hand-edit generated AGENTS.md or
provider shims. With automatic approval, unrequested memory decisions default off: the
tale coder must record skipped changes through `/sase_new_task` as memory follow-up
work, checking duplicates and using the required memory/bead guidance.

## Completion criteria

The supported config and current API contain no reference cadence; current freshness
producers contain no reference review tier/counter/source. Ref tasks match ordinary
tasks across Ready/Pending/Next and all existing overlays, including confirmed explicit
keep decisions. CLI, footer, navigation, counts, documentation, tests, and the personal
config agree on the nine-tier order. Required checks pass, generated plugin files are
current and synced, and any live deployment/manual verification limitations are reported
accurately.
