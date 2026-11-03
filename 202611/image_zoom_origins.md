---
tier: epic
title: Image zoom remembers where images came from
goal: 'Every image bob copies into the vault remembers its original file path. The
  `Z` viewer can copy that path (`y`) or turn the link back into it (`r`), and `Z`
  on a bullet holding only an image path imports it the way `%c` does. One beautiful,
  undoable loop: path → embed → path.

  '
decisions:
  copy_form:
    ask: When bob knows an image's original path, which form should `y` copy?
    choices:
      absolute: /Users/bryan/tmp/screenshots/x.png. Works in any app, file dialog,
        or quoted shell argument
      tilde: ~/tmp/screenshots/x.png. Matches what you typed, but only shells and
        Finder expand ~
    default: absolute
    why: A copied path should work wherever it is pasted
    answer: tilde
  decision_record:
    ask: Record "attachment origins live in a per-host journal" as a decisions-web
      record?
    memory:
    - decisions:attachment-origins-live-in-per-host-journal
    default: false
    answer: false
phases:
- id: origins
  title: bob records attachment origins and ships `bob attachments`
  depends_on: []
  size: medium
  description: 'origins: add the `.bob/attachments/<host>.jsonl` origin journal, have
    `%c` record every attachment it copies, add `bob attachments import` and `bob
    attachments origin`, allowlist the journal in the vault, and document the contract.'
- id: viewer
  title: Viewer shows the origin, copies it with `y`, restores it with `r`
  depends_on:
  - origins
  size: medium
  description: 'viewer: expose a bob-ledger-tools `attachments` api namespace, then
    give the Z viewer an origin line, key-chip footer, toast pill, and the `y` copy
    and `r` restore gestures, with tests and docs.'
- id: pathz
  title: '`Z` on an image-path bullet imports it'
  depends_on:
  - origins
  - viewer
  size: small
  description: 'pathz: detect a bullet that holds only an image path, import it through
    bob, rewrite it to bob''s embed in one undoable edit, and open the viewer on the
    new image with an "Imported" pill.'
proposed_by: bbugyi200.athena.17n
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 14:53:39
status: wip
bead_id: bob-cli-gn
---

- **PROMPT:** [prompts/202611/image_zoom_origins.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/image_zoom_origins.md)
- **BEAD:** [bob-cli-gn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gn/README.md)

# Plan: Image zoom remembers where images came from

## Why this needs bob first

`%c` copies a file path from the clipboard into `img/<sanitized name>` (or `file/`) and
embeds `![[img/<name>|400]]` (`src/native/capture_clip/files.rs`: `plan_attachments`,
`attachment_reference`). The original path exists only in that one capture's JSON
(`clip.attachments[].source`), so nothing in the vault remembers it. The viewer can't
copy or restore a path nobody recorded, and turning a path bullet into an embed must use
the exact `%c` copy rules. Both point to the same design:

- **bob owns the copy and the record.** This follows
  `decisions:mac-capture-is-a-thin-client` and the per-host journal precedent
  (`decisions:theme-prefs-live-in-per-host-journal`, `src/native/pref_journal/mod.rs`):
  bob is the only writer, and clients call bob.
- **The plugins own presentation and the note edit.** The note edit stays an Obsidian
  editor transaction, so vim `u` undoes it.

## The experience

One loop, three gestures:

1. **`Z` on `- ~/tmp/screenshots/20261103_142837.png`.** bob copies the file exactly as
   `%c` would. The bullet becomes `- ![[img/20261103_142837.png|400]]` in one undoable
   edit, and the viewer opens on the image with the pill
   `Imported to img/20261103_142837.png · r to undo`.
2. **`y` in the viewer** copies where the image came from (the original path when bob
   recorded one) and confirms with a pill that shows exactly what was copied.
3. **`r` in the viewer** puts the original path back in place of the link, closes the
   viewer, and confirms with a notice. The vault copy is kept. Pressing `Z` on that
   bullet again re-imports it and reuses the identical file, so `Z`/`r` round-trips. Vim
   `u` also undoes either edit.

Viewer sketch:

```
 20261103_142837.png                                                       84%
 ↳ ~/tmp/screenshots/20261103_142837.png

                               [ image ]

                  ✓ Copied /Users/bryan/tmp/screenshots/20261103_142837.png
  Esc close  + − zoom  0 fit  1 1:1  hjkl pan   ·   y copy original path   r restore path
```

Visual rules (all in `plugins/bob-navigation-hotkeys/styles.css`, `bob-image-zoom-*`):

