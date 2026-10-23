---
tier: tale
title: Finish Alt+P quick roll and land bob-cli-aq
goal:
  Hide the whole quick-roll modal, explain cancel confirmation, recover stale pickers,
  and close bob-cli-aq after verification.
size: small
proposed_by: bbugyi200.athena.bob-cli-aq.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-aq
create_time: 2026-10-23 08:12:35
status: wip
---

- **PARENT:**
  [202610/quick_roll_alt_p.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/quick_roll_alt_p.md)
- **BEAD:**
  [bob-cli-aq](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-aq/README.md)

# Finish Alt+P quick roll and land bob-cli-aq

This tale completes the interrupted landing of **bob-cli-aq**, whose original plan is
`plan:202610/quick_roll_alt_p.md`. Implement the remaining work below, then close the
epic in this same turn. There is no separate tale land agent. Do not wait for this
turn's commit SHA, push, or CI result before closing; host finalization commits after
the turn ends.

## Verified scope and constraints

The land audit read the epic and every note on its four closed phases, the approved
plan, and these implementation commits:

- bob-plugins `92d5809`: Alt+P command and one-shot Task Card intent.
- bob-plugins `391277f`: shared recommendation planner, landing previews, quickRoll v1.
- bob-plugins `060395d`: review-footer preview.
- bob-cli `8da0c843`: review/API documentation and authorized footer glossary update.

Keep the accepted decisions: **Alt+P**, cancel confirmation, and optional Work summary
for Next/Pending. Use the existing Task Card recommendation writer, inbox routing,
live-target checks, and review capture/settle. Preserve the additive quickRoll v1
namespace, root API version 3, and PRE/POST preview exclusion. The approved memory edit
is already complete; this repair requires no memory changes.

At the audit, both repositories matched their refreshed `origin/master`. Since the epic
started, bob-plugins has only its three epic commits. The sole unrelated bob-cli commit
is `510c8a44` (highlight continuation migration tests/documentation); its actual diff
does not overlap the feature. Recheck for newer drift before closeout and integrate any
relevant change.

Open bob-plugins with `/sase_repo`
(`sase repo open bob-plugins -r "Finish bob-cli-aq landing repairs"`) and read its
AGENTS.md. Edit source fragments, never generated `main.js`; keep fragments at or below
1000 lines. Rebuild and deploy with `bob plugins sync` after changes.

## 1. Hide the complete modal and preserve every reveal path

In nav `350-picker-task-card.js`, `457-picker-quick-roll.js`, and `styles.css`, the
pending class currently targets **modalEl**, although the approved plan requires
**containerEl**. The outer modal container and backdrop therefore remain visible. The
existing ModalStub models only modalEl, and the current test asserts that same
incomplete behavior.

Move pending visibility to the real outer container, with a suitable fallback for
older/test hosts. Disable the entrance animation for the complete pending modal and its
children as needed. Install the pending state before the first paint. The immediate
successful path must close without revealing; Work summary, cancellation, no
priority/unavailable, inbox route/back-out, stale-target rebuild/refusal, and failed
dispatch must reveal whenever interaction is required. Keep reveal idempotent and never
leave a still-open picker invisible. Avoid revealing on the initial card render or the
initial link-target resolution rebuild.

Add a realistic container/backdrop test fixture (locally or in the shared modal harness,
with appropriate neighboring regression checks). The lander's instrumented onOpen
supplied a separate containerEl and observed `containerClasses: []` while modalEl
contained `bob-task-card-quick-pending`. Tests must fail that old behavior, check the
outer container remains hidden through a delayed immediate write, and exercise
reveal/cleanup for the interaction paths above, including Task Link resolution and a
thrown dispatch.

While editing these styles, complete the original keycap requirement: give the
`.is-alias` Alt+P keycap reduced opacity so it is quieter than Ctrl+Enter. Currently it
only changes the border to dashed.

## 2. Complete cancellation guidance and stale-picker recovery

In `457-picker-quick-roll.js`, a first press on a cancel recommendation currently
reveals the danger banner and returns without a Notice. Keep the zero-write first press,
but emit the approved guidance using `QUICK_ROLL_KEY_LABEL` and the configured
last-level label:

`Decayed past P4 — Alt+P again cancels it, Esc keeps it`

Use truthful batch wording if a counted/Task Link recommendation contains multiple
targets, preserving the all-target confirmation gate. Test the exact single-task
message, second-press cancellation, Escape with no write, and a mixed batch held without
writing. Do not introduce a separate cancel writer or bypass Work summary.

In `681-plugin-quick-roll.js`, apply the canonical `isStaleRegisteredPicker` semantics
before the active-picker no-op guard. Today a slot with `pickerOpen: true` and
`containerEl.isConnected: false` makes Alt+P return true without opening/writing, while
`openBulletPropertyPicker` replaces the same stale slot. Preserve the guard for a
genuinely live picker so its in-card alias handles the key exactly once, and preserve
Vim count consumption. Test detached and unopened stale slots, a live slot, and counted
invocation through the actual command entry point.

