---
tier: tale
title: Finish the Day strip typical-day landing
goal:
  Share the usable-day rule, recover the approved glossary update without losing the
  bars switch, and close bob-cli-cn.
size: small
proposed_by: bbugyi200.athena.bob-cli-cn.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-cn
status: done
---

- **PARENT:**
  [202610/day_strip_typical_day.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/day_strip_typical_day.md)
- **BEAD:**
  [bob-cli-cn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-cn/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-cn.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-cn.land.md)
- **COMMITS:**
  - [29b3884](https://github.com/bobs-org/bob-cli--plans/commit/29b3884db3bc673d71ed728039e2b7f0d4fedb72)
    — chore(plans): mark day strip typical-day plans done

# Finish the Day strip typical-day landing

Finish the two bounded gaps found while landing epic `bob-cli-cn`, then close that epic
in this same implementation turn. Its approved plan is
`plan:202610/day_strip_typical_day.md`, stored at
`sase/repos/plans/202610/day_strip_typical_day.md` in the active checkout. This tale has
no later land agent. Do not wait for this turn's commit, SHA, push, or CI result before
closeout; the host commits after the turn ends.

## Verified context

The lander read the epic and every note on phases `bob-cli-cn.1`, `.2`, and `.3`, all
closed. It reviewed the Rust computation, serialization, golden and series tests, plugin
model/painter/cache/stylesheet, and the implementation commits: `3f417646` and
`bd9c7a8c` in bob-cli and `cc5d045` in bob-plugins. The model paints bob-owned values
and sentences, preserves blank reference lines, suppresses today's marks, rejects
grouped default reports, and repaints on reference changes.

All 108 focused plugin tests passed on bob-plugins `0bf8920`:

```sh
node --test scripts/test-ledger-tools-day-strip.cjs scripts/test-ledger-tools-day-strip-hosts.cjs scripts/test-ledger-tools-day-strip-lanes.cjs scripts/test-ledger-tools-day-strip-themes.cjs scripts/test-ledger-tools-day-strip-plan-host.cjs
```

Phase notes record read-only live parity for October 6, 22, 25, and 26; the Chromium
matrix passed 246 cells; plugin 1.77.0 was synced. Blank lines with an older installed
bob are the approved compatibility state until the daily release train ships the
additive field. Do not install a local bob or alter its version or changelog to bypass
that release policy.

Drift review covered bob-cli through `c4922e66` and bob-plugins through `0bf8920`,
including changes since epic creation at 2026-10-26 13:12 EDT. The bars switch
(`c0489557` / `b70cb00`) must survive the glossary repair. The command-mode cache
refresh, dependency-reader cleanup, relation-index coalescing, and freshness placement
changes do not need typical-day calculations; the latter changes do not modify the
reference model. Check any newer drift before closing.

The epic's final reviewer DECISIONS remain authoritative: `glossary_day_strip=yes`
authorizes `glossary:day-strip`; `decision_record=no` declines the decision strand. This
is completion of that accepted glossary edit, not a new memory choice. Use
`/sase_memory_read` and `/sase_memory_write` before its repair, and read `docs.md`
before regenerating documentation.

## 1. Actually share the usable-day predicate

`src/native/stats/series.rs` defines `is_usable_value`, `DAY_WINDOW`, and
`DAY_WINDOW_MIN`. Rolling medians call the helper, but
`src/native/stats/typical.rs::add_typical_days` independently pattern-matches
`(Some(value), false)` from a copied date map. That contradicts phase 1's explicit
single-rule requirement and the helper's comment.

Make the typical-day input selection call `is_usable_value` too. A date-to-value map
filtered through the helper is sufficient; keep calendar selection of D−7 through D−1
and the existing threshold and rounding. Preserve all JSON fields, words, omission
rules, and rolling-median behavior.

Use the existing hand-derived tests. Add one focused case with an actual missing
calendar date and a provisional historical outlier: an out-of-window D−8 value must not
enter the window; a provisional earlier point must not contribute; real zero and partial
data must count. Hand-derive the median and usable-day count. The existing long-series
parity and short-history tests should remain green.

## 2. Recover and integrate the approved glossary update

Phase 3 reported commit `5d9bbead6ffc7d332e7371d927267d3ae1eab15f`, whose parent is
`bd9c7a8c`. The object exists locally, but neither local `master` nor fetched
`origin/master` contains it. `sase memory read glossary:day-strip` consequently still
lists only facts and lanes. The reported commit also replaces the already landed
bars-switch wording with a fixed two-lane sentence, so blindly applying it would lose
concurrent work.

Read current memory through its audited command. Repair only the existing contents
sentence in `sase/memory/glossary/day-strip.md`: describe the quiet typical line under
each numeric fact, the median of the previous seven days, and ▲/▼/= once the day is
finished, while retaining the root-parent/theme lanes, full-width bars, the
together-or-one-at-a-time bars switch, and the timestamp on the top lane row. Retain the
current links and placement. Read the commit diff as evidence, but reapply the merged
intent instead of restoring its generated files wholesale.

Run `sase memory init`, `just docs-gen`, and `just docs-check`. Review generated diffs
so the glossary and source remain consistent and the bars switch stays documented. The
old phase-3 commit also regenerated existing decisions documentation; regenerate from
current canonical inputs rather than adding a decision strand.

## 3. Verify the repair and final readiness

Run `just check` (the named `sase tool run check` wrapper is appropriate). Never run
`just check-full`. If a named run escalates, use `/sase_monitor` to join that same run
and carry the closeout instructions below into its continuation.

The lander's clean-tree `just docs-check` failed only at
`src/native/highlights_ref/source_store.rs:194`, undocumented production environment
name `BOB_TEST_FIXTURE_PDF`. This belongs to active epic `bob-cli-cc`; its landing note
#5 owns the exact exception. The cn lander independently reproduced it and added a
`DISCOVERED ISSUE` note there. Use any fix that has since landed. If still present,
record that unrelated gate failure accurately; it is not remaining cn work. Any new
failure caused by this tale must be fixed before closing.

Open bob-plugins with `/sase_repo` for any renewed inspection. The focused plugin tests
already passed and no plugin change is expected. If newer drift or a source change
warrants rerunning them, use the command above; run build/docs/preview and
`bob plugins sync` if you actually change that repository. Do not regenerate or sync an
unchanged plugin merely to repeat the earlier phase.

Reread the epic and its three children for new notes, confirm their closed status, and
inspect newer bob-cli and bob-plugins commits relative to the reviewed tips. Do not
reopen completed phases merely to make these two landing repairs.

## 4. Close bob-cli-cn in this turn

Follow-up triage is finished and recorded on `bob-cli-cn`; carry every outcome into the
close note:

- `cn.1` #2 (verified-field build failure) and #4 (catalog, completion, and source-store
  test failures): fixed by landed `e9a955f9`; no new tasks.
