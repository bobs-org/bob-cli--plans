---
tier: epic
title: 'Successor Links: a closed planned task hands its slot to the tasks it unblocks'
goal: 'When a Bob close gesture completes a task planned in today''s ledger, every
  direct dependent that this close fully unblocked is linked into the predecessor''s
  slot and becomes Next in the same write. Bob close gestures are Obsidian Ctrl+Enter,
  `bob capture` `!note:id`, and the `=x…` / `=!` Pomodoro closes. The slot is right
  after the predecessor''s Task Link, or the same-name continuation when the whole
  session closed. One grouped notice says what was linked, where, and why: a card
  in Obsidian, the live preview plus one notification line in Bob Mac Capture, and
  rows in the CLI. Capture gets faster, not slower: on apollo, plain and `=x` previews
  take ≤ 40 ms, and closing a real prerequisite takes ≤ 70 ms.'
phases:
- id: snapshot
  title: Lazy, shared, prefiltered vault snapshot for capture (bob-cli-5v)
  depends_on: []
  size: medium
  description: 'snapshot: in bob-cli, fix bead bob-cli-5v. Make DependencyContext
    lazy and resolve `!` and `&` notes from a walk-only index. Replace the full-vault
    recovery base with one batch-scoped, parallel, prefiltered dependents snapshot
    overlaid with staged text, and gate `!` recovery on `[id::]`. Output is unchanged
    apart from the documented gate edge. Ships guard tests and apollo timings.'
- id: contract
  title: Specify Successor Links once, in docs and vectors
  depends_on: []
  size: medium
  description: 'contract: in bob-cli docs, add task-dependencies §12 (rule, anchors,
    placement, writes, reporting model, copy) plus §11.6 SL and SB vectors, with SB
    pinned by a Rust unit test. Rewrite the capture.md `!`, close, JSON, and human-output
    contracts. Document `plan.link_unblocked` and nav `api.notice`.'
- id: capture_complete
  title: Successor planner and `!note:id` wiring in bob capture
  depends_on:
  - snapshot
  - contract
  size: medium
  description: 'capture_complete: in bob-cli, add the pure successor planner (graph-transition
    eligibility, anchors, ordering, breaker, minting, link form, placement) and the
    `plan.link_unblocked` config key. Wire the planner into `!note:id` before ledger
    retirement. Extend `unblocked[]`, add `still_blocked` and `unblocked_check` plus
    the top-level `day_file`. Add human rows, help text, and SL tests.'
- id: capture_close
  title: Recovery and successor links inside Pomodoro closes
  depends_on:
  - capture_complete
  size: medium
  description: 'capture_close: in bob-cli, run recovery and successor linking in every
    close that completes tasks: `=x` embeds, `=x!M`, `=!`, and `^route:id=x…`. Successors
    go into the same-name continuation, created when needed. Add `pomodoro_close.unblocked`
    and the `pomodoro_blocks` line `reason`, report the net batch result, and update
    help text and tests.'
- id: mac
  title: Successor Links in Bob Mac Capture previews and notifications
  depends_on:
  - capture_close
  size: medium
  description: 'mac: in bob-mac-capture, decode the additive successor JSON defensively.
    Render successors in the `!` and close previews: destination capsule, minted-ID
    caption, reasons, and muted still-blocked rows. Badge unblocked lines in block
    diffs, add one notification line, and fix Open Note(s). Ships real-bob fixtures,
    tests, a README update, and green macOS CI.'
- id: nav_card
  title: The Unblocked notice card and nav `api.notice`
  depends_on:
  - contract
  size: medium
  description: 'nav_card: in bob-navigation-hotkeys, add the Unblocked notice card
    in a new fragment: `is-unblock` styles on existing tokens, clickable rows, chips,
    and breaker and failure variants. Expose it as the additive `api.notice` v1. Ships
    tests, a version bump, a README update, and a sync.'
- id: cycler_engine
  title: Pure successor helpers in task-status-cycler
  depends_on:
  - contract
  size: medium
  description: 'cycler_engine: in task-status-cycler, add pure helpers that mirror
    the Rust planner: a dependents index built from the Tasks cache or from documents,
    eligibility, anchors, ordering, the breaker, the ported block-ID suggester, link
    form, placement edits, notice text, and config and today-path loaders. Ships SL
    and SB vector tests and no wiring.'
- id: cycler_wiring
  title: Recover-and-link on every Ctrl+Enter close, with one notice
  depends_on:
  - nav_card
  - cycler_engine
  size: medium
  description: 'cycler_wiring: in task-status-cycler, run the gated recover-and-link
    pass inside finalizeClosedTasks, with a closing-entry hint from Pomodoro-line
    closes. Write through open editors or through preimage-checked vault.process.
    Show the nav card, or compose the text into the walk toast and nav''s completeTaskAtCursor
    caller, and report failures. Ships guard tests, version bumps, a README update,
    and a sync.'
- id: ledger_glyph
  title: Read-time 🔓 hand-off glyph in today's ledger
  depends_on:
  - contract
  size: small
  description: 'ledger_glyph: in bob-ledger-tools, draw a faint read-time 🔓 after
    a live Task Link under today''s open Pomodoros whose task had a prerequisite completed
    today, with a tooltip naming it. It is derived from the Tasks cache and never
    stored. Ships tests, a version bump, a README update, and a sync.'
