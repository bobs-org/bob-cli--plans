---
tier: tale
title: Finish and land the Bank Badge epic
goal:
  Correct the remaining badge period, text, visibility and empty-history gaps, verify
  integration, and close bob-cli-lh.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-lh.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-lh
create_time: 2026-11-14 09:04:21
status: wip
---

- **PARENT:**
  [202611/bank_badge_strip_family.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_badge_strip_family.md)
- **BEAD:**
  [bob-cli-lh](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lh/README.md)

# Finish and land the Bank Badge epic

Finish the remaining work in `bob-cli-lh`, then close that epic in this coding turn.
This tale has no later land agent. Do not wait for this tale's commit, push, SHA, or CI
result before the closeout below.

## Scope and verified context

Read `sase bead read bob-cli-lh -r "Need the landing scope and latest notes"` and
`sase artifact read plan:202611/bank_badge_strip_family.md "Need the approved badge contract"`.
All four original phases are closed. The lander reviewed their complete notes, the Rust
report/CLI/output/tests, the shared plugin decoder/model/painter, navigation and fence
lifecycle, and the following commits:

- bob-cli: `28f48be9` (initial glossary), `575f9e66` (report periods), `9a6013a2`
  (completed vocabulary/docs).
- bob-plugins: `008700b` (shared badge), `6a1778b` (fence hosts).
- Non-epic bob-cli drift since the first epic commit is `74a5db16` and `50b015b1`,
  documentation refreshes. They are integrated on master; preserve their
  project/Gmail/configuration corrections while completing the reference omissions
  below. No non-epic bob-plugins commit followed its first badge commit at audit time.
  Recheck later drift before closing.

The approved memory decisions are final: only Bank Badge, Day Strip and Stats Strip
glossary changes were authorized; the color decision record was rejected. The authorized
strands now exist and the rejected record was not added. This tale needs no memory edit.
Do not edit Cargo versions or CHANGELOG.md.

Use `/sase_repo` to open bob-plugins and read its AGENTS.md. Edit source fragments and
rebuild main.js, never edit generated main.js. Keep each hand-edited fragment at or
below 1000 lines. All file paths below are relative to the named repository.

## 1. Enforce the period identity at the badge boundary

In bob-plugins, `241n-plugin-bank-badge-model.js::bankBadgeModel` currently validates
kind, and only a day's start date. A valid W45 report (`key: 2026-W45`, November 2–8)
with context `{kind: "week", periodId: "2026-W46", today: "2026-11-14"}` returns
`state: "badge"`. The approved design requires mismatched kind or dates to show Bank
unavailable.

Pass the requested period identity from `241nb-plugin-bank-badge-hosts.js` into the pure
model. Validate day start/end and the requested canonical week, quarter, or ISO-year
identity and bounds, reusing the existing ISO period helpers. Handle W53 and the longer
Q4 correctly. For all time, accept bob's bank-history bounds rather than comparing
against the Stats daily-note history; validate the canonical all-time identity and valid
dates. Preserve the older-bob day/week path without requiring coverage or navigation
fields.

Add regression cases for wrong same-kind period IDs and wrong date spans, including a
right key with wrong bounds, plus valid day/week/Q4/ISO-year/all-time and older-bob
reports. Repair the host tests' `reportFor` fixture: it currently uses the period ID
itself as `period.end` for non-day periods, which is not real CLI JSON. Exercise the
actual host callback so losing `periodId` in the adapter cannot silently undo the
validation.

## 2. Make the painted text match the model and wrap by whole parts

`241na-plugin-bank-badge.js::bankBadgePaintPart` appends adjacent word/amount spans
without spaces or an inner gap. The realistic concatenating FakeNode in
`test-ledger-tools-bank-badge-hosts.cjs` reproduces
`BANK·in+$2,015.00·out$1,362.56·1waiting` while the model says
`BANK · in +$2,015.00 · out $1,362.56 · 1 waiting`.

Emit real whitespace between words, amounts and separators so normalized textContent
matches `model.text`, retaining the separate color spans and aria-hidden separators. Add
CSS that keeps each money/waiting/coverage part together and allows wrapping only
between parts; never ellipsize an amount. Preserve the existing quiet colors, focus,
forced-colors, stale styling, role, keyboard activation and read-only targets. Avoid
doubled visible spacing from the outer flex gap. Fix the fake-DOM test that currently
inserts spaces in its textContent getter and therefore masks missing text nodes. Add
meaningful DOM text and wrapping assertions using realistic concatenation or a real DOM.

## 3. Subscribe when a previously hidden fence becomes visible without an observer

`dayStripObserve` and `periodStripObserve` attempt badge subscription once when
IntersectionObserver is absent. If `statsBlockViewIsShown` is false at mount, later
`dayStripCheck`/`periodStripCheck` calls never retry it. Reproduction using the
badge-host harness: mount W46 with `el.isShown = () => shown` and `shown = false`; set
shown true, call `periodStripCheckAll`, and drain. No W46 bank report is fetched and no
badge record exists until an unrelated `bankRefreshBadgeHosts` call.

Wire the existing shown/check cycles to retry badge subscription for the same fence
record, without recursion or repeated subscriptions. Cover both `bob-day-stats` and
`bob-stats-strip`, including its day delegation. Preserve no subscription before actual
visibility, no future-period request, hub/pref/ desktop gating, and unsubscribe on
unload. If the period day delegation uses a temporary day record, retain or explicitly
sync the owning fence record.

Add hidden-to-shown and detached-to-connected tests without an observer, plus the
existing observer path, unload and pref-off tests. Assert only the period's report
request; plugin startup can legitimately request the hub report.

