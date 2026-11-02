---
tier: epic
title: Keep leaf projects moving — offer a sub-project's ready tasks when its last
  planned task closes
goal: 'When any managed Done gesture, on any surface, closes the task behind a leaf
  project''s last live Task Link in today''s Pomodoros, Bob offers that project''s
  ready tasks in one picker. The picker says plainly why it appeared and that the
  rows are not dependents. Rust and the JS cycler apply one rule, pinned by shared
  vectors. Obsidian, `bob capture`, and Bob Mac Capture present it with one copy and
  visual system.

  '
phases:
- id: contract
  title: Contract, glossary, and docs
  depends_on: []
  size: medium
  description: 'contract: pin the leaf-project definition, the keep_moving trigger,
    rule, reporting, copy, kill switch, and KM/LP vectors in bob-cli docs, and apply
    the accepted memory decisions.'
- id: rust
  title: Rust engine and capture reporting
  depends_on:
  - contract
  size: large
  description: 'rust: implement leaf detection, the last_planned trigger, keep_moving
    offers, picks, net pass, plan.keep_moving, and the JSON and human output in bob
    capture.'
- id: nav
  title: Obsidian picker and notice presentation
  depends_on:
  - contract
  size: medium
  description: 'nav: teach bob-navigation-hotkeys'' successor picker (v2) and Unblocked
    card (notice v2) to render keep_moving groups, captions, folds, and lane names.'
- id: cycler
  title: Obsidian cycler engine
  depends_on:
  - contract
  - nav
  size: large
  description: 'cycler: port the rule to task-status-cycler for every Obsidian close
    surface. Covers leaf detection, gates, offers, the picker model, notice text,
    the receipt, and the config loader.'
- id: mac
  title: Bob Mac Capture presentation
  depends_on:
  - rust
  size: medium
  description: 'mac: decode keep_moving arrays and present them in the Pull-In card,
    preview cards, block-diff badges, and notification line, with fixtures from a
    real bob.'
decisions:
  offer_lanes:
    ask: Which lanes should a keep-moving offer list from the leaf project?
    choices:
      open: Pending, Next, and Ready tasks, in that pull order
      ready: Only Ready tasks; Next and Pending residents are left out
    default: open
    why: Next and Pending tasks are already commitments, so leaving them out hides
      the likeliest pick
    answer: open
  memory_glossary:
    ask: Add the Leaf Project glossary term and extend Successor Offer and Task Successor?
    memory:
    - glossary
    default: true
    requested: We should add a good new glossary term for "leaf project".
    answer: true
  memory_decision_record:
    ask: Record this design as a new decision record, decisions:leaf-projects-keep-moving?
    memory:
    - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.bz
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 12:29:32
status: done
bead_id: bob-cli-fb
---

- **PROMPT:** [prompts/202611/leaf_project_keep_moving_offers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/leaf_project_keep_moving_offers.md)
- **BEAD:** [bob-cli-fb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fb/README.md)

# Plan: Keep leaf projects moving

## Context

Bryan wants a prompt whenever a close drains today's plan for a sub-project. If the
closed task was the last live Task Link in today's daily note pointing at that project,
Bob should offer the project's ready tasks as successors. This should happen from every
surface. It applies only to **leaf projects**, which Bryan defines as project notes that
are not root parents. The prompt must make clear why it appeared, so the rows are never
mistaken for task dependents. Bryan asked for a new glossary term for "leaf project". He
also asked the design to be intuitive, reliable, and beautiful.

### Verified facts that shape the work

- **The offer infrastructure already exists.** `docs/task-dependencies.md` §12
  ("Successor offers", §12.1–§12.14) pins one rule for two engines. It has three groups:
  `hand_off`, `project`, and `project_finished`. Rows carry the triggers `dependency`,
  `project_empty`, and `project_created`. Selection uses offer keys, an explicit pick,
  no preselection, and per-row revalidation. Every pick does the same writes:
  - raise to Next;
  - mint a missing `^block-id`;
  - stamp `[fresh:: today]`;
  - insert the Task Link.

  The accepted decision is `decisions:unblocked-tasks-are-offered`. Its design came from
  epic `bob-cli-7c` (`plan:202610/successor_offers.md`). This feature adds a fourth
  offer class to that machinery. It does not build a second mechanism.

- **Rust engine.** `src/native/task_complete/successors.rs` (2755 lines) holds:
  - `plan_successors` → `plan_successors_inner`;
  - `OfferGroup`;
  - `DayView`, whose `live` map is keyed by `(note path, block id)` and lists live links
    resolved through `NoteIndex`;
  - anchors, `route_successor_offers` (D7 type routing), keys, numbering, selection, and
    placement.

  Pull-ins live in `project_pull_in.rs`, and project recovery lives in
  `project_close.rs`. There are two callers, each with its own cheap pre-gate:
  - `src/native/capture/task_complete.rs`, used by `!` and complete-label commits.
    `pre_day_text` is the baseline, and the planner runs only when
    `gate.has_candidates || gate.needs_dependents`.
  - `src/native/capture/pomodoro_close.rs`, used by `x…` closes. It has both `pre_day`
    and `post_day`.

  The batch net pass (SL20) lives in `src/native/capture/plan.rs`. JSON and human output
  live in `src/native/capture/output.rs`, Pomodoro block-diff reasons in
  `capture/pomodoro_blocks.rs` (`reason = Some("unblocked")`), and task-block roles in
  `capture/task_blocks.rs` (`TaskBlockRole`, which the app does not depend on). Config
  is `src/native/config/plan.rs` (`link_unblocked`).

- **The root-parent rule already exists in Rust.**
  `src/native/pomodoro_homes/resolve.rs` has
  `HomeResolver::resolve_note(path) -> Home {kind, key, root_parent, …}`. Its
  live-surface backend is `DiskHomeSource`, and `successors.rs` already builds a
  `HomeResolver<DiskHomeSource>` for type routing. `docs/pomodoro-homes.md` R2 is the
  rule:
  - an area at any depth is its own root;
  - otherwise the root is the first project with `root: true` on the walk up to the
    first area;
  - otherwise it is the outermost project;
  - non-area, non-project wrappers are walked through;
  - the walk is capped by `MAX_PROJECT_DEPTH = 16`.
