---
tier: tale
title: Finish intention card state and land bob-cli-nh.7.1
goal:
  Repair the remaining intention card state and field-stage gaps, verify and deploy
  them, then close bob-cli-nh.7.1 and parent phase bob-cli-nh.7 normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-nh.7.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-nh.7.1
create_time: 2026-11-20 08:09:27
status: wip
---

- **PARENT:**
  [202611/intention_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/intention_card.md)
- **BEAD:**
  [bob-cli-nh.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nh/bob-cli-nh.7.1.md)

# Finish the intention card state paths and land bob-cli-nh.7.1

Complete the bounded gaps found while landing the intention-card child epic, then close
**bob-cli-nh.7.1** and its direct parent phase **bob-cli-nh.7** in this same coding
turn. This tale owns the complete remaining landing; there is no later land agent for
this tale. No step depends on this work's future commit SHA, push, or CI. Leave the
containing epic **bob-cli-nh** to its already-waiting land agent.

## Scope and verified baseline

Read `sase bead read bob-cli-nh.7.1 -r "Need the landing audit and triage"`, its notes,
and `sase bead read bob-cli-nh.7 -r "Need the delegated phase scope"`. Approved context
is `plan:202611/intention_card.md` and the card section of
`plan:202611/period_intentions.md`; read both with `sase artifact read`. This is
remaining implementation work, not a redesign of the four closed phases.

The lander audited all phase notes and plugin commits d642146 (display), d90ebda
(editor), ea47caf (navigation), and b1a655a (acceptance fixes). The acceptance report
`file:explicit:1919355ed6090b82b47ff817` records 65/65 mandatory real Obsidian checks,
zero console errors, 11 actual writer applies, and 37 reviewed screenshots. It used
bob-cli d15432aa and the then-uncommitted plugin acceptance changes. Preserve that
evidence; it does not cover the defects below.

At this audit, bob-cli HEAD and origin/master are e5f51081 and bob-plugins HEAD and
origin/master are b1a655a. Fresh focused card/host/ownership/registry tests pass 97/97,
with one explicitly skipped real-writer test because BOB_BIN was not supplied;
`npm run build:check` passes. No fresh full gate is claimed.

Runtime probes against b1a655a are preserved as
`file:explicit:6b496c43f135269d2bbcaada`: an overlay stores `[Saved]` but the display
model still reads `[Old]`; a metadata event leaves the overlay stored; retrying a
retained `intentions.remove` instead constructs `intentions.replace`; a restored drafted
summary keeps modal state `edited`; Undo runs with `pendingApply: false`. The probe's
`expected: null` on retry is the retained-op slot after the implementation cleared it,
not the intended removal operation.

Open bob-plugins through `/sase_repo`; use only the returned path and read its
AGENTS.md. Paths in steps 1–3 are relative to that linked checkout. Edit source
fragments and regenerate main.js; handwritten fragments must remain below 1000 lines
(297c is already near the cap). No memory changes, Bob release-version changes, vault
content migration, or user-authored intentions/reflections. The approved daily-summary
branch remains 1–3 sentences up to 60 words. Read
`decisions:q-and-ctrl-bracket-mirror-esc` through `/sase_memory_read` and preserve the
shared one-layer close dispatch and q's text-control exception.

## 1. Wire successful-write display state through the real host

The overlay store in `297d-plugin-intention-card-entrypoints.js` is currently write-only
outside its unit tests. `296c-plugin-intention-card-host.js` neither consumes it in
`intentionCardModelFor` nor calls `intentionCardSettleOverlay` in
`intentionCardOnMetadata`. Follow the established record-sheet overlay pattern:

- An apply-confirmed, path-scoped overlay must bridge stale metadata in the actual
  Reading and Live Preview display model, including the first field added to an empty
  note and removal of the last visible field. Never display a pending preview as saved
  or override invalid/source/embed/mobile gates.
- Preserve the summary's authoritative reported provenance in the overlay, including
  Undo; do not pretend an old digest belongs to newly edited text. Keep passive painting
  independent of bob subprocesses.
- Settle on matching metadata, discard on a later conflicting owned-field update, and
  expire with the existing bounded timer. Do not discard a just applied overlay merely
  because the metadata still describes the prior snapshot. Pin asynchronous source reads
  to the path and invalidation epoch so a late old read cannot resurrect stale content
  after settlement/unload.
