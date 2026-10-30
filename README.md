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



# MetaTrader 5 Python Bridge (`mt5linux`) Setup & Architecture Guide

This guide details how to establish a robust RPyC bridge between a native **Linux Python environment** and a **Windows MetaTrader 5 terminal** running under Wine.

---

## 1. How the Architecture Works

Because MetaTrader 5 relies on Windows C++ dynamic link libraries (`.dll`), Linux-native Python cannot load the official `MetaTrader5` package directly. 

The `mt5linux` package solves this using an **RPyC (Remote Python Call)** client-server architecture over TCP port `22346`:

```
┌──────────────────────────────────────┐       TCP Port 22346        ┌──────────────────────────────────────┐
│       Linux Python Environment       │  ───────────────►─────────  │         Wine Prefix (~/.wine-mt5)    │
│                                      │                             │                                      │
│  Script: extarct_yearly_1m_data.py   │                             │   RPyC Server (mt5server.exe)        │
│  Library: mt5linux                   │                             │               │                      │
└──────────────────────────────────────┘                             │               ▼                      │
                                                                     │     MetaTrader 5 (terminal64.exe)   │
                                                                     └──────────────────────────────────────┘
```

1. **Client (Linux Side):** Runs your trading/extraction scripts using native Linux Python.
2. **Server (Wine Side):** Runs `mt5server.exe` (or Windows Python with `mt5linux`), listening on port `22346` to translate commands into native MT5 Windows API calls.

---

## 2. Server Setup Methods

### Method A: Standalone `mt5server.exe` (Recommended)

The standalone `mt5server.exe` binary bundles Windows Python, RPyC, and the MetaTrader5 library into a single executable—no Python installation inside Wine is required.

