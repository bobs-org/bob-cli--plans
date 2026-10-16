---
tier: epic
title: Link a reference to today from the Bob Refs panel (⌘N)
goal: 'In Bob Mac Capture''s Refs panel, ⌘N opens a Link to Today card with the current
  or next Pomodoro preselected. Return (or a second ⌘N) sets the selected reference''s
  reading task Next and links it under that Pomodoro in today''s daily note. Arrows
  or a typed name pick another Pomodoro or create a new named one. A glass toast confirms
  exactly what changed. A new `bob ref today` command owns every vault decision through
  capture''s existing-task link planner.

  '
decisions:
  after_link:
    ask: Where should the Refs panel go after a ⌘N link lands?
    choices:
      stay: Panel stays open; the card closes, the row gains its pink Today pill,
        a toast floats over it
      hide: Panel hides like Copy does; the toast shows centered where the panel was
    default: stay
    why: Planning several references is a browse task, and the row's new pill is the
      confirmation
    answer: stay
phases:
- id: cli
  title: bob ref today
  depends_on: []
  size: medium
  description: 'cli: add `bob ref today <REF>` to bob-cli. It resolves a reference
    and its v2 reading task, then reuses capture''s existing-task Pomodoro link planner
    and atomic commit. Capture''s ledger selection gains exact-name and stale-safe
    entry-ref targets. The command emits a schema-1 JSON envelope (open Pomodoros
    with running/default/linked flags plus capture''s own pomodoro_link item object)
    with stable error codes. Ships docs, help text, and parity tests against `bob
    capture ''^route:block-id[#name]''`.'
- id: mac_core
  title: RefsCore decoding, fetcher, card state machine, and copy
  depends_on:
  - cli
  size: medium
  description: 'mac_core: in bob-mac-capture''s portable RefsCore/CaptureCore targets,
    decode the `ref today` envelope (reusing CaptureCommandSuccess). Add BobRefsFetcher.linkToToday
    with preview and write lanes, plus the pure RefsTodayLinkCard state machine (ranking,
    selection-to-selector mapping, stale-preview drop, running/failed/refused phases)
    and its key map. Add RefsTodayLinkPresentation for every string, row meta, lane
    chip, preview line, toast, notification, and announcement. Add a fake-bob `ref
    today` arm with real-bob fixtures and tests.'
- id: mac_ui
  title: Refs panel card, ⌘N routing, toast, and docs
  depends_on:
  - mac_core
  size: medium
  description: 'mac_ui: wire ⌘N through RefsKeyRouter, RefsPanelModel, and RefsPanelController.
    Build the RefsTodayLinkCardView in the rename card''s slot. Generalize the copy
    HUD into a reusable glass toast with a Today style and panel-anchored placement.
    Add an optimistic Today-pill overlay, refresh, VoiceOver announcements, and system
    notifications when the panel is hidden. Add the ⌘K action, footer hint, README
    sections, and model, router, and design tests.'
proposed_by: bbugyi200.apollo.7y
decided_by: auto
create_time: 2026-10-16 10:04:15
status: wip
bead_id: bob-cli-8f
---

- **PROMPT:** [prompts/202610/refs_link_to_today.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/refs_link_to_today.md)
- **BEAD:** [bob-cli-8f](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8f/README.md)

# Plan: Link a reference to today from the Bob Refs panel (⌘N)

## 1. The experience

### 1.1 The gesture

In the Bob Refs panel, Bryan selects a reference and presses **⌘N**. A **Link to Today**
card opens in the same slot the Rename card uses. That is the inspector column when the
panel is at least 760 pt wide, and the list otherwise, with the same 6 pt-rise and fade
transition.

The fast path:

- **⌘N then Return**, or **⌘N ⌘N**, links the reading task under the current/next
  Pomodoro without looking. Bob's own default applies:
  - If the task is already linked under an open Pomodoro, it stays there.
  - Otherwise the task goes under the single running timed Pomodoro.
  - Otherwise it goes under the first open Pomodoro.
- **Typing** filters today's open Pomodoros. A `New Pomodoro NAME` row is offered for a
  name that doesn't exist yet. Arrows pick any row.
- **Return** commits.

The reading task becomes Next exactly as capture's link gesture makes it: Ready or
own-prerequisite-Blocked rises to `[*]`, while Next and In Progress keep their lane.

The double-tap mirrors the Obsidian "Link to today" picker, where Ctrl+Shift+Enter
inside the picker commits like Enter.

```
┌ LINK TO TODAY ───────────────────────────────── Ready → Next ┐
│ 📖 Attention Is All You Need                                  │
│    sase.md · ^ref-attention-is-all-you-need                   │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ ⏱  Pomodoro — ↵ BUGS                                      │ │
│ └──────────────────────────────────────────────────────────┘ │
│ ◔  BUGS       09:20–09:50 · 12m left · 2 tasks   Running   ↵ │
│ ◯  READING    planned · empty                    Next up     │
│ ◯  Untitled   planned · 1 task                               │
│ ──────────────────────────────────────────────────────────── │
│ Ready → Next · Linked under BUGS · removed future schedule    │
│ ↑↓ select   ↵ link   esc cancel                              │
└───────────────────────────────────────────────────────────────┘
```

### 1.2 Card anatomy