- **JS engine.** These paths are relative to bob-plugins
  `plugins/task-status-cycler/src/`.
  - **Entry.** Every Obsidian Done gesture funnels through `finalizeClosedTasks`
    (`130-plugin-references.js`). That calls `planAndApplySuccessorsNow`
    (`137-plugin-successor-project.js`), which plans, awaits `api.successors.pick`,
    re-reads, re-plans with the picks, and applies.
  - **Gate.** That function's early `quietGate` return fires when there are no ids, no
    pull-in paths, and no parent candidates. It must learn the new pre-gate.
  - **Rule port.** The rule is ported in `085-successor-plan.js` (999 lines, at the
    1000-line fragment cap), `085b-successor-offers.js`, and
    `089*-successor-project.js`. The picker model comes from `buildSuccessorPickerModel`
    in `085b`, which hard-codes group headers and order. Notice text is
    `successorNoticeText` (`087-successor-place.js`), and the reopen receipt is
    `136-plugin-successor-receipt.js`.
  - **Callers.**
    - Ctrl+Enter variants (`140`, `160`, `170`).
    - Alt+] / Alt+[ (`finalizeCycledClosedTasks`).
    - The walk's `completeTaskAtCursor`.
    - nav's Links-stage complete-label commit (`597-relation-complete-compose.js` →
      `api.finalizeClosedTasks`).
  - **Parent resolution.** The cycler resolves only one parent hop
    (`resolveProjectParentPath`, `084`). The R2 root-parent walk exists in JS only
    inside bob-ledger-tools (`pomodoroHomeResolve`, metadataCache-based). Plugins may
    not import each other, so the cycler must carry its own port.
- **Nav presentation.** These paths are relative to bob-plugins
  `plugins/bob-navigation-hotkeys/src/`.
  - `normalizeSuccessorPickerModel` (`206a`) hard-codes the allowed groups, so **an
    unknown group is silently dropped**.
  - `successorPickerGroupTitle` and `renderResults` live in
    `206-successor-picker-modal.js`.
  - The `Unblocked` / `Project empty` card is `285-unblocked-notice.js`.
  - Walk composition is `536`; while the picker is open,
    `holdReviewWalkForSuccessorPicker` holds the walk.
  - CSS is `.bob-successor-*` in `styles.css`.
  - Row details currently print the raw status symbol (`" "`, `"*"`).
  - `api.successors` and `api.notice` are both v1.
- **Mac presentation.** These paths are relative to bob-mac-capture.
  - All offer fields decode as plain `String`s with `decodeIfPresent`. A new group or
    trigger therefore never breaks decoding, but it degrades: the raw group string
    becomes the header, rows get a `trigger: …` caption, and the context reads
    "Unblocked by …".
  - Offers are gathered only from `taskComplete.unblocked`, `pomodoroClose.unblocked`,
    `projectNote.successors`, and `relationCompletes[].taskComplete`
    (`CapturePanelModel.makePullInPreviewSnapshot`).
  - Sections, titles, rank, and glyph live in `CapturePullInChoice.swift` and
    `CapturePullInChoiceView.swift`.
  - Preview reuse is `CapturePanelView.pullInPreviewSummary`.
  - The notification line is `CaptureSuccessorPresentation.notificationLine` /
    `NotificationService.pulledInSummary`.
  - Fixtures live in `Tests/Fixtures/*successors*.json`. They are regenerated from a
    real `bob` by `Scripts/generate-project-note-offer-fixtures.sh`.
- **One doc claim becomes false.** `docs/pomodoro-homes.md` R2 and `docs/projects.md`
  both call `root: true` "reporting only". After this change, `root: true` also decides
  leafness, so a root project never gets keep-moving offers. The contract phase corrects
  both.

## Shared design (binding for every phase)

### Vocabulary

- **Leaf project.** A project note that is **not its own root parent**. Leafness is
  computed with the R2 walk on the note tree and never stored. The rule:
  - A top-level project, a project marked `root: true`, and an orphan project (no
    project or area above it) are roots, so they are never leaves.
  - A sub-project at any depth without `root: true` is a leaf.
  - "Leaf" names the reporting role, not the tree position, so a leaf project may have
    sub-projects of its own.
  - A task's `[root::]` field never changes its note's leafness.
  - Areas, daily notes, inbox files, `done/` archives, and terminal projects are never
    leaves.
- **Keep-moving offer.** A successor offer in the new group `keep_moving`, with the new
  row trigger `last_planned`. It offers a ready task of a leaf project N because this
  gesture closed the task behind N's last live Task Link today. **It is never a
  dependent**, and every surface says so.
- **Live link.** The existing §12 term: an unstruck Task Link sub-bullet (plain, embed,
  or `#`-deferred) under an open `[ ]` entry of today's `## Pomodoros`. Planned `()`
  placeholders are open entries.

### Trigger (both engines, every managed Done gesture)

Read the trigger on the **same baseline §12.2 uses for anchors**: today's day text
before this gesture strikes, moves, retires, or drops anything, but after any explicit
link step of the same item. Read note statuses from the staged post-gesture text.

```text
C = tasks this gesture moved open → Done (roots + closed embedded subtasks)   (§12.2)

For each note N that holds some c ∈ C:
  K1  N is a project note that sync recognizes: outside done/, non-terminal, exactly
      one open ^prj R, no scan issue (the §12.10 recognition)
  K2  LEAF(N): the R2 root parent of N is not N        (Rust: HomeResolver::resolve_note)
  K3  PRED(N) = { c ∈ C : c lives in N, c ≠ R, c has a block ID, and c has its OWN live
                  link on the baseline }               — must be non-empty
  K4  no live link on the baseline targets an OPEN task of N outside C
      (the ^prj R counts as open N work; links to tasks already closed do not count;
       links to tasks in N's sub-projects do not count — residence is N only)
  K5  R ∉ C, and R is not Blocked: no own open prerequisite, no future SCHED(N), no
      #prj_is_blocked
  K6  N is not EMPTY(N) (§12.10). An emptied N is a project_finished pull-in instead.
  K7  this gesture produced no `project` group and no `project_finished` row for N
  K8  plan.link_unblocked and plan.keep_moving are both true, and the close is not a
      cancel
  ⇒ N opens a keep_moving group with PRED(N) as its predecessors (`unblocked_by`)
```

Two consequences follow from the baseline:

- A link that this same close carries forward (an incomplete task in a closed Pomodoro)
  keeps N planned, so nothing is offered.
- A link that this close drops with `x~K` also blocks the trigger: the drop wins,
  exactly as in §12.2.

A task closed outside its link still triggers, for example with Ctrl+Enter in its own
note or with Alt+] into Done, provided its own live link exists.

**Never triggers:**

