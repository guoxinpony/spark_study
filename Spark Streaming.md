### 核心思想：微批处理（Micro-Batch）

- [ ] Spark Streaming 把连续不断的实时数据流切成一个个很小的时间片段，每一批本质上就是一个小的 RDD
- [ ] Spark Streaming 是微批、延迟略高但吞吐大；Flink 是逐条处理、天然支持事件时间；Structured Streaming 在两者之间做了折中
- [ ] 



### Spark 中的 Watermark 机制

- [ ] Spark 会追踪它见过的最大事件时间 maxEventTime，然后减去你指定的延迟容忍度 outOfOrderness，得到 Watermark;  **Watermark = maxEventTime -  outOfOrderness, 和flink中的watermark计算方式一致**

**和 Flink Watermark 的关键区别**

- [ ] **更新粒度：** Spark 是微批模式，Watermark 在每个微批的数据中找到maxEventTime， 决定是否更新；Flink 的 Watermark 是逐条更新的，每来一条数据都可能推进 Watermark
- [ ] **WaterMark 策略灵活度：** Spark 就只能指定一个字段和一个outOfOrderness，Flink 里可以自定义 WatermarkStrategy
- [ ] **对迟到数据的处理：** Spark Watermark 之内的迟到数据会被正常处理，超过 Watermark 的直接丢弃，没有重新计算的机制；flink中有outOfOrderness + Allowed Lateness + sideOutput三层防线，保证结果的最终正确性；Spark 的 Watermark 延迟通常要设得比 Flink 的大一些，来弥补结果正确性



### Spark Checkpoint 机制

- [ ] Structured Streaming 利用了微批的天然边界, 不需要像 Flink 那样往流里插 Barrier 来对齐，不存在 Barrier 对齐导致的反压问题
- [ ] 每个微批开始前，Spark 会把当前的处理进度写到 Checkpoint 里，包含三类关键信息：**Kafka 的 offset、聚合算子的中间状态、以及已提交的微批编号**
- [ ] 当作业挂掉重启时，Spark 会读取 Checkpoint 目录，找到最后一个成功提交的微批编号，从对应的 Kafka offset 开始重新消费，恢复状态，然后继续处理



### Spark Streaming 端到端 Exactly-Once 的实现

- [ ] Source 端需要可重放,Kafka 天然支持按 offset 重新消费

- [ ] Spark 引擎内部，Structured Streaming 通过 Checkpoint 保证每个微批要么完整执行成功并记录，要么失败后从上一个成功点重做。因为是微批，不存在"处理到一半"的中间态——一个微批是原子的。

- [ ] Sink 端依靠幂等写入或者事务；**幂等写入：**写入的目标支持按主键去重，重复写同一条数据不会产生副作用；**事务写入：** 在每个微批**写入时带上 batchId**，Sink 端通过这个 ID 来判断这批数据是否已经写过了，从而实现去重

- [ ] Flink 在 Checkpoint 触发时让 Sink 执行预提交, Checkpoint 全部完成后再通知 Sink 执行正式提交, 下游消设置**读已提交**的隔离级别

  



### 什么是事务？

**特性：**

- [ ] 事务就是一组操作，**要么全部成功，要么全部失败**，不存在"做了一半"的中间状态，包括ACID 四大特性
- [ ] **原子性（Atomicity）** 是最核心的，就是刚才说的"要么全做要么全不做"。转账的两步操作要么都成功，要么都回滚，不会出现中间态
- [ ] **一致性（Consistency）** 是说事务执行前后，数据必须满足业务规则。转账前后 A 和 B 的余额总和不变，不会凭空多钱或少钱
- [ ] **隔离性（Isolation）** 是说并发事务之间互不干扰。A 给 B 转账的同时 C 也在给 B 转账，两个事务不能互相覆盖对方的结果。隔离性有不同的级别，从低到高依次是**读未提交、读已提交、可重复读、串行化**，**级别越高一致性越强但并发性能越差**。
- [ ] **持久性（Durability）** 是说事务一旦提交成功，结果就永久保存了，即使系统崩溃也不会丢失。通常通过写日志（WAL/Redo Log）来实现



**在Flink 写入 Kafka这个事务中：**

- [ ] **原子性 (Atomicity)：**两阶段提交，预提交阶段数据写入 Kafka 但标记为“未提交”，正式提交阶段只有当所有算子的 Checkpoint 都成功并持久化后，Flink 才会向 Kafka 发送 `commit` 指令；**如果中间任何环节失败，Kafka 中那些预提交的数据永远不会被标记为“已提交”，逻辑上实现了回滚**
- [ ]  **一致性 (Consistency)：** **Exactly-once (精确一次)** 语义，即便发生故障，Flink 恢复后的状态（State）与 Kafka 中的数据偏移量（Offset）是完全同步的
- [ ] **隔离性 (Isolation)：**消费者就只能看到那些被 Flink 正式 `commit` 的数据，从而避免了**脏读**
- [ ]  **持久性 (Durability)：** Flink 端 Checkpoint 将算子状态持久化，Kafka 端 通过 **WAL (预写日志)** 和 **副本机制 (Replication)** 保证数据不会丢失



### Spark Streaming 的流流 Join 

**为什么流流 Join 必须配合 Watermark**

- [ ] 核心问题是状态无限增长。流流 Join 的本质是：左边来一条数据，去右边的历史数据里找匹配项；右边来一条数据，去左边的历史数据里找匹配项。如果不设限制，两边的状态会一直膨胀，最终 OOM。
- [ ] Watermark 的作用就是告诉引擎"多老的数据可以不用等了"，从而让引擎安全地清理过期状态。



**和Flink 流流 Join的区别**

- [ ] Spark 的流流 Join 必须靠 Watermark 加上你手写的时间范围条件来推算状态什么时候可以清理，Flink 的 Interval Join 根据 between 范围自动管理，Flink 的 Regular Join 则依赖手动配置 TTL
- [ ]  **Join 触发时机：** Spark 是在每个微批结束时，把这一批左边的数据和右边的历史状态做匹配，再把这一批右边的数据和左边的历史状态做匹配；Flink 是逐条触发的，左边来一条立刻去右边状态里找，找到就立刻输出









