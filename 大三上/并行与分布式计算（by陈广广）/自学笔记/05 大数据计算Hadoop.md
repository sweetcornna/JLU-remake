# 第 5 章 大数据多机计算：Hadoop

整理自 `并行与分布式笔记.pdf` 第 139–153 页（第 5 章），Spark 示例的最后几段排在第 154 页上半部分，也一并收在这里。原稿是学长的 Markdown 笔记导出的 PDF，图片链接已失效，需要图的地方改成了文字描述；Hadoop 和 Spark 的对比表、WordCount 完整代码只存在于截图里，这里已经抄成文字。

## 本章要点

- 大数据在硬件上要解决集群规模、互联、散热能耗、可靠性、异构；在软件上要解决分片、容错、减少节点间通信。
- 谷歌三驾马车分工：GFS 存文件，BigTable 存结构化数据，MapReduce 算。Hadoop 是它们的开源实现。
- HDFS 用 master/slave，**NameNode 管元数据，DataNode 管块**，副本放置靠机架感知。
- YARN 是 Hadoop 2.0 从 MR 里拆出来的资源调度框架，四个组件 **ResourceManager / ApplicationMaster / NodeManager / Container**。
- MapReduce 的输入输出全是 `<k,v>`，map 产中间对，reduce 按 key 汇总。WordCount 要能默写。
- Spark 把数据放内存（RDD），解决 MR 的磁盘 IO 和延迟问题。

## 1 大数据与分布式：基础概念

### 1.1 数据有多大

伴随信息技术发展，接入互联网的用户和终端体量越来越大，产生的数据量已经是天文数字。根据 IDC 预测，到 2025 年全球将能够生产 **175ZB** 的数据。

原稿给的单位换算：

> ZB: Zettabyte; 1ZB=10EB; 1EB=10PB; 1PB=10TB;
> 1ZB=10 Byte

（更正：原文作 `1ZB=10EB; 1EB=10PB; 1PB=10TB`，换算系数写错了。二进制下 $1\mathrm{ZB}=1024\mathrm{EB}$、$1\mathrm{EB}=1024\mathrm{PB}$、$1\mathrm{PB}=1024\mathrm{TB}$，十进制下是 1000 倍。原文 `1ZB=10 Byte` 掉了指数，应为 $1\mathrm{ZB}=10^{21}\ \mathrm{Byte}$。）

这么多数据对传输、存储、计算全提出了挑战，传统的机器和软件已经远不足以处理。所以面向大数据的数据中心，以及跑在上面的各种软件，成了热门话题。

### 1.2 从硬件角度描述大数据（考点）

一两台机器干不了大数据的存储和计算，因此需要：

1. 大量机器的集群构成数据中心。
2. 使用高速互联网络对大量机器进行连接以确保数据传递。
3. 综合考量数据中心的散热问题、能耗问题，以及各方面成本。
4. 集群中硬件发生故障的概率很高，如何确保可靠性。
5. 单一架构的机器难以胜任各种计算类型，考虑异构计算。

可靠性问题展开讲：数据中心的集群往往包含数以万计的计算机，为顾及成本，用的是较为廉价的普通商用硬件。

原稿的试算题：假设集群有 50000 台机器，每台机器有 4 块硬盘，每块硬盘的年度故障率为 1%，平均每天遇到几次硬盘故障？

$$\frac{50000 \times 4 \times 1\%}{365} \approx 5\ \text{次}$$

（算出来是 5.48，原稿取约 5 次。这个数就是"故障是常态"的量化依据，后面 HDFS 的副本、YARN 的任务重启都是冲着它来的。）

### 1.3 数据中心机房网络拓扑（要会画）

原稿此处有图，PDF 中图片已丢失，结构是这样：

```
                    核心交换机
             /          |          \
      接入交换机    接入交换机   ……   接入交换机
        | 节点        | 节点            | 节点
        | ……          | ……              | ……
        | 节点        | 节点            | 节点
```

一句话说明：一个机架内的节点连接到接入交换机，接入交换机再连接到核心交换机。画图时机架内的节点竖着排，上面顶一个接入交换机，所有接入交换机再往上汇到一个核心交换机。

### 1.4 从软件角度描述大数据（考点）

