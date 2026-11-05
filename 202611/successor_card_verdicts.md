---
tier: epic
title: Successor card verdicts — mark offers Done or Cancel from the card
goal: 'While reviewing a successor card in Obsidian, in Bob Mac Capture, or through
  `bob capture`, Bryan can stage Done (Ctrl+X) or Cancel (Ctrl+Shift+X, with an optional
  inline reason) on any eligible offer row. Those verdicts apply together with pull-ins
  in the gesture''s existing write pass. The engine reports eligibility and consequences,
  closes are root-only, cascades are recover-only, failures are reported honestly,
  and the card uses Bob''s existing ✓/⊘ visual grammar.

  '
decisions:
  memory_verdict_record:
    ask: Add decision record successor-cards-resolve-with-staged-verdicts to the decisions
      web?
    memory:
    - decisions:successor-cards-resolve-with-staged-verdicts
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
  memory_offers_supersede:
    ask: Mark decisions:unblocked-tasks-are-offered superseded in part by the new
      record?
    memory:
    - decisions:unblocked-tasks-are-offered
    default: true
    corrects: Unpicked rows recover without a link and resurface through the existing
      freshness rules.
    answer: true
  memory_glossary:
    ask: Teach the successor-offer glossary strand that card rows can also be marked
      Done or Cancel?
    memory:
    - glossary:successor-offer
    default: false
    answer: false
phases:
- id: contract
  title: Contract, vectors table, and decision record
  depends_on: []
  size: medium
  description: 'contract: write the binding bob-cli docs contract in docs/task-dependencies.md
    §12.22–12.23 and the §12.5/§12.13/§12.14 additions (the SV vector table, canonical
    copy, CLI/JSON/API shapes), plus the memory changes the accepted decisions authorize.'
- id: engine
  title: Rust offer-close engine and per-row close descriptors
  depends_on:
  - contract
  size: medium
  description: 'engine: build the pure Rust offer-close engine (eligibility, root-only
    Done, Task-Card-parity Cancel, recover-only cascade, consequence copy), attach
    a `close` descriptor to every capture offer row, and publish tests/fixtures/offer_close/vectors.json.'
- id: cli
  title: bob capture -x/-X flags, batch post-pass, and receipts
  depends_on:
  - engine
  size: medium
  description: 'cli: add `-x/--complete-offer` and `-X/--cancel-offer`, resolve and
    validate verdicts, apply them in a capture post-pass, and emit offer.verdict,
    pull_in.close, offer_closes[], refusals, human output, help, completion, and docs.'
- id: nav-card
  title: Obsidian successor picker verdict UI (api.successors v4)
  depends_on:
  - contract
  size: medium
  description: 'nav-card: add per-row verdict state, keys, the inline cancel reason,
    the visuals, footer, two-step Esc, and hover buttons to SuccessorOfferPickerModal,
    and bump api.successors to v4 with the {picks, closes} result.'
- id: nav-apis
  title: Nav cancel API v1 and Unblocked notice v4
  depends_on:
  - nav-card
  size: small
  description: 'nav-apis: expose a pure Task-Card-parity cancel planner as api.cancel
    v1, and extend api.notice to v4 with offer-close rows, refusal lines, tickler
    lines, and a guarded Undo action.'
- id: cycler
  title: Cycler eligibility, verdict apply step, receipts, and undo
  depends_on:
  - engine
  - nav-apis
  size: large
  description: 'cycler: compute close descriptors that match the shared vectors, attach
    them when nav v4 is present, apply returned closes in a guarded verdict step with
    recover-only cascades, send receipts with Undo restores, and pin the walk and
    reopen rules.'
- id: promotion
  title: Verdicts on the Obsidian project-promotion card
  depends_on:
  - cycler
  size: small
  description: 'promotion: expose the cycler''s describe/apply offer-close functions
    as cycler api v5 and wire them into nav''s project-promotion successor card. The
    task-merge card stays pull-in-only.'
- id: mac-core
  title: CaptureCore decoding, outcome model, argv, and fixtures
  depends_on:
  - cli
  size: medium
  description: 'mac-core: decode the new bob fields and refusals, add the portable
    CapturePullInOutcomes model, gate `--complete-offer` and `--cancel-offer` argv
    on pull_in.close, and generate fixtures from a real bob.'
- id: mac-card
  title: Mac Pull-In card verdict UI
  depends_on:
  - mac-core
  size: large
  description: 'mac-card: wire the outcome model into the Pull-In card (⌃X/⌃⇧X routing,
    inline reason field, glyphs, capsule, consequence strip, labels, two-step Esc,
    reopen on refusal with choices intact, VoiceOver actions), then update docs and
    gate on the exact-SHA macOS CI run.'
proposed_by: bbugyi200.athena.191
proposed_with: claude/opus
decided_by: reviewer
decided_via: cli
create_time: 2026-11-05 18:41:45
status: wip
bead_id: bob-cli-hn
---

- **BEAD:** [bob-cli-hn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hn/README.md)

# Plan: Successor card verdicts — mark offers Done or Cancel from the card

## Context

Bryan wants to close or cancel tasks directly from the successor card. That is the
Obsidian `SuccessorOfferPickerModal` and Bob Mac Capture's Pull-In card. It is the
moment of maximum context: successors that are already finished or now obsolete get
resolved then, instead of costing a review touch later as a TICKLER or ROTTEN row.

**Design source.** The consolidated research report
`research:202611/successor_offer_verdicts/successor_offer_verdicts.md`. Read it with
`sase artifact read research:202611/successor_offer_verdicts/successor_offer_verdicts.md "<reason>"`.

- **Bryan accepted every requirement in it (A1–A14).** Where this plan is more specific
  than the report, this plan is binding.
- **I resolved the report's open questions as follows:**
  - Cascades recover and come back as a tickler in v1.
  - Every cancel opens the optional reason field.
  - Muted rows are not eligible.
  - Obsidian rows get hover buttons.
  - Verdicts are on for note-pull and project-promotion cards, for existing tasks only.
  - The task-merge card never gets verdicts, because its rows are Pomodoro entries, not
    tasks.

**Verified seams that shape the phases:**

- **bob-cli capture.**
  - `src/native/capture/cli.rs` holds the arg constructors. `pull_in_arg` is at :681 and
    `parse_inbox_route_pick` (split on the first `=`) is at :978. `CaptureRequest` is at
    :823.
  - `src/native/capture/pull_in.rs` has `plan_with_pull_in`, the probe plan that maps
    numbers to keys, `reject_unknown_keys`, and `choice_required_error`.
  - The batch post-passes in `capture/plan.rs` are `apply_successor_net_post_pass`
    (:739) and then `apply_inbox_routes` (:759).
  - Offer rows live in `capture/output.rs`: `TaskCompleteOfferJson` :1262, `PullInJson`
    :1309, `all_offer_rows` :3019, and the human printers around :6010–6700.
  - Error constructors to model new refusals on: `CaptureError::inbox_route_stale`
    (usage, exit 2) and `inbox_route_refused` (io, exit 1) in `output.rs`.
  - Short flags in use: `b c d D E f h i m n p r R s t`. **`-x` and `-X` are free.**
    `src/native/capture_flag_guard.rs` `capture_spec()` must learn them. Today an
    undeclared `-x…` is treated as TEXT.
- **bob-cli engines.**
  - `task_complete/successors.rs` has `plan_successors`, `apply_selection`,
    `plan_recovery_only` :812, and the recovery writes (`apply_recovery_writes`,
    `freshness::placement::mark_unblocked`).
  - **There is no root-only Done primitive.** `complete_task_tree` always walks embeds,
    even under `RootPolicy::Explicit`. The single-line pieces are in
    `task_complete/tree.rs` `finish_close_node` :641–679 and
    `capture_task_toggle/task_update.rs` `set_task_line_status`.
  - `complete_existing_task` needs a block ID, so it **cannot** close the ID-less offer
    rows whose key is `path@digest~n`.
  - The cancel writers are `task_cancel/line.rs` `edit_source` and
    `edit_source_create_log`. With an empty reason they write no log entry. **Rust has
    no `🤷 no reason given` fallback.**
  - Eligibility verdicts already exist: `task_complete/verdicts.rs` `complete_verdict`
    and `task_cancel/verdicts.rs` `cancel_verdict`.
  - Shared JSON vectors live in `tests/fixtures/<name>/vectors.json`, each with a
    README; bob-plugins copies them byte for byte. Capture JSON has no `schema_version`.
    Feature detection is through `supported: true` objects.
  - Offered rows are never "already planned". Rows with a live Task Link under today's
    open Pomodoros fold as `already_planned` and are not offered (§12.2 ladder). **So a
    verdict row never needs ledger edits.**
