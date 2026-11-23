---
tier: tale
title: Dual-read task drawers in Rust
goal:
  Read bold and legacy task drawers consistently, keep imported text inert, and preserve
  existing header writers and capture draft grammar.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ph.1--1
proposed_with: codex/gpt-6-astra
bead: bob-cli-ph.1
create_time: 2026-11-23 03:47:16
status: wip
---

- **BEAD:**
  [bob-cli-ph.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.1.md)

# Plan: Dual-read task drawers in Rust

## Scope and authority

Complete assigned phase `bob-cli-ph.1` (`drawer-contract`) of
`plan:202611/true_case_drawers.md`. Read that plan, this phase's latest notes, and
`research:202611/true_case_drawers/true_case_drawers.md` with audited SASE reads before
implementing. The epic's design supersedes the research wherever they differ: ship dual
readers first, preserve header writers and capture draft grammar, and defuse both legacy
ALL-CAPS and whole-bold imported text during this window.

This is one medium tale because the architecture and caller inventory are known: one
Rust recognition contract, its existing consumers, five import boundaries, and shared
fixtures. It requires no plugin implementation, host rollout, or design choice. There is
no unfinished coding remainder from an earlier attempt; the previous turn only waited
for the explicit prerequisite.

The prerequisite was verified on 2026-11-23: `bob-cli-on` is closed with resolution
`done`. Landing commit `7703011d` fixes the req-owning `^prj` Notes home and includes
RL53–RL60 / RW54–RW57. Two captures now reuse `## Notes` with column-zero bullets; an
existing `## NOTES` keeps its spelling. The documentation registry tests pass 8/8 on
this base. Preserve these fixes instead of reimplementing step 0 of the epic. The
formerly missing research ref is also resolved: refreshing the research sidecar through
`sase repo open bob-cli--research` made the original canonical ref readable.

The epic decisions are final: `drawer_glyph = tray`, and the two approved memory
decisions remain work for its later `memory` phase. This tale edits no memory. It
neither changes the release version nor CHANGELOG.md. It does not migrate a vault,
deploy plugins, alter Mac code, switch capture syntax, change rename NEW casing, or
enable bold header creation. Record discovered work only as `PROPOSED FOLLOW-UP:` notes
on `bob-cli-ph.1`; create no task beads and close no ancestor.

## Result and compatibility boundary

On a task with these direct children, `bob capture-drawers` and every vault reader
recognize both drawers and retain their exact titles:

```markdown
- [ ] #task Example ^example
  - NOTES
    - old note
  - **API Design**
    - new note
```

`__API Design__`, bold leads ending in a colon, managed lines, and whole-bold bullets
below a note or requirement are not drawers. Empty eligible headers are recognized.
`**Notes**` is the existing Notes home by key; adding a note must not create another
`NOTES` header or rewrite the existing header. New headers still use today's legacy
bytes, and new project sections keep today's casing rules. Capture drafts and all
existing writer-vector expectations stay unchanged. The intentional changes to imported
prose are the `__…__` escape and the resulting gcal rendered-span hashes.

## Implementation

### 1. Centralize the shape, identity, provenance, and mapping contract

Keep `capture_drawers` as the public owner; extract a focused child module if that keeps
its already large scanner readable. Separate a reusable body/title check from the
contextual `recognize_drawer_header` entry point, so imports can ask whether a body
would be a header without fabricating a vault, while vault readers enforce the real task
owner, direct-child relation, fence/quote exclusion, and managed precedence. Do not
label a lexical match a drawer until its context qualifies.

- Decouple `is_pomodoro_name` first, with its existing ASCII alphabet and punctuation
  behavior unchanged (`C++` and `A+B` accepted; `WHY?` and `OPTIONS:` refused).
- Rename today's drawer title predicate to `is_legacy_drawer_title` and preserve its
  grammar exactly, including its trailing-colon allowance and rejection of internal
  colons, non-ASCII letters, and `+`. Draft parsing and legacy creation validation
  continue to call this predicate in this phase; do not broaden them by aliasing
  `is_drawer_title` to the new recognizer.
