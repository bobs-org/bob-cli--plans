---
tier: epic
title: Pull a note's ready tasks with a lone @route capture
goal: 'Capturing a lone `@foobar` item (in `bob capture` or Bob Mac Capture) opens
  the existing successor Pull-In choice. It lists every ready task that resides in
  `foobar.md`, previews where each would land, and links only the picked tasks into
  today''s Pomodoros. The flow is atomic, explicit, and presented with one copy and
  visual system on every surface.

  '
decisions:
  lanes:
    ask: Which lanes should a note pull offer?
    choices:
      open_lanes: Pending, Next, and Ready residents not linked today (keep-moving's
        ready set)
      ready_only: Only Ready `[ ]` residents; unlinked Next/Pending fold as "in flight"
    default: open_lanes
    why: Yesterday's unlinked Next work is exactly what a morning pull should surface
    answer: open_lanes
  empty_pull:
    ask: What should `@note` do when the note has nothing to offer?
    choices:
      refuse: Refuse `nothing_to_pull_in` with the fold; the Mac shows a calm empty
        state
      succeed: Succeed and write nothing; the CLI prints the fold and the Mac panel
        closes on Return
    default: refuse
    why: Return never looks like it worked while doing nothing, and the reason stays
      on screen
    answer: refuse
  memory_decision_record:
    ask: Add a decision record that a lone @route pulls its note's ready tasks?
    memory:
    - decisions
    default: false
    answer: false
  memory_successor_offer:
    ask: Teach the Successor Offer glossary strand about note-pull offers?
    memory:
    - glossary:successor-offer
    default: false
    answer: false
phases:
- id: contract
  title: Contract docs, copy, and NP vectors
  depends_on: []
  size: medium
  description: 'contract: write the docs-first contract for the note pull. Cover the
    grammar row, the new capture.md section, the capture-parse and JSON shapes, task-dependencies
    §12.20/§12.21 with the NP vectors and copy table, the reference pages (grammar
    7), and any accepted memory updates.'
- id: cli
  title: bob-cli note pull grammar, planner, picks, and output
  depends_on:
  - contract
  size: large
  description: 'cli: implement the lone-header `note_pull` kind in the capture language
    and capture-parse. Build the staged planner that classifies residents, keys, orders,
    places, and writes picks; wire it into pull-in numbering, the net pass, refusals,
    inbox routing, JSON, and human output; add tests for every NP vector.'
- id: mac
  title: Bob Mac Capture note-pull preview, card, and notification
  depends_on:
  - cli
  size: medium
  description: 'mac: decode `note_pull` results and render them. Covers the preview
    card with the Pull In verb, the Pull-In card glyph, tint, lanes, and footer, the
    calm empty state, the Shift-Return list helper, and notifications. Includes fixtures
    captured from the real bob and compiled, tested Swift.'
proposed_by: bbugyi200.athena.17g
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 11:56:51
status: done
bead_id: bob-cli-gj
---

- **PROMPT:** [prompts/202611/note_pull.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/note_pull.md)
- **BEAD:** [bob-cli-gj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gj/README.md)

# Plan: Pull a note's ready tasks with a lone `@route` capture

## 1. The experience

You are planning the day. You open Bob Mac Capture and type `@sase`. Before you press
anything, the preview shows the following (a real `bob capture --dry-run`, like every
preview):

```text
 📥 Pull from ‹sase›                                   ready in sase
    1  Ship the badge renderer           Pending   → FIX
    2  Wire badges into the README       Next      → FIX
    3  Write badge docs                  Ready     → new DOCS session
    2 already planned · 3 blocked · 1 scheduled later · 1 hidden
    ↩ choose what to pull in · ⇧↩ start a list
                                                   ⇧↩ List    ↩ Pull In
```

Return opens the existing Pull-In card. You mark rows 1 and 3 with Space and press
Return. Bob links both tasks into today's Pomodoros in one atomic write. Each picked
task is raised to Next, stamped fresh, and given a block ID if it lacked one. A
notification reads `📥 Pulled in 2 from sase`. Esc goes back to the draft and writes
nothing.

The same gesture from a terminal:

```text
$ bob capture @sase
Pull from sase · 3 ready tasks
  offer 1  Pending Ship the badge renderer  sase.md ^ship-renderer → FIX
  offer 2  Next Wire badges into the README  sase.md ^readme-badges → FIX
  offer 3  Ready Write badge docs  sase.md → new DOCS session
  not offered  2 already planned · 3 blocked · 1 scheduled later · 1 hidden
  nothing pulled in · choose with -p 1,3 or -p all

$ bob capture -p 1,3 @sase
Pull from sase · 2 of 3 pulled in
  linked Pending Ship the badge renderer  sase.md ^ship-renderer → FIX (next up)
  linked Ready → Next Write badge docs  sase.md ^write-badge-docs → new DOCS session · added ^write-badge-docs
  not pulled in  offer 2 Wire badges into the README
  not offered  2 already planned · 3 blocked · 1 scheduled later · 1 hidden
```

