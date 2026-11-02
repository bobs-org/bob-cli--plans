---
tier: epic
title: Finish capture lists — section names with spaces, header completion, and undoable
  ID splices
goal: 'A list header can name any section, including one whose title has spaces (`@sase#future-work`
  or `@sase#"Future Work"` both reach `## Future Work`). Picking a section from the
  menu always inserts a selector that reaches that section. Route and section completion
  work on a list header even after bullets are typed below it. Accepting a block ID
  in Bob Mac Capture is one native Undo step. With this done, bob-cli-fn can land.

  '
parent_bead: bob-cli-fn
phases:
- id: section-selectors
  title: Section selectors with spaces and header-line completion in bob
  depends_on: []
  size: medium
  description: 'section-selectors: add slug matching to `@route#selector` section
    selectors, make section completion insert a selector that resolves back to the
    picked heading (raw, slug, or quoted literal), decode partial quoted section queries,
    treat an unterminated `#"…` section as incomplete in parse and execution, complete
    route and section tokens on a list header that already has bullets, add the off-clock
    list-vs-batch test, and document it all.

    '
- id: mac-undo
  title: Undoable ID splices and section picks in Bob Mac Capture
  depends_on:
  - section-selectors
  size: medium
  description: 'mac-undo: apply an accepted block-ID splice as one native undoable
    NSTextView edit, insert Bob''s section replacement verbatim (including quoted
    literals), treat the new `section` need as a calm incomplete state, regenerate
    list and section fixtures from the phase-1 bob, and verify on macOS CI.

    '
proposed_by: bbugyi200.apollo.bob-cli-fn.land
proposed_with: claude/opus
create_time: 2026-11-02 07:14:34
status: wip
bead_id: bob-cli-fn.5
---

- **PROMPT:** [prompts/202611/capture_list_section_names.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_list_section_names.md)
- **BEAD:** [bob-cli-fn.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fn/bob-cli-fn.5.md)

# Plan: Finish capture lists — section names with spaces, header completion, undoable ID splices

