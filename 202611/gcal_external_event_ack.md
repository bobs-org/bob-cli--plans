---
tier: epic
title: Ack external Google Calendar events (bob gcal ack)
goal: "Every external Google Calendar event bound for an Event Task is asked about once
  through a one-key `bob gcal ack` review queue; until acked (or noted) its open row
  wears a hard-to-miss Bob-owned warning tag, sync keeps creating rows regardless, and a
  weekly vault chore drives the sweep.

  "
decisions:
  tag_spelling:
    ask: Which exact tag should sync write on unacked external Event Task rows?
    choices:
      unacked:
        "#unacked — names the state; matches the `bob gcal ack` command (research pick)"
      needs_ack: "#needs_ack — names the action; reads as an imperative in the vault"
    default: unacked
    why: It names the row's state, pairs with the command noun, and research chose it.
    answer: unacked
  mem_decision_record:
    ask: File the new decision record decisions:external-gcal-events-are-acked-once?
    memory:
      - decisions:external-gcal-events-are-acked-once
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  mem_glossary_term:
    ask: Add the new glossary strand glossary:event-ack for the Event Ack term?
    memory:
      - glossary:event-ack
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
phases:
  - id: journal
    title: gcal.ack journal kind, ack fold, and the external classifier
    depends_on: []
    size: medium
    description:
      "journal: add the gcal.ack record (parse, fold into an AckBook, append),
      note-history ack evidence, the load_state skip, the account guard, reset/-F
      survival, model.rs is_external(), and the gcal.ack config kill switch."
  - id: tag
    title: Unacked row tag reconciliation and sync/list surfaces
    depends_on:
      - journal
    size: medium
    description:
      "tag: teach split_suffix the warning-tag marker, reconcile the tag through the
      want/present/last table in the sync planner (including the delete-the-tag inferred
      ack), journal the additive unacked row field, and surface warnings in sync and
      list human and JSON output."
  - id: cli
    title: The bob gcal ack command
    depends_on:
      - journal
      - tag
    size: large
    description:
      "cli: build the interactive one-key review queue (termios raw mode), the note
      branch through the shared note writer, one-shot/list/clear/dry-run/JSON forms,
      exit codes, the note.rs sibling-list fix, doctor acks section, stale-seen notices,
      and shell completion."
  - id: docs
    title: Docs, decision record, and glossary
    depends_on:
      - cli
    size: medium
    description:
      "docs: author docs/gcal-acks.md, update docs/gcal.md (commands, row-ownership
      table, JSON, known limits) and docs/gcal-notes.md cross-links, keep
      docs-gen/docs-check green, and file the decision record and glossary strand
      through /sase_memory_write."
  - id: vault
    title: Weekly reminder row at rollout
    depends_on:
      - cli
    size: small
    description:
      "vault: after the release train ships the command, add the weekly Monday ack chore
      row to ~/bob/gtd_daily.md and sanity-check the rollout."
proposed_by: bbugyi200.athena.1c2
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 16:39:02
status: wip
---

- **PROMPT:**
  [prompts/202611/gcal_external_event_ack.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gcal_external_event_ack.md)

# Plan: Ack external Google Calendar events (`bob gcal ack`)

## Context

Today an external Google Calendar event (one Bryan does not organize) lands as an Event
Task with a default root, no buffer, and on-clock billing unless he happened to run
`bob gcal note` first. This epic adds a **soft** ack requirement: sync still always
creates the row (a missed meeting costs far more than a wrong root), but until Bryan
reviews the event — an explicit no-op **ack** or a saved private **note** — the open row
carries a hard-to-miss, Bob-owned warning tag, and a new one-key-per-card
**`bob gcal ack`** queue makes the review take seconds.

The design was settled by consolidated research that every phase worker MUST read before
starting:

```bash
sase artifact read research:202611/gcal_external_event_ack/gcal_external_event_ack.md "Implementing the ack epic"
```