- cancel of any kind (nav cancel, Alt+[ into Cancelled, capture cancel labels);
- reopen;
- raw checkbox clicks;
- `bob task reconcile` / hooks;
- `x*` parks;
- Ctrl+Enter on a Depends-On line;
- capture's `@route:id@` project creation, which closes nothing.

This is the §12.1 exclusion list, unchanged.

### Offer set

For a triggered N, classify every open `#task` line of N on the staged post-gesture
text, other than R and C:

| Resident                                                                         | Outcome                                                                          |
| -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| a direct dependent offered as `hand_off` by this gesture                         | stays in `hand_off` (dedup: hand_off > project > project_finished > keep_moving) |
| `[?]`, or own open prerequisite (`task_dependency_states`), or `#prj_is_blocked` | folded: `blocked`                                                                |
| `scheduled` after today                                                          | folded: `scheduled`                                                              |
| whole-token `#hide`                                                              | folded: `hidden`                                                                 |
| `[ ]` Ready, `[*]` Next, `[/]` Pending, none of the above                        | **offered** (see `offer_lanes`)                                                  |
| custom or unknown mark                                                           | ignored                                                                          |

> [!decision] offer_lanes = open Offer Pending, Next, and Ready residents. **Order:**
> lane in pull order (Pending → Next → Ready, as in `decisions:task-lanes-are-sticky`),
> then line in N.

> [!decision] offer_lanes = ready Offer only `[ ]` Ready residents, ordered by line in
> N. Next and Pending residents are neither offered nor folded.

The remaining rules for the offer set:

- **What gets offered.** `#ref` tasks, nested subtasks, and recurring tasks whose
  occurrence is today or past are offered like any resident. Residents of N's
  sub-projects are never offered.
- **Already planned.** By K4 no N resident is live-linked, so there is no
  `already_planned` fold.
- **No selectable rows.** When no row is selectable, the group is omitted entirely: no
  picker, no notice, and no JSON group. This is silence, as §12.6 requires.
- **Prerequisite checks.** A candidate that carries a non-empty `[dependsOn::]` needs
  the vault-wide dependents snapshot to prove its prerequisites are closed. Build the
  snapshot only in that case, using the §12.10 gate pattern. If it is unavailable, fold
  those candidates as `blocked`, because Bob must never offer a task it cannot prove
  unblocked.
- **Keys, numbering, selection, picks.** These reuse §12.2 KEYS, the existing numbering
  (`n`), and SELECTION.
  - Ordering puts `keep_moving` after every other group, including `project_created`
    rows. Within the group, rows sort by anchor ledger position, then note path, then
    the lane order above.
  - Destination follows the §12.3 anchors over PRED(N): `slot` places the row right
    after the closed task's bullet, and `closing` is used for Pomodoro closes. D7 type
    routing applies as for any offer, through `route_successor_offers` and its JS twin.
    K3 guarantees an anchor, so there is no implicit fallback.
- **Picks** write exactly like a picked `hand_off`:
  - `' '` → `'*'`; `'*'` and `'/'` stay, so Pending is never demoted;
  - mint a missing block ID;
  - stamp `[fresh:: today]`, clearing `[unblocked::]` and `[keeps::]`;
  - insert the plain Task Link.
- **Unpicked rows.** Nothing is written: no recovery, no `[unblocked::]`, no mint.
- **Re-validation.** A picked row that has since become Blocked, linked, or closed is
  refused per row (SO30).
- **Multi-item capture drafts** extend the SL20 net pass:
  - a later item that links, completes, re-blocks, or drops an offered task drops that
    row;
  - a later item that links any open N task, or closes R, drops N's whole group.

### Reporting (additive; capture JSON stays schema version 1)

- **The `keep_moving` array.** A new array `keep_moving` is added to every
  `task_complete` and `pomodoro_close` object, always present and possibly empty. Its
  rows use the **exact `unblocked[]` row shape**, with:
  - `trigger: "last_planned"`;
  - `offer.group: "keep_moving"` and `offer.project: <N path>`;
  - `unblocked_by` = PRED(N), keeping the row shape's predecessor field;
  - `previous_status_*` equal to `status_*` unless picked;
  - `link` set only when picked;
  - `not_linked: "not_selected"` when unpicked.

  Keep-moving rows **never** appear in `unblocked[]`. That keeps every existing consumer
  of `unblocked[]` truthful. Nested `relation_completes[].task_complete` carries the
  array automatically.

- **`pull_in`.**
  - `offer_count` and `picked_count` include keep-moving offers.
  - `groups` gains one entry per group:
    `{"group": "keep_moving", "project": "<N path>", "folded": {"hidden": h, "already_planned": 0, "sub_projects": 0, "blocked": b, "scheduled": s}}`.
    `blocked` and `scheduled` are additive folded keys.
  - The `pull_in_choice_required` refusal and its flat `offers` payload include
    keep-moving offers.
  - `-p` numbers and keys select them like any other offer.
- **Diffs and blocks.**
  - In Pomodoro block diffs, an added line that is a surviving keep-moving pick gets
    `reason: "keep_moving"`; dependency picks keep `"unblocked"`.
  - `TaskBlockRole` gains `KeepMoving` (`"keep_moving"`).
- **Dry run.** Dry-run JSON equals real-run JSON with the same flags.
- **No breaking change.** This is additive JSON, so commits do not need `!`.

### Copy and visual system (identical on every surface)

‹N› is N's note stem, and ‹T› is the predecessor's cleaned text, truncated to 48
characters with `…`. Keep-moving surfaces always lead with **why** and always say **not
dependents**. They never use 🔓 or the word "unblocked". Their glyph is **⏭**:

- emoji/text: ⏭;
- Obsidian lucide: `skip-forward`;
- Mac SF Symbol: `forward.end.fill`.

They use the existing **Next** status colour token, because picks land in Next. No new
hex values are added.

| Element                                        | Copy                                                                                |
| ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| Group title                                    | `Keep ‹N› moving`                                                                   |
| Why caption, one predecessor                   | `✓ ‹T› was the last ‹N› task planned today`                                         |
| Why caption, k ≥ 2 predecessors                | `✓ ‹T› and k−1 more were the last ‹N› tasks planned today`                          |
| Contrast tag                                   | `ready in ‹N› · not dependents`                                                     |
| Fold line (omit zero parts)                    | `2 blocked · 1 scheduled later · 1 hidden`                                          |
| Picker title, only keep_moving groups          | `Keep ‹N› moving` / `Keep 2 projects moving` (else the existing title)              |
| Notice, one pick                               | `⏭ Next in FIX: Ship the badge renderer · keeps ‹N› moving`                        |
| Notice, several picks                          | `⏭ 2 linked → FIX · keeps ‹N› moving`                                              |
| Notice, mixed with dependency rows             | existing 🔓 forms; keep-moving picks count as linked                                |
| Notice, nothing picked and nothing else to say | silence                                                                             |
| Nav card title, every row keep_moving          | `Keep ‹N› moving` with ⏭ in the glyph slot                                         |
| Nav card row chip in a mixed card              | `keeps ‹N› moving`                                                                  |
| CLI group header                               | `Keep N moving · ‹T› was its last task planned today (ready tasks, not dependents)` |
| CLI linked suffix                              | `· keeps N moving`                                                                  |
| CLI fold row                                   | `not offered  2 blocked · 1 scheduled later · 1 hidden`                             |
| Mac capsule / section                          | `Keep moving` / `Keep ‹N› moving`                                                   |
| Mac notification line                          | `⏭ Pulled in 1 · keeps ‹N› moving`                                                 |

Rows on every surface show the lane name (`Pending`, `Next`, `Ready`, `Blocked`) and
never a raw status symbol.

**Obsidian picker (only keep_moving groups):**

```text
┌──────────────────────────────────────────────────────────────────────┐
│ ⏭  Keep ‹sase_badges› moving                                          │
│ [ Filter tasks, notes, and groups                                  ]  │
│ ┃ Keep ‹sase_badges› moving        ready in sase_badges · not dependents │
│ ┃ ✓ Draft the badge spec was the last sase_badges task planned today  │
│ ┃ □ Ship the badge renderer                                           │
│ ┃   sase_badges · Pending · → FIX                                     │
│ ┃ □ Wire badges into the README                                       │
│ ┃   sase_badges · Next · → FIX                                        │
│ ┃ □ Write badge docs                                                  │
│ ┃   sase_badges · Ready · → FIX · needs ^id                           │
│ ┃   2 blocked · 1 scheduled later · 1 hidden                          │
│  plan 6/10 → 7/10                                                     │
│  Tab toggle and move · Ctrl+A mark group · ↵ pull in · Esc none        │
└──────────────────────────────────────────────────────────────────────┘
```

The `┃` rule is a left accent in the Next token. In a mixed picker the keep_moving group
renders last, below `Hand-offs` and the project groups, with the same heading, caption,
and tag. Its contrast with the dependents above is the point.

**Obsidian card after a pick:**

```text
┌──────────────────────────────────────────────────────────────────┐
│ ⏭  Keep ‹sase_badges› moving   1 → FIX      ✓ Draft the badge spec │
│  ＋ Ship the badge renderer                           [ Pending ]  │
│ ──────────────────────────────────────────────────────────────── │
│  (plan 7/10 · 7/10)  (Alt+N releases)                             │
└──────────────────────────────────────────────────────────────────┘
```

**`bob capture` human output:**

```text
✓ completed [*] → [x] Draft the badge spec  sase_badges.md ^draft-spec
  ledger  Task Link struck in FIX · 2026/20261101.md
  Keep sase_badges moving · Draft the badge spec was its last task planned today (ready tasks, not dependents)
  offer 1  [/] Ship the badge renderer  sase_badges.md ^ship-renderer → FIX
  offer 2  [*] Wire badges into the README  sase_badges.md ^readme-badges → FIX
  offer 3  [ ] Write badge docs  sase_badges.md → FIX
  not offered  2 blocked · 1 scheduled later · 1 hidden
```

The existing dry-run hint (`-p 1,2 · -p all`) and the real-run "not pulled in" line keep
working. A pick prints
`linked [ ] → [*] Write badge docs  sase_badges.md ^write-badge-docs → FIX · keeps sase_badges moving · added ^write-badge-docs`.

**Bob Mac Capture Pull-In card:**

```text
 ⏭ Keep moving                                                (capsule)
 Draft the badge spec was the last sase_badges task planned today · ready tasks, not dependents
 ⏭ Keep ‹sase_badges› moving
   ☐ 1  Ship the badge renderer          Pending   → FIX
   ☐ 2  Wire badges into the README      Next      → FIX
   ☐ 3  Write badge docs                 Ready     → FIX
   2 blocked · 1 scheduled later · 1 hidden
```

Preview cards (`!` / `x`) get the same section with the hint
`↩ choose what to pull in`. Block-diff lines with `reason == "keep_moving"` get a
trailing `forward.end.fill` badge with `.help("Keeps ‹N› moving")`.

### Kill switch

`plan.keep_moving: true` in `~/.config/bob/config.yml` controls only this group.

- Rust and the cycler both read it.
- `false` disables the trigger.
- An invalid value falls back to `true` through the same warning paths as
  `link_unblocked`.
- `plan.link_unblocked: false` already turns off all offering, including this group.

### Walk, undo, and plan budget

These rules are unchanged and apply as-is:

- **Walk.** The picker holds the review-walk gesture lock. After it settles, Esc
  included, the walk advances exactly once, and a pick composes into the single walk
  toast line.
- **Undo.** A same-day Obsidian reopen of a predecessor takes back its untouched
  keep-moving links and restores the picked rows' lines from the receipt. Alt+N releases
  one pick. Rust has no reopen take-back.
- **Plan budget.** The plan meter warns and never refuses.

### Gates and performance

Order the checks from cheapest to most expensive, in both engines:

1. **G1, no I/O.** Some c ∈ C has a block ID, is not a `^prj`, and lives in a note whose
   held staged text has project frontmatter and is non-terminal.
2. **G2, no I/O.** N's staged text has at least one candidate: open `' '`, `'*'`, or
   `'/'`, not `#hide`, not future-scheduled, and neither R nor in C.
3. **G3, byte check on the day text.** The day text contains `#^<c.block_id>`. Capture
   already holds the day text. The cycler reads the daily note only now, and only for
   project-note closes.
4. **G4.** LEAF(N), using frontmatter reads of N's ancestors only.
5. **G5.** K3 and K4 on the `DayView` / live-link parse. This is the first step that may
   need the `NoteIndex`.

Budgets:

- Closes outside project notes, and closes whose task is not linked today, add zero I/O.
  The cycler's zero-read wiring test must keep passing.
- A triggering `!` on apollo, built in release mode against a **temporary copy** of
  `~/bob`, stays within §12.7's ≤ 70 ms target.

### Interplay with accepted decisions

- **`decisions:unblocked-tasks-are-offered`.** This change extends that decision:
  nothing is auto-linked or preselected, there is one picker per gesture, and there is
  no cap.
- **`decisions:task-lanes-are-sticky`.** A pick raises the task to Next like any link.
- **`decisions:today-is-read-from-the-ledger`.** Planned work is read from live links at
  gesture time and never stored.
- **`decisions:mac-capture-is-a-thin-client`.** bob computes and writes; the Mac
  presents.
- **`decisions:bob-cli-releases-ride-a-daily-train`.** No phase edits `Cargo.toml`'s
  version or `CHANGELOG.md`.

### Why this shape (rejected alternatives)

- **Every project, roots and areas included.** Roots are broad containers whose daily
  links routinely run out, so a prompt there would be noise. Bryan scoped the feature to
  leaf projects.
- **Reuse `unblocked[]` for the rows.** Every existing client would then label these
  rows "unblocked". The Mac's "Unblocked by …" text and the `· unblocked A, B`
  notification are examples, and that mislabeling is exactly the confusion Bryan ruled
  out. `project_note.successors` is the precedent for a separate array with the same row
  shape.
- **Read the trigger on the post-gesture day.** That would re-offer work that an `x~K`
  drop just removed, and it would diverge from the §12.2 baseline.
- **Muted rows for blocked residents.** They add noise to a "what's next" choice. One
  fold line still accounts for them.
- **Offer on cancel.** That would break §12.1's one rule that cancel never offers, for
  every group.
- **Auto-link the first ready task.** Rejected by
  `decisions:unblocked-tasks-are-offered`: even one candidate needs an explicit choice.

## Conformance vectors (the contract copies these into `docs/task-dependencies.md`)

### LP — leaf vectors (both engines; drawn from `docs/pomodoro-homes.md` R2)

| #    | Note                                                            | Leaf?                                |
| ---- | --------------------------------------------------------------- | ------------------------------------ |
| LP1  | `sase_goals.md` (project → `sase` project → `dev` area)         | leaf                                 |
| LP2  | `sase.md` (project → `dev` area)                                | no (top-level)                       |
| LP3  | `sase_blog.md` (project, `root: true`, → `sase`)                | no (root project)                    |
| LP4  | `sase_pre_blog.md` (project → `sase_blog`)                      | leaf                                 |
| LP5  | `p.md` (project → sub-area `gtd_daily`)                         | no                                   |
| LP6  | orphan project `o.md` (no `parent`)                             | no (outermost project)               |
| LP7  | project whose `parent` is unresolved or ambiguous               | no (the walk stops; it is outermost) |
| LP8  | project `k.md` → untyped wrapper note → `sase`                  | leaf (wrappers are walked through)   |
| LP9  | `m.md` (project → `sase`) that has its own child project        | leaf ("leaf" is the reporting role)  |
| LP10 | task in `sase_goals.md` with `[root:: sase_blog]`               | its note is still a leaf             |
| LP11 | area, daily note, inbox file, `done/` archive, terminal project | never a leaf                         |
| LP12 | `parent` cycle `a ↔ b`, neither with `root: true`              | leaf (matches `resolve_note`)        |

### KM — keep-moving vectors (select-all unless a row says otherwise)

All rows use leaf N = `sase_badges.md` (→ `sase` → `dev`). Its `^prj` R is open and
unblocked. Its predecessor T has a live link in FIX.

| #    | Situation                                                                         | Outcome                                                                                                                                                                                                                                                       |
| ---- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| KM1  | N has Ready A, Next B, Pending P; close T with `!N:t`                             | `keep_moving` group for N: P, B, A (lane order; A only under `offer_lanes = ready`); `trigger: last_planned`; `unblocked_by: [T]`; slot after T's bullet; nothing written without `-p`; pick A → `[*]`, minted ID, `[fresh:: today]`, linked after T's bullet |
| KM2  | N's U (open) is also live-linked today                                            | nothing                                                                                                                                                                                                                                                       |
| KM3  | U's live link targets a task already Done                                         | group (closed targets do not count)                                                                                                                                                                                                                           |
| KM4  | U's link is struck, or sits under a closed Pomodoro                               | group                                                                                                                                                                                                                                                         |
| KM5  | R itself is live-linked today                                                     | nothing                                                                                                                                                                                                                                                       |
| KM6  | N is top-level, `root: true`, or orphan (LP2/3/6)                                 | nothing                                                                                                                                                                                                                                                       |
| KM7  | N is a middle project with child K (LP9); K's tasks are Ready                     | group lists N's residents only                                                                                                                                                                                                                                |
| KM8  | T has no live link (not planned today), even though no N work is planned          | nothing                                                                                                                                                                                                                                                       |
| KM9  | T's link sits under a planned `()` placeholder                                    | group; destination is that placeholder                                                                                                                                                                                                                        |
| KM10 | `x` closes FIX completing T; U in FIX is incomplete and carried                   | nothing                                                                                                                                                                                                                                                       |
| KM11 | `x` closes FIX completing T only                                                  | group at the §12.3 closing anchor (continuation, same-name entry, or created placeholder)                                                                                                                                                                     |
| KM12 | the same close drops U with `x~K`                                                 | nothing (the drop wins; pre-gesture baseline)                                                                                                                                                                                                                 |
| KM13 | every remaining resident is Blocked, hidden, or future-dated                      | no group, no picker, silence                                                                                                                                                                                                                                  |
| KM14 | T was N's last open task                                                          | `project_finished` pull-in only (§12.10); no group                                                                                                                                                                                                            |
| KM15 | A is also a direct dependent of T                                                 | one row, in `hand_off`                                                                                                                                                                                                                                        |
| KM16 | R has an open prerequisite, a future date, or `#prj_is_blocked`                   | nothing                                                                                                                                                                                                                                                       |
| KM17 | resident `[?]` / Ready with an open prerequisite / `scheduled` tomorrow / `#hide` | folded as `blocked` / `blocked` / `scheduled` / `hidden`; `scheduled` today is offered                                                                                                                                                                        |
| KM18 | `#ref` resident, nested subtask, recurring task due today                         | offered                                                                                                                                                                                                                                                       |
| KM19 | `plan.keep_moving: false`, or `plan.link_unblocked: false`                        | no group; other groups follow their own switch                                                                                                                                                                                                                |
| KM20 | any cancel, reopen, `x*` park, raw checkbox click, or reconcile                   | nothing                                                                                                                                                                                                                                                       |
| KM21 | draft `!N:t` then `@route:a` (links A)                                            | the net pass drops N's group; a later item completing B drops only B's row                                                                                                                                                                                    |
| KM22 | one `x` drains two leaf projects                                                  | two groups, ordered by anchor then path                                                                                                                                                                                                                       |
| KM23 | T carries `[root:: sase]`                                                         | still triggers (leafness is by note)                                                                                                                                                                                                                          |
| KM24 | picked A became Blocked or linked before apply (cycler re-plan)                   | A refused per row; others apply                                                                                                                                                                                                                               |
| KM25 | Esc / `-p none`                                                                   | nothing written, no IDs minted, no notice                                                                                                                                                                                                                     |
| KM26 | Obsidian: T closed with Ctrl+Enter in its own note, or Alt+] into Done            | group                                                                                                                                                                                                                                                         |
| KM27 | walk-landing Ctrl+Enter                                                           | picker holds the gesture lock; the walk advances once after it settles                                                                                                                                                                                        |
| KM28 | same-day Obsidian reopen of T after picking A                                     | A's untouched link is taken back, and A's line is restored                                                                                                                                                                                                    |
| KM29 | T is R                                                                            | nothing (the project closed)                                                                                                                                                                                                                                  |
| KM30 | only keep-moving offers, plus `--require-pull-in-choice` without `-p`             | write-free refusal `pull_in_choice_required` listing them                                                                                                                                                                                                     |
| KM31 | a candidate carries `[dependsOn::]` and the dependents snapshot is unavailable    | that candidate is folded `blocked`; the others are offered                                                                                                                                                                                                    |
| KM32 | close of a task outside any project note                                          | zero added reads (wiring test)                                                                                                                                                                                                                                |

