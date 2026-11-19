---
tier: tale
size: small
title: Repair relative bank import globs and finish bob-cli-no
goal:
  Quoted ./ file patterns import the same exports as their unprefixed forms, the
  remaining approved deletion safety cases are tested, and epic bob-cli-no is closed
  normally.
proposed_by: bbugyi200.apollo.bob-cli-no.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-no
create_time: 2026-11-19 17:45:30
status: wip
---

- **PARENT:**
  [202611/bank_import_delete.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_import_delete.md)
- **BEAD:**
  [bob-cli-no](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-no/README.md)

# Repair relative bank import globs and finish bob-cli-no

## Scope and landing audit

This is the remaining work from epic `bob-cli-no`, whose approved plan is
`plan:202611/bank_import_delete.md`. The lander read the epic, every note on its three
closed phases, the approved plan, and the implementation commits: `5b547f13` (operand
expansion), `c94a718d` (delete flag), and bob-plugins `c3ca244` (Bank queue
weekly-import chip). The main delete lifecycle, additive JSON/error reporting, docs,
catalog invalidation, completion filter, and chip wiring are implemented. All decisions
remain final: no memory changes, including the declined host-file decision record, and
no release version or changelog edits.

At CLI `992fd33d`, `host_paths::collect_files` rechecks every glob result with
`Pattern::matches_path_with`. The glob iterator drops an initial `./`, while the
compiled pattern retains it. With an existing `a.qfx` in the process cwd, the current
production module returns:

```text
*.qfx: Ok(["a.qfx"])
./*.qfx: Err(NoMatch { operand: "./*.qfx" })
././*.qfx: Err(NoMatch { operand: "././*.qfx" })
```

The lander reproduced this both with an installed phase-1 binary and by compiling the
current production module in a tiny scratch harness (only its unused HOME environment
dependencies were stubbed). This is epic-owned work, not an unrelated feature request.

Audited evidence, including the reproduction and focused test output, is
`file:explicit:a98fd800433efd364c6531af`. Read it with
`sase artifact read file:explicit:a98fd800433efd364c6531af "Need landing evidence"`.

The approved delete tests also require three cases the shipped tests do not actually
exercise: FIFO refusal, a failed unlink in the middle followed by a successful later
unlink, and a malformed member alongside a valid source with the entire batch preserved.
The existing cleanup test puts the failing unlink last, and the malformed CLI test
supplies only the malformed file.

Integration audit through CLI `992fd33d` reviewed `a29a64f6`, `ea538f78`, and `992fd33d`
(periods/intention changes) and found no bank integration needed. Linked bob-plugins
history since epic creation includes copy-chip `d6d2c33`, Log Recall
`795fad3`/`3968f0d`, and property-card chrome `2f1e016`; these coexist with the Bank
queue chip and shared preview helper. Focused Bank cards, Keep inbox and Command Mode
refresh tests passed 65/65; the four reported timing files passed 55/55 in isolation;
generated plugin entrypoints passed build:check. No plugin source edit is planned.
Recheck post-audit drift before closeout.

A final fetch also exposed CLI `8bbb3041` (intention task-rewrite integration),
`2a5bc1f8` (copy-chip glossary clarification), and plugin `db89aec` (task-scoped
copy-chip lookup). Their changed-file sets and relevant code were inspected; none
changes bank operands, deletion, the command-preview API, or ledger Bank cards. The
checks above remain checks on the named earlier heads, not claimed verification of these
newer commits. The coder should start from current master.

## Implementation

1. Fix `src/native/host_paths.rs` so valid relative patterns with leading `./`
   components survive expansion. Keep the glob crate as the matcher. Make the secondary
   check agree with the iterator's representation, or remove the unnecessary check from
   the normal traversal and retain a correct normalized check where the explicit-dot
   fallback needs it. Do not canonicalize a pattern or collapse `..` across symlinks.
   Preserve the original operand for no-match messages, existing-file precedence,
   hidden-entry filtering, case sensitivity, sorted order, and canonical first-wins
   deduplication. Add regressions for `./*.qfx`, repeated `./`, a relative subdirectory,
   explicit hidden patterns, and overlap with an equivalent literal operand. Avoid
   changing process cwd from parallel Rust unit tests; subprocess CLI cases can set
   their own cwd.

2. Extend the existing focused tests without introducing a second fixture framework. In
   `tests/cli/bank.rs`, run bob with cwd set to a temporary Downloads directory and
   literal argv `./*.qfx`, verifying preview and actual `-D` import: counts, successful
   commit, deletion only after apply, and no doubled count for overlapping operands. In
   `src/native/bank/import_delete.rs`, add the Unix FIFO preflight refusal and put an
   injected unlink failure between two successful files, asserting all per-file outcomes
   and that the later unlink ran. Add a valid-plus-malformed CLI batch with `-D`; assert
   refusal, both source byte sequences unchanged, and unchanged vault HEAD/journal. Keep
   the existing symlink/vault-file, seen-replay, digest-change, dry-run, and retry
   tests.

