---
tier: epic
title: Successor Offers — pick which unblocked tasks to pull in after a close
goal: 'Closing a task that unblocks other tasks (in Obsidian, bob capture, or Bob
  Mac Capture) still recovers everything immediately, but no successor Task Link is
  created automatically. Instead one grouped picker per gesture offers the direct
  dependents, every available task of a project whose ^prj the close unblocked (the
  sase.md ^bar → sase_foo.md case), and the emptied-project / parent-project ^prj
  pull-ins, with no 5-successor limit. Only the picked tasks are linked, raised to
  Next, and stamped fresh.

  '
phases:
- id: contract
  title: Successor Offers contract, vectors, and docs
  depends_on: []
  size: medium
  description: 'contract: rewrite task-dependencies §12 as Successor Offers (offer
    ladder, project offers, pull-in offers, breaker deletion, selection, pick writes,
    keys, JSON, CLI flags, nav picker API), add the SO vector table, retire SL17/PP15/PDC2,
    and update capture/plan/freshness docs plus the README, getting-started, and hooks
    mentions.'
- id: rust_offers
  title: Rust planner emits offers and applies a selection
  depends_on:
  - contract
  size: large
  description: 'rust_offers: change plan_successors, project_close, and project_pull_in
    so every would-be link becomes a keyed offer, project residents are offered, the
    breaker is deleted, unplanned predecessors use the implicit destination, and a
    selection input drives link, mint, Next, and freshness writes; port the SO vectors
    as unit tests.'
- id: rust_capture
  title: bob capture --pull-in, offer JSON, and the Mac guard
  depends_on:
  - rust_offers
  size: large
  description: 'rust_capture: add -p/--pull-in and the hidden --require-pull-in-choice
    to bob capture, thread the selection through every close path and the batch net
    pass, emit offer/pull_in JSON and numbered human rows, and cover the sase_foo
    case and the SO vectors with integration tests.'
- id: cycler_offers
  title: task-status-cycler plans offers, awaits the picker, then applies
  depends_on:
  - contract
  size: large
  description: 'cycler_offers: port the offer rule to the cycler, delete SUCCESSOR_BREAKER_LIMIT,
    await nav''s picker between plan and apply, re-plan with the selection against
    fresh text, stamp picks, guard the editor insertion branch, extend the reopen
    receipt, and fall back to a no-link notice without nav.'
- id: nav_picker
  title: Navigation hotkeys successor picker modal and walk integration
  depends_on:
  - contract
  size: large
  description: 'nav_picker: add the api.successors v1 multi-select picker (groups,
    muted rows, filter, Tab/Ctrl+A marks, double-press guard, plan meter), hold the
    review-walk gesture through the picker so the walk advances once after it settles,
    and update the unblocked notice copy.'
- id: obsidian_destination
  title: Per-row destination override in the Obsidian picker
  depends_on:
  - cycler_offers
  - nav_picker
  size: medium
  description: 'obsidian_destination: expose block-id-prompt''s Pomodoro link picker
    as a versioned api, chain it from the successor picker''s Ctrl+Shift+Enter to
    retarget rows, and make the cycler insert picks at an overridden existing or new
    entry.'
- id: mac_choice
  title: Bob Mac Capture in-panel pull-in choice step
  depends_on:
  - rust_capture
  size: large
  description: 'mac_choice: decode offer/pull_in fields, number offer rows on the
    preview cards, add a keyboard multi-select choice step on Return, pass --pull-in
    on submit, use the --require-pull-in-choice race guard, fix missing copy, and
    verify on macOS CI.'
- id: decision_record
  title: Decision record, supersession marks, and epic acceptance
  depends_on:
  - rust_capture
  - cycler_offers
  - nav_picker
  - obsidian_destination
  - mac_choice
  size: small
  description: 'decision_record: record the accepted Successor Offers decision, mark
    the records it partly supersedes, optionally add the glossary term, re-run the
    cross-repo acceptance checks, and record the deferred follow-ups.'
decisions:
  memory_offers_record:
    ask: Record "Unblocked tasks are offered, never auto-linked" and mark the two
      records it partly supersedes?
    memory:
    - decisions
    - decisions:unblocked-tasks-are-offered
    default: true
    corrects: '`^prj` is pulled in as Next (`docs/task-dependencies.md` §12.10 trigger
      (c)).'
    answer: true
  memory_glossary_offer:
    ask: Add a "Successor Offer" glossary term?
    memory:
    - glossary
    default: false
    answer: true
proposed_by: bbugyi200.apollo.research.11.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 09:48:08
status: done
bead_id: bob-cli-7c
---

- **PROMPT:** [prompts/202610/successor_offers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/successor_offers.md)
- **BEAD:** [bob-cli-7c](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7c/README.md)

