---
tier: epic
title: Finish ref create rereads - landing fixes for bob-cli-c9.3.1
goal: 'Every gap the bob-cli-c9.3.1 land review found is fixed: rereads join only
  the reference they really repeat, attach and intake keep working on later readings,
  partial overlaps never reread, every route honours the reread contract, and the
  docs and tests match the shipped behavior, so the bob-cli-c9.3.1 landing can resume
  and close.

  '
parent_bead: bob-cli-c9.3.1
decisions:
  exact_wins_over_collection:
    ask: When a URL set exactly matches one family but also sits inside a larger collection,
      reread the exact family?
    default: true
    why: -f recaptures only the exact URL set; that set is unambiguous here
    answer: true
  full_scan_collision:
    ask: How should a full bob ref scan handle a queued __vN PDF whose name another
      host took?
    choices:
      abort: Whole scan stops before any move, like today's same-path destination
        collision
      skip: Only that reread stays queued with a collision report; other queued items
        still file
    default: abort
    why: matches the existing plan_xlib_intake collision contract
    answer: abort
phases:
- id: intake_recheck
  title: Locked reread recheck for audio units, full scan, and previews
  size: small
  depends_on: []
  description: 'intake_recheck: stop the locked recheck from refusing audio attach
    onto __vN readings, add the whole-bundle recheck to the full-scan xlib move path,
    and make dry-run previews agree.'
- id: family_matching
  title: Identity-correct family matching and attach targets
  size: medium
  depends_on:
  - intake_recheck
  description: 'family_matching: title identity on title routes, exact-only URL family
    candidates, deleted-root naming, refusal reading numbers, the -o guard, and newest-reading
    -L attach for single sources and collections.'
- id: route_guards
  title: Route guards, audio, vault ordering, redirects, and parser edges
  size: medium
  depends_on:
  - family_matching
  description: 'route_guards: overwrite exact-output permission, narrated-reading
    hint, no audio auto-discovery on rereads, library-PDF -f -L refusal, vault-note
    preflight ordering, redirect guard, parser raw-argv fixes, and typed overwrite
    fallbacks.'
- id: contract_docs
  title: Docs, cleanup, test hardening, and acceptance evidence
  size: small
  depends_on:
  - route_guards
  description: 'contract_docs: correct help and docs, remove stale interim markers,
    add producer, inline-filing, and byte-snapshot tests, and run the full acceptance
    checks.'
proposed_by: bbugyi200.athena.bob-cli-c9.3.1.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-25 19:48:32
status: done
bead_id: bob-cli-c9.3.1.6
---

- **PROMPT:** [prompts/202610/ref_reread_landing_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_reread_landing_fixes.md)
- **PARENT:** [202610/ref_create_reread.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_create_reread.md)
- **BEAD:** [bob-cli-c9.3.1.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-c9/bob-cli-c9.3.1.6.md)

# Finish ref create rereads: landing fixes

The land review of epic `bob-cli-c9.3.1` (plan `plan:202610/ref_create_reread.md`) found
that the five closed phases left regressions and unmet plan items. This child epic fixes
exactly those. Its `parent_bead` is `bob-cli-c9.3.1`: when this child epic lands, its
land agent resumes the interrupted `bob-cli-c9.3.1` landing. No phase here closes
`bob-cli-c9.3.1`, `bob-cli-c9.3`, or any ancestor, and no phase edits that parent plan
file.

Before starting, read the parent plan with `sase bead read bob-cli-c9.3.1 -r "<why>"`
and its plan file. Its "Shared contract and implementation seams" and "Validation and
land-agent completion" sections still govern. Its inherited decisions are final:
`legacy_force = no`, `keep_created = yes`, no overwrite or `recaptured:` implementation,
and no memory changes, including the glossary. Never edit the `Cargo.toml` version or
`CHANGELOG.md`. Use the existing test helpers (`bob_command`, `write_highlights_pdf`,
`configure_test_git_identity`, the fake clip/curl/pandoc/listen tools in
`tests/cli/highlights/create.rs`, the temp-vault fixtures in
`tests/cli/highlights/scan_intake.rs`). Never run against `~/bob` or the real network.