- `cn.1` #3, `cn.2` #2, `cn.3` #3: one BOB_TEST_FIXTURE_PDF docs lint, routed with fresh
  evidence to its causal active epic `bob-cli-cc`; no duplicate task.
- `cn.3` #1: decline a new decision-record task because the human reviewer chose
  `decision_record=no`. Existing `bob-cli-bs` note #3 satisfies the selected plan branch
  by recording the need for Bryan's re-scope. Leave bs open; the generic original "done
  means bs resolved" does not override that explicit branch.
- `cn.3` #2: task `bob-cli-cp` (feature, small) created and snoozed until 2026-11-16
  09:00 America/New_York. It contains the two exact feedback questions, the optional
  marks question, the predefined plugin-only responses, and a check that three weeks of
  visible use actually elapsed after the bob release.

Run `sase bead epic-symbols bob-cli-cn`. It currently reports no entries. For every
entry that now appears, wire up, privatize, give a justified non-test pragma, or delete
the symbol per the Symvision policy. Re-key its Justfile exemption only when a concrete
still-open later bead needs it. Do not leave exemptions keyed to cn or its phases.

Then close normally:

```sh
sase bead close bob-cli-cn --note "Verified all three phases and their notes, additive typical JSON and plugin rendering/cache behavior, live-parity and preview evidence; integrated post-start bars-switch wording and restored the approved glossary update; typical and rolling medians now share the usable-day rule. Recorded actual check and docs-check results. Follow-ups: cn.1 #2/#4 resolved by e9a955f9; cn.1 #3, cn.2 #2, cn.3 #3 routed to causal epic cc; cn.3 #1 declined per human decision_record=no with bs re-scope note retained; cn.3 #2 tracked by snoozed cp. Epic symbols resolved and descendant readiness rechecked."
```

Replace the generic check-results clause with actual outcomes and name any newer
integration work. If close reports leftover symbols, finish their cleanup and retry. If
it reports incomplete named phases, inspect and finish/reopen the actual work. Never
force a successful landing; `--force` is only for a deliberate canceled or superseded
outcome with a reason.

After close, run `just symvision` if the recipe exists. There is no such recipe at the
reviewed tips; record that explicitly if still absent. Open the plans sidecar with
`/sase_repo`, use its returned path and audited artifact reads, and set only
`status: done` in the frontmatter of the epic's linked plan,
`202610/day_strip_typical_day.md` (the PLAN path above). Include that sidecar change in
`/sase_final`'s repository obligations.

Finally run `sase bead read bob-cli-cn -r "Need the parent link"`. It has no
`parent_bead` at review time, so no ancestor closure is required. If a parent has since
been attached, follow the original landing rules: close only a verified parent phase
normally; for a parent plan, re-audit all descendants, notes, linked plan readiness and
drift, retire its symbol exemptions, and close/mark done only while fully complete. Stop
with a note at the first incomplete or ambiguous parent. Submit `/sase_final` as the
final action of the completed turn.
