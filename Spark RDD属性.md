### Spark RDD 五个属性

1. **分区列表 (A list of partitions)**
   - [ ]  RDD 会将数据集切分成多个逻辑上的**分区（Partition）**
   - [ ] **分布式计算的基础**，每一个分区都会由一个单独的任务（Task）在 Executor 上处理。
2. **计算函数 (A function for computing each split)**
   - [ ]  Spark 会对每一个分区进行计算
   - [ ] RDD 并不存储真正的数据，而是存储**计算逻辑**。当你对 RDD 进行**转换**（如 `map`, `filter`）时，实际上是定义了如何在分区上执行这些计算。
3.  **RDD 之间的依赖关系 (A list of dependencies on other RDDs)**
   - [ ] 记录了当前 RDD 是由哪个父 RDD 转换而来的，形成了所谓的**血缘（Lineage）**
   - [ ] 这是 **“弹性（Resilient）”** 的由来。如果某个分区的数据丢失了，Spark 可以根据血缘关系重新计算该分区，而不需要回溯整个数据集
4. **分区器 (A Partitioner for key-value RDDs)**
   - [ ] 对于 `(Key, Value)` 类型的数据，RDD 可以指定如何根据 Key 将数据分发到不同的分区
   - [ ] 决定了数据在集群中的分布方式。常见的有 `HashPartitioner` 和 `RangePartitioner`
5. **优先位置列表 (A list of preferred locations)**
   - [ ]  存储每个分区优先存放在哪些节点上
   - [ ] **移动计算优于移动数据**
   - [ ]  Spark 在调度任务时，会尽量把计算任务分配到数据所在的服务器上，从而**减少网络带宽**的开销，实现**数据本地化**（Data Locality）



### 总结：RDD 到底是什么？

如果用一句话总结，RDD 是一个**只读、分区、记录了计算逻辑且具备容错能力**的分布式对象集合。它**本身不直接存数据**，而是描述了：

1. 数据在哪（优先位置）。
2. 数据怎么分（分区）。
3. 怎么算（计算函数）。
4. 算坏了怎么办（依赖关系）。