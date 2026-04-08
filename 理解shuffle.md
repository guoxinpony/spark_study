## Hadoop、Spark、Hive和Flink在数据处理中涉及的`JOIN`操作

**首先理解一下他们的层级关系**

- hadoop是一个分布式框架，提供了最基础的储存和批处理计算模型。
- Hive是构建在Hadoop之上的数据仓库工具，将SQL查询转换为底层计算作业。join操作对于用户来说是纯sql语法。
- Spark是一个统一的分析引擎，支持流处理、批处理、机器学习等。提供内存计算和更丰富的API（ADD、DataFrame、SQL）。 join操作可以在SQL或者DataFrame API 中完成。
- Flink是一个以流处理为核心的实时计算引擎。提供DataStream和TableSQL API. join是流处理中的关键。



### **MapReduce**

Join在MapReduce中实现方式主要用两种。

1. **Reduce-side Join  :**Map阶段读取所有需要Join的表，为每条数据打上来源标签，并以join key作为新的key输出。Shuffle阶段将所有Key的数据通过网络传输到同一个Reducer。Reduce阶段 在同一个Reducer中，收到来自不同表的数据，进行连接操作。**优点**： 通用，不关心数据大小和排序。**缺点**： Shuffle开销巨大，是性能瓶颈。
1. **Map-side Join：** 利用分布式缓存将小表广播到每个Map任务所在节点，在Map端直接进行连接，无需经过Shuffle和Reduce。**条件**必须有一个表足够小，可以完全加载到内存。



### **Hive**

Hive将`join`的复杂性隐藏在SQL层，由查询引擎自动转换为底层作业(MapReduce、Spark)    SQL `JOIN`语法（INNER, LEFT, RIGHT, FULL, CROSS）。 Hive的核心价值在于其丰富的优化器。

- **Common Join (Reduce-Side Join)**： 默认方式，相当于Hadoop的Reduce-Side Join。性能最差。
- **Map Join**： 对应Hadoop的Map-Side Join。Hive可以自动判断小表。

- **Bucket Map Join**： 如果两个表都以相同方式（相同key、相同桶数量）分桶，并且其中一个桶数是另一个的倍数，可以**避免全表Shuffle**，直接在Map端按桶进行连接，效率极高。
- **Sort Merge Bucket Join**： 在分桶的基础上，每个桶内的数据还按key排序。连接时只需按顺序归并即可，无需在内存中缓存整个桶，可以用于更大表的连接。

- **Skew Join**： 处理数据倾斜的专用优化。Hive能检测到某个key的数据量异常大，会启动两个作业：一个处理倾斜key（将其随机分散），另一个处理正常key，最后合并结果。



### **Spark**

Spark的`join`通过DataFrame API或Spark SQL提供，以其**内存计算和DAG调度**为核心优势。

**DataFrame API中的Join**