- id: cycler_polish
  title: Reopen takes successors back; Alt+] closes join the pass
  depends_on:
  - cycler_wiring
  size: medium
  description: 'cycler_polish: in task-status-cycler, add an in-memory reopen receipt:
    reopening a predecessor removes its untouched successor links and restores their
    status. Route Alt+]/Alt+[ closes through finalizeClosedTasks (bead bob-cli-3k);
    cancels recover only. Ships tests, versions, a README update, and a sync.'
- id: closeout
  title: End-to-end verification and memory
  depends_on:
  - mac
  - cycler_polish
  - ledger_glyph
  size: small
  description: 'closeout: verify end to end across the repos: CLI sandbox, plugin
    harness, deployed plugins, Mac CI, and apollo timings. Apply the accepted memory
    decisions, then summarize the shipped behavior on the epic.'
decisions:
  decision_record:
    ask: After landing, record "a closed planned task hands its slot to its successors"
      as a decisions strand?
    memory:
    - decisions
    default: false
    answer: false
  glossary_term:
    ask: After landing, add a "Successor Link" glossary term?
    memory:
    - glossary
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.0p.linker.w0
decided_by: auto
create_time: 2026-10-09 11:54:13
status: wip
bead_id: bob-cli-5w
---

- **PROMPT:** [prompts/202610/successor_links.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/successor_links.md)
- **BEAD:** [bob-cli-5w](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5w/README.md)

# Plan: Successor Links — a closed planned task hands its slot to the tasks it unblocks

## Request

Bryan wants Bob to add Task Links automatically for tasks that depend on a task he
closes from today's plan:

- The trigger is a close through Obsidian's Ctrl+Enter or `bob capture`'s `=x!` / `=!`.
- The links go into the Pomodoro the closed task was in, or into the newly created
  Pomodoro if that whole Pomodoro was closed.
- A good toast says which links were added and why. It appears in Obsidian or in Bob Mac
  Capture, depending on where the close happened.
- It must be fast. Bob Mac Capture in particular must stay blazing fast.

Bryan agreed with **every** requirement in the consolidated research report
`research:202610/unblocked_successor_links_on_close/unblocked_successor_links_on_close.md`:

- the narrowed scope;
- adjustments A1–A10;
- every default under "Open questions for Bryan":
  - the planned slot;
  - mint and report;
  - link inbox successors with a chip;
  - a breaker at more than 5;
  - the `plan.link_unblocked` kill switch;
- the P0–P5 implementation outline, including the polish items.

He asked the planner to lead the design and make it intuitive, reliable, and beautiful.
This plan turns that report into buildable phases and settles the details it left open.
The report's open details are settled as follows, with reasons inline:

- the identity used for matching;
- subtask anchors;
- the "already planned" baseline;
- the continuation search;
- the reporting rows;
- the human, card, and Mac copy;
- the minting algorithm and its parity;
- the kill switch's exact meaning;
- the shared notice model;
- the reopen receipt.

## The promise

> "When I finish something that's on today's plan, Bob puts whatever that finally
> unblocked into the same spot — or into the next session of the same name if I closed
> the whole session — and tells me."

That sentence is the design. Every rule below exists to keep it true, quiet when nothing
happened, and cheap when nothing can happen.

# Design

## Vocabulary

- **Gesture.** One Obsidian Ctrl+Enter, or one capture **item**. A multi-item draft is
  several gestures, planned in order against the staged batch.
- **Predecessor (P).** A task this gesture moved from open to Done. That covers the root
  and every embedded subtask closed with it.
- **Successor (D).** An open direct dependent of a predecessor whose **last** blocker
  this gesture removed.
- **Anchor.** Where a predecessor was planned today. It is read from today's day text
  **before** this gesture strikes or moves anything.
- **Successor link.** The plain Task Link bullet the gesture inserts for D, for example
  `\t- [[sase#^relaunch-agents]]`. It is never an embed and never annotated.
- **Live link.** An unstruck Task Link sub-bullet (plain, embed, or `#`-deferred; 🍅
  markers ignored) under an **open** `[ ]` entry of today's day file's `## Pomodoros`
  section. Struck links and links under closed entries are history, not plans.

## Trigger (A1)

Successor linking runs inside these gestures, and only when they **complete** a task:

| Gesture                                                                                                                              | Where                    | Phase            |
| ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------ | ---------------- |
| Ctrl+Enter on a Task Link (plain or embed) under any Pomodoro, on the task line in its own note, or through the fallback link branch | task-status-cycler       | cycler_wiring    |
| Ctrl+Enter on a Pomodoro line (its closed embeds)                                                                                    | task-status-cycler       | cycler_wiring    |
| nav's PRE/POST walk Ctrl+Enter (`completeTaskAtCursor`)                                                                              | task-status-cycler + nav | cycler_wiring    |
| `bob capture '!note:id'`                                                                                                             | bob-cli                  | capture_complete |
| `bob capture` closes that complete links: `=x` with embeds, `=x!M`, `=!`, `^route:id=x…`                                             | bob-cli                  | capture_close    |
| Alt+] / Alt+[ status cycling into Done                                                                                               | task-status-cycler       | cycler_polish    |

It **never** runs on any of the following:

- cancel (the nav Ctrl+Shift+P cancel, Alt+[ into Cancelled);
- reopen;
- raw checkbox clicks;
- `bob task reconcile` / hooks;
- `=*` parks;
- Ctrl+Enter on a Depends-On line.

Cancels still recover dependents exactly as today.

## The rule (one definition, two engines)

```text
C        = tasks this gesture moved open → Done (roots + closed embedded subtasks),
           read from the staged post-close text
GATE     : if no c ∈ C carries an [id::] field → stop. No dependents lookup and no extra
           reads. unblocked_check = "checked"; unblocked = still_blocked = [].
match(c) = c's [id::] value                      (see "Identity" below)

ANCHORS (computed on today's day text BEFORE this gesture's strikes, moves, or retirement,
         but AFTER any explicit link step of the same item, e.g. ^route:id=x…)
  closing    : c's link sits under the Pomodoro this gesture closes
  slot(E, b) : else the first live link to c in ledger order, under open entry E
               (b = that bullet)
  inherit    : a closed subtask with no live link of its own takes its root's anchor
  none       : otherwise (no today day file, no ## Pomodoros, or not planned today)

CANDIDATES = open tasks D (status type Todo/InProgress/OnHold: ' ', '?', '*', '/'),
             outside done/, whose [dependsOn::] contains match(c) for some c ∈ C

FOR EACH D, in this order:
  post_open = D's open prerequisites in the staged post-close snapshot
              (exactly task_dependency_states)
  1. post_open ≠ ∅                → still_blocked{reason: waits_on, waits_on: |post_open|}
  2. D.scheduled > today          → still_blocked{reason: scheduled, scheduled}
  3. D's block id is prj          → recover; not_linked: project_task
  4. D carries #hide              → recover; not_linked: hidden
  5. D had a live link before this gesture
                                  → recover; not_linked: already_planned
  6. no predecessor of D in C has an anchor
                                  → recover; not_linked: not_planned_today
  7. plan.link_unblocked is false → recover; not_linked: disabled
  8. otherwise                    → SUCCESSOR, anchored at the earliest (ledger order)
                                    anchor among D's predecessors in C

BREAKER  : more than 5 successors in one gesture → link none; each becomes
           recover + not_linked: breaker
ORDER    : successors by anchor ledger position, then note path, then line
recover  : '?' → derived rank on the POST-gesture day text: '*' if D has a live link
           there, else ' '. ' ', '*', '/' are unchanged.
SUCCESSOR: '?' or ' ' → '*'; '*' and '/' are unchanged. Mint a missing ^block-id, then
           insert the successor link (Placement).
```

Notes that settle the report's open points:

- **Identity.** Matching uses only c's `[id::]` value. That is the one identity under
  which `task_dependency_states` resolves blocking: its identity map is built from
  explicit `[id::]` fields. A dependent that names only the canonical `note__block-id`
  of a target **without** `[id::]` was never blocked by that target under the derived
  rule, so it cannot be "unblocked" by closing it. The hooks heal that rare stale case
  on their next run, because they write the target's `[id::]`.
  - This drops the canonical-id match the Rust `!` recovery uses today. The JS cycler
    already matches only `[id::]` values.
  - The gate is therefore exact: about 96% of open tasks carry no `[id::]`, and closing
    one of them can never unblock anyone.
- **Graph transition, not checkbox.** Eligibility depends on `post_open` and on D
  depending on something this gesture closed. Every c in C was open before the gesture,
  so "pre_open ∩ C ≠ ∅" holds automatically. A stale `[ ]` dependent therefore qualifies
  (SL15), and a `[?]` with another open prerequisite never does.
- **The already-planned baseline** is the day text **before** the gesture. Three
  consequences:
  - A dependent whose link this same close carries forward stays planned and is not
    duplicated.
  - A dependent whose link this same close **drops** (`=x~K`) is never re-linked. The
    drop wins, and the dependent recovers to Ready.
  - The recovered rank is derived from the **post**-gesture text, because that is what
    the hooks will see.
- **Rows report what the gesture did.** A dependent appears in `unblocked[]` only if
  this gesture changed its status or linked it. An already-Next dependent that is
  already planned produces no row. `still_blocked[]` lists every open dependent of C
  that stays blocked.
- **No recursion.** Linking D is not completing D, so D's own dependents wait (SL13).
- **No managed-line guard in v1.** The field is the single identity, exactly as recovery
  and the hooks use it. Staleness is the same window Blocked derivation already has.

## Placement

The inserted line is always `<indent>- <link>`. The indent is the anchor bullet's indent
for slot anchors, and one tab for entry children. Nothing else goes on the line: prose
would stop it counting as a Task Link.

**Slot anchors** (`slot(E, b)`): insert immediately after bullet `b`'s subtree (`b` plus
its deeper-indented children), in successor order.

- **Rust (`!`).** Insert **before** retirement runs. Retirement then strikes `b` in
  place, or moves `b`'s subtree into the running entry, while the successor stays at the
  vacated position in E. Because E still has a child, it is never removed as an emptied
  placeholder (SL2).
- **Obsidian.** Insert before the cycler strikes the cursor link. The insertion is below
  the cursor line, so the strike target is unaffected.

**Closing anchors.** The target entry is the first that applies:

1. the continuation this close created (`next_pomodoro.created`);
2. else the first open entry **after** the closed one with the same non-empty name;
3. else a new placeholder, `- [ ] () — NAME` (or `- [ ] ()` when the closed entry was
   unnamed). It goes immediately after the closed entry's sub-bullet range, exactly
   where the close would have created its continuation. Successors count as carried.

Successors are appended after the target's existing children; a lone `\t- ` stub is
replaced. `link.entry_created` is true when the entry did not exist before this gesture,
in case 1 or 3.

**Next up.** `link.next_up` is true when, after the gesture, the target entry is the
first open untimed placeholder in the ledger: the one a bare `=` would start. This is
the accepted consequence the report describes: a created continuation becomes next up,
exactly like carried work today. The notice always says so, so it is never a surprise.
Bryan can still start another session with `=#NAME` or move the bullet with
Ctrl+Shift+M.

### Placement examples on today's real ledger

**(a) Ctrl+Enter on `[[sase#^fix-apollo]]` in FIX.** The link is struck in place and the
successor takes the next line. `sase.md:82` goes `[?]` → `[*]`.

```markdown
- [ ] () — FIX
  - [[sase_agent_history#^launch-agent-archive-view]]
  - [[sase#^just-install-venv]]
  - ~~[[sase#^fix-apollo]]~~
  - [[sase#^relaunch-failed-agents]]
  - [[sase#^fix-muse-reply]]
```

**(b) `bob capture '!sase:fix-apollo'` while BOB runs.** Retirement moves the struck
link into BOB, as it does today, and the successor keeps FIX's vacated slot.

**(c) `bob capture '^sase:fix-apollo=!'`.** The item links P into BOB, then closes BOB,
completing everything. Nothing is carried and FIX follows, so today no continuation
would be created. With a successor, one is created:

```markdown
- [x] (**1000-1020** [t:: 20m]) — BOB
  - ~~[[bob#^better-refs]]~~
  - ~~[[sase#^fix-apollo]]~~
- [ ] () — BOB
  - [[sase#^relaunch-failed-agents]]
- [ ] () — FIX
```

**(d) Fan-out.** Closing a task with three dependents inserts all three after it. The
plan meter may read `links 12/10` in red: a lint, not a refusal, as with every other
writer.

## Writes and side effects

Each gesture makes one write set: one staged batch in capture, one planned pass in
Obsidian.

- **Successor's note.** Its status changes per the rule. A missing `^block-id` is
  appended as ` ^<mint>` at the end of the task line.
- **Day file.** Successor bullets are inserted, plus the created placeholder when one is
  needed.
- **Freshness.** Never touched: no `[fresh::]` stamp, and an existing stamp is kept
  byte-for-byte. Automation never stamps (glossary _Task Freshness_). An unstamped
  successor surfaces in tomorrow's NEXT review, which is correct.
- **Schedule.** Never touched: no `scheduled` change and no Schedule Log line. Eligible
  successors have no future date by definition.
- **Fields.** No `[id::]` / `[dependsOn::]` writes; the hooks own those.
- **Do not call `plan_task_link`** (Rust) or nav's `planTargetTaskUpdate`: both stamp
  freshness and retire schedules. Use the lower-level status writer
  (`set_task_line_status`) and the new placement helper.

**Block-ID minting (A5).** `mint(D)` is the first entry of bob's
`capture_block_ids::suggest_ids_with_used(description, '^', used)`.

- `description` is `note_tasks::clean_description` of D's body: the trailing block ID,
  inline fields, and the Tasks global filter are removed, and whitespace is collapsed.
- `used` is every block ID in D's note (staged text), plus IDs this gesture already
  minted in that note.
- When the suggester returns nothing, mint `task`, `task-2`, `task-3`… (first free).
- The ID is deterministic and readable. Capture and Ctrl+Enter must mint byte-identical
  IDs, so the JS cycler ports the suggester and §11.6 SB vectors pin both.

**Link form.** Same rules as dependency links (`docs/task-dependencies.md` §3):

1. `[[basename#^id]]` when the basename is unique;
2. otherwise `[[dir/note#^id]]`.

The one exception: a successor that lives in the day file itself still names the note
(`[[20261009#^id]]`), never the bare `[[#^id]]`. Rust uses
`task_dependencies::format::canonical_link` with the same `NoteIndex` capture uses for
`&` dependency links. `contract` documents the exact file set that counts for
uniqueness, and the JS port counts the same set.

## Kill switch (A9)

`plan.link_unblocked: true` lives in `~/.config/bob/config.yml` and is read by Rust and
the cycler. Setting it to `false` turns off **linking and minting only**:

- close-time recovery still runs and still uses the derived rank;
- each unblocked row reports `not_linked: "disabled"`.

Recovery is the honest half of the feature and costs nothing extra once the dependents
are found. An invalid value falls back to `true` and surfaces through the existing
invalid-plan-config warning paths (`docs/plan.md` "Invalid values").

## `bob capture` result contract

The contract is additive under schema version 1, so old clients keep working. Dry-run
JSON equals real-run JSON except for `dry_run`. There is no new subcommand and no new
CLI option.

Both `task_complete` and `pomodoro_close` objects carry the same three fields:

```json
"unblocked": [{
  "note_path": "sase.md", "block_id": "relaunch-failed-agents", "line": 82,
  "text": "Re-launch all failed agents on apollo!",
  "previous_status_symbol": "?", "previous_status_name": "Blocked",
  "status_symbol": "*", "status_name": "Next",
  "inbox": false,
  "unblocked_by": [{"note_path": "sase.md", "block_id": "fix-apollo",
                    "text": "Fix apollo machine!"}],
  "link": {"day_file": "2026/20261009.md", "entry_name": "FIX", "entry_line": 37,
           "entry_created": false, "next_up": false, "line": 40,
           "block_link": "[[sase#^relaunch-failed-agents]]",
           "block_id_created": true},
  "not_linked": null
}],
"still_blocked": [{"note_path": "sase_agents_repo.md", "block_id": "badges", "line": 21,
  "text": "Start adding sase--<name> badges", "status_symbol": "?",
  "reason": "scheduled", "waits_on": 0, "scheduled": "2026-10-13"}],
"unblocked_check": "checked"
```

**`unblocked[]` rows**

- The existing fields keep their meaning; `unblocked` stays on `task_complete` and is
  new on `pomodoro_close`.
- `block_id` holds the minted ID when one was minted, and `""` when D has none and none
  was minted.
- `link` is non-null exactly when this gesture linked D.
  - Line numbers are 1-based in the day text after this item.
  - `day_file` is vault-relative.
  - `entry_name` is `""` when the entry is unnamed.
- `not_linked` is one of `already_planned`, `not_planned_today`, `breaker`,
  `project_task`, `hidden`, `disabled`.
  - The cycler-only model adds `failed` and `cancelled`; Rust never emits them.

**`still_blocked[]` rows**

- `reason` is `waits_on` or `scheduled`.
- `waits_on` counts the open prerequisites left.
- `scheduled` is set only for `scheduled`, otherwise `null`.

**`unblocked_check`**

- `"checked"`: the lookup ran, or the gate proved it unnecessary.
- `"unavailable"`: the dependents snapshot could not be built. The close still succeeds
  and nothing is linked.
- A missing key means an older `bob`.

**Elsewhere in the result**

- **`task_complete` top-level `day_file`.** Set to the absolute day file path whenever
  the gesture changed the day file (`ledger` present or a successor linked). Previously
  it was always `null`. This makes Bob Mac Capture's Open Note(s) include the daily note
  (see `mac`).
- **`pomodoro_close.next_pomodoro`** names the successor's continuation when the
  successor logic created it.
- **`pomodoro_blocks[].lines[].reason`.** Optional `"unblocked"` on added lines that are
  surviving successor links.
- **Net batch reporting (SL20).** After all items are planned, a successor whose link no
  longer survives live in the final staged day text, or whose task is no longer open, is
  dropped from its item's `unblocked[]` and loses its block-line `reason`.
- **`task_blocks`.** Successors keep role `unblocked`.

**Human output** follows the existing word-led rows (`dry run`: `would link`). It prints
after the `ledger` line for `!`, and after the task rows and before `next:` for closes:

```text
✓ completed [*] → [x] Fix apollo machine!  sase.md ^fix-apollo
  ledger  Task Link struck in FIX · 2026/20261009.md
  linked [?] → [*] Re-launch all failed agents on apollo!  sase.md ^relaunch-failed-agents → FIX · added ^relaunch-failed-agents
  unblocked [?] → [ ] Book flights  travel.md ^book-flights · not planned today
  still blocked  Start adding sase--<name> badges  sase_agents_repo.md ^badges · until 2026-10-13
plan 3/3 themes · 10/10 links
```

- **Destinations:**
  - `→ FIX`
  - `→ FIX (next up)`
  - `→ new BOB session (next up)`
  - `→ new session` for an unnamed created entry
  - `→ line 37` for an unnamed existing entry
- **Reason suffixes:**
  - `· already planned`
  - `· not planned today`
  - `· project task`
  - `· hidden`
  - `· not linked, more than 5`
  - `· linking off`
- **Still-blocked suffixes:** `· waits on N more`, `· until YYYY-MM-DD`.
- A row with no block ID omits ` ^`, as today.

## Feedback design: what → where → why

**Copy rules (shared by every surface).**

- One notice per gesture, and silence when nothing was unblocked.
- Each row reads successor text first, then the destination, then the cause.
- Use only existing glyphs: 🔓 unblocked (Mac `lock.open.fill`), 🔒 still blocked (Mac
  `lock.fill`), ✓ done.
- Use only existing `--task-status-*` colour tokens, with no new hex values.
- Show at most 3 rows (linked, then not linked, then still blocked), then `+N more`.
  This is a display limit, not an insertion limit.
- Never lead with a block ID.
- Truncate task text to 48 characters with `…`.

**Notice text** (`successorNoticeText(model)` in the cycler, and the Mac notification
line). This is the plain fallback and walk-toast form:

| Situation                  | Text                                                                             |
| -------------------------- | -------------------------------------------------------------------------------- |
| One linked                 | `🔓 Next in FIX: Re-launch all failed agents on apollo!`                         |
| Several linked, one entry  | `🔓 3 linked → SASE: Review memory beads, Ship AGENTS.md…, +1`                   |
| Several entries            | `🔓 3 linked · FIX, SASE`                                                        |
| Created continuation       | `🔓 Next in new BOB session (next up): Re-launch…`                               |
| Breaker                    | `🔓 7 unblocked · not linked (more than 5)`                                      |
| Recovered only             | `🔓 Unblocked: Book flights (Ready)`                                             |
| Partial failure (Obsidian) | `⚠ Closed Fix apollo machine! — couldn't link 1 successor (daily note changed)` |

**Obsidian: the `Unblocked` card** (nav's notice family, `bob-nh-notice is-unblock`).

```text
┌──────────────────────────────────────────────────────────────────┐
│ 🔓  Unblocked   1 → FIX                     ✓ Fix apollo machine! │
│  ＋ Re-launch all failed agents on apollo!               [ Next ] │
│  🔒 Start adding sase--<name> badges · waits until Oct 13         │
│ ──────────────────────────────────────────────────────────────── │
│  (plan 3/3 · 10/10)  (added ^relaunch-failed-agents)  (Alt+N releases) │
└──────────────────────────────────────────────────────────────────┘
```

- **Header.**
  - 🔓 sits in the existing glyph slot.
  - The title is `Unblocked`.
  - The count chip reads `n → NAME` / `n → new NAME session` / `n linked`, plus
    `· next up` when it applies.
  - The right-aligned muted receipt reads `✓ <predecessor>`, or `✓ N tasks`.
- **Rows.**
  - **Linked rows** show `＋ text`, a lane pill (`Next`), `↗ <note>` when D lives in a
    different note than its first predecessor, and `inbox` when D lives in an inbox
    file.
  - **Not-linked rows** show the text, a lane pill (`Ready`/`Next`), and a muted reason:
    `already planned`, `not planned today`, `not linked · more than 5`, `project task`,
    `hidden`, `linking off`, `couldn't link`, or `cancelled`.
  - **Still-blocked rows** are muted: `🔒 text · waits on N more` or
    `· waits until Oct 13`.
  - Clicking a row opens that task (`app.workspace.openLinkText`).
- **Footer chips.**
  - The plan meter (`getCancelPlanBudgetChip` on the post-write daily content), which
    turns warn/🔴 when over.
  - `added ^id`, or `added N block IDs`.
  - `Alt+N releases` (muted), shown when anything was linked.
- **Breaker variant.** The header reads `7 · not linked`, followed by one reason line:
  `More than 5 at once — link the ones you want with Ctrl+Shift+Enter`.
- **Duration.** 6 s plus 1 s per extra visible row, capped at 10 s.
- **No nav.** Without nav (or with `api.notice` missing), show a plain
  `new Notice(successorNoticeText(model))`.
- **Walk landings.** The gesture keeps its **single** walk toast
  (`answering-advances-the-walk`). The notice text is composed into `outcome.notice` as
  the line after `✓ Done · <task>`, and no card is shown. Inserted rows never steal
  focus or cause a second advance.

**Shared notice model** (cycler → nav `api.notice.showUnblocked(model)`). It is the JSON
row shape above, kept in snake_case on purpose so one vocabulary serves Rust, JS, Swift,
and the vectors:

```js
{
  version: 1,
  predecessors: [{ note_path, block_id, text }],
  unblocked: [/* rows exactly as JSON, plus not_linked "failed" | "cancelled" */],
  still_blocked: [/* rows exactly as JSON */],
  daily_path: "2026/20261009.md",
  daily_content: "<post-write day text, for the plan chip>" | null,
  failure: null | { count, reason },
}
```

**Bob Mac Capture: preview first, notification second.** Preview is the primary surface
because it explains before anything is written.

- **`!` card unblocked rows.**
  - **Linked.** `lock.open.fill` in the Next tint, monospaced `[?] → [*]  Re-launch…`,
    and a trailing destination capsule (`→ FIX` / `→ new BOB · next up`). A caption
    reads `added ^id` when an ID was minted. An `inbox` capsule appears when it applies.
  - **Not linked.** `lock.open.fill` in secondary colour, `[?] → [ ]  text`, and a
    caption with the reason (`Ready · not planned today`).
  - **Still blocked.** `lock.fill` in tertiary colour, `text`, and a caption reading
    `stays Blocked · waits on 1 more` / `· until Oct 13`.
- **`=x` close card.** Gains an **Unblocked** section under its task rows, with the same
  row views.
- **Block diff.** An added line with `reason == "unblocked"` gets a small trailing
  `lock.open.fill` badge with `.help("Unblocked successor")`, so the successor visibly
  sits under its struck predecessor.
- **Notification.** One extra body line, taken from the notice-text table above. There
  is never a second notification. With only recovered rows, keep today's
  `· unblocked A, B`.
- **Open Note(s)** includes the daily note whenever anything was linked.

## Performance design (A10)

| Layer                                         | What                                                                                                                                                                                                                                                                           | Effect                                                                                                 |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| 0. `[id::]` gate                              | Before any lookup, check the **staged** post-close lines of C for an `[id::]` field.                                                                                                                                                                                           | Zero extra I/O for about 96% of closes, in Rust and JS. It also speeds up today's Ctrl+Enter recovery. |
| 1. Lazy context (`snapshot`)                  | `DependencyContext` builds `discover` only for items that need the dependency pool (`&`). `!` resolves notes from a walk-only index. `previous_daily` becomes lazy.                                                                                                            | `hello` / `=x` previews go from about 270 ms to about 40 ms on apollo.                                 |
| 2. Prefiltered parallel snapshot (`snapshot`) | One batch-scoped dependents snapshot. Byte reads run under `std::thread::scope`, and a `memchr::memmem` prefilter keeps only notes containing `dependsOn` or `id::`. Staged files always overlay disk. Shared by every `!` and close item in the batch, never cloned per item. | About 30 ms per batch, paid only when the gate passes.                                                 |
| 3. Obsidian Tasks cache (`cycler_wiring`)     | When Tasks reports Warm, build a reverse index `id → dependents` from `getTasks()`. Open editor buffers override the cache. Read only the candidate notes and today's daily note. A cold cache falls back to the full scan.                                                    | The notice lands about 50 ms after the keypress.                                                       |
| No daemon, no persistent cache                | A validated cache needs a stat walk (32–55 ms) that costs as much as layer 2.                                                                                                                                                                                                  | Respects `mac-capture-is-a-thin-client`.                                                               |

**Targets.** Measured on apollo with a release build, against a temporary **copy** of
`~/bob` (never the live vault):

- `hello` and `=x` dry runs take ≤ 40 ms;
- a `!` or `=x!` that closes a real prerequisite takes ≤ 70 ms;
- `capture-parse` does no new work;
- no extra spawn is needed to report results.

## Reliability and undo

- **Capture is atomic.** Recovery, minted IDs, status changes, the continuation, and
  links join the existing staged batch, so they commit or roll back together with
  preimage validation.
  - Successors are planned **inside each item**, so `=x!1 =` starts a continuation that
    already holds the successor.
  - A snapshot failure gives `unblocked_check: "unavailable"` and a normal close.
- **Obsidian order of operations.**
  1. Close (as today).
  2. Inside `finalizeClosedTasks`: compute anchors, then recover and link in one planned
     pass, then retire embeds (as today).
  3. Show one notice.

  Writes go through the open editor when the note is open, as one transaction per note.
  Otherwise they go through `vault.process` with a line preimage check.
  - Linking is best-effort. A failure leaves the successor recovered but unlinked, and
    the notice says so.
  - A close is never rolled back for a linking failure.
  - Everything runs on `referenceMutationQueue`, using only `…Now` variants
    (re-entrancy).

- **Hooks.** The hooks never add links. Their Next-if-linked rule agrees with what was
  written. Reopening a predecessor re-blocks the successor through derivation, and
  sticky lanes do not survive Blocked.
- **Undo.**
  - **One successor.** Alt+N on its link releases it, as the inverse of what was
    written. The card's chip teaches this.
  - **The whole gesture, in Obsidian** (`cycler_polish`). Reopening the predecessor the
    same day takes back its successor links if they are untouched. This uses an
    in-memory receipt; nothing is stored in Markdown.
  - **Not undo.** Deleting the bullet or pressing `u` in the editor leaves the sticky
    Next. That is the general sticky-lane cost.

## Conformance vectors

`contract` copies these into `docs/task-dependencies.md` §11.6. They are written as
outcomes of the shared rule. Rust tests run them through capture (`!` and close), and
cycler tests run them through the pure helpers.

| #    | Situation                                                                            | Outcome                                                                                                       |
| ---- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| SL1  | P linked in the running entry; D `[?]` depends only on P                             | link right after P's bullet subtree; D `[?]`→`[*]`; `unblocked_by: [P]`                                       |
| SL2  | P linked in queued FIX while BOB runs; `!` retirement moves P's struck link into BOB | D at P's vacated FIX position; FIX not removed as empty                                                       |
| SL3  | D also depends on open Q                                                             | no link; `still_blocked{waits_on, 1}`; D stays `[?]`                                                          |
| SL4  | D's only remaining blocker is a future `scheduled`                                   | no link; `still_blocked{scheduled}`; schedule untouched                                                       |
| SL5  | D already has a live link under another open entry                                   | no link (`already_planned`); `[?]`→`[*]`; existing link not moved                                             |
| SL6  | D's only links today are struck or under closed entries                              | linked as in SL1                                                                                              |
| SL7  | `=!` closes BOB; nothing carried; FIX follows                                        | `- [ ] () — BOB` created right after BOB holding only D; `entry_created`, `next_up`; `next_pomodoro` names it |
| SL8  | `=x` close with carried links; an embed P completes                                  | D appended after the carried lines in the continuation                                                        |
| SL9  | P has no live link today                                                             | D `[?]`→`[ ]`; `not_planned_today`; successor logic leaves the ledger alone                                   |
| SL10 | D has no `^block-id`                                                                 | ` ^<mint>` appended (SB rule); `block_id_created: true`; `block_id` is the mint                               |
| SL11 | D is `^prj` / carries `#hide`                                                        | recover only; `project_task` / `hidden`                                                                       |
| SL12 | One gesture closes P1 and P2; D depends on both                                      | D linked once, after the earlier anchor; `unblocked_by: [P1, P2]`                                             |
| SL13 | Chain P → D → E                                                                      | D linked; E untouched and not reported                                                                        |
| SL14 | D's basename is ambiguous                                                            | `[[dir/note#^id]]`                                                                                            |
| SL15 | D is a stale `[ ]` whose only prerequisite was P                                     | linked; `[ ]`→`[*]`                                                                                           |
| SL16 | No task in C carries `[id::]`                                                        | gate: no lookup, no extra reads; empty arrays; `checked`                                                      |
| SL17 | Six successors from one gesture                                                      | none linked; all `breaker`, recovered to derived rank                                                         |
| SL18 | `plan.link_unblocked: false`                                                         | recovery to derived rank; no links or mints; `disabled`                                                       |
| SL19 | The same close re-applied (already Done) / a double press                            | no writes, no rows                                                                                            |
| SL20 | Draft `!P` then `!D`                                                                 | the first item's `unblocked` omits D (net)                                                                    |
| SL21 | Cancel, reopen, raw checkbox, hooks run, `=*` park                                   | never link                                                                                                    |
| SL22 | Ctrl+Enter on a Depends-On line                                                      | unchanged (nothing)                                                                                           |
| SL23 | P closes as a subtask of an embedded root R planned today                            | P anchors at R's link (`inherit`)                                                                             |
| SL24 | Close creates no continuation; a later open entry named BOB exists                   | D appended to that BOB entry; `entry_created: false`                                                          |
| SL25 | D already `[*]` and already planned                                                  | nothing written, no row                                                                                       |
| SL26 | D lives in an inbox file                                                             | linked; `inbox: true`                                                                                         |
| SL27 | `=x~K` drops D's link in the same close that completes P                             | D not re-linked (`already_planned`); recovers to `[ ]`                                                        |
| SL28 | D lives in today's day file                                                          | link names the note (`[[20261009#^id]]`), never `[[#^id]]`                                                    |

**SB vectors.** Eight to ten `(raw task line, note's used IDs) → mint` rows. `contract`
generates them by running bob's real `clean_description` and `suggest_ids_with_used`,
and pins them with a Rust unit test. They must cover:

- plain prose;
- a leading verb;
- a backticked phrase;
- a wikilink alias;
- inline fields and a trailing global-filter tag;
- a collision that takes `-2`;
- all suggestions taken, which falls back to `task-N`;
- non-ASCII text.

# snapshot

Work in bob-cli. This phase **is** bead `bob-cli-5v`; read it with
`sase bead read bob-cli-5v -r "<why>"`. It changes no user-visible output except the
documented gate edge.

- **Lazy `DependencyContext`** (`src/native/capture/dependencies.rs`, built at
  `src/native/capture/plan.rs:84`).
  - `new` must do no I/O.
  - Turn `discovered`, `previous_daily`, and the note index into lazily initialised
    accessors (`OnceCell`-style fields or `Option` + getter). The full
    `capture_dependency_tasks::discover` runs only for items that need the dependency
    pool (`&` edits, selected/explicit dependents).
  - `!` and `&` note resolution (`resolve_prerequisite_note`, `staged_note_index`)
    should work from a walk-only `NoteIndex` (paths and basenames, no reads) when
    `discover` has not run.
- **Dependents snapshot.** Replace `recovery_base_snapshot` (a full read, cloned per
  call) with a batch-scoped `DependentsSnapshot`, built once on first use and borrowed
  afterwards.
  - Walk `task_status_hooks::markdown_files(bob_dir)` on the main thread. Resolve every
    env-dependent path before fanning out: `bob_env` overrides are thread-local; see
    `src/native/env.rs`.
  - Read bytes under `std::thread::scope` with an atomic work queue. The pattern is in
    `src/native/highlights_ref/sync.rs` (`plan_pdfs`).
  - Keep a file only when `memchr::memmem` finds `dependsOn` or `id::`. Add `memchr` as
    a direct dependency; it is already in `Cargo.lock`.
  - `staged_snapshot_for_recovery` (`src/native/capture/task_complete.rs`) becomes an
    overlay. Every staged Markdown path uses its staged text, prefiltered the same way,
    and replaces or removes the disk version. Build the overlay per call without cloning
    the base map (an iterator or `Cow`).
  - A walk or read failure is non-fatal. Expose it as a `Result`/flag so
    `capture_complete` can report `unblocked_check: "unavailable"`.
- **`[id::]` gate.** Before building the snapshot for `!` recovery, read the staged
  completed lines (root and closed subtasks). If none carries an `[id::]` field
  (`task_metadata(..).task_id`), skip recovery entirely.
  - Replace the linear `snapshot.iter().find` in `completed_ids` with direct lookups in
    the staged planner text.
  - The gate edge is a dependent naming only the canonical id of a target without
    `[id::]`. It is left to the hooks. Record this in the phase's commit message;
    `contract` documents it.
- **Equivalence.** Add a unit test proving the prefiltered snapshot yields the same
  `task_dependency_states` and the same `recover_blocked_dependents` result as a full
  read on a fixture vault with:
  - same-note and cross-note edges;
  - a `(id:: …)` paren form;
  - a staged-new note;
  - a staged deletion.
- **Guard tests** (`tests/cli/capture/…`):
  - A plain `hello` capture, and an `=x` that completes nothing, succeed and emit no
    warning about a note that only discovery would touch. Use a fixture note with
    invalid UTF-8, or an unreadable mode where the platform allows it.
  - `!` on a task without `[id::]` neither reads nor reports dependents.
  - Existing `!` recovery tests stay green, except tests that relied on canonical-only
    matching; update those with a comment citing the gate.
- **Timings.**
  1. Build with `cargo build --release`.
  2. Copy the vault to a temp dir:
     `rsync -a --exclude .git --exclude .obsidian ~/bob/ "$tmp/"`. Never point bob at
     `~/bob` itself.
  3. Time 7 runs each, with
     `BOB_DIR="$tmp" BOB_NOW='<today> 10:10:00' target/release/bob capture --dry-run --format json -- '<draft>'`,
     for the drafts `hello`, `=x`, and one `!note:id` whose task carries `[id::]` and
     has a dependent (find one with `rg -l dependsOn "$tmp"`).
  4. Paste before and after numbers into a `sase bead note` on this phase. Targets: ≤ 40
     ms for `hello`/`=x`, and well under today's ~400 ms for `!`.
- **Gate.** `just check` is green.
- **Close the bead.** Run `sase bead close bob-cli-5v --note "<timings + commit>"`.

# contract

Work in bob-cli docs, plus one Rust unit test. Write in present tense: the epic lands as
a unit.

- **`docs/task-dependencies.md`.**
  - In §5 "Closing", add one bullet: closing a planned prerequisite links the dependents
    it fully unblocked into its slot (§12).
  - Add **§12 Successor links** after §11. Content:
    - 12.1 Trigger (the gesture table);
    - 12.2 Rule (the pseudo-code and every note under "The rule", including identity,
      the gate, graph transition, the already-planned baseline, and row semantics);
    - 12.3 Anchors and placement (both anchor kinds, the continuation search, `next_up`,
      insertion before retirement or strike, the never-annotate rule);
    - 12.4 Writes (status, minting, link form including the day-file exception and the
      exact file set that counts for basename uniqueness, the freshness/schedule/field
      prohibitions, and why `plan_task_link` is not used);
    - 12.5 Reporting model (the JSON rows, the shared notice model, reasons including
      the cycler-only `failed`/`cancelled`);
    - 12.6 Copy (the notice-text table, card anatomy, Mac rows, CLI rows);
    - 12.7 Performance (gate, snapshot, Tasks cache, targets);
    - 12.8 Kill switch;
    - 12.9 Undo.
  - Add **§11.6 SL — successor-link vectors** (SL1–SL28 verbatim) and **§11.7 SB —
    successor block-ID vectors**.
  - In §9 (plugin api v3), document the additive nav member
    `notice: { version: 1, showUnblocked(model) → boolean }`.
  - To fix the basename file set, check what `NoteIndex` capture uses for
    `prerequisite_link_text` (`src/native/capture/dependencies.rs:509`,
    `src/native/task_dependencies/format.rs:36`) and write that set down precisely.
- **SB vectors.**
  - Add a `#[test]` in `src/native/capture_block_ids.rs` that feeds each SB raw line
    through `note_tasks::clean_description` (with the default `#task` global filter) and
    `suggest_ids_with_used(.., '^', used)`, plus the `task-N` fallback described above.
  - Implement the fallback as a small
    `pub(crate) fn mint_block_id(description, used) -> String` beside the suggester, so
    `capture_complete` reuses it.
  - Assert the table, and copy the asserted outputs into §11.7.
- **`docs/capture.md`.**
  - In "Completing tasks with `!`", step 6, describe recovery plus successor linking,
    and the step order: the successor link is placed before retirement.
  - In "Closing the running Pomodoro", replace "Vault-wide Blocked recovery and
    completed-reference retirement outside the session stay in `bob task reconcile`"
    with:
    - close-time recovery plus successor linking (with a §12 link);
    - the continuation rule (successors count as carried; same-name entry search);
    - the statement that completed-reference retirement **outside** the session still
      stays with reconcile.
  - In the JSON contract paragraphs for task-complete (around the `unblocked`
    description) and the close summary, add:
    - the extended `unblocked[]`;
    - `still_blocked`, `unblocked_check`;
    - task-complete's top-level `day_file`;
    - the `next_pomodoro` effect;
    - the `pomodoro_blocks` line `reason`;
    - net batch reporting;
    - dry-run equality.
  - Update the human-output paragraphs with the new rows, destinations, and suffixes.
- **`docs/plan.md` "Config".**
  - Add
    `link_unblocked: true # link a closed planned task's unblocked dependents into its slot (docs/task-dependencies.md §12)`.
  - Add the validation rule: a boolean.
  - Add the invalid-value behavior: default `true` plus the existing warning paths.
- **Stale statements.** Grep bob-cli `docs/` and `README.md` for "always Ready", "stay
  in `bob task reconcile`", "intentionally narrower", and "unblocked" near close or
  Ctrl+Enter. Update each so no doc contradicts §12 (for example
  `docs/task-status-hooks.md`).
- **Gate.** `just check` is green (docs lint, plus the new unit test).

# capture_complete

Work in bob-cli. Read `docs/task-dependencies.md` §12 and §11.6–11.7 first; they are the
contract.

- **Config.** In `src/native/config/plan.rs`:
  - add `link_unblocked: bool` (default `true`) to `PlanConfig`, with a getter and
    loader validation (boolean only);
  - add unit tests.
  - Capture reads it once per batch. An invalid plan config uses `true` and keeps the
    existing warning.
- **Pure planner** — new `src/native/task_complete/successors.rs`, exported from
  `task_complete/mod.rs`. It takes:
  - the completed set C: relative path, block ID, task ID, display text, and anchor
    (`Closing` / `Slot { entry_index, bullet_line }` / `None`), plus the root reference
    that `inherit` uses;
  - the borrowed dependents snapshot view with staged overlay (from `snapshot`);
  - the day text before this gesture (the already-planned check) and after it (derived
    rank);
  - `today`, `TasksSettings`, `link_unblocked`, and the `NoteIndex`.

  It returns:
  - per-dependent outcomes (successor / recovered + reason / still-blocked);
  - note edits (status changes plus minted ` ^id`, via `set_task_line_status` and the
    `append_block_id` shape from `src/native/capture_task_id.rs:598`);
  - the ordered successor placements.

  Rules:
  - Build `post_open` with `task_dependency_states` over the snapshot.
  - Implement steps 1–8, the breaker, and the ordering exactly as in §12.
  - Identify project tasks by block ID `prj` (case-insensitive, as
    `capture_language/project_tasks.rs:63`).
  - Identify hidden tasks with `capture_dependency_tasks`'s `has_hide_tag`.
  - Inbox classification mirrors nav's `isInboxNotePath`
    (`plugins/bob-navigation-hotkeys/src/255-inbox-route.js`; open bob-plugins with
    `sase repo open bob-plugins -r "<why>"`). Add a small Rust helper with unit tests.
  - Minting uses `mint_block_id` (from `contract`).
  - Link text uses `canonical_link` with the day-file exception.
  - Unit-test SL1, SL3–SL6, SL9–SL19, SL23, SL25, SL26, and SL28 against in-memory
    snapshots, in the style of `task_complete/tests/recovery_tests.rs`. Cite §11.6 in a
    comment.

- **Placement helper.** Add
  `insert_successor_links(day_text, placements) -> (String, Vec<PlacedSuccessor>)` next
  to the retirement code (`task_complete/`).
  - It inserts after an anchor bullet's subtree, or into a closing target (continuation
    / same-name entry / created placeholder).
  - It reports each link's 1-based `entry_line`, `line`, `entry_created`, and `next_up`
    on the resulting text.
  - It reuses `logical_lines`, `pomodoros_section_range`, and `scan_pomodoros` from
    `task_status_hooks`.
  - Closing targets are implemented here, but `capture_close` wires them.
- **Wire `!note:id`** (`src/native/capture/task_complete.rs`,
  `plan_task_complete_item`):
  1. After the tree close is staged, compute C and anchors on `pre_day`, the staged day
     text before retirement.
  2. Apply the gate; when it passes, build or borrow the dependents snapshot and run the
     planner.
  3. Stage the note edits.
  4. Insert successor links into the day text.
  5. **Then** run `retire_completed_links` on that text. Recompute
     `compute_link_statuses` after insertion, so the successor link resolves Live.
  6. Remove the old `recover_blocked_dependents` call. Recovery is now the planner's
     recover branch. Keep the function only if other callers remain.
  7. Stage the day text once.
  8. Emit `task_blocks` role `unblocked` for every row.
- **Output** (`src/native/capture/output.rs`).
  - Extend `TaskCompleteUnblockedJson` with `inbox`, `unblocked_by`, `link`, and
    `not_linked`.
  - Add `still_blocked` and `unblocked_check` to `TaskCompleteSummaryJson`.
  - Set the item's top-level `day_file` (absolute) whenever the day file changed.
  - Human rows go in `print_human_task_complete_success` per the contract, including
    dry-run tense.
  - Update the `bob capture --help` wording for `!` (grep the help text for "unblock").
- **CLI tests** (`tests/cli/capture/task_complete.rs`, helpers in
  `tests/cli/support.rs`). Cover:
  - SL1, SL2, SL9, SL10, SL14, SL16, SL18, and SL19 end to end, with exact file bytes
    and JSON;
  - dry-run JSON equals real-run JSON (minus `dry_run`);
  - a `pomodoro_blocks` diff that contains the successor line.
- **Gate.** `just check` is green. Re-run the `snapshot` timing for the `!` draft and
  note it on the phase bead (target ≤ 70 ms).

# capture_close

Work in bob-cli on top of `capture_complete`.

- **Completed set.** In `plan_pomodoro_close_item` and `plan_pomodoro_close_link_item`
  (`src/native/capture/pomodoro_close.rs`, around lines 591 and 957):
  - Pass in the batch's `DependencyContext` (`plan.rs` dispatch around lines 418–422 and
    598–603).
  - Derive C from the close plan's task rows with role `embedded`/`subtask`,
    `status_symbol == 'x'`, and `status_changed`. Expose the needed resolved path
    (`relative_target`) and task ID from `PomodoroCloseTask`. `resolved_path` is private
    today; add an accessor rather than re-resolving.
  - Anchors are `Closing` for links in the closed entry; subtasks inherit.