- **Origin line.** It sits under the title, in `var(--font-monospace)` at
  `var(--font-ui-smaller)`, about 60% white, prefixed `↳`. Long paths are
  middle-truncated and the full path goes in the `title` attribute. The state suffix
  uses a quieter tone: `· changed since import`, `· no longer on disk`, or
  `· on <host>`. It fades in over 120 ms when the lookup returns. Images with no
  recorded origin show no line.
- **Footer.** Replace the plain hint string with key chips. Each key is a `<kbd>` (1px
  `rgba(255,255,255,.28)` border, 4px radius, monospace, `0 5px` padding) followed by a
  muted label. The view group and the source group are separated by a centered `·`.
  Source chips render only for actions that are currently possible, and their labels
  name exactly what `y` will copy (`copy original path` / `copy file path` /
  `copy URL`). The footer is centered and may wrap to at most two lines.
- **Toast pill.** One pill, centered just above the footer: 999px radius,
  `rgba(0,0,0,.72)` fill, 1px hairline border, one line, middle-truncated. Success shows
  a `✓` in `var(--color-green)` and a refusal shows `!` in `var(--color-orange)`. It
  stays 1.8 s (2.6 s for the import pill) and fades over 160 ms; a new pill replaces the
  current one. `@media (prefers-reduced-motion: reduce)` drops all transitions.
- **Notices.** Never raise an Obsidian `Notice` while the viewer is open. Use the pill
  then, and use a `Notice` only after the viewer closes (`r`) or when it never opens
  (import failure).

## Shared contract (owned by bob; phase `origins` implements it)

### Origin journal

The journal is `.bob/attachments/<host>.jsonl`, written through `pref_journal` with a
new `ATTACHMENTS` namespace (dir `.bob/attachments`, noun `attachment`). Update the
module doc so it no longer claims every journal holds a preference. One event per copied
attachment:

```json
{
  "v": 1,
  "ts": "2026-11-03T14:28:40-05:00",
  "host": "<hostname_slug>",
  "kind": "attachment.import",
  "saved": "img/20261103_142837.png",
  "source": "/Users/bryan/tmp/screenshots/20261103_142837.png",
  "display": "~/tmp/screenshots/20261103_142837.png",
  "sha256": "<64 hex>",
  "via": "capture"
}
```

- `source`: the absolute path `%c` resolved (it handles quotes, `file://`, and `~/`).
  `display`: `source` with the writer's home-directory prefix written as `~` (otherwise
  identical to `source`). `sha256`: hash of the copied bytes. `via`: `capture` for `%c`,
  otherwise the import command's `--via` value (default `cli`; the viewer passes
  `zoom`).
- Reused destinations still get an event, because the new source is still this embed's
  origin. Snippet files (`file/clip-*.md`) get none.
- The fold skips malformed lines with the shared bounded warning.

### Origin resolution

`bob attachments origin <VAULT_PATH>` resolves a vault-relative path P. If P does not
exist, it fails with `not_found`.

1. **Candidates.** First, events whose `sha256` equals P's current SHA-256; this
   survives Obsidian renames and dedupe. If there are none, events whose `saved` equals
   P; this covers a vault copy edited after import. If there are none, the result is
   `origin: null`.
2. **Pick.** Take the newest candidate in writer order that was recorded on this host
   and whose source is `present`. If there is none, take the newest candidate.
3. **State.**
   - Recorded on another host → `elsewhere` (never probed).
   - Source missing → `missing`.
   - Source bytes differ from the event's `sha256` → `changed`.
   - Otherwise → `present`.

### `bob attachments` (new command group)

The group goes in the **Vault** section of `bob --help` and in its completion group.
Bare `bob attachments` prints help: it has no flag-only read-only member (see
`sase/memory/cli_rules.md`). Every public long option gets a short alias, and options
are sorted alphabetically.

- **`bob attachments import [-b DIR] [-f human|json] <PATH>...`** (plus the hidden
  internal `--via CLIENT`, like `bob relations hide`).
  - Takes 1–10 paths. Each is classified exactly as a `%c` clipboard line is
    (`classify_path_candidate`) and copied with the `%c` planner: `img/` vs `file/`, the
    name sanitizer, reuse of an identical file, and the 8-character hash suffix on a
    conflict. It never edits notes.
  - Runs under `bob_sync.lock` with a 3 s wait (`ob::acquire_lock_waiting`; busy gives
    `lock_busy`), as `relation_labels/write.rs` does. It appends journal events after
    the files are saved. If the append fails, it removes the files it created (reuse
    `capture_clip::cleanup_created`) and fails.
  - JSON output:
    `{"schema_version":1,"attachments":[{"source","display","saved","kind","reused","embed","sha256"}]}`.
    `embed` is `attachment_reference` output (`![[img/x.png|400]]` / `[[file/x.pdf]]`).
  - Failures reuse `bob relations hide -f json`'s error envelope, with codes `missing`,
    `not_a_file`, `not_absolute`, `too_many`, `lock_busy`, and `io`.
  - Exit codes: 0 ok, 1 failure, 2 usage.
  - Human output is one colored line per file: `imported  ~/tmp/…/x.png → img/x.png` or
    `reused    ~/tmp/…/x.png → img/x.png`.
