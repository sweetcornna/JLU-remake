# 第 2 章 线程级并行：OpenMP 编程

整理自 `并行与分布式笔记.pdf` 第 22–67 页（第 2 章 线程级并行：OpenMP 编程）。原稿的图片链接已失效，需要看图的地方另作说明。

## 本章要点

1. 单核性能提升困难，加核心数就是线程级并行（TLP）。多处理机共用一个地址空间，按访存是否一致分 UMA 和 NUMA。
2. 多线程共用数据会引出竞态、数据竞争、指令重排序、缓存与互联网络造成的可见性问题，解决手段都是同步。
3. OpenMP 是共享内存下的工业标准，基于 FORK-JOIN 模型，由编译制导、运行时库函数、环境变量三部分组成，支持把串行程序增量并行化。
4. 编译制导的写法是 `#pragma omp 指令 子句`，指令分并行域结构、任务划分结构、同步结构三类。
5. 数据作用域子句（`private` / `firstprivate` / `lastprivate` / `shared` / `default` / `reduction`）和调度子句 `schedule` 是考试重点，要能看代码说输出。
6. 循环能不能并行化，取决于有没有跨迭代依赖（流依赖、反依赖、写依赖）。

---

## 1 共享内存模型与线程基础

### 1.1 访存模型：UMA 和 NUMA

一个处理器不够用时，增强单核性能很难，最直观的办法是加核心数量，这就是**线程级并行（TLP，Thread Level Parallelism）**。多个处理机共用一块内存的结构叫**多处理机**，也叫**共享内存模型**。

从存储角度分两类：

| 模型 | 全称 | 含义 | 典型结构 |
| --- | --- | --- | --- |
| UMA | Uniform Memory Access | 一致访存，所有处理器访问内存的开销一致，可以各有私有 cache | 集中式共享存储器 SMP |
| NUMA | Non Uniform Memory Access | 非一致访存，处理器有各自的存储器，访问本地和远程的开销不同 | 分布式共享存储器 DSM |

两者都有**统一的地址空间**，都属于共享内存模型，本章只讨论这种情况。

（原稿此处有 SMP 和 DSM 的结构图，PDF 中图片仍在但笔记里不便复制：SMP 是多个处理器各带一或多级私有缓存，往下接共享缓存、主存储器和 I/O 系统；DSM 是每个处理器 P 各带自己的存储器 M 和 I/O，所有节点挂在一张互连网络上。）

### 1.2 CPU 相关的几个术语

这几个词考试爱拿来辨析：

| 术语 | 含义 |
| --- | --- |
| Core | 一个计算核心 |
| Die | 一颗晶片，可以有一或多个 Core |
| Package | 一个封装好的处理器芯片，可以有一或多个 Die |
| Socket | 主板上的一个插槽，对应一个 Package |
| CPU / Processor | 多重含义。描述硬件时一般指一个物理处理器（一个 Package）；在操作系统里一般指一个逻辑处理器（一个 Core） |

举例：

- 一块消费级主板上往往只有 1 颗 CPU（package），里面可能有多个 core，共用主存，所以属于 **UMA**。
- 企业级主板可以同时插多颗 CPU，每颗 CPU 有自己的内存控制器，CPU 之间用外部总线连接。所有 CPU 共用一个内存地址空间，但各自管着属于自己的内存，所以属于 **NUMA**。

### 1.3 线程与进程

- **进程**：一个正在执行程序的实例，包括程序计数器、寄存器和变量的当前值。更独立，有独自的地址空间。
- **线程**：轻量级进程，共享地址空间，但各有一套堆栈。

在操作系统里，线程和进程可以在单一处理器上由内核实现，靠的是对多道程序的快速切换（分配时间片）。这是软件方式的伪并行，叫**并发（concurrency）**，并不会让程序跑得更快。本章说的"线程级并行"指的是多处理器上的硬件级线程并行。换个说法：**线程是在一个处理机上运行着的一段程序**。

并发与并行的区别（原稿用图说明）：并发是 core 0 一个核上轮流跑 A0、B0、A1、B1、A2、B2；并行是 core 0 跑 A0、A1、A2，core 1 同时跑 B0、B1、B2，对应 thread 0 和 thread 1。

### 1.4 并行计算编程模型

多线程协同计算需要专门的编程方式。多处理器（SMP、DSM）和多机（MPP、COW）环境下，数据作为计算资源的共用是主要矛盾，数据要靠进程（线程）交互来分享，所以按**进程交互方式**把并行编程模型分成 3 类：

1. **隐式交互**：完全由编译器实现，本课不展开。
2. **共享变量**：英特尔 Cilk、OpenMP。
3. **消息传递**：MPI。

共享变量和消息传递的对比：

| | 共享变量 | 消息传递 |
| --- | --- | --- |
| 适合的体系结构 | SMP、DSM | MPP、COW（不止） |
| 地址空间 | 单一地址空间 | 多地址空间 |
| 通信方式 | 隐式通信 | 显式通信 |
| 集群中的位置 | 一般用于一个节点的多个核上 | 一般用于集群中的多个节点上 |

---

## 2 共享变量编程的隐藏问题

多处理机上用多线程，主要问题是线程之间怎么协调，从而正确操作计算的对象，也就是数据。数据从硬件角度看就是内存，属于被计算使用的资源。多个核心共用这些资源（共享变量）时，就会因为对数据的竞争（谁先谁后）而产生奇怪的问题，归结成一句话：**多个线程访问一个变量时，会发生什么？**

### 2.1 CASE1：最简单的自加

```c
// thread 0            // thread 1
// init: x = 0         // init: x = 0

x = x + 1;             x = x + 1;
```

### 2.2 CASE2：改写成汇编看交错

把代码改写为汇编（`x` 初始化为 0，`a1` 中为 `x` 的内存地址），问这两段代码有几种可能的执行顺序：

```asm
# thread 0             # thread 1

ld    t0, (a1)         ld    t1, (a1)
addi  t0, t0, 1        addi  t1, t1, 1
sd    t0, (a1)         sd    t1, (a1)
```

我们期待的是执行了两次自加，`x` 变成 2，对应这种交错：

| thread 0 | thread 1 | 备注 |
| --- | --- | --- |
| `ld t0, (a1)` | | t0 = x = 0 |
| `addi t0, t0, 1` | | t0 = 1 |
| `sd t0, (a1)` | | x = t0 = 1 |
| | `ld t1, (a1)` | t1 = x = 1 |
| | `addi t1, t1, 1` | t1 = 2 |
| | `sd t1, (a1)` | x = t1 = 2 |

但也可能出现这种情况：

| thread 0 | thread 1 | 备注 |
| --- | --- | --- |
| `ld t0, (a1)` | | t0 = x = 0 |
| `addi t0, t0, 1` | | t0 = 1 |
| | `ld t1, (a1)` | t1 = x = 0 |
| | `addi t1, t1, 1` | t1 = 1 |
| `sd t0, (a1)` | | x = t0 = 1 |
| | `sd t1, (a1)` | x = t1 = 1 |

### 2.3 竞态与数据竞争

上面的问题叫**竞态（race condition）**：并发程序中，多个线程或进程执行时，因为执行顺序不同，程序行为不可预测，产生错误或不一致的结果。

- 当程序的正确运行**依赖于各线程的特定时序**时（执行顺序不同会产生不同结果），就会出现竞态。
- 这种依赖往往发生在多个线程对同一个资源的竞争中，尤其是其中存在修改资源状态的操作（写操作）。在内存上这被称为**数据竞争（data race）**。
- 数据竞争的解决方法：利用**同步**，确保操作是**原子性**的，从而对操作进行排序，把资源与操作保护起来。工具有锁、信号量等。
- 注意："排序"并不意味着顺序是确定的，所以解决了数据竞争并不意味着完全解决了竞态。

### 2.4 指令重排序

再看一个例子：

```c
// thread 0                   // thread 1
// init: ready=0, z=0         // init: ready=0, z=0

z = 3;        // (a)          while (!ready);   // (c)
ready = 1;    // (b)          z++;              // (d)
```

设想中，由于 $b \to c \Rightarrow a \to d$，`z` 的最终结果为 4。但现在的超标量处理器都支持**指令重排序**。同时在多个处理器之间，由于存在 cache 和互联网络，你无法保证一个线程的写操作何时可以被其他线程看见，所以也可能发生**内存上的重排序**。

对 `z` 的数据依赖发生在不同线程中，处理器检测不到，因此能主动重排序：

```c
// thread 0                          // thread 1
// 假设 z 脱靶（cache miss），ready 命中

z = 3;      // cache miss, can't issue     while (!ready);
ready = 1;  // issue first                 z++;

// 结果：z = 0 -> z = 1 -> z = 3
```

`ready = 1` 先发射出去，thread 1 看到 `ready` 已置位就做 `z++`（此时 z 还是 0，变成 1），之后 thread 0 的 `z = 3` 才落地，`z` 最终是 3 而不是 4。