- **Order.** After `stage_close_plan`, run the gate, the planner, the note edits, and
  then the closing-target placement on the **post-close** day text.
  - Use the continuation `LedgerClosePlan.next_pomodoro` names when `created`.
  - Otherwise search for the first open same-name entry after the closed one.
  - Otherwise create the placeholder at the continuation position, after the closed
    entry's sub-bullet range.
  - Update the summary's `next_pomodoro` to
    `{line, name, time_range: None, created: true}` when the successor logic created it.
  - The already-planned baseline is the day text before the item. That means pre-link
    for `^route:id=x…`, because the explicit link step happens first.
  - `=x` closes that complete nothing (no embeds) stay byte-identical. Add a regression
    test.
- **JSON.**
  - Add `unblocked`, `still_blocked`, and `unblocked_check` to
    `PomodoroCloseSummaryJson` (`build_close_summary_json`).
  - Add an optional `reason` field to the block-line JSON in
    `src/native/capture/pomodoro_blocks.rs`. Set it to `"unblocked"` for added lines
    that equal a surviving successor bullet. Track the successor block links per day
    file through `PomodoroBlockTracker`.
- **Net batch post-pass** (`plan_capture_batch`, after the item loop). For every row
  with `link`, verify both conditions against the final staged text:
  - the bullet still exists unstruck under an open entry of the final day text;
  - the task is still open.

  Otherwise drop the row and its block-line reason (SL20).

