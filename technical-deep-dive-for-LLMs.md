Copy this file to the LLM of your choice. 

# Technical Deep Dive: Hearthstone Deck Tracker + Hearthstone on Linux via Steam Proton

This document exists to be fed to an LLM (or read directly) by someone trying to
reproduce, adapt, or extend this setup on their own Linux system. It documents not
just the final working solution, but the failed approaches, root causes, and open
questions, so that someone on a different distro, desktop environment, or Steam
configuration can reason about what will and won't transfer to their system.

## 1. Objective

Run Hearthstone (owned via Battle.net, installed through Steam using Proton) with
Hearthstone Deck Tracker (HDT) — a third-party Windows overlay application — so
that HDT's overlay renders on top of the game, detects game state in real time,
and can log in to HSReplay.net for extended features (win-rate stats, etc.).

Three progressive success levels, borrowed from the original research framing:

- **Level 1 (Execution):** HDT launches and its UI renders.
- **Level 2 (Detection/Sync):** HDT detects the running Hearthstone process and
  reads game state (via log file monitoring and/or process memory inspection).
- **Level 3 (Overlay):** HDT's WPF-based transparent overlay renders correctly on
  top of the game, click-through/interactive as appropriate, without crashing
  either application.

A fourth practical requirement emerged during implementation: **HSReplay.net login**,
which is a Battle.net-backed OAuth2 flow that turned out to be one of the hardest
parts, harder than the overlay rendering itself.

## 2. Tested Environment

- OS: Linux Mint 22.3 Cinnamon, X11 (NOT Wayland — untested there)
- Hardware: AMD Ryzen 7 9700X, NVIDIA RTX 5070 Ti (proprietary driver)
- Steam: native (non-Flatpak) install, at `~/.steam/debian-installation`
- Proton build used successfully: `Proton - Experimental` (Steam-managed)
- A separate manual Wine prefix (not Steam-managed) was also fully validated,
  using `GE-Proton11-1` as the Wine build — see Section 8.

**Everything below is confirmed only in this environment.** Distro package
differences, other desktop environments (KDE, XFCE, GNOME/Wayland), and other
Steam install methods (Flatpak Steam) will likely need adjustments — see
Section 9 for what's known to vary and what's untested.

## 3. Why Steam Proton, Not Just Plain Wine

