---
tier: epic
title: Preserve intention links through vault rewrites
goal: Links inside daily and period intentions, summaries, and reflections survive
  archive, task, and reference rewrites with valid YAML and correct provenance; the
  shared periods problems report identifies malformed or stale period text.
phases:
- id: owned_repair
  title: Verified owned-field repair, archive integration, and digest carry
  size: medium
  depends_on: []
  description: 'owned_repair: add a pure canonical-note adapter that rewrites only
    stored intentions, summary, and reflection through the verified fields writer.
    Integrate moved block and archive-rename repair, extend draft digest carry, and
    test YAML preservation and archive behavior.'
- id: task_repair
  title: Task move, merge, and split link parity
  size: medium
  depends_on:
  - owned_repair
  description: 'task_repair: cover task move source/destination/bystander notes, preserve
    merge''s retained source identities, and repair cross-note split anchors using
    the shared adapter and existing body resolvers. Verify previews, postimages, counts,
    cursor positions, and drafted versus edited/reflected provenance.'
- id: reference_repair
  title: Reference rename and overwrite owned-field repair
  size: medium
  depends_on:
  - owned_repair
  description: 'reference_repair: route canonical daily and period owned fields through
    typed verified repair in the reference link planner, including its overwrite and
    highlight-only callers. Preserve selectors, ambiguity handling, preview evidence,
    and unrelated YAML; restamp only eligible drafted notes.'
- id: findings
  title: Period field findings and public documentation
  size: medium
  depends_on:
  - owned_repair
  description: 'findings: add dangling_period_link, intentions_shape, intentions_over_soft_cap,
    and legacy_period_text to the shared report-only collector for canonical daily
    and period notes. Test human/JSON/nightly behavior, document the contract, and
    regenerate CLI documentation.'
proposed_by: bbugyi200.athena.bob-cli-nh.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-nh.4
create_time: 2026-11-19 13:01:11
status: done
bead_id: bob-cli-nh.4.1
---

- **PROMPT:** [prompts/202611/intention_link_repair.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/intention_link_repair.md)
- **PARENT:** [202611/period_intentions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/period_intentions.md)
- **BEAD:** [bob-cli-nh.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nh/bob-cli-nh.4.1.md)

# Preserve intention links through vault rewrites

## Assignment and accepted scope

This is the implementation child epic for `bob-cli-nh.4`, the `repair` phase of
`plan:202611/period_intentions.md`. The enclosing epic's decisions are final:
`day_summary_cap = sixty`, both glossary decisions are accepted only for their listed
strands, and `memory_decision_records = no`. This child changes no memory, summary caps,
plugin code, installed binary, or live vault content.

Read the enclosing plan with `sase artifact read` and retain its authority over this
implementation detail. The accepted research is
`research:202611/period_intentions_card/period_intentions_card.md`; use an audited
artifact read, opening the research sidecar through `/sase_repo` first if it is not
materialized. Both artifacts were read while preparing this plan.

The requested result is that an intention, summary, or reflection retains its working
links when Bob archives or moves their targets. Bob's mechanical edits must not convert
an untouched agent draft into a human edit. Problems reporting must expose malformed
fields, missing targets, excess intentions, and remaining legacy text without writing
the vault.

## Existing work and reason for an epic

At inspection, `bob-cli-nh.4` was in progress with no prior-work notes. `bob-cli-nh.1`
was closed and its commit `e7aa392d` was the checkout's HEAD. That commit already
supplies the typed reader, canonical subject resolver, verified span writer, unified
states, and reader projections. Do not reimplement them. No repair implementation or new
findings have landed in that tree. `sase bead epic-symbols bob-cli-nh.4` reported no
entries.

This work crosses three different rewrite systems, an additional split planner, and a
diagnostic collector. Their failures differ: invalid YAML, stale draft stamps, lost
source identity, incorrect highlight selection, and misleading findings. Four bounded
coding phases keep those responsibilities reviewable. `owned_repair` lands first; the
other three phases can then run independently. Each phase is medium direct
implementation work, not another planning task. The normal child-epic land agent owns
the integrated acceptance work below.