Without `-p`, `bob capture @sase` is a read-only answer to "what's ready in sase?". This
matches the protocol rule that capture never prompts and never auto-links.

## 2. Why this shape

- **A free slot in the grammar, read the way `@` already reads.** A lone `@route` header
  with no bullets is refused today as `empty_list` (shipped in `0ebd0714`). `@` says
  _where_ (`decisions:capture-sigils-say-where-what-how`). An `@sase` with nothing else
  still names a place, and the only useful thing to do with a place alone is to look
  inside it. Adding bullets below the header still turns it into a list, so the two
  readings never compete. The live preview shows which one you are writing.
- **Reuse the Pull-In protocol instead of inventing a picker.** Offers, keys, `n`
  numbering, `-p/--pull-in`, `--require-pull-in-choice`, `pull_in.groups` copy, the Mac
  Pull-In card, inbox routing, and atomic batch writes all exist already
  (`docs/task-dependencies.md` §12). A note pull is one more offer class on that
  protocol: group `note_pull` and trigger `note_pull`. Nothing is preselected, and only
  picks are written (`decisions:unblocked-tasks-are-offered`).
- **One honest definition of "ready".** Bob's keep-moving offers already define a note's
  "ready tasks" (§12.15). They are open Pending, Next, and Ready residents that are not
  blocked, future-scheduled, or hidden. The note pull uses exactly that set and adds one
  more filter, _not already linked today_, so the rows are things you could actually
  pull in. "Ready" here means the unblocked open lanes, **not** the freshness-gated
  READY bucket (`decisions:ready-is-freshness-gated`). NEW and ROTTEN Ready tasks are
  offered, and picking one stamps it fresh, which confirms it.
- **bob decides and the Mac presents** (`decisions:mac-capture-is-a-thin-client`). The
  Mac receives structured rows, folds, and bob-authored copy.

Rejected alternatives:

- A new sigil or command (`@sase?`, `bob pull sase`): this would add grammar for
  something the existing route already says, and `bob capture` must never grow
  subcommands.
- Auto-linking every ready task: this violates the explicit-pick rule and over-plans.
- Offering sub-project residents: residence is the file holding the checkbox line, and
  the user named one file. Pull a sub-project by its own route.
- Making the pull configurable through `plan.link_unblocked` / `plan.keep_moving`: those
  switches govern automatic offers. A typed `@sase` is an explicit request and needs no
  kill switch.

## 3. Behavior contract

### 3.1 Grammar

- A capture item that is **exactly one task-list header line and nothing else** is a
  note pull: `@route` alone on its line, with no `#section`, no bullets, no other text,
  and no other markers. It parses as the new `CaptureKind::NotePull { route }` and as
  `capture-parse` mode `note_pull`.
- Route resolution is identical to task capture: `[A-Za-z0-9_-]`, lower-cased,
  `<route>.md` at the vault root. Aliases do not resolve (they do not for task routes
  either).
- Unchanged:
  - `@route#section` alone still refuses `empty_list` (`needs: ["list_item"]`).
  - `@route` plus bullets is still a list.
  - `@@route` is still a declaration.
  - `@route:id`, `@route+id`, `@route^id`, and `@route:id@` keep their forms.
  - Forced-destination flags (`-r`, `--section`, …) keep `@tokens` literal.
  - Any other marker on the line (`@sase s:2`, `@sase #fix`, `@sase %c`) means the line
    is not a lone header and follows today's rules.
- A header that is directly followed by another header (`@dev` / `@cash` / `- b`) starts
  a new list today. The first header is now a note pull, which is consistent with "an
  item is judged alone".
- `GRAMMAR_REVISION` becomes 7. The commit is `feat(capture)!:` with a
  `BREAKING CHANGE:` footer: a lone `@route` is a note pull instead of an `empty_list`
  refusal, and `capture-parse` reports `note_pull` instead of `incomplete`/`list_item`.
  Do not edit `Cargo.toml`'s version or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).

### 3.2 `capture-parse`

The lone header reports:

- mode `note_pull`;
- `route`;
- `needs: []`;
- no diagnostics;
- the same route spans a header gets today;
- the same per-item `list` object it reports today (`kind: "tasks"`, `count: 0`,
  `header_line`, `header_range`), so clients can still offer "start a list" from it.