### 2.5 缓存和互联网络带来的问题

**缓存导致的问题。** 图中有三个处理器，每个处理器有自己的缓存和内存，通过互连网络（Interconnection Network）与共享内存相连。各处理器执行操作时会更新自身缓存内容，但这些更新不会立即同步到共享内存，可能导致不一致的状态：

- 左侧处理器执行 `A = 1`，把值 1 写入缓存，尚未同步到共享内存。
- 中间处理器在等待条件 `A == 1` 成立后执行 `B = 1`，此时 `A = 1` 的更新未同步到共享内存，中间处理器会误认为 `A` 的值仍为 0。
- 右侧处理器在 `B == 1` 条件满足后执行操作，但由于缓存和内存不一致，仍然读到旧的 `A = 0`，出现 `Oops!` 错误。

**互联网络导致的问题。** 图中有多个处理器，每个处理器可以访问内存，通过互联网络与其他内存模块连接：

- 左侧处理器执行 `A = 1` 和 `Ready = 1`，把值写入缓存或本地内存，但这些写操作不会立刻传播到共享内存，或在网络传输中发生延迟。
- 右侧处理器等待 `Ready == 1` 成立，认为 `Ready` 已更新，但由于互联网络延迟，可能读到 `A = 0` 的旧值。
- `Ready` 的更新比 `A` 更早到达，导致处理器错误地认为 `A` 已更新。

一个线程内的几条指令之间（在对于另一个线程的可见性上）出现了难以预见的重排序。这种重排序对单线程没有影响，对多线程就出问题了。解决方法同样是同步（锁、栅栏等技术）。

---

## 3 OpenMP 概述

### 3.1 是什么

OpenMP API（Open Multi-Processing Application Programming Interface，开放多处理应用编程接口）是共享存储体系结构上的一个编程模型，支持 Unix/Linux/Win 等多种平台。它简单、可移植性好、可扩展，是**共享存储系统编程的工业标准**。

OpenMP 是一个工业标准规范的实现，包含三类基本 API：**编译制导（Compiler Directive）**、**运行库例程（Runtime Library）**和**环境变量（Environment Variables）**。用户用这些机制与编译器和运行时系统交互，控制并行程序的行为。

OpenMP 支持用户对程序添加制导，从而把串行程序**增量并行化（Incremental Parallelization）**。

几句话概括：

- 一种基于 fork-join 模型的多线程并行编程 API。
- 在 C、C++、Fortran 等语言上提供接口。
- 主要适用于共享内存结构的多处理机。

### 3.2 FORK-JOIN 模型

OpenMP 是基于线程的并行编程模型，一个共享的进程由多个线程组成。主线程（MASTER THREAD）串行执行，直到编译制导的并行域（PARALLEL REGION）出现。

- 把程序划分为许多段的串行部分或并行部分。
- 串行部分使用单个线程处理。
- 并行部分的各个子任务使用多个线程分而治之。
- 并行任务拆分出的子任务可以继续拆分。
- 使用 **fork** 进入并行（子任务）部分，使用 **join** 回到串行（父任务）部分。
- 一般情况下，串行部分（父任务）所在线程被视为主线程。该线程在 fork 时会被分配给其中一个子任务；join 时所有其他线程被移除，主线程继续分配给下一段串行部分使用。

### 3.3 OpenMP 存储模型

OpenMP 把存储分为 **shared** 和 **private** 两类：

- shared 变量在各个线程之间共享。对它操作时要注意竞态和重排序问题，合理使用同步。
- private 变量是各线程独有的，互不影响。

原稿配图：thread 0 和 thread 1 各自连着一块 Private，中间共同连着一块 Shared。

### 3.4 OpenMP 体系结构

原稿画成四层结构，从上到下：

| 层 | 内容 |
| --- | --- |
| 最上层 | 应用（左）、用户（右） |
| 第二层 | 编译制导（面向应用）、环境变量（面向用户） |
| 第三层 | 运行库例程 |
| 最下层 | OS 线程 |

应用通过编译制导、用户通过环境变量，都要落到运行库例程上，运行库例程再去用 OS 线程。

### 3.5 第一个程序

```c
/* 用 OpenMP/C 编写 Hello World */
#include <omp.h>

int main(int argc, char *argv[])
{
    int nthreads, tid;
    omp_set_num_threads(4);         // 设置线程数为 4

    /* Fork a team of threads */
    #pragma omp parallel private(tid)
    {
        tid = omp_get_thread_num(); // 获取当前线程 ID
        printf("Hello, world from OpenMP thread %d\n", tid);
        if (tid == 0)               // 仅主线程执行
        {
            nthreads = omp_get_num_threads(); // 获取线程总数
        }
    }
    printf("Number of threads %d\n", nthreads);
    return 0;
}
```

（更正：原文作 `#pragma omp paralled private(tid)`，拼写应为 `parallel`。）

代码解析：

1. `omp_set_num_threads(4)`：设置线程数为 4。
2. `#pragma omp parallel private(tid)`：创建一个并行区域，每个线程有独立的 `tid` 变量（线程私有）。
3. `omp_get_thread_num()`：获取线程 ID。
4. `omp_get_num_threads()`：获取线程总数。

编译与运行：

```bash
gcc -fopenmp -o hello.o hello.c
./hello.o
```

（更正：原文作 `gcc–fopenmp–o hello.o hello.c`，连字符被排版成了长横线。）

输出：

```text
Hello, world from OpenMP thread 0
Hello, world from OpenMP thread 3
Hello, world from OpenMP thread 1
Hello, world from OpenMP thread 2
Number of threads 4
```

前四行顺序不固定，每次运行都可能不一样。

---

## 4 环境变量与运行时库函数

OpenMP 的语法主要包含三个部分：环境变量、运行时库函数、编译制导。编译制导又分并行域结构、任务划分结构、同步结构。

### 4.1 环境变量

改环境变量的值就能控制 OpenMP 的行为。

| 环境变量 | 作用 | 取值 |
| --- | --- | --- |
| `OMP_SCHEDULE` | 只能用于 `parallel for` 和 `for`，决定循环中各个迭代的调度方式 | `static`（静态分配任务，线程按顺序划分工作）、`dynamic`（动态分配，任务完成后继续领取新任务） |
| `OMP_NUM_THREADS` | 执行中所能使用的最大线程数量 | 整数 |
| `OMP_DYNAMIC` | 是否能够动态设定并行域执行部分的线程数 | `true` 根据系统负载自动调整；`false` 固定线程数 |
| `OMP_NESTED` | 是否允许嵌套并行 | `true` 允许在并行区域内再开启并行；`false` 禁止嵌套并行 |

（更正：原文把最后一个写成 `OMP_NETSTED`，正确拼写是 `OMP_NESTED`。）

### 4.2 运行时库函数

OpenMP 标准定义了一套库函数。C/C++ 程序开头需要 `#include <omp.h>`。

```c
/* 设置线程数 */
#include <omp.h>
omp_set_num_threads(4);
```

常用函数：

| 函数 | 作用 |
| --- | --- |
| `omp_get_num_procs()` | 返回运行本线程的处理器个数，可用于动态调整线程数以适应硬件资源 |
| `omp_set_num_threads(nthreads)` | 设置并行代码使用的线程数 |
| `omp_get_num_threads()` | 返回当前并行区域中活动线程数 |
| `omp_get_max_threads()` | 返回程序可用的最大线程数 |
| `omp_get_thread_num()` | 返回线程 ID，从 0 开始 |

### 4.3 互斥锁函数

| 函数 | 作用 |
| --- | --- |
| `omp_init_lock` | 初始化一个锁 |
| `omp_set_lock` | 上锁，获取锁；锁被其他线程占用时线程阻塞 |
| `omp_test_lock` | 尝试获取锁，不会阻塞 |
| `omp_unset_lock` | 释放锁，要和 `omp_set_lock` 配对使用 |
| `omp_destroy_lock` | 销毁一个锁，`omp_init_lock` 的配对操作 |

```c
#include <stdio.h>
#include <omp.h>

static omp_lock_t lock;             // 声明锁变量

int main()
{
    int i;
    omp_init_lock(&lock);           // 初始化锁

    #pragma omp parallel for        // 并行化循环，每个线程执行部分循环体
    for (i = 0; i < 5; i++) {
        omp_set_lock(&lock);
        printf("%d +\n", omp_get_thread_num());   // 加号表示已获取锁
        printf("%d -\n", omp_get_thread_num());   // 减号表示即将释放锁
        omp_unset_lock(&lock);      // 释放锁
    }

    omp_destroy_lock(&lock);        // 销毁锁
    return 0;
}
```

输出（线程先后顺序不一定，但同一线程的 `+` 和 `-` 一定挨在一起）：

```text
1 +
1 -
2 +
2 -
0 +
0 -
4 +
4 -
3 +
3 -
```

这个例子的看点是：加锁之后，`+` 和 `-` 之间不会插进别的线程的输出。

