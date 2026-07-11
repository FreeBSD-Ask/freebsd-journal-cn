# Vagrant

作者：STEVE WILLS

Vagrant（<https://www.vagrantup.com/>）是一个命令行工具，用于构建和管理可移植的开发环境，例如虚拟机（VM）。它能轻松创建、启动、部署和销毁 VM。它常被用作在笔记本等设备上搭建开发和测试环境的工具。

Vagrant 让你能以自动化且可复现的方式搭建这些环境，最终提升生产力，并让你能与他人分享这些自动化环境。Vagrant 用 Ruby（<https://www.ruby-lang.org/>）编写，是一个跨平台工具，可在 Mac OS、Windows、Linux 和 FreeBSD 上轻松运行。通过支持多种 hypervisor——包括 VirtualBox、VMWare、Bhyve（需做一些工作）——以及云托管——如 Amazon AWS、谷歌 Cloud Compute、Azure 等——Vagrant 提供了很大的灵活性。在部署或初始 VM 设置方面，它支持多种工具，如 Ansible、Chef、Puppet、Salt，甚至 shell 脚本。Hypervisor 和 provisioner 的支持由插件提供，因此最终用户可以轻松为特定的 provider 或 provisioner 开发插件。已有许多插件可用（<https://github.com/mitchellh/vagrant/wiki/Available-Vagrant-Plugins>）。

## 为什么用 Vagrant？

你可以创建 Vagrant 环境并轻松与同事分享，无论他们的笔记本或台式机上用的是什么操作系统。你还可以用一条 `vagrant up` 命令启动单机或多机部署。通过让开发环境可复现，你可以避免常见的“在我机器上能跑”这类问题。Vagrant 通常用于在笔记本或台式工作站上搭建开发和测试环境，但也可用于其他场景。例如，一个有趣的用例是为 Jenkins 等自动化构建环境或工具创建构建节点或临时主机。

Vagrant 一般由软件开发者或测试者使用，对运维人员也有用，比如用于模拟生产环境部署。Vagrant 是一款非常有用的工具，用于可复现的、临时的开发和测试环境。但它并不适合管理承载关键数据的生产主机。Vagrant 简洁的设计使它太容易毁掉你用它创建的一切，不适合那类用途。

## 如何获取 Vagrant

在 FreeBSD 上，可以使用 `pkg install vagrant` 命令安装 Vagrant。这会安装 Vagrant 及其所有依赖。你还需要安装一个 hypervisor。出于本文目的，我将演示 VirtualBox 和 Bhyve 两种 hypervisor。在 FreeBSD 上安装 VirtualBox，使用 `pkg install virtualbox-ose`。请务必遵循列出的关于系统设置的说明。

## Vagrant 项目的组成

Vagrant 环境有两个主要组件。第一个是所谓的 Vagrant “box”。这是包含一个已安装客户机操作系统的磁盘镜像以及该磁盘镜像和操作系统的元数据的封装。用户可以自己——手动——创建并分享它们，或使用其他用户或团队创建好的 box。预构建的 box 可在 <https://atlas.hashicorp.com/boxes/search> 找到。`vagrant` 命令会默认在此位置搜索本地尚不存在的 box。可以用 `vagrant box add` 命令手动将 box 添加到本地 Vagrant 缓存，例如：

```sh
vagrant box add hashicorp/precise64
```

可以用 `vagrant box list` 列出缓存的 box。

第二个主要组件是 Vagrantfile。这个文件列出你的 Vagrant 项目中有哪些 VM 以及如何设置这些 VM。Vagrantfile 用 Ruby 编写，但编辑它不需要深入的 Ruby 知识，它被设计得很简单。关于 Vagrantfile 的更多信息见 <https://www.vagrantup.com/docs/vagrantfile/>。Vagrantfile 还用于向 Vagrant 标记你项目的根目录在哪里。这有助于 Vagrant 找到 Vagrantfile 中可能用相对路径引用的其他文件。

## 设置项目

设置一个项目最简单的方式是运行以下命令：

```sh
vagrant init hashicorp/precise64
```

这会在当前目录创建 Vagrantfile。接下来运行：

```sh
vagrant up
```