The current deterministic cancellation fixture is a Ready task with
`[priority:: lowest] [scheduled:: 2026-10-01]` followed by a Schedule Log containing
`*2026-10-01* — 🎲 P4 roll · in **200** (91–365) days`, using
`createPriorityPickerConfig()`, base date `new Date(2026, 8, 30)`, and random `() => 0`.
The existing quick-roll tests already contain it. On the reviewed tree the first press
returns `effects.cancel: 1`, leaves content unchanged, and has `notices: []`.

## 3. Verify and deploy

Run the meaningful quick-roll, preview, modal lifecycle, review-advance, inbox-route,
and footer regressions. Keep the existing byte-identical roll/decay/project/counted/
Task Link equivalence checks. Run bob-plugins build, full `npm test`,
`npm run validate`, and `npm run docs:check`; update its manifest patch version and
generated inventory as appropriate. If documentation changes are necessary, read the
docs reference memory first. Run `bob plugins sync` and confirm the deployed nav
version.

Use `just check` for bob-cli file-change verification. Do **not** run `just check-full`.
Use `/sase_monitor` for verification commands that cannot finish inline, with
continuation instructions preserving this tale's obligation to close the epic. The
original phase already cleared the schedule template's Alt+P override and reported a
successful vault sync; retain that binding. A graphical Obsidian smoke check is useful
if available; do not claim one from the DOM harness alone.

The lander ran the complete plugin suite on clean `060395d`: **3456/3462 passed**, with
exactly six known failures. Build/manifest validation and docs checks passed, including
447 documentation tests (one optional-font skip). The test log is
`file:explicit:0a34f570c8df4a1a3090c7f9`; consume it through `sase artifact read` if
needed. Do not turn these existing failures into extra epic work:

- Four unblocked-glyph clock-fixture failures: **bob-cli-6v**, corroborated by the
  lander; also recorded on causally owning active epic **bob-cli-5w** (phase .9,
  introducing commit `a06b403`).
- Keep notice versus missing review-journal API notice: **bob-cli-6w**, corroborated.
- Forward/reverse project conversion retaining `BLOCKS [[Work_ship#^prj]]`:
  **bob-cli-7i**, corroborated. The differently worded aq.3 report is this same defect.

All `PROPOSED FOLLOW-UP:` notes from **aq.1 #1, aq.2 #1, aq.3 #1** have been triaged
through `/sase_new_task`; the existing tasks received +1 evidence identifying the
proposing beads. Duplicate creation was declined because these are the same root causes.
No distinct outside-epic proposal remains. This outcome is already recorded on
bob-cli-aq and must appear in its close note. Any new epic-caused regression remains
this tale's work.

## 4. Finish bob-cli-aq's landing in this turn

Re-read bob-cli-aq and its four descendants through `sase bead read`, including the
lander audit and triage notes, and confirm their readiness. Re-read
`plan:202610/quick_roll_alt_p.md` through `sase artifact read`; the repairs above must
complete the original scope and accepted decisions. Check current source and any new
post-audit commits before declaring success.

Run **`sase bead epic-symbols bob-cli-aq`** immediately before closing. Resolve every
listed symbol by wiring, privatizing, adding a valid non-test pragma, or deleting it;
only re-key an exemption when an identified still-open later bead needs it. The audit
currently lists none. Never leave a whitelist entry keyed to a closing bead.

Close normally with **`sase bead close bob-cli-aq --note "<verification>"`**. The note
must record: source/commit and all-child-note review; completion of the outer modal
hiding, cancellation guidance, stale-picker fix, and alias styling; honored decisions;
post-start integration findings; actual checks/deployment and limitations; and the three
follow-up outcomes above, including the duplicate-decline reasons and bob-cli-5w causal
note. If close rejects leftover symbols, fix them and retry. If it names incomplete
descendants, finish or reopen them; never force a successful landing or use force simply
to make closure succeed.

After close, run **`just symvision` if available** (the reviewed bob-cli justfile has no
such recipe). Open the plans repository using `/sase_repo` and set **`status: done`** in
the frontmatter of **`202610/quick_roll_alt_p.md`** there. Its workspace-relative PLAN
path is `sase/repos/plans/202610/quick_roll_alt_p.md`; obtain the current resolved path
with `sase artifact path plan:202610/quick_roll_alt_p.md` and use the repository path
returned by `sase repo open plans` for edits. Include the changed plans repo in
finalization.

Finally run **`sase bead read bob-cli-aq -r "Need the parent link"`**. This epic has
**no parent_bead** in the audited record, so there is no containing phase or ancestor
epic to close. If that relationship has changed, apply the original land prompt's parent
rules: close a verified parent phase only, or fully re-audit a parent plan's
descendants, linked plan, previous landing notes and post-child drift before normal
closure; stop and note any incomplete/ambiguous ancestor. Use `/sase_final` as the last
action of the completed coder turn.