The help text and `docs/reference/cli/capture-parse.md` (regenerated with
`just docs-gen`) state this, along with `grammar: 7`.

### 3.3 What is offered

Let N be the staged text of `<route>.md`. Staged text includes earlier items of the same
draft. If N is neither on disk nor staged, refuse `pull_note_missing` (§3.7). Classify
every `#task` checkbox line residing in N, including nested subtasks. The first matching
rule wins:

| Resident                                                    | Outcome                                                                        |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------ |
| N's own `^prj` task                                         | never offered or counted (a project's lifecycle line, as in every offer class) |
| Done, cancelled, custom, or unknown status                  | ignored, not counted                                                           |
| `[?]`, own open prerequisite, or `#prj_is_blocked`          | folded `blocked`                                                               |
| `scheduled` after today                                     | folded `scheduled`                                                             |
| whole-token `#hide`                                         | folded `hidden`                                                                |
| has a live Task Link in today's Pomodoros (§12 "Live link") | folded `already_planned`                                                       |
| `[/]` Pending, `[*]` Next, `[ ]` Ready                      | **offered**                                                                    |

> [!decision] lanes = ready_only Only `[ ]` Ready residents are offered. Unlinked `[*]`
> and `[/]` residents are folded under a new additive `in_flight` key with the fold text
> `2 in flight`, placed right after `already planned`. Vectors NP1 and NP2 change
> accordingly.

Additional rules:

- **What else is offered.** `#ref` tasks, nested subtasks, and recurring tasks whose
  occurrence is today or past are offered like any resident (KM18 parity). A `scheduled`
  date of today is offered.
- **Prerequisite proof.** Prove a candidate's prerequisites closed only when it carries
  a non-empty `[dependsOn::]`. Build the vault-wide dependents snapshot only in that
  case (the KM31 gate). If the snapshot is unavailable, fold that candidate as
  `blocked`. Bob never offers a task it cannot prove unblocked.
- **Blocked projects.** If N is a project whose `^prj` is Blocked, every resident
  carries `#prj_is_blocked` and folds as `blocked`. The empty-pull message names the
  project reason (§3.7).
- **Inbox files.** Inbox files are valid sources (`@mac_inbox` is triage-by-pulling).
  Rows report `inbox: true`. A pick that changes an inbox task's status (Ready → Next)
  becomes an inbox-route candidate through the existing batch detector
  (`src/native/capture/inbox_route.rs`), with no new code. Bob Mac Capture's route card
  and `bob capture -i` ask where each candidate goes.

### 3.4 Order, keys, numbering

- **Order.** Lane pull order (Pending → Next → Ready), then line in N. This is the same
  order keep-moving uses (`decisions:task-lanes-are-sticky`).
- **Keys.** §12.2 KEYS: `<note>#^<id>`, or `<note>@<digest>[~k]` for an ID-less row.
  Keys come from N as staged _before_ any pick, so a preview's keys still match on
  submit.
- **Numbering.**
  - `note_pull` groups sort after every other offer group, then by draft item order.
  - Numbers continue across a draft exactly as other offers do (the
    `number_draft_offers` order key).
  - One row per key per draft: the earliest item that offers a key owns it, and a later
    item omits its duplicate. Only a later `!` close can make a task newly offerable, so
    a later note pull is the side that drops it.

### 3.5 Destinations

There is no predecessor and no anchor, so destinations use normal typed placement, the
same way a capture `@route:id` link or a `project_created` offer places:

- An open Pomodoro whose name carries the task's effective type is `typed`.
- If none exists, a new `— TYPE` placeholder is `created`.
- With no effective type, the current or next open entry is `implicit` with
  `fallback: true`.

Placement is sequential in offer order. If the first DOCS offer would create the `DOCS`
session, the next DOCS offer shows `typed`.

A placeholder is written only when a picked link lands there. Because previews need
destinations, a note pull always reads today's daily note. It fails with the usual
missing-file or no-eligible-Pomodoro errors when it cannot.

### 3.6 Picks and unpicked rows

A pick writes exactly what a picked `keep_moving` / `hand_off` offer writes (§12.4):

- `' '` → `'*'`; `'*'` and `'/'` stay unchanged.
- Mint a missing block ID with the shared deterministic minting.
- Stamp `[fresh:: today]` with `stamp_fresh`, clearing `[unblocked::]` and `[keeps::]`.
- Insert the plain Task Link at its destination.
- Never touch the schedule, `[id::]`, or `[dependsOn::]`.
- Do not call `plan_task_link`, because it retires schedules.

