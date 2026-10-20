---
tier: tale
size: medium
title: Repair product reference audit gaps and finish its landing
goal:
  Correct the environment guard and hand-written reference gaps, verify their
  integration, and close bob-cli-a0.7.1 and its enclosing phase bob-cli-a0.7.
proposed_by: bbugyi200.apollo.bob-cli-a0.7.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-a0.7.1
create_time: 2026-10-20 00:02:38
status: wip
---

- **PARENT:**
  [202610/product_reference.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/product_reference.md)
- **BEAD:**
  [bob-cli-a0.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.7.1.md)

# Finish the product reference landing

## Assignment and boundaries

This is the remaining work found by the land agent for **bob-cli-a0.7.1**, the
hand-written product reference epic. Implement this tale directly, then finish the
landing in the same turn. No later land agent resumes after this tale.

The epic's linked plan is **plan:202610/product_reference.md**, relative to the plans
sidecar as `202610/product_reference.md`. Its parent is the **phase bob-cli-a0.7**,
inside **bob-cli-a0**. Close the child epic and then that parent phase only. Do not
close bob-cli-a0 or mark its `bob_public_docs_site.md` plan done. Do not wait for this
tale's commit SHA, push, or CI: the host commits after the coding turn ends, and
successful closeout must happen before then.

Use audited reads:

```bash
sase bead read bob-cli-a0.7.1 -r "Need landing audit and remaining scope"
sase bead read bob-cli-a0.7.1.1 -r "Need completed capture and vault scope and notes"
sase bead read bob-cli-a0.7.1.2 -r "Need completed environment guard scope and notes"
sase bead read bob-cli-a0.7.1.3 -r "Need completed JSON scope and notes"
sase bead read bob-cli-a0.7 -r "Need enclosing phase scope and decisions"
sase artifact read plan:202610/product_reference.md "Need original reference acceptance requirements"
```

Follow `docs/develop/writing-docs.md` and read the required `docs.md` memory through
`/sase_memory_read`. The MIT license, search indexing, and published design decisions
remain the accepted branches. This work needs no memory edits, Rust behavior changes,
new public CLI, documentation moves, or sibling product changes. Preserve existing
contracts and generated blocks. Use `/sase_repo` before reading other repositories, and
`sase artifact read` for sidecar artifact contents.

One agent can finish this bounded Python/docs repair directly. The completed catalogs
and guard should be repaired, not rebuilt.

## Evidence already collected

All three child phases are closed and each has one completion note, all reviewed. The
implementation commits are:

- `5d3ad7d0`: configuration reference, environment inventory, lint wiring, tests.
- `6f6841b5`: capture syntax, vault format, nav and search cases.
- `f30389c7`: JSON catalog, Reference hub, goal cases, token coverage test.

At audit HEAD `f30389c7`, `git fetch origin` confirmed no base-branch drift. Non-epic
changes since the epic started were:

- `f3678291`: source-owned mounted hubs, including assembled-lint changes.
- `d18b45de`: publication source planner and docs CLI tests.
- `7751f505`: executable first-day tutorial, demo vault, shared sandbox, and tutorial
  gate in `docs-check`.

These changes coexist with the reference additions. `docs/_meta/mounts.yml` was empty,
so there was no landed mounted reference to retarget yet. The guard intentionally
inventories production Rust, not Python docs-tool environment names. The tutorial's new
sandbox/fixture is useful to this work and should be reused.

Verification already performed:

```bash
uv run --project tools/docs pytest -q tools/docs/tests/test_environment.py tools/docs/tests/test_capture_token_coverage.py tools/docs/tests/test_lint.py
```

Result: **65 passed in 22.60s**. The current environment inventory finds **51** distinct
production names with no coverage findings. The capture index has **53** operation
anchors. These passing checks do not detect the issues below.
`sase bead epic-symbols bob-cli-a0.7.1` and the corresponding parent-phase check both
returned no entries. The current justfile has no `symvision` recipe.

Follow-up triage is already complete and recorded on bob-cli-a0.7.1: **zero
`PROPOSED FOLLOW-UP:` entries** across the three children; no duplicates, new tasks, or
declined proposals are outstanding. Every issue below was caused by this epic and
remains its work. Do not file these as unrelated follow-ups.

## 1. Fix the environment scanner's Rust delimiter handling

In `tools/docs/bob_docs/environment.py`, `rust_tokens` distinguishes strings,
characters, identifiers, and punctuation, but `_matching`, `_skip_item`, and related
scope logic compare `token.value` without consistently checking `token.kind`. String or
character contents equal to a delimiter can therefore change the lexical scope.

These reproductions used `_production_strings`:

```rust
#[cfg(test)]
fn test_only() {
    let _ = "{";
    let _ = "BOB_TEST_ONLY";
}
fn production() {
    let _ = "BOB_PRODUCTION";
}
```

