### Spark driver, executor 做什么

##### Driver: 运行着应用的 `main()` 函数并创建 `SparkContext`

1. **解析代码，生成执行计划：**Spark 代码转换成逻辑执行计划，再优化成 DAG（有向无环图）；然后将 DAG 切分成多个 **Stage (阶段)**，每个 Stage 再拆分成一组 **Task (任务)**。
2. **资源申请：**向 Cluster Manager (如 YARN, K8s, Standalone) 申请资源以启动 Executor。
3. **任务调度** ：Driver根据**“数据本地性”原则**把每个 task 分发给集群上各个 Executor 执行。
4. **状态监控：**监控 Task 的执行情况。如果某个 Task 失败了，Driver 负责重新调度（重试）。

##### Executor: Executor 是运行在工作节点 (Worker Node) 上的一个 **JVM 进程**。一个 Worker 节点可以运行多个 Executor。

1. **执行 Task：**接收 Driver 发来的 多个 Task，反序列化代码并在本地运行。
2. **数据存储 (BlockManager):** 负责数据的**缓存** (Caching/Persisting) 和 **Shuffle** 过程中的中间数据存储。它有一部分内存专门用来存 RDD/DataFrame。
3. **状态汇报:** 定期向 Driver 汇报自己的健康状态 (Heartbeat) 和任务执行进度。

##### Spark Driver & Executor 工作流程

1. **阶段一：启动与注册**

   **启动**：Driver 启动并初始化 `SparkContext`

   **申请资源:** Driver 向 Cluster Manager (如 YARN ResourceManager) 申请资源。

   **启动 Executor:** Cluster Manager 在各个 Worker 节点上启动 Executor 进程。

   **反向注册:** Executor 启动后，第一时间主动连接 Driver 并进行注册 (Register)。

2. **阶段二：任务执行**

   **发送任务:** Driver 根据 DAG 划分好 Task，通过网络将 **序列化的代码** (即你要执行的函数) 和 **元数据** 发送给指定的 Executor。

   **执行:** Executor 接收到 Task，在自己的线程池中运行代码。

   **心跳与监控:** Executor 运行过程中，会定期向 Driver 发送 **Heartbeat (心跳)**和执行进度

3. **阶段三：结果和资源回收**

   **Shuffle 数据传输:** Executor 之间会直接通信传输数据

   **结果回传：**Executor 将计算结果传回给 Driver。

   **资源回收:** 应用程序结束，Driver 通知 Cluster Manager 关闭 Executor，释放资源。





### 宽依赖，窄依赖

**宽/窄依赖决定了 Spark 如何把一个大 Job 切分成小的Stage，也决定了需不需要进行昂贵的 Shuffle。**

**窄依赖：父 RDD 的一个 partition 只被一个子 RDD 的 partition 使用。**

- 典型操作：map， filter，flatMap，mapPartitions，coalesce(n, shuffle = false)
- 为什么叫做窄依赖narrow dependency: 源自 RDD lineage 的依赖图形状, 图结构呈现为 细长链条（一对一），依赖路径“窄”，没有分支