The one exception: a row created by an earlier item of the same draft is **not**
stamped, because creation never stamps.

Unpicked rows write nothing. There is no recovery, no `[unblocked::]`, and no minting.
JSON reports them with `not_linked: "not_selected"`.

### 3.7 Selection and refusals

| Input                                                 | Result                                                                                                                                                    |
| ----------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| no `-p`, or `-p none`                                 | success; nothing written; offers reported unselected; the human output ends with the `choose with -p …` hint                                              |
| `-p 1,3`, `-p sase.md#^ship-renderer`, `-p all`       | normal pull-in selection (`src/native/capture/pull_in.rs`); stale, unknown, or out-of-range picks are the existing teaching error, and nothing is written |
| `--require-pull-in-choice` without `-p`, offers exist | existing write-free `pull_in_choice_required` refusal, carrying `pull_in` and the flat `offers`                                                           |
| nothing offered (any `-p`)                            | `nothing_to_pull_in` refusal (see the callout); writes nothing                                                                                            |
| `<route>.md` absent on disk and not staged            | `pull_note_missing` refusal; writes nothing                                                                                                               |

> [!decision] empty_pull = refuse `nothing_to_pull_in` carries an additive
> `note_pull: {note, route, folded}` payload so clients can render it.
>
> | Case                  | Message                                                                                                                                                                  |
> | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
> | some residents folded | `nothing to pull in from sase.md: 2 already planned · 3 blocked · 1 hidden`                                                                                              |
> | no open residents     | `nothing to pull in from sase.md: it has no open tasks`                                                                                                                  |
> | blocked project       | `nothing to pull in from sase_badges.md: the project is blocked (its ^prj waits on 1 task)` (or `is scheduled for 2026-11-20`), reusing the project block reason wording |

> [!decision] empty_pull = succeed An empty pull succeeds with
> `note_pull.successors: []` and the fold, and writes nothing. The CLI prints
> `Nothing ready in sase · 2 already planned · …`. The Mac preview still shows the calm
> empty state, and Return closes the panel with no notification.

The `pull_note_missing` message:

```text
`@foo` pulls ready tasks from foo.md, which does not exist: add `- <task>` bullets below it to start a list there, or run 'bob capture-targets' to list routable notes
```

### 3.8 Multi-item drafts

Items plan in order against the staged batch.

- An earlier item that links a resident makes it `already_planned`.
- An earlier item that creates a task in N makes it offerable, unstamped if picked.
- The SL20 net pass applies per row: a later item that links, completes, re-blocks, or
  drops an offered task drops that row.
- There is no keep-moving-style group invalidation, because a note pull is a request,
  not a trigger.
- Any failure anywhere still rolls back the whole draft.

### 3.9 Reporting (additive JSON; capture stays unversioned / schema 1)

```json
{
  "kind": "note_pull",
  "note_pull": {
    "note": "sase.md",
    "route": "sase",
    "successors": [
      {
        "...": "the exact unblocked[] row shape",
        "trigger": "note_pull",
        "unblocked_by": [],
        "inbox": false,
        "offer": { "group": "note_pull", "project": null, "...": "…" }
      }
    ],
    "folded": {
      "already_planned": 2,
      "blocked": 3,
      "scheduled": 1,
      "hidden": 1,
      "sub_projects": 0,
      "already_started": 0,
      "waits_for": 0
    }
  },
  "pull_in": {
    "supported": true,
    "offer_count": 3,
    "choice": null,
    "picked_count": 0,
    "groups": [
      {
        "group": "note_pull",
        "note": "sase.md",
        "folded": { "...": "…" },
        "copy": {
          "title": "Pull from ‹sase›",
          "why": null,
          "tag": "ready in sase",
          "fold": "2 already planned · 3 blocked · 1 scheduled later · 1 hidden",
          "glyph": "note_pull"
        }
      }
    ],
    "copy": { "title": "Pull from ‹sase›", "capsule": "Pull in", "glyph": "note_pull" }
  }
}
```

- `note` is a new additive field on `pull_in.groups[]`. It is omitted for other groups.
- Pomodoro block diffs mark added picked lines with `reason: "note_pull"`.
- `TaskBlockRole` gains `NotePull`.
- Dry-run JSON equals real-run JSON for the same flags, apart from the `dry_run` and
  preview wording.

### 3.10 Copy and visual system (identical on every surface)

‹N› is the note stem. The glyph metaphor is pulling work into your tray:

- emoji `📥`;
- Mac SF Symbol `tray.and.arrow.down.fill`;
- JSON glyph key `note_pull`.

