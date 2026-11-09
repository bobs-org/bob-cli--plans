---
tier: epic
title: Close-outcome letter aliases (x → !, s → *) in Bob Mac Capture
goal: 'In the Bob Mac Capture editor, a plain lowercase `x` typed where a close selection
  accepts an outcome marker becomes `!`, and a plain lowercase `s` becomes `*`. Examples:
  `xx` → `x!`, `xs` → `x*`, `x1s` → `x1*`, `x=1=s` → `x=1=*`, and `@r:id=xx` → `@r:id=x!`.
  Bob''s `capture-rewrite` alone decides where the alias fires. `bob capture`, `capture-parse`,
  paste, and CLI text keep the letters. Typing speed never changes the outcome: keys
  typed before Bob answers are held and replayed, and a fast Return never files the
  literal draft.

  '
phases:
- id: bob-close-alias
  title: Bob capture-rewrite close-outcome alias
  depends_on: []
  size: medium
  description: 'bob-close-alias: add an x/s branch to the expand_sigil_alias rule
    in bob-cli capture-rewrite. It locates a close slot on the draft before the key,
    then checks the swapped selection with the close lexer. Ships with unit and CLI
    tests, capture-rewrite help, and capture docs as a non-breaking feat(capture).'
- id: mac-type-ahead-hold
  title: Mac type-ahead hold for meaning-changing rewrite aliases
  depends_on: []
  size: medium
  description: 'mac-type-ahead-hold: generalize the past-segment dash Return deferral
    into a meaning-changing-alias flag. Add a bounded type-ahead hold that buffers
    key-downs while such a rewrite is in flight and replays them after Bob answers.
    Its first client is the existing x= dash alias. Mac only.'
- id: mac-close-alias
  title: Mac forwards x/s to Bob's close-outcome alias
  depends_on:
  - bob-close-alias
  - mac-type-ahead-hold
  size: medium
  description: 'mac-close-alias: forward lowercase non-repeat x/s keys through a lexical
    send prefilter into the rewrite lane, mark them meaning-changing so the hold and
    Return deferral cover them, and add a one-time old-bob probe. Adds recorded fixtures,
    tests, Mac docs, the optional decision record, and an end-to-end check.'
decisions:
  memory_close_alias_record:
    ask: Add a new decisions strand, "Close Outcomes Have Letter Aliases In The Editor",
      once the feature lands?
    memory:
    - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.3r.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 11:42:37
status: done
bead_id: bob-cli-is
---

- **PROMPT:** [prompts/202611/close_outcome_x_s_typing_aliases.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/close_outcome_x_s_typing_aliases.md)
- **BEAD:** [bob-cli-is](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-is/README.md)

# Plan: Close-outcome letter aliases (`x` → `!`, `s` → `*`)

## Context

Bryan wants two new Shift-free typing aliases in Bob Mac Capture, matching the existing
`,`→`+`, `.`→`^`, `;`→`:`, `=`→`~`, `\`/`/`→`#`, and past-segment `-`→`~` family:

- `x` types `!` (complete) and `s` types `*` (park) inside the `x` close syntax. For
  example, `xx` → `x!`.
- The same letters work after the second `=` of the `x=` past syntax. For example,
  `x=1=s` → `x=1=*`.

The consolidated research report
`research:202611/close_outcome_x_s_typing_aliases/close_outcome_x_s_typing_aliases.md`
designs the feature. Bryan accepted **every** requirement it recommends, including
A1–A10 (with A10, link-close suffixes, set to yes). He also accepted its answers to the
open questions: no one-time hint in v1, and no close-drop (`-` → `~`) letter alias. Each
phase worker should read the report with `sase artifact read` before starting, focusing
on "Verified grammar facts", "Requirement adjustments", and "Recommended solution". This
plan restates the binding contract, so workers do not need to re-derive it.

Governing records (read with `sase memory read decisions:<key> -r "<why>"`):

