# FreeBSD 中对 SYN 段的处理

- [The Handling of SYN Segments in FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/downstreams/the-handling-of-syn-segments-in-freebsd)
- 作者：Randall Stewart & Michael Tüxen


## TCP 连接设置

传输控制协议（TCP）是一种面向连接的传输协议，提供了可靠的双向字节流服务。TCP 连接设置需要交换三个 TCP 段，这被称为三次握手。发起 TCP 连接并发送第一个 TCP 段（SYN 段）的 TCP 端点称为客户端。等待接收第一个 TCP 段的端点称为服务器，服务器响应接收到的 SYN 段并发送一个 SYN ACK 段。当客户端接收到这个 SYN ACK 段时，它通过发送 ACK 段完成握手。

TCP 握手不仅用于同步两个端点之间的状态，包括提供可靠性的初始序列号，还用于通过 TCP 选项协商使用 TCP 扩展。如今的互联网中，在握手过程中（SYN 和 SYN ACK 段中包含）最广泛部署的 TCP 选项是：

1. 最大报文段长度（MSS）选项  
    MSS 选项包含一个 16 位数字（介于 0 和 65535 之间），它表示发送此选项的端点愿意在单个 TCP 段中接收的最大负载字节数。假设此数字不使用 IP 层和 TCP 层的选项。如果使用了这样的选项，则该数字必须减去选项的大小。这有助于 TCP 发送方避免发送需要在 IP 层进行分段的 TCP 段。

2. SACK-允许选项  
    此选项宣布发送方能够处理选择性确认（SACK）选项。这在发生数据包丢失时有助于提高性能。

3. TCP 窗口缩放选项  
    此选项包含一个介于 0 到 14 之间的自然数。如果双方都发送了此选项，则启用接收窗口缩放。这允许使用比 TCP 头格式允许的更大的接收窗口，因为接收窗口在 TCP 头中被限制为 16 位（因此为 65535 字节）。这避免了 TCP 中接收窗口字段的大小限制了 TCP 连接的吞吐量的问题。

4. TCP 时间戳选项  
    此选项包含两个 32 位数字，通常以毫秒级别编码一些时间信息。它用于提高 TCP 性能。

TCP 使用状态事件机来指定。最初，一个端点处于 CLOSED 状态。当端点愿意接受 TCP 连接（在服务器端），TCP 端点进入 LISTEN 状态。当接收到来自客户端的 SYN 段并回复 SYN ACK 段时，端点进入 SYN RECEIVED 状态。一旦 TCP 端点接收到客户端发送的 ACK 段，TCP 端点进入 ESTABLISHED 状态。可以使用 `netstat` 或 `sockstat` 命令行工具来观察这些状态。

应用程序编程接口（API）用于控制 TCP 端点的是套接字 API。程序通常使用监听套接字，告诉 TCP 实现可以在此端点上接受 TCP 连接，并且对于每个已接受的 TCP 连接，程序为每个 TCP 连接使用一个单独的套接字。应用程序可以设置监听套接字的参数，并且大多数情况下这些设置会被接受的套接字继承。本文重点讨论服务器端的 TCP 连接设置。需要注意的是，这一功能适用于所有 TCP 栈（默认的、RACK、BBR 等）。

## SYN 洪水攻击

当 TCP 最初实现时，每当接收到一个 SYN 段时，都会为 LISTEN 状态的 TCP 端点创建一个新的 TCP 端点。这需要分配内存，并且导致在 SYN RECEIVED 状态下创建一个新的 TCP 端点。所有必要的信息，包括与接收到的 SYN 段中的 TCP 选项相关的信息，都存储在 TCP 端点中。这个过程没有验证提供的信息、IP 地址和 TCP 端口号。

这会让攻击者向服务器发送大量的 SYN 段，而服务器会不断分配 TCP 端点，直到资源耗尽。因此，攻击者可以进行拒绝服务攻击，因为只要服务器没有更多资源，它将无法再接受来自有效客户端的 SYN 段。攻击者只需要发送 SYN 段，特别是，攻击者不会响应接收到的任何 SYN ACK 段。攻击者甚至可以使用伪造的 IP 地址（即攻击者不拥有的 IP 地址）。

SYN 洪水攻击的目的是使接收方耗尽资源，因此无法提供其预期的服务。在 FreeBSD 中，TCP 栈实现了两种缓解 SYN 洪水攻击的机制：

1. **减少内存分配**：当 TCP 端点从 CLOSED 状态转移到 SYN RECEIVED 状态时，通过使用 SYN 缓存减少分配的内存量，如下一节所述。

