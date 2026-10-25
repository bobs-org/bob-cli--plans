---
tier: epic
title:
  "bob ref create -f|--force[=reread|overwrite]: recapture a source you already have"
goal: "A source that is already a reference can be captured again on purpose. Bare -f
  adds a new <root>__vN reading linked to the earlier ones. -f overwrite recaptures a
  reference in place and keeps its path, id, and reading task. Before that, anything you
  made in the old note is frozen into ref_history/. Without -f, a duplicate is refused
  before any fetch, with one clear message that names the existing copies and the exact
  fix.

  "
decisions:
  legacy_force:
    ask:
      Should -f on a legacy note without a PDF start a reading family (a __v2 linked by
      earlier:)?
    default: false
    why: -f only changes what create would refuse, and legacy-only matches never refuse
    answer: false
  keep_created:
    ask:
      Should overwrite keep the note's created date and record the new material as
      recaptured?
    default: true
    why:
      Overwrite keeps the reference's identity; created dates the reference, not the
      material
    answer: true
  memory_decision_record:
    ask:
      Add a decisions record for recapture (reread adds __vN; overwrite keeps identity,
      freezes to ref_history/)?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  memory_glossary_ref_task:
    ask:
      Correct the Reference Task glossary strand to describe the ^ref-<slug> reading
      task in the parent note?
    memory:
      - glossary:reference-task
    default: true
    corrects:
      exact trailing block ID `^ref`, that tracks progress through the external
      material.
    answer: true
phases:
  - id: family
    title: Reading families in the library read model
    depends_on: []
    size: medium
    description:
      "family: teach ref_library the note-local earlier: list, reading numbers, and
      families. Doctor stops flagging families as duplicate_identity, find/show/resolve
      rank the newest reading first, list/show render vN chips and earlier/later rows,
      and JSON grows additively; uses hand-written fixtures only."
  - id: refusal
    title: One already-captured refusal, before parent resolution
    depends_on: []
    size: medium
    description:
      "refusal: route every duplicate site (URL dedupe, same-reference stem walk,
      library PDF as TARGET, collections) through one renderer that lists every copy and
      prints several hint rows. Duplicate detection runs before the missing -P error. -f
      semantics are unchanged."
  - id: reread
    title: The --force[=MODE] parser and bare -f reread
    depends_on:
      - family
      - refusal
    size: large
    description:
      "reread: turn -f into --force[=MODE], with adjacent-mode adoption and a stray-mode
      guard, and replace the bool with Option<ForceMode>. Ship reread end to end: the
      <root>__vN allocator, earlier: written at birth (inline and queued), the -vN
      block-id suffix, the 'Title (vN)' alias, -P/-t defaults from the family, hints,
      refusals, the reread: output line, collections with an exact URL set, docs.
      overwrite parses but refuses until the next phase."
  - id: overwrite
    title: In-place overwrite of unannotated references
    depends_on:
      - reread
    size: large
    description:
      "overwrite: -f overwrite on any host, for references with no PDF annotations.
      Includes rename-style preconditions, -N member selection, a render-first locked
      transaction with a host-local intent marker and HEAD rollback, sidecar removal,
      and a frozen inert note in ref_history/ when there is authored prose. Also
      earlier: carry-forward, recaptured:, success output, the overwrite refusal hints,
      and docs."
  - id: annotated
    title: Annotated overwrite on the Mac
    depends_on:
      - overwrite
    size: large
    description:
      "annotated: overwrite references that carry highlights, image selections, or
      .assets/, on macOS only and only under highlights.coordinate_pdf_writes. Rename
      the annotated PDF and the assets into ref_history/, move inbound #^h- links to the
      frozen note, add the scan write-back guard, extend the acceptance runbook, and
      refuse with a hint on other hosts."
  - id: record
    title: Decision record, glossary correction, and final docs pass
    depends_on:
      - annotated
    size: small
    description:
      "record: write the accepted recapture decision record and correct the Reference
      Task glossary strand, then republish memory. Run the final cross-page docs
      consistency pass and docs-check, and record proposed follow-ups."
proposed_by: bbugyi200.athena.14a
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 14:15:30
status: wip
---

- **PROMPT:**
  [prompts/202610/ref_create_force_recapture.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_create_force_recapture.md)

# Plan: `bob ref create -f|--force[=reread|overwrite]`

## Why

Today the only way to recapture a filed reference is the hint
`open the note, or delete it to recapture`. Following that hint loses your annotations,
orphans the reading task, and leaves dangling `🔖` follow-up links. This epic gives each
real intent its own gesture:

- **reread** (bare `-f`): _"I want to read this again, separately."_
- **overwrite** (`-f overwrite`): _"This capture is wrong (a paywall stub, a truncated
  render, a revised source I haven't read yet), so replace it."_

