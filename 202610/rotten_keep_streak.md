---
tier: epic
title: Rotten keep streaks and user-approved decay
goal: "Explicit due-Ready keeps have a trustworthy streak and a quiet freshness-mark
  display; repeated keeps offer an approved decision that enters the existing priority
  ladder, with no silent decay or change to the freshness trial.

  "
phases:
  - id: contract-rust
    title: Keep-streak contract and Rust support
    depends_on: []
    size: medium
    description:
      "contract-rust: specify the shared behavior, implement reading and reset
      semantics, preserve seed behavior, and publish schema 4 with parity vectors."
  - id: ledger-marks
    title: Ledger keep helper and folded marks
    depends_on:
      - contract-rust
    size: medium
    description:
      "ledger-marks: mirror the contract in freshness namespace v5, add the sole
      increment helper, and render accessible folded pips with truthful decision
      annotations."
  - id: nav-counting
    title: Exact explicit-keep counting
    depends_on:
      - ledger-marks
    size: medium
    description:
      "nav-counting: wire single, counted, and Task Link refreshes through exact
      pre-write matching, atomic writes, accurate notices, and cache-lag regressions."
  - id: decay-planner
    title: Shared approved-decay action planner
    depends_on:
      - nav-counting
    size: medium
    description:
      "decay-planner: compose existing priority and log planners into stable previewed
      decisions, including implicit P0 entry and a non-cancelling default."
  - id: decision-card
    title: Decision card and review-walk integration
    depends_on:
      - decay-planner
    size: medium
    description:
      "decision-card: add the consent interaction, guarded action application, batch
      skipping, leaf signal, and trial activation guard."
  - id: rollout
    title: Integrated verification, documentation, and rollout
    depends_on:
      - decision-card
    size: medium
    description:
      "rollout: verify cross-repository reset and rendering behavior, publish the
      accepted memory changes, deploy from the linked source, and document trial and
      calibration checks."
proposed_by: bbugyi200.apollo.4o
create_time: 2026-10-03 10:36:58
status: wip
---

- **PROMPT:**
  [prompts/202610/rotten_keep_streak.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/rotten_keep_streak.md)

# Rotten keep streaks and approved decay

## Outcome and accepted scope

Implement the recommendations Bryan explicitly accepted in
`research:202610/rotten_keep_streak_and_approved_decay/rotten_keep_streak_and_approved_decay.md`.
Read that artifact through `sase artifact read` when implementing. The report was read
before this plan; its source census is historical context, not a new census.

The stored field is **`[keeps:: N]`**, a streak of due-Ready bare keeps, replacing the
originally suggested lifetime `refresh_count`. Bryan's acceptance of all the research
recommendations resolves its naming, counting, default-limit, P0-entry, keep-anyway, and
display questions. Do not introduce a second lifetime counter or alias. Three weekly
keeps normally mean the fourth due review asks for a decision.

Alt+F remains a fast confirmation. Repeated keeps earn a question about urgency, never
an automatic priority/status/schedule change. The existing freshness mark gains faint
dots; its due glyph becomes a leaf when a decision is available. A small keyboard-first
card makes **Not now**, **Less often**, **Reword**, **Drop**, and **Keep** explicit.
Enter can defer but can never cancel from this card.

This is an epic because it changes a shared Rust/JavaScript contract, multiple gesture
consumers, two rendering paths, and guarded multi-note interactions. Each phase above is
bounded direct implementation work. Dependencies are serial because the linked plugins
share large source files and depend on the previous phase's API; do not concurrently
edit those files.

No new review tier, dashboard chip, rotten-page group, task relocation, bulk migration,
backfill, seed rerun, lifetime history service, capture grammar, or Mac frontend logic.
Existing sticky lanes, bucket partition, comparators, Today, Ready caps, and
priority-roll behavior outside the new card remain authoritative. Optional P0
Ctrl+Shift+P recommendation parity is deliberately outside this first release; the
shared planner should allow it later without duplicating policy.

## Repositories and integration points

