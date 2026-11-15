---
tier: tale
title: Root unattributed Event Tasks at meet_inbox and make meet.md a notebook
goal:
  bob gcal roots Event Tasks with no other root at a new meet_inbox area under inbox,
  and meet.md stops being an area.
size: medium
decisions:
  meet_parent:
    ask: Which note should own meet.md once it becomes the Meetings notebook?
    choices:
      meet_inbox:
        Catalog in meet_inbox.md lists Meetings, with Legacy meet notes nested under it
      gtd:
        Catalog in gtd.md lists Meetings beside GTD ideas; meet_inbox.md gets no catalog
      job:
        Catalog in job.md lists it, though job.md covers job searches and many meetings
        are personal
    default: meet_inbox
    why: Keeps meeting notes beside unattributed meetings, like legacy_inbox under inbox
    answer: meet_inbox
  pin_config:
    ask:
      Should chezmoi's bob config pin gcal.root to meet_inbox so hosts on an older bob
      keep a valid root?
    choices:
      pin:
        Live root line; every host is right on its next chezmoi apply, though it repeats
        the default
      comment:
        Only the commented example changes; hosts on an older bob write rootless rows
        until rebuilt
    default: pin
    why:
      Vault sync reaches every host in minutes; when each host rebuilds bob is unknown
    answer: pin
  review_time:
    ask:
      Should built-in review_time stop counting MEET minutes now that default-rooted
      meetings sit under gtd?
    choices:
      keep:
        Unchanged; meetings rooted at meet_inbox count as gtd review time until you
        re-root them
      exclude:
        Built-in review_time gains exclude_themes MEET; docs/stats.md and registry tests
        change
    default: keep
    why: review_time already counts every gtd-tree minute, bob project work included
    answer: keep
  memory_inbox_file:
    ask:
      Should the Inbox File glossary strand list meet_inbox.md as an inbox file that
      roots default Event Tasks?
    memory:
      - glossary:inbox-file
    default: true
    corrects: (today `mac_inbox.md`, `gkeep_inbox.md`, and `gmail_inbox.md`)
    answer: true
  memory_meet_record:
    ask:
      Should a new decisions strand record that meet.md is a notebook and Event Tasks
      default to meet_inbox?
    memory:
      - decisions:event-tasks-default-to-meet-inbox
    default: false
    answer: false
proposed_by: bbugyi200.apollo.e5
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 07:15:10
status: wip
---

# Root unattributed Event Tasks at `meet_inbox`; make `meet.md` a notebook

## Goal

Today `bob gcal sync` writes `[root:: meet]` on every Event Task that has no other root
source, because `meet.md` is a top-level area note. Bryan has ruled that `meet.md` must
not be an area: a meeting is not a sphere of responsibility, and its time belongs to the
area or project it serves (job, love, body, and so on). An Event Task with no
attribution is an untriaged item.

After this change:

1. A new area note, `~/bob/meet_inbox.md`, is a child of `~/bob/inbox.md` and is the
   default root parent for Event Tasks (`[root:: meet_inbox]`).
2. bob-cli's built-in `gcal.root` default is `meet_inbox`, not `meet`.
3. `~/bob/meet.md` becomes the **Meetings notebook**. It keeps its role as the
   `[[meet]]` hub for meeting notes, but it is no longer an area and no longer a root
   parent.

## Why `meet.md` becomes a notebook

- Its real content is meeting _notes_: the `Related:` hub links (`the_monthly`,
  `all_hands`, `recur_meet`, `legacy_meet`), many prose `[[meet]]` links across the
  vault ("[[meet]] notes", "Meeting Notes" in `work.md`), and the `legacy_meet`
  notebook. It holds no tasks; its `## Tasks` section is empty.
- A notebook is "a named collection of related notes without the obligations of an area
  or project". It is never a root parent, Pomodoro home, Ready lane, or review item.
  That is exactly what `meet.md` should stop being.
- It keeps `legacy_meet` valid: a notebook may parent another notebook. If `meet.md`
  became an untyped hub instead, `legacy_meet` would fail with `notebook_parent_kind`.
- A capture `@meet` then writes a meeting note into `meet.md` (notebook route), instead
  of filing a task in a non-area.
- The untyped zorg children (`the_monthly`, `all_hands`, `recur_meet`, with
  `parent: [[meet]]`) need no change.

