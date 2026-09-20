# Unix 与 Linux 概述

整理自 `LinuxPPT整理.docx` 的 2.1 到 2.4 节，典型题取自 `Linux整理（客观题）.docx`。这一章全是记忆型知识点，考客观题。

## 本章要点

- UNIX 诞生于 1969 年的 AT&T 贝尔实验室，作者 Ken Thompson 和 Dennis Ritchie。最初用汇编写，1973 年用 C 重写。
- UNIX 分成两个流派：System V 和 Berkeley UNIX（BSD）。两个重要标准是 SVID 和 POSIX。
- 了解 Linux 要记住 **2 个人 4 个 1**：Stallman、Torvalds；GNU、FSF、Copyleft、GPL。
- GNU 做出了编译器、调试器、bash 等工具，但没做成内核。Linus Torvalds 1991 年发布 Linux 内核，在 GPL 下发行。
- Linux = GNU 工具 + Linux 内核，所以也叫 GNU/Linux。
- Linux 的版本分内核版本和发行版本。
- 类 Unix 系统的特征：可移植、多用户、多任务、分级文件系统、与设备无关的输入输出、shell、系统工具。

## 1. UNIX 的历史

### 诞生

| 项目 | 内容 |
| --- | --- |
| 时间 | 1969 年 |
| 地点 | AT&T 贝尔实验室 |
| 人物 | Ken Thompson、Dennis Ritchie |
| 动机 | 玩游戏（课件原话） |
| 设计理念 | 简单易用 |

UNIX 最初用汇编语言开发。1971 到 1972 年 C 语言诞生，1973 年 Thompson 和 Ritchie 用 C 重写了 UNIX 的源代码，UNIX 和 C 从此绑在了一起。判断题 124 专门考这一点：UNIX 一开始是汇编写的。

### 成长

1983 年，Thompson 和 Ritchie 获得图灵奖。

课件用"十年磨一剑"来形容 UNIX 的成长。贝尔实验室内部专门成立了开发小组支持它，酝酿阶段从 1969 年到 1979 年，整整 10 年。等它作为产品面向用户时，已经在实验室内部被大量使用，在各个重要部门经过了检验。课件拿它和很多没完善就匆忙推向市场的商业软件做了对比。

### 演绎发展

1975 年，贝尔实验室以较低价格向教育机构提供 UNIX，并且以源代码形式发行。大学纷纷开设 UNIX 课程，学生毕业后又把 UNIX 带进了商业和工业领域。后来形成两个重要流派：

| 流派 | 代表系统 |
| --- | --- |
| UNIX System V | AIX、Solaris、HP-UX、IRIX |
| Berkeley UNIX | FreeBSD、NetBSD、OpenBSD |

### 标准化

20 世纪 80 年代 UNIX 版本剧增，版本之间差别越来越大。标准化的做法是对每种实现必须定义的各种限制进行说明。两个重要的 UNIX 标准：

1. SVID（System V Interface Definition），系统 V 接口定义
2. POSIX（Portable Operating System Interface），可移植操作系统接口

POSIX 的作用是对 UNIX 进行标准化（单选 3）。

## 2. Linux：2 个人 4 个 1

| 类别 | 内容 |
| --- | --- |
| 2 个人 | Richard Stallman、Linus Torvalds |
| 1 个项目 | GNU |
| 1 个组织 | FSF |
| 1 个理念 | Copyleft |
| 1 个许可证 | GPL |

### Richard Stallman 与 GNU 项目

Stallman 当时在 MIT 人工智能实验室（课件写作"MIT AI"），想"再开发个 UNIX"。他启动了 GNU 项目，发起自由软件运动，创建 FSF 组织，提出 Copyleft 理念，又制定了 GPL 协议。

GNU 是 GNU's Not UNIX 的缩写，1983 年 9 月 27 日公开发起。它的目标有四条：

1. 创建一个自由共享、可以被任何人修改的类 UNIX 操作系统
2. 与 UNIX 系统兼容
3. 不受 UNIX 名字和源代码私有权的限制
4. 能运行 UNIX 程序