- **Human output.** In `print_human_pomodoro_close_success`, print the same rows after
  the task rows, and make `next:` mention the successors:
  `next: BOB (created) at line N · carries K links · 1 unblocked`.
- **Help text.** Add one sentence to the `=x!` / `=!` help: completing planned tasks
  also links what they unblocked.
- **CLI tests** (`tests/cli/capture/…` close suites). Cover:
  - SL7, SL8, SL12, SL17, SL20, SL24, SL27, and the `^route:id=x…` example (c);
  - a two-item `=x!1 =` draft proving the next start opens the continuation that holds
    the successor;
  - dry-run equality.
- **Gate.** `just check` is green. Time `=x!1` on the apollo copy, with a selected embed
  that carries `[id::]`, and note it (target ≤ 70 ms).

# mac

Work in bob-mac-capture. Open it with `sase repo open bob-mac-capture -r "<reason>"`; if
no linked checkout exists, `sase repo open gh:bobs-org/bob-mac-capture`. Use only the
printed path. It has no AGENTS.md; follow its README `## Development` section.

- **Toolchain.** Locally, `swift build --target CaptureCore` and the `CaptureCoreTests`
  run on Linux with swiftly: export `PATH="$HOME/.local/share/swiftly/bin:$PATH"`
  explicitly. macOS CI is the gate for the app target.