- **`bob attachments origin [-b DIR] [-f human|json] <VAULT_PATH>`** is read-only.
  - JSON output:
    `{"schema_version":1,"path":"img/x.png","origin":{"source","display","host","ts","via","state"}|null,"warnings":[]}`.
  - Human output:
    `img/x.png ← ~/tmp/…/x.png  present · <host> · 2026-11-03 14:28 · capture`, or
    `img/x.png has no recorded origin`.
  - Exit codes: 0 resolved (including `null`), 1 `not_found` or I/O, 2 usage.

## Phase `origins`: bob records origins and ships `bob attachments` (bob-cli)

1. **Journal and resolution.**
   - Add the `ATTACHMENTS` namespace in `src/native/pref_journal/mod.rs`.
   - Create `src/native/attachments/` with `mod.rs` (facade), `journal.rs` (event
     encode/parse/fold), `origin.rs` (resolution and states as specified above), and
     `cli.rs` (both commands, modeled on `src/native/relation_labels/cli.rs` and
     `write.rs`).
   - Register them in `src/native.rs` and `src/runner.rs` and put them in the Vault
     help/completion section.
2. **Shared planner.** In `src/native/capture_clip/`, expose a crate entry point that
   plans explicit paths through the same `classify_path_candidate` → `plan_attachments`
   path (fresh `ClipReservations`, empty indent) and returns the `ClipPlan`. That plan
   carries per-attachment `source`, `saved`, `kind`, `reused`, the `embed` string, and
   the SHA-256 of the planned bytes. Import saves with `ClipPlan::save`. Any new field
   stays out of `bob capture`'s JSON, which must remain byte-identical.
3. **`%c` records origins.** After a non-dry-run capture commits, append one event per
   committed attachment (`via: "capture"`, history entries included). Put this next to
   `append_committed_relation_label_hides` in `src/native/capture/mod.rs`. Check the
   other `commit_capture_batch` callers (`inbox_route.rs`, `existing_task_link.rs`) and
   record there too if they can carry a clip plan. A failed append becomes a capture
   warning (`could not record attachment origin: …`) and never fails the capture.
4. **Vault allowlist.**
   - Add `!/.bob/attachments/*.jsonl` to `~/bob/.gitignore` and
     `.bob/attachments/*.jsonl merge=union` to `~/bob/.gitattributes`, each with a
     comment in the style of its neighbours.
   - Verify with `git -C ~/bob check-ignore -v .bob/attachments/x.jsonl` and
     `git -C ~/bob check-attr merge -- .bob/attachments/x.jsonl`.
   - Let `bob vault-sync` commit them; do not hand-commit the vault.
5. **Tests.**
   - Unit tests for event round-trip, malformed-line skip, the candidate and pick rules
     (rename via hash, edited copy via `saved`, this host preferred, `elsewhere`,
     `missing`, `changed`, `present`), and the tilde `display`.
   - Import tests: reuse, hash suffix, `too_many`, `missing`, `not_a_file`, quotes,
     `file://`, `lock_busy`, and cleanup when the journal append fails.
   - A capture test that `%c` appends events and leaves capture JSON unchanged, a test
     that `--dry-run` appends nothing, and CLI integration tests under `tests/cli/` for
     both commands in both formats.
6. **Docs** (route per `sase/memory/docs.md`).
   - New root contract `docs/attachments.md` covering the journal, resolution, states,
     import rules, and "vault copies are never deleted". Add it to `mkdocs.yml` nav.
   - In `docs/capture.md`'s clipboard section, add one sentence plus a link saying each
     attachment's origin is recorded.
   - Add an "Attachment origin journal" section to `docs/vault-git-sync.md`, mirroring
     the relation journal section and adding the new `.gitattributes` line to the union
     list.
   - Add rows for both commands to `docs/reference/json-outputs.md` and add
     `.bob/attachments/` to the `docs/reference/vault-format.md` tree.
   - Run `just docs-gen` for the CLI reference pages, then `just docs-check`.
7. **Gate.** `just check` passes.