这条命令会把 box 下载到本地目录（默认 **~/.vagrant.d/boxes/**）并启动 VM。之后你可以运行：

```sh
vagrant ssh
```

通过 ssh 进入 VM 并开始使用它。完成后，可以用以下命令停止 VM：

```sh
vagrant halt
```

如果想彻底删除 VM，运行：

```sh
vagrant destroy
```

这会妥善移除该 VM 实例及其本地修改，但把缓存的 box 副本保留在 **~/.vagrant/boxes/** 中。在我们的示例 VM 中，注意我们没有指定任何部署或配置步骤。我们可以在 Vagrantfile 中添加步骤来配置 VM。例如，取消 Vagrantfile 中以下行的注释：

```sh
config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y apache2
SHELL
```

下次运行 `vagrant up` 或 `vagrant provision` 时，Vagrant 会运行 `apt-get` 命令安装 Apache Web 服务器。关于 provision 的更多信息见 <https://www.vagrantup.com/docs/getting-started/provisioning.html>。Vagrant 的一个有趣功能是同步文件夹功能。它会在宿主操作系统和客户操作系统之间同步文件，例如让你更容易地编辑要在 VM 中构建或运行的代码。VM 有一个默认网络连接，允许通过 ssh 连接到 VM。你可以按需在任何额外端口上向 VM 添加额外的端口转发。

## 创建新的 Vagrant 项目

作为示例，我们将创建一个运行 Web 服务器的新 Vagrant 项目。创建新 Vagrant 项目的第一步是创建一个新目录并在其中运行 `vagrant init`。例如：

```sh
mkdir journal
cd journal
vagrant init freebsd/FreeBSD-11.0-RELEASE-p1
```

这会在我们的新目录中创建一个 Vagrantfile。注意，在使用前我们需要对这个默认文件做若干修改。由于 Vagrant 的默认值，你需要在新的 Vagrantfile 中添加以下行：

```sh
config.vm.guest = :freebsd
```

此外，由于 FreeBSD box 中的一个 bug（已有修复即将提交），你需要在新 Vagrantfile 中添加以下行：

```sh
config.vm.base_mac = "080027FC8C33"
```

为避免首次启动时出现启动超时——因为系统在首次启动时会执行首次启动更新——我们需要在 Vagrantfile 中添加以下行：

```sh
config.vm.boot_timeout = 3600
```

在尝试初始化 box 之前，还有一处要改，即需要禁用默认共享文件夹：

```sh
config.vm.synced_folder ".", "/vagrant", id: "vagrant-root", disabled: true
```

我们还想通过添加以下行为 Web 服务器暴露一个端口：

```sh
config.vm.network :forwarded_port, guest: 80, host: 8080
```

由于 FreeBSD 默认没有安装 `bash`，我们需要配置 Vagrant 使用 `sh` 作为其 ssh shell：

```sh
config.ssh.shell = "sh"
```

默认的内存是 512MB，我们应把它增加到 1GB，并为 VM 分配两个 CPU：

```sh
config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
    vb.customize ["modifyvm", :id, "--cpus", "2"]
end

config.vm.provider :bhyve do |vm|
    vm.memory = "1024M"
    vm.cpus = "2"
end
```

注意我们为每个 hypervisor provider 设了单独的块。我们再为 VM 添加一块额外磁盘用于 ZFS。这同样是 hypervisor 特定的。在 virtualbox 部分添加：

```sh
file_to_disk = File.realpath( "." ).to_s + "/extradisk.vdi"

if ARGV[0] == "up" && ! File.exist?(file_to_disk)
    vb.customize [ 'createhd',
        '--filename', file_to_disk,
        '--format', 'VDI',
        '--size', 3 * 1024 # 3 GB
    ]
end
vb.customize [ 'storageattach', :id,
    '--storagectl', 'IDE Controller', # 名称可能不同
    '--port', 1,
    '--device', 0,
    '--type', 'hdd',
    '--medium', file_to_disk
]
```

在 bhyve 部分添加：

```sh
vm.disks = [{name: "extradisk", size: "3G", format:"raw",}]
```

下一步是安装 Web 服务器进程并确保它正在运行：

```sh
config.vm.provision "shell", inline: <<-SHELL
    pkg install -y apache24
    sysrc apache24_enable=yes
    service apache24 start
SHELL
```

去掉注释后，我们的 Vagrantfile 应如下所示：

```sh
Vagrant.configure("2") do |config|
    config.vm.box = "freebsd/FreeBSD-11.0-RELEASE-p1"
    config.vm.guest = :freebsd
    config.vm.base_mac = "080027FC8C33"
    config.vm.boot_timeout = 3600
    config.vm.network :forwarded_port, guest: 80, host: 8080
    config.vm.synced_folder ".", "/vagrant", id: "vagrant-root", disabled: true
    config.ssh.shell = "sh"

    config.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "1024"]
        vb.customize ["modifyvm", :id, "--cpus", "2"]
        file_to_disk = File.realpath( "." ).to_s + "/extradisk.vdi"

        if ARGV[0] == "up" && ! File.exist?(file_to_disk)
            vb.customize [ 'createhd',
                '--filename', file_to_disk,
                '--format', 'VDI',
                '--size', 3 * 1024 # 3 GB
            ]
        end
        vb.customize [ 'storageattach', :id,
            '--storagectl', 'IDE Controller', # 名称可能不同
            '--port', 1,
            '--device', 0,
            '--type', 'hdd',
            '--medium', file_to_disk
        ]
    end

    config.vm.provider :bhyve do |vm|
        vm.memory = "1024M"
        vm.cpus = "2"
        vm.disks = [{name: "extradisk", size: "3G", format:"raw",}]
    end

    config.vm.provision "shell", inline: <<-SHELL
        pkg install -y apache24
        sysrc apache24_enable=yes
        service apache24 start
    SHELL
end
```

现在我们可以用 `vagrant up` 启动 VM。Vagrant 创建并启动 VM 时，会把它创建为不可见的无头 VM。你可能想打开 VirtualBox 管理器，以便在 VM 列表中看到它。你还可以点击该 VM，再点击“Show”按钮查看 VM 控制台。shell provision 脚本运行时，我们可以查看命令的输出。完成后，我们应该能访问默认网站：

<http://localhost:8080/>

要访问该 box，可以用前述 `vagrant ssh` 命令通过 ssh 进入。

## 现有项目

Vagrant 最强大的能力之一来自用自动化创建 box 和 Vagrantfile 并与他人分享。我们来看一些现有项目。注意这些示例是使用 VirtualBox hypervisor 测试的。

### Minio

Minio（<https://min.io/>）是一个“为云应用和 devops 构建的分布式对象存储服务器”。它与 Amazon S3 兼容，相当有用。本例中，我创建了一个使用 Ansible 在 FreeBSD 上配置 Minio 的 Vagrantfile。首先运行 `pkg install ansible` 安装 Ansible；然后克隆或下载此项目（<https://github.com/swills/minio.vagrant>）。然后运行 `vagrant up` 命令，只需几条简单命令就能得到一个可用的 Minio 服务器。

```sh
mkdir minio
cd minio
git clone https://github.com/swills/minio.vagrant.git .
vagrant up
```

`vagrant up` 步骤会花几分钟，因为 FreeBSD VM 会在首次启动时执行自动的 freebsd-update。完成后，通过 `vagrant ssh` 进入 VM 并获取 Minio 的授权密钥。然后在 VM 中执行以下命令：

```sh
sudo grep accessKey /usr/local/etc/minio/.minio/config.json
"accessKey": "KO2E80M9H87CDIJKP6A1",
sudo grep secretKey /usr/local/etc/minio/.minio/config.json
"secretKey": "2mMkACOoq7v8oUrZqZs3+S9gRwqNLws5MZTNjgnE"
```

然后可以登录 Minio：

<http://localhost:9000/minio/login>

你需要使用在 VM 中找到的 `accessKey` 和 `secretKey`。（这些在服务首次启动时生成，会有所不同。）

### Gluster

GlusterFS（<http://www.gluster.org/documentation/About_Gluster/>）是一个自由软件并行分布式文件系统，可扩展到数 PB。与前面的示例类似，我创建了一个使用 Ansible 在一对 FreeBSD VM 上配置 Gluster 的 Vagrantfile。你可以克隆此项目（<https://github.com/swills/okapi>）并运行 `vagrant up` 开始。

```sh
mkdir gluster
cd gluster
git clone https://github.com/swills/okapi.git .
vagrant up
```

接下来运行 `vagrant ssh server1` 命令。进入该 VM 后，运行以下命令：

```sh
sudo gluster peer probe server2
sudo gluster volume create gv0 replica 2 server1:/gluster/gv0 server2:/gluster/gv0
sudo gluster volume start gv0
sudo gluster volume info
sudo mount_glusterfs server1:/gv0 /mnt
```

这就设置好了 Gluster。然后你可以 `vagrant ssh server2`，在该 VM 中运行：

```sh
sudo mount_glusterfs server1:/gv0 /mnt
```

至此你应该有一个可用的 Gluster 集群。在任一台 **/mnt** 下创建或修改的文件都应在另一台上可见。（gluster 命令是手动运行的，以便在两台机器都启动之后执行。）

