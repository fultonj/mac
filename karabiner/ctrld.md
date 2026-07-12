# Ctrl+D and Ghostty surfaces

When `ctrl+d` is pressed, I like it when:

- It deletes characters in front of text on the terminal (if any)
- It closes the termainl if there are no characters in front of text

By default I had first behavior but not the second.

I got the second behavior working as documented below, but it then
broke the first behavior and `ctrl+d` was suddenly closing the whole
window instead of deleing characters. Thus, I reverted it as per the
current goal below but I'm documenting all the fun I had setting it
up in case I return and try to get both behaviors above working.

## Current goal (2026-07-12, later)

`ctrl+d` should not close the current Ghostty surface
(window/tab/split) for now. This documents the reversal of the earlier
goal below.

## Status: Karabiner change reverted (2026-07-12)
The earlier fix (adding Ghostty to Karabiner's emacs exclusion lists) was undone.
Karabiner again intercepts `ctrl+d` → forward-delete *before* Ghostty sees it, so
Ghostty's `ctrl+d=close_surface` keybind never fires and `ctrl+d` no longer closes
the window.

- Removed all 14 `"^com\.mitchellh\.ghostty$"` lines from the
  `frontmost_application_unless` exclusion sets in
  `~/.config/karabiner/karabiner.json` (undoing the fix documented below).
- Verified: JSON still valid (`python3 -m json.tool`), 0 ghostty lines remain.
- Karabiner reloads on save automatically — no restart needed.
- Ghostty config (`~/.config/ghostty/config`) was intentionally left untouched;
  it still contains `keybind = ctrl+d=close_surface`, but that binding is now
  inert because Ghostty never receives the `ctrl+d` event.
- Backup of the pre-revert file:
  `~/.config/karabiner/karabiner.json.bak.undo-20260712-112234`.

### Behavior now
- `ctrl+d` in Ghostty is rewritten to forward-delete by Karabiner (`ESC[3~`).
- At an empty shell prompt, readline runs `delete-char` with nothing to delete
  and rings the bell (no EOF, no window close). This is the original pre-fix
  behavior described in the diagnosis below.
- To re-enable `ctrl+d`-closes-surface later, redo the fix: re-add
  `^com\.mitchellh\.ghostty$` to the exclusion sets (see "The fix" below).

---

# Historical: earlier work that MADE ctrl+d close the surface

## Original goal
`ctrl+d` should close the current Ghostty surface (window/tab/split) immediately,
both outside tmux and inside tmux, with no confirmation dialog.

## Status: FIXED (2026-07-12) — later reversed, see top of file
Root cause was **Karabiner-Elements**, not Ghostty. See below.

## Root cause
Karabiner-Elements had an "Emacs key bindings [control+keys]" rule that rewrites
**`ctrl+d` → forward-delete** in every app *except* a hardcoded allow-list of
terminals (Terminal, iTerm2, Alacritty, kitty, VSCode, …). **Ghostty's bundle id
`com.mitchellh.ghostty` was NOT in that list.**

So in Ghostty, Karabiner turned `ctrl+d` into forward-delete (`ESC[3~`) at the
driver level, *before* Ghostty ever saw the key event. Ghostty therefore never
received a `ctrl+d` trigger, and its keybind matching never ran.

The "bell icon" we saw was misread in the original notes: it was NOT the raw
0x04/BEL byte being forwarded. It was `ESC[3~` (forward-delete) reaching
readline, which runs `delete-char`; with nothing to delete at an empty prompt,
readline rings the bell. Same visible symptom, different cause.

This explains everything we were stuck on:
- Happened identically in and out of tmux (interception is upstream of Ghostty).
- No Ghostty config change ever helped (`close_surface`, `confirm-close-surface`,
  `physical:d+ctrl`, `all:`, full restarts) — Ghostty never got the event.
- `ctrl+j` worked fine because Karabiner has no emacs rule for J.

## How it was diagnosed
1. Confirmed `ctrl+d=close_surface` WAS registered at runtime
   (`ghostty +list-keybinds`, `ghostty +show-config`). Config was never the problem.
2. Ghostty 1.3.1 release build logs no key events even at `--log-level=debug`,
   so logs couldn't show the decision.
3. Behavioral test with throwaway instances (CLI `--keybind` overrides, so the
   real config was untouched):
   - `ctrl+d=text:CAUGHT` → pressing `ctrl+d` produced nothing/bell → trigger not matching.
   - `f9=text:...` → worked → `text:` binds work.
   - `physical:d+ctrl=text:...` → still failed → not a logical-vs-physical issue.
   - `ctrl+j=text:...` → worked → Ctrl+letter binds work in general.
   Conclusion: `ctrl+d` specifically was being swallowed *before* Ghostty.
4. Checked system remappers → Karabiner-Elements running. Found the
   control+keys emacs rule mapping `d` → `delete_forward` with a
   `frontmost_application_unless` terminal allow-list missing Ghostty.

## The fix (now reverted)
Added `^com\.mitchellh\.ghostty$` to the terminal-exclusion list in all **14**
`frontmost_application_unless` condition sets across the three emacs rules in
`~/.config/karabiner/karabiner.json`:
- `Bash style Emacs key bindings (rev 2)`
- `Emacs key bindings [control+keys] (rev 10)`
- `Emacs key bindings [option+keys] (rev 5)`

Ghostty then behaved like the other terminals in that config: all emacs remaps
(Ctrl+A/E/K/D…, Option+D, bash-style binds) passed through raw, so Ghostty's own
keybinds and normal terminal control chars worked.

Karabiner watches `karabiner.json` and reloaded automatically on save — no
Ghostty or Karabiner restart was needed.

Backup from the original fix: `~/.config/karabiner/karabiner.json.bak.20260712-105052`.

## Notes
- Ghostty config (`config`) has the correct lines and needed no change:
  ```
  quit-after-last-window-closed = true
  confirm-close-surface = false
  keybind = ctrl+d=close_surface
  ```
- If `ctrl+d` behavior ever regresses/changes unexpectedly, first suspect Karabiner
  (or any HID remapper), not Ghostty: check whether `com.mitchellh.ghostty` is in
  the emacs-rule exclusion lists.

## Files touched
- `~/.config/karabiner/karabiner.json` — Ghostty added to emacs-rule exclusions
  (the fix), then removed again (the revert). Backups alongside it:
  `.bak.20260712-105052` (pre-fix) and `.bak.undo-20260712-112234` (pre-revert).
- `~/.config/ghostty/config` — has `confirm-close-surface = false` and
  `keybind = ctrl+d=close_surface`; unchanged by both the fix and the revert.
- `~/.inputrc` — not relevant (only a `kill-whole-line` binding).