- **Header.** A small-caps `LINK TO TODAY` label, matching the Rename card's `RENAME`
  header. A trailing lane chip comes from bob's listing preview: `Ready → Next`,
  `Blocked → Next`, `stays Next`, `stays In Progress`, or `won't link` (orange, when
  bob's link guard would consume the gesture). This chip vocabulary is the Obsidian
  picker's.
- **Title.** The reference title with the kind tile the row uses. Below it is a
  secondary caption with the reading task's residence and block ID (`sase.md · ^ref-…`).
- **Field.** A monospaced filter field that reuses `CapturePickerFilterNSTextField`, the
  same focus machinery as the Refs search field, with its own accessibility identifier.
  - The placeholder tells you what Return does: `Pomodoro — ↵ BUGS` (bob's default
    destination name, or `Untitled Pomodoro`).
  - When there is no default it reads `Name a Pomodoro`.
- **Rows.** Every open Pomodoro today, in document order when the query is empty.
  - **Glyph column:**
    - the running entry shows a pink progress ring filled to its elapsed fraction;
    - the default-but-not-running entry shows an accent ring;
    - every other row shows a hollow ring.
  - **Title:** the entry's name, or `Untitled Pomodoro` in secondary text.
  - **Meta line:** `09:20–09:50 · 12m left · 2 tasks`, `planned · empty`, or `Nm over`.
  - **Pills:** `Running` in pink, matching the Refs Today pill and capture's NOW pill;
    `Next up` in accent, on the default row when it isn't running; `Here` in secondary,
    on the row that already holds this reading task's link.
  - The highlighted row shows the ↵ keycap that the Copy card uses.
  - Clicking a row commits it, like Copy card rows do.
- **Create row.** `⊕ New Pomodoro NAME` with a `New` pill. It appears after the matches
  when the query is non-empty and no listed row has exactly that canonical name. The
  displayed name is whitespace-collapsed and uppercased, for display only; bob validates
  it.
- **Preview strip.** One calm line describing what Return will do. It comes from bob's
  dry run of the current selection, worded by CaptureCore's
  `CapturePomodoroLinkPresentation` (ledger text plus chips such as
  `removed future schedule` or `dropped dependency on …`). For a guarded target the line
  is orange and comes from `CaptureLinkGuardPresentation`: "Would not link — …". While a
  preview is pending, the previous line stays visible, dimmed.
- **Hints.** `↑↓ select   ↵ link   ⇧↵ new NAME   esc cancel`. The `⇧↵` hint hides while
  the query is empty.

### 1.3 Keys inside the card

| Key                               | Action                                                                                                                                                                                                                       |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Return, keypad Enter, ⌘N          | Link the highlighted row, or bob's default when the query is empty and nothing was arrowed. Ignored while marked text is composing. Beeps when nothing is selectable (for example, two running Pomodoros and an empty query) |
| Shift-Return                      | Create or reuse the Pomodoro named exactly as typed (exact-name selector; never a prefix match)                                                                                                                              |
| ↑/↓, Ctrl-P/Ctrl-N, Ctrl-K/Ctrl-J | Move the highlight; wraps and includes the create row                                                                                                                                                                        |
| Esc, Ctrl-[                       | Close the card and return to the list (ignored while linking)                                                                                                                                                                |
| Tab, Shift-Tab                    | Consumed, so focus never escapes the card                                                                                                                                                                                    |
| Anything else                     | Edits the field                                                                                                                                                                                                              |

### 1.4 States

| State                                           | Looks like                                                                                                                                                                  |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Loading (first listing dry run)                 | Header and title render immediately; three skeleton rows; field focused. Return or ⌘N with an empty query already works and submits bob's default                           |
| Ready                                           | As mocked above                                                                                                                                                             |
| Two Pomodoros running                           | Banner inside the card: `2 Pomodoros are running — pick one`. No default; empty-query Return beeps                                                                          |
| No open Pomodoros                               | Empty state: `No open Pomodoros today` / `Type a name to plan one`. The create row is the only target                                                                       |
| Refused (bob cannot link this reference at all) | A calm centered SF Symbol, bob's message, and bob's hint, for example `No reading task to plan` for trackerless legacy refs, or `Already read`. Esc closes                  |
| Linking                                         | Spinner plus `Linking…`. Input is disabled, and Esc, selection changes, and other cards are blocked (the rename-card precedent)                                             |
| Failed                                          | The orange message and hint stay inline, the field refocuses, and editing clears the failure. A `stale_pomodoro` failure reloads the rows automatically and keeps the query |

⌘N works in Highlights Tabs mode for real references. Synthetic tabs get the same
warning Copy and Rename show. ⌘N does nothing while a rename runs. It is allowed during
a scan, because bob's capture commit refuses stale pre-images.

### 1.5 The toast

On success the card closes and a glass toast appears. It reuses the copy HUD's
`NSGlassEffectView` capsule and its Reduce Transparency fallback.

```
 ╭──────────────────────────────────────────────────╮
 │  (⏱)  Linked under READING                        │
 │       Attention Is All You Need                    │
 │       Ready → Next · removed future schedule       │
 ╰──────────────────────────────────────────────────╯
```

- The **badge** is a pink circle with a white `timer` glyph, the same visual as
  `RefsTodayPill`. Bryan's eye connects the toast to the pill that just appeared on the
  row.
- The **title** is 13 pt semibold. The **value** is the reference title, 11.5 pt
  secondary, middle-truncated. The **detail** is caption-size tertiary text.

| bob outcome (`capture.pomodoro_link_action`, `creates_pomodoro`, kind) | Badge                | Title                      | Detail                                                        |
| ---------------------------------------------------------------------- | -------------------- | -------------------------- | ------------------------------------------------------------- |
| `linked`                                                               | pink timer           | `Linked under BUGS`        | `Ready → Next` plus chips                                     |
| `linked` and `creates_pomodoro`                                        | pink timer           | `Linked under new READING` | lane phrase plus chips                                        |
| `moved`                                                                | pink timer           | `Moved to READING`         | `from BUGS · stays Next`                                      |
| `already_current`                                                      | pink timer, softened | `Already in BUGS`          | `already Next · no change`                                    |
| kind `link_guard`                                                      | orange `lock.fill`   | `Not linked`               | bob's `link_guard.detail`, for example promoted prerequisites |

Timing and announcement:

- **Fade in:** 0.12 s with a 6 pt rise; opacity only under Reduce Motion.
- **Hold:** 2.2 s. Copy keeps its 1.0 s, since its single line is read at a glance.
- **Fade out:** 0.3 s.
- **VoiceOver** announces the same sentence: "Linked Attention Is All You Need under
  BUGS. Ready to Next."

> [!decision] after_link = stay The panel stays open. The card closes, the selection
> stays on the same reference, and the row's pink Today pill appears in the same frame
> as the toast through an optimistic Today overlay. The follow-up library refresh
> replaces the overlay. The toast is anchored to the panel: centered horizontally, its
> bottom edge 14 pt above the footer.

> [!decision] after_link = hide On success the panel hides, exactly as Copy does. The
> toast is centered on the panel's former frame through the existing `centeredOn:`
> placement. The optimistic overlay and refresh still run, so the pill is there on the
> next open.

In either branch, if the panel is hidden for another reason when the result lands (for
example, focus loss hid it mid-write), bob's outcome is posted as a system notification
through `NotificationService`, following the rename precedent. Failures notify only when
hidden.

## 2. Architecture

bob owns every vault decision; the app renders bob's answers (decision
`mac-capture-is-a-thin-client`). The app does not compose capture grammar such as
`^route:block-id#name` in Swift. There are three reasons:

1. **Mapping a reference to a reading task is ref-library semantics.** That covers v2
   residence, null/ambiguous/closed/v1 trackers, and refusal copy, and bob's resolver
   and `RefTaskView` already own it.
2. **The card needs two selectors capture's grammar doesn't have.** An existing row is
   picked by a stale-safe Pomodoro ref (`line:digest`), so unnamed Pomodoros are
   selectable too. A "New Pomodoro READ" row must not prefix-match an existing READING.
3. **One call should list the rows and their flags.** The listing call must return the
   Pomodoro rows with bob-computed `running`, `default`, and `linked` flags. The app
   then never re-derives the implicit current/next rule. That rule differs subtly from
   the agenda's `next` role.

The vault mutation itself is not re-implemented. `bob ref today` calls the same planner,
link guard, §13.5.1 dependency drop, schedule pull-forward, `[fresh::]` stamp, and
atomic commit path that `bob capture '^route:block-id[#name]'` uses. Its JSON embeds
capture's own item object, so the Mac decodes it with the existing
`CaptureCommandSuccess`, and the CLI and both clients describe a link in one vocabulary.

Live data behind this design, from the current vault's `bob ref list -R all -A -f json`:

- All 41 open reading tasks are v2 `^ref-<slug>` lines in root area or project notes: 10
  `[ ]`, 8 `[*]`, 3 `[/]` and 20 `[?]`.
- No open v1 in-note `^ref` trackers remain.
- 602 legacy references have no task.

Most Blocked refs are blocked by a future `scheduled` (capture pulls that forward) or by
their own `dependsOn` (capture drops those first). So the guard and pull-forward paths
are everyday paths, not edge cases.

## 3. Phase `cli`: `bob ref today` (bob-cli)

### 3.1 Command

```
bob ref today <REF> [-b|--bob-dir PATH] [-d|--dry-run] [-e|--exact] [-f|--format human|json] [-p|--pomodoro NAME] [-P|--pomodoro-ref LINE:DIGEST] [-r|--ref-dir PATH]
```

**Help and registration**

- The about line is "Pull a reference's reading task into today's Pomodoro (sets it
  Next)".
