---
tier: epic
title:
  Saved Markdown for every research reference (birth pointer fix and two-week backfill)
goal:
  Every research report the SASE research-highlights file hook turns into a reference
  carries its saved Markdown, so the Bob Refs Copy card offers C (code block), L (link),
  and M (Markdown). That holds for every new report on athena and apollo, and for every
  hook-matched report from the last two weeks.
decisions:
  hook_timeout:
    ask: Should the research-highlights file hook's timeout rise from 120s to 300s?
    choices:
      raise:
        The hook gets a 300s timeout in chezmoi's sase.yml, applied on athena and apollo
      keep:
        The hook keeps 120s; apollo's slowest recent run (94.5s) leaves 25s of headroom
    default: raise
    why:
      A timed-out create files nothing, so a slow report on apollo silently never
      becomes a ref
    answer: raise
phases:
  - id: pointer-fix
    title: Newborn reference notes keep their birth source_markdown pointer
    depends_on: []
    size: small
    description:
      "pointer-fix: make both PDF-sync executors carry the planned birth-only
      source_markdown pointer into the note they write (today they rebuild metadata and
      drop it), with regression tests for inline create intake and queued scan drains."
  - id: adopt
    title: bob ref attach-source adopts orphan saved-source stores
    depends_on: []
    size: medium
    description:
      "adopt: let `bob ref attach-source` point a reference at its own unreferenced
      render-time store after verifying it against the PDF, leave its code.json
      untouched, make --search optional, and name the repair in the doctor orphan
      warning."
  - id: backfill
    title:
      Deploy, verify the hook on both hosts, and backfill two weeks of research
      references
    depends_on:
      - pointer-fix
      - adopt
    size: medium
    description:
      "backfill: install the fixed bob on athena and apollo, apply the hook_timeout
      decision, prove the hook path end to end on both hosts, then adopt orphan stores
      and attach hook-time Markdown to every research reference the hook matched since
      2026-10-12."
proposed_by: bbugyi200.athena.14o
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 07:42:23
status: wip
---

- **PROMPT:**
  [prompts/202610/research_ref_saved_markdown.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/research_ref_saved_markdown.md)

# Plan: Saved Markdown for every research reference

## The outcome

A research agent on athena or apollo finishes a report. The SASE `research-highlights`
file hook runs `bob ref create`, and the new reference note's frontmatter carries
`source_markdown: .bob/ref-sources/chat/<stem>--<hash12>/source.md`. In Highlights,
**⌘Y** opens the Copy card with **C · Code block…**, **L · Link…**, and **M · Markdown**
rows. The same is true for every report the hook turned into a reference since
2026-10-12. `bob ref doctor` reports no orphan stores.

## What the investigation found

### The SASE hook configuration is correct

- `~/.config/sase/sase.yml` (chezmoi-managed, identical on athena and apollo per
  `sase file-hook list --json`) declares:

  ```yaml
  file_hooks:
    - use: sase-research-artifacts@research-highlights
      command: bob ref create --include-id -P "$SASE_FILE_HOOK_PROJECT"
  ```

- SASE appends the report's absolute `.md` path (run logs under
  `~/.sase/file_hooks/runs/` show
  `bob ref create --include-id -P "$SASE_FILE_HOOK_PROJECT" <…>/research/202610/<stem>/<stem>.md`).
  That is the Markdown route. It renders with pandoc and writes the write-once store
  `.bob/ref-sources/<type>/<stem>--<hash12>/{source.md,code.json}` by default. No flag
  is missing. `--include-id` gives stable ids, and `-P` sets the project as the parent.
- Both hosts run current `bob` builds installed from the bob-cli primary checkout
  (`bob 0.3.1`, built 2026-10-26 ~21:41 EDT on athena and ~21:43 EDT on apollo). Both
  have `ref code`, `ref links`, `ref recatalog`, and `ref attach-source`.
