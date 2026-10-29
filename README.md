# MT5-Setup-on-Xubuntu26.04.1
How to fix the terminal64.exe debugger error


# MetaTrader 5 Wine-Staging Master Setup Guide (Xubuntu / Ubuntu)

This reference playbook provides step-by-step instructions to automate and troubleshoot the installation of MetaTrader 5 on modern Ubuntu/Xubuntu 26.04.1 LTS distributions using `wine-staging` to bypass debugger flags.

---

## Phase 1: Enable 32-Bit Multiarch Support
Wine requires 32-bit libraries to handle legacy Windows components even inside a 64-bit prefix.

```bash
sudo dpkg --add-architecture i386
sudo apt update
```

---

## Phase 2: Install Wine-Staging & Resolve Dependency Conflicts
If package manager version locks or library mismatches occur between native system packages and `wine-staging`, use this clean-slate approach:

1. **Force-purge any stuck or partially installed Wine components:**
   ```bash
   sudo dpkg --purge --force-depends wine-staging wine-staging-i386 wine-staging-amd64 winehq-staging
   sudo apt-get clean
   sudo apt --fix-broken install -y
   ```

2. **Install Wine-Staging cleanly:**
   ```bash
   sudo apt update
   sudo apt install --install-recommends winehq-staging wine-staging-amd64 -y
   ```

---

## Phase 3: Force a Clean 64-Bit Prefix Initialization
Wine defaults to a 32-bit prefix if background processes are active. Always kill active servers and wipe corrupt folders before starting fresh.

1. **Kill any active background Wine servers:**
   ```bash
   wineserver -k
   ```

2. **Remove any half-initialized prefix folders:**
   ```bash
   rm -rf ~/.wine-mt5
   ```

3. **Initialize a fresh 64-bit prefix explicitly:**
   ```bash
   export WINEPREFIX="$HOME/.wine-mt5"
   export WINEARCH="win64"
   wineboot --init
   ```

---

## Phase 4: Launching and Configuring the Installer

1. **Execute your MetaTrader 5 setup file** within the configured environment:
   ```bash
   export WINEPREFIX="$HOME/.wine-mt5"
   export WINEARCH="win64"
   wine ~/Downloads/mt5setup.exe
   ```
   *(Note: Replace `~/Downloads/mt5setup.exe` with your exact file path or wrap it in quotes if it contains spaces, e.g., `wine "$HOME/Downloads/mt5setup (1).exe"`)*

2. **Handle GUI Setup Prompts:**
   * Click **Install** when the **Wine Mono Installer** pop-up appears (handles .NET dependencies).
   * Click **Install** if a **Wine Gecko** prompt appears (handles HTML rendering).

Once completed, the official MetaTrader 5 graphical setup wizard will open cleanly, allowing you to proceed with your automated data extraction setup!