- `decisions:mac-capture-is-a-thin-client`. Bob alone decides where the alias fires. The
  Mac only forwards keys, holds and replays them, and applies Bob's one native edit. The
  Mac prefilter is a send filter that decides only when **not** to ask Bob; it never
  decides syntax.
- `decisions:relation-equals-alias-and-flips`. Timing-based or editor-only syntax never
  reaches the grammar: `bob capture` and `capture-parse` stay literal.
- `decisions:bob-cli-releases-ride-a-daily-train`. This change is non-breaking:
  `feat(capture): …` with no `!`. Never edit the `Cargo.toml` version or `CHANGELOG.md`.

### Binding behavior contract

"Fires" means `bob capture-rewrite --cursor <end> -- <draft>` returns rule
`expand_sigil_alias`, one one-byte edit, the cursor unchanged, and the summary
`Typed ! for x` / `Typed * for s`. The cursor sits at the end of the draft unless noted.

| Draft typed (last byte is the trigger)     | Result                        | Note                                                      |
| ------------------------------------------ | ----------------------------- | --------------------------------------------------------- |
| `xx` / `xs`                                | `x!` / `x*`                   | Whole-item close                                          |
| `x1x` / `x1s`                              | `x1!` / `x1*`                 | Every outcome-marker position (A2)                        |
| `x!s` / `x*x`                              | `x!*` / `x*!`                 | Competing wildcards are a viable prefix (`x!*1` valid)    |
| `x*1x`, `x@1s`, `x1~2s`                    | `x*1!`, `x@1*`, `x1~2*`       |                                                           |
| `x=1=s` / `x=1=x` / `x==s`                 | `x=1=*` / `x=1=!` / `x==*`    | After the past token's second `=` (A1)                    |
| `x=fix=s`, `x=1!=x`, `x=1=1s`              | `x=fix=*`, `x=1!=!`, `x=1=1*` |                                                           |
| `+2 xs`                                    | `+2 x*`                       | Lead chain close                                          |
| `@r:id=xx`, `^r:id=x1s`, `Fix it @r:id=xs` | `=x!`, `=x1*`, `=x*` suffixes | Link-close suffixes (A10)                                 |
| `Plan\n\nxx`                               | `Plan\n\nx!`                  | A later batch item                                        |
| `x=1s`, `x=s`, `x=xs`, `x=1x`              | unchanged                     | Past `E` names, counts, and the teaching error (A4)       |
| `x!x`, `x*s`, `x1,s`, `x@s`                | unchanged                     | Duplicate group; dangling `,` / `@` (A3)                  |
| `Xx`, `Xs`, `@r:id=Xs`, `xS`, `xX`         | unchanged                     | Lowercase trigger **and** lowercase opener (A5)           |
| `x log xs`, `x\n- xs`                      | unchanged                     | Chain close after inline Work Log text; child bullet (A4) |
| `=xs`, `=x` + `s`                          | unchanged                     | Whole-item named start; retired `=x` (A4)                 |
| `Plan xs`, `tax`, `session`, `s:2`         | unchanged                     | Prose                                                     |
| `x1s2` with the caret after `s`            | unchanged                     | The token must end at the cursor                          |
| templates, wikilinks, code spans           | unchanged                     | Existing guards                                           |
| any draft without `--cursor`               | unchanged                     | Paste, CLI, and restored drafts stay literal (A6)         |

On the Mac (A5–A9):

- Only plain lowercase `x`/`s` key-downs are forwarded: no ⌘/⌃/⌥, not a key repeat, no
  marked text, a collapsed selection, the main editor, and no modal field. `X`, `S`, ⌘X,
  and ⌃S never alias.
- Undo restores the literal key in one step. Capital `X…` is the prose escape. There is
  no double-tap restore and no one-time hint.