**Working rule for every item.** First write the failing test that reproduces the
reported behavior. If an item does not reproduce on the current tree, record that on
your phase bead with the test you used and move on. Do not invent a different change.
Line numbers below are approximate as of commit `9923063e`.

## Deliberate non-goals (decided at landing; do not implement)

- URL reread dry runs keep using the network. This matches the documented URL dry-run
  table in `docs/highlights-create.md`. The reread decision and the `reread:` line are
  already computed before fetch.
- `-f overwrite -L` on a filed reference keeps the old companion-audio attach
  permission. The parent plan keeps audio replacement permissions under either mode.
- Building the library and task index once per `bob ref create` is by design.
- Whole-bundle checks for ordinary (non-`__vN`) full-scan moves, and redirect-aware
  dedupe for ordinary captures, are pre-existing and out of scope.

## Phase intake_recheck

Files: `src/native/highlights_ref/intake.rs`, `src/native/highlights_ref/doctor.rs`,
tests in `tests/cli/highlights/scan_intake.rs` (plus `create.rs` for `-L`).

1. **Audio units.** `recheck_reread_occupancy` (`intake.rs` ~683) runs for every unit
   whose `library_pdf()` stem is `<base>__vN`. For `IntakeUnit::Audio` that PDF is the
   already-filed library PDF. Its own PDF, ref note, and id therefore count as
   occupants, so every `-L` attach or `scan -i` audio drain onto `foo__v2`, or onto a
   natural `protocol__v2`, is refused with "reread … is already filed". Audio units
   attach to an existing bundle. Skip the reread recheck for them. The existing audio
   destination and companion checks in `validate_unit_collisions` still apply. For
   `IntakeUnit::LibraryPdf`, find out whether its ref note can already exist at ingest
   time. If it can, treat that note and its id as the unit's own.
2. **Full-scan path.** A full `bob ref scan` moves queued xlib PDFs through
   `plan_xlib_intake` / `execute_xlib_intake_counting` (`doctor.rs` ~964/~1138). That
   path checks only same-path destinations; `IntakeSession::ingest` is not used.
   `create -f` queues `xlib/<type>/<root>__vN.pdf` whenever inline intake cannot run, so
   the epic's guarantee ("locked races file nothing over an occupant") is unmet there.
   Extract the occupancy check from `recheck_reread_occupancy` into one shared helper:
   `split_reread_stem` gate, `DefaultTargetOccupancy::occupants_except` minus the unit's
   own files and companions. Call it from both intake and `plan_xlib_intake` for `__vN`
   stems only.

   > [!decision] full_scan_collision = abort A taken name reports through the existing
   > `plan_xlib_intake` collision error before any move: nothing moves, and every queued
   > source stays.

   > [!decision] full_scan_collision = skip Only that queued reread is left in `xlib/`,
   > with a collision line naming the occupants and "queued source preserved". Every
   > other queued item still files, and the scan exit status reports the skipped item.

3. **Previews.** `preflight_unit` (`intake.rs` ~1142) and the inline `WouldIngest`
   preview must run the same recheck. Then `scan -i -n` and `create -d` predict exactly
   what a real run does.
4. **Tests:**
   - `-L` (fake listen) on a captured URL whose newest reading is `foo__v2` files the
     episode.
   - Same for a natural `protocol__v2` with no lineage.
   - `scan -i` drains a queued audio for `foo__v2`.
   - Full `bob ref scan` with a queued `foo__v2.pdf` and a fabricated other-type note,
     or an id-only occupant, follows the chosen decision branch. The queued PDF bytes
     are preserved.
   - `scan -i -n` predicts the refusal.
   - The existing `full_scan_files_queued_reread_with_lineage` test and the drain tests
     stay green.

## Phase family_matching

Files:
`src/native/highlights_ref/{recapture.rs,collection.rs,create.rs,clip.rs,birth.rs,duplicate.rs,stamp.rs}`.