The glyph pairs deliberately with the inbox route card's `tray.and.arrow.up`, so an
`@mac_inbox` pull reads "pull in, then route out". Tint is the existing **Next** status
colour token, because picks land in Next (as with keep-moving). Add no new hex values.
Rows always show the lane name (Pending / Next / Ready), never a raw symbol.

| Element                                 | Copy                                                                                                                           |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Group title                             | `Pull from ‹N›`                                                                                                                |
| Contrast tag                            | `ready in ‹N›`                                                                                                                 |
| Fold line (omit zero parts, this order) | `2 already planned · 3 blocked · 1 scheduled later · 1 hidden`                                                                 |
| Top title, only note_pull groups        | `Pull from ‹N›` / `Pull from 2 notes` (mixed: the existing title)                                                              |
| Capsule                                 | `Pull in`                                                                                                                      |
| CLI group header                        | `Pull from N · k ready tasks` (`1 ready task`); after picks `Pull from N · p of k pulled in`                                   |
| CLI fold row                            | `not offered  2 already planned · …`                                                                                           |
| CLI no-pick hint                        | `nothing pulled in · choose with -p 1,3 or -p all` (the example uses the first and last offer numbers; with one offer: `-p 1`) |
| Mac footer verb                         | `Pull In`                                                                                                                      |
| Mac preview hint                        | `↩ choose what to pull in · ⇧↩ start a list`                                                                                   |
| Mac prompt status / VoiceOver           | `Choose tasks to pull in from ‹N›`                                                                                             |
| Mac empty state                         | `Nothing ready in ‹N›` above the fold line                                                                                     |
| Mac notification title                  | `📥 Pulled in k from ‹N›` (mixed drafts: the existing summary, counting note-pull picks as pulled in)                          |
| Mac notification body                   | picked task texts with destinations, `Ship the badge renderer → FIX · Write badge docs → DOCS`                                 |

### 3.11 Gates and performance

- Non-pull drafts pay nothing: detection is lexical, at parse time.
- A pull reads N (staged first), today's daily note, and the `NoteIndex` needed for
  live-link resolution. It reads the dependents snapshot only under the KM31 gate.
- Budget: in a release build, `bob capture --dry-run -f json @<route>` against a
  **temporary copy** of `~/bob` stays within §12.7's ≤ 70 ms. Use the largest
  area/project and `mac_inbox`. Never run captures against the live vault.

### 3.12 Non-goals

- Filtering a pull by type (`@sase #fix`), naming the destination Pomodoro, or pulling
  sub-project residents.
- Freshness chips on rows; ready counts on `@` route completion candidates.
- An Obsidian surface (capture has none) or an undo/take-back (vault git is the undo).

## 4. NP vectors (the contract phase writes these as §12.21; implementations cite them)

Fixture: area `sase.md` (→ `dev`), a daily note with an open `FIX` Pomodoro and no
`DOCS` entry, and `BOB_NOW` fixed. Select-all applies unless a row says otherwise.

