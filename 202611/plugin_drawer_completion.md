---
tier: tale
title: Finish the landed plugin drawer readers
goal:
  Repair standalone fixture tests, verify and deploy all four dual-read plugin bundles,
  and close bob-cli-ph.2 with rollout evidence.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ph.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-ph.2
status: done
---

- **BEAD:**
  [bob-cli-ph.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.2.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ph.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ph.2.md)
- **COMMITS:**
  - [63e9e76](https://github.com/bobs-org/bob-plugins/commit/63e9e7609338d91de8f592c3c134d69abd92b9ab)
    — test(plugins): load drawer DV/DH vectors from the cycler fixture

# Finish bob-cli-ph.2 from the landed plugin readers

## Outcome and scope

Complete only assigned phase `bob-cli-ph.2`: preserve the landed dual-read plugin
readers, repair their standalone fixture test, finish verification and deployment,
record usable rollout evidence, and close the phase. Do not redo the recognizer or
advance the writer, rendering, migration, or memory phases.

This remainder fits one coding agent. The implementation and documentation already
landed; the prior continuations stopped on unrelated timing assertions rather than an
unfinished reader implementation. The only newly confirmed implementation gap is a
bounded test-fixture loading defect. A child epic is unnecessary for this test fix and
operational closeout. Long verification and any post-commit work use `/sase_monitor`
with an explicit completion instruction, not repeated open-ended retries or a new
implementation plan.

## Read first and preserve the existing work

- Read `sase bead read bob-cli-ph.2 -r "Finish landed plugin readers and deployment"`
  and the parent epic's current notes. Never hand-set the phase status or reopen it.
- Read these artifacts with `sase artifact read`, never directly from a sidecar:
  `plan:202611/true_case_drawers.md` and `plan:202611/plugin_drawer_readers.md`.
- Open `bob-plugins` using `/sase_repo` and read its `AGENTS.md`. All plugin paths below
  are relative to that returned checkout; other source paths are relative to bob-cli.
  Follow its source-fragment and deployment rules.
- The epic decisions are final: tray glyph, and the specified memory changes only. This
  phase has no rendering or memory edits. Do not re-ask those decisions.
- Preserve unrelated landed work and any new phase progress. Do not edit `Cargo.toml`
  versions, `CHANGELOG.md`, installed plugin files, or vault note text.

## Evidence from this planning turn

- The phase remains `in_progress` and has no notes recording remaining work. Its history
  confirms no prior close. Its predecessor `bob-cli-ph.1` is closed.
- Plugin commit `19ac4a3e4bafc2362b2b6189553dec99cfba1500` implements the shared
  recognizer, four fragment copies, exports, consumer adapters, copied fixtures, and
  `scripts/test-task-drawers-twin-copies.cjs`.
- bob-cli commit `b1fa8ad6de8c3818152296f7c9af7cb2ec21ed1e` documents the plugin reader
  parity in `docs/task-drawers.md`.
- Planning inspected clean bob-cli `588c6210` and clean bob-plugins
  `a73f70bf4f22c0916e95546275594fd825036e9a`, which contains the implementation. Do not
  reset to these historical revisions when resuming.
- The three suites below pass all 22 tests in the linked checkout. This includes DV/DH
  across all four bundles, byte equality, requirement readers, RL/RW, Notes reuse,
  preserved legacy creation, promotion spelling, and split scope:

  ```sh
  node --test scripts/test-task-drawers-twin-copies.cjs scripts/test-task-reqs-twin-copies.cjs scripts/test-navigation-task-split-vectors.cjs
  ```

- Prior completed transcript
  `gh_bobs_org__bob_cli-ace_run-bob_cli_ph_2__3-261123_135121`, recovered with
  `/sase_chats`, reports `just check` passed in the preceding continuation. Its
  full-suite failure review identified only two 25 ms ledger budgets:
  `test-ledger-tools-event-link-times.cjs` (74.4 ms) and
  `test-ledger-tools-frees-marks.cjs` (115 ms). Both had passed in isolation on the
  unchanged tree. That run explicitly did not deploy or close. Treat this as historical
  evidence, not a fresh passing gate or a deployment receipt.
- Those two timing test files were unchanged by `19ac4a3e`.
- `sase bead epic-symbols bob-cli-ph.2` reports no entries; recheck before close.

## 1. Repair standalone fixture loading

`scripts/test-task-drawers-twin-copies.cjs` currently unconditionally reads
`DRAWER_FIXTURE/vectors.json` at module initialization (line 76 at planning time), and
compares every copied file directly to that external directory. The directory is
computed by walking five parents from `scripts/`, so it exists only in a linked bob-cli
workspace. A read-only simulation hiding that directory reproduced an `ENOENT` during
module loading, before the DV/DH tests could be registered.

The earlier approved plan explicitly required standalone operation. Follow the
established pattern in `scripts/test-task-reqs-twin-copies.cjs` and
`scripts/task-reqs-vectors.cjs`:

1. Load the DV/DH vectors from task-status-cycler's committed plugin-local fixture.
2. Always compare every plugin's drawer README and vectors byte for byte to that
   plugin-local canonical copy. Always execute all four bundles' DV/DH tests.
3. Separately compare the plugin-local canonical files to bob-cli when its canonical
   fixture directory is present. If it is absent, emit a diagnostic explaining that only
   the cross-repository comparison is unavailable. Do not skip the bundle/vector tests
   or the plugin-to-plugin equality assertions.
4. Missing or malformed plugin-local fixtures and drift against an available bob-cli
   fixture must still fail. Do not catch arbitrary filesystem or JSON errors as an
   absent external checkout.

Keep this a test-only repair unless the focused checks reveal a concrete reader
regression. Preserve the fragment bytes, runtime bundles, fixture contents, and legacy
writer behavior already verified. No new package-list entry is necessary if the fix
stays in the existing registered test file.

Verify the actual runner both with and without the external fixture directory. Use a
scoped subprocess filesystem shim or a disposable fixture/test copy inside the opened
checkout; do not move, rename, or hide the real bob-cli fixture tree. Assert the
standalone run still executes all four DV/DH tests and detects a deliberately mismatched
plugin copy in the isolated test setup. The linked run must still detect drift from
bob-cli. Remove disposable setup after checking it.

## 2. Finish checks and classify failures once

Run the focused suites above, plus:

```sh
node --test scripts/test-req-close-twin-copies.cjs scripts/test-navigation-hotkeys-project-from-task.cjs scripts/test-navigation-hotkeys-project-reversal.cjs
npm run build
npm test
npm run validate
npm run docs:check
```

Use `/sase_monitor` for long gates. Preserve complete logs and explicit exit codes for
each command. Run independent gates independently: a red `npm test` must not prevent
`just check` or another required gate from running through shell `&&`. Confirm the
previously landed bob-cli documentation still matches this phase; reuse recorded
verification only when its revision and command are known. Run `just check` for current
closeout evidence. If bob-cli source/docs change, also run the corresponding focused
checks, `just docs-gen`, and `just docs-check`, reading `docs.md` before documentation
changes.

Do not widen timing thresholds, delete assertions, or change unrelated ledger behavior
to make this phase green. For a full-suite failure:

- Inspect the entire log, not only its tail. Fix failures caused by this phase.
- Reproduce apparently unrelated failures on the clean pre-fix tree, retaining the
  command, exact SHA, failing assertion, and outcomes. For intermittent timing failures
  retain the unchanged-tree failing run and passing isolated result; do not claim they
  are deterministic or hide the nonzero full gate.
- Search existing tracking before citing it. Record confirmed unrelated failures as
  `PROPOSED FOLLOW-UP:` notes on `bob-cli-ph.2`, including any existing bead and the
  evidence. Do not create a task bead.
- Identical clean-base failures do not keep this phase open. The user explicitly
  authorizes completing the phase despite those failures once in-scope checks and
  deployment are verified. Avoid another loop that stops solely because the same
  unrelated timing budget is red.

## 3. Deploy and give reader-rollout concrete evidence

Read `docs/develop/deployment.md` in the opened plugin checkout. Read the vault memory
with `/sase_memory_read` before vault-specific inspection. Keep every repository read
within paths opened by `/sase_repo`; use Bob's supported deployment/list interfaces for
vault state where possible.

Deploy these four plugin IDs only:

| Plugin                 | Recognizer fragment             |
| ---------------------- | ------------------------------- |
| block-id-prompt        | `src/164a-task-drawers-twin.js` |
| bob-ledger-tools       | `src/138a-task-drawers-twin.js` |
| bob-navigation-hotkeys | `src/693a-task-drawers-twin.js` |
| task-status-cycler     | `src/193a-task-drawers-twin.js` |

From the prepared plugin checkout, preview then apply each selected deployment:

```sh
bob plugins sync --repo "$PWD" --no-pull --dry-run -p <id> -f json
bob plugins sync --repo "$PWD" --no-pull -p <id> -f json
bob plugins list --repo "$PWD" --no-pull -f json
```

Use the actual `sase repo open` path, not a default plugin repository. Honor the
overwrite guard and follow the documented fetch/landed-deployment recovery. Never use
`-F` without the explicit authorization the deployment instructions require. If the
deployed build is still being authored, use the supported monitor workflow to wait
rather than overwriting it or silently leaving this phase unfinished.

Run the documented isolated Obsidian startup and mandatory post-sync smoke checks, using
the selected source checkout and supported vault location. Keep the JSON report. Missing
prerequisites or skipped checks remain explicit limitations, never a fabricated pass;
use the existing supported host workflow when local execution cannot satisfy the
deployment gate. The all-host release/version gate belongs to `bob-cli-ph.3`, not this
phase.

Record a note on `bob-cli-ph.2` with one row per plugin containing:

- exact deployed source commit and source cleanliness;
- `bob-task-drawer-dual-read-v1` marker presence in the source bundle;
- source `main.js` SHA-256 and verified equality with deployed managed files;
- sync/list JSON status, deployment provenance, and smoke result;
- explicit confirmation that newly created Notes headers remain legacy.

`bob plugins list` compares source and vault bytes but exits zero even on drift; inspect
its JSON rows and require each selected plugin to be `synced`. Equality with the
marker-bearing source bundle proves the deployed marker as well. Record the source hash
and exact SHA, not just the marker string or a successful exit.

SASE finalization commits the test repair. A dirty checkout's HEAD is not the commit
containing that repair. If needed, stage a `/sase_monitor` continuation and explicitly
commit through `/sase_final` before its clean-source deployment step. Finish the staging
command and end through `/sase_final`; never wait for a staged successor or manually
commit. The continuation must verify the committed source/provenance and finish this
closeout, not rerun an unbounded test loop.

## Completion

All four bundles retain dual-read semantics and legacy writer outputs. The drawer suite
runs standalone and enforces local fixture equality; linked execution also enforces
bob-cli equality. Required checks have passing results or recorded, substantiated
unchanged-base exceptions. All four deployments have revision, hash, marker, and actual
smoke evidence on this phase.

Run `sase bead epic-symbols bob-cli-ph.2` immediately before closing. Resolve every
entry or re-key it to the still-open owning parent/later phase. Then run only:

```sh
sase bead close bob-cli-ph.2 --note "<verified reader tests, base exceptions, deployed revisions and smoke evidence>"
```

Do not close the parent epic or an ancestor plan. Record unrelated discoveries only with
`sase bead note bob-cli-ph.2 'PROPOSED FOLLOW-UP: ...'`. Finish through `/sase_final`,
declaring all repositories changed by the work.