Work in the implementing agent's bob-cli checkout. Obtain bob-plugins through
`sase repo open bob-plugins -r '<specific reason>'`; use only the printed path and read
its AGENTS.md. Do not edit installed plugin copies. The linked repo requires
`bob plugins sync` after changes; pass its returned source path with `--repo` and
`--no-pull`, plus `--plugin` to limit deployment. Verify actual synced bytes: a zero
exit can still report dirty files skipped.

Relevant bob-cli files:

- `docs/freshness.md` is the cross-language contract and vector catalog;
  `docs/projects.md` owns priority roll/decay and log behavior.
- `src/native/freshness/{placement,state,scan,cli,seed}.rs`,
  `src/native/config/freshness.rs`, `placement_tests.rs`, `state_tests.rs`,
  `tests/cli/freshness.rs`, `tests/cli/capture/freshness_stamps.rs`.
- Capture's existing `stamp_fresh` consumers and description/preview cleaners.

Relevant paths relative to bob-plugins:

- `plugins/bob-ledger-tools/main.js`: `readFreshness`, `freshnessStampLine`,
  `freshnessSetRefreshLine`, suffix/rebuild helpers, config/evaluator/queue/counts,
  `freshnessMarkSource*`, `freshnessMarkModel`, mark consensus and widget equality,
  rendered-view postprocessor, and the public freshness namespace (currently v4).
- `plugins/bob-navigation-hotkeys/main.js`: `planFreshStampBatch`,
  `matchFreshStampRefs`, `refreshTaskFreshnessOnTasks/OnLinks`, `finishFreshStamp`,
  `buildReviewJumpNotice`, refresh picker, priority planners, Schedule/Cancel Log
  writers, and existing preimage/undo/link-write machinery.
- Corresponding `styles.css`, manifest versions, README, and scripts named
  `test-navigation-{freshness,stamps,roll-decay}.cjs`,
  `test-ledger-tools-freshness*.cjs`, `test-task-status-cycler.cjs`, and
  `test-block-id-prompt.cjs`.

## Shared behavior contract

### Counting, clearing, and storage

| Event                                                                                                    | Result                                                                            |
| -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Alt+F / Alt+Shift+F, exact due Ready target in `rotten` or `returned`                                    | Stamp and increment once in the same write, unless an active decision is required |
| NEW, FRESH/early, Pending, Next, Blocked, Today-linked, other excluded target, or unresolved cache match | Stamp with the keep helper and preserve the streak; do not count                  |
| Repeated same-day keep                                                                                   | Preserve streak; canonical input is byte-identical                                |
| Any other supported human stamp                                                                          | Clear the streak, even when `fresh` already equals today                          |
| Close/cancel                                                                                             | Preserve the streak on the closed line; no stamp                                  |
| Automation, hooks, randomize, and seed                                                                   | Neither increment nor reset                                                       |
| Recurring or closed Alt+F target                                                                         | Existing refusal; no write                                                        |

Counting eligibility must use the **pre-write** task snapshot, with all of
`entry.path === path`, `entry.line === editorLine + 1`,
`entry.originalMarkdown === rawLine`, `lane === 'ready'`, and
`tier in {'rotten','returned'}`. Require exactly one matching row. The current
`matchFreshStampRefs` accepts line-only or raw-only matches; it is not an authorization
predicate. Add a strict predicate and use it for accounting and anchor exclusions where
appropriate. Do not infer eligibility from age, glyph, bucket alone, block ID alone, a
changed line number, or a selected DOM row.

The one increment helper is JS `api.freshness.keepLine(line, dateText, { counted })`.
Independently require a valid prior `fresh < today` before incrementing, even if
`counted` is true. This prevents NEW and same-day inflation under stale caches or
repeated events. Rust reads/clears/reports; it has no increment path. All generic human
stamp helpers clear `keeps` by default, covering nav, cycler, block-id-prompt, and
capture consumers without scattering reset lists through gesture code.

