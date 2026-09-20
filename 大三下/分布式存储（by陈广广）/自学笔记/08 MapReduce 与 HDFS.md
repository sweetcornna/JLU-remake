# 08 MapReduce 与 HDFS

整理自 `分布式存储笔记.pdf` 第 94 到 114 页（4.3 MapReduce 并行计算模型、4.4 HDFS、4.5 Hadoop 中的调度问题）。PageRank 的迭代实现和延迟调度的概率推导是这一章的两道计算题来源。

## 本章要点

- MapReduce 两个用户函数加一个分区过程，五个阶段 Map、Sort、Copy、Merge、Reduce。
- PageRank 的 MapReduce 实现要点：Map 阶段同时发出节点结构和 PR 分量，Reduce 阶段靠类型判断把两者分开。
- HDFS 副本放置三条规则：本地、本地机架、远程机架。
- Secondary NameNode 不是热备，它只做元数据快照和日志合并。
- 延迟调度解决的是公平性与数据本地性的冲突，核心是跳过队首作业但限制跳过次数 $D$。

## 4.3 MapReduce 并行计算模型

### 特点

- 并行化和分布式
- 容错
- 状态监控工具
- 简洁的抽象模型

一句话定位：MapReduce 是大型集群上的大规模数据处理框架。

### Map 与 Reduce

- **Map**：对输入文件中每一个逻辑输入记录进行用户自定义的操作，在数以千计的计算机上运行，以 key/value 形式产生新的数据，即中间结果。
- **Reduce**：根据用户定义的方式汇总数据。

基本过程两步：

1. 用户指定 Map 程序，处理输入，产生中间结果文件 key/value 对；
2. 用户指定 Reduce 程序，处理中间结果对，合并中间结果中有相同 key 值的 value 数据，产生输出。

### WordCount 示例

Map：

```
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
        EmitIntermediate(w, "1")
```

这个 map 函数输出每个单词 w 及其关联的出现次数，在这段伪代码里只简单记为 `"1"`。

Reduce：

```
reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0
    for each v in values:
        result += ParseInt(v)
    Emit(AsString(result))
```

reduce 函数对特定单词所有已输出的计数求和。

### 分区 Partition

分区（Partition）过程把 Map 任务和 Reduce 任务联系起来。Map 的输出按 key 的哈希值分到若干个分区，同一个 key 的所有中间结果必然落到同一个分区，从而被同一个 Reduce 任务处理。

### PageRank

**背景**：Google 的创始人之一 Larry Page（斯坦福）于 1998 年提出 PageRank，应用在 Google 搜索引擎的检索结果排序上，是 Google 早期的核心技术之一。

PageRank 依据网页之间的链接关系评价网页重要程度。级别从 1 到 10 级，PR 值越高说明网页越受欢迎，一般 PR 值达到 4 就算不错的网站，Google 把自己网站的 PR 值定到 10。

**基本设计思想**：被许多优质网页所链接的网页，多半也是优质网页。一个网页要想拥有较高的 PR 值需要两个条件：有很多网页链接到它；有高质量的网页链接到它。

**简化模型**

把互联网上各个网页之间的链接关系看成一个有向图。一个网页的影响力等于所有入链集合的页面的加权影响力之和：

$$PR(u) = \sum_{v \in B_u} \frac{PR(v)}{L(v)}$$

其中 $u$ 为待评估的页面， $B_u$ 为页面 $u$ 的入链集合， $L(v)$ 是页面 $v$ 的出链数量。含义是页面 $v$ 把影响力 $PR(v)$ 平均分配给了它的出链，统计所有能给 $u$ 带来链接的页面 $v$，总和就是 $PR(u)$。

两个直觉：

- 入链多说明跳转过去的概率大，页面往往比较重要（类似买某个商品的人越多，这个商品看起来越好）。
- 页面本身影响力大，也会相应增加它出链指向的页面的影响力（类似朋友都是大牛，别人对你的印象也会提升）。

**超链接矩阵 H**

$$H_{ij} = \begin{cases} 1/L_j & \text{if } P_j \in B_i \\ 0 & \text{otherwise} \end{cases}$$

