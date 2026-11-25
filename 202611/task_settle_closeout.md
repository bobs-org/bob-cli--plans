---
tier: tale
title: Repair taskSettle guards and finish the provider landing
goal:
  Reject stale task identities and unguarded staged writes, enforce fixture parity,
  integrate docs, and close bob-cli-q4.8.1 and parent phase bob-cli-q4.8.
size: medium
proposed_by: bbugyi200.athena.bob-cli-q4.8.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-q4.8.1
create_time: 2026-11-25 13:53:34
status: wip
---

- **PARENT:**
  [202611/task_settle_twin.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_settle_twin.md)
- **BEAD:**
  [bob-cli-q4.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q4/bob-cli-q4.8.1.md)

# Repair taskSettle guards and finish the provider landing

Complete the remaining work for epic **bob-cli-q4.8.1**, then close its parent phase
**bob-cli-q4.8**. This tale has no land agent. Its coder must perform the closeout below
in the same turn as its final code, before host finalization; never wait for this work's
own commit SHA, push, or CI to close these beads. Do not close the containing epic
**bob-cli-q4**.

## Context and fixed boundaries

Read `bob-cli-q4.8.1` and `bob-cli-q4.8` with `sase bead read`, and read
`plan:202611/task_settle_twin.md` through `sase artifact read`. The original epic's
three phases are closed. The lander reviewed every child note and the implementation
commits: bob-cli `5c0f8160`; bob-plugins `824d8e49`, `f764a4be`, `c9454208`, `abbac5ac`,
and `84ba9d41`. The audited trees were bob-cli `e6eb52a1` and bob-plugins `84ba9d41`;
recheck drift before editing. The exact built-bundle reproducer and measured evidence
are saved as `file:explicit:985ff95df01c3b7cda7a1c8c`; use `sase artifact read` to
inspect it.

Use `/sase_repo` to open `bob-plugins` and the plans sidecar, and use only the paths it
prints. Read the plugin AGENTS.md. Read `docs.md` with `/sase_memory_read` before
documentation changes. No memory changes are allowed. The approved decisions remain
`settle_cursor = stay`, `o_in_group = intake_end`, `wasm_spike = yes`, and
`memory_settle_record = no`. The containing epic owns the WASM research task and
gesture-consumer wiring; neither is this tale's work.

Keep Rust authoritative. The pure transform still only applies future local schedule
blocking, eligible groups, and badges. Preserve requirements, drawers, authored child
bytes, sticky lanes, no-I/O behavior, and the frozen v1 API. Do not add reconcile/bob
subprocess calls or vault writes to this provider. Edit plugin fragments, never
generated main.js; keep fragments <=1000 lines. Do not change Cargo.toml's version or
CHANGELOG.md.

## 1. Enforce pending task identity and staged-write preimages

The defects are in `plugins/bob-ledger-tools/src/139g-task-settle-editor-apply.js` and
its use by `139h-task-settle-pending.js`.

- `taskSettleFindIdentityLine` falls through when a supplied block ID is absent. With no
  `text`, it accepts any line at the hint. The lander reproduced
  `taskSettleFindIdentityLine(["- [ ] #task Replacement ^replacement"], {blockId: "deleted", line: 0}) === 0`.
  A real pending-provider call with `{blockId: "deleted", line: 8}` against the editor
  suite's `AREA_INPUT` returned `applied: true`, `changed: true` and performed one
  write. Make a supplied block ID authoritative: absent or duplicate means unresolved.
  With no block ID, require exact text at the hinted line or an unambiguous exact-text
  match. A line number alone is never identity. Preserve rapid-tap rearming with
  refreshed text and movement of an existing unique ID.
- `taskSettleApplyToEditor` defaults `expected` to the current live buffer even when
  `contents` supplies staged text. The lander passed current text `"Authored text\n"`,
  staged `"Old snapshot\n"`, and no `expected`; the method wrote the old snapshot.
  Require an explicit matching preimage whenever staged contents are supplied. A missing
  or mismatched expected preimage must return a non-writing outcome without
  cursor/scroll/history changes. Current-buffer calls without `contents` must keep
  working without `expected`.