```sql
// ========== 1. 基础Join类型（完整写法） ==========
// 内连接
df1.join(df2, df1("key") === df2("key"), "inner")

// 左外连接（left）
df1.join(df2, df1("key") === df2("key"), "left")

// 右外连接（right）
df1.join(df2, df1("key") === df2("key"), "right")

// 全外连接（full）
df1.join(df2, df1("key") === df2("key"), "full")

// 左半连接（leftsemi）- 仅保留左表匹配行，无右表列
df1.join(df2, df1("key") === df2("key"), "leftsemi")

// 左反连接（leftanti）- 仅保留左表不匹配行，无右表列
df1.join(df2, df1("key") === df2("key"), "leftanti")

// ========== 2. 多条件Join（扩展） ==========
// 多等值条件 + 额外过滤条件
df1.join(df2, 
  df1("key1") === df2("key1") && 
  df1("key2") === df2("key2") && 
  df1("dt") === df2("dt") && 
  df1("status") === "active", 
  "left"
)

// 等值+不等值混合条件
df1.join(df2, 
  df1("key") === df2("key") && 
  df1("score") > df2("score_threshold") && 
  df1("create_time") >= df2("start_time"), 
  "inner"
)

// ========== 3. Column对象进阶用法 ==========
// 导入col函数（需import org.apache.spark.sql.functions._）
val joinCond1 = col("t1.key") === col("t2.key")
val joinCond2 = col("t1.version") >= col("t2.min_version")
val fullJoinCond = joinCond1 && joinCond2

// 结合别名+多Column条件
df1.alias("t1").join(df2.alias("t2"), fullJoinCond, "full")

// 动态构建连接条件
val keyCols = Seq("key1", "key2", "key3")
val eqConditions = keyCols.map(c => col(s"t1.$c") === col(s"t2.$c"))
val finalCond = eqConditions.reduce(_ && _)
df1.alias("t1").join(df2.alias("t2"), finalCond, "inner")

// ========== 4. 等值Join简化写法（Seq扩展） ==========
// 单字段简化
df1.join(df2, Seq("key"), "inner")

// 多字段简化（最常用）
df1.join(df2, Seq("key1", "key2", "dt"), "left")

// 结合别名的多字段简化（需字段名完全一致）
df1.alias("t1").join(df2.alias("t2"), Seq("key1", "key2"), "right")

// ========== 5. 不等值Join（完整场景） ==========
// 纯不等值连接（无等值键，需谨慎，易产生笛卡尔积）
df1.join(df2, df1("value") > df2("value"), "inner")

// 范围匹配不等值Join
df1.join(df2, 
  df1("user_id") === df2("user_id") && 
  df1("amount").between(df2("min_amount"), df2("max_amount")), 
  "inner"
)

// 字符串不等值匹配
df1.join(df2, 
  df1("key") === df2("key") && 
  df1("name").like(df2("name_pattern")), 
  "left"
)

```

Spark根据数据大小、分区情况、配置参数等自动选择Join策略：

1.**Broadcast Hash Join**： 当一张表很小时，Spark将其广播到所有Executor节点，与另一张大表在本地进行Hash Join。**性能最佳**。

**2.Shuffle Hash Join**： 如果两表都不小，但其中一张仍然可以放入每个Executor的内存中，则会先按key Shuffle分区，然后在每个分区内进行Hash Join。

3.**Sort Merge Join**:先排序、后合并，适用于大表与大表的关联场景。两表按 Join Key 哈希分区 Shuffle（同 Key 进同一 Task），Task 内分别排序后，双指针遍历匹配 Key 完成连接。

Spark 的 Join 是**宽依赖**操作，需要在集群间移动数据（Shuffle），是性能瓶颈的常见来源。



### **Flink**

Flink的`JOIN`分为批处理和流处理。**批处理JOIN**： 与Spark类似.  

**流处理JOIN**：流是无界的，因此`JOIN`需要定义**时间边界**。Flink通过**状态后端**来保存需要参与`JOIN`的数据。

类型：

1. **Regular Join**：
   - 最常见的双流`JOIN`。每来一条数据，都会与另一流状态中的所有数据进行匹配。
   - **问题**： 状态会无限膨胀。**必须结合TTL**来清理过期状态。
2. **Interval Join**：
   - 为`JOIN`增加了时间范围约束。例如，只JOIN两个流中时间戳相差在1小时内的数据。
   - **优点**： 状态可以自动清理，因为超出时间范围的数据不会被JOIN，也无需保留。
3. **Temporal Join**：
   - 通常指**版本表JOIN**。例如，用交易流去`JOIN`一个汇率变更日志流（CDC流），得到交易发生时准确的汇率。Flink支持`FOR SYSTEM_TIME AS OF`语法。
4. **Lookup Join**：
   - 流表`JOIN`。一个流去查询外部数据库（如MySQL、Redis），类似于广播`JOIN`。通常通过异步IO来提高性能。

**对比总结表**

