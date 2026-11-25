---
tier: tale
title: Finish and close the meeting-notes epic (bob-cli-pw)
goal:
  Meeting recaps follow the approved layout and never drop a word. Meeting PDFs render
  on every TeX Live. Obsidian's ref-meet toast offers Open event. `bob ref meet` help
  reads as user documentation. Epic bob-cli-pw is closed with a verification note and
  its plan file is marked done.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-pw.land
proposed_with: claude/opus
bead: bob-cli-pw
status: done
---

- **PARENT:**
  [202611/meeting_notes_event_refs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/meeting_notes_event_refs.md)
- **BEAD:**
  [bob-cli-pw](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pw/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-pw.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-pw.land.md)
- **COMMITS:**
  - [a458c96](https://github.com/bobs-org/bob-plugins/commit/a458c96f448b6c298709685951c438c6015f04b5)
    — fix(command-mode): accept wikilink relation targets for ref-meet toast

# Plan: Finish and close the meeting-notes epic (bob-cli-pw)

## Context

Epic `bob-cli-pw` ("Meeting notes become event-linked references") built `bob ref meet`
across bob-cli, bob-plugins, and Bob Mac Capture. Its approved design is
`sase/repos/plans/202611/meeting_notes_event_refs.md`; the next section quotes the parts
you need. All eight phases are closed. The land agent audited the code and found five
epic-caused gaps, listed below. This tale fixes them and then closes the epic. No other
agent resumes this landing, so the closeout in step 7 is part of this work.

The land agent has already triaged follow-ups and recorded them on the epic. Do not file
any task beads for the items below; they are epic work.

Hard rules inherited from the epic:

- bob-cli, bob-plugins, and bob-mac-capture are **public** repositories. Fixtures,
  tests, docs, and commit messages use invented people, companies, and products only.
  Never copy anything from the private vault (`~/bob`) or from `~/tmp/*meet_notes*`.
- Do not edit any SASE memory note (the epic's reviewer decisions forbid it).
- Keep the reviewer decisions `recap_scope = full` (the recap holds the TL;DR, next
  steps, and every section) and `scaffolding = rehang` (already done; nothing to do).
- File-change verification is `just check`. Do **not** run `just check-full`.
- Before editing CLI help or docs, read the `cli_rules.md` and `docs.md` reference
  memories with `/sase_memory_read`.

## Design of record for the recap (from the epic plan)

The approved recap region is:

```markdown
<!-- bob:meeting-recap:begin v1 sha256:<16 hex> -->

## Notes

_Mon 2026-11-23 · 15:00–15:30 ·
[[2026/20261123#^cedar-review|Cedar Labs quarterly review]] · VoiceQuill_

> [!summary] TL;DR <the lede, verbatim>

### Next steps

- **<Owner>** — <action, verbatim>

### <Section heading, verbatim>

- <paragraph, verbatim>
<!-- bob:meeting-recap:end -->
```

Today `render_recap` (`src/native/meetings/recap.rs`) has five departures from this:

- It emits no `## Notes` heading.
- The meta line has no weekday and no event-title alias. Birth passes the bare `event`
  link, for example `[[20261124#^cedar-review]]`.
- The TL;DR is a plain paragraph instead of a callout.
- Section and next-steps headings copy the notes' own levels (clamped to 2–3). An H2
  section therefore becomes a sibling of the note's `## Highlights`.
- Headings and blocks are not separated by blank lines.

The current goldens show this shape: `tests/fixtures/meetings/golden/recap_wispr.md` and
`recap_adversarial.md`.

## Steps

### 1. Recap layout (bob-cli)

In `src/native/meetings/recap.rs` and its callers, make `render_recap` produce the
design-of-record layout:

- **Inner text.** It starts with `## Notes`, then a blank line, then the meta line.
  Separate every later block with one blank line.
- **Meta line.**
  - Format: `_<Ddd YYYY-MM-DD> · <span> · <event link> · <producer>_`.
  - Reuse `format_meeting_day` from `src/native/meetings/profile.rs` for the weekday.
    Keep the raw date when it does not parse.
  - The event link becomes `[[<target>|<alias>]]` when an event title is known, and
    stays the bare link otherwise.
- **Event title for the alias.** Add `event_title: Option<String>` to `RecapMeta`
  (`src/native/meetings/mod.rs`). Fill it at every construction site:
  - birth from the dossier (`src/native/highlights_ref/meeting_birth.rs`, from the
    dossier's `event_title`)
  - the overwrite paths (`src/native/highlights_ref/create/meet_route.rs`,
    `src/native/highlights_ref/overwrite/execute.rs`)
  - the store-missing empty recap: pass `None` there, since no title is available
- **Alias text.** The title comes from Google, so make it alias-safe and inert.
  - Prefer the helper that already sanitizes reading-task wikilink aliases (the
    `[[ref/meetings/<stem>|<title>]]` task link) over a new one.
  - The alias must contain no `|`, `[`, `]`, `#`, `^`, newline, or `<!--`.
  - Omit the alias when sanitizing leaves it empty.
- **Own title.** When notes carry their own H1 title, render it as a bold line
  `**<escaped title>**` after the meta line.
- **TL;DR.** Render an Obsidian callout: the line `> [!summary] TL;DR`, then every TL;DR
  line escaped and prefixed with `> `. Blank lines inside a multi-paragraph TL;DR become
  `>`.
- **Headings.**
  - Next steps: always `### <heading>`.
  - Sections: notes level 2 becomes `###`, and level 3 becomes `####`.
  - Unheaded sections keep no heading.
  - Items and next-step rows keep their current escaping and `- **Owner** — action`
    shape.
- **Store-missing recap** (`render_empty_recap` in `src/native/meetings/birth.rs`): only
  `## Notes` and the meta line. It must stay `Owned`.
- **Unchanged.** Keep the begin/end markers, the digest
  (`crate::native::intentions::digest` over the inner text), `recap_state`,
  `replace_recap`, `apply_overwrite_recap`, and the preservation APIs.
- **Existing notes.** A note born with the old layout still digests as `Owned`. `-O`
  re-seeds it in the new layout, and nothing else rewrites it. Add a test that proves
  this: an old-layout owned region is replaced on overwrite.
- **Goldens.** Regenerate `recap_wispr.md` and `recap_adversarial.md`, and update every
  test that asserts recap bytes. Find them with
  `rg -n 'meeting-recap|render_recap|render_birth_recap' src tests`; they include
  `src/native/meetings/tests.rs`, `src/native/highlights_ref/tests/region.rs`, the
  meeting-birth unit and CLI tests, and `tests/cli/highlights/meet_cli.rs`. The inline
  and queued births must stay byte-identical (the existing equality test).
- **Word-sequence test.** Adjust it to strip only bob-authored scaffolding: the
  `## Notes` heading, the meta line, and the `[!summary] TL;DR` callout header. It must
  still prove every notes word survives in order.
- **Docs.** Add a short recap-layout paragraph to `docs/meeting-notes.md` (fictional
  example only). Stay under the docs word caps.

### 2. Never drop words in the Wispr normalizer and dossier round trip (bob-cli)

To reproduce, normalize this Wispr-shaped text through `normalize` → `render_dossier` →
`parse_dossier` → `render_recap`, which is exactly what birth does:

```text
Cedar Labs reviewed the rollout.


Next steps
(Ari) Ship the connector
(Morgan) Confirm the adapter name
```

The recap shows an empty "Next steps" and loses `Ari … name`. Two bugs cause this:

1. `is_short_heading` in `src/native/meetings/normalize.rs` accepts a multi-line
   paragraph. The epic's rule is "a section whose first paragraph is **one** short line
   with no terminal punctuation has that line as its heading". Fix it as follows:
   - Require a single line.
   - Narrow exception: when a block's first paragraph has several lines and its first
     line passes `is_next_steps_heading`, use that line as the next-steps heading. Parse
     the remaining lines as next steps, one per line, with `(Owner) Action` handling.
   - Otherwise every paragraph of the block becomes items, as today.
2. `parse_body` in `src/native/meetings/dossier.rs` silently drops non-blank lines that
   are neither headings nor `- ` items. Make it lossless:
   - Under a next-steps heading, each such line becomes a next step through
     `parse_next_step`.
   - Under a section, each becomes an item.
   - Before any heading, each becomes an item of an unheaded section.

   Also make `render_heading` (dossier) collapse internal whitespace, so a heading can
   never span lines.

Add regression tests:

- The input above, plus a section-heading variant ("Migration risks" followed directly
  by a sentence).
- Each test asserts the word-sequence invariant for the dossier body and for the recap
  rendered from the parsed dossier.
- Each also asserts that `normalize(render_dossier(n, m))` reproduces `n`.

### 3. Meeting PDF title hook outside the preamble (bob-cli)

`src/native/highlights_ref/meeting_profile/meeting.lua` (around line 518) emits a body
raw block `\AtBeginDocument{\hypersetup{pdftitle={…}}}`. It runs after
`\begin{document}`, and older LaTeX kernels abort with "Can be used only in preamble".
This deterministically failed 2 of 4 `native::meetings::profile` tests on another host.

- Delete that emission. The PDF Info Title already comes from pandoc's `title` metadata
  (`render_profile_pdf` passes `--metadata title=<reference title>`). The land agent
  confirmed that all 4 profile tests, including the lopdf Title assertion, still pass
  without it.
- Drop the then-unused `bob-meeting-pdf-title` extra metadata in
  `src/native/meetings/profile.rs`.
- Add a cheap guard test that `MEETING_FILTER` contains no `\AtBeginDocument`, with a
  comment explaining why.
- Run `cargo test -p bob-obsidian --lib native::meetings::profile` (real pandoc/xelatex
  are installed on apollo).

### 4. `bob ref meet` help and receipts read as user docs (bob-cli)

In `src/native/highlights_ref/meet_cli.rs`:

- **Value help.** Rewrite the argument and option help strings, and the after-help
  "Modes", "Meeting and destination selection", "Notes, config, and editing", and
  "Output and failures" sections. Describe behavior to a user instead of the spec.
  Remove phrases such as "Required codes:", "Missing block IDs need an actionable
  refusal", "joined as the existing parser expects", and "using the gcal note parser".
  Use the epic contract's wording as the source:
  - `-s` lists `next`, `ready`, `wip`, `read` and the default (`meetings.status`, else
    `next`).
  - `-w` says it defaults to `meetings.window_days`, else 7.
  - `-p` names the `meetings.producer` default.
  - EVENT lists the accepted forms with one short example.
  - Exit codes are 0 success, 1 failure, and 2 when bob needs a choice (ambiguous or
    missing meeting, a needed parent, already filed).
- **Keep the rest.** Keep the alphabetical option order, every short alias, the Examples
  and Environment sections, the `See docs/meeting-notes.md` pointer, and the strings the
  help test asserts. Update `help_lists_options_alphabetically_with_contract` to match.
- **Receipts.**
  - The filing and queued receipts print `(N pages)`. Use `1 page` for one.
  - The dry-run `notes` line prints the Rust Debug shape name (`WisprPlainText`). Print
    the same shape string the JSON payload uses.
  - Keep the line prefixes the Obsidian adapter parses unchanged:
    `Filed meeting notes as `, `Replaced meeting notes as `, and `relation  `.
- **Docs.** Run `just docs-gen` to regenerate the CLI reference.

### 5. Obsidian ref-meet toast never offers Open event (bob-plugins)

Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`). Read its
`AGENTS.md` and work only in the printed checkout. Never hand-edit `main.js`; edit
`src/` fragments and build.

- **The bug.** `cmAdapterRefMeet` in `plugins/bob-command-mode/src/130-adapters.js`
  matches the relation line with `/^relation\s+.*\s+(\S+#\^[A-Za-z0-9-]+)\s*$/im`. bob
  actually prints a wikilink: `relation  notes from [[20261124#^cedar-review]]`. The
  trailing `]]` defeats the match, so the Open event action never appears.
  - Accept a bare `note#^id`, a wikilink `[[note#^id]]`, a path form
    `[[2026/20261124#^id]]`, and an optional `|alias`.
  - Pass the bare `note#^id` target, without brackets or alias, as `eventRef`.
- **Tests.** In `scripts/test-command-mode-outcomes-api.cjs`, make the main ref-meet
  case use bob's real receipt shape (the wikilink form above). Keep a bare-token case,
  and add a path-form-with-alias case.
- **Build and gates.** Run `npm run build`, the command-mode unit tests
  (`node --test scripts/test-command-mode-outcomes-api.cjs scripts/test-command-mode-sheet.cjs`),
  and the repo's other required gates per its `AGENTS.md`. Known unrelated failures:
  docs budgets tracked by bob-cli-ob and bob-cli-jz, and timing flakes tracked by
  bob-cli-km and bob-cli-io. Do not fix those here.
- **Ship.** Commit in bob-plugins, then deploy with `bob plugins sync`.

### 6. Verify bob-cli

- Run `just fmt`, then `just check`. It takes a long time, so start it through
  `/sase_monitor` and wait for it. Do not run `just check-full`. Also run
  `just docs-check`.
  - The bob-plugins `bob-navigation-hotkeys` reference page is already over its cap
    (bob-cli-jz). If that is the only docs-check failure, record it and continue.
  - Any other failure is yours to fix.
- End-to-end smoke test with a scratch vault under `/tmp`, never `~/bob`:
  - Set up a `job` project note and a daily note `2026/20261124.md` whose `## Events`
    section holds
    `- [*] #task **1400-1430** Cedar Labs quarterly review [root:: job] ^cedar-review`.
  - Make the vault a git repo with a bare remote, and add a `.gitignore` containing
    `/xlib/`.
  - Set `BOB_DIR`, `BOB_CONFIG_FILE`, `BOB_NOW="2026-11-24 15:00:00"`, `XDG_STATE_HOME`,
    `BOB_VAULT_SYNC_LOCK_FILE`, and `BOB_VAULT_SYNC_STATE_FILE`, all pointing into the
    scratch dir.
  - Run `bob ref meet 20261124#^cedar-review -i notes.txt -p "Wispr Flow"` with the step
    2 repro text and the real pandoc/xelatex.
  - Confirm that the ref note recap has the step 1 layout, keeps every next-step word,
    and that the receipt says `1 page` or `N pages`.

### 7. Close epic bob-cli-pw (final step)

1. Run `sase bead epic-symbols bob-cli-pw`. It listed no entries at planning time. If
   any appear, resolve each one (wire it up, privatize it, add a non-test pragma, or
   delete it) before closing.
2. Close the epic. Do not use `--force`.

   ```bash
   sase bead close bob-cli-pw --note "<verification>"
   ```

   Write a verification note that covers all of the following:
   - **Repairs**, with their commits: the recap layout, the lossless Wispr/dossier round
     trip, the removed preamble-only PDF title hook, the user-facing `ref meet` help and
     receipts, and the bob-plugins Open event adapter fix plus deploy.
   - **Results**: `just check` passed; the docs-check result; the smoke-test result.
   - **The land agent's audit** (already recorded as an epic note): all 8 phases
     verified. The Mac native repairs 3931c308 are on origin/master and green on the
     exact SHA (run 45614436809). No integration changes were needed from the non-epic
     commits since dbf405e3.
   - **Follow-up dispositions**:
     - +1 on bob-cli-gd, bob-cli-ob, bob-cli-jz, bob-cli-km, and bob-cli-io.
     - New features bob-cli-pz, bob-cli-q0, bob-cli-q1, bob-cli-q2, and bob-cli-q3.
     - Declined as fixed or moot: pw.2 #1, pw.3 #1, pw.4 #1, pw.6 #1–#4.
     - The pw.6 #8 recap and epic note #4 TeX defects are fixed here.
     - The nested dispositions recorded on bob-cli-pw.5.1 and bob-cli-pw.8.1 stand.

3. Run `just symvision` if the justfile has that recipe. It had none at planning time;
   say so in your final response if it is still missing.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `sase/repos/plans/202611/meeting_notes_event_refs.md` (the PLAN path shown by
   `sase bead read bob-cli-pw -r "<why>"`). Change nothing else in that file.
5. The epic has no `parent_bead`, so no ancestor needs handling.