2. **不分配内存**：在处理传入的 SYN 段时，不分配任何内存。通过使用 SYN cookie 来实现，如下节所述。

## SYN 缓存

SYN 缓存的初始实现是在 2001 年 11 月加入到 FreeBSD 源代码树中的。它通过不分配完整的 TCP 端点，而是分配一个 TCP SYN 缓存条目（`struct syncache`，在 `sys/netinet/tcp_syncache.h` 中定义），来减少 SYN RECEIVED 状态下的 TCP 端点的内存开销。一个 TCP SYN 缓存条目比 TCP 端点小，仅允许存储 SYN RECEIVED 状态下相关的信息。此信息包括：

- 本地和远程 IP 地址以及 TCP 端口号。
- 用于执行基于定时器的 SYN ACK 段重传的信息。
- 本地和远程 TCP 初始序列号。
- 同步段中对方在 MSS 选项中报告的 MSS 值。
- 在 SYN 和 SYN ACK 段中交换的本地和远程窗口缩放位移值。
- 是否协商了窗口缩放、时间戳和 SACK 支持。
- 精确的 ECN 状态。
- 额外的 IP 层信息。

当接收到一个针对监听端点的 SYN 段时，会分配一个 SYN 缓存条目，并将相关信息存储其中，同时发送一个 SYN ACK 段作为响应。如果禁用 SYN cookie 且发生桶溢出，则会丢弃桶中最旧的 SYN 缓存条目。如果收到相应的 ACK 段，则使用 SYN 缓存条目中的数据创建一个完整的 TCP 端点，然后释放该 SYN 缓存条目。SYN 缓存还确保在未及时收到相应 ACK 段的情况下重传 SYN ACK 段。

`sysctl` 变量 `net.inet.tcp.syncookies`（默认值为 `1`）控制着是否使用 SYN cookie，结合 SYN 缓存来覆盖无法分配或查找 SYN 缓存条目的情况。

SYN 缓存是特定于 vnet 的，并且以哈希表形式组织。桶的数量由加载器可调参数 `net.inet.tcp.syncache.hashsize`（默认值为 512）控制。每个哈希桶中的最大 SYN 缓存条目数由加载器可调参数 `net.inet.tcp.syncache.bucketlimit`（默认值为 30）控制。还有一个总的 SYN 缓存条目限制，由加载器可调参数 `net.inet.tcp.syncache.cachelimit`（默认值为 15360，即 512 * 30）控制。当前使用的 SYN 缓存条目数通过只读的 `sysctl` 变量 `net.inet.tcp.syncache.count` 报告。

还有一些与 SYN 缓存相关的其他 `sysctl` 变量。这些变量包括：

- `net.inet.tcp.syncache.rst_on_sock_fail`：控制在无法成功创建套接字时是否发送 RST 段（默认值为 1）。
- `net.inet.tcp.syncache.rexmtlimit`：SYN ACK 段的最大重传次数（默认值为 3）。
- `net.inet.tcp.syncache.see_other`：控制 SYN 缓存条目的可见性（默认值为 0）。

TCP SYN 缓存能让服务器端在最小化内存资源的情况下执行完整的握手。对于 SYN RECEIVED 状态下的 TCP 端点，使用完整的 TCP 端点与使用 SYN 缓存没有功能上的区别。即使是像 `netstat` 或 `sockstat` 这样的工具也会报告 SYN 缓存中的条目。

支持额外的 TCP 选项也不是问题，因为 TCP SYN 缓存条目可以扩展。

`sysctl` 变量 `net.inet.tcp.syncookies_only`（默认值为 0）可以用来禁用 SYN 缓存的使用。在这种情况下，只会使用下一节描述的 SYN cookie。

## SYN Cookie

为了进一步防范 SYN 洪水攻击，SYN 缓存的实现于 2001 年 12 月进行了增强。在处理接收到的 SYN 段时，服务器不再分配较少的内存，而是将相关信息存储在一个所谓的 SYN cookie 中，并在 SYN ACK 段中发送给客户端。然后，客户端需要在 ACK 段中反射该 SYN cookie。当服务器处理 ACK 段时，所有相关信息都包含在 SYN cookie 和 ACK 段中。因此，服务器可以基于这些信息创建一个 ESTABLISHED 状态的 TCP 端点。通过这种方式，SYN 洪水攻击不会导致内存资源的耗尽。然而，SYN cookie 的生成不能消耗过多的 CPU 资源。如果生成 SYN cookie 的过程过于耗费 CPU，可能会导致另一种拒绝服务攻击：这次攻击的目标不是内存资源，而是 CPU 资源。

