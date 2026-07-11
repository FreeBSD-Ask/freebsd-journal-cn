# SVN 动态

- 作者：**Steven Kreuzer**

新年伊始，我们已迎来几项令人兴奋的更新，凸显了 FreeBSD 对高性能网络的持续投入。此外，FreeBSD 正迅速成为虚拟化项目中成熟而强大的平台，无论作为裸机上的宿主机，还是云中的客户机。你很快就能将生产机器升级到 10.3-RELEASE，与此同时，也应密切关注 HEAD，因为 2016 年对 11-CURRENT 来说将是令人激动的一年。

## 支持异步 I/O 的新 sendfile(2)

进入 HEAD 的最令人兴奋的变化，是新增了 **sendfile(2)** 系统调用的实现。这是 NGINX 与 Netflix 持续合作的成果，新的 sendfile 通过支持异步 I/O，显著加速了大型 TCP 数据传输。更妙的是，新 sendfile 可直接替换旧版，因此你无需修改应用即可利用这些改进。在旧版 sendfile 因磁盘 I/O 阻塞的场景下，现在能获得显著更好的性能。（<https://svnweb.freebsd.org/changeset/base/293439>）

## bhyve 客户机支持 Netmap

bhyve 的开发依然十分活跃，并迅速获得广泛采用。最近的一次提交有助于解决困扰所有虚拟化部署的最大短板。网络 I/O 虚拟化往往性能较差，在重负载下尤甚，而直到最近，消除这些瓶颈仍非易事。在 bhyve 下运行的客户机操作系统现在可以利用 netmap（高速数据包 I/O 框架）获得接近原生的性能。（<https://svnweb.freebsd.org/changeset/base/293459>）

## EC2 增强型网络默认启用

也称为 SR-IOV（Single Root I/O Virtualization，单根 I/O 虚拟化），这项 PCIe 规范的扩展让网卡等设备对 hypervisor 或客户机操作系统而言呈现为多个独立的物理设备。SR-IOV 使网络流量绕过虚拟化栈，从而获得几乎与非虚拟化环境相同的网络性能。在为 Amazon Web Services 构建 EC2 镜像时，SR-IOV 现已默认启用。（<https://svnweb.freebsd.org/changeset/base/293739>）

## LLDB 在 amd64 和 arm64 上默认启用

LLDB 是下一代高性能调试器，以 BSD 风格许可证发布。大量测试表明其表现与树内 gdb 版本相当，随后它被提升为 amd64 和 arm64 上的默认调试器。LLDB 也对 FreeBSD 的 arm、mips、i386 和 powerpc 提供了一定程度的支持，但尚不能在这些平台上取代 gdb 作为默认调试器。（<https://svnweb.freebsd.org/changeset/base/292350>）

## ZFS 启动环境

如果系统以 ZFS 启动，loader 中会出现一个新菜单项，其中包含自动生成的 ZFS 启动环境列表，方便切换到备用根文件系统。如果你想在不同版本的 FreeBSD 之间切换，或需要从失败的升级中恢复，这会是非常实用的功能。（<https://svnweb.freebsd.org/changeset/base/293001>）UEFI loader 中也提供了 ZFS 启动环境功能。（<https://svnweb.freebsd.org/changeset/base/294073>）

## UEFI 新增终端模拟支持

基于既有的 vidconsole 实现，现在可以在 UEFI 中模拟视频终端。（<https://svnweb.freebsd.org/changeset/base/293233>）此项变更引入不久，Beastie 菜单也加入了 UEFI 控制台。（<https://svnweb.freebsd.org/changeset/base/293234>）

## head/contrib 更新

FreeBSD 基本用户空间由相当多的工具组成，其中一些工具在项目外部开发。过去几个月里，这些第三方软件有不少更新，它们共同带来了出色的用户体验。

- clang 和 LLVM 升级至 3.7.1 版本（<https://svnweb.freebsd.org/changeset/base/292735>）。
- less 更新至 v481 版本（<https://svnweb.freebsd.org/changeset/base/293190>）。
- ntp 更新至 4.2.8p5 版本（<https://svnweb.freebsd.org/changeset/base/293423>）。
- bmake 升级至 20151220 版本（<https://svnweb.freebsd.org/changeset/base/292733>）。
- OpenBSM 升级至 1.2a4 版本（<https://svnweb.freebsd.org/changeset/base/292432>）。
- Unbound 升级至 1.5.7 版本（<https://svnweb.freebsd.org/changeset/base/292206>）。

---

**Steven Kreuzer** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车情有独钟。他与妻子、女儿和一只狗住在纽约皇后区。
