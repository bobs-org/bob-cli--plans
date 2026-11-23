---
tier: tale
title: Shared dual-read task drawer readers in every plugin
goal:
  Complete bob-cli-ph.2 with one verified recognizer twin, synchronized Rust-owned
  fixtures, unchanged legacy writers, and four deployed plugin bundles.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ph.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-ph.2
status: done
---

- **PARENT:**
  [202611/true_case_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/true_case_drawers.md)
- **BEAD:**
  [bob-cli-ph.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.2.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ph.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ph.2.md)
- **COMMITS:**
  - [b1fa8ad](https://github.com/bobs-org/bob-cli/commit/b1fa8ad6de8c3818152296f7c9af7cb2ec21ed1e)
    — docs: document plugin task drawer reader parity

# Complete bob-cli-ph.2: shared dual-read task drawer readers

## Outcome and scope

Complete assigned phase `bob-cli-ph.2` of `bob-cli-ph`. All four Obsidian plugins must
read existing legacy ALL-CAPS and whole-bold task drawer headers using one
byte-identical JavaScript recognizer twin. Preserve authored bold titles, find the Notes
role by its Unicode key, pass bob-cli's canonical fixtures, and deploy the four updated
bundles. Newly created drawer headers remain legacy throughout this phase, so the
following reader-rollout phase can safely verify every host before any writer changes.

This is one bounded medium coding task: Rust's implementation and vectors already
landed; the remaining work is a shared JS twin and its existing consumers. There are no
previous implementation/remainder notes on this phase. Do not create child beads or new
task beads. Report unrelated discoveries with `PROPOSED FOLLOW-UP:` notes on
`bob-cli-ph.2`.

Read the assigned bead and final epic decisions with
`sase bead read bob-cli-ph.2 -r "Implement approved plugin-reader plan"`. Use audited
artifact reads for:

- `plan:202611/true_case_drawers.md`, especially the design of record and
  `plugin-readers` phase;
- `research:202611/true_case_drawers/true_case_drawers.md`.

If the research ref is missing, open/refresh `bob-cli--research` through `/sase_repo`,
then retry the audited artifact read. This resolved the missing-ref result during
planning. Do not read sidecar files directly.

Open `bob-plugins` through `/sase_repo` and read its `AGENTS.md`; all plugin paths below
are relative to that returned checkout. Other paths are relative to bob-cli. Never edit
installed plugin files in the vault. Honor the epic's tray-glyph and memory decisions
without re-asking them: rendering belongs to a later phase and all memory editing
belongs to the epic's memory phase. This plan edits no memory.

## Evidence already checked

- `bob-cli-ph.1` is closed, with the reader implementation in `37b2080e` and successful
  Rust, CLI, repository, and docs verification recorded on the bead. Planning inspected
  clean bob-cli `c25d92fd` and bob-plugins `5c9b4fbf`.
- `src/native/capture_drawers.rs` owns body recognition, contextual recognition, Unicode
  keys, source markers and UTF-8 byte ranges, reserved names, creation validation, and
  the pure drawer/H2 title mapping. `docs/task-drawers.md` describes the shipped reader
  window.
- `tests/fixtures/task_drawers/vectors.json` currently contains contextual DV1–DV4, body
  DV5–DV24, and DH1–DH6. `task_reqs` has RQ22 and RL61, and `task_split` has SP52.
  Plugin copies have not caught up.
- Existing duplication remains in `taskReqIsDrawerTitle` in the cycler, navigation,
  Block ID Prompt, and ledger tools; `isTaskSplitDrawerTitle` is another independent
  grammar. The first three RQ copies use an ASCII-only key. Ledger tools has the
  regex-range defect that accepts `NOTE: X` and `KEY:: VALUE`.
- A focused baseline of the five suites listed under verification passed 93/94 tests.
  The sole failure was `README.md matches bob-cli's canonical fixture` in
  `test-task-reqs-twin-copies.cjs`. Parent epic note #1 assigns that semantic fixture
  drift to this phase; fix the fixtures and readers together, not only the README.
- `sase bead epic-symbols bob-cli-ph.2` currently reports no entries. Recheck at
  completion.

Recheck the current trees before editing so already-landed work is preserved. Use
current canonical fixtures rather than freezing the revision listed above.

## Implementation

### 1. One pure drawer twin, four identical copies

Add a focused `task-drawers-twin.js` source fragment to each of:

- `plugins/task-status-cycler/src/`;
- `plugins/bob-navigation-hotkeys/src/`;
- `plugins/block-id-prompt/src/`;
- `plugins/bob-ledger-tools/src/`.

Use task-status-cycler's copy as the plugin-side canonical copy, matching existing
RQ/RL/RW conventions. Keep all four byte-identical. Register each fragment in
`src/fragments.json` before consumers can invoke it and expose its helpers through each
plugin's test exports. Edit source fragments only and regenerate `main.js`. Every
hand-edited fragment must remain at or below 1000 lines; extract a coherent companion
fragment if needed instead of expanding near-cap files.

The twin must provide the equivalents of `drawer_body_title`, `is_legacy_drawer_title`,
`drawer_key`, `recognize_drawer_header`, and the DH `heading_for_drawer` /
`drawer_for_heading` helpers. Preserve recognition versus creation validation as
separate operations. Keep it pure and independent of Obsidian; avoid implicit
dependencies on RQ helpers absent from ledger tools. Use thin adapters for callers'
owner/line representations, with recognition rules owned by the shared twin. Do not
create a separate parser in each adapter.

Mirror the landed Rust contract, including these easy-to-miss details:

- Only real task direct children qualify; reject checkbox headers, quotes, fenced list
  content, managed lines, reserved keys, and structural H2 names. A deeper bold bullet
  remains note content. Empty headers are recognized.
- Whole-bold bodies must contain a nonempty single-line title with no `*`, no edge
  whitespace inside the delimiters, at least one Unicode alphanumeric, no terminal
  colon, and no content after `**`. `__Title__` stays a note.
- Preserve the legacy predicate exactly, including its punctuation and ASCII alphabet.
  Follow Rust's explicit compatibility exception for unadorned legacy
  Schedule/Work/Rotten Log labels; their managed rendered rows remain excluded.
- Retain the authored title and `bold` / `legacy_caps` marker. A project's H2 adapter
  retains `section`. Keys collapse Rust-compatible whitespace and use Unicode uppercase,
  including `Straße` → `STRASSE` and `Café` → `CAFÉ`.
- Report half-open UTF-8 header/title byte ranges, excluding line endings and delimiters
  as in the fixture. JS string indexes are UTF-16; test the conversion with Unicode and
  CRLF rather than exposing character offsets as byte offsets.
- DH preserves bold titles and internal whitespace verbatim; legacy promotion retains
  today's Title Case mapping. Port empty-section and named-refusal behavior from Rust.
  DH helpers do not authorize switching reversal writers.

Build a twin-copy test modeled on `scripts/test-task-reqs-twin-copies.cjs`. Check source
bytes, manifest inclusion/order, generated bundle inclusion, exports, and each bundle's
execution of DV/DH. Include a stable searchable bundle marker, for example
`bob-task-drawer-dual-read-v1`, for phase `bob-cli-ph.3` to inspect.

### 2. Retarget all existing plugin readers

Audit callers of the old predicates, raw drawer-body comparisons, and nearby ALL-CAPS
regexes. Remove independent drawer grammars; compatibility wrappers may delegate to the
shared twin where existing callers/tests need their names. Distinguish body-shape checks
from contextual readers as Rust does. Do not change Pomodoro or task-type naming rules
when removing incidental drawer sharing.

Primary integration points:

| Surface                        | Existing locations                                                                                                                                | Required behavior                                                                                                                                                                                                     |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RQ structure and guards        | cycler `194-task-reqs-lines-twin.js`, nav `694-task-reqs-lines-twin.js`, Block ID Prompt `165-task-reqs-lines-twin.js`, ledger `139-task-reqs.js` | Recognize both forms consistently; drawer-nested checkbox children remain excluded from requirements; retire the ledger regex bug. Preserve identical RQ twins.                                                       |
| Layout and Notes home          | cycler `196-task-block-layout-twin.js`, nav `696-task-block-layout-twin.js`, their tidy twins                                                     | Classify contextual drawers with decoded title/marker, compare the Notes role by `NOTES_KEY`, and reuse existing `**Notes**` without a duplicate header. Keep project `## Notes` geometry.                            |
| Writer-side reads              | cycler `198-task-reqs-writers-twin.js`, nav `698-task-reqs-writers-twin.js` and callers                                                           | Parent/refusal and scoop/filing checks see bold drawers; keep `req_in_drawer` and the first-note birth policy. New headers still serialize as legacy `NOTES`.                                                         |
| Split classification and scope | nav `256-task-split-plan.js`, `257-task-split-apply.js`, `653-plugin-task-split.js`, carry helpers                                                | Use contextual recognition, including when a bold bullet is deeper ordinary content. `parentDrawer`, candidate fields, picker labels, and subtitles use the decoded authored title, never raw `**…**`.                |
| Promotion reads                | nav `100-yank-and-paths.js`, `110-project-seed.js`, `111-project-seed-reqs.js`                                                                    | `parseProjectSectionBulletTitle` and Notes comparisons use the twin: bold titles map verbatim, legacy titles keep Title Case. Preserve requirements, ownership, existing seed outputs, and lossless/refusal behavior. |
| Close-card consumers           | cycler `198a-req-close-twin.js`, nav `699-req-close-twin.js` and existing RQ users                                                                | Continue passing existing RV cases through the updated readers; retain byte-identical close twins.                                                                                                                    |

Split `TASK_BLOCK_NOTES_KEY` from the writer-title constant where necessary. Keep the
writer title `NOTES` in this phase. Do not globally replace uppercase strings or promote
a permissive body check into a contextual recognizer.

Promotion's read adapter must accept bold drawers without rewriting their title. Do not
remove existing seed transformation safeguards or change seed serialization policies
merely to make DH round trips pass. The later `obsidian-drawers` phase owns reversal
changes, bold writer output, rendering, and typing assistance.

### 3. Synchronize fixtures and cover consumer behavior

Copy `tests/fixtures/task_drawers/{README.md,vectors.json}` byte for byte from bob-cli
into each plugin's `scripts/fixtures/task_drawers/`. Its README already states that
bob-cli is authoritative and must be updated first.

Refresh all existing copies of canonical `task_reqs` files (README, vectors, layout,
writes as applicable) and nav's `task_split` fixtures together. Keep the unchanged RW
writer expectations as a guard against an early writer flip. Preserve fixture identity
checks against bob-cli when its checkout is available; always enforce equality across
copies even when running standalone bob-plugins. Do not weaken checks or skip new cases
to make the baseline failure disappear.

Add the new suite(s) to `package.json`'s explicit `test:unit` list. Execute DV/DH
against all four built bundles, RQ against every requirement reader, RL/RW against both
layout/writer twins, SP against nav, and existing RV suites. Add meaningful consumer
regressions for:

- an existing bold Notes drawer being reused with its exact header preserved;
- mixed legacy/bold headers, Unicode keys, invalid legacy colon bodies, managed and
  reserved lookalikes, fences/quotes/depth, and LF/CRLF source ranges;
- a bold drawer's nested requirement lookalikes remaining excluded, with correct
  `req_in_drawer` refusal on attempted conversion;
- splitting a note under `**API Design**` while the picker shows `API Design`, and a
  deeper whole-bold note still being splittable;
- promotion reading `API Design`, `C++ Notes`, `Q1: Goals`, and `Café` verbatim, while
  the existing legacy promotion/reversal and seed tests stay green;
- creation still writing `- NOTES` and no background rewriting of existing headers.

When a missing behavioral case belongs to the shared contract, add it to bob-cli's
fixture and exercise it in Rust first, then copy the full fixture. Local plugin
integration/UI-adapter tests can remain JS tests. Do not hand-edit canonical copies
independently or alter unrelated Rust semantics.

### 4. Document, verify, deploy, and record evidence

Read `docs.md` using `/sase_memory_read` before documentation edits. Update bob-cli
`docs/task-drawers.md` to state that all four plugin readers now share the dual-read
contract and that writers remain legacy. Keep future rendering, capture syntax, and
writer behavior out of the shipped claim. Update plugin developer documentation only as
needed to explain fixture/twin maintenance. Regenerate the plugin docs inventory if its
inputs change.

Run these focused suites (the planning baseline used exactly these five):

```sh
node --test scripts/test-task-reqs-twin-copies.cjs scripts/test-req-close-twin-copies.cjs scripts/test-navigation-task-split-vectors.cjs scripts/test-navigation-hotkeys-project-from-task.cjs scripts/test-navigation-hotkeys-project-reversal.cjs
```

Also run the new DV/DH suite and touched integration suites. Then run plugin gates:

```sh
npm run build
npm test
npm run validate
npm run docs:check
```

For bob-cli changes run `just docs-gen`, `just docs-check`, and `just check`, plus
focused Rust tests for any added fixture cases. Use `/sase_monitor` for long gates; wait
for each staging command to exit and finish via `/sase_final` as required. Reproduce
unrelated failures on an unchanged base before classifying them as pre-existing. Record
identical base failures as `PROPOSED FOLLOW-UP:` notes on this phase and do not leave it
open solely for those failures. The known fixture mismatch is in scope and must be
fixed.

After checks, deploy only the changed plugins from the prepared source checkout:

```sh
bob plugins sync -p block-id-prompt
bob plugins sync -p bob-ledger-tools
bob plugins sync -p bob-navigation-hotkeys
bob plugins sync -p task-status-cycler
```

Follow bob-plugins `AGENTS.md` and `docs/develop/deployment.md` if sync protects another
checkout's deployment; do not use `-F`. Verify the marker and matching bundle bytes
using the supported sync/deployment evidence. Add a phase note with each plugin ID,
marker, bundle hash, verification result, and the exact deployed bob-plugins commit so
`bob-cli-ph.3` has concrete rollout evidence.

SASE finalization owns commits. A pre-finalization HEAD is not proof that newly authored
files are committed. If deployment occurred from a dirty source tree, record that
honestly and use a short `/sase_monitor` continuation after the host commit to record
the landed revision and refresh/verify deployment provenance on that committed tree. Do
not close the phase until this required evidence is complete. Use an additive `feat`
commit description; no release version or CHANGELOG edits.

## Completion criteria

The shared twin and fixture bytes match across all four plugins, their bundles pass the
canonical cases, every audited reader accepts both forms, and existing writer
expectations remain unchanged. The fixtures that caused the baseline failure match Rust.
Required checks pass or have documented identical base failures. All four deployments
have verified revision/marker evidence on `bob-cli-ph.2`.

Finally run `sase bead epic-symbols bob-cli-ph.2`. Resolve any entries or re-key them to
a still-open later phase/parent before closing. Close only the assigned phase with
`sase bead close bob-cli-ph.2 --note "<verified checks and deployment evidence>"`. Never
hand-set its status, reopen it, or close `bob-cli-ph` or any ancestor plan. Finish the
provider turn through `/sase_final` with every edited repository included.
