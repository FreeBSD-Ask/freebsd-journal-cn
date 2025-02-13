# 将 OpenBSD 的 pf syncookie 代码移植到 FreeBSD 的 pf

- 原文链接：[Porting OpenBSD’s pf syncookie Code to FreeBSD’s pf](https://freebsdfoundation.org/wp-content/uploads/2022/03/Porting-OpenBSDs-pf-syncookie-Code-to-FreeBSDs-pf.pdf)
- 作者：**KRISTOF PROVOST**

互联网可以是一个无序的地方，我们的在线服务可能会受到各种方式的攻击。我们运行的防火墙是我们保护系统的一部分，但事实证明它们也可能成为攻击的渠道。

一种攻击服务的方式是通过伪装成打开连接来耗尽系统资源。这通常被称为 SYN 洪水攻击，这是一种相当隐蔽的攻击。让我们先回顾一下 TCP 连接的工作原理。打开一个 TCP 连接是一个三步过程：

1. **SYN**：客户端发送一个 SYN 数据包，表示它希望打开连接。它在 SYN 数据包中设置一个初始的序列号。
2. **SYN+ACK**：服务器回应，确认客户端的序列号，并设置自己的序列号。
3. **ACK**：客户端确认连接已经打开，并确认服务器的序列号。

当服务器收到初始的 SYN 时，它会设置内部数据结构以支持新的连接。这需要 CPU 时间和内存。

这意味着恶意客户端可以生成 SYN 数据包（这些数据包很小，且容易生成），消耗服务器的可用内存和 CPU 资源。更糟糕的是，攻击者只需要发送一个 SYN 数据包。攻击者无需接收服务器的 SYN+ACK 回复。这意味着源 IP 地址可以伪造，使这些攻击难以过滤掉。它们还可以针对服务的主要 TCP 端口（例如，Web 服务器的 443 端口），使攻击与真实客户端请求难以区分。

## SYN cookies

1996 年，Daniel J. Bernstein 和 Eric Schenk 提出了抵御此类攻击的方法，称为 SYN cookies。简而言之，SYN cookies 确保服务器在接收到 SYN 数据包时不会为新连接分配任何内存，从而避免内存耗尽。

我们仍然生成一个 SYN+ACK 回复，但在客户端用 ACK 回复我们的 SYN+ACK 之前，服务器不会创建服务器端的状态。这确保客户端确实存在（即源 IP 地址没有被伪造）。

这个方法的明显问题是，我们仍然需要在 SYN 数据包到达时通常会保存的信息。这些信息包括最大报文段大小（MSS）和窗口缩放（WSCALE）。虽然这些选项是... 可选的，但它们对 TCP 性能很重要，我们不希望拒绝它们。

此外，我们还需要某种方法来确保客户端 ACK 中的确认号与我们在 SYN+ACK 消息中使用的序列号匹配。如果我们不进行检查，恶意客户端只需发送一个 SYN，稍等片刻，然后盲目地发送一个 ACK，也就是说，无需实际接收服务器的 SYN+ACK，只需使用一个随机的确认号。

那么，我们如何做到这一点呢？我们通过将所有选项编码到 SYN+ACK 数据包的序列号中来实现这一点。

在 pf 实现中，这是通过 `pf_syncookie_generate()` 函数完成的：

```c
uint32_t
pf_syncookie_generate(struct mbuf *m, int off, struct pf_pdesc *pd,
    uint16_t mss)
{
    uint8_t i, wscale;
    uint32_t iss, hash;
    union pf_syncookie cookie;
    PF_RULES_RASSERT();
    cookie.cookie = 0;

    /* 映射 MSS（最大报文段大小） */
    for (i = nitems(pf_syncookie_msstab) - 1;
        pf_syncookie_msstab[i] > mss && i > 0; i--)
        /* 无操作 */;
    cookie.flags.mss_idx = i;

    /* 映射 WSCALE（窗口缩放因子） */
    wscale = pf_get_wscale(m, off, pd->hdr.tcp.th_off, pd->af);
    for (i = nitems(pf_syncookie_wstab) - 1;
        pf_syncookie_wstab[i] > wscale && i > 0; i--)
        /* 无操作 */;
    cookie.flags.wscale_idx = i;
    
    cookie.flags.sack_ok = 0; /* XXX */

    cookie.flags.oddeven = V_pf_syncookie_status.oddeven;

    /* 计算哈希值 */
    hash = pf_syncookie_mac(pd, cookie, ntohl(pd->hdr.tcp.th_seq));

    /*
     * 将标志值放入哈希中，并对其进行异或操作，以获得更好的 ISS（初始序列号）数值变异。
     * 这不会增强加密强度，目的是防止 8 个 cookie 位直接显示在网络上。
     */
    iss = hash & ~0xff;
    iss |= cookie.cookie ^ (hash >> 24);
    
    return (iss);
}
```

目光敏锐的读者会注意到，对于 MSS 和 WSCALE，我们并没有真正编码正确的值，而是通过查找表找到最接近的匹配值。这减少了编码信息所需的位数，但仍然能够很好地近似真实值。稍微小一点的最大报文段大小（MSS）或窗口缩放因子（WSCALE）会稍微影响性能，但不会产生显著的影响。所选择的值是根据最常用的 MSS 或 WSCALE 值来表示的，因此对于大多数客户端，几乎没有性能损失。

这些信息被编码成一个经过认证的哈希值。也就是说，为了重建哈希，你需要输入信息（如 MSS、WSCALE 等）和一个秘密密钥。换句话说，攻击者无法预测哈希的结果，因此也无法预测服务器将选择的序列号。这一过程由 `pf_syncookie_mac()` 函数处理：

```c
uint32_t
pf_syncookie_mac(struct pf_pdesc *pd, union pf_syncookie cookie, uint32_t seq)
{
    SIPHASH_CTX ctx;
    uint32_t siphash[2];
    PF_RULES_RASSERT();
    MPASS(pd->proto == IPPROTO_TCP);

    SipHash24_Init(&ctx);
    SipHash_SetKey(&ctx, V_pf_syncookie_status.key[cookie.flags.oddeven]);

    switch (pd->af) {
        case AF_INET:
            SipHash_Update(&ctx, pd->src, sizeof(pd->src->v4));
            SipHash_Update(&ctx, pd->dst, sizeof(pd->dst->v4));
            break;
        case AF_INET6:
            SipHash_Update(&ctx, pd->src, sizeof(pd->src->v6));
            SipHash_Update(&ctx, pd->dst, sizeof(pd->dst->v6));
            break;
        default:
            panic("unknown address family");
    }

    SipHash_Update(&ctx, pd->sport, sizeof(*pd->sport));
    SipHash_Update(&ctx, pd->dport, sizeof(*pd->dport));
    SipHash_Update(&ctx, &seq, sizeof(seq));
    SipHash_Update(&ctx, &cookie, sizeof(cookie));
    SipHash_Final((uint8_t *)&siphash, &ctx);

    return (siphash[0] ^ siphash[1]);
}
```

此时，pf 处理停止。我们不会创建状态，也不会对数据包进行任何进一步的检查。这也意味着，如果防火墙保护的是另一台主机（即它运行在客户端和服务器之间的路由器上），服务器将根本不会意识到客户端尝试发起新连接。我们希望这样做，因为这意味着服务器可以免受 SYN 洪水攻击，而无需任何代码或配置更改。

如果客户端从未回应，则什么也不会发生。服务器没有记住任何关于此特定 SYN 消息的信息，也没有为其分配任何内存。另一方面，如果客户端确实回应（即它是一个合法客户端，至少在本讨论中是如此），我们必须重建在接收到原始 SYN 消息时未保留的信息。

在接收到 SYN+ACK 消息后，我们首先在 `pf_syncookie_validate()` 中验证它：

```c
uint8_t
pf_syncookie_validate(struct pf_pdesc *pd)
{
    uint32_t hash, ack, seq;
    union pf_syncookie cookie;

    MPASS(pd->proto == IPPROTO_TCP);
    PF_RULES_RASSERT();

    seq = ntohl(pd->hdr.tcp.th_seq) - 1;
    ack = ntohl(pd->hdr.tcp.th_ack) - 1;
    cookie.cookie = (ack & 0xff) ^ (ack >> 24);

    /* 我们在设置 cookie（联合体）之前不知道 oddeven。*/
    if (atomic_load_64(&V_pf_status.syncookies_inflight[cookie.flags.oddeven]) == 0)
        return (0);

    hash = pf_syncookie_mac(pd, cookie, seq);

    if ((ack & ~0xff) != (hash & ~0xff))
        return (0);

    counter_u64_add(V_pf_status.lcounters[KLCNT_SYNCOOKIES_VALID], 1);
    atomic_add_64(&V_pf_status.syncookies_inflight[cookie.flags.oddeven], -1);

    return (1);
}
```

我们检查 cookie 是否包含正确的认证字符串。如果是，我们继续进入 pf_syncookie_recreate()，在那里我们重建原始的 SYN 包。这对于 syncookie 系统本身来说并不是严格要求的，但我们需要告诉 pf 关于我们最初丢弃的 SYN 包，以便它可以创建相关的状态条目。

这还允许 pf 继续处理，并可能将重建的 SYN 包转发到远程服务器。远程服务器随后会以不同的序列号回复其自己的 SYN+ACK 包。pf 必须修改客户端和服务器之间所有流量的序列号和确认号。幸运的是，这是 pf 的标准功能。

此时，连接已在双方完全建立，并且与没有使用 syncookies 设置的连接没有显著差异。关闭连接时不需要采取特殊的操作，因为这不会为恶意客户端生成内存压力提供新的机会。

## 缺点

到目前为止，我们已经讨论了 syncookies 如何帮助我们，但我们并没有花太多时间讨论任何缺点。这是否意味着没有缺点？遗憾的是，不是。

我们已经谈到了 MSS 和 WSCALE。使用 syncookies 时，我们无法完全忠实地反映客户端提出的值。这可能意味着对于某些客户端，我们会损失一些 TCP 性能。通常情况下，这并不值得担心。

另一个缺点是 syncookies 工作方式中隐含的问题：我们无条件地对 SYN 包回复 SYN+ACK。即使端口实际上是关闭的。这意味着客户端可能认为连接正在工作，直到它完全建立，然后收到一个 RST。那并不理想，可能会引发客户端的意外行为。也就是说，这可能会让用户觉得连接失败的表现与“正常”失败不同。

通过确保防火墙立即拒绝发送到关闭端口的包，这种情况可以大大减轻。无论如何，这通常是一个好主意，如果启用了 syncookies，更是如此。

另一个缺点是，丢失的 SYN+ACK 包没有重传机制。因为一旦我们发送了 SYN+ACK，我们就会忘记关于它的所有信息，无法进行重传。这并不太令人担心，因为客户端会假设它的 SYN 包丢失，并重新发送 SYN 包。这将导致服务器生成一个新的 SYN+ACK，希望这次不会丢失。

最后需要注意的是，syncookies 并非魔法。它们对抗 SYN flood 攻击非常有效，但无法防止其他类型的攻击。例如，如果一个特殊构造的 HTTP 请求消耗了 Web 服务器的过多系统资源，这种情况是 syncookies 无法阻止的。

此外，没有任何机制可以阻止攻击者从多个不同的客户端 IP 地址发起连接，而不伪造源地址。在这种情况下，攻击者仍然有可能打开足够多的连接来耗尽服务器的资源。然而，syncookies 使得攻击者的代价大大增加。单个攻击主机就可以执行 SYN flood 攻击，且带宽需求适中。而使用非伪造的 TCP 连接进行相同效果的攻击将需要更多的攻击主机。

## 历史

OpenBSD pf 的 syncookie 代码基于 FreeBSD TCP 栈中的 syncookie 代码，最初由 Jonathan Lemon 于 2001 年开发（a9c96841638186f2e8d10962b80e8e9f683d0cbc）。

看起来 OpenBSD 的提交信息在归功于 Andre Oppermann 时是错误的。Andre 确实在 2013 年对 syncookie 代码做出了重大改进（81d392a09de0f2eeabaf68787896863eb9c370a8），这可能是误解的来源。

## 实现笔记

虽然 OpenBSD 和 FreeBSD 的 pf 版本多年来已经有所不同，但相似性仍然远远超过差异。因此，将 OpenBSD pf 特性移植到 FreeBSD 通常是相对简单的。主要的障碍是锁定策略的不同。OpenBSD 的 pf，像 OpenBSD 的网络栈一样，受到单一锁（NET_LOCK）的保护。这有一个很大的优点，就是简单性，但也带来了一些性能上的缺点。

FreeBSD 的 pf 采用了更复杂的锁定策略，但在性能上得到了更好的回报。

这在自适应模式中尤为重要。syncookie 代码的其他部分与现有的锁定方法很好地契合。然而，OpenBSD 采用的是递增和递减一个单一计数器的方式来跟踪半开状态和正在传输的 syncookie 包。这在 FreeBSD 中需要使用原子操作，因为没有等效的 NET_LOCK，并且多个核心可以同时处理 TCP SYN 或其他包。

虽然这确保了我们不会低估或高估半开状态或正在传输的 syncookie 包的数量，但它仍然不完美。虽然值的检索是原子的，但由于我们检索多个值，它们并不总是能反映一个完美的快照。幸运的是，这里并不要求严格的准确性。最坏的情况是我们稍微提前或稍晚地启用或禁用 syncookies。由于 syncookie 介导的连接和正常连接可以同时建立，因此对用户来说，这不是一个显著的问题。

## 配置

经过上述讨论，读者可能会认为配置 syncookies 是一项复杂的工作，但实际上并非如此。只需在 pf.conf 的选项部分添加一行即可：

```sh
set syncookies always
```

或

```sh
set syncookies adaptive
```

第一个模式将始终对 SYN 数据包做出响应，并返回一个 syncookie SYN+ACK。在自适应模式下，pf 只有在大量连接处于半开状态时才会这样做。也就是说，我们已对 ACK 消息回复 SYN+ACK，并在等待对方的 ACK 响应。理想情况下，这结合了两者的优点：我们既能获得正常 TCP 连接处理的所有优点（即完全的选项协商，当连接无法打开时可以立即得到反馈），又能在一定程度上防范 SYN flood 攻击。

自适应模式、低水位和高水位标记（即分别启用和禁用 syncookies 的位置）也可以进行配置：

```sh
set syncookies adaptive (start 25%, end 12%)
```

这些值以状态表大小的百分比表示。如果没有配置 syncookie 行，功能将默认为禁用。这意味着除非用户明确启用 syncookies，否则行为不会改变。

## 结论

syncookies 适合你吗？如果你的系统遭遇 SYN flood 攻击，它们可能会适合你。如果没有，你可能想保持它们禁用，但即便如此，了解它们的存在也是好的。SYN flood 攻击是一种非常古老的攻击方式，但只要它们偶尔有效，攻击者可能会决定使用它们。防御者必须准备好合适的工具。

新的 pf syncookie 功能已经在最近的 12.3 版本中出现，并将在即将发布的 13.1 版本中提供。

将 OpenBSD pf syncookie 代码移植到 FreeBSD 的 pf 的工作得到了 Modirum MDPay 的赞助。

---

**KRISTOF PROVOST** 是一位自由职业的嵌入式软件工程师，专注于网络和视频应用。他是 FreeBSD 的提交者，FreeBSD 中 pf 防火墙的维护者，也是 EuroBSDCon 基金会的董事会成员。Kristof 不幸地有一种倾向，总是会遇到 uClibc 的 bug，对 FTP 怀有强烈的厌恶。不要和他谈论 IPv6 分片。
