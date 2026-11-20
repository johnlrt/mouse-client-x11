<div align="center">

# Universal Wireless Mouse Battery Tray (`mouse-client-x11`)

**Ultra-lightweight (~0.31 MB RAM, 107 KB) native Win32 C Windows taskbar battery monitor for wireless 2.4 GHz, USB & Bluetooth mice — supporting Logitech, Razer, Compx/VXE/ATK/VGN/Darmoshark/Attack Shark, Standard HID Battery, and BLE GATT.**

> Also includes an optional clean-room `.NET 8` WPF Control Center and HID protocol spec for Attack Shark X11.

[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%2F%2011-0078D4?style=flat-square&logo=windows)](https://github.com/johnlrt/mouse-client-x11)
[![Native Tray](https://img.shields.io/badge/Tray%20Binary-Win32%20C%20%C2%B7%20107%20KB-E2211C?style=flat-square&logo=c)](src/X11BatteryTrayNative/main.c)
[![Memory](https://img.shields.io/badge/Private%20RAM-~0.31%20MB-2EA043?style=flat-square)](src/X11BatteryTrayNative/mouse_manager.c)
[![Control Center](https://img.shields.io/badge/Control%20Center-.NET%208%20WPF-512BD4?style=flat-square&logo=dotnet)](src/X11Mouse.App/X11Mouse.App.csproj)
[![Tests](https://img.shields.io/badge/Tests-19%2F19%20Unit%20%2B%205%2F5%20Protocol-2EA043?style=flat-square)](tests/X11Mouse.Tests/X11Mouse.Tests.csproj)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)

**English** | [Русский](README.ru.md) | [简体中文](README.zh-CN.md)

</div>

---

## Visual Showcase

<table>
  <tr>
    <td align="center" width="50%">
      <img src="docs/assets/tray-flyout.png" alt="Native Dark Flyout Popup" />
      <br />
      <sub><b>Native GDI Dark Flyout</b> — shows active mouse, transport badge, live percentage, backend telemetry, pin shortcut, and manual refresh.</sub>
    </td>
    <td align="center" width="50%">
      <img src="docs/assets/tray-icons.png" alt="Pixel-Snapped Tray Icons (20px and 16px)" />
      <br />
      <sub><b>Pixel-Snapped 32-bit DIB Tray Icons</b> — 3 icon styles across <code>20×20</code> (125% DPI) and <code>16×16</code> (100% DPI): Digits + Bar, Battery Gauge, and Large Digits.</sub>
    </td>
  </tr>
</table>

---

## Why This Exists & Live RAM Benchmark

Official mouse software often stays heavy in memory or forces OEM background services. Here, the tray battery monitor is written in **pure Win32 C** with zero `.NET CLR`, zero `WinForms`, zero `GDI+`, pixel-aligned `CreateDIBSection` icons, and a **Single-Active-Backend** engine (`107 KB` single `.exe`):

| Metric (Measured Live on Windows 11) | `.NET 8 WinForms` Version | `Native Win32 C` Version | Improvement |
| --- | --- | --- | --- |
| **Task Manager (`Private Working Set`)** | `11.2 MB` | **`0.31 MB` (`320 KB`)** | **~36× smaller** |
| **Total Working Set** | `74.48 MB` | **`0.98 MB`** | **~76× smaller** |
| **Private Bytes** | `16.75 MB` | **`1.68 MB`** | **~10× smaller** |
| **Handles / Threads** | `491` / `16` | **`127` / `5`** | **~3.9× / 3.2× fewer** |
| **Loaded Bluetooth DLL on 2.4G Mouse** | Runtime-dependent | **`False` (`BluetoothAPIs.dll` not loaded)** | **Lazy fallback only** |
| **Executable Footprint** | Multi-file .NET output | **`107.0 KB` standalone `.exe`** | **Zero runtime dependencies** |

---

## Single-Active-Backend Multi-Mouse Engine

Instead of running poll loops for every protocol at once, [`mouse_manager.c`](src/X11BatteryTrayNative/mouse_manager.c) performs **one** fast read-only discovery pass on startup or hot-plug (`WM_DEVICECHANGE`), binds **only the single highest-priority backend**, and runs **one** 64 KB-stack worker thread:

| Priority | Backend | Devices / VID:PID | Transport & Strategy |
| :---: | --- | --- | --- |
| `100` / `90` | **Attack Shark X11 Passive** | `VID_1D57` (`PID_FA55` Wired, `PID_FA60` 2.4G) | Strictly read-only `UsagePage=0x000A` Report `0x03` listener (preserves `%` across 6s sleep) |
| `80` | **Compx / CX52850 17-Byte** | `VID_3554` / `248A` / `25A7` (`FF02:0002`, 17B `0x08/0x04`) | VXE, ATK, VGN, Kysona, Darmoshark, Attack Shark X3 — query + mV telemetry every `60 s` |
| `76` / `74` | **Logitech HID++ 2.0** | `VID_046D` (`FF00:0002` / `0001`) | Lightspeed / Bolt / Unifying — slots `0xFF, 1..6`, mouse check (`0x0005`), `0x1004` → `0x1000` → `0x1001` |
| `72` / `68` | **Razer 90-Byte Feature** | `VID_1532` (`Feat=91`) | Viper, DeathAdder, Basilisk, Orochi, Naga, Cobra — `0x07/0x80` & `0x07/0x84` XOR CRC every `60 s` |
| `50` | **Standard HID Battery** | Value Caps `0x06/0x20`, `0x85/0x65–0x66` | Generic HID battery mice (UPS `0x84/0x04`, keyboards & headsets excluded) |
| `35` / `30` | **Bluetooth LE GATT** | Services `0x180F` + `0x1812` + GAP Appearance | Lazy-loads `BluetoothAPIs.dll` only when no USB/2.4G mouse is detected |

---

## Quick Start

### Option A: Prebuilt Tray Monitor
1. Download **`X11BatteryTray.exe`** from [**GitHub Releases**](https://github.com/johnlrt/mouse-client-x11/releases).
2. Run it — no installer or `.NET` runtime required for the native tray utility.
3. Left-click the tray icon for the dark popup flyout, or right-click to switch icon style, enable Windows autostart (`HKCU\...\Run`), or trigger a rescan.

> **Windows 11 Tip:** If the tray icon starts inside the `^` overflow chevron, drag it once onto the taskbar next to the Windows battery/sound icons (or click **«Закрепить на панели задач Windows ->»** inside the flyout).

### Option B: Build from Source

```powershell
# 1. Build the ultra-lightweight native Win32 C tray binary -> .\bin\X11BatteryTray.exe
#    Uses .tools\w64devkit\bin\gcc.exe if present, or gcc from PATH (MinGW-w64 / w64devkit)
& ".\src\X11BatteryTrayNative\build.ps1"

# 2. Or simply double-click Start-BatteryTray.bat (builds automatically if needed and launches)
.\Start-BatteryTray.bat

# Or launch directly
& ".\bin\X11BatteryTray.exe"

# 3. Build the clean-room .NET 8 WPF Control Center & run xUnit tests
dotnet test X11Mouse.sln -c Release
& ".\src\X11Mouse.App\bin\Release\net8.0-windows\X11Mouse.exe"
```

---

## CLI Reference (`X11BatteryTray.exe`)

| Flag | Behavior |
| --- | --- |
| `--status` | Probes the winning mouse once and prints mode, backend, VID/PID, product name, and battery/voltage |
| `--scan` | Lists all detected mouse/battery HID & BLE interfaces read-only alongside the selected backend winner |
| `--self-test` | Runs built-in protocol parser, checksum/CRC, scaling, and backend priority self-tests |
| `--smoke-test` | Renders icons across all styles/themes/modes and opens/updates/closes the GDI flyout headlessly |
| `--preview <path.bmp>` | Exports a tiled BMP grid comparing all 3 icon styles across `20×20` and `16×16` sizes |

---

## Bonus: Clean-Room Control Center & Protocol Spec (Attack Shark X11)

- **[X11Mouse.App](src/X11Mouse.App/X11Mouse.App.csproj)** (`.NET 8` WPF) & **[X11Mouse.Core](src/X11Mouse.Core/X11Mouse.Core.csproj)**:
  - Full interactive Attack Shark X11 control suite: vector top-down mouse diagram with bidirectional button highlighting, 8 DPI stages (`220`-entry register table), radial polling selector (`125`–`1000 Hz`), lighting/sleep/debounce settings, 18-slot button remapping (`151`-entry `Hook.dll` VK/scan/HID table), `WH_KEYBOARD_LL` / `WH_MOUSE_LL` macro recorder, legacy `pro.data` / `.profile` / `macro.data` / `.macro` binary codec, and `--probe` / `--smoke-test`.
- **[X11Mouse.BatteryTray](src/X11Mouse.BatteryTray/X11Mouse.BatteryTray.csproj)**:
  - Original `.NET 8` WinForms reference implementation kept in the solution alongside **19** automated tests in **[X11Mouse.Tests](tests/X11Mouse.Tests/X11Mouse.Tests.csproj)**.
- **[Protocol & Binary Format Specification](rea-analysis/protocol-spec.md)**:
  - Complete documentation of Attack Shark X11 HID endpoints (`0x1D57:0xFA55` Wired, `0xFA60` 2.4G), async Report `0x03` notifications, Feature write packets (`0x04`–`0x0C`), checksum framing, and profile/macro serialization formats.

---

## Safety & Non-Intrusiveness Guarantees

- **Shared HID Access:** Opens endpoints with `FILE_SHARE_READ | FILE_SHARE_WRITE` so tools coexist safely alongside official vendor drivers without killing processes or claiming exclusive locks.
- **Zero Official File Mutation:** Settings are isolated in `%APPDATA%\AttackSharkX11Clean` and `HKCU\Software\AttackSharkX11Clean\Tray`. `C:\Program Files (x86)\X11 Mouse` and `%APPDATA%\X11 Mouse` are never modified.
- **No Blind Hardware Writes:** `--scan`, `--probe`, and X11 `--status` are strictly read-only; vendor query frames (`Compx`, `HID++`, `Razer`) are sent only after positively matching the corresponding mouse protocol.

---

## Repository Structure

```text
├── src/
│   ├── X11BatteryTrayNative/   # Pure Win32 C multi-mouse tray monitor (107 KB binary)
│   ├── X11Mouse.Core/          # Clean-room X11 protocol, HID transport & storage library
│   ├── X11Mouse.App/           # .NET 8 WPF Control Center UI
│   └── X11Mouse.BatteryTray/   # Legacy .NET 8 WinForms tray monitor (reference)
├── tests/
│   └── X11Mouse.Tests/         # 19 xUnit protocol, storage & mocked device service tests
├── rea-analysis/
│   ├── protocol-spec.md        # Reverse-engineered X11 protocol & binary format spec
│   └── hook-table-151.json     # Extracted 151-entry keycode table
├── docs/assets/                # README UI & icon screenshots
├── Start-BatteryTray.bat       # One-click launcher (auto-builds if needed)
└── X11Mouse.sln                # Visual Studio / dotnet solution
```

---

## License

Licensed under the [MIT License](LICENSE).
