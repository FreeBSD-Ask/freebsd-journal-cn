# 嵌入式控制器的 ACPI 支持

- 原文链接：[ACPI Support for Embedded Controllers](https://freebsdfoundation.org/wp-content/uploads/2022/04/ACPI-Support-for-Embedded-Controllers.pdf)
- 作者：**MARCIN WOJTAS**
  

ARM64 是一种被广泛应用于各种产品的单一架构——它既可用于最小型的嵌入式设备，也可用于移动设备、企业级产品，甚至是服务器级解决方案。对服务器级设备的支持要求遵循一定的标准，例如启动方式、与固件的交互以及平台描述等。而事实证明，这种模式同样适用于非服务器设备。本文将探讨 FreeBSD 对这些设备的支持情况，重点关注 ACPI 体系下的嵌入式控制器处理方式。  

## 处理遗留问题  

近十年前，64 位 ARM 架构问世时，直接继承了其 32 位前身的生态系统，而 32 位 ARM 在嵌入式市场中长期占据主导地位。然而，为每个平台维护一个完全定制化的板级支持包（BSP）成为新架构发展的沉重负担。  

在一定程度上，设备树（DT）的引入提高了可移植性，并使得可以使用单一内核镜像运行多个设备，但这并未彻底解决问题。设备树的描述方式十分灵活，不幸的是，这种灵活性经常被厂商滥用，导致设备树绑定在时间推移中变得不一致。哪怕在今天，U-Boot 使用的设备树 blob 仍然可能与操作系统引导时使用的设备树不同（尽管它们描述的是同一块硬件！），而且通常缺乏向后兼容性。在这样的限制下，想要实现广泛的软件生态系统和多操作系统支持将变得十分困难。  

不过，早已有成熟的解决方案存在。x86 体系所使用的接口被引入并扩展到了 ARM64，包括启动过程、EFI、SMBIOS 和 ACPI。在符合这些标准并使用合适固件的服务器级设备上，现在可以直接使用安装镜像来安装 FreeBSD、其他操作系统或虚拟机管理程序（hypervisor）。  

那么，小型嵌入式平台呢？幸运的是，它们同样可以以类似的方式利用这一成熟生态系统。当然，这需要一定的前提——硬件不能过于偏离标准，并且固件需要满足严格的要求。这些规范被归纳在一系列标准中，即 [BSA](https://developer.arm.com/documentation/den0094/latest)（ARM 基本系统架构）和 [BBR](https://developer.arm.com/documentation/den0044/latest)（ARM 基础启动要求）。最终的结果是，现在已经有一些 ARM64 平台可以通过单一固件镜像和 ACPI 描述成功引导 FreeBSD、Windows 和多个 Linux 发行版。

这些设备有何特殊之处？与传统服务器（通常拥有大量 CPU、DRAM 和 PCIe 根复合体）相比，嵌入式领域的 SoC 还支持连接到其内部总线的各种控制器。因此，这些控制器不会在 PCIe 枚举过程中被自动发现，而是需要采用不同的处理方式。硬件描述必须明确引用这些接口，并包含能够被操作系统解析和解释的平台数据。最近，FreeBSD 内核的能力得到了扩展，新增了一些特性，使其能够从 ACPI 表中获取这些信息。  

## 什么是 ACPI？  

在深入讨论细节之前，值得简要介绍一下 ACPI（高级配置与电源管理接口）。ACPI 是固件与操作系统之间的接口，主要用于描述和配置硬件。该[标准](https://uefi.org/specs/ACPI/6.4/index.html)已经发展了近 30 年，并涵盖了多个关键[概念](https://uefi.org/specs/ACPI/6.4/03_ACPI_Concepts/ACPI_Concepts.html#acpi-concepts)，例如电源管理、温度/电池管理、硬件配置以及嵌入式控制器的描述。ACPI 还定义了一种 ACPI 源语言（ASL），它能创建低级硬件配置例程，并被编译为 ACPI 机器语言（[AML](https://uefi.org/specs/ACPI/6.4/20_AML_Specification/AML_Specification.html)），由内核解释并执行。  

平台信息存储在所谓的“表”（table）中，这些表本质上是系统内存地址空间中的一组层次化结构。ACPI 的起点是根系统描述指针（RSDP），它由固件配置，并指向扩展系统描述表（XSDT），后者进一步链接到次级表。首个次级表始终是固定 ACPI 描述表（FADT），其中包含描述硬件固定 ACPI 特性的各种固定长度条目。

![](https://github.com/user-attachments/assets/051fa3e1-faf8-40aa-863e-bec5ee50dd65)

**图 1. 根系统描述指针 (RSDP) 和表**  

来源：[UEFI ACPI 6.4 规范](https://uefi.org/specs/ACPI/6.4/05_ACPI_Software_Programming_Model/ACPI_Software_Programming_Model.html#overview-of-the-system-description-table-architecture)

ACPI 规范定义了多种[专用表](https://uefi.org/specs/ACPI/6.4/05_ACPI_Software_Programming_Model/ACPI_Software_Programming_Model.html#acpi-system-description-tables)，但在嵌入式设备的上下文中，有几种表格尤为重要，例如：

- **通用定时器描述表 (GTDT)**  
- **多 APIC 描述表 (MADT)**  
- **处理器属性拓扑表 (PPTT)**  
- **串口控制台重定向表 (SPCR)**  
- **PCI Express 内存映射配置空间基地址描述表 (MCFG)**  
- **差异化系统描述表 (DSDT)**  

上述表格中的最后一项——DSDT，尤为关键。DSDT 始终由 FADT 引用，并包含 CPU 列表、电源管理特性、PCIe 根复杂 (root complex) 以及所有嵌入式控制器的描述。通常，它会配有 SSDT (次级系统描述表)，可能有一个或多个实例。这种结构允许程序员在平台描述代码中逻辑地拆分各种功能。  

上述表格的定义已扩展，以涵盖 ARM64 特定的值和类型（例如中断控制器）——所有这些都被汇总到 **ACPICA (ACPI 组件架构)** 中。ACPICA 是一个开源参考代码库，供各操作系统使用和补充。FreeBSD 始终与其最新版本保持同步。现在，让我们看看这些表格在 FreeBSD ARM64 端口中的处理方式。  



## ARM64 的 ACPI 基础部分  

ARM64 SoC 依照标准由 ACPI 表格描述。例如，GTDT 列出了定时器和看门狗，MADT 则包含中断控制器（目前仅支持 GICv2 和 GICv3）。进一步来看嵌入式控制器，控制台由 SPCR（可选 DBG2 表）描述——推荐使用 **ARM SBSA UART (PL011)** 或兼容 **16550** 的 UART，不过近年来 ARM 生态中的更多 UART 类型也已被纳入支持[列表](https://docs.microsoft.com/en-us/windows-hardware/drivers/bringup/acpi-debug-port-table#table-3-debug-port-types-and-subtypes)。

PCIe 控制器的描述更为复杂，必须包含在 MCFG 和 DSDT/SSDT 表中。对于 **ARM64**，唯一允许的类型是完全兼容标准 **ECAM (Enhanced Configuration Access Mechanism)** 的通用 PCIe 控制器，该类型受 **[pci_host_generic_acpi](https://cgit.freebsd.org/src/tree/sys/dev/pci/pci_host_generic_acpi.c)** 驱动程序支持。建议新设计在芯片中保留未经修改的标准 **ECAM** 实现，但对于现有产品，往往无法满足这一要求。因此，在 FreeBSD 的 **pci_host_generic_acpi** 驱动中，已允许通过[配置空间访问 quirks](https://cgit.freebsd.org/src/commit/?id=2de4c7f6d08798fb6269582907155703d1ab5ef4) 来处理对标准的偏离。另一种解决方案是通过 **SMCCC (Secure Monitor Call Calling Convention)** 接口，支持从固件执行低级例程。目前，该机制已在[树莓派 4](https://github.com/ARM-software/arm-trusted-firmware/commit/ab061eb732dcd2ee711b6960c37c4b25c44f3f9d) 上可用，但 FreeBSD 仍未实现该选项。

## 处理嵌入式控制器  

连接到 **SoC** 内部总线的嵌入式控制器在 **ACPI** 表中可以通过两种方式进行处理：  

1. 使用“方法” (methods) 编译为 AML—— 这样 **OS** 可以直接解析并执行相应指令。例如，[热管理](https://uefi.org/specs/ACPI/6.4/11_Thermal_Management/Thermal_management.html) (thermal management)、[SMBUS](https://uefi.org/specs/ACPI/6.4/12_ACPI_Embedded_Controller_Interface_Specification/smbus-devices.html) 或 [GPIO](https://uefi.org/specs/ACPI/6.4/19_ASL_Reference/ACPI_Source_Language_Reference.html#gpioio-gpio-connection-io-resource-descriptor-macro) 就采用这种方式。  
2. 使用标准对象 (Standard Objects) 描述—— 对于 **ACPI** 规范未明确定义的设备或子系统，可利用 **ACPI** 提供的标准对象，让操作系统解析并获取驱动程序所需的所有硬件资源。  

第二种方式是 **ACPI** 支持非服务器级 ARM64 SoC 的关键，并且已在 **FreeBSD** 内核中得到实现。

![](https://github.com/user-attachments/assets/54f31e12-ac61-4356-a94e-17c2fe9dc2e0)

**图 2. ACPI 和设备树世界中 FreeBSD 总线层次结构示例的高层比较。**

从高层来看，**FreeBSD** 中嵌入式控制器的总线层级在 **ACPI** 和 **DT (Device Tree)** 体系下是相似的（参见 **图 2**）。这对于设计设备驱动程序非常有帮助，因为在内核初始化过程中，平台数据结构可以在两种情况下以相同方式填充。随后，已探测到的驱动程序可以通过 **ACPI** 的 `_HID` 字段值进行匹配，该字段可以视为 **Device Tree** 中 **compatible** 字符串的等效项。此外，其他标准类型的资源也以类似方式进行处理。  

**FreeBSD** 目前支持的 **ACPI** 方式管理的 **ARM64** 嵌入式控制器主要有 [USB](https://cgit.freebsd.org/src/tree/sys/dev/usb/usb_generic.c) 和 [SATA](https://cgit.freebsd.org/src/tree/sys/dev/ahci/ahci_generic.c)。其中，**SATA** 的匹配方式较为特殊，并非通过 `_HID` 进行匹配，而是通过 **设备类值 (device class value)** 进行匹配，即 **ACPI** `_CLS` 对象（参见 **列表 1**）。

```c
Device(AHC0) {
    Name(_HID, "LNRO001E") // _HID: 硬件 ID
    Name(_UID, 0x00) // _UID: 唯一标识 ID
    Name(_CCA, 0x01) // _CCA: 缓存一致性属性
    Method(_STA) // _STA: 设备状态
    {
        Return(0xF)
    }
    Name(_CLS, Package(0x03) // _CLS: 类别代码
    {
        0x01,
        0x06,
        0x01
    }) Name(_CRS, ResourceTemplate() // _CRS: 目前的资源设置
    {
        Memory32Fixed(ReadWrite, 0xF2540000, // 基地址 (MMIO)
                      0x00030000, // 地址长度
                     ) Interrupt(ResourceConsumer, Level, ActiveHigh, Exclusive,,, ) {
            CP_GIC_SPI_CP0_SATA_H0
        }
    })
}
```

**清单 1. [ACPI 表](https://github.com/tianocore/edk2-platforms/blob/master/Silicon/Marvell/OcteonTx/AcpiTables/T91/Cn913xDbA/Dsdt.asl#L57)中的 AHCI 控制器描述示例**

**FreeBSD** 的 **XHCI** 和 **AHCI** 驱动程序需要在 **DSDT/SSDT** 中使用完全标准化的描述。**列表 1** 展示了 **XHCI** 控制器的一个示例，它包含引用 **唯一 ID** 的对象，以及 **缓存一致性信息** 和 **内存/中断资源**。所有 **非标准** 配置（例如 **寄存器映射、时钟管理** 或 **电源管理** 处理）都必须由 **固件** 预先配置并实现。  

## 自定义 ACPI 描述

如果某个控制器需要由其专用驱动程序处理的自定义绑定，该如何实现？  

在 FreeBSD 中，过去只能在 **DT (Device Tree)** 体系下实现自定义绑定，方法是使用节点属性。但 **ACPI** 规范实际上定义了一个可选对象 `_DSD` ([Device Specific Data](https://uefi.org/specs/ACPI/6.4/06_Device_Configuration/Device_Configuration.html#dsd-device-specific-data))，它可以包含相同的信息。  

借助 **FreeBSD** 的总线层级体系（参见 **图 2**），开发者[设计并实现了](https://cgit.freebsd.org/src/commit/?id=3f9a00e3b577)一种新的通用方案，支持在不依赖具体描述方式的情况下获取控制器特定的数据。此外，还引入了以下辅助函数：  

- [`device_get_property`](https://www.freebsd.org/cgi/man.cgi?query=device_get_property&apropos=0&sektion=9&manpath=FreeBSD+14.0-current&arch=default&format=html)
- `device_has_property`

这些函数允许子设备驱动以相同方式访问父总线提供的设备特定数据，无论系统是以 **ACPI** 还是 **DT** 启动，都能执行相同的代码路径。该方案后来[进一步扩展](https://cgit.freebsd.org/src/commit/?id=b344de4d0d16)，以涵盖 **ACPI** 和 **DT** 体系下的多种属性类型。  


该方案的一个典型应用是 **SD/MMC** 子系统，其中包括：  

- [通用代码](https://cgit.freebsd.org/src/commit/?id=8a8166e5bcfb)
- 适用于 Marvell Xenon 控制器的驱动  

其中，Marvell Xenon 驱动被拆分为三个文件：  

1. [通用部分](https://cgit.freebsd.org/src/tree/sys/dev/sdhci/sdhci_xenon.c)  
2. [ACPI](https://cgit.freebsd.org/src/tree/sys/dev/sdhci/sdhci_xenon_acpi.c) 适配代码  
3. [simplebus](https://cgit.freebsd.org/src/tree/sys/dev/sdhci/sdhci_xenon_fdt.c) 适配代码（适用于 **Device Tree**）  

除了 `DRIVER_MODULE`/`DEFINE_CLASS_1` 宏 的不同使用方式，simplebus 适配代码还额外解析了电源管理 (regulators) 和卡检测 GPIO 引脚 (card detect GPIO pins)，而 **ACPI** 方式下这些信息则由固件预先设定。

```c
&ap_sdhci0 {
    compatible = "marvell,armada-cp110-sdhci";
    reg = <0x780000 0x300>;
    interrupts = <27 IRQ_TYPE_LEVEL_HIGH>;
    clock-names = "core", "axi";
    clocks = <&CP11X_LABEL(clk) 1 4>, <&CP11X_LABEL(clk) 1 18>;
    dma-coherent;
    bus-width = <8>;
    /*
     * 在 HS 模式下不稳定 —— PHY 需要“进一步校准”，因此添加“slow-mode”，并禁用 SDR104、SDR50 和 DDR50 模式。
    */
    marvell,xenon-phy-slow-mode;
    no-1-8-v;
    no-sd;
    no-sdio;
    non-removable;
    status = "okay";
    vqmmc-supply = <&v_vddo_h>;
};
```

**清单 2. 设备树中的 Marvell Xenon SD/MMC 控制器**

```c
Device (MMC0)
{
    Name (_HID, "MRVL0002") // _HID: 硬件 ID
    Name (_UID, 0x00) // _UID: 唯一标识 ID
    Name (_CCA, 0x01) // _CCA: 缓存一致性属性
    Method (_STA) // _STA: 设备状态
    {
        Return (0xF)
    }
    Name (_CRS, ResourceTemplate () // _CRS: 当前资源设置
    {
        Memory32Fixed (ReadWrite,
                       0xF06E0000, // 基地址 (MMIO)
                       0x00000300, // 地址长度
                      )
        Interrupt (ResourceConsumer, Level, ActiveHigh, Exclusive,,, )
        {
            48
        }
    })
    Name (_DSD, Package () {
        ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
        Package () {
            Package () { "clock-frequency", 400000000 },
            Package () { "bus-width", 8 },
            Package () { "marvell,xenon-phy-slow-mode", 0x1 },
            Package () { "no-1-8-v", 0x1 },
            Package () { "no-sd", 0x1 },
            Package () { "no-sdio", 0x1 },
            Package () { "non-removable", 0x1 },
        }
    })
}
```

**清单 3. ACPI 中的 Marvell Xenon SD/MMC 控制器**

**代码示例 2 和 3** 展示了同一控制器实例的 DT（设备树）和 ACPI 节点，以说明这两种描述方式的相似性。借助 FreeBSD 内核的新方法，该控制器可以在所有固件配置下正常运行。  

## 结论  

随着 FreeBSD 内核的最新改进，嵌入式产品中使用的 ARM64 SoC 现在可以同时通过 ACPI 和设备树（DT）以类似的方式获得支持。对于 ACPI 而言，自定义控制器的分层表示也被证明足够灵活，能够适用于大多数类型的设备和子系统。例如，一些复杂的网络控制器和通用 MDIO 层的实现都已经验证了这一点。  

FreeBSD 现在可以无限制地继续沿着这条路径发展，尤其是其总线架构允许以干净且优雅的方式进行扩展，就像 **SD/MMC 控制器** 示例所展示的那样。  

---  

**MARCIN WOJTAS** 是 **Semihalf 工程主管**，同时也是 **FreeBSD src 提交者（mw@）**。他对嵌入式软件和硬件充满热情，并为多个开源项目作出贡献，包括 **Linux 内核、Tianocore EDK2 和 TF-A**。
