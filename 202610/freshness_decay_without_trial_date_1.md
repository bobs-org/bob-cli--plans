---
tier: tale
title: Remove the freshness-decay trial activation date
goal:
  Make gesture-triggered freshness-decay cards, marks, skips, and CLI reports available
  on every date while preserving eligibility, explicit consent, and the user off-switch.
size: medium
proposed_by: bbugyi200.athena.0w4.f0
create_time: 2026-10-04 06:25:27
status: wip
---

# Remove the freshness-decay trial activation date

## Outcome

Freshness-decay decisions are available immediately after installing compatible updated
plugins. There is no October 19 switch, replacement date, opt-in, or calendar-based
counting-only period. On an exact, due Ready ROTTEN/RETURNED task at the configured keep
limit, Alt+F opens the existing consent card; Alt+Shift+F uses the same card and
existing advance behavior. Opening or dismissing it writes nothing. The existing
explicit choices remain Not now, Less often, Reword, Drop, and Keep.

Counted source sessions and every Task Link session still skip decision-needed targets,
report `N needs a decision`, and leave their stamp/streak and walk accounting untouched.
The leaf, `Alt+F to decide`, threshold tooltip, queue flags, and CLI counts agree before
and after the former boundary. `freshness.decay: false` remains the deliberate user
off-switch: count and show pips, without cards or decision skips. Defaults remain
enabled with 3 keeps; a zero threshold asks at every due Ready re-confirmation, never
NEW.

This is a medium tale for one implementation agent. The gate and its callers are known;
the card, eligibility evaluator, writers, and test harnesses already exist. The bounded
work spans Rust, two JavaScript plugins, their contracts/tests, docs, and deployment.
Separate implementation phases are unnecessary.

## Context and authority

The current user request overrides the activation-date clause of
`decisions:rotten-keeps-use-priority-decay` and the instruction to preserve that gate in
`plan:202610/task_card_only.md`. Every other accepted decay rule continues to apply.
Read the decision through `sase memory read`; read the two existing plans through
`sase artifact read` if their context is needed. The original keep-streak implementation
is described by `plan:202610/rotten_keep_streak.md`.

The Task Card plan concerns Ctrl+Shift+P. This tale removes the separate decay gate and
preserves the Task Card's single surface, direct Less often stage, and close keys. Its
edits may overlap the previous coder's docs and navigation file: inspect current source
and retain those changes. Do not reset another agent's work or copy older whole files
over newer code.

Canonical decision memory needs a narrow supersession, tracked separately in
`bead:bob-cli-44`. This tale does not edit memory, generated instruction files, or
provider shims, and implementation must not expand its scope into that follow-up.

## Repository map and root cause

Work from the assigned bob-cli checkout. Open the source plugin repository with
`sase repo open bob-plugins -r "Remove the freshness-decay trial date"`, use only its
printed path, and read its AGENTS.md. Paths below are relative to their owning repo.
Installed plugin files in the vault are deployment destinations, never source files.

In bob-cli:

- `src/native/config/freshness.rs` defines `decay_active_from()` as 2026-10-19 and
  `decay_active(today)` as a calendar comparison, with a matching boundary unit test.
- `src/native/freshness/state.rs::decide_for` includes that comparison in eligibility.
- `src/native/freshness/cli.rs` stores activation fields in ListReport, prints
  `asks from 2026-10-19`, and exports `config.decay.active_from` / `active` under
  freshness JSON schema 7.
- `tests/fixtures/freshness_keeps/vectors.json` and the Rust state/CLI tests encode a
  no-decision early-date result.

In bob-plugins:

- `plugins/bob-ledger-tools/main.js` has `FRESHNESS_DECAY_ACTIVE_FROM`,
  `freshnessDecayActive`, a gated `freshnessDecideFor`, date-sensitive mark resolution,
  leaf/tooltip gates, and `activeFrom` / `active` in `apiFreshnessConfig` including its
  fallback. The freshness namespace is currently version 5.
- `plugins/bob-navigation-hotkeys/main.js` repeats the constant in
  `freshnessDecayCardActive`. Its callers guard batch skip, single-card context,
  precommit validation, and card opening. It advertises card capability version 1.

Changing only one constant or moving it into the past would retain obsolete policy and
would let the CLI, marks, and gesture handlers disagree. Delete the calendar machinery
and coordinate the published contracts.

## Implementation