Bryan accepted all of that research's requirement adjustments (R1–R11), including: the
command is **`bob gcal ack`**, never `bob gmail ack` (the original request's `gmail` was
a slip — do not ship a gmail alias); the queue covers **only events that will land as
Event Tasks**; and a **saved private note counts as an ack**. All research line-number
anchors were re-verified against this repo at `70f69c23` (identical to the research's
`b04120b0` for every cited file).

Core vocabulary used below:

- **External event**: `!organizer_is_self || event_type == "fromGmail"`.
  `organizer_is_self` is derived at `src/native/gcal/model.rs:285` as
  `raw.organizer.is_some_and(|o| o.is_self)`, so a missing organizer is conservatively
  external. Never use `organizer_or_no_guests` (true for guest-less family-calendar
  events, which must enter the queue). `fromGmail` events (flights, reservations) are
  task-capable (`src/native/gcal/classify.rs:134`), list Bryan as organizer, and lock
  their Google description — exactly the events where a `-90 Drive to the airport`
  buffer matters most.
- **Acked**:
  `acked(event) = live gcal.ack on the event or its series (acked: true wins the fold) OR any persisted nonempty gcal.note record ever for the event or its series`.
  Review is a historical gesture: clearing a note's text never un-acks; clearing the
  sole explicit ack (`ack -c`) reopens review only when no note evidence remains.
  Existing notes qualify on upgrade with no rewrite.
- **The warning**: an exact, Bob-owned tag written by sync as the last token before the
  row's `^id`, on open rows only. Deleting the tag by hand is itself an ack.

> [!decision] tag_spelling = unacked The tag token is `#unacked` everywhere below (code,
> marker, docs, glossary, vault copy).

> [!decision] tag_spelling = needs_ack Spell the token `#needs_ack` everywhere the plan
> says `#unacked`: the rendered tag, the `split_suffix` marker, sync reports, docs, the
> glossary strand, and the decision record. Nothing else changes.

## Out of scope (do not build in this epic)

- **v2 display surfaces**: the amber Ledger Tools pill, MEET link-time dot
  (bob-plugins), and a Mac Capture **Ack** action driven by `bob gcal ack -i … -f json`.
  Cross-repo work, deliberately after the CLI is real (thin-client decision). A
  follow-up task bead covers this.
- **Automatic re-ack on reschedule (restack)**: v1 acks survive edits of the same event
  identity; we record `seen` and surface a stale notice instead (see phase `cli`).
- **The full ownership classifier** (organizer-email identity, calendar `accessRole`):
  the documented reopen path if a shared or read-only calendar is ever configured.
- **Any Google write**: the grant stays read-only
  (`decisions:bob-gcal-is-read-only-and-proposes-once`).
- **Release mechanics**: bob-cli ships through the daily release train; never edit the
  Cargo.toml version or CHANGELOG.md. Everything here is additive — no `!` break marks.

## Phase `journal` — gcal.ack journal kind, ack fold, and the external classifier

All in `src/native/gcal/`. This phase is pure model/persistence; no behavior change yet.

**Record shape** — a new kind beside `gcal.note` in `.bob/gcal/<host>.jsonl`:

```json
{
  "v": 1,
  "ts": "2026-11-16T07:12:04-05:00",
  "host": "kellys-macbook-pro",
  "kind": "gcal.ack",
  "account": "you@example.com",
  "key": { "calendar_id": "primary", "event_id": "…" },
  "scope": "event",
  "acked": true,
  "via": "ack",
  "seen": { "title": "…", "start": "…", "end": "…", "original_start": null }
}
```

- `scope: event` keys `{calendar_id, event_id}`; `scope: series` keys
  `{calendar_id, series}` on Google's `recurringEventId` (instance IDs already embed
  original start; never key by current start time or bare `iCalUID`).
- `acked: false` is a deliberate clear (`bob gcal ack -c`); last writer wins.
- `via` ∈ `ack | row | mac | plugin` — display/debugging only.
- `seen` (title/start/end/original_start at ack time) is for display and stale detection
  only, never identity transfer.

**Work:**