**Seed exception discovered in source:** `seed.rs::stamp_change` calls `stamp_fresh`.
Explicitly preserve keeps there with a private preserve-mode primitive; do not let the
new default reset silently turn seed into a resetter. Test it with fixtures; never run
the seed against the live vault.

Canonical form:

```markdown
- [ ] #task Rename queue input [fresh:: 2026-10-08] [refresh:: 14] [keeps:: 2]
      [created:: 2026-09-12] [priority:: medium] ^rq
```

`keeps` is a decimal integer 1–999; absence means 0 and writers omit zero. Increment
saturates at 999 and never wraps. Readers accept existing bracket or paren field syntax,
select the first valid value, and emit `keeps_invalid` and `keeps_duplicate` as
appropriate; if none is valid, report 0. `fresh_misplaced` also covers keeps inside the
Tasks suffix. A successful keep canonicalizes and repairs malformed/duplicate placement.
Generic stamps remove all keeps fields. An uncounted keep preserves the valid semantic
value while canonicalizing it.

Extend suffix scanners with `keeps` as a run-extending **non-Tasks** key. Never add it
to the Tasks key registry. Output order is fresh, optional refresh, optional keeps, then
the existing Tasks suffix, tags, and block ID. Both Rust parsers must see the same Tasks
fields before/after writes. Preserve task body, child blocks, quote prefixes, CRLF, and
final-newline state under existing writer rules. No metadata should leak into clean task
descriptions or capture previews.

No backfill: seeded dates, captured dates, and old `fresh` stamps are not evidence of
keeps. Git sync may lose an increment during conflict recovery; do not claim this scalar
is an exact global event counter. Cache uncertainty under-counts. Hand editing remains
unobserved: edit-then-Alt+F on an exactly resolved due task can count; **Reword**
provides the explicit reset and this limit is documented.

### Configuration, schema, and trial protection

```yaml
freshness:
  decay:
    keeps: 3
    # enter: P2
```

`decay` absent, null, true, or `{}` means enabled with 3 keeps. `false` keeps
counting/display but never asks or skips. `keeps` accepts integer 0–999; 0 asks on every
due Ready re-confirmation, never NEW. `enter` is an optional nonempty configured
priority label; absent/null uses interval-aware entry. Invalid scalar types,
fractional/negative/out-of-range values, and malformed blocks follow the current
freshness config failure contract: Rust exit 2; plugin defaults with `invalid`
diagnostics. Unknown/nonexistent entry labels produce a clear diagnostic and make **Not
now** unavailable, without disabling Keep/Reword or inventing a fallback priority.
Priority config validity is resolved against the existing priority loader, not a second
hard-coded P1–P4 table.

Use a temporary, explicit release boundary of **2026-10-19 in the vault's local
calendar**, after the accepted trial (October 5–18). Before that day, count and show
pips only: no cards, leaf, decision skip, or 'next review asks' promise. Implement this
as one documented activation constant per language, covered by shared boundary vectors;
it is rollout policy, not a new editable config knob. After that date a card still
requires an explicit gesture, never a timer write. Do not rely on the report's weekly
arithmetic: short intervals, RETURNED tasks, hand-set counts, and `keeps: 0` can reach a
threshold early. If implementation finds the documented trial was extended, move the
boundary to the day after its recorded end consistently before release. Agents need not
wait for the date or claim trial observations that have not happened.

`bob freshness list` moves schema 3 → 4: queue rows gain `keeps` and `decide`; counts
gain `decide`; `config.decay` reports normalized `enabled`, `keeps`, `enter` (label or
null), and read-only `active_from` / `active` rollout metadata.
`decide = active && enabled && lane ready && tier rotten/returned && keeps >= limit`.
The annotation means a choice is due, not permission to execute an action.
Invalid-config fallback may render/count using the documented defaults, but must not
offer a priority-changing recommendation from an invalid config; the card explains the
unavailable action and retains its safe alternatives. The shared schema constant also
changes the seed envelope; seed contents stay otherwise unchanged. CLI human rows show
`kept N×` and `· decide` where true; header explains the threshold, off state, or
pre-activation date. Counts cover the full queue regardless of `--limit`. No new CLI
subcommands/options.

