---
tier: tale
size: small
title: Fix the image zoom viewer's invisible footer keys and dead `r`
goal:
  The `Z` viewer footer shows readable key chips, and `y`, `r`, the origin line, and `Z`
  path import see bob's recorded origin, so `r` restores a `%c` image to its original
  path.
proposed_by: bbugyi200.athena.17w
proposed_with: claude/opus
create_time: 2026-11-04 07:21:30
status: wip
---

# Plan: fix the image zoom viewer's invisible footer keys and dead `r`

This plan follows up on closed epic `bob-cli-gn` ("Image zoom remembers where images
came from", plan `plan:202611/image_zoom_origins.md`). Bryan's screenshot of the `Z`
viewer on the Mac shows two problems:

1. The footer key chips (`Esc`, `+ / −`, `0`, `1`, `hjkl`, `y`) render as solid white
   boxes with unreadable text.
2. Pressing `r` on `img/20261104_062032.png` shows `! No original path recorded`. The
   footer offers only `y copy file path` and never shows `r restore path`, even though
   the image was imported with `%c` and has a recorded origin.

All the work is in the linked **bob-plugins** repo. Open it with `/sase_repo` and read
its `AGENTS.md`. Edit only `src/` fragments and `styles.css`, run `npm run build`, and
never hand-edit `main.js`. bob-cli needs no changes.

## Root causes (verified)

### 1. The attachments API drops bob's payload (breaks `r`, `y`, the origin line, and path import)

`plugins/bob-ledger-tools/src/173-plugin-attachments-api.js` decodes bob's output with
`attachmentsDecode(...)`. The decoder returns `{ ...body, ok: true }`, so the bob
payload fields sit at the top level. `bobSpawnJson` (`src/236-plugin-stats-data.js`, the
`decodeOutput` closure) rebuilds every success result as
`{ ok: true, report: result.report, exitCode }`. Every other caller's decoder returns
`{ ok: true, report: body }` (task merge, stats, period strip, keep inbox, relation
labels), so for them the payload survives. The attachments decoder never sets `report`,
so its fields are dropped.

Reproduced on athena by calling `attachmentsApi().origin("img/20261104_062032.png")`
through the real `bobSpawnJson` with the real bob binary and `BOB_DIR=~/bob`. The result
was `{"ok":true,"exitCode":null}`, with no `origin`. Running
`bob attachments origin img/20261104_062032.png -f json` directly returns the full
payload: state `elsewhere` on athena, and `state: "present"` on the Mac (checked over
`ssh mac`; the source `~/tmp/screenshots/20261104_062032.png` still exists there).

The rest of the chain follows from the dropped payload:

- In `plugins/bob-navigation-hotkeys/src/207a-image-zoom-origin.js`,
  `imageZoomNormalizeOrigin({ok: true})` follows `value.origin` (undefined) and returns
  `null`.
- So the viewer believes no origin was recorded. The footer says `copy file path`, the
  `r` chip is never offered, and `r` refuses with `No original path recorded`.
- The same bug breaks `Z` on an image-path bullet (`src/679a-image-zoom-path.js`,
  `importImagePathOnLine`). bob copies the file, but `result.attachments` is undefined,
  so the plugin raises `Not an image: <path>` and leaves the note unchanged. The journal
  shows no `via: "zoom"` events, so no orphan copies exist yet.

The epic's tests missed this because `scripts/test-ledger-tools-attachments-api.cjs`
stubs `attachmentsSpawn` and calls the decoders directly. Nothing ever ran a decoder
through `bobSpawnJson`.

### 2. The `kbd` chips have no background (invisible footer keys)

In `plugins/bob-navigation-hotkeys/styles.css`, `.bob-image-zoom-chip kbd` sets a white
`color` but no `background`. Obsidian's built-in `kbd` style supplies a light background
(no vault theme or snippet styles bare `kbd`), which gives white text on a white box. In
the screenshot the chip boxes measure 250–255 brightness.

## Changes

### A. bob-ledger-tools: keep the attachments payload (`src/173-plugin-attachments-api.js`)

1. Make `attachmentsDecode` return `{ ok: true, report: body }` on success, matching the
   other `bobSpawnJson` decoders. Failure results keep their current shape and codes.
2. In `attachmentsSpawn`, unwrap the success result so the public contract documented in
   `docs/develop/plugin-apis.md` holds exactly:
   - `origin()` resolves `{ ok: true, schema_version, path, origin, warnings }`.
   - `import()` resolves `{ ok: true, schema_version, attachments }`.

   That is, `result.ok === true && result.report` is an object → return
   `{ ...result.report, ok: true }`, and pass failures through unchanged. The API must
   still never reject.

3. Add a test seam: pass `execFile` to `bobSpawnJson` from an optional
   `this.attachmentsExecFile` when it is a function. Leave it undefined in production so
   the existing `child_process` lookup is used.

### B. bob-navigation-hotkeys: make the chips readable (`styles.css`)

Restyle the chip as `.modal.bob-image-zoom-modal .bob-image-zoom-chip kbd`. The higher
specificity beats Obsidian and theme `kbd` rules. Keep the existing border, radius,
monospace font, size, padding, and line height, and add:

- `background: rgba(255, 255, 255, 0.08)`: a faint key fill that reads on the dark
  footer in both light and dark themes.
- `box-shadow: none`, `text-shadow: none`, and `color: rgba(255, 255, 255, 0.92)`.

Do not change the footer layout, the chip set, or the labels.

### C. bob-navigation-hotkeys: don't report a failed lookup as "no origin" (`src/207a-image-zoom-origin.js`)

A failed lookup currently collapses to `null` and reads as "No original path recorded".
That is what hid this bug. Change it as follows:

- When the origin input resolves to `{ ok: false, … }`, record the failure message on
  the modal (for example `this.imageZoomOriginError`) and keep `imageZoomOrigin = null`.
  Treat `{ ok: true }` without an own `origin` property as a failure with the message
  `unexpected response from bob`. A rejected promise is a failure too.
- Leave these cases unchanged:
  - `ok: true` with `origin: null` still means "no origin recorded".
  - Raw origin objects (the import flow) still normalize as before.
  - A `null` input (no ledger-tools api) is still treated as no origin.
- `r`: when there is no origin and a lookup error was recorded, show
  `! Couldn't look up the original · <message>` instead of
  `! No original path recorded`. Keep the other refusals and their order (URL first).
- `y` and the footer are unchanged. On failure `y` still falls back to the vault file's
  absolute path, labelled `copy file path`.

Keep the fragment at or below 1000 lines (it is about 275 now).

## Tests

1. `scripts/test-ledger-tools-attachments-api.cjs`:
   - Update the decoder assertions to the `{ ok: true, report: {...} }` shape.
   - Add an end-to-end seam test through the real `attachmentsSpawn` → `bobSpawnJson`.
     Set `attachmentsIsDesktop = () => true`, give the plugin `apiStatsBobPath` and
     `statsVaultBasePath`, and set a fake `attachmentsExecFile` that calls back with
     bob's real stdout. Capture that stdout from the bob 0.4.0 command:

     ```json
     {
       "origin": {
         "display": "~/tmp/screenshots/20261104_062032.png",
         "host": "kellys-mbp",
         "source": "/Users/bbugyi/tmp/screenshots/20261104_062032.png",
         "state": "present",
         "ts": "2026-11-04T06:31:44-05:00",
         "via": "capture"
       },
       "path": "img/20261104_062032.png",
       "schema_version": 1,
       "warnings": []
     }
     ```

     Assert that `api.origin(...)` resolves with `ok: true` and
     `origin.display`/`origin.state` intact. Also assert that the fake saw the expected
     args, `BOB_DIR`, and `NO_COLOR`.

   - Add the same seam test for `api.import(...)` with an import payload. Assert that
     `attachments[0].embed`, `saved`, `kind`, and `reused` survive.
   - Add a failure case: the fake calls back with an error whose `code` is `1` and
     stdout
     `{"code":"not_found","message":"no such vault file: img/nope.png","ok":false}`.
     Assert that the result is `ok: false`, `code: "not_found"` with that message.

2. `scripts/test-navigation-image-zoom-origin.cjs`:
   - When the origin input resolves to the fixed API shape
     (`{ ok: true, schema_version: 1, path, origin: { display, state: "present", … }, warnings: [] }`),
     the footer shows `copy original path` and `restore path`, and `r` performs the one
     restore transaction.
   - When it resolves to `{ ok: false, code: "bob-not-found", message: "…" }`, `r` shows
     the new `Couldn't look up the original · …` pill, and the footer shows
     `copy file path` with no `r` chip.
   - `{ ok: true }` without `origin` behaves as a lookup failure, and
     `{ ok: true, origin: null }` still shows `No original path recorded`.
   - Add a stylesheet contract test. Read `plugins/bob-navigation-hotkeys/styles.css`,
     find the rule whose selector ends in `.bob-image-zoom-chip kbd` and includes
     `.bob-image-zoom-modal`, and assert that it declares `background`,
     `box-shadow: none`, and a white `color`.
3. Run the existing image-zoom suites unchanged and confirm they still pass:
   `test-navigation-image-zoom-{viewer,command,target,path}.cjs`. The path-import flow
   test should keep passing; it feeds the API's documented shape.

## Docs

`docs/develop/plugin-apis.md` already describes the correct contract; leave it alone. In
`docs/image-zoom.md`, add one clause to the `r` paragraph: when bob can't be reached or
the lookup fails, `r` says so instead of claiming no origin was recorded. Run
`npm run docs:check`.

## Verification and deploy

1. Run `npm run build`, then the focused suites:

   ```sh
   node --test scripts/test-ledger-tools-attachments-api.cjs \
     scripts/test-navigation-image-zoom-origin.cjs \
     scripts/test-navigation-image-zoom-viewer.cjs \
     scripts/test-navigation-image-zoom-path.cjs \
     scripts/test-navigation-image-zoom-command.cjs \
     scripts/test-navigation-image-zoom-target.cjs
   ```

   Then run `npm run build:check` and `npm run docs:check`.

2. Live seam check on athena (no vault writes). Load `scripts/ledger-tools-harness.cjs`
   from a throwaway script outside the repo and build a `LedgerToolsPlugin` with
   `attachmentsIsDesktop = () => true`, the real bob path, and `~/bob` as the vault.
   Call `attachmentsApi().origin("img/20261104_062032.png")`. The result must include
   `origin.display === "~/tmp/screenshots/20261104_062032.png"` with
   `state: "elsewhere"`, since the origin was recorded on `kellys-mbp`. Before the fix
   this printed `{"ok":true,"exitCode":null}`.
3. Run the full `npm test`. Failures that match the known open reports are acceptable;
   record them, but do not fix them here:
   - `bob-cli-et`: freshness-namespace
   - `bob-cli-f7`: keep-inbox forced-check
   - `bob-cli-eh`: note-ready, relation-index
   - `bob-cli-ff`: schedule-work-log
   - `bob-cli-fz`: reduced-motion capture
   - `bob-cli-f8`: task-wikilink CPU budget

   Any new failure in the image-zoom, attachments, or lifecycle suites must be fixed.

4. Run `bob plugins sync` to deploy `bob-ledger-tools` and `bob-navigation-hotkeys`. If
   sync refuses because of unrelated vault drift, report it rather than forcing it.
   `bob vault-sync` carries the deployed files to the Mac.
5. Commit the bob-plugins changes as one `fix(image-zoom): …` commit.
6. In the final summary, tell Bryan to reload both plugins on the Mac (or restart
   Obsidian) and then retry `Z` on `img/20261104_062032.png`. The footer should show
   readable chips, `y copy original path`, and `r restore path`, and `r` should restore
   `~/tmp/screenshots/20261104_062032.png`.

## Out of scope

- bob-cli changes: `bob attachments` output is correct on both hosts.
- Other `bobSpawnJson` callers: they already return `{ ok, report }`.
- Backfilling origins, and the epic's known flaky suites.