## 4. Bound empty all-time report iteration

In bob-cli, `report.rs::all_time_period` can have start > end when the only configured
non-cash account starts in the future and there are no rows. Such configuration is
accepted by `config.rs`. `build` currently enters the day loop unconditionally for
non-summary reports and stops only at equality or chrono's maximum date; it can allocate
days far beyond the requested interval.

Ensure no day is constructed outside the inclusive start/end bounds and an
empty/reversed interval produces no days. Keep the approved all-time account/ row
history metadata and totals semantics; do not reject valid future account configuration
merely to avoid this case. Add a regression with a future account start and no bank rows
that completes promptly with empty days, plus full/ summary consistency. Test the pure
bound construction before invoking the old non-summary path, so a failing regression
cannot allocate millions of days.

## 5. Finish documentation integration and verify

Read `sase memory read docs.md -r "Need the documentation workflow"`.
`docs/reference/json-outputs.md#bank-reports` still contains only a schema/link line
despite the original report phase requiring an update. Briefly describe the new period
kinds, summary days=[], coverage and navigation, linking to the full contract in
docs/bank.md. Add the canonical `stats.bank_badge` and its boolean fallback alias
`stats.bank_chip` to the configuration reference, which currently lists neither. Keep
those reference pages consistent with the newer configuration documentation commits.

Run the existing relevant bank/period Rust tests and plugin badge/decode/data/
day/period-host tests with the new regressions. Run `npm run build`,
`npm run build:check`, `npm test`, plugin docs checks, `just docs-check`, and the
file-change gate `just check` in bob-cli. Use `/sase_monitor` for long commands;
`sase tool run check` can begin inline and its exact run must be joined if it escalates.
Never run `just check-full` for this tale. A check-full failure despite a passing check
is infrastructure, not unfinished epic scope.

The lander independently passed 104 targeted plugin tests, build:check, and
`npm run docs:check` (all gates, with 447 documentation unit tests passed and one
optional font-generation test skipped). The missing-Chromium production DOM test exits 0
with one explicit skip; do not reopen bob-cli-f1 based on the earlier phase's stale
claim. Obsidian runtime E2E is conditional in the approved plan: run it when supported,
otherwise name the missing runtime. Do not call offline/fake-DOM testing real Obsidian
proof.

Run `bob plugins sync` from the opened plugin checkout after changes. The separate
rollout task bob-cli-li owns the reproduced missing apollo plugins.sync list: chezmoi
c4e38c8a already supplies it, but the local file has not converged and its whole-file
diff also includes unrelated changes. Do not blindly overwrite home configuration or
bypass the allowlist to deploy. If the known external rollout still blocks sync, record
that concrete result and task ID; do not invent an epic code fix or claim deployment
succeeded. Follow the repo's protected-deploy rules without unapproved -F.

## 6. Finish this epic's landing in the same turn

Read bob-cli-lh, every descendant and its notes again, and the linked plan; confirm all
named original phases and linked plans are complete. Review any commits that landed
after this audit in both repositories and integrate relevant drift. The three original
PROPOSED FOLLOW-UP notes were already triaged by the lander and recorded on bob-cli-lh:
glossary fixed in phase .4; apollo rollout filed as bob-cli-li; absent runtime is an
allowed verification limitation, missing-Chromium test now skips, and historical timing
reports were attached to existing bob-cli-km, bob-cli-ki and bob-cli-ju without
inventing a fresh repro. Carry those outcomes into the close note. No new task is needed
for these proposals. The fixes in this tale are epic work.

Resolve the tale's own readiness normally if it is represented by a descendant bead,
then close the epic without waiting for this turn's code commit:

1. Run `sase bead epic-symbols bob-cli-lh`. Resolve every entry by wiring, privatizing,
   non-test pragma or deletion under the Symvision policy, or re-key only to a
   still-open later bead that actually needs it. Audit currently found none. Resolve
   entries in opened repositories too when listed.
2. Run
   `sase bead close bob-cli-lh --note "<what you verified: original phases, source and commits, this tale's four fixes, post-start drift, gates and limits, and every follow-up outcome>"`.
   If symbols block close, fix them and retry. Never use --force to make a successful
   nested landing advance. Incomplete phases must be finished or reopened;
   canceled/superseded outcomes need a deliberate --force --reason and non-done
   resolution, never a shortcut.
3. Run `just symvision` if available to confirm the whitelist is clean. The current
   bob-cli justfile has no such recipe and bob-plugins has no justfile; recheck and
   record absence if still true.
4. Open the plans sidecar through `/sase_repo`, use the PLAN path resolved by
   `sase bead read`, and change only the original epic plan frontmatter to
   `status: done` (`plan:202611/bank_badge_strip_family.md`). Do not leave this to
   another lander.
5. At audit time bob-cli-lh has no parent (`parent_id: null`). Re-read with
   `sase bead read bob-cli-lh -r "Need the parent link"` and finish normally if
   unchanged. If a link was added, a phase parent is verified and closed normally while
   its containing epic is left to its lander. For a plan parent, re-audit its previous
   landing note, all descendants/notes, linked plan and post-child drift; retire its
   symbols, close normally, run available symvision, mark its plan done, and repeat
   direct plan ancestors only while fully complete. Note and report the first incomplete
   or ambiguous parent.

Submit `/sase_final` last, with commit decisions for every changed repository. The host
commits after the turn; visible commit/push/CI evidence from this turn is not a
prerequisite to the successful same-turn closeout.