## Verified code map

| Surface              | Existing implementation and required consequence                                                                                                                                                                                                                                                                     |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Owned fields         | `src/native/intentions/{fields,subject,write,state,legacy}.rs`. `Fields` retains the typed frontmatter mapping and field provenance. `OwnedPatch` and `write::apply` verify planned YAML and preserve unrelated values.                                                                                              |
| Archive              | `collect_done/link_repair.rs::repair_links_in_note_with` skips frontmatter when `skip_code` is true. `apply_link_repairs_to_plan` also runs `repair_rename_links` and currently carries only the Review digest.                                                                                                      |
| Move                 | `task_move/plan.rs` repairs the destination before inserting fragments. `task_move/transplant.rs::repair_vault_postimages` separately repairs source and bystander notes. YAML changes can change line counts, so preserve insertion/landing coordinates.                                                            |
| Merge                | `task_merge/{plan,payload,retirement}.rs`. Current merge copies sanitized payloads and retains source anchors on a cancelled source. It does not use the old global transplant path for ordinary inbound links. `docs/task-merge.md` MG3 requires those links to remain on the source.                               |
| Split                | `task_split/plan.rs::plan_task_split` stages source and destination notes, reuses selected bullet anchors, then computes cursor lines. It currently has no vault-wide retarget pass; cross-note extraction needs the actual relocated-anchor map.                                                                    |
| Ref rename           | `highlights_ref/rename/links.rs::{plan_vault_links_with_known_highlights,rewrite_text,rewrite_contents}` scans raw lines, including YAML. It tracks original line edits, counts, ambiguity, and highlight evidence.                                                                                                  |
| Ref overwrite        | `highlights_ref/overwrite.rs` calls the same reference planner and `rewrite_contents` for bundle/frozen-note and highlight-only rewriting. Preserve `LinkSelector` and known-highlight guards.                                                                                                                       |
| Generic transactions | `vault_rewrite/mod.rs::run` invokes `periods/note.rs::carry_review_digest` after planning. Its callers include theme rename/split/migrate, relation-label rename/reverse, and periods ensure/migrate. Most alter vocabulary or scaffolds rather than note destinations; do not invent new link retargeting for them. |
| Findings             | `periods/problems.rs::collect_problems`, `scan_canonical_dailies`, and `scan_period_notes` feed list and nightly. `periods/list.rs` renders the existing JSON envelope. Existing daily hierarchy checks use adopted ISO years; row filters currently do not filter the problems collection.                          |

Audit these paths again on the coding agent's current tree; sibling phases may have
landed since this plan. Search for `link_repair::`, `repair_links_in_note_with`,
`repair_wiki_link_inner`, `repair_markdown_destination`, `plan_vault_links`,
`rewrite_contents`, and `vault_rewrite::run`. Record the final caller inventory and
coverage in commit bodies or the finalizer commit message, as required by the enclosing
plan.

## Shared implementation contract

### Scope and YAML safety

Use `NoteSubject::from_canonical_note` / `fields::canonical_subject` with the
vault-relative path. Only canonical days and week/quarter/year notes qualify.
`all_time`, notebooks, areas, reference notes, and lookalike noncanonical paths do not
acquire intention-field semantics.

Add a small pure adapter under `intentions/` (for example `repair.rs`) accepting the
note path/subject and a caller-supplied Markdown rewrite operation. Read actual stored
YAML values, not legacy fallback text. Apply the caller's resolver and supported link
syntax to the decoded strings, including aliases, embeds, fragments, and supported
Markdown destinations. Reuse the caller's code/comment exclusions; do not treat YAML
syntax as Markdown or turn off protection for an entire frontmatter block. Reflection
paragraphs and folded/quoted YAML must reach the rewriter as their decoded values.

