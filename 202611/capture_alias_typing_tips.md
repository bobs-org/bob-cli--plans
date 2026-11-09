---
tier: epic
title: Capture alias typing tips
goal: 'When Bryan Shift-types a capture sigil that an unshifted alias key would have
  typed at that exact spot, the list or picker the sigil opens shows one bob-proved
  "Next time" line in its first frame. The tip fades as the alias becomes habit, and it
  never slows or changes capture.

  '
phases:
  - id: bob-alias-tips
    title: bob proves tips and stamps alias ids
    depends_on: []
    size: medium
    description:
      "bob-alias-tips: stamp an AliasId on every forward sigil alias, add the
      sigil_alias_tip reverse check that runs the whole rewrite pipeline, emit tip and
      alias JSON from capture-rewrite, and pin it with round-trip, negative, and CLI
      tests plus docs."
  - id: mac-tip-core
    title: Portable tip model, policy, and ledger
    depends_on: []
    size: medium
    description:
      "mac-tip-core: decode the new capture-rewrite fields in CaptureCore and add the
      lesson map, presentation strings, the pure fade-by-fluency policy, session rules,
      and the UserDefaults JSON ledger with weekly counts, all unit-tested."
  - id: mac-completion-tips
    title: Trigger, tip lane, and the completion-list host
    depends_on:
      - bob-alias-tips
      - mac-tip-core
    size: medium
    description:
      "mac-completion-tips: record physically Shift- or Option-typed @ and # sigils, run
      the non-blocking alias-tip lane, admit tips only in the list's first frame, count
      adoption and escapes, add the Settings toggle and reset, pin the shared tip row in
      CompletionList, and add the Typing tips docs page."
  - id: mac-picker-tips
    title: The picker host for the remaining lessons
    depends_on:
      - mac-completion-tips
    size: medium
    description:
      "mac-picker-tips: widen the trigger to ^ : + ~ & *, admit tips in the picker's
      first frame in the chosen slot, add accessibility, escapes, tests, and docs, apply
      the accepted memory decisions, and propose the three-week stop-rule review."
decisions:
  picker_tip_slot:
    ask:
      Should picker tips sit at the foot of the picker card, the same slot the
      completion-list tip uses?
    choices:
      card_foot:
        Last line of the picker card under a hairline; the key legend keeps only live
        keys
      key_legend:
        Extra line under the panel footer's live key hints; picker card height never
        changes
    default: card_foot
    why:
      One placement rule for every host, and a not-live key never sits among live key
      hints
    answer: card_foot
  memory_capture_alias_glossary:
    ask: Should the Capture Alias glossary strand gain one sentence about typing tips?
    memory:
      - glossary:capture-alias
    default: false
    answer: true
  memory_typing_tips_decision:
    ask: Should the decisions web gain a typing-tips record once picker tips ship?
    memory:
      - decisions:typing-tips-are-bob-proved
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.3y.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 12:18:37
status: wip
---

- **PROMPT:**
  [prompts/202611/capture_alias_typing_tips.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_alias_typing_tips.md)

# Capture alias typing tips

## Goal

Bob Mac Capture swaps easy keys for capture sigils (`Buy milk .` → `Buy milk @`), but
nothing teaches that. When Bryan pays for Shift to type a sigil that an unshifted alias
key would have typed at that exact spot, the list or picker the sigil opens shows one
quiet line at its foot:

```text
 Buy milk @▌
 ╭───────────────────────────────────────────────╮
 │ ▌errands                            area      │  ← row 0 selected, unchanged
 │  cash                               project   │
 │  home                               area      │
 │ ───────────────────────────────────────────── │  ← the card's own hairline
 │  Next time  ⌈ . ⌉ → @  after a word           │  ← pinned tip row, 24 pt
 ╰───────────────────────────────────────────────╯
```

This epic implements the consolidated research report
`research:202611/capture_alias_typing_tips/capture_alias_typing_tips.md`. Bryan accepted
every requirement it recommends (R1–R12). Read it before starting any phase. It cites
`b81302b`, but the Mac line numbers below are re-verified at bob-mac-capture `90fa59d`.
This plan changes the research in four places:

- **New aliases.** bob-cli `0f31c592` added `.` → `&` and `;` → `*`. They join the v1
  lessons.