- A bold body is exactly `**title**` after trimming external whitespace. The title is
  nonempty, single-line, has no whitespace just inside the delimiters, contains no `*`,
  has at least one Unicode alphanumeric, and does not end in `:`. Internal colons and
  punctuation such as `C++`, `Q&A`, and `iOS / OAuth 2.0` are valid. Trailing prose,
  IDs, tags, and fields outside the closing delimiter disqualify it. `__title__` is
  always a note. Children are optional for recognition.
- Run existing managed-line matchers before recognition, including SUB-PROJECTS,
  dependency/relation lines, and all managed logs. Share one reserved-key check derived
  from their owners plus TASKS and every `is_structural_h2_key` name (including
  generated catalogs). Reject reserved bold titles as drawers. Preserve the legacy
  syntax itself; use the same contextual reserved/managed guard for eligible containers
  rather than duplicating a broadening legacy regex.
- `drawer_key` collapses whitespace then calls Unicode `to_uppercase`. Titles remain
  authored bytes, never reconstructed from keys. Pin `Straße` → `STRASSE` and `Café` →
  `CAFÉ`. Update key comparisons and query matching that currently perform a local ASCII
  fold; retain ASCII transforms that deliberately create legacy titles or implement
  unrelated Pomodoro, task-type, or notebook contracts.
- Add source marker `bold | legacy_caps | section` and header/title byte ranges to
  `Drawer` and the recognizer result. Document exact range semantics: half-open UTF-8
  offsets into the supplied source, title excluding markers, header excluding the line
  ending. Test slices against LF, CRLF, indentation, and multibyte titles.
  `project_h2_drawers` supplies `section` and accurate heading/title ranges while
  preserving its existing section eligibility and insertion geometry.
- Split `NOTES_KEY = "NOTES"` from `NOTES_TITLE = "NOTES"` for this window. All identity
  comparisons use the former. Keep `NOTES_SECTION_TITLE = "Notes"` and the landed `^prj`
  handling. A later writer phase changes `NOTES_TITLE` to `Notes`.
- Separate title validity from creation policy (`can_create_drawer`). Title validation
  produces named refusals (`not_a_drawer_title`, `reserved`); creation additionally
  requires note content. Do not wire the new creation policy into capture's draft
  grammar or header emitters yet.
- Implement the pure DH helpers: `heading_for_drawer` returns a bold title verbatim, or
  today's `format_project_section_title` result for `legacy_caps`; `drawer_for_heading`
  preserves a valid title verbatim and returns a named refusal for an invalid/reserved
  one. An empty eligible H2 is representable by the mapping; capture's nonempty-creation
  policy must not reject it. Keep these helpers separate from actual project
  capture/promotion writers, which change in later phases.

### 2. Retarget vault readers and expose additive discovery metadata

Use the contextual recognizer everywhere a vault line is interpreted as a drawer. The
current inventory is:

| Surface                                                       | Required adaptation                                                                                                                                                                          |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `capture_drawers.rs`                                          | Task enumeration, catalog scan, existing-target matching, key/query comparisons, marker/range construction, and JSON listing.                                                                |
| `task_children.rs`                                            | Return the unwrapped authored title after managed/checkbox checks; direct-child and fence filtering remain authoritative.                                                                    |
| `task_reqs.rs`                                                | Recognize bold structural containers without creating requirements under them.                                                                                                               |
| `task_block_layout.rs`, `task_block_layout/{tidy,writers}.rs` | Notes-role lookup by key, zone classification, existing-home reuse, and `req_in_drawer` parent refusals. Preserve header creation bytes.                                                     |
| `task_split/{plan,carry}.rs`                                  | Drawer scope and labels use the unwrapped title and actual direct-child ownership. A deeper bold note stays splittable note content.                                                         |
| `task_merge/containers.rs`                                    | Read/select both kinds of existing containers; preserve mutation geometry and legacy creation.                                                                                               |
| `task_drawer_rename/mod.rs`                                   | Update `old_bullet_lines`, owner detection, lookalikes, `header_kind`, and content-preservation accounting. Preserve existing NEW validation, same-key refusal, and legacy retitling policy. |

Audit `rg` results for `is_drawer_title`, `parse_drawer_title`, `NOTES_TITLE`, literal
`- NOTES`, and drawer-adjacent `to_ascii_uppercase` throughout `src/native`. Classify
remaining callers explicitly as vault recognition, legacy draft/creation validation, or
unrelated title grammar. In particular, `capture_language/drawer_bullet.rs`,
`capture_project_note::is_project_section_title`, record/section draft classification,
and editor parsing keep their current draft behavior. Reader changes must not turn a
nested bold note into another drawer through a line-only classifier. Use the same fence
interpretation as `task_children`/requirements, including list-item fences.

