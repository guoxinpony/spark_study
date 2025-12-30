### Spark 任务划分，调度和重试

##### 1. Stage Task 划分

**Application 应用**：

1. 用户编写的 Spark 程序
2. 一个 Application 对应一个 `SparkContext` 实例

**Job：**

1. 每触发一次 **Action 算子**（如 `collect()`, `count()`, `saveAsTextFile()` 等），Spark 就会提交一个 Job

**Stage：**

1. 根据 **宽/窄依赖**划分stage, 如果是窄依赖（map, flatMap, filter），这些操作可以以Pipeline的形式连续执行，**不会断开 Stage**；如果是宽依赖（reduceByKey, groupByKey），切分，划分为新的 Stage
2. stage 从最后的 RDD 开始**从后往前**回溯, 遇到窄依赖，将其并入当前 Stage，遇到宽依赖，就地“砍断”，划分为新的 Stage

**Task：**

1. Spark 中最小的执行单元，被发送到 Executor 上的线程中执行
2. **最后一个 RDD 的分区个数** 就是该 Stage 的 Task 个数



##### 2. Spark 任务调度

1. **DAGScheduler：**根据宽/窄依赖划分stage, 再根据RDD分区数划分task, 将task打包成一个**TaskSet**，发送给TaskScheduler
2. **TaskScheduler：**为每一个 TaskSet 创建一个对应的 **TaskSetManager**。它负责该 Stage 内所有 Task 的状态**监控、重试逻辑以及本地化级别计算**，根据配置（**FIFO** 或 **FAIR** ），决定多个 TaskSet 之间的**执行优先级**
3. **TaskScheduler：**根据**数据本地化（Data Locality）**原则，尽量将 Task 分配给离数据最近的 Executor；进程本地 (**PROCESS_LOCAL**) > 节点本地 (**NODE_LOCAL**) > 机架本地 (**RACK_LOCAL**) > 任意 (**ANY**)
4. **RPC 通信**：通过 **RpcEnv**（内部包含 **Inbox** 消息收件箱和 **Outbox** 发件箱）将**序列化后的 Task** 发送到对应节点的 **ExecutorBackend**，ExecutorBackend 接收到 Task 后进行**解压和反序列化**，最后由 Executor 的线程池 (Thread Pool) 分配一个线程来运行 Task 逻辑。
5. **总结：** **DAGScheduler** 负责“做什么”（逻辑划分），**TaskScheduler** 负责“在哪做”（物理调度），而 **Executor** 负责“开始做”（实际计算）



##### 3. 失败重试与黑名单机制

1. **Task运行失败：**会被告知给TaskSetManager，对于失败的Task，会记录它失败的次数，如果失败次数还没有超过最大重试次数，那么就把它放回待调度的Task池子中，否则整个Application失败。
2. **黑名单机制：**失败同时会记录它上一次失败所在的Executor Id和Host，避免它被调度到上一次失败的节点上，起到一定的容错作用



##### 4. 容错机制：推测执行（时间换空间，解决task运行时间长尾效应）

**流程：**

1. 如果某个 Task 的运行时间明显超过了该 Stage 中 Task 运行时间的平均值或者阈值，Driver 就会认为这个任务遇到了“性能瓶颈”
2. Driver 会在**另一个** Executor（通常是在不同的节点上）启动一个完全相同的 Task 副本。
3. **胜者为王**：两个相同的 Task 同时运行。哪一个先执行完，Driver 就会接受哪一个的结果，并且**立即发送指令 kill 掉另外一个**



**为什么原任务会有问题：**

1. **硬件故障/性能损耗**
2. **节点负载过高：**该 Executor 所在的物理机上运行了其他消耗资源的进程，导致资源竞争。
3. **Java GC 停顿：**原线程所在的 Executor 正在经历长时间的 Full GC
4. **数据倾斜：**启动新线程可能也无法解决问题，这是推测执行的一个局限性。



**局限性：**

1. **浪费资源**：同时运行两个相同任务，意味着消耗了双倍的 CPU 和内存。
2. **写冲突**：如果任务涉及写文件，需要 Spark 内部确保只有一个任务能成功提交输出，否则会导致数据重复或损坏。
3. 只能解决因环境问题导致的长时间运行，不能解决数据倾斜导致的长时间运行



##### 5. Task 执行优先级：FIFO先进先出

**核心逻辑：**

1. 优先级由 **Job ID** 和 **Stage ID** 决定。**ID 越小**的任务集，表示提交越早，**优先级越高**。
2. 高优先级的 TaskSet 会**抢占所有可用的资源（CPU Slots）**。只有当队列头部的**任务集不需要全部资源**，或者**执行完毕释放**了资源时，后面的任务集才有机会获得资源运行。



**层级关系**：

1. **Job 级别**：如果你在一个 Spark 程序中用多线程同时触发了**两个 Action**（例如两个 `collect`），默认情况下，第一个触发的任务会“霸占”集群资源，直到它处理完核心阶段。
2. **Stage 级别**：在一个 Job 内部，父 Stage 一定先于子 Stage 提交，因此也遵循 FIFO。



##### 6. Task 执行优先级：FAIR公平调度

**核心逻辑：资源的动态平衡**

1. **资源共享**：如果集群中有多个 Job 在运行，FAIR 调度会尝试给每个 Job 分配大致相等的 CPU 资源。
2. **对短任务友好**：如果一个长作业正在运行，此时提交了一个短作业，短作业能够迅速获取到一部分闲置资源并开始运行，而不必等待长作业完全结束。

**工作原理：**

`TaskScheduler` 在分发任务（receiveOffers）时，会查看所有活跃的 `TaskSetManager`。它会根据每个任务集已获得的资源量进行排序，优先把资源给到那些“目前分得最少”的任务集。





