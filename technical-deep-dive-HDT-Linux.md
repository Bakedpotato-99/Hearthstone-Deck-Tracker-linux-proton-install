# Hearthstone Deck Tracker on Linux via Steam Proton — Technical Reference

**Purpose:** paste this into an LLM if you are following the companion guide on a
distro, desktop environment, or Steam setup other than the one it was tested on.
It states verified root causes and fixes so the assistant can adapt the
procedure instead of guessing.

**Last verified:** September 2026 — HDT 1.57.10, Proton Experimental
(`experimental-11.0-20260910b`), Linux Mint 22.3 Cinnamon on X11,
winetricks 20240105, system wine 9.0.

---

## Instructions for the assisting LLM

1. **Ask the user what OS, desktop environment, display server (X11 or
   Wayland), and Steam install type (native or Flatpak) they have before giving
   any commands.** Paths and package names differ. Do not assume they match
   Section 2.
2. **Check which dependencies they already have** rather than assuming.
3. **Do not propose anything listed under "Ruled out" (Section 9).** Each was
   tested on the working setup and failed or proved irrelevant. Suggesting them
   wastes the user's time.
4. **State verified facts as facts and uncertain things as uncertain.** Several
   confident-sounding explanations in this project's history turned out to be
   wrong; Section 10 lists them specifically so they are not repeated.
5. **Wine emits large volumes of harmless noise.** See Section 8. Do not treat
   `fixme:`, `err:ole:`, `err:module:import_dll`, or Fontconfig messages as the
   cause of a problem unless nothing else explains it.

---

## 1. What this setup achieves