在 TCP 头中，唯一可以由服务器任意选择并被客户端反射的字段是服务器的初始序列号。该字段是一个 32 位整数，因此用作 SYN cookie。

在 FreeBSD 中，这 32 位被分割为一个 24 位的消息认证码（MAC）和 8 位，具体如下所示：

- 3 位用于编码 8 个 MSS 值中的一个：216、536、1200、1360、1400、1440、1452、1460。如果客户端在 MSS 选项中发送的值不在这个列表中，则使用不超过该值的最大值。
- 3 位用于编码对端是否不支持窗口缩放或使用以下 7 个值之一：0、1、2、4、6、7、8。如果客户端发送的值不在此列表中，则使用不超过该值的最大值。
- 1 位用于编码客户端是否发送了 SACK-permitted 选项。
- 1 位用于选择两个密钥中的一个。

MAC 使用一个秘密密钥，该密钥每 15 秒更新一次。当前和上一个秘密密钥都会被保留下来，并根据 SYN cookie 中的位来选择使用哪个密钥。

MAC 的计算包括本地和远程 IP 地址、客户端的初始序列号、上述 8 位和一些内部信息。从 MAC 中生成 24 位，并与上述 8 位结合，构造出 SYN cookie。

当服务器接收到三次握手的 ACK 段时，会验证 MAC。如果验证成功，服务器将根据 SYN cookie 中的信息创建 TCP 端点，该信息提供了 MSS 选项的近似值、窗口位移的近似值以及客户端是否声明支持 SACK 扩展。所有其他相关信息必须从 ACK 段中恢复。这些恢复的信息包括本地和远程 IP 地址和端口号、本地和远程初始序列号、是否使用 TCP 时间戳选项，以及如果使用，当前的时间戳参数。

## SYN 缓存与 SYN Cookies 的比较

与 SYN 缓存相比，SYN cookies 的优势非常明显：接收到新的 SYN ACK 段时无需进行内存分配。然而，使用 SYN cookies 也有其缺点：

- MSS 被 8 个值近似，所有值都小于或等于 1460。因此，不支持 IPv4 中大于 1500 字节的 MTU。
- 用于窗口缩放的位移被 7 个值近似，所有值都小于或等于 8。这意味着不支持大于 8 的窗口位移，因此连接的窗口大小会更小。
- 不支持除了当前广泛部署的 TCP 选项之外的其他 TCP 选项。这使得支持用于协商新 TCP 特性的新的 TCP 选项变得困难。
- 如果 SYN ACK 段丢失，则不会重新传输 SYN ACK 段，发起连接的端点必须重试发送其 SYN 段。
- 不可见 SYN RECEIVED 状态下的 TCP 端点。
- 不支持 IP 层信息。

使用 SYN 缓存没有这些限制，并且是透明的，但需要为 SYN RECEIVED 状态下的每个 TCP 端点分配内存。

## SYN 缓存与 SYN Cookies 的联合使用

仅使用 SYN cookies 相比使用 SYN 缓存能更好地缓解 SYN 洪水攻击，但也带来了一些功能限制。因此，FreeBSD 的默认配置启用了 SYN 缓存，并与 SYN cookies 结合使用。这意味着，当接收到一个 SYN 段时，系统会生成一个 SYN 缓存条目，而发送的 SYN ACK 段则包含一个 SYN cookie。如果 SYN 缓存的一个桶溢出，系统会认为这是由于正在进行的 SYN 洪水攻击导致的，因此暂停使用 SYN 缓存。在此期间，仅使用 SYN cookies。

这一额外功能于 2019 年 9 月引入，在正常操作期间提供 SYN 缓存的优势，但在 SYN 洪水攻击发生时也能提供 SYN cookies 的改进保护。

---

**RANDALL STEWART**（[rrs@freebsd.org](mailto:rrs@freebsd.org)）是一位操作系统开发人员，拥有 40 余年的经验，并自 2006 年起成为 FreeBSD 开发者。他专注于传输协议，包括 TCP 和 SCTP，但也涉足操作系统的其他领域。目前，他是一名独立顾问。

**MICHAEL TÜXEN**（[tuexen@freebsd.org](https://freebsdfoundation.org/our-work/journal/browser-based-edition/downstreams/the-handling-of-syn-segments-in-freebsd/)）是明斯特应用技术大学的教授，同时也担任 Netflix 的兼职承包商，自 2009 年起成为 FreeBSD 源代码提交者。他的研究重点是传输协议，如 SCTP 和 TCP，以及它们在 IETF 的标准化和在 FreeBSD 中的实现。