Expected production name: `BOB_PRODUCTION`. Actual: none. Replacing `"{"` with `r#"{"#`
gives the same false negative. Replacing it with `"}"` instead leaks `BOB_TEST_ONLY`
into the production inventory.

Make structural comparisons accept actual punctuation tokens only, including brace-depth
calculation and cfg-expression delimiters. Preserve the conservative inventory,
test-item exclusions, production code after test items, exact documentation matching,
and reasoned ignore handling.

Add meaningful regression fixtures through `scan_environment_names` and
`check_environment_coverage`: isolated `{` and `}` in ordinary/raw strings and character
literals cannot hide subsequent production names or expose test-only names. Assert the
missing production variable still produces its file/line diagnostic. Cover other
delimiters where the same code path uses them. Avoid tests that only duplicate helper
implementation.

## 2. Correct the JSON and configuration references against source

Repair `docs/reference/json-outputs.md`:

- Add **`bob ref scan -f json`**, including `--intake-only` where its report differs. It
  is public and missing from the catalog. Inspect `src/native/highlights_ref/cli.rs` and
  `scan_json.rs`: schema 1 comes from `REF_SCHEMA_VERSION`; reports include `mode`,
  `write_pdfs`, `hook`, `intake`, `summary`, `notes`, `failures`, `intake_only`, and
  `queued`. Describe success/partial/hard-error framing accurately. Link the relevant
  contract section rather than inventing a new schema.
- Correct **`bob ref today <REF> -f json`**. It pulls one reference's reading task into
  a Pomodoro and can write the vault; it is not a listing of today's references. `-d`
  plans without writing. The envelope is defined in `src/native/ref_library/today.rs`
  (`mode`, `changed`, `ref`, `task`, `pomodoros`, `capture`).
- Remove the blanket claim that all `capture-*` helpers are read-only. `capture-task-id`
  and `capture-pomodoro-name` write by default and have dry-run modes; template
  operations also mutate the shelf. Inspect their command definitions and write paths
  before describing them.
- Scope framing to each command family. `vault-sync status -j` directly serializes
  `StatusRecord` in `src/native/vault_sync.rs`, with no `ok` or `schema_version`. Do not
  promise that all command failures produce one shared error envelope. Capture helper
  error envelopes also need not carry the success version field.
- Complete the other-format section: `ref show -f markdown` is accepted in
  `src/native/ref_library/cli.rs` but absent from the table. Use actual flags, such as
  `-f table`, rather than saying "`-f json` off". Audit registered public commands and
  their serializers once more for similar omissions; human-only operations and internal
  stored JSON are not public JSON outputs.
- Verify the two purportedly captured JSON examples. The current plan example uses empty
  objects for structured fields. Show actual output from a clearly described synthetic
  fixture, or label an explicitly shortened illustration as an excerpt rather than
  implying it is a verbatim captured envelope.

Repair the plugin-install subsection of `docs/reference/configuration.md`. It currently
tells readers to run bare `bob plugins sync` from a checkout, which does not select that
checkout. `repo_from_matches` in `src/native/plugins/cli.rs` uses explicit `--repo`,
then `BOB_PLUGINS_DIR`, then the default. `src/native/plugins/guard.rs` refuses bare
sync from a different checkout. Show an explicit checkout path, or `--repo "$PWD"` with
its stated prerequisite; similarly make the list/preview command select the same source.
Do not execute a real plugin deployment.

Retain the configuration path precedence and section tables already verified against
`src/native/config/`; preserve current source-owned settings and reasoned environment
exceptions.

## 3. Finish the capture examples, literal coverage, and tutorial integration

The original plan requires syntax and one or two examples for every indexed operation.
Many sections currently give only a description and contract link: for example shared
parent, section shorthand, named links, active-task spelling, dependency relations,
shortening/shifting, wildcard close outcomes, and task-link toggles. Complete those
missing examples without expanding this into a second contract. Incomplete picker
examples must be labeled as draft states and show their expected refusal/parse need, not
executable successful captures. Provide `bob-task` result illustrations where task text
changes.

Audit both Grammar-at-a-glance tables in `docs/capture.md` against the existing index.
Equivalent forms may share a heading; retain the distinct operations and explanatory
links. Keep each hand-written reference within 6,000 words and the hub within 400.

`tools/docs/tests/test_capture_token_coverage.py` currently checks only anchor
locations, never the smoke case's `query`. Strengthen this structural check to require
the indexed token spellings to be represented by literal queries at the appropriate
heading, including grouped spellings as required by the approved plan. Keep the Markdown
index as the source of truth; do not add a second registry or pin ordinary prose. Add a
negative fixture proving an unrelated query at the correct location cannot satisfy token
coverage. Update `docs/_meta/search_smoke.yml` and headings as needed while preserving
existing cases and punctuation-only literal-heading semantics.