# Plan: Successor Offers — pick which unblocked tasks to pull in after a close

## Context

Bryan wants closes that unblock other tasks to stop auto-adding successor Task Links to
the current / next Pomodoro and to ask which unblocked tasks, if any, to pull in. The
design source is the consolidated research report
`research:202610/successor_pull_in_offers/successor_pull_in_offers.md` (read it with
`sase artifact read research:202610/successor_pull_in_offers/successor_pull_in_offers.md "<reason>"`).
**Bryan accepted every requirement in it (A1–A12) and every default in its "Open
questions" list.** This plan is binding where it is more specific than the report.

Verified facts that shape the work:

- **Two engines, one rule.** Rust: `src/native/task_complete/successors.rs`
  (`plan_successors`, ~1620 lines), `project_close.rs` (`recover_project_members`),
  `project_pull_in.rs` (§12.10), called only from `src/native/capture/task_complete.rs`
  (`!`) and `src/native/capture/pomodoro_close.rs` (`=x…` closes, three call sites),
  with the batch net pass in `capture/plan.rs` and JSON/human output in
  `capture/output.rs`. JS: bob-plugins `task-status-cycler/src/085-successor-plan.js`
  (`planSuccessors`), `080`/`088` (project member recovery), `089-successor-project.js`
  (pull-ins + `applySuccessorVerdicts`), mixins `135`/`136`/`137`
  (`planAndApplySuccessorsNow`, the plan → apply seam), all funnelled through
  `130-plugin-references.js` `finalizeClosedTasks`.
- **The breaker.** Rust `MAX_SUCCESSORS_PER_GESTURE = 5` (`successors.rs:54`, applied at
  `:788-804`); JS `SUCCESSOR_BREAKER_LIMIT = 5` (`085:7`, applied at `089:890`,
  `:956-961`). It is all-or-nothing and fired on a real close on 2026-10-13 (11
  dependents, none linked; Bryan hand-linked 2, one into a different session).