1. **Title identity on title routes (blocking regression).** `families_for_stem` /
   `preflight_for_stem` (`recapture.rs` ~304–571) match on stem alone. Before the epic,
   title routes went through `plan_default_target` with `DefaultTargetIdentity::Title`
   (`stamp.rs` `candidate_is_same_reference` / `title_matches_pdf`). Under those rules,
   a same-stem occupant whose library or intake PDF carries a different page-1 marker
   title is a different reference and yields `<stem>_2`. Ref-note-only occupants were
   always different. Today such a capture refuses as "already captured", even with `-P`.
   With `-f` it becomes an unlinked `<stem>__v2` that takes the stranger's parent and
   type, while birth finds no lineage.
   - A family member at the probed stem selects only when it is the same reference by
     those pre-epic rules: Markdown and local PDF compare titles via the shared
     `stamp::normalize_title`; same-day vault snapshots compare `source_note`. Otherwise
     return `Ok(None)` so ordinary `_2` naming resumes.
   - Make `birth.rs` (~195–231) use the same normalization so create and birth agree.
   - Restore the title-route remedy `-N <name>  capture under a different name`
     alongside `-f  read it again as <root>__vN`.
   - Offer `-L` only on routes with an attach path. Markdown, vault-note, and
     outside-vault PDF targets have none.
2. **Exact-only URL candidates (blocking).** `resolve_for_url_keys` (~221–297) returns
   `Ambiguous` for any 2+ overlapping families without checking exactness. Nothing
   downstream checks the chosen family's set either, so `URL1 URL2 URL3 -f -N a` can
   reread single-URL family `a` with three URLs. Fix it this way:
   - Candidates are only families whose complete key set equals `wanted`.
   - One exact family is `Selected`; 2+ exact families are `Ambiguous`.
   - No exact family but any overlap is `PartialOverlap`, naming every overlapping
     family's members.
   - `finish_selection` must refuse to plan a family whose `source_keys != wanted` (a
     defensive guard).
   - Count overlapping _URLs_, not families, in "N of M URLs are already in the
     library".

   > [!decision] exact_wins_over_collection When one exact family exists, other
   > partially overlapping families (for example a collection containing the URL) are
   > not candidates: `-N <collection>` exits 2 as "not a reading of", and the exact
   > family rereads.

   > [!decision] exact_wins_over_collection = no Any partially overlapping family
   > refuses the whole capture as a partial overlap under every force mode, even when an
   > exact family also exists.

3. **Deleted root.** When v1 is deleted, `lineage.rs` falls back to the smallest live
   path (`foo__v2.md`) as the family key. `select_family` then sets
   `root_stem = foo__v2` and allocates `foo__v2__v4`; birth gives task id
   `ref-foo-v2-v4`. Reopen (`birth.rs` ~374) instead takes the root from the last
   `earlier:` entry. Use one rule for allocation, birth, and reopen: when the family's
   key row itself carries Reading `earlier:` entries, the root stem is the stem of the
   oldest `earlier:` Reading target. Do not change `lineage.rs`'s family key, and never
   infer reading numbers from filenames. Expected: with `foo` deleted and `foo__v2` and
   `foo__v3` live, a reread allocates `foo__v4` with task id `ref-foo-v4`.
4. **Refusal reading numbers.** `filed_refusal` (~616) builds copies with
   `copy_from_occupant`, which sets `reading: None`. Set each copy's `reading` from its
   `FamilyMember`, so multi-reading refusals show `reading vN` newest-first.
5. **`-o` guard.** `finish_selection` raises "a reread is named <root>__vN; drop -o"
   even with no force. Without force, the normal `already captured` refusal (exit 1)
   must win. Keep the exit-2 guard under reread force. Overwrite keeps its interim
   message.
6. **`-L` attach targets the newest reading.** Collections: `check_collection_overlap`
   (`collection.rs` ~1613) requires `refusing.len() == 1`, so after a reread the family
   (v1 and v2 share the set) is refused as a partial overlap by `attach_gate`. Group
   refusing hits by library family using the index rows: a family whose members all
   carry exactly the wanted set is one exact identity, and attach uses its newest
   reading. Single sources: `find_refusing_hit` returns the first hit by path, which is
   v1 (`foo.md` sorts before `foo__v2.md`). Make the attach callers pick the family's
   newest reading. These are the `create.rs` `check_duplicate_before_parent` and URL
   route, and `clip.rs`. Earlier readings must keep their bytes.