Only changed values become `OwnedPatch` entries. Return the verified resulting bytes and
accurate rewrite counts/metadata; an owned-only change must make the note enter the
caller's planned writes. Keep unchanged fields and all unrelated frontmatter bytes
exact, including comments, key order, BOM, CRLF, and trailing body bytes. Preserve the
existing dedicated Review-stamp behavior separately. On canonical notes the generic
raw-text pass must leave frontmatter alone, so it cannot rewrite an unrelated property
or apply the same replacement twice. Preserve existing noncanonical-note metadata
behavior, including reference companion fields such as `source_pdf`.

Never serialize the reader's filtered intentions list back over invalid stored items:
that could silently delete Bryan's text. A valid scalar intention can use the verified
writer's normal list representation when its link changes. For an invalid sequence or
invalid YAML, leave that field/block byte-identical and expose its existing diagnostic
through problems reporting. Other valid fields can be repaired independently only when
the mapping itself is valid. When a planned patch fails verification, propagate a
contextual planner error before any write; do not silently accept a partial proposed
postimage. Thread a fallible result through affected planner boundaries where necessary.
Existing body repair may proceed around an unparseable untouched frontmatter block.

Do not add a second YAML serializer or duplicate the full link-resolution algorithm. Use
typed values for detection and the existing verified writer for replacement. Reuse
`yaml_spans` if source locations are needed for previews.

### Provenance

Extend the behavior of `periods/note.rs::carry_review_digest` to cover summary digests
on days and periods. Make canonical subject context available where needed while
retaining the existing Review behavior, including legacy Review notes and `all_time`
Review bookkeeping. Derive eligibility from the original note before any stage of a
compound rewrite, not an intermediate edited state.

- A pre-write `drafted` note stays drafted after a mechanical link rewrite. Restamp each
  changed present agent part: summary and/or Review body.
- A pre-write `edited` or `reflected` note is never restamped, even if just its summary
  digest previously matched. Still repair its links.
- An absent/invalid digest is never repaired into ownership. Empty summaries and
  unchanged parts get no new stamps. Keep drafted dates unchanged.
- A summary-only day works without `review_digest`. A period with both agent parts must
  match both after repair. A legacy Review opener keeps its existing whole-Review rule
  and is not migrated by link repair.
- Apply the same rule to previewed/planned bytes and written bytes. Avoid a
  transaction-only restamp making the displayed proposal differ from apply.

Use the typed read/state logic and shared digest recipe. Avoid substring-only presence
checks that miss quoted YAML keys. Preserve no-op byte identity.

## Phase owned_repair

1. Implement the shared adapter and a reusable provenance-carry operation. Integrate an
   actual caller in this phase so production APIs are used.
2. Route archive moved-block repair and archive-file rename maintenance through the
   adapter. Include both pathful and pathless references where the existing resolver
   supports them; preserve pre-rename ambiguity behavior and all dependency metadata
   handling. Do not count YAML values as dependency fields.
3. Extend the existing digest hooks in archive and `vault_rewrite` to use the new carry
   logic. Keep existing transaction stale checks and rollback intact. Audit
   vocabulary/scaffold callers to confirm no target mapping is omitted.
4. Remove newly obsolete `expect(dead_code)` attributes only when the API has acquired a
   production consumer. Coordinate with the existing `nh.2` writer markers; do not
   delete another phase's still-needed markers blindly.

Tests: archive a block referenced by a weekly intention, daily reflection, and drafted
summary; include an archive-file rename. Assert valid YAML, retained
alias/embed/fragment forms, accurate counts, and exact unrelated-key/body bytes. Test
quoted and flow YAML, block lists, multiline reflection, BOM/CRLF, malformed/duplicate
keys, an invalid mixed sequence, no-op rewrites, and writer refusal. Pin a state matrix
for day, period, legacy, edited, and reflected notes, including both summary and Review
changes in one transaction.

## Phase task_repair

