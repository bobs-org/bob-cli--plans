---
tier: tale
title: One-line rows for one-placeholder capture templates
goal:
  A `{0}`-only `@@template:` accepts several space-separated values per row line, each
  its own capture.
size: medium
decisions:
  url_listen_rows:
    ask:
      How should a one-placeholder row keep a `URL %` listen marker on its URL now that
      rows split on spaces?
    choices:
      quote: Quote `"URL %"` or put `%` in the template; every row shares one lexer
      glue:
        A lone `%` or `@` after a value joins it, so `URL %` rows work unquoted; adds a
        lexer rule
    default: quote
    why:
      One lexer for every row is easier to predict; `{0} %` covers the all-listen case
    answer: quote
  bulk_capture_glossary:
    ask:
      Should the Bulk Capture glossary say a `{0}`-only template takes one capture per
      value?
    memory:
      - glossary:bulk-capture
    default: true
    corrects:
      Template rows are later nonblank physical lines, one capture per line, not
      blank-line-separated items
    answer: true
proposed_by: bbugyi200.apollo.83
decided_by: auto
create_time: 2026-10-16 13:18:19
status: wip
---

# One-line rows for one-placeholder capture templates

## Goal

When a `@@template:` block uses only one placeholder (`{0}`, possibly repeated), a row
line can hold several values separated by spaces or tabs. Each value becomes its own
capture, exactly as if each value were on its own line. This draft:

```text
@@template:
foo: {0}! @sase:foo-{0}

bar baz bam
```

must behave exactly like this one, byte for byte in `bob capture -E`, and identically in
`bob capture`, `--dry-run`, `capture-parse`, and the saved-template shelf:

```text
@@template:
foo: {0}! @sase:foo-{0}

bar
baz
bam
```

## Current behavior (what changes)

- `src/native/capture_language/template.rs` `lex_row` returns the whole trimmed line as
  one value when `field_count == 1`. Multi-placeholder rows go through the general
  lexer, which splits on spaces and tabs, supports `"..."` / `“...”` quoting, and treats
  `\"` and `\\` as the only escapes.
- `analyze` builds one `TemplateRow` per nonblank physical line after the separator.
- `docs/capture.md` (Bulk templates table) says: "One placeholder | The whole trimmed
  row is its value, including spaces". The unit test
  `one_field_rows_keep_spaces_and_template_line_endings_normalize` in
  `src/native/capture_language/tests/template.rs` pins that behavior.

## New semantics

1. **Single-placeholder detection:** `field_count == 1` (the only placeholder index is
   `{0}`). Multi-placeholder templates do not change: one row per line, and arity is
   still enforced.
2. **Lexing:** Every one-placeholder row line goes through the same lexer as
   multi-placeholder rows. Values split on runs of spaces and tabs. A value that
   contains spaces must be quoted (`"Two Sigma"` or `“Two Sigma”`), with the same
   escapes and the same rule that a closing quote must be followed by a separator or the
   end of the line.
3. **Each value is its own row.** One `TemplateRow` per value, with:
   - `row`: a sequential number across the whole draft, left to right then top to
     bottom.
   - `line`: the physical line, which rows on the same line share.
   - `range`: the value's own source range, including quotes when quoted.
   - `fields`: exactly one `TemplateField { field: 0, .. }` with the same range.

   So `row_count`, `template_row {row, line}`, capture-parse `items`, and
   `template.rows` all count values. Keep "row" as the unit in JSON keys, issue codes,
   and messages: a row is one capture. No wire shape changes.

4. **Line-level checks still run per physical line, before splitting:**
   - The `template_row_bullet` check (a line starting with `- `, `* `, or `+ `) still
     produces one row for the whole line, as today.
   - Blank lines are still ignored.