- 矩阵本质：描述网页间跳转概率的转移矩阵。
- $H_{ij}$ 是用户从页面 $P_j$ 随机点击链接到达 $P_i$ 的概率。
- 非零条件：仅当 $P_j$ 存在指向 $P_i$ 的链接（即 $P_j \in B_i$）。
- 权重值 $1/L_j$，所有出链等概率跳转。

课件的三页面例子：P1 与 P2 互相链接，P2 还链向 P3，P3 链向 P1。P2 有两个出链，用户访问 P2 时跳到 P1 或 P3 的概率均为 $1/2$。转移矩阵是

$$H = \begin{bmatrix} 0 & 1/2 & 1 \\ 1 & 0 & 0 \\ 0 & 1/2 & 0 \end{bmatrix}$$

**稳态方程**

$$R = HR$$

其中 $R$ 是所有页面的 PR 值组成的列向量 $R = [R(P_1), R(P_2), \dots, R(P_n)]^T$。

数学本质： $R$ 是矩阵 $H$ 的特征向量（eigenvector），对应的特征值（eigenvalue）为 1。

**迭代流程**三步：每个节点从初始值出发，按照出度给其他节点投票；按照入度依次计算其他节点给予的投票值；获得本轮结果。

对上面那个三页面的矩阵，用 Python 迭代或直接求特征向量，归一化后的稳态解是 $R = (0.4, 0.4, 0.2)$，验算见 [11 典型计算与数值复核.md](<11 典型计算与数值复核.md>)。

**随机浏览模型**

令

$$H' = d \cdot H + (1-d) \cdot [1/N]_{N \times N}$$

则

$$R = H'R$$

其中 $R$ 为列向量代表 PageRank 值， $H'$ 是转移矩阵， $d$ 是**阻尼因子**通常设为 0.85。 $d$ 是按照超链进行浏览的概率， $1-d$ 是随机跳转到一个新网页的概率。

由于 $R = HR$ 满足马尔可夫链的性质，如果马尔可夫链收敛，则 $R$ 存在唯一解。理论上证明了不论初始值如何选取，算法都保证网页排名的估计值能收敛到真实值。

展开成分量形式：

$$PR(p_i) = \frac{1-d}{N} + d \sum_{p_j \in M(p_i)} \frac{PR(p_j)}{L(p_j)}$$

课件下面还写了另一个版本：

$$PR(u) = 1 - d + d \sum_{v \in B_u} \frac{PR(v)}{L(v)}$$

这两个式子不是同一个归一化下的。第一个式子里所有 PR 值之和为 1（概率解释）；第二个式子是 Brin 和 Page 原论文的写法，所有 PR 值之和为 $N$，相当于把第一个式子整体乘了 $N$。两个式子排出来的**名次完全一样**，考试写哪个都可以，但同一道题里不要混用。

**邻接表表示**：网页数目巨大，网页之间连接关系的邻接矩阵是一个很大的稀疏矩阵，所以实际采用邻接表来表示。

**PageRank 的 MapReduce 实现**

算法核心思想：

- **迭代式计算**：通过多轮 MapReduce 任务模拟 PageRank 传播过程，每轮对应一次迭代。
- **数据双重传递**：既要维护图结构（节点邻接表），又要传递 PageRank 值（节点间的权重分配）。

Map 阶段：

```
class MAPPER
  method MAP(nid n, node N)
    p <- N.PageRank / |N.adjacencyList|   # 计算单条出链分配的 PR 值
    EMIT(nid n, N)                        # 关键操作1：传递节点结构（类型为 node）
    for all node m in N.adjacencyList do
      EMIT(nid m, p)                      # 关键操作2：向邻居发送 PR 值（类型为 float）
```

- 输入是（节点 ID，节点对象），节点对象包含 PageRank 值和邻接表。
- 输出两类数据：结构数据（当前节点 ID，节点对象）用于维护图拓扑；权重数据（邻居节点 ID，PR 分配值）向邻节点传递权重。

Reduce 阶段：

```
class REDUCER
  method REDUCE(nid m, [p1, p2, ...])   # 相同目标节点 m 的数据聚合
    M <- empty
    s <- 0
    for all p in [p1, p2, ...] do
      if IsNode(p) then                 # 判断数据类型
        M <- p                          # 恢复图结构（唯一 node 类型数据）
      else
        s <- s + p                      # 累加所有入链 PR 值
    M.PageRank <- s                     # 更新该节点的 PageRank 值
    EMIT(nid m, node M)                 # 输出新状态节点
```

