---
tier: tale
title: Restore new_note.md on Cmd+N via Templater's device-local trigger
goal:
  Every device's new empty notes, including Cmd+N notes, expand _templates/new_note.md
  again.
size: small
decisions:
  trigger_policy:
    ask:
      Should bob-navigation-hotkeys turn on Templater's device-local new-file trigger
      itself when unset?
    choices:
      enable:
        Turn it on at startup with a notice; each device fixes itself after `bob plugins
        sync`
      warn:
        Leave it off; show a sticky notice and a command, so each device needs one
        manual step
    default: enable
    why:
      bob-* plugins deploy per device, so syncing them is already Templater's per-device
      trust step
    answer: enable
proposed_by: bbugyi200.apollo.8o
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.8o](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.8o.md)
- **COMMITS:**
  - [b760f5e](https://github.com/bobs-org/bob-plugins/commit/b760f5ec8d8d14c9b5116fa769f1e2f51be40c0e)
    — feat(bob-navigation-hotkeys): enable Templater's per-device new-note trigger

# Plan: Restore `_templates/new_note.md` on Cmd+N (Templater device-local trigger)

## Root cause

Cmd+N is Obsidian's built-in `file-explorer:new-file` command. Nothing in the vault
`hotkeys.json` remaps it. It creates an empty `Untitled.md` in the vault root and relies
on Templater's folder-template rule (`"/" -> _templates/new_note.md`, added in vault
commit `9ce416e4`) to fill the note in.

On 2026-09-08 Templater was updated to 2.25.x on the MacBook. The vault commit is
`ada826b1 vault(kellys-mbp): …templater-obsidian/data.json…`. Templater's
settings-schema migration (`data_version: 2`) **deleted
`"trigger_on_file_creation": true` from `data.json`**. That setting now lives in
per-device Obsidian localStorage, under the key `templater-local-settings`, together
with `enable_system_commands` and `enable_startup_templates`, and it **defaults to
`false`**. Evidence from the installed 2.25.1 `main.js`:

- `var an={trigger_on_file_creation:!1,…},sn="templater-local-settings";function Ve(e){let t=e.loadLocalStorage(sn);return Object.assign({},an,t??{})}`
- `static async on_file_creation(t,r,i){if(!Ve(r).trigger_on_file_creation||…)return;…`
  The trigger is read **live** on every `vault.on("create")`.
- `load_settings` shows a one-time Notice, "settings were reset because they are now
  device-local … Re-enable them in Templater settings". Only the device that ran the
  migration sees it (the MacBook). Every other device receives the already-migrated
  `data.json` through git and never sees the notice.

The MacBook's Obsidian localStorage leveldb has vault keys such as
`743d678dd02bde4e-recent-commands`, but **no
`743d678dd02bde4e-templater-local-settings`**. So the trigger falls back to `false`, and
Templater silently skips every new file. `data.json` still holds the folder rule and
`trigger_on_file_creation_mode: "folder"`. Other plugins were ruled out: obsidian-linter
lints on save only, metadata-menu inserts nothing at creation, and no bob plugin binds
Mod+N.

Re-adding the key to the git-synced `data.json` would do nothing, because Templater no
longer reads it from there. The fix has to reach each device's localStorage.

## Fix

Work in the **`bob-plugins`** linked repo (open it with `/sase_repo`; read its
`AGENTS.md`). Plugin: `plugins/bob-navigation-hotkeys`. It already owns Templater-based
note creation: `getTemplaterPlugin()`, and
`NOTE_TEMPLATE_PATHS.default === "_templates/new_note.md"`. It uses the committed source
build: edit `src/` fragments and never hand-edit `main.js`.

### 1. New fragment `src/678-plugin-templater-trigger.js`

Register it in `src/fragments.json` immediately after `677-plugin-ref-pdf-open.js`. Keep
it under 1000 lines (it should be well under 150).

- Constants: `TEMPLATER_PLUGIN_ID = "templater-obsidian"`,
  `TEMPLATER_LOCAL_SETTINGS_KEY = "templater-local-settings"`,
  `TEMPLATER_LOCAL_TRIGGER_KEY = "trigger_on_file_creation"`.