- **Why `^bar` did not pull in `sase_foo.md`.** By design: the residents are not
  dependents of `^bar` (no `[dependsOn::]`), they recovered through PROJECT RECOVERY as
  `project_member`, which is recover-only (`project_close.rs:224-292` downgrades even a
  resident's own Successor verdict; JS `080:256-284` only considers `[?]` residents),
  and `decisions:unblocked-tasks-resurface-as-ticklers` rejected "successor-linking
  residents". **Project offers must therefore be sourced by enumerating the unblocked
  project's note, not from dependents.**
- **Bob Mac Capture needs custom UI but no custom mechanism (verified).**
  `completeSubmit` hides the panel on success (`CapturePanelModel.swift:5213-5282`),
  `CapturePickerState` is single-select, notifications cannot host a list, and nothing
  in the app does multi-select. The app stays a thin client
  (`decisions:mac-capture-is-a-thin-client`): bob computes offers and does the write;
  the app adds an in-panel choice step before submit and passes the picks to the one
  atomic `bob capture`.
- **Freshness stamp helpers exist in both engines.** Rust
  `freshness::placement::stamp_fresh` (clears `[keeps::]` and `[unblocked::]`); JS
  `applyFreshStampLine` (`010-core.js:136-145`) with `getFreshnessStampLine`
  (`190-plugin-targets.js:428-450`).
- **Vectors are Markdown tables** in `docs/task-dependencies.md` (SL §11.6, PP §12.11,
  PDC §13.16) that both engines port by hand, citing the vector id in a comment.

## Shared design (binding for every phase)

### What stays automatic

Every managed Done gesture still performs all recovery immediately, exactly as today:
Blocked → Ready (derived rank), `#prj_is_blocked` stripped, `[unblocked::]` written per
`docs/freshness.md` §5's WRITE rule, and `#hide` removed from an emptied project's
`^prj` — **whether or not anything is picked**. Picking nothing never undoes an unblock;
the NEW and TICKLER review tiers resurface unpicked work.

### Offers: one set, three groups

What used to be auto-linked becomes an **offer**. One gesture produces one offer set,
grouped:

1. **`hand_off`** — direct dependents this gesture fully unblocked (today's SUCCESSOR
   verdicts).
2. **`project`** — for each project N whose `^prj` R this gesture recovered (R has no
   own or inherited reason left): every **open resident T of N**, `[?]` **or** `[ ]`,
   including `#ref` and inbox tasks, evaluated on the staged post-gesture text:
   - own open prerequisite or future date → still blocked: a **muted, unselectable row**
     with its reason;
   - `#hide` → recover only, folded into a count line;
   - live-linked today → recover to derived rank, folded into a count line;
   - otherwise → recover **and** offer.
   - R itself is the **group header, not a choice**, when N still has open work.
   - Descendant (sub-)projects are untouched (`close_recovers_subprojects = no`, PDC5);
     the group shows one muted line counting N's open sub-projects from R's
     `🧩 **Sub-projects:**` marker ("N sub-projects stay blocked until the next
     reconcile").
3. **`project_finished`** — §12.10 pull-ins (triggers a/b/c: emptied project, `^prj`
   dependency, parent project). Steps 1–4 are unchanged (`#hide` removal included); the
   link step becomes an offer. An unpicked `^prj` stays at its recovered rank.

**Dedup.** One row per task identity; causes are merged. A resident that is also a
direct dependent appears once, in `hand_off`. In a multi-item capture draft, a later
item that links, completes, re-blocks, or explicitly drops (`=x~K`) a task removes its
offer; a later item that adds open work to N removes N's `project_finished` offer.

### Ladder changes (both engines)

The §12.2 ladder is unchanged through still-blocked, `^prj`, `#hide`, and
`already_planned`. Then:

- **No anchor (predecessor not planned today)** → no longer a reason. Offer with the
  **implicit destination**: today's single open timed entry, else the first open entry
  in document order (the same resolution as the plain `@route:id` Ensure-Next form). If
  that resolution fails (no day file, no `## Pomodoros`, no open entry, or several open
  timed entries) → recover with `not_linked: no_destination` and no offer.
- **`plan.link_unblocked: false`** → recover only, `not_linked: disabled`, no offers
  (the config keeps its boolean spelling; `true`, the default, now means "offer").
- **Cancelled closes (cycler)** → recover only, never offer (unchanged).
- **Otherwise** → OFFER with destination = the §12.3 anchor (slot / closing / inherit)
  for hand-offs and pull-ins; for project residents, the earliest anchor among
  `preds(R)` (the tasks that unblocked R; in closing mode the closing target); falling
  back to the implicit destination.
- **Breaker deleted** in both engines, with SL17, PP15, and the breaker notice tests. No
  replacement cap: selection is explicit, and large lists get grouping and a filter.
  Keep the unrelated embedded-completion safeguards (25 levels / 250 targets).

### Selection and pick writes

Selection is an **input to each engine's existing single write pass** (plan → choose →
apply). Each picked offer behaves like a manual link (Ctrl+Shift+Enter):

- status `' '`/`'?'` → `'*'` (`'*'`, `'/'` unchanged; a Pending task is never demoted);
- mint a missing `^block-id` (existing deterministic mint, §12.4);
- strip `#prj_is_blocked`; stamp `[fresh:: today]`, clearing `[unblocked::]` and
  `[keeps::]` (Rust `stamp_fresh`; JS `applyFreshStampLine` + `getFreshnessStampLine`).
  This changes §12.4's "Freshness. Never touched" **for picked rows only**;
- insert `<indent>- [[link]]` at its destination (§12.3 rules; implicit destination
  inserts as an entry child, one tab, after existing children, replacing a lone stub).

Each unpicked offer recovers exactly as today's recover rule (including the
`[unblocked::]` WRITE rule) and reports `not_linked: not_selected`. Invariants: no
recursion (picking D never completes D); **no minting for unpicked rows**; a picked row
that became Blocked or was linked meanwhile is refused per row and never satisfied by
linking its prerequisites instead.

### Offer keys

Opaque, stable identity strings:

- row with a block ID → `<note_path>#^<block_id>`;
- otherwise → `<note_path>@<first 8 hex of SHA-256 of note_tasks::clean_description>`,
  and the k-th (k ≥ 2) row in the same note with the same digest, in document order,
  gets a `~k` suffix.

Rust pins this format (the Mac and the CLI round-trip it). The cycler may use any
deterministic synchronous digest in the same shape, because its keys never leave one
Obsidian pass; it must never key by line number.

### When a picker appears

Only on **managed Done gestures** (§12.1's table: cycler Ctrl+Enter variants, walk
Ctrl+Enter / `completeTaskAtCursor`, Pomodoro-line closes, Alt+]/Alt+[ into Done;
capture `!`, `=x…` closes) and **only when the offer set is non-empty**. Cancel, reopen,
raw checkbox clicks, reconcile, hooks, and `=*` parks never prompt. A batch gesture
(Pomodoro line, counted Alt+], multi-item capture draft) gets **one** picker over the
union of its offers. **Nothing is pre-checked**; ↵ applies the marked rows, or the
highlighted row when none are marked; Esc means none. `bob capture` **never prompts and
never auto-links**: without a pick flag it recovers and reports numbered offers.

### Transitional states (acceptable)

After `rust_offers`, capture reports offers as `not_selected` until `rust_capture` adds
the flag. After `cycler_offers` but before `nav_picker` is deployed, Obsidian lists
offers as "not pulled in". Both only remove automation, which is the requested
direction.

## Phase: contract