1. `journal.rs`: `parse_ack_record` (shape/size limits mirroring `parse_note_record`,
   journal.rs:733, including the malformed-line warning budget), an `AckBook` fold
   mirroring `NoteBook`/`load_notes` (journal.rs:533): last-writer-wins per
   `(account, calendar, scope key)` ordered by `(ts, host, path, line)` via
   `pref_journal::sort_by_writer_order`; foreign-account records never apply (same
   account guard as notes). `append_ack` mirroring `append_note` (journal.rs:585)
   through `pref_journal::append_line` — claim nothing stronger than existing durability
   (no fsync).
2. **Note-history evidence**: in the same pass that folds notes, derive a per-identity
   "has ever had a nonempty text" flag (event- and series-keyed), separate from the
   last-writer-wins fold of current text. Expose `acked(event) -> bool` combining the
   AckBook fold with that history, per the Context definition. No write amplification: a
   note save appends no ack record.
3. **Critical compatibility fix** — `load_state` (journal.rs:278) skips only
   `gcal.series_root` and `gcal.note` by exact name (journal.rs:300–306) and sets
   `newest` (line 307) _before_ parsing, so an unknown kind both counts as malformed and
   feeds the `list` header's "last sync" fallback. Add `gcal.ack` to that exact-name
   skip list: acks must neither fold as occurrences, nor count malformed, nor advance
   `newest`. Test this directly (an ack line leaves `newest` and the malformed count
   unchanged).
4. **Reset/`-F` survival**: acks fold in their own book exactly like notes, so
   `gcal.reset` records and `sync -F` never clear them. Add a test beside the existing
   reset tests.
5. `model.rs`: `is_external(&self) -> bool` implementing the Context rule
   (`!organizer_is_self || event_type == "fromGmail"`).
6. `config.rs` + `bob_config`: new `gcal.ack: external | off` key (default `external`),
   modeled on existing gcal keys. Under `off`, the ack system is inert: `want` is false
   everywhere (phase `tag`) and the queue is empty (phase `cli`). Document the key
   string now; wiring happens in later phases.

**Tests** (unit, in-module like existing journal tests): fold vectors — last-writer-wins
across hosts, `acked: false` clear, account guard, note-history evidence including a
cleared note still acking and a series note covering future instances, series vs event
scope, moved-instance stability (ack keyed by event id survives a time change);
`load_state` skip; reset survival. `just all` green.

## Phase `tag` — unacked row tag reconciliation and sync/list surfaces

The highest-risk work in the epic: an ownership-preserving tag on rows whose other spans
Google and Bryan both edit. The tag must never cause a false `kept_edit`, never survive
on closed rows, and never be re-added over a hand-deletion.

**Row grammar** (`src/native/gcal/lifecycle.rs`): `split_suffix` (lifecycle.rs:159–163)
opens the suffix only at whitespace-preceded `[root::`, `[type::`, `[completion::`, or
the trailing `^id`. On a row with no root/type a trailing `#unacked` would fall into the
hashed badges span and read as a user edit. Add an exact, whitespace-preceded
` #unacked` marker to the same `marker_at` min-scan. Verify `escape_title` already
writes a mid-title `#unacked` as `\#unacked` (it escapes `#`), and add a test proving a
title containing the literal word cannot collide with the marker.

**Render** (`render.rs`): when a planned open row is unacked (per phase `journal`'s
`acked()` + `is_external()` + `gcal.ack` config on), render the tag as the last token
before `^id`.

**Journal**: additive `unacked: true` field on `gcal.row` records (downgrade-safe: older
binaries read row fields with `object.get` — documented in `journal.rs`). This is `last`
below: "Bob wrote the tag last sync".

**Reconcile** (`plan.rs`, beside the existing span reconciliation): three inputs —
`want` (open row ∧ external ∧ ¬acked ∧ config `external`), `present` (exact token on the
live row), `last` (journaled flag):

| Situation                                          | Action                                       | Reported                               |
| -------------------------------------------------- | -------------------------------------------- | -------------------------------------- |
| `last ∧ ¬present ∧ ¬acked` — Bryan deleted the tag | Append `gcal.ack {via: "row", scope: event}` | Once: `acked by removing #unacked`     |
| `want ∧ ¬present`                                  | Add the tag                                  | `⚠ unacked` on the row line            |
| `¬want ∧ present ∧ last` — acked, or row closed    | Remove the tag                               | Row `updated`, detail `acked`/`closed` |
| `¬want ∧ present ∧ ¬last` — Bryan typed it himself | Leave it                                     | —                                      |

