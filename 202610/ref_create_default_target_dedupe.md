---
tier: tale
title: Deduplicate the default bob ref create target filename
goal:
  Without -o/--output, bob ref create (and the shared clip and ingest planners) picks
  <stem>_2, <stem>_3, … when the derived name is taken by a different reference, still
  refuses when the same reference is already tracked, and leaves -o paths unchanged.
size: medium
decisions:
  local_identity:
    ask:
      How should Markdown and local-PDF captures decide a stem occupant is the same
      reference?
    default: title
    why:
      Works for existing captures and refuses exact re-runs; refuses only on same stem
      and title.
    choices:
      title: Same marker or Info title means same reference (refuse); otherwise suffix.
      always_suffix:
        Never refuse local sources on a name clash; re-running makes a duplicate _2.
      stem:
        Keep refusing local sources on any stem clash; auto-suffix URL captures only.
    answer: title
  dedupe_ingest:
    ask:
      Should the capture worker / gkeep URL ingest also auto-suffix colliding default
      names?
    default: true
    why:
      It shares the default-target planner; otherwise it keeps failing with a collision
      error.
    answer: true
proposed_by: bbugyi200.athena.0y6
decided_by: auto
create_time: 2026-10-08 09:00:46
status: wip
---

# Deduplicate the default `bob ref create` target filename

## Problem

`bob ref create` derives a default target `<xlib-dir>/<ref-type>/<stem>.pdf` whenever
`-o, --output` is not given. Today that default path goes through the same strict
`refuse_target_collisions` check as an explicit `--output` path
(`src/native/highlights_ref/stamp.rs::plan_default_target`). So a _different_ reference
whose derived stem happens to collide with an existing capture fails outright:

- `target PDF already exists: …; pass --force to overwrite it`. This is an intake PDF
  that has not been scanned yet. Worse, `--force` would overwrite a different
  reference's queued PDF.
- `refusing to create … because the library destination already exists`. The reference
  was already scanned into `lib/`.
- `… library destination sidecar already exists` and
  `… Highlights would treat the existing Markdown file as its sidecar`.