三个核心操作：结构恢复（识别唯一的节点对象数据，靠 `IsNode(p)` 判断）、PR 聚合（对同目标节点的所有入链 PR 值求和）、状态更新（把聚合值赋给节点对象）。

**为什么 Map 要把节点自己也发一遍**：这是这道题最常被问的点。MapReduce 每轮的输出要作为下一轮的输入，而图结构（邻接表）本身不会变化，如果 Map 只发 PR 分量，Reduce 输出的就只有一个数字，下一轮没有邻接表可用。所以 Map 必须把节点对象也以自己的 ID 为 key 发出去，Reduce 再把它捡回来、填上新算的 PR 值输出。

注意伪代码里的 `M.PageRank <- s` 只做了求和，没有加阻尼因子。完整实现还需要在这一步算 $\frac{1-d}{N} + d \cdot s$，课件的伪代码省略了这部分。

**TeraSort 排序**

课件只给了一张采样树的图，没有文字说明。TeraSort 的思路是先对输入采样，选出 $R-1$ 个分割点把 key 空间切成 $R$ 段，用一棵 Trie 树做快速查找，Map 阶段按分割点把记录送进对应的 Reduce，每个 Reduce 内部排序后直接输出，各 Reduce 输出首尾相接就是全局有序。

## 4.4 HDFS

### 架构

Hadoop 内核的两个基本模块：

1. MapReduce 引擎（runtime 或 library）
2. HDFS 分布式文件系统

MapReduce 引擎是运行在 HDFS 之上的计算引擎，使用 HDFS 作为它的数据存储管理器。

HDFS 按主从结构设计：一个单个 NameNode 作为 master，多个 DataNode 作为工作机（slave）。

分布式文件管理：

- HDFS 将文件分割成固定大小的块（早期 64MB）
- 这些块存到工作机 DataNodes 中
- 元数据（DataNodes 和块的映射）由 NameNode 存储

五个特点：主从集群、文件分片、分布式存储、副本管理、并发性和容错性。

### 副本数据存放位置（考点）

| 副本 | 位置 |
| --- | --- |
| 副本 1 | 提交节点，或随机轻负载节点 |
| 副本 2 | 与副本 1 节点相同机架（本地机架） |
| 副本 3 | 与副本 1 不同机架（远程机架） |
| 其他副本 | 随机位置 |

这个策略的用意：副本 1 放本地省一次网络传输；副本 2 放同机架，写入快（机架内带宽高）又能扛住单机故障；副本 3 放另一个机架，能扛住整个机架断电或交换机故障。

### 数据分块大小的选择

早期版本 64M，后续版本 128M、256M、512M。影响因素是网络速度、寻址速度、处理速度。

**最佳传输损耗理论**：在一次传输中，寻址时间占用总传输时间的 1% 为佳。以 10ms 寻址时间为例，传输时间应该是 1 秒：

- 磁盘 100MB/s，1 秒传 100MB，所以块大小取 128MB
- 磁盘 500MB/s，1 秒传 500MB，所以块大小取 512MB

这是一道标准的计算题，算式见 [11 典型计算与数值复核.md](<11 典型计算与数值复核.md>)。

### 特点

- 并发性
- 可扩展性
- 容错性：块复制（默认三个副本）；副本存储位置（本地、本地机架、远程机架）；Heartbeat 和 Blockreport 消息做状态更新
- 安全性
- 高可用性

### HDFS 的局限性

- 实现了 GFS 的几乎所有功能，但**不支持文件更新**（不能修改和追加），所有文件都是一次性写入。
- 不支持按需创建副本。
- 有很好的顺序读（streaming）性能，随机读性能差。

### NameNode

角色：

- 整个 HDFS 集群的主节点
- 管理全局元数据
- 服务用户请求

问题：单点错误。

### Secondary NameNode（重点，最容易答错）

单一的 NameNode 节点存在安全隐患，于是有了 Secondary NameNode。它的角色是：

- **不是 NameNode 节点的热备份**
- **不能在错误恢复中实现快速切换**
- 记录 NameNode 数据的元数据信息快照
- 支持在线合并操作日志
- 支持上传快照到主 NameNode 节点

**NameNode 出错时会怎样**：