- A pure helper, `planTemplaterLocalTrigger(templaterSettings, storedLocal)`, that
  returns a frozen `{ action: "enable", next }` or `{ action: "skip", reason }`.
  Evaluate the rules in this order:

  | Condition                                                                                                                                                                             | Result                                                                                                                            |
  | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
  | `templaterSettings` is not an object                                                                                                                                                  | skip `no-templater`                                                                                                               |
  | `typeof templaterSettings.trigger_on_file_creation_mode !== "string"` (pre-2.25 Templater keeps the trigger in `data.json`)                                                           | skip `legacy-templater`                                                                                                           |
  | mode is `"none"`, or mode is `"folder"` with no `folder_templates` entry that has a non-empty `template`, or mode is `"regex"` with no such `file_templates` entry, or any other mode | skip `no-creation-templates`                                                                                                      |
  | `storedLocal` is neither `null`/`undefined` nor a plain non-array object                                                                                                              | skip `unknown-local-shape` (never clobber an unrecognized value)                                                                  |
  | own `trigger_on_file_creation === true`                                                                                                                                               | skip `already-enabled`                                                                                                            |
  | own `trigger_on_file_creation` present with any other value                                                                                                                           | skip `device-opt-out`. Only Templater's settings toggle writes this key, so a stored `false` is an explicit choice on that device |
  | otherwise                                                                                                                                                                             | enable, with `next = { ...storedLocal, trigger_on_file_creation: true }`                                                          |

  The helper must never add, change, or remove `enable_system_commands` or
  `enable_startup_templates`. The spread preserves them exactly as stored.

- A mixin, `BobNavigationHotkeysTemplaterTriggerMixin`, with
  `ensureTemplaterNewNoteTrigger()`:
  - Returns quietly when `this.app.loadLocalStorage` or `this.app.saveLocalStorage` is
    not a function.
  - Reads `this.app.plugins?.plugins?.[TEMPLATER_PLUGIN_ID]?.settings`. Do not reuse
    `getTemplaterPlugin()`, which checks for `create_new_note_from_template` rather than
    `settings`. Then reads `this.app.loadLocalStorage(TEMPLATER_LOCAL_SETTINGS_KEY)`.
  - Acts on the plan's result:

> [!decision] trigger_policy = enable On `enable`: call
> `this.app.saveLocalStorage(TEMPLATER_LOCAL_SETTINGS_KEY, next)`, then show exactly one
> Notice:
> `Templater: turned on "Trigger on file creation" for this device, so new notes use the vault's folder templates`.
> On a `skip`, show no notice and write nothing.

> [!decision] trigger_policy = warn On `enable`, write nothing. Show one sticky Notice
> (`new Notice(text, 0)`):
> `Templater's device-local "Trigger on file creation" is off, so new notes skip _templates/new_note.md. Run "Enable Templater new-note trigger on this device".`
> Register a palette command in `490-plugin-lifecycle.js` with no default hotkey: id
> `enable-templater-new-note-trigger`, name
> `Enable Templater new-note trigger on this device`. The command reruns the plan and
> performs the `saveLocalStorage` write, with the `enable` branch's notice. On a skip it
> shows a short notice naming the reason, for example
> `Templater new-note trigger is already on for this device`.

- Wrap the read/write in `try/catch`. A failure must never throw out of startup. On a
  write failure, show at most one Notice:
  `Could not turn on Templater's new-note trigger for this device`.
- Return the plan result, so tests can assert on it.

### 2. Wiring

- `src/690-install-plugin-mixins.js`: add `BobNavigationHotkeysTemplaterTriggerMixin` to
  the install list. Placing it next to `BobNavigationHotkeysRefPdfOpenMixin` is fine.
- `src/490-plugin-lifecycle.js`: at the end of the existing
  `this.app.workspace.onLayoutReady(() => { … })` callback, call
  `this.ensureTemplaterNewNoteTrigger();`. Templater loads before layout-ready, and
  because it reads the trigger live, no reload is needed after the write. Add a one-line
  comment: Templater 2.25 made the trigger device-local (default off), and this restores
  the vault's folder-template intent on each device.
- `src/700-exports.js`: export the three constants and `planTemplaterLocalTrigger` on
  `module.exports.helpers`.

### 3. Tests: `scripts/test-navigation-hotkeys-templater-trigger.cjs`

Follow `scripts/test-navigation-hotkeys-ref-pdf.cjs`: require
`./navigation-hotkeys-harness.cjs`, build the plugin with
`Object.create(NavigationHotkeysPlugin.prototype)` and a stub `app` that has
`loadLocalStorage` / `saveLocalStorage` (record calls) and
`plugins.plugins["templater-obsidian"].settings`. Add the file to the `test` script in
`package.json`, next to the other `test-navigation-hotkeys-*` entries. Cover:

