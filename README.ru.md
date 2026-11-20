<div align="center">

# Универсальный индикатор заряда беспроводных мышей в трее (`mouse-client-x11`)

**Сверхлёгкий (~0.31 MB RAM, 107 KB) нативный монитор заряда батареи в области уведомлений Windows (Win32 C) для беспроводных мышей 2.4 GHz, USB и Bluetooth — с поддержкой Logitech, Razer, Compx/VXE/ATK/VGN/Darmoshark/Attack Shark, Standard HID Battery и BLE GATT.**

> Также включает опциональный clean-room Центр управления на `.NET 8` WPF и спецификацию HID-протокола для Attack Shark X11.

[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%2F%2011-0078D4?style=flat-square&logo=windows)](https://github.com/johnlrt/mouse-client-x11)
[![Native Tray](https://img.shields.io/badge/Tray%20Binary-Win32%20C%20%C2%B7%20107%20KB-E2211C?style=flat-square&logo=c)](src/X11BatteryTrayNative/main.c)
[![Memory](https://img.shields.io/badge/Private%20RAM-~0.31%20MB-2EA043?style=flat-square)](src/X11BatteryTrayNative/mouse_manager.c)
[![Control Center](https://img.shields.io/badge/Control%20Center-.NET%208%20WPF-512BD4?style=flat-square&logo=dotnet)](src/X11Mouse.App/X11Mouse.App.csproj)
[![Tests](https://img.shields.io/badge/Tests-19%2F19%20Unit%20%2B%205%2F5%20Protocol-2EA043?style=flat-square)](tests/X11Mouse.Tests/X11Mouse.Tests.csproj)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)

[English](README.md) | **Русский** | [简体中文](README.zh-CN.md)

</div>

---

## Визуальная демонстрация

<table>
  <tr>
    <td align="center" width="50%">
      <img src="docs/assets/tray-flyout.png" alt="Нативное тёмное всплывающее окно" />
      <br />
      <sub><b>Нативный тёмный flyout на GDI</b> — отображает активную мышь, бейдж типа подключения, текущий заряд в процентах, телеметрию бэкенда, кнопку закрепления и ручное обновление.</sub>
    </td>
    <td align="center" width="50%">
      <img src="docs/assets/tray-icons.png" alt="Пиксельно-выровненные иконки трея (20px и 16px)" />
      <br />
      <sub><b>Пиксельно-выровненные 32-битные DIB-иконки для трея</b> — 3 стиля иконок в размерах <code>20×20</code> (DPI 125%) и <code>16×16</code> (DPI 100%): «Цифры + шкала», «Индикатор батареи» и «Крупные цифры».</sub>
    </td>
  </tr>
</table>

---

## Зачем это нужно и живой бенчмарк RAM

Официальное ПО для мышей нередко потребляет много памяти или навязывает фоновые службы OEM. Здесь монитор заряда в трее написан на **чистом Win32 C** без `.NET CLR`, `WinForms` и `GDI+`, с пиксельно-выровненными иконками через `CreateDIBSection` и движком с **одним активным бэкендом** (`107 KB`, один `.exe`):

| Метрика (замер в реальном времени на Windows 11) | Версия `.NET 8 WinForms` | Нативная версия `Win32 C` | Улучшение |
| --- | --- | --- | --- |
| **Диспетчер задач (`Private Working Set`)** | `11.2 MB` | **`0.31 MB` (`320 KB`)** | **~36× меньше** |
| **Общий рабочий набор (`Total Working Set`)** | `74.48 MB` | **`0.98 MB`** | **~76× меньше** |
| **Выделенная память (`Private Bytes`)** | `16.75 MB` | **`1.68 MB`** | **~10× меньше** |
| **Дескрипторы / Потоки** | `491` / `16` | **`127` / `5`** | **~3.9× / 3.2× меньше** |
| **Загрузка Bluetooth DLL при 2.4G-мыши** | Зависит от среды выполнения | **`False` (`BluetoothAPIs.dll` не загружена)** | **Только ленивый фолбэк** |
| **Размер исполняемых файлов** | Многофайловая сборка .NET | **`107.0 KB` автономный `.exe`** | **Без внешних зависимостей** |

---

## Мультибрендовый движок с одним активным бэкендом

Вместо одновременного опроса всех протоколов [`mouse_manager.c`](src/X11BatteryTrayNative/mouse_manager.c) выполняет **один** быстрый проход обнаружения в режиме только чтения при запуске или горячем подключении (`WM_DEVICECHANGE`), привязывает **только один бэкенд с наивысшим приоритетом** и запускает **один** рабочий поток со стеком 64 KB:

| Приоритет | Бэкенд | Устройства / VID:PID | Транспорт и стратегия |
| :---: | --- | --- | --- |
| `100` / `90` | **Attack Shark X11 Passive** | `VID_1D57` (`PID_FA55` проводная, `PID_FA60` 2.4G) | Строго пассивный слушатель `UsagePage=0x000A` Report `0x03` только для чтения (сохраняет `%` при 6-сек. сне) |
| `80` | **Compx / CX52850 17-Byte** | `VID_3554` / `248A` / `25A7` (`FF02:0002`, 17B `0x08/0x04`) | VXE, ATK, VGN, Kysona, Darmoshark, Attack Shark X3 — запрос + телеметрия mV каждые `60 s` |
| `76` / `74` | **Logitech HID++ 2.0** | `VID_046D` (`FF00:0002` / `0001`) | Lightspeed / Bolt / Unifying — слоты `0xFF, 1..6`, проверка мыши (`0x0005`), `0x1004` → `0x1000` → `0x1001` |
| `72` / `68` | **Razer 90-Byte Feature** | `VID_1532` (`Feat=91`) | Viper, DeathAdder, Basilisk, Orochi, Naga, Cobra — `0x07/0x80` и `0x07/0x84` с XOR CRC каждые `60 s` |
| `50` | **Standard HID Battery** | Value Caps `0x06/0x20`, `0x85/0x65–0x66` | Универсальные HID-мыши с батареей (UPS `0x84/0x04`, клавиатуры и гарнитуры исключены) |
| `35` / `30` | **Bluetooth LE GATT** | Сервисы `0x180F` + `0x1812` + GAP Appearance | Лениво загружает `BluetoothAPIs.dll` только если не найдена мышь USB/2.4G |

---

## Быстрый старт

### Вариант A: Готовый монитор для трея
1. Скачайте **`X11BatteryTray.exe`** из раздела [**GitHub Releases**](https://github.com/johnlrt/mouse-client-x11/releases).
2. Запустите файл — для нативной утилиты в трее не требуются ни установщик, ни среда `.NET`.
3. Нажмите левой кнопкой мыши по иконке в трее, чтобы открыть тёмное всплывающее окно, или правой кнопкой для смены стиля иконки, включения автозапуска Windows (`HKCU\...\Run`) либо принудительного пересканирования.

> **Совет для Windows 11:** Если иконка в трее сначала скрыта под стрелкой `^`, перетащите её один раз на панель задач рядом со значками батареи/звука Windows (или нажмите **«Закрепить на панели задач Windows ->»** внутри всплывающего окна).

### Вариант B: Сборка из исходников

```powershell
# 1. Сборка сверхлёгкого нативного Win32 C бинарника для трея -> .\bin\X11BatteryTray.exe
#    Использует .tools\w64devkit\bin\gcc.exe при наличии или gcc из PATH (MinGW-w64 / w64devkit)
& ".\src\X11BatteryTrayNative\build.ps1"

# 2. Либо просто дважды кликните Start-BatteryTray.bat (автоматически соберёт при необходимости и запустит)
.\Start-BatteryTray.bat

# Или запустите напрямую
& ".\bin\X11BatteryTray.exe"

# 3. Сборка clean-room Центра управления на .NET 8 WPF и запуск тестов xUnit
dotnet test X11Mouse.sln -c Release
& ".\src\X11Mouse.App\bin\Release\net8.0-windows\X11Mouse.exe"
```

---

## Справка по CLI (`X11BatteryTray.exe`)

| Флаг | Поведение |
| --- | --- |
| `--status` | Однократно опрашивает приоритетную мышь и выводит режим, бэкенд, VID/PID, название продукта и заряд/напряжение |
| `--scan` | В режиме только чтения выводит все обнаруженные HID- и BLE-интерфейсы мышей/батарей вместе с выбранным бэкендом-победителем |
| `--self-test` | Запускает встроенные самотесты парсера протокола, контрольной суммы/CRC, масштабирования и приоритетов бэкендов |
| `--smoke-test` | Отрисовывает иконки во всех стилях/темах/режимах и без GUI-взаимодействия открывает/обновляет/закрывает GDI-окно flyout |
| `--preview <path.bmp>` | Экспортирует мозаичный BMP-файл для сравнения всех 3 стилей иконок в размерах `20×20` и `16×16` |

---

## Бонус: Clean-Room Центр управления и спецификация протокола (Attack Shark X11)

- **[X11Mouse.App](src/X11Mouse.App/X11Mouse.App.csproj)** (`.NET 8` WPF) и **[X11Mouse.Core](src/X11Mouse.Core/X11Mouse.Core.csproj)**:
  - Полнофункциональный интерактивный пакет управления Attack Shark X11: векторная схема мыши (вид сверху) с двунаправленной подсветкой кнопок, 8 ступеней DPI (таблица регистров из `220` записей), радиальный переключатель частоты опроса (`125`–`1000 Hz`), настройки подсветки/сна/антидребезга (debounce), переназначение 18 слотов кнопок (таблица VK/scan/HID из `151` записи из `Hook.dll`), рекордер макросов на `WH_KEYBOARD_LL` / `WH_MOUSE_LL`, бинарный кодек совместимости с `pro.data` / `.profile` / `macro.data` / `.macro`, а также флаги `--probe` / `--smoke-test`.
- **[X11Mouse.BatteryTray](src/X11Mouse.BatteryTray/X11Mouse.BatteryTray.csproj)**:
  - Исходная эталонная реализация на `.NET 8` WinForms, сохранённая в решении вместе с **19** автотестами в **[X11Mouse.Tests](tests/X11Mouse.Tests/X11Mouse.Tests.csproj)**.
- **[Спецификация протокола и бинарных форматов](rea-analysis/protocol-spec.md)**:
  - Полная документация HID-конечных точек Attack Shark X11 (`0x1D57:0xFA55` проводная, `0xFA60` 2.4G), асинхронных уведомлений Report `0x03`, пакетов записи Feature (`0x04`–`0x0C`), формирования контрольной суммы и форматов сериализации профилей/макросов.

---

## Гарантии безопасности и неинвазивности

- **Совместный доступ к HID:** Конечные точки открываются с флагами `FILE_SHARE_READ | FILE_SHARE_WRITE`, благодаря чему утилиты безопасно сосуществуют с официальными драйверами производителей без завершения их процессов и эксклюзивных блокировок.
- **Отсутствие изменений официальных файлов:** Настройки изолированы в `%APPDATA%\AttackSharkX11Clean` и `HKCU\Software\AttackSharkX11Clean\Tray`. Каталоги `C:\Program Files (x86)\X11 Mouse` и `%APPDATA%\X11 Mouse` никогда не изменяются.
- **Никаких слепых записей в оборудование:** `--scan`, `--probe` и `--status` для X11 работают строго в режиме только чтения; запросы вендорных протоколов (`Compx`, `HID++`, `Razer`) отправляются только после точного совпадения соответствующего протокола мыши.

---

## Структура репозитория

```text
├── src/
│   ├── X11BatteryTrayNative/   # Нативный монитор в трее для разных мышей на чистом Win32 C (бинарник 107 KB)
│   ├── X11Mouse.Core/          # Clean-room библиотека протокола X11, HID-транспорта и хранения данных
│   ├── X11Mouse.App/           # Интерфейс Центра управления на .NET 8 WPF
│   └── X11Mouse.BatteryTray/   # Устаревший эталонный монитор в трее на .NET 8 WinForms
├── tests/
│   └── X11Mouse.Tests/         # 19 xUnit-тестов протокола, хранилища и мокового сервиса устройства
├── rea-analysis/
│   ├── protocol-spec.md        # Спецификация реверс-инжиниринга протокола X11 и бинарных форматов
│   └── hook-table-151.json     # Извлечённая таблица кодов клавиш из 151 записи
├── docs/assets/                # Скриншоты интерфейса и иконок для README
├── Start-BatteryTray.bat       # Запускатель в один клик (собирает проект автоматически при необходимости)
└── X11Mouse.sln                # Файл решения Visual Studio / dotnet
```

---

## Лицензия

Распространяется под [лицензией MIT](LICENSE).