## Facts established during planning (do not re-derive)

- The default lives in one constant, `DEFAULT_ROOT = "meet"` in
  `src/native/gcal/config.rs` (used at `from_settings`). Bryan's live
  `~/.config/bob/config.yml` does **not** set `gcal.root`. Its chezmoi source,
  `home/dot_config/bob/config.yml` in the `chezmoi` linked repo, has only the commented
  line `#   root: meet # default [root::] for Event Tasks` under `gcal:`.
- The default root is validated by `validate_root_with` in `src/native/gcal/root.rs`. It
  must resolve to an area at any depth, or to a root project, that is its own root
  parent. A nested area such as `meet_inbox` (under `inbox` → `gtd`) qualifies. A
  notebook does not: it resolves to its owner's home, so it is rejected as
  `root_field_not_root`.
- **MEET type inheritance.** `render::needs_meet_type` writes `[type:: MEET]` only when
  the root walk does not already suggest MEET. `meet.md` currently supplies
  `task_type: MEET`. Without that key, the walk `meet_inbox → inbox → gtd` would reach
  `gtd`'s `task_type: PLAN`. PLAN is in `plan.exempt`, so hand-written rows rooted at
  `meet_inbox` would then stop counting toward the budget. `meet_inbox.md` therefore
  **must** carry `task_type: MEET`.
- **Hues.** The MEET _type_ hue comes from config (`types: MEET: blue`), so making
  `meet.md` a non-area does not change it. The _root parent_ hue (Day Strip and Stats
  bars) uses the nearest area or project `color:`. Without a color, `meet_inbox` would
  fall through to `gtd`'s grey. So `meet_inbox.md` carries `color: violet`, the color
  `meet.md` has today.
- **Clock.** Neither `meet.md` nor `inbox`/`gtd` declares `offclock` or
  `offclock_types`, so default-rooted meetings stay on the clock, as today.
- **Stats.** `review_time` (`in: [gtd]`) follows the root's residence chain. Minutes
  from meetings rooted at `meet_inbox` therefore count toward it; see the `review_time`
  decision.
- **Inbox semantics.** `meet_inbox.md` becomes an inbox file: an area whose parent
  resolves to `inbox.md`. Inbox detection is by residence path, and Event Tasks reside
  in daily notes, so no Event Task gains inbox-routing prompts. Route pickers never
  offer inbox files, so tasks will not be routed into `meet_inbox`. Set `ready_cap: off`
  like its siblings.
- **Vault usage.** Only one daily-note row still says `[root:: meet]`:
  `~/bob/2026/20261030.md`, block `^product-engineer-bryan`. It is closed, and its gcal
  journal entry has `series_id: null`, so a hand edit teaches no series memory. Other
  matches are prose or preserved sources that must stay untouched: `done/bob_done.md`,
  `done/gkeep_inbox_done.md`, and `.bob/ref-sources/**` (write-once). No `@meet` route
  tokens exist anywhere. The gcal journals (`.bob/gcal/*.jsonl`) hold past
  `"root":{"source":"default","value":"meet"}` records. Leave them alone; sync owns
  them.
- **bob-plugins and bob-mac-capture.** Every `"meet"` there is the MEET _type/Pomodoro
  name_, not the note. The Mac's `Tests/Fixtures/gcal-note/*.json` copies of bob-cli
  goldens are opaque sample data and have already drifted from bob-cli. **Do not edit
  either repo.**

## Steps

### 1. bob-cli code

- `src/native/gcal/config.rs`: `DEFAULT_ROOT = "meet_inbox"`; update the unit assertion
  `assert_eq!(config.root(), "meet")`. Leave the explicit `root: meet` YAML parse test
  in `src/native/config/gcal.rs` as is: it tests parsing, not the default.
- `src/native/gcal/root.rs`: change the `invalid` rejection hint
  `"write the bare note name, for example meet"` to `… for example meet_inbox`. Grep
  tests for that string.
- Do not change MEET-name logic (`is_meet_name`, `normalize_component(..) == "meet"`,
  `[type:: MEET]`): it is about the type, not the note.

### 2. bob-cli tests

Update only the tests that depend on the default, so that their fixture vault and
expectations use `meet_inbox`:

