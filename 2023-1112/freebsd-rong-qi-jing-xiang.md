# FreeBSD 容器镜像

- 原文链接：[FreeBSD Container Images](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-14-0/freebsd-container-images/)
- 作者：Doug Rabson

OCI 容器引擎，比如 [containerd](https://containerd.io/) 和 [podman](https://podman.io/)，需要容器镜像。容器镜像是个只读目录树，通常包含一款应用程序及其支持文件和库。容器镜像在容器引擎上运行时，会创建该镜像的可写副本，并在某种隔离环境（如 jail）中执行该应用程序。通过注册中心分发容器镜像。注册中心存储镜像数据，提供了一个简单的 REST API 来访问镜像及其元数据。由[开放容器标准](https://opencontainers.org/)对注册中心的 API、镜像格式和元数据进行标准化，基本上取代了早期的 Docker 格式。

## OCI 镜像

镜像被表示为一系列层，每层都存储成 tar 压缩文件。为了解压镜像，我们从一个空目录开始，然后依次解压每层，后面的层可以增加文件及修改先前层中的文件。通常，这个过程的结果会被容器引擎缓存。

除了层数据，还额外使用了两个元数据对象。清单（manifest）列出了各个层，可包含描述镜的注解。镜像配置（image config）描述了目标操作系统和架构，并可使用默认命令来运行镜像。

所有这些都存储在一个“内容可寻址”的结构中，其中组件的哈希值用于为其命名。例如，我用于静态链接应用程序的小型基础镜像如下所示：

```sh
$ ls -lR
total 6
drwxr-xr-x 3 root dfr 3 Sep 8 10:36 blobs
-rw-r--r-- 1 root dfr 275 Sep 8 10:36 index.json
-rw-r--r-- 1 root dfr 31 Sep 8 10:36 oci-layout

./blobs:
total 25
drwxr-xr-x 2 root dfr 6 Sep 8 10:36 sha256

./blobs/sha256:
total 950
-rw-r--r-- 1 root dfr 1143 Sep 8 10:36
190e4f8bf39f4cc03bf0f723607e58ac40e916a1c15bd212486b6bb0a8c30676
-rw-r--r-- 1 root dfr 496 Sep 8 10:36
5657eb844c0c0142aa262395125099ae065e791157eaa1e1d9f5516531f4fe30
-rw-r--r-- 1 root dfr 34916 Sep 8 10:36
5af368a2a6078dc912135caed94a6375229a5a952355f5fea60dad1daf516f78
-rw-r--r-- 1 root dfr 911102 Sep 8 10:36
fdb4ee0a131a70df2aae5c022b677c5afbacb5ec19aa24480f9b9f5e8f30fd18
```

此捆绑包中的所有元数据文件都是 JSON 格式，如此处所描述。顶层的 `index.json` 文件通过其哈希链接到清单（manifest）：

```ini
$ cat index.json | jq
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:190e4f8bf39f4cc03bf0f723607e58ac40e916a1c15bd212486b6bb0a8c30676",
      …
    }
  ]
}
```

该清单描述了两个数据层，一层仅包含 FreeBSD 标准目录结构，另一层包含最小的支持文件，如 `/etc/passwd` 和 SSL 证书。它还链接到了配置文件，其中包含目标操作系统和架构。

使用这样的内容可寻址格式，可以更容易地共享存储空间，并减少在使用多个基于相同基础镜像派生的镜像时，所需的下载数据量。

OCI 镜像规范还支持多架构镜像，它们只是清单的列表：

```ini
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
      "manifests": [
          {
              "mediaType": "application/vnd.oci.image.manifest.v1+json",
              "size": 1116,
              "digest":
"sha256:598b927b8ddc9155e6d64f88ef9f9d657067a5204d3d480a1b1484da154e7c4",
              "platform": {
                 "architecture": "amd64",
                 "os": "freebsd"
              }
          },
          {
              "mediaType": "application/vnd.oci.image.manifest.v1+json",
              "size": 1118,
              "digest":
"sha256:ac732db0f4788d5282a8d16fefbea360d937049749c83891367abd02801b582",
              "platform": {
                 "architecture": "arm64",
                 "os": "freebsd"
              }
         }
    ]
}
```

## FreeBSD 基本镜像

为了更方便地在 FreeBSD 上使用容器，需要合适的基本镜像。传统的 FreeBSD 发布流程生成的基本镜像，包含了一小部分用于在物理机和虚拟机上安装完整功能的 FreeBSD 操作系统的包。我们可以使用 base.txz 这个包来构建我们的基本镜像，但这会生成个大小为 1GB 的镜像，对于大多数应用程序来说，其中超过 90% 的内容是多余的。相比之下，大多数 Linux 发行版提供的基本镜像要小得多——比如，Ubuntu 官方镜像约为 80MB。

幸运的是，pkgbase 项目正在努力构建一个精细化的包集合，将传统的 base.txz 压缩包拆分成数百个更小的包。目前，这个集合包括许多单独的库和实用程序包，以及两个较大的包：FreeBSD-runtime 包含 shell 和一些核心实用程序，FreeBSD-utilities 包含一组常用的实用程序。

早先，我使用 pkgbase 创建了一个“最精简”镜像，包含 FreeBSD-runtime、SSL 证书和 pkg。这个镜像约为 80MB，足以支持简单的 shell 脚本，并且能够安装其他软件包。与类似的 Linux 镜像相比，这个镜像效果相当好，尽管它还无法与基于 busybox 的 alpine 镜像相媲美：后者的大小仅为 7.5MB。此后，我制作了一个小型的镜像家族，部分受 distroless 项目的启发：

- “static”：仅包含 SSL 证书和时区数据。可以作为静态链接应用程序的基础。
- “base”：在“static”基础上，增加了一些共享库，以支持大量的动态链接应用程序。
- “minimal”：在“base”基础上，增加了 FreeBSD-runtime 包和包管理功能。
- “small”：在“minimal”基础上，增加了 FreeBSD-utilities，能支持更多基于 shell 的应用程序。

为了支持多种 FreeBSD 版本，我将版本号嵌入镜像名称中，例如，“freebsd13.2-minimal:latest”包括来自最新版本的 releng/13.2 分支的包，而“freebsd13-minimal:latest”则是从 stable/13 构建的。我为 amd64 和 arm64 架构构建了所有这些镜像，容器引擎可自动从清单列表中选择正确的镜像。

## 安全性

容器镜像的安全性至关重要，必须能够验证其来源是否可信，并确保在传输到容器引擎的过程中未被篡改。

镜像的清单通常包含镜像数据层的 SHA256 哈希值以及对应的镜像配置文件的哈希值。这意味着可以通过清单的哈希值唯一标识镜像，并使用它来验证镜像，例如通过在可信的位置列出可信的镜像哈希。

另外，哈希值可以用于创建签名，证明该镜像得到了某个公钥持有者的信任。常用的两种机制是：Podman 使用的工具 [sigstore](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/assembly_signing-container-images_building-running-and-managing-containers)，它使用 PGP 创建镜像签名，提供了将镜像集合与签名存储（可以是本地目录或可信网站）关联的机制。这可以用于在拉取镜像时，验证镜像是否与签名匹配。sigstore 的替代方案是 [cosign](https://github.com/sigstore/cosign)，它将签名与镜像一起存储在镜像库中。

## 短板与后续工作

虽然这些镜像很有用，但它们包含的已安装软件包是根据相对无序的选择来决定的。最初，我维护了 `alpha.pkgbase.live` 软件仓库，这简化了通过安装额外包来扩展镜像的过程。不幸的是，该项目失去了资助，一段时间内，没有公开的 pkgbase 仓库可用。幸运的是，这个问题已经得到解决，现在可以从标准的 FreeBSD 软件仓库获取 pkgbase 包。

当前的镜像构建机制使用 pkg 安装 pkgbase 包到镜像层中。这种方式很方便，并且能够记录安装到镜像中的内容。不幸的是，pkg 的元数据存储在 sqlite 数据库中，它不支持可重现的构建。sqlite 数据库中包含了软件包安装的时间戳——虽然可以将其覆盖为一个合适的常量时间，但即便如此，sqlite 数据库仍然不是可重现的。更大的问题是可信度——我将这些镜像托管在自己的个人仓库中（docker.io 和 quay.io），但从潜在用户的角度来看，缺乏理由认同这些镜像是可信的。即便我使用来自 FreeBSD 包仓库的软件包构建镜像，可这些镜像未签名，也未得到 FreeBSD 项目的支持。

在我看来，这是 FreeBSD 容器引擎潜在用户面临的一个重大障碍，阻碍了这些项目从当前的“实验性”状态过渡到可以考虑用于生产的状态。这一点已通过与多个开源项目的讨论得到证实，这些项目正在构建和使用镜像，并希望支持 FreeBSD 这个平台。

在理想情况下，除了托管 pkgbase 包之外，FreeBSD 项目还应该构建 FreeBSD 容器镜像，可以托管一个镜像注册中心，或者将这些镜像发布到像 docker.io 这样的公共仓库。我计划原型化对发布构建基础设施的扩展，将容器镜像构建集成到现有的 pkgbase 框架中，这可能有助于推动这一进展。

---

**Doug Rabson** 是一位软件工程师，拥有三十余年的经验，涉及从 1980 年代的 8 位文字冒险游戏到 2020 年代的每秒 TB 级分布式长期聚合系统。自 1994 年以来，他始终是 FreeBSD 项目的成员和提交者，目前正在致力于改善 FreeBSD 对现代容器编排系统（如 Podman 和 Kubernetes）的支持。
