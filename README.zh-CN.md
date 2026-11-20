<div align="center">

# 通用无线鼠标电量托盘工具 (`mouse-client-x11`)

**超轻量（约 0.31 MB 内存、107 KB）原生 Win32 C 开发的 Windows 任务栏电量监控工具，适用于 2.4 GHz、USB 和蓝牙无线鼠标 — 支持 Logitech、Razer、Compx/VXE/ATK/VGN/Darmoshark/Attack Shark、标准 HID 电池以及 BLE GATT。**

> 同时附带可选的洁净室（clean-room）`.NET 8` WPF 控制中心与 Attack Shark X11 HID 协议规范。

[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%2F%2011-0078D4?style=flat-square&logo=windows)](https://github.com/johnlrt/mouse-client-x11)
[![Native Tray](https://img.shields.io/badge/Tray%20Binary-Win32%20C%20%C2%B7%20107%20KB-E2211C?style=flat-square&logo=c)](src/X11BatteryTrayNative/main.c)
[![Memory](https://img.shields.io/badge/Private%20RAM-~0.31%20MB-2EA043?style=flat-square)](src/X11BatteryTrayNative/mouse_manager.c)
[![Control Center](https://img.shields.io/badge/Control%20Center-.NET%208%20WPF-512BD4?style=flat-square&logo=dotnet)](src/X11Mouse.App/X11Mouse.App.csproj)
[![Tests](https://img.shields.io/badge/Tests-19%2F19%20Unit%20%2B%205%2F5%20Protocol-2EA043?style=flat-square)](tests/X11Mouse.Tests/X11Mouse.Tests.csproj)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)

[English](README.md) | [Русский](README.ru.md) | **简体中文**

</div>

---

## 界面展示

<table>
  <tr>
    <td align="center" width="50%">
      <img src="docs/assets/tray-flyout.png" alt="原生深色弹出面板" />
      <br />
      <sub><b>原生 GDI 深色弹出面板</b> — 显示当前活动鼠标、连接方式徽标、实时电量百分比、后端遥测信息、固定快捷操作以及手动刷新。</sub>
    </td>
    <td align="center" width="50%">
      <img src="docs/assets/tray-icons.png" alt="像素对齐的托盘图标（20px 与 16px）" />
      <br />
      <sub><b>像素对齐的 32 位 DIB 托盘图标</b> — 在 <code>20×20</code>（125% DPI）和 <code>16×16</code>（100% DPI）尺寸下提供 3 种图标样式：数字 + 电量条、电池仪表盘和大号数字。</sub>
    </td>
  </tr>
</table>

---

## 设计初衷与实时内存基准测试

官方鼠标软件往往长期占用大量内存，或强制运行厂商后台服务。本项目的托盘电量监控器采用**纯 Win32 C** 编写，不依赖 `.NET CLR`、`WinForms` 和 `GDI+`，使用像素对齐的 `CreateDIBSection` 图标以及**单活动后端（Single-Active-Backend）**引擎（单个 `.exe` 仅 `107 KB`）：

| 指标（在 Windows 11 上实时测量） | `.NET 8 WinForms` 版本 | `Native Win32 C` 原生版本 | 优化幅度 |
| --- | --- | --- | --- |
| **任务管理器（`Private Working Set`）** | `11.2 MB` | **`0.31 MB`（`320 KB`）** | **约小 36 倍** |
| **总工作集（`Total Working Set`）** | `74.48 MB` | **`0.98 MB`** | **约小 76 倍** |
| **专用字节（`Private Bytes`）** | `16.75 MB` | **`1.68 MB`** | **约小 10 倍** |
| **句柄数 / 线程数** | `491` / `16` | **`127` / `5`** | **约少 3.9 倍 / 3.2 倍** |
| **使用 2.4G 鼠标时是否加载蓝牙 DLL** | 取决于运行时 | **`False`（未加载 `BluetoothAPIs.dll`）** | **仅作为按需延迟回退** |
| **可执行文件体积** | 多文件 .NET 输出 | **`107.0 KB` 独立 `.exe`** | **零运行时依赖** |

---

## 单活动后端多鼠标引擎

与同时为每种协议启动轮询循环不同，[`mouse_manager.c`](src/X11BatteryTrayNative/mouse_manager.c) 在启动或热插拔（`WM_DEVICECHANGE`）时仅执行**一次**快速的只读设备发现扫描，仅绑定**优先级最高的单个后端**，并仅运行**一个** 64 KB 栈空间的工作线程：

| 优先级 | 后端 | 支持设备 / VID:PID | 传输方式与策略 |
| :---: | --- | --- | --- |
| `100` / `90` | **Attack Shark X11 Passive** | `VID_1D57`（`PID_FA55` 有线、`PID_FA60` 2.4G） | 严格只读的 `UsagePage=0x000A` Report `0x03` 监听器（在 6 秒休眠期间保留电量 `%`） |
| `80` | **Compx / CX52850 17-Byte** | `VID_3554` / `248A` / `25A7`（`FF02:0002`，17B `0x08/0x04`） | VXE、ATK、VGN、Kysona、Darmoshark、Attack Shark X3 — 每 `60 s` 查询一次电量与毫伏（mV）遥测数据 |
| `76` / `74` | **Logitech HID++ 2.0** | `VID_046D`（`FF00:0002` / `0001`） | Lightspeed / Bolt / Unifying — 槽位 `0xFF, 1..6`，鼠标校验（`0x0005`），按 `0x1004` → `0x1000` → `0x1001` 顺序读取 |
| `72` / `68` | **Razer 90-Byte Feature** | `VID_1532`（`Feat=91`） | Viper、DeathAdder、Basilisk、Orochi、Naga、Cobra — 每 `60 s` 发送带 XOR CRC 的 `0x07/0x80` 与 `0x07/0x84` 请求 |
| `50` | **Standard HID Battery** | Value Caps `0x06/0x20`、`0x85/0x65–0x66` | 通用 HID 电池鼠标（已排除 UPS `0x84/0x04`、键盘和耳机） |
| `35` / `30` | **Bluetooth LE GATT** | 服务 `0x180F` + `0x1812` + GAP Appearance | 仅当未检测到任何 USB/2.4G 鼠标时延迟加载 `BluetoothAPIs.dll` |

---

## 快速开始

### 方式 A：使用预编译托盘监控程序
1. 前往 [**GitHub Releases**](https://github.com/johnlrt/mouse-client-x11/releases) 下载 **`X11BatteryTray.exe`**。
2. 直接运行即可 — 原生托盘工具无需安装程序，也无需 `.NET` 运行时。
3. 左键点击托盘图标可打开深色弹出面板，右键点击可切换图标样式、开启 Windows 开机自启（`HKCU\...\Run`）或触发重新扫描。

> **Windows 11 提示：** 如果托盘图标最初隐藏在 `^` 溢出折叠菜单中，可将其拖拽一次到任务栏上（Windows 电池/音量图标旁），或在弹出面板中点击 **«Закрепить на панели задач Windows ->»**（固定到 Windows 任务栏）。

### 方式 B：从源码构建

```powershell
# 1. 构建超轻量原生 Win32 C 托盘可执行文件 -> .\bin\X11BatteryTray.exe
#    若存在 .tools\w64devkit\bin\gcc.exe 则优先使用，否则使用 PATH 中的 gcc（MinGW-w64 / w64devkit）
& ".\src\X11BatteryTrayNative\build.ps1"

# 2. 或者直接双击 Start-BatteryTray.bat（如有需要会自动构建并启动）
.\Start-BatteryTray.bat

# 或直接运行
& ".\bin\X11BatteryTray.exe"

# 3. 构建洁净室 .NET 8 WPF 控制中心并运行 xUnit 测试
dotnet test X11Mouse.sln -c Release
& ".\src\X11Mouse.App\bin\Release\net8.0-windows\X11Mouse.exe"
```

---

## 命令行参数参考 (`X11BatteryTray.exe`)

| 参数 | 行为说明 |
| --- | --- |
| `--status` | 探测一次优先级最高的鼠标并输出模式、后端、VID/PID、产品名称及电量/电压 |
| `--scan` | 以只读方式列出所有检测到的鼠标/电池 HID 与 BLE 接口，以及最终选中的后端 |
| `--self-test` | 运行内置的协议解析器、校验和/CRC、电量换算及后端优先级自检 |
| `--smoke-test` | 渲染所有样式/主题/模式下的图标，并在无界面环境下打开/更新/关闭 GDI 弹出面板 |
| `--preview <path.bmp>` | 导出平铺对比图（BMP），展示 `20×20` 和 `16×16` 尺寸下的全部 3 种图标样式 |

---

## 附加组件：洁净室控制中心与协议规范（Attack Shark X11）

- **[X11Mouse.App](src/X11Mouse.App/X11Mouse.App.csproj)**（`.NET 8` WPF）与 **[X11Mouse.Core](src/X11Mouse.Core/X11Mouse.Core.csproj)**：
  - 完整的交互式 Attack Shark X11 控制套件：俯视矢量鼠标示意图（支持按键双向高亮）、8 档 DPI（`220` 项寄存器表）、环形轮询率选择器（`125`–`1000 Hz`）、灯光/休眠/去抖（debounce）设置、18 槽位按键重映射（`151` 项 `Hook.dll` VK/扫描码/HID 映射表）、基于 `WH_KEYBOARD_LL` / `WH_MOUSE_LL` 的宏录制器、兼容旧版 `pro.data` / `.profile` / `macro.data` / `.macro` 的二进制编解码器，以及 `--probe` / `--smoke-test` 参数。
- **[X11Mouse.BatteryTray](src/X11Mouse.BatteryTray/X11Mouse.BatteryTray.csproj)**：
  - 保留在解决方案中的原始 `.NET 8` WinForms 参考实现，并在 **[X11Mouse.Tests](tests/X11Mouse.Tests/X11Mouse.Tests.csproj)** 中附带 **19** 项自动化测试。
- **[协议与二进制格式规范](rea-analysis/protocol-spec.md)**：
  - 完整记录 Attack Shark X11 的 HID 端点（`0x1D57:0xFA55` 有线、`0xFA60` 2.4G）、异步 Report `0x03` 通知、Feature 写入数据包（`0x04`–`0x0C`）、校验和帧结构以及配置文件/宏序列化格式。

---

## 安全性与非侵入性保证

- **共享 HID 访问：** 使用 `FILE_SHARE_READ | FILE_SHARE_WRITE` 打开端点，确保工具能与官方厂商驱动安全共存，既不强行结束官方进程，也不独占锁定设备。
- **零官方文件篡改：** 设置隔离保存在 `%APPDATA%\AttackSharkX11Clean` 与 `HKCU\Software\AttackSharkX11Clean\Tray` 中，绝不修改 `C:\Program Files (x86)\X11 Mouse` 和 `%APPDATA%\X11 Mouse`。
- **绝不盲目写入硬件：** `--scan`、`--probe` 以及 X11 的 `--status` 均严格保持只读；仅在明确匹配对应鼠标协议后，才会发送厂商查询帧（`Compx`、`HID++`、`Razer`）。

---

## 仓库结构

```text
├── src/
│   ├── X11BatteryTrayNative/   # 纯 Win32 C 多鼠标托盘电量监控程序（107 KB 可执行文件）
│   ├── X11Mouse.Core/          # 洁净室 X11 协议、HID 传输与存储库
│   ├── X11Mouse.App/           # .NET 8 WPF 控制中心界面
│   └── X11Mouse.BatteryTray/   # 旧版 .NET 8 WinForms 托盘监控程序（参考实现）
├── tests/
│   └── X11Mouse.Tests/         # 19 项 xUnit 协议、存储与模拟设备服务测试
├── rea-analysis/
│   ├── protocol-spec.md        # 逆向分析得到的 X11 协议与二进制格式规范
│   └── hook-table-151.json     # 提取的 151 项键码映射表
├── docs/assets/                # README 界面与图标截图
├── Start-BatteryTray.bat       # 一键启动脚本（如有需要会自动构建）
└── X11Mouse.sln                # Visual Studio / dotnet 解决方案文件
```

---

## 开源协议

基于 [MIT 协议](LICENSE)开源。
