# 在 FreeBSD 上使用 ZFS 部署 Hadoop 教程

作者：**Benedict Reuschling**

Hadoop 是开源的分布式框架，使用 map-reduce 编程范式将大型计算任务分解为更小的片段。这些片段随后分发到集群中的所有节点（map 步骤），参与的 datanode 并行计算问题的部分。在 reduce 步骤中，收集这些部分结果并计算最终结果。结果存储在 Hadoop 分布式文件系统（HDFS）中。由称为 namenode 的主节点协调。Hadoop 由许多组件组成，可以在商用硬件上运行，无需大量资源。参与集群的节点越多，并且问题可以用 map-reduce 表示，性能就比在单节点上运行计算更好。Hadoop 主要用 Java 编写，旨在提供足够的冗余，允许节点故障，同时维持计算集群正常运行。围绕 Hadoop 发展出丰富的附加软件生态系统，这使得为大数据应用搭建 Hadoop 机器集群成为困难任务。

本文提供脚本教程，使用 Ansible playbook 从基础开始构建集群。手动安装和配置节点增多时很快变得繁琐，因此自动化安装为 BSD 系统管理员减轻了一些负担。

本教程在 ZFS 上构建 Hadoop，以利用该文件系统的强大特性，通过集成卷管理器增强 HDFS 的功能集。

## 需求

本教程使用三台 FreeBSD 11.2 机器，可以是物理机或虚拟机。其他较旧的 BSD 版本只要支持近期版本的 Java/OpenJDK，也应该能工作。使用 OpenZFS 即可，如果 OpenZFS 不可用，常规目录也可以。一台机器将作为主节点（称为 namenode），另外两台作为计算节点（Hadoop 术语中的 datanode）。它们需要能够在网络上相互连接。此外，必须有一套 Ansible 环境供 playbook 工作。这涉及包含三台机器的清单文件，以及目标机器上必需的软件（python 2.7 或更高版本），供 Ansible 向其发送命令。

注意，此 playbook 不使用 FreeBSD 的 Port/package Hadoop 默认路径。这样，在 Port 更新之前就可以使用更高版本的 Hadoop。默认的 FreeBSD 路径在需要时可以轻松替换。本教程中介绍的配置文件只包含使基本 Hadoop 设置运行所需的最少部分。FreeBSD Port/package 包含的示例配置文件有比最初需要更多的配置选项。然而，Port 是一旦集群搭建完成，扩展和学习 Hadoop 集群的绝佳资源。

不熟悉 Ansible 的读者应该能够抽象出设置步骤，要么用不同的配置管理系统（puppet、chef、saltstack）实现，要么手动执行步骤。

## 定义 Playbook 变量

为便于管理，使用独立的 `vars.yml` 文件保存所有变量。此文件集中存放安装集群所需的所有信息。例如，当需要使用更高版本的 Hadoop 时，只需更改变量 `hdp_ver`。

```ini
  java_home: "/usr/local/openjdk8"
  hdp: "hadoop"
  hdp_zpool_name: {{hdp}}pool
  hdp_ver: "2.9.0"
  hdp_destdir: "/usr/local/{{hdp}}{{hdp_ver}}"
  hdp_home: "/home/{{hdp}}"
  hdp_tmp_dir: "/{{hdp}}/tmp"
  hdp_name_dir: "/{{hdp}}/hdfs/namenode"
  hdp_data_dir: "/{{hdp}}/hdfs/datanode"
  hdp_mapred_dir: "/{{hdp}}/mapred.local.dir"
  hdp_namesec_dir: "/{{hdp}}/namesecondary"
  hdp_keyname: "my_hadoop_key"
  hdp_keytype: "ed25519"
  hdp_user_password: "{{vault_hdp_user_pw}}"
```

**文件：`vars.yml`**

第一行存储 Ports 安装的 OpenJDK 位置。为节省 playbook 中的输入，并替换 Hadoop 一词的常见出现，使用较短的变量 `hdp` 作为所有其余变量的前缀。zpool 名称（本教程中为 hadooppool）已经使用了我们为 Hadoop 名称定义的变量。如上所述，Hadoop 版本用于跟踪此集群所基于的版本。变量描述了 ZFS 数据集（或目录），供 Hadoop 用户在运行时使用。最后几行定义了 Hadoop 安全连接集群节点所需的 SSH 密钥。密码等机密信息也存储在 Ansible-vault 中供 Hadoop 用户使用。这样，playbook 可以与他人共享，而不暴露为该集群设置的密码。