## Phase: contract — Contract, glossary, and docs

This phase is bob-cli only and changes no code. Read `docs.md` reference memory first.
The phase writes the shared design above into the contract, verbatim where this plan
gives copy:

1. **`docs/task-dependencies.md`.**
   - Add **§12.15 Keep-moving offers (leaf projects)**. It holds the vocabulary, trigger
     (K1–K8), offer set, keys and order, destination, writes, reporting, copy, kill
     switch, gates and performance, walk, undo, and interplay.
   - Add **§12.16 KM — keep-moving vectors** and **§12.17 LP — leaf vectors**.
   - Insert these after §12.14 so nothing is renumbered.
   - Amend the existing sections with short cross-references:
     - §12 vocabulary: add `keep_moving` to the Groups and Offer (D) bullets.
     - §12.1: add that every managed Done row also runs the keep-moving trigger. Cancel
       and the other exclusions stay unchanged.
     - §12.2: add the DEDUP priority and the ORDER placement.
     - §12.5: add the `keep_moving[]` array, `trigger: "last_planned"`, the
       `offer.group` value, the `pull_in.groups` folded keys, the block-diff
       `reason: "keep_moving"`, and the task-block role.
     - §12.6: add the copy table and the four mocks.
     - §12.8: add `plan.keep_moving`.
     - §12.13: document **picker API v2**. Groups may be `keep_moving` and carry
       `caption` and `tag`. Folded gains `blocked` and `scheduled`. Rows gain a `lane`
       string. The cycler offers keep_moving groups only when
       `api.successors.version >= 2`. Also document **notice API v2**, where the model
       gains `keep_moving: [rows]`.