| 特性             | Hadoop (MapReduce)                  | Hive                               | Spark                                      | Flink (流处理)                      |
| :--------------- | :---------------------------------- | :--------------------------------- | :----------------------------------------- | :---------------------------------- |
| **编程范式**     | 过程式（写Java代码）                | 声明式（SQL）                      | 混合式（SQL/API）                          | 混合式（SQL/API）                   |
| **JOIN核心思想** | Shuffle + Reduce端合并              | 基于MR/Spark的多种优化策略         | 内存优先，多策略自动选择                   | 基于状态和时间的流式连接            |
| **主要JOIN类型** | Reduce-Side, Map-Side               | Common, Map, Bucket, Skew          | Broadcast, Sort-Merge, Shuffle-Hash        | Regular, Interval, Temporal, Lookup |
| **性能关键**     | 减少Shuffle数据量，使用Map-Side     | 分桶、排序、小表自动识别           | 内存、广播、自适应执行                     | 状态大小、TTL、时间语义             |
| **适用场景**     | 底层原理学习，定制化极强计算        | 基于HDFS的离线数据仓库，传统BI分析 | 需要速度的离线分析、交互式查询、微批流处理 | 实时流处理、事件驱动应用、实时ETL   |
| **数据倾斜处理** | 需手动在Partitioner或程序逻辑中处理 | 有Skew Join等优化                  | 有AQE动态倾斜处理                          | 较难处理，需自定义逻辑或增大并行度  |



## Hadoop、Spark、Hive和Flink在数据处理中涉及的Shuffle

**Shuffle本质**：将不同节点的中间结果，按照Key进行重新分区、排序和传输，保证相同Key的数据落到同一个Reducer上进行处理。

### **MapReduce**

所有现代框架Shuffle机制的鼻祖

**Map端：**1.Map输出先写入环形内存缓冲区（默认100MB）2.缓冲区达到阈值（80%）时，会进行快速排序，按分区和Key排序。3.排序后的数据写入本地磁盘的**溢写文件**。4.多个溢写文件合并成一个已分区、已排序的大文件。 5.溢写前可执行本地聚合减少数据量（**可选Combiner**）

**Reduce端**：1.从各个Map任务的服务端**拉取**属于自己的分区数据。2.将所有Map任务的对应分区数据在内存或磁盘中进行**多路归并排序**。3.形成最终有序的输入流，提供给Reduce函数

缺点：1.**磁盘I/O密集**：Map输出和Reduce输入都涉及大量磁盘读写。 2.**网络传输效率低**：每个Reduce任务与每个Map任务都建立HTTP连接。**内存使用保守**：无法充分利用集群内存 **固定Map/Reduce阶段**：计算模式僵化

### **Spark**

Spark Shuffle 是**宽依赖算子触发的跨分区数据重分布机制**，核心作用是让数据按计算需求（如分组、关联）重新组合，是 Spark 作业性能的 “关键瓶颈”.

**核心流程**

**第一步：Map 端 - 数据分区、缓存与落地**

Map Task 处理完原始数据后，不会直接传输，而是先在本地磁盘生成**有序**的输出文件，核心动作：

1. **数据分区**：按分区规则（如 `key` 的哈希值 % Reduce 分区数），将每条数据分配到对应 Reduce Task 的 **“分区桶”**（下游 1 个 Reduce Task 对应 1 个分区桶）；
2. **内存缓存与溢写：**分区后的数据**先写入内存缓冲区**，缓冲区使用率达 **80% 时触发 溢写**：先对缓冲区数据**按 `key` 排序**，再**写入磁盘临时文件**（每个 Map Task 会生成多个溢写文件）；
3. **合并最终文件**：Map Task 处理完所有数据后，将所有溢写文件合并为 **1 个有序的最终数据文件** + **1 个索引文件**（记录每个分区桶在数据文件中的**偏移量 / 长度**，供 Reduce 端定位）。

**第二步：Shuffle 传输 - Reduce 端主动拉取**

Reduce Task 不会被动等待数据，而是主动**从 Map 端拉取对应分区的数据**，核心动作：

1. **定位数据**：Reduce Task 通过 Driver 获取所有 Map Task 的索引文件位置，明确自己需要拉取哪些 Map Task 的哪个分区桶数据；
2. **批量拉取**：向 Map Task 所在 Executor 发送拉取请求，单次拉取数据量由 `spark.reducer.maxSizeInFlight` 控制（默认 48MB），避免单次拉取过大导致 OOM；
3. **本地缓存**：拉取的数据优先存内存，内存不足时写入磁盘（形成 “内存 + 磁盘” 混合存储）。