JS freshness namespace becomes v5 (top-level API remains v3). It exposes the same
normalized semantics in the existing camelCase conventions plus keepLine. Old nav cannot
increment. New nav on pre-v5 ledger uses the old stamper as an uncounted fallback. A v5
provider missing/throwing keepLine must fail without writing, rather than falling back
to the now-resetting generic stamper. Display decision affordances only when the
compatible card capability is present, the rollout is active, and resolution establishes
a due Ready task. Rust headless reporting does not depend on whether Obsidian is
running.

### Display and interaction specification

Reuse the existing 0.8em interface-font mark, 16-unit SVG box, 1.9 stroke, baseline,
tabular numerals, source reveal, session toggle, and theme variables. No generated
bitmap asset is needed; extend the established code-native glyphs.

| Situation                      | Mark                                                       |
| ------------------------------ | ---------------------------------------------------------- |
| No keeps                       | Existing mark, unchanged                                   |
| Aging with two keeps           | `◔ 3d ••`                                                  |
| Confirmed today with two keeps | `✓ today ••`; dots stay faint, never green                 |
| Due below threshold            | Existing orange `⟳` capsule plus subdued dots              |
| Active decision due            | Existing capsule, leaf replacing `⟳`, `data-decide="true"` |
| Four keeps at limit three      | `leaf 8d •••+1`                                            |

Dots are filled, about 0.34em with 0.14em gaps, `--text-faint` normally and subdued
orange in a due capsule. No red, extra border, empty progress dots, or achievement
styling. Default dot cap is three; for thresholds 1–2 use that threshold, for
larger/custom/zero thresholds cap at three to keep line width bounded. Overflow is `+N`;
the exact count is always in accessible text.

Fold only one valid square-bracketed keeps field, exactly one space after the last
folded fresh/refresh field. Noncanonical fields remain visible Dataview pills with the
existing dashed repair styling extended to `keeps`. Extend both Live Preview and
rendered Tasks/reading/embeds/hover paths; avoid double decoration and skip code/pre/raw
source. Selection overlapping any folded field reveals the entire raw span. Clicking
reveals source and never writes. Include count and decision eligibility in model
equality and consensus: ambiguous rendered matches stay neutral, never guess a leaf.
Out-of-scope/closed tasks may show historical dots quietly but never a decision glyph.

Tooltip adds `Kept 2 reviews in a row · Bob asks at 3` when the card is active; at
threshold change the key hint to `Alt+F to decide`. Before activation or with decay off,
use truthful counting-only/off wording. Use proper singulars and an explicit sentence
for threshold zero. Keep the current multiline aria-label and source-edit accessibility.
Notices add `kept N×`; only active capabilities promise `next review asks`. No claim of
'no progress' or low task value.

### Decision planner and card

The trigger is a **single source-task** Alt+F/Alt+Shift+F with exact eligibility and
pre-write `keeps >= limit` after activation. The press opens the card and writes
nothing. A press moving 2 → 3 simply stamps; the next due press asks.

```text
leaf  Kept 3 reviews in a row
      Rename queue input · gkeep_inbox.md · captured 6 weeks ago

Enter  Not now     P0 → P2 · back in 17 days (8–30)    recommended
L      Less often review every 14 days instead of 7
E      Reword     edit the task · start the count over
D      Drop       cancel · dropped after 3 keeps
Alt+F  Keep       still right · asks again next review

Esc changes nothing · 1–4 pick a P-level instead
```

Adapt the existing picker modal/styles and key handling, with a clear title, muted
source/context, aligned key hints, focus visibility, screen-reader row labels, and
narrow-screen wrapping. Age is optional when created is unavailable. The card consumes
its opening gesture; key repeat, bubbling, and a double callback cannot approve or apply
twice. It never opens nested cards.

