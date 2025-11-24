# Overlord：让部署 Jail 像编程一样快

- 原文：[Overlord: Deploy Jails as Fast as You Code](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/overlord-deploy-jails-as-fast-as-you-code/)
- 作者：Jesús Daniel Colmenares Oviedo

当我创建 [AppJail](https://github.com/DtxdF/AppJail) 时——这是一款完全用 **sh(1)** 和 C 语言编写的、基于 BSD-3 许可证的开源框架，用于利用 FreeBSD jail 创建隔离、便携且易于部署的类似应用程序的环境——我的初衷是用它来测试 Ports，以避免破坏我的主环境。如今，AppJail 不仅仅是一个用于测试 Ports 的脚本，它变得高度灵活，具备许多非常实用的自动化功能。

当 AppJail 达到稳定阶段并被用于多种系统后，我意识到为每个想要的服务都部署一个 jail 并不可行，尤其是在需要部署越来越多服务的情况下。于是，Director 应运而生。

[AppJail Director](https://github.com/DtxdF/director) 是一款基于 AppJail 的多 jail 环境管理工具，使用简单的 YAML 配置文件定义如何配置组成应用程序的一个或多个 jail。有了 Director 文件，你只需一条命令 **appjail-director up** 即可创建并启动你的应用程序。

Director 是 AppJail "一切皆代码" 理念的首次实践。它将 jail 组织成项目，让你以声明式方式创建含有一个或多个 jail 的项目；当你修改了该配置文件或相关文件（例如 Makejail 文件），Director 会检测到变更，并毫不犹豫地销毁并重新创建 jail。听起来有点激进，但用 *The Ephemeral Concept*（流变的概念）来解释最为恰当：

Director 将每个 jail 视为 "流变" 的。这并不意味着 jail 停止或系统重启后数据会丢失，而是意味着 Director 认为销毁 jail 是安全的，因为你已经明确区分了应持久保存的数据和被视为流变的数据。

更多细节见 **appjail-ephemeral(7)** 手册页，原则和上述相同。

值得注意的是，Director 本身不负责部署 jail，它依赖执行配置、安装包等操作的指令，因此大量利用了 AppJail 的 Makejails 功能——这是一种简易文本文件，自动化创建 jail 的步骤。Centralized Repository（集中仓库）中已有许多 Makejail 文件，但你也可以使用自己的仓库来托管。

AppJail 和 Director 极大简化了我的工作，但有一个问题二者都未能解决——即如何在多台服务器上协调管理 jail。对少量服务器，结合 SSH 使用 AppJail 和 Director 尚可，但随着服务器数量增多，这会变得非常痛苦。因此 Overlord 应运而生。

[Overlord](https://github.com/DtxdF/overlord) 是一个面向 GitOps 的快速分布式 FreeBSD jail 编排器。你只需定义一个文件说明集群上运行的服务，部署就能在几秒到几分钟内完成。

幸好 Overlord 诞生时，AppJail 和 Director 已经相当成熟，重用这两个经过充分测试的工具并与 Overlord 结合，是个明智的选择。继承 Director "一切皆代码" 的哲学，使得 Overlord 易于使用。另一个设计决策是 Overlord 采用全异步架构。大型服务的部署可能耗时较长，但即使部署很快，声明式发送指令并让 Overlord 处理工作也是更优的体验。本文后续会详细介绍上述内容的诸多细节。

## 架构

Overlord 架构被描述为一种树形链式架构。每个运行 API 服务器的 Overlord 实例都可以配置为管理其他链（chains）的分组。每个成员链还可以进一步配置为管理更多的链，层层递进。虽然这种层级结构几乎可以无限扩展，但不加控制地扩展会引入延迟，因此了解你计划如何组织服务器是非常重要的。

选择这种架构的原因在于它非常简单且具备良好的可扩展性，通过将多个链连接起来形成集群，即可在多台服务器间共享资源，从而方便地进行项目部署。

该架构同时抽象了项目的部署方式。想要部署项目的用户无需了解每个链的具体终端节点，只需知道第一个链（也称根链，root chain）即可。这是因为每条链都会被标记一个任意的字符串标签，用户只需在部署文件中指定根链的终端地址、访问令牌和标签即可。虽然标签本质上是主观的，但它们可以表达需求。例如，我们可以用字符串 vm-only 标记那些具备部署虚拟机能力的服务器，用 db-only 标记数据库服务器，实际上标签是完全任意的。


<img width="535" height="125" alt=" Y__T6KJ`DC9R @T5A89{9ON " src=" https://github.com/user-attachments/assets/ccbf707e-3e4c-4ed5-a076-daa22943b3e4 " />


假设只有 charlie 和 delta 拥有 **db-only** 标签。要将项目部署到带有指定标签的 API 服务器，客户端必须向 main 发起 HTTP 请求，指定链为 **alpha.charlie** 和 **alpha.charlie.delta**。这个过程是透明进行的，无需用户干预。


<img width="274" height="83" alt="FEG5 )NST){2T18L3(TJDNQ" src="https://github.com/user-attachments/assets/b6b09e0e-b7df-4506-96fc-3137f37b810f" />


如果某条链宕机，会发生什么情况？如果根链宕机，则无法进行任何操作，虽然可以指定多个根链（但本文档其余部分只使用一个）。然而，如果根链之后的某条链宕机，会出现一个有趣的处理机制。

当根链之后的某条链检测到错误时，它可以将该失败的链暂时加入黑名单。黑名单的作用是不显示失败的链，尽管并不禁止通过其他链尝试连接到被标记为失败的链，因为一旦成功连接到黑名单中的链，它会被自动重新启用。

每个被列入黑名单的链都会被分配一个保持黑名单状态的时间。过了一段时间后，该链会从黑名单中移除；但如果链依旧失败，它会被重新加入黑名单，且保持时间更长。这个时间延长机制有上限，避免链被永久禁用。

上述机制的好处是，HTTP 请求的完成时间可以缩短，因为不必去尝试连接那些已知失败的链。

## 项目部署

演示 Overlord 最好的方式是部署一个小项目。

需要注意的是，一个项目可以包含多个 jail，但在以下示例中只需一个 jail。

filebrowser.yml:

```yml
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
projectName: filebrowser
projectFile: |
  options:
    - virtualnet: ':<random> default'
    - nat:
  services:
    filebrowser:
      makejail: 'gh+AppJail-makejails/filebrowser'
      volumes:
        - db: filebrowser-db
        - log: filebrowser-log
        - www: filebrowser-www
      start-environment:
        - FB_NOAUTH: 1
      arguments:
        - filebrowser_tag: 14.2
      options:
        - expose: '8432:8080 ext_if:tailscale0 on_if:tailscale0'
  default_volume_type: '<volumefs>'
  volumes:
    db:
      device: /var/appjail-volumes/filebrowser/db
    log:
      device: /var/appjail-volumes/filebrowser/log
    www:
      device: /var/appjail-volumes/filebrowser/www
```

你可能已经注意到，我并未直接指定访问令牌和入口点，而是通过环境变量来传递，这些变量是通过 **.env** 文件加载的：

.env:

```ini
ENTRYPOINT=http://127.0.0.1:8888
TOKEN=<access token>
```

那么另一个问题是：访问令牌是如何生成的？这很简单，令牌是由运行你想要访问的 Overlord 实例的机器生成的，但只有拥有访问密钥权限的人才能生成令牌。这个密钥默认是伪随机生成的。

```sh
# OVERLORD_CONFIG=/usr/local/etc/overlord.yml overlord gen-token
```

下一步就是简单地应用部署文件。

```sh
$ overlord apply -f filebrowser.yml
```

如果没有任何输出，说明一切正常，但这并不意味着项目已经部署完成。当应用一个部署文件且其中包含部署项目的规格（如上例中的 directorProject）时，项目会进入队列等待执行。由于当前没有其他项目正在运行，我们的项目会尽快被部署。

```yml
$ overlord get-info -f filebrowser.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    filebrowser:
      state: DONE
      last_log: 2025-04-22_17h57m45s
      locked: False
      services:
        - {'name': 'filebrowser', 'status': 0, 'jail': 'e969b06736'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 7 minutes and 42.41 seconds
        job_id: 14
        restarted: False
        labels:
         error: False
         message: None
```

## 元数据

元数据用于创建小型文件（例如配置文件），这些文件可以在部署项目或虚拟机时使用。虽然像 GitLab、GitHub、Gitea 等 Git 托管服务与 Makejails 配合使用非常方便，但你也可以使用元数据来代替依赖 Git 托管，以进一步配置你正在部署的服务或虚拟机。

元数据的另一个优点是可以在不同部署之间共享。例如，通过部署共享相同 **sshd\_config(5)** 和 **authorized\_keys** 文件的虚拟机，实现配置复用。

tor.yml:

```sh
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
    deployIn:
      labels:
        - desktop
    projectName: tor
    projectFile: |
      options:
        - virtualnet: ':<random> address:10.0.0.50 default'
        - nat:
    services:
      tor:
        makejail: !ENV '${OVERLORD_METADATA}/tor.makejail'
        volumes:
          - data: '/var/db/tor'

volumes:
  data:
    device: '/var/appjail-volumes/tor/data'
```

metadata.yml:

```sh
kind: metadata
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
    deployIn:
      labels:
        - desktop
    metadata:
      tor.makejail: |
        OPTION start
        OPTION overwrite=force

        INCLUDE gh+DtxdF/efficient-makejail

        PKG tor

        CMD echo "SocksPort 0.0.0.0:9050" > /usr/local/etc/tor/torrc
        CMD echo "HTTPTunnelPort 0.0.0.0:9080" >> /usr/local/etc/tor/torrc

        SERVICE tor oneenable
        SERVICE tor start
```

.env:

```ini
ENTRYPOINT=http://127.0.0.1:8888
TOKEN=<access token>
```

从用户的角度来看，部署项目和部署元数据没有区别。然而，元数据不会进入队列，而是直接（异步地）写入磁盘。

```sh
$ overlord apply -f metadata.yml
$ overlord apply -f tor.yml
$ overlord get-info -f metadata.yml -t metadata

datacenter: http://127.0.0.1:8888
entrypoint: main
chain: None
labels:
- all
- desktop
- services
- vm-only
metadata:
  tor.makejail: |
    OPTION start
    OPTION overwrite=force

    INCLUDE gh+DtxdF/efficient-makejail

    PKG tor

    CMD echo "SocksPort 0.0.0.0:9050" > /usr/local/etc/tor/torrc
    CMD echo "HTTPTunnelPort 0.0.0.0:9080" >> /usr/local/etc/tor/torrc

    SERVICE tor oneenable
    SERVICE tor start

$ overlord get-info -f tor.yml -t projects --filter-per-project

datacenter: http://127.0.0.1:8888
entrypoint: main
chain: None
labels:
- all
- desktop
- services
- vm-only
projects:
  tor:
    state: UNFINISHED
    last_log: 2025-04-22_18h40m30s
    locked: True
    services:
      - {'name': 'tor', 'status': 0, 'jail': '7ce0dfdcef'}
    up:
      operation: RUNNING
      last_update: 38.01 seconds
      job_id: 16
```

## 部署 FreeBSD 虚拟机

Overlord 能够借助出色的 [vm-bhyve](https://github.com/churchers/vm-bhyve) 项目部署虚拟机。虚拟机可以隔离很多 jail 无法做到的部分，尽管这样会带来一定的开销，但根据你的使用场景，这种开销可能并不是问题。

这个部署过程如下：会创建一个 director 文件（由 Overlord 内部完成），该文件用于进一步创建一个 jail，代表一个必须安装了 [vm-bhyve](https://github.com/churchers/vm-bhyve) 的环境，且需要配置使用 FreeBSD 支持的防火墙，以及配置虚拟机使用的桥接网络。听起来很复杂，但有一个 [Makejail](https://github.com/DtxdF/vm-makejail) 专门完成这些工作，可以查看该项目了解细节。上述 Makejail 会创建一个安装了 [vm-bhyve-devel](https://freshports.org/sysutils/vm-bhyve-devel) 的环境，配置 pf(4) 防火墙，并创建一个带有分配的 IPv4（192.168.8.1/24）的桥接，因此我们必须给虚拟机分配一个该网段内的 IPv4 地址。pf(4) 并未配置进一步隔离连接，因此虚拟机内的应用程序可以 "逃逸" 访问其他服务，这是否理想取决于应用的具体需求。

vm.yml:

```
kind: vmJail
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
    deployIn:
      labels:
        - !ENV '${DST}'
    vmName: !ENV '${VM}'
    makejail: 'gh+DtxdF/vm-makejail'
    template:
      loader: 'bhyveload'
      cpu: !ENV '${CPU}'
      memory: !ENV '${MEM}'
      network0_type: 'virtio-net'
      network0_switch: 'public'
      wired_memory: 'YES'
    diskLayout:
      driver: 'nvme'
      size: !ENV '${DISK}'
      from:
        type: 'components'
        components:
          - base.txz
          - kernel.txz
    osArch: amd64
    osVersion: !ENV '${VERSION}-RELEASE'
    disk:
      scheme: 'gpt'
      partitions:
        - type: 'freebsd-boot'
          size: '512k'
          alignment: '1m'
        - type: 'freebsd-swap'
          size: !ENV '${SWAP}'
          alignment: '1m'
        - type: 'freebsd-ufs'
          alignment: '1m'
          format:
            flags: '-Uj'
          bootcode:
            bootcode: '/boot/pmbr'
            partcode: '/boot/gptboot'
            index: 1
    fstab:
      - device: '/dev/nda0p3'
        mountpoint: '/'
        type: 'ufs'
        options: 'rw,sync'
        dump: 1
        pass: 1
      - device: '/dev/nda0p2'
        mountpoint: 'none'
        type: 'swap'
        options: 'sw'
        dump: 0
        pass: 0
    script-environment:
      - HOSTNAME: !ENV '${HOSTNAME}'
    script: |
      set -xe
      set -o pipefail

      . "/metadata/environment"

      sysrc -f /mnt/etc/rc.conf ifconfig_vtnet0="inet 192.168.8.2/24"
      sysrc -f /mnt/etc/rc.conf defaultrouter="192.168.8.1"
      sysrc -f /mnt/etc/rc.conf fsck_y_enable="YES"
      sysrc -f /mnt/etc/rc.conf clear_tmp_enable="YES"
      sysrc -f /mnt/etc/rc.conf dumpdev="NO"
      sysrc -f /mnt/etc/rc.conf moused_nondefault_enable="NO"
      sysrc -f /mnt/etc/rc.conf hostname="${HOSTNAME}"

      if [ -f "/metadata/resolv.conf" ]; then
        cp -a /metadata/resolv.conf /mnt/etc/resolv.conf
      fi

      if [ -f "/metadata/loader.conf" ]; then
        cp /metadata/loader.conf /mnt/boot/loader.conf
      fi

      if [ -f "/metadata/zerotier_network" ]; then
        pkg -c /mnt install -y zerotier

        zerotier_network='head -1 -- "/metadata/zerotier_network"'

        cat << EOF > /mnt/etc/rc.local
      while :; do
        if ! /usr/local/bin/zerotier-cli join ${zerotier_network}; then
          sleep 1
          continue
        fi

        break
      done

      rm -f /etc/rc.local
      EOF

        sysrc -f /mnt/etc/rc.conf zerotier_enable="YES"
      elif [ -f "/metadata/ts_auth_key" ]; then
        pkg -c /mnt install -y tailscale

        ts_auth_key='head -1 -- "/metadata/ts_auth_key"'

        echo "/usr/local/bin/tailscale up --accept-dns=false --auth-key=\"${ts_auth_key}\" && rm -f /etc/rc.local" > /mnt/etc/rc.local

        sysrc -f /mnt/etc/rc.conf tailscaled_enable="YES"
      fi

      if [ -f "/metadata/timezone" ]; then
        timezone='head -1 -- "/metadata/timezone"'

        ln -fs "/usr/share/zoneinfo/${timezone}" /mnt/etc/localtime
      fi

      if [ -f "/metadata/sshd_config" ]; then
        sysrc -f /mnt/etc/rc.conf sshd_enable="YES"
        cp /metadata/sshd_config /mnt/etc/ssh/sshd_config
      fi

      if [ -f "/metadata/ssh_key" ]; then
        cp /metadata/ssh_key /mnt/etc/ssh/authorized_keys
      fi

      if [ -f "/metadata/sysctl.conf" ]; then
        cp /metadata/sysctl.conf /mnt/etc/sysctl.conf
      fi

      if [ -f "/metadata/pkg.conf" ]; then
        mkdir -p /mnt/usr/local/etc/pkg/repos
        cp /metadata/pkg.conf /mnt/usr/local/etc/pkg/repos/Latest.conf
      fi
metadata:
  - resolv.conf
  - loader.conf
  - timezone
  - sshd_config
  - ssh_key
  - sysctl.conf
  - pkg.conf
  - ts_auth_key
```

metadata.yml:

```sh
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
projectName: tor
projectFile: |
  options:
    - virtualnet: ':<random> address:10.0.0.50 default'
    - nat:
  services:
    tor:
      makejail: !ENV '${OVERLORD_METADATA}/tor.makejail'
      volumes:
        - data: '/var/db/tor'
  volumes:
    data:
      device: '/var/appjail-volumes/tor/data'
```

.profile-vmtest.env:

```ini
ENTRYPOINT=http://127.0.0.1:8888
TOKEN=<access token>
VM=vmtest
CPU=1
MEM=256M
DISK=10G
VERSION=14.2
SWAP=1G
HOSTNAME=vmtest
DST=provider
```

与其每次部署虚拟机都复制粘贴部署文件，不如创建多个环境（或类似配置文件）来管理。

```yml
$ overlord -e .profile-vmtest.env apply -f metadata.yml
$ overlord -e .profile-vmtest.env apply -f vm.yml
$ overlord -e .profile-vmtest.env get-info -f vm.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - provider
    - vm-only
  projects:
    vmtest:
      state: UNFINISHED
      last_log: 2025-04-22_20h19m34s
      locked: True
      services:
        - {'name': 'vm', 'status': 0, 'jail': 'vmtest'}
      up:
        operation: RUNNING
        last_update: 58.85 seconds
        job_id: 17
```

根据安装类型，安装过程可能需要一些时间。以上例中，我们选择从 FreeBSD 组件进行安装，因此如果服务器尚未拥有这些组件，或者已有组件但远程发生了变化（例如修改时间变动），Overlord 会自动下载最新组件。

```sh
$ overlord -e .profile-vmtest.env get-info -f vm.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - provider
    - vm-only
  projects:
    vmtest:
      state: DONE
      last_log: 2025-04-22_20h19m34s
      locked: False
      services:
        - {'name': 'vm', 'status': 0, 'jail': 'vmtest'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 6 minutes and 10.02 seconds
        job_id: 17
        restarted: False
$ overlord -e .profile-vmtest.env get-info -f vm.yml -t vm --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - provider
    - vm-only
  projects:
    vmtest:
      virtual-machines:
          operation: COMPLETED
          output: |
            md0 created
             md0p1 added
             md0p2 added
             md0p3 added
             /dev/md0p3: 9214.0MB (18870272 sectors) block size 32768, fragment size 4096
                using 15 cylinder groups of 625.22MB, 20007 blks, 80128 inodes.
                with soft updates
             super-block backups (for fsck_ffs -b #) at:
              192, 1280640, 2561088, 3841536, 5121984, 6402432, 7682880, 8963328, 10243776,
              11524224, 12804672, 14085120, 15365568, 16646016, 17926464
             Using inode 4 in cg 0 for 75497472 byte journal
             bootcode written to md0
             partcode written to md0p1
             ifconfig_vtnet0:  -> inet 192.168.8.2/24
             defaultrouter: NO -> 192.168.8.1
             fsck_y_enable: NO -> YES
             clear_tmp_enable: NO -> YES
             dumpdev: NO -> NO
             moused_nondefault_enable: YES -> NO
             hostname:  -> vmtest
             [vmtest.appjail] Installing pkg-2.1.0...
             [vmtest.appjail] Extracting pkg-2.1.0: .......... done
             Updating FreeBSD repository catalogue...
             [vmtest.appjail] Fetching meta.conf: . done
             [vmtest.appjail] Fetching data.pkg: .......... done
             Processing entries: .......... done
             FreeBSD repository update completed. 35950 packages processed.
             All repositories are up to date.
             The following 2 package(s) will be affected (of 0 checked):
            
             New packages to be INSTALLED:
                ca_root_nss: 3.108
                tailscale: 1.82.5
            
             Number of packages to be installed: 2
            
             The process will require 35 MiB more space.
             11 MiB to be downloaded.
             [vmtest.appjail] [1/2] Fetching tailscale-1.82.5.pkg: .......... done
             [vmtest.appjail] [2/2] Fetching ca_root_nss-3.108.pkg: .......... done
             Checking integrity... done (0 conflicting)
             [vmtest.appjail] [1/2] Installing ca_root_nss-3.108...
             [vmtest.appjail] [1/2] Extracting ca_root_nss-3.108: ....... done
             Scanning /usr/share/certs/untrusted for certificates...
             Scanning /usr/share/certs/trusted for certificates...
             Scanning /usr/local/share/certs for certificates...
             [vmtest.appjail] [2/2] Installing tailscale-1.82.5...
             [vmtest.appjail] [2/2] Extracting tailscale-1.82.5: ...... done
             =====
             Message from ca_root_nss-3.108:
            
             --
             FreeBSD does not, and can not warrant that the certification authorities
             whose certificates are included in this package have in any way been
             audited for trustworthiness or RFC 3647 compliance.
            
             Assessment and verification of trust is the complete responsibility of
             the system administrator.
            
             This package installs symlinks to support root certificate discovery
             for software that either uses other cryptographic libraries than
             OpenSSL, or use OpenSSL but do not follow recommended practice.
            
             If you prefer to do this manually, replace the following symlinks with
             either an empty file or your site-local certificate bundle.
            
               * /etc/ssl/cert.pem
               * /usr/local/etc/ssl/cert.pem
               * /usr/local/openssl/cert.pem
             tailscaled_enable:  -> YES
             sshd_enable: NO -> YES
             vm_list:  -> vmtest
             Starting vmtest
               * found guest in /vm/vmtest
               * booting...
            newfs: soft updates journaling set
             + set -o pipefail
             + . /metadata/environment
             + export 'HOSTNAME=vmtest'
             + sysrc -f /mnt/etc/rc.conf 'ifconfig_vtnet0=inet 192.168.8.2/24'
             + sysrc -f /mnt/etc/rc.conf 'defaultrouter=192.168.8.1'
             + sysrc -f /mnt/etc/rc.conf 'fsck_y_enable=YES'
             + sysrc -f /mnt/etc/rc.conf 'clear_tmp_enable=YES'
             + sysrc -f /mnt/etc/rc.conf 'dumpdev=NO'
             + sysrc -f /mnt/etc/rc.conf 'moused_nondefault_enable=NO'
             + sysrc -f /mnt/etc/rc.conf 'hostname=vmtest'
             + [ -f /metadata/resolv.conf ]
             + cp -a /metadata/resolv.conf /mnt/etc/resolv.conf
             + [ -f /metadata/loader.conf ]
             + cp /metadata/loader.conf /mnt/boot/loader.conf
             + [ -f /metadata/zerotier_network ]
             + [ -f /metadata/ts_auth_key ]
             + pkg -c /mnt install -y tailscale
             + head -1 -- /metadata/ts_auth_key
             + ts_auth_key=[REDACTED]
             + echo '/usr/local/bin/tailscale up --accept-dns=false --auth-key="[REDACTED]" && rm -f /etc/rc.local'
             + sysrc -f /mnt/etc/rc.conf 'tailscaled_enable=YES'
             + [ -f /metadata/timezone ]
             + head -1 -- /metadata/timezone
             + timezone=America/Caracas
             + ln -fs /usr/share/zoneinfo/America/Caracas /mnt/etc/localtime
             + [ -f /metadata/sshd_config ]
             + sysrc -f /mnt/etc/rc.conf 'sshd_enable=YES'
             + cp /metadata/sshd_config /mnt/etc/ssh/sshd_config
             + [ -f /metadata/ssh_key ]
             + cp /metadata/ssh_key /mnt/etc/ssh/authorized_keys
             + [ -f /metadata/sysctl.conf ]
             + cp /metadata/sysctl.conf /mnt/etc/sysctl.conf
             + [ -f /metadata/pkg.conf ]
             + mkdir -p /mnt/usr/local/etc/pkg/repos
             + cp /metadata/pkg.conf /mnt/usr/local/etc/pkg/repos/Latest.conf
          last_update: 5 minutes and 12.6 seconds
          job_id: 17
```

由于我使用了 Tailscale，上述虚拟机即将被配置加入我的 tailnet，过一段时间后，它应该会出现在节点列表中：

```sh
$ tailscale status
...
100.124.236.28  vmtest               REDACTED@    freebsd -
$ ssh root@100.124.236.28
The authenticity of host '100.124.236.28 (100.124.236.28)' can't be established.
ED25519 key fingerprint is SHA256:Oc61mU8erpgS2evkwL9WhOOl4Ze94sSNfhImLy3b4UQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '100.124.236.28' (ED25519) to the list of known hosts.
root@vmtest:~ #

```

## 服务发现

服务已经部署完成，然而根据你集群中服务器的数量，确定服务端点可能会变得复杂。你当然知道使用的端口和外部接口，IP 地址也相对容易获取，但更方便的做法是使用 DNS，这正是 DNS 的主要用途。服务虽然可以部署在不同服务器上，但它的访问端点始终保持一致。

SkyDNS 是一种较为老牌但功能强大的协议，结合 Etcd 使用时，可以实现便捷的服务发现。Overlord 可以配置使用 Etcd 和 SkyDNS。接下来，我们来部署 Etcd 集群。

etcd.yml:

```yml
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
    - r2
    - centralita
projectName: etcd-cluster
projectFile: |
  options:
    - alias:
    - ip4_inherit:
  services:
    etcd:
      makejail: 'gh+AppJail-makejails/etcd'
      arguments:
        - etcd_tag: '14.2-34'
      volumes:
        - data: etcd-data
      start-environment:
        - ETCD_NAME: !ENV '${NAME}'
        - ETCD_ADVERTISE_CLIENT_URLS: !ENV 'http://${HOSTIP}:2379'
        - ETCD_LISTEN_CLIENT_URLS: !ENV 'http://${HOSTIP}:2379'
        - ETCD_LISTEN_PEER_URLS: !ENV 'http://${HOSTIP}:2380'
        - ETCD_INITIAL_ADVERTISE_PEER_URLS: !ENV 'http://${HOSTIP}:2380'
        - ETCD_INITIAL_CLUSTER_TOKEN: 'etcd-demo-cluster'
        - ETCD_INITIAL_CLUSTER: !ENV '${CLUSTER}'
        - ETCD_INITIAL_CLUSTER_STATE: 'new'
        - ETCD_HEARTBEAT_INTERVAL: '5000'
        - ETCD_ELECTION_TIMEOUT: '50000'
        - ETCD_LOG_LEVEL: 'error'
  default_volume_type: '<volumefs>'
  volumes:
    data:
      device: /var/appjail-volumes/etcd-cluster/data
environment:
  CLUSTER: 'etcd0=http://100.65.139.52:2380,etcd1=http://100.109.0.125:2380,etcd2=http://100.96.18.2:2380'
labelsEnvironment:
  desktop:
    NAME: 'etcd0'
    HOSTIP: '100.65.139.52'
  r2:
    NAME: 'etcd1'
    HOSTIP: '100.109.0.125'
  centralita:
    NAME: 'etcd2'
    HOSTIP: '100.96.18.2'
```

收获：

```yml
$ overlord apply -f etcd.yml
$ overlord get-info -f etcd.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    etcd-cluster:
      state: DONE
      last_log: 2025-04-23_02h28m36s
      locked: False
      services:
        - {'name': 'etcd', 'status': 0, 'jail': 'f094a31c46'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 8 minutes and 11.51 seconds
        job_id: 20
        restarted: False
        labels:
         error: False
         message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    etcd-cluster:
      state: DONE
      last_log: 2025-04-23_02h28m37s
      locked: False
      services:
        - {'name': 'etcd', 'status': 0, 'jail': '1ff836df47'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 5 minutes and 37.82 seconds
        job_id: 2
        restarted: False
        labels:
         error: False
         message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: r2
  labels:
    - all
    - r2
    - services
  projects:
    etcd-cluster:
      state: DONE
      last_log: 2025-04-23_02h28m38s
      locked: False
      services:
        - {'name': 'etcd', 'status': 0, 'jail': '756ae9d5ca'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 5 minutes and 5.04 seconds
        job_id: 1
        restarted: False
        labels:
         error: False
         message: None
```

部署完成后，就可以使用以下参数配置每个 Overlord 实例。

/usr/local/etc/overlord.yml:

```yml
etcd:
  100.65.139.52: {}
  100.109.0.125: {}
  100.96.18.2: {}
```

请记得重启 Overlord 进程以使更改生效。

```sh
supervisorctl restart overlord:
```

下一个需要部署的服务是 CoreDNS。多亏了它，我们可以通过 [Etcd 插件](https://coredns.io/plugins/etcd/) 使用 SkyDNS。

coredns.yml:

```sh
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
    - r2
projectName: dns-server
projectFile: |
  options:
    - alias:
    - ip4_inherit:
  services:
    coredns:
      makejail: !ENV '${OVERLORD_METADATA}/coredns.makejail'
```

metadata.yml:

```yml
kind: metadata
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
    - r2
metadata:
  Corefile: |
    .:53 {
      bind tailscale0
      log
      errors
      forward . 208.67.222.222 208.67.220.220
      etcd overlord.lan. {
        endpoint http://100.65.139.52:2379 http://100.109.0.125:2379 http://100.96.18.2:2379
      }
      hosts /etc/hosts namespace.lan.
      cache 30
    }
  coredns.hosts: |
    100.65.139.52    controller.namespace.lan
    100.96.18.2      centralita.namespace.lan
    100.127.18.7     fbsd4dev.namespace.lan
    100.123.177.93   provider.namespace.lan
    100.109.0.125   r2.namespace.lan
    172.16.0.3      cicd.namespace.lan
  coredns.makejail: |
    OPTION start
    OPTION overwrite=force
    OPTION healthcheck="health_cmd:jail:service coredns status" "recover_cmd:jail:service coredns restart"

    INCLUDE gh+DtxdF/efficient-makejail

    CMD mkdir -p /usr/local/etc/pkg/repos
    COPY ${OVERLORD_METADATA}/coredns.pkg.conf /usr/local/etc/pkg/repos/Latest.conf

    PKG coredns

    CMD mkdir -p /usr/local/etc/coredns
    COPY ${OVERLORD_METADATA}/Corefile /usr/local/etc/coredns/Corefile

    COPY ${OVERLORD_METADATA}/coredns.hosts /etc/hosts

    SYSRC coredns_enable=YES
    SERVICE coredns start
  coredns.pkg.conf: |
    FreeBSD: {
      url: "pkg+https://pkg.FreeBSD.org/${ABI}/latest",
      mirror_type: "srv",
      signature_type: "fingerprints",
      fingerprints: "/usr/share/keys/pkg",
      enabled: yes
    }
```

正如你在 CoreDNS 配置文件中看到的，假设使用了 **overlord.lan** 域名区域，但默认情况下 Overlord 仅使用根域名 **.**，这对于当前场景没有意义，因此请根据这一点配置 Overlord 并部署 CoreDNS。

/usr/local/etc/overlord.yml：

```yml
skydns:
  zone: 'overlord.lan.'
```

注意：请记得重启 Overlord 进程，使配置更改生效。

```yml
$ overlord apply -f metadata.yml
$ overlord apply -f coredns.yml
$ overlord get-info -f coredns.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    dns-server:
      state: DONE
      last_log: 2025-04-23_13h32m49s
      locked: False
      services:
        - {'name': 'coredns', 'status': 0, 'jail': '8106aaca6d'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 2 minutes and 30.14 seconds
        job_id: 25
        restarted: False
        labels:
         error: False
         message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: r2
  labels:
    - all
    - r2
    - services
  projects:
    dns-server:
      state: DONE
      last_log: 2025-04-23_13h32m54s
      locked: False
      services:
        - {'name': 'coredns', 'status': 0, 'jail': '9516eb48aa'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 3 minutes and 26.9 seconds
        job_id: 4
        restarted: False
        labels:
         error: False
         message: None
```

我们的 Etcd 集群和 DNS 服务器均已启动运行。客户端应配置为通过这些 DNS 服务器解析主机名，因此请在它们的 **resolv.conf(5)** 或类似文件中进行配置。

示例 **/etc/resolv.conf** 配置如下：

```ini
nameserver 100.65.139.52
nameserver 100.109.0.125
```

我们的“弗兰肯斯坦”（**译者注：弗兰肯斯坦是个制造怪物的科学家，参见小说《科学怪人》**）活过来了！下一步是部署一个服务，并测试所有部分是否如预期般正常工作。

homebox.yml:

```yml
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - centralita
projectName: homebox
projectFile: |
  options:
    - virtualnet: ':<random> default'
    - nat:
  services:
    homebox:
      makejail: gh+AppJail-makejails/homebox
      options:
        - expose: '8666:7745 ext_if:tailscale0 on_if:tailscale0'
        - label: 'overlord.skydns:1'
        - label: 'overlord.skydns.group:homebox'
        - label: 'overlord.skydns.interface:tailscale0'
      volumes:
        - data: homebox-data
      arguments:
        - homebox_tag: 14.2
  default_volume_type: '<volumefs>'
  volumes:
    data:
      device: /var/appjail-volumes/homebox/data
```

如此简单。Overlord 会拦截我们在 [Director](https://github.com/DtxdF/director) 文件中定义的标签，并基于这些标签创建 DNS 记录。

```yml
$ overlord apply -f homebox.yml
$ overlord get-info -f homebox.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    homebox:
      state: DONE
      last_log: 2025-04-23_15h44m38s
      locked: False
      services:
        - {'name': 'homebox', 'status': 0, 'jail': '1f97e32f36'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 4 minutes and 4.1 seconds
        job_id: 6
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             homebox:
               error: False
               message: None
         skydns:
           services:
             homebox:
               error: False
               message: (project:homebox, service:homebox, records:[address:True,ptr:None,srv:None] records has been updated.
```

最后，我们的服务端点是 [http://homebox.overlord.lan:8666/](http://homebox.overlord.lan:8666/)

## 负载均衡

关于 SkyDNS 的一个有趣特性是，多个域名可以被分组管理。因此，如果我们在多台服务器上部署同一个服务，且它们使用相同的分组，DNS 请求将返回三个 A 记录（对于 IPv4），也就是三个 IPv4 地址。

hello-http.yml:

```yml
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - services
projectName: hello-http
projectFile: |
  options:
    - virtualnet: ':<random> default'
    - nat:
  services:
    darkhttpd:
      makejail: 'gh+DtxdF/hello-http-makejail'
      options:
        - expose: '9128:80 ext_if:tailscale0 on_if:tailscale0'
        - label: 'appjail.dns.alt-name:hello-http'
        - label: 'overlord.skydns:1'
        - label: 'overlord.skydns.group:hello-http'
        - label: 'overlord.skydns.interface:tailscale0'
      arguments:
        - darkhttpd_tag: 14.2
```

这带来的一个额外好处是，服务会以轮询（round-robin）的方式进行负载均衡，尽管这完全取决于客户端，但大多数现代客户端都会这么做。

```yml
$ overlord apply -f hello-http.yml
$ overlord get-info -f hello-http.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_16h26m08s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '7c2225c5fe'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 2 minutes and 43.3 seconds
        job_id: 28
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: None
         skydns:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, records:[address:True,ptr:None,srv:None] records has been updated.
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_16h26m09s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '3822f65e97'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 2 minutes and 18.56 seconds
        job_id: 13
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: None
         skydns:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, records:[address:True,ptr:None,srv:None] records has been updated.
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: r2
  labels:
    - all
    - r2
    - services
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_16h26m10s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '0e0e64eb3c'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 51.17 seconds
        job_id: 8
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: None
         skydns:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, records:[address:True,ptr:None,srv:None] records has been updated.
$ host -t A hello-http.overlord.lan
hello-http.overlord.lan has address 100.65.139.52
hello-http.overlord.lan has address 100.109.0.125
hello-http.overlord.lan has address 100.96.18.2
$ curl http://hello-http.overlord.lan:9128/
curl http://hello-http.overlord.lan:9128
Hello, world!
UUID: 472ffbdb-9472-4aa2-95ff-39f4bde214df
$ curl http://hello-http.overlord.lan:9128/
Hello, world!
UUID: 7db3b268-87bb-4e81-8be3-e888378fa13b
```

不过，我知道在大多数情况下，需要更复杂的配置。更糟糕的是，正如前面提到的，这还取决于客户端，因此可能符合，也可能不符合你的预期。

幸运的是，Overlord 提供了与 HAProxy 的集成，更准确地说，是与 Data Plane API 的集成，因此你的配置可以复杂到满足任何需求。

haproxy.yml:

```yml
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
    - r2
projectName: load-balancer
projectFile: |
  options:
    - alias:
    - ip4_inherit:
  services:
    haproxy:
      makejail: !ENV '${OVERLORD_METADATA}/haproxy.makejail'
      arguments:
        - haproxy_tag: 14.2-dataplaneapi
      options:
        - label: 'overlord.skydns:1'
        - label: 'overlord.skydns.group:revproxy'
        - label: 'overlord.skydns.interface:tailscale0'
```

metadata.yml:

```yml
kind: metadata
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
    - r2
metadata:
  haproxy.makejail: |
    ARG haproxy_tag=13.5
    ARG haproxy_ajspec=gh+AppJail-makejails/haproxy

    OPTION start
    OPTION overwrite=force
    OPTION copydir=${OVERLORD_METADATA}
    OPTION file=/haproxy.conf

    FROM --entrypoint "${haproxy_ajspec}" haproxy:${haproxy_tag}

    INCLUDE gh+DtxdF/efficient-makejail

    SYSRC haproxy_enable=YES
    SYSRC haproxy_config=/haproxy.conf
    
    SERVICE haproxy start

    STOP

    STAGE start

    WORKDIR /dataplaneapi

    RUN daemon \
            -r \
            -t "Data Plane API" \
            -P .master \
            -p .pid \
            -o .log \
                ./dataplaneapi \
                    -f /usr/local/etc/dataplaneapi.yml \
                    --host=0.0.0.0 \
                    --port=5555 \
                    --spoe-dir=/usr/local/etc/haproxy/spoe \
                    --haproxy-bin=/usr/local/sbin/haproxy \
                    --reload-cmd="service haproxy reload" \
                    --restart-cmd="service haproxy restart" \
                    --status-cmd="service haproxy status" \
                    --maps-dir=/usr/local/etc/haproxy/maps \
                    --config-file=/haproxy.conf \
                    --ssl-certs-dir=/usr/local/etc/haproxy/ssl \
                    --general-storage-dir=/usr/local/etc/haproxy/general \
                    --dataplane-storage-dir=/usr/local/etc/haproxy/dataplane \
                    --log-to=file \
                    --userlist=dataplaneapi
  haproxy.conf: |
    userlist dataplaneapi
      user admin insecure-password cuwBvS5XMphtCNuC

    global
      daemon
      log 127.0.0.1:514 local0
      log-tag HAProxy

    defaults
      mode http
      log global
      option httplog
      timeout client 30s
      timeout server 50s
      timeout connect 10s
      timeout http-request 10s

    frontend web
      bind :80
      default_backend web

    backend web
      option httpchk HEAD /
      balance roundrobin
```

我们即将在两台服务器上部署 HAProxy / Data Plane API。这样做的原因是为了避免出现 SPOF（单点故障）。至少在任意时刻，如果某个 HAProxy / Data Plane API 实例宕机，另一个实例仍能提供服务。

然而，Overlord 只能指向一个 Data Plane API 实例。因此，如果我们使用如下的两台服务器，就必须在每台机器上指定不同的 Data Plane API 实例。

如你在 [Director](https://github.com/DtxdF/director) 文件中所见，我们使用了 SkyDNS，这样客户端就可以使用 **revproxy.overlord.lan** 这个域名，而无需直接使用各自的 IP 地址。其优势在于，即使某个 HAProxy / Data Plane API 实例宕机，另一个仍可接手，客户端也能继续向其发送请求。

```yml
$ overlord apply -f metadata.yml
$ overlord apply -f haproxy.yml
$ overlord get-info -f haproxy.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    load-balancer:
      state: DONE
      last_log: 2025-04-23_17h04m01s
      locked: False
      services:
        - {'name': 'haproxy', 'status': 0, 'jail': '8d92fc6d2d'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 2 minutes and 20.12 seconds
        job_id: 30
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             haproxy:
               error: False
               message: None
         skydns:
           services:
             haproxy:
               error: False
               message: (project:load-balancer, service:haproxy, records:[address:True,ptr:None,srv:None] records has been updated.
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: r2
  labels:
    - all
    - r2
    - services
  projects:
    load-balancer:
      state: DONE
      last_log: 2025-04-23_17h04m02s
      locked: False
      services:
        - {'name': 'haproxy', 'status': 0, 'jail': '05c589c8a1'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 2 minutes and 53.27 seconds
        job_id: 10
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             haproxy:
               error: False
               message: None
         skydns:
           services:
             haproxy:
               error: False
               message: (project:load-balancer, service:haproxy, records:[address:True,ptr:None,srv:None] records has been updated.
```

/usr/local/etc/overlord.yml (centralita) 总机

```yml
dataplaneapi:
auth:
username: 'admin'
password: 'cuwBvS5XMphtCNuC'
entrypoint: 'http://100.65.139.52:5555'
```

/usr/local/etc/overlord.yml (provider): 提供者

```yml
dataplaneapi:
  auth:
    username: 'admin'
    password: 'cuwBvS5XMphtCNuC'
  entrypoint: 'http://100.65.139.52:5555'
```

hello-http.yml:

```yml
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - centralita
    - provider
projectName: hello-http
projectFile: |
  options:
    - virtualnet: ':<random> default'
    - nat:
  services:
    darkhttpd:
      makejail: 'gh+DtxdF/hello-http-makejail'
      options:
        - expose: '9128:80 ext_if:tailscale0 on_if:tailscale0'
        - label: 'overlord.load-balancer:1'
        - label: 'overlord.load-balancer.backend:web'
        - label: 'overlord.load-balancer.interface:tailscale0'
        - label: 'overlord.load-balancer.interface.port:9128'
        - label: 'overlord.load-balancer.set.check:"enabled"'
      arguments:
        - darkhttpd_tag: 14.2
```

收获！

```yml
$ overlord apply -f hello-http.yml
$ overlord get-info -f hello-http.yml -t projects --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: provider
  labels:
    - all
    - provider
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_17h57m16s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '79f16243de'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 1 minute and 22.1 seconds
        job_id: 1
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:fa8f94b1-6b2b-4cb4-a808-e9da46014c86, code:202, transaction_id:8fcf5d67-12df-4fcd-a2e3-1f5f18fe1844, commit:1) server has been successfully added.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_17h57m16s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '52dfa071cb'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 1 minute and 19.53 seconds
        job_id: 15
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:f4b9e170-67bb-403e-88da-112c55b45fce, code:202, transaction_id:0fa886c8-68a6-4716-aa6b-824aa3e776ad, commit:1) server has been successfully added.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
$ curl http://revproxy.overlord.lan
Hello, world!
UUID: 8579af73-7d11-40b3-8444-6dac62e34b8e
$ curl http://revproxy.overlord.lan
Hello, world!
UUID: e463b1d5-13eb-4f04-9b0a-caf4339a8058
```


## 横向自动扩展

即使在拥有数百台服务器的情况下，部署项目也变得非常容易。然而，这种方式的问题在于资源的浪费，因为客户端很可能只使用了集群资源的不到 5%；或者相反，你可能将项目部署在少数几台你认为 "够用" 的服务器上，直到某一时刻你发现资源根本不够用，更糟糕的是，这些服务器还可能因各种原因随时宕机。这正是 Overlord 的自动扩展机制能够解决的问题。

hello-http.yml:

```yml
kind: directorProject
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - desktop
projectName: hello-http
projectFile: |
  options:
    - virtualnet: ':<random> default'
    - nat:
  services:
    darkhttpd:
      makejail: 'gh+DtxdF/hello-http-makejail'
      options:
        - expose: '9128:80 ext_if:tailscale0 on_if:tailscale0'
        - label: 'overlord.load-balancer:1'
        - label: 'overlord.load-balancer.backend:web'
        - label: 'overlord.load-balancer.interface:tailscale0'
        - label: 'overlord.load-balancer.interface.port:9128'
        - label: 'overlord.load-balancer.set.check:"enabled"'
      arguments:
        - darkhttpd_tag: 14.2
autoScale:
  replicas:
    min: 3
  labels:
    - services
    - provider
```

正如你可能已经注意到的，我们指定了两种类型的标签。这体现了自动扩缩部署与非自动扩缩部署之间的细微差别。与非自动扩缩部署不同，**deployIn.labels** 中的标签用于匹配相应的服务器以进行自动扩缩和监控，换句话说，匹配这些标签（在本例中为 **desktop**）的服务器将负责部署、监控，以及在必要时重新部署。另一方面，匹配 **autoScale.labels** 中标签（在本例中为 **services** 和 **provider**）的服务器则用于以非自动扩缩部署的方式部署项目。我们已指定该项目至少包含三个副本。我们还可以指定其他内容，例如 **rctl(8)** 规则，但为了简洁起见，这样就足够了。

```yml
$ overlord apply -f hello-http.yml
$ overlord get-info -f hello-http.yml -t projects --filter-per-project --use-autoscale-labels
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_19h36m17s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '0524bcf91b'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 58.24 seconds
        job_id: 31
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:0d67d160-61af-4810-b277-5fb9e20da8eb, code:202, transaction_id:baa5b939-f724-4bd3-9d65-2ef769def3f5, commit:1) server has been successfully added.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: provider
  labels:
    - all
    - provider
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_20h00m11s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '2c2d22d2a5'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 4 minutes and 46.3 seconds  
        job_id: 6
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:fa8f94b1-6b2b-4cb4-a808-e9da46014c86, code:202, transaction_id:6792e6fe-a
778-44a7-b23a-1b2c23fe5904, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_20h04m25s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': 'a6549318ce'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 33.34 seconds
        job_id: 21
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:f4b9e170-67bb-403e-88da-112c55b45fce, code:202, transaction_id:00e632ce-c215-4784-9e61-9507d914ba6a, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
$ overlord get-info -f hello-http.yml -t autoscale --filter-per-project
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    hello-http:
      autoScale:
        last_update: 7.65 seconds
        operation: COMPLETED
        output:
         message: None
$ curl http://revproxy.overlord.lan
Hello, world!
UUID: 08951a86-2aef-4e85-9bfc-7fe68b5cc62d
$ curl http://revproxy.overlord.lan
Hello, world!
UUID: 5a06a89d-6109-438e-bc04-1ef739473994
```

假设 **provider** 中的服务由于某种原因宕机。

```yml
$ overlord get-info -f hello-http.yml -t projects --filter-per-project --use-autoscale-labels
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_19h36m17s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '0524bcf91b'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 13 minutes and 37.64 seconds
        job_id: 32
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:0d67d160-61af-4810-b277-5fb9e20da8eb, code:202, transaction_id:a2ba93d7-6ce6-4a36-aab2-09be13a00c17, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: provider
  labels:
    - all
    - provider
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_20h11m58s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 66, 'jail': '2c2d22d2a5'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 1 minute and 13.9 seconds
        job_id: 7
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:fa8f94b1-6b2b-4cb4-a808-e9da46014c86, code:202, transaction_id:b19e7997-871c-4293-a8a9-51ce03f2bbaa, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_20h04m25s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': 'a6549318ce'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 8 minutes and 29.3 seconds
        job_id: 21
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:f4b9e170-67bb-403e-88da-112c55b45fce, code:202, transaction_id:00e632ce-c215-4784-9e61-9507d914ba6a, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
```

在没有任何人工干预的情况下，让我们看看魔法是如何发生的。

```yml
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: None
  labels:
    - all
    - desktop
    - services
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_19h36m17s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '0524bcf91b'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 14 minutes and 30.7 seconds
        job_id: 32
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:0d67d160-61af-4810-b277-5fb9e20da8eb, code:202, transaction_id:a2ba93d7-6ce6-4a36-aab2-09be13a00c17, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: provider
  labels:
    - all
    - provider
    - vm-only
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_20h13m37s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': '2c2d22d2a5'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 27.47 seconds
        job_id: 8
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:fa8f94b1-6b2b-4cb4-a808-e9da46014c86, code:202, transaction_id:98ca3a6a-65e4-450b-a4c3-4f135e36be37, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    hello-http:
      state: DONE
      last_log: 2025-04-23_20h04m25s
      locked: False
      services:
        - {'name': 'darkhttpd', 'status': 0, 'jail': 'a6549318ce'}
      up:
        operation: COMPLETED
        output:
         rc: 0
         stdout: {'errlevel': 0, 'message': None, 'failed': []}
        last_update: 9 minutes and 22.37 seconds
        job_id: 21
        restarted: False
        labels:
         error: False
         message: None
         load-balancer:
           services:
             darkhttpd:
               error: False
               message: (project:hello-http, service:darkhttpd, backend:web, serverid:f4b9e170-67bb-403e-88da-112c55b45fce, code:202, transaction_id:00e632ce-c215-4784-9e61-9507d914ba6a, commit:1) server has been successfully updated.
         skydns:
           services:
             darkhttpd:
               error: False
               message: None
```

服务已经恢复运行。

## 信息、指标及更多内容……

多亏了 [AppJail](https://github.com/DtxdF/AppJail)，我们可以从 jail 中获取大量信息。Overlord 有一个特殊的部署方式叫做 **readOnly**，它可以与 **get-info** 命令完美结合。

info.yml:

```yml
kind: readOnly
datacenters:
  main:
    entrypoint: !ENV '${ENTRYPOINT}'
    access_token: !ENV '${TOKEN}'
deployIn:
  labels:
    - all
$ overlord get-info -f info.yml -t projects --filter adguardhome
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  projects:
    adguardhome:
      state: DONE
      last_log: 2025-04-07_17h32m40s
      locked: False
      services:
        - {'name': 'server', 'status': 0, 'jail': '2a67806954'}
$ overlord get-info -f info.yml -t jails --filter 2a67806954
datacenter: http://127.0.0.1:8888
  entrypoint: main
  chain: centralita
  labels:
    - all
    - centralita
    - services
  jails:
    2a67806954:
      stats:
        cputime: 91
        datasize: 8400896 (8.01 MiB)
        stacksize: 0 (0 bytes)
        coredumpsize: 0 (0 bytes)
        memoryuse: 75104256 (71.62 MiB)
        memorylocked: 0 (0 bytes)
        maxproc: 4
        openfiles: 296
        vmemoryuse: 1367982080 (1.27 GiB)
        pseudoterminals: 0
        swapuse: 0 (0 bytes)
        nthr: 13
        msgqqueued: 0
        msgqsize: 0
        nmsgq: 0
        nsem: 0
        nsemop: 0
        nshm: 0
        shmsize: 0 (0 bytes)
        wallclock: 363548
        pcpu: 0
        readbps: 0 (0 bytes)
        writebps: 0 (0 bytes)
        readiops: 0
        writeiops: 0
      info:
        name: 2a67806954
        network_ip4: 10.0.0.3
        ports: 53/tcp,53/udp,53/tcp,53/udp
        status: UP
        type: thin
        version: 14.2-RELEASE
      cpuset: 0, 1
      expose:
        - {'enabled': '1', 'name': None, 'network_name': 'ajnet', 'hport': '53', 'jport': '53', 'protocol': 'udp', 'ext_if': 'tailscale0', 'on_if': 'tailscale0', 'nro': '3'}
        - {'enabled': '1', 'name': None, 'network_name': 'ajnet', 'hport': '53', 'jport': '53', 'protocol': 'tcp', 'ext_if': 'jext', 'on_if': 'jext', 'nro': '0'}
        - {'enabled': '1', 'name': None, 'network_name': 'ajnet', 'hport': '53', 'jport': '53', 'protocol': 'tcp', 'ext_if': 'tailscale0', 'on_if': 'tailscale0', 'nro': '2'}
        - {'enabled': '1', 'name': None, 'network_name': 'ajnet', 'hport': '53', 'jport': '53', 'protocol': 'udp', 'ext_if': 'jext', 'on_if': 'jext', 'nro': '1'}
      fstab:
        - {'enabled': '1', 'name': None, 'device': '/var/appjail-volumes/adguardhome/db', 'mountpoint': 'adguardhome-db', 'type': '<volumefs>', 'options': 'rw', 'dump': '0', 'pass': None, 'nro': '0'}
      labels:
        - {'value': '1', 'name': 'overlord.skydns'}
        - {'value': 'adguardhome', 'name': 'appjail.dns.alt-name'}
        - {'value': 'tailscale0', 'name': 'overlord.skydns.interface'}
        - {'value': 'adguardhome', 'name': 'overlord.skydns.group'}
      nat:
        - {'rule': 'nat on "jext" from 10.0.0.3 to any -> ("jext:0")', 'network': 'ajnet'}
      volumes:
        - {'mountpoint': 'usr/local/etc/AdGuardHome.yaml', 'type': '<pseudofs>', 'uid': None, 'gid': None, 'perm': '644', 'name': 'adguardhome-conf'}
        - {'mountpoint': '/var/db/adguardhome', 'type': '<pseudofs>', 'uid': None, 'gid': None, 'perm': '750', 'name': 'adguardhome-db'}
```

## 后续工作

Overlord 能为你做的事情远不止本文件所展示的内容，更多示例请参见 [Wiki](https://github.com/DtxdF/overlord/wiki)。

Overlord 是一个较新的项目，仍有很大的改进空间，未来也将加入更多功能以提升可用性。如果你愿意支持该项目，请 [考虑捐款](https://www.patreon.com/AppJail)。

---

**Jesús Daniel Colmenares Oviedo** 是一位系统管理员、开发者及 Ports 提交者，同时也是 AppJail 及其相关项目（如 Director、Makejails、Reproduce、Overlord 等）的创建者。