- The only configuration risk is the timeout. The preset allows 120s. Over two weeks
  apollo's runs had p90 44s and max 94.5s; athena's max was 30s. See decision
  `hook_timeout`.

### The real gap: newborn notes lose their pointer (bead bob-cli-cy)

Every `bob ref create` from Markdown since the source store shipped writes the store,
and the create report even prints `source_markdown: …`. But the reference note it files
has **no** `source_markdown` line. All three references created this way are orphaned:

| Reference                                  | Host   | Store                    |
| ------------------------------------------ | ------ | ------------------------ |
| `machine_load_accounting_transparency`     | athena | `…--66d9d991ad1e` orphan |
| `muse_commentary_missing_from_exec_stream` | apollo | `…--ed2f4a29e3e3` orphan |
| `gmail_gog_reauth_apollo_athena`           | apollo | `…--d512d6ace044` orphan |

`bob ref doctor` agrees: `ref sources: warn (3 orphans …)`. The 235 references that do
have pointers all came from the bob-cli-cc.6 `attach-source` backfill.

**Reproduced deterministically** with today's master build, in a throwaway git vault
with real pandoc. Inline create intake and a `-k` create drained by `bob ref scan -i`
both file a note without the pointer, while the store's `code.json` `pdf.sha256` equals
the library PDF digest.

**Root cause.** `plan_pdf_sync` / `plan_pdf_sync_v2`
(`src/native/highlights_ref/sync.rs`) correctly derive the birth-only pointer
(`derive_birth_source_markdown` → `source_store::find_birth_store`) into
`plan.stable_metadata.source_markdown`. Then the executors throw it away:

- `execute_pdf_sync_v2` re-renders every new note (`is_new_note`) with a fresh
  `pipeline_metadata(…)`. It copies over only `metadata.birth_lineage` from
  `plan.stable_metadata`, so `source_markdown` is `None` in the note it writes.
- `execute_pdf_sync_v1` has the same shape and copies over neither.

`intake.rs` reads `plan.stable_metadata.source_markdown` to add the store files to the
scoped ingest commit. So the planner thinks a pointer exists while the written note has
none. The existing test `store_markdown_create_records_route_and_pointer`
(`tests/cli/highlights/store_gaps.rs`) hit exactly this. It explained the missing
pointer away in a comment ("Real renders embed the pointer in the PDF marker"), which
contradicts `decisions:reference-sources-are-preserved-write-once` (the pointer is
note-local).

### Two weeks of hook-matched reports (snapshot 2026-10-26 22:10 EDT)

These come from `sase file-hook history -j -n 0` on both hosts, audits since 2026-10-12
whose `matched_hook_names` include `research-highlights`, with batch run logs mapped to
their `ingested: <pdf> → <note>` lines.

| Measure                                       | athena | apollo | Total |
| --------------------------------------------- | -----: | -----: | ----: |
| Dispatched hook runs (all producer `commit`)  |     67 |     75 |   142 |
| Runs that failed                              |      1 |      0 |     1 |
| References that already have a pointer (cc.6) |     34 |     33 |    67 |
| References with an orphan store (adopt)       |      1 |      2 |     3 |
| References with no store at all (attach)      |     32 |     40 |    72 |

- The no-store 72 by project: sase 29, bob-cli 25, job 15, tailnet 3.
  - The tailnet count includes `athena_outage_recovery_and_remote_power`. Its 10-17
    athena run failed (`no area or project named 'tailnet'`), but the reference exists
    now with no pointer.
- All 25 bob-cli hook-time files are still readable at the audit's `commit_sha` in the
  bob-cli research sidecar.
- The `job` project is registered only on apollo.
- The research files live in four sidecars: bob-cli, sase, job, and tailnet research.
  The hook's absolute paths point into ephemeral workspace clones that may be gone or
  changed. The audit's `commit_sha` is the durable handle on the exact bytes the hook
  rendered.

"Copy them to the appropriate location" therefore means letting bob install each file in
the reference's store and set its pointer. That is `bob ref attach-source` (text
verified against the library PDF, scoped revertable commits). Nobody copies files into
`.bob/` or edits notes by hand.

