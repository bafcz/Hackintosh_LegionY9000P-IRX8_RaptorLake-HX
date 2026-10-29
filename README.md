# Legion Y9000P IRX8 / Raptor Lake-HX Hackintosh EFI

适用于联想拯救者 Y9000P 2023（Raptor Lake-HX 平台）的 OpenCore EFI。

当前配置以 `Intel Core i9-13900HX`、`MacBookPro16,4` SMBIOS 和 macOS 26 / Darwin 25.6.0 为主要验证环境。本文重点记录两项针对本机硬件完成的适配：I2C 触摸板的 GPIO 中断驱动，以及 13900HX 在 macOS 下的 CPU 频率修复。

> 本 EFI 包含针对具体机器、CPU 和当前内核验证过的自定义组件。升级 macOS、替换 CPU 或直接更新相关 kext 前，请保留可启动的 EFI 备份。

## 硬件支持状态

下表来自当前 EFI 和实际运行系统，不代表同系列其他硬件配置可以直接通用。

| 硬件 | 当前状态 | 驱动或实现 |
| --- | --- | --- |
| Intel Core i9-13900HX | 已驱动 | `CpuTopologyRebuild`、`CpuTscSync`、CPUFriend 和 HWPRequestFix；24 核 32 线程可识别，支持正常降频和最高 5.4 GHz 轻载睿频 |
| NVIDIA GeForce RTX 4060 Laptop GPU 8 GB | 已驱动 | NullMoth NVRM 2.6.3；PCIe x8、Metal 3、内屏输出和图形加速可用 |
| 16 英寸内置显示屏 | 已驱动 | 2880×1800，当前可运行 240 Hz；支持亮度调节和亮度快捷键 |
| ELAN I2C 触摸板 | 部分可用 | VoodooI2C + 自定义 VoodooGPIO + VoodooI2CHID；GPIO8 硬件中断、点击、滚动和多点触控可用，但双击后拖动不可用，可启用 macOS 的三指拖移作为替代 |
| 内置键盘和 Fn 热键 | 已驱动 | VoodooPS2Controller、VoodooPS2Keyboard、BrightnessKeys |
| 内置扬声器 | 已驱动 | AppleALC + SimpleHDA，立体声输出可用 |
| 内置麦克风 | 已驱动 | 系统识别双声道内建麦克风并可作为默认输入设备 |
| Realtek RTL8168H/8111H 千兆网卡 | 已驱动 | RealtekRTL8111，系统接口为 `en0`，最高 1 Gb/s |
| Intel Wi-Fi | 已驱动 | AirportItlwm 2.4.0，系统识别为 AirPort，接口为 `en1` |
| Intel 蓝牙 | 已驱动 | IntelBluetoothFirmware、IntelBTPatcher、BlueToolFixup；蓝牙控制器和常用服务可用 |
| NVMe 固态硬盘 | 已驱动 | 原生 IONVMeFamily + NVMeFix；已验证三星 1 TB 和 Predator GM7 4 TB，PCIe 4.0 x4、TRIM 和 SMART 正常 |
| 电池和电源信息 | 已驱动 | VirtualSMC、ECEnabler、SMCBatteryManager；电量、充电状态、循环次数和健康状态可读取 |
| USB 端口 | 已驱动 | USBToolBox + UTBMap 定制端口映射 |
| 内置摄像头 | 部分可用 | 系统可识别为 UVC `Integrated Camera`，并加载 UVCProbeFix，但 Photo Booth 主界面当前不显示摄像头画面 |
| CPU/主板传感器 | 已驱动 | VirtualSMC、SMCProcessor、SMCSuperIO、SMCLightSensor |

### 已知问题

- 触摸板双击后拖动暂不可用。可在“系统设置 -> 辅助功能 -> 指针控制 -> 触控板选项”中启用“三指拖移”替代，不影响普通点击、滚动和多点触控。
- 内置摄像头能够被系统枚举为 UVC 设备，但 Photo Booth 主界面目前无摄像头画面。

## 关键组件