- Register `today` in `src/native/highlights_ref/cli.rs`:
  - `HELP_GROUPS` "Library", alphabetically after `show`;
  - `all_subcommands`;
  - a `REF_AFTER_HELP` example: `bob ref today <REF> -p reading`.
- Add dispatch in `src/native/highlights_ref/mod.rs`.
- Update the `bob ref` about string and the root `SUBCOMMANDS` description of `ref` in
  `src/runner.rs` so they mention planning a reference for today.
- Follow `cli_rules.md`: options alphabetical, every long option has a short alias, and
  colored human output.
- The after-help explains:
  - the selector table below;
  - that the rules match `bob capture '^route:block-id[#name]'`;
  - that it never starts a Pomodoro and never commits or syncs (vault-sync picks the
    change up, as with capture);
  - the `BOB_DAY_FILE` / `BOB_DIR` / `BOB_NOW` environment variables, as
    `capture-pomodoro-name` documents them.

**`REF` resolution**

`REF` resolves exactly as `bob ref show` and `bob ref rename` do: path, stem, id, URL,
arXiv ID, DOI, or exact title, through `src/native/ref_library/resolve.rs` and the
index. Resolution failures reuse `unknown_reference` / `ambiguous_reference` with
`candidates`. Generalize `print_show_error` in `src/native/ref_library/output.rs` to
take the command name rather than copying it.