> [!decision] decision_record Use `/sase_memory_write` to add the `decisions` web strand
> `attachment-origins-live-in-per-host-journal` in the house record format (Applies to /
> Claim / Why with rejected alternatives / Cost / Reopens when). Rejected alternatives:
>
> - Source path in the link alias or an HTML comment: it clutters every embed.
> - Guessing from the file name: it is unreliable.
> - A machine-local store: it would not sync to the Mac.
>
> Cite this plan and `docs/attachments.md`. Then run `sase memory init`.

## Phase `viewer`: origin line, `y`, and `r` (bob-plugins)

Work in the linked `bob-plugins` repo, opened with `/sase_repo`; read its `AGENTS.md`.
Edit only `src/` fragments and run `npm run build`; never hand-edit `main.js`. Keep each
fragment at or below 1000 lines. `207-image-zoom-modal.js` is at 750, so put the new
viewer chrome (origin line, chips, pill, key actions) in a new fragment such as
`207a-image-zoom-origin.js`, listed after `207` in `fragments.json`.

1. **bob-ledger-tools `attachments` api namespace (v1).** Add it beside `relationLabels`
   in `plugins/bob-ledger-tools/src/170-plugin-lifecycle.js`.
   - `origin(vaultPath)` spawns `bob attachments origin <path> -f json`.
   - `import(path, {via})` spawns
     `bob attachments import --via <via> -f json -- <path>`.
   - Both use `bobSpawnJson` with `apiStatsBobPath()` and `statsVaultBasePath()` (mirror
     `relationLabelsWriteSpawn`). They check `schema_version === 1`, return
     `{ok:true, …}` or `{ok:false, code, message}`, never reject, and refuse off desktop
     with `code: "desktop_only"`.
   - The top-level api version stays unchanged (additive). Document the namespace where
     the ledger-tools api namespaces are documented.
2. **Viewer context.** `zoomImageOnLine` (`679-plugin-image-zoom.js`) passes
   `ImageZoomModal` a `context` with these fields:
   - `kind` (`vault`/`url`), `vaultPath`, `absolutePath` (`getVaultBasePath()` joined
     with the vault path), and `url`.
   - `restore`: `{view, editor, filePath, line, lineText, start, end}` for the picked
     reference on the cursor line.
   - `origin`: a value or a promise.
   - `justImported`.

   For vault images it starts `attachments.origin(vaultPath)` as the viewer opens and
   renders the origin line when the lookup resolves.

3. **Keys.** `imageZoomKeyAction` maps unmodified `y` → `copySource` and `r` →
   `restoreSource`, and ignores auto-repeat. Both work before the image finishes
   loading. Existing keys are unchanged.
4. **`y` copy.** If an origin was recorded (any state), copy its path in the form chosen
   by `copy_form`. Otherwise copy the vault file's absolute path for vault images, or
   the URL for web images. If the origin lookup is still in flight, `y` waits for it. It
   writes with `navigator.clipboard.writeText` directly (not `writeTextToClipboard`,
   which raises a `Notice`), then shows `✓ Copied <value>` in the pill, or
   `! Couldn't copy` on failure.

   > [!decision] copy_form = absolute Copy the origin's `source`. Every source chip
   > label still reads `copy original path`.

   > [!decision] copy_form = tilde Copy the origin's `display`.

5. **`r` restore.** `r` is offered only when `restore` exists and the origin `state` is
   `present`. Otherwise it shows a refusal pill:
   - no origin → `No original path recorded`
   - `missing` → `Original is no longer on disk`
   - `changed` → `Original changed since import · y still copies it`
   - `elsewhere` → `Original lives on <host>`
   - URL → `Web images have no original path`

   On success:
   - Confirm that the active view and file are unchanged and that `editor.getLine(line)`
     still equals `lineText`. If not, show `! Line changed · nothing restored`.
   - Replace `[start, end)` with the origin's `display` in one editor change
     (`editor.transaction`, falling back to `replaceRange`) so vim `u` undoes it.
   - Point the restore snapshot's cursor at the start of the path, close the viewer, and
     raise `Notice("Restored ~/tmp/…/x.png · vault copy kept at img/x.png")`.
   - Never delete the vault copy.

6. **Chrome.** Build the origin line, key-chip footer, and toast pill exactly as
   described in "The experience" (replacing `IMAGE_ZOOM_FOOTER_HINT`), with the CSS in
   `plugins/bob-navigation-hotkeys/styles.css`.