GNU 项目的执行者是自由软件基金会（Free Software Foundation，FSF），1985 年成立。FSF 的目标是执行 GNU 计划，提供技术、法律和财政支持，开发更多自由软件。FSF 里的 Free 指自由，判断题 125 说它是"免费"，这是错的。

GNU 项目仿制了许多 UNIX 上的应用程序，重要的有：

| 工具 | 用途 |
| --- | --- |
| GCC | C/C++ 编译器 |
| GDB | 源代码级的程序调试工具 |
| GNU make | 软件构建工具 |
| bash | 命令解释器（shell） |
| GNU Emacs | 文本编辑器 |

这些软件质量比之前的 Unix 软件好，许多 Unix 系统也装了 GNU 软件，它们还被移植到了 Windows 和 Mac OS 上。**GNU 项目没能开发出操作系统内核**，原稿把这句加粗了。

### Copyright、Copyleft 与 GPL

发行大型软件需要合适的许可证。在 Copyleft 出现之前，已有的是 Copyright。

| 概念 | 含义 |
| --- | --- |
| Copyright（版权） | 软件的著作权和其他一切权利归作者私有。用户只有使用权，没有复制、修改后重新发布等权利 |
| Copyleft（著佐权） | 只有著作权归原作者，其他权利可以与任何人共享。使用者可以运行、复制、修改、发行修改后的程序，但不能在修改后的软件上添加限制，衍生作品必须以同等授权方式发布 |
| GPL（GNU General Public License，GNU 通用公共许可证） | Stallman 基于 Copyleft 提出的许可证 |

GPL 的两条规定：

1. 在 GPL 下发行的软件，任何人都可以运行、查看源代码、修改，并发行修改后的软件。
2. 重新发行软件的人不能剥夺软件的使用自由，也不能添加自己的限制。

GPL 的意义在于，用自由软件创建的新产品必须也在 GPL 下发行；以自由软件为基础修改后重新发行，也必须公开源代码。GNU 开发的工具都在 GPL 下发行。课件的口号是 "No GNU and GPL, No Linux!"

### Linus Torvalds 与 Linux 内核

- 赫尔辛基大学计算机系二年级学生，希望开发一个自由（开放源代码）的 Unix。
- 1991 年发布第一版内核，内核在 GPL 协议下发布。
- 参与开源运动后，Linux 内核更新速度极快。吉祥物是 tux。
- 目前 Linux 内核在 Linux 基金会的支持下由贡献者共同开发，Torvalds 对接受哪些更改、谁能成为维护者有最终决定权。

### 从自由软件运动到开源软件运动

自由软件运动蓬勃发展之后，并非所有自由软件用户和开发者都赞同它的目标。1998 年，自由软件阵营中的一部分成员分裂出来，以"开源"为名继续活动。此后开源理念不断发展，课件的说法是其声势与影响力早已超过自由软件运动。

常见的开源许可证有 Apache、BSD、MIT、Mozilla、木兰公共许可证等。开源软件可以选这些许可证中的任何一种，不必用 GPL（判断题 129）。

课件这一节引用了"吉大-华为 openEuler 开源实践课"的讲座材料，课程作业用的也是 openEuler。

### GNU/Linux

Linux = GNU 工具 + Linux 内核，也称 GNU/Linux，两者共同的基石是 GPL。

"Linux"一词有两种含义：

1. 操作系统内核
2. 基于 Linux 内核的操作系统，由内核、GNU 工具和其他应用三部分组成

### Linux 的版本

| | 内核版本 | 发行版本 |
| --- | --- | --- |
| 由谁发布 | Linux 内核社区统一发布，网址 https://www.kernel.org/ | 名称和版本号由发行版维护者决定 |
| 组成或分类 | 主版本号、次版本号、修订次数 | 商业发行版、社区发行版 |
| 例子 | | RHEL 7.3、8、9 由 Red Hat 公司发布；Ubuntu 22.04、23.04 由 Ubuntu 社区发布 |

## 3. 国产操作系统与开源创新

原稿这一节只整理了 CentOS 的变局：

