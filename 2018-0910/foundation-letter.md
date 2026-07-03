# 基金会书信

                               ®
                     J O U R N A L                      来自董事会
  E d i t o r i a l B o a r d                           的信


       John Baldwin •  FreeBSD 开发者，FreeBSD
                         Core Team 成员，FreeBSD
                         Journal 编辑委员会联合主席。
       Brooks Davis •   SRI International 高级计算机科学家，
                                 剑桥大学访问学者，FreeBSD
                                 Core Team 成员。
       Bryan Drewery •   EMC Isilon 高级软件工程师，
                       FreeBSD Portmgr Team 成员，
                       FreeBSD Committer。
        Justin Gibbs •  FreeBSD Foundation 创始人，
                          FreeBSD Foundation
                             董事会理事，Facebook 软件工程师。
         Daichi Goto •   BSD Consulting Inc.（东京）
                       董事。
        Joseph Kong •   Dell EMC 高级软件工程师，
                       《FreeBSD Device Drivers》作者。
       Steven Kreuzer •  FreeBSD Ports Team 成员。
        Dru Lavigne •   iXsystems 存储工程
                              总监，《BSD Hacks》和
                          《The Best of FreeBSD Basics》作者。
     Michael W Lucas •  《Absolute FreeBSD》作者。
          Ed Maste •   项目开发总监，
                         FreeBSD Foundation。
         Kirk McKusick •   FreeBSD Foundation
                            董事会财务主管，
                            《The Design and Implementation》
                         书系主要作者。
George V. Neville-Neil •   FreeBSD Foundation
                            董事会主席，
                            《The Design and
                          Implementation of the FreeBSD
                             Operating System》合著者。
            Philip Paeps •  FreeBSD Foundation
                            董事会秘书，FreeBSD Committer，
                          独立顾问。
           Hiroki Sato •   FreeBSD Foundation
                          董事会理事，Asia
                        BSDCon 主席，FreeBSD
                          Core Team 成员，东京工业
                            科技大学助理教授。
      Benedict Reuschling •  FreeBSD
                          Foundation 董事会
                          副主席，FreeBSD Doc-
          umentation Committer，
                          FreeBSD Core Team 成员。
 Robert N. M. Watson •   FreeBSD Foundation
                            董事会理事，TrustedBSD
                                项目创始人，剑桥大学高级讲师。

欢迎阅读 FreeBSD Journal 2018 年秋季网络专刊。我们的第一篇网络文章由 Jonathan Looney 撰写，涵盖了我一直非常感兴趣的测试主题。软件测试已经很难，测试网络代码——实际上就是测试分布式系统——难度更大。Netflix 在其缓存服务器中使用 FreeBSD，他们对在 FreeBSD TCP 栈中所做变更的性能和正确性都很关注，因此良好的测试系统的严谨性对他们和整个 FreeBSD 来说都非常受欢迎。

Ayaka Koshibe 贡献了一篇关于软件定义网络（SDN）和 Mininet 的文章，Mininet 是一个知名的 SDN 协议模拟器。过去几年她一直在会议上展示围绕这个主题的工作，我很高兴我们说服她为 Journal 撰写一篇关于这个主题的文章。

第三篇文章《Internetworking with FreeBSD》由 humble Editor in Chief 贡献，所以我在这里就不剧透了。

为了完善这个专题，我们还有一篇 Matt Macy 撰写的文章，讲述如何在现代服务器硬件上扩展 FreeBSD 的性能。虽然 FreeBSD 部署在许多类型的硬件和许多环境中，但在具有许多 CPU 核心和大量内存的高端服务器上的性能，仍然是 FreeBSD 整体成功的关键。

请查看我们所有的专栏，特别是新的读者来信专栏，Michael Lucas 以精妙而诙谐的笔触回答你的来信。

                                                      George Neville-Neil
                                                      FreeBSD Foundation 董事会主席

 S&W PUBLISHING LLC        P O B O X 4 0 8 , B E L F A S T , M A I N E 0 4 9 1 5
             Publisher •  Walter Andrzejewski
                          walter@freebsdjournal.com
       Editor-at-Large • James Maurer
                          jmaurer@freebsdjournal.com
        Copy Editor •  Annaliese Jakimides
           Art Director • Dianne M. Kischitz
                          dianne@freebsdjournal.com
     Advertising Sales •  Walter Andrzejewski
                          walter@freebsdjournal.com
                                致电 888/290-9469

      FreeBSD Journal（ISBN: 978-0-615-88479-0）每年出版 6 期
       （1/2 月、3/4 月、5/6 月、7/8 月、                9/10 月、11/12 月）。
                  FreeBSD Foundation 出版，
            5757 Central Ave., Suite 201, Boulder, CO 80301
                   电话: 720/207-5142 • 传真: 720/222-2350
                     邮箱: info@freebsdfoundation.org
       Copyright © 2018 FreeBSD Foundation。保留所有权利。未经出版商书面许可，不得
           全部或部分复制本杂志。                                                        Sept/Oct 2018 3