- **Models** (`Sources/CaptureCore/CaptureModels.swift`).
  - Extend `TaskCompleteUnblocked` with:
    - `inbox` (`decodeIfPresent ?? false`);
    - `unblockedBy: [TaskCompleteUnblockedCause]`;
    - `link: TaskCompleteSuccessorLink?`;
    - `notLinked: String?`.
  - Add `stillBlocked: [TaskCompleteStillBlocked]` and `unblockedCheck: String?` to
    `CaptureTaskComplete` and to `PomodoroCloseSummary`, and add `unblocked` to
    `PomodoroCloseSummary`.
  - Decode every new nested value defensively and **individually**: `(try? …) ?? []` /
    `?? nil`. A malformed field must never fail the whole capture decode (the close
    summary uses a plain `try`) or drop the whole completion card.
  - Add `reason: String?` to `CapturePomodoroBlockLine` with `try?`, so one bad line
    cannot wipe every block.
- **Presentation (CaptureCore, pure).**
  - `CaptureTaskCompletePresentation.UnblockedRow` gains:
    - `kind` (`.linked` / `.recovered` / `.stillBlocked`);
    - `destinationText` (`→ FIX`, `→ new BOB · next up`, …);
    - `captionText` (`added ^id`, `Ready · not planned today`,
      `stays Blocked · until Oct 13`);
    - `isInbox`.
  - Keep `transitionText` and `locatorText`.
  - Build `stillBlockedRows`.
  - `notificationBody` uses the notice-text line from §12.6 when anything was linked or
    the breaker fired. Otherwise it keeps today's `· unblocked A, B`.
  - `CapturePomodoroClosePresentation` gains `unblockedRows` and `stillBlockedRows`
    built by the same shared row builder (put it in a small shared file), adds its
    notification line, and adds an accessibility summary.
  - Copy matches bob's human output and §12.6.