## Principles

1. **bob is the only vault writer.** Stores and pointers change only through `bob`
   commands that take the reference-mutation locks and make scoped commits. Never
   hand-edit `.bob/` or note frontmatter.
2. **Exact bytes first.** Prefer the Markdown the hook actually rendered, taken from the
   research sidecar at the audit's `commit_sha`. Fall back to a later version only when
   verification says the PDF changed, and never loosen `attach-source` thresholds.
3. **Keep the better catalog.** An orphan store was written at render time, with anchor
   pages and render-time link records. Adopt it; never replace it with an `attached`
   (text-match) catalog.
4. **Plan and execution agree.** Birth-only plan state (`earlier:` lineage,
   `source_markdown`) flows from one place into whatever the executor writes. A dry-run
   preview and the real write must render the same frontmatter.

## Conventions for every phase

- Read `AGENTS.md` first. Before changing CLI help or docs, read `cli_rules.md` and
  `docs.md` with `/sase_memory_read`. Phase `backfill` also reads `obsidian.md` (vault
  conventions and recovery).
- Never edit the `Cargo.toml` version or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).
- Gates: `just check` and `just docs-check` must pass, apart from failures that
  reproduce identically on the untouched base (name them in the phase bead).
- Phase workers never create beads. Append `PROPOSED FOLLOW-UP:` notes to your own phase
  bead.
- **Throwaway-vault create check** (used by `pointer-fix` with a dev build and by
  `backfill` with the installed build). In a new `mktemp -d` scratch directory:
  - Make a bare `remote.git` and a `vault/` git repo with a master branch and that
    remote.
  - Seed `initial.md`, `sase.md` (`type: "[[area]]"`), and `.gitignore` (`/xlib/`), then
    commit and push.
  - Write a small `report.md` with one fenced code block and one `<https://…>` link.
  - Run `bob ref create <scratch>/report.md -P sase -i -b <scratch>/vault` with
    `BOB_DIR=<scratch>/vault`, `BOB_HIGHLIGHTS_INGEST_ON_CREATE=1`,
    `BOB_REF_INTAKE_LOCK_WAIT_SECONDS=0`,
    `BOB_VAULT_SYNC_LOCK_FILE=<scratch>/bob_sync.lock`,
    `BOB_VAULT_SYNC_STATE_FILE=<scratch>/vault-sync.json`,
    `XDG_STATE_HOME=<scratch>/state`, and `BOB_CONFIG_FILE=/dev/null` (real pandoc and
    XeLaTeX, about 30s).
  - Pass when `ref/chat/report.md` carries the `source_markdown:` line the create report
    printed, and `bob ref code report -b <scratch>/vault -f json` reports
    `"status": "available"`.
  - Delete the scratch directory afterward. Never point this at `~/bob`.

## Phase pointer-fix

Repository: bob-cli. Fixes bead bob-cli-cy.

- **Fix.** Add one helper in `src/native/highlights_ref/sync.rs` that copies a plan's
  birth-only state into execution metadata for a new note, for example
  `carry_birth_only_metadata(&mut metadata, &plan.stable_metadata)`.
  - It sets `birth_lineage` and `source_markdown` from the plan.
  - Call it from both `execute_pdf_sync_v1` and `execute_pdf_sync_v2` wherever
    `is_new_note` re-renders, replacing v2's lone `birth_lineage` copy.
  - Existing notes keep their authored `source_markdown` line through frontmatter
    preservation. Do not set the field on them, or the line duplicates.
- **Sweep.** Grep the other executors and publishers that re-render a note with a fresh
  `pipeline_metadata(…)` (`reference_mutation.rs`, `intake.rs`, `overwrite/execute.rs`,
  and the snapshot route). Confirm each one either sets the pointer explicitly
  (overwrite) or now goes through the helper.
