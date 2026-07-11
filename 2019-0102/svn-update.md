# SVN 动态

- 原文链接：[svn Update](https://freebsdfoundation.org/our-work/journal/browser-based-edition/)
- 作者：**STEVEN KREUZER**

希望你的新年开局美好，并在假期抽空把机器升级到了 FreeBSD 12。虽说 2018 年投入 FreeBSD 的时间与精力可以说非同凡响，但我强烈预感 2019 年会是开发上更加激动人心的一年。今年还没满一个月，我已经看到一些非常有趣的改动被提交到 HEAD。我热切期待未来会带来什么，希望你也怀有同感。

## **vmm(4)**：在 AMD 主机上屏蔽 Spectre 特性位 —— <https://svnweb.freebsd.org/changeset/base/343166>

为了与 Intel 主机保持一致——后者已经屏蔽掉指示 SPEC_CTRL MSR 存在的 CPUID 特性位——AMD 上做同样处理。最终我们可能需要为客户机提供更好的支持方案，但眼下先限制因错误地暴露尚未支持的 MSR 而造成的损害。

## nvdimm：为 NVDIMM 根设备添加驱动 —— <https://svnweb.freebsd.org/changeset/base/343143>

NVDIMM 根设备是各个 ACPI NVDIMM 设备的父设备。为 NVDIMM 根设备添加驱动，使其能够枚举系统中的 NVDIMM 设备、NVDIMM SPA 范围。

## **vmm(4)**：向多核 bhyve AMD 支持迈进 —— <https://svnweb.freebsd.org/changeset/base/343075>

bhyve 的 CPUID 仿真向客户机呈现 Intel 拓扑信息，却禁用了 AMD 拓扑信息，某些情况下还把垃圾数据透传过去。例如，CPUID leaves 0x8000_001[de] 被透传给客户机，但客户机 CPU 可以在线程间迁移，因此呈现的信息并不一致。这一点可以轻易通过 `cpucontrol -i 0xfoo /dev/cpuctl0` 观察到。

通过启用 AMD 拓扑特性标志位，并至少呈现 FreeBSD 自身在更现代的 AMD64 硬件（Family 15h+）上探测拓扑所用的 CPUID 字段，对这一情况进行轻微改善。更老的硬件大概不那么值得关注。我未能通过实验确认它是否足够，但也不会造成回归。

## 修复 bhyve 的 NVMe Completion Queue 入口值 —— <https://svnweb.freebsd.org/changeset/base/342762>

处理 Admin 命令的函数未在 Completion Queue Entry 的 Dword 0（CDW0）中返回 Command Specific 值。这影响诸如 Set Features、Number of Queues 这类在 CDW0 中返回设备所支持队列数的命令。在此情况下，主机只会创建 1 个队列对（Number of Queues 从零开始计数）。这也掩盖了队列计数逻辑中的一个 bug。

## 创建新的 EINTEGRITY 错误，提示信息为“Integrity check failed” —— <https://svnweb.freebsd.org/changeset/base/343111>

完整性检查（如校验和或互相关性检查）失败。该完整性错误位于标识系统调用参数错误的 EINVAL 与标识底层存储介质错误的 EIO 之间。EINTEGRITY 通常由文件系统或内核内 GEOM 子系统等中间内核层在检测到不一致时引发。用途之一是允许 **mount(8)** 命令返回不同的退出值，从而在系统启动时自动执行 **fsck(8)**。

这些改动并未使用新的错误，只是将其加入。后续会有提交使用该新错误号，并视情况加入更多手册页。

## 添加对将中断处理程序标记为已挂起的支持 —— <https://svnweb.freebsd.org/changeset/base/342170>

本次改动的目标是修复 PCI 共享中断在挂起与恢复期间的问题。我曾观察到以下场景的若干变种。设备 A 与设备 B 位于同一 PCI 总线并共享同一中断。设备 A 的驱动先被挂起，设备下电。设备 B 触发中断。两个驱动的中断处理程序都被调用。设备 A 的中断处理程序访问已下电设备的寄存器，得到无效值（我假定全为 0xff）。这些数据被解释为中断状态位等等。于是中断处理程序陷入混乱，可能产生噪音或进入死循环等。

本次改动仅影响 PCI 设备。**pci(4)** 总线驱动在调用子设备的挂起方法之后、设备下电之前，将子设备的中断处理程序标记为已挂起。仅对传统 PCI 中断执行此操作，因为只有它们可共享。

## 优化 RISC-V 的 **copyin(9)**/**copyout(9)** 例程 —— <https://svnweb.freebsd.org/changeset/base/343275>

RISC-V 上现有的 **copyin(9)** 与 **copyout(9)** 例程仅做简单的逐字节复制。在可能的情况下按字复制以提升性能。

---

**STEVEN KREUZER** 是 FreeBSD 开发者与 Unix 系统管理员，对复古计算与风冷大众汽车情有独钟。他与妻子、女儿和狗居于纽约皇后区。