大数据的计算与存储分布在未必可靠的大量计算机组成的集群上，因此需要：

1. **分而治之**，使用分片存储策略和分布式算法对大数据进行存储与处理。
2. 考虑**存储与计算的容错性**，以使得故障发生时造成的损失最小化。
3. 算法设计方面要**尽可能减少节点间通信**（因为这很耗时）。

### 1.5 分布式研究的两个方向（考点）

在单机上存储和处理大数据是不可能的。分布式就是把任务分配到许多节点（机器）上去，是一种借助网络产生的并行方法。在大数据方向，研究分两类：

- **分布式存储**：分布式文件系统、分布式数据库。
- **分布式计算**：本质上是并行计算模型与算法，上一章的 MPI 就可以用于分布式计算。

## 2 谷歌"三驾马车"

谷歌提出三项技术，分别解决**文件存储、结构化数据存储、分布式计算模型**这三个关键问题，由此开启了大数据时代。有些已经停用并有了替代品，但思想还得了解。

| 技术 | 解决什么 | 论文出处 | Hadoop 里的开源实现 |
| --- | --- | --- | --- |
| GFS | 分布式文件系统 | SOSP 2003 | HDFS |
| BigTable | 分布式存储系统（结构化数据） | OSDI 2006 | HBase |
| MapReduce | 分布式运算编程框架 | OSDI 2004 | Hadoop MapReduce |

### 2.1 GFS

Google File System 是一个可扩展的分布式文件系统，适用于大型分布式数据密集型应用程序。它能在廉价的商用硬件上提供容错能力，并为大量客户端提供较高的总体性能。

四条要点：

1. 把一个较大的文件切分成不同的**单元块**。
2. 把每个单元块存储在一个 **ChunkServer** 服务器节点上，并且每一块都会复制在多个 ChunkServer 服务器。
3. 每一个文件包含多少块和哪些块，这些**元数据存储在 GFS Master 服务器上**。
4. 构成一个低成本的分布式存储系统，用来处理数据量非常大的存储场景，为 MapReduce 的大数据处理模型提供输入和输出的存储系统。

### 2.2 BigTable

Bigtable 是一个分布式存储系统，用于管理**结构化数据**，目标是扩展到非常大的规模：数千个商用服务器上的 PB 级数据。Google 的许多项目（60 个）都把数据存在 Bigtable 里，包括网络索引、Google 地球和 Google 财经。

### 2.3 MapReduce

MapReduce 是一种用于处理和生成大型数据集的分布式运算程序编程模型和相关实现。用户指定 **map 函数**来处理键值对（KV）以生成一组中间键值对，以及 **reduce 函数**来合并与同一中间键关联的所有中间值（相当于分组合并），最后得到结果。

### 2.4 键值存储的优缺点（考点）

优点：

- **简单**：数据结构中只有键和值，成对出现，值在理论上可以存放任一数据，并支持大数据存储。
- **快速**：以内存运行模式为主，数据处理快是其最大优势。
- **高效计算**：数据结构简单化，数据集之间的关系简单化，再加上基于内存的数据集计算、分布式计算，形成了高效计算的前提条件。
- **分布式处理**：分布式处理能力使键值数据库具备了处理大数据的能力。

缺点：

- 对值进行**多值查找功能很弱**。
- **缺少约束**容易出错。
- 不容易建立复杂关系。

从缺点能看出来：很多查询、排序、统计等功能需要程序员在业务代码里做编程约束。

## 3 Hadoop

### 3.1 Hadoop 是什么（3 点 + 组成，考点）

原稿列的三点：

1. Hadoop 是**一系列开源软件的集合**，是为大数据的处理而设计的**分布式框架**。
2. Hadoop 将单机扩展到数千台机器，**每台机器都提供本地计算和存储**。
3. Hadoop 通过对应用层故障的检测和处理，在由**不可靠硬件**构成的集群上实现**高可用**。

组成：

- 谷歌三驾马车的开源实现：GFS → **HDFS**，MapReduce → **Hadoop MapReduce**，BigTable → **HBase**（不展开）。
- 其他相关组件：**Hadoop Common**（支持其他 Hadoop 模块的通用实用程序）、**Hadoop YARN**（作业调度和集群资源管理的框架）等。

