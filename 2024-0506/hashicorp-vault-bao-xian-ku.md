# Hashicorp Vault

- 作者：Dave Cottlehuber
- 原文链接：<https://freebsdfoundation.org/our-work/journal/browser-based-edition/configuration-management-2/hashicorp-vault/>

居家工作已成为新常态。但从安全性角度看，事情变得更为复杂。安全的办公室不复存在，那些精心锚定的安全边界和全天候的物理安全亦已消失。

专业的安全人员将我们关心的此类风险称为“威胁格局（threat landscape）”和“安全态势（security posture）”。

这是一种花哨的说法，即便我们可以下定决心，对诸如 GCSB（政府通信安全局）、克格勃（KBG，苏联国家安全委员会）、CIA（美国中央情报局）和摩萨德（Mossad，以色列情报和特殊使命局）及其他政府资助的攻击者漠不关心。但我们确实在意丢失在火车上笔记本电脑，或有人潜入办公室，窃取高价值物品——那些我们在乎的密码和凭据。

## 但是要把秘密放在哪呢？

尤其值得注意的是，攻击者和我们这些系统管理员和开发人员的共同兴趣是管理和定期轮换秘密。

你见过有多少环境，那里的数据库凭据从未修改过呢？跨系统、长时间运行的批处理作业，却使用着相同的密码？或者改变其中任何一个都可能导致意外停机，似火一样在公司内蔓延，并最终砸掉你的饭碗？

很显然，把便笺、凭据存放在 git 存储库，既不具可扩展性，也不安全。我们需要一些适用于绝大多数用例的，较为通用的解决方案。

那么聪明的 DevOps 工程师该怎么做呢？

传统解决方案是密码存储（如 BitWarden、1Password），并将这些工具扩展到团队中的各种命令行工具中。但这些工具只能解决小部分问题，并不能彻底解决所有问题。

它们主要是面向用户的工具，很难与复杂的 git 存储库、puppet 和 ansible 部署管道连接起来；也很难确保凭据定期轮换，并确保仅有指定的用户和系统才能访问。

## 秘密管理平台

秘密管理平台（SMP）有时也被称为密钥管理系统（KMS），通常被云供应商集成。Azure、Amazon、Google、Oracle 都为其平台提供了紧密耦合且完美集成的工具。

但是，如果你现在正在阅读着这本 FreeBSD 杂志，你极有可能对把所有秘密交给商业公司的做法感到极度不安。在理想情况下，我们希望自己来管理秘密，而不依赖外部第三方。

## 权衡利弊

这些系统专注于以高度安全和受控的方式分发和管理机要，使运维团队能够进行操作。

它们旨在解决在现代复杂环境中遇到的安全管理和编排秘密的挑战，在这些环境中，应用程序、系统和用户需要访问极度敏感的信息。

例如，它们能够集成修改秘密、触发采用该秘密的容器换代到新版本等功能。

他们可以使批处理作业能够临时解密金融信息，然后在安全加密后返回更新信息。

或者它可以提供一次性令牌：让新配置的服务连接到指定数据库，确保令牌既未在传输过程中被劫持，也无法被多个实例使用。

当配置（部署）新实例时，将获得新的一次性令牌，绑定实例和时间。

此类型的功能非常适合将系统部署和相关机密信息的访问分离开。这种掩人耳目的方式通常被称为单间部署——单个仅限一次性的令牌，紧密绑定到单个部署，在部署过程中由自动化工具集注入。在实例启动时使用此令牌，用于获取专用于该实例的运行时机密，亦可通过 IP 地址，或其他附加条件来绑定。

## 入门指南

本文主要介绍了 Hashicorp Vault，它有一个功能等价的开源分支叫 OpenBao——类似于 Terraform 是 OpenTofu 许可下的复刻。

