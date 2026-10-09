---
tier: tale
title: Capture and Keep choose the reference parent
goal:
  Routed URL captures and Keep imports select and report canonical reference parents,
  with safe prompt and journal replay behavior.
size: medium
proposed_by: bbugyi200.athena.bob-cli-5y.10
bead: bob-cli-5y.10
create_time: 2026-10-09 19:54:13
status: wip
---

- **PARENT:**
  [202610/ref_tasks_live_with_parent.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_tasks_live_with_parent.md)
- **BEAD:**
  [bob-cli-5y.10](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5y/bob-cli-5y.10.md)

# Capture and Keep choose the reference parent

Complete the assigned phase `bob-cli-5y.10` (`capture-gkeep-parent`) of `bob-cli-5y`,
following the final design in `plan:202610/ref_tasks_live_with_parent.md`, especially
section 8 and the `capture-gkeep-parent` phase. This is one bounded coding unit: the
strict parent resolver, required `ref create -P`, typed ingest parent, and optional
ref-job parent already exist. The phase has no prior implementation/remainder notes, the
checkout was clean during planning, and `sase bead epic-symbols bob-cli-5y.10` reported
no entries. Recheck these facts in the implementation checkout.

## Scope and rules

Work only in the bob-cli checkout and temporary fixture vaults. No live vault writes,
installation, linked-repository changes, or memory edits are authorized for this phase.
The Mac consumer work belongs to phase `mac-file-under`; provide its additive protocol
fields here without changing the Mac app. Keep schema versions at 1.

Read the assigned bead and design through the audited SASE commands, not direct sidecar
paths. Do not set bead status, close an ancestor, or create follow-up beads. The epic's
final decisions are binding: `cancel_dropped_wrapper_refs = no`,
`memory_ref_parent_decision = no`, and `memory_glossary_ref_terms = no`. Record the
skipped memory updates as `PROPOSED FOLLOW-UP:` notes on this phase; do not re-ask these
decisions or edit any memory file.

## 1. Keep reference grammar consistent across execution and editor parsing

In `src/native/capture_language/`, extend the lexical whole-item reference claim in
`item.rs` and its editor counterpart in `editor_parse.rs`. Share the shape detection
where practical so capture, parse, and completion agree. An admitted bare URL is a
reference when it has exactly one ordinary leading or trailing `@route`, a plain
`@@route` inherited destination, or a forced `-r route`. The body/URL intent excludes
the route token. Preserve the distinction between the input token and its canonical
resolved parent and record parent source as `explicit`, `global`, or `default`.

Update `draft.rs` and editor global inheritance: the current code deliberately re-parses
refs under `@@` as tasks; replace that behavior for a plain route declaration while
retaining `@@route+block-id` task/sub-bullet semantics, local override precedence,
duplicate declaration errors, and existing competing-destination errors.

Reference claiming stays policy-gated. `-R`, excluded hosts, disabled routing, child
lines, extra words or tags, schedule/priority/clipboard markers, dependencies,
operators, task references, and destination flags beyond `-r` retain existing task or
diagnostic behavior. Capture's current `explicit_destination` boolean includes forced
route; distinguish allowed `-r` from the other disqualifying flags. Do not teach
ordinary task capture to resolve aliases.

Resolve reference parents with `parent_notes::resolve_parent` in the vault-aware
planning layer before staging jobs or writing anything. Unknown, ambiguous, terminal, or
non-parent routes fail the item with the resolver's hints and preserve batch rollback.
With no selected route, use `mac_inbox` as the reference parent. Keep editor parsing
lexical: a bare URL remains submittable and does not acquire a required `needs` entry
merely because a client can offer a parent picker.

## 2. Publish parent previews, jobs, parse fields, and completion

In `capture/plan.rs`, replace the interim hard-coded inbox parent with the canonical
resolved route. Stage `ref_jobs::NewJob.parent` with that route and preview a fallback
whose target is `<parent>.md` and whose task line contains only the URL body. Preserve
library verdicts, URL deduplication, and side-effect-free dry runs.

In `capture/output.rs` and batch serialization, add
`ref.parent: {route, label, kind, source, alias}` to reference results, including
already-known URLs. `route` and `label` are canonical, `kind` comes from the resolver,
`source` is `explicit|global|default`, and `alias` is the matched alias or null. The
human queued headline is `would queue example.com/essay → sase` on a dry run (and the
corresponding real-run form). Its detail names the destination reading task:
`new to your library · reading task lands in sase.md`; for the default inbox, say
`→ mac_inbox` and `file it later`. Retain informative library-hit details.

Add `ref_parent: {token, source}` to `ref` items in `capture-parse` (both single-item
summary and per-item output as appropriate to the existing protocol). The explicit token
is the selected route name, global uses the inherited route, and default uses
`mac_inbox`; source uses the same enum above. Put it outside `needs`. A local route
token keeps an ordinary `route` span with correct byte offsets, and existing global
route spans remain intact. Partial route input must still request route completion.

Extend `capture_complete/candidates.rs` route ranking using the target scan's
`project_name_aliases`. Return the canonical replacement for an alias match, with
additive `match_kind: "alias"`; rank alias matches after canonical prefix matches and
retain stable order for the remaining match groups. Keep one candidate per route and the
existing empty-query behavior. Update route candidate models and shape tests.