| 组件 | 当前版本 | 用途 |
| --- | --- | --- |
| Lilu | 1.7.2 | HWPRequestFix、CPUFriend 等插件的运行基础 |
| VoodooI2C | 2.9.1 | Intel I2C 控制器和 I2C 设备总线 |
| VoodooGPIO | 1.1.5（自定义） | 为 `INTC1085` 提供 Raptor Lake-S/HX GPIO 引脚和中断支持 |
| VoodooI2CHID | 1.0 | 驱动 ELAN I2C-HID 触摸板 |
| VoodooInput | 1.1.6 | 向 macOS 多点触控输入栈传递事件 |
| CPUFriend | 1.3.1 | 注入定制的 X86PlatformPlugin 频率向量 |
| CPUFriendDataProvider | 1.0.1 | 保存本机使用的定制 `FrequencyVectors` |
| HWPRequestFix | 0.2.2（自定义） | 修正 macOS 对 13900HX 下发的 HWP 性能请求 |

## I2C 触摸板

### 硬件路径

本机触摸板为 ACPI 节点 `TPD0`，位于 `I2C5` 控制器下，HID 为 `ELAN06FA`。运行时已确认：

- I2C 地址：`0x15`
- GPIO IRQ：`8`
- 中断模式：`GPIO`
- GPIO 控制器 ACPI ID：`INTC1085`
- macOS 输入设备由 `VoodooI2CHIDDevice` 和 `VoodooI2CMultitouchHIDEventDriver` 接管

当前方案使用真实 GPIO 中断，不使用轮询。

### 1. VoodooI2C 控制器匹配

原版 VoodooI2C 2.9.1 没有匹配本机 13 代平台的 I2C PCI ID。在以下位置加入：

```text
VoodooI2C.kext/Contents/Info.plist
IOKitPersonalities -> VoodooI2CPCILakeController -> IOPCIMatch
```

新增匹配项：

```text
0x7a7c8086&0xFFFCFFFF
```

XML 文件中 `&` 写作 `&amp;`。这项修改让 VoodooI2C 识别 Raptor Lake-HX 的 I2C 控制器；只完成此项时，系统可以看到触摸板，但缺少 GPIO 中断支持时触摸板仍不会产生输入。

### 2. Raptor Lake-HX VoodooGPIO

VoodooI2C 内置的 `VoodooGPIO.kext` 已替换为自定义 1.1.5 版本，主要改动如下：

- 新增 `VoodooGPIORaptorLakeS` 平台类。
- 通过 `IONameMatch = INTC1085` 匹配本机 GPIO 控制器。
- 加入 Raptor Lake-S/HX 的 pin、GPP、community 和寄存器偏移定义。
- 平台数据基于 Linux `pinctrl-alderlake.c` 中的 Raptor Lake-S / Alder Lake-S GPIO 布局整理。
- 修正 `VoodooGPIOIntel::start()` 的启动校验：GPIO-only 平台只要求有效的 pin/community 数据，不再错误地强制要求未使用的 pin-mux group/function 表。
- 最终版已清除临时日志、轮询和诊断实验代码。

VoodooGPIO 源码仓库：<https://github.com/bafcz/VoodooGPIO>

对应提交：

- `838c53f`：加入 Raptor Lake-S/HX（`INTC1085`）支持
- `fc651c9`：完成 1.1.5 最终版启动修复并清理诊断逻辑

### 3. ACPI / XOSI

当前 EFI **不需要 `SSDT-TPD0-GPIO.aml`**。该文件已从当前工作区移除，不应再添加到 `ACPI -> Add`。

当前启用了：

- `SSDT-XOSI.aml`
- `_OSI` 到 `XOSI` 的 ACPI Rename 补丁

XOSI 让固件在相关 ACPI 分支中获得 Windows 兼容的 `_OSI` 返回结果，避免设备进入非 Windows 的降级或不完整路径。当前原生 `TPD0` 资源可以被 VoodooI2C 解析为 `gpioIRQ = 8` 和 `Interrupt Mode = GPIO`，因此不再需要额外的 TPD0 GPIO SSDT 覆盖。

### 4. VoodooI2C 加载顺序

`Kernel -> Add` 中保持以下相对顺序：

1. `VoodooI2CServices.kext`
2. `VoodooGPIO.kext`
3. `VoodooInput.kext`
4. `VoodooI2C.kext`
5. `VoodooI2CHID.kext`

前三项是 `VoodooI2C.kext/Contents/PlugIns/` 内的插件，必须在 VoodooI2C 主 kext 之前声明。

### 5. 触摸板验证

