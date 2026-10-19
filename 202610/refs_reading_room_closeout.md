---
tier: tale
title: Finish the Bob Refs reading-room landing
goal:
  Repair image rendering and verification gaps, then close bob-cli-9v with audited
  checks and its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-9v.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-9v
create_time: 2026-10-19 14:59:26
status: wip
---

- **PARENT:**
  [202610/refs_reading_room.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/refs_reading_room.md)
- **BEAD:**
  [bob-cli-9v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9v/README.md)

# Finish the Bob Refs reading-room landing

Repair the bounded defects found while landing **bob-cli-9v**, verify the repaired
reading room, and close that epic in this coder's turn. This tale has no land agent; do
not leave closeout to a successor or wait for this work's own commit, push, or CI.

All application changes belong in **bob-mac-capture**. Use `/sase_repo` and
`sase repo open bob-mac-capture -r "Finish bob-cli-9v reading-room landing"`, and read
that checkout's `AGENTS.md`. Do not edit a user's installed app or run `just install`.

## Audited context and preserved behavior

Read `sase bead read bob-cli-9v -r "Need landing audit and final decision"` and
`sase artifact read plan:202610/refs_reading_room.md "Need reading-room acceptance"`.
The epic's three phases are closed. Their source commits are `c424756`, `1d7242a`, and
`73d1106` in bob-mac-capture. The final decision is **standalone_notes = include**.
Preserve the implemented geometry, 5-item default and explicit stored page size,
compact/wide typography, read-only vault presentation, unchanged Copy projections, and
underline-aware quote rendering. Do not redesign those completed pieces.

The lander reviewed every phase note; there were **no PROPOSED FOLLOW-UP entries**.
Out-of-scope triage is already complete and recorded on bob-cli-9v: corroborated
**bob-cli-4k** (capture preview timeouts), **bob-cli-9s** (capture retry invocation
counts), and **bob-cli-8j** (pre-existing intrinsics-discard test), with evidence
`file:explicit:172ae9fb0f75b6f6bb453599`. No new task is needed for those failures. Read
that evidence with `sase artifact read` if needed. Epic-caused issues below remain epic
work and must not be filed away as follow-ups.

Integration audit: Mac `origin/master` at `73d1106` contains no non-epic commits after
the first phase. On bob-cli, concurrent `1dc6fe06`/`435e9c14` add and repair underline
emphasis without changing image JSON, and `67a26ea8` repairs vault sync. The current
inspector already uses `RefsQuoteExcerpt`, `RefsQuoteText`, and `RefsQuoteEmphasis`; no
additional integration edit was needed. Recheck new drift at implementation time.

## 1. Repair the image fixture crash and missed paging expectations

Latest CI at `73d1106`:
<https://github.com/bobs-org/bob-mac-capture/actions/runs/39304814575>. Lint and Build
pass. XCTest traps with signal 5 in
`RefsInspectorLoaderTests.testAnnotationImageLoaderReadsRasterPng`, before most Refs
tests and the new image PNGs. The lander reproduced the crash on the `mac` host (macOS
26.5, Swift 6.3.2) using the exact `makeTestImage` helper from
`Tests/BobMacCaptureTests/RefsInspectorTests.swift`: its default `hasAlpha=false`
chooses a three-sample RGB bitmap, and drawing with `NSRect.fill()` traps after
`color.setFill()`. The same standalone probe with a four-sample RGBA bitmap succeeds and
produces PNG bytes.

- Make this helper draw into a supported bitmap representation, preserving its point and
  pixel dimensions and intentional transparency behavior. Reuse the corrected helper in
  the scroll and image fixtures. Keep transparent fixtures meaningfully transparent. Do
  not suppress or skip the crashing test.
- Fix `testRevealMoreWithOnlyHighlightsEmitsRange`: it still creates only five
  annotations and expects `3..<5`, although the default is now five. Give it a real
  second page (for example eight annotations, expecting `5..<8`) and preserve its
  event/id assertions. This fails in phase-1 CI run **39289627052** and phase-2 run
  **39298547579**, and remains unchanged in `73d1106`.