- **bob-plugins navigation-hotkeys (nav).**
  - `src/206-successor-picker-modal.js` (816 lines) holds `handleKeydown` :257, which
    uses the `ctrlKey||metaKey` convention, has a 350 ms Enter guard, and ignores
    repeat/IME. Also there: `markedKeys`, `prospectiveKeys` :399, `renderResults` :431,
    the footer :411, and `pickSuccessorOffers` :717. **A result with zero picks resolves
    `null` today.**
  - `createSuccessorPickerApi` (`{version: 3, pick}`) is at :615. It is exposed in
    `480-review-jump-and-nav-api.js:936`, which has 5 lines of headroom.
  - The model normalizer is `206a-successor-picker-model.js:250`. Muted rows are
    `selectable:false` with `mutedReason`.
  - The cancel planner is `240-cancel-and-lanes.js` `planTaskCancelBatch` (pure). The
    Cancel Log policy is in `030` `planCancelLogEntry`. The `🤷` text is the constant in
    `010:373`.
  - `654-task-undo-notice.js` `showGuardedUndoNotice` is not on `api`.
  - The notice API is `285-unblocked-notice.js` `createNoticeApi` (v3).
  - The `Ctrl+X` merge lives only in `PomodoroEntryMovePickerModal` (`200:765`).
  - The walk hold is `holdReviewWalkForSuccessorPicker` (`206:664`), with
    `continueReviewWalkAfter` in `536`.
  - Project promotion (`667`, model `122`) and task merge (`429a`) reuse the same modal.
- **bob-plugins task-status-cycler (cycler).**
  - `137-plugin-successor-project.js` `planAndApplySuccessorsNow` does plan → await
    `pick()` → re-read → re-plan → apply. Its `cancelledClose` path (:45) is
    recover-only.
  - The Done line writer is `160` `completeTaskLineText`.
  - `[unblocked::]` is written by `088` `rewriteRecoveredTaskLine`.
  - The receipt goes through `presentSuccessorModel` (`135:685`), and take-back records
    through `rememberSuccessorPass` (`136`).
  - **Deadlock rule:** inside the queued pass, never call queued APIs
    (`api.recoverBlockedDependents`, `finalizeClosedTasks`, nav's
    `runPostCancelEffects`). Use the `…Now` variants.
  - The cycler api is v4 (`110-plugin-lifecycle.js:12`).
- **Fragment cap.** Each hand-edited fragment must stay at or below 1000 lines. Headroom
  is tight in nav 206 (184), 200 (37), 480 (5), 285 (4), and 700 (1), and in cycler 137
  (20), 085 (9), and 085b (34). New logic goes in new fragments listed in
  `src/fragments.json`. Every new `scripts/*.cjs` test must be added to the `test`
  script in `package.json`.