`docs/highlights-create.md` ("Markdown and local PDFs outside the vault whose planned
library destination already exists keep refusing; identity is not proven by stem alone")
documents this as intended, so the contract and its docs both change.

Desired behavior: an automatically chosen name that is taken by a **different**
reference picks the next free name (`<stem>_2`, `<stem>_3`, …). `bob ref create` must
still fail when the **same** reference is already tracked in the vault. An explicit
`-o, --output` path keeps today's strict behavior unchanged.

The same default-target planner is shared by the article (clip) route
(`src/native/highlights_ref/clip.rs::capture_article`) and by the URL ingest used by the
capture worker and `bob gkeep pull` (`src/native/highlights_ref/ingest.rs`). Those
callers should get the same dedupe, so every automatically chosen intake name behaves
the same way. Today the ingest callers fail with a permanent `collision` error instead.

## Behavior contract

### 1. Scope

The new rules apply only when `-o/--output` is absent. That includes runs with `-N`
and/or `-t`, because those still go through default target derivation. `--output`
planning (`plan_exact_output`, `plan_exact_output_guarded`) stays byte-for-byte
unchanged.

### 2. Candidate walk

The candidates are `<stem>`, `<stem>_2`, `<stem>_3`, … up to `<stem>_999`. A suffixed
candidate is still a valid name under `clip_url::validate_name`.

A candidate stem `s` under ref type `t` is **occupied** when any of these exist:

- `xlib/t/s.pdf`
- `xlib/t/s.md` or `xlib/t/s.textbundle` (intake Highlights sidecars)
- `lib/t/s.pdf`
- `lib/t/s.md` or `lib/t/s.textbundle` (library sidecars, i.e.
  `library_destination_sidecars`)
- `ref/t/s.md`, the note path `bob ref scan` would write for that PDF (see
  `io.rs::ref_note_path` / `pdf_path_metadata`). Scan merges into whatever note already
  sits there and never checks who owns it, so an existing note must block the stem.

Companion audio is deliberately not part of occupancy. Existing audio planning
(identical bytes reused, different bytes refused, mirrored library audio refused) keeps
working, and so does the "pre-placed companion at the target stem" reuse.

Walk the candidates in order:

1. **Free candidate:** choose it.
2. **Occupied by the same reference** (see the identity rules below):
   - If the matching occupant is the intake PDF and `--force` is set, choose this
     candidate and overwrite it. This is the existing "`--force` only overwrites the
     same intake target" semantics. The existing library-destination and sidecar checks
     still apply to it.
   - Otherwise refuse:
     - Library or ref-note occupant: `already captured as <path> …`.
     - Intake occupant without `--force`:
       `already queued in <path> …; pass --force to overwrite it`.
     - Both refusals keep today's
       `hint: to add audio to that capture, run bob ref create <existing PDF> --listen`.
     - Keep the `already captured` / `already queued` wording. `ingest.rs::classify`
       maps those phrases to `IngestErrorKind::Collision`.
3. **Occupied by a different reference:** continue to the next candidate.

If all 999 candidates are taken, fail with
`no free default filename for <stem>; pass -N/--name or -o/--output`.

`--force` never selects or overwrites a candidate whose occupant is a different
reference. This removes today's footgun where `--force` clobbered an unrelated queued
intake PDF.

### 3. Identity: "same reference" per route

The match is checked per candidate. It considers only PDFs at the candidate (intake or
library) and, for URL routes, recorded ref notes. Sidecar-only, ref-note-only (for local
routes), and unmarked-PDF occupants always count as a different reference.

- **URL routes:** PDF URL, arXiv, web article (clip), and ingest.
  - The source URL dedupe key is the identity. The existing pre-fetch and post-plan
    `sources::check_dedupe` / `find_refusing_hit` logic stays as is and still refuses
    PDF-backed ref-note hits and queued-intake hits anywhere in the vault.
  - In the walk, a candidate is the same reference when a `RecordedSource` carries the
    capture's dedupe key and either:
    - its `path` is the candidate's intake PDF, or
    - it is a PDF-backed ref note at the candidate's ref note path.
  - arXiv checks both `paper.dedupe_key()` and `url.dedupe_key`.
  - In practice the walk only decides the `--force` same-intake overwrite (and race-time
    hits). Everything else is a different reference and gets a suffix.
  - Example: a library PDF at `lib/blogs/<slug>.pdf` with no matching `source_url`/`url`
    now yields `<slug>_2` instead of refusing.
- **Local routes:** Markdown and local PDF.

  > [!decision] local_identity = always_suffix
  >
  > Local routes pass identity `None`: every occupant counts as a different reference,
  > so re-running on the same source creates `<stem>_2`. Skip the title helpers and the
  > Info-title reader. Replace the local "same title refuses" tests with "re-run
  > suffixes" tests.

  > [!decision] local_identity = stem
  >
  > Local routes keep today's strict behavior: they call the strict planner (base stem
  > plus `refuse_target_collisions`, no walk). Only URL routes walk. Skip the title
  > helpers and keep the existing local-route collision tests as they are. In the docs,
  > state that local captures still refuse any stem clash.

  > [!decision] local_identity = title
  - There is no source URL, so the identity is the create-time title. A candidate is the
    same reference when its intake or library PDF has either of these equal to the
    planned title (after trimming, collapsing whitespace, and comparing
    case-insensitively):
    - the page-1 marker `title`, via `marker::read_pdf_marker` + `parse_marker`; sync
      keeps this aligned with ref-note edits;
    - or the raw PDF Info `Title`, which `stamp` sets at creation.
  - Add a small raw Info-title reader in `pdf_meta.rs`. Do not reuse
    `pdf_info_metadata`: its `plausible_title` filter drops titles equal to the stem.
  - The planned title is the `-T` override if given, otherwise the route's derived
    title: Markdown frontmatter, then H1, then stem; for PDFs, the Info title, then the
    filename.
  - Result: re-running `bob ref create report.md` or `bob ref create paper.pdf` on the
    same source still refuses. A different source with the same filename but a different
    title is suffixed.
  - Conservative failure mode: two genuinely different sources with the same stem _and_
    the same title refuse. The refusal message should name the matching title so the
    user knows to pass `-N`.

- The existing guards stay ahead of the walk, unchanged:
  - in-vault local PDF identity (`pdf_target::check_local_pdf_identity`);
  - a source that is already marked (`refuse_marked_pdf`);
  - `--listen` attach mode for already-captured URLs and in-vault PDFs.

### 4. The final stem drives everything downstream

Everything derived from the stem uses the final (possibly suffixed) stem:

- target, sidecar, and `library_destination`;
- marker `id`: URL routes always stamp `id` = stem; local routes stamp it with `-i`, and
  with `-N` the name;