- **Card foot.** The picker tip uses the same slot as the list tip. See
  `picker_tip_slot`.
- **Capability marker.** New bob sends an explicit `"sigil_alias_tip": null`, so the Mac
  can tell an old bob apart from a draft that has no tip.
- **Weekly counts.** The ledger keeps weekly counts, so the three-week stop rule can be
  answered from data.

## Principles every phase keeps

- **bob proves every tip; the Mac never keeps an alias table**
  (`decisions:mac-capture-is-a-thin-client`). bob owns:
  - eligibility
  - the lesson id
  - the key or keys
  - the position phrase

  The Mac owns:
  - when to ask
  - whether the keystroke was physical
  - whether a lesson is due
  - the pixels, accessibility, and local learning state

- **The tip never slows or changes capture.** It runs on its own lane and gates nothing.
  With tips on, the draft bytes, selection, Undo history, completion choice, and capture
  result match tips off exactly.
- **First frame or never.** A tip appears in the opening frame of the surface the sigil
  opened. A tip that arrives later is dropped. It stays put, unchanged, while the query
  filters, and it ends when that surface closes.
- **At most one tip per panel session.** A panel session runs from show to hide.
- **No icon, emoji, "Tip:" label, button, focus stop, or animation.**

## Wire contract (all phases code against this)

`bob capture-rewrite --cursor N --format json` grows two additive fields.
`schema_version` stays `1`.

**1. `sigil_alias_tip`.** It appears on every response that has `--cursor` and
`changed: false`:

- an object when bob proves an alias for the sigil that ends at the cursor
- an explicit `null` otherwise

It is absent when there is no `--cursor` or `changed: true`. The explicit `null` is the
capability marker: an older bob omits the key entirely.

```bash
bob capture-rewrite --cursor 10 --format json -- 'Buy milk @'
```

```json
{
  "ok": true,
  "schema_version": 1,
  "input": "Buy milk @",
  "text": "Buy milk @",
  "changed": false,
  "cursor": 10,
  "edits": [],
  "sigil_alias_tip": {
    "id": "route_after_text",
    "sigil": "@",
    "keys": ["."],
    "range": { "start": 9, "end": 10 },
    "where": "after a word"
  }
}
```

- `range` holds the UTF-8 byte range of the typed sigil.
- `keys` lists bob's preference order; the Mac shows `keys[0]` only.
- `where` is optional. bob omits it when no short phrase is true.

**2. `alias`.** It appears on every `rule: "expand_sigil_alias"` response:

```json
"alias": { "id": "route_after_text", "key": ".", "sigil": "@" }
```

The Mac counts adoption from `alias.id` and never parses `summary`.

### Alias ids and v1 lessons

Every forward alias gets a stable id. Only the **tipped** ids ever appear in
`sigil_alias_tip`.

