# 使用 CHERI 调试

## CHERI 旨在提升既有与未来软件的安全性。

### John Baldwin

[CHERI](https://cheri.cst.cam.ac.uk/) 是一种面向安全的新型处理器技术。CHERI 借鉴历史能力系统（capability system）的原理，通过扩展 C/C++ 的隐式与显式指针，使其包含边界和权限等元数据，从而提升用 C 和 C++ 编写的既有软件的安全性。例如，在 CHERI 架构上，调用 `malloc()` 返回的指针会被限定在所请求的大小范围内，若试图访问分配边界之外的数据，将触发处理器异常。类似地，指向栈上缓冲区的指针在传递给另一函数时，会被限定在该缓冲区的大小范围内，而非授予对整个线程栈的访问权限。为存储这些额外信息，CHERI 定义了一种新的硬件类型：CHERI capability（能力）。

CHERI 能将某些未定义行为（UB）转化为崩溃，从而缓解漏洞。这种缓解从安全角度看很有价值，但 CHERI 还能辅助调试，既能为用户提供更多信息（例如内存分配的边界），也能更早捕获错误。

随着时间推移，支持 CHERI 的软件库不断壮大，包括 FreeBSD 的完整移植版——[CheriBSD](https://www.cheribsd.org/)，以及若干第三方应用，其中涵盖 KDE 图形桌面环境的大部分组件。CHERI 研究团队还维护着 GNU 调试器的一个[分支](https://github.com/CTSRD-CHERI/gdb)，支持检查 CHERI 架构（包括 ARM 的 Morello 和 CHERI-RISC-V）。CHERI GDB 既支持调试 CheriBSD 用户进程，也支持调试内核。由于 CheriBSD 紧跟 FreeBSD 的开发进展，针对 FreeBSD 报告的 bug 通常也能在 CheriBSD 上复现并排查。

本文将考察 FreeBSD bug 数据库中几份高质量 bug 报告在 CheriBSD 上的排查过程，借此展示在 CHERI 上调试的体验，以及使用 CHERI 时可能遇到的一些出乎意料的结果。排查期间，我使用了运行在 [ARM Morello](https://www.arm.com/architecture/cpu/morello) 系统上的 CheriBSD，以及在我 FreeBSD/amd64 桌面机上通过 QEMU 运行的 C