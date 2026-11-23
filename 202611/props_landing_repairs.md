---
tier: tale
size: medium
title: Repair property ownership and YAML writes, then finish the props landing
goal:
  Make props ownership and reversible writes satisfy the approved contract, verify
  integration, and close bob-cli-pf.2.1 and parent phase bob-cli-pf.2.
proposed_by: bbugyi200.apollo.bob-cli-pf.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-pf.2.1
status: done
---

- **PARENT:**
  [202611/props_cli.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/props_cli.md)
- **BEAD:**
  [bob-cli-pf.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/bob-cli-pf.2.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-pf.2.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-pf.2.1.land.md)
- **COMMITS:**
  - [3966100](https://github.com/bobs-org/bob-cli/commit/396610031495186d77f64f580a02a3ceb8f114ac)
    — feat(props): repair ownership, reversible writes, and CLI contract

# Repair and land generic note properties

Finish the remaining work in `bob-cli-pf.2.1`, then close that epic and its parent phase
`bob-cli-pf.2` in this coding turn. This is one bounded repair of the existing props
implementation, not a new feature or another phased epic. No land agent will resume
after this tale. Do not wait for this turn's commit, push, SHA, or CI to perform
closeout; closeout belongs with the final code.

## Context already verified

Read `bob-cli-pf.2.1` and its three child phases using `sase bead read`, and read
`plan:202611/props_cli.md` and `plan:202611/card_owned_frontmatter.md` using
`sase artifact read`. The original epic plan is `202611/props_cli.md` in the plans
repository; its current logical checkout path is `sase/repos/plans/202611/props_cli.md`.
Use `/sase_repo` to open that repository before editing it and use the printed path.

The land audit at `3e480e4a` reviewed every phase note and the implementation commits
`e98c7fe9`, `fdd94725`, and `3e480e4a`. Ordered inventory, strict JSON envelopes, all
six operation dispatches, lock/replan/atomic apply, capability, catalog, and completion
are present. Keep these foundations. The defects below are remaining epic work.

Honor the accepted parent decisions: `bank_foot=yes`, `drawer_memory=note`,
`memory_glossary_cards=yes`, and `memory_decision_record=no`. This tale requires no
memory or plugin edits. Do not change Cargo versions or CHANGELOG. Read `cli_rules.md`,
`docs.md`, and the daily-release decision through `/sase_memory_read` before the
relevant work. Keep schema version 1 and the existing operation names.

Intervening commits reviewed: `c72c941d` and `513ec478` (requirement close/consumer),
`c17dd80a` and `9d0cbfbe` (requirement docs), `badcd30c` (tick contention), `8c019eb4`
(Gmail HTML), `063a8768` and `33f6d08d` (reference scan dependency refresh), and
`7f104e52` (Pomodoro mention history). None adds a competing generic property writer or
requires conversion to props. The ref scan observes note changes and the props runtime
uses the existing vault lock. A fetch during landing found no commits on `origin/master`
beyond `3e480e4a`. Recheck subsequent drift before final close.

The lander completed follow-up triage and recorded it on `bob-cli-pf.2.1`:

- Phase `.1` note #2: capture performance flake corroborated `bob-cli-bo`.
- Phase `.2` note #2: two fixed-delay gcal acknowledgment tests became small task
  `bob-cli-pk`, marked ready.
- Phase `.3` note #1: executable-stub ETXTBSY flakes corroborated `bob-cli-5l`. They are
  distinct from the gcal timing defect, despite the proposing note's grouping.
- Existing `bob-cli-p4` was notified that its BOM discovery defect now affects props
  ownership. The safety integration below remains props epic work.

Do not refile these reports. They are attributed phase fail/pass reports, not new
full-suite reproductions by the lander. No `just check` or docs gate was run in the
audit turn; run those gates after repair.

## 1. Finish ownership classification

Address the epic's original `DISCOVERED ISSUE` note in `src/native/props/roles.rs`, with
source evidence and prospective-key tests.

- `done_tasks` is written by `collect_done/archive.rs` and determines archive pairing in
  `collect_done/pair.rs`. Protect the contexts in which the archive writer owns it,
  including area/project notes; do not permit arbitrary replacement through props.
- Project `status` is synchronized from the project task in `projects/sync.rs`. Project
  frontmatter `scheduled` is legacy input hoisted onto that task by the same writer.
  Classify them as managed with useful owning-command reasons.
- `task_count` and `open_task_count` are written on project notes and removed from
  non-project notes by `bob-project-tasks::refreshFile`. The lander confirmed this in
  the linked plugin's `main.js`. Respect that ownership; read its current source through
  `/sase_repo` if needed. Do not rewrite that plugin or duplicate `bob-cli-pi`.
- Deliberately resolve the proposed `template` classification: the audited writer in
  `capture_project_note.rs` only stamps a creation default. Under the approved rule,
  this alone does not make a key derived. Leave it loose unless a continuing owner is
  found, and document the disposition. `created` remains loose.
- Do not downgrade ownership on BOM input. `PropsDocument` accepts a BOM, whereas
  `projects::parse_frontmatter` currently requires the first line to be exactly `---`.
  Resolve valid BOM-prefixed typed notes consistently, or refuse when canonical
  record/album discovery is genuinely unavailable. Cover album, record, project, and
  reference cases, including a BOM on the album schema source. A shared parser fix is
  acceptable with existing-consumer regression coverage and a note on `bob-cli-p4`;
  otherwise keep the fallback safe within props. Do not claim schema ownership from an
  unrelated guessed type or weaken identity/card precedence.

Exercise all six operations, rename destinations, new keys, and forged restores against
protected names. Retain edits next to protected neighbors and genuinely loose custom
fields. Document each added ownership rule's actual writer.

## 2. Repair spans and verify edited values

Work in `document.rs`, `emit.rs`, and `plan.rs`; reuse the existing parser and pure
planner. The following reproductions identify concrete failures:

```yaml
---
x: |- # inline
  old
---
body
```

`rebuild_entry_with_value` computes the comment gap as
`source[entry.value_range.end..comment.start]`. Here the value ends after `old`, while
the comment starts on the header. The lander's small Rust harness compiled the exact
production function with these spans and panicked (`24 > 10`). Repair the span/trivia
model and rebuilding so both scalar and multiline replacements preserve the inline
comment without panic, duplicating old children, or moving body bytes.

The lander also compiled the exact `emit_block_scalar` function. For requested `"\n"` it
emits `x: |\n` (empty value after YAML parsing); for `"a\n\n"` it emits `x: |+\n  a\n`
(one terminal newline). Requested `"  a\nb"` emits inconsistent implicit indentation.
Correct blank-only strings, trailing newline counts, leading spaces, and nesting in
lists/maps. Preserve the meaning of untouched block-scalar blank lines when neighboring
keys move. Include CRLF and inline-comment variants.

`plan::verify` currently skips every touched key. Implement the approved semantic
postcondition: every operation must produce its expected typed YAML value, alongside
public JSON checks where representable. Track state in operation order so a later
operation cannot hide an earlier wrong edit. Retain untouched-byte, BOM, body, and
untouched-semantic checks. Return `prop_verify_failed` with useful operation context
without installing a candidate when verification fails. Account explicitly for the
documented datetime separator/offset spelling and tag normalization; do not disable
target checks to accommodate those transformations.

`parse_yaml_fragment` and `validate_raw` currently accept an incoming anchor/tag without
checking its opacity. Reject incoming unsafe fragments structurally, including nested
cases and merges/aliases that could affect siblings. A successful update must not create
an opaque entry its own inverse then refuses. Preserve quoted or block text that merely
resembles YAML syntax. Keep validated restore subject to the same role, opacity, and
semantic rules; never restore an entire saved file over fresh body bytes. Cover invalid
raw values as well as sibling/fence injection.

Add regressions through the real planner and golden fixtures, not a copy of the
implementation. For the repaired cases assert the precise candidate, expected parsed
value, exact undo, and redo. Exercise each operation and an ordered batch with a later
failure. Existing goldens must continue to pass; don't weaken byte assertions.

## 3. Complete the command contract and integration evidence

- `format_arg` accepts `human|json`, but both approved plans and `docs/props.md` promise
  `text|json`. Accept and advertise `text`, retaining `human` as a compatible alias.
  Repair examples in `src/runner.rs` and `props/cli.rs` that pass unsupported `-n`;
  dry-run is `-d`.
- Fix target normalization so an absolute NOTE within a vault supplied with relative
  `-b` works. The existing lexical prefix check compares an absolute note to a relative
  vault. Keep traversal and symlink-escape refusals and repeat confinement after
  locking. Test both path forms and refusal cases.
- Extend `tests/cli/props.rs` with stdin input and each operation through preview/apply
  and inverse replay, plus the regressions above. The existing CLI tests primarily
  exercise set and its restore inverse; pure tests don't prove command dispatch.
- Add the approved deterministic schema-change-under-lock test: while an update waits
  for the lock, an initially loose record key becomes schema-owned; on acquiring the
  lock the writer must refuse it without writing. Also cover formatting-only edits under
  lock, preserving their bytes, and a changed fingerprint. Synchronize readiness
  observably rather than relying solely on a fixed sleep. Preserve show/dry-run/no-op
  read-only behavior and no Git/sync behavior.
- Update `docs/props.md` with the completed owner audit and corrected behavior. Generate
  CLI help with `just docs-gen`, inspect the generated changes, and run
  `just docs-check`. Do not expand unrelated docs or alter plugin implementation.

## 4. Verify and finish this landing in the same turn

Run focused props, help/catalog/completion, and any shared-parser regression tests, then
`just check` and `just docs-check`. Never run `just check-full`. Use `/sase_monitor` for
long verification; its result successor must finish the closeout, not leave it to a
nonexistent tale land agent. A passing `just check` alongside a `just check-full`
failure is infrastructure work, not remaining epic work.

Read `obsidian.md` before the original plan's real-vault sampling. Run the freshly built
`props show` read-only on available note kinds; mutate only temporary fixtures. Record
any absent live kind or inaccessible vault and cite its isolated fixture. Do not use an
older installed bob as evidence for the repaired source.

Re-read epic/phase notes, confirm the original acceptance is complete, and inspect
commits since `3e480e4a` for new integration needs. Include the previously recorded
follow-up outcomes in the final verification note. Resolve any new epic-caused defects
before closing; genuinely distinct unrelated discoveries follow `/sase_new_task` rather
than being silently deferred.

The final step is concrete closeout:

1. If this tale is represented by an open descendant bead, complete and close that
   assigned tale normally after its implementation and verification, before the ancestor
   close. Do not wait for its own commit or CI. Recheck descendant readiness; every
   required phase must be complete.
2. Run `sase bead epic-symbols bob-cli-pf.2.1` and
   `sase bead epic-symbols bob-cli-pf.2`. For every listed allowance, resolve the symbol
   (wire it, privatize it, add a legitimate non-test pragma, or delete it per policy),
   or re-key its Justfile entry only to an identified still-open later bead that needs
   it. The land audit found none; recheck rather than assuming that remains true.
3. Close with
   `sase bead close bob-cli-pf.2.1 --note "<completed scope, audit repairs, decisions, intervening-change review, follow-up outcomes, and verification>"`.
   If symbols or incomplete descendants reject the close, finish them and retry. Never
   use `--force` merely to succeed or to advance this successful nested landing.
4. Run `just symvision` if available (the audited justfile has no such recipe). Mark
   `status: done` in the frontmatter of `202611/props_cli.md` in the plans repository
   opened through `/sase_repo`. Commit that repository through `/sase_final` too.
5. Read `bob-cli-pf.2.1` again for the parent link and read `bob-cli-pf.2` for scope. It
   is the parent **phase** for this implementation. Once verified complete, close only
   it with
   `sase bead close bob-cli-pf.2 --note "<verified props-cli acceptance and child epic completion>"`.
   Leave containing epic `bob-cli-pf` and its parent plan status to its waiting land
   agent. Do not force-close any ancestor.
6. Finish through `/sase_final` with code, documentation, and plan status changes
   declared for commit. Report the closed epic and phase and verification outcome.