- Fix `renderWideHighlightsUnified` in `RefsPanelDesignTests.swift`: only entries 6 and
  7 are hidden, neither has a note, yet the expected disclosure includes
  `1 with a note`. Keep the fixture meaningful and make the expected text agree with the
  actual hidden entries. Phase-2 CI already demonstrates the mismatch.

## 2. Make image caching bounded and respect file changes

`Sources/BobMacCapture/Refs/RefsAnnotationImages.swift` gives the provider a 128 MB
`NSCache`, but `RefsInspectorImageStore.loaded: [String: NSImage]` holds every decoded
image strongly under `path|maxPixel`. This bypasses eviction and the provider's mtime
key, so browsing many references retains all bitmaps and loading a changed file at the
same path returns old pixels. Reset/discard must also prevent an old asynchronous load
from repopulating newly invalidated state.

- Keep synchronous cache hits and deterministic preview injection, but remove the
  unbounded production retention. Use a bounded cache or bounded ownership shared with
  the provider, with a consistent byte-cost policy.
- Ensure cache identity/invalidation accounts for the resolved vault root, file version,
  and requested pixel limit. Clamp pixel limits consistently on lookup and insertion.
  Preserve the provider protocol's injectability without forcing tests to create
  unrelated real vault files.
- Prevent canceled or pre-reset results from overwriting newer state; coalesce equal
  requests if the implementation needs that to avoid duplicate retained work.
- Add targeted regression coverage for cache reuse, a same-path file replacement, reset
  during an outstanding load, and bounded retention. Do not assert a precise platform
  `NSCache` eviction order; test an explicit deterministic budget abstraction if one is
  introduced. Keep previews returned for arbitrary requested sizes.

## 3. Carry image availability through disclosure and accessibility

`RefsInspectorContent.feed` currently constructs the feed from raw annotations only. The
loader's `images` metadata map knows that missing, unreadable, or invalid paths failed,
but `hiddenImageCount` still counts every asset/attached path as an image.
`Entry.accessibilityLabel` only announces `unresolved_images`, while the unavailable row
hides its accessibility children. Thus a missing primary crop is silent to VoiceOver and
a missing attachment is counted as an available hidden image.

- Pass the already-prefetched availability into the presentation feed, preserving raw
  annotations and Copy-card projections. Count only available primary and attached
  images in the disclosure's image clause. Announce unavailable primary images, failed
  attachments, and serialized unresolved images accurately, without double-counting.
  Preserve byte-identical image-free strings and note-card counts.
- Test one mixed feed containing an available crop, a missing crop, an available
  attachment, a missing attachment, an invalid `../` path, an unresolved source, and a
  standalone note. Assert both hidden counts and entry accessibility strings.
- `RefsInspectorImageView` also needs a terminal decode-failure state: a readable header
  followed by a nil bitmap currently leaves the photo placeholder forever. Show and
  announce failure without changing the reserved height after publication. Initial
  metadata failures still use the compact unavailable row; for a late failure keep the
  reserved space and put the unavailable caption within it. Preserve stable layout, and
  guard view state against stale task completion when its request changes.

## 4. Make the visual and no-jump verification effective

The lander downloaded phase-2 `render-fixtures` from run **39298547579** and inspected
all 1344-point browse, paper, chat, highlights, noted, revealed, and Copy states in both
appearances. Hero, thumbnail, and readable-column geometry are present. However, the
disclosure and reveal backgrounds render bright yellow unsupported-view markers with red
prohibition glyphs: the static `ImageRenderer` preview is still mounting
`NSViewRepresentable` anchors. Phase-3 image fixtures were never reached in its CI run
because of the helper crash.

- In `RefsInspectorView.swift`, omit disclosure and reveal AppKit anchors in
  `previewMode`, as is already done for the live scroll container. Keep anchors in the
  live panel so Ctrl+D containment and follow-tail work unchanged.
- Strengthen `testHostedRevealKeepsDocumentHeightWhileImageBitmapsLoad`. Its current
  `offset > 0` can already be true before reveal, and fixed sleeps do not prove that
  bitmap completion lies between its two height measurements. Use a bounded,
  controllable provider gate with observable request/completion state: reveal page two,
  observe the new reveal target/anchor while pixels are held, record height, release the
  bitmap, observe completion, then assert unchanged height and correct follow position.
  Every wait/gate must time out and release during cleanup.
