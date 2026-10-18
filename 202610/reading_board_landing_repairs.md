---
tier: tale
title: Finish Reading Board refresh correctness and land bob-cli-99
goal:
  Reading Board reflects current Bob facts, reports stale data honestly, and bob-cli-99
  closes with verified integration and complete follow-up triage.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-99.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-99
create_time: 2026-10-18 16:11:36
status: wip
---

- **PARENT:**
  [202610/reading_board.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/reading_board.md)
- **BEAD:** bob-cli-99

# Finish Reading Board refresh correctness and land bob-cli-99

## Outcome

Finish the bounded remaining work from the landing review of epic `bob-cli-99`, then
close that epic in this coding turn. This tale has no land agent afterward. The host
commits the coding turn after it ends, so closeout must not depend on this turn's own
commit SHA, push, or CI run.

The board already exists and the four original phases are closed. Repair its
data-to-view contracts and refresh lifecycle, correct the rollout documentation, and
preserve the original read-only design. Honor the epic's final decisions:
`pdf_action=yes`, `human_reason=yes`, `memory_board_record=no`. The deferred decision
record has its own task, `bob-cli-9e`.

## Context and verified baseline

Read `sase bead read bob-cli-99 -r "Need landing audit and triage outcomes"` and
`sase artifact read plan:202610/reading_board.md "Need approved board contracts"`. The
epic's plan is `plan:202610/reading_board.md`, currently resolved under
`sase/repos/plans/202610/reading_board.md`; use the PLAN path returned by the bead and
open the plans repository through `/sase_repo` before modifying it.

The lander reviewed every note of `bob-cli-99.1` through `.4`, the original plan and
research, and the implementation:

- bob-cli `b88195e7`: selected-task facts, typed `blocked_by`, and human reasons;
  `b291f23f`: board/dashboard documentation.
- bob-plugins `8a0be74`: shared spawn, cache, and lane models; `74e3d9b`: renderer,
  interaction, PDF API, CSS, and preview.
- 73 Rust `ref_library` unit tests pass. The 85 focused board/stats/PDF JS tests pass,
  and plugin build/manifest validation passes. Existing tests miss the regressions
  below.
- Live Bob JSON at landing has 47 modern open references: Reading 2, Next 5, Ready 17,
  Snoozed 23, matching the board model; all 23 blocked rows have reasons. Eight rows
  have audio path strings, while the board decodes zero audio markers. Counts may change
  with normal vault activity.
- Full `npm test` at clean bob-plugins `74e3d9b`: 3,077 passed, seven known failures.
  The log is `file:explicit:fcf6e006c27bd0600d8b345a` and the failures were corroborated
  on `bob-cli-6v`, `bob-cli-6w`, `bob-cli-7i`, and `bob-cli-3w`. The ranker suite passes
  alone on the unchanged tree.
- A fresh clean-tree Rust run at `47036193` reproduced
  `native::highlights_ref::note_snapshot::profile::tests::pandoc_maps_every_contract_construct_to_tex`
  panicking at `profile.rs:1012` with `lifted link`. The lander attached it to causally
  responsible active epic `bob-cli-87`.
- `cargo test --lib diagrams_` at the same head: eight passed, two failed.
  `pdf_haystack` falls back to lossy PDF bytes when `pdftotext` is missing on apollo, so
  its text assertions fail or pass vacuously. This unrelated test infrastructure defect
  is `bob-cli-9i`; log `file:explicit:2ad4163e19db21052bd615c2`. It also covers the CLI
  Mermaid family reported by the phases, whose helper contains the same invalid
  fallback.

## Remaining defects and implementation

### 1. Preserve audio presence across the JSON boundary

Open `bob-plugins` with `/sase_repo`, read its `AGENTS.md`, and edit source fragments,
rebuilding generated `main.js` afterward.

`src/native/ref_library/row.rs` defines `audio: Option<String>`; the producer returns a
path string or null. In `plugins/bob-ledger-tools/src/241-plugin-reading-board-data.js`,
`readingBoardNormalizeRow` currently uses `audio: value.audio === true`. Consequently a
real audio path becomes false and the card's promised ♫ never renders. The synthetic
preview's boolean audio values concealed the mismatch.

Normalize the real string/null contract to the presentation's audio-presence flag.
Generate/update the shared synthetic Bob JSON fixture with a bound audio path, then test
raw JSON -> decoder -> model -> DOM. Verify that an audio path produces one marker and
null/absent audio does not. Keep real vault titles out of committed fixtures and
screenshots.

This also integrates newer intake work: `5848c39d` now files `--listen` audio at birth
through `attach_intake_unit`/`finish_intake`. It still emits a string-valued audio
field, which the board must consume without a second scan or a new parser.

### 2. Repaint when displayed content or navigation changes

In `242-plugin-reading-board-models.js`, `readingBoardSignature` excludes `parent`,
kind/type, task path/block/link, and prerequisite identities. In
`244-plugin-reading-board-block.js`, an equal render key retains the old DOM and
handlers even though `view.model` is replaced.