| #    | Situation                                                                                         | Outcome                                                                                                                                                                                   |
| ---- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| NP1  | `@sase`; Ready A, Next B, Pending P, none linked                                                  | one `note_pull` group: P, B, A; `trigger: note_pull`; `unblocked_by: []`; nothing written; exit 0                                                                                         |
| NP2  | `-p all @sase`                                                                                    | P and B are linked with unchanged status; A goes `[ ]`→`[*]`; missing IDs are minted; all three are stamped fresh, clearing `[unblocked::]`/`[keeps::]`; links land at typed destinations |
| NP3  | B has a live link today                                                                           | B is folded `already_planned`                                                                                                                                                             |
| NP4  | B's link is struck, or sits under a closed Pomodoro                                               | B is offered                                                                                                                                                                              |
| NP5  | residents `[?]` / Ready with open prerequisite / `#prj_is_blocked` / scheduled tomorrow / `#hide` | folded blocked / blocked / blocked / scheduled / hidden; scheduled today is offered                                                                                                       |
| NP6  | N is a project                                                                                    | `^prj` is never offered or counted                                                                                                                                                        |
| NP7  | N has sub-projects with Ready tasks                                                               | only N's residents are offered                                                                                                                                                            |
| NP8  | done, cancelled, or custom-status residents                                                       | ignored, uncounted                                                                                                                                                                        |
| NP9  | `#ref` resident, nested subtask, recurring task due today                                         | offered                                                                                                                                                                                   |
| NP10 | every resident folded, or no open tasks (any `-p`)                                                | `nothing_to_pull_in` with the fold; writes nothing                                                                                                                                        |
| NP11 | N's `^prj` waits on an open task                                                                  | `nothing_to_pull_in` naming the project reason                                                                                                                                            |
| NP12 | `@nosuch` with no `nosuch.md`                                                                     | `pull_note_missing`; writes nothing                                                                                                                                                       |
| NP13 | `--require-pull-in-choice` without `-p`                                                           | `pull_in_choice_required` with `pull_in` and flat `offers`; writes nothing                                                                                                                |
| NP14 | `-p 3`, `-p sase.md#^ship-renderer`, `-p 9`                                                       | numeric pick, key pick, and the teaching error listing current offers                                                                                                                     |
| NP15 | `-p none`                                                                                         | success; nothing written; no IDs minted                                                                                                                                                   |
| NP16 | `-p 1 @mac_inbox` picks a Ready inbox task                                                        | an inbox-route candidate (`-i 1=health` moves it in the same batch); a picked Next inbox task is no candidate                                                                             |
| NP17 | draft `Write docs @sase` ⏎⏎ `@sase`                                                               | the new task is offered; picking it does not stamp it fresh                                                                                                                               |
| NP18 | draft `@sase` ⏎⏎ `@sase:docs`                                                                     | the later link drops A's row (SL20)                                                                                                                                                       |
| NP19 | draft `!sase:t` (recovers dependent D in sase.md) ⏎⏎ `@sase`                                      | D appears once, in `hand_off`; the pull omits it                                                                                                                                          |
| NP20 | draft `@sase` ⏎⏎ `@dev`                                                                           | two groups in item order; numbers continue                                                                                                                                                |
| NP21 | A has type DOCS and no DOCS entry exists                                                          | destination `created`; the placeholder is written only when A is picked; a second DOCS offer shows `typed`                                                                                |
| NP22 | candidate has `[dependsOn::]` and the snapshot is unavailable                                     | that candidate is folded `blocked`                                                                                                                                                        |
| NP23 | no daily note / no `## Pomodoros`                                                                 | the usual missing-file / no-eligible-Pomodoro error                                                                                                                                       |
| NP24 | `@sase` ⏎ `- Call the bank`                                                                       | a list, unchanged                                                                                                                                                                         |
| NP25 | `@sase#Ideas` alone                                                                               | `empty_list`, unchanged                                                                                                                                                                   |
| NP26 | `@dev` directly followed by `@cash` / `- b`                                                       | `@dev` is a note pull; `@cash` is a list                                                                                                                                                  |
| NP27 | `bob capture-parse -f json -- @sase`                                                              | mode `note_pull`, `needs: []`, list `count: 0`, `grammar: 7`                                                                                                                              |
| NP28 | the same vault and flags, dry run vs real run                                                     | identical JSON apart from dry-run wording                                                                                                                                                 |

## 5. Phases

### 5.1 `contract` — contract docs, copy, and NP vectors

Read `sase memory read docs.md` first. The contract states observable behavior only.

- `docs/task-dependencies.md` §12:
  - Add note pull to the "Offer (D)" and "Groups" vocabulary.
  - Add a §12.1 trigger-table row: `bob capture '@route'`, a lone header that completes
    nothing, trigger `note_pull`.
  - Extend ORDER and the earliest-item-owns-a-key rule.
  - Write the new **§12.20 Note pull offers** from §3 above: vocabulary, offer set,
    order, destinations, picks, selection and refusals, multi-item, reporting, copy
    table with the Mac card and CLI mocks, gates, and interplay with
    `unblocked-tasks-are-offered`, `task-lanes-are-sticky`,
    `today-is-read-from-the-ledger`, `ready-is-freshness-gated`,
    `mac-capture-is-a-thin-client`, and `bob-cli-releases-ride-a-daily-train`.
  - Write **§12.21 NP — note pull vectors** from §4.
  - State plainly that note-pull rows ride the successor-offer protocol but have no
    predecessor.
- `docs/capture.md`:
  - Add a Grammar-at-a-glance row: `@route` alone (no bullets).
  - In "Lists", change "A header with no bullets captures nothing" so that only a
    section header still refuses `empty_list`, and link to the new section.
  - Add a new `### Pulling ready tasks from a note {#pulling-ready-tasks-from-a-note}`
    after Lists. It holds the §1 walkthrough, the selection table, the refusals, and a
    link to §12.20.
  - Update the `capture-parse` and JSON sections (mode `note_pull`, `kind: "note_pull"`,
    `note_pull` object, the new refusal codes in the diagnostic/code lists).
- `docs/reference/json-outputs.md` (the capture and capture-parse rows) and
  `docs/reference/compatibility.md` (grammar 7).