- **Views (BobMacCapture).**
  - In `completePreviewItem` (`CapturePanelView.swift`, around the `lock.open.fill` rows
    near line 2958), render the three row kinds:
    - Linked rows use the Next tint (reuse the colour `CaptureTogglePresentation` uses
      for `[*]`) and a trailing destination capsule styled like the existing chips.
    - Captions are secondary.
    - Still-blocked rows use `lock.fill` with tertiary styling.
  - `closePreviewItem` gains an "Unblocked" section under the task rows.
  - `BlockDiffCard.swift` / `CaptureBlockDiffRow.swift`: an added line with
    `reason == "unblocked"` shows a trailing `lock.open.fill` mini badge with
    `.help("Unblocked successor")`. Add a matching VoiceOver phrase in
    `CapturePomodoroBlockPresentation`.
- **Open Note(s).**
  - `NotificationService.dayFileChanged` (around line 654) and
    `CapturePanelModel.captureWroteDayFile` (around line 5451) treat `task_complete` as
    changed when `ledger != nil` or any `unblocked[].link != nil`.
  - These checks work now because bob sets the top-level `day_file`
    (`capture_complete`).
  - Pomodoro closes keep their existing path logic.
- **Fixtures** (`Tests/Fixtures/*.json`) come from real bob, built from bob-cli master
  with this epic's capture phases. Use a sandbox vault and record every generating
  command in the test file header, the way `CaptureTaskCompletePresentationTests.swift`
  does. Generate:
  - a linked successor;
  - a minted ID;
  - not planned today;
  - still blocked (scheduled and `waits_on`);
  - the breaker;
  - an `=!` close that creates a continuation with `next_up`;
  - a close appending to carried lines.

  Add fake-bob routes for these drafts.

- **Tests.**
  - `CaptureModelTests`: decoding, including malformed and absent fields.
  - Presentation tests for every row kind, tense, notification line, and accessibility
    string.
  - `NotificationServiceTests`: the Open Note(s) daily-note inclusion.
  - A block-diff badge test.
  - Design renders behind `BOB_MAC_CAPTURE_RENDER_DIR` for the `!` card and the close
    card with successors, in light and dark, like `TaskCompleteDesignTests`.
- **README.** Update "Completing tasks with `!`", the close card, and Pomodoro blocks
  sections with the successor rows, badge, and notification line.
- **Commit, then CI.** This plan explicitly instructs you to commit the bob-mac-capture
  changes with `/sase_git_commit` before waiting on CI, because macOS CI only runs on
  the pushed commit.
  - Use subject `feat(capture): preview and announce successor links`. Pass `-B keep` if
    `$SASE_BEAD_ID` is set.
  - Wait with `/sase_monitor` on the `CI` run for that SHA
    (`gh run list --commit <sha> --limit 1`, then `gh run watch <id>`), with a timeout
    of at least 20 minutes.
  - If it is red, read `gh run view <id> --log-failed`, fix forward, and repeat.
  - The phase is done only when the latest run is green.
  - No Swift code may compute dependents or placement (`mac-capture-is-a-thin-client`).

# nav_card

Work in bob-plugins (`sase repo open bob-plugins -r "<why>"`; read its AGENTS.md).
Plugin: `plugins/bob-navigation-hotkeys`.