- Typing speed never changes the outcome (A7):
  - `xx⏎`, `xs1⏎`, `x=1=s3`, and `x1s2x3` typed in one burst give the same draft and
    submit as typing slowly.
  - If Bob does not answer in time, the letter stays literal and Return is dropped with
    "Bob did not answer; press Return again". The app never silently submits the literal
    draft.
- With an old `bob` that returns `changed: false` for `xx`, the Mac stops forwarding
  `x`/`s` and says once "This bob predates x/s close aliases · update bob". `xx⏎` then
  files the literal task, as it does today.

### Phase graph

```text
bob-close-alias ──────────┐
                          ├──► mac-close-alias
mac-type-ahead-hold ──────┘
```

`bob-close-alias` (bob-cli) and `mac-type-ahead-hold` (bob-mac-capture) touch different
repositories and run in parallel. `mac-close-alias` needs Bob's rule for recorded
fixtures and the hold for safety, so it runs last. The hold lands before `x`/`s`
forwarding, so the Mac never ships an `x`/`s` alias that fast typing can drop.

### Repositories and verification (every phase)

- **bob-cli** (this repo):
  - Run `just check`.
  - For help and doc changes, first read the `cli_rules.md` and `docs.md` reference
    memory with `/sase_memory_read`, then run `just docs-gen` and `just docs-check`.
  - `docs/reference/mac-capture/` is mounted from bob-mac-capture. Never edit it here.
- **bob-mac-capture** (linked repo): open it with `/sase_repo` and read its `AGENTS.md`.
  Never land Swift you have not compiled.
  - Run `just portable-check` on a host with `swift` (athena when this host lacks it).
  - Compile a scratch copy on the mac: rsync without `.build`/`.git`, run `bundle.sh`
    and `just typecheck-tests`, then delete the copy. Read the `tailnet.md` reference
    memory for access. Never use Bryan's checkout or `just install`.
  - Run `Scripts/macos-ci-check` before landing. After every push, watch the exact-SHA
    macOS CI run to green with `/sase_monitor`.
  - If the mac is offline, say in the final report that the macOS targets are
    unverified.
  - Format only the lines you change (`.swift-format`). Docs changes need
    `just docs-check`.
  - Mac tests use fake-bob replies, delayed replies, injected `NSEvent` timestamps, and
    an injected deadline clock. **Never use wall-clock sleeps.** Async tests must never
    await an untimed continuation or gate.

## Phase `bob-close-alias`: Bob capture-rewrite close-outcome alias

Main file: `src/native/capture_language/rewrite.rs`. Also touched:
`src/native/capture_language/close_selection.rs`. The rule code stays
`expand_sigil_alias`. The JSON `schema_version` stays 1, the capture grammar version
stays 19, and no response field is added. Feeding any output back with the same cursor
is a no-op.

1. **Dispatch.** In `expand_sigil_alias`, before tokenizing, route `alias == "x"` /
   `alias == "s"` to a new `expand_close_outcome_alias(raw_text, start, cursor)`. Put it
   next to `expand_past_segment_dash_alias`. Only lowercase bytes trigger. Update the
   doc comments on `expand_sigil_alias` and `rewrite_draft` to name the new aliases.
