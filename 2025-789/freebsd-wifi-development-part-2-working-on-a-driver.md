# FreeBSD WiFi 开发第二部分：驱动开发

- 原文：[FreeBSD WiFi Development Part 2: Working on a Driver](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/freebsd-wifi-development-part-2-working-on-a-driver/)
- 作者：Tom Jones

这是 FreeBSD 上 WiFi 开发系列的第二篇文章。在[第一篇文章](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/freebsd-wifi-development/)中，我们介绍了 WiFi/80211 网络的一些术语，简要讲解了典型的网络架构，并展示了如何使用 `ifconfig` 和一些无线网卡创建 station、host ap 和 monitor 模式的 WLAN 接口。我们还介绍了实现 WiFi 子系统的两个不同内核层 —— **驱动** 和 **net80211**。

**驱动**：如 iwx、rtwn 和 ath，通过 USB 或 PCIe 等物理总线与无线网卡通信，通常通过固件接口实现。

**net80211**：加入网络、发送数据包以及执行其他复杂操作所需的抽象状态机，在许多驱动中是通用的。

为了在硬件需求上保持灵活性，**net80211** 层本身能实现 IEEE 802.11 状态机的大多数部分。这种架构使我们能够用一个标准接口与整个网络栈集成，屏蔽不同网卡支持能力的差异。同时，它还能支持纯软件的 WLAN 适配器，这在测试环境中非常有用。

网卡对功能的支持程度不同，范围从 **FullMAC 接口**（几乎所有处理都在网卡上直接完成），到几乎完全依赖 **net80211** 堆栈，仅由网卡管理射频。在 FullMAC 卡中，固件会为操作系统驱动暴露一个配置接口，所有的数据包收发和管理操作（如切换信道、扫描）都由固件完成。OpenBSD 和 NetBSD 中的 **bwfm Broadcom 驱动**就是一个 FullMAC 驱动的例子。

其他网卡需要 **net80211** 堆栈提供多种服务以支持驱动的运行。一些设备（如 iwx）提供扫描和加入网络等管理接口，但大多数操作仍由 **net80211** 完成。

较老的驱动则必须自己实现更多的 **net80211** 状态机逻辑，而不是重复大量类似的代码。

实际上，所有 WiFi 驱动都处在这样一个范围内：一端是固件几乎完成所有工作，另一端是操作系统管理大部分射频和传输。要实际了解其运行方式，最好的方法就是直接看一个驱动。

接下来我们先看看驱动是如何附加并出现在 `net.wlan.devices` 列表中的，然后再看一个数据包是如何从 **net80211** 堆栈发往 WiFi 射频的。

本文将主要关注 **if_iwx** 驱动，原因有两点：第一，我对它非常熟悉，因为我曾将该驱动从 Future Crew 的源码引入 FreeBSD 树；第二，作为一个新驱动，它还有很多“低垂的果实”（容易改进的地方）。


## 将驱动连接到硬件

驱动的生命周期通常包括：

- **probe**
- **attach** （执行一些初始化工作）
- **detach**

许多驱动只有在关机时才会经历 detach。**probe** 和 **attach** 阶段是我们在为已有驱动添加新硬件支持或编写新驱动时需要重点处理的。

当总线发现某设备时，会依次询问所有已注册的驱动是否能支持该硬件。在所有驱动被询问后，总线会按照探测响应顺序，请求驱动是否能 `attach` 设备。第一款成功附加的驱动“获胜”。

在之后的某个时刻，驱动可能会被移除：可能由于总线错误、设备移除（如 USB 拔出），或者系统关机、重启。

在这些阶段中，每一步都通过与总线注册的回调来实现。举例来说，下面是 `if_iwx.c` 中的 `pci_methods` 结构体。


```c
static device_method_t iwx_pci_methods[] = {
        /* 设备接口 */
        DEVMETHOD(device_probe,         iwx_probe),
        DEVMETHOD(device_attach,        iwx_attach),
        DEVMETHOD(device_detach,        iwx_detach),
        DEVMETHOD(device_suspend,       iwx_suspend),
        DEVMETHOD(device_resume,        iwx_resume),

        DEVMETHOD_END
};
```

`if_iwx` 注册了 probe、attach 和 detach 方法，以及 suspend 和 resume 方法，这些方法都会在需要时被调用。