- 出错期间无法提供任何服务，Secondary NameNode 并不会作为备份节点进行热切换。
- 那为什么还需要 Secondary NameNode？因为**数据安全比服务连续性更重要**。软件错误可以尝试重启；硬件错误会导致元数据信息丢失，这时 Secondary 保存的快照就是救命的。

**错误恢复流程**：

- 如果原 NameNode 可以被恢复：从 Secondary 获得最新的元数据快照，进行恢复。
- 如果原 NameNode 不可以被恢复：创建一个新的 NameNode，使用 Secondary 保存的元数据进行恢复，重启集群。

**避免重启集群的恢复方案**：

- 问题在于 DataNodes 记住了 NameNode 的地址。
- 方案：创建新的 NameNode；修改 DNS 信息，指向新的 NameNode；注意 Secondary 本身不能作为新的 NameNode。

**其他可靠性手段**：把 NameNode 元数据备份到多个源，例如 NFS 共享文件系统，代价是可能使系统性能受损。

### HDFS 与 MapReduce 协同工作

MapReduce 引擎也是主从结构：一个单独的 JobTracker 作为主服务器，许多 TaskTracker 作为从服务器。

- **JobTracker**：在集群上管理 MapReduce 作业，负责监视作业和分配任务给 TaskTracker，相当于调度器。
- **TaskTracker**：管理集群上单个计算节点的映射和化简任务的执行，相当于运行环境。

作业和数据的映射关系：

```
Job = Map Task(s) + Reduce Task(s)
Input File = Data Block(s)，中间结果也是 Data Block(s)
```

输入文件在 HDFS 中分片存储（数据块），**每一个 Map 任务处理一个数据块，一一映射**。这条是理解 Data Locality 问题的前提。

Map 和 Reduce 的完整过程五个阶段：**Map、Sort、Copy、Merge、Reduce**。

## 4.5 Hadoop 中的调度问题

### 拉取模式

JobTracker 为用户作业分配资源采用的是 Pull Task 方式：

- TaskTracker 通过心跳函数汇报状态
- 如有空闲 slot，提出作业要求
- 每一个 Task（Map 或 Reduce）分配一个 Slot
- 每一个 Mapper 任务依赖于一个数据块（HDFS）

两个条件：

1. 为每一个空闲 slot 分配一个任务（系统角度）
2. 为每一个作业的所有任务（M 或 R）分配 Slot（作业角度）

Mapper 任务占据大部分运行时间，而且 Mapper 任务依赖于特定的数据。**如果数据不在本地怎么办？** 需要进行数据传输。数据在本地显然更高效，不需要额外的数据传输开销。

### Data Locality 问题

- **Local task**：数据块存储于本地 TaskTracker 上
- **Remote task**：数据块存储于远程 TaskTracker 上

$$\text{Data locality} = \frac{\text{本地化任务数}}{\text{总任务数}}$$

两种环境的对比：

- 传统集群模型：数据集中存储，运行前 Stage-In，不存在 Data Locality 问题。
- 网格环境：数据分布存储，各节点自治管理，引出 Data Aware 调度。

课件有一句反直觉的话：数据本地性越高越好吗？并非如此，需要延迟任务执行才能提升数据本地性。调度问题的本质就是**延迟调度（Delay Scheduling）**。

### 公平性与数据本地性的冲突

- **公平共享（Fair sharing）**：满足最大最小公平原则（MaxMin），在多个 job 或 user 之间考虑资源如何公平分配，站在用户角度。
- **数据本地性（Data locality）**：考虑如何达到数据本地性，提高系统吞吐量，站在系统角度。

基本概念：

- **Slot**：一个节点的 slot 数量表示该节点的资源容量或能力大小，是调度的基本资源单位。
- **Job**：一个数据处理应用的一次运行，一个 job 一般有多个 task，每个 task 使用一个 slot。

假设只考虑 Map 任务，因为 Map 阶段执行时间较长。

### 公平调度的两个问题

实现集群作业间公平调度的简单方法（算法 1）：始终把空闲计算槽位分配给当前运行任务最少的作业。只要 slot 能够很快变为空闲，分配结果就满足最大最小公平原则。

**问题 1：Head-of-line Scheduling（队首阻塞）**

- 产生位置：小型作业（输入数据的数据块少）
- 举例：一个 job 只在 10% 的节点上存在数据，最高只能达到 10% 的本地性。