2. **Locate on the pre-key draft.** Remove the trigger byte from the draft. Find a close
   selection whose text ends exactly at the trigger offset, on the first physical line
   of its item (use `split_capture_draft`, as the `.` choice alias does). The token
   holding the trigger must end at the cursor in the original draft. Reuse the existing
   classifiers; write no new grammar scans. There are three slots:
   - **Whole-item or lead-chain close.** Run `classify_whole_item_close` on the token.
     The token must be the line's first token, or every token before it must pass
     `is_session_chain_token`. Use the same token view as `session_chain_tokens` (masked
     tokens plus merged past literals). This admits `+2 xs` and excludes chain closes
     after inline Work Log text (`x log xs`), per `split_inline_close_tokens`.
   - **Past close part.** The token passes `past_token_len` (lowercase `x=` only), and
     `split_past_token` reports `has_second_eq`. The trigger offset equals the token end
     and falls in `close_text`. Past tokens follow the same first-token-or-lead rule. A
     trigger in `E` (names, counts, repair `!`) never aliases.
   - **Link close (A10).** The token is a Pomodoro link marker (`@route:id=…`,
     `^route:id=…`, including mid-line after task text). Confirm this with the existing
     link-marker parsers in `tokens.rs` (for example `parse_pomodoro_route_token`,
     `parse_caret_link_token`, or `classify_caret_token`), never by string slicing
     alone. Its suffix after `=` must satisfy `close_after_eq_parts` with `x_len == 1`
     and a **lowercase** `x` opener. An empty suffix (the `x` is the opener) and every
     uppercase `X…` opener never alias.
   - **Excluded everywhere.** Templates (the existing `rewrite_draft` early return),
     `wikilink_claims` on the candidate, code spans (odd backtick parity before the
     trigger, as in `relation_alias_position_ok`), child and continuation lines, and
     whole-item `=x…` retired spellings. Always locate on the pre-key draft, never on
     the candidate: `=` + `x` gives candidate `=!`, which `close_after_eq_parts` would
     claim as a retired close.
3. **Verify the candidate (A3).** Append `!` (for `x`) or `*` (for `s`) to the located
   selection body and run `lex_close_selection` on it.
   - Accept `Valid` and `Incomplete`.
   - Also accept an error whose **only** cause is competing park-all and complete-all
     wildcards (`x!*`, `x*!`), because adding a number makes it valid.
   - Add a small predicate in `close_selection.rs` (for example
     `close_marker_may_follow`) and give `CloseSelectionError` a structured kind (for
     example a `CloseSelectionErrorKind::CompetingWildcards` variant). Never match on
     message text.
   - Reject every other error (a second `!`/`*` group, a marker after a dangling `,` or
     `@`, and so on).
4. **Edit.** Call
   `finish_sigil_alias(raw_text, &candidate, start, cursor, '!' | '*', "x" | "s")`. That
   gives one one-byte edit, `cursor: Some(cursor)`, and the summary `Typed ! for x` /
   `Typed * for s`.
5. **Unit tests** in `src/native/capture_language/tests/rewrite.rs`:
   - every row of the binding contract;
   - UTF-8 offsets with an emoji earlier in the draft;
   - a CRLF draft;
   - idempotence (the result fed back with the same cursor is unchanged);
   - the predicate's kinds in `close_selection.rs`'s own test module.

   For every "Fires" row, also assert that `capture-parse` classifies the result as a
   close, link close, or past close, with no error other than the competing-wildcards
   diagnostic.

6. **CLI tests** in `tests/cli/capture/rewrite.rs`: JSON-shape cases for a whole-item,
   past, link-suffix, and lead-chain alias. Add a no-`--cursor` case that stays
   unchanged, and a `bob capture-parse`/`bob capture --dry-run` case showing `xx` is
   still prose. Keep the existing assertion in the bare-close tests that `xx` parses as
   prose.
7. **Help and docs** (read `cli_rules.md` and `docs.md` first):
   - `src/native/capture_rewrite.rs`: extend the "Sigil alias (requires --cursor)" help
     paragraph with the `x`/`s` close-outcome alias and one or two examples. Then run
     `just docs-gen` to regenerate `docs/reference/cli/capture-rewrite.md`.
   - `docs/capture.md`: add a close-outcome paragraph beside the past-segment `-`
     paragraph under **Sigil alias**. Cover the three slots, the viable-prefix rule,
     lowercase-only, the inline-entry and child-bullet exclusions, and "editor assist
     only". Then grep for statements that `xx` is prose or never closes, and that
     `=xx`/`=xs` are named starts. Scope each one: "grammar and CLI: prose / named
     start; in Mac Capture's editor, typing `xx` gives `x!`". The research cites L337,
     L3579, L4655, and L4682 (line numbers may drift).
   - `docs/reference/capture-syntax.md`: the "`xx`, `x-ray`, and capital `X` never
     close" sentence (near L1006) and the overloaded-marker rows get the same scoping.
   - Run `just docs-check`.
