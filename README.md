# MoreFine S500+ · macOS Sequoia EFI

适用于 MoreFine S500+（Ryzen 9 5900HX、Cezanne Vega 核显）的 OpenCore 配置。项目以一台 32 GB 主机的在用 EFI 为基础，保留了它的 USB 端口映射、驱动加载顺序和启动参数。同型号主机也请核对网卡、硬盘和端口布局。首次启动前请填写自己的 SMBIOS 身份。需要驱动替换卡 BCM94360Z3 的 Wi-Fi 时，可按下文选装 SimpleBCM。

## 硬件与验证范围

| 硬件 | 配置与实机记录 |
| --- | --- |
| Ryzen 9 5900HX | 按 8 个物理核心配置 AMD 内核补丁，使用 `MacPro7,1`、VoodooTSCSyncAMD 和 AppleMCEReporterDisabler。 |
| Radeon Vega | 此版启用 weachy 的 NootedRed fork（1.3.0），本机实测 H.264 和 H.265 硬件解码均通过。此前在 Sequoia 15.7.8 上也验证过 Metal 3 和 HDMI 4K 60 Hz。 |
| RTL8125 / RTL8168 | 配备 2.5G 和千兆有线驱动。在线 Recovery 曾通过 RTL8125 完成安装。 |
| ALC269 | AppleALC 1.9.7，layout 6。该机此前验证过内置输入、输出和 HDMI 音频。 |
| BCM94360Z3（本机后换） | Wi-Fi 使用 SimpleBCM 1.2.1 与 `SSDT-GPP4-WIFI.aml`。此版缺少独立的无线连接测试记录。蓝牙走 USB，机型映射将 `XHC1/HS03` 设为内置端口。 |
| 原装无线网卡 | 原装卡的蓝牙可能已有开源驱动支持，本项目尚未实测。 |
| aigo P7000E 2 TB | 本机此前使用原生 NVMe 驱动，验证过 PCIe x4、TRIM 和 SMART。 |
| 虚拟化 | 已启用 RyzenHV 0.7.8。原机配合 BIOS SVM、宿主端适配和 Parallels Desktop 26.3.2 运行过 Windows 11；宿主工具和 PD 适配需要另行配置。 |

## 准备启动 U 盘