---

## 5 编译制导的语法

### 5.1 基本格式

编译制导是对程序设计语言的扩展，OpenMP 通过给串行程序加制导语句实现并行化。语法是：

```c
#pragma omp 指令 子句
```

完整格式：

```c
#pragma omp parallel [clause, clause, ...] newline
```

四个组成部分：

1. `#pragma omp`：OpenMP 的固定前缀，所有编译指导语句都以此开头。
2. `directive-name`：具体的制导指令，例如 `parallel`、`for`、`critical`。制导指令前缀和子句之间必须有一个正确的制导指令。
3. `[clause, ...]`：可选子句，提供额外的修饰参数，例如 `private`、`shared`，相当于制导指令的修饰参数。没有其他约束条件时子句可以无序，也可以任意选择，这一部分也可以没有。
4. `newline`：以换行符结尾，表明这条制导语句终止。

**子句（Clause）**是编译指导语句的修饰参数，用来控制线程的行为或指定变量的作用域。例如 `private(tid)` 声明 `tid` 为线程私有变量，`shared(nthreads)` 声明 `nthreads` 为共享变量。

### 5.2 静态范围与动态范围

**静态范围（Lexical Extent）**：从语句位置开始到代码结构块结束的区域。编译指导语句会被包含在一个单一的静态代码块中，不能超出块范围。

```c
#pragma omp parallel
{
    // 此处为并行区域的静态范围
    #pragma omp for
    for (int i = 0; i < 10; i++) {
        // 代码执行
    }
}
// 超出并行区域，不再属于静态范围
```

**动态范围**：包括静态范围以及所有嵌套的函数调用。动态范围可以跨越多个代码块，会随函数调用的嵌套层次扩展。

```c
#pragma omp parallel
{
    sub1();          // 动态范围包含该函数的调用
}

void sub1() {
    #pragma omp critical
    {
        // 此处属于并行区域的动态范围
    }
}
```

### 5.3 编译指导语句分类

| 类别 | 指令 | 一句话说明 |
| --- | --- | --- |
| 并行结构 | `parallel` | 说明代码段将被多个线程并行执行 |
| 任务划分结构 | `for` | 用于 for 循环之前，把循环分配到多个线程并行执行，必须保证每次循环之间无相关性 |
| 任务划分结构 | `sections` | 划分多个代码段，每个代码段由不同线程执行 |
| 任务划分结构 | `single` | 指定代码段只由一个线程执行 |
| 同步结构 | `critical` | 保护临界区代码，同一时间只有一个线程可以访问 |
| 同步结构 | `barrier` | 设置线程同步点，所有线程执行到该点后才能继续 |
| 同步结构 | `atomic` | 对单一内存位置的访问进行原子操作，避免数据竞争 |
| 同步结构 | `master` | 指定代码段仅由主线程执行 |
| 同步结构 | `ordered` | 保证并行循环中的操作按顺序执行 |

各自的最小示例：

```c
#pragma omp parallel
printf("Hello from thread %d\n", omp_get_thread_num());
```

```c
#pragma omp parallel for
for (int i = 0; i < 10; i++) {
    printf("Thread %d processes iteration %d\n", omp_get_thread_num(), i);
}
```

```c
#pragma omp parallel sections
{
    #pragma omp section
    { printf("Section 1\n"); }
    #pragma omp section
    { printf("Section 2\n"); }
}
```

```c
#pragma omp single
printf("This is executed by a single thread\n");
```

```c
#pragma omp critical
{
    printf("Thread %d in critical section\n", omp_get_thread_num());
}
```

```c
#pragma omp parallel
{
    printf("Before barrier\n");
    #pragma omp barrier
    printf("After barrier\n");
}
```

```c
#pragma omp master
printf("This is executed by the master thread\n");
```

```c
#pragma omp parallel for ordered
for (int i = 0; i < 10; i++) {
    #pragma omp ordered
    printf("Iteration %d\n", i);
}
```

`atomic` 的示例原稿给了一段完整程序，结果很值得算一遍：

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int data = 1;
    #pragma omp parallel num_threads(4) shared(data) default(none)
    /*
      `#pragma omp parallel` 创建一个并行区域，启动多个线程，同时执行其后的代码块
      `num_threads(4)`：并行区域中创建 4 个线程。不显式指定时，OpenMP 默认按系统
                        硬件核心数决定线程数
      `shared(data)`：指定 data 为共享变量，所有线程访问同一个内存地址的 data。
                      默认情况下 OpenMP 中的变量为共享变量，除非显式声明为 private
      `default(none)`：强制要求所有变量必须显式声明作用域，防止意外的变量共享或
                       私有化，未显式声明的变量会引发编译错误
    */
    {
        #pragma omp atomic   // 保证同一时刻只有一个线程更新 data
        data += data * 2;
    }
    printf("data=%d\n", data);
    return 0;
}
```

输出：

```text
data=81
```

算法：每个线程把 `data` 变成原来的 3 倍，4 个线程被原子操作串起来依次执行，$1 \times 3^4 = 81$。

（更正：原文该代码块最前面多了一行游离的 `#pragma omp parallel for`，是上一个示例残留，删掉不影响。另外严格按 OpenMP 规范，`atomic` 要求赋值右侧不含被更新的变量本身，`data += data * 2` 属于教学示例写法。）

---

## 6 并行域结构

### 6.1 格式和子句

一个并行域就是一个能被多个线程并行执行的程序块，是最基本的 OpenMP 并行结构。

```c
#pragma omp parallel [clause, clause, ...] newline
{
    BLOCK
}
```

可用的 clause：

| 子句 | 作用 |
| --- | --- |
| `if (scalar_expression)` | 条件并行 |
| `private (list)` | 指定线程私有变量 |
| `shared (list)` | 指定线程共享变量 |
| `num_threads(n)` | 设置并行区域中线程的数量 |
| `default (shared \| none)` | 指定默认作用域 |
| `firstprivate (list)` | 私有变量并继承域外初值 |
| `reduction (operator: list)` | 对变量进行归约操作 |
| `copyin(list)` | 用主线程的值初始化 threadprivate 变量 |

要点：

- `#pragma omp parallel` 指定接下来的代码块为并行区域，默认所有线程同时执行区域内的代码。
- 并行域**结尾有一个隐式同步（barrier）**。
- 子句用来说明并行域的附加信息，C/C++ 中子句之间用空格分开。

### 6.2 执行流程

1. Master 线程启动并行区域。程序从主线程进入并行域，主线程创建多个工作线程（Worker threads），一起执行并行区域的代码块。
2. 工作线程并行执行代码块（BLOCK）内的代码，具体任务分配可以通过调度策略控制。
3. 同步点（Barrier）。并行区域结束后所有线程同步，等所有线程完成任务再继续执行后续代码。
4. 返回主线程。工作线程完成任务后销毁，主线程继续执行。

### 6.3 if 子句

`#pragma omp parallel if(condition)` 控制并行域是否以并行方式执行。condition 为真（非零）时代码以多线程并行执行；为假（零）时代码以单线程方式（主线程）执行。

```c
#pragma omp parallel if(n > thr) shared(n) private(i, tid)
/*
  如果 n > thr，并行域以多线程方式执行；否则所有代码在主线程中执行
  shared(n)：n 为共享变量，所有线程访问同一块内存
  private(i, tid)：i 和 tid 为私有变量，每个线程有自己的独立副本
*/
{
    #pragma omp for
    for (i = 0; i < n; i++) {
        tid = omp_get_thread_num();  // 单线程执行时始终返回 0
        printf("%d %d\n", tid, i);
    }
}
```

---

## 7 数据作用域子句

### 7.1 shared 和 private 怎么选

通过 `shared` 和 `private` 子句指定变量在并行域的各个线程间是公有还是私有。经验规则：

- **循环变量、临时变量、写变量一般是私有的**。
- **数组变量、仅用于读的变量通常是共享的**。

`shared` 子句：变量在并行区域中由所有线程共享，所有线程访问同一个内存地址。适合只读或需要在多个线程之间共享的变量。默认情况下 OpenMP 中的变量是共享的，除非显式声明为私有。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int n = 10;
    #pragma omp parallel shared(n)
    {
        printf("Thread %d reads n = %d\n", omp_get_thread_num(), n);
    }
    return 0;
}
```

输出示例：

```text
Thread 0 reads n = 10
Thread 1 reads n = 10
Thread 2 reads n = 10
Thread 3 reads n = 10
```

`private` 子句：变量在并行区域中为每个线程私有，每个线程有独立的副本。对 private 变量的修改只对当前线程有效，不会影响其他线程，也不会影响主线程的值。

- 初始化：线程中的私有变量**未初始化**，除非使用 `firstprivate`。
- 作用域：仅在并行区域内有效。进入和退出并行域时都是"未定义"的，值带不进来也带不出去。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int i;
    #pragma omp parallel for private(i)
    // 变量 i 为每个线程私有，各线程分别处理自己的循环迭代
    for (i = 0; i < 4; i++) {
        printf("Thread %d processes i = %d\n", omp_get_thread_num(), i);
    }
    return 0;
}
```