### 1. Remove the Rust gate and publish the updated JSON contract

Delete the two activation helpers and their obsolete imports/tests. Drop `today` from
`decide_for` and its call sites when it has no remaining use in that predicate. Keep
calendar dates in the surrounding evaluator: freshness ages, due tiers, future
schedules, same-day confirmation, and local-day validation still depend on them.

The decision predicate is now exactly enabled decay, Ready lane, ROTTEN/RETURNED tier,
and keeps at or over the limit. Preserve tracker exclusions, visibility/recurrence/Today
eligibility, queue order, buckets, review cadence, Ready caps, and keep count semantics.

Remove ListReport's activation fields and the human meter's deferred-activation branch.
Enabled headers show `keeps N`, zero shows `keeps 0 · asks every review`, and disabled
shows `keeps N · decay off` on every date.

Remove `active_from` and `active` from `config.decay`. Its complete mapping becomes
`{ enabled, keeps, enter }`. Bump the existing shared freshness JSON schema constant
from 7 to 8, consistently for list, seed, and error payloads that use it. Update their
schema assertions and documented contracts. This explicitly versions the removed fields
instead of leaving a misleading date, sentinel date, or permanent rollout placeholder.

### 2. Remove the plugin calendar gates and coordinate capabilities

In ledger-tools, delete the activation constant/helper and their helper exports. Remove
the date argument from the decision predicate if unused. Remove `decayActive` from mark
resolution and equality/model plumbing, and `activeFrom` / `active` from public config
and its fallback. The mark's asking language and leaf depend on a resolved eligible row,
enabled decay, and a compatible installed card. Below-threshold resolved rows may
promise the configured limit when that card is available. Unresolved, contradictory,
old-version, or missing-card cases retain truthful counting-only wording.

Publish freshness namespace version 6 for the date-independent decision/config contract;
keep the top-level plugin API version unchanged. Existing methods, tracker capabilities,
and keep/reset behavior remain available. Replace rollout terminology in mark helpers
with card availability; leave unrelated `active` fields elsewhere in the plugin alone.

In navigation-hotkeys, delete its activation constant and calendar comparison, updating
every caller and exported test helper. A small config/capability predicate may replace
`freshnessDecayCardActive`; it must not choose availability by a date. Keep valid-date
checks at input/planning boundaries and the existing precommit local-day, config, exact
task, child-log, and queue revalidation. Preserve all guarded writers and existing card
dismissal, undo, Reword focus, and Alt+Shift+F continuation semantics.

Advertise `api.freshnessDecayCard.version = 2` for the ungated handler contract.
Ledger-tools must require card version >= 2 before promising a leaf or asking hint;
version 1 still gates by date and cannot fulfill that promise. Updated navigation must
require freshness namespace >= 6, together with the existing required methods, for card
interception and batch decision skipping. Keep ordinary counting support at >= 5. New
navigation with an older ledger falls back to the existing counted keep path; new ledger
with older/missing navigation shows pips without a card promise. Missing or throwing
APIs retain existing safe behavior, including refusal when a keeps-capable namespace
lacks keepLine. Capability registration/unload must refresh marks correctly. Do not
infer a decision from raw counts when the queue lacks an exact eligible row.

### 3. Replace the old-date regressions with immediate-availability coverage

Update the shared keep fixture's B/D cases and descriptions: no activation metadata, no
early-date suppression. Preserve K placement and C config cases. Its current `schema: 4`
identifies the original keep-vector contract and is asserted by the placement runner; do
not confuse it with freshness output schema 8. Revise the fixture version or description
deliberately if needed without gratuitously rewriting unrelated vectors.

Use due fixtures with fresh dates sufficiently old for each tested date. Check
2026-10-04 and 2026-10-08 as well as 2026-10-18, 19, and 20: an at-limit Ready due row
decides on all of them. Test a date sufficiently earlier than the former boundary to
catch a hidden replacement boundary. Preserve below-limit, enabled/off/zero, RETURNED,
NEW, same-day, Next/Pending, project/reference, blocked, future-scheduled, recurring,
hidden, and Today-linked exclusions. The state counts test that currently expects
`silent.decide == 0` must reflect the new policy when its row is due.

Update actual handlers and DOM/model tests, not just pure activation-helper tests:

- Before October 19, single Alt+F opens the card with zero writes; dismissal leaves it
  due, and explicit approval performs only the existing selected outcome.