Constraints: never tag context rows or rows born done (`[x]` at birth — a past meeting
has nothing left to decide); lift the tag when a row closes; compute `needs_meet_type`
and MEET filing on the line _without_ the tag so type derivation and
`meet-filing-follows-google-until-edited` are unaffected; `sync -F` resets day state
only — acks survive (phase `journal`), so a forced day rebuild re-renders the tag only
when still unacked. Under config `off`, `want` is false everywhere so the next sync
removes Bob-written tags (and only those).

**Surfaces:**

- `sync` human output: `⚠ unacked` marks on affected row lines plus exactly one
  end-of-run summary hint `⚠ N unacked external event(s) · bob gcal ack` — never a
  per-row scream. `sync -q -f json` carries `counts.unacked`.
- Additive `ack` object on row JSON in `sync -f json` and `bob gcal list -f json`:
  `{state: unacked|acked|noted|not_required, scope, via, at}` (plus `ack_stale` in phase
  `cli`). Note/ack text never appears in list/sync JSON (existing redaction rule). This
  one object is the contract later clients (Ledger Tools, Mac Capture) render from — no
  second membership implementation.
- `list` (`list.rs`): a dim `⚠` beside unacked task rows, sibling of the `✎` note glyph,
  plus the same one-line footer hint.

**Tests** (`tests/gcal_sync.rs`, `tests/gcal_lifecycle.rs`, `tests/gcal_list.rs`): the
full want/present/last table including rootless/typeless rows (the `split_suffix`
marker), edited-title and edited-root rows with no false `kept_edit`, born-done,
context, and closed rows, `-F`, tombstoned (`removed_by_you`) rows, replaced Events
sections, config `off` removing Bob-written tags, and JSON shape. Fake gog
(`BOB_GCAL_GOG`) throughout.

## Phase `cli` — the `bob gcal ack` command

This phase is sized `large`: plan the module layout (new `src/native/gcal/ack.rs`,
possibly an `ack/` directory mirroring `note/`) before implementing. Follow
`sase/memory/cli_rules.md` (read it): options alphabetical, every long option
short-aliased, excellent `-h|--help`, colored output where it helps.

**Contract:**

```text
bob gcal ack [-a|--all] [-b|--bob-dir DIR] [-C|--calendar ID] [-c|--clear]
             [-D|--day DAY] [-d|--dry-run] [-f|--format human|json] [-h|--help]
             [-i|--event-id ID] [-l|--list] [-n|--next-n-days N]
             [-q|--quiet] [-s|--scope event|series] [QUERY]
```