原因是小作业排在队首（运行任务最少），一有空闲 slot 就分给它，而它在这个节点上大概率没有数据。

**问题 2：Sticky Slots（槽位粘滞）**

- 产生位置：所有作业
- 举例：假设集群有 100 个节点，每个节点有一个 slot。有 10 个 job，每个 job 有 10 个 running tasks。如果 job $i$ 在节点 $k$ 上完成一个 task，此时节点 $k$ 请求一个新任务。由于 $i$ 只有 9 个 running tasks 而其他 jobs 有 10 个，按算法 1，节点 $k$ 的 slot 又重新分配给 $i$。最终导致 jobs 永远都离不开 slots（sticky）。

粘滞的后果是每个作业被锁死在固定的一批节点上，无法按数据分布挪动，本地性上不去。

绝对公平算法产生本地化问题的根本原因：**遵循严格的排队调度顺序会导致没有本地数据的 job 被调度**。

### 延迟调度

解决思路两条：

1. 一个节点请求任务的时候，如果队首的 job 不能够启动一个本地 task，就跳过它，查看后续 jobs。
2. 为防止饿死，如果一个 job 被跳过多次，就允许其启动非本地 tasks。

伪代码（课件第 113 页 Algorithm 2 Fair Sharing with Simple Delay Scheduling）：

```
Initialize j.skipcount to 0 for all jobs j.
when a heartbeat is received from node n:
    if n has a free slot then
        sort jobs in increasing order of number of running tasks
        for j in jobs do
            if j has unlaunched task t with data on n then
                launch t on n
                set j.skipcount = 0
            else if j has unlaunched task t then
                if j.skipcount >= D then
                    launch t on n          # 一个 job 最多允许被跳过 D 次
                else
                    set j.skipcount = j.skipcount + 1
                end if
            end if
        end for
    end if
```

**等待的前提是否成立**：尽管第一个 slot 可能没有数据，但由于 tasks 都会迅速完成，有数据的 slot 可能在几秒内就会被释放。形式化的条件是

$$J \gg \frac{T}{S}$$

| 符号 | 含义 |
| --- | --- |
| $T$ | job $j$ 的平均 task 长度 |
| $S$ | 共享集群内的 slots 总数 |
| $T/S$ | 共享集群内一个 slot 平均多久释放一次 |
| $F$ | job $j$ 在集群上公平可用的 slot 总数 |
| $FT/S$ | 一个 job 获得它所有 slots 需要的时间 |
| $J$ | job $j$ 在私有集群上运行花费的时间 |

意思是：只要作业本身的运行时间远大于一个 slot 的平均释放间隔，等几秒钟换一个有数据的 slot 就是划算的。

### 分析 1：等待次数 D 对数据本地性比例的影响

设集群有 $M$ 个节点，每个节点有 $L$ 个 slots，共有 $S = ML$ 个 slots。设 $P_j$ 为 job $j$ 有可用数据的节点集合，则：

- 可实现本地化的节点数量为 $|P_j|$
- 可用节点数量比为 $p_j = \dfrac{|P_j|}{M}$

某任务跳过 $D$ 次都没有本地化启动的可能性为

$$(1 - p_j)^D$$

非本地性随 $D$ 指数下降。课件给了两个数：

- $p_j = 0.1$， $D = 10$ 时： $(1-p_j)^D \approx 0.35$，即有 65% 可能性启动本地任务。
- $p_j = 0.1$， $D = 40$ 时：即有 99% 可能性启动本地任务。

（更正：原文第二行写作 $(1-p_j)^D \approx 0.99$。按公式算 $(1-0.1)^{40} \approx 0.0148$，本地化概率是 $1 - 0.0148 \approx 0.985$，所以"99%"指的是本地化概率而非 $(1-p_j)^D$ 本身，原文把左边的式子写错了。第一行 $(1-0.1)^{10} \approx 0.3487$ 是对的，对应本地化概率 65%。）

### 分析 2：如何设置最大跳数 D

简化假设：所有 tasks 长度 $T$ 相同，且集合 $P_j$ 之间不相关。结论是

$$D \ge -\frac{M}{R}\ln\left(\frac{(1-\lambda)N}{1 + (1-\lambda)N}\right)$$