1. Wire the adapter and provenance carry into move's destination-before-insert pass and
   source/bystander postimages. Preserve fragment-only rewrite behavior and code
   exclusions; fragments are not standalone canonical notes.
2. Check all direct/capture move entry points reach the same pure planner. Recompute
   landing offsets after any YAML expansion, or order edits before offset calculation;
   never return a cursor pointing into the wrong row.
3. Audit merge against its current copy-and-retire contract. Ordinary links to retained
   source anchors, whether in body or owned fields, stay unchanged and resolve to that
   source. Only actual relocation paths (including the existing bare-note move dispatch)
   use moved-target repair. Do not globally replace mentions of the source with the
   merge target or treat ordinary prose as a Depends-On relationship. Pin this
   distinction with merge goldens.
4. For split, derive relocation identities only for anchors actually moved to a
   different note, including retained descendant anchors. Use the same map for body
   links and owned-field strings, and update final postimages before task/cursor-line
   reporting. Same-note extraction and freshly minted anchors must not fabricate
   retargets. Preserve stubs, IDs, links inside the extracted payload, and existing
   collision behavior. Keep the change inside the existing split transaction and capture
   dispatch rather than doing a second write.

Tests: move, merge, and split fixtures with links from all three fields; same-note and
cross-note cases; ambiguous targets; pathless links; destination YAML that changes line
count; capture delegation; source-anchor retention for merge; cross-note split anchor
resolution; dry-run/apply agreement; unchanged unrelated frontmatter; eligible restamps
and edited/reflected exclusions.

## Phase reference_repair

1. Integrate the adapter into `highlights_ref/rename/links.rs` for canonical notes. Feed
   decoded values through the existing bundle resolver and selection rules. Keep
   original-source line reporting meaningful after YAML re-rendering; ensure owned-only
   rewrites appear in `files`, `edits`, `rewritten`, and preview output.
2. Preserve bare-name ambiguity handling, percent encoding, Markdown suffixes,
   note/asset mapping, alias/embed/fragment syntax, and the `All` versus
   `HighlightFragments` distinction. Keep missing-highlight warnings and known-highlight
   checks; an owned-key link must not bypass overwrite guards.
3. Cover all callers in `highlights_ref/overwrite.rs`, including the in-memory
   frozen-document path. Apply provenance carry against original note bytes. Preserve
   existing reference identity behavior: IDs follow stems under the current decision,
   while reading-task block IDs remain stable.
4. Leave generic reference-note YAML behavior intact outside the canonical intention
   scope. A canonical note's unrelated `parent`, `custom`, or other properties must
   remain exact even if they contain a matching target.

Tests: extend ref rename unit and CLI fixtures with daily/week/year owned fields,
escaped quotes, encoded Markdown targets, flow lists, multiline reflection, drafted and
edited/reflected summaries, unrelated YAML, and an owned-only change. Extend overwrite
planner coverage for selected versus unselected fragments, known versus missing
highlight anchors, and ambiguous bare targets. Reuse the existing fixture/fake tooling;
no live PDFs or vault writes are needed.

## Phase findings

Extend `periods/problems.rs`, keeping its existing problem envelope, deterministic
ordering, fix hints, human rendering, nightly integration, and exit-0 report-only
semantics. Add no CLI flags. Reuse one note index per collection and cache target
contents for fragment checks rather than rescanning the vault per link.

Scan canonical daily notes in the collector's existing adopted-year report scope and
canonical week/quarter/year notes. New field findings exclude `all_time`. Keep
pre-existing hierarchy findings and the existing list row-filter behavior; do not
introduce an unrelated range/filter redesign. Check canonical paths even though the
historical period scanner is permissive about directory names.