**Selectors** (`-p` conflicts with `-P`; `-e` requires `-p`)

| Flags            | Destination                                                                                                                                                                                                                             |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| none             | Capture's no-name rule. If the task already has a dedicated live link under an open Pomodoro (Q), it stays there (`already_current`). Otherwise the implicit current/next entry: the single open timed entry, else the first open entry |
| `-p NAME`        | Capture's `#name` rule: whole-slug match among open entries, then prefix match, else create the canonical named future Pomodoro. Q's subtree moves                                                                                      |
| `-p NAME -e`     | Whole-slug match among open entries only, else create. Never a prefix match                                                                                                                                                             |
| `-P LINE:DIGEST` | Exactly that entry, resolved stale-safe through `PomodoroScan::by_ref` with the `ref` values the dry run lists. A completed entry gives `pomodoro_closed`; stale or ambiguous gives `stale_pomodoro`                                    |

### 3.2 Implementation seams

**Reading task.** Use the located task `ref list` reports: `RefTaskView`, built in
`src/native/ref_library/mod.rs` around the `task` field. Refuse a task that can't be
linked:

- `task: null` gives `no_reading_task`.
- The `multiple_open_ref_tasks` diagnostic gives `ambiguous_reading_task`.
- An in-note v1 `^ref` gives `v1_reading_task`, with hint `bob ref migrate-tasks`.
- `archived`, or a closed mark (`x`, `X`, `-`, or any non-linkable status), gives
  `reading_task_closed`. The message names the reading state, for example `already read`
  or `abandoned`.
- A residence that is not a root note gives `reading_task_unroutable`.

The route is the residence file stem. Fail loudly rather than guess if it isn't a valid
capture route.

**Capture planner.** Factor the `CaptureKind::PomodoroLink` branch of
`src/native/capture/plan.rs` (about lines 764-899) into a shared helper that builds the
`PlannedCaptureItem` / `CaptureItemResult`. Then add a crate-visible entry point in
`src/native/capture/`, for example `existing_task_link.rs`:

- It plans one existing-task Pomodoro link for `(route, block_id)` with a target
  selector, using `plan_pomodoro_link_capture`, the same `DependencyContext`, link guard
  and dependency drop, and `CaptureBatchPlanner`.
- It commits through `commit_capture_batch`, keeping pre-image validation and rollback.
- It returns the `CaptureItemResult` plus whether anything changed.

Capture's grammar, parse, and output are unchanged; `bob capture` keeps byte-identical
behavior.

**Ledger target.** Replace `plan_pomodoro_link_ledger`'s `pomodoro_name: Option<&str>`
in `src/native/capture_task_toggle/ledger.rs` with a target enum:
`Implicit | Named(&str) | ExactName(&str) | Entry(PomodoroRef)`.

- Capture maps only to `Implicit` and `Named`.
- `ExactName` uses whole-slug matching only (a sibling of
  `capture_pomodoros::select_named`), else `insert_named_placeholder`.
- `Entry` resolves with `scan.by_ref`, then shares the `Named` → `Found` branch: move
  Q's subtree, insert, or already-current.
- Expose a small read-only helper that reports the default destination (Q, else
  `select_implicit_open_entry`), so the listing's `default` flag comes from the same
  code as the write.

**Pomodoro rows.** Build them from `capture_pomodoros::scan`, with `starts_at`/`ends_at`
from the agenda helpers in `src/native/capture_pomodoros_agenda.rs`:

- `running` is `is_current`;
- `default` comes from the helper above;
- `linked` comes from `find_movable_task_links` for this task's `[[route#^block-id]]`.

The rows describe the pre-image on a dry run and the post-image after a write.

**Errors.** Classify from typed errors (`LinkRelocationError`, the link-plan error
behind `select_implicit_open_entry`, `PomodoroRefLookup`, and the capture commit's
pre-image mismatch). Where only a formatted string exists today, add a small typed kind.
Never string-match messages.

### 3.3 JSON contract

The envelope is schema 1 with `command: "ref today"`: compact one-line JSON on stdout
with no ANSI. Every field below is additive-friendly.

```json
{
  "ok": true,
  "schema_version": 1,
  "command": "ref today",
  "generated_at": "2026-10-16T09:12:00",
  "mode": "dry_run",
  "changed": true,
  "ref": {
    "path": "ref/papers/attention.md",
    "link": "[[ref/papers/attention]]",
    "title": "Attention Is All You Need",
    "marker_id": "attention"
  },
  "task": {
    "path": "sase.md",
    "block_id": "ref-attention",
    "link": "[[sase#^ref-attention]]",
    "status_symbol": " ",
    "status_name": "…"
  },
  "pomodoros": [
    {
      "ref": "41:0b1c2d3e",
      "line": 41,
      "name": "BUGS",
      "slug": "bugs",
      "time_range": "0920-0950",
      "placeholder": false,
      "starts_at": "2026-10-16T09:20",
      "ends_at": "2026-10-16T09:50",
      "running": true,
      "default": true,
      "linked": false,
      "task_link_count": 2
    },
    {
      "ref": "44:9a8b7c6d",
      "line": 44,
      "name": "READING",
      "slug": "reading",
      "time_range": null,
      "placeholder": true,
      "starts_at": null,
      "ends_at": null,
      "running": false,
      "default": false,
      "linked": false,
      "task_link_count": 0
    }
  ],
  "capture": {
    "…": "the exact item object bob capture -f json reports for ^sase:ref-attention, kind pomodoro_link or link_guard"
  }
}
```