2. **`docs/projects.md`.** Add a `### Leaf projects` subsection that defines the term
   and links §12.15. Correct "It is reporting only" for `root: true`: a root project is
   never a leaf, so it never gets keep-moving offers.
3. **`docs/pomodoro-homes.md` R2.**
   - Add the leaf definition sentence.
   - Amend the claim "Reporting only: it changes no blocking, …" to name the single
     workflow effect: a `root: true` project is never a leaf, so it gets no keep-moving
     offers.
4. **`docs/plan.md`.** Add `keep_moving: true` to the Config block with this comment:
   `# offer a leaf project's ready tasks when you close its last task planned today (docs/task-dependencies.md §12.15); false turns off only these offers`.
   Add it to the boolean validation and to the invalid-value fallback bullet.
5. **`docs/capture.md`.** In the successor-offer output passage, add the `keep_moving`
   array and the human header, and link §12.15.
6. **Memory**, each step only as its decision allows. Apply each accepted change with
   the `/sase_memory_write` skill.

   > [!decision] memory_glossary Make three glossary edits.
   >
   > **New term.** Create `sase/memory/glossary/leaf-project.md` with
   > `keyword: Leaf Project` and summary "A project note that is not its own root parent
   > — a sub-project whose time reports under a root project above it." Body: a
   > [[project-note]] that is not its own [[root-parent]], meaning a sub-project at any
   > depth whose `parent` walk meets another project before any [[area-note]] and that
   > does not set `root: true` itself. A [[top-level-project]], a `root: true` project,
   > and an orphan project (nothing above it) are roots, never leaves. "Leaf" names the
   > reporting role, not tree position: a leaf project may have sub-projects. Leafness
   > is derived from the note tree at gesture time, never stored, and a task's
   > `[root::]` field never changes it. Closing the task behind a leaf project's last
   > live [[task-link]] in today's Pomodoros offers the project's ready tasks as
   > keep-moving [[successor-offer]]s, which are never dependents (see
   > `docs/task-dependencies.md` §12.15).
   >
   > **Successor Offer correction.** In `glossary/successor-offer.md`, the sentence "A
   > managed Done gesture offers the tasks it recovered, grouped as direct hand-offs,
   > residents of a recovered project, or project-finished pull-ins." becomes incomplete
   > once this epic ships. After the three recovered groups, add: "and, when it closes
   > the task behind a [[leaf-project]]'s last live Task Link today, that project's
   > ready tasks as keep-moving offers, which are never dependents."
   >
   > **Task Successor correction.** In `glossary/task-successor.md`, the sentence "The
   > successors of a managed Done gesture are the tasks it unblocks: …" is likewise no
   > longer complete. Extend it: "…and project-finished pull-ins, plus, when it closes
   > the task behind a [[leaf-project]]'s last live Task Link today, that project's
   > ready tasks, which need not depend on it."

   > [!decision] memory_decision_record Create
   > `sase/memory/decisions/leaf-projects-keep-moving.md`. Follow the format of
   > `decisions/unblocked-tasks-are-offered.md`: `metadata.status: accepted`; title
   > "Closing A Leaf Project's Last Planned Task Offers Its Ready Work"; and a summary
   > rule that fits the roster. Write the Applies to, Claim, Why, Cost, and Reopens when
   > sections from this plan's design and rejected alternatives. Evidence: Bryan's
   > request of 2026-11-01 and this epic. Link [[decisions:unblocked-tasks-are-offered]]
   > and [[glossary:leaf-project]]. This record does not supersede any existing record.

   Then run `sase memory init`.

