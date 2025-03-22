# 使用 ZFS 原生加密保护数据

- 原文链接：[Protecting Data with ZFS Native Encryption](https://freebsdfoundation.org/our-work/journal/browser-based-edition/storage-and-filesystems/protecting-data-with-zfs-native-encryption/)
- 作者：Roller Angel

ZFS 原生支持加密数据集，能让你轻松地使用行业标准的密码套件来保护数据。与磁盘的全盘加密相比，将数据集加密的主要优势在于，当未使用数据集时，可以将其卸载，而全盘加密要求在静止状态下加密时，磁盘必须关闭。请记住，ZFS 原生加密有加载和卸载密钥的概念。仅仅卸载加密的数据集是不够的，你还必须卸载与该数据集关联的密钥。如果密钥仍然处于加载状态，数据集可以被挂载并且数据将可用。卸载密钥会使挂载操作失败。加载密钥是挂载数据集的前提。嵌套的子数据集会继承其父数据集的加密密钥，但这并非必须。即使父数据集使用不同的加密设置，也可以使用不同的加密密钥和密码套件。最后，更改密钥就像在数据集上执行 `zfs change-key` 命令一样简单。

这些是开始使用的基础概念。

为新创建的数据集启用加密参数，再设置密钥格式就足够了。如果未指定加密密码套件，则默认使用 aes-256-gcm。默认值可能会随着未来新增密码套件而变化。现有数据集的加密属性是只读的，无法修改未加密的数据集的属性来启用加密。要指定加密属性，你需要了解有哪些参数可用。我建议阅读 `zfsprops` 手册页，你可以输入命令 `man zfsprops` 来查看。我还建议阅读 `zfs-load-key` 的手册页。对于我们的第一个加密数据集，我们将使用默认的密码套件、口令密钥格式，来创建一个名为 `secrets` 的数据集。我使用的是我实验室中创建的 FreeBSD  jail 机器，名为 alice。实验室中的所有 jail 都位于名为 `lab` 的 zpool 上。我已将叫 `zroot` 的 zpool 分配给 jail。在 jail 中，我必须使用完整路径 `lab/alice/zroot` 作为 zpool 名称，以便在其中创建数据集。作为对比，我的笔记本电脑上，我可以直接使用我的 zpool 名称并在那里创建数据集。以下是创建加密数据集的命令，适用于 alice jail 和我的笔记本电脑。同其他 ZFS 数据集一样，设置挂载点是一个好主意，但请记住 ZFS 是个分层文件系统，因此不要使用现有路径作为新数据集的挂载点。

alice jail:

```sh
zfs create -o encryption=on -o keyformat=passphrase -o mountpoint=/secrets
lab/alice/zroot/secrets
```

我的笔记本：

```sh
zfs create -o encryption=on -o keyformat=passphrase -o mountpoint=/secrets zroot/secrets
```

在运行 `zfs create` 命令后，提示我输入一个足够长的口令。现在，我有了一个已挂载的加密数据集，可以在其中存储需要保护的数据。当数据集处于挂载状态时，我可以像使用其他未加密的数据集一样使用它。当我完成机密数据的添加后，我可以通过输入命令 `zfs unmount -u lab/alice/zroot/secrets` 一次性卸载数据集和密钥。要解密、重新挂载数据，只需运行命令 `zfs mount -l lab/alice/zroot/secrets`。这会提示我输入口令，加载密钥，然后挂载数据集。如果在卸载命令中省略参数 `-u`，只会卸载数据集，密钥仍然会保持加载状态。数据集仍然可以通过 `zfs mount lab/alice/zroot/secrets` 挂载，而无需输入口令。要在数据集已卸载后卸载密钥，我运行 `zfs unload-key lab/alice/zroot/secrets`。现在，之前的挂载命令将失败，因为密钥没有加载，并且我未提供参数 `-l` 来让 ZFS 在挂载之前加载密钥。要加载密钥且允许挂载数据集，我会运行 `zfs load-key lab/alice/zroot/secrets`。系统会提示我输入口令，之前的挂载命令现在会成功，因为密钥已经加载。要检查密钥是否已加载，可以查看数据集的属性。当我运行命令 `zfs list -o name,mountpoint,encryption,keylocation,keyformat,keystatus,encryptionroot lab/alice/zroot/secrets` 时，会显示一些有用的属性。`KEYSTATUS` 列显示为 "available" 时，表示密钥已加载。要查看所有数据集属性，可以使用 `zfs get all lab/alice/zroot/secrets`。

接下来，我在 `secrets` 数据集下创建一个嵌套的数据集，并使用不同的密码套件和密钥格式。这次，我将使用密钥文件而非口令。要创建使用密钥文件的数据集，我首先需要生成密钥并将其存储在文件中。我通过输入命令 `dd if=/dev/urandom bs=32 count=1 of=/media/more-secrets.key` 来完成。由于密钥文件要求长度为 32 字节，因此我使用了 `bs=32`。输出路径选择 `/media`，因为我在此路径下挂载了一个便携式 USB 驱动器，还使用 `dd` 命令生成了密钥文件，再将其直接存储到驱动器上。这样，当我卸载密钥并卸载和移除 USB 驱动器时，密钥文件就不会留在我的机器上。我建议将密钥文件存储在多个 USB 驱动器上，以防某个 USB 驱动器损坏。现在，密钥文件已经生成，我可以通过运行命令 `zfs create -o encryption=aes-256-ccm -o keyformat=raw -o keylocation=file:///media/more-secrets.key -o mountpoint=/secrets/more-secrets lab/alice/zroot/secrets/more-secrets` 来创建使用 AES-256-CCM 密码套件的嵌套数据集。查看这个新数据集的属性时，我可以看到 `ENCROOT` 列设置为 `lab/alice/zroot/secrets/more-secrets`。我可以使用与 `secrets` 数据集相同的方法来卸载和卸载密钥。当我拥有更多数据集和密钥时，我可能想考虑使用 `zfs unload-key -a` 卸载所有密钥，或使用 `zfs unload-key -r lab/alice/zroot/secrets` 来仅卸载 `secrets` 数据集及其所有后代数据集的密钥。而且，如果我想加载 `secrets` 数据集及其所有后代数据集的密钥子集，可以运行 `zfs load-key -r lab/alice/zroot/secrets`。

如果我后来决定 `more-secrets` 数据集中的数据不需要单独的密钥文件，而是希望它继承父数据集 `secrets` 的设置（即从自定义生成的密钥文件切换到之前配置的口令），我只需要运行命令 `zfs change-key -i lab/alice/zroot/secrets/more-secrets`。再次查看属性，注意到 `ENCROOT`、`KEYLOCATION` 和 `KEYFORMAT` 都已经发生了变化。然而，密码套件不会改变，因为密码套件只能在数据集创建时设置。由于 `more-secrets` 是包含在 `secrets` 中的，它将作为卸载 `secrets` 数据集的一部分被卸载。虽然挂载 `secrets` 数据集不会自动挂载 `more-secrets`，但它需要单独挂载。不过，由于它们共享相同的密钥，所以密钥只需要加载一次。要切换回使用密钥文件，我运行命令 `zfs change-key -o keyformat=raw -o keylocation=file:///media/more-secrets.key lab/alice/zroot/secrets/more-secrets`。如果我想永久销毁 `more-secrets` 中的数据，我只需卸载数据集、卸载密钥，并销毁密钥文件及我所做的任何备份副本。现在，数据将无法恢复。然后，我可以运行命令 `zfs destroy lab/alice/zroot/secrets/more-secrets` 来移除该数据集。

最后，我想分享的一点是关于加密数据的备份。如你所见，ZFS 原生加密能轻松地使用加密来保护数据。加密数据集的快照可以以加密形式传输到不受信任的备份服务器上。没有密钥，远程备份服务器就无法挂载该数据集。可以使用 `zfs send` 命令的参数 `--raw` 来实现这一点。有关更多细节，我建议阅读 `zfs-send` 的手册页，以了解其工作原理，然后获取《ZFS Mastery: Advanced ZFS》一书，深入研究具体细节，并学习一系列技巧，以进一步提升你的 ZFS 技能。

希望你喜欢这篇操作指南，并且开始使用 ZFS 文件系统提供的原生加密来保护你的敏感数据。

---

**Roller Angel** 大部分时间都在帮助人们学习如何使用技术实现他们的目标。他是一位热衷于 FreeBSD 系统管理的 Python 爱好者，喜欢学习开源技术（尤其是 FreeBSD 和 Python）解决问题的神奇方法。他坚信人们可以学习所有他们愿意去做的事情。Roller 总是在寻找创造性的解决方案，享受解决问题的挑战。他有很强的学习动机，喜欢探索新想法，并保持技能的敏锐。他喜欢参与研究社区，并分享自己的想法。