![https://media.geeksforgeeks.org/wp-content/uploads/20240503182626/Narrow-dependencies.png?utm_source=chatgpt.com](https://media.geeksforgeeks.org/wp-content/uploads/20240503182626/Narrow-dependencies.png?utm_source=chatgpt.com)

**宽依赖：父 RDD 的一个 partition 被子 RDD的多个partition 使用 → 必须 shuffle。**

- 必须要跨节点重分布数据（shuffle）；数据不能 以pipeline的形式处理，需要划分新 Stage；
- 典型操作：groupByKey，reduceByKey，sortByKey，join
- 为什么叫做宽依赖wide dependencies: 源自 RDD lineage 的依赖图形状, 图结构呈现为 宽扇形扩散（一对多）, 依赖范围很“宽”，会产生大规模数据重新分布 (shuffle). 

<img src="https://images.openai.com/static-rsc-1/506aLCtizx-GdqBL0IcAuyOMmHp1UL-BNNJSv84R1s6Js6mQlAUq1-8MQoizGyQS5xwJCNB9ZP7o0TnuunpvnIo00llC4swDYEk-HYJCTEoruaXu92cdxwOPB8ZVi4hdMaDnocw9aenDIU2nQ59Kdw?utm_source=chatgpt.com" alt="In SPARK, why Narrow Dependency strictly doesn't require schuffle over the  network? - Stack Overflow" style="zoom: 150%;" />

##### 为什么要区分宽窄依赖？ 

1. Stage 的切分点：

   **窄依赖** (**Pipeline** 式执行): Spark 会把连续的窄依赖算子（比如 `map -> filter -> map`）合并到同一个 Task 中执行。 数据读进内存后，一条条通过这些算子，中间不落盘，效率极高。

   **宽依赖**：spark 必须把前面的任务都做完，数据重新分发（Shuffle）好，才能开始下一个阶段。Stage 0 没做完，Stage 1 不会开始

2. 容错恢复 (Lineage) 如果数据丢失：

   **窄依赖:** 只需要重新计算**丢失的那一个分区**即可。因为父RDD和 子RDD是一对一的关系

   **宽依赖:** 可能会导致**父 RDD 的所有分区都要重新计算**。因为子RDD分区的数据是由前面多个父RDD分区合并而来





### Spark通用执行流程

###### Step 1：构建逻辑计划 (Logical Plan) & 惰性求值

- **RDD 的 Lineage：**当写下 `map`, `filter`, `join` 等代码时，Spark **并不会**立即执行。它只是记下了这些操作的**依赖关系**
- **Lazy Evaluation (惰性求值)：**只有当你调用了 **Action 算子**（如 `collect`, `count`, `saveAsTextFile`）时，Spark 才会真正触发执行。
- 此时产生了一个 **Job**

###### Step 2：DAGScheduler把 Job 切分成多个 Stage

- **DAGScheduler** 根据 RDD 的依赖关系，形成一张 **DAG**， 然后会根据 **宽/窄依赖类型** 把 DAG 切分成多个 **Stage** ；
- **窄依赖 (Narrow Dependency):** 数据不需要跨节点传输，这种操作会被合并在同一个 Stage 里流水线执行（Pipeline）。
- **宽依赖 (Wide Dependency / Shuffle):** 数据需要跨节点重组（如 `reduceByKey`, `groupByKey`, `join`）。**遇到宽依赖，Stage 就会被切断**。

###### Step 3：TaskScheduler：分配Task

- **分配任务：**DAGScheduler 把每个切分好的 Stage 打包成 `TaskSet` 交给 **TaskScheduler**。TaskScheduler 负责把这些 Task分配给 Executor 去执行。
- **本地化调度：**它会根据 Data Locality 原则，尽量把任务发到数据所在的节点

###### Step 4：Executors 执行 Task

- 如果 Stage 处于“Map 端”，Executor 会一边计算一边把结果写到**本地磁盘**（为 Shuffle 做准备）。
-  **Shuffle**：如果 Stage 处于“Reduce 端”，Executor 需要去拉取（Fetch）上一个 Stage 各个节点计算好的数据。

![Apache Spark Explained: Architecture, Internal Flow, and Optimisation Tips](https://substackcdn.com/image/fetch/%24s_%21RGKt%21%2Cf_auto%2Cq_auto%3Agood%2Cfl_progressive%3Asteep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F383f19cc-ec30-49cd-a99f-a2b72a2bed34_1626x1232.png?utm_source=chatgpt.com)





### Spark 分区和机器节点的关系

什么是 Spark 分区：RDD/DataFrame 的最小并行单位，每个 partition 对应 **一个 task**（在某个 executor 上执行），partition 本身不存储在哪个节点，而是执行时由 executor 读取数据生成

**（1）层级关系**

1. **Cluster (集群):** 由多个 Node 组成。
2. **Node (机器节点):** 一台物理机或虚拟机。它提供内存 (RAM) 和 CPU。
3. **Executor (执行器):** 运行在 Node 上的 JVM 进程。一个 Node 可以运行一个或多个 Executor。
4. **Core (核/线程):** 一个Executor 内有多个core/线程。
5. **Task (任务):** 最小的计算单元，一个core在某个时间处理一个 Task，也就是对一个 Partition计算。
6. **Partition (分区):** 数据的逻辑拆分。

##### **（2）一个节点可以处理多个 partition**

- 每个 executor 有多个 core
- 每个 core 一次执行一个 task（对应一个 partition）

例子：

- executor 有 4 cores
- 可以并行执行 4 个 partition（4 个 task）
- 如果有 100 个分区 → executor 会分批执行

##### **3）一个 partition 只能由一个node的一个 executor 的一个 core 在某一时刻执行**

不会并行拆开，同一个 partition 的任务不会被多个 executor 同时处理。

##### **4）partition 会按需在不同节点之间调度**

调度由 Driver 决定，基于：

- 数据 locality（本地读取 HDFS block）
- executor 空闲 core 数
- load balance

#####  为什么不能一对一？

###### **1. 分区数量通常远大于节点数量**

例如：

- HDFS 文件有 2000 个 block → 2000 个 partition
- 集群只有 10 台机器，如果要一对一，根本不可能运行。

###### **2. Executor 根据 core 数决定并行度，而不是 partition 数**

partition 是逻辑切片， 并行度由 executor 的 cores 决定

###### **3. 混合调度和负载均衡要求 partition 可以自由调度**

如果 partition 固定绑定机器，会导致：

- hot spot（某节点超忙）
- shuffle 无法高效分布
- executor 故障后无法重新调度

**Spark 的 partition 是逻辑划分，executor 是执行资源；partition 会动态调度到不同 executor 的 core 上，一个 executor 可执行多个 partition，但一个 partition 在同一时间只能由一个 core 执行。**



### Spark还有哪些shuffle

Hash Shuffle：每个下游分区生成一个文件 → 文件爆炸，已经移除

1. Sort Shuffle Manager ：核心改进是每个 Map Task 最终只生成**一个数据文件**和**一个索引文件**，大大减少了文件数量。包括以下三种模式：

   - [ ] **Bypass Merge Sort Shuffle Writer**：绕过了排序过程，为每个分区写入单独的缓冲区，写满溢写到磁盘，最后将所有分区文件合并成一个大文件。

     触发条件：reduce 分区数 < 200，没有 map-side combine， key 不要求排序

     效果：避免了排序的开销

   - [ ] **Tungsten-Sort：**直接操作**二进制数据**，利用 Spark 的 **Tungsten**内存管理机制，直接在 Page 内存中对二进制流进行排序。

     触发条件：序列化器（如：Kryo ）支持 Relocation，不需要 Map 端聚合

     效果：节省堆外内存，GC 压力极小

   - [ ] **Sort Shuffle Writer** ：使用 `ExternalSorter`，在内存中维护 Java 对象，如果内存不够，会溢写（Spill）到磁盘

     触发条件：不满足上述两种情况时，需要 Map 端聚合 或者 使用了不支持 Relocation 的序列化器

     效果：支持聚合，内存和 GC 开销最大





### Spark executor和task是什么关系

##### **Executor 是“JVM进程”（Process） → 运行 Task 的容器**

- 由集群启动（YARN/K8s）
- 一个节点上可有多个 executor
- 一个 executor 内包含：
  - 多个 CPU cores
  - 一个 JVM 进程
  - 一块统一管理的内存（堆 + off-heap）
  - 缓存、shuffle 文件

##### **Task 是“计算单元”（Unit of Work），是一个线程**

- 1 个 task = 处理 1 个 partition
- task 是最小调度单位
- task 不包含资源（资源由 executor 供给）

##### Executor 与 Task 的真实关系

Executor（4 cores）
    ├── Core 1 → Task 1
    ├── Core 2 → Task 2
    ├── Core 3 → Task 3
    └── Core 4 → Task 4

如果有 100 个 task，则：

- 前 4 个同时执行
- 剩下的 96 个排队，前面执行完再继续处理。**Task 是被调度到 Executor 上执行的工作单位；Executor 提供资源（cores/memory）让多个 task 并行运行。**

**Spark 中 Executor 是 JVM 进程，用来执行任务并提供 CPU/内存资源；Task 是最小执行单元，对应一个分区。一个 Task 在某一时刻只能运行在一个 Executor 的一个 core 上，而一个 Executor 可以同时执行多个 Task（取决于核心数）。Driver 负责调度 Task 到 Executor。**





### Spark reducebyKey 和groupbyKey 有什么区别？是不是优化前/后的 hashShuffle 的区别

###### reducebyKey 和groupbyKey 区别：

1. reducebyKey 和 groupbyKey 的区别在于**是否在 Shuffle 之前进行预聚合（Map-side Combine）**
2. groupByKey Mapper 端**不进行任何计算**，直接把所有数据写入 Shuffle。
3. reduceByKey Mapper 端，在本地对相同 Key 进行一次合并。

###### hashShuffle 优化：

1. Hash Shuffle优化前：每个 Map Task 为每个 Reduce Task 生成一个文件
2. Hash Shuffle优化后：一个 Map Task 里的多个 Core 复用文件组。减少了文件数。





### <u>groupbyKEY 加优化后的hashShuffle, reducebyKey加上优化前的hashshuffle， 这两种是否都可以实现，这两种哪个效率高？</u>

1. 方案A：groupbyKEY 加优化后的hashShuffle：写磁盘时文件数量少，Shuffle 数据量极大；
2. 方案B：reduceByKey + 优化前 HashShuffle：写磁盘时产生大量小文件，Shuffle 数据量极小；
3. 比较：
   - [ ] 磁盘 I/O 和 网络 I/O 的总字节数，由是否 Combine 决定，方案B胜出
   - [ ] Reduce 端的内存压力，由是否 Combine 决定，方案B胜出
   - [ ] 磁盘寻址，由 Shuffle Manager 决定，方案A胜出
4. 优化后 HashShuffle 并没有减少写入磁盘的字节数，只是解决了文件数量过多导致的随机 IO， 因此，**`reduceByKey` + 优化前的 HashShuffle** 效率远高于 **`groupByKey` + 优化后的 HashShuffle**。





### sortshuffle, bypass sortShuffle都有什么作用

**SortShuffleWriter** 和 **BypassMergeSortShuffleWriter** 是两种处理 Map 端数据输出（Write path）的不同机制，它们决定了**Map Task 如何将数据写入磁盘**，以便 Reduce Task 能够读取。

##### SortShuffle ：

1. **写入内存缓冲区：** 数据首先被写入一个内存数据结构（根据是否需要聚合，可能是 `PartitionedAppendOnlyMap` 或 `PartitionedPairBuffer`）。
2. **排序（Sort）：** 当内存不够时，数据会根据 **Partition ID** 进行排序（如果有聚合需求，还会根据 Key 排序）。
3. **溢写（Spill）：** 排序后的数据被溢写（Spill）到磁盘上的临时文件。
4. **归并（Merge）：** 最终，将所有溢写的临时文件和内存中剩余的数据进行**归并排序**（Merge Sort），合并成**一个**大的最终数据文件。
5. **索引文件：** 同时生成一个索引文件（Index File），记录每个 Partition 在由于数据文件中的偏移量（Offset），这样 Reduce Task 就能知道去哪里读属于自己的数据。

##### BypassShuffle ：

**⭐ 触发条件**：

- 下游 reduce 分区数 小于 spark.shuffle.sort.bypassMergeThreshold（默认 200）

- 没有 map-side combine，比如使用 `groupByKey`

工作原理：

1. **直接分区写入：** 它不进行内存缓冲和排序。它会为每个下游 Partition 创建一个独立的临时文件。
2. **分发数据：** Map Task 每处理一条记录，就根据 Partition ID 计算出它该去哪里，直接写入对应的临时文件。
3. **文件拼接（Concatenate）：** 当所有数据写完后，它将这几百个小临时文件直接**拼接**成一个大的最终数据文件（这一步极快，因为只是文件流的拷贝，没有排序开销）。
4. **创建索引：** 同样生成索引文件。





### 除了 byKey聚合类的shuffle算子，还有哪些

- Join 类算子：intersection，join，OuterJoin 
- 排序类算子：需要全局排序，sortByKey，sortBy，orderBy
- 重分区类算子：repartition，coalesce(n, shuffle = true)，distinct（RDD/DataFrame），dropDuplicates（DataFrame），randomSplit（RDD/DataFrame）



### DISTINCT 算聚合类算子吗

distinct 不算 “byKey 聚合类算子”（比如 reduceByKey / aggregateByKey 这种），但它内部会触发一种 特殊的、非业务语义的聚合（去重聚合），因此在实现上属于 Shuffle 聚合类算子。

##### 语义层面：distinct **不属于** byKey 聚合类算子

- byKey 聚合类算子输入要求 `(K, V)`，提供一个聚合函数（sum, count, max ...），语义上的目的：**把 value 根据 key 聚合**
- **DISTINCT：** 只是过滤掉重复的行，返回的仍然是**行集**（Result Set），并没有进行数学计算。

##### 实现层面：distinct **本质上是 reduceByKey 的变种**

- **全量扫描与收集：** 引擎必须查看该列的所有数据。
- **分组/哈希（Shuffle）：** 为了判断是否重复，引擎必须把相同的数据放在一起（通过排序 Sort 或 哈希 Hash）。在分布式系统（如 Spark）中，这会触发 **Shuffle**（洗牌），这是典型的聚合操作特征。
- **保留 Key：** 在每一组相同的数据中，保留一个 Key，丢弃其他的。

```
rdd.map(x => (x, null))        // 提取每个值作为 key
    .reduceByKey((a,b) => a)   // 去重（只保留任意一个）
    .map(_._1)
```

**distinct 在语义上不是聚合算子，但在底层实现上是通过 reduceByKey 做局部 + 全局聚合来实现的，因此从执行机制来说，它属于使用了聚合流程的 shuffle 算子。**





说话抓重点，

专业名词：英文

规划：时间

spark 3.5 的新特性0.