| 符号 | 含义 |
| --- | --- |
| $\lambda$ | 本地性程度（目标本地化比例） |
| $N$ | 一个 job 的 tasks 数量 |
| $M$ | 一个集群的节点数量 |
| $R$ | 复制因子（副本数） |

定性看这个式子：目标本地性 $\lambda$ 越接近 1，括号里的值越接近 0，对数越负，所需的 $D$ 越大；副本数 $R$ 越大，命中本地数据越容易，所需的 $D$ 越小；集群节点数 $M$ 越大，单个节点恰好有数据的概率越低，所需的 $D$ 越大。

## 典型题

**Q1. 用 MapReduce 实现 WordCount，写出 Map 和 Reduce。**

Map 拿到（文档名，文档内容），对内容里的每个单词 w 输出 `(w, "1")`。中间结果按 key 分区，同一个单词的所有计数进同一个 Reduce。Reduce 拿到（单词，计数列表），把列表求和后输出。

**Q2. 用 MapReduce 迭代计算 PageRank，Map 和 Reduce 分别做什么？为什么 Map 要输出两种类型的数据？**

Map 收到（节点 ID，节点对象），先算出每条出链该分到的 PR 值 $p = PR(n) / |adj(n)|$，然后输出两种记录：以自己的 ID 为 key 输出整个节点对象，以每个邻居的 ID 为 key 输出 $p$。Reduce 收到同一个目标节点的一堆值，用类型判断把唯一的节点对象捡出来恢复图结构，把其余的浮点数累加成新的 PR 值，写回节点对象输出。

输出两种类型是因为 MapReduce 是无状态的：本轮的输出就是下轮的输入，如果不把邻接表一起传递下去，下一轮 Map 就不知道每个节点有哪些出链了。

**Q3. HDFS 的三个副本分别放在哪里？为什么这么放？**

副本 1 放提交节点或随机的轻负载节点，副本 2 放与副本 1 同机架的另一台机器，副本 3 放另一个机架，再多的副本随机。这样安排是在写入带宽和容错半径之间折中：前两个副本在同一机架内复制，走的是机架内高带宽链路，写得快，并且已经能扛住单机故障；第三个副本跨机架，能扛住整机架断电或交换机故障。如果三个副本全跨机架，写入代价翻倍；全在一个机架，机架一坏就全丢。

**Q4. HDFS 的块为什么从 64MB 改到 128MB？**

按最佳传输损耗理论，寻址时间应该占总传输时间的 1%。寻址时间按 10ms 算，传输时间就该是 1 秒。磁盘顺序读写速度从早年的几十 MB/s 涨到 100MB/s 以后，1 秒能传 100MB，块大小取 128MB 比较合适；如果用 500MB/s 的 SSD，对应的块大小就是 512MB。

**Q5. Secondary NameNode 是 NameNode 的热备吗？它的作用是什么？**

不是。它不能在 NameNode 故障时快速接管服务，NameNode 出错期间集群无法提供任何服务。它做的是定期拉取 NameNode 的元数据快照（fsimage）和操作日志（editlog），在线把日志合并进快照，再把合并后的快照传回 NameNode。这样做的价值在于保证元数据不丢：软件故障重启就行，硬件故障导致元数据丢失时，可以用 Secondary 上的快照重建 NameNode。课件的原话是"数据安全比服务连续性更重要"。

**Q6. 什么是 Head-of-line Scheduling 和 Sticky Slots？延迟调度怎么解决？**

Head-of-line 是队首阻塞：严格按"运行任务最少的作业优先"排队，小作业总排在队首，而小作业的数据只在少数节点上，本地性上不去，一个只占 10% 节点的作业本地性最高就是 10%。Sticky Slots 是槽位粘滞：作业 $i$ 在节点 $k$ 上完成一个任务后运行数减 1 变成最少，节点 $k$ 的空槽又分回给它，导致每个作业被固定在同一批节点上挪不动。

延迟调度的做法是：节点请求任务时，如果队首作业在这个节点上没有本地数据就跳过它去看下一个作业；为了不饿死，给每个作业记一个 `skipcount`，跳过次数达到 $D$ 就允许它启动非本地任务。代价是少量等待时间，收益是本地性按 $1 - (1-p_j)^D$ 指数逼近 1： $p_j = 0.1$ 时 $D = 10$ 能到 65%， $D = 40$ 能到 99%。