## Probe

WiFi 设备通常由一颗芯片组和一些辅助硬件组成。芯片组由 Realtek 或 Intel 这样的公司制造，但实际设备则是由另一家公司基于芯片组生产的。这种模式意味着我们会得到基于 rtwn 的设备，但它们可能由 TP-Link 这样的公司制造。围绕芯片组构建设备的公司会提供驱动和配置信息，从而使少量驱动可以支持更多的设备 ID。

这也意味着，FreeBSD 新贡献者常见的首个补丁，就是为某些尚未覆盖的设备添加设备 ID（我自己的第一个补丁就是在一台 MIPS 路由器中添加了一颗闪存芯片 ID！）。

在 FreeBSD WiFi 中，你的第一个改动也可能很直接：买一台你认为应该能用的设备并测试它（按照本系列第一篇文章中的说明操作）。

如果没有驱动能探测到该硬件，你可以列出 USB 或 PCIe 设备 ID，然后参考其他平台来判断应该由哪个驱动支持它们。

我最近为一位外部贡献者合并的两个 FreeBSD 改动就是这样的：为 `if_run` 和 `if_rum` 驱动添加了硬件设备 ID。下面是 run 驱动部分的改动示例：

```c
diff --git a/sys/dev/usb/wlan/if_run.c b/sys/dev/usb/wlan/if_run.c
index 00e005fd7d4d..97c790dd5b81 100644
 a/sys/dev/usb/wlan/if_run.c
+++ b/sys/dev/usb/wlan/if_run.c
@@ -324,6 +324,7 @@ static const STRUCT_USB_HOST_ID run_devs[] = {
     RUN_DEV(SITECOMEU,         RT2870_3),
     RUN_DEV(SITECOMEU,         RT2870_4),
     RUN_DEV(SITECOMEU,         RT3070),
+    RUN_DEV(SITECOMEU,         RT3070_1),
     RUN_DEV(SITECOMEU,         RT3070_2),
     RUN_DEV(SITECOMEU,         RT3070_3),
     RUN_DEV(SITECOMEU,         RT3070_4),
```

你在 FreeBSD 上的第一个改动可能就只是为某个设备的设备 ID 列表添加一行代码。在完成这一步之后，你需要在相关驱动里加上对应条目，测试它，然后把 diff 邮件发送给我 [thj@freebsd.org](mailto:thj@freebsd.org)，我会帮你提交。

## 连接到 net80211

WiFi 驱动所需的状态存储在驱动 softc 中的一个 `ieee80211com` 变量里（通常命名为 `ic`）。

驱动使用 `ic` 来设置能力标志，并通过覆盖函数指针来挂接或替换 **net80211** 栈提供的默认功能。

在本系列的上一篇文章中，我展示了如何通过 `ifconfig` 命令在一个驱动之上创建虚拟 WLAN 接口（VAP）。VAP 允许我们在同一块网卡上创建多个接口，并让它们在不同模式下运行，例如 **sta**、**host ap**、**monitor** 等。驱动通过 `ic_caps` 位字段来管理这些模式的可用性。

这些字段的值会在驱动的 attach 过程中设置。下面是 `if_iwx` 驱动中 `iwx_attach` 函数的一个示例：

```c
...
ic->ic_softc = sc;
ic->ic_name = device_get_nameunit(sc->sc_dev);
ic->ic_phytype = IEEE80211_T_OFDM; /* 不仅仅是 OFDM，但此处未使用 */
ic->ic_opmode = IEEE80211_M_STA;   /* 默认设置为 BSS 模式 */

/* 设置设备能力 */
ic->ic_caps =
    IEEE80211_C_STA |          /* 支持 STA 模式 */
    IEEE80211_C_MONITOR |      /* 支持监控模式 */
    IEEE80211_C_WPA |          /* 支持 WPA/RSN */
    IEEE80211_C_WME |          /* 支持 WME (QoS) */
    IEEE80211_C_PMGT |         /* 支持电源管理 */
    IEEE80211_C_SHSLOT |       /* 支持短时隙 */
    IEEE80211_C_SHPREAMBLE |   /* 支持短前导码 */
    IEEE80211_C_BGSCAN         /* 支持后台扫描 */
...
```