- **Bob Mac Capture.**
  - `Sources/BobMacCapture/CapturePullInChoice.swift` (1092 lines) holds the state:
    `markedKeys`, `effectiveSelectionKeys` with the highlight fallback, `markAll`, and
    `pullInActionTitle`.
  - `CapturePullInChoiceView.swift` (996 lines) has `CapturePullInCheckbox`, the
    capsule, rows, and the footer. The primary button is disabled at zero picks (:947).
  - In `CaptureKeyCommandRouter.swift`, `pullInChoiceCommand` (:623) has no `X`
    handling. `KeyCode.x = 7` already exists.
  - The Move card's portable state,
    `Sources/CaptureCore/CaptureMoveChoicePresentation.swift`, is the precedent for
    exclusive per-row outcomes, outcome keys that advance, no Return fallback, and
    `reseed`.
  - Decoding is in `Sources/CaptureCore/CaptureModels.swift` (`CapturePullIn` :7046,
    `TaskCompleteOffer` :6861). The argv is built in
    `Sources/CaptureCore/BobProcessClient.swift` `capture(...)` :751, with the
    `CapturePullInSelection {keys, none}` capability gate.
  - **Today a stale or refused submit drops the card's marks.**
  - Linux `just portable-check` covers only CaptureCore, RefsCore, and refs-rank.
    `Sources/BobMacCapture*` needs a Mac compile and the exact-SHA macOS CI run (see the
    repo's `AGENTS.md`).

## Shared design (binding for every phase)

### Vocabulary

- **Verdict.** Every offer row holds exactly one verdict: **Leave** (the default; the
  row recovers only), **Pull in** (today's mark), **Done**, or **Cancel**.
- **Card-resolved row.** A row applied as Done or Cancel.
- **Close descriptor.** The engine's per-row report of whether Done and Cancel are
  allowed, why not, and what each would do.
- **Engine.** Whichever planner produced the offers: Rust capture for the CLI and the
  Mac, the cycler for Obsidian Done and park gestures, and the cycler API for Obsidian
  promotion. **Only the engine decides eligibility and authors consequence copy.**
  Presenters (nav picker, Mac card) render what they are given and never compute either.
  This extends `decisions:mac-capture-is-a-thin-client` to nav.

### Row verdict rules

1. **Stage, never write on the keypress.** Only an explicit commit writes (A1).
2. **One verdict per row.** A different verdict key replaces the current verdict. Tab or
   Space on a Done row makes it Pull in, and `Ctrl+X` on a marked row makes it Done.
   Pressing the same key again returns the row to Leave (A4).
3. **Verdict keys act on the highlighted row only, never on the marked set (A3).**
   - Bulk keys (Obsidian `Ctrl+A`, Mac `⌘A`) touch only Leave and Pull-in rows. They
     never override a Done or Cancel row.
   - Retarget (`Ctrl+Shift+Enter`) applies to Pull-in rows only.
4. **Setting a verdict advances the highlight; clearing one stays put.**
   - Setting moves to the next selectable row in visual order, and stays put at the end.
   - This holds on both surfaces, so a wrong press is fixed with ↑ and the same key.
   - Pull-in keys keep their current movement (Obsidian Tab moves; Mac Space stays).
5. **Cancel opens an optional inline reason (A5).**
   - `Enter`/`↩` confirms; a blank reason is allowed. Then the highlight advances.
   - `Esc` restores the row's previous verdict.
   - Blur confirms the text typed so far.
   - The reason is one trimmed line, with newlines collapsed to spaces.
   - Done never prompts, including for a Work Log.
6. **Verdicts ride every commit; only Esc discards them (A9).**
   - Once any Done or Cancel exists, Return no longer falls back to the highlighted row.
   - Mac `⇧↩` applies the verdicts and skips pull-ins.
   - Mac `⌃↩` applies everything and opens notes.
7. **Two-step Esc (A10).**
   - While at least one Done or Cancel is staged, the first Esc (or `⌃[`, or an Obsidian
     click outside the card) arms a warning instead of closing. The second Esc discards.
   - Any other key disarms the warning.
   - Pull-in marks alone never arm it; that behavior is unchanged.
8. **Literal Control only (A12).**
   - Obsidian matches `ctrlKey && !metaKey && !altKey` (plus `shiftKey` for Cancel) and
     key `x`/`X`. Match `event.code === "KeyX"` as a layout fallback.
   - Mac matches exactly `⌃X` and `⌃⇧X`.
   - Cmd+X stays Cut everywhere.
   - On Linux Obsidian, if the filter input has a non-empty text selection, `Ctrl+X` is
     left to the native Cut. Otherwise the card owns it and calls `preventDefault`.
   - The chords live only in the successor card. They never go in `FilteredPickerModal`,
     and are never global.
9. **Repeat and IME guards.**
   - Ignore `event.repeat`, `isComposing`, and keyCode 229 (Mac: `isRepeat`), so holding
     a key resolves one row, not a sweep.
   - The 350 ms Obsidian Enter guard is unchanged and also covers Enter in the reason
     field.
10. **Ineligible rows.**
    - A chord on an ineligible row writes nothing. Obsidian shows a ~200 ms horizontal
      shake (none under reduced motion); the Mac plays `NSBeep`.
    - The card's status line then shows the engine's `why` in refusal styling.
    - Muted (still-blocked) rows have no verdict slot at all.
11. **Feature gating.**
    - Hints, hover buttons, and chords appear only when the model or preview carries
      close support **and** at least one visible row is eligible.
    - Otherwise the card looks and behaves exactly as today. This covers older bob,
      older nav, the task-merge card, and capture-created project cards whose rows are
      all new.

### Eligibility (engine-owned; Done and Cancel reported separately)

A row is eligible when **all** of these hold:

- **E1.** It is a selectable offer row of an existing task. Tasks created by this
  capture or gesture are refused with `new_task`.
- **E2.** It is not recurring (`recurring`).
- **E3.** Its current status is open (` `, `/`, `*`, or `?` before recovery) and it is
  not Done or Cancelled (`status`).
- **E4.** The existing complete and cancel verdict functions would accept it.
  - Rust: `complete_verdict` and `cancel_verdict`.
  - JS: the Task Card / `completeTaskLineText` refusals.
- **E5 (Obsidian only).** The surface can write it: nav `api.successors.version >= 4`
  and `api.cancel.version >= 1`.

In v1, Done and Cancel eligibility are always equal. They stay separate booleans for
later.

The `close` descriptor per offer row has these fields. Rust emits snake_case; the JS
model uses the camelCase names in brackets.

```text
close: {
  complete: bool,
  cancel: bool,
  why?: string,                 // present iff complete or cancel is false
  complete_consequence?: string // [completeConsequence] present iff complete
  cancel_consequence?: string   // [cancelConsequence]  present iff cancel
  scope_note?: string           // [scopeNote] open subtasks / ^prj residents left open
  unblocks: int                 // dependents this row alone would recover
}
```

- The consequence copy assumes that **only this row closes**.
- The receipt reports the exact joint result. For example, two verdict rows that jointly
  unblock C are reported once C actually recovers.

### Writes (byte-identical in Rust and JS; pinned by SV vectors)

|                | Done (`Ctrl+X`, `-x`)                                                                                                                                                                                                         | Cancel (`Ctrl+Shift+X`, `-X`)                                                                                                                                                                                                                 |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Task line      | **Root line only.** Status `x` plus `[completion:: today]`, byte-identical to the existing Done line writers (Rust `finish_close_node` line edit; JS `completeTaskLineText`). Indented or embedded child tasks are untouched. | `[-]` plus `[cancelled:: today]` from the existing cancel line writers. Root line only.                                                                                                                                                       |
| Log            | None. No Work Log prompt.                                                                                                                                                                                                     | **Task Card parity.** An authored reason prepends `- *date* — reason` to the `❌ CANCEL LOG`, creating the log if it is missing. A blank reason writes the `🤷 no reason given` entry only if a Cancel Log already exists; otherwise nothing. |
| Other fields   | Untouched, including `[fresh::]`, `[keeps::]`, and `[unblocked::]`. Closing ends review, so nothing is stamped.                                                                                                               | Same.                                                                                                                                                                                                                                         |
| Today's ledger | None. Verdict rows never have a live Task Link under today's open Pomodoros. If one exists at apply time, the row counts as changed (see Failure).                                                                            | Same.                                                                                                                                                                                                                                         |
| Dependents     | Recover exactly as a recover-only close does today: Blocked → Ready, `#prj_is_blocked` stripped, and `[unblocked:: today]` per the WRITE rule. **Never offered (A7).** Reported in the receipt.                               | Same, as cancel recovery always is.                                                                                                                                                                                                           |

- **Done on a `^prj` row.** This is usually a `project_finished` row, and it is the
  feature's first demo. It is the same root-only Done. Its consequence reads
  `✓ Finishes ‹P›`. Recovery runs exactly what a recover-only close of that `^prj` runs
  today.
- **Disjoint by construction.** A row is never both picked and closed.
- **Inbox routes.** In capture, an inbox-route answer that names a card-resolved task is
  an `offer_verdict_conflict`.

### Cascades

- **v1 is recover-only.** Dependents of a card-resolved row recover. They get
  `[unblocked:: today]` under the WRITE rule and come back as a TICKLER in tomorrow's
  `]s` walk. They are never offered. One offer round per gesture holds on every surface.
- **v2 is out of scope.** It would add an in-card "Newly available · after completing B"
  section.
- **Cancellation never generates offers**, either way.

### Failure, identity, and ordering

- **Identity is the offer key.**
  - `path#^id` for rows with a block ID.
  - `path@digest~n` for ID-less rows. ID-less rows **can** be closed; only a pick mints
    an ID.
  - Verdicts bind to the line the card showed. The engine refuses rather than resolving
    a look-alike neighbour.
- **Capture (CLI and Mac) is all-or-nothing.**
  - The verdict post-pass runs on the staged batch snapshot right after
    `apply_successor_net_post_pass` and before `apply_inbox_routes`.
  - Any refusal writes nothing. The existing disk-preimage check in `write_staged_files`
    still guards external edits.
  - A blind replay after an unknown-outcome timeout is refused by bob, because rows that
    were already closed are no longer offers. The Mac still never auto-retries.
- **Obsidian is best-effort per row.**
  - The predecessor close is never rolled back.
  - After `pick()`, the cycler re-reads, re-plans with the picks, and applies them
    through the existing path. Then, **in the same queued mutation**, it runs a separate
    **verdict step**.
  - The verdict step revalidates each close against the line the pick pass left. That is
    the planned post-pass line when the pass touched the row, else the pre-card line.
  - A row that changed, disappeared, or gained a live link is refused alone. The rest
    still applies.
  - The verdict step writes its line edits and their recoveries through the guarded
    writer. It records `{path, before, after}` per touched file for Undo.
  - A partial failure must never look like a successful Apply.
- **Review walk.**
  - The existing gesture hold covers the card.
  - The walk advances **once** for the original landing.
  - It must never land on a row resolved in the same commit.
- **Reopen (A11).** Reopening the predecessor the same day takes back only untouched
  pull-in links. Card-resolved rows stay closed.
- **Undo.**
  - **Obsidian:** the receipt carries a guarded **Undo** for the verdict step. It
    restores exact preimages only while every touched file still equals what the step
    wrote. It is all-or-nothing and says why when it refuses. After that, `Alt+]` and
    `Alt+[` reopen tasks as usual.
  - **Mac and CLI:** there is no take-back, the same as `!`. The receipt names every
    resolved task.

### Canonical copy

The contract phase copies this table into `docs/task-dependencies.md` §12.22. Every
surface uses the doc table verbatim. Notation:

- `{S}` is the row's task text and `{D}` a dependent's text.
- `{P}` is a `^prj` row's project note stem.
- `{Status}` is the row's status name as the card shows it, after the gesture's
  recovery.
- Plurals follow the count.

| Where                                              | Text                                                                                                                                                        |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Consequence, Done                                  | `✓ Completes “{S}” · {Status} → Done`                                                                                                                       |
| Consequence, Done on a `^prj`                      | `✓ Finishes ‹{P}› · {Status} → Done`                                                                                                                        |
| Consequence, Cancel                                | `⊘ Cancels “{S}” · {Status} → Cancelled`                                                                                                                    |
| Unblock suffix                                     | `· unblocks “{D}”` / `· unblocks {k} tasks`                                                                                                                 |
| Tickler suffix (every recovered dependent stamped) | `· back tomorrow as a tickler` / `· back tomorrow as ticklers`                                                                                              |
| Scope suffix                                       | `· its {N} open subtask stays open` / `· its {N} open subtasks stay open`; on a `^prj`: `· its {N} open task stays open` / `· its {N} open tasks stay open` |
| `why` (new)                                        | `“{S}” is new in this capture — drop it from the draft instead`                                                                                             |
| `why` (recurring)                                  | `“{S}” repeats — close it in Obsidian so Tasks writes the next one`                                                                                         |
| `why` (status)                                     | `“{S}” is {Status} — only open tasks close here`                                                                                                            |
| `why` (old nav; Obsidian only)                     | `Update Bob Navigation Hotkeys to close tasks from this card`                                                                                               |
| Receipt, Done                                      | `✓ completed “{S}” · {Status} → Done`, plus `· unblocked “{D}”` / `· unblocked {k} tasks`; a dry run uses `✓ would complete …` / `would unblock`            |
| Receipt, Cancel                                    | `⊘ cancelled “{S}” · {Status} → Cancelled · “{reason}”` (no reason clause when blank), plus the unblocked clause; a dry run uses `⊘ would cancel …`         |
| Receipt, tickler line                              | `↳ “{D}” back tomorrow as a tickler` / `↳ {k} unblocked tasks back tomorrow as ticklers`                                                                    |
| Receipt, Obsidian refusal                          | `{n} not closed · changed since the card opened`                                                                                                            |
| Status line, Esc armed                             | `Esc again to discard {N} close` / `Esc again to discard {N} closes`                                                                                        |
| Footer summary                                     | `pull in {a} · ✓ {b} done · ⊘ {c} cancel`, omitting zero segments; the plan meter counts pull-ins only                                                      |
| Reason field                                       | placeholder `Why cancel? (optional)`, trailing hint `↵ · Esc back`                                                                                          |
| Cancel row detail                                  | `“{reason}”`, or `no reason` (faint)                                                                                                                        |
| Accessible state                                   | `will be completed` / `will be cancelled: {reason}` / `will be cancelled, no reason`                                                                        |

### The look (beautiful by reuse; no new chrome)

The goal is that **a glance at the card says what Return will do.**

| Verdict | Mark slot                                        | Title                                                      | Trailing capsule             | Accent                                                                                                                                      |
| ------- | ------------------------------------------------ | ---------------------------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Leave   | `□` (today)                                      | normal                                                     | `→ DEST` (today)             | —                                                                                                                                           |
| Pull in | `☑` (today)                                      | normal                                                     | `→ DEST` (today)             | today's                                                                                                                                     |
| Done    | ✓ (Lucide `circle-check`; SF `checkmark.circle`) | secondary text colour, **no strikethrough** (CL6/CP6, A13) | `✓ Done` replaces `→ DEST`   | Done green: the existing `.bob-cnp-relation-complete-*` colour on Obsidian, `.relationCompleteLabel` on Mac                                 |
| Cancel  | ⊘ (Lucide `circle-slash`; SF `circle.slash`)     | secondary, **no strikethrough**                            | `⊘ Cancel` replaces `→ DEST` | Cancelled tint: the existing `.bob-cnp-relation-cancel-*` colour on Obsidian, `RelationSourceEffect.cancel` / `.relationCancelLabel` on Mac |

- **Cascade at a glance.** Eligible rows with `unblocks > 0` show a small muted
  `unblocks {k}` chip in the detail line. The cascade is visible before you choose.
- **Status line.** One line above the footer replaces the research's per-row strip,
  because verdict keys advance and the user would otherwise never see the consequence.
  - It shows the consequence sentence of the most recent Done or Cancel staging.
  - Moving the highlight onto a Done or Cancel row shows that row's sentence.
  - Clearing the verdict it describes clears it.
  - An armed Esc warning (amber, the caveat colour) or an ineligible `why` (refusal
    styling) temporarily replaces it.
  - It is a polite live region: VoiceOver and aria announce it.
- **Hints.**
  - Obsidian chips: `Tab mark · Ctrl+X done · Ctrl+Shift+X cancel · ↵ apply · Esc none`.
    The `↵` chip reads `pull in` until a Done or Cancel exists.
  - Mac footer adds `⌃X Done · ⌃⇧X Cancel`.
- **Mac buttons.**
  - Primary: `Pull In {a}` / `Close {n}` / `Pull In {a} · Close {n}`. It is enabled
    whenever picks or verdicts exist.
  - `⇧↩`: `Skip pull-ins · Close {n}` while verdicts exist.
- **Pointer.**
  - Obsidian rows reveal two 14 px icon buttons (✓, ⊘) on hover or highlight, left of
    the capsule. Clicking one does what the chord does, minus the advance. On ineligible
    rows they are disabled, with the `why` as their tooltip.
  - Mac rows get a context menu and VoiceOver custom actions: "Mark Done", "Mark
    Cancelled…", "Pull In", and "Leave".
- **Motion.**
  - The glyph and capsule crossfade over ~150 ms: CSS `transition` on Obsidian,
    `.snappy(duration: 0.15)` on the Mac, matching `CapturePullInCheckbox`.
  - Rows never reorder and focus never jumps.
  - Everything is disabled under reduced motion.
- **Colour is never the only signal.** The glyph and words always carry the verdict.

Mock (Obsidian; `◀` marks the highlighted row):

```text
╭─ ⎇ Choose successor tasks ────────────────────────────────────────╮
│ Fix flaky gkeep test · 4 offers                                   │
│ ▏Filter tasks, notes, and groups                                  │
│ HAND-OFFS                                                         │
│ ☑  Re-enable gkeep sync in CI                       → CAPTURE     │
│    sase · Ready                                                   │
│ ✓  Write gkeep fixture README                       ✓ Done        │
│    sase · Ready · unblocks 1                                      │
│ ⊘  Add retry wrapper around gkeep client            ⊘ Cancel      │
│    sase · Ready · “Fixed it a different way”                      │
│ □  Port gkeep tests to nextest                 ✓ ⊘  → CAPTURE  ◀  │
│    sase · Ready                                                   │
│ ⊘ Cancels “Add retry wrapper around gkeep…” · Ready → Cancelled   │
│ pull in 1 · ✓ 1 done · ⊘ 1 cancel                plan 5/8 → 6/8   │
│ Tab mark · Ctrl+X done · Ctrl+Shift+X cancel · ↵ apply · Esc none │
╰───────────────────────────────────────────────────────────────────╯
```

### Contract: `bob capture` CLI and JSON (additive; no `!`)

These flags sit alphabetically among the capture options per `cli_rules.md`. Each has a
short alias. There is no subcommand.

```text
  -X, --cancel-offer <N|KEY[=REASON]>   Cancel a successor offer instead of pulling it in;
                                        REASON goes to its Cancel Log. Repeatable, one
                                        offer per flag.
  -x, --complete-offer <N|KEY>          Mark a successor offer Done instead of pulling it
                                        in. Repeatable; accepts a comma list.
```

**Parsing:**

- `-X` splits on the first `=`, like `-i`. A missing or empty REASON is blank.
- Numbers index the offers the same command reports without a selection, through the
  same probe plan `--pull-in` uses.
- `-p all` means every selectable offer **without** a verdict.
- A verdict counts as a choice for `--require-pull-in-choice`. With verdicts and no
  `-p`, picks are `none` and `pull_in.choice` reports `"none"`.
- A note-pull item with zero picks and at least one verdict is valid.

**Refusals.** Each writes nothing and carries `pull_in` plus the flat `offers` list with
each row's `close` descriptor.

| Code                     | Class         | When                                                                                                                                    |
| ------------------------ | ------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `offer_verdict_conflict` | usage, exit 2 | The same row in two of `-p`, `-x`, and `-X`; `-X` twice for one row; or an `-i` answer on a card-resolved task                          |
| `offer_close_stale`      | usage, exit 2 | An unknown, stale, or out-of-range `-x`/`-X` selector. The message lists the current offers (`n key text`), as `inbox_route_stale` does |
| `offer_close_refused`    | io, exit 1    | The row's descriptor refuses the outcome. Extra `refused: {key, n, outcome, why}`                                                       |

**JSON.** Every addition is omitted when it does not apply:

- **On every offer row:** `offer.close` (the descriptor above).
- **On a resolved row:** `offer.verdict: "done" | "cancelled"` (dry runs included),
  `offer.selected: false`, and `not_linked: "resolved_in_card"`.
- **`pull_in.close: {supported: true, done_count, cancelled_count}`.** This is the
  capability flag. It is emitted by every bob that accepts `-x`/`-X`, whenever `pull_in`
  is emitted.
- **Top-level `offer_closes: [...]`**, one entry per resolved row:
  - `n`, `key`, `outcome: "done"|"cancelled"`, and `reason?` (an authored reason only).
  - The row: `note_path`, `line`, `block_id?`, `text`, `previous_status`,
    `previous_status_name`.
  - `cancel_log?: "written"|"created"|"fallback"|"none"`.
  - `recovered: [{note_path, block_id, text, previous_status_name, status_name, unblocked_stamped}]`.
  - `message`: the canonical receipt row.

**Human output:**

- A resolved row prints its receipt message in place of `linked` / `not pulled in`,
  followed by its `↳` tickler line.
- The dry-run pull-in hint becomes `-p 1,2 · -p all · -x N done · -X N[=why] cancel`.

### Contract: plugin APIs (feature-detected; older versions behave as today)

**`api.successors` v4** (nav):

- The model gains `close?: {supported: true}` and row `close?` (the camelCase
  descriptor).
- `pick(model)` resolves
  `{picks: [{key, destination?}], closes: [{key, outcome: "done"|"cancelled", reason?}]}`.
- **Zero picks with at least one close is not `null`.**
- `null` still means nothing was chosen: Esc, discard, click-outside, unload, or a
  second picker.
- The normalizer keeps row `close` only when the model has `close.supported`.

**`api.cancel` v1** (nav, new). It is pure, so callers do the writing:

```js
cancel: {
  version: (1, planCancel(content, targets /* [{line, rawLine}] */, { date, reason }));
  // → the planTaskCancelBatch result; reason "" applies the 🤷 fallback rule
}
```

**`api.notice` v4** (nav). `showUnblocked(payload)` additionally reads:

- `offer_closes` rows:
  `{outcome, text, message, recovered: [{text, unblockedStamped}]}`.
- `offer_close_refused: {count, rows: [{text, why}]}`.
- `undo?: {restores: [{path, before, after}]}`.

Undo renders a `Undo closes` action that restores through nav's guarded restore, the
same semantics as `showGuardedUndoNotice`. Older payloads render as before.

**Cycler api v5** (promotion phase):

- `describeOfferCloses({rows, gesture})` returns descriptors keyed by offer key.
- `applyOfferCloses({closes, expectedLines})` runs the verdict step and returns the
  receipt payload plus restores.
- Both are queued like the other public cycler methods.

### Contract: Bob Mac Capture

- **Decoding.** Decode `offer.close`, `offer.verdict`, `pull_in.close`, `offer_closes`,
  and the three refusal codes with `decodeIfPresent` / `try?`.
- **Older bob.** No close support: no hints, and the chords beep.
- **Submission.** Submit keys only. `--complete-offer` with a comma list of keys;
  `--cancel-offer KEY=REASON` repeated, with REASON omitted when blank. Never send these
  flags unless the preview advertised `pull_in.close.supported`.
- **Stale or refused submits.** `offer_close_stale`, `offer_close_refused`,
  `offer_verdict_conflict`, and stale-pick refusals reopen the card with bob's error and
  with surviving marks, verdicts, reasons, and highlight intact. This is Move card
  `reseed` style, and the lost count is announced.

## Phase: contract (bob-cli docs + memory)

Work in bob-cli only; this phase changes no code.

1. **Read first.** Read `docs.md` and the research report with `sase memory read` /
   `sase artifact read`.
2. **`docs/task-dependencies.md` §12.22 "Resolving offers in the card".** Restate the
   Shared design as numbered rules, prefixed **RV1…**:
   - verdict model, keys, eligibility E1–E5;
   - writes, cascades, failure and identity, walk, reopen, and undo;
   - the canonical copy table and the look table. Write it so each surface can implement
     from the doc alone.
3. **§12.23 "SV — successor verdict vectors".** Add a Markdown vector table, SV1…SVn,
   with each row's input, verdict, and expected result. It must cover at least:
   - Done on a plain row with a block ID, and on an ID-less row (no mint, no `^id`).
   - Done on a row with open indented subtasks: root only, with the scope note.
   - Done on a `project_finished` `^prj`: the `✓ Finishes ‹P›` copy.
   - Cancel with a reason and an existing Cancel Log; with a reason and no log
     (created); blank with an existing log (`🤷` entry); blank with no log (no entry).
   - Close that recovers one dependent (`[unblocked:: today]`, tickler suffix), and one
     that recovers two dependents (plural copy).
   - Eligibility refusals: new task, recurring, already Done.
   - The conflicts: `-p` with `-x`, `-x` with `-X`, `-X` twice, `-i` on a resolved task.
   - A stale key.
   - `-p all` skipping verdict rows.
   - Verdict-only `--require-pull-in-choice`.
   - Verdict rows reporting `not_linked: resolved_in_card`.
   - A dependent recovered by a card close and not offered (A7).
   - Joint unblock by two verdict rows. The engine phase turns these into
     `tests/fixtures/offer_close/vectors.json`.
4. **Update the existing sections.**
   - §12.5 Reporting: the JSON additions.
   - §12.13 Obsidian picker API: `api.successors` v4, `api.cancel` v1, `api.notice` v4,
     and cycler api v5.
   - §12.14 CLI selection: the flags, parsing, and refusals.
   - §11 SL21 stays true (cancel never offers). Add a cross-reference from §12 DEDUP
     noting that card-resolved rows never re-enter as offers.
5. **Memory changes**, under `/sase_memory_write`.

   > [!decision] memory_verdict_record

   Create the decision strand `decisions:successor-cards-resolve-with-staged-verdicts`,
   titled "Successor Cards Resolve With Staged Verdicts; Card-Resolved Rows Recover But
   Never Offer".
   - **Applies to.** bob-cli, bob-plugins, Bob Mac Capture, vault.
   - **Claim.** The one-verdict-per-row model, staged-not-written, Ctrl+X / Ctrl+Shift+X
     on literal Control, root-only closes, the Task-Card-parity cancel reason,
     engine-owned eligibility, recover-only cascades that never offer, best-effort
     Obsidian versus all-or-nothing capture, and a guarded Undo in Obsidian only.
   - **Why.** Cite the research report and the vault evidence it gives: 115 Cancel Logs
     and 0 `🤷` entries, and 5 of 6 same-day closes.
   - **Rejected alternatives.** Write on keypress; nested Task Card; receipt buttons; a
     tray; auto-close; `!note:id` draft lines; and the bare `x` alias.
   - **Cost.** Four surfaces share one contract and one vector file; Obsidian makes a
     second write step.
   - **Reopens when.** Bryan wants dependents of a resolved row today (v2's "Newly
     available" section), or another picker adopts the verdict vocabulary.
   - Use status `accepted` and today's `decided` date. Link
     `[[decisions:unblocked-tasks-are-offered]]`,
     `[[decisions:unblocked-tasks-resurface-as-ticklers]]`, and
     `[[decisions:mac-capture-is-a-thin-client]]`.

   > [!decision] memory_offers_supersede

   Mark `decisions/unblocked-tasks-are-offered.md` with
   `metadata.status: superseded-in-part` and `superseded_by` naming the new record. Add
   a one-line `[[decisions:successor-cards-resolve-with-staged-verdicts]]` back-link
   stating what was retired: an offer row may now be resolved Done or Cancel in the
   card, and that close's dependents recover without being offered. Do not otherwise
   edit the body.

   > [!decision] memory_glossary

   Add two sentences to `glossary/successor-offer.md`: a card row may instead be marked
   Done or Cancel and closed root-only in the same commit, and its dependents recover
   without being offered. Link the new decision record.

   Run `sase memory init` after any memory edit. If a memory decision was declined,
   record a `PROPOSED FOLLOW-UP:` note on this phase's bead instead.

6. **Gate.** `just docs-check` must pass. Commit as `docs(task-dependencies): …`.

## Phase: engine (bob-cli Rust, no CLI surface yet)

1. **New module `src/native/task_complete/offer_close.rs`**, with unit tests beside it.
   It is pure: text in, postimages out.
   - **`offer_close_descriptor`.** Inputs are the staged snapshot, the row's
     `note_path`/line/key, `is_new`, the status settings, and the dependency context. It
     returns
     `OfferCloseDescriptor {complete, cancel, why, complete_consequence, cancel_consequence, scope_note, unblocks}`.
     - Reuse `complete_verdict` and `cancel_verdict` for E1–E4.
     - Compute `unblocks` and the tickler suffix by simulating the close with
       `plan_recovery_only` on a cloned snapshot.
     - Count open child tasks (indented subtasks plus embedded tasks that
       `complete_task_tree` would have closed) for the scope note.
   - **`plan_offer_done`.** A root-only single-line Done. Factor the line edit out of
     `finish_close_node` (`tree.rs:641-679`) so the `!` path and this path share one
     writer. Do not add a `RootPolicy` variant: a single-line edit needs no tree walk.
   - **`plan_offer_cancel`.** Uses the Task-Card reason policy. Add a small policy
     wrapper in `task_cancel/line.rs`:
     - An authored reason calls `edit_source_create_log`.
     - A blank reason with an existing log writes the `🤷 no reason given` entry. Define
       one Rust constant whose bytes match nav's.
     - A blank reason with no log writes nothing. It reports `CancelLogOutcome` as
       written / created / fallback / none.
   - **`plan_offer_closes(snapshot, verdicts)`.** Applies all line edits, then runs one
     `plan_recovery_only` over the union of closed rows with `link_unblocked: false`. It
     returns the postimages, per-row results, and the recovered rows: identities,
     whether each was stamped, and their messages.
2. **Attach descriptors in capture.** Add a read-only post-pass that sets `offer.close`
   on every offer row reachable from `all_offer_rows`, in both dry runs and real runs.
   It runs on the final staged snapshot after `apply_successor_net_post_pass`. Rows that
   the same capture creates get `new_task`.
   - **Performance.** One recovery simulation per offer row. Bound it by reusing the
     batch's dependency context.
   - **Gate.** A `--dry-run` with 20 offers must stay within the existing preview timing
     (`tests/cli` timing guard if one exists; otherwise measure and record it in the
     commit body).
3. **Fixture.** Write `tests/fixtures/offer_close/vectors.json` and its README,
   following the `task_cancel` README's layout and provenance section.
   - Cover every SV vector that is not CLI-only.
   - Sections: `eligibility`, `done`, `cancel`, `recovery`, `copy`.
   - Mark Rust-only cases `rust_only`.
   - Rust tests load it, as `task_cancel/tests.rs:139` does.
   - If a vector in the doc is wrong, fix the doc table in the same commit and say so.
4. **No CLI flags in this phase.** `pull_in.close.supported` is not emitted yet. The
   only observable change is the additive `offer.close` field.
5. **Gate.** `just` lint, `cargo test`, and `just docs-check` must pass. Commit as
   `feat(capture): …`. Do not touch the Cargo.toml version or CHANGELOG.md
   (`decisions:bob-cli-releases-ride-a-daily-train`).

## Phase: cli (bob-cli `bob capture` surface)

1. **Flags.**
   - Add `complete_offer_arg()` (`-x`, Append, comma lists) and `cancel_offer_arg()`
     (`-X`, Append, one per flag) in `capture/cli.rs`, registered alphabetically in
     `build_cli`, with excellent help text and examples.
   - Extend `CaptureRequest` with `offer_verdicts`.
   - Update `capture_flag_guard.rs` `capture_spec()` (`short_values`, `LONGS`; the
     `assert_spec_matches` test), `completion/kinds.rs`, `tests/cli/help_options.rs`,
     and `tests/cli/completion/protocol.rs`.
2. **Resolution, in `pull_in.rs` or a sibling `offer_verdicts.rs`.**
   - Map numbers to keys through the existing probe plan.
   - Detect `offer_verdict_conflict` before any planning side effects.
   - Exclude verdict keys from `-p all`.
   - Treat verdicts as a choice for `--require-pull-in-choice`.
   - Report `offer_close_stale` and `offer_close_refused` using the descriptors.
   - Add the three `CaptureError` constructors in `output.rs`, modelled on
     `inbox_route_stale` and `inbox_route_refused`.
3. **Post-pass.**
   - Call `plan_offer_closes` on the staged snapshot right after
     `apply_successor_net_post_pass` (`plan.rs:739`) and before `apply_inbox_routes`.
   - Stage the postimages into the batch planner so the one `write_staged_files` commit
     carries everything.
   - Verdict rows go through `apply_selection` as not-selected recover rows, then the
     verdict line edit replaces their status. Recovery writes on a verdict row stay as
     planned; nothing is stripped.
4. **Output.**
   - JSON: `offer.verdict`, `not_linked: "resolved_in_card"`, `pull_in.close`, and
     top-level `offer_closes`.
   - Human output: receipt rows and `↳` lines in the offer group printers, and the new
     pull-in hint.
   - Dry runs use the `would` forms.
5. **Docs.**
   - `docs/capture.md`: "Choosing from a terminal", "Command-line options", the JSON
     section, and a short worked example closing a `project_finished` `^prj`.
   - Run `just docs-gen` for the CLI reference, then `just docs-check`.
6. **Tests.** Add `tests/cli/capture/successor_verdicts.rs`, covering every CLI-level SV
   vector end to end, plus:
   - verdicts combined with picks and `-p none`;
   - a disk-preimage change between planning and commit refusing the batch;
   - a replay after success returning `offer_close_stale`.
7. **Gate.** Full `cargo test` plus lint must pass. Commit as `feat(capture): …`
   (additive, no `!`).

## Phase: nav-card (bob-plugins `bob-navigation-hotkeys`)

Read the bob-plugins `AGENTS.md`.

1. **Placement.** Put the verdict logic in new fragments, for example
   `206c-successor-picker-verdicts.js` for state and keys and
   `206d-successor-picker-verdict-render.js` for rendering, listed in
   `src/fragments.json`. `206` keeps under 1000 lines.
2. **State.** `verdicts: Map<key, {outcome, reason?}>` alongside `markedKeys`, with
   exclusivity enforced in one setter. Also `reasonEditKey`, `escArmed`, and
   `statusLine`.
   - `prospectiveKeys()` loses the highlight fallback once any verdict exists.
   - `Ctrl+A` skips verdict rows.
   - Retarget ignores verdict rows; with no marks and a verdict on the highlighted row
     it shows a soft refusal.
3. **Keys.**
   - In `handleKeydown`, add Ctrl+X and Ctrl+Shift+X exactly per rules 2–10, including
     the Linux Cut exception.
   - The reason field is an `<input>` that replaces the row's detail line. It owns Enter
     and Esc while focused and returns focus to the filter afterwards.
   - Register `Escape` on the modal scope so the two-step Esc works.
   - Guard background-click closes while verdicts are staged. Programmatic settles still
     close.
4. **Render.**
   - Mark-slot glyphs (Lucide `circle-check` / `circle-slash` via `setIcon`), the muted
     title, and the capsule swap.
   - The `unblocks {k}` chip, the reason detail, the status line (`aria-live="polite"`),
     and the footer summary.
   - The dynamic `↵` chip and the extra hint chips, shown only under rule 11.
   - Hover buttons with `aria-label`s.
   - Row `aria-label` suffixes per the canonical copy.
   - CSS goes in `styles.css` next to the successor picker block (`:2786-3002`). Reuse
     the relation complete/cancel colour variables; animate with transitions gated by
     `prefers-reduced-motion`.
5. **API.**
   - `createSuccessorPickerApi` returns `version: 4`.
   - `pickSuccessorOffers` resolves `{picks, closes}`, re-filtering closes to keys whose
     descriptor allows that outcome.
   - The normalizer (`206a`) passes `close` through. If `480` lacks room, move the api
     wiring into a new `7xx` fragment.
6. **Tests.** Extend `scripts/test-navigation-successor-picker.cjs`, or add
   `…-successor-verdicts.cjs` and register it in `package.json`, covering:
   - each row rule;
   - the Linux Cut exception and held Ctrl+X;
   - the reason field (Enter, blank, Esc, blur);
   - two-step Esc and the click-outside guard;
   - no fallback once verdicts exist;
   - `Ctrl+A` skipping verdicts;
   - a v3-shaped model rendering exactly as before;
   - the task-merge model having no chords;
   - zero picks with closes not being `null`.
7. **Docs.** Update `docs/bob-navigation-hotkeys.md` (the successor picker key table)
   and `docs/develop/plugin-apis.md` (successors v4).
8. **Gate.** `npm run build`, `npm test`, and `npm run docs:check` must pass. Then run
   `bob plugins sync`. Old cyclers never send `close`, so the deployed card is unchanged
   until the cycler phase ships.

## Phase: nav-apis (bob-plugins `bob-navigation-hotkeys`)

1. **`api.cancel` v1.**
   - `planCancel` wraps `planTaskCancelBatch` with `fallbackReason` fixed to the
     `🤷 no reason given` constant, so callers cannot diverge from the Task Card.
   - Wire it on the frozen api object, using a new `7xx` fragment if `480`/`700` are
     full.
   - Test it against `scripts/fixtures/task_cancel/vectors.json` plus the four reason
     cases.
2. **`api.notice` v4.**
   - Extend `buildUnblockedNoticeModel` and the renderer (`285`; split into a new
     fragment if needed) for:
     - `offer_closes` rows: ✓/⊘ glyphs in the same tints as the card, no strikethrough;
     - `↳` tickler lines;
     - the refusal line, in refusal styling;
     - the `Undo closes` action.
   - Undo reuses the `654` restore core. Factor a `restoreGuardedSnapshots(restores)`
     helper that both `showGuardedUndoNotice` and the notice call. On success it says
     `Closes undone`; on refusal it names the changed note.
   - Payloads older than v4 render byte-identically, so pin the existing notice tests.
3. **Docs and gate.**
   - Update `docs/develop/plugin-apis.md` (cancel v1, notice v4).
   - Tests: `test-navigation-hotkeys-unblocked-notice.cjs` plus a new cancel-API test,
     registered in `package.json`.
   - Run build, test, and docs:check, then `bob plugins sync`.

## Phase: cycler (bob-plugins `task-status-cycler`)

The worker plans first. These requirements are binding.

1. **Vectors.** Copy bob-cli `tests/fixtures/offer_close/vectors.json` byte for byte
   into `plugins/task-status-cycler/scripts/fixtures/offer_close/`, with a README
   recording the source commit. Where the canonical file exists, the test also asserts
   byte equality with it, as `test-task-status-cycler-relation-successors.cjs` does.
2. **Descriptors.**
   - In a new fragment, compute the camelCase `close` descriptor for every selectable
     offer row the cycler builds in `buildSuccessorPickerModel` (`085b:219`): hand_off,
     project, project_finished, keep_moving, and relation.
   - Copy, `unblocks`, and the scope note come from a recover-only simulation over the
     cycler's dependency index. The copy must match the SV vectors exactly.
   - Attach descriptors and `model.close = {supported: true}` only when nav reports
     `successors.version >= 4` and `cancel.version >= 1`. Otherwise E5 applies: the card
     looks like today.
3. **Picks.**
   - `normalizeSuccessorPicks` accepts `closes`.
   - The pick re-plan excludes close keys from picks.
   - The `null` and zero-pick paths stay as they are.
4. **Verdict step.** This runs in the same queued mutation, after the existing pick
   apply, before `rememberSuccessorPass` and the notice.
   - Resolve each close by key against the fresh re-plan. Revalidate it against the
     expected line (Failure rules). Unresolvable rows are refused individually.
   - **Done:** `completeTaskLineText(line, {date})`.
   - **Cancel:** nav `api.cancel.planCancel`.
   - Group same-note edits and apply them through the guarded writer.
   - Then run recover-only planning for the closed rows. Generalize the `cancelledClose`
     path: no picker, relation offers off, `linkUnblocked: false`, `[unblocked::]` via
     `rewriteRecoveredTaskLine`. **Use the `…Now` functions only (deadlock rule).**
   - Record `{path, before, after}` per touched file across both writes.
5. **Receipt.**
   - `presentSuccessorModel` passes `offer_closes`, `offer_close_refused`, and `undo` to
     `api.notice` v4.
   - The fallback `new Notice` text uses the canonical receipt lines.
6. **Reopen and walk.**
   - Verify that `rememberSuccessorPass` records no verdict rows, so a same-day reopen
     takes back only untouched pull-in links (A11).
   - Add a test that a Ctrl+Enter on a `]s` landing whose card resolves the walk's next
     item advances once and never lands on a resolved row.
   - If nav's `continueReviewWalkAfter` (`536`) uses a pre-write queue that still
     contains it, fix it in nav in this phase.
7. **Tests.** Add `scripts/test-task-status-cycler-successor-verdicts.cjs`, registered
   in `package.json`, covering:
   - every SV vector that is not CLI-only;
   - mixed picks, Done, and Cancel with one combined receipt;
   - a row edited while the card is open being refused alone;
   - a closed row's dependent recovered with `[unblocked:: today]` and not offered;
   - old nav (v3) giving today's card;
   - Undo restoring exactly, and refusing after a later edit;
   - closes on a park gesture's relation card.
8. **Docs and gate.**
   - Docs: `docs/task-status-cycler.md` ("Successor offers and take-backs") and
     `docs/task-relations.md` in bob-plugins if it lists the picker keys.
   - Run build, test, and docs:check, then `bob plugins sync`.
   - **First demo:** Ctrl+Enter on a project's last task, then `Ctrl+X` on the
     `project_finished` `^prj` row, then ↵.

## Phase: promotion (bob-plugins cycler + nav)

1. **Cycler api v5.** Expose `describeOfferCloses` and `applyOfferCloses` (queued),
   implemented on the cycler phase's internals. Bump the cycler `api.version` to 5 and
   document it in `docs/develop/plugin-apis.md`.
2. **Nav project promotion** (`667-plugin-project-promotion.js`, model
   `122-project-promotion-successors.js`).
   - When the cycler api is v5 or later and nav's own successors api is v4, attach
     descriptors to the promotion card's rows.
   - After `pick()`, apply the picks as today, then `applyOfferCloses` for the closes.
   - Show the receipt through `api.notice` v4.
3. **Task-merge card.** `429a` never attaches `close`. Pin that with a test.
4. **Tests and gate.**
   - Tests: promotion with closes, an older cycler giving today's card, and the merge
     card staying pull-in-only.
   - Run build, test, and docs:check, then `bob plugins sync`.

## Phase: mac-core (bob-mac-capture `CaptureCore`, Linux-checkable)

Read the bob-mac-capture `AGENTS.md`.

1. **Decoding.**
   - `TaskCompleteOfferClose` (snake_case), with `TaskCompleteOffer.close` and
     `.verdict`.
   - `CapturePullIn.close` (`supported`, `doneCount`, `cancelledCount`).
   - `CaptureCommandSuccess.offerCloses: [CaptureOfferClose]`.
   - The three refusal codes on `CaptureCommandFailure`, with `refused` and `offers`.
   - Everything uses `decodeIfPresent` / `try?` defaults.
2. **`CapturePullInOutcomes`** (new file in `Sources/CaptureCore/`). This is the
   portable value type for the whole Row verdict rules model.
   - State: marks plus `verdicts: [String: Verdict]`, where `Verdict` is `.done` or
     `.cancel(reason)`.
   - Operations: toggle, replace, and clear; set-advances and clear-stays (returning the
     advance target); bulk mark skipping verdicts; and effective picks with the fallback
     disabled once any verdict exists.
   - Labels: footer summary text, primary label, skip label, and Esc-armed state and
     text.
   - `argvSelection()` builds keys plus closes. `reseed(validKeys:)` returns the lost
     count.
   - Rows use only bob's descriptors. Nothing is computed.
3. **Argv.** `BobProcessClient.capture(...)` gains `offerVerdicts:` and builds
   `--complete-offer a,b` and repeated `--cancel-offer key=reason`. It throws
   `unsupportedOption` unless `pull_in.close.supported` was advertised, as the
   `--pull-in` gate does.
4. **Fixtures.**
   - Add `Scripts/generate-offer-close-fixtures.sh`, modelled on
     `generate-keep-moving-fixtures.sh`, using a bob built from bob-cli master at or
     after the cli phase.
   - Record provenance in `Tests/Fixtures/offer-close-provenance.md`.
   - Cover: a dry run with descriptors, a real run with `offer_closes`, each refusal,
     and an older-bob payload without `close`.
   - Teach `Tests/Fixtures/fake-bob` the new flags.
5. **Tests.** `Tests/CaptureCoreTests/CapturePullInOutcomesTests.swift`,
   `…OfferCloseDecodingTests.swift`, and argv cases in `CapturePullInClientTests.swift`.
6. **Gate.** `just portable-check` (run it on athena if `swift` is missing). Run
   `Scripts/macos-ci-check` before landing. After the push, watch the exact-SHA macOS
   run to green with `/sase_monitor`.

## Phase: mac-card (bob-mac-capture `BobMacCapture`, Mac-only)

The worker plans first. These requirements are binding.

1. **State and model.**
   - `CapturePullInChoiceState` holds a `CapturePullInOutcomes` instead of raw
     `markedKeys` semantics.
   - `CapturePanelModel` actions: `toggleOfferDone`, `beginOfferCancel`,
     `confirmOfferCancel`, `restoreOfferCancel`, and the two-step `cancelPullInChoice`.
   - `confirmPullInChoice` and `chooseNoPullInOffers` pass verdicts.
   - The primary button is enabled when picks or verdicts exist.
   - On the refusal codes and stale picks, reopen the card through `openPullInChoice`
     with the outcomes reseeded, not fresh, and announce the lost count.
2. **Router.** In `pullInChoiceCommand`:
   - `⌃X` and `⌃⇧X` (exact modifiers; repeats consumed) map to new commands.
   - While the reason field is focused, add a `pullInReasonFocused` routing context:
     Return confirms, Esc/`⌃[` restore, and other keys stay native.
   - Esc and `⌃[` follow the two-step rule.
   - `⌘A` skips verdict rows.
   - Add a `CapturePanelFocusTarget` for the reason field.
3. **View.**
   - Mark-slot glyphs (`checkmark.circle` / `circle.slash`) through
     `RelationSourceEffect` symbols and the `CaptureEditorPalette` relation colours.
   - The muted title, the `✓ Done` / `⊘ Cancel` capsule replacing
     `CapturePullInDestinationCapsule`, and the `unblocks {k}` chip.
   - The inline reason `TextField` in the row's detail line.
   - The status line above the footer, in the Move card's detail-strip style.
   - Footer hint items and button labels per the canonical copy.
   - `.snappy(duration: 0.15)` transitions, gated on Reduce Motion.
   - The context menu and `.accessibilityActions`, the accessibility label suffixes, and
     announcements for staging, the Esc-armed warning, and refusal `why`s.
4. **Tests.**
   - Extend `CapturePullInChoiceTests` (router, footer contract),
     `CapturePullInChoiceModelTests` (commit with verdicts only, ⇧↩ with verdicts,
     reopen with choices intact, two-step Esc), and `CapturePullInChoiceDesignTests`
     (rendered Done, Cancel-with-reason, and reason-editing rows).
   - Async tests must never await an untimed continuation.
5. **Docs.**
   - `docs/capture-previews.md` ("Successor offers"), `docs/keyboard.md` ("Successor
     choices"), and `docs/protocol.md` (argv, the compatibility table, refusal reopen).
   - Run `just docs-check`.
6. **Gate.**
   - Compile a scratch copy on the Mac per `AGENTS.md`: `bundle.sh` and
     `just typecheck-tests`.
   - Run `Scripts/macos-ci-check`, push, and watch the exact-SHA macOS run to green with
     `/sase_monitor`.
   - If the Mac is offline, say in the final report that the macOS targets are
     unverified.

## Acceptance matrix (whole epic)

| Scenario                                                   | Required result                                                                                                            |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| B Done, C Cancel with a reason, D marked, Return           | Only B and C resolve, only D links, one combined receipt (Obsidian notice, `offer_closes` in capture)                      |
| Verdicts staged; Return with the cursor on an unmarked row | No highlighted-row fallback                                                                                                |
| Every row Done, nothing marked                             | Apply is enabled and nothing is linked; Mac shows `Close N`                                                                |
| Esc once, then twice, with verdicts                        | Warning, then discard. Obsidian keeps the predecessor; the Mac writes nothing                                              |
| Blank cancel, with and without an existing Cancel Log      | Task Card bytes in both Rust and JS (SV vectors)                                                                           |
| Row with open subtasks                                     | Only the root closes; the scope note is shown before Apply                                                                 |
| Recurring, new-in-capture, or muted row                    | The chord is refused with the engine's `why`; nothing is staged                                                            |
| Closing B recovers C                                       | C gets `[unblocked:: today]`, is not offered, and is named in the receipt                                                  |
| Row edited while the card is open                          | Obsidian refuses that row and applies the rest; capture refuses the batch and the Mac reopens the card with choices intact |
| Same-day predecessor reopen                                | Pull-in links are taken back; resolved rows stay closed                                                                    |
| Linux filter with a text selection, `Ctrl+X`               | Cut happens; no verdict                                                                                                    |
| Held `Ctrl+X`                                              | One verdict, no multi-row sweep                                                                                            |
| macOS Obsidian `Cmd+X`                                     | Cut; no verdict                                                                                                            |
| Old bob with the new Mac, or old nav with the new cycler   | Today's pull-in-only card                                                                                                  |
| Task-merge card                                            | No chords, hints, or hover buttons                                                                                         |
| Walk landing gesture                                       | One advance, never onto a row resolved in the same commit                                                                  |
| `project_finished` `^prj`, `Ctrl+X`, ↵                     | The project finishes in the same gesture as its last task, and the copy reads `✓ Finishes ‹P›`                             |

## Non-goals

- The in-card "Newly available" cascade section (v2).
- Cancel on muted, still-blocked rows.
- Verdicts in other pickers: the Mac Move card, the `:` picker, and the `/` picker. The
  names are chosen so they can adopt the vocabulary later.
- Take-back on the Mac or the CLI.
- Work Log prompts.
- Any automatic close inferred from text.
- Changing `[unblocked::]` handling on close paths.

## Risks

- **Four engines, one contract.**
  - Mitigation: the doc table plus a byte-shared `vectors.json`, consumed by Rust and
    the cycler.
  - The presenters (nav, Mac) never compute eligibility or copy.
- **Fragment caps in nav and cycler.**
  - Mitigation: new fragments from the start; never squeeze code into files near the
    cap.
- **Deadlock in the cycler's verdict step.**
  - Mitigation: only `…Now` functions. A test runs a verdict step with recovery under
    the queue.
- **Mixed versions during rollout.**
  - Every phase ships safely on its own: descriptors alone are inert, nav v4 without a
    cycler is inert, and the Mac gates on `pull_in.close.supported`.
  - `bob plugins sync` after each plugin phase is safe.
- **Per-row simulation cost in capture previews.**
  - The engine phase measures it and bounds it by reusing the batch dependency context.