### 3.2 Hadoop 与云计算的区别和联系

原稿只列了关键词，没有展开，背的时候按关键词扩句即可：

区别：

- **范围与定位**
- **资源利用率**
- **可靠性**

联系：

- **互补性**
- **数据共享与交互**

（整理补充，原文未展开：Hadoop 是大数据处理框架，云计算是按需交付资源的服务模式；两者常一起用，云提供弹性资源，Hadoop 跑在上面处理数据。答题时把第 6 章云计算的定义搬过来对照说。）

## 4 HDFS

### 4.1 HDFS 的假设

HDFS 的设计建立在六条假设上：

1. **硬件故障经常发生**，因此需要能检测故障并快速恢复。
2. 面向**流式数据访问**，为批处理而非用户交互使用，更注重高吞吐而非低延迟，因此并未兼容 POSIX。
3. 针对**大型数据集**，典型文件大小为 GB 到 TB 级，不适合小文件读取，并应当在数百个节点上支持数千万的文件。
4. **简化的一致性模型**：一个文件一旦创建、写入和关闭就不需要更改，除了追加和截断。这样简化了一致性问题且提高了吞吐。
5. **移动计算而非移动数据**，尤其当数据集很大时，这将会减少网络拥塞并提升吞吐。（更正：原文作"这将会较少网络拥塞"，"较少"是"减少"的笔误。）
6. 跨软硬件平台的**可移植性**。

### 4.2 HDFS 是什么（考点）

- HDFS 是一种依照 GFS 设计的分布式文件系统。
- 运行在低成本商业硬件上，提供高容错性。
- 提供高吞吐量访问，支持具有大量数据集的应用程序。
- 运行在**用户态**，并非内核级文件系统。

### 4.3 架构与 NameNode / DataNode（考点，要会画）

HDFS 使用 **master/slave 架构**，集群包含一个 NameNode 和多个 DataNode。

原稿此处有两张图，PDF 中图片已丢失。第一张的结构：

```
                     Client
                        |  TCP/IP Networking
                        +------- NameNode（Metadata）
                        |
   +---------+----------+----------+---------+
DataNode  DataNode  DataNode  DataNode
```

第二张是 Hadoop 官方的 HDFS Architecture 图，要点是：Client 向 NameNode 发 Metadata ops，NameNode 持有 `Metadata (Name, replicas, …)` 这样的记录并对 DataNode 发 Block ops；DataNode 分布在 Rack 1 和 Rack 2 两个机架上，机架之间有 Replication 箭头；Client 的 Read 从 DataNode 直接读，Write 直接写到 DataNode。画图时把"元数据走 NameNode、真实数据走 DataNode"这条线表达出来就够了。

| | NameNode | DataNode |
| --- | --- | --- |
| 数量 | 每个集群一个（也可以有备份） | 多个 |
| 管什么 | 文件系统的**元数据**（命名空间） | 实际存储**块**（Block） |
| 干什么 | 执行命名空间上的操作：打开、关闭、重命名文件和目录；确定块和 DataNode 的映射 | 处理块的创建、删除；处理来自文件系统客户端的读取和写入请求 |
| 角色 | master | slave |

在 HDFS 中，文件被分为一到多个块（Block）。

### 4.4 数据的复制（考点）

HDFS 要在大型集群中可靠地存储很大的文件，做法是**将文件分块，并为每个块生成多个副本**。

- 每个文件的**块大小和副本数量是可配置的**。
- **NameNode 做出有关块复制的所有决定**。它定期从集群中的每个 DataNode 接收 **Heartbeat** 和 **Blockreport**。收到心跳意味着 DataNode 运行正常；Blockreport 包含 DataNode 上所有块的列表。

副本放在哪很关键。同一个机架内的访问速度要快过跨机架访问，但机架也会出现故障，有一定可能整个机架一起挂掉：

- 全放在一个机架上，机架挂掉数据就访问不到了。
- 全放在不同机架上，写操作成本就变得很高。

Hadoop 拥有 **Rack Awareness（机架感知）** 功能，通过它可以制定不同的副本放置策略。