输出示例（4 线程时）：

```text
Thread 0 processes i = 0
Thread 1 processes i = 1
Thread 2 processes i = 2
Thread 3 processes i = 3
```

原稿还给了一个 `shared` 有坑的例子：

```c
int x = 10;
#pragma omp parallel shared(x)
{
    x += omp_get_thread_num();
}
printf("x: %d\n", x);
```

这段代码里多个线程同时读改写 `x`，本身就是数据竞争，结果不确定。要想正确累加，得用 `atomic`、`critical` 或 `reduction`。

### 7.2 default

`default` 子句显式声明并行域内未指定变量的默认作用域，让用户自行规定在一个并行域的静态范围中所定义的变量的缺省作用范围。缺省值是 `shared`。

- `shared`：未声明的变量默认为共享。
- `private`：未声明的变量默认为私有。
- `none`：强制声明每个变量作用域，避免意外的共享或私有化。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int n = 10, x = 20;
    #pragma omp parallel default(none) shared(n) private(x)
    {
        x = omp_get_thread_num();   // 每个线程独立初始化私有变量 x
        printf("Thread %d: n = %d, x = %d\n", omp_get_thread_num(), n, x);
    }
    return 0;
}
```

输出示例：

```text
Thread 0: n = 10, x = 0
Thread 1: n = 10, x = 1
Thread 2: n = 10, x = 2
Thread 3: n = 10, x = 3
```

### 7.3 firstprivate

`firstprivate` 指定每个线程都有自己的私有副本，并且**继承主线程中的初值**。声明的变量为线程私有，初始值由对应的共享变量的值给出。

- 初始化：继承并行区域外的初始值。
- 作用域：仅在并行区域内有效。
- 适用场景：需要把并行域外的变量值传递给每个线程的私有变量时。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int x = 10;
    #pragma omp parallel firstprivate(x)
    {
        x += omp_get_thread_num();   // 每个线程独立操作私有变量 x
        printf("Thread %d: x = %d\n", omp_get_thread_num(), x);
    }
    return 0;
}
```

输出示例：

```text
Thread 0: x = 10
Thread 1: x = 11
Thread 2: x = 12
Thread 3: x = 13
```

（更正：原文讲 `firstprivate` 时贴的示例代码与上面 `private` 的示例完全相同，写的是 `#pragma omp parallel private(x)`，应为 `firstprivate(x)`。这里按 `firstprivate` 的语义给出正确写法，输出取自原稿"并性域结构：FIRSTPRIVATE/LASTPRIVATE"一节。）

### 7.4 lastprivate

`lastprivate` 把线程中私有变量的值在并行处理结束后复制回主线程中的对应变量。声明的变量为线程私有，在对应的并行任务结束时把自己的私有变量的值赋给对应的共享变量。

"最后一个"指的是**语法上的最后一个**：循环的最后一次迭代，或 `sections` 的最后一个 section，与哪个线程先跑完无关。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int x = 0;
    #pragma omp parallel for lastprivate(x)
    for (int i = 0; i < 4; i++) {
        x = i;                       // 每个线程独立更新私有变量 x
        printf("Thread %d processes i = %d, x = %d\n", omp_get_thread_num(), i, x);
    }
    printf("Last value of x: %d\n", x);
    return 0;
}
```

输出示例：

```text
Thread 0 processes i = 0, x = 0
Thread 1 processes i = 1, x = 1
Thread 2 processes i = 2, x = 2
Thread 3 processes i = 3, x = 3
Last value of x: 3
```

### 7.5 firstprivate 和 lastprivate 一起用

进入并行区域时初始化私有变量，离开并行区域时把最终结果传回主线程。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int x = 10;
    #pragma omp parallel for firstprivate(x) lastprivate(x)
    for (int i = 0; i < 4; i++) {
        x += i;                      // 每个线程操作自己的私有变量 x
        printf("Thread %d processes i = %d, x = %d\n", omp_get_thread_num(), i, x);
    }
    printf("Final value of x: %d\n", x);
    return 0;
}
```

输出示例：

```text
Thread 0 processes i = 0, x = 10
Thread 1 processes i = 1, x = 11
Thread 2 processes i = 2, x = 12
Thread 3 processes i = 3, x = 13
Final value of x: 13
```

这个输出建立在"4 个线程各分到一次迭代"上。线程数变了结果就变：2 个线程时 T0 拿 i=0,1 得 11，T1 拿 i=2,3 得 15，最后一次迭代 i=3 在 T1 上，`lastprivate` 传回 15；单线程时是 $10+0+1+2+3=16$。考试如果问"输出是多少"，先确认线程数和调度方式。

### 7.6 reduction

作用：把每个线程的私有变量值按某种操作（加法、乘法、最大值等）合并，在并行区域结束后写回共享变量，解决多线程同时操作同一变量的数据竞争。

初始时每个线程保留一份私有拷贝，在结构尾部根据指定操作对线程中相应变量进行规约，更新该变量的全局值。

支持的操作：加法 `+`、乘法 `*`、最大值 `max`、最小值 `min`、按位操作 `&`、`|`、`^`。

```c
#pragma omp parallel reduction(operator:variable)
{
    // 线程私有副本的操作
}
```

**并行加法。** 每个线程计算自己的部分和，OpenMP 在并行区域结束时把所有线程的结果加总到 `sum`。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int sum = 0;
    int n = 10;
    #pragma omp parallel for reduction(+:sum)
    for (int i = 1; i <= n; i++) {
        sum += i;
    }
    printf("sum=%d\n", sum);
    return 0;
}
```

输出：

```text
sum=55
```

**并行最大值。** 每个线程求自己负责范围内的最大值，结束时合并成全局最大值。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int max_val = 0;
    int array[] = {1, 3, 5, 7, 9, 2, 4, 6, 8};
    int n = 9;
    #pragma omp parallel for reduction(max:max_val)
    for (int i = 0; i < n; i++) {
        if (array[i] > max_val) {
            max_val = array[i];      // 每个线程独立寻找局部最大值
        }
    }
    printf("Max value=%d\n", max_val);
    return 0;
}
```

输出：

```text
Max value=9
```

**数据竞争与解决。** 下面这段有典型的数据竞争：

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int data = 0;
    #pragma omp parallel for
    for (int i = 0; i < 10000; i++) {
        data++;                      // 多个线程可能同时修改 data
    }
    printf("Final data = %d\n", data);
    return 0;
}
```

问题在于多个线程同时对 `data` 自增，线程间操作互相覆盖，输出可能小于理论值 10000。改用 `reduction` 之后每个线程有自己的 `data` 副本，并行区域结束时累加到主线程：

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int data = 0;
    #pragma omp parallel for reduction(+:data)
    for (int i = 0; i < 10000; i++) {
        data++;
    }
    printf("Final data=%d\n", data);
    return 0;
}
```

输出：

```text
Final data=10000
```

### 7.7 其他子句

| 子句 | 作用 |
| --- | --- |
| `nowait` | 忽略指令中暗含的等待。`parallel for`、`sections` 等结束后隐含一个 barrier，`nowait` 让线程不等待直接继续 |
| `num_threads(n)` | 指定线程的个数 |
| `schedule(kind[, chunksize])` | 指定如何调度 for 循环迭代 |
| `ordered` | 指定 for 循环的执行要按顺序执行 |
| `copyprivate` | 用于 `single` 制导，把指定变量广播到并行区中其他线程 |
| `copyin` | 用主线程的值初始化 threadprivate 变量，为线程组中所有线程的 threadprivate 变量赋相同的值 |

`nowait` 的例子：

```c
#pragma omp parallel
{
    #pragma omp for nowait
    for (int i = 0; i < 10; i++) {
        printf("Thread %d: iteration %d\n", omp_get_thread_num(), i);
    }
    printf("Thread %d finishes early\n", omp_get_thread_num());
}
```

`num_threads` 的例子：

```c
#pragma omp parallel num_threads(4)
{
    printf("Thread %d\n", omp_get_thread_num());
}
```

---

## 8 任务划分结构

任务划分结构（work-distribution constructs）把代码中的任务明确分配到多个线程执行。它**不创建新线程**，只在已有线程组中分配任务。

- 入口点没有路障，但**结束处有一个隐含的路障（barrier）**，所有线程必须完成任务才能继续。
- 一个共享任务结构必须**动态地封装在一个并行域中**，制导语句才能并行执行。

三种：

| 制导 | 并行类型 | 适用场景 |
| --- | --- | --- |
| `DO/for` | 数据并行 | 把循环的迭代分配给多个线程执行 |
| `sections` | 功能并行 | 每个线程执行不同的代码段，任务彼此独立 |
| `single` | 串行执行 | 只有一个线程执行代码块，其他线程跳过，适合初始化或非并行化任务 |