| Choice          | Required behavior                                                                                                                                                                              |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Enter / Not now | Preview and apply a deferral through existing priority/schedule writers; stamp and clear keeps                                                                                                 |
| L / Less often  | Set next research preset strictly above current interval (14, 30, 90), stamp and clear; at 90+ open existing custom refresh picker constrained to a longer value ≤365; at 365 show unavailable |
| E / Reword      | Stamp and clear; put cursor at end of task body before metadata for editing; stay on this task even for Alt+Shift+F                                                                            |
| D / Drop        | Existing guarded cancel writer and side effects; keep history on closed line; dated Cancel Log `🍂 dropped after N keeps`                                                                      |
| Alt+F / Keep    | Counted keep with saturation; stamp; do not reset; next due review asks again                                                                                                                  |
| 1–4             | Explicit configured priority pick using the existing mapping and shown window, with reset; do not invent absent levels                                                                         |
| Esc / dismiss   | No writes, log entries, accounting, or walk advance; retain anchor                                                                                                                             |

P0 is **absence** of priority, not an unknown priority string. For P0, choose the first
configured level in ladder order with `min_days > effective interval` (defaults: 7 → P2,
30 → P3), unless `enter` selects a valid fixed level. If no qualifying level exists,
show Not now unavailable and allow an explicit level choice; do not secretly shorten the
lease, silently pick the last level, or cancel. Unknown priority values and invalid
windows also disable that action.

Prioritized tasks delegate to `planPriorityRollRecommendation`. Use its same-level roll
or next-level decay unchanged. If it recommends cancel, substitute a same-level roll
**only for this card** and label it truthfully. Existing Ctrl+Enter cancellation
semantics elsewhere are unchanged. Respect a disabled priority decay ladder. A Not now
result must actually defer into the future; configurations that cannot provide a future
date show an unavailable row.

Roll once for the displayed action and persist that exact plan/date until approval.
Revalidate task line, relevant child log/preimages, local day, config, and eligibility
immediately before commit. If any relevant input changed, write nothing and
rebuild/reopen for a fresh choice; never apply new unseen random dates or overwrite
another edit. Reuse existing transactional writers, status reconciliation/grouping,
dependency recovery, and Today-link pruning. One source-note decision is one undo step;
cross-note side effects retain the picker's documented guards/recovery limitations and
are tested honestly.

Schedule-changing decisions append ` · kept N×` after the existing reason head: P0 entry
uses `🎲 P0 → P2 decay · in **17** (8–30) days · kept 3×`. Existing same-level roll and
decay heads keep their classifier meaning. Drop uses the existing Cancel Log. Less often
and Reword are explicit decisions and get a dated review-reason entry under the existing
Schedule Log, with the action, before/after interval where relevant, and `kept N×`; do
not fabricate a scheduled date change. Use the existing log insertion/indentation
primitive. Keep anyway remains a line-only increment (its over-limit count plus git
history is its evidence), and Esc has no record. No new REVIEW LOG structure. Pin the
non-scheduling decision-entry grammar and classifier reset behavior in docs/tests; these
are deliberate decisions, not roll events.

Successful Alt+Shift+F outcomes, including the card's Keep, advance exactly once after
commit except Reword, which must leave focus for editing. Failed or dismissed secondary
pickers stay due and do not advance. Outcomes opened by normal Alt+F stay on the source
task.

Counted source sessions and **all Task Link sessions**, even a single link, never open
cards. Below threshold they count normally. With active decay they skip exact at-limit
targets without changing fresh/count and report `1 needs a decision`; source navigation
via the review walk reaches them later. Skipped targets never enter completed-key/anchor
exclusions or inflate upkeep. Deduplicate references to the same source target before
planning. Preserve existing whole-batch refusal and all-or-nothing preimage semantics
for other errors; skip is a named decision outcome, not a swallowed write failure.

## Implementation phases and acceptance

### contract-rust

Publish the shared contract above in `docs/freshness.md` and a machine-readable fixture
such as `tests/fixtures/freshness_keeps/vectors.json`. Implement Rust
parsing/lints/canonical placement/reset, explicit seed preservation, normalized config
and trial eligibility, schema 4 rows/counts/header, and metadata-clean descriptions.
Keep existing schemas' other fields and tier semantics stable.