[来自 `if_iwx` 的 attach](https://cgit.freebsd.org/src/tree/sys/dev/iwx/if_iwx.c#n10127)

这段代码位于 `if_iwx` 的 attach 方法末尾。前面的 attach 代码则负责执行驱动的初始化工作，例如识别具体的 PCIe 设备，以及确定该网卡的具体 Intel Wireless 型号。

`if_iwx` 驱动支持 **station 模式**（`IEEE80211_C_STA`）和 **monitor 模式**（`IEEE80211_C_MONITOR`）；如果它支持 **host AP 模式**（像 rtwn 那样），那么在能力位掩码中就会额外包含 `IEEE80211_C_HOSTAP` 标志。

除了模式以外，iwx 还支持：WPA 加密（`IEEE80211_C_WPA`）、差分服务的多媒体扩展（`IEEE80211_C_WME`）、电源管理（`IEEE80211_C_PMGT`）、短时隙（`IEEE80211_C_SHSLOT`）、短前导码（`IEEE80211_C_SHPREAMBLE`）以及后台扫描（`IEEE80211_C_BGSCAN`）。

完整的能力标志列表在 **ieee80211.h** 头文件中。驱动能声明哪些能力，既取决于硬件特性，也取决于驱动是否实现。在驱动开发阶段，某些功能（例如 WPA 硬件卸载）可能尚未实现，因此缺少某个标志并不意味着硬件不支持该特性。

驱动在 attach 阶段执行的第二个任务，是接管或实现 **net80211** 的功能，这通过 `iwx_attach_hook` 配置回调完成。在这里，驱动会覆盖 `ic_caps` 位字段所声明的大量特性的函数指针。

首先，`if_iwx` 会创建 **信道映射表**。对于这类网卡，驱动必须向网卡固件请求，以获取一组受支持的信道。

```c
iwx_init_channel_map(ic, IEEE80211_CHAN_MAX, &ic->ic_nchans,
        ic->ic_channels);

ieee80211_ifattach(ic);
ic->ic_vap_create = iwx_vap_create;
ic->ic_vap_delete = iwx_vap_delete;
ic->ic_raw_xmit = iwx_raw_xmit;
ic->ic_node_alloc = iwx_node_alloc;
ic->ic_scan_start = iwx_scan_start;
ic->ic_scan_end = iwx_scan_end;
ic->ic_update_mcast = iwx_update_mcast;
ic->ic_getradiocaps = iwx_init_channel_map;

ic->ic_set_channel = iwx_set_channel;
ic->ic_scan_curchan = iwx_scan_curchan;
ic->ic_scan_mindwell = iwx_scan_mindwell;
ic->ic_wme.wme_update = iwx_wme_update;
ic->ic_parent = iwx_parent;
ic->ic_transmit = iwx_transmit;

sc->sc_ampdu_rx_start = ic->ic_ampdu_rx_start;
ic->ic_ampdu_rx_start = iwx_ampdu_rx_start;
sc->sc_ampdu_rx_stop = ic->ic_ampdu_rx_stop;
ic->ic_ampdu_rx_stop = iwx_ampdu_rx_stop;

sc->sc_addba_request = ic->ic_addba_request;
ic->ic_addba_request = iwx_addba_request;
sc->sc_addba_response = ic->ic_addba_response;
ic->ic_addba_response = iwx_addba_response;

iwx_radiotap_attach(sc);
ieee80211_announce(ic);
```

接着，驱动会替换或拦截 net80211 通过设备的 IC 所发起的调用。`ic_vap_create` 和 `ic_raw_xmit` 由驱动提供实现，而 `sc_ampdu_rx_start` 和 `stop` 等调用则是被拦截的。

最后，驱动会附加到 **radiotap 子系统**，这使得原始数据包能够传递给 BPF，然后驱动会向 net80211 系统声明自身的存在。

在 attach 方法中的两个 `ieee80211_` 调用就是我们与 net80211 系统交互的例子。第一个调用会把我们的驱动附加到 net80211 子系统（这一步会让驱动被加入到 `net.wlan.devices` sysctl 后面的列表中）。这样一来，驱动就能被 `ifconfig` 使用。

第二个调用（`ieee80211_announce`）负责声明设备已被创建；此时会打印出该网卡的信道与特性支持情况。

待驱动附加到 net80211 子系统，它就会处于空闲状态，直到外部事件触发它进入运行状态。运行的下一部分由 net80211 处理，它会调用我们在 `attach_hook` 回调中覆盖的方法。


## 实现 station 模式

在第一篇文章中，我们为示例创建了一个 station 模式的 VAP。我们运行的命令是：

```sh
ifconfig wlan create wlandev iwx0
```

`wlan` 参数让系统为我们分配一个设备号，而 `iwx0` 则告诉 net80211 子系统使用名为 `iwx0` 的设备来创建这个 VAP。

该命令会通过 `ifconfig` 的库转换为一次 `net80211_ioctl` 调用。最终结果是 net80211 会在我们的驱动 `ic` 上调用 `ic->ic_vap_create` 回调。正如前文所述，这个回调映射到 `iwx_vap_create`。

```c
struct ieee80211vap *
iwx_vap_create(struct ieee80211com *ic, const char name[IFNAMSIZ], int unit,
    enum ieee80211_opmode opmode, int flags,
    const uint8_t bssid[IEEE80211_ADDR_LEN],
    const uint8_t mac[IEEE80211_ADDR_LEN])
{
        struct iwx_vap *ivp;
        struct ieee80211vap *vap;

        if (!TAILQ_EMPTY(&ic->ic_vaps))         /* 一次只允许一个 */
                return NULL;

        ivp = malloc(sizeof(struct iwx_vap), M_80211_VAP, M_WAITOK | M_ZERO);
        vap = &ivp->iv_vap;

        ieee80211_vap_setup(ic, vap, name, unit, opmode, flags, bssid);
        vap->iv_bmissthreshold = 10;            /* 覆盖默认值 */

        /* 用驱动方法覆盖默认方法 */
        ivp->iv_newstate = vap->iv_newstate;
        vap->iv_newstate = iwx_newstate;

        ivp->id = IWX_DEFAULT_MACID;
        ivp->color = IWX_DEFAULT_COLOR;

        ivp->have_wme = TRUE;
        ivp->ps_disabled = FALSE;

        vap->iv_ampdu_rxmax = IEEE80211_HTCAP_MAXRXAMPDU_64K;
        vap->iv_ampdu_density = IEEE80211_HTCAP_MPDUDENSITY_4;

        /* 硬件加密支持 */
        vap->iv_key_alloc = iwx_key_alloc;
        vap->iv_key_delete = iwx_key_delete;
        vap->iv_key_set = iwx_key_set;
        vap->iv_key_update_begin = iwx_key_update_begin;
        vap->iv_key_update_end = iwx_key_update_end;

        ieee80211_ratectl_init(vap);

        /* 完成设置 */
        ieee80211_vap_attach(vap, ieee80211_media_change,
            ieee80211_media_status, mac);

        ic->ic_opmode = opmode;

        return vap;
}
```

`iwx_vap_create` 会做一些内务处理来管理内存，并建立 net80211 系统需要使用的回调。对于 `iwx`，它会建立特定于驱动的状态（`IWX_DEFAULT_MACID` 和 `IWX_DEFAULT_COLOR` 值），用于与固件协调，确定默认使用哪个 station。

对于 `iwx_vap_create` 所挂接的一些函数，我们保留默认方法，但会拦截对它的调用。例如，我们覆盖了 `iv_newstate` 回调，并通过 `iwx_newstate` 进行过滤。

`iwx` 的固件自身管理了大量状态；例如在 **探测 (probe)** 时，可以请求硬件在受支持的信道上发送探测报文，而我们无法直接自己构造并发送这些报文。

因此，iwx 驱动必须挂接 `newstate` 方法，以便向固件发出请求并更新其状态机。通过这种方式，**net80211** 与固件的状态机能够保持与主机层面变化同步。


## 发送数据包

到目前为止，我们已经覆盖了足够的驱动部分，可以用 `ifconfig` 启动接口，并让操作系统开始发送数据包。

在测试接口时，我们可能会使用 ifconfig 按如下流程操作：

```sh
# ifconfig wlan0 ssid open-network up
```

这些命令指示 `ifconfig` 启用该接口，并请求 **net80211** 堆栈加入开放 Wi-Fi 网络 `open-network`。它还为接口设置了地址，但这并不会在物理链路上产生任何数据包（准确地说，是在空中）。

接下来看看这一系列命令会映射到哪些驱动方法。

在我们的 attach 钩子中，为 **net80211** 层设置了两个用于发送数据包的回调：`ic_transmit` 和 `ic_raw_transmit`，以及一个用于控制接口状态的回调：`ic_parent`。

```c
ic->ic_raw_xmit = iwx_raw_xmit;
...
ic->ic_parent = iwx_parent;
ic->ic_transmit = iwx_transmit;
```

`ifconfig` 命令中的 `up` 部分最终会调用 `ic_parent` 回调。对于 iwx，这个回调是 `iwx_parent`：

```sh
static void
iwx_parent(struct ieee80211com *ic)
{
        struct iwx_softc *sc = ic->ic_softc;
        IWX_LOCK(sc);

        if (sc->sc_flags & IWX_FLAG_HW_INITED) {
                iwx_stop(sc);
                sc->sc_flags &= ~IWX_FLAG_HW_INITED;
        } else {
                iwx_init(sc);
                ieee80211_start_all(ic);
        }
        IWX_UNLOCK(sc);
}
```

`iwx_parent` 会直接控制硬件：如果正在运行，就调用 `iwx_stop` 来清除所有硬件状态；如果尚未运行，就调用 `iwx_init` 让硬件完成初始配置。只要硬件准备就绪，我们就调用 `ieee80211_start_all` 通知 net80211 栈可以开始工作了。

看似简单的 `ifconfig up` 动作，实际上会导致 `iwx` 驱动修改大量硬件状态。这也是为什么“把接口关掉再打开”常常被当作解决网络问题的“魔法修复”的原因之一。

`ifconfig` 命令的第二部分让 net80211 栈执行更多操作。通过向 `ifconfig` 传入 `ssid open-network`，我们请求 net80211 子系统去发现并加入名为 `open-network` 的网络。

加入一个 IEEE 802.11 网络的过程分为几个步骤：

- **探测 (probe)** 网络
- **认证 (authenticate)** 到网络
- **关联 (associate)** 到网络

每一步都需要设备发送管理帧。首先要发现目标网络 —— 网络会定期通过 beacon 报文广播自己的存在（这就是你菜单栏里能看到的 Wi-Fi 列表）。操作系统据此获得候选网络列表。当设备要加入某个网络时，它会向目标网络发送探测请求 (probe request)，并等待探测响应 (probe response)。这一过程在主机和网络之间传递配置信息，确认该网络确实可用。

接下来是认证和关联。在这一步完成后，接口进入 **RUN** 状态，就可以像普通网络接口一样开始使用了。

随着协议栈在各个状态之间切换，它会触发对 `iv_newstate` 函数的调用。在 `iwx` 驱动中，这个调用会首先经过 `iwx_newstate` 拦截，从而允许驱动控制状态转换时的报文发送。这是必要的，因为在 `iwx` 中，某些状态转换是由固件处理的，而不是通过 net80211 栈直接发包完成的。

例如，探测请求并不是直接发出的，而是通过固件接口触发一次信道扫描。一旦发现网络并决定加入，就向固件发送一条“添加站点”的消息，而不是从 net80211 栈发出关联报文。

并非所有管理帧都通过固件抽象发送，在这些情况下，系统会调用 `iwx_raw_xmit` 回调。如果你在调试驱动时发现发送路径并不总是被触发，那可能是因为管理帧是通过原始路径 (raw path) 发出的。



## 总结

本文介绍了驱动如何进行探测、附加以及发送首批数据包。通过使用现有驱动，我们可以很快覆盖驱动的大量逻辑。不过，如果你查看源码，就会发现 `if_iwx.c` 长达一万多行，这远远超出了本文的范围。

这篇文章作为 Wi-Fi 驱动入门，省略了许多细节。要真正加入一个网络，我们必须能够从接口既发送又接收数据包。

如果收不到任何数据包，我们能如何调试？系统提供了哪些工具？

在本系列的第三部分，我们将介绍 **net80211 栈的内置调试功能**，以及它们是如何与驱动结合，用于开发、测试和故障排查的。

---

**Tom Jones** 是一名 FreeBSD 提交者，关注于保持网络栈的高速性能。