- Alt+Shift+F advances exactly once after a successful applicable decision and never
  after dismissal, stale rejection, or a cancelled secondary stage.
- Counted and Task Link sessions, including one Task Link and duplicate references, skip
  exact decision targets before October 19; mixed/all-skipped accounting stays correct
  and no skipped target is stamped or excluded from the review walk.
- Capable marks before October 19 show leaf/decision hints; off, below-limit,
  unresolved, and mixed-version marks use appropriate existing non-decision visuals.
- Test both version-skew directions, unload, invalid dates, changed local day, changed
  task/config/log, and missing/throwing APIs. Keep consent and guarded-write assertions.
- Human/JSON CLI output shows the same decision count on early dates, schema 8 and
  exactly three decay config keys, with no `asks from` text.

Relevant files include Rust config/state/placement tests and `tests/cli/freshness.rs`,
plus plugin scripts `test-ledger-tools-freshness.cjs`,
`test-ledger-tools-freshness-keeps.cjs`,
`test-ledger-tools-freshness-decision-card.cjs`, `test-ledger-tools-freshness-mark.cjs`,
`test-ledger-tools-freshness-mark-surfaces.cjs`, `test-navigation-decision-card.cjs`,
`test-navigation-decision-card-handlers.cjs`, `test-navigation-keep-counting.cjs`, and
the existing freshness/stamp scripts. Update version fixtures throughout the suite;
preserve tests of intentionally older namespaces. Keep the shared Rust/JS expectations
aligned using the existing fixture/mirroring arrangement without a fixed checkout path.

### 4. Docs, versions, validation, and deployment

Rewrite the activation claims in bob-cli README and `docs/freshness.md` §§1/2a/6/7/10a/
11/12/14: immediate gesture-driven availability, new JSON/API/capability contracts,
compatible-version fallbacks, and the preserved off-switch. Update bob-plugins README's
decay prose and version/API table. Preserve the separate historical Ready trial in
freshness §13 and its metrics; removing this decay gate does not assert that an
experiment succeeded or remove unrelated trial dates. Retain historical deployment facts
as history and add the actual ungated release facts when known. Calibration is relative
to this rollout, not a future October 19 activation.

Bump both modified plugin manifests and matching README entries from the actual current
source versions (observed navigation 2.1.0 and ledger 1.27.0 during planning); use an
appropriate minor release for the contract change, retaining concurrent version bumps.
Search targeted source/docs/tests for the removed symbols and old rollout promises.
Ordinary scheduling examples in docs/randomize.md and roll tests may still contain
October 19: they are real date data, not the removed gate.

Run focused Rust freshness/config/CLI/capture regressions and the relevant plugin
handler/mark tests, then the existing repository gates: bob-cli `just all`, bob-plugins
`npm test` and `npm run validate`. If an unrelated failure occurs, reproduce and report
it with existing tracking rather than weaken an assertion or claim a clean full run. Do
not run implementation tests during this planning turn or modify source before plan
approval. Use the SASE monitor skill for long commands during implementation.

After checks, install the updated bob binary using the repository's existing
`just install` workflow. Deploy both tested plugin sources from the opened repo with
scoped `bob plugins sync --no-pull --repo <opened-path> --plugin <id>`, first with
`--dry-run`, then without it. Verify `bob plugins list --no-pull --repo <opened-path>`
reports the expected versions/bytes; a dirty-file skip is incomplete deployment and must
be reported rather than hidden by exit code 0. Do not force-overwrite local edits.

This request arose on the MacBook: report which vault actually received the deployment.
If only the current host is accessible, give the tested plugin versions and a precise
Mac update/reload checklist. Do not claim athena's sync updated a Mac Obsidian session;
plugins are gitignored by vault sync. Follow the tailnet reference memory before any
authorized remote access. Confirm reloaded compatible plugins in an available GUI;
otherwise report the headless limitation and fixture smoke steps. Never modify live task
dates or keeps merely to trigger a card.

## Completion criteria

All date comparisons selecting decay availability are gone. Early-date regressions
exercise the actual single/batch/Task Link handlers, marks, and CLI, while every
consent, eligibility, off-switch, and stale-write rule still holds. Published versions
describe the changed contracts and handle old peers truthfully. Tested source is
deployed to the available target with verification; any Mac/GUI limitation is stated
precisely. The Ctrl+Shift+P Task Card work remains intact. No canonical memory or live
task data was rewritten by this tale.