8. **Commit** as `feat(capture): alias x and s to close outcomes while typing` (no `!`).
   Make no version or changelog edits.

## Phase `mac-type-ahead-hold`: type-ahead hold for meaning-changing aliases

Mac only, in bob-mac-capture. Main files: `CapturePanelModel.swift`
(`startCaptureRewrite`, `applyCaptureRewrite`, `submit`, `waitForRewriteBeforeSubmit`,
`editorTextDidChange`, `invalidateRewrite`) and `CapturePanelController.swift`
(`routeKeyDown`, the local key monitor, dismissal and focus paths). Precedent for a
capped key buffer: `RefsPanelModel.pendingHighlightsCopyBuffer`.

Today any keystroke that lands before Bob answers cancels the rewrite through
`editorTextDidChange` → `invalidateRewrite()`. Only a `-` in an `x=` draft defers
Return. This phase makes typing speed irrelevant for every **meaning-changing alias**:
an alias whose literal draft is valid input with a different meaning. Its first client
is the existing past-segment `-`, which already has Return deferral. The
`mac-close-alias` phase adds `x`/`s`.

1. **Meaning-changing flag.**
   - Replace `deferSubmitForPastSegmentDash` with one helper that decides whether a
     forwarded trigger is meaning-changing. Today it returns true exactly for `-` when
     the expected draft contains `x=`, so behavior is unchanged. Give it a neutral name,
     for example `deferSubmitForMeaningChangingAlias`.
   - Generalize the echo check in `applyCaptureRewrite` (today: replacement `~` for
     trigger `-`) into one trigger→replacement table, so the next phase only adds
     `x`→`!` and `s`→`*`.
   - A transport failure cancels the queued submit and leaves the literal editable,
     never submitting it silently. When a Return was queued, show "Bob did not answer;
     press Return again".
2. **Hold state machine.** Put the queue and state in a small type that can be unit
   tested without AppKit (for example `CaptureTypeAheadHold`). The controller does only
   event plumbing.
   - **Engage** when `startCaptureRewrite` starts a meaning-changing rewrite. Record its
     rewrite generation and start time.
   - **End** when that generation's reply is applied (`changed` or not), fails, is
     cancelled, or misses the deadline.
   - **Never engage** with marked text, an active modal field, or a first responder
     other than the main draft editor.
3. **Interception.** At the top of `CapturePanelController.routeKeyDown`, before any arm
   is taken (`takeEditorLeader`, `takeBackslashLiteralArm`, `takeHashAliasLiteralArm`,
   the equals-gesture checks), check the hold. When it is engaged:
   - **Buffer** (return `nil`) key-downs that would edit the main draft or submit it:
     text-producing keys without ⌘/⌃ (Shift allowed), Space, Backspace / forward delete,
     and Return/Enter, including the ⌘Return submit variant.
   - **Flush first** for everything else: ⌘/⌃ commands (⌘Z, ⌘V, …), Escape, Tab, arrows,
     and function keys. Then route the key normally.
   - **Flush first** as well on a mouse-down in the panel, a first-responder change away
     from the editor, the panel resigning key, dismissal, and process-client
     replacement.