1. 备份原 EFI，保留一份已知可启动的介质。将本项目的 `EFI/` 复制到测试 U 盘 FAT32 分区的根目录，确认能看到 `EFI/BOOT/BOOTx64.efi` 和 `EFI/OC/config.plist`。
2. **按需配置 Wi-Fi。** 若保留原装卡或只用有线网络，可跳过下载，在 U 盘的 `EFI/OC/config.plist` 中找到 `Kernel → Add` 里 `BundlePath` 为 `SimpleBCM.kext` 的条目，将 `Enabled` 设为 `false`。若已换装 BCM94360Z3 并需要其 Wi-Fi，则保留 `Enabled=true`。若 U 盘上缺少 `EFI/OC/Kexts/SimpleBCM.kext`，从上游下载 `SimpleBCM-v1.2.1-RELEASE.zip`，提取其中的同名 kext 到该路径。原机副本中 `Contents/MacOS/SimpleBCM` 的 SHA-256 为 `3e3a46ec980f4a9f6a8c994baba54f36b30d4067d72b7ec78b87d2ad09f92059`，与该发行包一致。[SimpleBCM 1.2.1 下载](https://github.com/laobamac/SimpleBCM/releases/tag/v1.2.1)
3. **只编辑 U 盘上的配置。** 在 `PlatformInfo → Generic` 填写本机的 `SystemSerialNumber`、`MLB`、`SystemUUID` 和 `ROM`（6 字节 Data）。公开配置中的四项均为空。同一台主机重装可沿用原身份，新机器需为 `MacPro7,1` 准备独立身份。
4. 核对 BIOS。本机使用 UEFI、Above 4G Decoding、XHCI Hand-off 和 AHCI，关闭 CSM、Secure Boot、Fast Boot、Re-Size BAR 和 IOMMU。将 UMA 设为固定分配（`UMA_SPECIFIED`），显存至少 2 GB。本机配备 32 GB 内存，实际分配 8 GB，并关闭 GPU Host Translation Cache。使用 RyzenHV 时开启 SVM。
5. 使用在线 Recovery 时，把 `BaseSystem.dmg` 和 `BaseSystem.chunklist` 放入与 `EFI/` 同级的 `com.apple.recovery.boot/`。网线先接 2.5G 口，键盘和安装盘直连后置 USB。安装中途重启继续选择 `macOS Installer`，待该项消失再选系统卷。验证 U 盘引导、输入设备和有线网络后，再迁移到内置 EFI 分区。

## 配置取舍与排错

- **安全和诊断设置。** 原机配置保留 `csr-active-config=0x803`、`SecureBootModel=Disabled` 和 `amfi_get_out_of_my_way=0x1`，降低了系统的完整性保护。它也保留了 verbose、内核调试、SimpleBCM 调试和 RyzenHV 用户访问参数。部署前请确认这些设置符合自己的使用场景。
- **启动停滞。** 若末屏是 `CoreAnalyticsHub start completed`，记录后续 panic 回溯或自动重启位置。早期实测在 `MacPro7,1` 下遇到过 `AppleIntelMCEReporter` panic，补齐 AppleMCEReporterDisabler 后才进入安装界面。
- **首次进桌面。** 安装助手优先选静态头像，账户创建阶段等待写入完成。若 HDMI 画面发灰，先核对显示器与 macOS 两端的 HDR 开关。
- **无线和 USB。** Wi-Fi 应实测扫描、关联和传输，蓝牙应单独检查 USB 枚举。更换主板或发现端口布局有差异时，重新核对 UTBMap，保留 `XhciPortLimit=false`。此前旧 OCLP 无线栈的实测结果仅适用于当时的驱动组合。

## 来源与许可

本项目新编写的说明和原创配置整理采用 [MIT 许可证](LICENSE)。引导程序、内核补丁、ACPI 表和各驱动沿用各自上游的许可和版权声明。随附的上游许可及 SimpleBCM 第三方声明见 [LICENSES/](LICENSES/)。

- 引导与 AMD 内核补丁：[OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)、[AMD Vanilla](https://github.com/AMD-OSX/AMD_Vanilla)。
- 核显、音频与传感器：[weachy 的 NootedRed fork（本 EFI 使用）](https://github.com/weachy/NootedRed)、[NootedRed 上游](https://github.com/ChefKissInc/NootedRed)、[RadeonSensor](https://github.com/ChefKissInc/RadeonSensor)、[SMCAMDProcessor](https://github.com/trulyspinach/SMCAMDProcessor)、[AppleALC](https://github.com/acidanthera/AppleALC)。
- 补丁与 SMC：[Lilu](https://github.com/acidanthera/Lilu)、[VirtualSMC](https://github.com/acidanthera/VirtualSMC)、[RestrictEvents](https://github.com/acidanthera/RestrictEvents)。
- 有线网卡驱动采用 [GPLv2](LICENSES/GPL-2.0.txt)。对应版本的源码见 [LucyRTL8125Ethernet 1.2.2](https://github.com/Mieze/LucyRTL8125Ethernet/tree/v1.2.2) 和 [RealtekRTL8111 2.4.2](https://github.com/Mieze/RTL8111_driver_for_OS_X/tree/2.4.2)。
- USB 与无线：[USBToolBox](https://github.com/USBToolBox/kext)、[SimpleBCM 的第三方组件说明](https://github.com/laobamac/SimpleBCM/blob/v1.2.1/NOTICE.md)。
- 虚拟化与宿主适配：[RyzenHV](https://github.com/OverlordDev/ryzen-hypervisor.framework)。