Confirmed reproduction: take the same ready row and change only `parent: old -> new` and
`task: {path: old.md, block_id: r, link: [[old#^r]]}` to the corresponding `new` task.
Both models have identical signatures. After such a move the topic and Task
button/Alt+Enter handler remain stale. Changing a prerequisite ref path with the same
title/count similarly leaves highlighting bound to the previous prerequisite.

Make the signature cover all values consumed by visible DOM, tooltip content, actions,
and prerequisite highlighting. Preserve the optimization that a fetch with only a new
`generated_at` updates time in place. Update its data-time tooltip in place too. Add
DOM-level regressions for a task move, a type/topic change, and prerequisite identity
changes without reason-text changes. Assert that Task/Alt+Enter targets the new
location, highlight targets follow the new prerequisite, and focus/scroll survive the
necessary repaint.

### 3. Deliver background failures and preserve freshness correctly

The data and block fragments currently deliver stale cached data immediately from
`apiReadingBoardFetch`. Its background flight only repaints on success; its failure
result is discarded. `revalidateReadingBoardBlocks` stamps `readingBoardViewDataApplied`
on that immediately returned cached report. `renderReadingBoardBlock` also discards
background mount-refresh results. Only manual `readingBoardRefresh` sets
`view.refreshError`.

Confirmed reproduction using the production mixin: seed the cache, register a view, make
`readingBoardSpawnFetch` return
`{ok:false, code:failed, message:test background failure}`, and call
`revalidateReadingBoardBlocks`. After the flight settles `view.refreshError` is
undefined, although the view has been marked current. The last frame silently looks
fresh.

Give completed fetch outcomes a consistent keyed delivery path to mounted views. On
automatic/mount refresh failure, retain the last good board and show the approved
timestamped stale-data lint with Retry and the first error line. Do not advance
freshness on a cached return or failure. On later success, clear the error and update
the data time. Automatic refresh remains silent in the live region; manual refresh
retains its busy state and announcement. Ensure late settlements do not resurrect
unloaded views or start spawns after teardown. Keep stats behavior and its tests
unchanged.

Use integrated tests with the real data mixin and fake DOM, not a stub that replaces
`apiReadingBoardFetch`. Cover background failure from metadata revalidation and
stale-cache mount, successful recovery, and unload while a fetch is pending.

### 4. Do not lose invalidations during a flight

The same-key-flight branch always shares the current promise, including forced
refreshes. `readingBoardViewDataApplied` records the current global vault generation
instead of the generation the fetch actually observed.

Confirmed reproduction: seed cache at generation 0; start a gated revalidation; advance
`readingBoardVaultGeneration` to 1 and revalidate the same mounted view; settle the
original report. There is one spawn, `view.dataGeneration` is 1, and no follow-up flight
exists. A snapshot captured before the edit is now treated as current, and switching
back to the leaf cannot repair it.

Capture the invalidation generation at fetch start and distinguish duplicate readers
from an invalidation/manual refresh arriving during a flight. Coalesce new invalidations
into a follow-up read while keeping at most one spawn in flight. Preserve the four-entry
LRU, mounted-only debounce, local-date rollover, and different-fence behavior. Do not
discard a mounted fence's refresh demand merely because another fence queues later.

Use controlled promises/clock in tests: edit during a flight, repeated edits coalesce,
manual force during a flight, and multiple mounted fences. The view must become current
only when a report fetched after the last relevant event has been delivered. No writes,
reconcile, or vault-text parsing are allowed.

### 5. Keep malformed fence values out of the subprocess arguments

The approved fence contract says bad values produce muted lints and are ignored.
`readingBoardParseFence` currently accepts `type: [papers` as the literal type
`[papers`, `parent: [sase, cash]` as a note name, and `parent: {note: sase}` as a note
name, all with `lints: []`. These malformed/incorrectly typed values reach Bob as
filters, hiding the board or producing a command error instead of the promised lint.

Validate the supported YAML surface before building argv: parent is one scalar note
name, type is one scalar type or a list of scalar types, quotes/brackets must balance,
and mappings/arrays in scalar positions are rejected. Prefer an existing safe YAML
parser if available in this plugin; otherwise keep the small supported grammar explicit.
Unknown or malformed options generate one muted lint each and leave the valid options
usable. Add parse -> argv tests for the three reproductions, wrong scalar types, valid
quoted names, and valid inline/block type lists. This is fence configuration parsing
only; the board still reads no vault text or blocking rules.

### 6. Correct docs and finish integration review

In `docs/ref.md` remove the stale "coming Reading Board" wording and correct the claim
that the Mac Refs panel consumes `blocked_by`, `task.priority`, and `task.scheduled`. At
Mac commit `5a0bff5`, `Sources/RefsCore/RefsDecoding.swift` still ignores those keys;
the shipped board consumes them and the Mac inspector addition is follow-up
`bob-cli-9f`. Do not implement that separate feature here.