### 8.1 for 循环制导

功能：把循环的迭代划分为多个块，分配给不同线程执行。C 语言中通过 `for` 循环配合 `#pragma omp for` 实现。

特点：

- 默认情况下**循环变量是私有变量**。
- 可以配合 `private`、`firstprivate` 等子句控制变量作用域。

两种写法：

```c
// 单独使用 for 指令（必须在已有并行域内）
#pragma omp for [clauses]
for (循环)
{
    // 并行执行的代码
}
```

```c
// 结合并行域使用
#pragma omp parallel for [clauses]
for (循环)
{
    // 并行执行的代码
}
```

常用子句：`private` 声明私有变量，`reduction` 归约，`schedule` 指定循环任务的分配方式。

例 1：简单循环并行化。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int n = 8;
    #pragma omp parallel for
    for (int i = 0; i < n; i++) {
        printf("Thread %d processes iteration %d\n", omp_get_thread_num(), i);
    }
    return 0;
}
```

输出（顺序可能不同）：

```text
Thread 0 processes iteration 0
Thread 1 processes iteration 1
Thread 2 processes iteration 2
Thread 3 processes iteration 3
...
```

例 2：用 `private` 子句，`sum` 每个线程一份副本，线程间互不干扰。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int n = 8;
    int sum = 0;
    #pragma omp parallel for private(sum)
    for (int i = 0; i < n; i++) {
        sum = i * 2;                 // 每个线程独立计算自己的 sum
        printf("Thread %d: sum=%d\n", omp_get_thread_num(), sum);
    }
    return 0;
}
```

输出：

```text
Thread 0: sum = 0
Thread 1: sum = 2
Thread 2: sum = 4
Thread 3: sum = 6
...
```

例 3：用 `reduction` 子句解决数据竞争，每个线程算部分和，最终规约到共享变量 `sum`。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int sum = 0;
    #pragma omp parallel for reduction(+:sum)
    for (int i = 0; i < 10; i++) {
        sum += i;                    // 每个线程独立计算部分和
    }
    printf("Total sum = %d\n", sum);
    return 0;
}
```

输出：

```text
Total sum = 45
```

（更正：原文这段代码漏写了 `printf` 和 `return`，但注释里给出了 `Total sum = 45`，与 $0+1+\dots+9=45$ 一致。）

### 8.2 schedule 调度子句

功能：控制循环并行化任务的分配方式，决定每个线程如何获得循环迭代任务以及任务的分配顺序。

原稿用一段下三角矩阵乘向量的代码说明为什么需要调度：

```c
#pragma omp parallel for
for (row = 1; row <= 6; row++) {
    for (col = 1; col <= row; col++) {
        res[row][col] += l[row][col] * y[col];
    }
}
```

`row` 越大内层循环次数越多，各次迭代的工作量不相等，平均分配会让拿到前几行的线程早早空闲。

格式：`schedule(kind[, chunksize])`。

- `kind` 取 `static`、`dynamic`、`guided`、`runtime`。
- `chunksize` 是每个线程一次获取的迭代块大小，可选。

**static 静态调度。**

- 将循环迭代静态划分为若干块，每个线程分配固定数量的迭代。
- 默认行为：省略 `chunksize` 时，迭代空间被划分成（近似）相同大小的区域，每个线程被分配一个区域。
- 指明 `chunksize` 时，迭代空间被划分为 `chunksize` 大小，然后**轮转**地分配给各个线程。

例：线程数为 4，`schedule(static, 2)`，16 个任务：

| 线程 | 分到的任务 |
| --- | --- |
| T0 | 0, 1, 8, 9 |
| T1 | 2, 3, 10, 11 |
| T2 | 4, 5, 12, 13 |
| T3 | 6, 7, 14, 15 |

原稿还有一张示意图：`schedule(static)` 把 1 到 40 平均切成四段给 T0 T1 T2 T3；`schedule(static, 4)` 把 1 到 40 切成 10 个宽度为 4 的块，按 T0 T1 T2 T3 T0 T1 ... 轮转。

```c
#pragma omp parallel for schedule(static, 2)   // 每个线程一次分配 2 次迭代
// 省略 chunksize 时任务会被平均分配
for (int i = 0; i < 10; i++)
{
    printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
}
```

**dynamic 动态调度。**

- 划分迭代空间为 `chunksize` 大小的区间，然后基于**先来先服务**方式分配给各线程。
- 默认块大小 `chunksize = 1`。
- 适用场景：循环任务的执行时间不均匀时。

```c
#pragma omp parallel for schedule(dynamic, 3)
// 每个线程一次获取 3 次迭代任务，完成后再动态领取剩余任务
for (int i = 0; i < 12; i++)
{
    printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
}
```

**guided 指导调度。**

- 类似动态调度，但区间开始大，然后迭代区间越来越小。
- `chunksize` 说明**最小**的区间大小，省略时默认值为 1。
- 适用场景：循环任务不均匀，且希望减少调度开销时。

```c
#pragma omp parallel for schedule(guided, 2)   // 开始时分配较大的任务块，逐渐减少
for (int i = 0; i < 20; i++)
{
    printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
}
```

块大小怎么缩具体由实现决定。以常见的"每次取 剩余迭代数 / 线程数，且不小于 chunksize"为例，20 次迭代、4 线程、`chunksize = 2` 会切成 `[0,4] [5,8] [9,11] [12,13] [14,15] [16,17] [18,19]`，块从 5 一路缩到 2。

**runtime 运行时调度。**

- 调度类型和块大小在运行时通过环境变量 `OMP_SCHEDULE` 指定，代码里不用写死。
- 使用 runtime 时，**指明 chunksize 是非法的**。

```bash
export OMP_SCHEDULE="dynamic,4"
```

```c
#pragma omp for schedule(runtime)
for (int i = 0; i < 10; i++)
{
    printf("Thread %d handles iteration %d\n", omp_get_thread_num(), i);
}
```

### 8.3 循环中的数据竞争

错误示例：

```c
#pragma omp parallel for
for (k = 0; k < 100; k++) {
    x = array[k];
    array[k] = do_work(x);
}
```

变量 `x` 是所有线程共享的，多个线程同时访问和修改 `x` 会导致数据竞争，`x` 的值被覆盖，计算结果不正确。

正确写法：

```c
#pragma omp parallel for private(x)
// 每个线程有独立的 x 副本，避免线程间冲突
for (k = 0; k < 100; k++)
{
    x = array[k];
    array[k] = do_work(x);
}
```

### 8.4 循环依赖

循环依赖是并行化的主要障碍，按依赖关系分三类：

| 类型 | 别名 | 例子 | 为什么不能并行 |
| --- | --- | --- | --- |
| 流依赖 | 跨迭代写后读 | `A[j] = A[j-1];` | 当前迭代的值依赖上一迭代的计算结果，值要顺着传 |
| 反依赖 | 跨迭代读后写 | `A[j] = A[j+1];` | 当前迭代会修改下一迭代所需的值，线程间依赖冲突 |
| 写依赖 | 跨迭代写相关 | `A[j] = B[j]; A[j+1] = C[j];` | 不同迭代的写操作对同一数据有交叉，结果互相覆盖 |

三段原始代码：

```c
// 流依赖
for (j = 1; j < MAX; j++) {
    A[j] = A[j - 1];
}

// 反依赖
for (j = 1; j < MAX; j++) {
    A[j] = A[j + 1];
}

// 写依赖
for (j = 1; j < MAX; j++) {
    A[j] = B[j];
    A[j + 1] = C[j];
}
```

消除流依赖的办法之一是**双缓冲**：先把要读的值拷到临时数组，再统一写回。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int A[100], temp[100];
    for (int i = 0; i < 100; i++) A[i] = i;

    #pragma omp parallel for
    for (int j = 1; j < 100; j++) {
        temp[j] = A[j - 1];          // 用临时数组打破依赖
    }

    #pragma omp parallel for
    for (int j = 1; j < 100; j++) {
        A[j] = temp[j];
    }
    return 0;
}
```

### 8.5 sections 制导

功能：把任务划分为多个互不相关的分段任务（结构化块），这些分段任务可以在多个线程之间并行执行。每个 section 块由一个线程执行，多个 section 可以同时运行。适用于不同任务彼此独立且可以并行化时。

```c
#pragma omp sections [clauses]
{
    #pragma omp section
    {
        // 第一段任务
    }
    #pragma omp section
    {
        // 第二段任务
    }
    ...
}
```

结合并行域使用：

```c
#pragma omp parallel sections [clauses]
{
    // sections 内容
}
```

支持 `private`（声明私有变量）和 `firstprivate`（声明并初始化私有变量）。**每个 section 必须包含一个结构块**（一对大括号）。