- **Tests.**
  - `store_markdown_create_records_route_and_pointer`: delete the incorrect comment.
    Assert that the intake-filed note's `source_markdown` equals the pointer in the
    create report, and that `bob ref code store_route -f json` no longer reports
    `no_source`. With the fake pandoc the catalog may be `no_catalog`; assert whatever
    the harness truly produces.
  - New CLI test: `-k` create, then `bob ref scan -i` drains it. The filed note carries
    the pointer.
  - Unit tests in the sync tests: plan then execute a newborn note for both the v1 and
    v2 branches, with a matching store present. The written note contains exactly one
    `source_markdown` line, and the dry-run `stable_rendered_note` equals the executed
    note's frontmatter. Then re-sync the existing note: the line is preserved and not
    duplicated.
  - After a create in a test vault, the store files are tracked by git (in the ingest
    commit or in the pre-sync commit before it).
- **Docs.** Make sure the Recapture/lifecycle text in `docs/highlights-create.md` and
  `docs/ref.md` says the pointer is set on the filed note at birth (inline intake and
  queued drains). Fix any wording that implies the PDF marker carries it.
- **Verify.** Build, then run the throwaway-vault create check with the dev binary
  (`target/debug/bob` or `cargo run --`). Record the result in the phase bead.

## Phase adopt

Repository: bob-cli. Repairs the orphans the bug created, and any future orphan.
`decisions:reference-sources-are-preserved-write-once` already allows "verified attach
sets it explicitly", and adoption is a verified attach of bytes already in the vault.

- **Adoption** (in `src/native/ref_library/attach.rs`, before candidate search) applies
  to an eligible reference: PDF-backed, no `source_markdown`, no `source_note`. Look for
  stores under `.bob/ref-sources/<ref_type>/<stem>--*/` that meet all of these:
  1. No reference note points at the store. Reuse the live-pointer set from
     `source_store`'s doctor inspection rather than a second scan.
  2. The store's `code.json` `pdf.path` equals the note's `source_pdf`.
  3. The store's `source.md` digest matches its directory's `hash12` (the `read_store`
     verification).

  When exactly one store qualifies, verify its `source.md` against the library PDF text
  with the same rules as a candidate (title plus blocks, or the five longest
  paragraphs).
  - **Pass:** set the pointer through the existing `set_note_pointer` and leave
    `code.json` byte-identical. Print `adopted (saved at create)`.
  - **Fail:** print `skipped (saved source does not match: <reason>)` and continue to
    candidate search when `-s` was given.
  - **Two or more qualifying stores:** print `skipped (ambiguous saved sources)` and
    never fall through, so a candidate can never overwrite one of them.

- **`-s, --search` becomes optional.**
  - Without it, only adoption runs. With no `REF` arguments, consider only references
    that have an orphan store and stay silent about the rest. A named `REF` with nothing
    to adopt prints `skipped (no saved source; pass -s)`.
  - With `-s`, behavior is unchanged except that adoption is tried first.
  - Keep the usage-error path for a `-s` path that does not exist.
- **Commits and dry run.** Adopted references join the same batches and the same
  `bob ref attach-source: N references` commits. Stage the note, plus the store paths as
  a harmless no-op. A dry run prints the planned `adopted` lines and ends with
  `writes: none`.
- **Doctor.** When orphans exist, the `ref sources:` warning line in
  `src/native/highlights_ref/doctor.rs` ends with ` — bob ref attach-source`, mirroring
  the ` — bob ref recatalog` hint style.
- **Help, completion, docs.**
  - Update the attach-source long help: describe adoption and the optional `-s`, and add
    the example `bob ref attach-source -d` with the comment "adopt orphan saved
    sources".
  - Keep options alphabetical with short aliases.
  - Refresh the help snapshot and any catalog or completion fixtures the change touches.
  - Update `docs/ref.md` (the attach-source and doctor rows).
