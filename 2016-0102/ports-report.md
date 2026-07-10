# Ports 报告

- 作者：**Frederic Culot**

2015 年 11–12 月期间，问题修复方面的工作异常活跃。在此期间，我们欣然欢迎 miwi@ 回归，并将 ian@ 的提交权限扩展到 Ports 仓库。同时，一些重要的 Port 也更新了，升级时可能需要谨慎对待，详见下文。

## 重要的 Port 更新

antoine@ 像往常一样执行了大量 exp-run（共 21 次），以检查主要 Port 的更新是否安全。此次重要更新中，比较突出的包括：

- CMake 更新至 3.4.1
- ruby-gens 更新至 2.5.0
- setuptools 更新至 19.2
- freetype2 更新至 2.6.2

其他重要变更还包括 ncurses 更新至 6.0——如果你不使用二进制包，需要重新构建所有依赖它的 Port。此外，tor 启动脚本变更后，需手动配置日志。一如既往，更多信息请参见 **/usr/ports/UPDATING** 文件，升级主要 Port 前应仔细阅读该文件。

## 新的 Ports 提交者与权限保管

11 月，我们欣然恢复了 miwi@ 的提交权限，因为他又有时间参与 Ports 工作；12 月，我们也将 ian@ 的提交权限扩展到 Ports 树，方便他更新 u-boot Port。

部分提交权限因超过 18 个月无活动而被收回保管（jhay@、max@、sumikawa@、alexey@、sperber@）。遗憾的是，nox@ 的提交权限被撤销，因为他已去世。Juergen Lock 对 FreeBSD 的贡献，可在我们的纪念页面查阅简要回顾。（<https://www.freebsd.org/doc/en/articles/contributors/contribdevelinmemoriam.html>）

## 统计数据

11–12 月期间的几项数据：约 110 名提交者向 Ports 树提交了 4,187 次变更，比上一周期略少。但问题修复方面的活动显著增加，共关闭 1,217 份问题报告，相比 2015 年 9–10 月周期增长超过 10%。感谢所有报告问题的用户，也感谢所有提交修复的提交者！

## 小技巧

以下是一些与 Port 更新相关的小技巧。首先，你需要识别出过时的软件包。可以使用如下命令：

```sh
pkg version -vl'<'
```

然后，需要查明先前识别出的软件包是否有特定的处理流程。使用如下命令，可以列出需要特殊处理的受影响软件包：

```sh
grep -B1 AFFECTS /usr/ports/UPDATING
```

最后，可以使用 `pkg upgrade` 更新二进制包，而任何使用了非默认选项的 Port 都需要重新编译。

要随时获知 Port 是否需要更新，可以将以下命令加入 crontab，让 portsnap（参见 <https://www.freebsd.org/doc/handbook/ports-using.html>）每日更新 Ports 树，并发送一封包含所有过时软件包的邮件。

```sh
0 0 1 * * root portsnap cron update && pkg version -vl'<'
```

---

**Frederic Culot** 取得物理学计算机建模方向的博士学位，过去 10 年在 IT 行业工作。业余时间他学习商业与管理，刚完成 MBA。Frederic 于 2010 年以 Port 提交者身份加入 FreeBSD，至今累计约 2,000 次提交，指导过 6 名新提交者，现在担任 portmgr-secretary。