Existing-target lookups return the source title/marker and preserve ambiguity: `NOTES`
beside `**Notes**`, or Unicode case twins, produces `ambiguous_drawer` with the
conflicting lines rather than silently merging. Preserve existing duplicate handling in
other consumers. Exercise Notes home selection and matching through real
capture/tidy/split/merge paths, not just isolated lexical checks.

Bump `DRAWER_CATALOG_VERSION` from 2 to 3. Store marker provenance per spelling and per
occurrence: the same title may occur as both bold and legacy, so one marker on a whole
key or overwriting a title's marker is insufficient. Keep occurrence counts,
distinct-note counts, recency, and deterministic tie-breaking. A v2 cache is rebuilt.
Record explicit spelling provenance now; defer the new explicit-spelling preference and
create-title rule to the later `capture-drawers` phase. Check that catalog v3 cannot
cause a mixed-case bare header to be emitted by existing creation callers.

Add `marker` to `bob capture-drawers` drawer JSON (and any recent drawer object that
represents a chosen source spelling), preserving existing fields, schema semantics, and
verbatim human titles. Do not add capture draft spans, preview annotations, or change
`GRAMMAR_REVISION` in this tale.

### 3. Defuse external text at the final child-body boundary

Add shared `task_text_escape::defuse_drawer_header(body)` beside checkbox escaping. Use
the union body recognizer for a hypothetical direct child. If the body qualifies as bold
or legacy, emit `__title__`, stripping only the bold markers and preserving the title.
Existing `__…__`, invalid/reserved bold leads, mixed prose, URLs, and nonheader bodies
remain unchanged. Preserve surrounding layout whitespace and make this helper
idempotent. Keep its ordering compatible with existing checkbox, field, task-token, and
block-ID escaping; never unescape already inert text.

Apply it after inline markup is assembled wherever external content becomes a task
child, not to arbitrary metadata or an entire serialized Markdown document:

- `gcal/description/bullets.rs::render_line`: replace the legacy `**…**` defuse with
  `__…__`; check the rendered body too, so `<b>Agenda</b>` becomes `__Agenda__`. Keep
  managed-line italicization and URL treatment. Existing owned-span hashes should detect
  this rendering change; preserve private/user-edited span ownership.
- `task_text_escape::escape_child_text`, used by `gkeep/render.rs` and
  `highlights_ref/ingest.rs`: defuse the final rendered child body and confirm that
  truncation or wrapping cannot recreate a header afterward.
- Gmail's `gmail/body/inert.rs`: its serializer does not use `escape_child_text`. Guard
  completed list-item/child bodies after `Inline::Strong` rendering. Audit task-child
  embedding call sites as well; keep normal mail headings, links, code, and
  mid-paragraph emphasis unchanged. Add behavioral coverage for plain ALL-CAPS and
  renderer-produced whole-bold list items rather than assuming a helper change covers
  Gmail.
- `%c` text flows through `capture_clip/render.rs::rendered_lines`, list/history
  planning, and capture insertion. Guard final inline, lines, list, and history child
  bodies. Preserve attachments, code snippets, list depth, existing checkbox escaping,
  and raw authored capture text. Test real committed clipboard notes as well as rendered
  output so a later normalization cannot undo the escape.

The observable assertion is that externally produced child text, placed below a real
task and scanned with production drawer and requirement readers, produces neither a
drawer nor a requirement. Test both `%c` into a new task and into an existing req-owning
task. The shared vectors should supply adversarial title cases; tests must assert final
bytes and scanner results, not only helper return values.

### 4. Add conformance fixtures and integration regressions

Create Rust-owned `tests/fixtures/task_drawers/{vectors.json,README.md}` with stable
case IDs and a documented schema. DV covers contextual recognition, title/key, marker,
ranges, and exclusion; DH covers mapping and named refusals. State that plugin twins
update bob-cli first, then copy the fixture byte for byte. Use the existing RQ/RL/SP
test harness style rather than introducing a second test framework.

