---
tier: tale
size: medium
title: "Land article notes (bob-cli-8g): fix verified spec gaps and close the epic"
goal:
  The bob articles engine, CLI, capture selectors, docs, and the bob-articles plugin
  match the article_notes spec on every gap the landing agent reproduced, and epic
  bob-cli-8g is verified, closed, and its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-8g.land
bead: bob-cli-8g
status: done
---

- **PARENT:**
  [202610/article_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/article_notes.md)
- **BEAD:**
  [bob-cli-8g](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8g/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-8g.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-8g.land.md)
- **COMMITS:**
  - [cf7a0bd](https://github.com/bobs-org/bob-cli--plans/commit/cf7a0bdc691edf394e9430be9c275ead4b5685fc)
    — docs(plans): mark article notes epic complete

# Plan: Finish and land the article-notes epic (bob-cli-8g)

## Context

Epic **bob-cli-8g** ("Article notes: bob-owned section numbers and calm layout") shipped
all nine phases:

- the pure engine, links, and capture work in `src/native/articles/` and
  `src/native/capture/sections.rs`;
- the `bob articles` CLI with `format`, `list`, `outline`, and `adopt`;
- inbound link repair and the nightly check;
- the vault scaffolding;
- the `bob-articles` Obsidian plugin in the bob-plugins linked repo.

Read the epic's spec with `sase bead read bob-cli-8g -r "<why>"`. The PLAN path it
prints is `plan:202610/article_notes.md`.

The landing agent checked every phase against that spec. It reproduced each gap listed
below with the installed `bob` or with real bob output. This tale fixes those gaps, then
closes the epic.

**Constraints**

- **Epic DECISIONS are final:** `memory_glossary = no` and `memory_decision = no`. Do
  not create or edit any SASE memory note.
- **Known baseline failures are not this work:**
  - `just check` fails only two lib tests, both owned by the active epic bob-cli-87:
    `note_snapshot::profile::tests::pandoc_maps_every_contract_construct_to_tex` (fails
    with "lifted link") and `renders_contract_with_clean_text_layer_and_metadata`
    (`titlesec.sty` is missing on apollo).
  - bob-plugins `npm test` fails only six tests, tracked by bob-cli-6v, bob-cli-6w, and
    bob-cli-7i. bob-cli-3w is an occasional stage-ranker flake.
  - Do not file beads for any of these.
- **Follow-up triage is done.** The landing agent recorded it on the epic as the note
  "LANDING FOLLOW-UP TRIAGE". Do not redo it.
- **Do not use `just check-full`.** File-change verification is `just check`.
- **Never run `bob articles format` against `~/bob` without `-c`, `-d`, or `--stdin`.**
  The format command writes by default. The only exception is the one silent re-run in
  Part C.

## Part A — bob-cli engine, CLI, capture, and docs

### A1. Embeds are never refreshed (`src/native/articles/links.rs`, `link_spans`)

- **Bug:** at `![`, the scanner treats the text as a Markdown link and skips past
  `![[…]]`. As a result, `![[#Beta]]` and `![[c#Alpha]]` are neither refreshed nor
  reported. Inbound repair uses the same scanner, so embeds in other notes are missed
  too.
- **Fix:** treat `![[…]]` as a wikilink span.
- **Tests:**
  - `![[#Beta]]` becomes `![[#1.1 Beta]]`;
  - `![[note#Beta|alias]]` keeps its alias;
  - a block embed `![[#^id]]` is left untouched;
  - an inbound embed in another note is rewritten (CLI test in `tests/cli/articles.rs`).

### A2. Indented code is not protected (`links.rs`, `layout.rs`)

- **Link bug:** `protects_links` omits `Region::IndentedCode`, so `    code [[#Alpha]]`
  is rewritten. Add that region, and make sure the pseudo-footnote scan treats indented
  code as protected too.
- **Blank-line bug:** F6 collapses the blank run between two chunks of the same indented
  code block. For example, `    code`, then two blank lines, then `    more code`. That
  changes the code's content. Leave a blank run alone when both its neighbours are
  indented code.
- **Tests:** add a case for each bug.

### A3. F5 ignores protected-block neighbours (`layout.rs`)

- **Bug:** a heading followed directly by a `$$`, `%%`, `<!--`, or HTML-block line gets
  no blank line after it. To reproduce, put `# Alpha` directly above `$$`.
- **Fix:** apply F5's blank line before and after the heading for these neighbours too.
- **Tests:** add a fixture case.

### A4. One bad note blocks the whole batch (`cli.rs` `run_format`, `vault.rs`)

- **Bug 1:** `run_format` writes only when no plan is skipped or in error
  (`plan_failed`). So `format -a` with one refused article (for example `holds-tasks`)
  writes nothing, yet still prints `✓ good · …`.
- **Bug 2:** `vault::write_plans` aborts the whole batch when one preimage has drifted.
- **What the spec says** (§1 and §7): skip only the refused or drifted note, and give a
  reason.
- **Fix:**
  - Write every `changed` plan, whatever the other notes' statuses are.
  - On preimage drift, set that plan's status to `error` with reason
    `{code: "preimage-changed", message: "<path> changed since the article plan was made; not written"}`,
    skip its inbound writes, and continue.
  - Keep exit 2 when any note was skipped or in error.
  - Render human output from the post-write statuses, so a note that was not written
    never gets a `✓`. JSON shows the final statuses.
- **Non-UTF-8 notes:** `discover_articles` and the inbound scan abort the whole run on
  the first note that is not valid UTF-8. That breaks `list`, `-a`, nightly, and
  completion. Treat `InvalidData` reads as "not an article / not a linker" and skip
  them. Other I/O errors still fail the run.
- **Tests:**
  - CLI: with `-a`, one holds-tasks article plus one good article. The good one is
    written, the refusal is printed, and the exit code is 2.
  - Drift: a unit test, or a CLI test if that is feasible.
  - A non-UTF-8 `.md` file does not break `bob articles`.

### A5. Counts, notice, and plurals (`sections.rs`, `mod.rs` `format_notice`, `render.rs`)

- **Bug:** `changes.numbered` increments for every heading whenever numbering is on. As
  a result:
  - a whitespace-only change reads `Article formatted · 2 sections numbered`;
  - an unchanged note reports `numbered: 10`.
- **What the counts must mean:**
  - `numbered` counts headings that had no written token and now get a number.
  - `renumbered` counts headings whose written token changed.
  - An unchanged note reports 0 for both.
  - A first adoption still reports `10 sections numbered` for `sase_blog_contents`.
- **Promotion clause:** `(#### → ###)` is hard-coded. Derive it from the actual
  promotions. Show the real levels when every promotion shares one from→to pair;
  otherwise omit the parenthetical.
- **Plurals:** bob-owned strings must use correct singular and plural forms:
  - `1 section numbered`, `1 heading promoted`, `1 link refreshed`,
    `1 Setext heading converted`;
  - the holds-tasks message: `#task on line 23` but `#task on lines 4, 9`;
  - the human summary lines, and the list footer (`1 article`).
- **Tests:** update `src/native/articles/tests/fixtures/plan_v1.expected.json`, the
  outline fixtures if they are affected, and the CLI expectations.

### A6. The pseudo-footnote check is too broad (`findings.rs` `pseudo_footnotes`)

- **Bug:** the check excludes a body that contains `/` or `#` anywhere. On the real
  note, `[b]: TODO: Update commit/line counts!` is therefore not flagged.
- **Fix:** test only the destination, which is the first whitespace-delimited token of
  the body. Exclude it when it:
  - contains `://`;
  - starts with `mailto:` (case-insensitive);
  - or starts with `#`, `/`, `./`, or `../`.
- **Tests:**
  - flagged: the `[b]` line;
  - not flagged: `[x]: https://…`, `[y]: ./a.md`, and `[z]: #h`.

### A7. Prose mentions of `#task` refuse the note (`marker.rs`)

- **Bug:** any line containing `#task` refuses the note. §1 says only `#task` task lines
  count.
- **Fix:** restrict the check to list-item checkbox lines (`- [ ]`, `* [x]`, `1. [?]`,
  with any status character) that carry a `#task` tag outside inline code.
- **Tests:** a prose mention no longer refuses; a real task line still does.

### A8. The `bad-option` line is wrong (`marker.rs`)

- **Bug:** the finding is always reported on line 1.
- **Fix:** report the 1-based line of `section_numbers:`.

### A9. Unresolved notes get no suggestions (`vault.rs` `resolve_note`)

- **Bug:** `resolve_note` uses `NoteIndex::resolve` and returns a generic error.
- **Fix:** use `VaultLinkResolver` from `src/native/vault_links.rs` (`resolve`,
  `candidates`, `near_stems`), the way other commands do (see
  `src/native/ref_library/migrate_tasks/write.rs`).
  - A missing note lists near-miss stems.
  - An ambiguous note lists its candidates.
  - Both exit 2.
- **Tests:** a CLI test for each case.

### A10. Capture does not try the raw title first (`src/native/capture/sections.rs`)

- **Bug:** `target_bullet_section` checks "raw or identity" per heading, in document
  order. With `### 1.1.1 Examples` and `### 1.2.1 Examples`, the exact selector
  `--section "1.2.1 Examples"` lands in 1.1.1.
- **What the spec says:** the raw-title match is tried first.
- **Fix:** search in two passes:
  1. raw matches, keeping the existing preference for the first non-H1 heading, then H1;
  2. only if nothing matched, identity matches with the same preference.
- Notes that are not articles keep their current behavior.
- **Tests:** a placement unit test in `src/native/capture/tests/placement.rs` and a CLI
  test in `tests/cli/capture/sections.rs`.

### A11. A lazy continuation becomes a Setext heading (`scan.rs`)

- **Bug:** in `- foo`, then `bar`, then `---`, the line `bar` is a lazy continuation of
  the list item. bob turns it into `## 1. bar`, which changes how the note renders.
- **Fix:** do not treat `---` or `===` as a Setext underline when the paragraph above it
  continues a list item with no blank line between them.
- **Tests:** add a fixture case.

### A12. A frontmatter-only article gains a blank line

- **Bug:** an article that holds only frontmatter gains a trailing blank line, against
  F8.
- **Fix:** the output must be the frontmatter followed by exactly one newline.
- **Tests:** add a unit test.

### A13. Human output polish (`render.rs`)

- `print_findings` prints every finding with the warning prefix. In `format` and `adopt`
  human output, print warnings with the warning prefix and info findings with a distinct
  dim or info prefix.
- The list header shows `$HOME` as `~` (`Articles · 1 · ~/bob`).

### A14. Reuse instead of duplicates (spec §5)

- `scan.rs` `closed_frontmatter_end` duplicates
  `markdown::strictly_closed_frontmatter_end`. Use the shared helper if their semantics
  match. If they differ, say why in a comment.
- Inside `src/native/articles/`, consolidate the duplicates into one helper each where
  the behavior is identical:
  - `is_escaped` (in `marker.rs` and `links.rs`);
  - inline masking (`marker::mask_inline_spans` and `links::mask_inline_ranges`);
  - whitespace `normalize` (in `findings.rs` and `links.rs`).
- These changes must not change behavior.

### A15. `docs/articles.md` is not the full contract

At 107 lines, the doc lacks most of the contract. Make it the full contract, accurate to
the code after A1–A14:

- the marker and its options (`title`, `section_numbers`) and the refusal table;
- the rules N1–N9 and F1–F11;
- the 10-code findings table, with severities;
- heading identity and the three-step resolution;
- the link forms that are covered and those left untouched;
- the word invariant, and the errors `words-changed` and `not-idempotent`;
- write safety: preimage checks, per-note skips, and inbound write order;
- the CLI and its exit codes, and the JSON contract;
- the `1\.5` escape and the "1.5 million users" known risk.

### Accepted deviations (leave as is)

- `articles` is registered as a runner leaf whose default behavior is `list`, not as a
  `Group`. The UX is identical.
- Outline indentation style.
- `replaced[].number` is `""` when numbering is off. The contract only grows.
- N6 `\d` also matches non-ASCII digits.
- `section_numbers` accepts lowercase booleans only.
- Inbound linker writes are not re-run through the word invariant. They only touch
  number tokens and are preimage-checked.
- `src/native/projects/scan.rs` is at about 1,012 lines.

## Part B — bob-plugins `bob-articles` plugin

Open the bob-plugins linked repo with `/sase_repo` and read its `AGENTS.md`. Work in
`plugins/bob-articles/src/*.js` and `styles.css`, with tests in
`scripts/test-bob-articles.cjs`.

### B1. Save-path formats are dropped (`020-article-core.js` `articleBuildChanges`)

- **Bug:**
  - A hunk that deletes through the last line of a note ending in a newline leaves one
    extra newline. Reproduce it with a note ending `Body.\n\n\n\n`: bob returns
    `{"line":8,"delete":3,"insert":[]}`.
  - The self-check in `articleFormatView` (`030-plugin.js`) then throws, and both the
    save-path format and Convert silently do nothing.
- **Fix:** when `insert` is empty, the run ends at the last line, the file has a
  trailing newline, and `index > 0`, also remove the separator before the deleted run.
  Check the analogous cases too: replacing through the last line, deleting every line,
  and CRLF.
- **Tests:**
  - assert
    `articleApplyChangesToText(sent, articleBuildChanges(sent, edits)) === articleApplyEditsToText(sent, edits)`
    for each trailing-newline variant, plus a small seeded random-edit loop;
  - actually assert the cursor-mapping case; today the test sets it up but never asserts
    it.

### B2. Leaving the note ignores the undo guard (`030-plugin.js`)

- **Bug:** `articleQueueLeave` and `articleFormatSavedFile` never check
  `articleUndoBlocked`. So `:w`, then Ctrl+Z, then switching notes re-formats the note
  through `vault.process`.
- **Fix:**
  - Skip leave-formatting while the path is undo-blocked.
  - Do not promise "Becomes X on save" in stale-number tooltips while saves are blocked.
- **Tests:** cover this path in the state-machine test.

### B3. Reading view misses numbers on first open (`035-plugin-look.js`)

- **Bug:** the post-processor wraps numbers only when an outline is already cached. The
  outline arrives asynchronously, and `articleRememberPlan` / `articleRefreshSecnoViews`
  refresh only Live Preview.
- **Fix:** when a new outline arrives for a file, re-render that file's open Reading
  views once (for example `previewMode.rerender(true)`). Guard against render loops.

### B4. CSS that can break Obsidian (`styles.css`)

- Remove `overflow: visible` from `.bob-article .markdown-preview-view`. That element is
  the Reading-view scroller. If hanging numbers need it, scope the overflow to the sizer
  or the headings instead.
- Replace `margin-block` on the Live Preview heading lines (`.HyperMD-header-N`) with
  padding. CodeMirror 6 cannot measure line margins, so margins cause click and cursor
  offsets.

### B5. Child process robustness (`030-plugin.js` `articleRunPlan`)

- Decode stdout as UTF-8 across chunks (`setEncoding("utf8")`, or concatenate the
  Buffers and decode once).
- Attach an `error` handler to `child.stdin`, so an EPIPE from an early bob exit becomes
  a run failure.
- When an old `bob` has no `articles` command and its output is not JSON, show the same
  single quiet per-session notice used for a missing bob, and suggest updating bob.

### B6. Convert refusals are shown only once per session

"Bob: Convert note to article…" must show bob's refusal message every time the command
runs. The once-per-session rule is only for save-path refusals.

### B7. Finish the plugin work

1. Bump the plugin version in `manifest.json` following the repo's convention.
2. Run `npm run build`.
3. Run `node --test scripts/test-bob-articles.cjs`, `npm run validate`, and `npm test`.
   Only the known failures may remain.

## Part C — Deploy and verify

Do the steps in this order:

1. In bob-cli, run `just check`. It must pass except for the two bob-cli-87
   note_snapshot tests.
2. Run the corpus test against `~/bob`, which only reads it:
   `BOB_ARTICLES_CORPUS=$HOME/bob cargo test --lib articles -- --ignored`. It must
   report zero `words-changed` or `not-idempotent` failures.
3. Run `just install`, so the installed `bob` (which the plugin calls) has the fixes.
4. Run these against `~/bob`:
   - `bob articles format -a -c` exits 0;
   - `bob articles` lists `sase_blog_contents` as `✓ formatted`, now with four findings
     (A6 adds `[b]`);
   - a re-run of `bob articles format sase_blog_contents` is silent.
5. Deploy the plugin with
   `bob plugins sync --no-pull -p bob-articles --repo <opened bob-plugins path>`, then
   confirm the deployed files match the repo.
6. The Obsidian visual pass cannot be done by an agent. That pass covers light and dark
   themes, a narrow pane, 200% zoom, PDF export, and a live `:w` renumber undone with
   one `u`. Say so in the close note and in the final response, and tell Bryan to reload
   Obsidian.

## Part D — Close out epic bob-cli-8g (the final step, in the same turn as the code)

1. Run `sase bead epic-symbols bob-cli-8g`. For each listed `--epic-symbol` entry,
   either resolve the symbol (wire it up, privatize it, add a non-test pragma, or delete
   it), or re-key the Justfile line to a still-open bead that needs it. The landing
   agent saw no entries, but recheck after your changes.
2. Run `sase bead close bob-cli-8g --note "<verification>"`. The note must state:
   - **Phase verification:** the landing agent verified all nine phases. Integration
     review of the commits since the epic started (the note-snapshot, capture-templates,
     and ref commits through a4c9ecf5) found nothing that should adopt or that conflicts
     with the articles feature.
   - **Follow-up triage:** the outcomes from the epic's "LANDING FOLLOW-UP TRIAGE" note:
     - note-snapshot failures recorded on active epic bob-cli-87;
     - +1 on bob-cli-6v, bob-cli-6w, bob-cli-7i, and bob-cli-3w;
     - capture-templates and rustfmt proposals declined as already fixed on master.
   - **Fixes:** the A and B fixes this tale made.
   - **Results:** the `just check`, corpus, `npm test`, and validate results, and the
     `format -a -c` exit 0.
   - **Manual acceptance:** the Obsidian visual pass is left for Bryan.

   If the close is rejected for leftover epic symbols, finish that cleanup and close
   again. Never use `--force` merely to make the close succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path shown by
   `sase bead read bob-cli-8g`, which is `plan:202610/article_notes.md` in the plans
   repo.
5. bob-cli-8g has no `parent_bead`, so the landing ends here.