- Cover both wide and compact image sizing, success and late failure, and the
  disclosure-visible-but-not-at-bottom behavior. Retain explicit page-size coverage.
- Generate and inspect both appearances of the wide states and 880/1344 image states:
  primary crop, comment plus attachment, attachment without comment, standalone note,
  unavailable image, transparent diagram on white, and a revealed image page. No
  unsupported-view markers, clipped images, or lost disclosure text. Make the long wide
  quote actually exceed six body lines so the intended truncation is visible. Preserve
  the original compact and 700-point list-only references.

## 5. Verify, recheck drift, and finish bob-cli-9v in this turn

Run `just portable-check` in the linked Mac repository. The host repo's file-change gate
is `just check`; run it before closeout, and **never run `just check-full`**. Use
`/sase_monitor` for long commands, with continuation instructions that retain this
tale's remaining verification and closeout obligations.

The `mac` host was reachable during the audit. After reading tailnet memory, use an
isolated scratch copy (rsync excluding `.build` and `.git`), run `Scripts/bundle.sh` and
`just typecheck-tests`, and delete the scratch copy afterward. Never use Bryan's Mac
checkout. The host has Command Line Tools, so the XCTest stub proves type checking only;
do not claim it executed tests. Re-run the standalone image fixture probe and focused
runtime image/cache probes there, using the actual repaired implementations. Render the
affected views in a scratch runnable ImageRenderer harness if current CI artifacts do
not contain the repaired code; reuse the existing fixture setup, and use real assertions
rather than the type-check-only XCTest stub. This is verification scaffolding, not a new
shipped test runner or CI redesign.

Inspect existing CI and distinguish known baseline failures (4k, 9s, 8j) from new
reading-room failures. Fix every epic-caused compile, test, fixture, or runtime failure
you observe. Record which checks actually ran. Do not wait for this tale's own commit
SHA, push, or future CI: its commit occurs only after its provider turn ends, and **the
epic closes in the same turn as the final code**. Report the resulting CI-only
limitation honestly; the portable checks, Mac compile, and direct runtime verification
above must not be replaced with a future-CI promise.

Recheck new non-epic commits in both repositories since the audited revisions and
integrate any relevant changes. Re-read all three child beads and the epic's notes to
ensure there are no new unresolved items. Follow-up outcomes already recorded are
4k/9s/8j corroboration, no child proposals, and no new tasks; include those dispositions
in the close note along with final verification.

The final actions are concrete:

1. Run `sase bead epic-symbols bob-cli-9v`. For **every** listed entry, wire it up,
   privatize it, add an appropriate non-test pragma, or delete it per policy; re-key
   only if a specifically identified still-open later bead needs the exemption. The
   audit found none, but check again before close.
2. Close normally with `sase bead close bob-cli-9v --note "<verification>"`. The note
   must name the three phase commits, preserved standalone-notes decision, drift review,
   repaired gaps, actual check outcomes/runtime limits, and the three
   duplicate-follow-up dispositions. If close rejects stale symbols, resolve and retry.
   If it names incomplete descendants, finish them. Never use `--force` just to get a
   successful close or advance a successful nested landing.
3. After closing, run `just symvision` wherever available; both audited justfiles
   currently lack it, so explicitly record unavailability if unchanged.
4. Set **`status: done`** in the frontmatter of the epic's linked plan,
   **`plan:202610/refs_reading_room.md`**. Obtain its PLAN path from the audited bead
   read and use `/sase_repo` to open the plans sidecar before editing; do not assume
   another agent's absolute path. Do not change its approved decision or content.
5. Run `sase bead read bob-cli-9v -r "Need the parent link"`. The audited epic has **no
   parent_bead**, so no ancestor closeout is currently required. If that link changed,
   apply the original landing prompt's normal parent-phase or fully rechecked
   parent-plan closeout rules, never a forced cascade.

Finish through `/sase_final` with the linked-repository and plan-file changes included.