- Run `just docs-check`.
- Memory (only for accepted memory decisions; use `/sase_memory_write`, then
  `sase memory init` and `just docs-gen` for the generated decision/glossary pages):

> [!decision] memory_decision_record Add a new record to the `decisions` memory web
> (keyword `lone-route-pulls-ready-tasks`), titled "A Lone @route Pulls Its Note's Ready
> Tasks". It needs the following sections:
>
> - **Applies to:** bob-cli, Bob Mac Capture.
> - **Claim:** a lone task-list header is a note pull that offers N's unblocked,
>   unlinked open residents through the Pull-In protocol; nothing is preselected.
> - **Why:** §2 above, including its rejected alternatives.
> - **Cost:** a lone header is no longer an `empty_list` refusal, so grammar 7 is
>   breaking; every pull reads the daily note.
> - **Reopens when:** a pull needs filtering or a Pomodoro override, or sub-project
>   residents are wanted.
>
> Cite this plan and the contract commit.

> [!decision] memory_successor_offer In the `glossary:successor-offer` strand, add one
> sentence saying that a capture of a lone `@route` offers that note's unblocked,
> unlinked open tasks as note-pull offers on the same protocol; they have no
> predecessor.

### 5.2 `cli` — grammar, planner, picks, and output

Implement §3 to the contract, citing NP vector ids in test comments. If the code and a
vector disagree, fix §12.20/§12.21 first in the same change. Starting points:

- **Grammar.**
  - `src/native/capture_language/item.rs` (`parse_list_bullet_item`'s `list.count == 0`
    branch) emits `CaptureKind::NotePull { route }` for a task-list header with no
    section. `src/native/capture_language/model.rs` defines the variant.
  - `src/native/capture_language/editor_parse.rs` (the `list.count == 0` branch near
    `Need::ListItem`) reports mode `note_pull` with `needs: []`, keeping `list` and the
    spans.
  - Bump `GRAMMAR_REVISION` in `src/native/capture_language/mod.rs`.
  - Update `src/native/capture_parse.rs` help.
  - Add the kind string `note_pull` in `src/native/capture/plan.rs` (next to
    `project_note`).
  - Route the kind in `src/native/capture/batch.rs` / `plan.rs`.