The design follows the accepted research report
`research:202610/ref_create_recapture_modes/ref_create_recapture_modes.md`. Bryan
accepted all of its requirement adjustments (A1–A12). The report decides why the design
looks this way, with its alternatives and its live-vault survey, so do not reopen those
choices. This plan settles the report's open questions (see `legacy_force`,
`keep_created`, and [resolved details](#resolved-details-beyond-the-research)) and
splits the work into phases that each ship safely.

## The contract every phase builds toward

### Words

- **Reading:** one live reference: a note under `ref/` plus its PDF bundle and its
  reading task.
- **Reading family:** live references linked through `earlier:`. A reference with no
  rereads is a family of one. The **root** is the member with no `earlier:` entry under
  `ref/`. The **newest reading** is the member with the highest reading number.
- **Reading number:** 1 + the number of the reference's `earlier:` entries whose link
  target is under `ref/`. It is computed from frontmatter, never parsed from a filename.
  A stem that merely looks versioned, such as `protocol__v2`, is reading 1.
- **Frozen note:** an inert, git-tracked copy of a replaced note under
  `ref_history/<type>/`, which overwrite writes. It is never a reference, a task source,
  or a dedupe hit.

### CLI surface and parsing

The existing `-f/--force` becomes `--force[=MODE]`. Its three old meanings still hold
under either mode: replace a queued same-reference PDF in `xlib/`, replace an exact `-o`
file, and replace different companion audio. A second force flag would be unlearnable.

```rust
Arg::new("force")
    .long("force").short('f').value_name("MODE")
    .num_args(0..=1).require_equals(true)
    .default_missing_value("reread")     // never default_value: absent must stay absent
    .value_parser(["reread", "overwrite"]).ignore_case(true)
```

After clap parses, two rules run:

1. **Adjacent-mode adoption.** If `force` came from a bare `-f`/`--force`, and the
   TARGET token directly after it is exactly `reread` or `overwrite` (any case), that
   token becomes the mode and is removed from TARGET. Adjacency comes from
   `index_of("force")` and `indices_of("target")`. An explicit `--force=reread` never
   adopts.
2. **Stray-mode guard.** Some TARGETs can be confused with a mode word: one equal to
   `reread` or `overwrite` (any case), or within edit distance 2 of one. When such a
   TARGET is a bare word (no `/`, `.`, or `[[`) and is not an existing path or a URL,
   exit 2 with:
   `` `overwite` looks like a --force mode; use -f overwrite (or --force=overwrite) ``.
   The escapes are `./overwrite` for a file and `[[overwrite]]` for a vault note. If
   adoption leaves no TARGET, the usual missing-TARGET usage error fires.

Lock this matrix as CLI tests (argv after `create`; the cases come from the research
report's clap 4.6.7 probe):

| argv                                                            | result                                           |
| --------------------------------------------------------------- | ------------------------------------------------ |
| `URL`                                                           | no force                                         |
| `URL -f`, `-f URL`, `--force URL`                               | reread                                           |
| `-f overwrite URL`, `URL -f overwrite`, `URL --force overwrite` | overwrite (adopted)                              |
| `--force=overwrite URL`, `-f=overwrite URL`                     | overwrite                                        |
| `-f Overwrite URL`                                              | overwrite (adopted, case-insensitive)            |
| `-fd URL`, `-df URL`, `-f -d URL`                               | reread + dry run                                 |
| `URL -f overwrite -d`                                           | overwrite + dry run                              |
| `-f overwite URL`                                               | exit 2, guard                                    |
| `-fd overwrite URL`                                             | exit 2, guard (not adjacent)                     |
| `--force=reread overwrite URL`                                  | exit 2, guard                                    |
| `-f overwrite` (no TARGET)                                      | missing-TARGET usage error                       |
| `-f overwrite -f URL`, `--force=reread --force=overwrite URL`   | clap usage error                                 |
| `URL1 URL2 -f overwrite`                                        | overwrite, collection `[URL1, URL2]`             |
| `-foverwrite URL`                                               | clap `UnknownArgument` (unsupported, documented) |

Internally, `CreateOptions.force: bool` becomes `force: Option<ForceMode>`, where
`enum ForceMode { Reread, Overwrite }`:

- every old boolean use becomes `force.is_some()`, so the three old meanings hold under
  either mode;
- the duplicate sites match on the mode.

The same change applies to the clip and collection option structs, and to every
`matches.get_flag("force")`, which would panic on a valued argument. Automatic producers
(`ingest_url`, `bob ref jobs`, `bob gkeep pull`) pass `None`, because a retried clip
must never mint `__v2`, `__v3`, and so on.

The help row keeps its alphabetical position:

```text
-f, --force[=<MODE>]  Capture a source that is already a reference
                      reread (bare -f): add a new <stem>__vN reading beside it
                      overwrite: recapture it in place; your earlier notes move to ref_history/
```

`after_help` gains a `Recapture:` section. It covers both modes, the queued-PDF rule,
`-N` as the overwrite selector, the `./overwrite` and `[[overwrite]]` escapes, and what
never needs `-f`. The `Output:` and `Audio:` paragraphs today say `--force` never
overrides a filed reference; that sentence is rewritten.

### Routing

- **Not captured:** create as today. `-f` is a no-op, except that `-f overwrite` refuses
  with `nothing to overwrite: <source> is not a reference yet` and a hint to drop
  `overwrite` (exit 1). A destructive intent that finds no target is information.
- **Captured, no `-f`:** the refusal below, before any fetch, adapter call, or write
  (exit 1).
- **Queued only, in `xlib/`, with no note:** any `-f` replaces the queued PDF, exactly
  as `--force` does today.
- **Filed, bare `-f` / `reread`:** [reread](#reread-contract).
- **Filed, `overwrite`:** [overwrite](#overwrite-contract).
- **Never a duplicate, so `-f` changes nothing:**
  - vault-note snapshots on a new day (a new dated stem);
  - legacy notes without a PDF, which keep warning and capturing fresh;
  - `-L` without `-f`, which still attaches audio to the existing capture;
  - automatic producers.

> [!decision] legacy_force = no A legacy-only match stays "warn and capture a fresh
> copy", with or without `-f`, and the legacy note is superseded as today. `-f` has
> exactly one rule: it only changes what create would otherwise refuse.

> [!decision] legacy_force With `-f`, a legacy-only match becomes a family root: the
> fresh capture is named `<legacy stem>__v2` (allocator rules apply). Its `earlier:`
> links the legacy note, the legacy bytes stay untouched, and the legacy note stays
> listed as a reading, not superseded. Without `-f`, today's warn-and-capture behavior
> is unchanged. Implement this in the `reread` phase.

### The refusal (final shape, after the `overwrite` phase)

```text
bob ref: error: already captured: Attention Is All You Need
  ref/papers/attention_is_all_you_need__v2.md · v2 · wip · 3 highlights · captured 2026-09-01
  also: ref/papers/attention_is_all_you_need.md · v1 · read · 41 highlights · captured 2026-03-02
hint: -f            read it again as attention_is_all_you_need__v3 (earlier readings stay as they are)
hint: -f overwrite  recapture attention_is_all_you_need__v2 in place (your notes move to ref_history/)
hint: -L            narrate attention_is_all_you_need__v2 instead
```

- One renderer serves every site, so every route looks the same.
- The first row is the newest reading, which is what `-f overwrite` targets. `also:`
  rows follow newest-first, with the same columns.
- Hints name the **computed** next stem and the **actual** overwrite target, so `-f` is
  never a surprise.
- Remedy columns align. Show `-L` only where `--listen` attaches today.
- Colors come from `Styler`: the red `error` label, `bob ref list`'s status colors on
  the status chip, dim `·` separators and `hint:` labels. Output is plain when piped or
  when `NO_COLOR` is set.
- The exit code stays 1. The text keeps the substrings `already captured` and
  `already queued`, because `ingest.rs` classifies collisions by them for automatic
  producers.
- Unrelated duplicates (a pre-existing `duplicate_identity` pair outside one family)
  list every path and ask for `-N <stem>`.
- A partial collection overlap, including a single URL that belongs to a collection,
  lists the overlapping members. It refuses under every mode with
  `hint: -f recaptures only the exact URL set of <stem>`.

### Reread contract

- **Name:**
  - The new stem is `<root stem>__v<N>`, where `N` is one more than the family's highest
    reading number (minimum 2).
  - A stem occupied anywhere is skipped. That covers the note, PDF, sidecar or
    `.textbundle`, audio, `.assets/`, and case-insensitive `id`s under `ref/`, `lib/`,
    and `xlib/`.
  - Never fall back to `_N`, which already means "taken by a _different_ reference".
  - Cap at `__v999`, matching today's `_999` failure.
- **`-N` with a reread:**
  - On a route that would refuse, `-N` must name a family member. For title-identity
    routes it is just the probe; for overwrite it is the selector. Any other `-N` on a
    URL-identity duplicate exits 2:
    `-N other is not a reading of foo; readings are named foo__vN`.
  - On title-identity routes (local Markdown, local PDF, vault note), a `-N` naming a
    free stem is not a duplicate at all, and creates as today.
- **`-o` with a captured source:** exit 2, `a reread is named <root>__vN; drop -o`. An
  exact path cannot take a suffix.
- **The library PDF itself as TARGET** (`bob ref create lib/papers/x.pdf -f`): refuse.
  It would copy v1's highlights into v2. The hint is: recapture from the recorded
  `source_url`, or from the original file. `-L` without `-f` still attaches.
- **Defaults that make `bob ref create URL -f` a complete command:**
  - `-P` defaults to the newest reading's task residence, and `-t` to its ref type.
  - Status is `ready` unless `-s` is given.
  - Audio is not inherited. When an earlier reading is narrated, print
    `hint: v1 is narrated (lib/papers/x.mp3); add -L to narrate this reading`.
- **Writes:**
  - A complete new bundle, with `id` equal to the new stem.
  - A new reading task with alias `Title (vN)` and block id `^ref-<slug>-vN`. Compute
    this by slugging the root stem, trimming it so `-vN` fits the 44-character cap, then
    appending `-vN`, so the suffix always survives. The generic `-2` collision suffix
    still applies after that.
  - `earlier:` on the new note.
  - **Zero bytes written to earlier readings.** Backlinks and `show`'s derived `later:`
    row supply the forward direction.
- **Birth derivation:**
  - `earlier:` is derived when the note is born, in the sync new-note path that both
    inline intake and the queued `bob ref scan` drain use. The trigger is a stem of the
    form `<base>__v<N>` (N ≥ 2) where `<base>` or another `<base>__vM` (M < N) is a live
    reference sharing identity with the new PDF.
  - Shared identity means equal `source_url` dedupe keys. Without them, it means equal
    `source_note`, and otherwise equal normalized titles.
  - The entries are every live family member, newest first. Each is labelled
    `vM · <captured date>`.
  - `-k`, or intake that cannot run, therefore needs nothing special. The queued
    `<root>__vN.pdf` gets the same `earlier:` when scan files it.
- **Lost races:** after intake's pre-sync under the locks, re-check the chosen stem. If
  another host took it, bump to the next free `__vN` and restamp the marker `id` before
  filing. If restamping inside intake proves invasive, instead refuse with nothing filed
  and say so. Either way, never file two references under one stem.
- **Collections:** an exact URL-set match (order-insensitive) rereads as `<root>__vN`
  like any other source.
- **Success output:** today's lines, plus one `reread:` line where `renamed:` prints:

```text
ok created Highlights-ready PDF
reread: attention_is_all_you_need → attention_is_all_you_need__v2 (v1 read · 41 highlights)
id: attention_is_all_you_need__v2
ingested: lib/papers/attention_is_all_you_need__v2.pdf → ref/papers/attention_is_all_you_need__v2.md · task → sase.md ^ref-attention-is-all-you-need-v2
sync: committed <sha7> · pushed
```

Dry run (`-d`) prints the same `reread:` line and ends with `writes: none`.

### Overwrite contract

**Overwrite replaces the material, not your progress.**

- **Kept:**
  - The note path and stem, and `id`.
  - The `^ref-…` reading task: line, lane, status, Task Links, Depends-On, relations,
    keep log, and freshness.
  - Inbound links to the note itself and its headings.
  - Companion audio, unless `-L` or `-a` is given.
- **`created`:**

  > [!decision] keep_created `created` is kept. A new note-local
  > `recaptured: <YYYY-MM-DD>` records the new material.

  > [!decision] keep_created = no `created` is reset to the overwrite date. No
  > `recaptured:` key is written.

- **Rebuilt from scratch:**
  - The PDF bytes and marker (same `id`; status = the task's current status).
  - The note body (template plus managed embed).
  - `captured`.
  - The stale Highlights sidecar (`lib/<type>/<stem>.md` or `.textbundle`) is removed.
    Otherwise the next scan would pair the old export with the new PDF.
- **Status** changes only with an explicit `-s`. "Explicit" means `value_source` is
  `CommandLine`, because `-s` has a clap default.
- **Target:** the newest reading of the matched family. `-N <stem>` selects another
  member and must name one. Unrelated duplicates refuse.
- **Incompatible, exit 2 with a hint:**
  - `-o`;
  - a `-t` different from the reference's type;
  - a `-P` different from the task's residence (`hint: bob task move …`);
  - `-k`.
- **Intake that cannot run** (ingest disabled, lock busy, vault unreachable) refuses.
  Overwrite never queues.
- **Preconditions** reuse `bob ref rename`'s codes where they exist:
  - `legacy_reference` and `intake_reference`: the reference must be modern, PDF-backed,
    and not an intake item.
  - `not_in_sync`, with `hint: run bob ref scan -w on the Mac first`.
  - `dirty_paths`: every touched path must be clean in vault git.
  - `open_note_tasks` (new): the note holds no open checkbox task outside the managed
    embed or tracker. Refuse and list them.
- **Freeze only what you made.** When the old note has authored prose outside the
  generated skeleton (or, from the `annotated` phase on, highlights, image selections,
  or `.assets/`), write a frozen note. Otherwise print
  `earlier: nothing to archive (0 highlights, no notes); git keeps the old capture`.
- **The frozen note:** `ref_history/<type>/<stem>__<YYYY-MM-DD>.md`, dated the day of
  the replacement, with `_2` on a same-day collision.
  - **Body:** verbatim, so every `^h-` and `^s-` anchor still resolves. The managed
    reading-task embed becomes this banner, so the live task is never transcluded:

    ```markdown
    > [!info] Earlier capture · replaced 2026-10-25 Captured 2026-03-02 · read · 41
    > highlights · current:
    > [[ref/papers/attention_is_all_you_need|Attention Is All You Need]]
    ```

  - **Legacy tracker:** an in-note `^ref` tracker loses its `#task` tag and block id.
  - **Frontmatter** is an allowlist: `title`, `source_url`, `created`,
    `parent: "[[ref/<type>/<stem>]]"` (the vault rule), `replaced`,
    `status_when_replaced`, `highlights_count`, and, from the `annotated` phase on,
    `annotated_pdf`. Nothing reads it as a reference, a task, or a duplicate.

- **Lineage:** the live note's `earlier:` gains
  `"[[ref_history/<type>/<stem>__<date>|replaced <date> · N highlights]]"` first. Every
  earlier entry carries forward, newest first. Two overwrites in a row leave both
  archives listed.
- **Transaction:** computed in full before any write, mirroring `bob ref rename`:
  1. **Plan.** Resolve the family and target, and check the lock-free preconditions.
     Render the new PDF in private scratch; a failed fetch or render writes nothing.
     `-d` stops before the fetch, prints the plan from known metadata, and ends with
     `writes: none`.
  2. **Lock and recheck.** Take `bob_sync.lock → scan.lock`, pre-sync, and recheck every
     precondition and every planned preimage hash. A changed preimage aborts.
  3. **Intent.** Write an intent marker that lists the touched and created paths. It
     goes in bob's **host-local state directory** (where the scan lock lives), never in
     the vault. Because `dirty_paths` demanded clean paths, git HEAD is the
     before-image. On its next run, `bob ref create` or `bob ref scan` on that host
     finds a leftover intent:
     - if HEAD carries the overwrite commit, it just clears the intent;
     - otherwise it restores the listed paths from HEAD, removes the created files,
       clears the intent, and prints
       `recovered: rolled back an interrupted overwrite of <stem>`.
  4. **Apply.** Write the frozen note (if any), remove the sidecar, and publish the new
     PDF with no-clobber semantics. Then write a fresh note skeleton that carries the
     kept note-local keys and `earlier:`, and let the single-PDF sync render its managed
     regions. That way the existing reading task is located, never born. Update the task
     alias if the title changed.
  5. **Verify.**
     - A dry-run sync of the PDF plans nothing.
     - The reading task's `(path, block_id)` and status are unchanged.
     - There is exactly one reading task.
     - No new `duplicate_identity` appears.
     - No rewritten link dangles.

     Any failure restores from HEAD, removes created files, and exits 1 with
     `verify_failed`.

  6. **Commit and push.** Make one scoped commit (`ob::commit_paths`), with subject
     `bob ref create: overwrite <type>/<stem>`. Then clear the intent and post-sync. A
     post-sync failure reports `overwritten; push pending (run bob vault-sync)`, never
     "run it again".

- **Collections:** overwrite only an exact URL-set match.
- **Success output:**

```text
ok replaced Highlights-ready PDF
overwrite: attention_is_all_you_need (read · 41 highlights)
earlier: ref_history/papers/attention_is_all_you_need__2026-10-25.md (41 highlights · 3 follow-up links moved · annotated PDF kept)
task: kept sase.md ^ref-attention-is-all-you-need [x] read · pass -s ready to reopen
audio: kept lib/papers/attention_is_all_you_need.mp3 (narrates the replaced capture; add -L to re-narrate)
sync: committed <sha7> · pushed
```

### Lineage: one note-local `earlier:` list, both modes

```yaml
# a reread v3
earlier:
  - "[[ref/papers/attention_is_all_you_need__v2|v2 · 2026-09-01]]"
  - "[[ref/papers/attention_is_all_you_need|v1 · 2026-03-02]]"
# after an overwrite (earlier entries carry forward, newest first)
earlier:
  - "[[ref_history/papers/attention_is_all_you_need__2026-10-25|replaced 2026-10-25 · 41 highlights]]"
```

- **Format and placement:**
  - Write the list in block style. That is Obsidian's own canonical Properties form, so
    an edit in Obsidian never churns it. Read inline style too.
  - The list is note-local, like `created`. Sync re-emits unmanaged keys verbatim, and
    the list stays out of the marker projection and its hash.
  - It is written once, at birth (reread) or by the overwrite transaction. After that,
    the index only reads it, never re-derives it.
- **Rename-safe:** `bob ref rename`'s link rewriter already rewrites frontmatter
  wikilinks.
- **Static labels only:** a version and a date, plus a highlight count for frozen notes.
  Live status is never baked in.

### `ref_history/` stays inert by construction

- Every reference walker (`sources.rs` dedupe, `ref_library`, `stamp` occupancy,
  `ref_tasks`, attach) walks only `ref/`, `lib/`, or `xlib/`, so a top-level directory
  is outside all of them.
- Vault-wide task walkers still see it. That is why the frozen note itself must carry no
  open task, no `#ref` line, and no `^ref` block id.
- Tests assert that `ref_history/` yields no library rows, no dedupe hits, no tasks, and
  no annotation tasks.
- Use only git-tracked extensions there (`.md`, `.pdf`, images). The vault `.gitignore`
  drops `.json` and `.txt`.

## Resolved details beyond the research

- **Primary row:** the refusal's primary row is the newest reading, not the oldest. It
  is what `-f overwrite` targets and what `find`/`show` rank first.
- **`-N` on a URL duplicate:** rereads are always `<root>__vN`, so birth derivation can
  recover the family from the stem. The research's "an explicit free `-N` is used as is"
  would silently lose `earlier:` on the queued path.
- **`-f` with `-o`:** refused for a captured source. An external `-o` would produce a
  PDF but no reading.
- **Partial overlaps:** a single URL that belongs to a collection is a partial overlap
  and refuses under every mode.
- **Intent marker location:** host-local state, not `.bob/`. Parts of `.bob/` are
  vault-synced, and a crash is host-local anyway.
- **Annotated overwrite on the Mac** requires `highlights.coordinate_pdf_writes`.
  Without it, refuse and hint to enable it after its acceptance run. bob never falls
  back to uncoordinated work on a PDF that Highlights may hold
  (`decisions:open-pdf-writes-are-coordinated`).

## Phase graph and interim states

```text
family ──┐
         ├─► reread ─► overwrite ─► annotated ─► record
refusal ─┘
```

- **`family` and `refusal` can run in parallel.** They touch disjoint code
  (`ref_library` + `doctor` vs the create, dedupe, and stem-walk paths) and disjoint
  docs pages.
- **Each phase lands on master as a coherent increment**, because the daily train may
  release between phases:
  - After `family`, families only exist if hand-written. Nothing changes for users.
  - After `refusal`, the refusal is better. `-f` semantics are byte-identical to today.
  - After `reread`, `-f overwrite` parses, but on a filed reference it exits 2 with
    `-f overwrite is not available yet; -f reads it again as <stem>__vN`. With no match
    it gives `nothing to overwrite`.
  - After `overwrite`, references with highlights, image selections, `.assets/`, or
    inbound `#^h-` links refuse:
    `overwriting an annotated reference is not available yet; -f starts a new reading`.

---

## Phase `family`: Reading families in the library read model

**Goal:** the library understands `earlier:` lineage before anything writes it. All
tests use hand-written fixture notes.

- **Shared lineage module** (for example `src/native/ref_library/lineage.rs`), reused by
  later phases:
  - parse `earlier:` (block or inline; `[[target|label]]` or bare) into
    `EarlierEntry { target, label, kind: Reading | Replaced }`. `Reading` means the
    target is under `ref/`; `Replaced` means it is under `ref_history/`;
  - render the block-style list;
  - compute the reading number.
- **`RefRow` gains:**
  - `earlier: Vec<EarlierEntry>`;
  - `reading: Option<u32>`, which is `Some` for rows in a family of two or more, and for
    rows that carry a `Reading` entry in `earlier:` even when it dangles;
  - a family key, for example the root path. Build families in `build_index` after the
    rows exist (`src/native/ref_library/mod.rs`), as connected components over `Reading`
    entries that resolve to live rows.
- **`apply_supersession`** (`mod.rs:781`):
  - do not emit `duplicate_identity` when every PDF-backed row sharing a key is in one
    family; still warn when an unrelated row shares the key, naming only the non-family
    pairs;
  - when a PDF-less legacy row shares the key, its `superseded_by` winner is the newest
    family reading.
- **Doctor** (`src/native/highlights_ref/doctor.rs`):
  - `append_library_identity_row` recomputes shared keys on its own and needs the same
    family exemption;
  - add an `earlier_dangling` warning for an `earlier:` entry whose target does not
    resolve, of either kind.
- **Ranking** (`resolve.rs:78` `primary_rank`, `show.rs:161-245`, `find.rs:109-176`):
  - within a family, the newest reading is primary, ahead of `reading_state_rank`;
  - outside families nothing changes;
  - `show` collapses an all-one-family hit set to the newest reading, where today it
    returns `Ambiguous`.
- **Rendering:**
  - The hard-coded `also: … (superseded legacy note)` (`show.rs:1329`) and `find`'s
    `also_line` (`output.rs:412`) become reason-aware:
    `also: <path> (earlier reading v1 · read)`.
  - `show` adds an `earlier:` row (readings and replaced captures, using their labels)
    and a derived `later:` row in both text and markdown output (`metadata_rows`,
    `render_one_markdown`).
  - `list` and `show` headers get a `vN` chip for N ≥ 2, and a `v1` chip on a root only
    when it has later readings.
- **JSON** stays additive under the current `REF_SCHEMA_VERSION`: `reading` and
  `earlier` on rows, `later` on show. Follow `RefRow`'s existing null/`[]` serialization
  convention.
- **Sync round trip:** a block-style `earlier:` must survive `bob ref sync`
  byte-for-byte. There is a suspected hazard in `model.rs` `render_with_projection`:
  continuation lines of a _removed_ block-style key can be kept and absorbed into the
  key above. Write a test for that hazard and fix it if the test fails.
- **Rename:** renaming a family member works, because there is no `duplicate_identity`
  left for `rename/plan.rs:980` to refuse. The later reading's `earlier:` link is
  rewritten. Test both.
- **Tests:** `tests/cli/ref_library/` (find, show, list, doctor, rename) with fixture
  families:
  - a two-reading family;
  - a three-reading family with a deleted middle reading (dangling warning, reading
    numbers unchanged);
  - a natural `protocol__v2` stem with no family;
  - a family plus an unrelated duplicate (still warns);
  - JSON field presence.
- **Docs:** `docs/ref.md` (families in find/show/list/doctor, the JSON fields) and
  `docs/highlights-ref-sync.md` (`earlier:` as a note-local key).
- **Commit:** `feat(ref): …`.

## Phase `refusal`: One already-captured refusal, before parent resolution

**Goal:** ship [the refusal](#the-refusal-final-shape-after-the-overwrite-phase) with
today's remedies. `-f` keeps exactly today's meaning, and no CLI argument changes in
this phase.

- **Renderer:** one module, for example `src/native/highlights_ref/duplicate.rs`,
  holding `AlreadyCaptured { title, copies: Vec<CapturedCopy>, remedies: Vec<Remedy> }`.
  - A copy holds its path, stem, status, highlight count, captured date, and whether it
    is queued. Read these with `ParsedFrontmatter`, as `sources.rs` already does.
  - Remedies are data, so later phases append `-f` and `-f overwrite` rows without
    touching the layout.
  - Queued hits render as `already queued: <title>` with the remedy
    `-f  replace the queued PDF` (today's `--force`).
- **Route every site through it:**
  - `sources::check_dedupe` and `find_refusing_hit` (URL dedupe). List **every**
    PDF-backed note that shares the key, not just the first.
  - `stamp::plan_default_target` (same reference at the stem, library, or note).
  - `pdf_target::check_local_pdf_identity` (library PDF as TARGET).
  - The collection dedupe (`collection.rs:1130-1180`).
  - `check_dedupe_with_listen_hint` (create.rs) and the clip article route. The `-L`
    hint becomes a remedy row.
  - In this phase the remedies are `-L` where attach applies, `-N <name>` on title
    routes (today's advice), and `-f` for queued hits. Drop "delete it to recapture"
    everywhere.
- **Multi-row errors:** `report_create_error` (`create.rs:637`) splits only on the first
  `\nhint: `. It must print the first line after `error:`, then indented detail rows,
  then **every** `hint:` row. Fix the collection and clip error printers too, and prefer
  one shared printer.
- **Duplicate before parent:** today `resolve_parent_for_target` errors
  `--parent is required` (exit 2) before dedupe runs. Make parent resolution lazy, so
  that on every non-vault-note route a duplicate refuses (exit 1) before a missing `-P`
  errors. As a side effect, `-L` attach no longer needs `-P`. The `reread` phase builds
  its `-P`/`-t` defaults on this.
- **Tests:**
  - Update every existing assertion on the old text in `tests/cli/highlights/create.rs`,
    `sources.rs`, `stamp.rs`, and `collection.rs` unit tests.
  - Keep the "refuses even with `--force`" tests: that is still true in this phase.
  - Add renderer unit tests: layout, several copies, aligned remedies, no ANSI when
    piped.
  - Add CLI tests: a captured URL without `-P` exits 1 with `already captured`;
    `ingest.rs` still classifies the new text as `Collision`.
- **Docs:** the `docs/highlights-create.md` Dedupe section shows the new refusal.
- **Commit:** `feat(ref): …`, no `!`. Exit codes for duplicates are unchanged.

## Phase `reread`: The `--force[=MODE]` parser and bare `-f` reread

**Goal:** ship the [CLI surface](#cli-surface-and-parsing) and the
[reread contract](#reread-contract) end to end. This phase is `large`, so plan it first.
A natural internal order is: parser and `ForceMode` plumbing, then the allocator and
birth derivation, then the create wiring, collections, and output.

- **Parser:**
  - the argument definition, adoption, and guard above;
  - the full matrix as CLI tests;
  - shell-completion support for `--force=<MODE>` (see `docs/completion.md` and its
    tests);
  - the hidden `bob ref clip` alias, which inherits everything.
- **Plumbing:** `force: Option<ForceMode>` through the create, clip, and collection
  options. Every `get_flag("force")` goes. Producers pass `None`.
- **Allocator:**
  - `<root>__v<max+1>` with whole-bundle occupancy (extend `DefaultTargetOccupancy` in
    `stamp.rs` rather than duplicating it), the `__v999` cap, and the lost-race recheck
    under the intake locks;
  - family and root resolution come from `family`'s lineage module and the library
    index.
- **Birth derivation:** in the sync new-note path (`sync.rs`, where `is_new_note` sets
  `created`; emitted via `PipelineMetadata` in `model.rs`), derive `earlier:` and the
  reading number by the rule above. Run it only for `__v<N>` stems, so normal births pay
  nothing. The inline intake (`apply_intake` and `finish_intake`) and the queued drain
  (`drain_queue`) share it.
- **Reading task:**
  - **Block id:** `allocate_ref_block_id` (`ref_tasks/line.rs:193`) reserves `-vN` for
    `__v<N>` stems. Check every caller and preview: `ref_tasks/insert.rs`, `sync.rs`,
    `ref_library/migrate_tasks/plan.rs`, `reading_execute.rs`.
  - **Alias:** `Title (vN)` for reading numbers ≥ 2, where the task line renders its
    title alias. If a later sync reconciles the alias with the note title, it must keep
    the same suffix. Test that a second `bob ref sync` leaves the task line
    byte-identical.
- **Create wiring:**
  - On a filed hit, reread replaces today's refusal on every site: URL, title stem walk,
    and vault-note same-day snapshot (`<note>_<YYMMDD>__v2`).
  - Add the `-P`/`-t`/`-s` defaults, the `-N` rule, the `-o` refusal, the library-PDF
    refusal, and the audio hint.
  - Add the `-f` remedy row to the refusal; its hint names the computed next stem.
  - On a filed hit, `overwrite` gives the interim exit-2 message.
  - Apply the `legacy_force` branch the reviewer chose.
- **Collections:** an exact-set reread; partial overlaps refuse under every mode.
- **Output:** the `reread:` line (real run and `-d`) and `writes: none` on a dry run.
- **Tests:**
  - **Flip** the "refuses even with `--force`" tests:
    - `highlights_create_refuses_same_title_library_pdf_with_or_without_force`;
    - the folded URL dedupe test near `create.rs:5113`;
    - the scanned-library loop near `create.rs:5783`.

    Without `-f` they still refuse; with `-f` they reread.

  - **Naming:**
    - `__v2` and `__v4` taken gives `__v5`;
    - an unrelated `foo_2` is ignored;
    - a sidecar-only, audio-only, or `id`-only occupant is skipped;
    - a long stem keeps `-vN` in the block id;
    - `protocol__v2` forms no family;
    - rereading `x__v2` gives `x__v3`, never `x__v2__v2`.
  - **Births:** a queued (`-k`) reread gets `earlier:` when scan files it.
  - **Writes:** a reread writes zero bytes to earlier readings (snapshot them).
  - **Ordering:** the refusal precedes any fetch or adapter call (fake clip and fake
    curl logs).
  - **Producers:** `ingest_url`, jobs, and Keep never mint `__vN`.
  - **Family health:** `bob ref doctor` stays clean on the new family.

- **Docs:**
  - `docs/highlights-create.md`: Dedupe, Output, and Audio, plus a new `## Recapture`
    section for reread;
  - `docs/ref.md`: the Identity section on `__vN` ids;
  - `after_help` and the help row; then run `just docs-gen` and `just docs-check`.
- **Commit:** `feat(ref)!: …`, because `-f` on a library hit changes from an error to a
  write.

## Phase `overwrite`: In-place overwrite of unannotated references

**Goal:** ship the [overwrite contract](#overwrite-contract) for references without PDF
annotations, on any host. This phase is `large`, so plan it first.

- **Reuse before writing anew:** `bob ref rename`'s machinery is almost all private
  (`src/native/highlights_ref/rename/plan.rs`). Lift what overwrite needs into a shared
  module rather than copying it:
  - the lock sequence (`ob::acquire_lock_waiting`,
    `acquire_scan_writer_lock_with_timeout`,
    `vault_sync::run_cycle_with_existing_lock_report`);
  - the precondition checks (`is_legacy`, `intake_query`, `is_in_sync`,
    `dirty_touched_paths`, `has_managed_highlights`);
  - before-images and rollback (`migrate_tasks::snapshot_before_images` /
    `restore_before_images`, `io::rename_no_replace`, `atomic_write`);
  - commit and recovery (`ob::commit_paths`, `unstage_paths`,
    `committed_despite_error`).

  Keep rename's behavior byte-identical; its tests must stay green.

- **"Authored prose" detection:** compare the note with managed regions removed against
  a freshly rendered skeleton for the same projection. Any difference is authored, and
  gets frozen.
- **What this phase ships:**
  - the frozen note (prose-only), the banner, the frontmatter allowlist, `earlier:`
    carry-forward, and the `keep_created` branch the reviewer chose;
  - the host-local intent marker and HEAD recovery, triggered at the start of
    `bob ref create` and `bob ref scan`;
  - sidecar removal and the verify step;
  - the commit subject and post-sync reporting;
  - a refusal for annotated references (the interim message);
  - the `nothing to overwrite` refusal, kept from `reread`;
  - the `-f overwrite` remedy row on the refusal, and the reread hint
    `hint: <stem> is still ready with 0 highlights; -f overwrite would replace it instead`;
  - overwrite of exact-set collections.
- **Tests (behavior, not implementation):**
  - **Two overwrites in a row:** both frozen notes exist, the live `earlier:` lists both
    newest-first, and the task's `(path, block_id)` and status are unchanged. There is
    exactly one reading task.
  - **No archive:** `nothing to archive`. Old highlights never reappear after the next
    scan, because the sidecar is gone.
  - **Preconditions:** each refusal code (`legacy_reference`, `intake_reference`,
    `not_in_sync`, `dirty_paths`, `open_note_tasks`, annotated) refuses with nothing
    written.
  - **Flags:** `-N` member selection, and `-N` naming a non-member. The `-o`, `-t`,
    `-P`, and `-k` refusals, and intake-unavailable.
  - **Status:** `-s ready` reopens explicitly; no `-s` keeps the status.
  - **Inert archive:** `ref_history/` yields no library rows, dedupe hits, tasks, or
    annotation tasks (`bob ref list`, `find`, `doctor`, create dedupe, and the task
    walkers).
  - **Fault injection:**
    - Unit tests on the shared transaction fail it after the render, after each apply
      step, at verify, and at commit. Each leaves either the intact old state or the
      committed new state.
    - One CLI test plants a leftover intent with a half-applied tree and asserts the
      `recovered:` rollback.
- **Docs:**
  - `docs/highlights-create.md` `## Recapture` (overwrite);
  - `docs/highlights-ref-sync.md` (`recaptured:`, `ref_history/`, frozen notes);
  - `just docs-gen` and `just docs-check`.
- **Commit:** `feat(ref): …` (additive: `overwrite` refused until now).

## Phase `annotated`: Annotated overwrite on the Mac

**Goal:** lift the interim refusal for references that carry highlights, image
selections, `.assets/`, or inbound `#^h-` links. This phase is `large`, so plan it
first.

- **Where it runs:**
  - Off macOS, refuse:
    `this reference has N highlights, which Highlights holds on the Mac; run the overwrite there`
    with `hint: -f starts a new reading from any host`.
  - On macOS with `highlights.coordinate_pdf_writes` off, refuse with a hint to enable
    it after its acceptance run.
  - On macOS with it on, read and move the old PDF inside one bounded `.forMerging`
    coordination, via `src/native/file_coordination.rs`'s `coordinate_merging_write`.
    Highlights saves first. Any hold (`editor_active`, timeout, marker changed) refuses
    with `close it in Highlights and retry` and writes nothing.
- **Freeze the annotations:**
  - Rename (never copy) the annotated PDF to
    `ref_history/<type>/<stem>__<date>__annotated.pdf`. The distinct basename keeps a
    Highlights export from overwriting the frozen note. A rename adds no git objects,
    and an open Highlights document follows a same-volume rename.
  - Record `annotated_pdf` and its sha256 in the frozen frontmatter, and link it from
    the banner.
  - Move `<stem>.assets/` to `…__<date>.assets/`, and rewrite the frozen note's embeds
    to match.
- **Inbound highlight links:**
  - Move every vault link of the form `[[ref/<type>/<stem>#^h-…]]` to the frozen note.
    Extend `rename/links.rs` with a selector, for example
    `LinkSelector::{All, HighlightFragments}`, threaded through `rewrite_text` and
    `rewrite_line`, and make the rewritten form path-qualified.
  - Links to the note itself, or to its headings, stay on the live note.
- **Open tasks:** an open annotation task anywhere in the old note refuses with
  `open_note_tasks`, listing the tasks. The frozen note never carries an open task.
- **Write-back guard:** if a scan finds that the live PDF's sha256 equals an archived
  `annotated_pdf` sha256 for that reference, it holds that PDF with
  `Highlights wrote the replaced capture back` and writes nothing for it.
- **`[h::]` processed ids across an overwrite:** pin today's behavior with a test.
  Re-highlighting the same passage mints the same `^h-` id and counts as the same
  follow-up.
- **Acceptance:** extend `scripts/dev/highlights_coordination_acceptance`, or add a
  sibling runbook, with an annotated-overwrite step:
  - a PDF open in Highlights with unsaved annotations;
  - a coordination hold;
  - a late autosave landing in the archive.

  Bryan runs it on the Mac. CI covers the Linux refusal and the non-macOS paths, plus
  unit tests of the planner with fake coordination.

- **Docs:** `docs/highlights-create.md` and `docs/highlights-ref-sync.md`;
  `just docs-gen` and `just docs-check`.
- **Commit:** `feat(ref): …`.

## Phase `record`: Decision record, glossary correction, and final docs pass

> [!decision] memory_decision_record Use `/sase_memory_write` to add a new strand to the
> `decisions` web, for example `recapture-reads-again-or-replaces-in-place`, titled
> "Recapture Reads Again Or Replaces In Place". Follow the web's record shape: **Applies
> to** (bob-cli, the Bob vault), **Claim**, **Why**, **Cost**, and **Reopens when**.
>
> The claim to record:
>
> - Without `-f`, a duplicate is refused before any fetch.
> - Bare `-f` adds a `<root>__vN` reading linked by note-local `earlier:` and writes
>   nothing to earlier readings.
> - `-f overwrite` keeps the path, `id`, and reading task, and rebuilds the PDF and
>   body. It freezes only authored material into inert `ref_history/` notes, in one
>   locked commit.
> - Annotated overwrite runs only on the Mac under coordination.
>
> Cite the research ref, this plan, and the phase commits. Link
> `[[decisions:reference-ids-follow-file-stems]]`,
> `[[decisions:reference-renames-keep-identity]]`, and
> `[[decisions:open-pdf-writes-are-coordinated]]`.

> [!decision] memory_glossary_ref_task Use `/sase_memory_write` to rewrite the
> `glossary:reference-task` strand. A modern reading task lives in its reference's
> parent note as `^ref-<slug>`, with `-vN` for rereads; a legacy in-note `^ref` tracker
> is still recognized. Keep the status-symbol mapping and the
> `docs/highlights-ref-sync.md` pointer.

- Run `sase memory init` after any memory edit.
- Do a final pass across `docs/highlights-create.md`, `docs/ref.md`, and
  `docs/highlights-ref-sync.md`: one vocabulary (reading, family, earlier, frozen note,
  `ref_history/`), no stale "never overrides a filed reference" text, and examples that
  match real output. Then run `just docs-check`.
- Record `PROPOSED FOLLOW-UP:` notes on this phase's bead (do not create beads):
  - **SASE research file hook:** the hook can dodge dedupe through hash-suffixed
    Markdown stems (the five `ref/chat/*-<12 hex>.md` duplicates). Possible fix: strip
    the artifact id when deriving stems, and choose `-f overwrite` while the earlier
    report is unread, otherwise `-f`. This spans bob-cli and chezmoi.
  - **Bob Mac Capture Refs panel:** a `vN` chip, under
    `decisions:mac-capture-is-a-thin-client`.
  - **bob-plugins reading board:** a `vN` chip.

## Cross-cutting rules for every phase

- **CLI rules** (`cli_rules.md`): excellent `-h`, options alphabetical, and colored
  output through `Styler` that is plain when piped (`assert_stdout_has_no_ansi`).
- **Docs** (`docs.md`): a new flag goes through clap help plus `just docs-gen`; behavior
  goes on the reference pages. Run `just docs-check` before committing.
- **Release:** never edit the `Cargo.toml` version or `CHANGELOG.md`. Only `reread`
  carries `!` (`decisions:bob-cli-releases-ride-a-daily-train`).
- **Tests** use the existing support helpers:
  - from `tests/cli/support.rs`: `bob_command` (isolates the sync lock and
    `XDG_STATE_HOME`), `write_highlights_pdf`, and `configure_test_git_identity`;
  - from `tests/cli/ref_library/rename.rs`: `synced_vault`;
  - the fake clip and fake curl helpers in `tests/cli/highlights/create.rs`.
- **Never** write the vault outside a test temp dir. Never run against `~/bob`.

## Out of scope

- A separate `bob ref recapture` verb, `-R/--reread`, `-F/--overwrite`, or
  `--existing <REF>`. The research rejected them.
- Renaming the original to `__v1` on the first reread.
- Re-anchoring old highlights into a new capture.
- Automatic rereads without a flag.
- Client chips in Bob Mac Capture and bob-plugins (proposed follow-ups).