- `--listen` (`plan_listen_flow` scratch audio name and `<target>.mp3`);
- companion-audio destinations;
- dry-run and success reports (`pdf:`, `id:`, `next:`).

When the walk skipped at least one candidate, dry-run and success output print one extra
stdout line just before `pdf:`:

`renamed: <base stem>.pdf is taken by <first occupant path>; using <final stem>.pdf`

### 5. Post-listen re-check

The re-check after a long `--listen` run still calls `refuse_target_collisions` on the
_chosen_ target (`create.rs::stamp_listen_and_install_pdf`, and the post-listen block in
`clip.rs`). If a race took the chosen name mid-run, it refuses with the kept-audio hint.
Do not re-walk there.

## Implementation

### A. `src/native/highlights_ref/stamp.rs`

- Add a default-target planner that walks the candidates. Suggested shape:
  - `plan_default_target(config, stem, ref_type, force, identity)`.
  - `identity` is a route-supplied predicate. For example, a small enum
    `DefaultTargetIdentity { Url { keys: Vec<String>, recorded: &[RecordedSource] }, Title(String), None }`
    or a `&dyn Fn` receiving the candidate's occupant paths. The `None` variant is for
    unit tests and "always different" callers.
  - It returns a `TargetPlan` that additionally carries the final `stem` and an optional
    `renamed_from` (base stem + first occupant path) for the report line.
  - `TargetPlan` derives `PartialEq`, so update its literals and tests. `--output` plans
    set `stem` from the output file stem and `renamed_from: None`.
- Factor occupancy into a helper, e.g.
  `default_stem_occupants(config, ref_type, stem) -> Vec<PathBuf>`, checking the six
  path kinds above. Reuse `library_destination_sidecars`.
- For the chosen candidate, still run `refuse_target_collisions`. For a free candidate
  it passes trivially. For a `--force` same-reference intake overwrite it keeps the
  library and sidecar refusals. This keeps a single source of truth for the strict
  checks that `--output` and the post-listen re-check use.
- Put the title-identity helper (marker title or raw Info Title equals the planned
  title, normalized) near the planner, or in `pdf_target.rs`. Add the raw Info `Title`
  reader to `pdf_meta.rs`.

### B. `src/native/highlights_ref/create.rs`

- `plan_target_for` takes the identity and forwards it to the default planner only when
  `output` is `None`.
- Markdown route (`plan_markdown`): the title is already computed before
  `plan_target_for`; pass `Title(title)`. With `-i`, recompute `id` from the final stem,
  and recompose the marker after planning if needed. The marker is currently composed
  before `plan_target_for`, so move its composition after planning. `CreatePlan.target`
  / `target_plan()` then carry the suffixed path, so the listen stem in
  `create_markdown_route` (taken from `plan.target`) follows automatically.
- Local PDF route: pass `Title(pdf_plan.title)`. Use the final stem for the `-i` id and
  for `plan_listen_flow` and the scratch `<stem>.pdf` (replace the `pdf_plan.stem`
  uses).
- PDF URL and arXiv routes: pass `Url { keys, recorded }`. Replace every downstream
  `pdf_plan.stem` use (marker `id`, `plan_listen_flow`, the `id:` report line) with the
  final stem. Keep the post-plan `check_dedupe_with_listen_hint` / `find_refusing_hit`
  calls as they are.
- `print_pdf_dry_run`, `print_plan`, and the success reports print the `renamed:` line
  when `renamed_from` is set.
- Update the `Output:` paragraph of the long help in `command()`. Say that the default
  target becomes `<stem>_2`, `<stem>_3`, … when the name is taken by a different
  reference, that the same reference refuses, and that `-o` keeps the exact path. If a
  help snapshot under `tests/fixtures/help/` covers this text, update it.

### C. `src/native/highlights_ref/clip.rs` (article route)

- Both `pre_plan` and the final plan in `capture_article` use the walking planner with
  `Url { … }` identity.
- `compose_marker(… Some(&stem) …)` currently runs before the final plan. Move it after
  planning and use the final stem. Do the same for
  `plan_listen_flow(&plan, &workdir, &stem)` and the `id:` report line.
- `pre_plan` still fails fast for same-reference refusals before the adapter runs.
- A pre-plan rename is only advisory. The final plan re-walks with the final stem; the
  final stem can differ when no URL slug existed.

### D. `src/native/highlights_ref/ingest.rs`

