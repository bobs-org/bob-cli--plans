---
tier: tale
title: Make Ctrl+Shift+Alt+L reach Obsidian on the MacBook
goal:
  Ctrl+Shift+Alt+L in Obsidian on the MacBook opens the Linked from stage instead of
  being swallowed by Amethyst.
size: small
decisions:
  apply_now:
    ask:
      Also run the new hook on the MacBook over SSH now, so the chord works without
      waiting for chezmoi update?
    default: true
    why:
      The Mac is reachable over SSH and the hook is idempotent, so a later chezmoi run
      is a no-op
    answer: true
proposed_by: bbugyi200.athena.13n.w0
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.13n.w0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.13n.w0.md)
- **COMMITS:**
  - [d3d7f17](https://github.com/bbugyi200/dotfiles/commit/d3d7f171654964e16007b7f743a0fd154965f006)
    — feat(chezmoi): free Amethyst linked-from shortcut

# Make Ctrl+Shift+Alt+L reach Obsidian on the MacBook

## Finding: the keymap already exists, but macOS never delivers the chord

Bryan asked for a new `Ctrl+Shift+Alt+L` Obsidian keymap that works like `Ctrl+Shift+L`
but links from another task to the current task. That command already exists:

- bob-plugins commit `3ba8bf9` ("feat(navigation): add linked-from relation stage",
  2026-10-20) registers `bob-navigation-hotkeys:task-card-linked-from` ("Task card:
  Linked from (task relation)") with the default hotkey `Ctrl+Shift+Alt+L`, next to
  `task-card-links` on `Ctrl+Shift+L`. See
  `plugins/bob-navigation-hotkeys/src/490-plugin-lifecycle.js` and
  `openLinkedFromStageAtCursor` in `src/427-relation-stage-plugin.js`.
- The command opens the Task Card's "🪢 Linked from" stage for the task under the
  cursor. Enter on a RESULTS task writes a link to the current task onto that task's 🪢
  LINKS line, so the current task's LINKED FROM row is updated. `⇥` marks several
  sources, and `⇧↵` adds a label (fixed today in bob-plugins `3b0fa77`).
- The plugin is deployed to both vaults. athena has 2.56.2, and the MacBook has 2.56.1.
  Both `main.js` files contain `task-card-linked-from`, and neither vault's
  `hotkeys.json` overrides it. The relation-stage tests pass (40/40), including
  `scripts/test-navigation-relation-links-stage.cjs` and
  `scripts/test-relation-index-linked-from-stage.cjs`.

**Root cause:** Bryan uses Obsidian on the MacBook, and Amethyst (the tiling window
manager, running there) claims the chord system-wide. Its preferences contain:

```
"KeyboardShortcuts_swap-screen-cw" = "{\"carbonKeyCode\":37,\"carbonModifiers\":6656}";
```

Carbon key code 37 is `kVK_ANSI_L`. Modifiers 6656 = Control (4096) + Option (2048) +
Shift (512). That is Amethyst's default "Swap focused window to clockwise screen"
(`mod2 + l`, where `mod2 = option+shift+control`). A Carbon global hotkey consumes the
keystroke before Obsidian sees it, so `Ctrl+Shift+Alt+L` silently does nothing (the Mac
has a single display, so the swap has no visible effect either). `Ctrl+Shift+L` is not
claimed by anything, which is why it works.

Audit of the other Alt chords: Amethyst's `mod2` keys are Space, H, J, K, L, Z, W, E, R,
Q, G, T, X, the digits, and Left/Right. Its `mod1` (Option+Shift) keys do not overlap
any Bob chord. The Bob Ctrl+Shift+Alt chords are B, L (bob-plugins defaults), and N
(vault `hotkeys.json`). **L is the only collision.** Nothing else claims the chord:
Hammerspoon binds only Cmd+Alt+Ctrl+V and Ctrl+Alt+Shift+S, Alfred binds only Cmd+Space
and Option+C, and no macOS symbolic hotkey or `NSUserKeyEquivalents` uses it.

So the fix is to free the chord on the Mac, through the chezmoi dotfiles that already
manage Mac hotkeys. A precedent is `run_onchange_after_disable_dictation_hotkey.tmpl`.

## Where the work happens

All changes are in the **`chezmoi`** linked repo (Chezmoi-managed dotfiles). Open it
with `/sase_repo` (`sase repo open chezmoi -r "<why>"`) and read its `AGENTS.md` before
editing. It requires `chezmoi update -a --force` after commits. On Linux the new hook
renders empty, so that is a no-op on athena. There are **no bob-cli or bob-plugins
changes**.

## How Amethyst stores the shortcut (verified upstream)

- Amethyst registers each command through `HotKeyManager.registerHotKey`
  (`Amethyst/Managers/HotKeyRegistrar.swift` in ianyh/Amethyst). The name is
  `KeyboardShortcuts.Name("swap-screen-cw")` with no `initial:` shortcut, and the
  UserDefaults key is `KeyboardShortcuts_swap-screen-cw`.
- The defaults are re-seeded only when
  `getShortcut(for:) == nil && (!isMigrated || override)`. On the Mac,
  `migrated-swap-screen-cw = 1` and there is no `~/.amethyst*` file, so `override` is
  false. A removed key therefore stays removed.
- Amethyst's own Settings → Shortcuts "clear" calls
  `KeyboardShortcuts.setShortcut(nil, for:)`. For a name without an initial shortcut,
  that **removes** the key (see `userDefaultsRemove` in sindresorhus/KeyboardShortcuts).
  Deleting the key is therefore exactly the state the app's UI produces.
- KeyboardShortcuts does not observe external `defaults` writes, so Amethyst must be
  quit before the write and relaunched after it.

## Changes

### 1. New darwin-only hook

Add `home/.chezmoiscripts/run_onchange_after_disable_amethyst_swap_screen_cw.tmpl`.
Model it on `run_onchange_after_restart_hammerspoon.tmpl`: wrap it in
`{{- if eq .chezmoi.os "darwin" -}} … {{- end -}}`, start with `#!/bin/bash` and
`set -uo pipefail`, and end with the `# vim: ft=bash` trailer. Do **not** source
`~/.local/share/chezmoi/lib/chezmoi_utils.sh`; plain `echo` keeps the hook testable on
Linux, as in the Hammerspoon hook.

The header comment explains the following:

- Why: Amethyst's default `mod2 + l` "Swap focused window to clockwise screen" (Carbon
  key code 37, modifiers 6656 = Ctrl+Option+Shift) swallows `Ctrl+Shift+Alt+L`, which is
  bob-navigation-hotkeys' "Task card: Linked from" command in Obsidian.
- How to restore it: rebind it in Amethyst Settings → Shortcuts.

Behavior:

1. `domain=com.amethyst.Amethyst`, `key=KeyboardShortcuts_swap-screen-cw`,
   `app=Amethyst`.
2. Read the binding: `value="$(defaults read "$domain" "$key" 2>/dev/null)"`. If the
   read fails (Amethyst is not installed, or the shortcut is already cleared), print
   `Amethyst swap-screen-cw is unbound; Ctrl+Option+Shift+L is free.` and exit 0 without
   touching anything.
3. If the value is not exactly key code 37 with modifiers 6656, print
   `Amethyst swap-screen-cw is bound to another chord; leaving it alone.` and exit 0.
   Match each field independently with anchored patterns such as
   `"carbonKeyCode":37[,}]` and `"carbonModifiers":6656[,}]`, so that key order does not
   matter and 370 or 66560 cannot match. This keeps a later deliberate rebind by Bryan
   from being clobbered.
4. If Amethyst is running (`pgrep -x Amethyst`), quit it with `pkill -x Amethyst`. Wait
   up to 5 s, then use `pkill -KILL -x Amethyst` and wait up to 2 s more, using the
   Hammerspoon hook's `wait_for_exit` pattern. If it still runs, print an error to
   stderr
   (`Amethyst did not exit; clear "Swap focused window to clockwise screen" in Amethyst Settings → Shortcuts instead.`)
   and exit 1 **without** writing. Remember whether it was running.
5. Run `defaults delete "$domain" "$key"`. On failure, print to stderr, relaunch
   Amethyst if it was running, and exit 1.
6. If Amethyst was running, relaunch it with `open -g -a Amethyst`. On success, print
   `Cleared Amethyst's Ctrl+Option+Shift+L (swap-screen-cw); Amethyst restarted.` On
   failure, print `... could not be relaunched; start it manually.` to stderr and
   exit 1. If it was not running, print
   `Cleared Amethyst's Ctrl+Option+Shift+L (swap-screen-cw); it applies on Amethyst's next launch.`

The hook is idempotent. Once the key is gone, every later run takes step 2 and does
nothing.

### 2. Regression test

Add `tests/bash/amethyst_swap_screen_cw_hook_test.sh`, a bashunit test modelled on
`tests/bash/hammerspoon_restart_hook_test.sh`. Reuse its structure: the `set_up` and
`tear_down` temp dirs, `require_chezmoi`, the `render_darwin_hook` sed substitution of
`eq .chezmoi.os "darwin"` → `true` piped to
`chezmoi --config <empty> --source <repo> execute-template`, the PATH-stubbed
`run_hook`, and the no-op `sleep` stub.

Add a stubbed `defaults` that reads and deletes a value held in a state file and records
every call. Add `pgrep`, `pkill`, and `open` stubs like the Hammerspoon ones, with the
process name `Amethyst`. Cover these cases:

- The template renders empty on Linux (skip on Darwin).
- The rendered darwin hook passes `bash -n`.
- Key absent: exit 0, prints "unbound", no `defaults delete`, no `pkill`, no `open`.
- Key bound to another chord (for example key code 4 with 6656, or key code 37 with
  2560): exit 0, prints "another chord", no delete, no `pkill`.
- Key bound to 37/6656, Amethyst not running: the key is deleted, there is no `pkill`
  and no `open`, and the output says "next launch".
- Key bound to 37/6656, Amethyst running: the calls are ordered `pkill -x Amethyst` →
  `defaults delete com.amethyst.Amethyst KeyboardShortcuts_swap-screen-cw` →
  `open -g -a Amethyst`, and the output says "restarted".
- Amethyst survives both TERM and KILL: exit 1, a stderr message, and **no**
  `defaults delete`.
- `open` fails after the delete: exit 1 with the "start it manually" message.
- A second run after a successful run is a no-op ("unbound").

### 3. Apply on the MacBook

> [!decision] apply_now After the tests pass, run the committed hook body on the Mac.
> Render it on the host with the same substitution the test uses and pipe it over
> Tailscale SSH:
> `sed 's/eq \.chezmoi\.os "darwin"/true/' home/.chezmoiscripts/run_onchange_after_disable_amethyst_swap_screen_cw.tmpl | chezmoi execute-template | ssh -o ConnectTimeout=8 mac bash`.
> Then confirm with
> `ssh mac 'defaults read com.amethyst.Amethyst KeyboardShortcuts_swap-screen-cw'`,
> which must now fail because the key is absent, and `ssh mac 'pgrep -x Amethyst'`,
> which must succeed. The Mac is offline unless its lid is open. If SSH times out, say
> so in the final report and leave the change to the Mac's next `chezmoi update`. Do not
> retry in a loop. When chezmoi later runs this `run_onchange` hook on the Mac, it finds
> the key absent and does nothing.

> [!decision] apply_now = no Do not touch the Mac. In the final report, tell Bryan to
> run `chezmoi update -a --force` on the MacBook, which quits and relaunches Amethyst
> once.

## Verification

- `bashunit ./tests/bash/amethyst_swap_screen_cw_hook_test.sh`, then `just test-bash`
  and `just lint` in the chezmoi repo (or `just check` if the toolchain is available).
  All must pass.
- `chezmoi update -a --force` on athena (required by chezmoi's `AGENTS.md`). The hook
  renders empty on Linux and must not run.
- Manual check for Bryan (state it in the final report): in Obsidian on the MacBook, put
  the cursor on a task and press `Ctrl+Shift+Option+L`. The Task Card opens on
  `🪢 Linked from · <task>`. Pick another task and press Enter, and that task's 🪢 LINKS
  line gains a link to the current task.

## Non-goals

- No bob-plugins change. The `task-card-linked-from` command, its default hotkey, docs
  (`docs/task-relations.md`, `docs/task-card.md`), and tests already exist. Do not add a
  second command or remap the chord.
- Do not change other Amethyst shortcuts. The paired `swap-screen-ccw` (`mod2 + h`) and
  everything else stay as they are.
- A counted `N<Ctrl+Shift+Alt+L>` stays refused by design ("Count works with Links to
  (Ctrl+Shift+L)").