7. **Tests:** unit tests in `recapture.rs` plus CLI tests:
   - Different-title Markdown over a filed same-stem note gives `_2`, both without force
     and with `-f`, and prints no `reread:`.
   - Same title refuses with the `-N` and `-f` remedies.
   - `-f` rereads.
   - Three URLs against two single-URL families: partial overlap under no force, `-f`,
     and `-f -N a`.
   - One URL filed alone and inside a collection follows the decision.
   - Deleted root.
   - Refusal reading numbers.
   - `-o` without force.
   - Collection `-L` after a reread attaches to the newest reading.
   - Single-source `-L` after a reread attaches to the newest reading, and the earlier
     note, PDF, and audio bytes stay unchanged.

## Phase route_guards

Files:
`src/native/highlights_ref/{create.rs,clip.rs,collection.rs,force.rs,pdf_target.rs,mod.rs,recapture.rs}`.

1. **Exact-output replacement under `-f overwrite`.** `triage_overwrite_url` (`force.rs`
   ~256; callers `create.rs` ~1146 and ~1388) and the collection gates (`collection.rs`
   `attach_gate` / `fresh_queued_gate`) consider only recorded sources. So
   `bob ref create URL -o ~/x.pdf -P p -f overwrite`, with an existing `~/x.pdf`, exits
   1 "nothing to overwrite", although `-f` used to replace it. When `-o` names an
   existing file, keep the old exact-output replacement permission (`force.is_some()`)
   instead of treating the source as fresh. Check that the Markdown and PDF routes
   behave the same. Add tests for URL, collection, and Markdown.
2. **Narrated-reading hint on a plain reread.** On a reread without `-L` or `-a`, when
   an earlier reading has audio, print the parent plan's hint
   `hint: vN is narrated (<vault-relative audio path>); add -L to narrate this reading`.
   Name the newest narrated reading. Keep the existing `-f -L` hint. Route create, clip,
   and collections through one helper next to `print_reread_listen_hint` (`recapture.rs`
   ~738), and test it.
3. **No audio auto-discovery on rereads.** On a reread, Markdown audio discovery
   (frontmatter `audio.episode_id`, sibling `<stem>_narration.md`; discovery sites near
   `create.rs` ~3521 and ~4030) must not run unless `-a/--audio` is given. Earlier audio
   is never inherited. Test with a source whose earlier reading got discovered audio.
4. **Library PDF as TARGET.** `-f -L -P X lib/<type>/foo.pdf` currently attaches.
   Preflight returns nothing for lib/xlib paths (`create.rs` ~1009–1022), and attach
   (~2772–2790) runs before the `pdf_target.rs` (~467–482) refusal. Under reread force,
   the library-PDF refusal (with its `source_url` hint) must win, whatever `-L` or `-P`
   say. Without force, `-L` still attaches. Test both, including `-L` without force on a
   library PDF, which the existing test's comment claims but never runs.
5. **Vault-note preflight ordering.** The vault route resolves its parent (`create.rs`
   ~872–883) before `preflight_for_stem` (~1790). A missing parent therefore exits 2
   before the duplicate refusal, and the newest reading's parent default is never used.
   Compute the dated stem (honoring `BOB_NOW` and `-N`) and run the shared preflight
   before parent resolution. New-day snapshots stay ordinary dated creates. Test
   same-day reread without `-P`, the refusal before the missing-parent error, and the
   new-day case.
