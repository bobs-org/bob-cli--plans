---
tier: epic
title: Finish private event notes — correctness fixes and the missing extras
goal: 'Close the gaps the bob-cli-ji land audit found in `bob gcal note`: previews
  that show a stale note, options that are silently swallowed as note text, orphan
  clear commands that fail, an account-blind note fold, held rows reported as updates,
  and an incomplete extras phase (`-e`, `-A`, the doctor `notes` check, and the fragile-description
  hint). Every fix ships with tests, and the JSON stays compatible with Bob Mac Capture.'
phases:
- id: engine
  title: Account-aware note fold, honest held rows, and accurate note provenance
  depends_on: []
  size: medium
  description: 'engine: fold private notes per account so a foreign note never hides
    this account''s note, report held rows as held (not updated, no duplicate gcal.row
    per run), make the note `fields` summary list only fields the note actually won,
    strip aliased title routes when a note route wins, and add the missing no-hold
    test.'
- id: select
  title: Exact selection, argv pre-pass, orphans, and list/show fixes
  depends_on:
  - engine
  size: medium
  description: 'select: build `-i`/URL previews from the pre-note Google event, honor
    this command''s options anywhere before `--`, make the printed orphan clear commands
    work, keep lookup failures from becoming orphans, check cancelled/declined/gone
    before past-hiding, key the list JSON by mode, guard show by account, print the
    URL fallback warning, and enrich the picker rows.'
- id: write
  title: Write contract, validation warnings, human output, and missing tests
  depends_on:
  - select
  size: medium
  description: 'write: fill `errors[].suggestions`, emit `clock` and `invalid` spans,
    add the missing directive-lookalike and reason option-like warnings, label the
    rail with the root, colour canonical lines and dim the lands-as block on a TTY,
    finish the docs, add the missing normalization vectors and tests, and regenerate
    the golden fixtures.'
- id: extras
  title: Finish -e, -A, the doctor notes check, and the fragile-description hint
  depends_on:
  - write
  size: medium
  description: 'extras: bring `-e` (TTY, vi fallback, git scissors, Edit again loop,
    empty clears), `-A` (bare buffers, existing-note refusal, success wording), the
    doctor `notes` check (shared fold, cancelled/declined/series orphans, 25-lookup
    cap, malformed records, per-issue fixes), and the `bob gcal list` fragile hint
    (conditions, wording, `fragile_directives` JSON) up to the approved plan, with
    tests and docs.'
proposed_by: bbugyi200.apollo.bob-cli-ji.land
proposed_with: claude/opus
parent_bead: bob-cli-ji
create_time: 2026-11-10 22:31:44
status: wip
bead_id: bob-cli-ji.8
---

- **PROMPT:** [prompts/202611/finish_private_gcal_event_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/finish_private_gcal_event_notes.md)
- **PARENT:** [202611/private_gcal_event_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/private_gcal_event_notes.md)
- **BEAD:** [bob-cli-ji.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ji/bob-cli-ji.8.md)

# Plan: Finish private event notes — correctness fixes and the missing extras

## Context

Epic `bob-cli-ji` (plan `202611/private_gcal_event_notes.md`, the **parent plan**) added
private event notes: `gcal.note` journal records layered over Google descriptions, the
`bob gcal note` command, and the Bob Mac Capture Event note mode. All seven phases are
closed and `just check` passes on master. But the land agent's audit of the source
against the parent plan found real bugs and an incomplete extras phase. This plan
finishes that work. It is bob-cli only.

Every phase agent must first read the parent plan's **Shared design** section and the
section of the phase it is finishing (engine, select, write, extras; the parent plan's
`hardening` section matters for the **engine** phase here). Read it with
`sase artifact read plan:202611/private_gcal_event_notes.md "<reason>"`. The parent
epic's reviewer decisions still hold: `auto_sync = yes` (the writer host syncs
immediately after a save or clear), and Bob never writes to Google.

**Ground rules for every phase:**

- Repo: bob-cli. Main code is in `src/native/gcal/` (`note.rs`, `note/write.rs`,
  `journal.rs`, `snapshot.rs`, `model.rs`, `root.rs`, `plan.rs`, `sync.rs`, `list.rs`,
  `doctor.rs`, `cli.rs`). Tests are `tests/gcal_notes.rs` (engine),
  `tests/gcal_note_cli.rs` (command), and `tests/gcal_auth.rs` (doctor). They use the
  fake gog (`tests/gcal_support/`); tests never touch Google.
