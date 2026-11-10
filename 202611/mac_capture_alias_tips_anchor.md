---
tier: tale
title: Make Mac Capture typing tips actually appear
goal:
  A Shift-typed capture sigil that bob proves an alias for shows its "Next time" row on
  the first list or picker anchored at that sigil, even when Bryan keeps typing the
  query or a list from the previous token is still on screen.
size: medium
decisions:
  drop_reason_counts:
    ask: Should the typing-tips ledger count why each proved tip was not shown?
    default: true
    why:
      The ledger alone could not name the cause of this bug; reason counts would have.
    answer: true
proposed_by: bbugyi200.athena.1at
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 07:39:55
status: wip
---

# Plan: Make Mac Capture typing tips actually appear

## Diagnosis (already established; do not re-investigate)

Bryan's suspicion is right. The capture alias typing tips ("Next time `.` → `@`") have
never appeared in the shipped Bob Mac Capture.

**bob is correct.** `bob capture-rewrite --cursor N -f json` returns a proved
`sigil_alias_tip` on Bryan's Mac. Examples: `Buy milk @` gives `route_after_text`,
`@file:` gives `link_after_route`, `@file^` gives `block_id_after_route`, and `@file+`
gives `parent_task`. The call takes about 10–20 ms there, while `capture-parse` takes
10–20 ms and `capture-complete` 70–80 ms. bob-cli needs no change.

**The Mac drops every proved tip.** Bryan's ledger on the Mac
(`defaults read org.bobs.bob-mac-capture org.bobs.bob-mac-capture.typing-tips`) has
`tipsShown: 0` for every lesson. Yet this week it recorded 19 `route_after_text` misses,
8 `link_after_route`, 2 `relation`, and 1 `parent_task`, all as `shown: false`. So the
intent → lane → reply path works, and the admission logic throws every tip away. All of
it lives in the linked repo `bob-mac-capture`,
`Sources/BobMacCapture/CapturePanelModel.swift`, in the typing-tips extension
(`startAliasTipLane`, `applyAliasTipRewrite`, `consumePendingAliasTipOnListUpdate`,
`consumePendingAliasTipOnPickerOpen`, `retireAliasTipDisplayEligibility`,
`aliasTipIsAdmissible`) and in the top of `editorTextDidChange`.

The repair commit `542a37d` ("repair alias tip lifecycle and undo accounting")
introduced two root causes:

1. **Exact-draft binding kills the tip during normal typing.** `editorTextDidChange`
   retires the request or pending tip whenever `aliasTipRequest.draft != draft`. It also
   retires a shown row whenever `presentedAliasTipDraft != draft`. The list or picker
   the sigil opens only appears after the 50 ms analysis debounce plus `capture-parse`
   plus `capture-complete`, and every keystroke restarts that debounce. Typing `@er`
   fluidly therefore always changes the draft before the surface exists, so the tip is
   dropped as a miss. If a tip ever did show, the first filter keystroke would erase it.
   That contradicts the original plan's rule ("valid while the draft prefix through
   `range.end` is unchanged … keep it unchanged while the query filters") and
   `docs/typing-tips.md` ("Filtering leaves the row in place"). This accounts for the
   `route_after_text` misses.
2. **A stale surface from the previous token is treated as the sigil's surface.**
   `startAliasTipLane` stamps `firstSurface` from `pickerVisible` / `completionVisible`
   at keystroke time, and `applyAliasTipRewrite` drops the reply when `firstSurface` is
   non-nil. When Bryan types `@errands` and then `:`, `^`, or `+`, the route list for
   `@errands` is still visible. The request is therefore born "already opened", and the
   tip is dropped every time. This accounts for the "right after a route" picker
   lessons. It also hits any sigil typed while the previous token's list is still up.

The existing tests miss both causes. The list tests type the sigil and then wait with a
200 ms debounce and no further keystrokes. The picker tests call `applyAliasTipRewrite`
and `presentPicker` directly on a draft that is only the sigil, with no preceding route
list.

## Design: the sigil's surface is the one anchored at the sigil

Replace the visibility-snapshot and exact-draft rules with one anchor rule. Keep every
other promise of the feature: bob proves every tip; first frame or never; late replies
are dropped; at most one tip per panel session; tips never gate capture; ledger and
fading are unchanged.