**第三步：Reduce 端 - 合并排序与计算**

Reduce Task 拉取完所有 Map 端数据后，需先合并再计算，核心动作：

1. **多文件合并排序：**若数据部分在内存、部分在磁盘，先将磁盘数据加载到内存，再按 `key` 合并最终得到一个按 key 有序的 数据集；
2. **执行算子逻辑：**遍历有序数据集，按算子需求处理（如 `reduceByKey` 聚合、`join` 关联、`sortByKey` 输出），最终将结果写入存储（如 HDFS）或传递给下一个 Stage。


Spark 的发展历程中，主要经历了三种 Shuffle 实现

1.**Hash Shuffle (已弃用)**

**核心特点与机制**

- **写入阶段**：每个 Mapper Task 会根据 Reducer 的数量创建 **N 个独立的磁盘文件**（N = Reducer 数量）。
- **文件管理**：每个 Task 会产生 N 个文件，如果 Mapper Task 数量为 M，那么总共会产生 **M × N** 个中间文件。
- **读取阶段**：每个 Reducer Task 需要从所有 Mapper Task 中提取对应分区的数据，即读取 M 个文件。

**问题与缺陷**

1. **文件数量爆炸**：假设 M=1000，N=1000，会产生 **100 万个中间小文件**。这会导致：
   - 大量的随机磁盘 I/O
   - 巨大的文件系统压力（inode 耗尽）
   - 垃圾回收压力大
2. **内存消耗高**：需要同时维护多个文件缓冲区和输出流。
3. **稳定性差**：不适合大规模数据处理。



2. **Sort Shuffle (Spark 1.2+ 的默认实现)**

   - **写入阶段**：
     1. **内存排序与聚合**：数据在内存中按 Partition ID（和可选的 Key）排序。
     2. **溢出到磁盘**：当内存缓冲区满时，会将数据排序后写入**单个临时文件**。
     3. **单个输出文件**：每个 Mapper Task **只产生一个数据文件**和一个索引文件。
   - **读取阶段**：
     1. Reducer 通过索引文件定位到数据文件中自己分区的位置。
     2. 按顺序读取，效率较高（顺序 I/O）。

   **优势**

   1. **文件数量少**：M 个 Mapper Task 产生 **2M 个文件**（数据文件 + 索引文件）。
   2. **内存效率更高**：通过排序和聚合减少内存占用。
   3. **磁盘 I/O 优化**：顺序写入和读取，性能更好。
   4. **支持多种操作**：天然支持 Sort、Sort-based Aggregation、Sort-based Join 等需要排序的操作。

   Sort Shuffle 根据数据大小和操作类型，有三种内部模式：

   | 模式                     | 触发条件                                                     | 特点                                                         |
   | :----------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
   | **Bypass Merge Sort**    | 1. `shuffle map tasks` ≤ `spark.shuffle.sort.bypassMergeThreshold` (默认200) 2. 不是聚合操作 | 类似 Hash Shuffle，但最后会合并成单个文件。性能好，避免排序开销。 |
   | **Serialized Sorting**   | 使用 `Unsafe` 序列化器（如 Kryo）且不需要聚合                | 直接在序列化的二进制数据上排序，减少反序列化开销和内存占用。 |
   | **Deserialized Sorting** | 其他情况（最通用）                                           | 数据以反序列化的 Java 对象形式在内存中处理，支持聚合操作。   |

3. **Tungsten-Sort (Unsafe Shuffle) (Spark 1.4+)**

**核心优化点**

1. **堆外内存管理 (Off-Heap Memory)**
   - 使用 `sun.misc.Unsafe` API 直接操作堆外内存
   - 避免 JVM 垃圾回收的开销
   - 更紧凑的数据存储格式
2. **缓存友好的排序**
   - 使用 8 字节的指针（Partition ID + 内存地址）进行排序，而不是移动实际数据
   - 减少数据移动，提高 CPU 缓存命中率