**Fields**

- `mode` is `dry_run` or `write`.
- `changed` says whether any file changed, or would change.
- `pomodoros` lists every open top-level entry in document order:
  - `default` is true on at most one row;
  - `linked` is true on the row holding the task's live link.
- `capture` is serialized by the same code as `bob capture`'s item. A guarded gesture is
  `ok:true` with `capture.kind == "link_guard"`; it is a real write that promotes
  prerequisites, as in capture.

**Failures**

- Failures print the shared envelope
  `{"ok":false,"schema_version":1,"command":"ref today","error":{"code","message","hint"}}`.
  They add `candidates` for resolution failures, and `ref`, `task`, and `pomodoros`
  whenever resolution got that far. That way a no-selector dry run on a day with two
  running Pomodoros still returns rows for the card.
- Exit codes: 1 for failures, 2 for usage.

**Stable error codes**

| Code                      | When                                                                 |
| ------------------------- | -------------------------------------------------------------------- |
| `unknown_reference`       | REF matches nothing                                                  |
| `ambiguous_reference`     | REF matches several notes                                            |
| `no_reading_task`         | The reference has no reading task                                    |
| `ambiguous_reading_task`  | Several open reading tasks claim the reference                       |
| `v1_reading_task`         | The task is an in-note v1 `^ref` tracker                             |
| `reading_task_closed`     | The task is archived, read, or abandoned                             |
| `reading_task_unroutable` | The task's residence is not a root note                              |
| `no_daily_note`           | Today's daily note does not exist                                    |
| `no_pomodoros_section`    | The daily note has no Pomodoros section                              |
| `no_open_pomodoro`        | There is no open entry and no selector                               |
| `multiple_running`        | Implicit selection or named creation with several open timed entries |
| `invalid_pomodoro_name`   | bob's `POMODORO_NAME_USAGE` text                                     |
| `stale_pomodoro`          | The `-P` ref no longer matches an entry                              |
| `pomodoro_closed`         | The `-P` ref names a completed entry                                 |
| `ledger_invariant`        | For example, several movable links                                   |
| `note_changed`            | A pre-image mismatch at write time; retry                            |
| `io`                      | Any other I/O failure                                                |

### 3.4 Human output

The header is `✓ linked 📖 <title>`, or `would link` on a dry run. Below it, reuse
capture's pomodoro-link human lines from `print_human_pomodoro_link_success` in
`src/native/capture/output.rs`: the status transition, the ledger line, and the
dependency-drop and schedule chips. Refactor that function into a string-returning
helper rather than duplicating the wording.

`--dry-run` also prints an `Open Pomodoros` block. Each row shows:

- `●` for running, `◆` for default, `✓` for linked here;
- the name, or `(untitled)`;
- the time range or `planned`;
- `-P LINE:DIGEST`, so CLI users can target unnamed entries.

Refusals print `bob ref: error: …` plus a `hint:` line on stderr.

### 3.5 Docs

**`docs/ref.md`**

- Add a "Choose a command" row: "Read a reference today",
  `bob ref today <REF> [-p NAME]`, "Set its reading task Next and link it under today's
  Pomodoro".