7. Run `just docs-gen`, which regenerates `docs/reference/glossary.md` and
   `docs/develop/decisions.md`, then `just docs-check`.

**Done when:**

- The contract reads as one coherent §12 extension.
- Every KM and LP row is present.
- `just docs-check` passes.

## Phase: rust — Rust engine and capture reporting

Implement §12.15 in bob-cli exactly as the contract pins it.

1. **New module `src/native/task_complete/keep_moving.rs`.** Do not grow
   `successors.rs`. Re-export the module from `task_complete/mod.rs`. It contains:
   - `is_leaf_project(resolver, path)`, which uses `HomeResolver::resolve_note`: the
     kind is Project, the key is the path, and `root_parent` differs from the path.
   - The K1–K8 trigger over a `DayView` of the baseline.
   - Candidate classification and folds, which reuse `task_dependency_states`,
     `has_hide_tag`, `has_prj_is_blocked_tag`, and SCHED / EMPTY from
     `project_pull_in.rs`.
   - Offer drafting with `OfferGroup::KeepMoving` (`"keep_moving"`, ordered after
     `ProjectFinished`).
2. **Wire it into `plan_successors_inner`.** Run it after the hand_off, project, and
   pull-in verdicts exist, so DEDUP and K7 can see them. Run it before key assignment,
   numbering, selection, `route_successor_offers`, and placement, so keep-moving rows
   reuse all of those unchanged.
   - Add `SuccessorInput.keep_moving: bool`.
   - Add `SuccessorPlan.keep_moving: Vec<UnblockedRow>` with trigger `"last_planned"`,
     plus per-group folds.
   - `plan_recovery_only` (cancel) never runs it.
