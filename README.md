# Hearthstone Deck Tracker on Linux (Steam Proton)

I managed to make HDT work on Linux Mint 22.3 – Cinnamon 64-bit with Hearthstone Battlegrounds running via Steam Proton.
Overlay works, combat simulation works, automatic update to a new version works. Login to hsreplay.net is also possible.


This setup was tested only on Linux Mint. If you run into any problem with the guide, use a different distro, or don't trust the guide, see the **[technical deep dive](https://github.com/Bakedpotato-99/Hearthstone-Deck-Tracker-linux-proton-install/blob/main/technical-deep-dive-HDT-Linux.md)**. It contains an in-depth explanation of the setup and is designed to be copied into the LLM of your choice so it can adapt the steps to your system.

Logging in to hsreplay.net requires booting up a Windows machine or a Windows VM. More on how to do that here: https://github.com/borisbabic/hearthstone_hdt_linux

You may want to enable "Show gameplay overlay while Hearthstone is in the background" if you encounter problems with HDT overlay responsiveness. 
---

## Step 1. Find the Hearthstone Wine prefix

Start **Battle.net** and **Hearthstone**. Then run the commands below. The first block finds where Hearthstone and Proton live, the second saves that info for future use. Copy each block separately.

```bash
PID=$(pgrep -f Hearthstone.exe)  # find the process ID of the running Hearthstone
export HSPFX=$(tr '\0' '\n' < /proc/$PID/environ | grep -m1 '^WINEPREFIX=' | cut -d= -f2-)  # read the game's Wine prefix path from its environment
export PROTON_WINE=$(readlink -f /proc/$PID/exe | sed 's|/files/.*||')/files/bin/wine  # work out which Proton's wine binary runs the game
echo "$HSPFX"  # print the prefix path so you can check it
echo "$PROTON_WINE"  # print the Proton wine path so you can check it
```

```bash
printf 'export HSPFX=%q\nexport PROTON_WINE=%q\n' "$HSPFX" "$PROTON_WINE" > ~/.hdt-env  # save both paths to ~/.hdt-env for later use
```

After that, **close Hearthstone, Battle.net and Steam**.

---

## Step 2. Install dependencies

Make sure **Steam, Battle.net and Hearthstone are closed**. Run the commands one by one. This installs several .NET versions and will take a while.

If you opened a new terminal since Step 1, load the saved paths first:

```bash
source ~/.hdt-env  # load HSPFX and PROTON_WINE saved in Step 1
```

If you don't have winetricks yet:

```bash
sudo apt install winetricks  # install the winetricks helper tool
```

Then:

```bash
WINEPREFIX="$HSPFX" winetricks dotnet472  # install .NET Framework 4.7.2 into the Hearthstone prefix
```

```bash
WINEPREFIX="$HSPFX" winetricks corefonts dotnet48  # install Microsoft core fonts and .NET Framework 4.8
```

The install breaks a setting. Fix it:

```bash
WINEPREFIX="$HSPFX" "$PROTON_WINE" winecfg /v win10  # set the prefix back to Windows 10 mode
```

```bash
WINEPREFIX="$HSPFX" winetricks msdelta  # install msdelta, needed for HDT's auto-updater
```

---

## Step 3. Download and unpack HDT

Download the latest HDT from https://hsdecktracker.net/download/ and keep `HDT-Installer.exe` in your **Downloads** folder.

Then run the commands one by one **in the same terminal** (later commands use the `HDTVER` variable set by the second one):

```bash
cd ~/Downloads && rm -rf HDT-res && 7z x -y -oHDT-res HDT-Installer.exe | tail -5  # extract the installer's contents into ~/Downloads/HDT-res
```

```bash
HDTVER=$(ls ~/Downloads/HDT-res/*-full.nupkg | sed -E 's/.*HearthstoneDeckTracker-(.+)-full\.nupkg/\1/'); echo "$HDTVER"  # detect the HDT version number from the package name
```

```bash
rm -rf ~/Games/HDT && mkdir -p ~/Games/HDT/packages ~/Games/HDT/app-$HDTVER && cp ~/Downloads/HDT-res/*-full.nupkg ~/Downloads/HDT-res/RELEASES ~/Games/HDT/packages/ && cp ~/Downloads/HDT-res/Update.exe ~/Games/HDT/  # create the HDT folder layout and copy the package, release list and updater
```

```bash
rm -rf ~/Downloads/HDT-app && unzip -q ~/Games/HDT/packages/*.nupkg 'lib/*' -d ~/Downloads/HDT-app && mv ~/Downloads/HDT-app/lib/net472/* ~/Games/HDT/app-$HDTVER/  # unpack the HDT program files into the versioned app folder
```

```bash
cp ~/Games/HDT/app-$HDTVER/HearthstoneDeckTracker_ExecutionStub.exe ~/Games/HDT/HearthstoneDeckTracker.exe  # place the launcher stub that always starts the newest installed version
```

---

## Step 4. Create a launcher script

Run this as a single command (copy the whole block):

```bash
mkdir -p ~/Games/HDT  # make sure the HDT folder exists
cat > ~/Games/HDT/launch_hdt_gui.sh << 'EOF'
#!/bin/bash

HDT_DIR="$HOME/Games/HDT"
source "$HOME/.hdt-env"

export WINEPREFIX="$HSPFX"
export WINEFSYNC=1   # match the game's thread-sync method (prevents a Bob's Buddy crash)
cd "$HDT_DIR"

setsid "$PROTON_WINE" "HearthstoneDeckTracker.exe" >/dev/null 2>&1 &
disown
EOF
chmod +x ~/Games/HDT/launch_hdt_gui.sh  # make the launcher script executable
echo "file://$HOME/Games/HDT/launch_hdt_gui.sh"  # print the script's location
```

---

## Step 5. Create a clickable desktop icon

```bash
cat > ~/Desktop/Hearthstone_Deck_Tracker.desktop << EOF
[Desktop Entry]
Version=1.0
Type=Application
Name=Hearthstone Deck Tracker
Exec=$HOME/Games/HDT/launch_hdt_gui.sh
Icon=input-gaming
Terminal=false
StartupNotify=true
EOF
chmod +x ~/Desktop/Hearthstone_Deck_Tracker.desktop  # make the desktop shortcut executable
```

Once the file is created: **Right click → Permissions → Allow executing file as program**.

---

## Step 6 (optional). Use HDT's real icon instead of the generic one

```bash
sudo apt install -y icoutils  # install tools for extracting images from .ico files
mkdir -p ~/.local/share/icons  # create the user icon folder
icotool -x "$HOME/Games/HDT/app-"*/Images/HearthstoneDeckTracker.ico -o ~/.local/share/icons  # extract all sizes from HDT's icon
LARGEST_ICON=$(ls -S ~/.local/share/icons/HearthstoneDeckTracker_*.png | head -n 1)  # pick the largest extracted image
cp "$LARGEST_ICON" ~/.local/share/icons/hdt.png  # save it as hdt.png
rm -f ~/.local/share/icons/HearthstoneDeckTracker_*.png  # delete the leftover sizes
sed -i "s|^Icon=.*|Icon=$HOME/.local/share/icons/hdt.png|" ~/Desktop/Hearthstone_Deck_Tracker.desktop  # point the desktop shortcut at the new icon
```