- 例如副本数为 3 时：选择写操作所在机架放置一个副本，另选一个机架放置两个副本。
- 类似地，读取时也会优先选择相同机架上的副本。

原稿此处有 Block Replication 图，PDF 中图片已丢失，内容是 NameNode 里记着 `(Filename, numReplicas, block-ids, …)`，下面八个 DataNode 里散落着编号 1~5 的块，同一编号出现在多个 DataNode 上。

## 5 YARN：调度器

### 5.1 为什么需要 YARN

- 在 Hadoop 中，计算是以**作业（job）** 的形式发布，并被划分为**任务（task）** 的形式执行。
- 计算任务的执行需要使用空闲计算资源（CPU 等硬件资源）。
- YARN 就是用来调度管理计算任务和计算资源的框架。
- 谷歌的 MapReduce 论文中给出了一些对调度器的要求，但并没有明确给出其实现。
- **Hadoop 1.0** 使用 JobTracker 与 TaskTracker 对 MR 任务进行调度，这种任务调度与 MR 框架深度耦合。
- **Hadoop 2.0** 把资源和作业管理部分提取为独立的 YARN 框架，与 MR 解耦，优化了调度方式，还能在其上支持更多的计算模型。

原稿此处有 Hadoop 1.0 与 2.0 的对比图，PDF 中图片已丢失。要点：1.0 里 MapReduce 自带 Scheduler，MapReduce 和 Spark 各占一个集群（Cluster 1 跑 MR + HDFS，Cluster 2 跑 Spark）；2.0 里 MapReduce on YARN 和 Spark on YARN 共用一层 YARN，YARN 下面是同一个 HDFS，同一个 Cluster 1。图上"两个集群变一个集群"就是解耦带来的好处。

### 5.2 YARN 的主要工作（5 点，考点）

记忆线索是题干给的五个词：作业、集群资源、容器、分配、任务。

1. 接受**作业**（应用程序），启动作业，失败时重启作业。
2. 管理**集群资源**（基于容器，囊括了内存、CPU、磁盘、网络等）。
3. 创建、管理、监控各节点上的**容器**（任务执行的微环境）。
4. 为应用程序按需**分配资源**（将任务发放到适当的节点、适当的容器）。
5. 跟踪**任务**的执行，监控任务健康状态，处理任务的失败。

### 5.3 YARN 的架构（4 个组件，考点，要会画）

原稿此处有图，PDF 中图片已丢失。画法：左边两个 Client（App 1、App 2）用虚线箭头把 Job Submission 指向中间的 **ResourceManager**；右边三个方框是三个节点，每个节点里有一个 **NodeManager** 和若干 **Container**，其中两个节点里有 **App Mstr**（ApplicationMaster）；NodeManager 用点划线把 Node Status 汇报给 ResourceManager，App Mstr 用点线向 ResourceManager 发 Resource Request，App Mstr 与 Container 之间是 MapReduce Status 实线。图例四种线：MapReduce Status、Job Submission、Node Status、Resource Request。

| 组件 | 职责 |
| --- | --- |
| **ResourceManager** | 在系统中所有应用程序之间仲裁资源的最终权威，包含 Scheduler 和 ApplicationsManager 两个子组件 |
| ├ Scheduler | 为各种应用程序按需分配资源，资源基于容器。分配策略是**可插拔的**，可以按需选择 |
| └ ApplicationsManager | 接收作业提交，协商用于启动 ApplicationMaster 的容器，并在失败时重启 ApplicationMaster 容器 |
| **ApplicationMaster** | 一个**框架特定的库**（MR 有 MR 的，Spark 有 Spark 的）。每个应用程序一个，是该应用的"管家"。负责与 ResourceManager 协商资源，与 NodeManager 一起执行和监视任务的执行进度和状态 |
| **NodeManager** | 每台机器上的代理。管理容器，监控其资源使用情况（CPU、内存、磁盘、网络）以及 Container 的运行状态，并把这些信息汇报给 ResourceManager 中的 Scheduler |
| **Container** | 动态资源分配单位，把内存、CPU、磁盘、网络等资源封装在一起，限定每个任务使用的资源量。可以理解为 YARN 的资源抽象，相当于一台小型机器，真正运行任务的地方（更正：原文作"限定每个人物使用的资源量"，"人物"是"任务"的笔误） |

