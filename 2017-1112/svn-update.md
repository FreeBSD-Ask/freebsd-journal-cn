# SVN 动态

svn 更新  作者：Steven Kreuzer

过去几期 SVN 动态都围绕某个主题展开，我也尽力寻找与特定主题相符的近期提交。这次苦于写作瓶颈，我决定把本月主题定为”我感兴趣的杂集”——我想你也能把它看作一个主题。

我希望你能像我一样享受这批随机挑选的有趣提交。希望新年到来时，我的脑子能恢复得更好。期待又一年精彩的 FreeBSD 开发！

## 使用估算 RTT 而非时间戳进行接收缓冲区自动调整

<https://svnweb.freebsd.org/changeset/base/316676>

从使用时间戳切换到使用 RTT 估算进行 TCP 接收缓冲区自动调整，因为并非所有主机都支持/启用 TCP 时间戳。在非批量接收模式下禁用接收缓冲区自动扩展重置，可带来额外 20% 的性能提升。

## amazon-ssm-agent package 默认安装在 EC2 AMI 构建中

<https://svnweb.freebsd.org/changeset/base/325254>

这使它在没有互联网访问（或因其他原因不能依赖 firstboot_pkgs 安装）的实例上立即可用。

## 支持压缩的内核转储

<https://svnweb.freebsd.org/changeset/base/324965>

使用以 GZIO 配置选项构建的内核时，`dumpon -z` 可用内核内的 zlib 副本配置 gzip 压缩。这在拥有大量 RAM、需要相应较大转储设备的系统上很有用。由于需要从转储设备复制的字节数更少，压缩转储的恢复也更快。

## 支持给 md(4) 设备打标签

<https://svnweb.freebsd.org/changeset/base/322969>

这一功能源于我们在构建过程中大量使用内存支持的 md(4)。不过，如果构建出问题，分配的资源（即 swap 和内存支持的 md(4)）需要被清理。能够给每个虚拟磁盘附加任意标签非常实用，便于在必要时识别和清理。

## 支持 Intel Software Guard Extensions（Intel SGX）

<https://svnweb.freebsd.org/changeset/base/322574>

Intel SGX 允许在用户虚拟地址空间中管理隔离的”Enclave”区段。Enclave 内存是处理器保留内存（PRM）的一部分，且始终加密。这能保护用户应用代码和数据，使其免受更高特权级别（包括操作系统内核）的访问。

## geli 口令长度在启动时被隐藏

<https://svnweb.freebsd.org/changeset/base/322923>

## CloudABI 兼容性已同步到 0.13 版本

<https://svnweb.freebsd.org/changeset/base/322885>

随着 Flower（CloudABI 的网络连接守护进程）日趋完善，不再需要创建任何未连接的套接字。套接字对配合文件描述符传递即可满足所需，因为 Flower 正是用它把网络连接从公共互联网传递给监听进程。

## 从 OpenZFS 导入对 ZFS Channel Programs 的支持

<https://svnweb.freebsd.org/changeset/base/324163>

ZFS Channel Programs（ZCP）增加通过 Lua 脚本在沙箱环境（含时间和内存限制）中执行复合 ZFS 管理操作的支持。初始提交既包含运行 ZCP 脚本的基础支持，也包含一个小型 API 调用库，支持获取属性以及列出、销毁和提升数据集。

## 支持暂停和恢复 zpool scrub

<https://svnweb.freebsd.org/changeset/base/323355>

scrub 可能是一项 I/O 密集型操作，人们一直在要求能暂停 scrub 一段时间。这能保留 scrub 进度，同时为其他 I/O 释放带宽。

## 支持设置 VGA 渲染器的光标颜色

<https://svnweb.freebsd.org/changeset/base/322878>

## 新增用于实验 SDIO 卡的工具

<https://svnweb.freebsd.org/changeset/base/320847>

## 新增 mmcnull，一个模拟的轻量级 MMC 控制器

<https://svnweb.freebsd.org/changeset/base/320845>

该模拟设备 attach 到 ISA 总线并注册为支持 MMC/SD 卡的 HBA。这使 MMC XPT 和 MMC/SDIO 外设驱动的开发和测试即便在 bhyve 等虚拟机中也能进行。

> STEVEN KREUZER 是一名 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗住在纽约皇后区。
