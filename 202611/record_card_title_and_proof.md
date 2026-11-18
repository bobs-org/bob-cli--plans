---
tier: epic
title:
  "Finish the Record Card: the title moves into the sheet, records drop their H1, and
  real Obsidian proves it"
goal:
  Every record note shows its title at the top of the record sheet instead of an H1. bob
  writes and keeps record notes without an H1, and the live vault has none. The
  record-sheet tests pass on any date. A real-Obsidian walkthrough on athena proves the
  Record Card epic acceptance, including the Properties hide.
parent_bead: bob-cli-ms
phases:
  - id: sheet-title
    title: Record sheet shows the title and mounts on H1-less records
    depends_on: []
    size: medium
    description:
      "sheet-title: in bob-ledger-tools, render the record title at the top of the
      record sheet with page-title styling, mount the sheet at the top of H1-less
      records (including frontmatter-only notes) in Live Preview and Reading, make the
      title's one home the sheet, pin the rollover test's clock, update docs, bump, and
      deploy."
  - id: h1-retire
    title: bob writes records without an H1 and tidy removes redundant ones
    depends_on:
      - sheet-title
    size: medium
    description:
      "h1-retire: bob records new, update, and capture write record notes with no H1,
      Notes insertion handles H1-less bodies, bob records tidy also removes an H1 that
      only repeats the frontmatter title, docs and tests follow, then just install and
      the live-vault cleanup per the first_cleanup decision."
  - id: obsidian-proof
    title: Real-Obsidian Record Card walkthrough, hide-selector fix, and vault coverage
    depends_on:
      - sheet-title
      - h1-retire
    size: large
    description:
      "obsidian-proof: add the obsidian-record-card-e2e CDP walkthrough with an offline
      npm test wrapper, run it on athena with a current BOB_BIN, fix the Properties-hide
      selectors and any other defects it finds, run the read-only live-vault coverage
      check, and record the evidence as artifacts."
decisions:
  first_cleanup:
    ask:
      Should the h1-retire coder run the real bob records tidy on the live vault once,
      right after just install?
    choices:
      now:
        One guarded tidy commit strips the H1 from all ~191 records as soon as the phase
        lands
      nightly:
        Coder only dry-runs; the next bob nightly strips the H1s, so titles show twice
        until then
    default: now
    why: Bryan asked for the H1 to go, and nightly leaves up to a day of doubled titles
    answer: now
proposed_by: bbugyi200.apollo.bob-cli-ms.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-17 22:26:11
status: wip
---

- **PROMPT:**
  [prompts/202611/record_card_title_and_proof.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/record_card_title_and_proof.md)
- **PARENT:**
  [202611/record_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_card.md)

# Finish the Record Card: the title moves into the sheet, records drop their H1, and real Obsidian proves it

## Why this epic exists

This is the remaining work of epic `bob-cli-ms` (Record Card), found by its land agent.
Its `parent_bead` is `bob-cli-ms`: when this epic lands, its land agent resumes the
interrupted `bob-cli-ms` landing (verify, close, `just symvision`, mark
`plan:202611/record_card.md` done, report to Bryan). None of that is a phase here.

Read first, through audited interfaces:

- `sase bead read bob-cli-ms -r "<why>"`: Bryan's notes, the reviewer DECISIONS, and the
  landing triage note. The DECISIONS stay binding: `inline_title = hide`, and
  `decision_record = no`. No memory note may be edited by this epic.
- `sase artifact read plan:202611/record_card.md "<why>"`: the design contract and the
  land-agent acceptance list. Its "Integration and land-agent acceptance" section is the
  checklist for `obsidian-proof`.
- `sase artifact read plan:202611/record_card_field_jump_co_landing.md "<why>"`: the
  co-landing rules with the Schema Card field-jump work (deploy discipline, item 8).
- `sase artifact read plan:202611/record_notes_on_demand.md "<why>"`: the shipped
  `bob records tidy` and Notes-on-demand rules that `h1-retire` extends.

### What the land agent verified (bob-cli `97f844ca`, bob-plugins `f8a8619`)

All seven `bob-cli-ms` phases are closed and their code is on both masters.

- bob-cli has `bob records show TYPE STEM` and `bob records update -i`. bob-plugins has
  the record sheet, the update session, the Record Card with typed stages, the entry
  points, `==`, and the Properties hide.
- The vault holds a clean origin/master deploy. bob-ledger-tools 1.133.0,
  bob-navigation-hotkeys 2.87.0, and bob-command-mode 0.2.0 each have a `main.js`
  byte-identical to the repo's.
