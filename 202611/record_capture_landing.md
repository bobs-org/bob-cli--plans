---
tier: tale
size: medium
title: Repair record capture contract gaps and finish its landing
goal:
  Complete the remaining grammar and adapter requirements of bob-cli-ir.5.1, then close
  that epic and its parent phase bob-cli-ir.5.
proposed_by: bbugyi200.athena.bob-cli-ir.5.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ir.5.1
status: done
---

- **PARENT:**
  [202611/capture_record_creation.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_record_creation.md)
- **BEAD:**
  [bob-cli-ir.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.5.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-ir.5.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-ir.5.1.land.md)
- **COMMITS:**
  - [903dd39](https://github.com/bobs-org/bob-cli--plans/commit/903dd39e46ee3ab3f21e34f3f97208acf6715a55)
    — docs(plans): mark record capture creation phase done

# Repair record capture contract gaps and finish its landing

This is the remaining work from the landing audit of **bob-cli-ir.5.1**, not a new
record feature. One coding agent can repair these bounded paths in the shared capture
grammar and native adapter, verify them, and finish the landing in the same turn. There
is no later land agent for this tale. Do not wait for this work's commit SHA, push, or
CI before closing the completed epic.

## Context and binding scope

Read `sase bead read bob-cli-ir.5.1 -r "Need landing findings and follow-up outcomes"`
and `sase bead read bob-cli-ir.5 -r "Need parent phase and inherited decisions"`. Read
the approved plans through `sase artifact read`:

- `plan:202611/capture_record_creation.md` is the epic being finished. Its four phases
  are closed; implementation commits are `83ad74e2`, `4c97ed41`, `f2561f96`, and
  `e081e729`.
- `plan:202611/record_notes.md` defines the original capture-records phase and the
  adjacent phases. Its containing epic **bob-cli-ir** belongs to its own land agent.

The audited base is `e081e729`, also fetched `origin/master` at review time. Recheck
post-audit drift before editing, especially siblings adding record completion, append,
or Mac consumers. Preserve their landed contracts. Do not implement those sibling
features here. Existing record creation, missing-folder creation, collision reservation,
no-replace installation, and rollback are already implemented; retain them and their
tests.

Inherited decisions remain final: `seed_records=yes` belongs to seeding phases;
`memory_glossary_capture=yes` is already satisfied; the other memory decisions remain
out of this child landing's scope. No memory edit is needed. Do not edit package
versions or CHANGELOG.md. Preserve schema version 1. The grammar is currently 20; these
are corrections to that approved record contract, not a new syntax feature. Use a final
Conventional Commit subject with `!` (for example
`fix(capture)!: complete record claiming and preview contracts`): the initial record
grammar commit accidentally omitted the required breaking marker. Do not rewrite
existing history or manually commit; declare the message through `/sase_final`.

## Reproductions and implementation

Use the checkout-built binary and isolated temporary vault/config/state, with
`BOB_NOW="2026-10-07 13:40:00"`. A minimal records config is:

```yaml
records:
  restaurant:
    dir: eat
    label: Restaurant
    fields:
      status:
        required: true
        values: { liked: Liked, to_try: To try }
      text: {}
      tags: { list: true }
      rating: { values: number }
```

1. **Keep record intent ahead of task and template parsing.**
   `src/native/capture_language/record.rs` currently examines only the first and last
   whitespace token. `Ramen @eat/ s:1\n- status:: liked` therefore succeeds as a
   scheduled inbox task. `Ramen @eat/^bad_stem\n- status:: liked` goes to generic
   task-block-ID diagnostics. A test named `task_sigils_in_the_tail_keep_task_meaning`
   even asserts fallback for malformed explicit record stems. Honor the approved
   configured-record refusal instead: recognized trailing task markers must not hide a
   record route, and malformed record tails must remain record diagnostics. Keep
   ordinary mid-prose route mentions, escaped/quoted routes, unknown nonexistent
   folders, and explicit forced destinations literal according to existing rules; do not
   reinterpret unrelated nested task routes. Apply shared claiming to strict and editor
   parsing and lexical policy preflight. Use `record_marker_unsupported` for task
   modifiers and an appropriate record stem/shape code for malformed stems. Cover both
   route/title orders, all supported task-marker families at both edges, and UTF-8
   diagnostic ranges. Change contradictory tests, not the approved scope.

   A normal claimed record with `--clip` currently attempts the clipboard (the audit
   reached an xclip timeout) instead of refusing the task-only execution modifier. Check
   modifiers before any clipboard read or other task side effect, while retaining the
   explicitly approved forced-destination literal behavior. Use a fake clipboard command
   to prove it is never invoked by a refused record capture.

   `Ramen @eat/\n- status:: liked\n- text:: %t` is currently seen as a template and
   refuses `template_missing_rows`. Template analysis runs before record field opacity
   in both strict and editor entry points. Protect record field values from template
   directive detection while preserving their exact bytes/ranges. Keep the existing `%t`
   expander and source mapping, including a real parent-line `%t` with opaque field
   values and template-injected sigils. Do not write a second expander or add filesystem
   access to capture_language/capture-rewrite.

   Finally, preserve established task literal decoding for record titles. The record
   `Ramen \#READ @eat/` retains the backslash in its title/frontmatter, whereas task
   capture of `Ramen \#READ` produces `Ramen #READ`. Reuse the relevant escape rules
   without rewriting field values; keep title and field source spans on original bytes.

2. **Share one invocation context and resolve template rows.** `capture/plan.rs`
   constructs a record policy, but `capture/record_note.rs` reloads it and creates a
   `RecordCatalog` for every item. Thread a shared native record context through the
   batch, with one immutable schema and one lazy per-folder catalog. Continue deriving
   reserved paths from all staged files, including record, project, and expanded
   template items. Ordinary task drafts must not acquire record folder reads. Use
   focused counters or test seams to verify one load/index per invocation for multiple
   records, plus no new record I/O for ordinary tasks.

   `CaptureParseResult::new_template` in `capture_parse.rs` bypasses
   `resolve_record_intents` and hard-codes `record: None`. This input reports no
   diagnostic in capture-parse, although capture refuses its second row:

   ```text
   {0} @eat/ %t
   - status:: {1}

   Ramen liked
   Udon WRONG
   ```

   Resolve expanded record rows with the shared records engine and a single catalog,
   carrying earlier row reservations. Expose the applicable record metadata and
   required/invalid states in the existing template wire structure, without replacing
   the template envelope or treating unresolved definition placeholders as final values.
   Map diagnostics, record spans, field `range`/`value_ranges`, and rewrites through the
   existing template maps to original UTF-8 draft bytes. Test invalid and missing
   values, same-draft collisions, repeated fields separated by notes, CRLF, Unicode, and
   parse/execution agreement. Preserve the ordinary first-item projection.

3. **Preserve typed values in note_files.** `capture/record_note.rs::note_file_property`
   copies `property.display` into both `value` and `display`, with `Option<String>` in
   `NoteFilePropertyJson`. On the same record, parse returns `liked`, `["hot", "soup"]`,
   and numeric `4.5`, but dry-run `note_files[].properties[]` returns `Liked`,
   `"hot, soup"`, and string `"4.5"`. Keep machine values and display labels separate.
   Share the existing record-value JSON conversion across parse and creation output;
   retain numeric/list types, canonical choice values, resolved dates, and explicit-none
   semantics. `serde_json::Value` can preserve the surrounding equality derives. Dry-run
   and committed output must agree with engine values and exact Markdown. Check the
   landed sibling CLI serializer for reuse rather than introducing another converter.
   Update the public JSON documentation to state the exact representation and include
   choice/list/number examples; read `docs.md` reference memory before documentation
   changes. Schema version remains 1.

## Verification and follow-up disposition

Add focused regression tests for the failures above. Retain the existing successful
record matrix: mixed task/record batches, later-item zero writes, on-disk/case and
same-draft collisions, explicit stems, defaults/none/lists, missing directories, `@@`
precedence, URL/wikilink opacity, no record-only ledger mutation, and install race
preservation. Use temporary data only.

Run `just check`, not `just check-full`. Run `just docs-gen` if generated help changes,
and `just docs-check` for the JSON contract update. Inspect `git diff --check` and the
final changes for unwanted task/ledger effects. A `check-full` infrastructure failure is
not remaining epic scope. If a new unrelated issue arises, apply `/sase_new_task` and
record its disposition before closing.

The lander already completed triage of every child `PROPOSED FOLLOW-UP:` and recorded it
on bob-cli-ir.5.1; do not create duplicate tasks:

- `.1` note #1: historical mounted Mac docs budgets match existing **bob-cli-h9**;
  supplementary note recorded there. Fresh `just docs-check` on the audited tree passed
  all gates, with 447 documentation tests passed and one skipped.
- `.1` note #2: declined an actionable task for an unidentified transient: no node,
  error, or retained log, and two unchanged reruns passed. Preserve the original note
  for recurrence; do not guess a root cause or corroborate an unrelated flake.
- `.3` note #2: the three records CLI catalog/completion/help failures were fixed by
  `.4` in `e081e729`; no follow-up remains for those reported failures.

Integration audit covered the unrelated commits since plan baseline `361974f4`: Gmail
routing/sync `8427bd96`, `ad2298d4`, `28729397`; records CLI `3e68a53a`; and Mac
close-alias documentation `274eeae7`. Record Gmail rewriting remains guarded, the CLI
contract repairs are integrated, task-only target consumers exclude record directories,
and the close-alias prose does not alter record grammar. Recheck newer commits at
implementation time and integrate relevant changes before the final gate.

## Final step: finish the landing now

After the repairs and verification, complete these actions in this coding turn; none
depends on this turn's future commit, push, or CI result:

1. Re-read bob-cli-ir.5.1 and all descendants/notes to confirm readiness; review new
   notes and any post-audit changes. The four original phases are closed. Confirm the
   repair satisfies the findings above and the inherited creation acceptance scope.
2. Run `sase bead epic-symbols bob-cli-ir.5.1`. Resolve every entry by wiring,
   privatizing, giving a justified non-test pragma, or deleting it; only re-key to a
   concrete still-open later bead that still needs that exemption. Run the command again
   until empty. It was empty at the audit.
3. Close normally with
   `sase bead close bob-cli-ir.5.1 --note "<concrete source/drift/decision review, repair and test evidence, and all three follow-up dispositions above>"`.
   Never force merely to succeed or to advance this successful nested landing. If close
   names unfinished descendants, resolve their real work/readiness first; if it names
   stale epic symbols, clean them up and retry.
4. Run `just symvision` if available and confirm the whitelist is clean. The audit found
   no symvision recipe; if still absent, explicitly record that and the empty
   epic-symbol results rather than fabricating a pass.
5. Open the plans sidecar with `/sase_repo` and set `status: done` only in the
   frontmatter of `plan:202611/capture_record_creation.md` (PLAN path from the bead).
   Declare the changed sidecar through `/sase_final` along with code changes.
6. Read `sase bead read bob-cli-ir.5.1 -r "Need the parent link"` and
   `sase bead read bob-cli-ir.5 -r "Verify child completion satisfies parent phase"`.
   The parent is phase **bob-cli-ir.5**. Confirm its capture-records scope is complete,
   run `sase bead epic-symbols bob-cli-ir.5`, retire any remaining entries, and close
   only that parent phase with
   `sase bead close bob-cli-ir.5 --note "<creation grammar, parse/template metadata, typed note_files, discovery, transaction, docs, and verification evidence>"`.
   Do not close **bob-cli-ir** or mark `plan:202611/record_notes.md` done; its other
   phases and already-waiting land agent still own that landing. If the parent is
   unexpectedly incomplete, record a precise blocker on it and report that instead.
7. Finish through `/sase_final` using the breaking `fix(capture)!:` message and the
   primary repository's completed-bead declaration where appropriate. Do not leave a
   subprocess running when ending the turn.
