<div align="center">

# 💠 end4-pC × DankMaterialShell

**Run [end4-pC](https://github.com/pctrade/end4-pC) next to DankMaterialShell (DMS) on Hyprland and switch between them with one command.**

![Hyprland](https://img.shields.io/badge/Hyprland-Lua%20config-58E1FF?style=flat-square)
![Quickshell](https://img.shields.io/badge/Quickshell-qs-7AA2F7?style=flat-square)
![Shell](https://img.shields.io/badge/shell-fish-4AAE46?style=flat-square)
![Distro](https://img.shields.io/badge/CachyOS-Arch-1793D1?style=flat-square)

*Setup notes, working scripts, and a debugging log, collected while getting this running.*

</div>

---

## 📑 Contents

1. [How it fits together](#-how-it-fits-together)
2. [Quick reference](#-quick-reference)
3. [Setup from scratch](#-setup-from-scratch)
4. [Scripts](#-scripts)
5. [Hyprland files](#-hyprland-files)
6. [Surviving system updates](#-surviving-system-updates)
7. [Debugging cookbook](#-debugging-cookbook)
8. [Rollback and emergency exits](#-rollback-and-emergency-exits)
9. [Presets and settings](#-presets-and-settings)
10. [Known limits](#-known-limits)

---

## 🧭 How it fits together

Two Quickshell shells, one Hyprland. Only one shell runs at a time, and a small overlay decides which Hyprland look is loaded.

```
                      shell-switch dms|end4
                               │
        ┌──────────────────────┴──────────────────────┐
        ▼                                             ▼
  dms.service (systemd --user)                 qs -c end4-pC  (the fork)
  DankMaterialShell                            needs illogical-impulse (ii) deps
        │                                             │
        └──── Hyprland ◄── ~/.config/hypr/hyprland.lua ──────┐
                    │                                        │
          require("dms.*")  always             pcall(require,"end4.mode")  overlay:
          (colors, layout, binds, ...)         mode file = dms  → nothing extra
                                               mode file = end4 → fork's look on top
```

| Piece | Where | Role |
|---|---|---|
| Fork | `~/.config/quickshell/end4-pC` | The shell itself, started with `qs -c end4-pC` |
| ii dependencies | installed by end-4's `./setup` | Python venv, scripts, `~/.config/quickshell/ii` that the fork expects |
| DMS | `dms.service` (user unit) | Your existing shell, kept intact |
| Overlay | `~/.config/hypr/end4/` | Fork keybinds + the dms/end4 mode switch for the Hyprland look |
| Theme generation | `~/.config/matugen` | Writes `hyprland/colors.lua` and friends when you pick a wallpaper |

**Key facts that explain most problems**

- The fork **requires illogical-impulse (ii) to be installed**. Cloning it alone is not enough.
- DMS runs as a **systemd user service**, so `dms kill` is undone by systemd. Use `systemctl --user stop dms`.
- The fork's launcher/sidebars open through **Hyprland global shortcuts** (`quickshell:*`). No bind, no menu.
- Window gaps, borders and colors come from **Hyprland**, not from the shell. That is why the overlay exists.

---

## ⚡ Quick reference

| I want to… | Command |
|---|---|
| Switch to the fork | `shell-switch end4` |
| Switch back to DMS | `shell-switch dms` |
| See what is running | `shell-switch status` |
| Save the working state | `end4-repair snapshot` |
| Diagnose the setup | `end4-repair check` |
| Repair after an update | `end4-repair restore` |
| Run the fork in the foreground (see errors) | `shell-switch dms; qs -c end4-pC` |
| List registered fork shortcuts | `hyprctl globalshortcuts \| grep quickshell` |
| Fire a shortcut by hand | `hyprctl dispatch 'hl.dsp.global("quickshell:searchToggle")'` |
| See which Hyprland modules loaded | `cat ~/.cache/end4-mode.log` |
| Update system with a safety snapshot | `upd` |

Default fork keys (from `end4/binds.lua`): `Super` tap = launcher, `Super+Tab` overview, `Super+A` left sidebar, `Super+N` right sidebar, `Super+Esc` settings, `Ctrl+Super+T` wallpaper selector.

---

## 🛠 Setup from scratch

> Commands are for **fish**. Fish has no heredocs, so multi-line files are created with an editor.

### 0. Back up

```fish
cp -r ~/.config/hypr ~/.config/hypr.bak
mkdir -p ~/.config/quickshell
command ls ~/.config | grep -i -E "dank|quickshell"
cp -r ~/.config/DankMaterialShell ~/.config/DankMaterialShell.bak 2>/dev/null
```

### 1. Install the ii dependencies without touching your Hyprland config

```fish
git clone https://github.com/end-4/dots-hyprland ~/dots-hyprland
cd ~/dots-hyprland
./setup install --help
./setup install --core --skip-hyprland --skip-hyprland-entry
```

- `--core` skips plasma integration, fish, misc configs and fontconfig.
- `--skip-hyprland` and `--skip-hyprland-entry` keep ii away from `~/.config/hypr`.
- Answer **Enter** at the greeting prompts and **y** to "backup clashing files" (goes to `~/ii-original-dots-backup`).
- Read any package replace/conflict prompt before accepting, especially for Quickshell.

Verify nothing of yours changed (only expect new folders):

```fish
diff -rq ~/.config/hypr ~/.config/hypr.bak
```

### 2. Clone the fork

```fish
cd ~/.config/quickshell/
git clone https://github.com/pctrade/end4-pC.git
```

### 3. Matugen config (needed because `--core` skips it)

```fish
grep -nE 'output_path|^\[templates' ~/dots-hyprland/dots/.config/matugen/config.toml   # review what it writes
for f in ~/.config/gtk-3.0/gtk.css ~/.config/gtk-4.0/gtk.css ~/.config/fuzzel/fuzzel_theme.ini
    test -f $f; and cp $f $f.pre-end4
end
cp -r ~/dots-hyprland/dots/.config/matugen ~/.config/matugen
```

The templates write GTK 3/4 CSS, the fuzzel theme and Hyprland colors, so the backups above matter.

### 4. Scripts

Create `shell-switch`, `use-dms`, `use-end4`, `end4-repair` from the [Scripts](#-scripts) section:

```fish
mkdir -p ~/.local/bin
fish_add_path ~/.local/bin
# paste each script into ~/.local/bin/<name>, then:
chmod +x ~/.local/bin/shell-switch ~/.local/bin/use-dms ~/.local/bin/use-end4 ~/.local/bin/end4-repair
```

### 5. Hyprland overlay and binds

Create `~/.config/hypr/end4/binds.lua`, `~/.config/hypr/end4/mode.lua` and the mode file from [Hyprland files](#-hyprland-files), then append the loader lines:

```fish
echo dms > ~/.config/hypr/end4/mode
printf '%s\n' 'pcall(require, "end4.binds")' 'pcall(require, "end4.mode")' >> ~/.config/hypr/hyprland.lua
hyprctl reload
```

### 6. First run and theme generation

```fish
shell-switch end4
```

Press `Ctrl+Super+T`, pick a wallpaper. Then:

```fish
command ls -l ~/.config/hypr/hyprland/colors.lua ~/.local/state/quickshell/user/generated/
hyprctl reload
head -8 ~/.cache/end4-mode.log      # expect "hyprland.colors: ok"
```

### 7. Snapshot the working state

```fish
end4-repair snapshot
end4-repair check
```

---

## 📜 Scripts

### `shell-switch`: swap shells (and the Hyprland look)

Stops one shell, starts the other, flips the mode file and reloads Hyprland. Exports the venv path the fork's color scripts need.

<details>
<summary><code>~/.local/bin/shell-switch</code></summary>

```bash
#!/usr/bin/env bash
# shell-switch: dms | end4 | status
FORK=end4-pC
export ILLOGICAL_IMPULSE_VIRTUAL_ENV="$HOME/.local/state/quickshell/.venv"
MODE="$HOME/.config/hypr/end4/mode"

stop_fork() {
  pkill -f "^qs .*$FORK" 2>/dev/null
  for i in 1 2 3 4 5 6 7 8 9 10; do
    pgrep -f "^qs .*$FORK" > /dev/null || return 0
    sleep 0.5
  done
  pkill -9 -f "^qs .*$FORK" 2>/dev/null
}

case "${1:-status}" in
  end4)
    echo end4 > "$MODE"
    systemctl --user stop dms
    stop_fork
    nohup qs -c "$FORK" > /dev/null 2>&1 &
    disown
    hyprctl reload > /dev/null
    echo "-> end4-pC"
    ;;
  dms)
    stop_fork
    echo dms > "$MODE"
    hyprctl reload > /dev/null
    systemctl --user start dms
    echo "-> DMS"
    ;;
  status)
    echo "mode file:   $(cat "$MODE" 2>/dev/null || echo dms)"
    echo "dms.service: $(systemctl --user is-active dms)"
    if pgrep -f "^qs -c $FORK" > /dev/null; then echo "end4-pC:     active"; else echo "end4-pC:     inactive"; fi
    ;;
  *)
    echo "usage: shell-switch dms|end4|status"
    exit 1
    ;;
esac
```

</details>

**Why it works this way**

- `systemctl --user stop dms`, not `dms kill`: systemd restarts a killed DMS within seconds.
- `pkill -f "^qs .*end4-pC"` only matches `qs` processes, so an editor with a file from that folder is not killed. It also catches the first-run `welcome.qml` process.
- The wait loop lets the fork exit before DMS starts, so they do not fight over the notification daemon.

**Debug**

```fish
shell-switch status
pgrep -af 'dms|quickshell|qs '
systemctl --user status dms | head -5
```

Exactly one of `dms.service: active` / `end4-pC: active` should be true.

### Wrappers

```bash
# ~/.local/bin/use-end4
#!/usr/bin/env bash
exec shell-switch end4
```

```bash
# ~/.local/bin/use-dms
#!/usr/bin/env bash
exec shell-switch dms
```

### `end4-repair`: snapshot, check, restore

`snapshot` saves the setup files, matugen config, the fork's `config.json`, git commits and package versions to `~/.local/share/end4-setup/<timestamp>` (and a `latest` link). `check` reports what is broken. `restore` puts things back, re-adds the loader lines to `hyprland.lua`, rebuilds the Python venv if needed, and reloads Hyprland. Files that differ from the snapshot are kept as `.bak-<time>`.

<details>
<summary><code>~/.local/bin/end4-repair</code></summary>

```bash
#!/usr/bin/env bash
# end4-repair: snapshot | check | restore [--deps] | list
SNAP_ROOT="$HOME/.local/share/end4-setup"
HYPR="$HOME/.config/hypr"
FORK_DIR="$HOME/.config/quickshell/end4-pC"
FORK_URL="https://github.com/pctrade/end4-pC.git"
DOTS="$HOME/dots-hyprland"
VENV="$HOME/.local/state/quickshell/.venv"
BIN="$HOME/.local/bin"

# paths relative to $HOME that make up the setup
ITEMS=(
  ".config/hypr/end4"
  ".config/matugen"
  ".config/illogical-impulse"
  ".local/bin/shell-switch"
  ".local/bin/use-dms"
  ".local/bin/use-end4"
)

ok()  { printf '  [ok]   %s\n' "$1"; }
bad() { printf '  [FAIL] %s\n' "$1"; FAILS=$((FAILS + 1)); }

snapshot() {
  local d="$SNAP_ROOT/$(date +%Y%m%d-%H%M%S)" f
  mkdir -p "$d/files"
  for f in "${ITEMS[@]}"; do
    [ -e "$HOME/$f" ] && (cd "$HOME" && cp -a --parents "$f" "$d/files/")
  done
  git -C "$FORK_DIR" rev-parse HEAD > "$d/fork-commit.txt" 2>/dev/null
  git -C "$DOTS" rev-parse HEAD > "$d/dots-commit.txt" 2>/dev/null
  pacman -Q 2>/dev/null | grep -E '^(quickshell|hyprland|matugen|qt6-base|dms|dankmaterial)|^python ' > "$d/packages.txt"
  ln -sfn "$d" "$SNAP_ROOT/latest"
  echo "snapshot saved: $d"
}

check() {
  FAILS=0
  echo "end4-pC setup check"
  command -v qs > /dev/null && ok "quickshell (qs)" || bad "qs not installed"
  command -v matugen > /dev/null && ok "matugen" || bad "matugen not installed"
  [ -f "$FORK_DIR/shell.qml" ] && ok "fork present" || bad "fork missing: $FORK_DIR"
  [ -f "$HOME/.config/quickshell/ii/shell.qml" ] && ok "ii config present" || bad "ii config missing"
  [ -f "$HOME/.config/matugen/config.toml" ] && ok "matugen config" || bad "matugen config missing"
  "$VENV/bin/python" -c "import materialyoucolor" 2> /dev/null && ok "venv + materialyoucolor" || bad "python venv broken or missing"
  for f in .config/hypr/end4/binds.lua .config/hypr/end4/mode.lua .config/hypr/end4/mode .local/bin/shell-switch; do
    [ -e "$HOME/$f" ] || bad "missing ~/$f"
  done
  grep -q "end4.binds" "$HYPR/hyprland.lua" && ok "binds loader in hyprland.lua" || bad "end4.binds not loaded by hyprland.lua"
  grep -q "end4.mode" "$HYPR/hyprland.lua" && ok "mode loader in hyprland.lua" || bad "end4.mode not loaded by hyprland.lua"
  systemctl --user cat dms > /dev/null 2>&1 && ok "dms.service" || bad "dms.service missing"
  [ -f "$HYPR/hyprland/colors.lua" ] && ok "generated hypr colors" || bad "hyprland/colors.lua missing (pick a wallpaper in the fork)"
  echo
  if [ "$FAILS" -eq 0 ]; then echo "all good"; else echo "$FAILS problem(s). Try: end4-repair restore"; fi
  return "$FAILS"
}

restore() {
  local snap="$SNAP_ROOT/latest" f c req m
  [ -d "$snap/files" ] || { echo "no snapshot yet. Run: end4-repair snapshot"; return 1; }
  echo "restoring from $(readlink -f "$snap")"

  if [ "${1:-}" = "--deps" ]; then
    [ -d "$DOTS/.git" ] || git clone https://github.com/end-4/dots-hyprland "$DOTS"
    (cd "$DOTS" && ./setup install --core --skip-hyprland --skip-hyprland-entry)
  fi

  if [ ! -f "$FORK_DIR/shell.qml" ]; then
    mkdir -p "$HOME/.config/quickshell"
    git clone "$FORK_URL" "$FORK_DIR"
    c=$(cat "$snap/fork-commit.txt" 2> /dev/null)
    [ -n "$c" ] && git -C "$FORK_DIR" checkout -q "$c"
  fi

  # copy snapshot files back; anything that differs is kept as .bak-<time>
  while IFS= read -r -d '' f; do
    f="${f#./}"
    mkdir -p "$(dirname "$HOME/$f")"
    if [ -e "$HOME/$f" ] && ! cmp -s "$snap/files/$f" "$HOME/$f"; then
      cp -a "$HOME/$f" "$HOME/$f.bak-$(date +%s)"
    fi
    cp -a "$snap/files/$f" "$HOME/$f"
  done < <(cd "$snap/files" && find . -type f -print0)
  chmod +x "$BIN/shell-switch" "$BIN/use-dms" "$BIN/use-end4" 2> /dev/null

  if [ ! -f "$HOME/.config/matugen/config.toml" ] && [ -d "$DOTS/dots/.config/matugen" ]; then
    cp -r "$DOTS/dots/.config/matugen" "$HOME/.config/matugen"
  fi

  # make sure hyprland.lua loads the overlay (DMS may rewrite that file)
  [ -n "$(tail -c1 "$HYPR/hyprland.lua")" ] && echo >> "$HYPR/hyprland.lua"
  for m in end4.binds end4.mode; do
    grep -q "$m" "$HYPR/hyprland.lua" || echo "pcall(require, \"$m\")" >> "$HYPR/hyprland.lua"
  done

  if ! "$VENV/bin/python" -c "import materialyoucolor" 2> /dev/null; then
    req=$(find "$DOTS" -name "requirements.txt" -path "*uv*" 2> /dev/null | head -1)
    [ -z "$req" ] && req=$(find "$DOTS" -name "requirements*.txt" 2> /dev/null | head -1)
    if [ -n "$req" ]; then
      echo "rebuilding python venv from $req"
      python -m venv --clear "$VENV" && "$VENV/bin/pip" install -r "$req"
    else
      echo "no requirements file found. Run: end4-repair restore --deps"
    fi
  fi

  hyprctl reload > /dev/null 2>&1
  echo
  check
}

case "${1:-check}" in
  snapshot) snapshot ;;
  check) check ;;
  restore) shift; restore "$@" ;;
  list) ls -1 "$SNAP_ROOT" 2> /dev/null ;;
  *) echo "usage: end4-repair snapshot | check | restore [--deps] | list"; exit 1 ;;
esac
```

</details>

`restore --deps` also reruns end-4's installer (interactive, asks for sudo). Re-run `end4-repair snapshot` after any deliberate change, otherwise `restore` brings back the older state.

### `upd`: update with a safety net (fish function)

```fish
function upd
    end4-repair snapshot; and sudo pacman -Syu $argv; and end4-repair check
end
funcsave upd
```

Swap `pacman` for `paru` / `yay` if that is what you use.

---

## 🧩 Hyprland files

### `~/.config/hypr/end4/binds.lua`: the fork's keybinds

The fork registers ~30 global shortcuts but Hyprland has no binds for them unless you add them. DMS's own binds stay in place (they call `dms ipc …` and do nothing while DMS is stopped).

```lua
-- end4-pC binds. Global shortcuts are no-ops when the fork is not running.
hl.bind("SUPER + SUPER_L", hl.dsp.global("quickshell:searchToggleRelease"), { description = "Shell: Toggle search" })
hl.bind("SUPER + Tab", hl.dsp.global("quickshell:overviewWorkspacesToggle"), { description = "Shell: Toggle overview" })
hl.bind("SUPER + A", hl.dsp.global("quickshell:sidebarLeftToggle"), { description = "Shell: Toggle left sidebar" })
hl.bind("SUPER + N", hl.dsp.global("quickshell:sidebarRightToggle"), { description = "Shell: Toggle right sidebar" })
hl.bind("SUPER + V", hl.dsp.global("quickshell:overviewClipboardToggle"))
hl.bind("SUPER + Period", hl.dsp.global("quickshell:overviewEmojiToggle"))
hl.bind("SUPER + Slash", hl.dsp.global("quickshell:cheatsheetToggle"), { description = "Shell: Toggle cheatsheet" })
hl.bind("SUPER + G", hl.dsp.global("quickshell:overlayToggle"), { description = "Shell: Toggle widget overlay" })
hl.bind("SUPER + escape", hl.dsp.global("quickshell:settingsToggle"), { description = "Toggle settings" })
hl.bind("CTRL + ALT + Delete", hl.dsp.global("quickshell:sessionToggle"), { description = "Shell: Toggle session menu" })
hl.bind("CTRL + SUPER + T", hl.dsp.global("quickshell:wallpaperSelectorToggle"), { description = "Shell: Wallpaper selector" })
```

Left out on purpose: `SUPER + J/K` (clash with DMS focus moves), `SUPER + SHIFT + S` (DMS screenshot). More shortcut names are in `~/dots-hyprland/dots/.config/hypr/hyprland/keybinds.lua`.

### `~/.config/hypr/end4/mode.lua`: the look overlay

Reads `~/.config/hypr/end4/mode`. In `dms` mode it does nothing. In `end4` mode it loads the ii look modules on top of DMS's. Every module is `pcall`-wrapped, and the result is logged to `~/.cache/end4-mode.log`.

```lua
-- end4-pC look overlay. Mode file: ~/.config/hypr/end4/mode (dms | end4)
local home = os.getenv("HOME")
local f = io.open(home .. "/.config/hypr/end4/mode", "r")
local mode = "dms"
if f then
  mode = f:read("*l") or "dms"
  f:close()
end

if mode == "end4" then
  HOME = HOME or home
  is_file_exists = is_file_exists or function(p)
    local h = io.open(p, "r")
    if h then h:close() return true end
    return false
  end
  local log = io.open(home .. "/.cache/end4-mode.log", "w")
  for _, m in ipairs({
    "hyprland.general",
    "hyprland.decoration",
    "hyprland.animations",
    "hyprland.rules",
    "hyprland.colors",
    "hyprland.shellOverrides.main",
  }) do
    local ok, err = pcall(require, m)
    if log then log:write(m .. ": " .. (ok and "ok" or tostring(err)) .. "\n") end
  end
  if log then log:close() end
end
```

### Mode file and loader lines

```fish
echo dms > ~/.config/hypr/end4/mode          # "dms" or "end4" (shell-switch edits this)
```

Last lines of `~/.config/hypr/hyprland.lua` (after the DMS `require` lines):

```lua
pcall(require, "end4.binds")
pcall(require, "end4.mode")
```

`pcall` means a broken overlay cannot take down the rest of your config. Convert plain `require` lines with:

```fish
sed -i 's/^require("end4\.\(binds\|mode\)")$/pcall(require, "end4.\1")/' ~/.config/hypr/hyprland.lua
```

Why not use ii's own `hyprland.lua`? It requires `hyprland.lib`, `hyprland.services` and `hyprland.colors`, which are missing here, and its `execs.lua` starts a shell by itself, which would fight `shell-switch`.

---

## 🔄 Surviving system updates

Nobody can promise an update will not break a rolling-release setup. The goal is cheap detection and cheap repair.

| Likely break | Symptom | Fix |
|---|---|---|
| Python minor version bump | Wallpaper picks no longer theme; `materialyoucolor` import error | `end4-repair restore` (rebuilds venv) or `restore --deps` |
| DMS rewrites `hyprland.lua` | Fork keys and look overlay vanish | `end4-repair restore` re-adds the loader lines |
| Quickshell / Qt update | Fork errors or crashes on launch | `shell-switch dms`; downgrade the package from `/var/cache/pacman/pkg` (`sudo pacman -U <old>.pkg.tar.zst`) |
| Hyprland Lua API change | Reload error bar, binds missing | `pcall` isolates the overlay; if DMS's own files break, that is a DMS-side issue |
| Fork updates upstream | New QML needs newer ii pieces | Do not auto-pull. `git -C ~/.config/quickshell/end4-pC pull`, then `end4-repair check` |

Habits that help: use `upd`, do full upgrades (`-Syu`, never partial), and if your root is btrfs take a filesystem snapshot before big updates. The repair script covers configs, not installed packages.

---

## 🐛 Debugging cookbook

Everything below actually happened during setup.

### Install and basics

| Symptom | Cause | Fix |
|---|---|---|
| `cp: cannot stat '~/.config/quickshell'` | DMS is not a user-level quickshell config, so the folder does not exist yet | `mkdir -p ~/.config/quickshell` |
| Fork does nothing / errors about missing ii files | ii is not installed | Step 1 of setup |
| `ls -lat` → `invalid value … for '--time'` | `ls` is aliased to `eza` | `command ls …` |
| `fish: No matches for wildcard '--include=*.conf'` | Fish expands unquoted globs | Quote them: `--include='*.lua'` |
| `<< 'EOF'` fails in fish | Fish has no heredocs | `printf '%s\n' '…' > file`, or run `bash` and paste the heredoc, or use an editor |

### Both shells are running

| Symptom | Cause | Fix |
|---|---|---|
| DMS still shown after `dms kill` | systemd restarted `dms.service` | `systemctl --user stop dms` (that is what `shell-switch` does) |
| Fork and DMS bars overlap | Both running | `shell-switch status`, then switch again |
| Polkit warning in fork log: "authentication agent already exists" | Another agent is registered | Harmless; your existing agent handles prompts |

### Keys and menus

| Symptom | Cause | Fix |
|---|---|---|
| Super does not open the fork launcher | No Hyprland bind for `quickshell:*` shortcuts (ii's Hyprland config was skipped) | Add `end4/binds.lua`, then `hyprctl reload` |
| `hyprctl dispatch …searchToggleRelease` does nothing | That one fires on key *release* | Test with `quickshell:searchToggle` or `overviewWorkspacesToggle` |
| Unsure the fork registered its shortcuts | | `hyprctl globalshortcuts \| grep quickshell` |
| Launcher pops up after Super combos (e.g. `Super+T`) | Release-triggered shortcut also fires after combos | Look at `searchToggleReleaseInterrupt` in `~/dots-hyprland/dots/.config/hypr/hyprland/keybinds.lua` |

### Look and colors

| Symptom | Cause | Fix |
|---|---|---|
| Windows still have DMS gaps/borders under the fork | Hyprland loads `dms.*` modules, shell-independent | Overlay (`end4/mode.lua`) in `end4` mode |
| `hyprland.colors: module not found` in `~/.cache/end4-mode.log` | `colors.lua` is generated by matugen on wallpaper pick, and generation failed or never ran | Fix the two causes below, then pick a wallpaper |
| `material_colors.scss` is 0 bytes, no `colors.json` | Theme generation failed | Run it by hand: `bash -x ~/.config/quickshell/end4-pC/scripts/colors/switchwall.sh --image <real image> 2>&1 \| tail -40` |
| Cause 1: `~/.config/matugen` missing | `--core` skips misc configs | `cp -r ~/dots-hyprland/dots/.config/matugen ~/.config/matugen` |
| Cause 2: `ModuleNotFoundError: materialyoucolor`, `/bin/activate: No such file` | `ILLOGICAL_IMPULSE_VIRTUAL_ENV` is unset, so the venv is never activated | `export ILLOGICAL_IMPULSE_VIRTUAL_ENV=~/.local/state/quickshell/.venv` (already in `shell-switch`). Check: `~/.local/state/quickshell/.venv/bin/python -c 'import materialyoucolor; print("ok")'` |
| `kde-material-you-colors: command not found` | Optional Qt-app theming tool not installed | Ignore, colors still work |

Remember to use a **real image path**; a placeholder path fails differently.

### Log noise that is safe to ignore

First-run file warnings (`config.json`, `states.json`, `todo.json`, `.face`, thumbnails), `Could not load icon "image-missing"`, `Cannot assign to read-only property "mirrored"` in the bar, `QWaylandTextInputv3::disableSurface`, GeoClue `NoReply`. These are cosmetic. If something visibly breaks, run `shell-switch dms; qs -c end4-pC` and read the first `ERROR` lines.

### Diagnostics toolbox

```fish
shell-switch status
end4-repair check
pgrep -af 'dms|quickshell|qs '
systemctl --user status dms | head -5
hyprctl globalshortcuts | grep quickshell
head -12 ~/.cache/end4-mode.log
command ls -l ~/.local/state/quickshell/user/generated/ ~/.config/hypr/hyprland/colors.lua
shell-switch dms; qs -c end4-pC        # foreground run, errors visible
```

---

## 🚑 Rollback and emergency exits

**Back to DMS (normal):**

```fish
shell-switch dms
```

**Overlay misbehaving, shell commands unavailable:**

```fish
echo dms > ~/.config/hypr/end4/mode
hyprctl reload
```

**Hyprland will not start properly:** switch to a TTY (`Ctrl+Alt+F3`), then:

```bash
echo dms > ~/.config/hypr/end4/mode
sed -i '/end4\.\(binds\|mode\)/d' ~/.config/hypr/hyprland.lua
# or restore everything: rm -r ~/.config/hypr && cp -r ~/.config/hypr.bak ~/.config/hypr
```

**Undo the GTK/fuzzel theming the fork wrote:**

```fish
for f in ~/.config/gtk-3.0/gtk.css ~/.config/gtk-4.0/gtk.css ~/.config/fuzzel/fuzzel_theme.ini
    test -f $f.pre-end4; and cp $f.pre-end4 $f
end
```

**Make the fork the default (optional)**, instead of DMS:

```fish
systemctl --user disable dms
# autostart in Hyprland:  hl.exec_cmd("shell-switch end4")
```

Reverse with `systemctl --user enable dms`.

**Full removal:** `shell-switch dms`, delete `~/.config/hypr/end4`, the two loader lines, `~/.config/quickshell/end4-pC`, and optionally `~/.config/matugen`.

---

## 🎛 Presets and settings

The Linux fork's README does not describe a preset feature. What exists:

- **Settings panel:** `Super+Esc` (needs the bind). Changes are written to `~/.config/illogical-impulse/config.json`.
- **Panel family:** `Ctrl+Super+P` cycles panel families (`quickshell:panelFamilyCycle`).
- **Wallpaper → theme:** `Ctrl+Super+T`; matugen derives the palette from the wallpaper.

To check whether your checkout has a preset feature:

```fish
grep -rli preset ~/.config/quickshell/end4-pC --include='*.qml' --include='*.json' --include='*.sh' | head
```

A simple home-made preset is a saved copy of `config.json`:

```fish
function end4-preset
    set dir ~/.local/share/end4-setup/presets
    switch $argv[1]
        case save
            mkdir -p $dir/$argv[2]; and cp ~/.config/illogical-impulse/config.json $dir/$argv[2]/
        case load
            cp $dir/$argv[2]/config.json ~/.config/illogical-impulse/config.json
        case list
            command ls $dir
    end
end
funcsave end4-preset
```

Usage: `end4-preset save night`, `end4-preset load night`, `end4-preset list`. The shell watches `config.json`, so changes usually apply live; if not, `shell-switch end4`. Colors come from the wallpaper, so re-pick it to regenerate them. `end4-repair snapshot` already includes `config.json`.

---

## ⚠ Known limits

- Tested on CachyOS (Arch), Hyprland with the **Lua** config, fish, Quickshell, DMS as a systemd user unit, on 2026-10-24. The `.conf` syntax differs.
- The overlay loads the fork's look but not ii's own keybinds or `execs`. That is deliberate.
- The venv rebuild inside `end4-repair restore` is a best effort (it looks for a `requirements.txt` in the dots repo). If it fails, use `restore --deps`.
- The repair script was dry-run in a scratch home directory, not against a real broken system.
- The fork is third-party and tracks upstream ii. Read what a Quickshell config executes before running it, and pin versions you trust.

---

<div align="center">

*Built on [end-4/dots-hyprland](https://github.com/end-4/dots-hyprland), [pctrade/end4-pC](https://github.com/pctrade/end4-pC) and [DankMaterialShell](https://github.com/AvengeMedia/DankMaterialShell).*

</div>