Acceptance vectors K1–K13: absent/first/increment/refresh placement, same-day, uncounted
preservation, ordinary stamp/set-refresh reset (including same-day),
misplaced/duplicate/malformed/0/negative/fractional/1000 cases, NEW, closed and
recurring refusal, parser invariance, and exact-match semantics. JS runs increment
vectors; Rust runs the common read/reset/placement cases without adding a production
increment API. Add saturation/seed/capture-reset cases and schema/config cases including
zero/off, full counts with limit, and day-boundary activation. Run focused Rust
freshness/config/capture tests and formatting.

### ledger-marks

Mirror contract and fixture vectors, publish v5 keepLine, and clear generic stamps. Add
config/evaluator/queue/count fields and lints. Implement folded pips in both renderers,
source reveal, tooltips, repair CSS, and model equality. Keep leaf/key promises gated by
a compatible nav card capability (absent until decision-card). Bump the ledger plugin
manifest and update its API documentation.

Acceptance: tests for all K read/reset/increment cases; MK cases for no count,
today/aging/due/resting, limits 0/1/3/999, overflow, invalid/duplicate/parens,
quote/code, adjacent/nonadjacent fields, unresolved/contradictory consensus,
selection/click/toggle and repeated rendering. Test both date sides and invalid config.
Use existing freshness, mark, mark-surfaces and dashboard parity scripts.

### nav-counting

Refactor planFreshStampBatch to accept per-target decisions with path/line/raw context.
Resolve exactly against queueBefore; use keepLine for every explicit keep, including
uncounted ones. Deduplicate target identities. Adapt single, counted, and Task Link
flows, fresh notices, walk anchors, and fallback behavior. Do not turn on card
interception/decision skipping before the card exists. Ship counting/pips as the first
usable milestone and record that trial-neutral instrumentation in the docs. Generic
non-keep gestures continue using the resetting stamper. Bump nav's manifest.

Acceptance: true ROTTEN/RETURNED increments; every excluded/NEW/lane/early case
preserves; same-day repeat stays identical; cache's right line/wrong raw, right
raw/wrong line, wrong path, ambiguous match, missing API, throwing API, and duplicate
Task Links cannot inflate counts. Test real plugin handlers, not only pure helpers:
stale source, multi-note stale preimages, rollback, CRLF, one undo step, and one
physical key dispatched through multiple routes. Run navigation freshness/stamps and
relevant cycler/link regression scripts.

### decay-planner

Add a pure freshness decision planner adjacent to the priority recommendation helpers.
Inputs include exact target/preimages, parsed keeps, interval, normalized
freshness/priority config, local date, and injectable randomness. Output is an immutable
card model and action plans, including unavailable explanations. Reuse existing
priority, refresh, cancellation, and log insertion planners. Document dated
review-decision entries and kept-count reason tails.

Acceptance D vectors: P0 at interval7 → P2 and interval30 → P3; fixed entry;
missing/unknown priority and no suitable level; custom level order; configured roll
limits/off; P2 existing roll streak decay; terminal cancel → same-level roll inside card
only; future-date/overflow refusal; Less often 7/14/30/90/365; stable previewed date;
saturation Keep; Drop preserves keeps; reason tails do not change roll classification;
Less often/Reword break it deliberately. Run existing roll-decay tests plus the new
planner tests.

### decision-card

Implement the card and guarded commit adapters above. Enable the shared activation guard
and capability indication, eligible leaf/tooltip/notices, batch skip, and walk
continuation only after handlers are installed. Clean up capability registration on
unload so marks cannot promise an absent card. Ensure a mixed-version session degrades
to counting or repair pills without an unintended reset on explicit keep. Bump modified
plugin manifests.

Acceptance: opening/Esc does zero writes; opening key/repeat cannot approve; Enter never
cancels including terminal ladder; D uses cancel/prune/recovery; approval uses the
preview and rejects stale task/log/config/date; selection, mouse and keyboard activate
the same action; Reword focus stays; Alt+Shift+F advances once for applied decisions;
secondary-picker cancel stays; mixed and all-skipped batches retain skipped tasks and
exact accounting. Test threshold 0 and short intervals before/after October 19, plus
unload/version-skew cases.