要创建新 vault，使用 **ansible-vault(1)** 命令的 `create` 子命令，后跟加密 vault 文件应存储的路径。

```sh
  $ ansible-vault create vault.yml
```

创建 vault 后会提示创建密码短语以打开 vault，然后在 vault 文件中打开编辑器，即可在其中存储机密信息。如需了解如何生成 Ansible `user` 模块可理解的加密密码，请参阅（<https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-crypted-passwords-for-the-user-module>）。vault 中的行应如下所示，将 `<password>` 替换为密码哈希：

```ini
  vault_hdp_user_pw: "<yourpassword>"
```

**文件：`vault.yml`**

## Playbook 内容

playbook 本身分为几个部分，以帮助更好地理解每个部分中执行的操作。第一部分是 playbook 的开头，描述 playbook 的功能（`name`）、工作的主机（`hosts`），以及变量和 vault 的存储位置（`vars_files`）：

```ini
    #!/usr/local/bin/ansible-playbook
    - name: "Install a {{hdp}} {{hdp_ver}} multi node cluster"
       hosts: "{{host}}"

       vars_files:
       - vault.yml
       - vars.yml
```

第一行确保 playbook 可以像常规 shell 脚本一样运行，方法是使其可执行（`chmod +x`）。`name:` 描述此 playbook 的功能，并使用 `vars.yml` 中定义的变量。`hosts` 稍后在命令行中提供，以便更灵活地添加更多机器。或者，当集群有预定的主机数量时，也可以在 `hosts:` 行中输入它们。

接下来，定义 playbook 应执行的任务（注意不要使用制表符缩进，这是 YAML 语法）：

```ini
      tasks:
        - name: "Install required software for {{hdp}}"
          package:
            name: "{{item}}"
          with_items:
            - openjdk8
            - bash
            - gtar
```

第一个任务是从 FreeBSD 包安装 OpenJDK、为 Hadoop 用户的 shell 安装 `bash`、以及安装 `gtar` 来解压从 Hadoop 网站下载的源代码 tarball（稍后的解压步骤）。数据集（如果无法使用 ZFS，则为目录）在下一步中创建：

```ini
    - name: "Create ZFS datasets for the {{hdp}} user"
      zfs:
        name: "{{hdp_zpool_name}}{{item}}"
        state: present
        extra_zfs_properties:
          mountpoint: "{{item}}"
          recordsize: "1M"
          compression: "lz4"
        with_items:
          - "{{hdp_home}}"
          - "/opt"
          - "{{hdp_tmp_dir}}"
          - "{{hdp_name_dir}}"
          - "{{hdp_data_dir}}"
          - "{{hdp_namesec_dir}}"
          - "{{hdp_mapred_dir}}"
          - "{{hdp_zoo_dir}}"
```

每个数据集使用 LZ4 压缩，记录大小最大可达 1 兆字节。这对于提高压缩率很重要，因为 Hadoop 分布式文件系统（HDFS）默认使用 128MB 记录。挂载点路径在 `vars.yml` 文件中定义，稍后将在 Hadoop 专用配置文件中再次使用。

```ini
- name: "Create the {{hdp}} User"
   user:
     name: "{{hdp}}"
     comment: "{{hdp}} User"
     home: "{{hdp_localhome}}/{{hdp}}"
     shell: /usr/local/bin/bash
     createhome: yes
     password: "{{vault_hdp_user_pw}}"
```

Hadoop 进程都应在名为 Hadoop 的独立用户账户下启动和运行。此任务将在系统中创建该指定用户。结果在密码数据库中如下所示（用户 ID 可能不同）：

```sh
$ grep hadoop /etc/passwd
hadoop:*:31707:31707:hadoop User:/home/hadoop:/usr/local/bin/bash
```

接下来，需要分发 SSH 密钥，以便 Hadoop 用户能够登录到每台集群机器，而无需密码。使用 Ansible 的查找功能读取运行 playbook 的机器上之前生成的 SSH 密钥（建议使用 `ssh-keygen` 为 Hadoop 生成这种单独的密钥）。SSH 密钥不能有密码短语，因为 Hadoop 进程执行登录时无任何用户交互。此任务会将 SSH 公钥添加到 **/home/hadoop** 中的 `authorized_keys` 文件。