- **Sigil range.** When `startAliasTipLane` consumes a `TypedSigilIntent`, it stores the
  sigil's UTF-8 byte range in the request: `[caret - sigil.utf8.count, caret)`. It sends
  bob exactly the keystroke-time draft, as it does today. That draft must not change:
  bob proves `Buy milk @` but answers `null` for `Buy milk @er`.
- **Sigil intact.** Generalize `aliasTipIsAdmissible` into one predicate used by the
  request, the pending tip, and the shown tip. The current `plainDraft` prefix through
  the sigil's end must equal the request draft's prefix through the same end, and the
  sigil bytes must still sit at that range. Typing after the sigil (the query) keeps the
  predicate true. Any edit at or before the sigil, including deleting it, makes it
  false.
- **Anchored surface.** A completion list (`completionResponse.replacement`) or a picker
  (`CapturePickerState.replacementRange`) is anchored at the sigil when its replacement
  range starts at the sigil's start or end. Real bob output confirms both forms:
  `Buy milk @er` route `{10,12}` and sigil `{9,10}`; `Buy milk #ho` task_type `{9,12}`;
  `@errands^` task_block_id `{9,9}`; `Fix it ~` relation_to `{7,8}`; `Buy milk +`
  task_parent `{9,10}`. A surface visible at keystroke time is never "already opened"
  for the new request unless it is anchored at the new sigil, which cannot happen
  because the sigil did not exist yet.
- **Admission.** The first current-generation list or picker opening that is anchored at
  the sigil, while the sigil is intact, is the sigil's surface. If a valid pending tip
  exists for that request and the existing gates pass, present it in that same
  main-actor update. The existing gates are: lesson host matches the surface, the
  session allows a tip, the lesson is due, and the panel budget fits. Then record
  `recordMiss(shown: true)` and `markShown()` as today. The opening consumes the request
  whether or not a tip was pending.
- **Late reply.** If an anchored surface already opened for this request before bob's
  reply arrives, drop the reply as a miss. This keeps "first frame or never".
  `applyAliasTipRewrite` keeps its existing reply validation: generation, `changed`,
  `input`/`text`/`cursor`/`edits`/`rule` equality against the request draft, capability,
  explicit null → `recordIneligible`, and unknown ids. It replaces only the
  `firstSurface == nil, displayEligible` gate with "sigil intact and no anchored surface
  consumed yet". Keep the fallback that builds a request when none exists, which test
  helpers use. In that fallback, an anchored surface that is currently visible counts as
  already opened, so `testPickerLessonsAreAdmittedOnOpenAndLateRepliesDrop` keeps its
  late-drop half.
- **Opportunity passed.** The first current-generation analysis that settles after the
  sigil decides. If it opens or updates a list or picker that is not anchored at the
  sigil, or ends in `handleMissingCompletion` with nothing to show, drop the pending tip
  (or the in-flight request) as a miss. Continuous typing does not settle an analysis,
  so this decision waits for Bryan's first pause, which is the moment the sigil's list
  can open.
- **Shown row lifetime.** While shown, the row stays as long as the sigil is intact and
  the visible list or picker stays anchored at the sigil. List filtering by typing more
  of the query and picker-filter typing both keep it. Clear it as today when the surface
  closes or re-anchors elsewhere, on accept, submit, panel hide, or tips toggled off,
  and also when an edit breaks the sigil. In `editorTextDidChange`, replace the two
  exact-draft comparisons with the sigil-intact predicate. Keep
  `retireAliasTipDisplayEligibility` for the explicit retire paths (accept, submit,
  client change).
- Remove `firstSurface` initialization from `startAliasTipLane` and drop the
  `listWasVisible` / `pickerWasVisible` "already visible" gates in favour of the anchor
  checks. A list or picker that was already anchored at this sigil and merely updates is
  not a new opening. It keeps or clears an existing row by the lifetime rule above.

> [!decision] drop_reason_counts Add an additive `dropped: [String: Int]` map to
> `AliasTipLedger`, decoded with `decodeIfPresent` so the version stays 1 and old blobs
> load. Each site that records `recordMiss(shown: false)` for a proved tip increments
> one stable snake_case reason: `late_reply`, `sigil_edited`, `unanchored`,
> `superseded`, `closed` (dismiss, accept, submit, hide, or client change before
> display), `not_due`, `session`, `host`, and `budget`. No draft text is stored. Reset
> clears it. Document the field in `docs/typing-tips.md` § Ledger. If the answer is no,
> skip this block.

## Tests (bob-mac-capture)

