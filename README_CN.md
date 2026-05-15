# macOS 15 (Sequoia) 在 Ivy Bridge 至强 E3-1230 V2 + RX 570 上的安装

[![English](https://img.shields.io/badge/lang-English-blue)](README.md)

本仓库包含一份定制的 OpenCore EFI 文件夹，用于在以下老旧台式机上安装和运行 macOS 15 (Sequoia)：

| 组件 | 规格 |
|------|------|
| 处理器 | Intel Xeon E3-1230 V2 (Ivy Bridge，4 核 8 线程，基频 3.30 GHz，睿频 3.70 GHz，TDP 69 W) |
| 显卡 | AMD Radeon RX 570 8 GB (Polaris, Ellesmere) |
| 主板 | MSI ZH77A-G43 (H77 芯片组，LGA1155) |
| 内存 | 16 GB DDR3 1600 MHz |
| 硬盘 | SATA SSD (APFS) |
| 有线网卡 | Realtek RTL8111E |
| 声卡 | Realtek ALC892 (layout-id 3) |
| USB | Intel 7 系列芯片组 (USB 2.0 / 3.0) |

配置文件使用 `MacPro7,1` 机型以获得 AMD 独显的最佳编解码支持，同时通过传统 ACPI 电源管理驱动和 AVX2 兼容补丁，使 macOS 15 能够在 Ivy Bridge 平台上运行。

## EFI 概览

### ACPI
- `SSDT-EC-DESKTOP.aml` – 为台式机仿冒嵌入式控制器。
- `SSDT-PM.aml` – 由 `ssdtPRGen.sh` 生成的 CPU 电源管理表，参数与 E3-1230 V2 匹配（频率范围 1200-3700 MHz，TDP 69 W）。  
  *若你的 CPU 不同，必须重新生成此文件。*
- **ACPI 删除规则** – 删除原厂 `CpuPm` 和 `Cpu0Ist` SSDT 表，避免冲突。

### 内核扩展 (Kexts)
| 驱动 | 用途 |
|------|------|
| Lilu.kext | 通用内核补丁引擎 |
| VirtualSMC.kext | SMC 仿真器 |
| WhateverGreen.kext | 显卡补丁 |
| AppleALC.kext | 声卡 (layout-id 3) |
| RealtekRTL8111.kext | 有线网卡 |
| CryptexFixup.kext | 强制安装 Apple Silicon 兼容的 cryptex；由于 Ivy Bridge 缺少 AVX2.0，此为必需 |
| AMFIPass.kext | 绕过 AMFI 以加载未签名的驱动 |
| RestrictEvents.kext | 拦截不必要的系统事件 |
| USBToolBox.kext + UTBMap.kext | USB 端口映射 (已为 MSI ZH77A-G43 定制) |
| AppleIntelCPUPowerManagement.kext | 传统 CPU 电源管理 (替代 XCPM) |
| AppleIntelCPUPowerManagementClient.kext | 上述驱动的用户态客户端 |
| CPUFriend.kext / CPUFriendDataProvider.kext | **已禁用** – 仅作参考；切勿与上述传统电源管理驱动同时启用 |

### 驱动程序
- HfsPlus.efi
- OpenCanopy.efi (图形引导界面)
- OpenRuntime.efi
- ResetNvramEntry.efi

### 主要 `config.plist` 设置
- **SMBIOS 机型**: `MacPro7,1`
- **启动参数（安装及补丁前）**: `alcid=3 -crypt_force_avx -amd_no_dgpu_accel`
  参数 `-amd_no_dgpu_accel` 在安装阶段和 OCLP 补丁完成前禁用显卡加速，防止因 AMD 驱动不完整而导致黑屏。
  **打上 OCLP 补丁后，必须从 boot-args 中移除 `-amd_no_dgpu_accel`**，以开启硬件加速。
- **内核怪癖 (Kernel Quirks)**: `AppleCpuPmCfgLock` = `true`, `AppleXcpmCfgLock` = `true`, `DisableIoMapper` = `true`, `PanicNoKextDump` = `true`, `PowerTimeoutKernelPanic` = `true`
- **安全启动模型**: `Disabled`
- **Vault**: `Optional`

## BIOS 设置

安装前请将主板 BIOS 调整为以下设置：

- 首先加载优化默认值，然后修改：
- 内置显卡：**禁用** (你使用独立显卡)
- 主显示适配器：**PCIe**
- Above 4G Decoding：**开启** (若主板不支持，则保持 `ResizeGpuBars` = -1)
- XHCI Hand‑off：**开启**
- Legacy USB 支持：**开启**
- SATA 模式：**AHCI**
- 启动模式：**纯 UEFI** (关闭 CSM)
- 安全启动：**关闭**
- Intel SpeedStep、Turbo Boost、C‑States：全部**开启**

## 安装步骤

### 1. 制作 macOS 安装 U 盘

#### 方案 A：使用 `macrecovery.py`
此方法从 Apple 服务器下载最小的 macOS 恢复镜像，并制作一个引导 U 盘，在安装过程中将联网下载完整的 macOS。

**准备条件：**
- 系统已安装 Python 3
- 一个 8 GB 或更大的 U 盘

**步骤：**

1. 从 [OpenCorePkg 官方发布页](https://github.com/acidanthera/OpenCorePkg/releases) 下载 OpenCorePkg。

2. 解压 ZIP 文件，在 `Utilities/macrecovery/` 目录下打开终端。

3. 运行以下命令下载 macOS Sequoia 恢复镜像：
   ```bash
   python3 macrecovery.py -b Mac-7BA5B2D9E42DDD94 -m 00000000000000000 download
   ```
   这会将 `BaseSystem.dmg` 和 `BaseSystem.chunklist` 下载到 `com.apple.recovery.boot` 文件夹中。

4. 将 U 盘格式化为 **FAT32**（MBR 分区方案）。

5. 在 U 盘根目录创建一个名为 `com.apple.recovery.boot` 的文件夹（名称必须完全一致）。

6. 将 `BaseSystem.dmg` 和 `BaseSystem.chunklist` 复制到 `com.apple.recovery.boot` 中。

7. （可选）为了让恢复条目在 OpenCore 引导菜单中显示友好的名称，在 `com.apple.recovery.boot` 中新建一个文本文件，写入如 `macOS Sequoia Recovery` 的标签，然后将文件重命名为 `.contentDetails`（无扩展名的隐藏文件）。

8. 挂载 U 盘的 EFI 分区，将本 EFI 文件夹复制进去。

**最终 U 盘结构：**
```
U 盘根目录
├── EFI
│   ├── BOOT
│   └── OC
└── com.apple.recovery.boot
    ├── BaseSystem.dmg
    ├── BaseSystem.chunklist
    └── .contentDetails (可选)
```

**注意：** `com.apple.recovery.boot` 文件夹必须放在 U 盘根目录，与 `EFI` 文件夹同级，不可放入 EFI 文件夹内。

#### 方案 B：使用完整 macOS 安装 App（仅限 macOS）
如果你已在真实的 Mac 或可正常工作的黑苹果上：
- 从 App Store 获取 macOS Sequoia 安装 App。
- 将 U 盘格式化为 `Mac OS 扩展（日志式）`，分区方案选择 GUID 分区图。
- 使用 `createinstallmedia` 命令：
  ```
  sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia --volume /Volumes/USB
  ```
- 挂载 EFI 分区，将本 EFI 文件夹复制进去。

### 2. 从 U 盘引导安装
- 插入 U 盘，开机，按启动菜单键（MSI 主板通常为 F11），选择 UEFI USB 启动项。
- 在 OpenCore 引导菜单中选择 `Install macOS Sequoia`。
- 安装器会通过 `-crypt_force_avx` 参数（已写入 boot-args）在缺少 AVX2 的 CPU 上运行。
- 使用磁盘工具将目标硬盘抹掉为 APFS 格式，然后开始安装。
- 系统会自动重启多次；**每次重启后都从 OpenCore 菜单选择 `Install macOS` 条目**（或新出现的 `macOS` 分区）。
- 整个安装过程中，画面会因为没有显卡加速而显得缓慢、分辨率低，这是正常现象，因为 AMD 驱动此时尚未打上 AVX2 补丁。
- 这是正常现象，因为此时我们特意使用了 `-amd_no_dgpu_accel` 参数以兼容安装环境。

### 3. 首次进入桌面与后续操作
最终重启后会进入桌面。**暂时不要登录 iCloud**。由于显卡加速和 CPU 电源管理都未启用，系统会感觉卡顿。

### 4. 使用 OpenCore Legacy Patcher (OCLP)
- 在 OCLP 开始打补丁前，请确认启动参数中仍然包含 `-amd_no_dgpu_accel`，否则补丁过程可能卡死或黑屏。
- 下载最新版 [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher)。
- 打开 OCLP，点击 **Post Install Root Patch**。
- OCLP 会检测到你的 AMD 显卡，并对相关驱动（如 `AMDRadeonX4000.kext`）打补丁，使其能在无 AVX2 的 CPU 上工作。
- 按照提示完成补丁，然后**重启**。
- 补丁完成后重启前，**务必从 config.plist 的 boot-args 中移除 `-amd_no_dgpu_accel`**，让显卡获得完整加速。
- 重启后，显卡应已获得完整加速（Metal 显示“支持”）。

### 5. 确认 / 重新生成 CPU 电源管理
EFI 中已包含为 E3-1230 V2 定制的 `SSDT-PM.aml`（由 `ssdtPRGen.sh` 生成）。若你使用其他 CPU，请重新生成：
```bash
# 以 E3-1230 V2 为例（请根据你的 CPU 修改 base/turbo/TDP）
~/ssdtPRGen.sh -bclk 100 -c 1 -f 3300 -lfm 1200 -l 8 -p 'i7-3770' -turbo 3700 -t 69 -x 0 -m iMac13,2
# 生成的文件位于 ~/Library/ssdtPRGen/ssdt.aml
cp ~/Library/ssdtPRGen/ssdt.aml /Volumes/EFI/EFI/OC/ACPI/SSDT-PM.aml
```
然后在 `config.plist` 的 `ACPI -> Add` 中启用 `SSDT-PM.aml`，并确保 `CPUFriend.kext` 与 `CPUFriendDataProvider.kext` 保持**禁用**。

### 6. 最终重启与验证
- 保存 `config.plist`，关机，再次开机。
- 在 OpenCore 引导菜单中按**空格键**，选择 `Reset NVRAM`，系统会自动重启。
- 正常启动进入 macOS。
- 验证显卡加速：打开“系统信息” -> “图形卡/显示器”，应显示 Metal “支持”。
- 验证 CPU 变频：使用 **Intel Power Gadget** 或执行 `sysctl -n machdep.xcpm.frequency_vectors`。你会看到 CPU 频率根据负载在约 1.6 GHz 到 3.5-3.7 GHz 之间动态变化。
- 若你更换了 SMBIOS 机型，请用 `GenSMBIOS` 重新生成序列号，并重新登录 Apple 服务。

## 已知问题与限制
- **缺失 AVX2.0**：尽管打了补丁，某些强制要求 AVX2 的应用程序仍可能崩溃。请在依赖本系统工作前，先对你的关键软件进行测试。
- **系统更新**：由于 `CryptexFixup.kext` 强制使用 Apple Silicon 版本的 cryptex，小幅增量更新可能无法安装。你很可能需要每次下载完整的安装包来进行版本更新。
- **CPU 最低频率**：待机频率可能在 1.6 GHz 左右，无法降至理论上的 1.2 GHz。这是 `MacPro7,1` 机型电源策略与 Ivy Bridge 传统电源管理路径之间的已知限制。系统日常使用依然流畅。
- **USB 端口映射**：自带的 `UTBMap.kext` 是专为 MSI ZH77A-G43 定制的。若你使用其他主板，必须通过 `USBToolBox` (Windows) 或 `USBMap` (macOS) 重新制作端口映射。

## 致谢
- [OpenCore Bootloader](https://github.com/acidanthera/OpenCorePkg)
- [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher)
- [ssdtPRGen.sh](https://github.com/Piker-Alpha/ssdtPRGen.sh)
- [CryptexFixup](https://github.com/acidanthera/CryptexFixup)
- [Lilu 及相关驱动](https://github.com/acidanthera)
- 广大的黑苹果社区