> [!decision] dedupe_ingest = no
>
> Leave ingest on the strict planner: the base stem plus `refuse_target_collisions`,
> exactly today's behavior. Keep a strict entry point in `stamp.rs` for it. In the
> Ingest boundary docs, state that ingest does not auto-suffix.

> [!decision] dedupe_ingest

- Switch the four `plan_default_target` call sites (PDF URL, arXiv, article pre-plan,
  article final plan) to the walking planner with `Url { … }` identity and
  `force = false`. The marker `id` uses the final stem.
- `IngestOutcome::Created` reports the suffixed PDF path.
- Leave `classify` alone; the `already captured` / `already queued` wording keeps
  mapping to `Collision`.

### E. Docs

- `docs/highlights-create.md`:
  - Replace the "Markdown and local PDFs … keep refusing; identity is not proven by stem
    alone" paragraph with the new contract (sections 2–4 above).
  - In the Dedupe section, note that `--force` only overwrites the same reference's
    intake PDF and that default names auto-suffix.
  - In the Ingest boundary section, note that ingest uses the same suffixing.
- `docs/highlights-ref-sync.md`, around the `create` target paragraphs:
  - Scope "refusing to create when that archived library PDF or sidecar already exists"
    and "An existing target PDF requires `-f, --force`; a same-stem Markdown file … is
    always refused" to `--output` paths.
  - Describe suffixing for default targets.
- `docs/highlights-clip.md`: if it restates the default-target collision rule, align it.

## Tests

### Unit tests (`stamp.rs`)

Rework the existing default-target tests: `target_refuses_existing_pdf_without_force`,
`target_refuses_highlights_markdown_sidecar_even_with_force`,
`target_refuses_existing_library_pdf_even_with_force`, and
`target_refuses_existing_library_sidecar`. Keep every `exact_*` test unchanged as the
`--output` regression guard. Cover:

- free base → base stem, `renamed_from: None`;
- each occupant kind bumps to `_2` (intake PDF, intake `.md`, library PDF, library `.md`
  / `.textbundle`, `ref/<t>/<s>.md`); `_2` also occupied → `_3`;
- same-reference intake occupant: refuses without `--force`; with `--force` the base is
  chosen;
- same-reference library occupant: refuses with and without `--force`;
- different base, same reference at `_2` → refuses (the walk does not skip past a
  matching occupant);
- the title identity matches on marker title or Info Title, and normalizes whitespace
  and case;
- the 999-candidate cap error.

### CLI tests (`tests/cli/highlights/create.rs`)

Update these:

- `highlights_create_refuses_existing_library_pdf_with_or_without_force`: the library
  PDF title "Archived Report" differs from the source `# Report`. With `-d` it now plans
  `xlib/chat/report_2.pdf` and prints the `renamed:` line. Add a sibling where the
  library marker title is `Report`: it refuses with and without `--force` and writes
  nothing.
- `landing_library_collision_hints_listen_attach`: make the library PDF carry title
  `Report` so it is the same reference and still refuses with the `--listen` hint. Add a
  bare-PDF variant that now renames.
- `highlights_create_folded_refuses_library_and_dedupe_collisions_early`: a library PDF
  at the slug without a matching URL now captures to `<slug>_2`. Split the test so the
  PDF-backed URL-note refusals, which never call the adapter, stay pinned.
- Keep `highlights_create_folded_force_overwrites_the_same_intake_target` passing as is.

Add these:

- **Local PDF re-capture:** the second run refuses with `already queued`; `--force`
  overwrites the same intake PDF. After moving the PDF to `lib/` (simulated scan) it
  refuses even with `--force`.
- **Local PDF, same filename, different Info title:** `xlib/papers/<stem>_2.pdf`; with
  `-i` the marker `id` is `<stem>_2`.
- **PDF URL whose slug collides with a queued intake PDF for a different `source_url`:**
  installs `<stem>_2.pdf`, marker `id` = `<stem>_2`, and the `renamed:` line appears.
- **`-o` pointing at an existing PDF** still refuses without `--force` (add one if no
  existing test pins it).
- **Ingest:** a different URL with a colliding slug returns `Created` at `<stem>_2`.
  Adjust `ingest_characterizes_url_failure_modes` and the ingest unit tests if they pin
  the old collision.

## Validation

Run `just check` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
`cargo test --no-fail-fast`) and get it green. Do not touch the vault (`~/bob`) during
testing; every test uses temp vaults via `-b`.