HDT, a Windows-only WPF application, runs on Linux against Hearthstone's own
Proton prefix, with:
- overlay rendering and responsive to clicks
- Battlegrounds combat simulator (Bob's Buddy) working
- HDT's own auto-update working
- HSReplay.net login working (requires a one-time Windows step — Section 7)

---

## 2. Reference environment

Everything below was verified on:

- Linux Mint 22.3 Cinnamon, **X11** (Wayland untested)
- AMD Ryzen (AM5), NVIDIA RTX 50-series, proprietary driver
- Steam: **native package**, not Flatpak
- Proton Experimental (`experimental-11.0-20260910b`)
- System wine 9.0, winetricks 20240105
- Hearthstone is **not** a Steam store title — Battle.net is added as a
  **non-Steam shortcut**, and Hearthstone runs inside Battle.net's prefix

---

## 3. Core architecture

HDT must run **in the same Wine prefix as Hearthstone**, so it can see the game
process and read its logs. It does not need to live *inside* that prefix's
`drive_c` — the application files can sit anywhere on the Linux filesystem, as
long as `WINEPREFIX` points at Hearthstone's prefix when HDT is launched.

### 3.1 Finding the prefix and wine binary

Steam generates the `compatdata` folder name for non-Steam shortcuts from a
hash of the executable path and shortcut name. **It is specific to each
install. Never hardcode it.** Derive both values from the running game:

```bash
PID=$(pgrep -f Hearthstone.exe)
export HSPFX=$(tr '\0' '\n' < /proc/$PID/environ | grep -m1 '^WINEPREFIX=' | cut -d= -f2-)
export PROTON_WINE=$(readlink -f /proc/$PID/exe | sed 's|/files/.*||')/files/bin/wine
```

Notes for adapting this:
- `STEAM_COMPAT_APP_ID` is **`0`** for non-Steam shortcuts. It is not the
  compatdata folder name. Do not use it.
- `PROTON_DIST_PATH` does **not** exist in the environment of recent Proton
  builds. Earlier drafts of this guide used it and broke. The `readlink` on
  `/proc/<pid>/exe` above works because the running process is
  `wine-preloader` inside the Proton directory tree.
- `STEAM_COMPAT_TOOL_PATHS` and `STEAM_COMPAT_DATA_PATH` are present and are
  viable alternative sources if `readlink` fails on the user's setup.

Persist them so the launcher can reuse them:

```bash
printf 'export HSPFX=%q\nexport PROTON_WINE=%q\n' "$HSPFX" "$PROTON_WINE" > ~/.hdt-env
```

### 3.2 The `steamuser` trap

Steam-launched processes run as an internal Windows account named
**`steamuser`**, not the user's Linux username. Anything writing into the
prefix's `AppData` must target:

```
<prefix>/drive_c/users/steamuser/AppData/Roaming/HearthstoneDeckTracker/
```

A manually launched `wine` process creates a **second, separate profile** under
the real Linux username in the same prefix. Using the wrong one produces
`System.IO.DirectoryNotFoundException`. Verify rather than assume:

```bash
find "<prefix>/drive_c/users" -iname "HearthstoneDeckTracker" -type d
```

### 3.3 Working directory

Wine child processes inherit the shell's working directory. If that directory
is deleted mid-session, later shell-outs from inside Wine fail silently with
`getcwd(): No such file or directory`. Any launcher must `cd` into a stable,
permanent directory (HDT's own install folder) before invoking wine.

---

## 4. Dependencies — order is load-bearing

### 4.1 The requirement

HDT's WPF UI needs **native .NET Framework**, not Wine-Mono. Without it, HDT
crashes at startup with:

```
System.IO.FileNotFoundException
  at MS.Internal.NativeWPFDLLLoader.LoadCommonDLLsAndDwrite()
```

or, if Wine-Mono is handling it, a `XamlParseException` referencing
`wpfgfx_cor3.dll`.

### 4.2 Two separate commands, in this order

```bash
WINEPREFIX="$HSPFX" winetricks dotnet472
```
then, separately:
```bash
WINEPREFIX="$HSPFX" winetricks corefonts dotnet48
```

**Running these as one combined command fails.** Verified twice on two
different user accounts: `winetricks -f corefonts dotnet48` into a
Steam-managed prefix with no prior native .NET aborts with:

```
warning: Note: command ... wine dotNetFx40_Full_x86_x64.exe returned status 67. Aborting.
```

The step that fails is `dotnet40`, the first link of the prerequisite chain.
When `dotnet472` is installed first as its own invocation, `dotnet40` succeeds
as part of that chain, and `dotnet48` afterwards then succeeds as an upgrade
over working 4.7.2. **This ordering is the single most important finding for
reproducibility** — it was the difference between a working and a failing
install on two separate attempts.

Do **not** add `-f`. Do **not** set `WINE=` on these commands.

### 4.3 Nothing may be running in the prefix

Steam, Battle.net and Hearthstone must all be closed. winetricks calls
`wineserver -w`, which blocks until **every** process in that prefix exits:

```
warning: Running .../wineserver -w. This will hang until all wine processes
in prefix=... terminate
```

With the game running this hangs indefinitely with no error. On the original
working setup this was satisfied by accident, via a reboot, and was never
documented until later.

### 4.4 Verifying .NET actually installed

**The registry check is unreliable.**
`HKLM\Software\Microsoft\NET Framework Setup\NDP\v4\Full\Release` returns a
plausible value (observed: `0x82348`) on a prefix with **no native .NET at
all**, because Wine-Mono fakes it.

The only reliable check is the presence and size of `wpfgfx_v0400.dll`:

```bash
ls -l "$HSPFX/drive_c/windows/Microsoft.NET/Framework64/v4.0.30319/WPF/wpfgfx_v0400.dll"
```

Approximately **1,764,520 bytes** after `dotnet472`; approximately
**2,056,752 bytes** after `dotnet48` replaces it. Absent means it did not
install, regardless of what winetricks printed.

Also note: `winetricks` may report a verb as "already installed, skipping"
when it is not actually present. Observed directly — a prefix reporting
`dotnet48` as installed was still running Wine-Mono's WPF. If the checkpoint
above fails despite a "skipping" message, force with `winetricks -f`.

### 4.5 Windows version must be reset afterwards

The .NET installer chain silently sets the prefix's reported Windows version to
XP or 7 (observed: both, on different runs). Battle.net then refuses to launch
with an "unsupported OS" error.

```bash
WINEPREFIX="$HSPFX" "$PROTON_WINE" winecfg /v win10
```

Check current state with:
```bash
grep -i "ProductName" "$HSPFX/system.reg"
```

---

## 5. `msdelta` — the blocker for every version after 1.55.6

### 5.1 Root cause

HDT's portable (non-Squirrel) build was **discontinued in v1.56.0** — the
portable release job was removed from the project's CI workflow in that commit,
and the source repo's GitHub Releases API is permanently capped at v1.55.6.
Every build published since is Squirrel-configured.

Squirrel-configured builds run an update check on **every launch**, via
`Core.Initialize()` → `StartupUpdateCheck()` / `CheckForUpdates()`. That path
applies delta patches through **`msdelta.dll`**, Microsoft's binary-diff API.

**Wine only ships a non-functional stub of `msdelta.dll`.** Every export is a
stub. Calling into it aborts the process:

```
wine: Call from <addr> to unimplemented function msdelta.dll.ApplyDeltaW, aborting
...
Description: The process was terminated due to an internal error in the
.NET Runtime at IP <addr> with exit code 80131506.
```

`80131506` is `COR_E_EXECUTIONENGINE` — an unrecoverable CLR-level fault that
bypasses normal .NET exception handling, which is why HDT's own try/catch
around the updater does not help.

The v1.55.6 portable build never hit this because it was compiled **without**
the `SQUIRREL` define, so that code path did not exist in it.

### 5.2 Fix

```bash
WINEPREFIX="$HSPFX" winetricks msdelta
```

This extracts the genuine Microsoft `msdelta.dll` from the Windows 7 SP1
servicing stack and sets a native DLL override. Downloads roughly 500 MB on
first use; cached afterwards.

### 5.3 Verifying it — check the size, not existence

Wine ships a **121-byte stub at the same path**. An `ls` existence check gives
a false pass. The real DLL is roughly **450,000 bytes**:

```bash
ls -l "$HSPFX/drive_c/windows/system32/msdelta.dll"
```

### 5.4 Side effect

Fixing this also resolved long-standing overlay click/responsiveness problems.
HDT's own v1.56.3 changelog notes *"improved the responsiveness of the overlay,
especially when running the deck tracker under Wine"* — an upstream fix that
was unreachable while the updater crashed on launch. If a user reports overlay
click problems on a version before 1.56.3, updating is the fix, not
window-manager tweaking.

---

## 6. `WINEFSYNC` — Bob's Buddy crash (September 2026)

### 6.1 Symptom

HDT starts normally, overlay works, then HDT **exits silently after a few
minutes**. Only during Battlegrounds, typically several combats into a game.
No error dialog.

### 6.2 Evidence

- Wine output: `.NET Runtime internal error ... exit code 80131506`, followed
  by `Unhandled exception code c0000005` (access violation).
- A `WINEDEBUG=+loaddll` run placed the crashing address inside native
  `clr.dll`, the core .NET 4.8 runtime.
- HDT's own logs ended during Bob's Buddy combat setup across six sessions.
  Three stopped at exactly
  `Running simulations with MaxIterations=10000 and ThreadCount=8...`.
- The prefix itself was healthy: real `msdelta`, real .NET 4.8, Windows 10, no
  rebuild needed.
- Versions at the time: Proton Experimental `20260910b`, HDT 1.57.10.

### 6.3 Cause

Steam launches Hearthstone with **`WINEFSYNC=1`**, so the game handles thread
waiting and locking through fsync. A hand-written HDT launcher that does not
set that variable runs with Wine's **default** sync method — while connected to
**the same wineserver** as the game. The two processes coordinate threads by
different rules.

Most of the time this goes unnoticed. When Bob's Buddy spawns 8 simulation
threads at once, the .NET runtime intermittently hits an access violation.
.NET cannot catch that class of fault, so HDT exits with no message.

**It is not confirmed why this surfaced when it did.** The HDT and Proton
updates may have made the timing window more likely, but the mismatch predates
them — one archived log from earlier in September already ends mid-simulation.

### 6.4 Fix

One line in the launcher, after `export WINEPREFIX`:

```bash
export WINEFSYNC=1   # match the game's thread-sync method
```

### 6.5 Verification

- A running HDT process shows `WINEFSYNC=1` in `/proc/<pid>/environ`.
- A test session completed 15 Bob's Buddy combats with no crash. Without the
  fix, HDT crashed at combats 7–8.

### 6.6 If it returns

Confirm the line is still present:
```bash
grep WINEFSYNC ~/Games/HDT/launch_hdt_gui.sh
```

Compare the sync method of both processes, in case a Proton update switched the
game to something else (for example ntsync):
```bash
for p in Hearthstone.exe HearthstoneDeckTracker.exe; do PID=$(pgrep -f "$p" | head -1); echo "== $p"; tr '\0' '\n' < /proc/$PID/environ | grep SYNC; done
```
Then set the launcher to match whatever the game reports.

**Fallbacks, never needed in testing:** `COMPlus_gcConcurrent=0` (disables
.NET's background garbage collector) or `WINE_CPU_TOPOLOGY=4:0,1,2,3` (makes
HDT see only 4 cores, reducing thread count). Disabling Bob's Buddy also avoids
the crash, at the cost of combat odds.

---

## 7. HSReplay.net login

### 7.1 What fails

Clicking Login triggers an OAuth2 flow: HDT calls `ShellExecuteW` on an
authorize URL with a `localhost` redirect, and listens on a local port for the
callback. **The browser never opens.** Every layer reports success —
`winebrowser.exe` calls `xdg-open`, `xdg-open` detects the desktop correctly,
`gio open` returns exit code 0 — and nothing appears.

### 7.2 What was ruled out

Two separate failure modes were found, and **neither fully explains it**:

- Under Steam's Proton sandbox, `xdg-desktop-portal` rejects the request:
  `Could not get pidns: pidns required but no pidfd provided` — Steam's
  `pressure-vessel`/`bwrap` layer puts the calling process in a nested PID
  namespace the portal cannot resolve across.
- **But login also fails in a clean, unsandboxed Wine prefix with no Steam
  involvement at all**, with a different signature:
  `recvmsg: Connection reset by peer` after Chrome's launcher has started —
  consistent with a Chrome singleton-IPC handoff failure, not a portal problem.

So the sandbox explanation is real but **not the sole cause**. Do not present
it as the root cause to a user on plain Wine.

Shadowing `xdg-open` with a script calling the browser binary directly was
tested and did **not** reliably work. Cause not isolated.

### 7.3 Working method — token transfer

Using `hdte.exe` from
`https://github.com/borisbabic/hearthstone_hdt_linux` (Apache-2.0; its
encryption scheme still matches current HDT builds):

1. On a Windows machine or VM with HDT already logged in:
   ```
   hdte.exe decrypt %AppData%\HearthstoneDeckTracker\hsreplay_oauth hsreplay_oauth.decrypted
   ```
2. Transfer the decrypted file to Linux.
3. Close HDT. Then:
   ```bash
   WINEPREFIX="$HSPFX" "$PROTON_WINE" hdte.exe encrypt hsreplay_oauth.decrypted \
     "$HSPFX/drive_c/users/steamuser/AppData/Roaming/HearthstoneDeckTracker/hsreplay_oauth"
   ```
   Note `steamuser` — see Section 3.2.
4. Relaunch HDT; it reads the transplanted token as authenticated.

**The source project's README states that on sufficiently new plain Wine
(their baseline: 5.0 staging), with no Steam sandbox involved, login may work
normally.** A user not going through Steam should try normal login first.

This is the one step that still requires Windows once. An attempt to eliminate
it using a throwaway Linux Wine prefix failed (Section 7.2). **Untested idea
worth trying:** HDT prints the authorize URL in a `WINEDEBUG=+winebrowser`
trace; pasting that URL into a normal Linux browser manually, completing login
there, and letting HDT's own localhost listener catch the redirect may work,
since the listener is bound to a real host socket. If it does, the Windows
dependency disappears entirely.

---

## 8. Log noise — safe to ignore

These appear constantly and are not causes of failure:

- `Fontconfig error: ... out of memory` and
  `Cannot load config file from /etc/fonts/fonts.conf` — appear on nearly every
  Wine invocation on this hardware, including on the fully working setup
- anything containing `fixme:`
- `err:ole:`, `err:setupapi:`, `err:module:import_dll`
- `wineserver: using server-side synchronization.`
- `err:ole:apartment_add_dll couldn't find function DllGetClassObject in
  ... uiautomationcore.dll` — Wine's incomplete UI Automation; produces a
  dismissable error dialog on some Battle.net/Wine operations
- `err:crypt:check_and_store_certs ... CERT_FIRST_USER_PROP_ID property absent`
- `err:combase:RoGetActivationFactory ... AsyncCausalityTracer`
- `warning: You are using a 64-bit WINEPREFIX...`
- `warning: You appear to be using Wine's new wow64 mode...`

---

## 9. Ruled out — do not suggest these

Each was tested on the working setup. Suggesting them wastes the user's time.

- **Lutris and Bottles** — both block running Battle.net and HDT concurrently
  in the same prefix. Root cause never isolated; the GUI-wrapper path was
  abandoned. (A *manual* shared Wine prefix, launched directly from two
  terminals, does work — but was noticeably choppier in gameplay than
  attaching to Steam's prefix.)
- **Running `HDT-Installer.exe` against the Steam-managed prefix** — crashes
  reliably. It **does** run to completion in a fresh, manually created prefix
  (`wineboot --init` + dependencies), which is a viable alternative route to
  obtain the file layout. The reason for the difference was never identified.
- **Adding HDT to Steam as a non-Steam shortcut** — tested with
  `STEAM_COMPAT_DATA_PATH` correctly redirected into Hearthstone's prefix
  (verified via `/proc/<pid>/environ`); hangs without producing any Squirrel
  log output. Direct `wine` invocation is the only reliable launch method.
- **`protontricks`** — crashed on the test system with
  `SyntaxError: Invalid file magic number` while parsing Steam's `appinfo.vdf`,
  on every invocation, regardless of a correct AppID. Use plain `winetricks`
  with `WINEPREFIX=` set. (This is a system-specific bug, not universal — if
  protontricks works on the user's machine, it is a valid substitute.)
- **`-force-d3d9` or forcing a renderer** — never needed; overlay renders
  without it.
- **`log.config` / enabling verbose Hearthstone logging** — never needed;
  detection worked automatically once HDT ran in the same prefix.
- **The HDT portable zip from the source repo** — permanently discontinued as
  of v1.56.0, capped at v1.55.6.
- **`WINE_LAYERED_OVERLAY_ALPHA` / `WINE_LAYERED_OVERLAY_SHAPE`** — these
  appeared in AI-generated research at the start of this project and do not
  appear to be real Wine or DXVK variables. The overlay rendered correctly
  without any such variable.
- **`mscoree=native` override** — set as a hypothesis, did not fix the crash it
  was aimed at, and removing it later did not fix a different crash either.
  Inert.

---

## 10. Explanations that were wrong

Listed so they are not re-derived. Each was asserted confidently before being
disproven.

- **"A new WinRT API call in v1.56 causes the startup crash."** Disproven by
  reading HDT's source. The real cause was `msdelta` (Section 5). The WinRT
  trace lines simply happened to sit near the fatal exception.
- **"Steam's `pidns`/portal sandboxing is the cause of login failure."**
  Disproven — login also fails outside any sandbox (Section 7.2).
- **"Fontconfig errors indicate running outside the Steam Runtime."** They
  appear on every Wine invocation including the working setup.
- **"The two `80131506` crashes share one root cause."** They share only the
  CLR's generic give-up code. One is a WPF GUI app, the other a console
  updater with a different dependency graph. Treat separately.

**The common failure mode in all four: an anomaly appeared near a failure and
was assumed to cause it.** When diagnosing, prefer reading the application's
source or an authoritative log over inferring causation from trace proximity.

---

## 11. Known-untested territory

- **Wayland** — entirely untested. The overlay diagnostics referenced here
  (`xprop`, `wmctrl`, `xwininfo`) are X11-only.
- **Other desktop environments** — overlay click/focus behaviour is known to
  vary. The `borisbabic/hearthstone_hdt_linux` README documents different
  behaviour on KDE (overlay hidden in fullscreen), XFCE, and tiling WMs
  (Awesome needs an explicit always-on-top rule). Not re-verified here.
- **Flatpak Steam** — path detection would need adapting
  (`~/.var/app/com.valvesoftware.Steam/data/Steam`), and its additional
  sandboxing layer may change the login behaviour in Section 7.
- **Non-NVIDIA GPUs** — all testing was on the NVIDIA proprietary driver.
- **Distros other than Linux Mint 22.3** — package names and winetricks
  versions will differ. The `msdelta` verb in particular requires a
  sufficiently recent winetricks; verify with
  `winetricks list-all | grep -c msdelta` (must return ≥ 1).

---

## 12. Reference: launcher script

```bash
#!/bin/bash

HDT_DIR="$HOME/Games/HDT"
source "$HOME/.hdt-env"

export WINEPREFIX="$HSPFX"
export WINEFSYNC=1   # match the game's thread-sync method (prevents a Bob's Buddy crash)
cd "$HDT_DIR"

setsid "$PROTON_WINE" "HearthstoneDeckTracker.exe" >/dev/null 2>&1 &
disown
```

`setsid` and `disown` let HDT survive the terminal closing. The `cd` is
required — see Section 3.3.

**Launch order matters:** start Hearthstone first, then HDT. Launching
Battle.net while HDT is already running can prevent Battle.net from starting
correctly. Not root-caused; simply avoid it.

---

## 13. References

- HDT: https://github.com/HearthSim/Hearthstone-Deck-Tracker
- HDT's actual release channel: https://github.com/HearthSim/HDT-Releases
- OAuth token tool (Apache-2.0):
  https://github.com/borisbabic/hearthstone_hdt_linux