- `tests/gcal_support/mod.rs`: rename `write_meet_area` to `write_meet_inbox_area`. It
  writes `meet_inbox.md` with `---\ntype: area\ncolor: violet\n---\n# Meet Inbox\n`, and
  its doc comment changes to match. Update the caller in `prepare_healthy_vault`.
- `tests/gcal_lifecycle.rs`, `tests/gcal_notes.rs`, `tests/gcal_sync.rs`,
  `tests/gcal_diagnostics.rs`, `tests/gcal_list.rs`, `tests/gcal_auth.rs`: each
  `prepare` that writes `meet.md` as an area writes `meet_inbox.md` instead, keeping its
  keys (`color: violet`, `task_type: MEET`). `[root:: meet]` becomes
  `[root:: meet_inbox]`, and JSON root expectations (`"root": "meet"`,
  `relearned["root"] == "meet"`, `primary["root"] == "meet"`) become `"meet_inbox"`.
  That includes tests that deliberately edit a row back to the default root. Be careful
  **not** to touch `[type:: MEET]`, `— MEET`, `=meet`, `meet-…` fixture ids,
  `meet.google.com` URLs, or `meet_…` series ids.
- Goldens: run `BOB_UPDATE_GCAL_NOTE_FIXTURES=1 cargo test --test gcal_note_cli`, then
  review `git diff tests/fixtures/gcal_note/json`. The only changes should be root
  `value` strings and `[root:: …]` inside `row` strings (`meet` → `meet_inbox`), with
  the hue still `violet`. Any other drift is a bug; stop and investigate it.
- Leave the unit tests that pass `"meet"` explicitly as a scratch root
  (`src/native/gcal/root.rs`, `render.rs`, `plan/tests.rs`, `src/native/areas/*`,
  `pomodoro_themes/*`). Leave the `tests/fixtures/capture_pomodoros` / `pomodoro_themes`
  `meet` slugs (MEET type). They do not depend on the default.
- New coverage, matching the real vault shape:
  - A unit test in `src/native/gcal/root.rs`. Scratch vault: `gtd.md` (area,
    `task_type: PLAN`, `color: grey`); `inbox.md` (area, `parent: "[[gtd]]"`);
    `meet_inbox.md` (area, `parent: "[[inbox]]"`, `color: violet`, `task_type: MEET`);
    and `meet.md` (`type: "[[notebook]]"`, `parent: "[[meet_inbox]]"`). Assert that
    `validate_root_candidate(.., "meet_inbox")` returns a target whose
    `home.root_parent` is `meet_inbox.md` and whose hue is violet. Assert that `"meet"`
    is rejected with `reason == "root_field_not_root"`, grouping under `meet_inbox`.
  - An integration test in `tests/gcal_sync.rs` with that same nested vault and no
    `gcal.root`. Sync one plain event. The row carries `[root:: meet_inbox]` and **no**
    `[type:: MEET]`, which proves MEET is inherited from `meet_inbox` rather than PLAN
    leaking from `gtd`.

> [!decision] review_time = exclude In `src/native/stats/registry.rs`, add
> `exclude_themes: [MEET]` to the built-in `review_time` YAML and extend its description
> ("…excluding MEET sessions"). Update the registry tests that pin `review_time`
> parameters, plus the `docs/stats.md` row for `review_time` (and M10 if it states
> parameters). Add a test that a MEET Pomodoro linked to a `meet_inbox` row contributes
> 0 to `review_time`.

> [!decision] review_time = keep Change no stats code or stats docs.

### 3. bob-cli docs

Read the `docs.md` reference memory first. Then make these edits:

- `docs/gcal.md`: the YAML example `root: meet # default [root::]` becomes
  `root: meet_inbox`; "then `gcal.root` (default `meet`)" becomes `meet_inbox`.
- `docs/reference/configuration.md`: the `gcal.root` row's default becomes `meet_inbox`.
- `docs/guides/connect-google-calendar.md`: the YAML example becomes `root: meet_inbox`.
  The doctor row "Create `meet.md` as `type: area`…" becomes "Create `meet_inbox.md` as
  `type: area` (Bob's own vault files it under `inbox`), or set `gcal.root` to an
  existing area."
- `docs/pomodoro-homes.md` (R8 example list): drop `meet: violet` from "top-level area
  assignments" and add that the `meet_inbox` sub-area (under `inbox`) sets `violet`, so
  default-rooted meeting time keeps its hue.
- `docs/reference/vault-format.md`: the inbox-files sentence "(`mac_inbox.md`,
  `gkeep_inbox.md` today)" becomes "(`mac_inbox.md`, `gkeep_inbox.md`, `gmail_inbox.md`,
  and `meet_inbox.md` today)".