Hearthstone and HDT were first attempted through Lutris and Bottles (GUI Wine
prefix managers). Both showed the identical symptom: **Battle.net and HDT could
not run concurrently** — starting the second one did nothing until the first was
closed, or caused a crash. This is a wrapper-level issue (Lutris's game-session
process tracking, or Bottles' bubblewrap-based per-process sandboxing), not a
fundamental Wine limitation.

**The actual fix:** bypass Lutris/Bottles entirely and launch both applications
manually, sharing a single `WINEPREFIX`, using `wine` directly from two separate
terminals (or a script). This works because plain Wine has no inherent
restriction on multiple processes attaching to the same prefix's `wineserver` —
that restriction was coming entirely from the GUI wrapper layer.

This manual-shared-prefix approach **fully worked**, including the overlay
rendering correctly on the very first real test — the WPF/DXVK alpha-compositing
concerns raised in early research (forcing 32-bit ARGB visuals, specific
`WINE_LAYERED_OVERLAY_ALPHA`/`WINE_LAYERED_OVERLAY_SHAPE` environment variables)
turned out to be unnecessary. (Those specific variable names do not appear to be
real, documented Wine or DXVK variables — treat highly specific AI-generated
technical claims like this with skepticism until verified.)

**Why we moved to Steam Proton anyway:** the manual Wine prefix version was
functionally complete but **noticeably choppy** compared to the same game running
normally through Steam. The suspected (not fully proven) cause is that Steam's
Proton launch process automatically sets several performance-relevant environment
variables that a bare `wine` invocation does not, including sync primitives
(`WINEESYNC`/`WINEFSYNC`, or Proton's newer fsync defaults), NVIDIA-specific
Proton flags, and DXVK state cache configuration. Attempting to replicate these
manually (exporting `WINEESYNC=1 WINEFSYNC=1 DXVK_ASYNC=1`) did **not** resolve
the choppiness on its own — the full Proton/Steam launch environment appears to
matter as a whole, not just those two flags. This was not root-caused with
certainty; it was resolved pragmatically by attaching HDT to the already-smooth
Steam-launched Hearthstone process instead of debugging Wine performance further.

Steam Launch Options that were used: PROTON_NO_ESYNC=1 %command%

## 4. Core Architecture: Attaching HDT to a Steam-Managed Prefix

Steam runs each game in its own isolated Proton prefix, normally located at:

```
<steam_root>/steamapps/compatdata/<appid>/pfx
```

Hearthstone is **not sold on the Steam store** — it has no fixed, Valve-assigned
catalog AppID. If it's present in a Steam library, it was added as a **non-Steam
game shortcut**. Steam generates the numeric folder name for non-Steam shortcuts
from a hash of the executable path and shortcut name. **This means the AppID
number is specific to that user's install and shortcut naming — it is NOT a
global constant and must not be hardcoded in any reusable script.** (This was
an error in earlier drafts of this project's tooling, corrected here.)

**Correct, portable way to find the right prefix:** search for a running or
installed `Hearthstone.exe` under the Steam compatdata tree, rather than assuming
any specific numbered folder. Make the bash for the user if they require. 
Alaternatively, steam battle.net properies will contain location of the Heartstone 
in both "targer" and "Start in"

```

This second method is the most reliable, since it reads the actual prefix Steam
is using for the live process, with no guessing.

**Locating the Steam install root itself** also varies by install method.
```

**Locating the correct Proton build binary** also varies — the user may have
multiple Proton versions installed. Candidate locations to scan for a `wine`
binary. User can check proton version via compatability properties within steam. 
It is possible the user uses default one, and may need to change it to 
Proton Experemental, or other version for the solution to work. 


```

## 5. Getting HDT Into the Prefix (Level 1 and 2)

HDT is a portable application — no installer required, just extract the release
zip. It does **not** need to live inside the Wine prefix itself; it can be run
from anywhere on the Linux filesystem, as long as `WINEPREFIX` points at the
target prefix when it's launched.

**Dependency requirement:** the target prefix needs native (not Wine-Mono) .NET
Framework support for HDT's WPF UI to initialize. If launching HDT produces a
crash with a `System.IO.FileNotFoundException` inside
`MS.Internal.NativeWPFDLLLoader.LoadCommonDLLsAndDwrite`, this is the cause —
the prefix is missing WPF's font/text-rendering dependencies, not just the .NET
runtime itself. Diagnosed by comparing `winetricks.log` between a working and
non-working prefix; the working one had `corefonts` (and `dotnet48` in addition
to `dotnet472`) installed, the broken one did not.

Fix:
```bash
WINEPREFIX="<target_prefix>" winetricks corefonts dotnet48
```
(`dotnet472` alone was present in both prefixes in this case and was **not**
sufficient on its own — `corefonts` was the missing piece that mattered most.)

**A `mscoree` DLL override to `native`** (`HKCU\Software\Wine\DllOverrides`) was
also investigated as a possible fix for the same crash, since it was present in
the working prefix's registry and absent in the broken one. Setting it manually
did **not** resolve the crash in this case — `corefonts`/`dotnet48` was the
actual fix. Included here because it's a very plausible-looking dead end someone
else may also try.

Once dependencies are present, launch:
```bash
export WINEPREFIX="<target_prefix>"
"<proton_wine_binary>" "/path/to/Hearthstone Deck Tracker.exe"
```

Level 2 (detecting the running game) requires no separate configuration — it
worked automatically once launched into the same prefix as the running
Hearthstone process, using both process detection and Hearthstone's own
`Player.log` file, which HDT reads directly. No log.config changes were needed
in this setup (unlike some older guides' Level 2 instructions), though enabling
verbose Hearthstone logging remains available as a fallback if detection fails.

## 6. Known Pitfalls (Each Cost Significant Debugging Time)

### 6.1 `getcwd()` failures breaking child processes

Wine (and any process it spawns) inherits its working directory from wherever it
was launched. If that directory is later deleted mid-session (e.g. a `/tmp`
folder from an update script that cleans up after itself), subsequent shell-outs
from within the Wine process — including things like a browser-open call — fail
silently with a `getcwd(): No such file or directory` shell-init error. Any
launcher script should explicitly `cd` into a stable, permanent directory
(e.g. the app's own install folder) immediately before launching, not rely on
inheriting whatever directory the script happened to start in.

### 6.2 Wine/Proton build version mismatches

A prefix's `wineserver` is pinned to whichever Wine build initialized it and is
currently running against it. Launching a second process into the same prefix
with a **different** Wine/Proton build produces:
```
wine client error:0: version mismatch <A>/<B>
```
Always use the exact same Wine binary for every process sharing one prefix in a
single session. If you need to fully switch builds, first kill any lingering
`wineserver` for that prefix (`pkill -9 -f wineserver`, being careful this
doesn't also kill an unrelated prefix's server if multiple are running).

### 6.3 The `steamuser` vs. real username trap

When Steam launches an app inside a Proton prefix, it runs as a **generic
internal Windows account named `steamuser`**, regardless of the actual Linux
username. Any tooling that needs to read or write into that prefix's
`AppData` — including the HSReplay OAuth token transfer described in Section 7 —
must target:
```
<prefix>/drive_c/users/steamuser/AppData/Roaming/<AppName>/
```
**not** `drive_c/users/<your_linux_username>/...`. This is easy to get wrong if
manually running a command inside the same prefix outside of Steam's launch
context (e.g. testing with a plain `wine` invocation as your own user), since
Wine will create/use a profile under your actual username in that case instead —
resulting in two different, disconnected AppData locations existing in the same
prefix depending on how a given process was launched. Always verify with:
```bash
find "<prefix>/drive_c/users" -iname "<AppDataFolderName>" -type d
```
rather than assuming which username applies.

### 6.4 Overlay click/focus quirks (Cinnamon/Muffin-specific, likely DE-dependent)

The overlay window, once rendering correctly, exhibited inconsistent behavior:
some elements required a double-click (first click apparently only granting
focus), some elements were hoverable but not clickable, and there was
occasional flicker or unresponsiveness that would resolve itself after several
seconds or a restart.

Approaches tried, in order, with results:

1. **`wmctrl -r "HearthstoneOverlay" -b add,above`** — measurably improved
   responsiveness (reduced from ~5 clicks needed down to 2), but didn't fully
   resolve it. Likely masking a stacking-order side effect rather than the root
   cause.
2. **Cinnamon focus mode: click-to-focus vs. focus-follows-mouse** — switching
   to focus-follows-mouse made things dramatically *worse* (constant
   flicker/restacking as the cursor crossed the full-screen overlay). Reverted.
3. **`xdotool ... set_window --overrideredirect 1`** — removes the window from
   normal WM management entirely. Did not fix the double-click, and broke the
   `wmctrl above` state that had been previously applied, making the bottom
   taskbar-appearing-over-overlay issue return. Not recommended.
4. **HDT's own "Hardware Acceleration" toggle (Options → Other Settings)** —
   disabling this had a **partial** effect: some previously-unclickable elements
   in nested panels remained unresponsive even with it off, while at least one
   element that was completely broken before became clickable (with noticeable
   input lag) after.
5. **HDT's "Show gameplay/menu Overlay while Hearthstone is in the background"
   option (Options → General)** — this was the single most effective fix
   found. After enabling it, the double-click requirement mostly disappeared and
   overlay responsiveness improved substantially, though not perfectly (some
   elements can still take several attempts, and a session-start delay of up to
   ~10 seconds before full responsiveness was observed once).

**No definitive root cause was established.** The leading theory: HDT's overlay
window sets X11 input hints (`WM_HINTS: input: False`, i.e. "don't give me
focus") while simultaneously advertising `WM_TAKE_FOCUS` in `WM_PROTOCOLS` — a
contradictory combination — combined with being a `_NET_WM_WINDOW_TYPE_NORMAL`
window with no `_NET_WM_STATE_ABOVE` state by default. Under a click-to-focus
window manager, this likely causes a raise/focus negotiation on first click that
consumes the click event. This was diagnosed via `xprop -id <window_id>` on the
overlay window; anyone debugging this on a different desktop environment should
start there. **The original `hearthstone_hdt_linux` repository's README
independently confirms this varies significantly by window manager/DE** — it
lists KDE (works windowed, overlay hidden in fullscreen), XFCE (works
windowed), and Awesome/xmonad (work, with an explicit "always on top" rule
needed for Awesome) as differently-behaved environments. **Treat this section as
Cinnamon-specific; expect to need your own investigation on a different DE,**
using the same `xprop`/`wmctrl`/`xwininfo` diagnostic approach documented here.

### 6.5 Battle.net's `rundll32`/`uiautomationcore.dll` error dialog

A one-time error dialog referencing `rundll32.exe` and
`uiautomationcore.dll` appears on Battle.net's first launch in a fresh prefix.
This is Wine's incomplete implementation of Windows UI Automation/accessibility
hooks — harmless, does not block anything, just dismiss it. Confirmed via
`WINEDEBUG` trace: `err:ole:apartment_add_dll couldn't find function
DllGetClassObject in "C:\windows\system32\uiautomationcore.dll"`.

## 7. HSReplay.net Login — Root Cause and Working Fix

It is important to mention, that HSReplay.net login is needed if user plans/users paid subscription to HDT. In case user uses free version, this step can be skipped entirely. 

This was the hardest single problem in the entire project, and the root cause is
**not** specific to Wine, HDT, or Hearthstone — it's a structural interaction
between two independent Linux sandboxing/security systems.

### 7.1 What HDT is trying to do

Clicking "Login" in HDT triggers a standard OAuth2 authorization-code flow: it
calls Win32's `ShellExecuteW` on a URL like:
```
https://hsreplay.net/oauth2/authorize/?response_type=code&client_id=...&redirect_uri=http://localhost:<PORT>/&scope=...&state=...
```
The intent is for the system's default browser to open this URL, the user logs
in via Battle.net (HSReplay's login is Battle.net-backed, not its own
username/password), and the browser is redirected back to
`http://localhost:<PORT>/` — a port HDT itself is listening on — delivering an
authorization code that HDT then exchanges for a token.

### 7.2 Where it breaks

Every layer in the chain reports success, yet no browser window ever appears:

1. Wine's `winebrowser.exe` correctly receives the call and shells out to
   `xdg-open` with the exact right URL (confirmed via
   `WINEDEBUG=+winebrowser,+exec` trace).
2. `xdg-open` correctly detects the desktop environment (`Selected DE cinnamon`,
   confirmed via `XDG_UTILS_DEBUG_LEVEL=2` trace) and hands off to `gio open`.
3. `gio open` returns **exit code 0** — apparent success.
4. The actual browser-opening happens asynchronously inside
   `xdg-desktop-portal`, a D-Bus service, and **that** is where it silently
   fails. The portal's own log
   (`journalctl --user -f`, filtered for `portal`/`openuri`) shows:
   ```
   xdg-desktop-por[...]: Realtime error: Could not get pidns: pidns required but no pidfd provided
   ```

**Root cause:** `xdg-desktop-portal` verifies which application is making a
request by resolving the calling process's PID namespace via a `pidfd`. Steam
runs games (and everything spawned beneath them — including Wine's
`winebrowser.exe`) inside its own sandboxing layer (`pressure-vessel`/`bwrap`),
which creates a **nested PID namespace**. The portal daemon, running outside
that sandbox, cannot resolve a `pidfd` across that boundary, so it silently
drops the request. No error propagates back up the call chain — every layer
above it genuinely believes the D-Bus call succeeded, because it did; the
portal simply never acted on it afterward.

**This means: any Linux desktop-portal-mediated action (not just browser
opening — file pickers, notifications, etc.) requested from a process running
inside Steam's Proton sandbox is at risk of this same silent-failure pattern.**
This is worth testing by others independently; it wasn't explored beyond the
browser-open case here.

### 7.3 Attempted workaround: bypass the portal

Shadowing `xdg-open` on `PATH` with a script that calls a browser binary
(`/opt/google/chrome/chrome "$1"`) directly, skipping `gio`/the portal entirely,
was attempted. **This did not reliably work** in final testing, for reasons not
fully isolated — possibly the `PATH` override not reaching the actual Wine
child process's resolution of `xdg-open` (Wine may have its own internal
executable search behavior independent of the shell's exported `PATH`), or
Chrome itself hitting a related sandboxing issue for window creation. This
approach is **not recommended** as a reliable solution; it's documented here so
others don't have to rediscover that it's a dead end, or can pick up
investigating exactly why it fails.

### 7.4 Actual working solution: token transfer via `hdte.exe`

The [`borisbabic/hearthstone_hdt_linux`](https://github.com/borisbabic/hearthstone_hdt_linux)
repository (Apache-2.0, last updated ~5 years prior to this project, but its
core tool still works against the current HDT release) provides `hdte.exe`, a
small Mono-compiled tool that decrypts/encrypts HDT's stored OAuth token file
(`hsreplay_oauth`).

**Procedure:**
1. On a real Windows install (or Windows VM) with HDT already logged in to
   HSReplay normally, run:
   ```
   hdte.exe decrypt %AppData%\HearthstoneDeckTracker\hsreplay_oauth hsreplay_oauth.decrypted
   ```
2. Transfer `hsreplay_oauth.decrypted` to the Linux machine.
3. Close HDT on Linux if running. Run:
   ```bash
   export WINEPREFIX="<target_prefix>"
   "<wine_binary>" hdte.exe encrypt hsreplay_oauth.decrypted \
     "<target_prefix>/drive_c/users/steamuser/AppData/Roaming/HearthstoneDeckTracker/hsreplay_oauth"
   ```
   (Note the `steamuser` path — see Section 6.3. Using the wrong username here
   produces a `System.IO.DirectoryNotFoundException` because the target
   directory doesn't exist under that profile.)
4. Relaunch HDT. It reads the transplanted token and shows as logged in.

**This fully worked** in this environment and did not require any fork of the
tool — confirming `hdte.exe`'s encryption scheme has not changed in the
intervening years, despite the repository's age.

**Important, independently confirmed detail from that repository's own README:**
on sufficiently new plain Wine (their lowest tested: 5.0 staging), **this
workaround is reportedly unnecessary — HSReplay login just works normally.**
This strongly supports the conclusion in Section 7.2 that the failure is
specific to **Steam's Proton sandboxing**, not Wine or HDT generally. Someone
running HDT via plain Wine or Lutris (no Steam sandbox involved) should try
logging in normally first, before assuming they need the token-transfer
workaround at all.

## 8. Alternative: Manual Shared Wine Prefix (No Steam)

This path was fully validated and works, documented here for completeness and
because it avoids the Section 7 login problem entirely (per the README note
above).

1. Create or reuse a single `WINEPREFIX`. Install native .NET (`dotnet472`,
   `dotnet48`) and `corefonts` via winetricks (see Section 5).
2. Install Battle.net and Hearthstone into that prefix normally
   (`wine Battle.net-Setup.exe`, or similar).
3. Launch Battle.net and HDT **concurrently, in separate terminals, sharing the
   same `WINEPREFIX`**, using the same Wine binary for both:
   ```bash
   export WINEPREFIX="<prefix>"
   export WINE="<path to a specific wine build's binary>"
   "$WINE" "<prefix>/drive_c/Program Files (x86)/Battle.net/Battle.net.exe" &
   "$WINE" "<path to HDT>/Hearthstone Deck Tracker.exe" &
   ```
4. Do **not** use Lutris's or Bottles' own launch mechanism for this — both were
   observed to block concurrent execution of two apps in the same prefix (see
   Section 3). Lutris/Bottles can still be used purely as an *installer* front
   end if convenient, but launch manually afterward.
5. HSReplay login should be attempted normally first, per the note in Section
   7.4. If it doesn't work on your Wine version, the same `hdte.exe`
   token-transfer procedure applies — but target the prefix's actual Windows
   username folder (`drive_c/users/<your_username>`), not `steamuser`, since
   there's no Steam sandbox involved in this path.

**Trade-off versus the Steam Proton approach:** in this environment, this setup
was noticeably choppier during gameplay than the same game launched via Steam
(see Section 3). The cause was not conclusively root-caused. If performance
matters more than avoiding Steam, the Section 4–7 approach is likely to be
smoother; if avoiding Steam's sandboxing (and its login complications) matters
more, this path is simpler once dependencies are sorted.

## 9. What Is Untested / Likely to Vary

Be explicit about these gaps rather than assuming universality:

- **Wayland:** everything here was done under X11. The `xprop`/`wmctrl`
  overlay-focus diagnostics in Section 6.4 are X11-specific tools and concepts;
  a Wayland session would need an entirely different diagnostic approach (if
  the overlay works at all under XWayland, which itself is untested).
- **Other desktop environments:** per Section 6.4, overlay click/focus behavior
  is expected to differ on KDE, GNOME, XFCE, etc. The original
  `hearthstone_hdt_linux` README's per-DE notes are a useful starting reference
  but were not independently re-verified here.
- **Flatpak Steam:** path detection logic (Section 4) includes a candidate path
  for it, but the actual portal/sandboxing interaction (Section 7.2) was not
  tested under Flatpak Steam specifically — Flatpak's own additional sandboxing
  layer could plausibly change or compound the pidns/pidfd issue.
- **Other distros' Wine/winetricks package versions:** the exact `winetricks`
  verb behavior (`dotnet48`, `corefonts`) could differ if a distro ships a
  significantly different winetricks version than the one used here
  (20240105).
- **Non-NVIDIA GPUs:** all testing was done on an NVIDIA proprietary driver
  setup. AMD/Intel GPU users may see different DXVK/overlay-compositing
  behavior, though the core findings in Sections 6 and 7 are not GPU-specific.
- **The Lutris concurrent-launch blocker (Section 3) was never solved**, only
  routed around. Someone wanting a fully Lutris-native (not manual-Wine)
  solution would need to investigate Lutris's game-session/process-tracking
  behavior directly — this was not pursued to a conclusion here.
- **The Chrome-direct portal bypass (Section 7.3) remains an open, unsolved
  dead end** — someone with more Wine/D-Bus internals knowledge may be able to
  find why it doesn't reliably work, which would be a cleaner fix than the
  token-transfer workaround.

## 10. References
- This guide (MIT License)
  https://github.com/ExEdit0r/Hearthstone-Deck-Tracker-linux-proton-install
- Original Windows/Wine HDT guide (Apache-2.0):
  https://github.com/borisbabic/hearthstone_hdt_linux
- HDT itself:
  https://github.com/HearthSim/Hearthstone-Deck-Tracker