This child epic finishes the remaining work on epic `bob-cli-fn` ("Capture lists under a
lone @file header", plan `plan:202611/capture_route_lists.md`). Its `parent_bead` link
hands landing back to `bob-cli-fn` once this epic lands. Do not close `bob-cli-fn` or
edit its plan file from a phase of this epic.

Read `bob-cli-fn` and its notes with `sase bead read bob-cli-fn -r "<why>"`. Note #5/#6
is Bryan's report, and "LAND AUDIT 2" is the audit this plan comes from. The parent
epic's DECISIONS are final:

- a column-zero header starts an item;
- project-note bullets share the list ID grammar;
- the Bulk Capture glossary update is done;
- no decisions record is written.

Make no memory edits. Keep grammar 6 and the Mac minimum grammar revision 3. Never edit
`Cargo.toml`'s version or `CHANGELOG.md`.

## Why

Bryan typed `@sase#Future Work` with `- boom` below it in Bob Mac Capture (note #5). He
wanted a list filed under `## Future Work` in `sase.md`. Instead Bob read `@sase#Future`
as the selector and `Work` as the bullet's text, so it captured the bullet `Work` with
child `boom`. Reproduced in a scratch vault at `a8f61375`:

| Draft                            | Today                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------- |
| `@sase#Future Work` / `- boom`   | bullet `Work` with child `boom` in the first `Future…` section                            |
| `@sase#"Future Work"` / `- boom` | list, `boom` lands under `## Future Work` (exact)                                         |
| `@sase#Future` / `- boom`        | list, `boom` lands under `## Future Work` (prefix)                                        |
| `@sase#future-work` / `- boom`   | list, but `boom` silently lands in the pre-heading zeroth section                         |
| `boom @sase#future-work`         | same silent zeroth-section fallback (pre-existing, ordinary capture)                      |
| `capture-complete` on `@sase#Fu` | `section` rows whose `replacement` is the raw title `Future Work`, which splits the token |
| `@sase#Fu` / `- boom`, cursor 8  | `context: null`: no section menu on a header once a bullet exists                         |
| `@sa` / `- boom`, cursor 3       | `context: null`: no route menu either                                                     |
| `@sase#"Fu` / `- boom`           | parse: a section literally named `"Fu`, no needs; execution: "task text is required"      |
| `boom @sase#"Fu`                 | parse: bullet in section `"Fu`; execution: a literal inbox task. They disagree            |

The root cause is the menu, which inserts a title that does not survive lexing. The
zeroth-section fallback for `#future-work` makes it worse: the drawer slug that works
everywhere else in Bob silently misfiles here.

### Design

- **One slug rule.** Drawers (`@foo+bar#future-work` ↔ `FUTURE WORK`) and Pomodoro
  names already use `selector_slug` (`src/native/capture_language/line.rs`): trim,
  collapse whitespace, ASCII-lowercase, join with `-`. Section selectors add it as a
  fallback tier. On the project note's own `^prj` task, `@sase+prj#future-work` already
  reaches `## Future Work`. With this change `@sase#future-work` reaches the same
  heading.
- **The menu inserts what resolves back.** This is the drawer completion's existing rule
  (`#future-work`, or a quoted `#"TITLE"` when the slug does not resolve back).
- **Quoted literals stay exact.** `@sase#"Future Work"` keeps exact, case-insensitive
  title equality and never slug-matches. An unterminated literal is incomplete and never
  a mistake, which is the Pomodoro-name literal rule.

Rejected: spelling a space as `__` (`#Future__Work`), Bryan's first idea in note #6. It
solves the same lexing problem, but it adds a second space spelling used only by section
selectors. Drawer slugs (`@sase+prj#future-work`, which already reaches `## Future Work`
on a project note), Pomodoro names, and the drawer menu's replacements all use hyphens,
so `__` would make the same heading answer to different spellings depending on the
marker. It also costs two Shift keystrokes per space. The hyphen slug plus quoted
literals covers every title.

Rejected: making a whole header line such as `@sase#Future Work` mean the section
"Future Work" when bullets follow. That line is already valid grammar (a section bullet
`Work` with children; "Plan trip @dev + bullets" stays unchanged in the parent plan).
The meaning would also depend on vault contents. The preview already shows the misread,
and the menu fix removes the common path into it.

## Phase `section-selectors` (bob-cli)

Relevant code: `src/native/capture/sections.rs` (`target_bullet_section`,
`heading_matches_bullet_selector`, `raw_heading_matches`, article identity tiers),
`src/native/capture_complete/engine.rs` and `candidates.rs` (`section_candidates`),
`src/native/capture_complete/shell.rs`, `src/native/capture_language/completion.rs`
(item lookup near the top of the completion entry point, `CompletionContext::Section`
near line 2177), `src/native/capture_language/draft.rs` (`detect_list_header`,
`parse_list_header_token`), `src/native/capture_language/pomodoro_literal.rs` (the
quoted-literal lexer), and `editor_parse.rs` / `item.rs` for parse and execution
agreement.

1. **Slug tier in the shared resolver.** In `target_bullet_section`, when a non-exact
   selector matches nothing through the raw-title prefix tier (and the article identity
   tier), try `selector_slug(heading title)`:
   - whole-slug equality with the selector's slug first, then slug-prefix;
   - the same `is_bullet_section` and reserved skips, and the same non-H1-before-H1
     preference, in document order.

   Exact selectors (quoted literals and `--section`) never slug-match. A bare `#` is
   unchanged, and the zeroth-section fallback applies only when every tier fails. This
   one resolver serves `<text> @route#sel`, `@route#sel <text>`, and list bullets, so
   the list ≡ batch equivalence holds automatically.

2. **Round-trip section replacements.** Each `section` candidate's `replacement` must
   resolve back to that candidate's heading through step 1's resolver, for the same
   destination note. Choose the first spelling that does:
   1. The raw title, when it has no whitespace and lexes as one unquoted selector.
      Existing single-word results stay byte-identical.
   2. Otherwise the hyphen slug.
   3. Otherwise a quoted literal `"Title"`, escaping `"` and `\`.

   `title` stays the display title. Candidate filtering also matches a query against the
   slug prefix, so `@sase#future-w` offers `Future Work`. Shell completion must emit a
   shell-safe word for a quoted replacement; follow the existing single-quote rule used
   for Pomodoro literals.

3. **Partial quoted queries.** With a cursor inside `@sase#"Fu` (ordinary item or list
   header), the `section` context decodes the query to `Fu` and offers rows. Its
   `replacement` range covers the whole partial literal, opening quote included, so
   accepting a row yields one complete token. Mirror the Pomodoro-name partial-literal
   behavior documented in `docs/capture.md`.
4. **Unterminated literals are incomplete.** An unterminated `@route#"…` is incomplete
   in `capture-parse` (`needs: ["section"]`, an additive need value, with no diagnostic)
   and is refused by `bob capture` with Bob-authored copy and zero writes. This applies
   to a list header line (`@sase#"Fu` + bullets) and to leading or trailing route tokens
   in ordinary items. It is never a section literally named `"Fu`, and never a literal
   inbox task. Parse and execution must agree. Add the vectors to the existing
   refusal-conformance tests.
5. **Header-line completion.** A cursor on a list header line completes the header token
   exactly as it does on a lone header, whatever follows. That covers route (`@sa|`),
   section (`@sase#Fu|`, `@sase#|`), and partial quoted sections. Today the header line
   belongs to no per-bullet item, so the item lookup returns `None`. Resolve the header
   through the draft's list context (`header_start` / `detect_list_header`) and complete
   against a header-only item. Also confirm that `capture-parse` and `capture-rewrite`
   cursor lookups on a header line resolve to the list's first item, as the parent plan
   requires, and fix them if they do not.
6. **Off-clock list ≡ batch.** This comes from the earlier closeout tale and is still
   missing. Add one dry-run comparison in `tests/cli/capture/lists.rs`:
   - a list containing an off-clock-linked task and another worked task;
   - a separate `=` start item;
   - compared against the equivalent blank-line batch;
   - assert identical written bytes, ledger clock fields, and warnings.

   Use the off-clock vocabulary from `8a9ec685` / `71a65dad`, never ad-hoc strings.

7. **Docs.**
   - `docs/capture.md`:
     - the "Section bullets" paragraph (slug tier, quoted literals, the round-trip
       rule);
     - the Lists section, adding a header example with a spaced section name (both
       spellings) and the unterminated-literal state;
     - the capture-complete `section` context;
     - the capture-parse `section` need.
   - `docs/reference/json-outputs.md`.
   - `capture-complete --help` (`src/native/capture_complete/cli.rs`).

   Run `just docs-gen` and `just docs-check`.

8. **Tests.** Unit and CLI tests for:
   - every tier order (raw prefix beats an earlier slug match; whole slug beats an
     earlier slug-prefix; exact never slugs; article notes; H1 fallback);
   - replacement round-trip, including a title whose slug collides with another heading,
     which forces a quoted literal;
   - quotes and backslashes in titles;
   - partial literals;
   - unterminated-literal parse ≡ execution;
   - header completion with one or more bullets, CRLF, and UTF-8 before the header;
   - the screenshot scenario end to end in a temp vault: complete `@sase#Fu`, apply the
     replacement, add `- boom`, capture, and assert `boom` lands under `## Future Work`.

   Do not change the block-ID suggester. LAND AUDIT 2 recorded the paired calibration
   gate as met at `a8f61375` (first-suggestion Jaccard 0.360290 vs 0.359630 required).
   Any incidental suggester change must re-measure with the ignored
   `block_id_calibration` harness and still pass.

9. **Gate.** Run `just check` and `just docs-check`. Then check by hand in a temp vault
   (`BOB_DIR` pointing at a temp dir, `BOB_NOW` pinned) with `bob capture --dry-run`
   (human and JSON), `capture-parse`, and `capture-complete` over the table in "Why".
   Every row must now land under `## Future Work`, offer the menu, or report the
   incomplete state, in agreement across surfaces. Never touch the live vault.

## Phase `mac-undo` (bob-mac-capture)

Open the linked repo with `/sase_repo` and read its `AGENTS.md` first. Bob stays the
grammar authority, and Swift adds no grammar.

1. **Undoable block-ID accept.** `CapturePanelModel.acceptPickerRow` and the block-ID
   type-through path apply Bob's `block_id.splice` through `setPlainDraft`. That path
   registers no native NSTextView undo step. The parent plan's mac-lists step 1 requires
   Undo to restore the pre-accept draft (fn.4 note #2).
   - Apply the splice as one native undoable edit through the controller-bridge pattern
     (`applyNativeRewriteEdit` / `applyTemplateInsert` in
     `CapturePanelController.swift`).
   - Restore editor focus after the picker closes, and check the exact draft and UTF-8
     boundaries before writing.
   - Keep stale-result rejection, and keep the caret at the end of Bob's written text.
   - Fall back to `replacement` when an older Bob sends no splice.

   Extend `CaptureListPanelTests` with hosted tests that accept through the real model
   and controller, then Undo and Redo. Assert that the draft and caret are restored,
   unrelated typing history survives, and owner announcements are unchanged. Cover the
   same-line, nested, owner-ID replacement, type-through remainder, project-note, and
   older-Bob cases.

2. **Section picks.** Accepting a `section` row must insert Bob's `replacement` verbatim
   (raw title, slug, or quoted literal) over Bob's range. The menu must also open on a
   list header that already has bullets, now that phase 1 returns rows there. Treat the
   new `section` need as a calm incomplete state, like `list_item`, never a flashed
   refusal. Add CaptureCore and hosted tests driven by fixtures from the phase-1 bob.
3. **Fixtures.** Regenerate the list fixtures with `Scripts/generate-list-fixtures.sh`
   from a real bob built at phase 1's commit, plus new section-completion fixtures
   (`@sase#Fu` with and without bullets below; a partial quoted literal). The script's
   self-checks must pass. `CaptureGrammarGate.minimumRevision` stays 3.
4. **Docs.** Add a sentence on spaced section names to `docs/capture-lists.md` and run
   `just docs-check`.
5. **Compile and verify.** Compiling is mandatory.
   - Run `just portable-check` (on athena if this host lacks Swift).
   - Make a best-effort scratch `bundle.sh` plus `just typecheck-tests` on the Mac,
     following `AGENTS.md`. Read `sase memory read tailnet.md -r "<why>"` first, and
     never use Bryan's checkout.
   - Run `Scripts/macos-ci-check` to push a precommit snapshot and wait for macOS CI.

   The new and existing `CaptureListPanelTests` must pass there. Master's Test step is
   already red on hosted tests tracked by `bob-cli-f3` (7) and `bob-cli-fu` (14). Those
   failures are not this phase's to fix, but record the exact run ID and the failing set
   on the phase bead. If the Mac is offline, say the macOS targets were verified only
   through the CI snapshot.

## Landing context for this epic's land agent

After both phases land, verify them, close this epic normally, then resume the
`bob-cli-fn` landing through `parent_bead`.

LAND AUDIT 2 established the state of `bob-cli-fn` as of this plan:

- All four fn phases are closed.
- The calibration gate is met on a paired run.
- `just docs-check` passes.
- `bob-cli-fn` has no `--epic-symbol` entries and no `parent_bead`, and the repo has no
  `symvision` recipe.

Its earlier follow-ups are already triaged in notes #2–#3, and in this audit:
`bob-cli-fu` was filed, and resolution notes were added to `bob-cli-fj` and
`bob-cli-fp`.

The lander fixed the only `just check` failure in the same turn as this plan. That was
the stale `^write-changelog-entry` expectation in `tests/cli/capture/keep_moving.rs`
after the 20-byte first-suggestion cap.

The remaining items for `bob-cli-fn` are exactly this epic's phases. When this epic
closes cleanly, `bob-cli-fn`'s land agent:

1. rereads every descendant;
2. reruns `sase bead epic-symbols bob-cli-fn`;
3. closes `bob-cli-fn` normally with a verification note;
4. runs `just symvision` if one exists;
5. sets `status: done` in `plan:202611/capture_route_lists.md`.