The lander fetched origin and reviewed all post-start main-repo commits through
`47036193`: Work badge threshold/docs changes (`674c9f55`, `44c04e24`), bare-x capture
(`c166ed26`, `6bc432b2`), and ingest/locking/drain/doctor changes (`85eac25a`,
`c4ac06fb`, `fa7745fc`, `5848c39d`, `47036193`). These preserve task status and Today
contracts; audio intake requires the decoder repair above. bob-plugins Work badge
changes (`1907419`, `6529c58`) are already combined with the board and do not require
changing its accents or Base count. Mac `5a0bff5` adds optional `added_at`/Newest
ordering independently; the board continues to preserve Bob's row order and tolerate
extra fields.

Recheck any commits that land after these review heads, particularly ref-row changes
from active epic `bob-cli-9d`. Integrate any actual conflicts with the board contract
without expanding this tale into the separately tracked Mac inspector or badge features.
Keep the existing dashboard board fence and Reading Queue Base embed.

## Verification and deployment

- Add the focused regressions above, run the board data/models/block suites, stats
  suites, and navigation PDF suite. Retain read-only spies and the older-Bob fallback
  tests.
- Bump the modified plugin's minor version and describe the repair; run `npm run build`,
  `npm run validate`, and `npm test` in the opened plugin repo. Identify failures
  individually against the recorded baseline; repair any caused by this work. Do not
  treat previously corroborated unrelated failures as unfinished Reading Board
  functionality.
- Run `just check` in bob-cli for file-change verification. Never run `just check-full`.
  Run the Rust reference-library CLI tests as needed for changes to the shared fixture.
  Existing unrelated rendering failures are tracked in the epic's triage note; record
  the actual gate result honestly. Use `/sase_monitor` for a long check, with a
  successor instruction that resumes this tale and completes the closeout below.
- Run the real Chromium `scripts/preview-reading-board.cjs` harness with synthetic data,
  inspect changed outputs, and verify focus/actions and honest error states. Render live
  `bob ref list -s ready,next,wip -A -f json` through the same decoder/model/preview and
  compare lane totals and audio presence. Use `/sase_repo` for any vault-repository
  access. Do not commit private-title preview files. Confirm the dashboard fence remains
  above its Base table.
- Deploy with `bob plugins sync --repo <opened-plugin-checkout>` (confirm the supported
  option via help), as required by the linked repo. Do not run `just install-all` on
  Bryan's MacBook: the original plan reserves that action for him. The final response
  should remind him to run it there and open References for the desktop confirmation.

## Final step: close bob-cli-99 in this turn

The lander has already triaged every original child proposal; use the epic's
`LAND FOLLOW-UP TRIAGE` note for the complete outcomes and carry every outcome into the
close note. Known outcomes include:

- Memory record (99.1 #1, 99.2 #1, 99.3 #1, 99.4 #4): `bob-cli-9e`, small, ready.
- Mac inspector (99.4 #1): `bob-cli-9f`, medium, ready, related to `bob-cli-9d`.
- REFERENCES tooltip (99.4 #2): `bob-cli-9g`, medium, ready.
- Two-week review (99.4 #3): `bob-cli-9h`, small, snoozed to 2026-11-01.
- Plugin baseline (99.2 #2, confirmed by .3/.4): corroborated existing `bob-cli-6v`,
  `bob-cli-6w`, `bob-cli-7i`, `bob-cli-3w`; no duplicates.
- Lifted-link test (99.1 #2, 99.2 #3, .4 close note): evidence on active `bob-cli-87`,
  not a new task. Mermaid PDF-text test failures from the same proposals: `bob-cli-9i`,
  small CI task, ready; missing `pdftotext` triggers an invalid compressed-byte fallback
  in both test helpers.
- Mac installation request (99.4 #5): operational reminder, declined as a new task
  because the approved rollout explicitly leaves it to Bryan.

After the repairs and verification:

1. Reread the epic and all four child beads, including newer notes, and confirm the
   remaining defects above are resolved and all descendants are complete. Inspect the
   parent link with `sase bead read bob-cli-99 -r "Need the parent link"`. It has no
   parent at this landing review, so no ancestor should be closed.
2. Run `sase bead epic-symbols bob-cli-99`. Resolve each entry before closing (wire,
   privatize, non-test pragma, or delete under the Symvision policy), or re-key only to
   an actually open later bead that still needs it. The lander's run found none. Do not
   leave the decision to a later agent.
3. Run `sase bead close bob-cli-99 --note "<verification>"` with a substantive note
   covering source/commit review, honored decisions, these repairs, post-start
   integration, actual tests and known failures, live/deployment evidence, and **every**
   follow-up outcome above and in the lander note. Close normally. If symbols remain,
   clean them and retry. If a named phase is incomplete, finish/reopen it; never force a
   successful nested landing. Cancellation/supersession with
   `--force --reason ... --resolution canceled|superseded` is only for an intentional
   such outcome, never a workaround for a failed close.
4. After close, run `just symvision` if available. The reviewed bob-cli justfile has no
   such recipe; record that if still true rather than inventing one. Open the plans repo
   via `/sase_repo` and set `status: done` in the frontmatter of the exact epic PLAN
   file, `202610/reading_board.md`.
5. Submit `/sase_final` as the last action with commit decisions for every changed
   repository. Report completion and the MacBook reminder. The host commit happens after
   this turn; do not wait for it before closing the epic.