Minimum DV matrix: bold and legacy, empty headers, ordinary mixed-case prose, `__…__`,
leading/trailing inner whitespace, empty/non-alphanumeric titles, internal and trailing
colons, embedded `*`, trailing prose/ID/tag/field, every reserved and structural family,
managed lines, checkboxes, quotes, plain and list-item fences, direct versus deeper
nesting, nested task ownership, tabs/spaces, LF/CRLF, Unicode folding, and same-task
case twins. Pin the preserved legacy grammar with `OPTIONS:` accepted but `NOTE: X`,
`KEY:: VALUE`, and `OPTIONS::` rejected.

Minimum DH matrix: byte-identical round trips for `API Design`, `C++ Notes`,
`Q1: Goals`, `Café`, and mixed internal whitespace; legacy Title Case mapping; empty
section bodies; named refusals for reserved titles and invalid text. Test the pure round
trip independently of project transformation.

Append bold-reader cases to RQ, RL, and SP fixtures and update their READMEs. Cover
Notes-by-key, drawer zones/tidy, requirement exclusion, `req_in_drawer`, and split
scope/title labels. Retain every existing writer expectation, including RW54–RW57 and
legacy NOTES output. Add focused Rust/CLI regressions for:

- discovery JSON marker values, verbatim human output, and a v2 → v3 cache rebuild;
- matching a bold Notes drawer for a note append without a second header;
- ambiguous bold/legacy and Unicode twins;
- split/merge and rename lookups, including content-preservation checks;
- imported ALL-CAPS/whole-bold text in each of the five external sources;
- unchanged new-header bytes and draft parse/completion behavior, including legacy
  creation when the catalog also holds mixed-case explicit spellings;
- unchanged Pomodoro names, ordinary emphasis, and prerequisite `^prj` behavior.

Do not change plugin fixture copies in this tale: `plugin-readers` depends on this phase
and owns their adoption. Record fixture/API details on the phase for it.

### 5. Publish the contract and run the gates

Read `docs.md` and follow the registry workflow. Create `docs/task-drawers.md` for the
epic's recognition, identity, dual-read window, outside-text rule, and pure DH mapping.
Distinguish shipped reader behavior from later writer and capture changes; do not teach
`- **Title**` as a new capture drawer syntax yet. Document byte-range conventions, JSON
marker values, Notes-by-key, and that capture creation stays nonempty. Keep the broader
project/task isomorphism outside this contract.

Register the page in `docs/_meta/contracts.yml` with a unique order. Generate the
contract indexes (`docs/develop/contracts.md`, `docs/README.md`), mkdocs navigation, and
llms.txt selection through `just docs-gen`, following existing tooling. Update the
pinned contract count and README-table digest in `tools/docs/tests/test_contracts.py`
from generated output. The current base has 56 contracts; derive the final count if
other work has landed. Link the contract from `docs/capture.md`,
`docs/task-requirements.md`, and `docs/reference/vault-format.md`; document the additive
discovery field in the relevant JSON/CLI reference. Update only importer documentation
made false by the new escape. Respect page budgets and mounted-repository ownership.

Run focused suites while implementing, then `just check`, `just docs-gen`,
`just docs-check`, and the docs contract tests. Use `/sase_monitor` for long gates and
wait for staging commands to exit before ending a turn. A full check includes
formatting, denied warnings/lints, and all test binaries. Do not broaden the tale to
repair unrelated failures: when a failure reproduces identically on the clean base,
record exact evidence as `PROPOSED FOLLOW-UP:` on this phase and continue as the
assigned-phase instructions require.

## Completion evidence

Record the implemented API, fixture ranges, test outcomes, and any clean-base failures
on `bob-cli-ph.1`. Show that both on-disk forms are read, imports are inert, writer
bytes/draft grammar remain on the legacy contract, and the prerequisite Notes-home
behavior still passes. The feature commit is additive/readers-first,
`feat(drawers): recognize bold task drawers alongside legacy headers`, without a
grammar-breaking `!`.

Before closing run `sase bead epic-symbols bob-cli-ph.1`; resolve each leftover or
re-key it to a still-open parent/later phase. Close only `bob-cli-ph.1` using
`sase bead close bob-cli-ph.1 --note "<verified evidence>"`. Do not close `bob-cli-ph`
or any ancestor. End the coding turn through `/sase_final` so the host commits all work.
The parent epic's reader-rollout gate owns release/host checks before any later writer
flip.