```sh
kmutil showloaded | grep -E 'VoodooI2C|VoodooGPIO|VoodooInput'
ioreg -l -w0 | grep -A20 -B5 TPD0
```

正常状态应看到：

- `org.coolstar.VoodooGPIO (1.1.5)`
- `VoodooGPIORaptorLakeS` 匹配 `INTC1085`
- `VoodooI2C (2.9.1)` 和 `VoodooI2CHID`
- `TPD0` 的 `gpioIRQ = 8`
- `Interrupt Mode = GPIO`
- `DeviceOpenedByEventSystem = Yes`

## CPU 频率修复

### 原始问题

13900HX 在 macOS 下可以在刚进入桌面时短暂达到 5 GHz 以上，随后 HWP 请求被限制，P 核最高频率长期停在约 4.2 GHz。开启 OpenCore 的 `AppleXcpmForceBoost` 虽然可以恢复高频，但会让 CPU 长时间保持高请求状态，导致空闲功耗和温度明显升高。

当前 EFI 保持：

```text
AppleXcpmForceBoost = false
```

最终方案由 CPUFriend 频率向量和 HWPRequestFix 运行时请求修正两部分组成。两者解决的问题不同，不能互相替代。

### CPUFriend 与 DataProvider

`CPUFriend.kext` 负责将 `CPUFriendDataProvider.kext` 中的定制平台数据注入 X86PlatformPlugin。当前 Provider 基于 `MacBookPro16,4` 的平台数据生成，保留三组 FrequencyVectors 变体，并将原本稀疏的频率状态扩展为 31 个状态：

```text
400, 600, 800, 1000,
1300, 1500, 1700,
2000, 2200, 2400, 2600, 2900,
3100, 3300, 3700, 3900,
4000, 4100, 4200, 4300, 4400, 4500, 4600, 4700,
4800, 4900, 5000, 5100, 5200, 5300, 5400 MHz
```

设计要点：

- 低频段使用约 200/300 MHz 的间隔，兼顾低负载降频和 31 项表长限制。
- 保留 E 核高频端需要的 `3100 / 3300 / 3700 / 3900 MHz` 状态。
- `4000-5400 MHz` 使用 100 MHz 间隔，为 P 核提供连续的高频请求档位。
- CPUFriend 的平台表是共享策略表，并不能单独定义一张 E 核专用频率表；E 核上限由 HWPRequestFix 按每个逻辑核心的 HWP capability 处理。
- 各新增状态的 deadline、idle 和 QoS 数据由相邻原生状态插值得到，不是简单向 plist 中填入频率数字。

CPUFriend 源码及本机频率表生成工具：<https://github.com/bafcz/CPUFriend>

### HWPRequestFix 0.2.2

HWPRequestFix 是一个依赖 Lilu 的内核插件，仅针对品牌字符串精确匹配的：

```text
13th Gen Intel(R) Core(TM) i9-13900HX
```

它不是轮询程序，也不持续写 MSR。插件在启动时执行以下操作：

1. 使用 Lilu 解析 Kernel Collection 中正在运行的内核映像。
2. 解析 `_xcpm_get_deadline`，在其前方最多 64 KiB 的 `__text` 范围内查找一对唯一的 HWP 请求指令序列。
3. 使用 Lilu `routeBlock` 在两处请求生成路径安装跳板。
4. 跳板保存全部通用寄存器和 RFLAGS，读取当前逻辑核心的 `IA32_HWP_CAPABILITIES`（MSR `0x771`），调用 C++ 请求转换逻辑，再恢复现场并继续执行原内核指令。

当前默认模式为 `scaled-intermediate`：

- 将 macOS 的 P 核 desired 请求按 Raptor Lake HWP 性能级映射转换，公式为 `(d * 325 + 254) >> 8`。
- 将旧的顶端 HWP 请求 `54` 转换到硬件性能级 `69`。这里的 `69` 是 HWP performance value，不是 6.9 GHz，也不是 CPU 倍频。
- 根据每个逻辑核心读取到的最高性能值区分 P/E 核。
- E 核使用原生 100 MHz 请求单位，并将 maximum/desired 限制在 `39`，对应 13900HX E 核最高 3.9 GHz。
- 保留 HWP minimum 和 EPP，不强制 CPU 常驻最高频率。

插件明确不修改：