Add behavioral regressions to the existing editor/pending suites. Cover absent ID with a
valid stale hint, absent ID even if fallback text matches elsewhere, duplicate ID, valid
moved ID, text-only unique identity, ambiguous text without a valid hint, and
deleted-task timer/explicit flush producing no writes. Cover missing/stale/matching
staged preimages and the ordinary current-buffer path. Keep staged gesture+settle
atomic, stay/jump semantics, and measured undo behavior. Register any new test files in
package.json if existing files are insufficient.

The lander also tested cursor-leave flushing in a disposable real Obsidian vault: it
applied the settle and drained pending work after the caret left the task. A suspected
nested-update restriction did **not** reproduce. Do not redesign the scheduler around
that discarded hypothesis.

## 2. Make canonical fixture comparison run in linked workspaces

`scripts/task-settle-vectors.cjs` computes `CANONICAL_DIR` with four parent traversals
from the scripts directory. In this linked layout it points at
`sase/tests/fixtures/task_settle`, which does not exist; both copy tests quietly return.
The working `scripts/task-reqs-vectors.cjs` uses five traversals. Correct the lookup
without hard-coding any numbered workspace path. Preserve the documented
standalone-checkout skip, but make a known linked checkout fail if its canonical fixture
location cannot be found.

Run `scripts/test-ledger-tools-task-settle-copies.cjs` and prove it actually reads the
canonical files, rather than printing the skip diagnostic. Add a meaningful lookup/drift
regression so a present canonical fixture mismatch fails. Direct cmp of README.md and
vectors.json already passed during landing; do not alter expected vectors merely to get
green. Run all 18 canonical vectors through the built bundle, including
byte/map/warning/second-pass assertions.

## 3. Integrate documentation with the concurrent drawer change

Measured with bob-cli `bob_docs.markdown.count_words`:

- `docs/bob-ledger-tools.md`: 5988 before the provider, 6000 at `c9454208`, then 6012
  after drawer documentation in `8254caa4`; its reference cap is 6000. Trim enough
  redundant prose to support both features below the cap while retaining the drawer-tray
  command, taskSettle entry point, and all behavior.
- `docs/develop/plugin-apis.md`: 9287 before the provider, 9961 now. Its 8000 develop
  limit is advisory and the pre-existing debt is tracked by bob-cli-ob, but this epic
  owns the 674-word growth. Move the detailed taskSettle contract to a focused
  `docs/develop/task-settle.md` page, keep the namespace table row and the existing
  `#tasksettle-v1` anchor as a short linked overview, and trim duplicate surrounding
  prose enough that the original page does not grow beyond its 9287-word baseline
  because of this epic. Keep all other published URLs/anchors intact. This approved tale
  authorizes this narrow docs split. Follow the existing mounted develop-page convention
  and navigation/index rules. Update the ledger-tools reference to point to the detailed
  contract.

Document the repaired identity and preimage refusal rules, existing coordinate systems,
version lookup, synchronous flush, 500 ms timing, and observed undo ordering accurately.
Keep logs/relation and other API prose outside the taskSettle subsection when splitting.
Preserve the task-status-hooks link and the warning that consumer phases still own
gesture wiring.

Post-start integration audit already found no pure Rust settle/vector changes:
`08a708d3` adds a postimage writer wrapper, while the capture/drawer/snooze commits do
not change the pure algorithm. Plugin property-card fixes (`49bdcb85`, `fdde3dcf`),
drawer typing/acceptance (`8d3c555c`, `8254caa4`), and settings tests (`689da092`) do
not require new taskSettle consumers here. The doc budget collision above is the
concrete integration work. Recheck later commits and address new provider conflicts; do
not take over q4.9/q4.10/q4.11 consumer phases.

## 4. Verify and deploy

The lander ran the settle vector/copy/regression/editor/pending tests, affected cycler
open-line/req-toggle/req-vector tests, and frees-marks tests successfully; the canonical
comparison skipped as diagnosed above. `npm run build:check` passed for all nine
plugins. Final bob-cli `just check` has not yet run.