- **New fragment `src/285-unblocked-notice.js`.** `280-notices-and-dates.js` is at 974
  lines; every fragment must stay ≤ 1000. Register it in `src/fragments.json` after 280.
  It holds:
  - `buildUnblockedNoticeModel(model, { app })`: validates and normalizes the shared
    model (§12.5) and returns null when there is nothing to show;
  - `renderUnblockedNoticeFragment(view)`;
  - `showUnblockedNotice(app, model)`: built through `showBulletPropertyNotice` like the
    cancel card, returning `true` when shown.
- **Anatomy and copy.** Exactly as in §12.6.
  - Header: 🔓 glyph, `Unblocked`, count chip, right-aligned muted `✓` receipt.
  - At most 3 rows, then `+N more`. Row grid: glyph, text, lane pill, chips.
  - Rows are clickable: `app.workspace.openLinkText("<path>#^<id>", daily_path)`, or the
    note path when there is no ID. Use `stopPropagation` so the click does not just
    dismiss the Notice.
  - Footer chips: plan via `getCancelPlanBudgetChip(app, daily_content)`, `added ^id`,
    `Alt+N releases`.
  - Breaker and failure variants. Duration 6 s plus 1 s per extra visible row, capped at
    10 s.
- **CSS** (`styles.css`, in the notice block around lines 1140–1368).
  - Add `.bob-nh-notice.is-unblock`, row-grid, pill, and muted-row rules.
  - Use the existing `--task-status-*` tokens and the cancel card's chip tones (`ok`,
    `info`, `warn`, `muted`), with no new hex colours.
  - It must look right in light and dark themes.
- **api.** In the api factory (`480-review-jump-and-nav-api.js`,
  `createDependencyNavApi`, currently 959 lines), add the frozen
  `notice: Object.freeze({ version: 1, showUnblocked: (model) => showUnblockedNotice(this.app, model) })`.
  - Keep the api `version: 3`; the member carries its own version, like `reviewWalk`.
  - If 480 would exceed 1000 lines, put a `createNoticeApi(plugin)` factory in 285 and
    reference it.
- **Tests.** Add `scripts/test-navigation-hotkeys-unblocked-notice.cjs` (stub `document`
  like `test-navigation-hotkeys-priority-notice.cjs:407`), and add it to the
  `package.json` test list. Cover:
  - single linked, several linked, a created continuation with next up;
  - recovered-only, still-blocked, `+N more`, breaker, failure;
  - the click handler;
  - the absence of the plan chip when `daily_content` is null;
  - the api shape.
- **Ship.**
  - Bump nav's manifest minor version.
  - Update the nav row in `README.md` and the manifest description.
  - Run `npm run build`, then `npm test`, then `bob plugins sync`.
  - Parallel phases also append to `package.json`'s test list and README. Rebase onto
    the latest master before committing and keep both sides.

# cycler_engine

Work in bob-plugins, plugin `plugins/task-status-cycler`. Add pure code only, with no
behavior change. Use new fragments, each ≤ 1000 lines, registered in
`src/fragments.json` after `080-dependencies.js`:

- **`085-successor-plan.js`**
  - `buildDependentsIndex(tasks)` takes normalized tasks
    (`{path, line, status, blockId, taskId, dependsOn[], text, rawLine, scheduled}`) and
    returns `{byDependsOn: Map<id, task[]>, byTaskId: Map<id, task>}`.
  - Two normalizers feed it:
    - `normalizeTasksCacheTask` (port the shape handling from nav's
      `readStageTasksCache` / `normalizeStageCacheTask`,
      `590-plugin-dependency-stage.js:226` and `300-dependency-stage.js:360`);
    - `tasksFromDocuments(documents)`, built on the existing
      `parseTaskDependencyMetadata` (`080:80`).
  - `findLiveLinks(dailyLines)` returns per-entry live links in ledger order, built on
    `findPomodorosSectionInLines` / `getSubBulletBlockRange` / the
    `classifyPomodoroSubBullets` conventions. Struck links don't count; open means a
    `[ ]` entry.
  - `computeSuccessorAnchors(closed, dailyLines, { closingEntry })` implements `closing`
    / `slot` / `inherit` / `none`, given closed identities with an optional `rootKey`.
  - `planSuccessors({ closed, index, dailyBefore, today, linkUnblocked, isInbox, basenameCounts })`
    implements steps 1–8, the breaker, and ordering. It returns the §12.5 rows
    (snake_case), `still_blocked`, note edits (`{path, line, before, after}`), and
    placements.
  - `planSuccessorInsertions(dailyLines, placements, { closingEntry })` returns editor
    edits plus each link's `entry_line`/`line`/`entry_created`/`next_up`.
  - `successorNoticeText(model)` implements the §12.6 text table.
- **`086-successor-ids.js`**
  - A faithful port of `capture_block_ids.rs` `suggest_ids_with_used`, with its
    STOPWORDS, LEADING_VERBS, `words_in`, `phrase_spans`, `wikilink_phrase`,
    `join_truncated`, and suffix rules (ASCII-only words, 32-character cut, `-2`…`-9`).
  - `cleanDescription(rawLine, globalFilter)` mirrors `note_tasks::clean_description`.
  - `mintBlockId(description, usedIds)` adds the `task-N` fallback.
  - `successorLinkText(targetPath, blockId, dailyPath, basenameCounts)` implements the
    §12.4 link form, including the day-file exception and the documented file set.
- **Loaders**, in either fragment.
  - `loadLinkUnblocked()` reads `plan.link_unblocked` from
    `$XDG_CONFIG_HOME/bob/config.yml` or `~/.config/bob/config.yml`, stat-cached on
    mtime and size. Port the approach of ledger's `planConfigPath` / `loadPlanCaps`
    (`plugins/bob-ledger-tools/src/320-plan-lane-and-config.js`); a missing or invalid
    value gives `true`.
  - `todayDailyPath(app, now)` mirrors ledger's `isTodayDailyFile` / `todayDailyPath`
    (`020-time-and-pomodoro.js:314`).
  - Mark duplicated code `// Duplicated from <plugin> intentionally`, following the
    precedent in `010-core.js:70`.
- **Exports and tests.**
  - Export the pure helpers in `210-exports.js` `helpers`.
  - Add `scripts/test-task-status-cycler-successors.cjs` and register it in
    `package.json`:
    - every SL vector expressible without a vault (all but SL2, SL20, and SL21's
      hooks/raw cases), citing `docs/task-dependencies.md` §11.6;
    - every SB vector from §11.7, verbatim.
  - `npm run build` and `npm test` are green. No version bump and no sync are needed,
    since nothing is wired yet.

# cycler_wiring

Work in bob-plugins, mostly `plugins/task-status-cycler`, plus one small nav edit.

- **The pass.** New mixin fragment `135-plugin-successors.js`, registered in
  `installTaskStatusCyclerMixins`; method names must be unique. It defines
  `planAndApplySuccessorsNow(closed, context)` and changes `finalizeClosedTasks`
  (`130-plugin-references.js:52`) to run, inside the same queued job:
  1. Run the gate. For identities without `taskId`, look up the closed line once in its
     own note (editor buffer first). With no `[id::]`, skip the rest; there is no
     recovery read either.
  2. Build the index. When Tasks reports Warm (port nav's state check), use
     `getTasks()`, with open editor buffers overriding their notes. Otherwise fall back
     to the existing full scan in `recoverBlockedDependentsNow`.
  3. Read today's daily note: the open editor first, else `cachedRead`. Compute anchors
     with `context.closingEntry`.
  4. Plan with `planSuccessors`, then apply:
     - dependents' note edits through the open editor (re-check the line text) or
       `vault.process` with a preimage check (a mismatch means `failed` for that
       dependent);
     - daily insertions as one editor transaction, or `vault.process` with an
       anchor-line preimage check.
  5. Retire embeds (`retireClosedTaskReferencesNow`, unchanged).
  6. Return
     `{ reopened, retired, recoveryFailures, retirementFailures, successors: model | null }`.
     The model is the §12.5 shared model, including `daily_content`.

  The `recoverBlockedDependents` api (nav cancel) keeps its contract and Ready
  semantics. It gains only the gate and the Tasks-cache index, a pure speed-up with
  identical output.

- **Branches** (`140-plugin-vim.js` `handleVimTaskToggleOpenDone`,
  `160-plugin-completion.js`, `170-plugin-pomodoro.js`):
  - **Pomodoro line.** `completeActivePomodoroTask` passes
    `closingEntry: { line, headline, name, continuationLine }`, captured before edits.
    The pass re-locates the entry and continuation from current editor text by headline.
    Work Log writes can shift lines (`adjustCursorAfterActiveNoteInsertions`). If
    re-location fails, report `failed` rather than guess.
  - **Link under a Pomodoro, plain or embed, and the fallback branch.** No change beyond
    presenting the result. Anchors come from today's daily text while the cursor link is
    still unstruck. Insertions land below the cursor line, so the strike after
    `finalizeClosedTasks` still targets the right line. Verify this with a test.
  - **Task line in its own note.** No change beyond presenting the result.
  - **Walk.** When the gesture settles a walk landing (the
    `reviewWalk.continue(origin, { kind: "complete" })` path), put
    `successorNoticeText(model)` into `outcome.notice` after the Done line and show no
    card.
  - **`completeTaskAtCursor`.** Additively return `successorNotice: string | null` and
    show nothing itself. In nav's caller (`470-keydown-and-freshness.js` around line
    142), append that line to the walk toast it already shows.
  - **Elsewhere.** Present through `getNavNoticeApi()`, which follows the
    `getReviewWalkApi` pattern (`160:63`) and checks `api.notice?.version >= 1`. It
    calls `showUnblocked(model)` and falls back to
    `new Notice(successorNoticeText(model))`.
- **Notice outcomes.**
  - Exactly one notice per gesture.
  - No notice when the model has no unblocked rows and no failure.
  - A failure shows the `⚠` text and never rolls back the close.
