**一、定位：先找到瓶颈在哪**

- [ ] 通过Spark Web UI的Jobs和Stages页面，找到**耗时最长的Stage**。
- [ ] 重点看**Task Duration的分布**，如果中位数和最大值差距很大，说明存在长尾效应。
- [ ] 同时关注**Shuffle Read/Write数据量**是否异常大，以及是否存在Spill（内存溢写到磁盘）。
- [ ] 另外用**explain查看物理执行计划**，判断join策略、谓词下推是否合理。

**二、诊断：判断慢的原因是什么**

- [ ] **数据倾斜**是最常见的，少数Task处理的数据量远大于其他Task。
- [ ] **Shuffle过重**也是常见瓶颈，Stage间数据交换量过大导致网络IO和磁盘IO压力大。
- [ ] **执行计划不合理**，比如本该broadcast的小表走了SortMergeJoin，或者谓词没有下推导致全表扫描。
- [ ] 排查**外部IO瓶颈**，比如读HDFS时DataNode负载高、写目标存储时下游压力大。

**三、优化：针对性解决问题**

- [ ] 针对数据倾斜，group by场景用两阶段聚合（先加随机前缀局部聚合，再去前缀全局聚合），join场景如果是大小表就用broadcast join，大大表则对热点key加盐打散。
- [ ] 开启AQE让Spark自动处理skew join。
- [ ] 针对Shuffle过重，减少不必要的shuffle操作，能map side combine就提前聚合，调整`spark.sql.shuffle.partitions`匹配数据量。
- [ ] 针对重复计算，对多次使用的RDD或DataFrame做持久化（persist/cache），选择合适的存储级别避免重复计算开销。

**四、资源与调度层面**

- [ ] 先看Executor页面的GC Time判断内存是否充足，而不是盲目加资源。
- [ ] 根据诊断结果针对性调整executor数量、内存、核数以及并行度。
- [ ] 调度上，开启推测执行（`spark.speculation=true`），让Spark对运行明显慢于其他Task的任务启动备份Task，解决因节点硬件老化或负载不均导致的慢Task。但要注意推测执行解决不了数据倾斜的问题，如果慢是因为数据量本身不均，备份Task拿到的数据一样多，反而浪费资源。