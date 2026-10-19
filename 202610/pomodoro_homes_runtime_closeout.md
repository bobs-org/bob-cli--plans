---
tier: tale
title: Finish Pomodoro homes runtime integration and nested landing
goal:
  Correct Obsidian home chips and Mac override labels, verify real client behavior, and
  normally close bob-cli-9k.8 and its ready parent bob-cli-9k.
size: medium
proposed_by: bbugyi200.athena.bob-cli-9k.8.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-9k.8
create_time: 2026-10-18 22:00:03
status: wip
---

- **PARENT:**
  [202610/pomodoro_homes_landing.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/pomodoro_homes_landing.md)
- **BEAD:**
  [bob-cli-9k.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9k/bob-cli-9k.8.md)

# Finish Pomodoro homes runtime integration and land bob-cli-9k.8

## Scope and authority

Finish the remaining work found by the land review of `bob-cli-9k.8`, then close that
epic and recheck its parent plan bead `bob-cli-9k` for normal closeout. This is one
coding agent's bounded repair and verification work. There is no later land agent for
this tale: its final step owns the landing in the same turn as the code.

Read `bob-cli-9k.8`, its four phase beads and notes, and the latest landing notes on
`bob-cli-9k`. Read the approved plans through audited artifact reads:
`plan:202610/pomodoro_homes_landing.md` and `plan:202610/pomodoro_homes.md`. The
normative contract is bob-cli `docs/pomodoro-homes.md`.

Keep all settled decisions: `review_row_time=yes`, the same-day unnamed review bridge,
per-task review shares, additive optional JSON fields, and display-only homes. Do not
edit memory. Named live CLI sessions still skip resolution; labels never route captures,
change authored names, or affect budgets. Swift only decodes and presents bob output.
Open linked repositories with `/sase_repo`, read their AGENTS.md, and use only the paths
returned.

## What the landing review already established

- Reviewed bob-cli commits `77c7741` and `c2c9f5c`, bob-plugins `ed72a42`,
  bob-mac-capture `3a67176`, and chezmoi `d9084960` against every child note. The Rust
  remainder, closed-state, review-share, theme-split, catalog, CLI polish, Ref Today and
  override-payload changes are present.
- The latest live 4w report has `review_time=1732m` and the same `1732m` sum of worked's
  review project parts. The shared vector files are byte-identical.
- Existing homes and Stats panel tests pass 33/33, and `npm run validate` passes all
  seven plugins. These tests mask the metadata-cache bug below.
- chezmoi `bashunit tests/bash/tmux_status_right_test.sh` passes all six tests,
  including the newly added unnamed-derived sample. No further chezmoi change is
  currently needed.
