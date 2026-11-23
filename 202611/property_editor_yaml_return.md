---
tier: tale
title: Preserve the property row through YAML Done and finish the nested landing
goal:
  YAML Done retains the selected occurrence or nearest surviving row, acceptance proves
  it, and the repair epic and its completed parents close normally.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-pf.4.1.7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-pf.4.1.7
create_time: 2026-11-23 11:33:18
status: wip
---

- **PARENT:**
  [202611/property_editor_landing_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_editor_landing_repairs.md)
- **BEAD:**
  [bob-cli-pf.4.1.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/bob-cli-pf.4.1.7.md)

# Preserve the property row through YAML Done and finish the nested landing

Finish the remaining work in epic `bob-cli-pf.4.1.7`, then complete its nested landing
through plan `bob-cli-pf.4.1` and phase `bob-cli-pf.4`. This is a single bounded repair,
acceptance, and closeout tale. It has no land agent; its coder must perform the closeout
below in the same work stream, without waiting for its own commit SHA, push, or CI.
Leave containing epic `bob-cli-pf` to its waiting lander.

## Context and verified baseline

Read the epic notes and these plans through audited commands:

- `sase bead read bob-cli-pf.4.1.7 -r "Recover landing audit and follow-up dispositions"`
- `sase artifact read plan:202611/property_editor_landing_repairs.md "Read the approved repair scope"`
- `sase artifact read plan:202611/property_editor_core.md "Preserve the core editor contract"`

The landing audit reviewed all notes on the four repair phases and the parent's six
original phases, current source, epic commits, and subsequent changes. On bob-plugins
`b2c37102`, all **306** property and harness tests pass:

```sh
node --test scripts/test-ledger-tools-property*.cjs scripts/test-obsidian-property-editor-e2e.cjs
```

The write acceptance report `file:explicit:8fc60a0e749f3dac06629c1b` records 75 passes,
one allowed missing-navigation skip, 36 exact-byte checks, no plugin errors, and the
previously verified Rust canonical-record ownership fix `de66b399`. The newer surface
report `file:explicit:49ee66906e8ea8acb9a84f58` records 129 passes, no skips, 37 byte
checks, and no plugin errors on bob-plugins `26613027` and bob-cli `37b2080e`. Its
navigation-loaded run also covers the prose band. The completed implementation already
contains the write serialization, session registration, guarded history, overlays,
old-bob handling, recovery diagnostics, value fidelity, focus, and shell repairs.
Preserve them; do not restart those phases or repeat unchanged acceptance without a new
reason.

The reports do **not** prove the following remaining requirement:

1. In `plugins/bob-ledger-tools/src/298m-plugin-property-editor-entrypoints.js`,
   `propertyEditorStartYamlExcursion` records `selectedKey` but loses its source line
   and duplicate ordinal. `propertyEditorYamlDoneAttempt` calls
   `propertyEditorNearestId` with `ordinal: 0, line: null`, then opens by key only. A
   deleted selection therefore returns to the first row. Duplicate-key Done similarly
   loses the selected occurrence. The existing shared excursion record in
   `242e-plugin-album-yaml-excursion.js` already has a `selection` field that can carry
   this information without changing the props wire protocol.
2. The real surface report itself demonstrates the defect: after deleting `keep_me`, its
   assertion named “Done lands on a nearest remaining row” shows `type` at line 2
   selected and `edit_me` at line 3 unselected. The deleted `keep_me` occupied line 3;
   `edit_me` is the correct nearest survivor. The assertion at approximately line 251 of
   `scripts/obsidian-property-editor-e2e-surfaces.cjs` only requires **some** selection.
   The unit test “Done saves, restores the leaf mode, and reopens the surviving key”
   also labels a first-row fallback as nearest without recording the original line.
3. A direct probe using the existing entrypoint test helpers on `b2c37102` started YAML
   on the second `repeat` at line 9 and produced
   `{yamlLine:9,savedSelection:null,doneOpens:[{filePath:"notes/groceries.md",key:"repeat"}]}`.
   Thus the outbound jump works, but the return cannot retain that occurrence.