### rollout

Run repository gates appropriate to the integrated change: bob-cli `just all`
(fmt/clippy/tests), bob-plugins `npm test` and `npm run validate`; ensure newly added
scripts join npm test. Carry shared vectors between languages using an explicit
fixture-path argument in tests or mirrored fixtures with a parity comparison, never a
hard-coded checkout location. Exercise fixture-backed capture previews and full reset
consumers, including automation non-mutation.

Use `/sase_memory_write` before the following **authorized** memory changes (Bryan
accepted the report's memory recommendation):

- Add `sase/memory/decisions/rotten-keeps-use-priority-decay.md`, titled 'Rotten keeps
  decay through the priority ladder', citing the approved research, this approved plan,
  and implementation evidence. Record the stored-streak exception, explicit approval,
  lane exclusion, and costs/reopening criteria.
- Mark `sase/memory/decisions/ready-is-freshness-gated.md` superseded in part only for
  'stamps remain the only write', preserving its accepted body and existing supersession
  links; add the new back-link and metadata target.
- Update `sase/memory/glossary/task-freshness.md`: freshness itself still never changes
  priority/schedule; the approved decision card invokes separate existing writers. Add
  `sase/memory/glossary/keep-streak.md` with the concise counting/reset definition and
  links. Run `sase memory init`; do not hand-edit generated AGENTS/provider shims or the
  decisions descriptor.

Update docs/freshness §§1–7/11–13, docs/projects decay/log integration, and plugin
README/changelogs. Record actual count/pip deployment, activation date, and rollback
procedure (`freshness.decay: false` disables interception while preserving counting;
mark toggle restores raw pills). Do not silently change the user's global config or note
intervals, and do not touch inbox residence.

Deploy tested plugin sources with scoped
`bob plugins sync --no-pull --repo <opened-path> --plugin <id>`, first dry-run then
sync. Coordinate ledger/nav versions; generic consumers get reset behavior through the
API. Install the updated bob binary through the repository's existing install workflow
where needed for schema4 and capture resets. Verify plugin list reports matching
versions/bytes; dirty-file skips are incomplete deployment, not success.

Visually verify on fixture tasks in an available Obsidian environment: light and dark
themes, dense lines, narrow widths, LP/reading/Tasks/embeds, dots/leaf, raw reveal,
focus, tooltip, explicit cancellation, and undo. Do not manipulate live task
dates/counters to force a card. If no Obsidian UI is available, report that limit and
leave a concise human smoke checklist in docs instead of claiming visual verification.
Automated mark-surface and modal interaction tests still run.

For calibration, document a lightweight human tally after one month: decision count,
Keep/Drop outcomes, and RETURNED load for two weeks after the first wave. If most cards
(>50%) are Keep, consider limit4; if most are Drop, consider limit2. Use counts,
decision logs and git history; no telemetry service or automatic tuning. Completion
means the feature and trial-safe rollout are ready, not that an agent waits a month or
claims a future experiment succeeded.

## Final review checklist

- No field or API means a lifetime refresh total; no lane refresh builds decay.
- Placement cannot hide priority/scheduled/id/dependencies from either parser.
- Same-day generic stamps clear; same-day explicit keeps preserve; seed preserves.
- Exact raw identity gates counting; approval checks fresh preimages and inputs.
- No decision UI or skip before the trial boundary; no mutation without a choice.
- Enter cannot cancel; Drop is explicit; previewed dates are the dates written.
- Counts/pips/leaf/CLI agree without changing queue order, buckets, or Ready caps.
- Card dismissal, failed secondary picker, stale writes, and skipped batches leave tasks
  due; notes/logs do not partially mutate through new code paths.
- Full meaningful test suites pass; deployment and UI checks are reported with evidence
  and limitations, and no historical or future trial data is invented.