```ini
- name: "Add SSH key for {{hdp}} User to authorized_keys file"
  authorized_key:
    user: "{{hdp}}"
    key: "{{ lookup('file', './{{hdp_keyname}}.pub') }}"
```

公钥和私钥必须放在 Hadoop 用户主目录下的 `.ssh` 中。由于已为密钥定义了变量，因此很容易提供公钥（`.pub` 扩展名）和私钥（无扩展名），而无需在此任务中拼出其真实名称。此外，通过设置适当的模式和所有权保护密钥，以便除 Hadoop 外无人能访问它。

```ini
- name: "Copy public and private key to {{hdp}}'s .ssh directory"
  copy:
     src: "./{{item.name}}"
     dest: "{{hdp_localhome}}/{{hdp}}/.ssh/{{item.type}}"
     owner: "{{hdp}}"
     group: "{{hdp}}"
     mode: 0600
  with_items:
     - { type: "id_{{hdp_keytype}}", name: "{{hdp_keyname}}" }
     - { type: "id_{{hdp_keytype}}.pub", name: "{{hdp_keyname}}.pub" }
```

将 Hadoop 用户添加到 **/etc/ssh/sshd_config** 中的 `AllowUsers` 行，允许其访问每台机器。正则表达式将确保 `AllowUsers` 行中的任何先前条目都保留，并将 Hadoop 用户添加到现有用户列表的末尾。

```ini
- name: "Add {{hdp}} to AllowedUsers line in /etc/ssh/sshd_config"
  replace:
    backup: no
    dest: /etc/ssh/sshd_config
    regexp: '^(AllowUsers(?!.*\b{{ hdp }}\b).*)$'
    replace: '\1 {{ hdp }}'
    validate: 'sshd -T -f %s'
```

SSH 在此之后显式重启，因为 playbook 很快就要使用 Hadoop SSH 登录。注意，此处不能使用 Ansible 处理程序，因为它会执行得太晚（在 playbook 所有任务执行完毕时）。

```ini
     - name: Restart SSH to make changes to AllowUsers take effect
        service:
          name: sshd
          state: restarted
```

下一个任务涉及从节点收集 SSH 密钥信息，以便 Hadoop 不必在建立首次连接时确认目标系统的主机密钥。我们需要能够本地 ssh 到主节点本身，因此必须将 0.0.0.0、localhost、每台机器的 IP 地址、主 IP 地址（这样客户端节点就知道它，无需额外任务）添加到 `.ssh/known_hosts`。这就是 `ssh-keyscan` 在此任务步骤中所做的。变量 `{{workers}}` 稍后将在命令行中提供，包含所有将作为 datanode 运行 map-reduce 作业的机器。（当然，当机器数量静态且不变时，这些也可以放在 `vars.yml` 中。）

```sh
      - name: "Scan SSH Keys"
        shell: ssh-keyscan 0.0.0.0 localhost \
      "{{hostvars[inventory_hostname]'ansible_default_ipv4']['address']}} {{master}}" >>
      "{{hdp_home}}/.ssh/known_hosts"

      - name: "Scan worker SSH Keys one by one"
        shell: "ssh-keyscan {{item}} {{master}} >> {{hdp_home}}/.ssh/known_hosts"
        with_items: "{{workers}}"
```

为了正常运行，Hadoop 需要设置一些环境变量。这些包括 `JAVA_HOME`、`HADOOP_HOME` 以及 Hadoop 用户使 Hadoop 集群与 Java 协同工作所需的其他变量。环境变量存储在 `.bashrc` 文件中，该文件从本地 Ansible 控制机部署到远程系统上的 Hadoop 主目录。

`.bashrc` 文件本身将作为模板提供。Ansible 中的这项强大功能使得存储填充了 Ansible 变量的配置文件成为可能（利用 Jinja2 语法）。部署时，不只是简单的复制操作。传输到远程机器时，变量被替换为其实际值。此时，`{{hdp_destdir}}` 被替换为 **/usr/local/hadoop2.9.0**。

```ini
     - name: "Copy BashRC over to {{hdp_localhome}}/{{hdp}}/.bashrc"
        template:
          src: "./hadoop.bashrc.template"
          dest: "{{hdp_home}}/.bashrc"
          owner: "{{hdp}}"
          group: "{{hdp}}"
```