3. **字节数组序列化**
   - 数据以紧凑的字节数组形式存储，而不是 Java 对象
   - 减少内存占用和序列化/反序列化开销

**Shuffle 对比总结**

| 特性         | Hash Shuffle (旧) | Sort Shuffle (当前) | Tungsten-Sort (集成在 Sort 中) |
| :----------- | :---------------- | :------------------ | :----------------------------- |
| **文件数量** | M × N             | 2M                  | 2M                             |
| **内存使用** | 高                | 中等                | 低（堆外）                     |
| **磁盘 I/O** | 随机读写          | 顺序读写            | 顺序读写                       |
| **排序开销** | 无                | 有                  | 优化过的排序                   |
| **适用场景** | 小规模，分区少    | 通用                | 大数据量，内存敏感             |
| **当前状态** | **已弃用**        | **默认且唯一**      | **优化特性已集成**             |

### **Hive**

Hive本身不实现Shuffle，而是依赖底层执行引擎。

**Hive特有的Shuffle优化：**

- **Bucket Shuffle**：如果表已分桶且桶数匹配，可以避免全局Shuffle
- **Map-side Join**：自动将小表广播，避免Shuffle
- **Skew Join优化**：倾斜Key单独处理，平衡Shuffle负载



### **Flink**

Flink的Shuffle设计围绕流处理特性，强调低延迟和Exactly-Once语义。

本质是将上游算子子任务（Subtask）的输出，按指定规则（哈希、广播、重平衡等）传递给下游算子子任务。

流程

1. **上游数据分区输出**：上游 Subtask 处理数据后，按 Shuffle 策略（如哈希分区、广播）将数据划分到不同输出通道，缓存至内存缓冲区（避免频繁网络传输）；
2. **数据传输**：缓冲区满 / 触发刷写时，数据通过网络（或本地）传输至下游 Subtask 的输入缓冲区，支持 “推（Push）” 模式（上游主动发送）为主；
3. **下游接收处理**：下游 Subtask 从输入缓冲区读取数据，按算子逻辑（聚合、Join 等）处理，完成数据重分布后的计算。

以 推模式”为主（上游主动推送数据给下游），而非 Spark 的 拉模式，适配 Flink 流处理的实时性

**四者对比总结**

| 维度            | MapReduce                          | Spark                      | Hive (MR引擎)           | Flink (流处理)              |
| :-------------- | :--------------------------------- | :------------------------- | :---------------------- | :-------------------------- |
| **设计目标**    | 稳定、可靠的大规模批处理           | 快速、内存计算的通用引擎   | SQL化的数据仓库查询     | 低延迟、高吞吐的流处理      |
| **Shuffle模式** | 全阻塞、全落盘                     | 可落盘、可内存             | 依赖底层引擎            | 流水线、Record-by-Record    |
| **数据交换**    | Map完成后Reduce才拉取              | Stage边界阻塞交换          | 同底层引擎              | 持续流水线交换              |
| **磁盘使用**    | 大量磁盘I/O（Map输出、Reduce输入） | 可能使用磁盘（内存不足时） | 大量磁盘I/O（MR引擎）   | 主要使用内存和网络缓冲      |
| **内存使用**    | 保守（环形缓冲区）                 | 积极（优先内存）           | 保守（依赖引擎）        | 积极（网络缓冲区、状态）    |
| **网络传输**    | HTTP拉取                           | Netty/HTTP，支持压缩       | HTTP拉取                | Netty，Credit-based流控     |
| **排序机制**    | Map端排序，Reduce端归并            | 可选排序或Hash聚合         | Map端排序，Reduce端归并 | 通常不排序（除非Keyed窗口） |
| **反压处理**    | 无天然反压                         | 基于速率限制的背压         | 无天然反压              | **自然反压传递**            |
| **容错机制**    | 重新执行失败的Map/Reduce任务       | 基于RDD血统重新计算        | 同底层引擎              | **分布式快照**              |
| **适用场景**    | 超大规模、稳定的离线批处理         | 交互查询、迭代计算、批处理 | 离线数据仓库查询        |                             |