## 3. Select and remember Keep parents before clipping

In `gkeep/plan.rs`, extend the URL-only rule to admit exactly one trailing `@route`
without weakening its existing restrictions on attachments, lists, shared notes, extra
body text, or link-title metadata. Track the route separately from the cleaned URL so
routing does not become part of the clip target.

Add public `-P/--parent ROUTE` to `gkeep pull` and its `PullArgs` in `gkeep/cli.rs`.
Provide complete help with alphabetically sorted options. Register the route value kind
in `completion/kinds.rs` as needed and update help/completion expectations.

Choose the parent for each new URL in this order: a valid note `@route`, explicit `-P`,
interactive answer, then `gkeep_inbox`. All explicit values use the strict resolver and
canonical route. An invalid note route warns with resolver hints and falls through to
the next source; an invalid CLI parent is an error before clipping or mutation.
Selection applies only to references; ordinary Keep tasks keep their current target
behavior.

In `pull.rs` and `ui.rs` (a focused helper module is fine), resolve all parents before
the clip pre-pass. Prompt only when stdin and stderr are terminals and neither `--quiet`
nor JSON output is enabled. Ask per new URL that lacks a higher-precedence parent:
`File example.com/essay under [gkeep_inbox]: `. Enter accepts the default; each accepted
answer becomes the next prompt's default. Invalid answers print hints and re-prompt.
Handle EOF without an infinite loop. Avoid spinner output during the prompt. Dry runs
stay free of clips, journal writes, vault changes, and archives.

Pass the selected canonical route into `IngestRequest.parent` and into permanent failure
retry text. Add optional `parent` with serde default to `JournalRecord` and write it on
`ref_created` events. Update all constructors and compatibility tests; older journal
lines still load. A matching `(id, fingerprint)` `ref_created` event remains
archive-only and never prompts or clips again; recover its parent for reports without
forcing a fresh selection. Changed notes follow existing reclassification.

In `gkeep/list.rs` and pull JSON/human reports, show the selected/planned parent
additively. The list hint is `🔗 ref → sase` for a resolved note route and
`🔗 ref → asks · gkeep_inbox` when a future interactive pull will choose. Listing never
prompts; existing offline verdict checks stay offline.

## 4. Document and verify the contracts

Update `docs/capture.md`: saving links, grammar table, route/global inheritance,
`capture-parse` additions, completion aliases, dry-run verdicts, and the examples that
currently list `@route`, `@@`, or `-r` as preventing ref capture. Update `docs/gkeep.md`
with the new option, source precedence, warning fallback, prompt eligibility and sticky
default, non-TTY default, journal replay, and list hints.

Use existing hermetic fixtures and fake adapters; do not access Keep or clip real URLs.
Add meaningful regressions in the language tests, capture CLI tests (ref, parse,
routing, completion), and gkeep tests covering:

- `URL @sase`, `@sase URL`, `URL @bob-cli` resolving to `bob`, default `mac_inbox`,
  `URL @nope` with hints and no queued jobs or accidental note, `@@sase` with three
  URLs, a local override, and `-r sase URL`.
- Extra words, child lines, task/destination flags, dependency/schedule/priority
  modifiers, `@@route+id`, `-R`, disabled routing, and excluded hosts keep their
  existing non-ref behavior. Update older tests that explicitly asserted the old
  route/URL behavior.
- Accurate route spans (including UTF-8 offsets), per-item and summary `ref_parent`,
  resolved `ref.parent`, alias completion ordering/canonical replacement, the staged job
  parent, fallback target, and side-effect-free dry-run previews.
- Keep URL-only trailing routes and alias canonicalization, precedence over `-P`,
  invalid note-route warning/fallback, invalid `-P` before clips, non-TTY/quiet/JSON
  default behavior, and old/new journal formats with replay that never re-asks.
- A real scripted pseudo-terminal integration test of two URLs: the first answer changes
  the next default, an unknown parent prints hints and re-prompts, and all answers occur
  before the fake clip adapter is invoked. Use existing dependencies or a small Unix PTY
  helper rather than requiring a live service or new runtime.
- Parent-bearing `gkeep list` hints and help/completion coverage for `-P`.

Run focused tests during implementation, format with `cargo fmt`, and finish with
`just check`. If a check fails, reproduce the failure on an untouched clean base using
permitted local-checkout access; fix failures caused by this phase. An identical
pre-existing failure is a `PROPOSED FOLLOW-UP:` note with the exact test, base evidence,
and any existing tracker, and does not keep this phase open.

## Completion

Record verification evidence and the two skipped memory updates using
`sase bead note bob-cli-5y.10 'PROPOSED FOLLOW-UP: ...'`. Record any unexpected public
interface changes as `INTERFACE CHANGE:` notes for dependent phases. Re-run
`sase bead epic-symbols bob-cli-5y.10` immediately before closing; resolve its entries
or re-key each affected justfile line to the open parent/later phase. Close only
`bob-cli-5y.10` with
`sase bead close bob-cli-5y.10 --note "<verified behavior and checks>"` once its scope
is complete. Submit the required SASE final declaration as the last action of the
implementation turn, using a Conventional Commit message that describes the capture and
Keep parent selection work. Do not close `bob-cli-5y` or any ancestor plan bead.