3. Run focused host_paths/bank tests and the canonical `just check` gate. Do not run
   `just check-full`. Use `/sase_monitor` for long commands and carry this concrete
   closeout forward in its continuation; do not end with a live shell subprocess. Fix
   failures caused by this tale or the bank epic. Existing unrelated failures must
   retain their precise attribution and existing tracker; do not silently claim a full
   pass or broaden the repair into their feature. The lander did not run a fresh full
   Rust/docs gate. Historical catalog/periods failures are routed to active
   `bob-cli-nh`; mounted docs budgets are `bob-cli-jz`. No docs/help contract edit is
   needed for this bug; if one becomes necessary, read the docs memory, regenerate with
   `just docs-gen`, then run `just docs-check`.

## Follow-up triage already completed by the lander

Use the epic's landing notes for the durable outcomes; do not create duplicates.

- `bob-cli-no.1` #1: new large feature task `bob-cli-nt`, shared expansion for
  attachments import/ref create, depends on this epic. Attachments already expand tilde;
  preserve its file-URI handling and reference URL/vault-note semantics.
- `.1` #2 and `.2` #1: baseline catalog, completion and ordered-intention failures
  recorded as a DISCOVERED ISSUE on active `bob-cli-nh`, whose phase `.2` added periods
  update. No new unrelated task.
- `.1` #3: docs mount collision corroborated on `bob-cli-it`; the existing
  untracked/ignored mount classification defect remains there.
- `.2` #2 and `.3` #1: mounted Mac reference budgets corroborated on `bob-cli-jz`.
- `.3` #2: timing reports mapped to existing `bob-cli-km` (event links), `bob-cli-io`
  (frees), `bob-cli-ki` (KT1–KT8 resident count, already in reopening evidence), and
  `bob-cli-m1` (dependency stage). The lander's isolated control passes 55/55; this does
  not claim a new full-suite failure.
- `.3` #3: protected uncommitted task-status-cycler deployment routed to active
  `bob-cli-nj`, with `.4` owning that Log Recall surface. No forced sync and no
  duplicate task. The Bank phase already deployed bob-ledger-tools.
- Adjacent existing defects `bob-cli-nk` (guide sample drift) and `bob-cli-nl` (missing
  bank invalidation) are resolved by this epic's existing commits and were verified
  separately during the audit; preserve their recorded dispositions.

No proposal was discarded without a destination. The only newly filed feature is outside
the bank scope; the relative-glob defect and missing approved tests remain inside this
tale.

## Final step: finish the epic landing in this coding turn

This tale has no land agent. After implementation and verification, finish all the
following bookkeeping yourself, without waiting for your own future commit, push, SHA,
or CI result:

1. Read `bob-cli-no` and its descendants again with `sase bead read ... -r ...`; verify
   there are no new unresolved notes or post-audit integration conflicts. Record the
   exact checks and any independently tracked baseline failures. If the host created a
   separate descendant bead for this tale, finish and close it normally so it is not an
   open descendant when the epic closes. If this tale continues directly on
   `bob-cli-no`, use the epic close in step 3. Finish any genuinely incomplete
   descendant rather than forcing it closed.
2. Run `sase bead epic-symbols bob-cli-no`. For every entry, resolve it by wiring,
   privatizing, deleting or an appropriate non-test pragma; re-key only when a
   still-open later bead actually needs it. The lander's audit found no entries.
3. Close the epic normally with
   `sase bead close bob-cli-no --note "<verified original phases and decisions; relative-glob repair and safety regressions; post-start integration review; exact verification results; every follow-up disposition>"`.
   If stale symbols cause refusal, finish their cleanup and retry. If named phases are
   unfinished, finish/reopen them; never use `--force` to advance a successful landing.
   Cancellation/supersession would require a deliberate documented outcome, not a
   workaround for a failed close.
4. Run `just symvision` if the recipe is available, confirming the whitelist is clean.
   The audited justfile had no such recipe; explicitly record absence if that remains
   true. Open the plans repo with `/sase_repo`, resolve the epic's linked
   `plan:202611/bank_import_delete.md`, and set its frontmatter `status: done`. Mark
   this tale's archived plan done as well.
5. Run `sase bead read bob-cli-no -r "Need the parent link"` after closure. The audited
   epic has **no parent bead**, so there is no ancestor close to perform. If that
   relationship unexpectedly changed, follow the landing prompt's normal
   phase/plan-parent verification rules; stop and note any ambiguous parent.
6. Finish through `/sase_final`, declaring every changed repository, including
   plans/bead metadata repositories surfaced by the final context. This commits the
   final code and closeout together; no step above depends on that future commit having
   happened first.