Add these to `Tests/BobMacCaptureTests/CaptureAliasTipPanelTests.swift`, driving the
real `typeSigil` / `insert` / `editorTextDidChange` path with fake bob. Generate the
fake-bob fixtures from a real `bob` build so they match the shipped contract. Use
`bob capture-rewrite --cursor N -f json -- "<draft>"` plus the matching `capture-parse`
/ `capture-complete` replies, and extend the existing exact-draft branches in
`Tests/Fixtures/fake-bob`. Bound async waits with `waitUntil`, and never await an
untimed gate.

1. **Typing the query before the list opens keeps the tip.** Shift-type `@` on
   `Buy milk `, then immediately insert `er` before the debounce elapses. When the route
   list opens anchored at byte 10, the `route_after_text` row is presented and
   `tipsShown == 1`. This fails on today's code.
2. **Filtering keeps the shown row.** After the row shows for `Buy milk @`, insert `e`.
   The list updates, stays anchored, and the row is unchanged.
3. **A route list left open does not block the picker tip.** Start from `@file` with its
   route list visible, Shift-type `:`, and the `pomodoro_block_id` picker opens. The
   `link_after_route` row is presented, with `tipsShown == 1`. Add the same case for `^`
   → `block_id_after_route`, or table-drive both. This fails on today's code.
4. **Editing at or before the sigil retires.** Deleting the sigil before the reply still
   drops the tip (existing `testEditBeforeTheReplyDropsTheTip`). Deleting it after the
   row shows clears the row.
5. **An unanchored settle drops the tip.** Shift-type `@`, then type a space quickly so
   the first settled analysis offers nothing. The tip is a miss with `shown: false`,
   plus `dropped.unanchored` under the decision.
6. Keep every existing typing-tips test green with its current meaning. That covers late
   replies, dismiss-then-reopen, submit, hide/show, one tip per session, the old-bob and
   null capability paths, hits and Undo withdrawal, escapes, and the short-screen picker
   budget. Adjust a test only where it asserted the removed exact-draft or
   visibility-snapshot behavior, and say so in the commit message.
7. Under the decision, add ledger unit tests in
   `Tests/CaptureCoreTests/AliasTipLedgerStoreTests.swift`: the reason counts
   round-trip, a blob without `dropped` decodes, and reset clears it.

## Docs

`docs/typing-tips.md` § "Where and when" must say three things:

- The tip waits for the first list or picker anchored at the sigil, even if Bryan keeps
  typing its query first.
- A list or picker left from the previous token does not count.
- Editing at or before the sigil, or a first pause that shows no anchored surface, drops
  it.

Keep "Filtering leaves the row in place", which becomes true. Update § Ledger under the
decision. Run `just docs-check` in bob-mac-capture. The bob-cli glossary strand
`glossary:capture-alias` ("shows it as a fading 'Next time' row in the list or picker
the sigil opened") stays accurate, so no memory edits are planned.

## Working in bob-mac-capture

Open the linked repo with `sase repo open bob-mac-capture`. Work only in the printed
path, and read its `AGENTS.md` first. All code, test, fixture, and doc changes land
there. Follow its gates exactly:

- Never land Swift you have not compiled. Run `just portable-check` on athena, which has
  Swift.
- Compile a scratch copy on the Mac on a best-effort basis: rsync without
  `.build`/`.git`, run `bundle.sh` and `just typecheck-tests`, then delete the copy.
  Never touch Bryan's checkout or run `just install`. If the Mac is offline, say in the
  final report that the macOS targets are unverified.
- Format only changed lines per `.swift-format`: 4-space indent, 100 columns.
- Run `Scripts/macos-ci-check` before landing. After the push, watch the exact-SHA macOS
  run to green with `/sase_monitor`.

## Hand-check for Bryan after install

On the Mac with tips on, type each of these with Shift-typed sigils and no pause before
the query:

- `Buy milk @er`: the route list opens with "Next time `.` → `@` after a word" at its
  foot, and the row stays while the query filters.
- `@<route>` then `:`: the picker shows "Next time `;` → `:` right after a route".

`defaults read org.bobs.bob-mac-capture org.bobs.bob-mac-capture.typing-tips` should
show `tipsShown` ≥ 1. Under the decision, it also shows the `dropped` reasons for
anything that still did not show. Lessons the ledger already marks fluent stay quiet by
design: `hash_menu`, `block_id_after_route`, and `blocked_task`.