## 6 MapReduce 编程模型

### 6.1 模型本身

MapReduce（后文称 MR）是一个用于编写并行大数据处理程序，并使其在集群上可靠运行的编程框架。

- MR 操作的数据存储在分布式文件系统上，也就是在**磁盘**中（在 Hadoop 上就是 HDFS）。
- MR 的输入输出数据形式均为**键值对 `<k,v>`**。
- 一个 MR 作业通常将数据分为多个部分，每个部分分别由 map 操作生成中间值，然后由 reduce 操作对具有相同 key 的所有 value 进行汇总。

三步分工：

1. 用户编写的 **Map 函数**，接受输入对，产出一系列中间 k-v 对。
2. **MR 库**将同一个中间 key 对应的所有中间 value 收集起来，并发送给 Reduce。
3. 用户编写的 **Reduce 函数**，接受中间 key 以及该 key 对应的所有 value 的集合，并将其整合为一个更小的集合（一般只有 0 或 1 个元素）。

### 6.2 示例：字数统计（伪代码）

```text
map(String key, String value):
    // key: document name          # 输入：<文档名, 文档内容>
    // value: document contents
    for each word w in value:      # 遍历文档中每个单词
        EmitIntermediate(w, "1");  # 产生中间对：<单词, "1">

reduce(String key, Iterator values):
    // key: a word                 # 输入：<单词, 数量的列表>
    // values: a list of counts
    int result = 0;
    for each v in values:          # 遍历数量列表，累加所有数量
        result += ParseInt(v);
    Emit(AsString(result));        # 产生结果
```

### 6.3 并行是怎么发生的

Map 与 Reduce 操作的是**数据分片**而非所有数据，因此在各机器、各分片上的操作是并行的：

1. 数据被切分成块。
2. 在各机器（map worker）上启动代码副本，执行 map 操作，读入分块并输出中间值。
3. （对中间值按 key 进行排序。）
4. 在各机器（reduce worker）上启动代码副本，执行 reduce 操作，读入各自 key 对应的中间值并生成结果。

原稿此处有一张"基于 Map 和 Reduce 的并行计算模型"图，PDF 中图片已丢失。流程是：海量数据存储 → 数据划分成若干初始 kv 键值对 → 多个 Map → 多个 Combiner → 中间结果 → Partitioner + Barrier → 多个 Reduce → 计算结果。图上标了一句"示例中不包含 Combiner"。

### 6.4 补充：API 与 ABI 的区别

原稿在这里插了一段 API vs ABI 的说明（和 MR 关系不大，但是考试可能问）：

POSIX 的 `printf()` 函数在 API 层面能保证所有支持 POSIX 标准的系统之间都一样，但它不能保证 `printf` 在实际系统中执行时，是否都遵循从右到左将参数压入堆栈、参数如何在堆栈中分布等这些实际运行时的二进制级别的问题。比如两台计算机，一台是 Intel x86，一台是 MIPS，都装了 Linux，因为 Linux 支持 POSIX 标准，所以两台电脑都支持 `printf`；但实际运行时，两台电脑 `printf` 被调用过程中，关于参数和堆栈分布的细节肯定不一样，甚至调用 `printf` 的指令都不一样。也就是说，**API 相同，ABI 完全可能不同**。

（更正：原文作 `pirntf()`，是 `printf()` 的笔误。）

## 7 Hadoop MapReduce 编程：WordCount

编程语言是 **Java**。本机没有 JDK，下面的代码只做逐行推演，没有实际编译运行过。

### 7.1 需要的包

```java
import java.io.IOException;
import java.util.StringTokenizer;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
```

注意 Hadoop 有自己的可序列化类型：`IntWritable` 对应 int，`Text` 对应 String。

### 7.2 在 main 里建立 Job

```java
public class WordCount { ...
  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    ...
  }
}
```

### 7.3 Mapper

应用程序通过实现 Mapper 接口来提供 map 方法。

- Mapper 将输入键值对映射到一组中间键值对。
- 转换后的中间记录不需要与输入记录的类型相同。**给定的输入对可以映射到零个或多个输出对**。