- **Planner.** Add a new module, `src/native/capture/note_pull.rs`, modeled on the offer
  path in `src/native/capture/project_note.rs` (`is_offerable_row`,
  `place_project_task_links`, scratch versus actual placements, `offer_entry_role`) and
  on the classifier in `src/native/task_complete/keep_moving.rs` (folds, lane order,
  `has_hide_tag`, `has_prj_is_blocked_tag`, the dependents-snapshot gate). It should:
  - read N and the day text from the `CaptureBatchPlanner` staged snapshot;
  - classify residents (§3.3);
  - build keys and an `OfferOrderKey` that sorts after `keep_moving`;
  - compute destinations sequentially in a scratch day text;
  - apply the selection from `request.selection` (`SuccessorSelection`);
  - write picks with the existing status writer, block-ID mint, `stamp_fresh`, and link
    insertion, skipping the stamp for tasks an earlier item created (compare against the
    planner's original text).
- **Integration.**
  - Add `OfferGroup::NotePull` (`src/native/task_complete/successors.rs`) and the
    `NOTE_PULL_TRIGGER` constant.
  - Add a `note_pull` field to `CaptureItemResult` (`src/native/capture/output.rs`).
    Include its rows in `all_offer_rows`, `unblocked_vecs_mut` (or a sibling iterator
    the callers share), and `completion_summary_count`, so that numbering
    (`src/native/capture/pull_in.rs`), `choice_required_error`, `unknown_pick_error`,
    the SL20 net pass (`apply_successor_net_post_pass` in
    `src/native/capture/task_complete.rs`), and inbox routing all see them.
  - Add the earliest-item-owns-a-key omission.
  - Add `GroupKey::NotePull` plus copy in `src/native/capture/pull_in_copy.rs`: the
    group copy, fold line, and top copy (`Pull from 2 notes`). Add the additive `note`
    on `PullInGroupJson`.
  - Add the `TaskBlockRole::NotePull` / `reason: "note_pull"` block diff.
  - Add the refusal codes `nothing_to_pull_in` (with the `note_pull` payload) and
    `pull_note_missing` on `CaptureError`.
- **Human output.** In `src/native/capture/output.rs`, add the group header, offer rows
  (reuse the keep-moving row printer and lane colouring), fold row, the no-pick hint (in
  place of the "offers were not pulled in; link later…" line for pull-only drafts), the
  linked rows, and the `not pulled in` rows, exactly as §1/§3.10 show. Use the existing
  styler; colour the lane names with their status colours and dim the folds and hints.
- **CLI help.** In `src/native/capture/cli.rs`, add examples `bob capture @sase` and
  `bob capture -p 1,3 -- @sase`, and a sentence in the pull-in paragraph. Then run
  `just docs-gen`.
- **Tests.**
  - Unit tests in `src/native/capture/tests/` (grammar) and the module (planner) cover
    every NP vector.
  - Add a CLI-level JSON fixture test for NP1, NP2, and NP13.
  - Pin the human output for NP1/NP2 and both refusals.
- **Verify.**
  - Run `just check` and `just docs-check`.
  - Run the §3.11 timing against a temporary copy of `~/bob`.
  - Save real `bob capture --dry-run -f json` and refusal outputs for NP1, NP2
    (`-p all`), NP10, NP12, NP13, and NP16 under `tests/fixtures/` (or the existing
    fixture home). The Mac phase copies them.

### 5.3 `mac` — preview, card, empty state, list helper, notification

Work in the linked `bob-mac-capture` repo (open it with `/sase_repo`; read its
`AGENTS.md`). Use the cli phase's saved real-bob JSON as fixtures.

- **Decode.**
  - `CaptureModels.swift` gets an optional `notePull` (`note_pull`: `note`, `route`,
    `successors`, `folded`) via `decodeIfPresent`, plus the group `note` field.
  - Collect `note_pull.successors` in `makePullInPreviewSnapshot`
    (`CapturePanelModel.swift`) and `CaptureSuccessorOfferRows.swift`, so that the
    preview path opens the card.
- **Preview card.**
  - Add a `CaptureNotePullPresentation` that renders the §1 preview: a header with the
    glyph, title, and tag; at most five rows (number, text, lane chip, `→ destination`
    capsule), then `+k more`; the fold line; and the hint
    `↩ choose what to pull in · ⇧↩ start a list`.
  - `primaryActionTitle` returns `Pull In` for pull-only drafts.
- **Pull-In card.**
  - Map `CapturePullInGlyph` `note_pull` → `tray.and.arrow.down.fill`.
  - Use the Next tint for `note_pull` sections (beside `keep_moving`).
  - Make `laneText` show lanes for `note_pull` rows.
  - Exclude the `note_pull` trigger from `metaDetails`.
  - Set `promptStatusText` to `Choose tasks to pull in from ‹N›`.
  - Hide **Skip ⇧↩** on pull-only cards, since skipping a pull writes nothing. ⇧↩ there
    announces that Esc returns to the draft.
  - Esc still returns to the draft unchanged. `@mac_inbox` picks still flow into the
    inbox route card through the existing `--require-inbox-route-choice` path.
- **Empty state.**

  > [!decision] empty_pull = refuse When a preview or submit fails with
  > `nothing_to_pull_in`, render a calm, non-error card from its `note_pull` payload:
  > `Nothing ready in ‹N›` plus the fold line. `pull_note_missing` stays an ordinary
  > error.

  > [!decision] empty_pull = succeed Render the same calm card from the successful dry
  > run's empty `successors`.

- **List helper.** Keep Shift-Return's "start a bullet" (`CaptureList.swift`,
  `CapturePanelController.swift`) working when the parse item is `note_pull` with a
  `list` of `count: 0`, in addition to today's `list_item` need. Older bobs keep their
  current behavior.
- **Notification.** `NotificationService.pulledInSummary` counts selected `note_pull`
  rows: title `📥 Pulled in k from ‹N›`, and a body of picked texts with destinations.
  `friendlyKindLabel("note_pull")` gives `Pull from note`.
- **Tests.**
  - Decoding: `CapturePullInDecodingTests`.
  - Presentation and model: `CapturePullInChoiceTests`, `CapturePullInChoiceModelTests`
    (preview path, refusal path, empty state, Skip hidden, Esc).
  - Inbox flow: `CaptureInboxRouteChoiceModelTests` for `@mac_inbox`.
  - List helper: `CaptureListPanelTests`.
  - Notifications: `NotificationServiceTests`.
  - Design snapshots: `CapturePullInChoiceDesignTests`.
- **Docs.** Update the repo's `docs/` page on the Pull-In card, then run
  `just docs-check`.
- **Verify.**
  - Run `just portable-check` (on athena if `swift` is missing here).
  - Type-check the app and test targets on the Mac through a scratch rsync copy
    (`bundle.sh`, `just typecheck-tests`), never Bryan's checkout.
  - Push and watch macOS CI to green.
  - If the Mac is offline, say in the final report that the macOS targets are
    unverified. Never land uncompiled Swift.