3. **Gates.** In `capture/task_complete.rs` and `capture/pomodoro_close.rs`, add a G1–G3
   pre-gate. When it passes, the planner (and the `NoteIndex`) runs even if
   `gate.has_candidates || gate.needs_dependents` is false. Build the dependents
   snapshot only per the KM31 rule. Keep both "nothing to do" paths I/O-free for the
   KM32 cases.
4. **Config.** Add `plan.keep_moving` in `src/native/config/plan.rs`: default true,
   boolean validation, and invalid → true plus the existing warning paths.
5. **Net pass.** Extend the SL20 net pass in `capture/plan.rs` for KM21.
6. **Output (`capture/output.rs` and friends).**
   - Add the `keep_moving` array on `task_complete` and `pomodoro_close`.
   - Count keep-moving offers in `pull_in` counts, and add the `pull_in.groups` entries
     with the `blocked` and `scheduled` folded keys.
   - Include them in the refusal's `offers` payload.
   - Add block-diff `reason: "keep_moving"` and `TaskBlockRole::KeepMoving`.
   - Set `day_file` when a keep-moving pick links.
   - Add the human header, offer rows, fold row, and linked suffix, verbatim from the
     copy table.
7. **Tests.**
   - Unit tests in `src/native/task_complete/tests/keep_moving_tests.rs` cover every KM
     row a Rust engine can express, plus LP1–LP12 against an in-memory `HomeSource`.
     Cite the vector ids in comments.
   - CLI tests in `tests/cli/capture/keep_moving.rs` cover JSON shape, numbering across
     groups, `-p` keys and numbers, the refusal, dry-run/real-run equality, human
     output, and the kill switch.
   - Existing SO, PP, and SL tests must stay green.
8. **Verify.**
   - Run `just` lint, test, and docs gates per the repo's `AGENTS.md`.
   - Measure §12.7 timing on apollo with a release build against a temporary copy of
     `~/bob` (never the live vault).
   - Do not edit the `Cargo.toml` version or `CHANGELOG.md`.

**Done when:**

- `bob capture '!sase_badges:draft-spec' --dry-run --format json` on a vault copy
  reports a `keep_moving` group shaped like KM1.
- All gates are green.

## Phase: nav — Obsidian picker and notice presentation

This is bob-plugins work only, in `plugins/bob-navigation-hotkeys`. Read the repo's
`AGENTS.md`/`CLAUDE.md`, which cover fragments, the 1000-line cap, and `npm run build`.

1. **Picker API v2.** Bump `api.successors.version` to 2.
   - `normalizeSuccessorPickerModel` (`206a`) accepts group `keep_moving`, optional
     group `caption` and `tag`, folded `blocked` and `scheduled`, and an optional row
     `lane`.
   - Unknown groups are still dropped.
   - v1 models render exactly as before.
2. **Rendering (`206`).**
   - `successorPickerGroupTitle` returns `Keep ‹N› moving` for keep_moving.
   - The heading shows the caption line and the right-aligned contrast tag.
   - Fold lines render `blocked` and `scheduled`
     (`2 blocked · 1 scheduled later · 1 hidden`).
   - The modal `headerIcon` is `skip-forward` when every group is keep_moving.
   - Row details show the lane name for **every** group, deriving it from `status` when
     `lane` is absent, so the raw symbol is never shown.
   - Add CSS under `.bob-successor-group.is-keep_moving`: a left accent rule, a caption,
     and a tag, using the existing Next status token and no new hex values.
3. **Notice v2 (`285-unblocked-notice.js`).** Bump `api.notice.version` to 2.
   `buildUnblockedNoticeModel` reads an optional `keep_moving` row array.
   - When every row is keep_moving: the title is `Keep ‹N› moving` with ⏭ in the glyph
     slot, and modifier class `is-keep-moving`.
   - In a mixed card, keep_moving rows render after the others, with the
     `keeps ‹N› moving` chip.
   - The fallback text matches the copy table.
4. **Walk composition (`536`).** No logic change is expected. Confirm with a test that a
   keep-moving notice line composes into the single walk toast.
5. **Tests.**
   - Extend `scripts/test-navigation-successor-picker.cjs` and
     `scripts/test-navigation-hotkeys-unblocked-notice.cjs`: v1 parity, the v2
     keep_moving group, caption, tag, folds, lane names, the title and icon rule, and
     mixed ordering.
   - Export the new pure helpers from `700-exports.js`.
6. **Ship.**
   - Bump the nav `manifest.json` (minor) and its description.
   - Update `docs/develop/plugin-apis.md` (successors v2, notice v2).
   - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.

**Done when:**

- A hand-built v2 model renders the picker mock above.
- v1 callers are unchanged.
- All npm gates pass, and the plugins are synced.

## Phase: cycler — Obsidian cycler engine

This is bob-plugins work only, in `plugins/task-status-cycler`. It depends on nav v2.