7. **Tests.**
   - Add `scripts/test-navigation-image-zoom-origin.cjs` and
     `scripts/test-ledger-tools-attachments-api.cjs`, and register both in
     `package.json`'s `test` script.
   - Extend the existing `test-navigation-image-zoom-{viewer,command,target}.cjs`. Their
     harness editor throws on `replaceRange`; keep that for non-`r` paths.
   - Cover: every `y` source and label; every `r` refusal; the exact text and single
     transaction of a successful `r`; the line-changed guard; modifiers ignored; pill
     replacement; each origin-line state; api decode errors and `desktop_only`.
8. **Docs.**
   - In `docs/image-zoom.md`, add `y`/`r` rows to the key table, an "Origin" paragraph,
     and replace "The command and viewer never write the vault" with the precise rule:
     only `r` writes, and only that one link.
   - Update the `bob-navigation-hotkeys` manifest description sentence that says `Z`
     "never writes the vault", and the matching prose in
     `docs/bob-navigation-hotkeys.md`.
   - Run `npm run docs:gen`.
9. **Gate.** Run `npm run build`, `npm test`, and `npm run docs:check`, then
   `bob plugins sync`.

## Phase `pathz`: `Z` on an image-path bullet imports it (bob-plugins)

1. **Detection.** Add a pure `findImagePathBullet(lineText)`, exported through `helpers`
   for tests. It runs only when the line has no image candidates and is outside a fence,
   so existing image references keep priority.
   - The line is optional indentation, then an optional `-`, `*`, `+`, or `N.`/`N)`
     marker and whitespace, then the content. Content starting with a task checkbox
     (`[ ]`, `[x]`, …) is rejected.
   - The trimmed content must be exactly one path: optionally wrapped in matching `'…'`,
     `"…"`, or `<…>`, starting with `~/`, `/`, or `file://`, and ending in an
     `IMAGE_ZOOM_EXTENSIONS` extension (case-insensitive). Spaces inside are fine
     because the path is the whole content.
   - It returns `{start, end, raw}` spanning the content, including any wrapper.
   - A path-only bullet with a non-image extension raises `Not an image: <path>`.
2. **Import.** Add `importImagePathOnLine`, which is async and single-flight
   (`this.imageZoomImporting`; a second `Z` while importing is ignored).
   - If the api is missing, raise
     `Notice("Importing images needs bob-ledger-tools on desktop")`.
   - Call `attachments.import(raw, {via: "zoom"})`; bob strips wrappers and expands `~`.
   - On failure, raise `Notice("Couldn't import <path>: <bob message>")`. Treat
     `kind !== "image"` as a failure.
3. **Rewrite.** Re-check that the active view and file are unchanged and that the line
   text equals the snapshot. If they differ, raise
   `Notice("Line changed while importing · image saved at <saved>; note unchanged")`.
   Otherwise replace `[start, end)` with bob's `embed` in one editor change, leaving the
   cursor at the embed's start.
4. **Open the viewer** on `saved`.
   - `src` comes from `app.vault.adapter.getResourcePath(saved)`, because the metadata
     cache may not know the new file yet. The title is the basename.
   - The context's origin is built from the import result
     (`{source, display, state: "present", via: "zoom"}`, so no second spawn), its
     `restore` span is the new embed's span, and `justImported: true`.
   - The pill reads `Imported to <saved> · r to undo`, or
     `Linked existing <saved> · r to undo` when `reused`.
5. **Tests.**
   - Detection matrix: the three bullet markers, numbered markers, a plain line, nested
     indent, quotes, `<…>`, `file://`, spaces, an uppercase extension, a rejected
     checkbox, a rejected path inside prose, a non-image extension, a fence, and a line
     with both an embed and a path (the embed wins).
   - Flow tests: import success (one transaction, the exact embed text, viewer context);
     each failure notice; the line-changed guard; single flight; and the round trip `Z`
     → `r`, which restores the bullet in `display` form.
6. **Docs.**
   - Add an "Import an image path" section to `docs/image-zoom.md`.
   - Update the manifest description and `docs/bob-navigation-hotkeys.md` prose.
   - Update the comment on line 60 of the vault's `~/bob/obsidian_vimrc.md` to: "Z opens
     the image on the cursor line; on a bullet holding only an image path it imports the
     file first." The `nnoremap` stays as is; vault-sync commits it.
7. **Gate.** Run `npm run build`, `npm test`, and `npm run docs:check`, then
   `bob plugins sync`.

## Out of scope

- Backfilling origins for images imported before this epic. They show `copy file path`
  and no `r`.
- Deleting or garbage-collecting vault copies. `r` never deletes, because other notes
  may embed the same file.
- `Z` importing non-image files or several paths per line; visual-mode selections,
  Reading view, and mobile.
- Bob Mac Capture changes: `%c`'s JSON output is unchanged.