| Form                                          | Behavior                                                                                                                                             |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bob gcal ack` (TTY)                          | Queue session: one card per unacked external task-bound event, soonest first, today through day 7                                                    |
| `bob gcal ack -n 14`                          | Two weeks ahead                                                                                                                                      |
| `bob gcal ack -a`                             | Adds awaiting-reply and Maybe-as-context externals so they can be acked early                                                                        |
| `bob gcal ack QUERY` / `-i ID [-C CAL]`       | One event, no queue — same matcher, `eid=` URL decoding, picker, and recurring scope prompt as `note`; `-s` required for a recurring match off a TTY |
| `bob gcal ack -c QUERY`                       | Clears an ack (`acked: false`); note history still acks                                                                                              |
| `bob gcal ack -l` (also bare `ack` off a TTY) | Prints the queue, exit 0, with a hint                                                                                                                |
| `-d` / `-f json`                              | Preview / structured output; never prompt, never write                                                                                               |

- **Window**: today .. today+(N−1) in Bob's local timezone, default `-n 7`, bound
  1..=366, `-D` anchor (reuse `classify::resolve_day`). Events already ended are
  excluded. `-n` here points _forward_ while sync's `-n|--last-n-days` points backward:
  both help strings must state the direction in their first words.
- **Queue membership** — one inclusion function (sync's `classify`), never a second:
  eligible and external, lands as task (`Inclusion::Task`: Yes, or Maybe under
  `gcal.tentative: task`), not acked, ends after now, starts within the window, and the
  day's journal has not tombstoned its row as `removed_by_you` (deleting the row already
  answered it). Fetch via the existing batched `FetchContext::fetch_days` (one range per
  calendar, all pages drained). Future days need no daily note; never pre-create one.
  Unanswered invites are a read-only footer count
  (`N awaiting your reply in Google Calendar`), included only under `-a`.
- **Terminal session**: single keypress, no Enter; libc termios raw mode (libc is
  already a dependency — do not add crossterm). Restore the terminal on drop and on
  panic; cooked mode for the note line; `BOB_GCAL_ASSUME_TTY` (config.rs:19) as the test
  hook; a non-TTY interactive attempt refuses with exit 2 and a hint toward `-l` /
  `-f json` / `-i ID`. Bare Return is inert; held/repeated keys must not sweep the queue
  (a short debounce or draining read).

**Keys:**

| Key                | Action                                                                                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `a`                | Ack. Recurring: This event / All events prompt, **default All events**, defaults printed                                                                 |
| `n`                | Note: one line of `bob gcal note` TEXT (default scope This event); empty line returns to the card; validation errors keep the draft and offer the editor |
| `e`                | Note in `$VISUAL`/`$EDITOR`, reusing `note -e` scissors + cheat sheet                                                                                    |
| `s`                | Skip this run; it returns next time                                                                                                                      |
| `q` / Esc / Ctrl-C | Quit; answers so far are saved; remaining count printed                                                                                                  |
| `?`                | All keys, one line each (includes `o` open Calendar link; `A` adopt when the fragile-directives hint applies)                                            |

Each card shows what Bob will write — the rendered row (with the warning tag), root and
its source, buffer, clock, and time rail — reusing the note card's existing preview
components (`ui.rs` hue dots, type swatches; the note preview renderer). Fits 80
columns; `NO_COLOR` drops color but keeps glyphs. Card sketch (from the research — match
its spirit, refine freely):

```text
─ 1 of 2 ─────────────────────────────────────────────────────────
Screening between Thomas Holvoet and Bryan
  Tue Nov 17 · 12:00–12:15 · in 5 days · from Thomas · ✓ Yes

  ● meet ━━━━━━━━━━━━━━━━━━━━━━━━━ 12:00 ─ 12:15
  meet (default) · no buffer · on-clock · ≡ 2 bullets from Google

  lands as
  - [*] #task **1200-1215** Screening between Thomas Holvoet and Bryan 📅 📹 [root:: meet] #unacked

  a ack   n note   s skip   q quit   ? more
```

- **The note branch reuses the note write path byte-for-byte**: same TEXT grammar
  (`@route`, `#off`/`#on`, `-30 reason`, `+15 reason`, `--`, prose), validation,
  preview, editor, adopt (`-A` semantics), and This event / All events scope prompt. No
  second parser (it would drift within a release).
- **Per-answer persistence**: each answer appends to the journal the moment it is given;
  quitting or crashing never loses prior answers. After a series answer, recompute the
  queue so covered occurrences disappear, reported as series-covered. End-of-session
  summary counts disjoint actions (`✓ 1 acked · 1 noted · 1 skipped`) plus the next step
  when anything remains.
- **Writer discipline**: any host may append acks (like notes); only the writer host
  applies row changes on its next sync. On a non-writer host the command saves and says
  where the row change will land. apollo has no gcal client (`no_client`); the
  interactive walk runs where OAuth lives.
- **Stale notice** (no auto-restack in v1): every ack records `seen`; `list`, `ack -l`,
  and `doctor` print `ack stale · moved since ack?` when live times disagree with `seen`
  on an event-scope ack; JSON sets `ack_stale: true`; re-acking refreshes `seen`. Docs
  copy: "Ack records that you reviewed this event; it does not approve later changes."