- Add a `## bob ref today` section with:
  - the semantics and the selectors table;
  - the shared-planner statement and link to `docs/capture.md` "Linking and starting
    existing tasks";
  - a JSON example;
  - the error-code table;
  - a note that the tracker change reads `status_sync: pending` until the next
    `ref scan -w` (the Refs panel's ⌘S), exactly as an Obsidian Ctrl+Shift+Enter on a
    ref task does today.
- Extend "Errors and exit codes".

**Elsewhere**

- `docs/capture.md`: add one cross-reference sentence in "Linking and starting existing
  tasks". Say that the exact-name and entry-ref targets are `bob ref today`-only and
  that the capture grammar is unchanged.
- `README.md`: add the usage line to the `bob ref` block, in alphabetical order.
- Shell completion: check whether it is clap-derived or a static list
  (`docs/completion.md` and its tests) and include `today`.

### 3.6 Tests

Add `tests/cli/ref_library/today.rs` and register it in that directory's `mod.rs`. Use
the `tests/cli/support.rs` conventions: `BOB_DIR` or `-b`, `BOB_DAY_FILE`, `BOB_NOW`,
and the ref-library fixture vault extended with v2 tasks in root notes plus a daily
note.

**Parity.** In two identical temp vaults, `bob ref today <ref>` and
`bob capture '^sase:ref-x'` produce byte-identical residence notes and day files. The
same holds for `-p bugs` versus `^sase:ref-x#bugs`. The dry-run `capture` object equals
the write's, apart from `dry_run`.

**Selectors**

- default with Q → `already_current`, with the file byte-identical;
- default without Q → linked under running, else under the first open entry;
- named prefix (`-p bu` → BUGS);
- exact (`-p read -e` with READING open → creates READ);
- exact reuse (`-p reading -e` → READING);
- `-P` on an unnamed entry;
- `-P` stale → `stale_pomodoro`, with nothing written;
- `-P` completed → `pomodoro_closed`;
- moving Q to another entry, including into a newly created one.

**Lanes and guard**

- Ready → Next with `[fresh::]` stamped;
- Next stays Next; In Progress stays In Progress;
- Blocked by a single future `scheduled` → pulled forward, linked, and a Schedule Log
  entry written when a log exists;
- own `dependsOn` → dropped and linked;
- project-blocked → `capture.kind == "link_guard"` with `ok:true`, no link, and
  prerequisites promoted.

**Refusals**

- `no_reading_task` (legacy ref), `ambiguous_reading_task`, `v1_reading_task` (in-note
  `^ref`), `reading_task_closed` (archived, `[x]`, `[-]`);
- `no_daily_note`, `no_pomodoros_section`;
- `multiple_running` with no selector, where the envelope still carries `pomodoros`;
- `no_open_pomodoro` with no selector, and the same day with `-p focus` creates FOCUS;
- `invalid_pomodoro_name`;
- unknown or ambiguous REF with `candidates`.

**Contract and help**

- The listing flags (`running`, `default`, `linked`, `ref`, `starts_at`/`ends_at`) on a
  realistic day.
- `bob ref --help` lists `today` in Library; `bob ref today --help` includes the
  selectors.
- Existing capture and ledger tests stay green unchanged. Add unit tests for the new
  `ExactName` and `Entry` targets in `src/native/capture_task_toggle/tests.rs`.

### 3.7 Verification

Run `just check` (or `just all`) and the new test file. Then run a live smoke against a
scratch copy of a realistic vault: dry run, then write, on a v2 ref.

Paste the real `-f json` outputs for these cases into the phase notes; phase `mac_core`
copies them as fixtures:

- listing;
- linked;
- created;
- moved;
- already-current;
- link_guard;
- multiple_running;
- no_reading_task;
- stale_pomodoro;
- reading_task_closed.

## 4. Phase `mac_core`: RefsCore model, fetcher, and copy (bob-mac-capture, portable targets)

Open the repo with `/sase_repo` (`sase repo open bob-mac-capture -r "…"`), read its
`AGENTS.md`, and work only in the printed path. This phase touches only the portable
`RefsCore` and `CaptureCore` targets plus their tests and `Tests/Fixtures`.

### 4.1 Decoding

Add `Sources/RefsCore/RefsTodayLinkModels.swift` with `RefsTodayLinkResponse`
(`Decodable & SchemaVersioned`, schema 1):

- `ok`, `mode`, `changed`;
- `ref`, `task`;
- `pomodoros: [RefsTodayLinkPomodoro]`;
- `capture: CaptureCommandSuccess?`, the existing CaptureCore decoder;
- `error: RefsTodayLinkFailure?` (`code`, `message`, `hint`) and `candidates`.

Every non-core field uses `decodeIfPresent` so a newer bob keeps decoding. Name types
`TodayLink…` so they never collide with the existing `RefsToday` signals.

### 4.2 Fetcher

Add `BobRefsFetcher.linkToToday(notePath:selector:dryRun:)` as a concrete method,
without a protocol requirement yet; `mac_ui` adds that. `RefsTodayLinkSelector` is
`.automatic | .entry(ref:) | .exactName(_)`, mapping to `[]`, `["-P", ref]`, and
`["-p", name, "-e"]`.

- **argv:**
  `["ref","today","-f","json"] + (dryRun ? ["-d"] : []) + selectorArgs + ["--", notePath]`.
- **Call:** `client.decodeReport`, so exit 1 still decodes the error envelope.
- **Dry runs:** lane `refs-today-preview`, `cancelsPreviousInLane: true`, default 20 s
  timeout.
- **Writes:** lane `refs-today`, `cancelsPreviousInLane: false`, 30 s timeout.

Add `RefsTodayLinkProblemMapping`, mirroring `RefsRenameProblemMapping`:

- `timed_out`;
- exit 2 → `bob_too_old`, "This bob can't link references to today yet — update bob on
  this Mac";
- `transport`.

### 4.3 Card state machine

Add `RefsTodayLinkCard`, a pure `Equatable, Sendable` value in
`Sources/RefsCore/RefsTodayLinkCard.swift`.

**Identity and phases**

- It freezes the item's `id`, note path, title, and task caption.
- Phases are `.loading`, `.ready`, `.refused(message:hint:)`, `.running`, and
  `.failed(message:hint:)`. Listing errors in {`multiple_running`, `no_open_pomodoro`}
  are `.ready` with a `defaultProblem`; any other listing error is `.refused`.

**Ranking** (for display only; it mirrors the Obsidian picker)

- With an empty query: document order, with bob's `default` row highlighted.
- With a non-empty query, in tiers that keep document order within each tier:
  1. exact canonical slug;
  2. slug/name prefix;
  3. `FuzzyMatcher` over the name;
  4. time match (`0920`, `9:20`, `09:20`).

  Untitled rows match only by time. The highlight goes to the first visible row, else
  the create row.

**Selection → selector**

- Empty query and never arrowed → `.automatic`.
- An existing row → `.entry(ref)`.
- The create row → `.exactName(query)`.
- Shift-Return → `.exactName(query)`.
- Each change returns a `.preview(selector)` effect.
- `applyPreview(_:for:)` drops responses whose selector or generation no longer matches.

**Effects**

- `.preview(selector)`, `.submit(selector)`, `.reload` (after `stale_pomodoro`),
  `.cancel`, `.none`.
- During `.loading`, submit is allowed only with an empty query (`.automatic`); with a
  typed query it is ignored.
- `.running` blocks edits and keys, as the rename card does.

### 4.4 Key map

`RefsTodayLinkKeyMap` lives in RefsCore and is portable, like `RefsRenameKeyMap`. It
implements the §1.3 table and returns nil for keys the field should edit.

### 4.5 Presentation

Every string and derived value lives in `RefsTodayLinkPresentation`, built on
`CapturePomodoroLinkPresentation` / `CaptureLinkGuardPresentation` for link wording:

- header and placeholder variants;
- row title and meta, with `Nm left` / `Nm over` from `ends_at` against an injected
  `now`;
- pills, lane chip, and create-row title;
- preview line and hints;
- a `RefsTodayLinkToast` value (`style: linked | moved | already | guarded`, `title`,
  `value`, `detail`) per §1.5;
- the VoiceOver announcement;
- success and failure notification title and body;
- refusal copy per error code. bob's message is primary; app copy only adds the short
  headline, such as `No reading task to plan`, `Already read`, or
  `Today's daily note doesn't exist yet`.

### 4.6 Tests and verification

**fake-bob.** Add a `ref today` arm to `Tests/Fixtures/fake-bob`, next to the existing
`ref` arms. It honors `FAKE_BOB_REF_TODAY_FIXTURE` / `_EXIT` and records argv through
`FAKE_BOB_RECORD_PATH`. Add `Tests/Fixtures/refs-today-*.json` fixtures copied from
phase `cli`'s real output, with paths scrubbed.

**Tests**

- `RefsFetchingTests`: argv for each selector and for dry run versus write, lanes and
  cancellation flags, exit-1 envelope decoding, exit 2 → `bob_too_old`, and a schema
  mismatch.
- New `RefsTodayLinkTests`:
  - decoding, including unknown extra fields;
  - every card transition, ranking tier, and selection → selector case;
  - stale-preview drop;
  - key map;
  - every presentation string and toast variant;
  - the time math.

**Verification.** Run `just portable-check` on a host with Swift. If `swift` is missing
locally, use athena over the tailnet, after reading the `tailnet.md` memory through
`/sase_memory_read`.

## 5. Phase `mac_ui`: card view, routing, toast, and docs (bob-mac-capture, macOS targets)

Open the repo with `/sase_repo` as in `mac_core`.

### 5.1 Fetching protocol

Add `linkToToday(notePath:selector:dryRun:)` to `RefsFetching`. Update every conformer:

- `BobRefsFetcher`, which already has it;
- `FakeRefsFetcher` (`Tests/BobMacCaptureTests/RefsInspectorTests.swift`);
- `TimedOutRefsFetcher` and `ScriptedRefsFetcher` (`RefsLibraryTests.swift`).

### 5.2 Routing

In `RefsKeyRouter`, the `KeyCode.n` case keeps Ctrl-N → `.move(.next)` and adds
`flags.subtracting(.capsLock) == .command` → `.showTodayLinkCard`, or nil while marked
text is composing.

`RefsPanelController.processKeyEvent` sends keys to `RefsTodayLinkKeyMap` while the card
is open, ahead of the copy-card branch. A nil result passes the key to the field.

### 5.3 Model

In `RefsPanelModel`:

- **State:** `@Published todayLinkCard: RefsTodayLinkCard?` and
  `todayLinkCardGeneration`, used for focus.
- **Exclusivity:** opening the Today card closes Copy and Rename, and opening either of
  those closes the Today card.
- **Guards:** an `isTodayLinking` guard mirrors `isRenaming` across selection changes,
  `.select`, `.activate`, `.showActions`, `prepareForPresentation`, and `panelDidHide`.
- **Esc ladder:** the Today card comes first (unless running), then rename, copy,
  banner, and so on.
- **Open checks:** the same checks as `openRenameCard`, including the synthetic-tab
  warning.

**Flow**

1. On open, run the listing dry run (`.automatic`).
2. On each selection change, run a preview dry run, debounced about 90 ms.
3. Submit on the write lane.
4. On success:
   - close the card;
   - call a new `RefsLibrary.applyTodayLink(taskPath:blockID:pomodoroName:)`, which
     patches `signals.today.taskEntries` so the pink pill shows now (skipped for
     `link_guard`);
   - then `library.refresh(reason:)` with a new `.todayLink` reason;
   - discard the inspector cache entry for the item;
   - show the toast (or notify if hidden, per §1.5);
   - post the VoiceOver announcement.
5. Failures stay inline. `stale_pomodoro` triggers `.reload`.

### 5.4 View

Add `Sources/BobMacCapture/Refs/RefsTodayLinkCardView.swift`, rendered in the rename
card's slot in both `RefsPanelView` column layouts with the same transition. Implement
§1.2–§1.4 exactly:

- skeleton, empty, banner, refused, running, and failed states;
- the progress-ring glyph;
- pills, keycap, preview strip, and hints.

Use adaptive system colors with pink for Today and running. Honor dark mode, Increase
Contrast, Reduce Motion, and Reduce Transparency. Add accessibility labels per row, for
example `BUGS, running, 09:20–09:50, 2 tasks, selected`. Add any sizes as named statics
in `RefsVisualTokens`.

### 5.5 Toast

Generalize `RefsCopyHUD.swift` into a reusable glass toast controller, keeping the copy
HUD visually identical and its tests green.

- **Content:** the copy content, plus a Today style (`RefsTodayLinkHUDView`, §1.5) fed
  by `RefsTodayLinkToast`.
- **Placement:** the existing `centeredOn:` and a new `anchoredAboveBottomOf:` (the
  panel's frame, footer offset).
- **Timing:** hold duration per style.
- **Dismissal:** keep the rule that `show`, `represent`, and `hide` dismiss a live
  toast. With the stay branch the toast is shown after the card closes, and `represent`
  must not fire in that path.

> [!decision] after_link = hide Call the panel dismisser first, then show the toast
> `centeredOn:` the former frame, exactly as `copySucceeded` does.

### 5.6 Notifications

Add `notifyRefsTodayLink` and `notifyRefsTodayLinkFailure` to `NotificationService`,
with `nonisolated static` content builders like the rename ones. Wire the notifiers in
`AppDelegate` next to the rename notifiers.

### 5.7 Discoverability

- **⌘K menu:** add `RefsAction.linkToToday` ("Link to Today…", ⌘N), placed after Copy in
  the middle section.
- **Footer:** add `⌘N Today` after `⌘Y Copy` in the widest two non-tabs hint variants
  and the widest tabs variant.
- **Tests:** update the pinned menu-section, footer, and shortcut-hint tests
  (`RefsInspectorTests`, `RefsFooterTests`, `RefsPanelGeometryTests`).

### 5.8 README

- Add the key table row: `Command-N | Open the Link to Today card for the selected row`.
- Add a `Link to Today card` section after the Rename card section, covering keys,
  states, the toast, and what bob decides.
- Add the action to the ⌘K list.
- Update the Esc ladder row.
- Mention `bob ref today` wherever the README lists the bob subcommands the app runs.

### 5.9 Tests

**`RefsKeyRouterTests`**

- ⌘N, and ⌘N with Caps Lock;
- nil while marked text is composing;
- Ctrl-N still moves;
- plain N is still nil.

**`RefsPanelModelTests`**, with `FakeRefsFetcher`:

- open guards and exclusivity;
- the Esc ladder;
- running guards;
- the listing → preview → submit argv sequence;
- the debounce and stale-preview drop;
- success: the card closes, the overlay is applied, a refresh is requested, the toast is
  requested, and the notifier fires only when hidden;
- an inline failure;
- `stale_pomodoro` reload;
- `bob_too_old`.

**Design tests**

- A new `RefsTodayLinkCardDesignTests` through `RenderFixtureWriter`, in light and dark:
  - loading;
  - ready with a running default;
  - a typed query with the create row;
  - two running;
  - no open Pomodoros;
  - guard preview;
  - refused (no reading task);
  - linking;
  - failed.
- Today-toast renders for linked, moved, already, and guarded.

### 5.10 Verification

Follow the repo's `AGENTS.md`:

1. Run `just portable-check`.
2. Best effort, compile a scratch copy on the Mac: rsync without `.build` and `.git`,
   then `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
   Bryan's checkout or `just install` there.
3. If the Mac is offline, say in the report that the macOS targets are unverified.

These Refs macOS CI tests are known to be flaky; do not chase them:

- bob-cli-61 (RefsPanelModelTests refresh-reorder timeout);
- bob-cli-67 (RefsLibrary follow-up coalescing);
- bob-cli-7h (RefsInspectorScrollTests paging);
- bob-cli-7p (queued-digit copy test).

The report should also say that the Mac's `bob` must be updated to the phase `cli` build
for ⌘N to work. Until then, the card shows the `bob_too_old` refusal.

## 6. Acceptance

**Fast path**

- ⌘N then Return on a Ready v2 ref with a running BUGS Pomodoro:
  - the task line gets `[*]` and `[fresh::]`;
  - `[[sase#^ref-…]]` appears under BUGS;
  - the row gains a pink `BUGS` pill in the same frame as the toast
    `Linked under BUGS · Ready → Next`;
  - the panel stays open (stay branch).
- ⌘N ⌘N does the same.

**Choosing a Pomodoro**

- Typing `read`, then Shift-Return, while READING is open creates a READ Pomodoro.
- Return with `read` typed links under READING.
- Arrowing to an untitled planned Pomodoro links there.
- ⌘N on an already-linked ref preselects the `Here` row. Return toasts `Already in BUGS`
  and leaves the files byte-identical.

**Blocked refs**

- A ref Blocked only by a future `scheduled` links and shows `removed future schedule`.
- One Blocked only by its own `dependsOn` links and shows `dropped dependency on …`.
- A project-blocked one previews `Would not link — …`. Return then toasts `Not linked`
  with bob's guard detail.

**Refusals and safety**

- A legacy ref with no task opens a calm refused card.
- Two running Pomodoros show the banner, and empty-query Return beeps.
- A ledger edited between listing and submit yields an automatic row reload, never a
  wrong-line write.
- Every vault change is identical to what `bob capture '^route:block-id[#name]'` would
  write.

## 7. Non-goals

- **Starting the Pomodoro** (`=<X>`): there is no session start from this card.
- **Changing capture grammar**, plugin gestures, or the Obsidian picker.
- **Creating reading tasks** for legacy trackerless refs.
- **Writing PDF markers:** `status_sync` stays `pending` until ⌘S, as it does today.
  bob-cli-6f tracks making the reading task the sole status authority.
- **Unlink or release from the card:** Alt+N release stays an Obsidian gesture.

## 8. Related beads

- **bob-cli-73:** Mac rendering of capture `link_guard` results. This card reuses
  `CaptureLinkGuardPresentation` and should not duplicate that work.
- **bob-cli-6f:** the reading task as sole status authority.
- **bob-cli-6c:** stale glossary ref-task terms, corroborated during this planning.