```java
public static class TokenizerMapper
     extends Mapper<Object, Text, Text, IntWritable>{

  private final static IntWritable one = new IntWritable(1);
  private Text word = new Text();

  public void map(Object key, Text value, Context context
                  ) throws IOException, InterruptedException {
    StringTokenizer itr = new StringTokenizer(value.toString());
    while (itr.hasMoreTokens()) {
      word.set(itr.nextToken());
      context.write(word, one);
    }
  }
}
```

逐行推演：`Mapper<Object, Text, Text, IntWritable>` 四个泛型依次是输入 key 类型、输入 value 类型、输出 key 类型、输出 value 类型。`one` 是常量 1，声明成 `static final` 是为了整个 Mapper 只建一个对象。`StringTokenizer` 默认按空白字符切词，`hasMoreTokens()` / `nextToken()` 逐个取出。每取一个词就 `context.write(word, one)`，也就是发出一个 `<单词, 1>`。注意 `word` 这个 Text 对象是复用的，每次 `set` 覆盖旧值，Hadoop 在 `write` 时已经把内容序列化出去了，所以复用没问题。

用 `Job.setMapperClass(Class)` 把 Mapper 传给 Job，框架会为输入中的每个分块（键值对）调用 map 函数：

```java
job.setMapperClass(TokenizerMapper.class);
```

可以选择使用 **Combiner** 在 Mapper 本地对输出进行聚合，从而减少传输到 Reducer 的数据量：

```java
job.setCombinerClass(IntSumReducer.class);
// IntSumReducer 是稍后编写的 Reducer，这里相当于先在 Map 本地进行了一次 Reduce
```

**需要多少个 Map？** Map 的数量一般由输入的大小（也就是有多少个输入块）决定。比如有 10TB 的输入，块大小（blocksize）是 128MB，就有约 82000 个 map 要运行。（核对过：$10 \times 1024 \times 1024 / 128 = 81920$，和原稿的"约 82000"一致。）

### 7.4 Reducer

应用程序通过实现 Reducer 接口来提供 reduce 方法。Reducer 将同一个 key 所对应的大量中间值进行约简。

```java
public static class IntSumReducer
     extends Reducer<Text,IntWritable,Text,IntWritable> {
  private IntWritable result = new IntWritable();

  public void reduce(Text key, Iterable<IntWritable> values,
                     Context context
                     ) throws IOException, InterruptedException {
    int sum = 0;
    for (IntWritable val : values) {
      sum += val.get();
    }
    result.set(sum);
    context.write(key, result);
  }
}
```

（原稿正文里这段写成了 `Iterable values`，尖括号里的 `IntWritable` 在 Markdown 渲染时被当成 HTML 标签吃掉了。第 152 页的完整代码截图里是 `Iterable<IntWritable> values`，以截图为准。）

逐行推演：`values` 是同一个单词对应的所有 1，`val.get()` 取出 int 值累加，`result.set(sum)` 写回 IntWritable，最后发出 `<单词, 总数>`。

用 `Job.setReducerClass(Class)` 把 Reducer 传给 Job：

```java
job.setReducerClass(IntSumReducer.class);
```

**reduce 分为三个阶段**：

1. 框架首先将所有相关的 map 输出分片取回。
2. 框架将取回的中间值按 key 进行分组、排序。前两步是同时进行的，取回数据时就会进行合并操作。可以用 `Job.setGroupingComparatorClass(Class)`、`Job.setSortComparatorClass(Class)` 控制分组和排序。
3. 以上操作结束后，开始进行 reduce。

**需要多少 Reduce？** Hadoop 建议的数量是

$$0.95 \quad \text{或} \quad 1.75 \ \times\ (\text{节点数} \times \text{每节点最大容器数})$$

- 取 0.95 时，所有 reduce 都能立即启动。
- 取 1.75 时，执行快的节点可以在第一轮结束后开启第二轮 reduce。
- 随着 reduce 数量增加，框架开销会增加，但也会增加负载均衡，降低发生故障后的成本。
- 因数略小于整数是为了给**推测任务**和失败任务留出一些空余。
- 可以用 `Job.setNumReduceTasks(int)` 设定 reduce 数量。