6. **Redirect guard on rereads.** The post-fetch rechecks reuse the pre-fetch key, so
   they only catch races: `create.rs` PDF URL ~3049–3070, ~3350–3372, and `clip.rs`
   ~496–520. Also compute the dedupe key of the fetched final URL (`fetch.rs`
   `FetchResult.final_url`, and the adapter's `final_url` in `clip_adapter.rs`). If it
   differs from the requested key and is recorded by a filed or queued reference outside
   the selected family, refuse before writing, naming that reference (exit 1, nothing
   written). Rereads only. Test with the fake curl and adapter.
7. **Parser raw argv.**
   - `mod.rs` `run()` hands the decoder an empty raw argv unless `args[0]` is `create`
     or `clip`. So `bob ref -n create --force=reread overwrite URL` adopts `overwrite`
     instead of exiting 2. Locate the subcommand token after the top-level options.
   - `force.rs` `is_explicit_force` (~144) must also treat a clustered short ending in
     `f=` (`-df=reread`) as explicit.
   - Add unit tests and one CLI test proving
     `bob ref -n create --force=reread overwrite URL` exits 2 with no fetch.
8. **Typed overwrite fallbacks.** Several fallbacks decide by searching error text for
   "already captured" and print the stem-less "not available yet for filed references":
   `create.rs` ~1237/1262/1308/1719/1741, `clip.rs` ~282/302/445/460, `collection.rs`
   ~1652. For each one, find out whether it is still reachable after the shared
   preflight. Delete the unreachable ones. Make reachable ones use the typed recapture
   result and `overwrite_filed_unavailable_for(<next stem>)`. Add no new string matching
   on rendered errors.

## Phase contract_docs

1. **Docs and help:**
   - `docs/highlights-create.md`:
     - The same-day vault-note snapshot text (~118–121) must say `-f` rereads it as
       `<dated-root>__vN`.
     - Fix the `-f  read` spacing (~522).
     - Describe the title-route remedies and the narrated-reading hint.
     - Describe newest-reading `-L` attach and exact-output overwrite permission.
   - `docs/highlights-ref-sync.md` (~2020): task ids are `ref-<root-slug>-vN` (the `-vN`
     is reserved before 44-char truncation, and the collision `-2` follows it),
     `Title (vN)` aliases, and birth `earlier:`.
   - `docs/ref.md` (~904 and Identity): single-source reread naming `<root>__vN`.
   - The `bob ref create` help Recapture text must cover:
     - `-N` as the later overwrite selector;
     - legacy-only matches and new-day snapshots as cases that need no force.
   - Avoid duplicating clap's own `[possible values: …]` in the `-f` row.
   - Keep options alphabetical. Regenerate with `just docs-gen` (never hand-edit
     generated files) and pass `just docs-check`. Read `docs.md` and `cli_rules.md` via
     `/sase_memory_read` first.
2. **Cleanup.** Remove the stale `#[allow(dead_code)]` and "consumed by single_sources"
   / "rereads pending" comments: `force.rs` ~222–247, `duplicate.rs` ~92–93,
   `recapture.rs` ~6–8, plus any other epic-era interim markers that are now false.
3. **Test hardening:**
   - Producer retry tests: `ingest_url` (`ingest.rs` ~610), reached through its real
     callers (Keep import, jobs, capture), never mints `__vN` against a filed family.
     Today only `highlights_create_retries_without_force_never_mint_readings` covers
     `bob ref create`.
   - An inline-filing reread (intake not kept) files the library PDF, a note with
     `earlier:`, and a versioned task in one run.
   - Byte snapshots of the earlier reading's note, PDF, sidecars, audio, and task line
     across local-PDF and URL rereads. Current tests compare note bytes only, or check
     `len > 0`.
4. **Acceptance.**
   - Run `just check`, `just docs-gen`, and `just docs-check` once everything is in. Use
     `/sase_monitor` for long commands.
   - Re-walk the parent plan's "completed result must satisfy" list.
   - Leave a concise evidence note on this phase: exact commands, results, any confirmed
     base failures (cite a tracking bead if known), and the `sase bead epic-symbols`
     output.

## Validation

Each phase runs focused tests for its changed paths plus `cargo fmt --check` and the
repo clippy gate (`just check`'s clippy line). Before closing only its own phase, it
runs `sase bead epic-symbols <its phase>`. Temporary unused symbols must be consumed,
removed, or keyed to a still-open owning bead; never loosen the lint gate. If a check
fails, establish whether it also fails on the clean base. For a confirmed base failure,
append `PROPOSED FOLLOW-UP:` to your phase bead with evidence and close the completed
phase. Commit descriptions use `fix(ref)` / `fix(highlights)` scopes. A user-visible
contract change (help text) still uses no `!` unless it breaks a CLI or JSON contract.