- **Mac compatibility.** Bob Mac Capture decodes `bob gcal note -f json`. Every JSON
  change here must stay additive or match what the Mac already expects. The Mac already
  maps span kind `clock`, reads `events` from `-l -a`, and decodes
  `errors[].suggestions`. Do not edit the `bob-mac-capture` repo in this epic.
- **Golden fixtures.** `tests/fixtures/gcal_note/json/*.json` are asserted by
  `tests/gcal_note_cli.rs`. Regenerate them with
  `BOB_UPDATE_GCAL_NOTE_FIXTURES=1 cargo test --test gcal_note_cli`, then review every
  diff by hand: each change must be explained by your phase's fixes. Name the changed
  fixtures in your phase's closing note. The Mac keeps copies in its
  `Tests/Fixtures/gcal-note/`. The final phase records a `PROPOSED FOLLOW-UP:` to
  refresh those copies.
- **Gates.** `just check` must pass. Run `just docs-gen` when help text changes.
  `just docs-check` must show no new failures; its known clean-base word-budget failures
  on the mounted Mac pages are tracked by `bob-cli-jz` and `bob-cli-h9`. Read the
  `cli_rules` and `docs` reference memory before changing CLI options, help, or docs.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`. Use `fix(gcal): …` commit
  subjects, or `feat(gcal): …` for the extras, never `!`. Use no `--epic-symbol`
  whitelist entries.
- No memory note changes. `decisions:private-event-notes-beat-google-per-field` stays
  true as written.

## Phase engine: account-aware fold, honest held rows, accurate provenance

1. **Fold per account.** `journal::load_notes` (`journal.rs`, about lines 509–559) runs
   last-writer-wins on `(calendar_id, event_id | series)` with no account. The account
   check only happens later, in `snapshot.rs::matching_note` (about line 199). So a
   newer note that account B saves on the same key (a shared invite on `primary` has the
   same event id for both accounts) wins the fold and is then discarded, and account A's
   note silently stops applying.
   - Make the fold key include the account, or keep each key's latest record per
     account.
   - Give `NoteBook` lookups that take the account. Update every caller: snapshot,
     `note.rs` candidate building and `lookup_exact`, show/`current_scope_text`, the
     hold check in `plan.rs` (about lines 1200–1234), and the write path's "unchanged"
     and clear comparisons.
   - Keep foreign-account notes listable: list orphans/warnings and the doctor still
     surface them.
   - A clear saved under account A clears only A's note.
2. **Held rows are held, not updated.** `plan.rs::hold_journaled_row` (about lines
   1236–1266) pushes the held row into `plan.rows_updated`. While a hold lasts, every
   run therefore:
   - marks the sync written;
   - prints the `~` updated sign;
   - reports Google's current buffer instead of the row's journaled one;
   - appends an identical `gcal.row` journal line (`sync.rs`, about lines 1552–1572).

   Report held rows separately (for example `rows_held`, or a held flag the report
   honours):
   - human output shows the hold warning and detail, not an update sign;
   - JSON keeps the row's `detail` and the `warnings` entry;
   - the buffer reported is the journaled one;
   - a run whose only "changes" are holds is not reported as written;
   - no new `gcal.row` line is appended while the journaled record is unchanged.

   The parent plan's hardening rules are unchanged: skip span, root, and description
   updates; never hold a deliberate clear.

3. **Provenance lists only what the note won.**
   - `model.rs::apply_notes` (about lines 319–326) adds `root` to `AppliedNote.fields`
     whenever the note has a route, even when root selection (`root.rs`, about lines
     122–128) rejects that route and falls through. It also adds `bullets` under
     `gcal.description: off`.
   - After root selection, drop `root` unless the chosen root's kind is
     `note`/`series_note`. Drop `bullets` when no note bullet is rendered.
   - The list JSON `note.fields`, sync JSON, the `gcal.row` provenance, and the note
     hash must agree with the corrected set.
4. **Aliased title routes.** When a note route wins, `root.rs` (about lines 336–341)
   strips the title's `@token` only when the raw token equals the note route. The
   title-route path (about lines 166–171) compares the resolved name instead. Strip a
   resolvable title route token whenever a note route wins, as the parent plan says. For
   example, `Breakfast @fam`, where `fam` resolves to `family`, displays `Breakfast`.
5. **Tests** (`tests/gcal_notes.rs` plus unit tests where they fit):
   - two accounts' notes on one key: the newer foreign note doesn't hide this account's
     note, and a clear is per account;
   - a held row across two runs: no `~`, not written, the journaled buffer reported, and
     no second `gcal.row` line;
   - `fields` omits `root` when the note route is invalid, and omits `bullets` under
     `description: off`;
   - aliased title-token stripping;
   - the hardening test the parent plan required but nobody wrote: **no hold when
     nothing is malformed**. A journaled row with note provenance has its note removed
     (no live note, no deliberate clear) while the malformed count is 0, so Google's
     values apply and no hold warning appears.

## Phase select: exact selection, pre-pass, orphans, list/show

1. **Previews from the pre-note event.**
   - In `note.rs::lookup_exact` (about line 740), `google_event` is cloned **after**
     `apply_notes`. The title-search path clones it before (about line 506).
     `write.rs::rebuild_candidate` starts from that clone, and `apply_notes` returns
     early when there is no note. So after `-c`, or a rewrite that drops a side or the
     clock tag, `-i`/URL previews still show the old buffer, reasons, and `#off`/`#on`.
     `tests/fixtures/gcal_note/json/cleared.json` currently records exactly that: event
     `e1` has no description but still shows `[buffer:: -30m +15m]` and reasons.
   - Clone before applying notes.
   - Add a test that writes a note via `-i`, clears it via `-c -i`, and asserts the
     cleared preview shows Google's values (and equals what sync then writes).
   - Regenerate the fixtures this changes.
