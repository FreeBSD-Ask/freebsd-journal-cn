# SVN 动态

作者：**Steven Kreuzer**

2014 年，作为谷歌 Summer of Code 项目，FreeBSD 开发者和学生共同设计并实现了一个用于 loader 脚本解释器的模块化接口——将解释器与 loader 解耦。历经四年，它终于被提交！FreeBSD 12 将附带一个全新的、基于 Lua 的 bootloader，这将使大多数人说早已“待得太久”的忠实 Forth loader 退出舞台。本专栏的多数文章倾向于涵盖各子系统的开发进展，但这一期我想专注于 bootloader，因为我从未见过如此低级别的系统组件上有这么多活动。

## 向 /boot/loader 添加 Lua 作为脚本语言

<https://svnweb.freebsd.org/changeset/base/329166>

`liblua` 将 Lua 运行时嵌入 boot loader。它实现了 Lua 所期望的所有运行时例程。此外，它还有一些标准的 C 头文件，会“中和”LUA 构建中那些过于 Lua 特定、不适合放入 `libsa` 的部分。相对原始代码做了大量改进，提升了实现质量并增加了所包含的 Lua 库数量。使用 `int64_t` 作为 `lua_Number`。将 **/boot/lua** 设为默认模块路径。

原始 GSoC 项目的大量清理工作，包括修改 `libsa`，使 Lua 仅在 `luaconf.h` 之外做一处修改即可构建。添加 Lua glue 的最后一块代码以引入 `liblua`，并接入先前已提交的多解释器框架。

目前这是一个实验性选项。必须通过定义 `WITH_LOADER_LUA` 和 `WITHOUT_FORTH` 显式启用。它只经过轻量测试，所以请保留旧 loader 的备份。下个提交到来的菜单代码尚未经过全面测试。LUA bootloader 比 FORTH loader 大 60k，而后者比无解释器的 loader 大 80k。大小的细微变化可能使一些微妙的界限被突破（用 LUA 构建时二进制约为 430k）。未来版本可能提供共存方案。

将 FreeBSD 版本号提升至 1200058 以标记这一里程碑。

## 添加 Lua-bootloader SoC 的 Lua 脚本

<https://svnweb.freebsd.org/changeset/base/329167>

这些是 Pedro Souza 2014 年 Summer of Code 项目的 `.lua` 文件。Rui Paulo、Pedro Arthur 和 Wojciech A. Koszek 也参与了贡献。

## 将内核/模块加载延迟到 boot 或菜单退出时

<https://svnweb.freebsd.org/changeset/base/329576>

加载内核和模块可能非常慢。在菜单绘制前加载、每次更改内核/启动环境时都加载更加痛苦。将加载延迟到 boot、auto-boot 或退出到 loader 提示符时。我们仍需处理启动环境变化带来的配置变更，但通常快得多。此提交从 `config.load`/`config.reload` 中剥离了所有 ELF 加载逻辑，使它们纯粹用于配置。新增 `config.loadelf` 用于处理内核/模块加载。卸载逻辑已被移除，因为菜单中不再需要处理它。

## 创建“旋转木马”菜单项类型

<https://svnweb.freebsd.org/changeset/base/329367>

这是 lualoader 中启动环境支持的前置工作。新增一种菜单项类型 `carousel_entry`，通常提供一个回调以获取条目列表、一个 `carousel_id` 用于存储当前值和标准的 `name`/`func` 函数。与普通菜单项在功能上的区别在于，选择旋转木马条目时会自动在可用条目间循环，并在列表耗尽时回到开头。

## 重构菜单跳过逻辑

<https://svnweb.freebsd.org/changeset/base/330020>

这是为了减少菜单被跳过时的堆使用量。目前，无论菜单是否被跳过都必须加载菜单模块，这增加了约 50–100KB 的内存使用。将菜单跳过逻辑移到核心代码中（并移除一条调试 print），然后在 `loader.lua` 中检查是否应跳过菜单，避免加载菜单模块。这使得菜单被剥离时内存使用量保持在约 115KB 以下。

## 修复多内核下的 module_path 处理

<https://svnweb.freebsd.org/changeset/base/329497>

成功加载内核后，我们会将其目录添加到 `module_path`。如果通过内核选择器切换内核，我们又会把内核目录前置到当前 `module_path`，最终出现多个内核路径，可能向 `module_path` 添加了不匹配的内核/模块。修复方法是在 `load()` 时缓存 `module_path`，并在加载新内核时使用缓存版本。

## 退出菜单时从菜单视角使屏幕失效

<https://svnweb.freebsd.org/changeset/base/329986>

在常见情况下，这实际上什么也不会做，因为无论屏幕是否被标记为无效，菜单在离开子菜单时都会被重绘。但是，当退回到 loader 提示符时，可以通过以下两种方式之一重新进入菜单系统：

方法 1：

```lua
require('menu').run()
```

方法 2：

```lua
require('menu').process(menu.default)
```

使用方法 1 时，菜单无论如何都会被重绘，因为我们在进入时会在 autoboot 检查之前做这件事。但使用方法 2 时，如果不进行失效处理，菜单就不会被重绘。两种方法都可以重新进入菜单系统，不过对于在本地模块中处理新的、有趣的菜单而言，后一种方法更符合预期。

---

**STEVEN KREUZER** 是一名 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗住在纽约皇后区。