5. **Issue row numbers name the value's row:**
   - `template_unterminated_quote` reports the row number of the unterminated value (for
     example `a "b c` on line 4 gives "template row 2 (line 4): quoted field has no
     closing quote"), and its range runs from the quote to the end of the line. Values
     before it on the line are still valid rows. They keep expanding in
     `expand_rows_for_editor`, because the existing overlap check only drops rows whose
     range intersects the issue.
   - The row-scoped `template_global_declaration` check (`check_global_tokens_in_row`)
     names the row whose value range contains the `@@` token's start, falling back to
     the line's first row. Example: `a @@route` gives "template row 2 (line 4)".
   - `MAX_ROWS` counts values, not lines. The `template_too_large` row issue fires on
     the 1,001st value, with that value's range, even when it sits mid-line.
6. **`template_row_arity`:** Unreachable for one-placeholder templates, because each row
   has exactly one field. No special case is needed beyond not emitting it.
7. **Missing-rows message for one field:** Change "add one row per capture: one value
   per row" to "add one value per capture, separated by spaces or new lines". Leave the
   0-field and N-field messages unchanged; the Mac app fixtures pin the 2-value one.

> [!decision] url_listen_rows = quote No lexer special case. Today's `{0}` +
> `https://example.com/one %` row becomes two rows: a plain ref, then `%`, which refuses
> as `template_clipboard` naming row 2. Update the two tests that use that form to quote
> the row (`"https://example.com/one %"`). Add an assertion that the unquoted form now
> refuses with `template_clipboard` and "template row 2 (line 4)". Document both
> spellings: the quoted row, and `{0} %` in the template.

> [!decision] url_listen_rows = glue In the one-placeholder split only, a lone unquoted
> `%` or `@` token directly after a value (separated only by spaces or tabs) extends
> that value's value and range to include it. `https://example.com/one %` stays one row,
> and the existing tests keep passing unchanged. A lone `%` / `@` at the start of a line
> is still its own value. Document the rule in the Bulk templates table.

## Implementation steps

### 1. `src/native/capture_language/template.rs`

- Remove the `field_count == 1` whole-row early return in `lex_row` so that every row
  uses the general lexer. Let the caller build the unterminated-quote message, or pass
  enough context for it to name the right row number. Keep the multi-placeholder message
  text unchanged: it still says "template row N (line L)" for the whole line's row.
- Restructure the row loop in `analyze`:
  - Per nonblank line, keep the existing `leading` / `trailing` trim and the bullet
    check.
  - For `field_count == 1`, lex the line and push one `TemplateRow` per lexed field,
    incrementing the row number per value.
  - Otherwise keep one row per line, as today.
  - Apply the `MAX_ROWS + 1` issue per row pushed, not per line.
  - Run `check_global_tokens_in_row` once per line, attributing each token to the
    containing row as described above. This may mean passing the line's rows, or
    returning token ranges for the caller to attribute.
- The `masked()` path blanks by `row.line`, and blanking a line more than once is
  harmless. `build_expansion`, `build_instance`, `row_number_for_issue`,
  `leading_instances`, and `expand_rows_for_editor` need no structural change. Verify
  they behave with several rows per line; the `row_range` and `source_map` field ranges
  are already per value.
- Update the `template_missing_rows` one-field message (semantics item 7).
- If `url_listen_rows = glue`, implement the glue in the one-placeholder split only.

### 2. `src/native/capture_complete/engine.rs` (`template_completion_gate`)

- Today a cursor anywhere inside a whole-line row range returns an empty completion.
  With per-value ranges, a cursor in the gap between two values (for example inside a
  double space) would fall through to masked completion.
- Keep suppression line-wide: treat the cursor as on a row when it falls between the
  first row's start and the last row's end among rows sharing the same `line`. A small
  grouping over `analysis.rows` is enough; add a helper to `template.rs` if it reads
  better.

### 3. User-facing text

- **`src/native/capture/cli.rs`:** In the `bob capture` long help (the "A draft
  beginning with `@@template:` …" paragraph), replace "quote multi-word fields in
  templates with multiple placeholders" with wording that says:
  - fields containing spaces are quoted;
  - a template that uses only `{0}` takes one capture per value, so `bar baz bam` on one
    line is three captures.
- **`docs/capture.md` › Bulk templates:**
  - Change "Each nonblank row supplies one capture" so it says a one-placeholder
    template takes one capture per space-separated value, so values may share a line or
    sit one per line.
  - Replace the "One placeholder" table row with the new rule: split on spaces and tabs,
    quote values that contain spaces, and each value is its own row, numbered left to
    right.
  - Merge the "Multiple placeholders" row with it, or keep the two rows distinct, so
    that the quoting rule is stated once.
  - Add a short one-placeholder example: the `foo: {0}! @sase:foo-{0}` / `bar baz bam`
    draft and its three-item expansion.
  - Update the `template_missing_rows` message cell (`: one value per row` → the new
    text).
  - Document the `URL %` spelling per the `url_listen_rows` decision. The existing
    sentence "`URL %` remains allowed when it is the reference listen marker" must stay
    true; say how to write it in a one-placeholder row.
- **`docs/capture.md` › Template drafts (capture-parse):** Add that rows of a
  one-placeholder template can share a `line`, and that each row's `range` is its
  value's range.
- **`README.md`:** In the capture section ("after a blank line, each row supplies one
  capture"), add that a `{0}`-only template takes one capture per space-separated value.

### 4. Memory (`bulk_capture_glossary` decision)

If accepted, use `/sase_memory_write`:

- In the `glossary:bulk-capture` strand, rewrite the sentence "Template rows are later
  nonblank physical lines, one capture per line, not blank-line-separated items" to
  something like: "Template rows are later nonblank physical lines, one capture per line
  (a template using only `{0}` takes one capture per space-separated or quoted value, so
  several can share a line), never blank-line-separated items".
- Keep the rest of the strand unchanged, then run `sase memory init`.

## Tests

Update or replace these tests, which pin the old behavior:

- `src/native/capture_language/tests/template.rs`:
  - Replace `one_field_rows_keep_spaces_and_template_line_endings_normalize` with a test
    that `Montréal 🧭 role` splits into three rows, and that `"Montréal 🧭 role"` and
    `“Montréal 🧭 role”` stay one row whose value keeps its spaces.
  - Keep the `\n` / `\r\n` / `\r` line-ending loop.
  - Update the `missing_template_or_rows_has_a_teaching_issue` one-value message.
- Depending on `url_listen_rows`:
  - `tests/cli/capture/template.rs`
    `template_url_listen_marker_is_not_clipboard_capture`;
  - the "reference listen markers" case in `src/native/capture/tests/template.rs`.

Add these tests:

- **Unit, `src/native/capture_language/tests/template.rs`:**
  - `template::expand` of the user's one-line example equals the three-line draft's
    `expand` text.
  - Rows are numbered 1–3, all on line 4, and each `row.range` / `fields[0].range` is
    the value's own range.
  - Tabs and repeated spaces split cleanly.
  - Mixed `a b\nc` gives rows 1–2 on line 4 and row 3 on line 5.
  - `a "b c` gives `template_unterminated_quote` naming row 2, and row 1 still builds in
    `expand_rows_for_editor`.
  - `a @@route` gives `template_global_declaration` naming "template row 2 (line 4)".
  - 1,001 values on one line give `template_too_large`.
  - A one-placeholder template never reports `template_row_arity`.
  - A multi-placeholder row with extra values still reports `template_row_arity`
    (regression guard).
- **CLI, `tests/cli/capture/template_expand.rs`:** `bob capture -E` on the one-line
  draft prints the same expansion as the three-line draft. With `-f json`, both report
  `template.row_count == 3`.
- **CLI, `tests/cli/capture/template.rs`:** A `--dry-run -f json` capture of the
  one-line draft yields three results whose `template_row` values are
  `{row: 1..3, line: 4}`, and whose bodies match the three-line draft's results.
- **CLI, `tests/cli/capture/parse_template.rs`:** `capture-parse` on the one-line draft
  returns:
  - three `template_row` items with distinct per-value ranges and
    `line_start == line_end == 4`;
  - three `template.rows` entries;
  - one `template_field` span per value (quoted values include their quotes).
- **Completion, `src/native/capture_complete/tests/routes_tasks.rs`:** Extend
  `template_completion_suppresses_header_rows_and_placeholders` with a cursor in the gap
  between two values on a one-placeholder row line (for example
  `"@@template:\nBuy {0} @not-a-route\n\nrow  two"` at the middle of the double space).
  It must return no context and no candidates.

## Non-goals

- Packing several calls of a multi-placeholder template onto one line. Arity stays
  enforced there.
- No new header syntax, option, or JSON field. Saved-template shelf behavior is
  unchanged, because the shelf stores only the definition block.
- No Bob Mac Capture change. The app is a thin client: it renders bob's spans, ranges,
  and `row_count`, and nothing in it assumes one row per line. Its `Template · N rows`
  capsule then counts captures, which is the intended meaning. Mention the semantic
  change in the bob-cli commit message so the app side is aware.

## Verification

Run `just fmt`, `just lint`, and `just test`; all must pass. Then spot-check by hand:

```sh
printf '@@template:\nfoo: {0}! @sase:foo-{0}\n\nbar baz bam\n' | bob capture -E
```

It must print the same text as the three-line draft.
