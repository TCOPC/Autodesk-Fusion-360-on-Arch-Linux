# 🛠️ Autodesk Fusion 360 on Arch Linux (via Wine)

> A clean, tested installation guide for running Fusion 360 on Arch-based Linux distributions (EndeavourOS, Manjaro, etc.) using Wine + DXVK.
>
> Tested on: **EndeavourOS** | Wine **11.6** | NVIDIA RTX 5060 Ti | Kernel **6.19**

---

## ⚠️ Requirements

- Arch-based Linux distribution (EndeavourOS, Manjaro, Arch, etc.)
- **Real GPU with Vulkan support** (NVIDIA, AMD, or Intel — no VMs!)
- `yay` or another AUR helper installed
- Internet connection

---

## 📦 Step 1 — Install Dependencies

```bash
sudo pacman -S p7zip curl wget cabextract wine wine-mono wine-gecko winetricks samba
```

> **Note:** `p7zip` covers `p7zip-full` and `p7zip-rar` from Debian/Ubuntu. `winbind` is part of the `samba` package on Arch.

---

## 🧰 Step 2 — Install .NET Framework via Winetricks

```bash
winetricks dotnet45
```

This will take a few minutes. Accept any dialogs that appear.

---

## 📥 Step 3 — Download the Fusion 360 Installer

```bash
wget "https://dl.appstreaming.autodesk.com/production/installers/Fusion%20Client%20Downloader.exe" -O ~/Fusion360installer.exe
```

---

## 🍷 Step 4 — Create a Clean 64-bit Wine Prefix

```bash
WINEPREFIX=~/.fusion360 WINEARCH=win64 wineboot --init
```

Warnings in the output are normal. Wait until the prompt returns.

---

## ⚙️ Step 5 — Install Winetricks Dependencies into the Prefix

```bash
WINEPREFIX=~/.fusion360 winetricks -q corefonts vcrun2017 msxml4 msxml6 dotnet45
```

This will take several minutes. Let it run to completion.

---

## 🪟 Step 6 — Set Wine to Windows 11 Mode

Fusion 360 will refuse to install on older Windows versions:

```bash
WINEPREFIX=~/.fusion360 winecfg
```

In the GUI that opens:
1. Go to the **Applications** tab
2. Change **Windows Version** to **Windows 11**
3. Click **Apply**, then **OK**

---

## 🚀 Step 7 — Install DXVK

DXVK replaces Wine's D3D11 with a Vulkan-based implementation, required for Fusion 360:

```bash
yay -S dxvk-bin
WINEPREFIX=~/.fusion360 setup_dxvk install
```

---

## 🏗️ Step 8 — Run the Fusion 360 Installer

```bash
WINEPREFIX=~/.fusion360 wine ~/Fusion360installer.exe
```

> This can take 30–60+ minutes. Fusion downloads itself (~2GB). The installer window closes on its own when done. In some cases Fusion launches automatically after installation.

---

## 🔑 Step 9 — Set Up the Login Protocol Handler

After installation, the `adskidmgr://` OAuth callback must be handled correctly. Without this, login will not work.

**Registry fix:**
```bash
HASH=$(ls ~/.fusion360/drive_c/users/$USER/AppData/Local/Autodesk/webdeploy/production/ | tail -1)

cat > ~/fusion360-adskidmgr.reg << EOF
Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\adskidmgr]
@="URL:adskidmgr Protocol"
"URL Protocol"=""

[HKEY_CLASSES_ROOT\adskidmgr\shell]

[HKEY_CLASSES_ROOT\adskidmgr\shell\open]

[HKEY_CLASSES_ROOT\adskidmgr\shell\open\command]
@="\"C:\\\\users\\\\$USER\\\\AppData\\\\Local\\\\Autodesk\\\\webdeploy\\\\production\\\\$HASH\\\\Autodesk Identity Manager\\\\AdskIdentityManager.exe\" \"%1\""
EOF

WINEPREFIX=~/.fusion360 wine regedit ~/fusion360-adskidmgr.reg
```