- Add integration-level host tests with deliberately delayed metadata. Test match, later
  conflict, expiry, parent refresh, first-add and last-remove; calling only the
  standalone setter/settler is insufficient.

## 2. Finish guarded retry, Undo, and report reconciliation

Repair the existing modal/session integration in 297a/297b/297c/297ca, retaining the one
shared guarded writer and the original server fingerprint semantics.

- `intentionRetry` currently discards `retainedOps`, opens a field stage, and saves its
  value. A stale remove or move thereby becomes a replacement. Preserve the intended
  operation and the exact text draft where relevant. Show its target against refreshed
  fields and require an explicit reviewed retry. Never silently apply an old index to a
  changed list without showing what will happen. Test stale remove and reorder,
  save-blocked retry, and a list insertion/deletion between preview and retry. A
  canceled draft must not leave a hidden retained operation that a later Enter can
  execute.
- `intentionUndo` must enter and leave the same busy state as other writes, with
  try/finally cleanup. Prevent a second mutation, navigation, or closing through either
  keyboard or pointer dispatch from superseding an in-flight apply. Test deferred Undo
  and deferred normal apply with competing keys and clicks, target pinning, and
  transport failure recovery.
- Consolidate field reconciliation for initial reads, apply, stale refresh, and Undo.
  Read `after.summary_state` (`drafted` maps to the UI's `agent`) instead of retaining
  the previous state. Null/unset summary and reflection must clear cached modal text.
  Restored stamps/provenance must appear without requiring a close/reopen, with dates
  refreshed from authoritative metadata when the report does not carry them.
- Honor the original empty-value contract: clearing an existing intention uses remove,
  clearing summary/reflection uses unset, and a blank Add must not create an invalid
  value. Keep real validation errors distinct from an unsupported old-bob command and
  preserve recoverable drafts on refusal.

Use controlled promises and real guarded sessions in regression tests, then a real
current BOB_BIN scratch round trip for these writes, stamped Undo, stale retry, and
body-only-edit preservation. Do not change the Rust protocol merely to hide a UI state
bug.

## 3. Complete the approved field-stage behavior

The reflection textarea in `intentionCardEditorPaintAll` currently has manual vertical
resizing only, and its summary context is rendered only for the summary stage. Render
the current summary read-only above Reflection and make the textarea grow with content
on entry, input, paste, and repaint while keeping the caret stable. Preserve
Enter/Shift+Enter/Ctrl-or-Cmd+Enter behavior.

The approved Alt+J/K reorder calls for shared arrange animation; current code computes
`reordered` and discards it, and never invokes the shared helpers. Wire
`schemaArrangeCollectRects`/`schemaArrangePlayFlip` (or their current equivalents)
around successful repaint with stable row identities, preserve selection, and honor
reduced motion. Do not animate a refused write as success. Test actual DOM behavior and
inspect light/dark/narrow editor screenshots for the reflection context/growth and
reorder state. Keep the rest of the accepted display and navigation behavior intact.

## 4. Integrate, verify, and deploy

Recheck commits since b1a655a and e5f51081, including origin/master, and integrate only
changes that interact with this work. The lander already audited the post-start CLI
relative-glob fix 3acbc1a3, intention-link rewrite f811397d, log recall d15432aa, and
docs rename e5f51081; plugin registry cleanup d99dcef, log recall
7eb0685/3d1f553/727cdc8, and command-chip rename/engine/grammar bf13f43/591998e/c8da83f.
The intention command rows and task-priority fallback survive; those changes require no
new intention feature. Keep the newer Command Mode APIs and chip files when regenerating
bundles.

Build, run `npm run build:check`, focused intention/host/ownership/registry/YAML and
command-entrypoint regressions, the real-writer tests with a current BOB_BIN, and
`npm run validate`. Add the newly discovered cases to the actual
`obsidian-intention-card-e2e` runner, run it in its isolated scratch vault under Xvfb,
inspect screenshots, and preserve its report as an audited artifact. Run the relevant
existing regressions when touching shared helpers. Run `npm test` and mounted
`BOB_CLI_DIR=<current checkout> npm run docs:check` as the approved plan requires; keep
unrelated, reproduced baseline failures separate. Read `docs.md` before any
documentation change; update `docs/intention-card.md` only where the corrected behavior
needs explanation and generate inventories.

File-change verification is `just check` in bob-cli. Do not run `just check-full`. Use
`/sase_monitor` for commands that exceed the inline limit, retaining the complete
closeout continuation in this tale's coder session. A check pass plus a check-full
failure is test infrastructure, not remaining feature work.

Deploy the edited plugins from this exact linked checkout with a dry run then real
`bob plugins sync --repo "$PWD" --no-pull`, following deployment.md's landed-build
recovery, never forcing a protected build. Verify source/stamp/vault hashes and run the
documented deployed-vault startup smoke gate. Do not require this turn's future commit
for closeout: a dirty stamp with matching authored source hashes is acceptable evidence
of the final deployed bytes; host finalization will commit the implementation afterward.

An existing unrelated rename issue is recorded on bob-cli-nu: the host allowlist can
still name bob-copy-chips after bf13f43 renamed it to bob-chips. Recheck whether it has
landed; if the broad sync is blocked by that known issue, deploy the changed intention
plugins explicitly (for example `-p bob-ledger-tools`) and record the exact broad-gate
limitation on its owner rather than forcing or claiming a full pass. Do not leave an
intention-specific deployment failure unresolved. Keep inherited real acceptance
evidence and fresh results distinct.

## 5. Finish the epic and parent phase in the same turn

The lander completed follow-up triage before proposing this tale. Carry these
dispositions into the final epic close note; do not file duplicates:

- nh.7.1.3 #1 and nh.7.1.4 #4: architecture URL -> existing bob-cli-nv, corroborated.
  The /bob-copy-chips/ URL half is already fixed by e5f51081 and bf13f43 and was
  declined as completed.
- nh.7.1.3 #2: album writer held-lock self-test -> existing bob-cli-n0, corroborated.
- nh.7.1.4 #5: timing nodes -> existing bob-cli-km, bob-cli-io, and bob-cli-m1, each
  corroborated; no new umbrella task.
- nh.7.1.4 #6: queue-card missing-import assertion -> DISCOVERED ISSUE on active
  bob-cli-no, causally owned by no.3's footer-chip change c3ca244.
- nh.7.1.4 #7: malformed empty-op read documentation -> DISCOVERED ISSUE on parent
  bob-cli-nh, owned by nh.2. The alternative API behavior change was not adopted without
  contract review.

After verifying the remaining repairs and all four original phases, re-read the
epic/children and confirm no new note or descendant changes the outcome. Run
`sase bead epic-symbols bob-cli-nh.7.1`. Resolve every reported entry or re-key it only
to a still-open later bead that truly needs the exemption. The audit found none, but do
not assume it remains so. Then close normally:

```sh
sase bead close bob-cli-nh.7.1 --note "<source/commit audit, repaired gaps, integration and verification evidence, deployment evidence, and every follow-up disposition above>"
```

If symbols or unfinished descendants reject close, resolve them and retry. Never use
`--force` to advance this successful nested landing. Run `just symvision` if the recipe
is available; the audit found no recipe in the current bob-cli justfile, so explicitly
record unavailability if still absent. Open the plans repo through `/sase_repo`, use the
printed path, and set `status: done` in the frontmatter of **202611/intention_card.md**,
the epic's linked plan. Read artifacts through `sase artifact read`, not direct reads.

After the epic closes, run `sase bead read bob-cli-nh.7.1 -r "Need the parent link"` and
verify the parent is still **bob-cli-nh.7**, a phase bead. Re-read that phase, verify
this completed child plan fulfills its complete card/editor/commands/deploy scope,
retire any `sase bead epic-symbols bob-cli-nh.7` entries, and close **only** it:

```sh
sase bead close bob-cli-nh.7 --note "<verified completion of the child plan and the original card/editor/commands/deploy phase, with evidence refs>"
```

Leave **bob-cli-nh** and **202611/period_intentions.md** open for their existing land
agent; do not force or close that containing epic. If the parent link or readiness is
unexpectedly ambiguous, note the concrete blocker on the parent and report it. Otherwise
finish normally through `/sase_final`, declaring the plugin implementation and plan
status edits for host commit in this turn.
