---
title: CherrySprings项目
date: 2022-08-29 02:04:00 -0400
categories: [Project, CherrySprings]
tags: []
img_path: /assets/img/Project/CherrySprings/
---

CherrySprings项目取名于宾州的Cherry Springs State Park（当然现在只是随便取的名字）。我希望可以设计一个可以运行Linux且可以调度多个进程运行的多核系统，在这个过程中了解诸如处理器设计、内存模型与缓存一致性协议、总线与片上网络等多个主题的内容，也许是一个比较综合的大项目（也可能到最后都是大饼），因此也希望邀请志同道合的小伙伴一同加入。

# 设计选择

项目可能有如下设计选择。

## 开发语言与工具

比较流行的开发语言有Verilog、SystemVerilog、Chisel等。我计划采用Chisel语言，原因在于项目本身比较复杂，需要Chisel的语言特性进行类型检查、一部分的逻辑综合等功能，简化工作量，并可以复用Berkeley与Sifive开发的开源IP。开发工具方面，我倾向于尽可能使用已有的开源工具，如香山社区使用的[敏捷开发工具](https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/difftest/)。

## 处理器指令集与微架构

目前最活跃的开源指令集应该只有RISC-V可供选择，运行完整的Linux或其他类Unix系统需要至少支持RV64IMA + SU + Sv39 + Zicsr + Zifencei。

微架构方面，既然重点在多核上，应该尽量简化单个处理器的设计，避免内存模型方面出现过于复杂的问题。我个人认为采用五级流水线即可，我计划直接使用去年自己的处理器设计项目，在此基础上进一步修改。

## 内存模型与缓存一致性协议

RISC-V支持RVWMO内存模型，但是既然采用五级流水线的顺序结构，直接使用顺序一致性（SC）模型应该也是自然而然的，如果之后有机会开发乱序的处理器，也可以考虑RISC-V的TSO扩展，但应该是很久之后的事情了。

缓存一致性协议方面，可以采用MSI、MESI、MOSI、MOESI等多种状态机，但是也要考虑到状态数量的增加也会导致过渡状态数量大量增加，也许MSI或MESI是比较合理的选择。在基于总线嗅探和目录的协议中，或许目录协议会更容易实现且具有可扩展性，可以很好地在片上网络上进行实现。在缓存一致性方面可能还有很大的设计空间，待进一步补充。

另外，在L2缓存或LLC上，可以选择学习借鉴或复用Sifive的[block-inclusivecache-sifive](https://github.com/sifive/block-inclusivecache-sifive)或香山的[HuanCun](https://github.com/OpenXiangShan/HuanCun)等开源IP，不过都有一些缺点，例如可能会有一些增加了设计复杂度的功能（如为了支持non-blocking实现的MSHR），或者像后者可能为香山处理器额外增加了一些可能用不到的功能（如处理缓存别名问题的逻辑）。

## 总线与片上网络

有多种总线协议支持片上缓存一致性协议的实现，如AMBA的ACE、CHI和Sifive的TileLink协议。TileLink的spec只有100页不到，实现起来比ACE、CHI简单很多，容易理解，最大的好处在于可以复用rocket-chip的TL接口与相关模块设计，如Crossbar、AXI Bridge等（这也是另一个原因关于为什么我倾向于采用Chisel开发）。

片上网络方面，如果核心数较少，可以直接使用Crossbar连接，不过如果需要支持任意数量的核心，需要实现片上网络。这一工作的优先级我认为是最低的，因为多核不需要复杂的片上网络一样可以进行设计，且初期不太可能一下子设计8核16核等过于复杂的情况。片上网络可以自行设计，但也可以采用开源的NoC设计，如Berkeley即将在今年10月1日开源的Constellation片上网络生成器（Chisel Community Conference 2022上的报告）。

# 参考资料

有非常多的入门与进阶的课程、资料与教材可以参考。

## 课程

1. CMU 18-447 Intro to Computer Architecture & 18-740 Modern Computer Architecture.

1. Princeton ELE475 Computer Architecture.

1. [一生一芯](https://ysyx.org)项目相关的课程讲义，包括处理器设计讲义、南京大学计算机系统基础课程讲义。

## 资料与教材

1. *A Primer on Memory Consistency and Cache Coherence*.

1. 待补充