- **Exit codes**: 0 — completed session, empty queue, list, or voluntary quit (JSON
  carries `complete: false` when skips remain); 1 — gog/network/persistence failure; 2 —
  usage, setup, non-TTY interactive attempt, ambiguous `-i`, or recurring `-i` without
  `-s` off a TTY; 130 — Ctrl-C after saving and restoring the terminal.
- **JSON** (`schema_version: 1`): window, queue rows (event identity, organizer,
  `external`, `lands`, root + source, buffer, `ack: {state, scope, via, at}`,
  `ack_stale`), the `waiting` lane, and counts. Ack/note text never appears.

**Required integration fixes:**

1. `note.rs:78` — `find_note_subcommand`'s sibling list
   `["calendars", "doctor", "list", "login", "logout", "sync"]` must gain `"ack"`, or
   `bob gcal ack @family` can parse as a `note` tail (verified footgun).
2. `cli.rs::build_cli` — add the `ack` subcommand (sorts first alphabetically among
   `bob gcal` subcommands), after-help with examples and the environment block
   (`BOB_GCAL_GOG`, `BOB_GCAL_ASSUME_TTY`, `BOB_NOW`, `VISUAL`/`EDITOR`).
3. `note` success on a previously-unacked event appends `· acked` to its confirmation.
4. `doctor` gains an acks section: fold counts, malformed, foreign-account, orphans
   (acks on gone/cancelled events — reported, never silently pruned), stale-`seen`
   warnings.
5. Shell completion for the new subcommand/flags wherever gcal completions are defined;
   run `just docs-gen` so generated help/command-catalog pages stay current, and
   `just docs-check`.

**Tests** (`tests/gcal_cli.rs` + a new `tests/gcal_ack.rs`, fake gog via `BOB_GCAL_GOG`,
no real Google): key-driven sessions via `BOB_GCAL_ASSUME_TTY` with piped bytes — ack,
note (valid and invalid TEXT), skip, series scope prompt both answers,
quit-saves-answers, Ctrl-C restores the terminal; non-TTY interactive attempt exits 2;
`-l`/bare-off-TTY list; one-shot `-i`/QUERY incl. ambiguous and recurring without `-s`;
`-c` clear vs note-history; `-d` writes nothing; JSON shape and redaction; `-a` waiting
lane; window bounds; config `off` empties the queue.

## Phase `docs` — docs, decision record, and glossary

Read `sase/memory/docs.md` conventions first (already summarized here): references own
exact behavior and CLI facts; run the gates.

1. **New `docs/gcal-acks.md`**, modeled on `docs/gcal-notes.md`'s structure (Contents /
   Model / Command / Keys / Storage / Precedence-with-notes / Reliability / JSON / Known
   limits). Lead with the one-sentence rule (also in `--help`): "Bob asks once about
   every event you can't describe in Google: press `a` when Bob's handling is right, or
   `n` to write a private note. Until you answer, that Event Task carries an `#unacked`
   tag." Cover: what counts as external, what counts as acked (note history rule and its
   one deliberate consequence — a series that ever had a valid series note stays acked
   after the note is cleared, including future instances), the stale notice, scope
   semantics, `-c`, exit codes, JSON, and the writer/host model.
2. **`docs/gcal.md`**: Commands section row for `ack`; the "Who owns each part of a row"
   table (docs/gcal.md:711) gains the warning tag (Bob-owned exact token; deleting it
   acks); Sync section: the `⚠ unacked` marks and summary hint; JSON output section: the
   additive `ack` object and `counts.unacked`; Known limits: older binaries print a
   per-line warning on `gcal.ack` journal lines _and_ misread the `list` header's
   last-sync freshness (`newest`), so the writer Mac upgrades first, reader and writer
   in the same release.
3. **`docs/gcal-notes.md`**: cross-link; state that a saved nonempty note acks the event
   and that clearing a note never un-acks.
4. `just docs-gen` and `just docs-check` green; mkdocs nav updated if the new page needs
   an entry (follow how gcal-notes.md is wired).