1. **New fragment `089c-successor-keep-moving.js`.** `085` is at the 1000-line cap.
   Register the fragment in `src/fragments.json`. It contains:
   - `successorLeafProject(app, path)`. This is an R2 port over `app.metadataCache`:
     walk the `parent` wikilinks through `getFirstLinkpathDest`, skip non-area,
     non-project notes, and stop at the first area. A project is a leaf when it lacks
     `root: true` and the walk meets another project first. The depth cap is 16, and the
     walk is cycle-safe. The `type` and `root` parsing must match bob-ledger-tools'
     `pomodoroHomeRootFlag` and Rust's `NoteFacts`.
   - The K1–K8 trigger over `parseSuccessorLedgerEntries` on the baseline (`dailyBefore`
     where the gesture supplies it).
   - Candidate classification and folds.
   - Offer rows with group `keep_moving` and trigger `last_planned`, ordered as pinned.
2. **Wiring.**
   - Extend the `quietGate` early return in `137` with the G1–G3 pre-gate.
   - Plan keep-moving offers inside `planSuccessors` / `applySuccessorVerdicts`, so
     DEDUP, keys (`path#^id` / `path@fnv8[~k]`), destinations, D7 routing, re-plan
     refusal (KM24), and pick writes reuse the existing code paths.
   - `closeKind: "cancelled"` never runs the trigger.
3. **Config.** Add `loadKeepMoving()` next to `loadLinkUnblocked()` (`087`), reading
   `plan.keep_moving` with the same fallback.
4. **Picker model.** `buildSuccessorPickerModel` (`085b`) adds the keep_moving group
   with `header`, `caption`, and `tag` per the copy table, folds, and row `lane`, plus
   the title rule. When `api.successors.version < 2`, the cycler plans **no**
   keep-moving offers. It must never hand an older nav a group it would drop.
5. **Notice.**
   - `successorNoticeText` gains the ⏭ forms.
   - The notice model carries `keep_moving` rows only when `api.notice.version >= 2`.
   - Otherwise the plain-text fallback reports keep-moving picks.
6. **Receipt.** `rememberSuccessorPass` records keep-moving picks, so a same-day reopen
   takes them back (KM28).
7. **Tests.** Add `scripts/test-task-status-cycler-keep-moving.cjs`, registered in the
   `package.json` `test` script.
   - Pure KM and LP rows run through the exported helpers (`210-exports.js`).
   - Wiring rows use the fake nav picker: v2 offers, v1 suppression, Esc, walk-landing
     hold, re-plan refusal, and the receipt.
   - The zero-read wiring test (KM32) must stay green.
   - Cite the vector ids in test names.
8. **Ship.**
   - Bump the cycler `manifest.json` (minor) and its description.
   - Update `docs/task-status-cycler.md`.
   - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.
   - Close a live-linked task in a leaf project of a **scratch copy** of the vault
     opened in Obsidian, if one is available. Otherwise report that live Obsidian
     verification is left for Bryan.

**Done when:**

- Every KM and LP row that an Obsidian surface can express passes in the JS tests.
- The keys, order, and destinations match the Rust vectors.
- The plugins are synced.

## Phase: mac — Bob Mac Capture presentation

This is bob-mac-capture work only. Read its `AGENTS.md`: Swift must be compiled, run
`just portable-check`, and the Mac scratch typecheck applies to `BobMacCapture*`.

1. **Decode** `keep_moving` arrays on `CaptureTaskComplete` and `PomodoroCloseSummary`
   (including nested relation completes) with `decodeIfPresent ?? []`. Reuse
   `TaskCompleteUnblocked`.
2. **Gather** keep-moving offers in `makePullInPreviewSnapshot`, so Return opens the
   choice step and `--pull-in` keys round-trip.
3. **Present.**
   - In `CapturePullInChoiceSection`, give `title` the value `Keep ‹N› moving`, give the
     section rank map rank 3 (last), and add a caption line built from `unblocked_by`.
   - In the view, `glyph` is `forward.end.fill` and `glyphColor` is the Next tint.
   - `metaDetails` / `detailText` know `last_planned`, so no `trigger:` caption appears.
   - `contextText` never says "Unblocked by" for keep-moving.
   - The filter-bar capsule reads `Keep moving` when every offer is keep-moving.
   - The fold text includes blocked and scheduled.
   - Preview cards get the section through `pullInPreviewSummary`.
   - The block-diff badge appears for `reason == "keep_moving"`.
   - The notification line is `⏭ Pulled in N · keeps ‹N› moving`.
4. **Fixtures.** Build `bob` from bob-cli master, which contains the rust phase. Open
   bob-cli with `/sase_repo` and use a scratch build, without editing versions. Generate
   a `task-complete-keep-moving.json` and a `pomodoro-close-keep-moving.json` fixture
   with a script beside `Scripts/generate-project-note-offer-fixtures.sh`.
5. **Tests.** Cover decoding (absent array, empty, populated), section title, rank,
   glyph, caption, and capsule, the absence of a `trigger:` caption, the notification
   line, choice-step gathering, and `--pull-in` round-trip. Add them to
   `CapturePullInDecodingTests`, `CapturePullInChoiceTests`,
   `CapturePullInChoiceDesignTests`, and `NotificationServiceTests`.
6. **Docs and verify.**
   - Update `docs/protocol.md` and run `just docs-check`.
   - Run `just portable-check` (on athena if `swift` is missing), plus the Mac scratch
     `bundle.sh` and `just typecheck-tests`.
   - If the Mac is offline, state that the macOS targets are unverified.

**Done when:**

- Fixture-driven tests show the Pull-In card mock.
- An older payload without `keep_moving` decodes unchanged.
- portable-check passes.

## Verification (epic level)

- **One rule, two engines.** The Rust and JS suites both carry KM1–KM32 and LP1–LP12,
  with matching keys, order, and destinations.
- **Every surface.**
  - Obsidian: Ctrl+Enter (link, task line, Pomodoro line), Alt+], the walk, and
    Links-stage complete labels go through the cycler.
  - `bob capture`: `!`, `x…` closes, `^route:id=x…`, and complete labels go through
    Rust.
  - Bob Mac Capture: Rust, through `bob capture`.
  - Cancel never offers.
- **Clarity.** No keep-moving surface uses 🔓 or "unblocked". Each one shows the why
  caption and the "not dependents" tag.
- **Safety.**
  - `plan.keep_moving: false` silences only this group.
  - An older nav or Mac never mislabels the rows: an older nav never receives them, and
    the Mac reads them from a separate array.
  - The zero-read and timing budgets hold.