模板文件本身需要在文件末尾添加以下内容：

```sh
     export JAVA_HOME={{java_home}}
     export HADOOP_HOME={{hdp_destdir}}
     export HADOOP_INSTALL={{hdp_destdir}}
     export PATH=$PATH:$HADOOP_INSTALL/bin
     export HADOOP_PREFIX=/opt/{{hdp}}
     export PATH=$PATH:$HADOOP_PREFIX/bin
     export PATH=$PATH:$HADOOP_HOME/bin
     export PATH=$PATH:$JAVA_HOME/bin
     export PATH=$PATH:$HADOOP_HOME/sbin
     export HADOOP_CLASSPATH=$JAVA_HOME/lib/tools.jar
     export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"
```

想要运行 map-reduce 作业的 Hadoop 以外的任何用户也需要设置这些环境变量，因此考虑也将该文件复制到 **/etc/skel/.bashrc**。

## 部署 Hadoop 配置文件

是时候部署构成 Hadoop 发行版的文件了。它基本上是可以解压到任何目录的 tarball，因为它主要包含 JAR 和配置文件。这样，Hadoop 可以轻松作为整体复制，因为该目录包含运行 Hadoop 所需的一切。这些文件可以从 Hadoop 网页下载（<http://hadoop.apache.org/releases.html>）。有许多支持的版本不断发展，意味着将有新版本定期发布。该页面底部列出了每个版本修复了多少 bug。与听起来相反，无需跟上 Hadoop 项目设定的节奏，如果需要，旧版本的 Hadoop 可以运行多年。本文选择了相当新的版本（2.9.0）。请确保选择二进制发行版来下载，因为从源码构建 Hadoop 需要额外时间。文件名为 `hadoop-2.9.0.tar.gz`，名称将在 playbook 中使用 `vars.yml` 文件中的定义再次构造。Ansible 的 `unarchive` 模块负责在远程机器上将 tarball 解压到 `{{hdp_destdir}}`，即 **/usr/local/hadoop2.9.0**。将版本包含在目录/数据集名称中，便于并排安装不同版本的 Hadoop 测试。

```ini
- name: "Unpack Hadoop {{hdp_ver}}"
  unarchive:
    src: "./{{hdp}}-{{hdp_ver}}.tar.gz"
    dest: "{{hdp_destdir}}"
    remote_src: yes
    owner: "{{hdp}}"
    group: "{{hdp}}"
```

## Hadoop 核心配置

是时候编辑随 Hadoop 附带的一组配置文件了。初学者理解哪个文件需要更改可能会不知所措。不幸的是，Hadoop 网站没有很好地解释完全分布式 Hadoop 集群需要更改哪些文件。根据我们的经验，Hadoop 网站上的文档不完整，即使严格遵循，结果也不是能正常工作的 Hadoop 集群。经过大量试错，作者确定了在 FreeBSD 上创建功能完整的 map-reduce 集群（底层为 HDFS）所需的重要文件。