4. **Replay.** When the hold ends with Bob's verdict, applied or unchanged:
   - Replay the queue in typed order through the same routing as a live key, so each
     replayed key is routed exactly once. Intent recording, every alias and arm, the
     close-list comma assist, and Return then behave as if typed after the reply.
   - Prove "exactly once" with a test: a replayed `=` or `\` must not be double-routed.
   - If a replayed key engages a new hold (a second meaning-changing alias in the
     burst), stop replaying. The remaining keys stay queued ahead of any later live key.
   - Live keys that arrive during replay join the queue's tail.
   - Replayed events keep their original `NSEvent.timestamp`.
5. **Flush, deadline, and cap.**
   - **Flushing a non-empty buffer** cancels the rewrite, so the letter stays literal.
     It then replays the text and edit keys and drops every buffered submit with the
     status line above.
   - **Flushing an empty buffer** leaves the rewrite running. Submits from buttons or
     menus still reach the model's Return deferral, which waits for Bob.
   - **Deadline.** Use a named constant of about 150 ms from engagement, tuned from the
     `CaptureSignpost.measure("rewrite")` data, with an injectable clock for tests. When
     it passes, treat it like a flush of a non-empty buffer.
   - **Cap.** Hold about 32 keys. One more key flushes.
   - **Failure reply.** Treat it like the deadline.
   - **Signposts.** Emit a signpost event with the hold's duration and key count, so the
     deadline can be tuned.
6. **Tests** (a new `CaptureTypeAheadHoldTests`, plus the existing rewrite-lane suites):
   - pure state-machine tests;
   - with a delayed fake Bob: `x=2-3` typed before the reply gives `x=2~3`; `x=2-3⏎`
     submits `x=2~3` exactly once; ⌘⏎ behaves the same; repeated Return submits once;
   - a no-change reply replays the literal;
   - a failure reply and a missed deadline keep the literal, drop Return, and show the
     status line;
   - ⌘Z, a click, a focus change, and dismissal flush first;
   - cap overflow;
   - marked text and modal fields are never held;
   - replayed keys are routed exactly once.

   Every existing rewrite-lane test must still pass unchanged.

7. **Docs** (`just docs-check`). In `docs/capture-panel.md`, near the rewrite-lane
   paragraph:
   - keys typed while Bob answers a meaning-changing alias are held and replayed;
   - Return waits for Bob;
   - when Bob does not answer in time, the typed character stays and Return must be
     pressed again.
8. **Commit** as `feat(mac-capture): hold type-ahead behind meaning-changing aliases`.

## Phase `mac-close-alias`: Mac forwards `x`/`s` to Bob's close-outcome alias

Mac work in bob-mac-capture. Bob's rule from `bob-close-alias` must be on master, and
the hold from `mac-type-ahead-hold` must have landed. Build `bob` from bob-cli master to
record fixtures.

1. **Forwarding.**
   - In `CapturePanelController.noteSeparatorToggleIntentIfEligible`, accept
     `characters == "x"` or `"s"` exactly, under the existing guards (no ⌘/⌃/⌥, no
     marked text, collapsed selection, main editor, no modal field), and add
     `!event.isARepeat`.
   - Add both letters to the allowlist in `CapturePanelModel.noteSeparatorToggleIntent`
     and to its doc comments.
   - Leader keys and arms that consume `x`/`s` keep priority. Forward only where the key
     would insert natively.
2. **Prefilter (A9).** A lexical **send filter**, so prose letters never spawn `bob`.
   Forward only when both hold:
   - the character before the caret is one of `x`, `0`–`9`, `=`, `!`, `*`, `~`;
   - the current line before the caret matches `(^|[ \t=])x`.

   This is a strict superset of Bob's slots. It decides only when not to ask, like
   today's `=` filter, and Bob still decides every alias. Pin the superset property with
   a Swift table test that forwards every "Fires" row of the binding contract.

3. **Meaning-changing.**
   - Mark forwarded `x`/`s` as meaning-changing in the helper from
     `mac-type-ahead-hold`. The type-ahead hold and Return deferral then cover them.
   - Add `x`→`!` and `s`→`*` to the echo table.
   - Apply replies through the existing `applyNativeSingleEditRewrite` path, so Undo
     restores the typed key in one step.
   - Arm nothing: there is no double-tap restore.
4. **Old-bob probe.**
   - When a process client resolves (each new `identityToken`), run
     `captureRewrite("xx", cursor: 2)` once. Cache the verdict by client identity:
     supported only when the reply is `changed` with text `x!`.
   - Until a supported verdict arrives, do not forward `x`/`s`. The letters stay
     literal.
   - On an unsupported verdict, the first time an `x`/`s` key would have been forwarded,
     show "This bob predates x/s close aliases · update bob" once per client identity.
   - On a transport failure, retry on the next resolution or Recheck Bob.
5. **Comma-assist parity.** After `xs` → `x*`, typing `1` and `2` must behave exactly as
   after a typed `*` (the `CaptureCloseTaskCommaAssist` and `closeTaskCommaArmed`
   paths). This includes when the digits were held and replayed.
6. **Fixtures.** Record `capture-rewrite-sigil-alias-close-*.json` fixtures (whole-item,
   past, link suffix, unchanged prose, and an old-bob `changed: false`) from the built
   `bob`. Extend an existing `Scripts/generate-*-fixtures.sh` or add one, following the
   current fixture generators.
7. **Tests** (`CaptureSeparatorToggleTests` and the model and hold suites):
   - forwarding, the prefilter table, repeats, modifiers, uppercase, and IME;
   - with a delayed fake Bob: `xx⏎`, `xs1⏎`, `x=1=s3`, `x1s2x3`, ⌘⏎, repeated Return,
     failure and no-op replies, Undo/Redo, caret moves, client replacement, and
     dismissal;
   - an emoji before a later item, for UTF-8 to UTF-16 conversion;
   - comma-assist parity;
   - the old-bob probe: no forwarding, the one-time status, and `xx⏎` filing the literal
     draft.
8. **Docs** (`just docs-check`):
   - `docs/keyboard.md`: editor rows for `x` and `s`.
   - `docs/capture-panel.md`: the rewrite-lane paragraph, plus notes that typing `xx`
     gives `x!` and that a named start beginning `xx`/`xs` must be typed uppercase
     (`=XS…`) or as `=#xs…`.
   - `docs/protocol.md`: a compatibility row covering new Mac with old bob (letters stay
     literal, plus the status line) and old Mac with new bob (letters never enter the
     lane).