| id                              | Key → sigil             | Tipped                                                   | `where`                                                                                   | Surface it opens                                             |
| ------------------------------- | ----------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `route_after_text`              | `.` → `@`               | yes                                                      | `after a word`                                                                            | Completion list                                              |
| `hash_menu`                     | `/` (then `\`) → `#`    | yes                                                      | `after a word` after a space or tab; `right after a route` glued to a route; else omitted | Completion list (type, drawer, section, name, heading menus) |
| `block_id_after_route`          | `.` → `^`               | yes                                                      | `right after a route`                                                                     | Picker                                                       |
| `link_after_route`              | `;` → `:`               | yes                                                      | `right after a route`                                                                     | Picker                                                       |
| `task_link_item`                | `;` → `:`               | yes                                                      | `at an item start`                                                                        | Picker                                                       |
| `move_bullet_caret`             | `.` → `^`               | yes                                                      | `on a move bullet`                                                                        | Move picker                                                  |
| `move_bullet_colon`             | `;` → `:`               | yes                                                      | `on a move bullet`                                                                        | Move picker                                                  |
| `parent_task`                   | `,` → `+`               | yes                                                      | `right after a route` for the note-scoped `Task` context; `after a word` for `TaskParent` | Picker                                                       |
| `relation`                      | `=` → `~`               | yes                                                      | `after a word`                                                                            | Links To picker                                              |
| `dependency`                    | `.` → `&`               | yes                                                      | `once routed`                                                                             | Depends On picker                                            |
| `blocked_task`                  | `;` → `*`               | yes                                                      | `after a word`                                                                            | Blocks picker                                                |
| `relation_flip`                 | `=` → `~~`, `~~=` → `~` | no: sits next to a fast double-tap escape (R8)           | —                                                                                         | —                                                            |
| `move_choice`                   | `- ^.` → `- ^^`         | no: sits next to a double-tap choice (R8)                | —                                                                                         | —                                                            |
| `next_parent`                   | glued `,` → ` +`        | no: a two-byte edit the one-byte proof cannot cover (R7) | —                                                                                         | —                                                            |
| `past_segment`                  | `-` → `~` in `x=`       | no: deferred (R7)                                        | —                                                                                         | —                                                            |
| `close_complete` / `close_park` | `x` / `s` → `!` / `*`   | no: deferred to the Close card (R7)                      | —                                                                                         | —                                                            |

`dependency` reads `once routed` because `.` gives `&` only where `.` → `@` cannot fire,
which means the item already has a route. In `Buy milk &`, `.` would give `@`, so the
proof fails and no tip shows. That is the reverse check doing its job.

## Compatibility

| bob | Mac | Behavior                                                                                            |
| --- | --- | --------------------------------------------------------------------------------------------------- |
| new | new | Tips are shown.                                                                                     |
| new | old | The old Mac ignores the new fields.                                                                 |
| old | new | The key is absent, so the Mac stops spawning the tip lane for that bob and shows no tips, silently. |
| old | old | Unchanged.                                                                                          |

## In-flight work to respect

Epic `bob-cli-j6` adds the `.` → `&` / `;` → `*` aliases.

- Its bob phase has landed (`0f31c592`).
- Its Mac phase, `bob-cli-j6.2`, was in progress at planning time. It adds escapes for
  the `&` and `*` aliases.

Mac phases must run `sase bead read bob-cli-j6 -r "<why>"` and build on whatever
`bob-cli-j6.2` landed. If it has not landed, do not touch its escape arms. Record a
`PROPOSED FOLLOW-UP:` for the `&`/`*` withdrawal hooks instead.

## Docs budget (applies to every phase)

Mounted Mac reference pages already exceed the 6000-word reference cap, tracked by
`bob-cli-h9`:

- `capture-panel.md`
- `capture-previews.md`
- `keyboard.md`
- `protocol.md`
- `refs.md`

Follow these rules:

- Never add words to a page that is over the cap.
- The new Mac page `docs/typing-tips.md` owns the Mac behavior and the compatibility
  table.
- bob-cli `docs/capture.md` owns the JSON contract.
- Read `sase memory read docs.md -r "<why>"` before editing docs.
- Run `just docs-check` in each repository you touch. Compare against the baseline
  failures on master; your change must add no new ones.

## Phase `bob-alias-tips` — bob proves tips and stamps alias ids

Work in bob-cli. This phase has no dependencies.

1. **Stamp alias ids.**
   - Add an `AliasId` enum with a `code()` that returns the ids in the table above.
   - Give `DraftRewrite` in `src/native/capture_language/rewrite.rs` an
     `alias: Option<AliasStamp>`, holding the id, key, sigil, and an optional `where`
     phrase.
   - Every non-alias constructor sets `None`.
   - Pass the id into `finish_sigil_alias`, near line 1059.
   - Stamp the three hand-built alias rewrites as well:
     - `finish_relation_double_flip_alias`
     - the glued-comma ` +` continuation in `expand_parent_task_comma_alias`
     - `expand_merge_in_source_comma_alias`
   - Pick each id at the branch that already knows its completion context:
     - `PomodoroBlockId` → `link_after_route`
     - `TaskLink` → `task_link_item`
     - `ProjectTaskMove` → `move_bullet_*`
     - the `Task` vs `TaskParent` comma contexts → `parent_task`
   - Set the `where` phrase at those same branches, so context-dependent phrases
     (`parent_task`, `hash_menu`) come from the code that knows the context.
   - Behavior and existing output stay byte-identical.

2. **Reverse check.** Add a new module, `src/native/capture_language/alias_tip.rs`, so
   `rewrite.rs` does not grow. It is lexical and does no vault I/O:

   ```text
   sigil_alias_tip(raw, cursor) -> Option<SigilAliasTip>
     return None for a template directive draft, or when the byte ending at cursor
       is not one of @ # ^ : + ~ & *
     for key in CANDIDATES[sigil]   # @:. #:/ then \  ^:.  ::;  +:,  ~:=  &:.  *:;
         candidate = raw with [cursor-1, cursor) replaced by key
         r = rewrite_draft(candidate, Some(cursor))        # the WHOLE pipeline
         keep key when r.rule == ExpandSigilAlias, r.text == raw,
           r.cursor == Some(cursor), and r.alias.id is tipped
     group the kept keys by id, keep the first id, keep keys in candidate order
   ```

   - Running the whole `rewrite_draft` pipeline keeps rule precedence: separator
     toggles, the label attach, and flips all run before `expand_sigil_alias`.
   - The candidate list only proposes substitutions; the forward rewrite decides.
   - Make the tipped vs untipped classification an exhaustive `match` on `AliasId`, with
     a one-line reason for each untipped arm. A new variant then fails to compile until
     someone classifies it.

3. **JSON and human output** (`src/native/capture_rewrite.rs`, `CaptureRewriteResult`).
   - Add `alias` from the stamp, skipped when `None`.
   - Add `sigil_alias_tip` as `Option<Option<TipJson>>` with
     `skip_serializing_if = "Option::is_none"`:
     - `Some(None)` serializes `null`.
     - Set it only when a cursor was given and nothing changed.
   - Serialize `where` only when present.
   - Human output for an unchanged reply with a tip keeps `no rewrite`, then adds one
     dim line, `tip  next time . → @ after a word`.
   - Add a short "Typing tips" paragraph to the `long_about`, and one example to
     `after_help`. Then run `just docs-gen`.

4. **Tests.**
   - **Completeness round-trip.** Route every positive `expand_sigil_alias` assertion in
     `src/native/capture_language/tests/rewrite.rs` through one shared helper. For a
     tipped id, the helper also asserts that `sigil_alias_tip(output, output_cursor)`
     returns that id and key. A new alias fixture cannot skip the reverse check.
   - **Coverage.** Every tipped `AliasId` has at least one positive fixture.
   - **Negative corpus.** None of these emits a tip:
     - `@`, `  @`, `- @`, `- [ ] @`
     - `@cash Buy milk @` and `foo@`
     - backtick and `[[` spans
     - a line-start `#`, `=/`, `[[folder/`
     - a template draft, and a `#"…"` literal
     - `~~` after `~=`, and `- ^^`
     - a standalone `*` or `!`
     - `Buy milk &`, where `.` would give `@`
     - `xs` → `x*` (close is untipped)
     - `x=2~` (past segment is untipped)
     - `@cash+goog-exit +` (next parent is untipped)
   - **Positive rows.** These emit a tip:
     - `Buy milk @`, `- [ ] Buy @`
     - `@errands^`, `@errands:`, a whole-item `:`
     - `@cash^goog-exit@\n- ^` (move)
     - `@cash+`, `Buy milk +`
     - `Buy milk #`, `@errands#`
     - `Fix it ~`
     - `@home &`, `@sase Buy milk &`
     - `Buy milk *`
   - **UTF-8.** `Café run @` returns byte offsets.
   - **CLI tests** in `tests/cli/capture/rewrite.rs` pin:
     - the tip object
     - the explicit `null`
     - absence without a cursor or on `changed: true`
     - the `alias` object on forward aliases
     - the human tip line

5. **Docs.**
   - In `docs/capture.md`, under `bob capture-rewrite`, add one paragraph that covers:
     - the two fields
     - the proof rule
     - the tipped ids and the null capability marker
   - Touch the `docs/reference/json-outputs.md` row only if the budget allows.
   - This is an additive release-train change. Commit as `feat(capture): …` with no `!`,
     and never edit `CHANGELOG.md` or the `Cargo.toml` version
     (`decisions:bob-cli-releases-ride-a-daily-train`).

**Done when** `just check` (or the repo's full test gate) and `just docs-check` pass,
with no new docs failures. Then probe the checkout build with
`target/debug/bob capture-rewrite`: `Buy milk @` must answer the documented JSON.

## Phase `mac-tip-core` — portable model, policy, and ledger

Work in bob-mac-capture's `CaptureCore`. Code against the wire contract above; real
`bob` output is not needed. This phase has no dependencies and can run alongside
`bob-alias-tips`.

1. **Decode.** In `CaptureRewriteResponse` (`Sources/CaptureCore/CaptureModels.swift`,
   near line 3536), decode with `decodeIfPresent`:
   - `sigilAliasTip: CaptureSigilAliasTip?`
   - `reportsSigilAliasTips: Bool`, which is `container.contains(.sigilAliasTip)` and
     true even for `null`
   - `alias: CaptureRewriteAlias?`

   `CaptureSigilAliasTip` holds:
   - `id: String`
   - `sigil`
   - `keys: [String]`
   - `range: CaptureRange`
   - `positionPhrase: String?`, with coding key `"where"`

   Unknown ids decode fine. Extend `CaptureRewriteResponseTests`.

2. **Lessons.** Add `CaptureAliasLesson`: the known tipped ids, each with a host kind.
   - `.completionList`: `route_after_text`, `hash_menu`
   - `.picker`: all the others

   An unknown id maps to `nil` and is ignored everywhere.

3. **Presentation** (pure). `CaptureAliasTipPresentation(tip:)` yields:
   - the lead text `Next time`
   - the key (`keys[0]`), the sigil, and the optional phrase
   - an accessibility sentence built from a glyph-name map
   - the hover text `Typing tips can be turned off in Settings`

   The accessibility sentence reads, for example, "Tip for next time: after a word,
   period types at sign."

   The glyph-name map covers `.`, `;`, `,`, `/`, `\`, `=`, `@`, `#`, `^`, `:`, `+`, `~`,
   `&`, `*`. It is presentation only.

4. **Policy** (pure `AliasTipPolicy`). The thresholds are hypotheses; keep them as named
   constants.
   - **Chances.** A chance is either:
     - a _miss_: a tipped lesson returned for a physically typed sigil, whether or not
       the tip was shown
     - a _hit_: a surviving forward alias with that `alias.id`

     `recent` keeps the last 8 chances, oldest first.

   - **Fluent.** A lesson enters Fluent with at least 4 recorded chances and hits at or
     above 75% of `recent`. It leaves Fluent when hits fall below 50%. On that lapse,
     the gap is set to 4 and `missesSinceTip` to 0.
   - **Due.** A non-fluent miss is due when either holds:
     - fewer than 3 tips have been shown, or
     - not counting itself, at least `gap` misses went untipped since the last shown
       tip.
   - **Gap.** It starts at 2. After each spaced tip (one shown when `tipsShown >= 3`),
     it becomes `min(gap * 2, 8)`.
   - **Dropped tips.** A tip that is dropped (late, surface already open, or session
     limit) counts as a miss, not as shown.
   - **Withdrawing a hit.** This removes that hit from `recent`.
   - **Session** (a pure `AliasTipSession` value). It records whether a tip was already
     shown this panel session and which lessons were escaped this session. An escaped
     lesson is silent for the rest of the session.

5. **Ledger and store.**
   - `AliasTipLedger` is a Codable `{version: 1, lessons, ineligible, weeks}`:
     - `lessons[id]` holds `{recent, tipsShown, missesSinceTip, gap, fluent}`.
     - `ineligible[sigil]` counts physical sigils from a tip-capable bob that returned
       `null`.
     - `weeks["YYYY-Www"][id]` holds `{hits, misses, shown}`, keeping the 12 newest ISO
       weeks.
   - It never stores draft text, target names, or keystroke history.
   - `AliasTipLedgerStore` reads and writes the ledger as a **JSON string** in
     `UserDefaults` under `org.bobs.bob-mac-capture.typing-tips`. Then
     `defaults read org.bobs.bob-mac-capture org.bobs.bob-mac-capture.typing-tips` is
     readable.
   - A corrupt or unknown-version blob resets to empty.
   - `reset()` clears it.
   - Inject the defaults and the clock.

6. **Tests.** Fake-ledger and fake-clock tests in `Tests/CaptureCoreTests` cover:
   - the three learning tips
   - the 2/4/8 spacing
   - fluency entry and the hysteresis exit
   - the lapse restart at 4
   - a dropped tip counting as a miss
   - hit withdrawal
   - one tip per session
   - escaped-lesson silence
   - reset
   - corrupt-blob recovery
   - week rollover and the 12-week cap
   - the accessibility sentence and glyph names

**Done when** `just portable-check` passes. It covers `CaptureCore`; run it on athena if
`swift` is missing. Then run `Scripts/macos-ci-check`, push, and watch the exact-SHA
macOS run to green with `/sase_monitor`, following the repo's `AGENTS.md`.

## Phase `mac-completion-tips` — trigger, tip lane, and the completion-list host

Work in bob-mac-capture. This phase depends on `bob-alias-tips` and `mac-tip-core`. It
ships the flagship `.` → `@` and `/` → `#` tips end to end. The trigger set in this
phase is **`@` and `#` only**.

1. **Typed-sigil intent.**
   - Record a new `TypedSigilIntent {sigil, expectedDraft, expectedCaretUTF8}` from the
     key monitor's native-passthrough branch. That is the same place
     `noteSeparatorToggleIntentFromEvent` runs (`CapturePanelController.swift`, near
     lines 3169 and 3199; the filter is near line 1214).
   - Keep it as a separate struct. The rewrite lane's intents, arms, probes, and holds
     stay untouched.
   - Record the intent only when all of these hold:
     - the character is in the phase's trigger set
     - Shift or Option is held, and Command and Control are not
     - the key is not a repeat
     - the main draft editor has focus, with no modal field
     - there is no marked text and the selection is collapsed
     - typing tips are on
   - In `editorTextDidChange` (`CapturePanelModel.swift`, near line 3080), consume the
     intent only when the new draft equals `expectedDraft` and the caret matches.
   - Paste, picker accepts, Undo and Redo, restores, and the app's own alias output
     never match, so they never count.

2. **Tip lane.**
   - On a consumed intent, call `captureRewrite(draft, cursor:, lane: "alias-tip")`.
   - Add an optional `timeout` parameter to `BobProcessClient.captureRewrite` and pass
     500 ms. The lane is latest-wins.
   - Start it before `scheduleAnalysis`. Never await it on any path, and never let it
     gate completion, preview, Similar Tasks, or Return.
   - Never apply edits from this lane, even when bob answers `changed: true`.
   - On the reply, on the main actor:
     - Drop it if a newer request superseded it, or if the draft prefix through
       `range.end` changed.
     - If `reportsSigilAliasTips` is false, mark this process client as tip-incapable.
       Stop spawning for it until the client identity changes.
     - If the tip is `null`, count `ineligible[sigil]`.
     - Ignore unknown ids.
     - Otherwise record a miss. If the lesson is due and the session allows, keep the
       tip as `pendingAliasTip` (tip plus draft snapshot).
   - The first-frame race is real but lopsided. The lane takes about 5 ms; the list
     needs at least 50 ms of debounce plus `capture-parse` (`scheduleAnalysis`, near
     line 16634, including the cached route path) before it opens.

3. **Admission (first frame or never).**
   - Find the main-actor update where the completion list becomes visible: the cached
     route path, `applyNonActiveTaskCompletion`, and `handleCompletionResponse`.
   - In that same update, if a valid `pendingAliasTip` exists and the list was **not**
     already visible, set `presentedAliasTip`. Then mark the tip shown in the ledger and
     the session.
   - A tip is valid when the draft prefix through `range.end` is unchanged and the sigil
     is still at `range`.
   - Any list open consumes the pending tip.
   - A tip that arrives while the list is already on screen is dropped.
   - Clear `presentedAliasTip` when the list closes, on accept, on submit, and on panel
     hide.
   - Keep it unchanged while the query filters.

4. **Adoption and escapes.**
   - When the main rewrite lane applies an `expand_sigil_alias` reply whose `alias.id`
     is a known lesson, record a hit.
   - When that alias is escaped, withdraw the hit and silence that lesson for the
     session. The escape paths are:
     - Undo of the alias edit
     - the fast second `.` (dot-route escape)
     - the Space/Escape restores
     - a second `\` or `/` (`replaceHashAliasWithLiteralKey`,
       `restoreHashAliasLiteralArm`)
   - Hits are recorded even with tips off.

5. **Settings.**
   - Add `typingTipsEnabled`, default on. Wire it exactly like `similarTasksEnabled`
     through `AppSettings`, `AppDelegate`, and `CapturePanelModel`.
   - Add `Section("Typing Tips")` after Similar Tasks in `SettingsView`, with:
     - `Toggle("Show typing tips")`
     - the caption "A one-line tip when a key without Shift would have typed the same
       sigil."
     - `Button("Reset typing tips")`, which calls the ledger store's `reset()`
   - With tips off: no intents, no spawns, no tips.

6. **The row** (beautiful, shared by both hosts).
   - Add `CaptureAliasTipRow`: one 24 pt line, leading-aligned with the row text (8 pt
     inset):
     - `Next time`: `.caption`, `.secondary`
     - 8 pt gap
     - the keycap
     - `→`: `.caption`, `.tertiary`
     - the sigil: `.caption.monospaced()`, semibold
     - 8 pt gap
     - the phrase: `.caption`, `.secondary`
   - **Shared keycap.** Extract the house keycap from `CapturePickerKeyHints` into one
     `CaptureKeycap` view that both use, so the two can never drift. The keycap is
     `.caption.monospaced()`, padding 6/2, radius 4, a 0.5 pt stroke at 12% primary, and
     a background at 60%. It is neutral, never tinted.
   - **Sigil color.** Color the sigil with `CaptureEditorPalette.color(for:)` of the
     parse span covering `range.start`, the same color the editor just painted that
     glyph. Fall back to `.primary`.
   - **Fitting.** Use `ViewThatFits(in: .horizontal)`: the full line, then the line
     without the phrase, then nothing.
   - Add `.help(...)` from the presentation.
   - The row needs no motion; it is part of the card's first frame.

7. **Completion-list host.**
   - Pin the row inside `CompletionList`'s material card, below the `ScrollView`, under
     a `Divider` hairline (`CapturePanelView.swift`, near line 3394). It never scrolls
     away, and it shows whether or not a row is selected.
   - Add the row height to the list's max height for every menu kind while
     `presentedAliasTip` is set. That covers the generic list, task type, and drawer
     (`completionViewportHeight`, near line 494).
   - Never pin it in the record-card fallback list.
   - Candidate row positions must not move when the tip appears.
   - If the panel height budget binds, drop the tip before any candidate row is clipped.
   - Append the presentation's sentence to the list's accessibility label once. Never
     route the tip through `announceStatus`, and never re-announce it while filtering.

8. **Tests.** Model tests use a stub process client, in the style of
   `CaptureSeparatorToggleTests`. Cover:
   - The tip is in the list's first frame.
   - A held tip reply never blocks the list. The list opens without a tip, and the late
     reply is dropped and counted as a miss.
   - An edit before the reply drops the tip.
   - Unshifted, repeat, paste, accept, Undo, and marked-text sigils spawn nothing.
   - Tips off spawns nothing.
   - The second eligible sigil in a session shows no tip.
   - An old bob (key absent) disables the lane.
   - A hit is recorded, and each escape path withdraws it.
   - The draft, Undo history, and capture result are identical with tips on and off.

   Add `CaptureAliasTipDesignTests`, rendering the route list with a tip at 760 and 620
   pt, in light and dark, like `SimilarTasksBandDesignTests`.

9. **Docs.**
   - Create `docs/typing-tips.md`, a reference page, and add it to `docs/nav.yml` after
     Keyboard. It covers:
     - what a tip is, and where and when it appears
     - the lesson table, with picker lessons marked as coming
     - the fading policy
     - Settings and reset
     - reading the ledger with `defaults read`
     - the compatibility table
   - Add the Typing Tips section to `settings.md`.
   - Respect the docs budget rules above.

**Done when:**

- `just portable-check` passes.
- The scratch-copy `bundle.sh` and `just typecheck-tests` pass on the mac (best effort,
  per `AGENTS.md`). If the mac is offline, say the macOS targets are unverified.
- `Scripts/macos-ci-check` passes, and the exact-SHA macOS run is watched to green.
- `just docs-check` passes in both repos, with no new failures.

## Phase `mac-picker-tips` — the picker host for the remaining lessons

Work in bob-mac-capture (memory edits, if accepted, go in bob-cli). This phase depends
on `mac-completion-tips`.

1. **Trigger set.** Widen it to `@ # ^ : + ~ & *`.
   - For `^ : + ~`, the existing rewrite-lane intent is still recorded too. The two stay
     independent.
   - Keys the key router consumes, such as a `~` relation flip, never reach the
     passthrough branch, so they never count.

2. **Picker admission.** Apply the same first-frame admission at the main-actor update
   where `model.picker` opens from a completion need. A tip must not jump from a list to
   a picker or the other way: the first surface that opens consumes the pending tip.

> [!decision] picker_tip_slot = card_foot Add `CaptureAliasTipRow` as the last line of
> the picker card. It goes under a `Divider`, below `CapturePickerDetailStrip`
> (`CapturePickerView.swift`, near line 237).
>
> - Give `CapturePickerHeightPolicy` a tip row height: 0 without a tip, 24 pt with one.
>   Fold it into `idealHeight`.
> - Admit the tip only if the card's ideal height including the tip fits the panel
>   budget at open. Otherwise drop it before shrinking the row budget. Decide once at
>   open; filtering never resizes the card.

> [!decision] picker_tip_slot = key_legend Render `CaptureAliasTipRow` as the last line
> of `CapturePickerKeyHints`.
>
> - Place it below the key row and any `operatorContinuationHint`, through the existing
>   measured-footer path.
> - The picker card's height policy is unchanged.
> - Keep the `Next time` lead so the tip never reads as a live key.

3. **Accessibility.** Append the tip sentence once to:
   - the picker card's accessibility label
   - its one-time open announcement (`appearedAnnouncement`)

   Never re-announce it while filtering.

4. **Adoption and escapes.** Record hits for the picker lessons' forward aliases.
   Withdraw on escape:
   - Undo, for every lesson
   - the fast `==` and `restoreRelationEqualsOpener`, for `relation`
   - whatever arms `bob-cli-j6.2` landed for `&` and `*`

5. **Tests.**
   - Model tests for each picker lesson's first-frame admission and late drop.
   - One test proving a list-then-picker sequence shows at most one tip.
   - Design tests for the picker with a tip at 760 and 620 pt, in light and dark, plus a
     short-screen fixture where the tip yields before rows.

6. **Docs.** Update `docs/typing-tips.md`: picker lessons are live, and list the picker
   slot. Keep within the budget rules.

7. **Memory**, only as the decisions below allow. Each accepted change goes in bob-cli
   `sase/memory/` through `/sase_memory_write`.

> [!decision] memory_capture_alias_glossary Add one sentence to
> `sase/memory/glossary/capture-alias.md`, then run `sase memory init`. The sentence
> says that `bob capture-rewrite --cursor` also returns `sigil_alias_tip`, proved by
> running the forward rewrite, and that Bob Mac Capture shows it as a fading `Next time`
> row in the list or picker the sigil opened.

> [!decision] memory_typing_tips_decision Write the decisions-web strand
> `typing-tips-are-bob-proved` ("Typing Tips Are Bob-Proved And Fade By Fluency"), then
> run `sase memory init`. The claim is the Principles section above. Rejected
> alternatives come from the research's "Alternatives considered". Cite this plan and
> the research report, and link `[[decisions:mac-capture-is-a-thin-client]]`.

8. **Stop-rule follow-up.** Record a `PROPOSED FOLLOW-UP:` on this phase's bead. About
   three weeks after picker tips ship, read the ledger's `weeks` counts:
   - Keep the feature if alias share rose for tipped lessons.
   - Delete it if share did not rise.
   - Stop immediately on any false tip.

**Done when** the same gates as `mac-completion-tips` pass, including the macOS run
watched to green on the exact SHA.

## Out of scope

- `x`/`s` close-outcome tips in the Close card
- `next_parent` and `past_segment` tips
- a Settings list of learned lessons
- TipKit
- any grammar change, CLI option, or subcommand