- **Tests** (`tests/cli/ref_library/attach.rs` plus unit tests). Use a fixture PDF from
  a fixture report with real pandoc, skipped when missing, as the existing attach tests
  do:
  - An orphan is adopted without `-s`. The pointer is set, `code.json` stays
    byte-identical, the commit contents are correct, and `bob ref code` reports the
    render-time catalog.
  - A dry run writes nothing.
  - These are each refused: two qualifying stores (ambiguous); a store another note
    points at; a store whose `pdf.path` differs; and a store whose text does not verify.
    The last falls through to a `-s` candidate when given.
  - A named `REF` without `-s` and nothing to adopt prints the skip line.
  - Doctor prints the new hint.

## Phase backfill

Operations on athena (the vault writer for this phase) and apollo. It depends on both
code phases being on `origin/master`.

### 1. Deploy the fix on both hosts

- On each host, check whether `~/.cargo/bin/bob` already contains the `pointer-fix` and
  `adopt` commits. Both hosts' bob was rebuilt within minutes of the last landing, so a
  post-landing install may already have happened. Compare the binary's mtime with the
  commits' times, and confirm with `bob ref attach-source --help` (adoption text) and
  the throwaway-vault check.
- If not:
  - On athena, run `just install` from this phase's own checkout at `origin/master`.
  - On apollo (`ssh apollo`), fast-forward its bob-cli primary checkout (the path
    `~/.cargo/.crates2.json` records for `bob-obsidian`) with `git pull --ff-only`, then
    run `just install`.
  - If that checkout is dirty or not on master, stop and report rather than force
    anything.

### 2. The hook configuration

> [!decision] hook_timeout = raise
>
> - Open the chezmoi repo with `/sase_repo` (`sase repo open chezmoi`).
> - In its source for `~/.config/sase/sase.yml`, add the per-hook timeout to the
>   `research-highlights` entry as 300 seconds, using the key name that SASE's
>   `file_hooks` schema uses.
> - Check that `sase file-hook list --json` reports `timeout_seconds: 300.0`. If the key
>   name is wrong, the value stays 120.
> - Commit and push in the chezmoi repo following its own AGENTS.md. Apply it with
>   `chezmoi apply` on athena and `chezmoi update` on apollo.
> - Re-run `sase file-hook list --json` on both hosts and confirm the same command and
>   timeout.

> [!decision] hook_timeout = keep Leave the hook entry untouched.

Either way, confirm on both hosts that `sase file-hook list --json` shows the
`research-highlights` command exactly as above, with `sidecars: [research]`, and record
it.

### 3. Prove the hook path end to end

Run the throwaway-vault create check with the **installed** `bob` on athena and on
apollo (over `ssh apollo`). Both must file a note that carries the pointer. If either
fails, stop. The backfill would otherwise race new orphans.

### 4. Rebuild the inventory

Recompute rather than trusting the snapshot above, because new reports may have arrived.

- On each host, run `sase file-hook history -j -n 0`.
  - Keep audits whose `created_at` falls on or after 2026-10-12T00:00:00-04:00. Compare
    as timezone-aware datetimes: apollo stamps differ in offset.
  - Keep only audits whose `matched_hook_names` include `research-highlights`.
  - Dedupe by `(project, rel_path, commit_sha)`.
- For each dispatched run, read the batch JSON (`~/.sase/file_hooks/batches/`) and its
  `log_path` run log. The note comes from `^ingested: <pdf> → <note>$`. For queued runs
  it comes from the `^pdf:` line (`ref/<type>/<stem>.md`). For a failed run, use the ref
  note whose stem equals the report's stem, if one exists.
- Classify each reference against athena's `~/bob`:
  - It has a `source_markdown` pointer: done.
  - It has an orphan store (cross-check `bob ref doctor`): adopt.
  - It has neither: attach.
  - No reference exists: list it, do not create one.

  Expect about 67 done, 3 or more adopt, and about 72 attach.

### 5. Collect the hook-time Markdown