- **Guard tests** (`scripts/test-task-status-cycler-*.cjs`). The harness needs a Tasks
  plugin stub in `createInMemoryObsidianApp`, plus a read counter. Cover:
  - SL1, SL5, SL7, SL8, SL9, SL10, SL12, SL17, SL18, SL22, SL23, SL24, and SL28 through
    real Ctrl+Enter actions;
  - a Warm close of a prerequisite reads only the candidate notes and the daily note,
    with no vault-wide `cachedRead`;
  - a close with no `[id::]` performs zero extra reads;
  - the cold fallback still recovers;
  - the walk composes a single toast;
  - nav absent falls back to text;
  - the strike still lands on the cursor link after insertion;
  - a preimage mismatch yields `failed`.

  Existing recovery and ctrl-enter tests stay green. Update expectations where recovery
  now uses the derived rank (Next when linked today).

- **Ship.**
  - Bump the cycler's minor version and nav's patch version (the 470 edit).
  - Update the README cycler row (the Ctrl+Enter close: successors, derived rank, gate,
    and `plan.link_unblocked`), the nav row, line 155's cycler api note (the additive
    `successorNotice`), and the manifest descriptions.
  - Run `npm run build`, then `npm test`, then `bob plugins sync`.

# ledger_glyph

Work in bob-plugins, plugin `plugins/bob-ledger-tools`.

- **When it shows.** In today's daily note (`isTodayDailyFile`), on each live Task Link
  sub-bullet under an open Pomodoro, a faint `🔓` appears after the link when all of
  these hold:
  - the linked task is open;
  - one of its `dependsOn` ids resolves, through the Tasks cache, to a task Done with a
    done date of **today**.
- **Tooltip.** `Unblocked today by ✓ <prerequisite text>`, with `+N` when there are
  several.
- **Rendering.** It is a pure read-time decoration: never written, and recomputed on
  `obsidian-tasks-plugin:cache-update`. With a cold cache, nothing is drawn.
  - Follow the existing ledger decoration and widget pipeline used for the `#task`-mark
    and chip decorations. Find the CodeMirror decoration fragment that renders
    ledger-line marks and add a widget beside it.
  - Style it with the muted `--task-status-*` token at reduced opacity, as a
    non-interactive span with `aria-label`.
- **Tests.**
  - A pure helper, `unblockedTodayLinks(lines, tasks, today)`, gets a unit test in a new
    or existing ledger test file, registered in `package.json`.
  - Cover: a done-today prerequisite shows the glyph; done yesterday does not; a struck
    link does not; a closed entry does not; and several prerequisites give `+N`.
- **Ship.** Bump the ledger minor version, update the README ledger row and manifest,
  and run `npm run build`, `npm test`, and `bob plugins sync`.

# cycler_polish

Work in bob-plugins, plugin `plugins/task-status-cycler`.

- **Reopen receipt.** After a successful pass, store an in-memory receipt keyed by the
  predecessor (`path#^blockId`):
  - `{ date, daily_path, links: [{ entry_headline, line_text }], statuses: [{ path, after_line, previous_symbol }] }`.
  - Drop it at local midnight, on plugin unload, or after use.

  When a predecessor is reopened the same day through Ctrl+Enter (the struck-link reopen
  in `handleActiveTaskBlockLinkOpenDone`, or `[x]` → open on its task line):
  - **Links.** Remove each successor bullet that still exists verbatim under the same
    entry and has no children. Remove a created placeholder only if it became empty
    **and** was created by that receipt.
  - **Statuses.** Restore each successor whose line still equals `after_line` to
    `previous_symbol`. That is the derived truth once the predecessor is open again.
  - **Untouched values.** Leave minted IDs alone, and leave anything edited since the
    pass alone.
  - **Notice.** One line: `↩ Reopened <P> · took back 1 successor link`.

  Capture-made closes have no receipt. Document that Alt+N is their per-link undo.

- **Alt+] / Alt+[ (bead `bob-cli-3k`; read it with
  `sase bead read bob-cli-3k -r "<why>"`).**
  - When a bare, counted, or transcluded-target status cycle moves a task into a Done
    type, collect the closed identities and call `finalizeClosedTasks`, the same as
    Ctrl+Enter, presenting the notice the same way.
  - When it moves a task into Cancelled, call `finalizeClosedTasks` with
    `{ closeKind: "cancelled" }`. That runs recovery only, never links (rows use
    `not_linked: "cancelled"`). A card appears only when something recovered.
  - First check the README and `docs/task-status-hooks.md` ("intentionally narrower") in
    case cycling was deliberately excluded. `contract` already reworded the bob-cli
    side, so update any remaining plugin wording.
- **Tests.**
  - Receipt: reopen takes back untouched links; edited links and edited lines survive; a
    different day is a no-op; and a capture-closed predecessor shows no undo.
  - Alt+] on a planned prerequisite links its successor.
  - Alt+[ recovers its dependents without linking.
- **Ship.**
  - Bump the cycler minor version and update the README and manifest.
  - Run `npm run build`, then `npm test`, then `bob plugins sync`.
  - Close bob-cli-3k with `sase bead close bob-cli-3k --note "<commit + tests>"`.

# closeout

Verify and record; there is no new feature code. Fix forward in the owning repo only for
real defects found here.

- **CLI sandbox.**
  - Build bob-cli master.
  - In a temp copy of the fixture vault, replay placement examples (a)–(d) with
    `bob capture` (dry run, then real).
  - Paste the human output into the epic bead note.
- **Timings.** Re-run the apollo timings on a temp copy of `~/bob` for `hello`, `=x`, a
  `!` prerequisite close, and an `=x!1` prerequisite close, and note them against the
  targets.
- **Plugins.**
  - `npm test` is green in bob-plugins.
  - `bob plugins sync` was run from master.
  - Versions in the manifests match the README table.
- **Mac.** The latest bob-mac-capture CI run on master is green.
- **Beads.** Confirm `bob-cli-5v` and `bob-cli-3k` are closed.
- **Memory.**
  - > [!decision] decision_record
    >
    > Via `/sase_memory_write`, add the decisions strand
    > `closed-task-hands-slot-to-successors`, titled "A Closed Planned Task Hands Its
    > Slot To Its Unblocked Dependents". It has:
    >
    > - **Applies to.** bob-cli, bob-plugins, Bob Mac Capture.
    > - **Claim.** Trigger, eligibility, placement, writes, notice, kill switch.
    > - **Why.** The research ref and this plan.
    > - **Rejected alternatives.** From the research table:
    >   - link every dependent;
    >   - suggest-only;
    >   - link only one successor;
    >   - ghost rows;
    >   - reconcile adds links;
    >   - a THEN grammar;
    >   - Obsidian shelling out;
    >   - Swift computing dependents;
    >   - a persistent index.
    > - **Cost.** Sticky Next from a wrong successor; minted IDs; the continuation
    >   becoming next up.
    > - **Reopens when.** More than about a third of successor links are released the
    >   same day.
    > - **Links.** `[[decisions/task-lanes-are-sticky]]`,
    >   `[[decisions/task-deps-are-depends-on-links]]`,
    >   `[[decisions/today-is-read-from-the-ledger]]`.
    >
    > Then run `sase memory init`.
  - > [!decision] glossary_term
    >
    > Via `/sase_memory_write`, add the glossary strand `successor-link`: "The plain
    > Task Link a Bob close gesture inserts for a dependent it fully unblocked, right
    > after the predecessor's link or in its same-name continuation; the dependent
    > becomes Next…". Link it to `[[task-link]]` and `[[task-dependency-link]]`, then
    > run `sase memory init`.
  - When a memory decision is no, do nothing for it.

# Non-goals

- No recursion and no chain linking (SL13).
- No new grammar (`⏭️ THEN:`), no new subcommand, no CLI option.
- No daemon, persistent index, or SQLite.
- The hooks never add links, and reconcile behavior is unchanged.
- No freshness, schedule, Schedule Log, or `[id::]` writes by the successor pass.
- Cancel, reopen, park, raw checkbox clicks, and hooks never link. Reopen only takes
  back through the receipt.
- No Swift-side dependency or placement logic.
- `plan.strict` stays theme-only, and successors never trip it.
- No managed Depends-On line guard in v1.
- Older `bob` / older app combinations keep working, because every field is additive.

# Acceptance

- **Ctrl+Enter in Obsidian.** Ctrl+Enter on `[[sase#^fix-apollo]]` in today's FIX does
  all of the following, and the notice appears about 50 ms after the keypress on a Warm
  Tasks cache:
  - strikes it and inserts `[[sase#^<successor>]]` on the next line;
  - flips the dependent `[?]` → `[*]` with no `[fresh::]`;
  - shows one `Unblocked · 1 → FIX` card naming the predecessor, with the plan chip and
    the `Alt+N releases` hint.
- **`!` capture.** `bob capture '!sase:fix-apollo'` while BOB runs:
  - moves the struck link into BOB;
  - leaves the successor in FIX's vacated slot;
  - reports it in JSON (`link.entry_name: "FIX"`) and in the human `linked` row.

  Bob Mac Capture's preview shows `[?] → [*] … → FIX` before Return, and its
  notification carries one 🔓 line.

- **`=!` close.** `=!` on a session whose last link unblocks a dependent creates
  `- [ ] () — NAME` holding only the successor. JSON says `entry_created` and `next_up`,
  and every surface says "new NAME session · next up".
- **Edge cases.** Still-blocked, scheduled, already-planned, project, hidden, breaker,
  and disabled cases link nothing and say why. A close with no `[id::]` reads nothing
  extra.
- **Timings on apollo.** `hello` / `=x` dry runs take ≤ 40 ms. A `!` / `=x!` closing a
  real prerequisite takes ≤ 70 ms.
- **Vectors.** SL1–SL28 and the SB vectors pass in both Rust and JS. `just check`,
  bob-plugins `npm test`, and bob-mac-capture macOS CI are green, and the plugins are
  synced.