5. **Memory** (through `/sase_memory_write`, covered by this plan's memory decisions):

> [!decision] mem_decision_record File `decisions:external-gcal-events-are-acked-once`:
> every external event bound for an Event Task is asked about once; an ack or a private
> note — including any historical nonempty note — answers it; until then its open row
> carries the Bob-owned warning tag, and deleting the tag acks; rows are never held
> back. Rejected: a hard gate, title placement, a tag between `#task` and the time span,
> a child-bullet warning as primary, empty-note-as-ack, auto-restack in v1,
> `bob gmail ack`. Reopens when: external volume exceeds ~10/week (organizer/domain
> trust), a shared or read-only calendar is configured (`organizer.self` is a
> per-calendar-copy fact — the rule then needs an access-role check), Bob gains RSVP
> writes, or a missed reschedule shows the stale notice isn't enough. Evidence: the
> research artifact, this plan, the landed commits, and `docs/gcal-acks.md`. Link
> related records (`bob-gcal-is-read-only-and-proposes-once`,
> `private-event-notes-beat-google-per-field`,
> `calendar-events-are-daily-note-event-tasks`).

> [!decision] mem_glossary_term Add `glossary:event-ack` (aka ack, unacked, event ack):
> the recorded once-per-event review of an external Google Calendar event — an explicit
> `bob gcal ack` receipt or any historical nonempty private event note — which sync
> reflects as the presence or absence of the warning tag on the open Event Task row.
> Link `glossary:event-task` and the new decision record.

Run `sase memory init` after the memory edits.

## Phase `vault` — weekly reminder row at rollout

Install the reminder only once the command exists in a shipped release (a reminder for a
nonexistent command is premature). bob-cli ships through the daily release train, so:

1. **Gate**: confirm a published release contains `bob gcal ack` (check the GitHub
   release/CHANGELOG for the first version containing the `cli` phase's commits, or run
   `bob gcal ack -h` under an upgraded binary). If the train has not shipped yet, wait
   on it with `/sase_monitor` rather than ending the turn.
2. **Edit `~/bob/gtd_daily.md`** (the live vault; `bob vault-sync` carries it — no git
   work in this repo): add one row under the existing weekly Monday rows (beside "Bank
   import", gtd_daily.md:22), with `[created::]` = the write date and `[scheduled::]` =
   the next Monday:

```markdown
- [?] #task #gtd #pre #terminal **Ack external events**: run `bob gcal ack -n 7` · a ack
  · n note · s skip · empty queue? completing this is the record [repeat:: every week on
  Monday when done] [created:: <write date>] [scheduled:: <first Monday>]
```

Keep the existing daily `#browser` "Review next week in Google Calendar" row untouched
(it is the RSVP beat; Bob cannot RSVP). The mid-week safety net is the row tag plus
sync's summary hint, not a daily chore. `ack` must never auto-complete this checklist
row. 3. **Rollout sanity check** (non-blocking): confirm on the Mac that the Swings
series classifies as external (`bob gcal list -f json` → `organizer_is_self == false`)
before relying on series-ack examples in docs copy; if the Mac is unreachable, record
the check as a note for Bryan rather than blocking.

## Testing and verification strategy

- Every code phase runs `just all` (fmt, lint, test) and keeps `just docs-check` green
  when help text changes (`just docs-gen` regenerates).
- All gcal tests run against fake gog (`BOB_GCAL_GOG`); nothing touches real Google.
- The riskiest surface is phase `tag`'s ownership-preserving reconciliation, not the
  journal append — weight review and tests there (no false `kept_edit`, no tag on
  born-done/context/closed rows, hand-deletion becomes an inferred ack exactly once).
- Interactive-session tests drive raw-mode keys through `BOB_GCAL_ASSUME_TTY` with piped
  bytes, including terminal restoration on Ctrl-C.

## Follow-up (not part of this epic)

Feature task bead `bob-cli-l4` tracks the v2 display surfaces (Ledger Tools pill, MEET
link-time dot in bob-plugins, and the Mac Capture Ack action over
`bob gcal ack -i … -f json`), to be planned after the CLI ships and its JSON contract is
real.