Use the newly landed `examples/demo-vault/` and `tools/docs/bob_docs/sandbox.py` for
verification setup. Add only necessary synthetic notes/tasks/ledger/config to disposable
copies. Use this checkout's current binary, isolated HOME/XDG state, fixed `BOB_NOW` and
`TZ`, `BOB_REF_JOBS_KICK=off`, and explicit avoidance/stubs for hooks, clipboard, sync,
login, and network jobs. No personal vault access.

Connect the new reference pages with `docs/start/first-bob-day.md` using short
related/next links, preserving executable fences and checkpoint markers. Link the
reference examples to the tutorial where it helps readers establish fixture
prerequisites. If mounted references or guides have landed since the audit, retarget the
hotkey/plugin links and smoke cases to those actual pages; otherwise retain real
external README links. Open sibling sources only via `/sase_repo`, and edit source-owned
pages only in their owning checkout.

## 4. Verify the finished integration

Review commits since `f30389c7` and reconcile any relevant changes before the final
gates. Preserve the source-owned hub support, publication planner, and tutorial gate.
Re-read all epic/child notes for any additions.

Run:

1. Focused environment, token-coverage, and lint tests, including the new regression
   fixtures.
2. `just check` as the file-change verification requested by the landing instruction.
   **Do not run `just check-full`.**
3. `just docs-check`, including environment coverage, generated drift, strict build,
   URLs/links, literal token/lane/goal smoke cases, executable tutorial, and
   documentation pytest.
4. `just docs-build-material` for the fallback.

Use `/sase_monitor` for commands too long to finish in the turn, with explicit
continuation instructions to finish this tale's closeout. Do not end a normal turn while
a command is still running.

Inspect built HTML for the four reference URLs, named token anchors, Pending/Next
anchors, and task-line anatomy. Preserve the original epic decisions. Recheck
representative JSON families against code and safe fixture invocations, including the
corrected scan/today behavior in dry-run mode. Do not test prose by brittle snapshots.
No new general catalog framework is needed.

If an unrelated failure is confirmed, apply the landing user's triage rules with
`/sase_new_task`, record its outcome on the epic, and distinguish it from any regression
introduced here. The original three child phases offered no follow-ups; retain that
finding in the close note.

## 5. Finish the epic and parent-phase closeout in this coding turn

Do this after verification, without waiting for this work's own commit. The closeout is
part of this tale's deliverable.

1. Recheck descendant readiness and all new notes. Every required phase must be
   completed; resolve genuine remaining epic work before proceeding.
2. Run `sase bead epic-symbols bob-cli-a0.7.1`. Resolve every exemption (wire the
   symbol, privatize it, add a justified non-test pragma, or delete it under the
   Symvision policy), or re-key the justfile line only when a still-open later bead
   genuinely owns it. Repeat until none remain.
3. Close normally:

   ```bash
   sase bead close bob-cli-a0.7.1 --note "<verified child notes and decisions; corrected guard/catalog/examples; post-start integration; actual check results; zero original proposed follow-ups and any new triage outcomes>"
   ```

   Do not use `--force` to make a successful landing advance. If named phases or
   exemptions block closing, finish that work and retry normally.

4. Run `just symvision` if the recipe exists. At audit it did not; if still absent,
   record that fact rather than introducing a dummy recipe.
5. Set `status: done` in the frontmatter of the original epic's linked
   **plan:202610/product_reference.md**. Use `/sase_repo` to open the plans sidecar
   before modifying it, use the path returned there, and include that repository in
   finalization. Consume its contents through audited artifact reads. Do not update the
   overarching docs site's plan to done.
6. After the epic closes, run:

   ```bash
   sase bead read bob-cli-a0.7.1 -r "Need the parent link"
   sase bead read bob-cli-a0.7 -r "Verify the enclosing phase is complete after child landing"
   sase bead epic-symbols bob-cli-a0.7
   ```

   The parent is **phase bob-cli-a0.7**. Verify that its four pages, environment guard,
   token/lane/goal search coverage, Reference hub, and integrated docs acceptance are
   all complete and that the child epic is closed. Resolve or legitimately re-key any
   parent-phase symbols before closing.

   ```bash
   sase bead close bob-cli-a0.7 --note "<verified completed child epic, four corrected reference pages, literal token/lane/goal coverage, guard regressions, integration, and actual gates>"
   ```

   Leave **bob-cli-a0** to its already-waiting land agent. Never force either close or
   close the containing epic. If the parent is now ambiguous or incomplete, record the
   specific blocker on it and report that state.

7. Follow `/sase_final` as the last action before a normal final response. The host
   commits after the turn; a not-yet-visible commit is not a blocker to the successful
   epic and parent-phase closes above.
