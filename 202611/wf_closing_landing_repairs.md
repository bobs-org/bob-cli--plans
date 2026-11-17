---
tier: tale
title: Repair and finish the WF Checking closing-balance landing
goal:
  Correct historical closing balances and navigation, finish approved vocabulary and
  deployment, and close bob-cli-mg.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-mg.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-mg
create_time: 2026-11-16 19:30:48
status: wip
---

- **PARENT:**
  [202611/wf_checking_period_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/wf_checking_period_close.md)
- **BEAD:**
  [bob-cli-mg](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mg/README.md)

# Repair and finish the WF Checking closing-balance landing

## Outcome

Finish the remaining work in epic `bob-cli-mg`, then close that epic and mark its plan
done in this same implementation turn. This is a bounded repair tale for one coder, not
a new feature epic. There is no later land agent to resume this work. Do not make
closeout depend on this tale's eventual commit SHA, push, or CI run.

Read `sase bead read bob-cli-mg -r "Need landing repair scope and decisions"` and
`sase artifact read plan:202611/wf_checking_period_close.md "Need the approved closing-balance contract"`.
The epic had no `parent_bead` at the landing audit. Its four phases are closed. Its only
original phase note that explicitly deferred work was `.3`'s plugin deployment. The
landing audit found two reproducible implementation defects and incomplete glossary
wording, described below. Resolve these as epic work.

## Evidence already established

- CLI commits: `e49dbd46` (resolver/check findings), `7b26a707` (report/output),
  `3f478691` (readout docs), `a57fdba0` (glossary). Plugin commit: `1795245`.
- The intervening CLI commit `ff563ebd` adds clipboard-video capture. The plugin history
  also includes album arrangement/card work, expanded Setup headers, category emoji
  preservation, emoji search/API changes, and video navigation. The current fragment
  build includes both the album work and balance readout; no shared bank-report contract
  conflict was found in those changes. Recheck any newer drift before closing,
  preserving these features.
- All four child notes were read; there were no `PROPOSED FOLLOW-UP:` entries. No
  independent follow-up task was warranted. These repairs are caused by the epic and
  stay within it. Carry that triage outcome into the final close note.
- 61 focused plugin tests passed using
  `node --test scripts/test-ledger-tools-bank-period-*.cjs scripts/test-ledger-tools-bank-decode.cjs`.
- `npm run build:check` passed for all eight fragment-built plugins, including the
  integrated ledger build.
- `sase bead epic-symbols bob-cli-mg` returned no entries at audit time. Check again
  during closeout.
- All examples below are synthetic. Never persist real account amounts in tests, docs,
  plans, bead notes, or screenshots.

## 1. Retire superseded provisional anchors before reconstructing balances

In `src/native/bank/balance.rs`, `fill_anchors` remembers `provisional` only while
discarding earlier observations with the _same_ as-of date. `EffectiveAnchor` then loses
that evidence, so a later import with a _different_ as-of date leaves an intraday
snapshot in the reconstruction and cross-check chain. It can both misstate a settled
balance and produce a false anchor-mismatch blackout. This violates the approved rule
that a provisional anchor must not be cross-checked once a later import supersedes it.

Reproduction, using one checking account and OFX coverage starting 2026-11-01:

1. Import at `2026-11-13T12:00:00Z`, window through Nov 13, anchor Nov 13 at 10,000
   cents (a provisional snapshot).
2. Import at `2026-11-16T12:00:00Z`, window through Nov 15, anchor Nov 15 at 9,000
   cents; its batch adds a -1,000-cent posting dated Nov 13.
3. With today Nov 17, `bob bank report -D ... -f json` currently says Nov 12 is settled
   at 11,000, Nov 13 settled at 10,000, Nov 14 unverified/anchor_mismatch, and Nov 15
   settled at 9,000. The correct closes are 10,000, 9,000, 9,000, and 9,000, with no
   mismatch finding from that superseded snapshot.

Preserve import timestamps, local dates, and coverage provenance long enough to identify
supersession. Retire a provisional observation when a later qualifying OFX import
covers/finalizes its date and supplies the later usable balance evidence in that
connected span. It must cease being an effective anchor for both `B(d)` and
cross-checks; merely skipping its mismatch finding still leaves wrong settled numbers. A
later import of unrelated dates must not erase useful evidence. Retain a latest
unsuperseded intraday anchor for open-period reporting. Preserve genuine non-provisional
disputes, segment isolation, CSV attribution, health gates, and the public JSON shape.

Add meaningful resolver tests for this cross-date case, including dates before the
provisional anchor, its date, and intervening dates, plus an unrelated later import and
a genuine final-anchor disagreement. Keep the existing same-date supersession test. Add
an import/report CLI regression with the same sequence so the proof includes actual
batch timestamps and journal folding. Use explicit configured-timezone coverage in at
least one supersession test.

## 2. Supply the promised month-note fallback target

`report.rs::build_balances` only looks up the exact `(month_path, as_of)` key in
`heading_lines`. `list.rs::collect_heading_lines` includes only existing day headings,
not month-note existence. Consequently a valid month note without that heading produces
`target: null`, contrary to the approved first-line fallback. This was reproduced with a
valid `bank/2026-11.md` containing only frontmatter and a title, plus a final Nov 15
anchor: the settled close had a null target.