### Poudriere

最后，对任何对 FreeBSD Ports 开发感兴趣的人而言可能特别有用：有一套 Vagrantfile 和 Ansible 脚本，可以搭建一个完整的 poudriere 开发环境。它包括一个完整的 Ports 树用于 Ports 开发，以及多个 jail 用于测试构建。注意，由于 jail 构建过程和 Ports 树检出，这一个的 provision 时间会明显更长，也需要更多磁盘空间。

```sh
mkdir poudriere_vagrant
cd poudriere_vagrant
git clone https://github.com/swills/gerenuk.git .
vagrant up
```

成功完成后，你可以 `vagrant ssh` 进入 VM，处理 Ports 并运行 poudriere 测试构建。你可以编辑 **/usr/local/poudriere/ports/default** 中的文件，并用标准 poudriere 命令构建，例如：

```sh
sudo poudriere bulk -j 110-amd64 shells/zsh
```

provision 脚本还设置了 NGINX，在以下地址提供 poudriere 状态页：

<http://localhost:10080/>

## 在 Bhyve 上使用 Vagrant

Vagrant 可以通过一个名为 vagrant-bhyve（<https://github.com/jesa7955/vagrant-bhyve>）的项目与 Bhyve 配合使用，该项目在 2016 年的谷歌编程之夏期间创建。要使用它，你可能还需要另一个项目——vagrant-mutate（<https://github.com/sciurus/vagrant-mutate>），帮助把 VM 转换为 vagrant-bhyve 能理解的格式。在 FreeBSD 上，两者都可通过运行以下命令安装：