9. **Commit** as `feat(mac-capture): alias x and s to close outcomes`.
10. **Decision record.**

    > [!decision] memory_close_alias_record

    Only when this decision is accepted, use `/sase_memory_write` and add a `decisions`
    strand titled "Close Outcomes Have Letter Aliases In The Editor" (keyword for
    example `close-outcome-letter-aliases`).
    - **Applies to:** bob-cli and Bob Mac Capture.
    - **Claim:** A1–A10 in one sentence each, at most.
    - **Why:** the research consolidation and its alternatives table. The rejected
      alternatives are a Swift-side replacement, a grammar-level alias, a next-key hint
      protocol, chords, `p` for park, the literal "instead of `=`" reading, and the
      double-tap escape.
    - **Cost:** whole-item `xx`/`xs` and `=xx`/`=xs` link suffixes are claimed while
      typing; one spawn per prefiltered letter; the hold's deadline.
    - **Reopens when:** a stolen prose or name parse actually bites, or Mac rewrite
      latency makes the hold noticeable.
    - **Evidence:** the research ref plus the landed commits.

    Run `sase memory init`. When the decision is declined, record a
    `PROPOSED FOLLOW-UP:` note on this phase's bead only if no human reviewed the plan.

11. **End-to-end check.**
    - Run every Bob row of the binding contract against the bob-cli master build with
      `bob capture-rewrite --cursor <N> --format json -- <draft>`.
    - Confirm the exact-SHA macOS CI runs for both Mac phases are green.
    - Report any row that could only be checked by reading code, and any Mac target left
      unverified.