其他工具也能用，但 [Vault](https://www.vaultproject.io/) 已经被移植到了 FreeBSD，我相信 [OpenBao](https://openbao.org/) 也很快就会被移植（**译者注：目前正在进行审查 <https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=280619>**）。

## Vault 的甜蜜点

Vault（和其他 KMS）并非存储 Webstorm IDE 激活码和护照扫描件的好地方。Vault 对终端用户并不友好，也注定无法在移动设备上使用。

但如果你正管理着服务器、数据库和网络，那它就非常棒。它可以轻松与 Terraform、Chef、Puppet、Ansible 以及几乎所有带有和使用命令行及终端界面的东西集成。

## 内部设计

Vault 将所有密钥和信息加密存储在磁盘上。因此，在启动时需要一个主密钥来解锁所有其他密钥。为了避免单一轻量化密钥，Vault 使用 [SSS](https://en.wikipedia.org/wiki/Shamir%27s_secret_sharing) 和 Shamir 的秘密共享将一个庞大又复杂的密钥分割成单独的秘密，这些秘密可重新组合用以解锁 Vault。在支持 WASM 的现代网页浏览器中可测试 [Shamir 示例](https://bakaoh.com/sss-wasm/%5D)。

[SSS](https://en.wikipedia.org/wiki/Shamir%27s_secret_sharing) 有着精巧地可配置的冗余度——比如仅需 5 把钥匙中的 3 把即可解锁保险柜。因此，你需要 3 位主系统管理员就能解锁它。但如果都不在场，你可以请求你的律师、会计借用他们的钥匙（如需要），以达到你的 3 位法定人数。每位管理员都在本地提交其解锁钥匙，并使用 API 质询来防止单一管理员窃取所有的主密钥。

在保险柜解锁后，从用户视角看，它的功能基本等价于其他 HTTP 可访问的键值存储。我们可以存储诸如 ssh 私钥、TLS 证书、常规密码，甚至让保险柜生成定时的限临时权限密码。

## 入门指南

Vault 支持使用共识协议和多个服务器的复杂部署，但我发现一台小型高度可靠的物理服务器（配备热备份和 zfs-replicate 备份）足矣。当然，我使用 Tarsnap 来进行彻底的离线备份——对于像我们所有的秘密如此关键的东西，这是绝对必要的！

## 安装和配置

通常须以 root 身份执行以下命令：

```
# pkg install -r FreeBSD security/vault
# mkdir -p /var/{db,log}/vault /usr/local/etc/vault
# chown root:vault /var/{db,log}/vault /usr/local/etc/vault
# chmod 0750 /usr/local/etc/vault
# chmod 0770 /var/{db,log}/vault
```

你可使用 `sysrc(8)`，或你首选的操作工具来设置 `rc.conf`。

```
# /etc/rc.conf.d/vault or where-ever you prefer
vault_enable=YES
vault_config=/usr/local/etc/vault/vault.hcl
```

以及 vault 的配置文件。它很多选项，但大部分意义自明。对于我们的测试部署，我们会禁用 TLS 并使用回环 IP。

```
# /usr/local/etc/vault/vault.hcl
default_lease_ttl = “72h”
max_lease_ttl = “168h”

ui = true
disable_mlock = false

listener “tcp” {
address = “127.0.0.1:8200”
tls_disable = 1
tls_min_version = “tls12”
tls_key_file = “/usr/local/etc/vault/vault.key”
tls_cert_file = “/usr/local/etc/vault/vault.all”
}

storage “file” {
path = “/var/db/vault”
}
```

现在以前台模式运行守护进程：

```
$ vault server -config /usr/local/etc/vault/vault.hcl
==> Vault server configuration:

Administrative Namespace

            Api Address: http://127.0.0.1:8200
...
```

打开新的终端，让我们检查状态：

```
$ export VAULT_ADDR=http://localhost:8200/
$ vault status
vault status
Key                 Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version             1.14.1
Build Date         2023-11-04T05:16:56Z
Storage Type       file
HA Enabled         false
```

注意，vault 尚未初始化，并且仍然处于封闭状态。让我们解决这个问题：

```
$ vault operator init --key-shares=3 --key-threshold=2
Unseal Key 1: jjcVgHTjWw3j4BsyDhugvS9we5t5qMAhJL8bSWzySjbG
Unseal Key 2: WfMeZPA7ixleQAMeeAqyey+gwrxDn9WNfSvdKzdLMaeA
Unseal Key 3: V9cd1eVBH6mstyoS2pbD6S80R7NJVz7jPvlPOcLOUVlw
Initial Root Token: hvs.RAeqzETRhOXOImMPw7xrXbAl

$ export VAULT_TOKEN=hvs.RAeqzETRhOXOImMPw7xrXbAl

$ vault status

Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed              true
Total Shares       3
Threshold          2
Unseal Progress    0/2
Unseal Nonce       n/a
Version            1.14.1
Build Date         2023-11-04T05:16:56Z
Storage Type       file
HA Enabled         false
```

注意，vault 已初始化，但仍处于封闭状态。接下来让我们使用新生成的密钥共享来解决这个问题：

```
$ vault operator unseal
Unseal Key (will be hidden):
Key Value
--- -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       3
Threshold          2
Unseal Progress    1/2
Unseal Nonce       6ce4351d-012b-df3f-a176-34d266f00795
Version            1.14.1
Build Date         2023-11-04T05:16:56Z
Storage Type       file
HA Enabled         false
```

请用不同的密钥重复解封，直至 `sealed` 变更为 `false` 。最后一步是启用审计，因为安全人员喜欢日志。

```
$ vault audit enable file path=/var/log/vault/audit.log
Success! Enabled the file audit device at: file/
```

可随时查看，这里从未存储过任何秘密，所以它只是请求的审计日志。

### Shamir 密钥环

现在你已经打开了 Vault，把你的秘密通过加密的信鸽分发给你选择的秘密保管者。需要进行某种对应的仪式，并确保这些秘密得到充分保护，既要避免失误和其他问题，也要防范摩萨德和朝鲜特工。

到现在，你应该已经准备好存储秘密了。

### 存储秘密

Vault 有引擎这么一个概念——涉及简单的键值存储，还有用于 ssh 证书、AWS 和 Google Cloud 集成、RabbitMQ、PostgreSQL 等的引擎。每个引擎都需要单独启用。

```
$ vault secrets enable -version=2 kv
Success! Enabled the kv secrets engine at: kv/
```

从这儿开始，我们需要指定引擎类型和其挂载路径。可以将数据检索为 JSON 或 yaml 格式，甚至可以直接存储文件。

```
$ vault kv put -mount=kv blackadder scarlet_pimpernel=”we do not know”
=== Secret Path ===
kv/data/blackadder

======= Metadata =======
Key Value
--- -----
created_time 2024-05-12T23:04:50.283028044Z
custom_metadata <nil>
deletion_time n/a
destroyed false
version 1

$ vault kv get -mount=kv -format=json blackadder
{
“request_id”: “48141452-8f8f-b497-9c53-1af71e24e2a5”,
“lease_id”: “”,
“lease_duration”: 0,
“renewable”: false,
“data”: {
“data”: {
“scarlet_pimpernel”: “we do not know”
},
“metadata”: {
“created_time”: “2024-05-12T23:04:50.283028044Z”,
“custom_metadata”: null,
“deletion_time”: “”,
“destroyed”: false,
“version”: 1
}
},
“warnings”: null
}

$ vault kv put -mount=kv blackadder scarlet_pimpernel=”comte de frou frou”
=== Secret Path ===
kv/data/blackadder

======= Metadata =======
Key Value
--- -----
created_time 2024-05-12T23:08:22.369551931Z
custom_metadata <nil>
deletion_time n/a
destroyed false
version 2

$ vault kv get -mount=kv -format=yaml blackadder
data:
data:
scarlet_pimpernel: comte de frou frou
metadata:
created_time: “2024-05-12T23:08:22.369551931Z”
custom_metadata: null
deletion_time: “”
destroyed: false
version: 2
lease_duration: 0
lease_id: “”
renewable: false
request_id: 686965d9-811f-8689-d75f-a02f7dded9a7
warnings: null

$ vault kv put kv/blackadder scarlet_pimpernel=@/etc/motd.template
```

## 基于角色的访问控制

Vault 可以配置为使用 GitHub 认证，并将角色和认证委托给除 LDAP 外的其他系统。你们中的许多人会为这个消息而高兴。使用 GitHub 认证可以强制所有用户使用双因素身份验证（2FA），因此对小团队来说这是一个合理的权衡。

```
$ vault auth enable github
vault auth enable github
Success! Enabled github auth method at: github/

$ vault write auth/github/config organization=skunkwerks
Success! Data written to: auth/github/config

$ vault write auth/github/map/teams/admin value=admins
Success! Data written to: auth/github/map/teams/admin
```

将这个小策略文件放置在 `/usr/local/etc/vault/admins.hcl`：

```
# 授予 GitHub 管理员组成员在 `kv/` 挂载点中的所有权限
path “kv/*” {
capabilities = [“create”, “read”, “update”, “delete”, “list”]
}
```

然后在 Vault 中启用它：

```
$ vault policy write admins /usr/local/etc/vault/admins.hcl
Success! Uploaded policy: admins
```

你还可以为各种群体制定更为严格的策略（例如在权限、路径和所选挂载点），比如部署机器人。

最后，那些希望通过 github 认证来访问 vault 的用户，必须访问 <https://github.com/settings/tokens>，添加新的个人令牌，权限为 `admin— read:org`。

现在就可以使用此令牌生成你的 vault 登录令牌了。

```
$ vault login -method=github token=$GITHUB
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run “vault login”
again. Future Vault requests will automatically use this token.

Key Value
--- -----
token hvs....
token_accessor ...
token_duration 72h
token_renewable true
token_policies [“admins” “default”]
identity_policies []
policies [“admins” “default”]
token_meta_org skunkwerks
token_meta_username dch

$ vault kv get -mount=kv -format=yaml blackadder
...
```

## Vault 自动化

诸如 Chef、Puppet、Ansible 等自动化工具可以在部署时使用 Vault 存储密钥来解密；或在某些情况下，仅运行时解密。

让我们来看看第一种情况，即在部署时解密密钥。这实际上扩展了自动化工具的模板化能力，并依赖于能够在推送新的和更新的密钥之后触发服务重启。

我们有 4 种方式来使用 Vault：

- [Ansible](https://www.ansible.com/) 和类似工具可以将秘密存储在 Vault 中，并且只在部署时使用查找功能解密它们
- 框架脚本 `rc.d` 可在启动使用 [app roles](https://developer.hashicorp.com/vault/docs/auth/approle) 时提取它们的凭据，让本地 root 用户拥有仅允许发出小单间令牌的委派令牌。守护进程本身仅可检索自己的凭据
- Vault 可以发行绑定时间和 IP 的动态凭据（过期后会被吊销），适用于守护进程、定时任务和批处理脚本的时间限制
- 我们还可以在运行时使用 Vault 代理来模板化文件

### Ansible

有许多 ansible 插件，令人困惑的是，ansible 自带模块“vault”与 Hashicorp Vault 并不兼容。

安装插件，并使用典型的 lookup 功能：

```
super_secret: “{{lookup('hashivault', 'kv', 'blackadder', version=2)}}”
```

### rc.d App Roles

AppRole 是一种自带的认证方法，专门用于机器和应用程序进行 Vault 认证，随后获取令牌，仅允许获取相关密钥。这通常被称为 cubby-hole（小单间）凭据，因为它只允许解包外层，获取内部密钥。

这些限制涉及时间限制、使用次数限制等。我们以受信任根进程生成这个受限的秘密 ID，并将其与角色 ID 一同传递给守护进程，以获取其自身的凭证。秘密 ID 可以设置为只有这些凭据才能被生成。

再次启用 `approle` 挂载点，然后创建我们的应用程序特定凭据，因为它是一种身份验证形式。为方便起见，这个 approle 将复用之前使用的 `admins` 组策略，但它应该有一个更为严格的策略，专门用于这个守护程序和服务。

```
$ vault auth enable approle
Success! Enabled approle auth method at: approle/

$ vault write auth/approle/role/beastie \
secret_id_ttl=60m \
token_num_uses=10 \
token_ttl=1h \
token_max_ttl=4h \
secret_id_num_uses=40 \
policies=”default,admins”
Success! Data written to: auth/approle/role/beastie

$ vault read auth/approle/role/beastie/role-id
Key Value
--- -----
role_id 6caaeac3-d8fa-a0e3-83ba-7d37750603c2

$ vault write -f auth/approle/role/beastie/secret-id
Key Value
--- -----
secret_id 8dd54c92-fe54-0d6d-bee6-e433e815aaa1
secret_id_accessor cb9bc17c-c756-42b3-c391-b61ebde12bff
secret_id_num_uses 0
secret_id_ttl 0s
```

如果我们想要使这些秘密在设想的守护程序 `beastie` 中可用，可以将这两个参数放入一个 `/etc/rc.conf.d/beastie` 文件，该文件可以安全地仅由 root 读取。

```
beastie_enable=YES
beastie_env=”
ROLE_ID=6caaeac3-d8fa-a0e3-83ba-7d37750603c2
SECRET_ID=8dd54c92-fe54-0d6d-bee6-e433e815aaa1
SECRET_PATH=kv/beastie
VAULT_ADDR=http://localhost:8200/
“
```

脚本 `/usr/local/etc/rc.d/beastie` 会运行一个预命令，以 root 身份获取秘密，并将其注入子环境。

```
start_precmd=${name}_vault
beastie_vault() {
# 使用 Vault 中的 approle 进行身份验证
VAULT_TOKEN=$(vault write auth/approle/login role_id=”$ROLE_ID” \
secret_id=”$SECRET_ID” \
-format=json | jq -r '.auth.client_token')

# 从 Vault 中提取秘密
export BEASTIE_SECRET=$(vault kv get -field=data -format=json ${SECRET_PATH} | jq -r .)
}
```

### 代理

Vault 还提供了代理模式，可以为你处理大部分凭据管理工作，并支持简单配置文件的模板化。

## 关停 Vault

在通常情况下，Vault 会保持长达数月之久的开启状态，除非进行补丁和升级。在发生安全事件时，只需中止运行 Vault 守护程序的服务器，或者执行命令 `seal`。这会关停 Vault，并卸载主密钥。

```
$ vault operator seal
Success! Vault is sealed.
```

---

在过去二十年间， **DAVE COTTLEHUBER** 一直致力于领先于网络上的恶意者至少一步之遥。从 OpenBSD 2.8 到 9.3 共九年，在这段时间内他获得了 Ports 提交权限。他倾向于使用 jail，和晦涩的的函数式编程语言，这与他喜欢分布式系统和边缘异常锋利的电动工具有异曲同工之处。


- 职业牦牛牧人，自 2000 年以来剃 BSD 色的牦牛（**译者注：即一直在开发使用 FreeBSD，解决相关故障**）
- FreeBSD ports@ 提交者  
- Ansible DevOops 大师
- Elixir 开发者  
- 使用 RabbitMQ 和 Apache CouchDB 构建分布式系统  
- 喜欢越野滑雪，并演奏各种乐器的凯尔特民间音乐