Bump the ledger-tools manifest version using repo practice, regenerate the bundle and
docs inventory, and run targeted guard and parity tests. Then run `npm test`,
`npm run build:check`, and `npm run test:e2e:task-settle-undo`. Extend the
disposable-vault check as needed to exercise a deleted pending task and refusal without
modifying the live vault. Run `npm run docs:check` with `BOB_CLI_DIR` explicitly
pointing to this coder's bob-cli checkout, so it uses reviewed source rather than a
separately fetched cache. If bob-cli doc configuration changes, run its prescribed docs
generation and check commands too.

Run **`just check` in bob-cli** as the final file-change gate regardless of whether Rust
source changed. Do not run `just check-full`. Use `/sase_monitor` for long commands,
carrying this tale's remaining closeout instructions into the successor. Never end while
an unstaged subprocess is still running. Distinguish new failures from independently
reproduced pre-existing ones; baseline failures do not become new provider scope merely
because the gate exposed them.

Deploy only ledger-tools with `bob plugins sync -p bob-ledger-tools`, without `-F`. On a
protected-deploy refusal follow AGENTS.md and the deployment runbook. Verify the
deployed manifest/bundle corresponds to the tested source. Rebuild generated assets
after any rebase.

## 5. Finish this epic and its parent phase

The lander already completed all seven child PROPOSED FOLLOW-UP dispositions and
recorded them on bob-cli-q4.8.1. Do not create duplicates:

- `.1 #1` / `.2 #1`: ten cycler failures fixed by `abbac5ac` (stale plain NOTES
  expectations updated to shipped bold Notes); targeted tests pass, so declined.
- `.1 #3` / `.2 #2`: frees-mark load flake fixed by `84ba9d41` CPU measurement; targeted
  test passes, so declined.
- `.1 #2` / `.3 #1` / `.3 #2`: existing bob-cli-jz corroborated for unchanged
  navigation-hotkeys 6068/6000, and bob-cli-ob for baseline plugin-apis advisory. The
  containing q4 epic already owns its q4.5 navigation increment in note #2.
  Capture-syntax 6011/6000 was causally assigned to active bob-cli-ph.6.1 with a
  DISCOVERED ISSUE note: `22fea308` and `8a1d16a6` caused the growth. This tale owns the
  provider API/ledger docs changes described above.

Once verification and deployment are complete, record the corrected API signatures,
identity/preimage guards, actual canonical-copy execution, check results, deployed
version, and integration findings on **bob-cli-q4.8** for downstream consumers. Include
the preceding follow-up dispositions in the epic close note.

1. Run `sase bead epic-symbols bob-cli-q4.8.1`. Resolve each entry by wiring,
   privatizing, a legitimate non-test pragma, or deletion under Symvision policy; re-key
   it only if a concrete still-open later bead genuinely owns the exemption. The lander
   found no entries, but rerun. Do not leave any keyed to closing beads.
2. Close normally:
   `sase bead close bob-cli-q4.8.1 --note "<verified guard repairs, parity, editor/scheduler checks, post-start integration, deployment, and all follow-up dispositions>"`.
   If symbols block close, clean them up and retry. Never force a successful nested
   landing; unfinished descendants must actually be completed.
3. Run `just symvision` if available. The audited bob-cli justfile has no such recipe;
   explicitly record unavailability if still absent.
4. Set `status: done` in the linked epic plan **202611/task_settle_twin.md** in the
   opened plans sidecar (PLAN ref `plan:202611/task_settle_twin.md`, linked checkout
   path `sase/repos/plans/202611/task_settle_twin.md`). Preserve all other
   frontmatter/content; use audited artifact reads before editing it.
5. Reread `bob-cli-q4.8.1` with reason `Need the parent link` and verify its parent is
   phase **bob-cli-q4.8**. Verify this complete frozen v1 provider satisfies that phase
   and its original decisions, run `sase bead epic-symbols bob-cli-q4.8`, and
   resolve/re-key any entries as above. Close **only that parent phase** normally with
   `sase bead close bob-cli-q4.8 --note "<complete provider and validation evidence>"`.
   Leave **bob-cli-q4** to its waiting land agent. Do not mark the containing epic's
   plan done. If parent verification is incomplete, record the concrete blocker on
   bob-cli-q4.8 and report it instead of forcing closure.
6. Finish through `/sase_final`, declaring commits for every changed repository,
   including the plan sidecar. The final code and closeout may land together; they must
   not depend on a commit SHA, push, or CI produced after this turn.
