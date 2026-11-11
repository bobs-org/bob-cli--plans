---
tier: tale
title: Finish the album consumer landing
goal:
  Repair album routing and diagnostic boundaries, complete docs and acceptance, then
  close bob-cli-k3.4.1 and parent phase bob-cli-k3.4.
size: medium
proposed_by: bbugyi200.athena.bob-cli-k3.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-k3.4.1
status: done
---

- **PARENT:**
  [202611/album_record_consumers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_record_consumers.md)
- **BEAD:**
  [bob-cli-k3.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.4.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-k3.4.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-k3.4.1.land.md)
- **COMMITS:**
  - [2814cf2](https://github.com/bobs-org/bob-cli--plans/commit/2814cf2080d90274459893f3498694682be87ea1)
    — docs(plans): mark album_record_consumers done after consumer landing

# Finish the album consumer landing

Finish the remaining work for epic **bob-cli-k3.4.1**, then close that epic and its
parent phase **bob-cli-k3.4** in this implementation turn. This is one bounded repair
and documentation task for one coder. There is no later land agent for this tale. Do not
close the containing epic **bob-cli-k3**.

## Context and verified work

Read these through their audited commands before implementation:

- `sase bead read bob-cli-k3.4.1 -r "Need the landing audit and remaining work"`
- `sase bead read bob-cli-k3.4 -r "Need the parent phase and final decisions"`
- `sase artifact read plan:202611/album_record_consumers.md "Need the approved consumer contracts and closeout acceptance"`
- `sase artifact read plan:202611/album_and_record_notes.md "Need outer-epic boundaries and album-records requirements"`

The original consumer epic's four phases are closed. The lander reviewed every child
note and the implementations in `f7332f58` (record context), `8edd5a5f` (capture),
`c36e6e75` (records CLI/check), and `3d1c0763` (completion). Those commits provide the
shared registry adapter, canonical album create identity, additive album metadata,
records command noun/stem selection, membership checks, task-home refusals, and
vault-aware completion. Retain that work; this tale plans only outstanding defects, the
unfinished landing docs, and final acceptance.

The audit was source review, not a fresh test pass. The child records phase records a
passing `just check` ToolRun `075822f0ee1567d32188ce6125793f24`; subsequent capture
completion work still needs the integrated gate. No child note contains a
`PROPOSED FOLLOW-UP:` entry. The lander found no independent out-of-scope defect to
file; the defects below belong to this epic. Preserve that triage outcome in the close
note. The research ref named by the original plan,
`research:202611/album_and_record_notes/album_and_record_notes.md`, returned `missing`
during audited read; the accessible approved plans contain the binding contracts.

Concurrent changes reviewed: `7eb82440` and `921e51d9` implement local SVG rendering;
`f9d998c1` implements staged notebook section completion and ambiguous-section refusals;
`db91ab20` updates keep-moving verification docs. The workspace was fast-forwarded to
`db91ab20`. The SVG and keep-moving work needs no album adaptation. Preserve its
documentation while editing configuration docs. The notebook change shares the
completion dispatcher, so the mixed-draft regression below must prove it still works
with album diagnostics. Fetch/review any newer base changes before closeout.

## Constraints

- Follow final DECISIONS: ledger mining is enabled elsewhere; only the outer epic's
  named glossary edits are authorized, and the albums decision record is declined. This
  tale edits no memory, vault data, version, or CHANGELOG. Outer wrapup owns the
  glossary updates and config retirement. Do not add new reviewer decisions.
- Keep the effective `--bob-dir`/`-b` over `BOB_DIR`. Album claims own a whole kind by
  noun or directory even when broken. Keep transitional config only for unclaimed kinds
  and preserve its diagnostics.
- Keep route-local album resolution bounded and invocation-local. Reuse supplied
  registry/policy context. Do not replace snapshot internals with recursive registry
  discovery or add full-vault scans to ordinary keystrokes.
- Keep existing JSON keys/schema versions and noun identity. Create album records with
  `parent: "[[stem]]"` and `type: "[[record]]"`; updates preserve authored spans and
  legacy types. Plans remain pure and batches atomic.
- Album indexes, owner catalogs, albums create/sync, Mac/plugin changes, and config
  retirement belong to later outer phases. `records init` intentionally remains
  config-backed here. If those phases land meanwhile, integrate without reverting their
  work and adjust acceptance to their shipped transaction scope.

## 1. Make resolved routing authoritative and retain its diagnostics

Start with regression tests, then repair `capture_record.rs`,
`capture_language/record.rs`, and their consumers:

- `record_policy_for_draft` currently loads the old config independently, then inserts
  per-route outcomes. It converts both `DiscoveryIncomplete` and I/O errors to
  `Unclaimed { config_diagnostic: None }`. `resolve_record_route` then falls through
  from that result to `configured_dir`, and `CaptureRecordContext::route_kind_for` falls
  back for every non-valid outcome. By contrast, parse planning skips explicit
  `Unclaimed` outcomes. This bypasses registry precedence and can make parse and
  execution disagree.
- Concrete fixture: `eat.md` claims noun `restaurant`, while transitional config
  declares `restaurant` with `dir: old_eat`. Resolve `Place @old_eat/`. The registry
  suppresses that entire config kind; capture must not resurrect its schema or write a
  legacy record. Test with a valid and a broken claiming album, both with and without an
  existing legacy folder. Preserve ordinary genuinely unknown-route semantics.
- Preserve incomplete-discovery and read-error outcomes as explicit diagnostics; never
  reinterpret them as proof that config fallback is safe. Use deterministic
  bad-byte/read-error fixtures or existing injected registry helpers rather than
  permission-only tests that fail under root. Assert no writes and parse/execute
  agreement. A healthy directly resolved album should remain usable despite an unrelated
  bad file when its bounded route does not need global discovery.
- Centralize production schema loading through the registry/context and make valid
  resolved kinds sufficient for capture planning (a one-kind planner view is already
  used for albums). Legacy-only policy constructors may remain for focused tests, but
  production per-route outcomes must always win. At the end, audit
  `rg -n 'load_records_config' src`: legitimate transition internals are the config
  loader, registry, snapshot, and the explicitly deferred `records init`; do not leave
  an independent capture loader bypass.
- Carry `records_config_deprecated` from resolved config kinds through capture parse,
  execution, target inventory, and completion using existing warning/diagnostic
  channels. The current capture adapter discards `Valid.warnings`; targets discard
  fallback findings; the shell providers return only candidate rows. Preserve machine
  row protocols and avoid duplicate warnings. Audit records CLI human and JSON paths:
  `print_warnings` currently drops warnings for JSON, and list does not call it. Emit
  the warning without changing existing payload meanings or exit codes.
- Cover no-config albums, malformed config plus a healthy album, ordinary draft
  no-record-I/O behavior, and mixed config/album batches. Keep the disjoint record spans
  fix from `01c8e822` intact.

## 2. Scope completion errors to the item at the cursor

In `capture_complete/records.rs`, `Session::route_diagnostic_result` selects the first
album/config error anywhere in `self.parsed.diagnostics`. `named_field` consults it
before resolving the current item. `complete` also returns it for a current item without
a record intent, intercepting the general completion dispatcher.

Scope route errors to the current item's route/range. A broken album item must keep its
diagnostic and remain non-actionable, while a different healthy album's field picker and
an ordinary or notebook item's completion continue working. Preserve a useful diagnostic
for incomplete current route tokens without leaking another item's error.

Add mixed-draft tests with the cursor on each item and with the broken item before and
after the healthy item. Exercise normal completion, `--record-field`, and
next-missing-field. Include a notebook section staged earlier in the same draft so the
`f9d998c1` staged-section behavior is verified through the integrated dispatcher. Retain
explicit-vault, shell quoting, section-create, and write-locked-vault coverage.

## 3. Finish the remaining membership and refusal boundary cases

- `records/check.rs::collect_orphan_records` currently skips whole album directories.
  The record catalog covers only immediate Markdown children, so a canonical
  `eat/nested/stray.md` escapes both member checks and orphan checks. Scan descendants
  as needed while excluding only actual immediate members. Test valid and broken albums:
  direct children remain members, nested canonical records get `record_outside_album`,
  and symlinks/standard vault exclusions stay excluded. Keep counts independent of
  schema validity.
- `records/cli.rs::report_discovery_errors` always returns `None` and emits nothing in
  JSON mode. Verify inventory/check never silently reports a complete healthy scan after
  partial discovery. Retain readable partial results and expose scan failures through an
  existing/additive diagnostic channel with appropriate failure status; do not block a
  healthy selected album solely on an unrelated album schema error.
- `capture_parse.rs` currently always offers the bare-album `@stem/` rewrite for
  `AlbumValid`, even when bounded presentation yields `owner: None`. Honor the approved
  owner-unresolved rule: report the album ownership problem and omit an unsafe rewrite
  instead of implying valid ownership. Keep a valid owner's rewrite limited to the route
  token, including Unicode titles, notes, and neighboring items. Reuse bounded owner
  resolution rather than adding a global scan.

## 4. Complete the landing documentation

Read `docs.md` with `/sase_memory_read` before edits. Update these existing pages to
describe the shipped transition accurately:

- `docs/records.md`: remove the stale claim that capture and records still read config
  until k3.4; explain album schemas, noun/stem selection, canonical storage, structural
  membership/counts, legacy-type warnings, all new check codes, and unchanged init.
- `docs/capture.md` and `docs/reference/capture-syntax.md`: replace config-only record
  claims; document per-album failure isolation, owner-directed task refusal and safe
  rewrite, album metadata, and album-aware targets/field/shell completion.
- `docs/reference/json-outputs.md`: show consistent additive source/album fields in
  records, parse, capture items and note_files, and enriched target fields. Document
  warning/partial-scan diagnostics from the repairs without changing version numbers.
- `docs/reference/configuration.md`: mark `records:` deprecated, serving only kinds
  unclaimed by albums; keep it documented until the outer wrapup removes it.

Do not claim indexes/catalog rebuilding has shipped unless newer outer-epic code
actually landed. Run `just docs-gen` and `just docs-check`, following `/sase_repo` for
any linked documentation sources needed by the tools.

## 5. Verify the integrated behavior

Run meaningful targeted records, capture, and completion unit/CLI tests as repairs are
made, then `cargo fmt --all`, **`just check`**, and the documentation gate. Do not run
`just check-full`. Use `/sase_monitor` for a genuinely long command; if using
`sase tool run check` inline, join the exact run on escalation. A monitor continuation
must retain the remaining closeout below. Do not end a turn with a live subprocess.

Complete the original isolated walkthrough with two albums and one unclaimed config
kind. Save comparable parse/dry-run/write/target payloads and verify:

- new/update by noun and stem; capture multi-item create followed by update of the
  staged record; exact dry-run/write bytes and zero dry-run writes;
- `-b` wins over a conflicting environment vault;
- broken/healthy album isolation and legacy-type warnings;
- task-home refusals through bare route, forced/default route, `-P`, task move and inbox
  routing; failure leaves every file unchanged;
- the same album identity/noun across records, parse, item JSON, note_files and target
  rows; config payloads retain their prior meaning plus diagnostics;
- only record files change under this consumer phase's implementation, and owner notes
  never change (if index phase landed, also verify its album edit in the transaction).

Record actual commands, results, and any baseline failures on bob-cli-k3.4.1 and
bob-cli-k3.4. Do not call a previous agent's pass a new pass. For a newly discovered
out-of-scope issue, follow `/sase_new_task` and record every triage outcome; fix defects
caused by this epic before closeout.

## 6. Finish this epic's landing in the same implementation turn

Do this after code and checks are complete, before `/sase_final`. No step depends on
this turn's future commit SHA, push, or CI run. The host commits the final code after
the turn; that is compatible with closing the epic now.

1. Re-read bob-cli-k3.4.1 and every child with
   `sase bead read ... -r "Need final descendant readiness and notes"`. Confirm every
   descendant is complete, all outstanding audit points above are addressed, and all
   notes are accounted for. Review new base commits for conflicts and integration needs.
   Do not reopen a closed phase just to record supplementary evidence; use a note.
2. Run `sase bead epic-symbols bob-cli-k3.4.1`. Resolve every listed exemption or re-key
   it only to a still-open later bead that needs it. Never leave an exemption keyed to a
   bead being closed. The lander saw no entries, but recheck now.
3. Close normally:
   `sase bead close bob-cli-k3.4.1 --note "<specific source/commit audit, repairs, integration, acceptance/test results, and follow-up triage outcomes>"`.
   If rejected, finish the named work and retry. Never use `--force` to advance a
   successful nested landing or merely to make the command succeed. Canceled or
   superseded work requires a deliberate reason and the matching non-done resolution.
4. Run `just symvision` if that recipe is available (it was absent at the lander's
   checkout), and record the absence otherwise. Through `/sase_repo`, open the plans
   sidecar, use the printed path, and set `status: done` in
   **`202611/album_record_consumers.md`**, the plan linked to bob-cli-k3.4.1. Read
   artifact content through `sase artifact read`; include the edited sidecar in the
   final declaration.
5. Read `sase bead read bob-cli-k3.4.1 -r "Need the parent link"` and verify that the
   linked parent remains phase **bob-cli-k3.4**. Re-read that phase and confirm this
   child delivered its full album-records scope, docs, and acceptance. Run
   `sase bead epic-symbols bob-cli-k3.4`, resolve or re-key leftovers, then close only
   that phase normally with
   `sase bead close bob-cli-k3.4 --note "<verified album-records behavior, final tests, and child landing>"`.
   Run `just symvision` if available. Leave **bob-cli-k3** and
   `202611/album_and_record_notes.md` open for its waiting land agent. If the parent is
   incomplete or ambiguous, record its concrete blocker and report it; do not force.
6. Finish with `/sase_final`, declaring all code/docs/plan-sidecar changes. Report the
   two completed closures and checks, or any explicitly recorded remaining blocker.