#### Step 1: Download the Server Binary
Download `mt5server.exe` from the official [lucas-campagna/mt5linux GitHub Releases](https://github.com/lucas-campagna/mt5linux/releases) into your `~/Downloads` folder.

#### Step 2: Place inside the Wine Prefix Drive C
Move the executable directly into the root directory of your Wine prefix (`drive_c`):

```bash
cp ~/Downloads/mt5server.exe ~/.wine-mt5/drive_c/mt5server.exe
```

---

### Method B: Installing Python inside Wine (Alternative)

If you prefer to run the server directly through Python in Wine:

1. Download the Windows Python installer (64-bit):
   ```bash
   wget https://www.python.org/ftp/python/3.10.11/python-3.10.11-amd64.exe -O ~/Downloads/python-3.10.11-amd64.exe
   ```
2. Install Python inside your prefix:
   ```bash
   export WINEPREFIX="$HOME/.wine-mt5"
   export WINEARCH="win64"
   wine ~/Downloads/python-3.10.11-amd64.exe
   ```
   *(Check **"Add Python.exe to PATH"** during setup).*

3. Install requirements inside Wine:
   ```bash
   wine cmd /c "python -m pip install --upgrade pip"
   wine cmd /c "python -m pip install MetaTrader5 rpyc mt5linux"
   ```

---

## 3. Automation Script (`start_mt5_bridge.sh`)

To launch both MetaTrader 5 and the bridge server cleanly from any location on your system, create a single shell script:

```bash
nano ~/Desktop/start_mt5_bridge.sh
```

Paste the following code:

```bash
#!/bin/bash

# Define Wine prefix environment
export WINEPREFIX="$HOME/.wine-mt5"
export WINEARCH="win64"

echo "[*] Cleaning up existing Wine server instances..."
wineserver -k

echo "[*] Launching MetaTrader 5 terminal..."
wine "C:\Program Files\MetaTrader 5\terminal64.exe" &

echo "[*] Waiting 6 seconds for MT5 terminal startup..."
sleep 6

echo "[*] Launching mt5server bridge on port 22346..."
wine "C:\mt5server.exe" --port 22346 &

echo "[*] Bridge active and ready for Linux Python connections!"
```

Make the script executable:

```bash
chmod +x ~/Desktop/start_mt5_bridge.sh
```

---

## 4. Client-Side Python Integration

In your Linux Python script (`extarct_yearly_1m_data.py`), initialize the connection using `mt5linux`:

```python
from mt5linux import MetaTrader5

MT5_HOST = "127.0.0.1"
MT5_PORT = 22346

# Initialize client connection to the Wine server bridge
mt5 = MetaTrader5(host=MT5_HOST, port=MT5_PORT)

if not mt5.initialize():
    print("[-] MT5 initialization failed:", mt5.last_error())
else:
    print("[+] Successfully connected to MetaTrader 5 via mt5linux bridge!")
```

---

## 5. Troubleshooting & Common Errors

### `EOFError: connection closed by peer`
* **Cause:** The Linux client tried to connect to port `22346`, but the Wine server dropped the connection.
* **Fixes:**
  1. Confirm `mt5server.exe` is running inside Wine.
  2. Verify that MetaTrader 5 is fully open and logged in prior to running your Python script.
  3. Ensure `rpyc` versions on both the Linux side and Wine side are compatible.

### Port Already in Use
If port `22346` is blocked by a hung process, terminate background Wine sessions before retrying:
```bash
wineserver -k
```


# MetaTrader 5 Python Bridge (`mt5linux`) Master Blueprint

Here is the complete, master blueprint for setting up the Linux-to-Wine MetaTrader 5 bridge from scratch. This method represents the most stable and robust approach, designed to avoid version mismatches and port conflict traps.

---

## Understanding the Architecture

MetaTrader 5 relies entirely on Windows C++ dynamic link libraries (`.dll` files). Because Linux uses `.so` shared objects, your native Linux Python cannot load the `MetaTrader5` library directly.

The `mt5linux` bridge solves this using **RPyC (Remote Python Call)** to bridge the gap over a local TCP network socket:

```
[ Linux Python Script (Client) ] 
       │ 
       │  (Sends commands via TCP Port 22346)
       ▼
[ Windows Python inside Wine (Server running mt5linux) ]
       │ 
       │  (Calls native Windows DLLs)
       ▼
[ MetaTrader 5 Terminal ]
```

---

## The Golden Rule for a Stable Setup

> **Always synchronize your `rpyc` package versions.**
>
> The most common point of failure is a protocol mismatch between your Linux client environment and your Wine server environment (e.g., `ValueError: invalid message type`). Both sides **must** run the exact same `rpyc` version.

---

## Step-by-Step Setup Guide (Portable Blueprint)

### Step 1: Install Windows Python inside Your Wine Prefix

Do not rely on pre-compiled standalone server binaries (`.exe`), as they lock you into rigid dependency versions. Instead, install native Windows Python inside your target Wine prefix.

1. **Set your Wine environment variables:**
   ```bash
   export WINEPREFIX="$HOME/.wine-mt5"
   export WINEARCH="win64"
   ```

2. **Download and run the official Windows 64-bit Python installer (e.g., Python 3.10) inside Wine:**
   ```bash
   wine python-3.10.11-amd64.exe
   ```
   * **Critical:** Check the box during installation to **"Add Python.exe to PATH"**.

---

### Step 2: Synchronize Package Versions (Linux & Wine)

1. **Check what version of `rpyc` your Linux Python environment is running:**
   ```bash
   pip show rpyc
   ```
   *(Suppose it returns version `5.2.3`).*

2. **Open a Windows command prompt inside Wine and install that exact matching version along with `mt5linux`:**
   ```bash
   export WINEPREFIX="$HOME/.wine-mt5"
   wine cmd
   ```

3. **Inside the Windows command window, run:**
   ```cmd
   python -m pip install --upgrade pip
   python -m pip install rpyc==5.2.3 mt5linux MetaTrader5
   exit
   ```

---

### Step 3: Implement a Clean Launch Sequence (Avoid Port Conflicts)

Port `22346` is prone to staying locked in the background if a previous process crashes (`WinError 10048`). Always purge stale processes before starting the bridge.

Create a master startup script (e.g., `start_bridge.sh`):

```bash
#!/bin/bash

export WINEPREFIX="$HOME/.wine-mt5"
export WINEARCH="win64"

echo "[*] Purging stale background processes..."
wineserver -k
pkill -9 -f wine
pkill -9 -f mt5linux

echo "[*] Launching MetaTrader 5 Terminal..."
wine "C:\Program Files\MetaTrader 5\terminal64.exe" &

echo "[*] Waiting for MT5 to initialize..."
sleep 8

echo "[*] Launching mt5linux Bridge Server on port 22346..."
wine python -m mt5linux --port 22346 &

echo "[*] Bridge is active and ready!"
```

---

### Step 4: Connect Your Client Script

In your Python extraction or trading script on the Linux side, initialize the client pointing to `localhost` on your designated port:

```python
from mt5linux import MetaTrader5 as mt5

# Connects securely to the Wine server running on port 22346
client = mt5(host="127.0.0.1", port=22346)

if client.initialize():
    print("[*] Successfully connected to MT5 via Wine bridge!")
```

---

## Why This Method Wins

1. **Full Compatibility:** Because you use native Windows Python inside Wine, `MetaTrader5` interacts with its native Windows DLL environment without wrapper translation issues.
2. **Version Control:** Pinning matching `rpyc` versions (`rpyc==5.2.3`) completely eliminates protocol handshake errors.
3. **Resilience:** Explicitly killing `wineserver` and orphaned processes in your launch script guarantees you will never hit `WinError 10048` address-in-use blocks when rebooting your environment.