```sh
pkg install rubygem-vagrant-bhyve rubygem-vagrant-mutate
```

如上安装插件后，我们需要通过运行以下命令确保 `vagrant` 使用它：

```sh
vagrant plugin install vagrant-mutate
vagrant plugin install vagrant-bhyve
```

如前所述，我们可以用 `vagrant box list` 命令列出当前本地缓存的 box。输出可能如下：

```sh
freebsd/FreeBSD-11.0-RELEASE-p1 (virtualbox, 2016.09.29)
hashicorp/precise64             (virtualbox, 1.1.0)
```

我们可以用 `vagrant mutate` 把这些 box 转换为 Bhyve 可用的格式。要用以下命令转换 box：

```sh
vagrant mutate ubuntu/xenial64 bhyve
vagrant mutate freebsd/FreeBSD-11.0-RELEASE-p1 bhyve
```

现在我们可以用之前生成的 journal Vagrantfile 创建 VM：

```sh
vagrant up --provider bhyve
```

一如既往，这会在首次启动时执行更新，因此可能需要一些时间。由于更新后自动重启，VM 会关机，Vagrant 会提示 VM 启动失败，但如果重新运行 `vagrant up --provider bhyve` 命令，应该就能正常工作。之后，你可以通过运行 `vagrant provision` 来运行 provision 步骤。然后，就像使用 VirtualBox 时一样，通过 `vagrant ssh` 进入。vagrant-bhyve 插件目前不支持设置端口转发，因此在这种情况下需要手动完成。你可以通过运行 `cu -l /dev/nmdm0B` 命令查看 VM 的控制台。

## 结语

希望你享受了这趟 Vagrant 简要之旅，在亲自尝试时也能发现它和我一样有用！

---

**STEVE WILLS** 是一位丈夫和父亲，住在美国北卡罗来纳州。他是 FreeBSD Ports 提交者，专注于 Ruby 和其他编程语言。