- PL1 / PL2 和电流限制
- 电压或电压 offset
- BIOS Turbo Ratio Limits（本机 P 核为 `54, 54, 52, 52, 50, 50, 50, 50`）
- 温度保护与硬件降频机制
- CPUFriend 的 FrequencyVectors 内容

HWPRequestFix 源码：<https://github.com/bafcz/HWPRequestFix>

当前 0.2.2 已在 macOS 26、Darwin 25.6.0（构建 `25G241`）上完成启动和运行验证。OpenCore 中使用：

```text
MinKernel = 25.6.0
MaxKernel = 25.6.0
```

升级 macOS 后不要直接删除版本限制。应先针对新 kernel/BootKC 执行离线指令签名检查，确认 hook pair 仍然唯一，再进行受控启动测试。

### CPU 组件加载顺序

保持以下相对顺序：

1. `Lilu.kext`
2. `HWPRequestFix.kext`
3. `CPUFriend.kext`
4. `CPUFriendDataProvider.kext`

HWPRequestFix 和 CPUFriend 均依赖 Lilu。DataProvider 本身没有可执行文件，必须与 CPUFriend 同时启用。

### HWPRequestFix 运行状态验证

```sh
kmutil showloaded | grep -E 'Lilu|HWPRequestFix|CPUFriend'
ioreg -l -w0 | grep -A20 -B5 HWPRequestFix
pmset -g therm
```

正常状态应看到：

```text
Version = 0.2.2
CPUProfileMatched = Yes
RequestMode = scaled-intermediate
HookStatus = both-hooks-installed
Active = Yes
```

如果 `Active = No`，不要只根据 kext 已加载就判断补丁生效，应继续检查 `HookStatus`。定位失败、指令序列不唯一或只安装成功一个 hook 时，插件会保持关闭状态，避免半安装状态修改 HWP 请求。

### 频率显示说明

Intel Power Gadget 中的 `REQ`、`MAX` 或 Core Req 是 HWP 请求/性能级的解释结果，不一定等于某个核心当下的真实物理频率。尤其是转换后的 HWP performance value 可能被工具按 100 MHz 错误换算为 6.x GHz；这不代表 CPU 在 6 GHz 以上运行或发生超频。判断实际频率应结合每核心 APERF/MPERF、有效倍频、功耗和温度数据。

## 已验证状态

当前运行环境已确认：

- 触摸板使用 GPIO8 硬件中断，普通点击、滚动和多点触控正常；双击拖动不可用，可使用三指拖移替代。
- `VoodooGPIORaptorLakeS` 正确匹配 `INTC1085`。
- 不加载 `SSDT-TPD0-GPIO.aml` 时，TPD0 仍被识别为 GPIO 中断模式。
- 内置摄像头可被系统识别为 UVC 设备，但 Photo Booth 主界面暂不显示画面。
- HWPRequestFix 0.2.2 报告 `both-hooks-installed`、`Active = Yes`。
- CPU 可按负载降频，并可恢复 13900HX 的 5.4 GHz 轻载最高睿频。
- `AppleXcpmForceBoost` 保持关闭，避免空闲常驻高频。

## 维护注意事项

- 不要用官方 VoodooI2C 包直接覆盖当前 `VoodooI2C.kext`，否则会同时丢失 I2C PCI ID 和自定义 VoodooGPIO。
- 更新 VoodooI2C 时，需要重新加入 `0x7a7c8086&0xFFFCFFFF`，并重新嵌入本机的 VoodooGPIO 1.1.5 或其后续兼容版本。
- OpenCore 更新后重新核对 kext 加载顺序和插件路径。
- CPUFriendDataProvider 与当前 FrequencyVectors 是成套生成的，不要只替换 CPUFriendDataProvider。
- HWPRequestFix 目前只允许 Darwin 25.6.0 加载；新内核必须重新验证。
- 保留一份不含实验补丁、可以进入 OpenCore 的备用 EFI。
- 本仓库当前工作区即为文档依据；已移除的 `SSDT-TPD0-GPIO.aml` 不属于当前方案。

## 鸣谢

- [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)
- [Lilu](https://github.com/acidanthera/Lilu)
- [CPUFriend](https://github.com/acidanthera/CPUFriend)
- [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C)
- [VoodooGPIO](https://github.com/VoodooI2C/VoodooGPIO)