示例 1：简单任务分段。任务被划分为两个 section，分别由两个线程执行。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    #pragma omp parallel sections
    {
        #pragma omp section
        {
            printf("Thread %d is executing section 1\n", omp_get_thread_num());
        }
        #pragma omp section
        {
            printf("Thread %d is executing section 2\n", omp_get_thread_num());
        }
    }
    return 0;
}
```

输出（可能）：

```text
Thread 0 is executing section 1
Thread 1 is executing section 2
```

示例 2：带私有变量的任务分段。每个线程进入 section 时都有独立的私有变量 `x`，各 section 内部的变量值互不干扰。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int x = 10;
    #pragma omp parallel sections private(x)
    {
        #pragma omp section
        {
            x = omp_get_thread_num();
            printf("Thread %d: x=%d in section 1\n", omp_get_thread_num(), x);
        }
        #pragma omp section
        {
            x = omp_get_thread_num() * 2;
            printf("Thread %d: x=%d in section 2\n", omp_get_thread_num(), x);
        }
    }
    return 0;
}
```

输出（可能）：

```text
Thread 0: x = 0 in section 1
Thread 1: x = 2 in section 2
```

注意事项：

1. 同步点：`sections` 块结束时所有线程在隐含同步点等待，不需要同步可以加 `nowait`。
2. 线程分配：每个 section 至少分配一个线程，线程数不足时部分 section 顺序执行。
3. 单独使用 `sections` 只能划分任务，无法创建新线程；结合 `parallel` 使用才会创建线程组。

### 8.6 single 制导

`#pragma omp single` 指定某段代码仅由一个线程执行，其他线程在同步点等待，直到该代码块执行完成。

特点：

1. 单线程执行：一个线程执行 single 代码块，通常是第一个到达的线程。
2. 隐含同步点：其他线程会在同步点等待，除非使用 `nowait`。

典型应用：初始化操作（分配资源、读取输入），以及不需要并行化的串行任务。

```c
#pragma omp single [clauses]
{
    // 需要由单线程执行的代码块
}
```

支持 `private` 和 `firstprivate`。

注意事项：

1. 隐含同步点：single 代码块结束后所有线程在同步点等待，不需要同步可用 `nowait` 优化性能。
2. 变量作用域：默认情况下 single 代码块中的变量为共享变量，需要私有时用 `private` 或 `firstprivate` 明确指定。

示例 1：所有线程到达 single 代码块时只有一个线程执行，其他线程等待，执行完后所有线程继续。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    #pragma omp parallel
    {
        printf("Thread %d: Before single block\n", omp_get_thread_num());
        #pragma omp single
        {
            printf("Thread %d: Executing single block\n", omp_get_thread_num());
        }
        printf("Thread %d: After single block\n", omp_get_thread_num());
    }
    return 0;
}
```

输出（可能）：

```text
Thread 0: Before single block
Thread 1: Before single block
Thread 2: Before single block
Thread 0: Executing single block
Thread 1: After single block
Thread 2: After single block
Thread 0: After single block
```

示例 2：结合变量作用域。`private(x)` 让 `x` 为 single 代码块中执行线程独有，其他线程的 `x` 不受影响。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int x = 0;
    #pragma omp parallel
    {
        #pragma omp single private(x)
        {
            x = omp_get_thread_num();
            printf("Thread %d: x=%d in single block\n", omp_get_thread_num(), x);
        }
        printf("Thread %d: Outside single block\n", omp_get_thread_num());
    }
    return 0;
}
```

输出（可能）：

```text
Thread 0: x = 0 in single block
Thread 1: Outside single block
Thread 2: Outside single block
Thread 3: Outside single block
```

---

## 9 同步结构

同步结构控制执行过程中各线程的同步，确保多个线程执行共享任务时保持正确性和一致性。**七种典型的同步结构**：

| 制导 | 作用 |
| --- | --- |
| `master` | 指定代码段只由主线程执行，线程组中其他线程忽略此代码段 |
| `critical` | 指定代码段为线程互斥临界区，同一时刻只能由一个线程执行 |
| `barrier` | 同步一个线程组中的所有线程，先到达的线程阻塞 |
| `atomic` | 指定特定的存储单元被原子地更新 |
| `flush` | 标识一个同步点，确保所有线程看到一致的存储器视图 |
| `ordered` | 指定所包含的循环以串行方式执行，任何时候只能有一个线程执行这个部分 |
| `threadprivate` | 使一个全局文件作用域的变量在并行域内变成每个线程私有，每个线程对该变量复制一份私有拷贝 |

### 9.1 critical

指定一个代码段为临界区，同一时间只允许一个线程执行。其他线程尝试进入时会被阻塞，直到当前线程退出临界区。用于对共享资源（变量或数据结构）的访问需要互斥保护时。

```c
#pragma omp critical [(name)]
{
    // 临界区代码块
}
```

示例 1：基本用法。每个线程进入 critical 区块时，保证同一时间只有一个线程操作 `shared_var`。输出顺序由线程竞争情况决定，但 `shared_var` 的值变化是线性递增的。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int shared_var = 0;
    #pragma omp parallel num_threads(4)
    {
        #pragma omp critical
        {
            shared_var += 1;
            printf("Thread %d incremented shared_var to %d\n",
                   omp_get_thread_num(), shared_var);
        }
    }
    return 0;
}
```

输出（可能）：

```text
Thread 1 incremented shared_var to 1
Thread 2 incremented shared_var to 2
Thread 3 incremented shared_var to 3
Thread 0 incremented shared_var to 4
```

示例 2：critical 和 single 的对比。这是很好的辨析题。

```c
int a = 0, b = 0;
#pragma omp parallel num_threads(4)
{
    #pragma omp single
    a++;

    #pragma omp critical
    b++;
}
printf("single: %d -- critical: %d\n", a, b);
```

输出：

```text
single: 1 -- critical: 4
```

区别在于：`single` 让 `a` 的增量操作**只由一个线程执行一次**；`critical` 让 `b` 的增量操作**每个线程都执行一次**，但同一时间只有一个线程进行。

**critical 的命名。**

- 给 `critical` 指定了 name，则所有相同名称的 critical 块被视为**一个区域**。
- 不同名称的 critical 块可以并行执行。
- 所有**未命名的 critical 区域均视为同一区域**。

示例 3：不同名称的 critical 区域，`critical1` 和 `critical2` 是两个独立的临界区，可以同时被两个线程执行。

```c
#pragma omp parallel sections
{
    #pragma omp section
    {
        #pragma omp critical(critical1)
        {
            for (int i = 0; i < 5; i++) {
                printf("section1 thread %d execute i=%d\n", omp_get_thread_num(), i);
                Sleep(200);
            }
        }
    }
    #pragma omp section
    {
        #pragma omp critical(critical2)
        {
            for (int j = 0; j < 5; j++) {
                printf("section2 thread %d execute j=%d\n", omp_get_thread_num(), j);
                Sleep(200);
            }
        }
    }
}
```

示例 4：相同名称的 critical 区域。把上面第二个 section 里的 `critical(critical2)` 改成 `critical(critical1)`，两个块就被视为同一个临界区，同一时间只能有一个线程进入，两段输出不会交叉。

```c
#pragma omp parallel sections
{
    #pragma omp section
    {
        #pragma omp critical(critical1)
        {
            for (int i = 0; i < 5; i++) {
                printf("section1 thread %d execute i=%d\n", omp_get_thread_num(), i);
                Sleep(200);
            }
        }
    }
    #pragma omp section
    {
        #pragma omp critical(critical1)
        {
            for (int j = 0; j < 5; j++) {
                printf("section2 thread %d execute j=%d\n", omp_get_thread_num(), j);
                Sleep(200);
            }
        }
    }
}
```

`Sleep(200)` 是 Windows 下的函数，需要 `#include <windows.h>`；Linux 下换成 `usleep(200000)`。加延时是为了把交叉执行的现象放慢到肉眼可见。

### 9.2 barrier

`barrier` 确保所有线程执行到该语句时等待，直到线程组中的所有线程都到达。等待结束后所有线程同时继续执行后续代码。

```c
#pragma omp barrier
```

特性：

1. 全局同步：线程组中的每个线程都必须到达 barrier，否则线程组会阻塞。
2. 隐式同步：一些制导语句（`DO/for`、`sections`、`single`）中隐含了 barrier，无需显式使用。

注意事项：

1. 线程同步：barrier 是所有线程的同步点，线程必须全部到达才能继续，否则会发生**死锁**。
2. 隐式 barrier：`parallel`、`for`、`sections`、`single`（除非使用 `nowait`）默认包含隐式 barrier。
3. 性能影响：频繁使用 barrier 会增加线程等待时间，影响效率。不需要同步时可以用 `nowait` 避免隐式 barrier。

典型应用场景：多阶段任务同步，把任务分解为多个阶段，确保每个阶段完成后再进行下一阶段。