- macOS CI run
  [39186590898](https://github.com/bobs-org/bob-mac-capture/actions/runs/39186590898) on
  `3a671761` passed Lint and Build. Test stops at the pre-existing
  `RefsNewestTests.zone`/`NSObject.zone()` compile error, tracked by `bob-cli-9m`; the
  lander independently corroborated it with `+1`. Mac SSH still timed out.
- There were no `--epic-symbol` entries for either `bob-cli-9k.8` or `bob-cli-9k`.
  Recheck immediately before each close.

Integration review: this child epic started at 2026-10-18 20:20 EDT. Concurrent
`bob-cli-9j` landing commits `af3b4bb` (bob-cli) and `9288244` (Mac) made empty `=#` /
`=3#` name fields show override context. Rust now supplies its homes, but the Mac
decoder and status text still discard them. There are no unrelated newer commits in the
inspected plugin and chezmoi branches. The parent's later intake commit `f4cdf0b` does
not add a homes-consuming surface. Fetch/review new drift again before finishing; do not
overwrite other work.

## 1. Repair the Obsidian homes mirror at its runtime boundary

Work in bob-plugins source fragments, rebuild generated `main.js`, and keep every
hand-edited fragment at or below 1000 lines.

### Metadata cache

`plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js`, `pomodoroHomeCacheFor`, calls
`metadataCache.getCache(file)` with a file object at both lookup sites. Obsidian's
`getCache` takes a path string. The current test double in
`scripts/test-ledger-tools-pomodoro-home-chips.cjs:fakeVault` accepts objects and hides
the failure. In the land review, changing only that double to assert
`typeof file === "string"` caused 12/18 homes tests to fail: linked projects became
`unfiled`, hues disappeared, and lints vanished.

Use the actual file-cache API (`getFileCache(TFile)`) with a resolved vault file, or a
correctly typed path lookup where needed. Reuse the established `noteFrontmatterFor`
pattern from fragment `330-note-ready-model.js`; its opening comment documents this
exact API mistake fixed elsewhere. Preserve safe handling of missing notes/caches. Make
the test doubles reject wrong argument types and exercise the runtime adapter, rather
than adding another permissive shim. Run every shared vector through those realistic
doubles.

### Parent shapes

`pomodoroHomeParentTarget` currently unwraps any singleton array and accepts ordinary
strings as parent paths. Rust's `NoteFacts` plus `projects::wikilink_path_target` only
accept scalar wikilinks. A scratch JS case with a ref note `parent: ["[[dev]]"]`
incorrectly resolves to `dev.md`; Rust's contract requires `@unfiled`. Plain
`parent: dev` is also accepted by the JS code.

Distinguish the Obsidian parsed shape of unquoted scalar `parent: [[dev]]` (`[["dev"]]`)
from actual YAML list parents such as `["[[dev]]"]`. Accept the former and quoted scalar
wikilinks; stop on list-form and bare-string parents. Do not broaden Rust's contract.
Add small hand-computed shared vectors for these shapes to bob-cli's vector file, copy
it byte-for-byte to the plugin fixture, update its provenance note, and assert
disk/indexed Rust and JS agreement. Retain the existing alias, self, depth, and
multiple-parent cases. Remove the duplicate `resolution-and-lints` row from the
contract's vector index while updating it.

### Reading-view refresh

In `274-plugin-pomodoro-home-chips-reading.js`, `pomodoroHomeReadingEntry` includes the
existing home chip in `ownText`. In the existing Reading-view DOM test, after the first
successful render of BLOG with a sase chip, calling the parser again returns authored
name `BLOGsase` instead of `BLOG`.

Exclude the plugin's own chip and other presentation decorations from source text, or
remove the old chip before parsing with safe cleanup. Add repeated
render/metadata-refresh coverage for named and unnamed entries, including an unnamed
review bridge. Assert stable authored identity, review kind, suppression, one chip, and
correct recomputation after a task or parent changes. Continue to skip Source mode and
support normal and embedded Task Links in Reading view.

## 2. Complete Mac override presentation and regenerate real fixtures

Work in bob-mac-capture. The current forecast rings, Later accessibility text, Refs
Today display-only routing, and toast lookup are already implemented.

- In `Sources/CaptureCore/CapturePomodoroStartPresentation.swift`, unnamed restarts
  still set `pomodoroName = summary.pomodoroName ?? "next session"`. The banner uses
  `previous.homes`, but the restart title, status, accessibility, and notifications
  still say "next session". For a restart, use the existing optional
  `override.previous.homes` label as the display fallback, respecting a non-null chip.
  Keep authored names and the legacy/no-homes fallback. Do not apply the displaced
  session's label to a different new swap destination. This is presentation only; never
  write the display fallback into the capture.
- Decode optional `homes` on `CaptureCompleteOverrideRunning` in
  `Sources/CaptureCore/CaptureModels.swift`. In `pomodoroStartPickerStatus` in
  `CapturePickerPresentation.swift`, use it to identify an unnamed running session for
  both ledger-kept `=#` and fresh `=3#` override contexts. Preserve authored-name,
  absent-object, and absent-homes behavior. This completes the concurrent 9j empty-name
  integration, not the unrelated candidate-label work tracked by `bob-cli-9n`.
- Add presentation/decoder coverage for unnamed-derived restart, named restart,
  swap/demoted text, both empty name-field override forms, and legacy payloads.
- Regenerate the relevant Ref Today, unnamed override, and completion fixtures from real
  current `bob ref today -f json`, `bob capture --dry-run -f json`, and
  `bob capture-complete` output against a deterministic scratch vault and pinned
  `BOB_NOW`/config. Document a repeatable command or fixture generator. Preserve legacy
  no-homes payloads. Do not touch the real vault to obtain mutation output. The current
  `refs-today-homes.json` is synthetic: it has named-row homes that this CLI omits, and
  nonempty work labels with empty parts. Keep artificial forward-compatibility model
  cases separate from wire fixtures if useful. Do not claim hand-edited JSON was
  regenerated.

These fixes are in portable Core targets. Run `just portable-check` to compile and test
all changed portable Swift. If app-only Swift needs modification, compile it in a
scratch Mac copy as required by AGENTS.md; attempt Mac `bundle.sh` and
`just typecheck-tests` best effort, without using Bryan's checkout or installing there.
If the Mac is offline, report the exact unverified targets. Existing CI proves the
current app-only ring/routing code builds. Do not make this tale's closeout wait for its
own future commit SHA, push, or CI result.

## 3. Validate the actual UI and complete the release checks

The required Obsidian check was never done by either plugin phase; it remains part of
this epic, not an out-of-scope task. Athena has an Obsidian executable, `xvfb-run`, and
Chrome. Check that the executable is usable and run an isolated Obsidian process/profile
with a scratch vault if the Mac is unavailable. Use the real application metadata cache
and rendering lifecycle, rather than counting the permissive fake-DOM test as live
validation. A scratch vault containing the current daily-note example and a Stats report
generated by the installed bob is sufficient; do not modify the user's working Obsidian
layout or ledger.

Inspect screenshots or rendered DOM in light and dark mode: Live Preview and Reading
chips show the correct homes and forecast/attributed dots, Source skips chips, repeated
refresh preserves identity, and clicking/pressing Enter on a Worked chart bar refreshes
Where the time went and Themes to the pinned span. Record evidence as SASE artifacts. If
a real app check truly cannot run, record the blocker honestly and keep the epic open;
do not silently waive acceptance.

Run the following after the final edits:

- bob-cli `just check`, covering the shared vectors and H-vectors; **never
  `just check-full`**. Then `just install` because the vector/doc update belongs to this
  landing and consumers should use the verified current CLI.
- `bob stats show worked review_time -f json -r 4w`: tile equals the sum of worked's
  `@review:*` project parts; repeat with pinned time for determinism.
- bob-plugins `npm run build`, `npm test`, and `npm run validate`. The six known
  failures in `bob-cli-6v`, `bob-cli-6w`, and `bob-cli-7i` are the documented baseline;
  new failures are not acceptable. Keep the mirrored vector bytes equal.
- Bump the bob-ledger-tools patch version, update its README table, run
  `bob plugins sync` from the opened source checkout, and confirm the subsequent dry run
  has nothing to copy.
- Mac `just portable-check` as above. The existing unrelated macOS XCTest compile
  failure remains `bob-cli-9m`, not this epic's work.

Use `/sase_monitor` for long checks and preserve the remaining closeout in its
continuation instructions. Any monitor continuation must still execute step 4; do not
use a prepared completion that only commits and forgets ancestor landing.

## 4. Land this epic, then recheck and close its parent when complete

Follow-up triage is already recorded in `bob-cli-9k.8`'s landing note: `9k.8.3` note 1's
live Obsidian proposal is retained as epic work; the Mac CI limitation in `9k.8.4` note
1 corroborates `bob-cli-9m`; no new task was created. The parent proposals retain their
earlier outcomes: scan_hooks rustfmt fixed; six plugin failures tracked by 6v/6w/7i,
ranker flake by 3w, Mac test compiler failure by 9m, and other pre-existing placeholder
surfaces by 9n. Include these outcomes and reasons in the close note. Triage any
genuinely new unrelated issue through `/sase_new_task`; finish epic-caused defects here.

1. Re-read
   `sase bead read bob-cli-9k.8 -r "Need final scope, notes, descendants and parent link"`
   and each child with its notes; inspect code and any new commits in all opened
   repositories. Confirm all required work and the accepted decisions are met.
2. Run `sase bead epic-symbols bob-cli-9k.8`. Resolve every listed exemption by wiring,
   privatizing, a legitimate non-test pragma, or deletion under Symvision policy; re-key
   only to a still-open later bead that needs it. Recheck until empty, then run
   `sase bead close bob-cli-9k.8 --note "<specific verification, integration, test results and all follow-up outcomes>"`
   normally. If named phases are incomplete, finish/reopen them. Never force a
   successful landing.
3. Run `just symvision` if that recipe is available (the current bob-cli justfile has no
   such recipe). Open the plans sidecar through `/sase_repo`, resolve
   `plan:202610/pomodoro_homes_landing.md`, and set only its frontmatter `status: done`
   after the successful close. Use the PLAN path reported for the current workspace, not
   a hardcoded checkout path.
4. The direct parent is plan bead **bob-cli-9k**. Re-read its previous landing note, all
   descendants and notes (9k.1-.7, 9k.8 and its children), the linked
   `plan:202610/pomodoro_homes.md`, and any post-child drift. Rerun descendant readiness
   and linked-plan readiness checks (`sase plan links validate --show-warnings`,
   inspecting findings relevant to these plans). Confirm the child is closed, all
   required descendant work is complete, and its plan is done. Inspect new issues
   instead of assuming closed beads prove completion.
5. If the parent is complete, first retire every entry from
   `sase bead epic-symbols bob-cli-9k`, then close it normally with
   `sase bead close bob-cli-9k --note "<what was rechecked, verification and follow-up dispositions>"`,
   run `just symvision` if available, and set `status: done` in its linked
   `202610/pomodoro_homes.md` plan file. Never use `--force` to advance a successful
   nested landing. The inspected parent has no parent of its own; if that changed, apply
   the same recheck to directly parented plan ancestors. Stop at an incomplete/ambiguous
   parent and append a precise blocker note there. If the chain reaches a phase parent
   instead, verify and close only that phase normally, leaving its containing epic to
   its land agent.
6. Use `/sase_final` for every changed repository, including plans and linked repos,
   after all work and checks. The host commits after the turn; closeout above must
   happen before that and needs no SHA or push from this tale.