### 7.5 推测执行

- 在集群上分布式执行任务时，总会有一些节点跑得比其他节点慢很多。
- Hadoop 默认不会一直等慢节点跑完。如果发现有的任务执行比平均速度慢，它会尝试开启一个与该任务相同的**推测任务**。
- 原任务和推测任务谁先跑完就用谁，另一个会被终止。
- 推测执行会占用更多集群资源，可以通过配置将其关闭。

### 7.6 完整代码

原稿标注"老师的文档"，以截图形式给出，抄录如下：

```java
public class WordCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

默写时抓住主干：两个静态内部类（Mapper、Reducer）加一个 main。main 里的顺序是 `Configuration` → `Job.getInstance` → `setJarByClass` → 三个 `setXxxClass` → 两个 `setOutputXxxClass` → 输入输出路径 → `waitForCompletion`。`args[0]` 是输入路径，`args[1]` 是输出路径。

## 8 新兴的大数据处理框架：Spark

### 8.1 MapReduce 怎么不够用了（4 点，考点）

MapReduce 解决了在集群上处理大数据的问题，当时看来是突破性的创造。随着技术发展，人们提出了更多要求，最成功的新框架是 Spark（论文 *Resilient Distributed Datasets: A Fault Tolerant Abstraction for In-Memory Cluster Computing*, NSDI 2012）。

MR 的局限性：

1. **编程范式较为局限**，有些复杂任务（如机器学习）可能需要非常多次 MR 任务连接起来才能完成。
2. 每次 MR 任务都要进行**大量磁盘 IO**，没有缓存，执行效率不高。
3. 不适用于**低延迟要求的流式数据处理**场合。
4. **语言支持有限**。

### 8.2 Spark 简介

- 一种专用于大规模数据分析的框架。
- 可置于内存的**弹性分布式数据集 RDD**（以及新版本的 Dataset）。
- 资源调度方面支持批处理或实时流式处理。
- 在大规模数据和集群上进行数据分析与机器学习。
- 多语言支持。

### 8.3 Spark 示例（Scala）

Spark 支持多种编程语言，原稿用 Scala（基于 JVM）展示一个交互式分析案例。首先运行 Spark Shell：

```bash
./bin/spark-shell
```

Dataset 是 Spark 中的一个主要抽象，是一个分布式的数据项目集合。它可以从 HDFS 或以其他方式输入。创建一个基于文本文件 `README.md` 的 Dataset：

```scala
scala> val textFile = spark.read.textFile("README.md")
textFile: org.apache.spark.sql.Dataset[String] = [value: string]
```

可以从 Dataset 使用操作直接获取数据，或对 Dataset 进行转换。这里进行**操作（action）**：

```scala
scala> textFile.count() // Number of items in this Dataset
res0: Long = 126        // May be different from yours as README.md will
                        // change over time, similar to other outputs

scala> textFile.first() // First item in this Dataset
res1: String = # Apache Spark
```

对 Dataset 进行**转换（transformation）**，这里用过滤器（filter），从子集产生一个新 Dataset：

```scala
scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
linesWithSpark: org.apache.spark.sql.Dataset[String] = [value: string]
```

Transformation 和 action 可以链接起来：

```scala
scala> textFile.filter(line => line.contains("Spark")).count()
// How many lines contain "Spark"?
res3: Long = 15
```

通过 map 和 reduce 操作，找到单词数量最多的行：

```scala
scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
res4: Long = 15
```

以 MapReduce 的方式进行 WordCount：用 `flatMap` 将行集合转为单词集合，然后用 `groupByKey` 进行聚合，使用 `count` 计算每个单词的数量：

```scala
scala> val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]