示例 1：显式 barrier。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    #pragma omp parallel num_threads(4)
    {
        printf("Thread %d: Before barrier\n", omp_get_thread_num());
        #pragma omp barrier
        printf("Thread %d: After barrier\n", omp_get_thread_num());
    }
    return 0;
}
```

输出（可能）：

```text
Thread 0: Before barrier
Thread 1: Before barrier
Thread 2: Before barrier
Thread 3: Before barrier
Thread 3: After barrier
Thread 1: After barrier
Thread 2: After barrier
Thread 0: After barrier
```

关键在于所有 `Before barrier` 一定排在所有 `After barrier` 前面。

示例 2：与循环配合使用。每个线程在每次循环迭代中先打印 `Iteration`，到达 barrier 后等待其他线程完成当前迭代，然后继续打印。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    #pragma omp parallel num_threads(3)
    {
        for (int i = 0; i < 3; i++) {
            printf("Thread %d: Iteration %d\n", omp_get_thread_num(), i);
            #pragma omp barrier
            printf("Thread %d: After barrier in iteration %d\n", omp_get_thread_num(), i);
        }
    }
    return 0;
}
```

输出（可能）：

```text
Thread 0: Iteration 0
Thread 2: Iteration 0
Thread 1: Iteration 0
Thread 1: After barrier in iteration 0
Thread 0: After barrier in iteration 0
Thread 2: After barrier in iteration 0
...
```

示例 3：分阶段任务同步。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    #pragma omp parallel num_threads(4)
    {
        printf("Thread %d: Phase 1 complete\n", omp_get_thread_num());
        #pragma omp barrier
        printf("Thread %d: Phase 2 complete\n", omp_get_thread_num());
    }
    return 0;
}
```

### 9.3 atomic

`atomic` 保证共享变量的特定存储单元操作是原子的，操作过程不可被其他线程中断，一般用于对共享变量的操作。不允许多个线程同时对变量进行读写，从而避免数据竞争。

特点：

- 只保护**单条语句**的执行，不是代码块。
- 提供一个最小的临界区，效率通常高于 `critical`。
- 常用于对共享变量的简单操作（加减、乘除等）。

```c
#pragma omp atomic
statement
```

注意事项：

1. 保护范围：`atomic` 只能保护单条语句，`critical` 可保护代码块。`atomic` 不支持复杂的逻辑或多个变量的操作。
2. 效率：`atomic` 开销小，适合对单个变量的简单操作。性能要求高的程序推荐优先用 `atomic`。
3. 常见用途：计数器累加或递减、标志位的原子性操作。

示例 1：基本用法。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int shared_var = 0;
    #pragma omp parallel num_threads(4)
    {
        #pragma omp atomic
        shared_var += 1;
        printf("Thread %d updated shared_var to %d\n",
               omp_get_thread_num(), shared_var);
    }
    printf("Final value of shared_var: %d\n", shared_var);
    return 0;
}
```

（更正：原文作 `#pragma omp parallel num_thread(4)`，子句名应为 `num_threads`。）

输出（可能）：

```text
Thread 0 updated shared_var to 1
Thread 1 updated shared_var to 2
Thread 2 updated shared_var to 3
Thread 3 updated shared_var to 4
Final value of shared_var: 4
```

这里 `printf` 在 atomic 保护之外，读到的中间值不一定是自己刚写的那个，但最终值 4 是确定的。

示例 2：与 critical 的比较。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    int atomic_var = 0, critical_var = 0;
    #pragma omp parallel num_threads(4)
    {
        #pragma omp atomic
        atomic_var++;

        #pragma omp critical
        {
            critical_var++;
        }
    }
    printf("Atomic result: %d\n", atomic_var);
    printf("Critical result: %d\n", critical_var);
    return 0;
}
```

输出：

```text
Atomic result: 4
Critical result: 4
```

结果一样，区别在于 `atomic` 只适用于简单的变量操作，保护的代码粒度小、开销低；`critical` 可用于更复杂的代码块，但同步开销较大。

示例 3：原子性减操作。

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int shared_var = 10;
    #pragma omp parallel num_threads(5)
    {
        #pragma omp atomic
        shared_var--;
        printf("Thread %d decremented shared_var to %d\n",
               omp_get_thread_num(), shared_var);
    }
    printf("Final value of shared_var: %d\n", shared_var);
    return 0;
}
```

输出（可能）：

```text
Thread 0 decremented shared_var to 9
Thread 1 decremented shared_var to 8
Thread 2 decremented shared_var to 7
Thread 3 decremented shared_var to 6
Thread 4 decremented shared_var to 5
Final value of shared_var: 5
```

### 9.4 flush

`flush` 标识一个同步点，在该点上 list 中的变量需要被写回主内存，而不是暂时存在寄存器或线程私有的缓存里，保证其他线程读到的是最新的共享变量值。

特点：

1. 不指定 list 则刷新所有共享变量。
2. 隐含刷新操作的指令（`barrier`、`critical` 等）无需显式调用 `flush`。

注意事项：

1. 刷新范围：未指定 list 时所有共享变量都会刷新，可能增加不必要的开销；指定 list 时仅刷新列表中的变量。
2. 隐式 flush：多数 OpenMP 指令包含隐式 flush，包括 `barrier`、`critical`、`atomic`、`ordered`、`parallel` 等；循环结构（`for`、`sections`、`single`）也自带同步点。
3. 性能影响：频繁调用 flush 会影响性能。

示例 1：手动 flush。线程 0 更新 `shared_var` 并显式调用 flush，线程 1 读取前也 flush 一次，确保读到主内存中的最新值。

```c
#include <stdio.h>
#include <omp.h>

int shared_var = 0;

void update_shared_var() {
    #pragma omp flush(shared_var)    // 刷新变量到主内存
    shared_var = 10;
    #pragma omp flush(shared_var)    // 再次刷新，确保其他线程可见
}

int main()
{
    #pragma omp parallel num_threads(2)
    {
        int tid = omp_get_thread_num();
        if (tid == 0) {
            update_shared_var();
        } else {
            int local_var;
            #pragma omp flush(shared_var)   // 确保读取最新值
            local_var = shared_var;
            printf("Thread %d reads shared_var: %d\n", tid, local_var);
        }
    }
    return 0;
}
```

（更正：原文作 `local_var-shared_var;`，应为赋值 `local_var = shared_var;`。另外这段代码只靠 flush 并不能保证线程 1 一定在线程 0 写完之后才读，要拿到 10 还得靠 barrier 之类的真正同步，原稿给的输出是理想情况。）

输出：

```text
Thread 1 reads shared_var: 10
```

示例 2：依赖隐式 flush。`barrier` 包含 flush 操作，确保所有线程在同步点之前完成共享变量的更新。

```c
#include <stdio.h>
#include <omp.h>

int shared_var = 0;

int main() {
    #pragma omp parallel num_threads(2)
    {
        int tid = omp_get_thread_num();
        if (tid == 0) {
            shared_var = 10;
        }
        #pragma omp barrier          // 隐式 flush
        if (tid == 1) {
            printf("Thread %d reads shared_var: %d\n", tid, shared_var);
        }
    }
    return 0;
}
```

输出：

```text
Thread 1 reads shared_var: 10
```

这个版本才是可靠的写法：barrier 既排了序又刷了内存。

### 9.5 ordered

在并行化的 for 循环中，`ordered` 保证一部分代码按循环迭代的顺序执行，强制并行线程按迭代次序执行某段特定代码。

特点：

- 必须配合 `for` 或 `parallel for` 使用。
- for 循环必须带 `ordered` 标志（原稿写作"必须使用 schedule 子句且带有 ordered 标志"）。
- 在一个循环中只能出现一次 `ordered`。

```c
#pragma omp ordered
structured-block
```

注意事项：

1. 限制条件：只能在 `for` 或 `parallel for` 中使用，循环必须包含 `ordered` 子句。
2. 性能影响：`ordered` 强制线程按顺序执行代码块，限制了并行化的效率，性能要求高时应谨慎使用。
3. 常见场景：需要按序输出日志信息或数据，或者某些后续操作依赖严格的迭代顺序。

示例：线程并行执行 for 循环，但 `ordered` 指定的代码块按 `i` 的顺序执行。线程可能无序获取循环的迭代值，但在 ordered 中按迭代顺序打印结果。

```c
#include <stdio.h>
#include <omp.h>

int main()
{
    #pragma omp parallel
    {
        #pragma omp for ordered
        for (int i = 0; i < 10; i++) {
            #pragma omp ordered
            printf("Thread %d executes i = %d\n", omp_get_thread_num(), i);
        }
    }
    return 0;
}
```

输出：

```text
Thread 0 executes i = 0
Thread 1 executes i = 1
Thread 2 executes i = 2
Thread 3 executes i = 3
Thread 0 executes i = 4
...
```

线程号乱，但 `i` 一定是 0, 1, 2, 3, 4 的升序。

---

## 10 易混点对比

### 10.1 并发与并行