- Open each project's research sidecar with `/sase_repo`. The bob-cli one is
  `sase repo open research`. For the sase and tailnet sidecars use their registered
  projects or `gh:` refs, per `sase repo open`'s three tiers.
  - `job` is registered only on apollo. Open its research sidecar there over
    `ssh apollo` (`sase repo open research -p job -r …`) and stream file bytes back. An
    alternative is opening its GitHub repo as an external repo on athena.
  - Never read another repo's files any other way, including the ephemeral workspace
    paths in the audits.
- For each attach reference, write
  `git -C <opened sidecar> show <commit_sha>:<rel_path>` to
  `<scratch>/hook_time/<ref_stem>.md`.
  - Name the file by the reference note's stem, so `attach-source` pairs it, even when
    it differs from the report's file name.
  - Keep the scratch directory outside the vault and outside every repo.
- If a commit is unreachable, `git fetch` in that checkout. Then try the same sidecar on
  the other host. Record anything still missing.

### 6. Adopt the orphans

Run `bob ref attach-source <orphan refs…> -d`, review it, then run it for real. Every
orphan should print `adopted (saved at create)`. If one does not, investigate before
going on.

### 7. Attach the rest

- Run `bob ref attach-source <attach refs…> -s <scratch>/hook_time -d`.
- **Gate:**
  - These are the exact rendered bytes, so nearly all should verify. If more than 10%
    skip, or any skip reason repeats in a systematic way, stop and report with the
    dry-run output instead of writing.
  - Otherwise run the write pass. It is at most 100 references per scoped, revertable
    commit.
- For each remaining skip:
  - Stage the sidecar's current `HEAD:<rel_path>` version in a second scratch directory
    (`<scratch>/head/<ref_stem>.md`), then dry-run and write those that verify.
  - Never pass a file you have not verified, and never loosen thresholds.

### 8. Verify and record

- `bob ref doctor` must show `ref sources: ok`, with no orphans and no dangling or
  mismatched stores.
- For every adopted reference and at least 10 attached ones:
  - `bob ref code <ref> -f json` is `available` or `no_code`.
  - `bob ref links <ref> -f json` returns its links status.
  - For one block, `bob ref code <ref> N` matches the PDF.
- Re-run the step 4 classification. Every hook-matched reference from the window has a
  pointer, except the recorded unresolved ones.
- In the phase bead, record:
  - the per-host deploy evidence, the hook-config check, and both end-to-end results;
  - inventory counts, adopted and attached counts, and vault commit SHAs;
  - every unresolved reference with its reason.
- Add a `PROPOSED FOLLOW-UP:` for the land agent listing which of bob-cli-co's skips
  this pass resolved, if any overlap.

## Land agent notes

- Close bead bob-cli-cy once `pointer-fix` and the `backfill` verification show newborn
  notes keep their pointer and the three orphans are adopted.
- Add a note to bob-cli-co with the overlap the backfill recorded. Close it only if its
  whole scope (older ambiguous and title-mismatch skips) is resolved.

## Out of scope

- **The Mac's own `bob`.** The Copy card shells out to `bob` on the Mac. Whether the
  Mac's build has `ref code` and `ref links` is tracked by bob-cli-cx (the Mac was
  offline during planning).
- **References older than 2026-10-12** that the cc.6 backfill skipped (bob-cli-co).
- **Creating references** for hook-matched reports that never became one. List them
  instead.
- **Re-rendering** existing PDFs to get render-time catalogs. That would change
  annotated PDFs.

## Risks

- **A report changed between the hook's commit and its render.** The hook runs right
  after its commit, so this is rare. Verification catches it, and the `HEAD` fallback
  covers it.
- **A reference PDF overwritten or reread after birth.** Its current PDF may not match
  the hook-time bytes. Step 7's fallback handles it, and the rest are recorded.
- **Orphans created between now and deployment** are adopted in step 6 and caught by
  step 8's doctor check.
- **The vault-sync daemon races the backfill.** `attach-source` runs under the
  reference-mutation session (`bob_sync.lock` then `scan.lock`), as in the cc.6 and ct.3
  backfills.