| Kind                       | Level and required evidence                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `dangling_period_link`     | Error. Inspect actual owned values; identify key (and intention item when applicable), target, and an actionable fix. Resolve note paths/basenames with the production rules; distinguish missing/ambiguous targets and missing supported block/heading fragments. Ignore external URLs and code examples. Existing non-Markdown assets must not become false missing-note errors.                                                                           |
| `intentions_shape`         | Error. Map the typed reader's scalar/list/type/value errors and malformed/duplicate/unclosed frontmatter into clear findings without dropping their reasons. A scalar reads as one item but still warns about shape. Distinguish absent/empty fields from malformed ones; avoid redundant reports for one underlying failure.                                                                                                                                |
| `intentions_over_soft_cap` | Info. More than three readable intentions, including legacy fallback where the reader reports it. Never refuse or truncate text.                                                                                                                                                                                                                                                                                                                             |
| `legacy_period_text`       | Error. Recognized old daily Headline/Reflection preamble slots, week Intentions, quarter Outcomes, year Theme/Outcomes, period Reflection, or Review opening Headline remain. Detect both legacy-only notes and coexistence with new YAML. Explain equal duplicates versus conflicts; new frontmatter wins. Do not flag examples in fences or ordinary prose mentions of these labels. Empty legacy slots/headings and placeholders are still legacy shapes. |

Use `intentions::read` and `intentions::legacy` instead of new fallback parsers; factor
small presence/span helpers there if detecting empty shapes requires it. Do not migrate
or normalize during a report. Fix hints should point to migration for old shapes and to
a precise field/target correction for link/shape errors; conflicts must tell Bryan to
reconcile his text rather than imply migration will overwrite it.

Update the problems and mechanical-rewrite sections of `docs/periods.md` plus
`periods/cli.rs` list help. Follow the docs memory, run `just docs-gen`, and keep
generated reference pages in sync. Do not change memory or glossary entries.

Tests: each finding in human and JSON; all canonical scales; valid/ambiguous/ missing
note and fragment links; valid attachment links and external URLs; a correct target
after archive repair; malformed YAML and mixed items; 3 versus 4 intentions;
legacy-only/equal/conflicting/empty/template-example cases; excluded paths and
`all_time`; existing adopted-year boundaries (including ISO rollover). Snapshot fixture
bytes before reporting and assert no write afterward. The nightly collector must report
the same new kinds without changing exit status.

## Verification and land responsibilities

Each coding phase runs focused unit/CLI tests for its changed planners plus
`just check`. The findings phase also runs `just docs-gen` and `just docs-check`. Use
`/sase_monitor` for a long verification command instead of leaving a provider subprocess
running at turn end. No manual release/version/changelog changes.

The child-epic land agent must:

1. Confirm all four phases landed and reconcile any interaction with the enclosing
   epic's update, summary, or migration work. Recheck the caller audit and ensure the
   tested previews include exactly the bytes later written.
2. Run `just check` and `just docs-check` on the integrated tree. Exercise one temporary
   fixture through archive repair and subsequent `periods list -p` to prove repaired
   owned links are no longer dangling and draft state is retained.
3. Record caller coverage and validation evidence on `bob-cli-nh.4`. A failure
   reproduced identically on the clean base belongs in a `PROPOSED FOLLOW-UP:` note on
   that bead (cite an existing tracking bead if known), not an indefinite open phase.
   Workers do not create follow-up beads themselves.
4. Resolve or re-key each child phase's epic symbols before its close, and rerun
   `sase bead epic-symbols bob-cli-nh.4` before completing the assigned repair phase. No
   stale symbol may reference a closed phase.
5. Complete this child epic and `bob-cli-nh.4` only through the host's land flow for the
   assigned repair phase, after descendants and acceptance are complete. The owning land
   agent uses `sase bead close bob-cli-nh.4 --note "<verified coverage and gates>"`.
   Individual child phase workers close only their own assigned bead; ancestor closure
   instructions are evidence for the land agent, never authorization for them. Do not
   close `bob-cli-nh` or any other ancestor.

This planning turn submits the validated proposal and leaves `bob-cli-nh.4` in progress.
Implementation starts through the plan approval handoff.