| | 并发 concurrency | 并行 parallelism |
| --- | --- | --- |
| 硬件 | 单个处理器（核）即可 | 需要多个处理器（核） |
| 实现方式 | 操作系统内核做时间片切换 | 多个核同时执行 |
| 同一时刻运行的任务数 | 1 | 多个 |
| 效果 | 软件方式的伪并行，不会更快 | 真正缩短执行时间 |
| 本章关注 | 回顾 | 线程级并行指这个 |

### 10.2 critical 与 atomic

| | `critical` | `atomic` |
| --- | --- | --- |
| 保护范围 | 代码块 | 单条语句 |
| 适用操作 | 任意复杂逻辑、多个变量 | 单个变量的简单操作（加减乘除、计数器、标志位） |
| 开销 | 较大 | 较小，是最小的临界区 |
| 实现 | 互斥锁 | 通常映射到硬件原子指令 |
| 命名 | 可以带 name，同名视为同一区域，未命名的全部视为同一区域 | 无命名 |

### 10.3 critical 与 single

| | `critical` | `single` |
| --- | --- | --- |
| 执行次数 | 每个线程都执行一次 | 整个线程组只执行一次 |
| 谁执行 | 所有线程轮流 | 第一个到达的那个线程 |
| 结束时是否同步 | 不隐含 barrier | 隐含 barrier，可用 `nowait` 去掉 |
| 4 线程各做一次 `x++` 的结果 | 4 | 1 |

### 10.4 三种（四种）调度方式

| | `static` | `dynamic` | `guided` | `runtime` |
| --- | --- | --- | --- | --- |
| 分配时机 | 编译期/循环开始前定好 | 运行中先来先服务 | 运行中先来先服务 | 由 `OMP_SCHEDULE` 决定 |
| 块大小 | 固定 `chunksize`，省略时平均分 | 固定 `chunksize`，默认 1 | 先大后小，`chunksize` 是最小块，默认 1 | 环境变量指定 |
| 分配顺序 | 按 chunk 轮转给各线程 | 谁空闲谁领 | 谁空闲谁领 | 同被选中的方式 |
| 调度开销 | 最小 | 最大 | 介于两者之间 | 看具体方式 |
| 适合场景 | 各次迭代工作量均匀 | 工作量不均匀 | 工作量不均匀且想少些调度开销 | 不改代码调参数 |
| 能否指定 chunksize | 能 | 能 | 能 | 不能，指定是非法的 |

### 10.5 private 系列子句

| 子句 | 进入并行域时 | 退出并行域时 | 典型用途 |
| --- | --- | --- | --- |
| `shared` | 就是域外那个变量 | 改动保留 | 只读数据、数组 |
| `private` | 未初始化 | 值丢弃，域外变量不变 | 循环变量、临时变量 |
| `firstprivate` | 用域外的值初始化每个副本 | 值丢弃 | 需要带初值进来 |
| `lastprivate` | 未初始化 | 把**语法上最后一次**迭代或最后一个 section 的值写回 | 需要把结果带出去 |
| `reduction` | 每线程一份私有副本，按操作符初始化 | 按操作符合并写回共享变量 | 求和、求积、求极值 |
| `threadprivate` | 每线程一份全局变量的私有拷贝 | 跨并行域保持 | 线程局部的全局状态 |
| `copyin` | 用主线程的值初始化各线程的 threadprivate | 无 | 给 threadprivate 赋统一初值 |

### 10.6 哪些地方有隐式 barrier 和隐式 flush

| 制导 | 隐式 barrier | 隐式 flush |
| --- | --- | --- |
| `parallel` | 有（结尾） | 有 |
| `for` | 有（结尾），`nowait` 可去掉 | 有 |
| `sections` | 有（结尾），`nowait` 可去掉 | 有 |
| `single` | 有（结尾），`nowait` 可去掉 | 有 |
| `critical` | 无 | 有 |
| `atomic` | 无 | 有 |
| `ordered` | 无 | 有 |
| `master` | 无 | 无 |
| `barrier` | 本身就是 | 有 |

---

## 11 典型题

### 11.1 两个线程各做一次自加，有几种执行顺序

题：`x` 初始化为 0，两个线程各执行 `ld / addi / sd` 三条指令，问有几种可能的执行顺序，`x` 最终可能是多少。

思路：两个线程内部顺序固定，交错数就是从 6 个位置里选 3 个给 thread 0，$\binom{6}{3} = 20$ 种。

答案：20 种交错。`x` 的结果只有 1 和 2 两种可能。其中只有 2 种交错能得到 2（一个线程的 `sd` 排在另一个线程的 `ld` 之前，也就是两段完全不重叠），其余 18 种都会丢一次更新，得到 1。

### 11.2 atomic 累乘

题：`data` 初值为 1，4 个线程各执行一次 `data += data * 2`（有 atomic 保护），最终 `data` 是多少。

思路：每个线程把 `data` 乘成 3 倍，原子操作让 4 次更新串行化。

答案：$1 \times 3^4 = 81$。

### 11.3 single 和 critical 的计数

题：4 个线程的并行域里，`a` 在 `single` 下自增，`b` 在 `critical` 下自增，输出是多少。

答案：`single: 1 -- critical: 4`。`single` 只让一个线程执行一次，`critical` 是每个线程都执行、只是不许同时执行。

### 11.4 firstprivate 配 lastprivate

题：`x = 10`，`#pragma omp parallel for firstprivate(x) lastprivate(x)`，循环 `i` 从 0 到 3 做 `x += i`，最后 `x` 是多少。

思路：先看线程数和调度。默认 static 调度下，4 个线程各拿一次迭代，每个线程的私有 `x` 都从 10 开始；`lastprivate` 取的是 `i = 3` 这次迭代结束时的值。

答案：4 线程时为 13。如果只有 2 个线程，T1 拿到 i=2 和 i=3，值为 $10+2+3=15$，答案变成 15；单线程时是 16。做题时要先确认线程数。

### 11.5 static 调度的任务分配

题：线程数 4，`schedule(static, 2)`，共 16 次迭代，各线程分到哪些迭代。

思路：按 chunksize 切成 8 块，轮转发给 4 个线程。

答案：T0 拿 0,1,8,9；T1 拿 2,3,10,11；T2 拿 4,5,12,13；T3 拿 6,7,14,15。

如果省略 chunksize，则平均分成 4 段：T0 拿 0–3，T1 拿 4–7，T2 拿 8–11，T3 拿 12–15。

### 11.6 判断循环能否并行

题：下列循环哪些能直接加 `#pragma omp parallel for`。

```c
for (j = 1; j < MAX; j++) A[j] = A[j-1];         // (1)
for (j = 1; j < MAX; j++) A[j] = A[j+1];         // (2)
for (j = 1; j < MAX; j++) { A[j] = B[j]; A[j+1] = C[j]; }   // (3)
for (j = 0; j < MAX; j++) A[j] = B[j] * 2;       // (4)
```

答案：只有 (4) 能直接并行。(1) 是流依赖（跨迭代写后读），(2) 是反依赖（跨迭代读后写），(3) 是写依赖（跨迭代写相关）。(1) 可以用双缓冲拆成两个循环来并行。

### 11.7 改错题

题：下面的代码为什么结果不对，怎么改。

```c
#pragma omp parallel for
for (k = 0; k < 100; k++) {
    x = array[k];
    array[k] = do_work(x);
}
```

答案：`x` 默认是共享变量，多个线程同时读写造成数据竞争。加 `private(x)` 即可：`#pragma omp parallel for private(x)`。

### 11.8 累加求和的三种写法

题：把 `for (i = 0; i < 10000; i++) data++;` 并行化，直接加 `parallel for` 结果会小于 10000，说明原因并给出修改方案。

答案：`data++` 是"读改写"三步，多个线程的更新会互相覆盖。三种改法：

1. `reduction(+:data)`，每线程一份副本最后归约，性能最好。
2. 在 `data++` 前加 `#pragma omp atomic`，正确但所有线程在同一个变量上排队。
3. 用 `critical` 把 `data++` 包起来，正确但开销最大。

### 11.9 ordered 的输出顺序

题：`#pragma omp for ordered` 的循环里用 `#pragma omp ordered` 包住 printf，输出是什么样。

答案：迭代变量 `i` 严格按 0, 1, 2, … 升序打印，线程号则是乱的。`ordered` 只约束被它包住的那段代码的执行次序，循环的其他部分仍然并行。

### 11.10 为什么 while(!ready) 那段会出错

题：thread 0 执行 `z = 3; ready = 1;`，thread 1 执行 `while (!ready); z++;`，`z` 的最终结果一定是 4 吗。

答案：不一定。超标量处理器允许指令重排序，`z = 3` 若 cache miss 就可能被 `ready = 1` 抢先发射；多处理器之间的 cache 和互联网络也会让写操作的可见顺序发生变化。原稿给出的一种结果是 `z = 0 -> z = 1 -> z = 3`，最终 `z` 为 3。修法是在两个线程之间加真正的同步，例如 `barrier` 或对共享变量的 `flush` 加互斥。