4. The surface report's screenshot `visualVerdict` values still say
   `pending inspection`. The harness marks “reviewed every screenshot” using geometry
   probes alone. Actual manual review exists in phase `bob-cli-pf.4.1.7.4` note #7;
   preserve that evidence and make new report claims distinguish automated layout checks
   from an actual image review.

Follow-up triage is **already complete** in the epic's LAND FOLLOW-UP TRIAGE note:
startup and shell proposals were fixed in this epic; missing task-drawers nav was fixed
by bob-cli `f6cf62cf`; task_reqs fixture drift was reproduced and attached as a
DISCOVERED ISSUE to active `bob-cli-ph` (reader phase `bob-cli-ph.2`); timing flakes
corroborated `bob-cli-km` and `bob-cli-io`. No new task is needed for those. The parent
landing note already triaged its earlier proposals, including `bob-cli-pg`. Do not
expand this repair into unrelated drawer work.

The integration audit covered bob-cli changes from `7703011d` through `f6cf62cf` and
bob-plugins `eb236b40`, `ddbf8bd9`, `ea00e182`, `c8846754`, and `b2c37102`: task
requirements/drawers, successor concurrency, docs, and additive tick network
classification introduce no necessary property-editor adaptation. Recheck only new drift
before closeout. There were no epic-symbol exemptions at this audit.

## 1. Preserve selection through the complete excursion

Open `bob-plugins` with `/sase_repo` and read its AGENTS.md. All product paths in this
tale are relative to that printed checkout. Edit source fragments and run the committed
build; never hand-edit `main.js`. Keep every hand-edited fragment at or below 1000
lines.

Add failing-first regressions to the existing entrypoint/row-fidelity suites:

- Select a middle or late property, enter YAML, delete it externally, then Done; choose
  the nearest surviving source position, deliberately different from the first row.
- Select the second occurrence of a duplicate key, enter YAML, then Done without repair;
  retain that occurrence and keep recovery read-only. Include a source line shift so
  this cannot be solved by an obsolete `key::line` alone.
- Retain ordinary same-key return, original leaf mode, failed save refusal, and
  stale-excursion no-reopen behavior.

Capture `{key, ordinal, line}` from the selected row **before** closing the modal; carry
it through the props excursion record and Done. Reuse `propertyEditorSelectionOf` /
`propertyEditorNearestId` from `298o` rather than introducing a second selection
algorithm. On the fresh report, resolve by key and duplicate ordinal, then nearest
source position if the key vanished. Preserve that resolved identity through cache-first
reopening and report reconciliation; passing only `key` to the opener cannot distinguish
duplicates. An internal optional selection argument is fine; preserve the public
`openPropertyEditor({filePath,key,mode})` contract and its callers. Keep parse-error
targeting distinct from the selection used for return. Legacy excursion records without
selection metadata may fall back safely.

No change to bob's props envelope, Cargo version, changelog, or memory is needed. The
inherited decisions remain final: bank foot enabled, drawer state per note; no memory
edits here. pf.5 retains rename/reorder/multi-select/clipboard/filter/ kind
conversion/YAML value stages; pf.6 retains native Properties takeover.

## 2. Strengthen and verify acceptance

Replace the permissive nearest-row assertion with the exact expected key, occurrence,
and focus. Add a real-Obsidian duplicate-return case. Keep these in the existing harness
and update its offline tests; do not substitute fake-bob success for the actual UI.
Preserve valid old reports as evidence for unchanged requirements.

Rename the automated screenshot-review assertion to describe the layout checks it really
performs. Following a new real-app run, inspect each new PNG and attach an explicit
per-image review to a reviewed report (including the image hash), then register that
report and images through `/sase_artifact`. It must not claim manual review while
verdicts remain `pending inspection`. Existing four-image observations in .4 note #7 may
substantiate unchanged old images; they do not review new images. Keep raw evidence
alongside reviewed evidence.

Run the focused property/entrypoint/harness suites, `npm run build`,
`npm run build:check`, `npm test`, and `npm run validate`. Read docs memory before any
necessary adjustment to `docs/develop/validation.md`; run `npm run docs:check` if docs
change. Apply the approved handling for km/io/pg timing flakes: verify the same failing
nodes in isolation and cite their trackers. Distinguish unrelated drawer rollout
failures (already attached to ph) from regressions in this repair. Do not suppress or
misreport failures.