scala> wordCounts.collect()
res6: Array[(String, Int)] = Array((means,1), (under,2), (this,3), (Because,1), (Python,2), (agree,1), (cluster.,1), ...)
```

拿它和第 7 节的 Java 版 WordCount 比一下：Java 版要写两个内部类加一个 main，Spark 一行 `flatMap().groupByKey().count()` 就够了。答"编程范式较为局限"那一条时可以举这个例子。

### 8.4 Hadoop 和 Spark 对比（6 点，考点）

原稿以表格截图给出，抄录如下：

| | Hadoop | Spark |
| --- | --- | --- |
| **类型** | 基础平台，包含计算、存储、调度 | 纯计算工具（分布式） |
| **场景** | 海量数据批处理（磁盘迭代计算） | 海量数据的批处理（内存迭代计算、交互式计算）、海量数据流计算 |
| **价格** | 对机器要求低，便宜 | 对内存有要求，相对较贵 |
| **编程范式** | Map + Reduce，API 较为底层，算法适应性差 | RDD 组成 DAG 有向无环图，API 较为顶层，方便使用 |
| **数据存储结构** | MapReduce 中间计算结果在 HDFS 磁盘上，延迟大 | RDD 中间运算结果在内存中，延迟小 |
| **运行方式** | Task 以进程方式维护，任务启动慢 | Task 以线程方式维护，任务启动快，可批量创建提高并行能力 |

## 9 易混对比

### 9.1 谷歌技术与 Hadoop 组件的对应

| 谷歌 | Hadoop | 干什么 |
| --- | --- | --- |
| GFS | HDFS | 分布式文件系统 |
| BigTable | HBase | 结构化数据存储 |
| MapReduce | Hadoop MapReduce | 分布式运算编程框架 |
| （论文只提要求，未给实现） | YARN | 资源调度与作业管理 |

### 9.2 Hadoop 1.0 与 2.0

| | Hadoop 1.0 | Hadoop 2.0 |
| --- | --- | --- |
| 调度组件 | JobTracker + TaskTracker | YARN（RM / AM / NM / Container） |
| 与 MR 的关系 | 深度耦合 | 解耦 |
| 能跑什么 | 只有 MR | MR on YARN、Spark on YARN 等多种计算模型 |
| 集群划分 | 不同框架各占一个集群 | 多框架共用同一集群和同一 HDFS |

### 9.3 Combiner 与 Reducer

| | Combiner | Reducer |
| --- | --- | --- |
| 跑在哪 | Map 本地 | Reduce 节点 |
| 目的 | 减少传输到 Reducer 的数据量 | 得到最终结果 |
| 代码 | WordCount 里直接复用 `IntSumReducer` | `IntSumReducer` |
| 是否必须 | 可选 | 必须 |

### 9.4 map 数量与 reduce 数量

| | map | reduce |
| --- | --- | --- |
| 由什么决定 | 输入的大小（有多少个输入块） | 人为设定 |
| 计算方式 | 输入总量 ÷ blocksize | $0.95$ 或 $1.75 \times (\text{节点数} \times \text{每节点最大容器数})$ |
| 设定接口 | 由框架决定 | `Job.setNumReduceTasks(int)` |

### 9.5 容错三件事在本章的体现

| 机制 | 层次 | 做什么 |
| --- | --- | --- |
| 副本 + 机架感知 | HDFS 存储 | 一个块存多份，跨机架放 |
| Heartbeat + Blockreport | HDFS 管理 | NameNode 判断 DataNode 死活、掌握块分布 |
| 作业重启 / 任务失败处理 | YARN 调度 | 作业失败时重启，AM 容器失败时由 ApplicationsManager 重启 |
| 推测执行 | MR 执行 | 慢任务另起一份，谁先完用谁 |

## 10 复习自测清单

以下是原笔记作者在第 139 页列的考点，原样保留。

- 从硬件角度描述大数据
- 从软件角度描述大数据
- 画出常见的数据中心机房网络拓扑并说明
- 在大数据方向，分布式的研究可以分为哪两类方向
- 分别介绍谷歌的"三架马车"（都要会详细说明）
- 介绍GFS
- 介绍BigTable
- 介绍MapReduce
- 说明键值存储的优点与缺点
- 介绍Hadoop（3点+组成）
- 画图介绍HDFS是什么以及架构
- 介绍NameNode以及DataNode
- 说明HDFS如何实现数据的复制
- 简述YARN的主要工作（5点，作业，集群资源，容器，分配，任务）
- 画图说明Hadoop YARN的架构

正文里还有两处标了考点记号，一并列在这里：

- MapReduce怎么不够用了？（MapReduce的局限性，4点）
- Hadoop和Spark对比（6点）