2. **Argv pre-pass** (`note.rs::rewrite_note_tail`, about lines 92–160).
   - The parent plan says this command's options are honored **anywhere before the first
     `--`**. Today the first directive token switches everything after it to TEXT, so
     `bob gcal note deerwood @family -30 Drive there -d` saves the reason
     `Drive there -d` instead of previewing. Keep scanning after TEXT starts: pull this
     command's known options (and their values) out of the TEXT run, and keep the
     inserted `--` before the TEXT words. After an explicit `--`, everything stays
     verbatim.
   - Short options with an attached value (`-iabc`, `-Dtomorrow`, `-fjson`) must not
     consume the next token.
   - A token matching `^[-−–+]\d` that is not a buffer token (for example `-30min`) is
     never an option. Treat it as a TEXT word that starts TEXT, protected by the `--`,
     so it never reaches clap as an unknown flag. The **write** phase warns about it.
   - Add pre-pass unit vectors for each case.
3. **Orphan clear commands must work.** List prints `bob gcal note -c -i <id>` for a
   not-found orphan (about line 2151) and `-c -s series -i <series>` for a series orphan
   (about line 2153). Both fail today:
   - the first with "no event with id", because `lookup_exact` finds nothing;
   - the second with "`-s series` needs a recurring event" (`write.rs`, about line 326),
     because the master has no `recurringEventId`.

   When `-c` names a key that has a live note for this account but no eligible event can
   be loaded (not found, cancelled, declined, or a gone or bare series master), append
   the clear record by key: for `-s series`, `-i` names the series id. Report
   `✓ Cleared`. Integration tests must run the exact command text that list prints.

4. **Lookup failures are not orphans.** Every gog error in the orphan lookups (about
   lines 1139 and 1159) is recorded as `not_found`, so a network or auth outage shows
   every out-of-window note as "no longer on your calendar". Only a real not-found (as
   `lookup_exact` already distinguishes, about lines 697–709) makes an orphan. Report
   other errors as one warning ("could not check N notes: …"), leave those notes out of
   the orphan list, and exit 0. Test with a fake gog that errors on one lookup.
5. **Gone before past.** The past-event check (about line 1178) runs before the
   cancelled/declined check (about line 1191). For a series note, the master's start is
   the first occurrence, so a deleted or declined series that began in the past is
   hidden instead of shown as an orphan. Check gone, cancelled, and declined first.
   Judge a series by its master's status, never by the master's start. Test both orders.
6. **List JSON key by mode.** `print_list_json` (about lines 1995–2037) chooses `events`
   only when some row has no note. Emit `events` for `-a` (every upcoming event; the Mac
   picker reads `events`) and `notes` otherwise, whatever the rows contain. Update
   `docs/gcal-notes.md` (it currently documents the heuristic) and add tests for an
   empty window and an all-noted window under `-l -a`.
