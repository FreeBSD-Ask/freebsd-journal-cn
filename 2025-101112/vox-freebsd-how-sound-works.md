# Vox FreeBSD：如何让音频工作

- 原文：[Vox FreeBSD: How Sound Works](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/vox-freebsd-how-sound-works/)
- 作者：Christos Margiolis

FreeBSD 的声音支持始于 1993 年，当时 Jordan K. Hubbard 将通用 Linux 声卡驱动移植到了 FreeBSD，该驱动后来被称为 VoxWare 声卡驱动，由 Hannu Savolainen 编写。1993 年至 1997 年间，多个新版本的 VoxWare 驱动被引入（并修改）到 FreeBSD 基本系统。当时，Amancio Hasty 和 Jim Lowe 负责了 FreeBSD 声音支持的大部分工作。VoxWare 驱动最终发展成为我们今天所知的 OSS（Open Sound System，开放声音系统），由 Hannu 及其团队在 4Front Technologies 维护。

然而，1997 年情况发生了变化，当时 Luigi Rizzo 构建了现代 FreeBSD 声音系统的基础。1999 年，Cameron Grant 为 FreeBSD 4.0 重写了声卡系统，使用了 newbus 接口来支持大量硬件。这取代了 Luigi 的驱动，其驱动在一个月后被从代码树中移除。随后几年，声音领域经历了诸多变化。针对不同硬件芯片的驱动数量显著增加（尤其是 PCI 和 USB 设备），声卡基础设施也得到了多项改进，这都得益于 Cameron Grant 和 Orion Hodson。Cameron 于 2005 年离世，这是 FreeBSD 社区的一大损失。

2005 年，Ariff Abdullah 接管了 FreeBSD 声音代码的维护工作。自此，FreeBSD 声音支持经历了显著变化，并进行了多轮设备驱动重构。

## OSS

FreeBSD 的声音 API 被称为 OSS，即 “Open Sound System”（开放声音系统）。有趣的是，它曾经也是 Linux 的默认声音 API，后被 ALSA 取代。而 FreeBSD 仍在使用 OSS，它通过标准设备文件提供了简单清晰的接口（`/dev/dsp*` 对应声卡，`/dev/mixer*` 对应混音器），并通过少量常用 POSIX 系统调用进行操作：

| 系统调用                          | 说明                                                                                                                                                            |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| open(2)                       | 打开设备。                                                                                                                                                           |
| close(2)                      | 关闭设备。                                                                                                                                                           |
| read(2)                       | 录制音频。                                                                                                                                                           |
| write(2)                      | 播放音频。                                                                                                                                                           |
| ioctl(2)                      | 查询或操作设置（采样率、格式、音量等）。                                                                                                                                            |
| select(2), poll(2), kqueue(2) | 事件轮询。`kqueue(2)` 为 FreeBSD 独有（从 [15.0 版本起](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/vox-freebsd-how-sound-works/) 支持）。 |
| mmap(2)                       | 内存映射 I/O。                                                                                                                                                       |

