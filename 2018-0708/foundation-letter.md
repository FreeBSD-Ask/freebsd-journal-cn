# 基金会书信

                               ®
                     J O U R N A L                      来自董事会
  E d i t o r i a l B o a r d                           的信


       John Baldwin •  FreeBSD 开发者，
                         FreeBSD Core Team 成员，FreeBSD
       Brooks Davis •   SRI International 高级计算机科学家，剑桥         读者朋友们也许并非都知道，
                                 大学访问学者，FreeBSD
                                 Core Team 成员。               Berkeley Software Distribution（BSD）是 FreeBSD
       Bryan Drewery •   EMC Isilon 高级软件工程师，
                       FreeBSD Portmgr Team 成员，                  的前身，它最初本质上是一个科研项目。自
                       FreeBSD Committer。
                                              FreeBSD 早期在加州大学伯克利分校诞生以来，
                          FreeBSD Foundation 创始         人，                  我们的系统就常常被用于推动科学进步，无论
                          FreeBSD Foundation
                             董事会理事，Facebook 软件工程师。    是计算机科学本身，还是支持物理、化学、生物
         Daichi Goto •   BSD Consulting Inc.（东京）         等非计算机科学领域。许多如今已成为主流的技术，
                       董事。
                                              例如虚拟内存和 TCP/IP 协议，最初都是在
                                              BSD 上开展的实验。早在"机器学习"这个词
          Dru Lavigne •   iXsystems 存储工程
                              总监，BSD Hacks 和              出现之前，全球各大学的实验室以及美国和其他
                          The Best of FreeBSD Basics 作者。  国家的国家实验室就已使用 BSD 来存储和处理
          Ed Maste •   项目开发总监，        科学数据。在本期 FreeBSD Journal 中，
                         FreeBSD Foundation。
         Kirk McKusick •   FreeBSD Foundation           我们将探讨现代 FreeBSD 在科学计算中的应用，
                            董事会财务主管，
                            The Design and Implementation            收录 Jason Bacon、Benedict
                         书系主要作者。
George V. Neville-Neil •   FreeBSD Foundation           Reuschling 和 Johannes Dieterich 撰写的文章。
                            董事会主席，
                            The Design and                本期另有关于存储的两篇文章，分别是 Jim Harris 与 Warner
                          Implementation of the FreeBSD
                             Operating System 合著者。    Losh 撰写的 NVM Express 文章——他们编写了 FreeBSD 中
            Philip Paeps •  FreeBSD Foundation            处理这项新型、持久化、高速存储技术的大量代码——以及
                            董事会秘书，FreeBSD Committer，
                          独立顾问。              Rick Macklem 介绍的 pNFS，它允许网络文件系统（NFS）在
           Hiroki Sato •   FreeBSD Foundation           一组服务器上并行运作，这正是科学计算
                          董事会理事，Asia
                        BSDCon 主席，FreeBSD        社区中的常见用例。
                          Core Team 成员，东京工业
                            科技大学助理教授。
      Benedict Reuschling •  FreeBSD          最后，本月照例带来专栏文章：Steven Kreuzer 在 svn Update
                          Foundation 董事会
                          副主席，FreeBSD Doc-       中讨论 pNFS 提交到 FreeBSD 一事，Michael Lucas 负责来信
          umentation Committer，         专栏，Dru Lavigne 通过活动日历让你了解未来几个月
                          FreeBSD Core Team 成员。
 Robert N. M. Watson •   FreeBSD Foundation           即将到来的全部活动。
                            董事会理事，TrustedBSD
                                项目创始人，剑桥大学高级讲师。                Journal 全体同仁希望，如果你正
                                         处于夏日炎炎的地方，不妨倒上一杯最令人放松的饮料，坐在 S&W PUBLISHING LLC        树荫下，享受这份 FreeBSD 精粹的清凉合集。   P O B O X 4 0 8 , B E L F A S T , M A I N E 0 4 9 1 5
                                             Publisher •  Walter Andrzejewski
                          walter@freebsdjournal.com
       Editor-at-Large • James Maurer
                          jmaurer@freebsdjournal.com
                                                            George Neville-Neil
     Copy Editor •  Annaliese Jakimides
          Art Director • Dianne M. Kischitz
                          dianne@freebsdjournal.com
     Advertising Sales •  Walter Andrzejewski
                                致电 888/290-9469
                                walter@freebsdjournal.com

      FreeBSD Journal（ISBN: 978-0-615-88479-0）每年出版 6 期
       （1/2 月、3/4 月、5/6 月、7/8 月、
                9/10 月、11/12 月）。                  FreeBSD Foundation 出版，
            5757 Central Ave., Suite 201, Boulder, CO 80301
                   电话: 720/207-5142 • 传真: 720/222-2350
                     邮箱: info@freebsdfoundation.org
       Copyright © 2018 FreeBSD Foundation。保留所有权利。未经出版商书面许可，不得
           全部或部分复制本杂志。                                                        July/Aug 2018 3