Bob-cli docs only; no code. Write the contract so the Rust, cycler, nav, and Mac phases
can work in parallel from it. If anything in this plan turns out to conflict with the
code, fix the contract here and say so in the phase notes.

1. **`docs/task-dependencies.md` §12** — rename "Successor links" to "Successor offers"
   and rewrite:
   - intro and terms (offer, pick, selection, groups, destination, key);
   - §12.1: prompts only on managed Done gestures with a non-empty offer set;
   - §12.2: the ladder changes, PROJECT OFFERS replacing PROJECT RECOVERY (keep PDC3
     semantics for residents with own reasons), DEDUP, SELECTION, the breaker deletion;
   - §12.3: implicit destination; resident destination = earliest anchor of `preds(R)`;
   - §12.4: pick writes (freshness stamp for picks only, no mint for unpicked rows);
   - §12.5: reporting — on `unblocked[]` rows an additive `offer` object (`key`, `n`,
     `group`, `project` for the project group, `destination` {day file, entry name,
     entry role such as slot/closing/implicit/created, `fallback`}, `selected`,
     `block_id_needed`); offered rows are emitted even when the status did not change (a
     stale `[ ]` resident); muted resident rows in `still_blocked[]` with their project;
     folded counts (hidden, already planned, sub-projects) per project group; a
     top-level `pull_in` object present on **every** successful capture JSON (dry-run
     too) with `supported: true`, the offer count, the choice made, and the picked
     count; new `not_linked` values `not_selected` and `no_destination`; `breaker` and
     `not_planned_today` are never emitted again (clients keep their copy for older
     bobs). Remove "There is no new subcommand and no new CLI option" and fix the
     `"not_linked": null` drift (serde omits the key). Dry-run JSON still equals
     real-run JSON with the same flags;
   - §12.6: copy for offer rows, group headers, "not pulled in", "no open Pomodoro",
     "project member", the pulled-in summary; drop breaker copy for new output;
   - §12.7: performance (project offers read only N's already-staged text plus the
     dependents snapshot the gate already allows);
   - §12.8: kill switch meaning; §12.9: undo (one pick: Alt+N; Obsidian same-day reopen
     receipt takes back picked links and restores the pre-pass lines it owns, now
     including picked rows' freshness fields; Rust: no take-back, as today);
   - §12.10: steps 5–9 (link step → `project_finished` offer; `#hide` removal
     regardless); fix the claim that the Mac needs no change.
   - New subsection: **Obsidian picker API** — nav `api.successors`
     `{version: 1, pick(model) → Promise<{picks: [{key, destination?}]} | null>}`; never
     rejects; `null` = none; the model shape (title, predecessor list, groups with
     header, rows {key, text, notePath, status, destination label, needsBlockId,
     selectable, mutedReason}, folded count lines, optional plan budget `{links, max}`);
     one picker at a time; `destination` override shape reserved for
     `obsidian_destination`
     (`{kind: "existing", entryLine, entryText} | {kind: "new", name}`).
   - New subsection: **CLI selection** — `-p, --pull-in <PICKS>` where PICKS is `all`,
     `none`, or a comma list of offer numbers and/or keys; numbers index the numbered
     offers the same command reports without a selection; an unknown, stale, or
     out-of-range pick is a teaching error that writes nothing and lists current offers;
     `all`/`none` with zero offers is a no-op. Hidden internal
     `--require-pull-in-choice`: when offers exist and no `--pull-in` was given, write
     nothing and fail with
     `{"ok": false, "code": "pull_in_choice_required", "error": "...", "pull_in": {...offers...}}`
     (same exit status and shape conventions as the `plan_theme_cap_exceeded` refusal).
     Human output: numbered `offer N` rows under group headers; a dry-run hint shows
     `-p 1,2 · -p all`; a real run without a pick says the offers were not pulled in and
     names Obsidian / Bob Mac Capture / Ctrl+Shift+Enter as the way to link later.
2. **Vectors.** State in the §11.6 SL and §12.11 PP preambles that rows expecting a link
   now mean "offered; linked when picked" (tests pass a select-all selection). Retire
   SL17 and PP15 (keep the ids, marked retired) and replace PDC2's expectation. Add a
   new **SO — successor offer vectors** table covering at least: the `^bar` → `sase_foo`
   case (two residents offered under the project group, `^prj` as header, pick one →
   minted id, `[*]`, `[fresh:: today]`, `[unblocked::]` cleared, linked under the
   predecessor's entry); an 11-way fan-out with zero pre-checked; pick all / none /
   subset / stale key; a resident that is also a direct dependent (one row, `hand_off`);
   a stale `[ ]` resident offered; muted still-blocked resident; hidden and
   already-planned residents folded; `=x~K` drop wins; later-item invalidation of an
   offer; unplanned predecessor with and without an open Pomodoro (and with two open
   timed entries → `no_destination`); emptied-project and parent-project pull-ins
   (`#hide` removed when unpicked, linked only when picked); `link_unblocked: false`;
   freshness bytes of a pick; walk advance-once and double-Enter guard (cycler/nav
   only); per-row destination override (Obsidian only).
3. **`docs/capture.md`** — the `!` completion step 6, the Pomodoro-close sections, the
   JSON contract (`offer`, `pull_in`, `not_linked` list including the missing
   `project_member`, `not_selected`, `no_destination`; the human suffix list), the
   refusal note ("`code` appears only on this refusal" now has two codes), the
   command-line options list (`-p, --pull-in`), and examples.
4. **`docs/plan.md`** — `link_unblocked` comment and meaning (offer vs. recover only).
5. **`docs/freshness.md`** §5 — picks stamp like manual links; unpicked recoveries keep
   the WRITE rule.
6. **Other mentions of auto-linking** — `README.md` (~198-203),
   `docs/getting-started.md` (~157-162), `docs/task-status-hooks.md` (~439-441,
   ~473-475); fix the doc drift that names nav's `planTargetTaskUpdate` (it lives in
   block-id-prompt).

Verify with the repo's doc/format checks (e.g. `just` targets or the markdown formatter
the repo uses) so the docs stay formatted.