7. **Show guards the account.** Show mode prints a note saved under a different account
   (about lines 1828–1846). Use the account-aware lookup from **engine**:
   - show displays only this account's note;
   - a foreign note appears as a warning ("a note saved under another account exists; it
     applies only when that account is signed in"), never its text.
8. **URL fallback warning.** When no configured calendar holds a pasted URL's `eid`, the
   "falling back to a title search" warning (about line 1323) is dropped before
   `show_from_query` returns its no-match error (about line 1368). Print the warning
   with the error, in both human output and JSON `warnings`. Tighten the test at
   `tests/gcal_note_cli.rs` (about line 980) so it requires the warning.
9. **Picker and scope prompt.** The parent plan's TTY picker shows day, time, title,
   "from <organizer>", the RSVP chip, the ↻ group, and the `✎` summary. Today it prints
   only `· ↻`/`· ✎` (about lines 1472–1490).
   - Show `↻ next <Day Mon D>` for a grouped series, and the note summary
     (`✎ this event|all events ● root −30m +15m`).
   - The scope prompt shows
     `<title> repeats · <weekday>s <time> · <organizer role> · <answer>`. Add a cadence
     word (weekly, daily, …) only when it is available without an extra Google request.
   - Assert the picker contents in the TTY picker test.

## Phase write: write contract, warnings, human output, tests

1. **`errors[].suggestions`.**
   - `write.rs::issue()` (about lines 1122–1135) hard-codes an empty list. For
     `route_unknown`, put the closest routes (the names the message already lists) in
     `suggestions`.
   - For `route_invalid` (a `validate_root_with` failure), also compute closest valid
     routes, and mention them in the message.
   - The Mac already decodes `suggestions`.
2. **Span kinds.**
   - Emit `clock` for `#off`/`#on` instead of `clock_off`/`clock_on` (about lines
     851–859). The parent plan, `docs/gcal-notes.md`, and the Mac all expect `clock`;
     the Mac renders today's kinds as unknown.
   - Emit `invalid` spans over tokens that produced an error: unknown or invalid route,
     the second token of a conflict, out-of-range minutes, a reason with 0.
   - Keep the nested `buffer` + `reason_sign` pair. The Mac overlays it. Document the
     nesting in `docs/gcal-notes.md` § JSON.
3. **Missing warnings.**
   - The parent plan's warning "a prose line whose first token looks like a directive"
     has no implementation. Warn when a prose line starts with a `^[-−–+]\d` token that
     isn't a buffer token (`+30min Drive`, `-30x`), or with a bare `@`. Suggest the
     canonical spelling.
   - The `option_like` warning (about lines 749–753 and 984–988) checks only prose. Run
     it inside reasons too: the reason branch (about lines 1001–1066) never checks it,
     so text after an explicit `--` like `-- -30 Drive -d` warns.
4. **Human output** (TTY colour, plain off a TTY).
   - Colour the canonical lines with the span kinds (route in its root hue, buffer
     tokens, reason text, prose); today they print uncoloured (about line 1294).
   - Dim the "lands as" block (about line 1304).
   - Label the rail with the root value (`● family`), not the colour name (about line
     1542; fixtures show `● orange`). The JSON `preview.rail` follows.
   - Fix the duplicated `-d` message "Preview · nothing saved · nothing was saved".
5. **Docs.**
   - `docs/gcal-notes.md` § JSON: the span kinds and nesting, every `status.state` value
     the code emits (including `preview`, `unchanged`, `context`), and `suggestions`.
   - `docs/gcal.md` § Commands: the `note` line lists every option (`-A`, `-a`, `-b`,
     `-C`, `-c`, `-D`, `-d`, `-e`, `-f`, `-h`, `-i`, `-l`, `-s`).
6. **Normalization vectors** (`tests/fixtures/gcal_note/normalize.json`): `@`/`#` inside
   a reason stay text; a reason ended by `;`; the directive-lookalike prose warning;
   `-- -30 Drive -d` with its warning.
7. **Missing tests** (`tests/gcal_note_cli.rs`):
   - write JSON `status.state` `waiting_reply` and `waiting_maybe` (an awaiting-reply
     event and a Maybe event under the default `gcal.tentative`);
   - the `route_invalid` and empty-note errors;
   - human write output: `✓ Noted …`, `✓ Cleared · Google's description decides again`,
     and the `lands as` block, asserted off a TTY;
   - `suggestions` non-empty for an unknown route;
   - `clock` and `invalid` spans.
8. Regenerate and review the golden fixtures.

## Phase extras: finish `-e`, `-A`, doctor `notes`, and the fragile hint

The parent plan's **extras** section is the spec. The shipped code (`note.rs`
`edit_text`, `adopted_google_text`, and `present_candidate`; `doctor.rs::notes_check`;
`list.rs::print_table`) deviates as follows. The phase shipped no tests.

