---
tier: tale
size: medium
title: Finish and land bulk capture templates (bob-cli-7w)
goal: "Template drafts behave the same in capture-parse, completion, execution, and Bob
  Mac Capture. Ctrl-J inserts a plain newline in a template draft. Epic bob-cli-7w is
  closed.

  "
proposed_by: bbugyi200.apollo.bob-cli-7w.land
bead: bob-cli-7w
status: done
---

- **PARENT:**
  [202610/bulk_capture_templates.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bulk_capture_templates.md)
- **BEAD:**
  [bob-cli-7w](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7w/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-7w.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-7w.land.md)
- **COMMITS:**
  - [de43948](https://github.com/bobs-org/bob-mac-capture/commit/de43948cbc3f3c48774ecfd3735674f11ab82102)
    — feat(mac-capture): template landing B1-B6

# Plan: Finish and land bulk capture templates (bob-cli-7w)

## Why

Epic **bob-cli-7w** (plan `plan:202610/bulk_capture_templates.md`) shipped `@@template:`
bulk templates across five phases. Every phase bead is closed. The bob-cli commits are
946c9b7d, 8f0baa76, 84085621, and 3f2b775a; the Mac commit is bob-mac-capture e959476.
The land agent verified bob-cli master with `just check`, which passes, then probed the
built `bob` and the Mac code. That review found:

- the epic's own note, which is still unaddressed;
- a failing Mac CI test;
- a few places where the editor protocol disagrees with `bob capture`, which the epic
  was written to prevent.

This tale fixes those issues and then closes the epic. It has no land agent of its own,
so its last step **is** the epic closeout.

Follow-up triage is already done and recorded on bob-cli-7w; do not redo it:

- memory task bob-cli-82 (declined decision record);
- feature task bob-cli-83 (Mac "Expand template" action);
- +1 on the pre-existing Mac CI failures bob-cli-7p and bob-cli-7h.

The decisions stay as the epic set them:

- `mac_expand_action = no`: do **not** add an Expand action;
- `memory_template_decision = no`: do **not** edit `sase/memory/decisions`;
- no memory note may be edited.

Repro shorthand used below: `$BOB` is `$CARGO_TARGET_DIR/debug/bob` after `cargo build`.
Run probes from `/tmp`. Never run a non-`--dry-run` capture against the real vault.

## Part A — bob-cli fixes (this repo)

### A1. capture-parse must refuse session suffixes on markers

**Bug.** `refusal_diagnostic` (`src/native/capture_parse.rs`, around line 1402) only
treats the whole-item modes as session forms: `PomodoroAdjust`, `PomodoroShift`,
`PomodoroClose`, and `PomodoroStart`. Execution
(`capture_language/draft.rs::is_template_session_kind`) also refuses a `start` or
`close` spec on `Pomodoro` / `PomodoroLink` items. So these drafts get **no**
capture-parse diagnostic, while `bob capture --dry-run` exits 2 with
`template_session_operator`:

- `@@template:\nTask {0} @job:{1}=3\n\na b\n`
- `@@template:\n^job:{0}=x\n\na\n`
- `@@template:\n^job:{0}=3\n\na\n`
- `@@template:\n{0}\n\n^job:a=x\n` (the row value supplies the suffix)

**Fix.** Make the parse-side check use exactly the same session predicate as execution:

- also flag an editor item, or the top-level parse, whose `pomodoro_start` or
  `pomodoro_close` spec is present on a `pomodoro_task` / `pomodoro_link` mode;
- range the diagnostic over the `pomodoro_start` / `pomodoro_close` span when there is
  one;
- map row-inserted suffixes through the existing per-instance source map, and
  deduplicate as today.

**Tests.** Add the four drafts to `tests/cli/capture/parse_template.rs`, asserting the
`template_session_operator` code and a typed range. Where practical, assert that
`bob capture --dry-run` refuses the same drafts (`tests/cli/capture/template.rs`), so
the parse and execution sides agree.

### A2. capture-parse must not refuse the `URL %` listen marker

**Bug.** The draft `@@template:\n{0} %\n\nhttps://example.com/a\n` gives capture-parse
`template_clipboard` at `[16,17]`. The cause is the template-level refusal check on the
masked copy (`capture_parse.rs`, around lines 967–974): `xxx %` reads as a clipboard
marker. But `bob capture --dry-run` (with or without `-n`) accepts the draft and queues
a Listen reference, and the epic plan says `URL %` stays allowed.

**Fix.** Instance parses are the source of truth, so make capture-parse's
`template_clipboard` follow them:

- report `template_clipboard` only from the expanded instances, mapped and deduplicated
  as today;
- drop the masked-template **clipboard** check, or keep it only when no row expanded;
- if you keep it for the no-rows state, it must not flag a template that is just
  `{N} %`.

Keep the masked-template **session** check (A1) as it is.

**Tests:**

- `{0} %` with a URL row gives no `template_clipboard`;
- `{0} task %` with a text row still gives `template_clipboard` on the template's `%`,
  reported once across several rows;
- a `%` typed in a row value is still refused.

### A3. Offer `template:` only on the first nonblank line

**Bug.** `partial_template_header` (`src/native/capture_complete/engine.rs`, around
lines 485–505) compares the cursor line's **text** with the first nonblank line's text,
not its position. So `capture-complete --cursor 10 $'@@te\n\n@@te'` (cursor on line 3)
offers `template:`.

**Fix.** Compare by position instead:

- the cursor's line must be the first nonblank physical line;
- compare byte offsets or line indices, still splitting on `\n`, `\r`, and CRLF.

**Test.** Extend `src/native/capture_complete/tests/routes_tasks.rs`: `@@te\n\n@@te`
with the cursor in the second `@@te` does not offer `template:`, and the cursor in the
first still does.

### A4. Use the full `template:` candidate label

**Bug.** The JSON contract specifies the label
`"Bulk template: one capture per row below"`, but the candidate's `label` is
`"Bulk template"` (`engine.rs`, around line 430). Bob Mac Capture shows this label as
the row's secondary text.

**Fix.** Change the label to `Bulk template: one capture per row below`, and update the
assertions in bob-cli tests and docs that pin it. B6 updates the Mac copies.

### A5. Use the row prefix for `@@` in a row

**Bug.** A `@@` token inside a **row** reports
`template line 4: a template draft can't declare …`. Row-scoped errors must use
`template row K (line L): …`. The cause is in `src/native/capture_language/template.rs`:
the `@@` check is called for rows around line 385 and builds the message around
line 906.

Probe: `printf '@@template:\nTask {0} {1}\n\na @@job\n' | $BOB capture -E` must print
`template row 1 (line 4): a template draft can't declare …`.

**Fix.** Give the global-declaration issue row scope when its token is in a row. Keep
`template line L:` for tokens in the template block, and keep the code and range
unchanged.

**Test.** Unit test in `src/native/capture_language/tests/template.rs`.

### A6. Execution reports the first issue in draft order

**Bug.** The epic plan says execution reports the first issue in draft order, but:

- `src/native/capture/sections.rs` (around line 42) runs `template::expand`, which
  covers every language check, before any per-row parse;
- `capture_language/draft.rs::parse_capture_draft_with_clip_control` (around line 628)
  does the same.

So for `@@template:\n{0}{1}\n\n= ""\na\n`, row 1 (instance `=`, a session start) should
fail first, but bob reports row 2's `template_row_arity`.

**Fix:**

1. When `expand` fails on a **row-scoped** language issue at row K, first expand and
   parse rows 1..K-1. Use the analysis's lexed rows, for example a small `template.rs`
   helper that expands only the valid leading rows.
2. Report the earliest row error.
3. Template-scoped issues (header, template block, placeholders) and
   `template_too_large` still come first, unchanged.
4. In the same area, stop deriving `template_clipboard` / `template_session_operator` in
   `sections.rs` by searching message text. Carry the code with the error from
   `parse_template_expansion`, for example a small error struct or a `(code, message)`
   pair. Keep every message byte-identical.

**Tests** in `tests/cli/capture/template.rs`:

- the probe above reports
  `template row 1 (line 4): a template can't drive the Pomodoro session…` with JSON
  `code` `template_session_operator`;
- a draft whose only problem is row 2's arity still reports arity.

### A7. Help text and dead-code cleanup

- **Help.** `bob capture --help`'s template paragraph (`src/native/capture/cli.rs`) does
  not say that every row needs exactly N fields. Add a short clause, for example: "each
  row must supply exactly one value per placeholder index". Check whether the long help
  is pinned by a fixture under `tests/fixtures/help/`, and update it if so.
- **Dead code.** Remove the stale `#[allow(dead_code)]` attributes that are now wrong:
  - `src/native/capture_language/template.rs:33`, with its "Consumed by the
    template-aware editor protocol phase" comment; `range()` is used by
    `capture_parse.rs`;
  - `src/native/capture_language/model.rs:646`, on `ParsedCaptureItem.template_row`,
    which `capture/plan.rs` reads.

  `just check` (`-D unused`) must still pass.

### A8. Docs

Update `docs/capture.md` wherever A1–A7 change documented behavior:

- § Bulk templates: the error-ordering sentence and the `URL %` note;
- § `bob capture-complete`: the `template:` label and first-line rule.

## Part B — Bob Mac Capture fixes (linked repo)

Open the repo with `sase repo open bob-mac-capture -r "<reason>"` and use only the path
it prints. Read its `AGENTS.md` first. Its rules are binding: never land Swift you have
not compiled.

### B1. Ctrl-J inserts a plain newline in template drafts (the epic's note #1)

Bryan's note on bob-cli-7w: _"make sure that the <ctrl+j> keymap never auto-inserts a
dash when the template syntax is being used (it should just insert a newline)."_

**How Ctrl-J works today.** `CaptureKeyCommandRouter.swift` (around lines 167–170) maps
Control-J to `.insertBulletNewline`. `CapturePanelController.perform` calls
`insertBulletNewlineInEditableTextView(firstResponder:model:)` (around line 751). That
function asks `CaptureBulletNewlineEditResolver.resolve(in:selectedRange:)` (around
lines 40–203 of the same file) for an edit. The resolver always produces a `- ` row or a
bullet-prefix removal.

**Fix.** Decide "template draft" from **bob's parse only**: the app never decides
template syntax (`decisions:mac-capture-is-a-thin-client`). Use the existing
`CapturePanelModel.parsedTemplate` (around line 197, set in `applyParse`). When it is
non-nil, Ctrl-J replaces the current selection with exactly one `\n`, puts the caret
after it, and inserts no indentation and no `-`. This applies everywhere in the draft:
header, template block, and rows.

- Apply the edit through the same `textView.insertText(_:replacementRange:)` +
  `setSelectedRange` path, so undo, IME, and accessibility stay AppKit-owned.
- Keep `model.dismissCompletion()`.
- Leave the picker/list Ctrl-J (next row) unchanged.
- Implement it as a pure, unit-testable branch. For example, add a `plainNewline`
  parameter or a sibling static resolver, and have the controller pass
  `model.parsedTemplate != nil`.
- `parsedTemplate` is the latest **applied** parse, so for about 50 ms plus bob's run
  time after an edit it can describe the previous draft. Accept that. Do not add a local
  `@@template:` check.

**Tests** in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, next to the existing
resolver tests (around lines 1782–2172, helper `applyBulletEdit`):

- template mode inserts only `\n`:
  - at the end of `@@template:`;
  - at the end of a template line;
  - on a `- ` child-bullet line;
  - on an empty `- ` placeholder row;
  - with a non-empty selection;
- non-template behavior is unchanged.

**README.** Update the Ctrl-J keyboard-table row (around line 1643) and the Ctrl-J prose
(around lines 1899–1907) to say that in a template draft, as reported by bob's parse,
Ctrl-J inserts a plain newline.

### B2. Fix the failing CI test

**Bug.** `Tests/BobMacCaptureTests/CaptureTemplatePresentationTests.swift:25`
(`testTemplateRowNeedIsQuietAndUsesBobsDiagnostic`) sets the draft to `"@@template:\n"`.
But `Tests/Fixtures/fake-bob` (around line 118) only matches
`template_header_draft='@@template:'`, so the test fails in macOS CI run 38807458658
(assertions at :30 and :134).

**Fix.** The header-only fixture was recorded with input `@@template:`. Make the test
type `"@@template:"`, or teach fake-bob to match both forms; choose whichever keeps the
fixture honest.

### B3. Fix the one-row fake-bob key

`Tests/Fixtures/fake-bob` (around line 117) has
`template_one_row_draft=$'Task {0}\n\nvalue'`, with no header. But
`capture-template-one-row.json` reports header line 1 and row line 4, so the recorded
draft was `$'@@template:\nTask {0}\n\nvalue'`. Fix the key. If you can, add a
model-level test that submits or previews that draft through fake-bob and sees the
one-row `template_row`.

### B4. Make the `template:` header candidate reachable

**Bug.** For a draft such as `@@te`, `scheduleAnalysis` tries
`cachedRouteCompletion(parse:cursor:draft:)` first (`CapturePanelModel.swift`, around
line 6494; implementation around 7017). That builds the list from the cached targets,
matched by substring, and returns without asking bob. So bob's `kind: "template"`
candidate never appears whenever any target matches, and it never appears for a bare
`@@`.

**Fix.** Skip the cache when the route span under the cursor is a `global_route` span
(the `@@` form), so `@@` route completion always goes through `bob capture-complete`.
This is a filter on span kinds bob reported, not a grammar decision. The other route
span kinds keep the cache.

**Test.** Fake-bob already returns `capture-complete-template-header.json` for the draft
`@@te` (around line 1790). Add a model test that:

- primes the target cache (look at how the existing cached-route tests prime it);
- types `@@te`;
- asserts that the completion list's first candidate has `kind == "template"`.

### B5. Calm header-only state, and "1 capture"

- **No red underline for the unfinished state.** The template-diagnostic underline loop
  in `applyParse` (`CapturePanelModel.swift`, around lines 6911–6923) underlines every
  non-info diagnostic. In the header-only state that includes `template_missing_rows`
  over `@@template:`, while the status line is deliberately calm. Skip
  `template_missing_rows` in that loop: it is the calm incomplete state, not a problem.
  Real template errors keep their underline. Extend the underline test to assert that
  the header-only fixture produces no underline.
- **Singular count.** `PreviewPane.previewContent` (`CapturePanelView.swift`, around
  line 1882) now shows the header for a one-row template, but it reads "1 captures" with
  the accessibility label "1 capture items". Use the singular form for count 1, "1
  capture" and "1 capture item", and keep the plural otherwise.
- **Doc comment.** Mention `template_row` in the `pickerNeed(in:)` precedence comment
  (around line 4568).

### B6. Mac label fixture

After A4, re-record `Tests/Fixtures/capture-complete-template-header.json` with the new
bob, using a scrubbed vault or editing only the template candidate's `label`. Update
`Tests/CaptureCoreTests/CompletionRowContentTests.swift` (around lines 44 and 54) and
any README text that quotes the label.

### B7. Mac verification (required by `AGENTS.md`)

- Run `just portable-check`. If `swift` is missing on this host, run it on athena; read
  `sase memory read tailnet.md -r "<why>"` first for access.
- B1, B4, B5, and the B2/B3 test edits are in `Sources/BobMacCapture*` /
  `Tests/BobMacCapture*`, which portable-check cannot type-check. Compile a scratch copy
  on the mac as `AGENTS.md` describes:
  1. rsync without `.build` / `.git`;
  2. run `bundle.sh`, then `just typecheck-tests`;
  3. delete the copy.

  Never use Bryan's checkout or `just install`. If `typecheck-tests` fails because it
  cannot load the stdlib on the CLT-only Mac (known issue bob-cli-7o), report it. Do not
  hide it.

- After the commit is pushed, macOS CI is expected to still show the pre-existing
  failures bob-cli-7p and bob-cli-7h. The template tests must not fail. If the mac or CI
  cannot be checked, say in the final report that the macOS targets are unverified.

## Part C — Verification

- In bob-cli: `cargo fmt`, then `just check` must pass. Do **not** run
  `just check-full`.
- Re-run every repro in A1–A6 against the rebuilt `bob` and confirm each new behavior.

## Part D — Close epic bob-cli-7w (final step; do it in this same turn)

Closing in the same turn as the final code is the normal landing. Do not wait for a
commit, a push, or CI before closing.

1. Run `sase bead epic-symbols bob-cli-7w`. The land agent saw none. If any entry
   appears now, resolve it: wire it up, privatize it, add a non-test pragma, or delete
   it per the Symvision epic-whitelist policy. Re-key it only if a still-open bead needs
   the exemption.
2. Close the epic with a verification note, for example:
   `sase bead close bob-cli-7w --note "Verified phases .1–.5 against plan:202610/bulk_capture_templates.md and fixed the landing gaps: …(list A1–A8 and B1–B6)… just check passes; Mac portable-check <result>; mac typecheck <result or unverified>. Follow-ups: bob-cli-82 (decision record), bob-cli-83 (Expand action); pre-existing Mac CI failures +1 on bob-cli-7p and bob-cli-7h. No integration changes were needed: the concurrent commits are ref-library work that does not touch capture."`
   - If the close is rejected for leftover `--epic-symbol` entries, clean them up and
     close again.
   - Never use `--force` merely to make the close succeed.
3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of
   `sase/repos/plans/202610/bulk_capture_templates.md` (the epic's PLAN path from
   `sase bead read bob-cli-7w`). It currently reads `status: wip`.
5. bob-cli-7w has no `parent_bead`, so nothing further needs closing.

## Out of scope

- The Mac "Expand template" action (bob-cli-83).
- The decisions strand (bob-cli-82).
- Pre-existing Refs CI failures (bob-cli-7p, bob-cli-7h).
- The `bob-mac-capture` typecheck SDK bug (bob-cli-7o).
- Unifying `capture_complete/shell.rs`'s header check with the engine gate. Shell
  completion already returns `None` for any template draft and never offers `template:`,
  so behavior is identical, and its `unreachable!` arm is correct.
- The template item's masked `block_id` in capture-parse (for example `apply-xxx`). The
  epic plan specifies that it comes from the masked parse, and no client uses it.