1. Key absent, mode `"folder"` with the `/ -> _templates/new_note.md` rule: one
   `saveLocalStorage("templater-local-settings", { trigger_on_file_creation: true })`
   and one notice.
2. Existing object `{ enable_system_commands: false, enable_startup_templates: true }`:
   the merged write preserves both keys untouched and adds only the trigger.
3. Stored `trigger_on_file_creation: true`: no write and no notice (`already-enabled`).
4. Stored `trigger_on_file_creation: false`: no write and no notice (`device-opt-out`).
5. Mode `"none"`, and mode `"folder"` with an empty or blank-template list: no write
   (`no-creation-templates`). Mode `"regex"` with a non-empty `file_templates` template
   enables.
6. Legacy settings (boolean `trigger_on_file_creation`, no mode key): no write
   (`legacy-templater`).
7. Templater not installed, or `settings` missing: no write and no throw
   (`no-templater`).
8. `loadLocalStorage` / `saveLocalStorage` missing: returns without throwing.
9. Stored value is a string or an array: no write (`unknown-local-shape`).
10. `saveLocalStorage` throws: no exception escapes, and exactly one failure notice.
11. Pure-helper cases for `planTemplaterLocalTrigger`, including that `next` never
    contains keys absent from the input other than the trigger.

Under `trigger_policy = warn`, swap cases 1–2 to assert no write plus the sticky notice,
and add tests for the palette command's write and its skip notices.

### 4. Docs and version

- `plugins/bob-navigation-hotkeys/manifest.json`: bump `version` 2.42.0 → 2.43.0, and
  append one sentence to `description`. For example:
  `Turns on Templater's device-local "Trigger on file creation" when it is unset, so Cmd+N and other new notes get the vault's folder template (_templates/new_note.md).`
  Reword it to match the warn branch if that branch is chosen.
- `README.md`:
  - In the Plugins table, update the Bob Navigation Hotkeys version cell to `2.43.0` and
    append the same sentence to that row.
  - Add a `## Templater new-note trigger` section right after
    `## Open reference PDF in Highlights`, in 4–8 lines. State that Templater 2.25 moved
    `trigger_on_file_creation` into per-device localStorage (default off) and dropped it
    from the synced `data.json`. Describe what the plugin does at layout-ready under the
    chosen branch, and say that a device's explicit "off" in Templater settings is
    respected. State that it never touches `enable_system_commands` or
    `enable_startup_templates`, and that editing `data.json` cannot restore the trigger.

### 5. Build, verify, deploy

From the bob-plugins checkout:

```bash
npm run build
npm test
npm run validate
```

All three must pass, and `npm run build:check` must report the generated `main.js` as
current. Then deploy to this host's vault as bob-plugins' `AGENTS.md` requires. A bare
`bob plugins sync` refuses from a linked checkout and prints the `--repo` form to run,
so run `bob plugins sync --repo <the bob-plugins checkout path printed by /sase_repo>`.
Confirm that `bob plugins list --no-pull --repo <same path>` shows
`bob-navigation-hotkeys` as `synced`.

## Rollout notes for the final response

Do not SSH into the MacBook to change it, and do not restart Obsidian anywhere. Tell
Bryan:

1. **Immediate workaround (any device):** Settings → Templater → turn on "Trigger
   Templater on new file creation". It takes effect immediately, because Templater reads
   it live.
2. **Durable fix on the MacBook:** after this commit is pushed, run `bob plugins sync`
   (or `just install-all`) on the Mac, then reload Obsidian (Cmd+R). At layout-ready the
   plugin enables the trigger (or, under `warn`, prompts) on that device. Cmd+N then
   produces a note with `parent:` and `created:` frontmatter from
   `_templates/new_note.md`.

## Out of scope / rejected alternatives

- **Re-adding `trigger_on_file_creation` to the vault's Templater `data.json`.**
  Templater 2.25 ignores it there, so the edit would have no effect.
- **Binding Cmd+N to a bob command that calls `create_new_note_from_template`.** That
  fixes only Cmd+N. File-explorer "New note", link-created notes, and every other
  empty-note path would stay untemplated, which defeats the root folder rule from
  `9ce416e4`.
- **Writing Obsidian's localStorage leveldb from bob-cli.** That is unsafe while
  Obsidian is running, and it is platform-specific.
- **Touching Templater's other device-local security settings, changing Templater
  itself, or changing bob-cli code.** None of this is needed.