Run real Obsidian on **athena**, using the scratch vault/profile procedure and flags
documented in `docs/develop/validation.md`. Use `/sase_memory_read` for tailnet
instructions before SSH and `/sase_monitor` for long commands. Use the current source
tree and an explicit bob binary with recorded SHA/hash provenance, never Bryan's live
vault. A focused added-scenario run is sufficient when it uses the same real harness and
preserves the earlier passing coverage; otherwise run the full harness. Review/register
its evidence before closeout.

After plugin edits run `bob plugins sync` as required by AGENTS.md; follow its overwrite
protection procedure without `-F`. Run **`just check`** as file-change verification from
bob-cli, using the named SASE check tool/monitor workflow when available. Do not run
`just check-full`. A check-full failure after a check pass would be test infrastructure,
not remaining epic work. Investigate any `just check` failure against the clean base and
attribute unrelated failures to their owner; the existing parent plan allows evidenced
clean-base failures to be recorded without holding a completed phase open.

## 3. Finish this landing and the directly linked parents

This is the final step of the tale. Perform it after verified code and evidence,
**before** requiring this tale's own commit/push/CI. If this tale has an assigned bead
beneath an ancestor being closed, complete and close that tale bead normally first so it
cannot block the ancestor. Do not wait for a land agent for this tale.

1. Reread `bob-cli-pf.4.1.7`, all four closed children and notes, and the new tale
   evidence. Confirm every original repair requirement and DECISION is honored. Recheck
   changes landed after the audit's bob-cli `f6cf62cf` / bob-plugins `b2c37102`
   baselines for integration. Run `sase bead epic-symbols bob-cli-pf.4.1.7`; resolve
   each entry (wire, privatize, non-test pragma, or delete per policy), or re-key only
   to a still-open later bead that needs it. Then close normally:

   ```sh
   sase bead close bob-cli-pf.4.1.7 --note "<verified selection fix, exact acceptance evidence, prior repairs, integration review, and every recorded follow-up disposition>"
   ```

   Run `just symvision` where available; record absence of that recipe honestly. Open
   the plans repo with `/sase_repo` and set only `status: done` in frontmatter of
   `plan:202611/property_editor_landing_repairs.md` (the PLAN path printed by the
   audited bead read, relative path `202611/property_editor_landing_repairs.md` under
   the opened plans checkout).

2. Inspect the parent link with
   `sase bead read bob-cli-pf.4.1.7 -r "Need the parent link"`. It is plan bead
   **`bob-cli-pf.4.1`**. Reread that parent's prior LAND VERIFICATION / TRIAGE notes,
   all descendants and their notes, and `plan:202611/property_editor_core.md`. Rerun
   descendant and linked-plan readiness checks and inspect post-child drift. If anything
   remains incomplete or ambiguous, note the exact blocker on `bob-cli-pf.4.1` and stop
   ancestor closing; never force a successful nested landing.

   When complete, run `sase bead epic-symbols bob-cli-pf.4.1` and retire/re-key its
   entries under the same rule, then:

   ```sh
   sase bead close bob-cli-pf.4.1 --note "<rechecked original six phases, child repair evidence, all descendant readiness, decisions and post-child integration>"
   ```

   Run `just symvision` if available and mark `202611/property_editor_core.md` in the
   opened plans repo `status: done`.

3. That plan's parent is phase **`bob-cli-pf.4`**. Read its scope and inherited
   decisions, confirm this completed core editor satisfies it, run
   `sase bead epic-symbols bob-cli-pf.4`, resolve/re-key any entries, and close only
   that phase normally:

   ```sh
   sase bead close bob-cli-pf.4 --note "<verified complete editor-core and real-app evidence; parity features remain pf.5 and native takeover remains pf.6>"
   ```

   Leave containing epic **`bob-cli-pf`** to its already-waiting land agent.

Closing never cascades. If a close names unfinished descendants or stale symbols, finish
the work and retry normally; never use `--force` to advance this successful landing. A
deliberate canceled/superseded outcome requires its own actual reason, not a shortcut
around a failed close. Finish through `/sase_final` with commit decisions for every
changed repository, including bob-plugins and the plans repo.