- Leave `docs/off-clock-link-chips.md` OR8b/OR8c alone (synthetic vectors), as well as
  the generated `docs/develop/decisions.md` and `docs/reference/glossary.md`. Those
  regenerate only through `just docs-gen` after a memory change.
- Run `just docs-check`.

### 4. Vault edits (`~/bob`, live; vault sync commits them)

Before editing, re-run
`rg -n '\[root:: ?meet\]' ~/bob --glob '!.bob/**' --glob '!done/**' --glob '!_conflicts/**'`
in case a host synced new default-rooted rows after planning.

1. **Create `~/bob/meet_inbox.md`**:

   ```markdown
   ---
   parent: "[[inbox]]"
   type: "[[area]]"
   summary:
     Default root for calendar Event Tasks no other root claims; re-root each to the
     area it serves.
   color: violet
   task_type: MEET
   ready_cap: off
   ---

   # Meet Inbox

   `bob gcal sync` writes `[root:: meet_inbox]` on a day's `## Events` row when no
   private event note, `@route`, event colour, or calendar root names another root.
   Re-root each row to the area or project the meeting serves. Meeting notes live in
   [[meet]].
   ```

> [!decision] meet_parent = gtd End the body's last sentence at "serves." and drop the
> `[[meet]]` link (the same applies for `meet_parent = job`).

2. **Rewrite `~/bob/meet.md`** as the Meetings notebook. Replace the whole file:

   ```markdown
   ---
   parent: "[[meet_inbox]]"
   type: "[[notebook]]"
   summary:
     Notes from meetings, appointments, and calls, plus links to recurring-meeting
     notes.
   ---

   # Meetings

   Calendar events land in each day's `## Events` rooted at [[meet_inbox]] unless the
   event names another root (see `bob gcal`); re-root each to the area or project it
   serves.

   - Related: [[the_monthly]] [[all_hands]] [[recur_meet]] [[legacy_meet]]
   ```

> [!decision] meet_parent = gtd Use `parent: "[[gtd]]"`; everything else in the file is
> the same.

> [!decision] meet_parent = job Use `parent: "[[job]]"`; everything else in the file is
> the same.

This removes the area-only keys (`type: "[[area]]"`, `color: violet`, and
`task_type: MEET`, which now live on `meet_inbox.md`), the empty `## Tasks` heading, and
the `## Notebooks` callout. A notebook is not a catalog host, so bob would never
maintain that callout again; `legacy_meet` now appears nested under Meetings in the
owner's catalog. It also removes the stale claim that events land with `[root:: meet]`.

3. **`~/bob/inbox.md` prose**: change "Root of the inbox files: [[mac_inbox]],
   [[gkeep_inbox]], and [[gmail_inbox]] hold new tasks awaiting triage." to "Root of the
   inbox files: [[mac_inbox]], [[gkeep_inbox]], and [[gmail_inbox]] hold new tasks
   awaiting triage, and [[meet_inbox]] roots calendar Event Tasks until each is
   re-rooted to the area it serves." Do not hand-edit its `> [!areas]` /
   `> [!notebooks]` callouts.

4. **Migrate the default-rooted row(s)**: in `~/bob/2026/20261030.md`, change
   `[root:: meet]` to `[root:: meet_inbox]` on the `^product-engineer-bryan` row. Change
   nothing else on the line. If the pre-check above found more rows in canonical daily
   notes, check each one's `series_id` in `.bob/gcal/*.jsonl` first. Migrate
   non-recurring rows the same way. Leave recurring ones for the next sync to update,
   and report them: a hand edit would teach series memory. Never touch `done/`,
   `.bob/ref-sources/`, or the journals.

5. **Regenerate catalogs** with the installed `bob`, previewing first:
   `bob areas sync -d`, then `bob areas sync`; `bob notebooks sync -d`, then
   `bob notebooks sync`. Expected: `meet` leaves `area.md`'s `## Areas`; `meet_inbox`
   appears under `inbox` in `area.md`, `gtd.md`, and `inbox.md`; and the owner's
   `## Notebooks` gains Meetings with Legacy meet notes nested. If the owner is
   `meet_inbox`, bob creates that callout at the end of `meet_inbox.md`.