1. **`-e|--edit`.**
   - Require a TTY (honour `BOB_GCAL_ASSUME_TTY`). Off a TTY, exit 2 with a hint to pass
     TEXT instead.
   - Pick the editor as `$VISUAL`, then `$EDITOR`, then `vi`.
   - Replace the `# --- Bob event note help …` marker and `#` comment lines with the
     parent plan's git-style scissors block: the
     `------------------------ >8 ------------------------` line, "Do not modify or
     remove the line above; everything below it is ignored.", the event identity line
     (`Private note for <Day Mon D HH:MM–HH:MM> <title> (from <organizer> · <answer>)`),
     the grammar cheat sheet, and "An empty note clears it." Everything from the
     scissors line down is dropped.
   - On validation errors, print them and ask `Edit again? [Y/n]`. Yes reopens the
     editor with the user's text; `n` exits 2 and saves nothing.
   - Empty text clears the note in the chosen scope. Report `unchanged`, appending
     nothing, when there was no note. Unchanged text reports `unchanged`.
2. **`-A|--adopt`.**
   - Adopt from the event's Google (pre-note) description. That includes bare buffer
     directives, which are dropped today: `-30`/`+15` with no reason, and an explicit
     `-0`/`+0`. Also adopt the route, the clock tag, and the reason lines, but not
     prose.
   - When the chosen scope already has a note for this account, exit 2 with "this event
     already has a private note · edit it with `-e`". `-A -e` instead opens the editor
     prefilled with the adopted lines.
   - When there is nothing to adopt, exit 2 with a clear message.
   - On success, say the lines "now live in Bob too: they survive the organizer's next
     update".
3. **Doctor `notes`** (`doctor.rs`, about lines 250–410).
   - Use `journal::load_notes` and the engine phase's account-aware lookups instead of
     the hand-rolled fold.
   - OK reads `N private notes`.
   - Warn on:
     - orphans: an event not found, cancelled, or declined, or a series whose master is
       gone, cancelled, or declined (series notes are not checked today);
     - routes that no longer resolve against today's note path;
     - foreign-account notes;
     - malformed `gcal.note` records.
   - Cap lookups at 25 and add "N not checked" beyond that. Lookup errors other than
     not-found count as not checked, never as orphans.
   - Give each issue kind a one-line fix that names a concrete command for its first
     offender (`bob gcal note -c -i <id> -C <cal>`, `bob gcal note -e -i <id>`, or the
     account sign-in hint).
   - Update `docs/gcal.md` § Doctor.
   - Test ok, orphan, cancelled, series-gone, unresolved, foreign, malformed, and the
     cap in `tests/gcal_auth.rs` or a new doctor test. Keep the
     `doctor_json_healthy_setup` expectation passing.
4. **Fragile-description hint** (`list.rs`, about lines 741–757).
   - Show it only when all of these hold:
     - the event lands as a task;
     - you don't organize it and it has guests (`!organizer_or_no_guests`);
     - its Google description carries Bob directives or reason lines;
     - it has no private note (`event.note.is_none()`).
   - Use the parent plan's wording:
     `⚑ <title>: its Bob lines live in the organizer's description and may vanish on their next update · bob gcal note -A -i <id> -C <cal>`.
   - Add `fragile_directives: <bool>` to each list JSON event (true only under those
     conditions).
   - Update `docs/gcal.md` § List/§ JSON and `docs/gcal-notes.md`.
   - Test the positive case and each negative condition.
5. **Docs and help.** Bring `docs/gcal-notes.md` (§ Command, editor, adopt) and the
   `bob gcal note`/`bob gcal list` help in line, then run `just docs-gen`.
6. **Closing note.** List every golden fixture that changed across this epic (use
   `git log -p -- tests/fixtures/gcal_note/json` since the epic's first commit). Record
   `PROPOSED FOLLOW-UP: refresh Bob Mac Capture Tests/Fixtures/gcal-note copies from bob-cli tests/fixtures/gcal_note/json (changed: …) and rerun its gcal-note tests`
   on this phase's bead.
