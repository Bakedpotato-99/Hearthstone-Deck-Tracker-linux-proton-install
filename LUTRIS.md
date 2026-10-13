# Hearthstone Deck Tracker on Linux via **Lutris** (Wine 11)

If you run Hearthstone through **Steam Proton**, use the [main guide](README.md) instead.

This is the **Lutris/Battle.net variant** of the same job, contributed by [@iampossiblyatwork](https://github.com/iampossiblyatwork). It assumes you've skimmed the main guide and the [technical deep dive](technical-deep-dive-HDT-Linux.md); it focuses on what's different under Lutris, plus a few things not written down elsewhere.

**Verified on one machine:** Linux Mint 22.3, Lutris 0.5.14, Wine 11.11-staging (Kron4ek, traditional multilib), Hearthstone installed via the Lutris Battle.net installer, HDT 1.57.12 installed September 2026, since self-updated through 1.58.x to 1.59.0 (October 2026) via delta packages. Older Wine will behave differently.

**Background:** HDT is a .NET Framework 4.7.2 **WPF** app. Wine Mono doesn't implement WPF at all, so HDT can't run until genuine Microsoft .NET is in the prefix. That much is well covered everywhere.

---

## 0. Back up the prefix first — `dotnet48` cannot be uninstalled

winetricks has no rollback. If you're installing into a Battle.net prefix with a working 20 GB Hearthstone install, take a restore point of the *non-game* parts first (~1.3 GB, a few seconds):

```bash
cd "$WINEPREFIX"
tar -I 'zstd -3 -T0' -cf ~/prefix-backup-$(date +%F).tar.zst \
  --exclude="drive_c/Program Files (x86)" \
  drive_c/windows drive_c/users drive_c/ProgramData "drive_c/Program Files" \
  system.reg user.reg userdef.reg .update-timestamp
```

Game files are excluded because `dotnet48` never touches them. The registry and system DLLs are what's at risk, and they're all in there.

**Close Hearthstone and Battle.net before any of this** — and actually check, don't assume:

```bash
pgrep -af "[.]exe"
```

Running winetricks against a live prefix is how prefixes get corrupted. (Ask me how I know.)

---

## 1. Point winetricks at the *Lutris runner*, not system Wine

This is the main Lutris-specific wrinkle. Your prefix was built by a Lutris-managed Wine build, so plain `winetricks` would use your distro's Wine instead — mixing two Wine versions in one prefix.

```bash
RUNNER="$HOME/.local/share/lutris/runners/wine/wine-11.11-staging-amd64"
export WINEPREFIX="$HOME/Games/battlenet"
export WINE="$RUNNER/bin/wine"
export WINESERVER="$RUNNER/bin/wineserver"
export PATH="$RUNNER/bin:$PATH"

winetricks -q -f dotnet48
```

Check which Wine you actually got before running it: `which wine && wine --version`.

Find your prefix and runner from the Lutris config rather than guessing:

```bash
grep -E 'prefix|version' ~/.config/lutris/games/*hearthstone*.yml
```

### The wow64 warning is a false positive on Wine 11

winetricks 20240105 prints:

> warning: You apppear to be using Wine's new wow64 mode. Note that this is EXPERIMENTAL

On Wine 11 this is usually wrong — Wine 11 merged `wine` and `wine64` into one binary, which trips winetricks' detection. Check for a real 32-bit loader instead:

```bash
ls "$RUNNER/lib/wine/i386-unix/ntdll.so"   # exists = traditional multilib, you're fine
```

If that exists, both halves of .NET will install. This matters: **`HDT-Installer.exe` is a 32-bit PE**, so 32-bit .NET isn't optional.

*(Also, for Lutris 0.5.14 with a Wine 11 build: create a `bin/wine64 -> wine` symlink in the runner directory or Lutris won't recognise it.)*

---

## 2. `dotnet48` leaves the prefix on Windows 7 — Battle.net refuses to launch on that

Also covered in the Proton guide, repeated here because it's the step that silently breaks a previously working Hearthstone install. `dotnet48` needs Win7 to install and winetricks **leaves it there**. Modern Battle.net won't run on Win7.

```bash
winetricks -q win10
```

Verify — build 19045, `CSDVersion` empty:

```bash
grep -A15 'Windows NT\\\\CurrentVersion' "$WINEPREFIX/system.reg" \
  | grep -E 'ProductName|CurrentBuildNumber|CSDVersion'
```

**Lutris will not fix this for you.** Unless the game's YAML has a `winver` key, Lutris leaves the Windows version alone, so a Win7 prefix stays Win7.

Re-check .NET survived: `NDP\v4\Full` should still read Release `528049`.

---

## 3. `msdelta` — and how to tell if you actually have it

Same step as in the [main guide](README.md#step-2-install-dependencies). HDT updates via Squirrel, and `HDT-Releases` ships `*-delta.nupkg` files that Squirrel applies using Windows' msdelta API.

```bash
winetricks -q msdelta
```

**The catch:** Wine ships a *builtin stub* `msdelta.dll`, so the file existing proves nothing. Compare it against the runner's builtin — identical means you still have the stub:

```bash
BUILTIN=$(find "$RUNNER/lib/wine/x86_64-windows" -iname msdelta.dll | head -1)
cmp -s "$BUILTIN" "$WINEPREFIX/drive_c/windows/system32/msdelta.dll" \
  && echo "STILL THE BUILTIN STUB" \
  || echo "native msdelta installed"
```

A native `msdelta.dll` is roughly 450 KB (451584 bytes here), matching the size check in the main guide.

Without native msdelta, expect Squirrel to fall back to full ~22 MB packages instead of small deltas, or fail to update.

The main guide also installs `corefonts`. HDT rendered fine here without it, but it's cheap insurance if your text looks wrong.

---

## 4. Download from `HDT-Releases`, not the source repo

The easiest one to get wrong:

- `HearthSim/Hearthstone-Deck-Tracker` — the **source** repo. Its releases ship a portable **zip**.
- `HearthSim/HDT-Releases` — the **distribution** repo, with `HDT-Installer.exe`.

They are **not** the same version. In September 2026 the source repo's latest was v1.55.6 while the distribution repo was on v1.57.12.

The zip has `Squirrel.dll` but **no `Update.exe`, no `.nupkg`, no `packages/`**. HDT detects this and shows:

> **Please reinstall Hearthstone Deck Tracker** — This installation can no longer update itself... it will not pick up support for new cards, sets or game modes.

It still runs, so it's easy to click Continue and forget — then a new set drops and cards stop being recognised. Note the Lutris ["Direct Extract" installer](https://lutris.net/games/install/38802/view) is pinned to v1.48.3 and leaves you in exactly this state.

Under Lutris you can simply run the installer inside the prefix (no manual extraction needed). Get it from https://hsdecktracker.net/download/ or:

```bash
curl -LO https://github.com/HearthSim/HDT-Releases/releases/latest/download/HDT-Installer.exe
"$RUNNER/bin/wine" HDT-Installer.exe
```

A correct install — `Update.exe` is what you're checking for:

```
$WINEPREFIX/drive_c/users/$USER/AppData/Local/HearthstoneDeckTracker/
├── app-1.57.12/
├── packages/
├── HearthstoneDeckTracker.exe   <- stub launcher
└── Update.exe                   <- self-updater
```

---

## 5. Verify it actually works — an open window proves nothing

WPF-on-Wine can render a perfectly good UI while tracking nothing. Two file checks beat squinting at the app:

**HDT wrote the game's logging config:**
```bash
cat "$WINEPREFIX/drive_c/users/$USER/AppData/Local/Blizzard/Hearthstone/log.config"
```
You want a `[Power]` section with `Verbose=True` — that's what drives card tracking.

**HDT found the game:**
```bash
grep -i HearthstoneDirectory \
  "$WINEPREFIX/drive_c/users/$USER/AppData/Roaming/HearthstoneDeckTracker/config.xml"
```

**Timing gotcha:** Hearthstone reads `log.config` **at startup**. If the game was running when HDT first wrote it, restart the game or nothing gets logged.

Then play one real match. That's the only proof that counts.

---

## 6. Adding HDT to your Lutris library

Easiest is the GUI: **+ → Add locally installed game**, runner Wine, then set the prefix to your Battle.net prefix and the same Wine version. (You can edit `~/.local/share/lutris/pga.db` directly, but close Lutris first — it caches the game list.)

Point the executable at the **stub launcher**:

```
drive_c/users/$USER/AppData/Local/HearthstoneDeckTracker/HearthstoneDeckTracker.exe
```

**Not** `app-1.57.12/HearthstoneDeckTracker.exe`. When HDT self-updates, the `app-<version>` folder is renamed and a direct path breaks. The stub always forwards to the current version.

Skip PRIME/GPU offload for this entry — HDT is a 2D app and doesn't need the dGPU.

Harmless log noise:
- `libgamemodeauto.so.0 ... wrong ELF class: ELFCLASS64` — 32/64-bit preload mismatch, ignored
- `libEGL warning: egl: failed to create dri2 screen`
- `Initial process has exited (return code: 0)` — the stub handing off; HDT keeps running, though Lutris may stop showing the game as running

---

## Footguns

**`pkill` matches its own command line:**
```bash
pkill -f "Hearthstone Deck Tracker.exe"     # kills the shell running it, too
pkill -f "[H]earthstone Deck Tracker.exe"   # use this
```

**`wineserver -w` hangs forever if anything in the prefix is alive.** It waits for *all* Wine processes. If Hearthstone is running it will never return — check with `pgrep -af "[.]exe"` first.