6. **Check**: `bob areas check` reports no findings for `meet_inbox`/`inbox`/`area`. In
   `bob notebooks check`, `meet.md` and `legacy_meet.md` gain no errors (pre-existing
   findings elsewhere are not in scope).

### 5. chezmoi config

Open the repo with `sase repo open chezmoi -r "…"` and read its `AGENTS.md`. In
`home/dot_config/bob/config.yml` under `gcal:`:

> [!decision] pin_config = pin Replace the commented `#   root: meet …` line with the
> live line `  root: meet_inbox # default [root::] for Event Tasks`. Keep the
> surrounding commented keys as they are. Then run
> `chezmoi apply ~/.config/bob/config.yml` on this host and confirm with
> `grep -n 'root: meet_inbox' ~/.config/bob/config.yml`.

> [!decision] pin_config = comment Change only the comment, to
> `#   root: meet_inbox # default [root::] for Event Tasks`.

Commit chezmoi through the normal finalizer flow. Other hosts pick the change up on
their next chezmoi apply.

### 6. Memory (only per the memory decisions)

Use `/sase_memory_write` before touching memory.

> [!decision] memory_inbox_file In `sase/memory/glossary/inbox-file.md`, change "(today
> `mac_inbox.md`, `gkeep_inbox.md`, and `gmail_inbox.md`)" to include `meet_inbox.md`.
> Add one clause to the defaults sentence: `meet_inbox.md` holds no tasks, and
> `bob gcal sync` writes `[root:: meet_inbox]` on Event Tasks that no other root source
> claims. Keep the rest of the strand as it is.

> [!decision] memory_meet_record Add
> `sase/memory/decisions/event-tasks-default-to-meet-inbox.md` in the existing record
> shape (frontmatter `origin: requested`, `scope: project`, `aliases`, `summary`; then
> **Applies to / Claim / Why / Cost / Reopens when / Evidence**). The claim: `meet.md`
> is the Meetings notebook, never an area, and `gcal.root` defaults to `meet_inbox`, an
> inbox area under `inbox` that holds no tasks. The why must cite only Bryan's stated
> reason ("the `meet.md` file should NOT be an area note") and the vault task
> `bob.md ^meet-inbox`. Do not invent further rationale. The cost: default-rooted
> minutes sit under `gtd`, which matters for `review_time`, and the record must state
> the chosen `review_time` answer. Add a `[[glossary:inbox-file]]` link.

After either memory change, run `sase memory init`, then `just docs-gen`, so the
generated glossary and decisions pages update, then `just docs-check`. If neither memory
decision is accepted, skip all three commands.

## Verification

- `just all` (fmt, lint, test) passes in bob-cli.
- With the workspace build and the edited vault, run
  `cargo run -q -- gcal doctor -b ~/bob`. The `default root` check reads
  `meet_inbox → meet_inbox.md · area` with a violet dot. Checks unrelated to the vault
  (gog, writer) may fail on this host; report them, don't fix them.
- Optional: `cargo run -q -- gcal list -b ~/bob -D tomorrow` (requires gog).
  Default-rooted events show `meet_inbox (default)`.
- Re-run the `rg` from step 4: no `[root:: meet]` remains in canonical daily notes,
  except any recurring rows you reported.

## Commit

Use the bob-cli subject `feat(gcal)!: root Event Tasks at meet_inbox by default`, with a
body naming the new `gcal.root` default and that `meet.md` is no longer an area. A
changed default is observable in `gcal list` / `gcal note` JSON (`root.value`), hence
`!`. Never edit `Cargo.toml`'s version or `CHANGELOG.md`. The vault commits itself
through vault sync; `bob areas sync` and `bob notebooks sync` make their own guarded
commits.

## Out of scope

- bob-plugins and bob-mac-capture (no note-level `meet` references).
- Re-rooting historical rows to specific areas (Bryan's call), and editing past decision
  records that mention `meet` as historical evidence.
- `job.md`'s `Related: … [[meet]]` and other prose `[[meet]]` links; they still resolve,
  now to the Meetings notebook.