## Phase: rust_offers

Bob-cli Rust planner. Keep capture compiling by passing an empty selection from the
existing call sites; `rust_capture` adds the flag.

1. **Selection input.** Add a selection type to `SuccessorInput` (none / all / set of
   keys) and an offer description to verdicts (group, project, destination, key,
   `block_id_needed`, picked).
2. **`successors.rs`.**
   - Delete `MAX_SUCCESSORS_PER_GESTURE` and the breaker block.
   - Replace `Recover("not_planned_today")` with an offer at the implicit destination,
     or `Recover("no_destination")` when the implicit resolution fails. Reuse the
     existing implicit current/next open-entry resolver used by the `@route` plain form
     rather than writing a new one.
   - SUCCESSOR verdicts become offers; picked offers keep today's link path (placement,
     `insert_successor_links`, `successor_new_status`, mint in placement order) **plus**
     the freshness stamp via `freshness::placement::stamp_fresh`; unpicked offers take
     the recover path with `not_selected` and the `[unblocked::]` WRITE rule.
   - Mint only for picked rows.
   - Emit rows for every offer (even when status is unchanged) and populate the offer
     metadata needed by `rust_capture`'s JSON.
3. **`project_close.rs`.** Replace `recover_project_members`' recover-only cascade with
   project offers: enumerate every open resident (`[?]` and `[ ]`; drop the
   `task.status != '?'` filter), classify per the shared design, produce muted
   still-blocked rows and folded counts (hidden, already planned, open sub-projects from
   R's marker), merge a resident's own direct-dependent verdict into one `hand_off` row
   instead of downgrading it to `project_member`, and compute the resident destination
   as `earliest_anchor(preds(R))` (pass the anchors in; `SlotAnchor` internals stay
   private to `successors.rs`). Keep `#prj_is_blocked` stripping and PDC3 untouched.
4. **`project_pull_in.rs`.** Step 9 becomes a `project_finished` offer; steps 1–4 and
   `#hide` removal stay unconditional; an unpicked pull-in recovers per the recover
   rank.
5. **Keys.** Implement the offer key format (block id, else SHA-256 digest of the clean
   description via the existing `sha2` dependency, `~k` for duplicates).
6. **Tests.** Update `successor_tests.rs`, `project_pull_in_tests.rs`, and
   `recovery_tests.rs` so link expectations run with a select-all selection; delete the
   SL17/PP15 breaker tests; port the SO vectors at planner level (cite ids). CLI
   integration tests under `tests/cli/` that asserted links now assert the offer
   (`not_linked: not_selected`, no link, no mint), because the CLI has no pick flag yet;
   `rust_capture` restores those link assertions with `--pull-in all`.

Run `cargo fmt`, `cargo clippy` (warnings gated per the repo's lint setup), and
`cargo test` (or the repo's `just` equivalents) until green.

## Phase: rust_capture

Bob-cli capture surface.

1. **CLI** (`src/native/capture/cli.rs`, clap builder): add `-p, --pull-in <PICKS>`
   (alphabetical between `--no-ref` and `--route`; `-p` is free on `bob capture`) and a
   hidden `--require-pull-in-choice` (no short alias; internal subprocess argument). Add
   both to `CaptureRequest`. Update `tests/cli/help_options.rs` order assertions and
   assert the hidden flag is absent from help. Follow `sase/memory/cli_rules.md`.
2. **Threading.** Pass the selection into `plan_task_complete_item`, every
   `plan_close_successors` call (`pomodoro_close.rs` has three call sites), and the
   batch path. Build the draft-level net offer list in `capture/plan.rs` (extend
   `apply_successor_net_post_pass` / `surviving_successor_links` with the DEDUP and
   later-item invalidation rules) and number it in ORDER. Resolve positional picks by
   planning the draft once without a selection, mapping numbers to keys, then planning
   again with keys; key picks need one pass. A pick that matches no current offer is a
   write-free teaching error.
3. **Guard.** With `--require-pull-in-choice` and no `--pull-in`, return the
   `pull_in_choice_required` refusal with the offers when any exist, writing nothing.
4. **Output** (`capture/output.rs`). JSON: `offer` on `unblocked[]` rows, top-level
   `pull_in` on every success, muted resident rows, folded counts, new `not_linked`
   values. Human: numbered `offer N` rows under group headers, copy for `not_selected`
   ("not pulled in"), `no_destination`, `project_member` ("project member"), the dry-run
   vs. real-run hints, and a pulled-in summary on the close line. Keep task-block
   `reason = "unblocked"` for inserted pick links.
5. **Integration tests** (`tests/cli/capture/`): the `^bar` → `sase_foo` case end to end
   (dry-run offers both residents under the project group with `^prj` as header; `-p 1`
   mints, raises, stamps, and links under the predecessor's entry); 11- and 20-way
   fan-outs with no cap; pick all / none / subset / stale key / out-of-range number;
   `=x~K` drop wins; later-item invalidation; unplanned predecessor with and without an
   open Pomodoro; emptied-project and parent (trigger c) pull-ins (`#hide` removed when
   unpicked); `--require-pull-in-choice` refusal is write-free and carries offers;
   `link_unblocked: false` produces no offers; dry-run JSON equals real-run JSON with
   the same flags. Restore the pre-existing link expectations (SL1/2/9/10/14/16/18/19,
   PP, PDC) by passing `--pull-in all`.

Run the full Rust check suite until green.

## Phase: cycler_offers

Bob-plugins `task-status-cycler` (open the linked repo with `/sase_repo` and read its
`AGENTS.md`: edit `src/` fragments only, ≤1000 lines per fragment, `npm run build`,
`npm test`, register new test files in the root `package.json` `test` script, bump
`manifest.json` + README row, then `bob plugins sync`).

1. **Planner parity** (`085`, `080`, `088`, `089`): offers with groups, keys, and
   destinations per the contract; delete `SUCCESSOR_BREAKER_LIMIT` and the breaker
   branch in `applySuccessorVerdicts`; implicit destination / `no_destination` replacing
   `not_planned_today`; project offers enumerating every open resident (not just `[?]`
   tagged ones) with muted rows and folded counts; pull-ins become `project_finished`
   offers with unconditional `#hide` removal; `planSuccessors` accepts a `selection`
   option (null = plan offers only).
2. **Pick writes.** Status raise, mint, `#prj_is_blocked` strip, and
   `applyFreshStampLine(line, this.getFreshnessStampLine(), today)` for picked rows;
   unpicked rows keep `rewriteRecoveredTaskLine` / `markUnblockedLine` behavior.
3. **Plan → pick → apply** in `137-plugin-successor-project.js`
   `planAndApplySuccessorsNow`: plan without a selection; if offers exist and
   `api.successors?.version >= 1` (nav), build the picker model per the contract and
   `await pick(model)`; then **re-read** the daily note and affected notes, re-plan with
   the selection, and apply with the existing preimage guards; insertion planning must
   use the re-read daily text (today it uses `dailyAfter` read before planning). Without
   the nav picker: no links; the notice lists offers as "not pulled in". The picker
   result (and Esc) must flow into the existing `successorNoticeTarget: "return"` path
   so walk toasts compose.
4. **Concurrency.** Add the missing snapshot comparison to the editor branch of `135`
   `applySuccessorDailyInsertions` (today only the vault branch checks
   `insertion.snapshot`), so a daily note edited while the picker was open fails that
   row loudly instead of splicing at a stale position.
5. **Receipt** (`136`): record picked rows' pre-pass lines including freshness fields;
   same-day reopen takes back picked links and restores them.
6. **Notice copy** (`087` `successorNoticeText`): `not_selected`, `no_destination`, the
   pulled-in summary; drop the breaker variant from new output.
7. **Tests**: port the SO vectors; update `test-task-status-cycler-successors.cjs`,
   `-project-pull-in.cjs`, `-successor-wiring.cjs` (fake nav picker returning a subset,
   `null`, and absent nav; the zero-read wiring test must still pass), and
   `-successor-polish.cjs` (receipt restores freshness); add an editor-branch stale
   snapshot test.

## Phase: nav_picker

Bob-plugins `bob-navigation-hotkeys` (same repo conventions as above).

1. **API.** Add `successors: {version: 1, pick(model)}` to the frozen nav api object
   (`480-review-jump-and-nav-api.js`), with the null-plugin variant on unload
   (`490-plugin-lifecycle.js`). `pick` returns a Promise that never rejects and resolves
   `null` on Esc / click outside / unload; a second call while one is open resolves
   `null`.
2. **Modal.** Build on `FilteredPickerModal` (`200-filtered-picker-modals.js`) and the
   existing Tab-mark convention (`450-picker-keydown.js`, `400-picker-local-task.js`):
   group headers (Hand-offs / Project ‹note› / Project finished), muted unselectable
   rows with reasons, folded count lines, a filter, project groups collapsed (`+N more`)
   when the model has more than ~12 rows and expanded by typing, Tab marks, Ctrl+A marks
   the highlighted group, ↵ applies marked rows or the highlighted row, footer states
   the count ("pull in N"), destination label and "needs ^id" per row, and a plan meter
   (`plan L/max → L'/max`) when the model carries a budget — it warns, never refuses.
   **Double-press guard:** ignore Enter carrying modifiers, key repeats, and presses
   within ~350 ms of opening, so the second half of a double Ctrl+Enter never applies
   the picker. Add styles to `styles.css`.
3. **Review walk** (`536-plugin-review-advance.js`, `535`): while a successor picker
   opened during a captured walk gesture is open, hold the gesture lock (the 3000 ms
   `REVIEW_GESTURE_LOCK_MS` must not expire mid-picker and let a `]s` stale the origin
   via `reviewLandingEpoch`/`reviewGestureSeq`); after the picker settles (including
   Esc) the walk advances exactly once and the pull-in result composes into the single
   walk toast. Cover both `claimReviewWalkCtrlEnter` → cycler `completeTaskAtCursor` and
   `completeReviewChecklistRow`; the PRE/POST boundary rule is unchanged.
4. **Notice** (`285-unblocked-notice.js`): copy for `not_selected`, `no_destination`,
   `project_member`, and a pulled-in count; keep breaker copy only for older payloads.
5. **Tests**: new picker tests (keys, marks, groups, filter, collapse, guard, settle
   semantics), walk tests in `test-navigation-review-advance*.cjs`, notice tests.

## Phase: obsidian_destination

Bob-plugins `block-id-prompt`, nav, and cycler.

1. **block-id-prompt api.** Expose a versioned `api.pomodoroLinkPicker`
   `{version: 1, pick(request)}` wrapping `PomodoroLinkPickerModal`
   (`085-pomodoro-link-picker-modal.js`), resolving
   `{kind: "existing", entryLine, entryText, title} | {kind: "new", name} | null`,
   following nav's api install/unload conventions.
2. **nav picker.** Ctrl+Shift+Enter on the marked (or highlighted) rows opens that
   picker when available (hide the hint otherwise), sets each row's `destination`, shows
   it in the row (`→ NAME`), and returns it with the picks.
3. **cycler.** Honor a pick's `destination`: an existing open entry (verified by entry
   line + text preimage; fail the row loudly if it moved or closed) gets the link as an
   entry child; a new name creates a named placeholder at the same position
   block-id-prompt uses for new entries, then inserts. Record created entries in the
   receipt.
4. Tests in all three plugins, version bumps, build, `bob plugins sync`.

## Phase: mac_choice

Bob Mac Capture (open with `/sase_repo`; Swift builds and tests run only on the repo's
macOS CI workflow `.github/workflows/ci.yml` — agent hosts have no Swift toolchain, so
run `swift-format`-clean edits locally where possible and verify through CI once the
change is pushed, monitoring the run with `/sase_monitor`).

1. **Decode** (`CaptureCore/CaptureModels.swift`): `offer`, top-level `pull_in`,
   `trigger`, `hide_removed`, muted resident rows / folded counts, and the
   `pull_in_choice_required` refusal, all with `decodeIfPresent` so older bobs work.
2. **Preview.** Number offer rows on the `!` and close cards (`CapturePanelView.swift`
   `completePreviewItem`, `closePreviewItem`) with group headers and the hint
   `↩ choose what to pull in`.
3. **Choice step.** On Return, when the latest preview has offers, open an in-panel
   multi-select step that gates submit like `taskIDPrompt` / `pomodoroNamePrompt`
   (extend `inlinePromptVisible`, `clearInlinePrompts`, `clearEditorInputLockIfFree`,
   draft-change auto-cancel, and `CaptureKeyCommandRouter` priority). Keys: ↑↓ move,
   Space toggles, ⌘A marks all, ↩ submits marked rows (else the highlighted one), ⇧↩
   submits with none. **Esc returns to the draft with nothing written** (on the Mac the
   close has not happened yet).
4. **Submit** (`BobProcessClient.capture`): append `--pull-in <keys|none>` before `--`
   only when bob advertised `pull_in.supported`. **Race guard:** when submitting without
   a fresh preview carrying `pull_in`, send `--require-pull-in-choice`; a
   `pull_in_choice_required` reply opens the choice step from the returned offers. If
   bob rejects the flag as an unknown option (old bob; clap fails before any write),
   cache "unsupported" for that binary and resubmit without it.
5. **Copy fixes** (`CaptureSuccessorPresentation.swift`): `project_member` ("project
   member", currently falls through to the raw string), `not_selected` ("not pulled
   in"), `no_destination`; the notification line reports what was pulled in;
   still-blocked `project` reason copy.
6. **Tests and fixtures**: update `Tests/Fixtures/fake-bob` fixtures to current bob
   output (`trigger`, `hide_removed`, `offer`, `pull_in`), add a `sase_foo`-style
   fixture, the refusal fixture, and argv assertions for `--pull-in` /
   `--require-pull-in-choice`; model tests for the choice step (open, toggle, Esc keeps
   the draft, ⇧↩ none, panel hides only after success); router tests; presentation
   tests.

## Phase: decision_record

Bob-cli memory plus epic acceptance. Use `/sase_memory_write` and `/sase_memory_read`;
author only what the accepted memory decisions cover.

> [!decision] memory_offers_record

Add a new strand to the `decisions` memory web with slug `unblocked-tasks-are-offered`
("Unblocked Tasks Are Offered, Never Auto-Linked"), following the existing strand
format: applies to bob-cli, bob-plugins, Bob Mac Capture, vault; claim = recovery stays
automatic, every would-be link (hand-offs, project residents, project-finished pull-ins)
is an offer chosen in one picker per Done gesture, nothing pre-checked, no cap, picks
behave like manual links (Next, mint, freshness stamp), capture never prompts or
auto-links; rejected alternatives from the research (higher cap, auto for direct
dependents only, auto-single skip, post-commit chooser, `>1,3` grammar, Mac preview
checkboxes, never prompting in the walk, a tray); evidence = the research ref, this
plan, and the phase commits; cost and reopen conditions. Link it to
`[[decisions:unblocked-tasks-resurface-as-ticklers]]`,
`[[decisions:projects-block-from-prj-line]]` (it satisfies that record's "re-linking on
unblock" reopen condition), `[[decisions:subprojects-inherit-project-blocking]]`, and
`[[decisions:mac-capture-is-a-thin-client]]`.

Also mark `decisions/unblocked-tasks-resurface-as-ticklers` with
`metadata.status: superseded-in-part` and `superseded_by` the new record, plus a
back-link line stating that residents are now offered (the rejected "successor-linking
residents" became an explicit offer); do not edit its body otherwise.

Mark `decisions/subprojects-inherit-project-blocking` the same way: the parent `^prj` of
trigger (c) is now offered, linked only when picked.

> [!decision] memory_glossary_offer

Add a short "Successor Offer" glossary strand (aka offer, pull-in offer) defining the
term and linking the new decision record.

Then run `sase memory init`. Finally, re-run the acceptance checks (Rust suite, plugin
`npm test`, confirm the Mac CI run is green, `bob plugins sync --dry-run` shows no
drift) and record `PROPOSED FOLLOW-UP:` notes on this phase's bead for the deferred
items below.

## Out of scope (deferred follow-ups)

- A "Pull in unblocked tasks…" rescue command in Obsidian (held offers, revalidated).
- A per-row destination override on the Mac and a destination grammar for `--pull-in`
  (the JSON already carries `destination`).
- An interactive TTY picker for `bob capture`.
- Immediate sub-project (descendant) recovery on close so their tasks can be offered.
- Offering on cancel; an opt-in "auto-link a lone offer" mode.

## Epic acceptance

- 1, 6, 11, or 20 offers are all reachable with zero pre-checked, and any subset links
  exactly that subset, in Obsidian, the Mac, and `bob capture -p`.
- Esc / none: recovery stands; no minted IDs; no successor-only placeholders.
- `^bar` → `sase_foo`: both residents are offered under the project group with `^prj` as
  the header; picking one mints its ID, sets `[*]`, stamps `[fresh:: today]`, clears
  `[unblocked::]`, and links it into the predecessor's entry.
- Emptied-project and parent-project pull-ins: `#hide` comes off unpicked; the link
  happens only if picked.
- Review walk: a walk-landing close advances exactly once after the picker settles,
  honoring the PRE/POST boundary; a double Ctrl+Enter never applies the picker.
- A stale line or daily note fails that row loudly; nothing is written to the wrong
  place.
- JSON callers, pipes, agents, and old clients never prompt and never auto-link.