- CentOS 曾是最流行的服务器开源发行版，和 RHEL 同源，同时免费、开源。课件称它在国内外广泛应用，市场份额超过 70%。
- CentOS 背后的支持者红帽公司被 IBM 收购。
- 2020 年红帽突然宣布 CentOS 终止既定的维护计划：CentOS 8 于 2021 年底结束支持，CentOS 7 按计划维护到生命周期结束（2024 年 6 月 30 日）。

## 4. 类 Unix 系统概要与特征

### 狭义 Unix 与广义 Unix

| | 狭义 Unix | 广义 Unix |
| --- | --- | --- |
| 写法 | UNIX | Unix |
| 含义 | 软件商标 | 一直被非正式使用，指任何类 UNIX（UNIX-like）操作系统，包括 Linux |
| 补充 | 商标先后由 AT&T、Novell、X/Open、The Open Group 持有 | |

### 七个特征

课件每一页都先说一句"Unix 有操作系统的共性，也有自己的特性"，然后分别讲下面几条。

| 特征 | 说明 |
| --- | --- |
| 可移植性 | C 语言保证了 Unix 的可移植性，从微机到巨型机都可以使用 |
| 多用户性 | 多个用户同时使用计算机，各自执行不同程序；系统提供安全机制隔离用户 |
| 多任务性 | 启动一个任务后可以继续执行其他任务；允许在前台和后台的多个任务间切换 |
| 分级文件系统 | 对数据和程序文件分组管理，便于查找文件 |
| 与设备独立的输入输出 | 打印机、终端、磁盘等所有设备都视为文件，像读写文件一样操作设备；命令输出可以重定向到任何设备或文件（`command > file`），命令输入可以重定向为从文件输入（`command < file`） |
| 用户界面 shell | shell 是命令解释器，控制用户与系统的交互，负责接收命令、输出结果。UNIX Shell 也是一种成熟的编程语言，shell 脚本是包含一系列命令的文本文件，能实现较复杂的功能 |
| 系统工具与系统服务 | UNIX 包括 100 多个系统工具程序（也称命令），是标准 UNIX 系统的组成部分，包括文本编辑格式化工具、文件操作工具、电子邮件工具、程序员工具等。"系统服务"原稿只有标题，没有展开 |

（更正：原稿 shell 一节的小标题作"shell脚本对数据和程序文件进行分组管理"，这句话是分级文件系统的说明，被误复制到这里，上表已按内容改正。）

## 易混对比

| 易混点 | 怎么区分 |
| --- | --- |
| FSF 的 Free | 自由，不是免费 |
| Copyright 与 Copyleft | Copyright 把一切权利留给作者；Copyleft 只保留著作权，其他权利共享，但不许再加限制 |
| GNU 与 Linux 内核 | GNU 做了工具，没做成内核；内核是 Torvalds 写的；两者合起来是 GNU/Linux |
| GPL 与开源许可证 | GPL 是开源许可证的一种，另外还有 Apache、BSD、MIT、Mozilla、木兰等 |
| System V 系与 BSD 系 | AIX、Solaris、HP-UX、IRIX 属于 System V；FreeBSD、NetBSD、OpenBSD 属于 BSD |
| SVID 与 POSIX | 系统 V 接口定义与可移植操作系统接口，都是 UNIX 标准 |
| UNIX 与 Unix | 狭义的商标与广义的类 UNIX 系统 |
| 内核版本与发行版本 | 内核版本由 kernel.org 统一发布；发行版本由各发行版自己定 |

## 典型题

完整题目和选项见 `10 客观题精编.md`。

- 单选 1：不属于类 UNIX 的是 Windows 10。
- 单选 4、5：Linux 内核和 GNU 工具都在 GPL 协议下发行。
- 单选 6：Linux 内核的开发者是 Linus Torvalds。
- 判断 124：UNIX 从一开始就是用 C 语言编写的。错，最初用汇编。
- 判断 126：修改 GPL 软件后再发布可以添加自己的限制。错。
- 判断 127：UNIX 被注册为软件商标，最早由 AT&T 持有。对。
- 判断 128：Solaris、OpenBSD、Linux 都是类 UNIX 系统。对。