**Desktop protocol handler (for the Linux browser):**
```bash
HASH=$(ls ~/.fusion360/drive_c/users/$USER/AppData/Local/Autodesk/webdeploy/production/ | tail -1)

cat > ~/.local/share/applications/adskidmgr.desktop << EOF
[Desktop Entry]
Name=Autodesk ID Manager
Exec=env WINEPREFIX=$HOME/.fusion360 wine "$HOME/.fusion360/drive_c/users/$USER/AppData/Local/Autodesk/webdeploy/production/$HASH/Autodesk Identity Manager/AdskIdentityManager.exe" "%u"
Type=Application
MimeType=x-scheme-handler/adskidmgr;
NoDisplay=true
EOF

update-desktop-database ~/.local/share/applications
xdg-mime default adskidmgr.desktop x-scheme-handler/adskidmgr
```

---

## ▶️ Step 10 — Launch Fusion 360

```bash
WINEPREFIX=~/.fusion360 wine ~/.fusion360/drive_c/users/$USER/AppData/Local/Autodesk/webdeploy/production/$(ls ~/.fusion360/drive_c/users/$USER/AppData/Local/Autodesk/webdeploy/production/ | tail -1)/Fusion360.exe
```

Log in via the browser that opens. After logging in, the browser will return to Fusion.

---

## 🖥️ Step 11 — Set Graphics to OpenGL (Required!)

After the first login there are **two** graphics settings that must be set to OpenGL. Without this, Fusion will crash while loading the 3D workspace, or the sidebar/menu will be white and unusable.

**Setting 1 — Graphics Driver:**
1. Open **Preferences** (top right → your name → Preferences)
2. Go to **General**
3. Set **Graphics Driver** to **OpenGL**
4. Click **Apply** and restart Fusion

**Setting 2 — Qt Rendering Interface:**
1. Open **Preferences** again after restart
2. Go to **Compatibility & Troubleshooting**
3. Set **Qt Rendering Hardware Interface API** to **OpenGL**
4. Click **Apply** and restart Fusion again

> Both settings are required. Setting only the first one fixes the crash but leaves the sidebar white. Set both to OpenGL.

---

## 🖥️ Optional — Create a Desktop Launcher

```bash
HASH=$(ls ~/.fusion360/drive_c/users/$USER/AppData/Local/Autodesk/webdeploy/production/ | tail -1)

cat > ~/.local/share/applications/fusion360.desktop << EOF
[Desktop Entry]
Name=Autodesk Fusion 360
Exec=env WINEPREFIX=$HOME/.fusion360 wine "$HOME/.fusion360/drive_c/users/$USER/AppData/Local/Autodesk/webdeploy/production/$HASH/Fusion360.exe"
Type=Application
Categories=Graphics;Engineering;
StartupNotify=true
EOF
```

---

## 🐛 Troubleshooting

| Problem | Solution |
|---|---|
| `apt: command not found` during install | You're using the wrong installer script — follow this guide instead |
| Fusion crashes immediately on launch | Make sure you're on real hardware, not a VM |
| Fusion crashes while loading the 3D workspace | Set Graphics Driver to OpenGL (Step 11, setting 1) |
| White sidebar / menu not visible | Also set Qt Rendering API to OpenGL (Step 11, setting 2) |
| Browser opens but login doesn't complete | Check that the desktop protocol handler is set up correctly (Step 9) |
| "OS no longer supported" error | Make sure Wine is set to Windows 11 in `winecfg` (Step 6) |
| Installer takes forever | Normal — Fusion downloads ~2GB. Let it run |
| Login/SSO not working | Try disabling your VPN if active |

---

## 📝 Notes

- Do **not** use VirtualBox or other VMs — Fusion 360 requires real GPU access
- The `cryinkfly` installer script (`fusion360-installer.sh`) is Debian-based and will fail on Arch
- Wine prefix is stored in `~/.fusion360` — kept separate from the default `~/.wine`
- Fusion 360 updates itself automatically through the launcher
- NVIDIA users: make sure Secure Boot is disabled in the BIOS for DXVK to work

---

## 🙏 Credits

- Tested and documented on EndeavourOS through trial and error
- OpenGL fix sourced from [cryinkfly issue #571](https://github.com/cryinkfly/Autodesk-Fusion-360-for-Linux/issues/571)
- DXVK: [https://github.com/doitsujin/dxvk](https://github.com/doitsujin/dxvk)
- Wine: [https://www.winehq.org](https://www.winehq.org)
- Original installer reference: [cryinkfly/Autodesk-Fusion-360-for-Linux](https://github.com/cryinkfly/Autodesk-Fusion-360-for-Linux)

---

## 📄 License

MIT — feel free to share, fork, and improve!