Pass explicit scanned month-note existence into the balance target builder (reuse an
existing scan/navigation representation where suitable; avoid inventing fake day
headings or altering unrelated navigation semantics). Use the exact day heading when
present; otherwise use line 1 of the existing month note with the same as-of date. Keep
`target: null` when no month note exists. The read path must not create notes or fall
back to `bank.md`.

Cover all three cases with report/CLI tests: exact heading, existing month with missing
heading, absent month. Keep single-day `awaiting_import` behavior as defined by the
approved resolver state table; it does not gain a number merely because an earlier day's
close exists.

## 3. Complete the already-approved vocabulary and deployment

The epic's final DECISIONS already authorize only `glossary:bank-closing` and
`glossary:stats-strip`, and explicitly reject the proposed decision record. These
authorizations are inherited from `bob-cli-mg`; do not re-ask them. Use
`/sase_memory_write` before touching memory and audited reads before consulting those
strands.

Complete the Bank Closing paragraph with the original plan's missing scope: period-note
headers only, never daily notes; an explicit `as of` label until the period end is
verified; posted rather than available balance; one account, never an account sum.
Include the planned `period close` alias alongside useful existing aliases. Make the
Stats Strip sentence explicitly say "on periods" so it cannot imply the daily member
shows a closing balance. Keep both notes short, link the relevant glossary terms, and
run `sase memory init` to publish them. Honor the declined decision-record choice.

Open `bob-plugins` with `/sase_repo` and read its `AGENTS.md` before using its files. Do
not modify vault plugin files directly. The balance readout was committed in `1795245`,
but the landing audit's clean-tree dry-run still refused to replace an uncommitted
deployed bundle based on `866bb77`, stamped 2026-11-16T23:34:42Z. That protected build
contains concurrent album arrangement work; preserve it. No force overwrite was
authorized or performed.

Refresh the linked checkout and follow `docs/develop/deployment.md` and the repo's
sync-refusal rules. If the protected bytes have landed, integrate that commit and use
the clean-sync path. If its author still runs, use `/sase_monitor` for any necessary
wait and keep concrete landing instructions in the successor. If the unlanded build is
abandoned, use the prescribed SASE question/gate flow, identifying what would be
discarded; do not improvise `-F`. Keep the epic open until the deployment obligation is
resolved.

Install the corrected CLI from this checkout with `just install` so the live readout
uses the repaired resolver, verify the installed `bob` emits `balances[]`, then sync the
up-to-date linked checkout's `bob-ledger-tools` without force. Run the deployment
guide's applicable smoke gates and report missing prerequisites or failures accurately.
A dry-run refusal or `plugins list` exit 0 is not deployment success.

## 4. Verify and finish this epic's closeout

Run `just check` for the CLI changes and `just docs-check` if docs change. Do not run
`just check-full`. Use `/sase_monitor` for long commands if needed, carrying the
remaining closeout forward in this same tale's successor. No verification step needs
this work's own commit or push to exist.

Verify the plugin build with `npm run build:check` and the focused bank-period and
decode tests above. If plugin source changes, build it through the fragment pipeline,
run its required tests/docs checks, and sync it according to its repo instructions.
Broaden tests only for changed integration paths. Do not turn the phase's historical
report of pre-existing npm environment failures into epic feature work without
reproducing and attributing a new regression.

Re-read `bob-cli-mg`, all four phase notes, this tale's notes, and the linked epic plan;
confirm every identified defect and deployment deferral has evidence of completion.
Recheck commits landed since the audit in both repositories and integrate conflicts or
duplicate behavior. Record any newly discovered independent follow-ups with
`/sase_new_task`, including their proposing bead and the disposition in the final close
note. Do not defer epic-caused defects.

The final step of this tale is the following concrete closeout:

1. Run `sase bead epic-symbols bob-cli-mg`. For every returned exemption, wire up,
   privatize, add a justified non-test pragma, or delete the symbol, or re-key it only
   to a still-open later bead that truly needs it. Leave none keyed to the closing epic
   or its phases.
2. Verify descendant and linked-plan readiness. If this tale was attached as a
   descendant, normally close its completed implementation bead first so it does not
   block the parent; never force a successful nested landing.
3. Close normally with `sase bead close bob-cli-mg --note "<verification>"`. The note
   must identify the original four phases/decisions, the provisional anchor and target
   repairs, vocabulary completion, intervening-commit review, test and deployment
   evidence, and follow-up triage outcomes. If close rejects unfinished named phases or
   exemptions, actually resolve them and retry. Never use `--force` merely to make the
   close succeed.
4. After close, run `just symvision` if that recipe is available. It was absent in the
   audited CLI justfile; state that fact if still absent, rather than claiming a pass.
5. Open the plans repository through `/sase_repo`, resolve
   `plan:202611/wf_checking_period_close.md`, and set `status: done` in that file's
   frontmatter. Use only the path returned for this coder's workspace. No phase-4 memory
   edit substitutes for this plan status update.
6. Re-read `sase bead read bob-cli-mg -r "Need the parent link"`. There was no parent at
   audit time, so normally finish here. If a parent was added in the meantime, apply the
   landing prompt's parent rules: close only a fully verified parent phase; for a
   directly parented plan, review its earlier landing note, all descendants/notes,
   linked plan, and drift, retire its exemptions and close normally only if fully
   complete, run available symvision, mark its plan done, then repeat for plan
   ancestors. Stop and note the first incomplete or ambiguous parent. Never force those
   closures.

Finish through `/sase_final`, including every repository changed by this tale in the
host's final declaration.