官方手册可在此获取：[http://manuals.opensound.com/developer/](http://manuals.opensound.com/developer/)

由于与声卡的交互使用的是常用系统调用，因此在终端中可以执行如下操作：

1. 播放白噪声：`cat /dev/random > /dev/dsp`
2. 将原始 PCM（Pulse Code Modulation，脉冲编码调制）流录入文件：`cat /dev/dsp > foo`
3. 非常粗略的输入监控，可用于检查麦克风是否工作：`cat /dev/dsp > /dev/dsp`

然而，要实现更复杂的功能，则需要编写实际的程序——幸运的是，OSS API 使用起来非常直观。`/usr/share/examples/sound` 有多个示例程序，并且在本文撰写时仍在不断增加。

除了 `/dev/dsp*` 设备外，还有一个 `/dev/dsp` 设备，上述示例中使用的就是它。它并不是真实的设备，而是当前默认设备的别名。建议应用程序在需要使用当前默认设备时访问该别名，而不要硬编码特定设备。

OSS 的另一个优点是它只依赖 POSIX 系统调用，因此可以非常容易地在任何编程语言中使用，无需语言绑定。若想将 OSS 完全“移植”到其他语言，只需定义一些全局 OSS API 常量和结构体，可在 `/usr/include/sys/soundcard.h` 中找到上述定义。

## `sound(4)`

OSS API 在 `sound(4)` 中实现。它将一些通用功能抽象为内核模块，使得各个声卡驱动（如 `snd_hda(4)`、`snd_uaudio(4)` 等）无需重复实现这些通用代码。通用功能包括：

* 设备的创建、删除和访问。
* 缓冲区管理。
* 音频处理。
* 提供全局以及大部分设备特定的 sysctl 接口。

驱动在初始化阶段必须附加到 `sound(4)` 并建立与其的通信管道。换言之，`sound(4)` 是用户空间与设备驱动之间的桥梁。这非常方便，因为 FreeBSD 内核为每个连接的声卡都暴露相同的 `/dev/dsp*` 文件和 sysctl（`hw.snd.*` 和 `dev.pcm.*`），为用户和应用程序提供了统一访问和配置，同时避免驱动层的重复实现。

`sound(4)` 还提供了 `/dev/sndstat`，用以显示所有已连接声卡和活动通道的信息，并被 `virtual_oss(8)` 和 `sndctl(8)` 内部使用。

`sound(4)` 处理 PCM 音频流，其具体参数（采样率、采样格式等）可由用户通过 `sndctl(8)` 手动配置。这意味着，对于使用其他格式（如 WAV、Opus、MP3 等）的音频，应用程序在播放时需要将其转换为 PCM 流，录制时则需从 PCM 转回目标格式。

## 高层实现概览

### 设备访问

如前所述，并在示例程序中展示，应用程序通过 `/dev/dsp*` 访问声音子系统，使用 `open(2)` 系统调用。可指定以下标志：

| open(2) 标志 | 功能   |
| ---------- | ----- |
| O_RDONLY   | 录音    |
| O_WRONLY   | 播放    |
| O_RDWR     | 录音和播放 |

在 `open(2)` 调用中，还可以额外指定 `O_NONBLOCK` 和 `O_EXCL` 标志，分别用于非阻塞 I/O 和对设备的独占访问。为了让 `sound(4)` 知道哪些通道属于哪个文件描述符，并正确路由音频和信息，它使用 `DEVFS_CDEVPRIV(9)`。

如前所述，还有 `/dev/mixer*` 设备。这是一个遗留接口，主要用于音量设置和录音源选择，例如被 `mixer(8)` 等应用程序使用。`/dev/mixer*` 设备与 `/dev/dsp*` 设备一一对应，例如 `/dev/mixer0` 对应 `/dev/dsp0`。该接口还提供部分物理混音器功能。从 OSS 4.0 版本开始，混音器 API 已被重写，但在 FreeBSD 上尚未完全实现。

### 通道（Channel）

通道保存了其状态的重要信息（如配置、使用该通道的进程 PID 与名称、诊断值等），以及最关键的组件：缓冲区，也就是实际的音频流。通道除了设备级的主音量外，还具有自己的独立音量。这一点尤其有用，因为应用程序可以拥有自己的音量控制，通常无需调整主音量。

需要注意的是，`sound(4)` 中有两类通道：**主通道/“硬件”通道** 和 **虚拟通道**。

主通道由设备驱动分配，通常硬件支持一个播放通道和一个录音通道。但某些驱动可能会将主通道数量设置为硬件提供的物理播放和录音端口数。每个主通道包含一对缓冲区：一个面向软件（userland），一个面向硬件。面向软件的缓冲区负责与用户空间交换音频数据，面向硬件的缓冲区则用于与硬件交换数据。当设备驱动准备好从硬件读取或向硬件写入数据时，会触发对 `sound(4)` 的中断，然后数据在两个缓冲区之间复制。播放时，由于数据写入硬件，软件缓冲区的数据会复制到硬件缓冲区，以便驱动将其发送给硬件；录音时则反向操作。

虚拟通道（通常称为 VCHAN）被视为主通道的子通道，但与主通道不同，它们不与硬件直接连接，因此只使用面向软件的缓冲区。虚拟通道存在的原因是希望允许无限数量的应用程序同时访问设备。如果没有虚拟通道，同时访问设备的进程数将等于主通道的数量，因为每个主通道只有一个面向软件的缓冲区，这意味着所有进程必须共享同一个缓冲区，这对于音频处理并不理想，因此每个缓冲区必须专用于一个进程。

前文介绍了如何通过主通道在用户空间与硬件之间交换音频流。当启用虚拟通道（默认情况）时，`sound(4)` 不再只是简单地将主通道的软件缓冲区复制到硬件缓冲区（播放时），或反向操作（录音时）；它首先需要混合主通道所有子虚拟通道的音频流，然后将最终混合后的流提供给主通道的硬件缓冲区。可以想象，这一额外层会引入轻微的开销，对于大多数场景影响不大，但在某些低延迟的音乐制作工作流程中可能不理想。对于这些场景，可以选择禁用虚拟通道。

要查看通道状态，可以使用 `sndctl(8)`：

```sh
$ sndctl -v
```

### 处理链（Processing chain）

`sound(4)` 一个有趣的功能是其处理链，包括以下内容：

* **混音（Mixing）**：实际上就是前一节中讲的虚拟通道音频流的混合与解混合。
* **音量控制（Volume control）**。
* **通道矩阵（Channel matrixing）**：`sound(4)` 可以进行任意通道映射，例如单声道到立体声，或立体声到 5.1 环绕声。这是通过将流从一种交错 PCM 格式转换为另一种实现的。
* **基础参数均衡（Basic parametric equalization）**。
* **格式转换（Format conversions）**。
* **重采样（Resampling）**，包括三种类型：

  * 线性（Linear）
  * 零阶保持（Zero-order-hold, ZOH）
  * 正弦基函数（Sine Cardinal, SINC）

每个通道都有自己的处理链，只包含必要的组件。例如，如果通道配置的采样率为 44100Hz，但应用程序提供的音频是 48000Hz，则该通道的处理链中需要包含重采样组件；如果音频流的采样率与通道相同，则该组件不需要。其他组件也是类似逻辑。

一个直观的方法是使用 `sndctl(8)` 打印处理链。下面的命令可以打印每个活动通道的处理链：

```sh
$ sndctl feederchain
```

在下一节中，我们将讨论在某些特殊情况下，为什么以及如何完全绕过处理链。

### 内存映射 I/O 与比特完美音频

`sound(4)` 的两个特性，尤其受到低延迟应用开发者和音频爱好者的欢迎：**比特完美模式**和**内存映射 I/O**。

* **比特完美模式（Bit-perfect mode）**：意味着音频流会跳过 `sound(4)` 的所有处理链，几乎直接送入声卡。使用该模式的应用程序需要自行确保音频流的配置（采样率、格式、通道矩阵）与声卡匹配。例如，如果应用程序播放 48000Hz 的音频，但声卡不支持此采样率，则应用程序必须负责重采样。比特完美模式默认是关闭的，仅由自行实现音频处理的应用或确信声卡支持比特完美模式的用户启用。

* **内存映射 I/O（Memory-mapped I/O）**：与比特完美类似，音频流跳过 `sound(4)` 的所有处理；实际上，使用内存映射 I/O 必须先启用比特完美模式。主要区别在于，内存映射 I/O 将所有音频缓冲区管理责任完全交给应用程序。应用不仅需要处理比特完美模式下的事项，还必须确保缓冲区正确同步，读写操作按时完成（在一定程度上借助 `sound(4)` 的帮助）。在合适的环境下正确实现，可以提高性能，但实现起来非常繁琐且容易出错，因此通常不推荐使用，除非开发者非常清楚自己在做什么。

## 设备驱动程序

正如 `sound(4)` 是用户空间与设备驱动之间的桥梁，设备驱动则是 `sound(4)` 与实际硬件之间的桥梁。除了所有声卡驱动都需要附加到 `sound(4)` 并与其通信之外，驱动的其他功能取决于驱动本身。在未来的文章中，可以介绍如何从零编写一个声卡驱动。

FreeBSD 内置支持以下声卡：

| 驱动程序             | 支持的声卡                                  | 默认启用平台      |
| ---------------- | -------------------------------------- | ----------- |
| snd_ai2s(4)      | Apple I2S                              | powerpc     |
| snd_als4000(4)   | Avance Logic ALS4000                   |             |
| snd_atiixp(4)    | ATI IXP                                |             |
| snd_cmi(4)       | CMedia CMI8338/CMI8738                 | amd64, i386 |
| snd_cs4281(4)    | Crystal Semiconductor CS4281           |             |
| snd_csa(4)       | Crystal Semiconductor CS461x/462x/4280 | amd64, i386 |
| snd_davbus(4)    | Apple Davbus                           | powerpc     |
| snd_emu10k1(4)   | SoundBlaster Live! 和 Audigy            |             |
| snd_emu10kx(4)   | Creative SoundBlaster Live! 和 Audigy   | amd64, i386 |
| snd_envy24(4)    | VIA Envy24 及兼容型号                       |             |
| snd_envy24ht(4)  | VIA Envy24HT 及兼容型号                     |             |
| snd_es137x(4)    | Ensoniq AudioPCI ES137x                | amd64, i386 |
| snd_fm801(4)     | Forte Media FM801                      |             |
| snd_hda(4)       | Intel 高保真音频 (High Definition Audio)    | amd64, i386 |
| snd_hdsp(4)      | RME HDSP                               |             |
| snd_hdspe(4)     | RME HDSPe                              |             |
| snd_ich(4)       | Intel ICH AC’97 及兼容型号                  | amd64, i386 |
| snd_maestro3(4)  | ESS Maestro3/Allegro-1                 |             |
| snd_neomagic(4)  | NeoMagic 256AV/ZX                      |             |
| snd_solo(4)      | ESS Solo-1/1E                          |             |
| snd_spicds(4)    | I2S SPI                                |             |
| snd_t4dwave(4)   | Trident 4DWave                         |             |
| snd_uaudio(4)    | USB 音频和 MIDI                           | 插入设备时自动加载   |
| snd_via8233(4)   | VIA Technologies VT8233                | amd64, i386 |
| snd_via82c686(4) | VIA Technologies 82C686A               |             |
| snd_vibes(4)     | S3 SonicVibes                          |             |

此外，还支持以下 ARM 芯片：

* Allwinner A10/A20 和 H3
* Broadcom BCM2835
* Freescale Vybrid
* Freescale i.MX6

如果你的声卡在你使用的架构上默认驱动未启用，或者你使用的是未编译声卡支持的自定义内核配置，并且不确定声卡使用的是哪个驱动，可以运行以下命令：

```sh
# kldload snd_driver
```

`snd_driver` 是元驱动，会加载所有可用驱动。若确定哪个驱动与你的声卡匹配，就可以仅加载该驱动。

## 最近改进

在过去两年里，声音子系统经历了许多改进，包括多个错误和崩溃修复、引入不断扩展的 Kyua 测试套件和测试驱动（`snd_dummy(4)`），以及多次代码清理和重构。

几个重要的面向用户的改进包括：

* 现在支持热拔插。使用旧版 FreeBSD 的 USB 声卡用户可能记得，热拔插声卡通常会导致 USB 总线卡住，直到手动终止使用已断开设备的应用程序
  ([PR 194727](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=194727))。
* 浮点音频支持。这有点容易误解，因为至少目前在设备驱动层并未真正支持浮点音频，而是允许用户空间应用程序使用 OSS 进行浮点音频处理。已修复了不少 Port，例如 Wine，需要 OSS 提供浮点音频支持。
* `sound(4)` 现在每个设备只暴露一个 `/dev/dsp*` 文件，并在内部完成所有音频流路由，通过 DEVFS_CDEVPRIV(9) 实现，而不是为每个分配的音频流暴露一个 `/dev/dsp*` 文件。当前方法在实现和向用户空间暴露的内容上都更清晰。
* 对高保真音频卡 (`snd_hda(4)`) 的开箱支持更好。这类声卡一直是开发者和用户的痛点，因为它们往往带有非标准配置，需要在驱动或 `/boot/device.hints` 内添加手动补丁来补偿。一个常见问题是插入耳机时声音不会自动切换，反之亦然。最近已有针对多款声卡的补丁，尤其是 Framework 笔记本。自 FreeBSD 15.0 起，提供了 devd(8) 配置 `/etc/devd/snd.conf` 尝试自动解决该问题。基本思路是，当 snd_hda(4) 检测到插孔被（拔）插时，会发送 devd(8) 通知，而 `/etc/devd/snd.conf` 会通过 `virtual_oss(8)` 将声音重定向到合适的设备。此功能仍处于实验阶段，未来会根据反馈进一步优化。
* 为 `sound(4)` 添加 kqueue(2) 支持。


## 用户空间工具

可在各自的手册页中找到以下工具的示例和更多信息。

### sndctl(8)

`sndctl(8)` 用于列出和操作声卡设置，例如采样率、采样格式、比特完美模式（bit-perfect）和实时模式（realtime）设置等。它旨在替代 `/dev/sndstat`（实际上内部就是使用它）以及部分 `sound(4)` 的 sysctl，至少在大多数使用场景下如此。

```sh
$ sndctl
pcm3: <Realtek ALC295 (Analog 2.0+HP/2.0)> on hdaa1 (play/rec)
    name                = pcm3
    desc                = Realtek ALC295 (Analog 2.0+HP/2.0)
    status              = on hdaa1
    devnode             = dsp3
    from_user           = 0
    unit                = 3
    caps                = INPUT,MMAP,OUTPUT,REALTIME,TRIGGER
    bitperfect          = 0
    autoconv            = 1
    realtime            = 0
    play.format         = s16le:2.0
    play.rate           = 48000
    play.vchans         = 1
    play.min_rate       = 1
    play.max_rate       = 2016000
    play.min_chans      = 2
    play.max_chans      = 2
    play.formats        = s16le,s32le
    rec.rate            = 48000
    rec.format          = s16le:2.0
    rec.vchans          = 1
    rec.min_rate        = 1
    rec.max_rate        = 2016000
    rec.min_chans       = 2
    rec.max_chans       = 2
    rec.formats         = s16le,s32le
```

### mixer(8)

`mixer(8)` 用于控制音量、静音/取消静音、录音源选择以及默认设备设置。它在 FreeBSD 14.0 中被完全重写，提供了改进的界面和功能。

```sh
$ mixer
pcm3:mixer: <Realtek ALC295 (Analog 2.0+HP/2.0)> on hdaa1 (play/rec) (default)
    vol       = 0.75:0.75     pbk
    pcm       = 1.00:1.00     pbk
    rec       = 0.37:0.37     pbk
    ogain     = 1.00:1.00     pbk
    monitor   = 0.67:0.67     rec src
```

### virtual_oss(8)

`virtual_oss(8)` 是由已故 Hans Petter Selasky 编写的 OSS 强大音频服务器。多年来它一直作为 ports（audio/virtual_oss）提供，但自 FreeBSD 15.0 起已成为基本系统的一部分。它目前正在积极开发中，已有许多重要改进在进行和规划中。

如 [15.0 发布说明](https://cgit.freebsd.org/src/commit/?id=c457acb4ee821cf015930a94f52c3870786468a7) 所述，FreeBSD 15.0 之前的 `virtual_oss(8)` 用户可以卸载 port audio/virtual_oss 并使用基本系统版本。唯一需要注意的是，某些依赖第三方库的功能已移至独立 port，包括：

* sndio 后端支持：`audio/virtual_oss_sndio`
* 蓝牙后端支持：`audio/virtual_oss_bluetooth`
* virtual_equalizer(8)：`audio/virtual_oss_equalizer`

### mididump(1)

`mididump(1)` 是一款简单的工具，可用于实时打印指定设备的 MIDI 事件。这对于确认 MIDI 设备工作正常，以及按键是否正确映射，非常实用。

```sh
$ mididump /dev/umidi0.0
Note on                 channel=1, note=53 (F3, 174.61Hz), velocity=109
Note off                channel=1, note=53 (F3, 174.61Hz), velocity=127
Note on                 channel=1, note=55 (G3, 196.00Hz), velocity=100
Note off                channel=1, note=55 (G3, 196.00Hz), velocity=127
Pitch bend              channel=1, change=1
```

### beep(1)

`beep(1)`，顾名思义，用于播放蜂鸣声。这是验证声音是否正常工作的简单方法。

### 其他工具

除了前面提到的工具，声音子系统还提供了以下内容：

| 功能                 | 说明                                                                     | 文档            |
| ------------------ | ----------------------------------------------------------------------- | ------------- |
| mixer(3)           | 用于与 OSS mixer 交互的 C 语言库。                                                | man 3 mixer   |
| sndstat(4)         | 一个 nv(9) 接口，用于列出设备信息，以及注册用户态声音设备。被 `sndctl(8)` 和 `virtual_oss(8)` 内部使用。 | man 4 sndstat |
| hw.snd.*           | 全局 sysctl(8) 变量。                                                        | man 4 sound   |
| dev.pcm.*          | 设备特定的 sysctl(8) 变量。                                                     | man 4 sound   |
| 驱动特定的 sysctl(8) 变量 |                                                                         | 请参考相应驱动的手册页。  |

## FreeBSD 用于音乐制作？！

你可能会觉得在开玩笑，但实际上，近年来这个话题越来越多地被提及，我们也已经在一些会议上看到了相关演讲，例如：

* Goran Mekić, FOSDEM 2019
* Goran Mekić, EuroBSDCon 2022
* Charlie Li, BSDCan 2024
* Christos Margiolis, FreeBSD DevSummit 09/2024
* Christos Margiolis, BSDCan 2025
* Charlie Li, EuroBSDCon 2025

毫无疑问，FreeBSD 并不是音乐人或制作人在考虑音乐制作系统时的首选操作系统，但部分原因在于缺乏“宣传”。实际上，FreeBSD 提供了稳定、高速、可高度配置的声音子系统，拥有不断增长的开源数字音频工作站（DAW）、LV2 插件以及其他类型的音乐/制作软件，并且在 OSS 不适用或不理想的情况下，还可以兼容其他非原生声音子系统（如 ALSA、sndio、JACK、Pulseaudio、Pipewire 等）。

我真诚地认为，如果我们继续保持和开发声音子系统，移植和开发更多软件，并在实践中展示 FreeBSD 在音频和音乐制作方面的潜力，无论是在会议上还是在线，有朝一日，我们可能会看到 FreeBSD 在发烧友和音乐人群体中获得显著关注和认可。

## 报告与解决 Bug

所有软件都有可能出现 Bug，声音子系统也不例外。提供足够的信息非常必要，仅仅提交“我的机器上声音无法工作”这样的陈述，并不能真正帮助定位问题。通常附上以下命令的输出即可满足大部分情况的需求：

1. `uname -a`
2. `sndctl -v`
3. `mixer -a`
4. `sysctl hw.snd dev.pcm`，以及任何驱动相关的 sysctl（如果有的话）。
5. dmesg，在设置 `hw.snd.verbose=4` 并重现 bug 后获取。
6. 与重现 bug 的应用程序相关的日志（如果有）。

## 结论

希望本文能够帮助读者了解当前 FreeBSD 声音子系统的整体结构。后续如果能有更多人关注 FreeBSD 的声音系统，那再好不过了！大部分讨论集中在邮件列表 [freebsd-multimedia@FreeBSD.org](mailto:freebsd-multimedia@FreeBSD.org)，因此建议持续关注该列表。

---

Christos Margiolis 是来自希腊的独立开发商，同时也是 FreeBSD 源代码提交者（src committer）。