- apollo's installed `bob` (0.5.0) reports `capabilities: ["record_values_v1"]`.
- These focused suites pass: record-card (19), record-card-stages (9), record-update
  (14), command-mode record surface (3), navigation parent-album-record (16), and
  schema-double-equals (21).

Three gaps remain:

1. **Bryan's epic note #3 was never implemented.** It reads: "get rid of the H1 section
   and render the title at the top of the schema card with appropriate formatting". That
   note replaces his note #1, which asked for a blank line between the H1 and the sheet.
   Today every record still has `# <title>` in its body, and the sheet header shows only
   the crumb (`242j-plugin-record-sheet-dom.js`, `recordSheetRenderCrumbs`). A read-only
   scan of the live vault found:
   - 191 records with `type: "[[record]]"`;
   - every one has an H1 whose text equals its frontmatter `title`;
   - 46 hold nothing but that H1 in their body.
2. **`scripts/test-ledger-tools-record-sheet.cjs` fails on every date but 2026-11-17.**
   The failing test is "midnight rollover uses the plugin tick and never leaves a
   hanging timeout". `setupRecordSheet` resets `recordSheetDay` to the real today
   (`bob-cli-ms.7` note #2). The epic caused this, and it fails today.
3. **The epic plan's real-Obsidian proof never ran.**
   `scripts/obsidian-record-card-e2e.cjs` does not exist, and no phase ran a CDP
   walkthrough (this container has no Obsidian or Xvfb). One concrete risk is the hide
   CSS in `plugins/bob-ledger-tools/styles.css`:
   - it uses direct-child selectors like
     `.markdown-source-view.bob-record-sheet-hides-properties > .metadata-container` and
     `… > .inline-title`;
   - the class goes on `.markdown-source-view` / `.markdown-preview-view`
     (`242k-plugin-record-sheet.js`, `recordSheetSetEditorScope`,
     `recordSheetReadingScope`);
   - in Obsidian, Properties and the inline title appear to sit deeper (under
     `.cm-sizer` in Live Preview, under the preview sizer's header block in Reading).

   So the hide may do nothing in real Obsidian. Only a DOM probe settles it.

## Shared rules

**bob-plugins.**

- Open it with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and read its
  `AGENTS.md`.
- Edit `src/` fragments and `src/fragments.json` order, then `npm run build`. Never
  hand-edit `main.js`. Keep every fragment at or below 1000 lines, and register new
  tests in `package.json` `test:unit`.
- Bump bob-ledger-tools' minor version; read the current value at implementation time.
- Run `npm run build`, `npm run docs:gen`, `npm test`, `npm run validate`, and
  `BOB_CLI_DIR=<your bob-cli checkout> npm run docs:check`.
- Deploy with `bob plugins sync`. Right before syncing, fetch and merge origin/master
  and rebuild, so the deployed build is the one that lands.
- Follow the repo's sync-refusal procedure. The `-F` pre-authorization in the co-landing
  plan covered only the two original epics, so it does not apply here: never pass `-F`
  on your own.

**bob-cli.**

- Read `cli_rules.md` and `docs.md` with `/sase_memory_read` before CLI or docs changes.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`.
- JSON stays `schema_version: 1`, and changes are additive.
- Run `just docs-gen` and the `just check` gate through `/sase_monitor`; do not run
  `just check-full`.

**Known reds you did not cause.** Leave these alone and do not re-report them:

- the 25ms timing budgets under full-suite load (`bob-cli-km`, `bob-cli-io`,
  `bob-cli-ki`, `bob-cli-ju`);
- the album schema-writer held-lock self-test (`bob-cli-n0`);
- `just docs-check`'s contract-count unit tests from `c50f8895`, noted on `bob-cli-mx`.

Report anything else that fails identically on a clean base as a `PROPOSED FOLLOW-UP:`
note on your phase bead. Phase workers never create task beads.

**Out of scope here.** These ideas are already filed as tasks: title or stem rename
(`bob-cli-n1`), quick-pick menus on sheet chips (`bob-cli-n2`), in-sheet editing
(`bob-cli-n3`), and provenance-key cleanup (`bob-cli-n4`). Mobile still has no sheet;
there, record notes show Obsidian's filename inline title and native Properties
(fail-open, as before).

## sheet-title

**Goal.** The record sheet carries the record's title, and it mounts correctly whether
or not the note has an H1. After this phase, removing the H1 bytes loses nothing.

**Design.**

- **Title row.** The sheet's first row is the title, then the toolbar (`⊞ Edit`, the
  collapse toggle) at its inline end. The crumb (`📇 Album › Noun`) moves to a second,
  muted UI-small line. Use a `div.bob-record-sheet-title` with `role="heading"` and
  `aria-level="1"`, not an `h1` element, so Obsidian and theme `h1` rules never style or
  count it.
- **Title styling.** Use the theme's page-title variables: `--inline-title-font`,
  `--inline-title-size`, `--inline-title-weight`, `--inline-title-color`, and
  `--inline-title-line-height`, with `--h1-*` fallbacks. The page then reads as titled,
  the way an ordinary note does. Long titles wrap; at 375px or narrower the toolbar
  wraps below the title. Check light and dark themes.
- **Title text.** Use the frontmatter `title`, trimmed. If it is absent, fall back to
  the first H1 text, then the stem. Insert it as a text node and never parse it as
  Markdown.
- **Collapsed sheet.** It shows the title row and one line: breadcrumb, then
  choice-field chips (group field first), then `· N fields`.
- **Broken album schema.** The warning sheet shows the title row too.
- **One home for the title.** The `title` key's home becomes the sheet title, whether or
  not an H1 exists. Retire the fold's `title` entry and its `differs from heading` lint,
  and update the coverage map, the coverage unit test, and the sheet signature. The
  sheet `aria-label` stays `<Title> record details`. The Record Card title
  (`<album emoji> <title>`) is unchanged.
- **Live Preview mount.** On a note without an H1, the sheet mounts right after the
  frontmatter. That covers notes whose document ends at the closing `---`, which
  `recordSheetAnchor` already attempts. Verify that the widget side and block placement
  put it above any `## Notes` or `## Source` heading. Verify the cursor can still reach
  the first body line and the end of the frontmatter.
- **Reading mount.** On a note without an H1, the sheet mounts at the top of the
  rendered body. That includes a frontmatter-only note, whose body yields no ordinary
  section for a post-processor to receive. If the post-processor path cannot reach such
  a note, mount from the view instead: the active Markdown view's preview container on
  `layout-change`, `file-open`, or `metadataCache` `changed`. Use the same ownership,
  stray-removal, and teardown bookkeeping, and do not double-mount. Embeds and hover
  popovers of H1-less records show the sheet as before, with embedded Properties left
  alone.
- **Transition.** Until `h1-retire` strips the bytes, existing records show both their
  H1 and the sheet title. That is accepted and needs no special hiding.
- **Other title readers.** Search bob-ledger-tools for code that derives a record's
  display title from its first H1 rather than frontmatter `title`;
  `notebookHeaderNoteTitle` is one candidate. Make record paths prefer `title`, so
  crumbs, cards, and popovers keep the pretty title once the H1 is gone.
- **Rollover test.** Inject the clock (or a fixture "today") so the "midnight rollover"
  test passes on any date. Keep its intent: rollover rides the plugin tick and leaves no
  hanging timeout.

**Tests.** Extend `scripts/test-ledger-tools-record-sheet.cjs` (and its card or DOM
siblings where touched). Cover:

- the title row's text, role, and level for a titled, an untitled, and a broken-schema
  record;
- collapsed and expanded layouts;
- text-node safety for a title containing `[[x]]`, `<b>`, and `#tag`;
- coverage with and without an H1;
- the retired fold entry;
- Live Preview anchor positions for frontmatter-only, frontmatter + `## Notes`, and
  legacy-H1 documents;
- the Reading mount on an empty-body note, with no double mount;
- the rollover test run with a fake date other than 2026-11-17.

**Docs.**

- `docs/record-card.md`: the sheet now leads with the record's title, and record notes
  carry no H1.
- `docs/bob-ledger-tools.md`: a sentence at most; it is near its word cap, so link out.
- Run `npm run docs:gen`.

**Deploy.** Bump bob-ledger-tools' minor version, run the gates, then
`bob plugins sync`.

## h1-retire

**Goal.** bob never writes an H1 into a record note, and `bob records tidy` removes the
redundant ones that exist. Depends on `sheet-title`, whose deploy must be live before
any live-vault cleanup.

**Build** (bob-cli; read `src/native/records/render.rs`,
`src/native/records/notes_section.rs`, and `src/native/records/tidy.rs` first).

- **Render.** In `render_record_with_identity`, the body no longer starts with
  `# {title}`.
  - A record with ordinary bullets gets a body of `\n## Notes\n\n- …`, followed by any
    named sections in today's order and spacing.
  - A record with neither notes nor sections ends right after the closing `---\n`.
  - Update the doc comment, which says the body always holds H1. The capture paths
    (`src/native/capture/record_note.rs`, `src/native/capture_record.rs`),
    `bob records new`, and the `--dry-run` previews all share this renderer; confirm
    each output.
  - Bob Mac Capture only renders the `body` string bob returns
    (`decisions:mac-capture-is-a-thin-client`). Open bob-mac-capture read-only through
    `/sase_repo` and check that nothing there assumes a record body starts with `# `.
    Record any finding as a `PROPOSED FOLLOW-UP:` note rather than editing Swift.
- **Update.** `append_bullets_under_notes` already inserts a missing `## Notes` before
  the first level-1 or level-2 heading when the body has no H1. Make sure an empty or
  whitespace-only body gets a canonical `\n## Notes\n\n- …\n` directly after the
  frontmatter, with no stray blank lines. Make sure a body with only `## Source` gets
  Notes before Source. Named sections follow the same rule.
- **Tidy.** `bob records tidy [TYPE]` also removes a redundant H1, defined as:
  - the first non-blank body line outside fenced code;
  - a level-1 ATX heading (`# …`);
  - whose text, trimmed of surrounding whitespace and closing `#`s, equals the
    frontmatter `title` string exactly.

  Remove that line and the blank lines that follow it. An H1 that differs from the
  title, an H1 that is not the first body content, and any later H1 are kept and counted
  as kept. A body that becomes empty leaves the file ending at the closing `---\n`.
  Preserve CRLF and BOM, guard each rewrite by its full preimage as today, and never
  touch frontmatter.

  Tidy runs the empty-`## Notes` removal and the H1 removal in one pass, with one scoped
  commit. Keep the subject format `records: tidy (N notes)`. JSON stays schema
  1. Add the counts `h1_removed` and `h1_kept` (and per-note reasons in the existing
     per-note detail), keeping every existing field. Human output names both kinds of
     removal. `bob nightly` already runs tidy, so stray H1s are cleaned nightly, which
     is intended.

- **Readers.** `bob records show` already reports `heading: null` without an H1; confirm
  the human card uses `title`. Search bob-cli for record-path code that reads the first
  H1 as a title: `records check`, the album index, `capture-related` similar-records
  display, the note catalog, and the period and stats readers. Make each prefer
  frontmatter `title` for records. Notebooks, references, and other note kinds keep
  their H1 behavior.

**Tests.**

- Render goldens: with notes, with sections only, and empty.
- Capture create and update for a new record (CLI tests under `tests/cli/records.rs` and
  `tests/cli/capture/record.rs`), plus existing goldens that pinned `# Title`.
- Notes insertion into an empty body and into a Source-only body.
- Tidy, covering:
  - an equal H1 removed;
  - a differing H1 kept;
  - an H1 after prose kept;
  - an H1 inside a fence ignored;
  - a quoted title with escapes (`title: "Ma \"Pa\" Deli"`);
  - CRLF and BOM;
  - an H1 plus empty Notes removed in one pass;
  - an empty result;
  - dry-run writing nothing;
  - one commit;
  - JSON counts.
- Fingerprint unchanged by a tidy run, since the body is excluded.

**Docs.** Update the record note shape and tidy sections of `docs/records.md` (including
"A note with no ordinary bullets holds only its H1" and the Notes-placement text that
mentions the H1 region), `docs/reference/vault-format.md`, any record body example under
`docs/capture*.md`, and the `records tidy` help text. Run `just docs-gen`.

**Land and install.** After `just check` passes and the work lands, run `just install`
from the landed tree on apollo. Confirm that `bob records tidy -d -f json` on the live
vault reports about 191 `h1_removed` and 0 `h1_kept`, and change nothing else.

> [!decision] first_cleanup = now After the dry-run looks right, confirm the deployed
> bob-ledger-tools carries the sheet title (`bob plugins sync` status, or `main.js`
> byte-identical to origin/master). Then run `bob records tidy` once on the live vault.
> Record the commit SHA and counts on the phase bead, and spot-check
> `sase_blog_zettel/bead_lifecycle.md` and one `eat` record with a `## Source` block.

> [!decision] first_cleanup = nightly Stop after the dry-run and record its counts. The
> next `bob nightly` on apollo performs the cleanup.

## obsidian-proof

**Goal.** Owns the real-Obsidian proof that the `bob-cli-ms` plan assigned to its land
agent, plus fixes for whatever the proof finds. Depends on both earlier phases, so it
proves the final, H1-less layout.

**Build** (bob-plugins).

- Add `scripts/obsidian-record-card-e2e.cjs` (split a `-lib` file if needed), modeled on
  `scripts/obsidian-album-ui-e2e.cjs`. Add the offline wrapper
  `scripts/test-obsidian-record-card-e2e.cjs` to `npm test`; it validates the fixture
  and probes without launching Obsidian. Use an isolated scratch Git vault and profile
  and real `BOB_BIN` built from current bob-cli master. Seed:
  - a zettel;
  - a sparse movie;
  - a legacy-keyed restaurant with a wikilink location;
  - a subscription (docked strip, list `match`);
  - a missing-required record;
  - a broken-schema album's record;
  - a `bank/2026-11.md` lookalike;
  - a frontmatter-only record (no body);
  - one record that still has a legacy H1 (pre-tidy shape).
- Document the run in `docs/develop/validation.md` next to the other CDP walkthroughs.

**Run** on athena, which has Obsidian and Xvfb per `docs/develop/validation.md`. Read
tailnet memory with `/sase_memory_read` for SSH and users. Use a disposable copy of the
plugins tree and a bob built from current bob-cli master, never Bryan's checkouts or
vault. Launch long runs through `/sase_monitor`. Inspect every screenshot.

**Verify** each item in `plan:202611/record_card.md` "Integration and land-agent
acceptance" step 1, adjusted for this epic:

- The sheet sits at the top of the note body:
  - it leads with the record title in page-title styling;
  - with the hide active, no inline title and no H1 appear;
  - values use bob's vocabulary, and each key appears exactly once;
  - check light and dark, at 375px and wide;
  - collapse and fold work, and reduced motion is honored.
- **Properties hide.** Probe the real Live Preview and Reading DOM for
  `.metadata-container` and `.inline-title`. Fix the selectors (and if needed which
  element carries the class) so the view's own Properties and inline title hide:
  - only when `hideEligible` holds;
  - never inside embeds or hover popovers;
  - with each removal path still restoring them: file switch, type or parent change,
    mode switch, view destroy, plugin disable.

  Old-bob/no-capability, broken schema, legacy type, outside folder, Source mode, and
  embeds all fail open. Add a unit test that pins the probed DOM shape.

- **Entry points.**
  - `==` opens the card with no reindent and no undo entry, while `3==`, Source mode,
    and non-records reindent.
  - Ctrl+Shift+P, the palette, Command Mode, `⊞ Edit`, and row/chip/ghost/missing clicks
    open the right field.
  - From the card, `=` opens the album Schema Card on the same field, where Shift+J and
    `3` navigate and never write. The Record Card shows no Schema Card keycaps
    (co-landing item 8).
- **Keyboard-only writes.**
  - A digit pick rewrites only the `status:` span and the `[!records]` callout in one
    write, leaving all other bytes identical.
  - `t`, Ctrl+D, and the Tab fill-out across kinds work.
  - Undo restores the value and the regrouping.
  - A stale edit (a Source-mode edit in another pane), an invalid value, and an empty
    required field each refuse and write nothing.
  - `y` → Done returns to the card.
- **Byte neutrality and freshness.** Rendering, navigating, folding, cancelling, and
  opening are byte-neutral by hash. The sheet updates right after a save with no
  metadataCache flash of old values.

Fix every defect the walkthrough finds in the plugin, with unit tests. If a fix needs
bob-cli, make it in bob-cli under the shared rules. Re-run until green, then bump, gate,
and deploy.

**Vault coverage check** (apollo, read-only). For every record,
`bob records show <album> <stem> -f json` must account for every frontmatter key across
`properties`, `identity`, and `other`, and report no unexpected findings. Record the
counts.

**Evidence.** Register the JSON report, the coverage counts, and the key screenshots as
SASE artifacts; read `sase_artifacts.md` with `/sase_memory_read` first. Reference them
from a note on this phase's bead. If athena cannot run Obsidian, record the run as
skipped with the exact reason, keep the offline wrapper green, and record a
`PROPOSED FOLLOW-UP:` note. The hide-selector fix must still be justified from a
recorded DOM source.

## Landing this epic

The land agent:

- audits the three phases against this plan;
- checks bob-cli `just check` and the bob-plugins gates;
- confirms the vault deploy is byte-identical to bob-plugins origin/master;
- triages phase `PROPOSED FOLLOW-UP:` notes;
- closes this epic.

Because `parent_bead` is `bob-cli-ms`, it then resumes the `bob-cli-ms` landing:

- re-verify the whole Record Card promise;
- retire any `sase bead epic-symbols bob-cli-ms` entries;
- close `bob-cli-ms` with a verification note;
- run `just symvision`;
- set `status: done` in `plan:202611/record_card.md`.

Its final report tells Bryan what he must do on the Mac:

- run `just install-all` before editing turns on there (until then the capability gate
  keeps Properties visible);
- try `==` on `bead_lifecycle`, a status digit pick, a claim edit, and Undo, and look at
  how the page reads with the title in the sheet.