在其核心，4 个文件构成此集群的站点特定配置部分，命名为 `*-site.xml`。需要更改它们，它们都位于 Hadoop 发行版的配置目录中。在本教程中，该路径为 **/usr/local/hadoop2.9.0/etc/hadoop/**，包含 `core-site.xml`、`yarn-site.xml`、`hdfs-site.xml` 和 `mapred-site.xml`。

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- 将站点特定的属性覆盖放在此文件中。 -->

<configuration>
 <property>
  <name>fs.defaultFS</name>
  <value>hdfs://{{master}}:9000</value>
 </property>
 <property>
      <name>io.file.buffer.size</name>
      <value>131072</value>
     </property>
     <property>
      <name>hadoop.tmp.dir</name>
      <value>{{hdp_tmp_dir}}</value>
      <description>其他临时目录的基础。</description>
     </property>
    </configuration>
```

**core-site.xml 模板**

```xml
    <?xml version="1.0"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
    <!-- 站点特定的 YARN 配置属性 -->
      <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
      </property>
      <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
      </property>
      <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>{{master}}</value>
      </property>
      <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>{{master}}:8025</value>
      </property>
      <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>{{master}}:8030</value>
      </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>{{master}}:8050</value>
      </property>
      <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>0.0.0.0:8088</value>
      </property>
    </configuration>
```

**yarn-site.xml 模板**

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
 <property>
  <name>dfs.namenode.name.dir</name>
  <value>file://{{hdp_name_dir}}</value>
 </property>
 <property>
  <name>dfs.datanode.data.dir</name>
  <value>file://{{hdp_data_dir}}</value>
 </property>
 <property>
  <name>dfs.replication</name>
  <value>2</value>
 </property>
 <property>
  <name>dfs.checkpoint.dir</name>
  <value>file://{{hdp_freebsd_namesec_dir}}</value>
  <final>true</final>
 </property>
</configuration>
```

**hdfs-site.xml 模板**

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>mapreduce.job.tracker</name>
    <value>{{inventory_hostname}}:8021</value>
  </property>
  <property>
    <name>mapred.job.tracker</name>
    <value>{{inventory_hostname}}:54311</value>
  </property>
  <property>
    <name>mapred.local.dir</name>
    <value>{{hdp_mapred_dir}}</value>
  </property>
  <property>
    <name>mapred.system.dir</name>
    <value>/mapredsystemdir</value>
  </property>
  <property>
    <name>mapred.tasktracker.map.tasks.maximum</name>
    <value>2</value>
  </property>
  <property>
    <name>mapred.tasktracker.reduce.tasks.maximum</name>
    <value>2</value>
  </property>
  <property>
    <name>mapred.child.java.opts</name>
    <value>-Xmx200m</value>
  </property>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>{{inventory_hostname}}:10020</value>
  </property>
</configuration>
```

**mapred-site.xml 模板**

playbook 中的以下任务负责把它们放在正确的位置，并用 `vars.yml` 定义文件中的变量正确替换。（特别是在这一点上，拥有集中的 Ansible 变量文件变得非常宝贵，因为这些文件中的拼写错误和错误会给调试已经复杂的分布式系统（如 Hadoop）带来很多头痛。）

```ini
     - name: "Templating *-site.xml files for the node"
       template:
         src: "./Hadoop275/freebsd/{{item}}.j2"
         dest: "{{hdp_destdir}}/etc/{{hdp}}/{{item}}"
         owner: "{{hdp}}"
         group: "{{hdp}}"
       with_items:
         - core-site.xml
         - hdfs-site.xml
         - yarn-site.xml
         - mapred-site.xml
```

名为 `slaves`（较新版本重命名为 `workers`）的文件包含应作为 datanode 的主机名。定义为主节点的机器也可以参与并处理 map-reduce 作业，因此文件中有 localhost。此任务将定义为 Ansible playbook 参数的 worker 添加到该文件中：

```ini
     - name: "Create and populate the slaves file"
       lineinfile:
         dest: "{{hdp_destdir}}/etc/{{hdp}}/slaves"
         owner: "{{hdp}}"
         group: "{{hdp}}"
         line: "{{item}}"
       with_items: "{{ workers }}"
```

既然对安装做了一堆文件更改，我们需要确保文件仍然归 Hadoop 所有，而不是运行 Ansible 脚本的用户。最后一个任务递归地设置所有权和组到 Hadoop 用户，作用于 playbook 迄今为止触及的文件和目录。

```ini
     - name: "Give ownership to {{hdp}}"
       file:
         path: "{{item}}"
         owner: "{{hdp}}"
         group: "{{hdp}}"
         recurse: yes
       with_items:
         - "{{hdp_home}}"
         - "{{hdp_destdir}}"
         - "/{{hdp}}"
```

这就是完整的 playbook，名为 `freebsd_hadoop2.9.0.yml`，可以用以下命令行执行它。

```sh
     $ ./freebsd_hadoop2.9.0.yml -Kbe 'host=namenode:datanode1:datanode2
     master=namenode' -e '{"workers":{"datanode1","datanode2"}}' --vault-id @prompt
```

主机 namenode、datanode1 和 datanode2 都需要在 Ansible 的清单文件中定义。参数 `--vault-id @prompt` 将询问创建 vault 时定义的 vault 密码。

## 启动 Hadoop 和第一个 Map-reduce 作业

运行 playbook 后，部署中没有错误，是时候登录 namenode 主机并切换到 Hadoop 用户（使用设置的密码）。第一个测试是验证该用户可以登录到每个 datanode1 和 datanode2，而不会被提示确认主机密钥或提供密码。如果登录时没有出现这些，那么可以启动 Hadoop 服务。第一步是使用 `hdfs namenode` 命令格式化分布式文件系统（Hadoop 的路径在 `.bashrc` 文件中，因此省略了 `hdfs` 可执行文件的完整路径）：

```sh
hadoop@namenode$ hdfs namenode -format
```

几条初始化消息会滚动过去，但最后应该没有错误。小心运行此命令第二次。每次都会生成唯一的 ID，用于识别该 HDFS 并与其他区分开来。不幸的是，格式化只在主节点上完成，而不是在所有其他集群节点上。因此，第二次运行它会混淆 datanode，因为它们仍然保留旧的 ID。解决方案是清除 `{{hdp_data_dir}}` 和 `{{hdp_tmp}}` 中定义的目录中的任何先前内容，在 datanode 和 namenode 上都要做。

接下来，必须按顺序启动构成 Hadoop 系统的所有服务。以下命令将负责此操作：

```sh
hadoop@namenode$ start-dfs.sh && start-yarn.sh && mr-jobhistory-daemon.sh start
historyserver
```

要确保所有进程都成功启动，运行 `jps` 验证以下服务已在 namenode 上启动：NameNode、ResourceManager、JobHistoryServer 和 SecondaryNameNode。

datanode 必须在 `jps` 输出中有这些进程：NodeManager 和 DataNode。（在 map-reduce 作业期间运行 `jps` 意味着将在 datanode 上生成更多进程，这些进程形成节点使用 YARN 框架处理的工作单元。）

集群已准备好运行其第一个 map-reduce 作业。Hadoop 提供示例作业，让用户无需先编写 Java 程序并将其打包到 jar 文件中作为作业执行即可了解框架。其中一个示例文件将尝试使用蒙特卡洛模拟计算 pi 值。以下 shell 脚本可以做到这一点：

```sh
#!/bin/sh
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
hadoop jar /usr/local/hadoop2.9.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-
2.9.0.jar pi 16 1000000
```

执行 shell 脚本将生成映射器来计算蒙特卡洛模拟的子集。根据选择的映射器数量（本例中为 16），结果的准确性会有所不同。可以使用指向 URL `http://<the.namenode.ip.address>:8088` 的浏览器监视作业。浏览 `http://<the.namenode.ip.address>:50070` 会显示整体集群状态以及 HDFS 的文件系统浏览器和日志（需要手动刷新以获取两个页面的更新信息）。

另一个有趣的示例是 random-text-writer，它在 HDFS 中跨节点创建一堆文件。使用时间戳使此命令可以连续多次运行：

```sh
#!/bin/sh
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
timestamp="`date +%Y%m%d%H%M`"
hadoop jar /usr/local/hadoop-2.9.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-
2.9.0.jar randomtextwriter /random-text_$timestamp
```

两个作业都无误运行后，Hadoop 集群即可接受更复杂的 map-reduce 作业。这作为学习练习留给读者，互联网上有许多关于如何编写 map-reduce 作业的教程。

本教程最后展示两个作业完成时实现的 ZFS 压缩率（结果可能有所不同）：

```sh
hadoop@namenode$ zfs get refcompressratio hadooppool/hadoop/hdfs/namenode
NAME                              PROPERTY             VALUE        SOURCE
hadooppool/hadoop/hdfs/namenode   refcompressratio     26.28x       -
```

datanode 也从 random-text-writer 示例中获得了相当不错的压缩率：

```sh
NAME                              PROPERTY             VALUE        SOURCE
hadooppool/hadoop/hdfs/datanode   refcompressratio     2.35x        -
```

这表明在 FreeBSD 上运行 Hadoop 有好处。OpenZFS 能够为存储在 HDFS 中的数据添加额外保护，并使得在底层磁盘上存储更多数据成为可能。在大数据世界中，这是巨大的优势。

---

**BENEDICT REUSCHLING** 于 2009 年加入 FreeBSD 项目。2010 年获得完整文档提交权限后，他积极指导他人成为 FreeBSD 提交者。他是 BSD 认证小组的监考人，并于 2015 年加入 FreeBSD 基金会，现任副总裁。Benedict 拥有计算机科学理学硕士学位，在德国达姆施塔特应用技术大学教授面向软件开发者的 UNIX 课程。
