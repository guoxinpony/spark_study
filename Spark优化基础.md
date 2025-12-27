##### 原则一：避免创建重复的RDD

- 对于同一份数据，只应该创建一个RDD，不能创建多个RDD来代表同一份数据。
- Spark作业会进行多次重复计算来创建多个代表相同数据的RDD，进而增加了作业的性能开销。

```java
// 需要对名为“hello.txt”的HDFS文件进行一次map操作，再进行一次reduce操作。也就是说，需要对一份数据执行两次算子操作。

// 错误的做法：对于同一份数据执行多次算子操作时，创建多个RDD。
// 这里执行了两次textFile方法，针对同一个HDFS文件，创建了两个RDD出来，然后分别对每个RDD都执行了一个算子操作。
// 这种情况下，Spark需要从HDFS上两次加载hello.txt文件的内容，并创建两个单独的RDD；第二次加载HDFS文件以及创建RDD的性能开销，很明显是白白浪费掉的。
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/hello.txt")
rdd1.map(...)
val rdd2 = sc.textFile("hdfs://192.168.0.1:9000/hello.txt")
rdd2.reduce(...)

// 正确的用法：对于一份数据执行多次算子操作时，只使用一个RDD。
// 这种写法很明显比上一种写法要好多了，因为我们对于同一份数据只创建了一个RDD，然后对这一个RDD执行了多次算子操作。
// 但是要注意到这里为止优化还没有结束，由于rdd1被执行了两次算子操作，第二次执行reduce操作的时候，还会再次从源头处重新计算一次rdd1的数据，因此还是会有重复计算的性能开销。
// 要彻底解决这个问题，必须结合“原则三：对多次使用的RDD进行持久化”，才能保证一个RDD被多次使用时只被计算一次。
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/hello.txt")
rdd1.map(...)
rdd1.reduce(...)
```



##### 原则二：尽可能复用同一个RDD

- 对不同的数据执行算子操作时还要尽可能地复用一个RDD
- 比如说，有一个RDD的数据格式是key-value类型的，另一个是单value类型的，这两个RDD的value数据是完全一样的。那么此时我们可以只使用key-value类型的那个RDD，因为其中已经包含了另一个的数据。
- 对于类似这种多个RDD的数据有重叠或者包含的情况，我们应该尽量复用一个RDD，这样可以尽可能地减少RDD的数量

```java
// 错误的做法。

// 有一个<Long, String>格式的RDD，即rdd1。
// 接着由于业务需要，对rdd1执行了一个map操作，创建了一个rdd2，而rdd2中的数据仅仅是rdd1中的value值而已，也就是说，rdd2是rdd1的子集。
JavaPairRDD<Long, String> rdd1 = ...
JavaRDD<String> rdd2 = rdd1.map(...)

// 分别对rdd1和rdd2执行了不同的算子操作。
rdd1.reduceByKey(...)
rdd2.map(...)

// 正确的做法。

// 上面这个case中，其实rdd1和rdd2的区别无非就是数据格式不同而已，rdd2的数据完全就是rdd1的子集而已，却创建了两个rdd，并对两个rdd都执行了一次算子操作。
// 此时会因为对rdd1执行map算子来创建rdd2，而多执行一次算子操作，进而增加性能开销。

// 其实在这种情况下完全可以复用同一个RDD。
// 我们可以使用rdd1，既做reduceByKey操作，也做map操作。
// 在进行第二个map操作时，只使用每个数据的tuple._2，也就是rdd1中的value值，即可。
JavaPairRDD<Long, String> rdd1 = ...
rdd1.reduceByKey(...)
rdd1.map(tuple._2...)

// 第二种方式相较于第一种方式而言，很明显减少了一次rdd2的计算开销。
// 但是到这里为止，优化还没有结束，对rdd1我们还是执行了两次算子操作，rdd1实际上还是会被计算两次。
// 因此还需要配合“原则三：对多次使用的RDD进行持久化”进行使用，才能保证一个RDD被多次使用时只被计算一次。
```



##### 原则三：对多次使用的RDD进行持久化

- 保证对一个RDD执行多次算子操作时，这个RDD本身仅仅被计算一次
- 每次你对一个RDD执行一个算子操作时，都会重新从源头处计算一遍，计算出那个RDD来，然后再对这个RDD执行你的算子操作。
- **对多次使用的RDD进行持久化**: Spark就会根据你的持久化策略，将RDD中的数据保存到内存或者磁盘中。以后每次对这个RDD进行算子操作时，都会直接从内存或磁盘中提取持久化的RDD数据，然后执行算子，而不会从源头处重新计算一遍这个RDD

```java
// 如果要对一个RDD进行持久化，只要对这个RDD调用cache()和persist()即可。
// cache(): 等同于 persist(StorageLevel.MEMORY_ONLY)
// persist()：功能更强大，允许你手动指定存储级别（Storage Level）

// 正确的做法。
// cache()方法表示：使用非序列化的方式将RDD中的数据全部尝试持久化到内存中。
// 此时再对rdd1执行两次算子操作时，只有在第一次执行map算子时，才会将这个rdd1从源头处计算一次。
// 第二次执行reduce算子时，就会直接从内存中提取数据进行计算，不会重复计算一个rdd。
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/hello.txt").cache()
rdd1.map(...)
rdd1.reduce(...)

// persist()方法表示：手动选择持久化级别，并使用指定的方式进行持久化。
// 比如说，StorageLevel.MEMORY_AND_DISK_SER表示，内存充足时优先持久化到内存中，内存不充足时持久化到磁盘文件中。
// 而且其中的_SER后缀表示，使用序列化的方式来保存RDD数据，此时RDD中的每个partition都会序列化成一个大的字节数组，然后再持久化到内存或磁盘中。
// 序列化的方式可以减少持久化的数据对内存/磁盘的占用量，进而避免内存被持久化数据占用过多，从而发生频繁GC。
val rdd1 = sc.textFile("hdfs://192.168.0.1:9000/hello.txt").persist(StorageLevel.MEMORY_AND_DISK_SER)
rdd1.map(...)
rdd1.reduce(...)
```

**如何选择持久化级别：**

1. 默认情况下，**性能最高**的当然是**MEMORY_ONLY**，但前提是你的内存必须足够足够大。对这个RDD的后续算子操作，都是基于纯内存中的数据的操作，不需要从磁盘文件中读取数据，性能也很高。在**实际的生产环境**中，恐怕能够直接用这种策略的场景还是有限的，如果RDD中数据比较多时（比如几十亿），直接用这种持久化级别，会导致JVM的**OOM内存溢出**异常。
2. 如果使用MEMORY_ONLY级别时发生了内存溢出，那么建议尝试使用**MEMORY_ONLY_SER**级别。该级别会将RDD数据序列化后再保存在内存中，大大减少了对象数量，并降低了内存占用。这种级别比MEMORY_ONLY多出来的性能开销，主要就是**序列化与反序列化的开销**。还是可能会导致**OOM内存溢出**的异常。
3. 如果纯内存的级别都无法使用，那么建议使用**MEMORY_AND_DISK_SER**策略，而不是MEMORY_AND_DISK策略。序列化后的数据比较少，可以**节省内存和磁盘**的空间开销。同时该策略会优先尽量尝试将数据缓存在内存中，内存缓存不下才会写入磁盘。
4. 通常**不建议**使用**DISK_ONLY**和后缀为2的级别：会导致**性能急剧降低**，后缀为_2的级别，数据复制以及网络传输会导致较大的性能开销，除非是要求作业的高可用性，否则不建议使用。



##### 原则四：尽量避免使用shuffle类算子

- 尽可能避免使用**reduceByKey、join、distinct、repartition**等会进行shuffle的算子，尽量使用map类的非shuffle算子。

```java
// 传统的join操作会导致shuffle操作。
// 因为两个RDD中，相同的key都需要通过网络拉取到一个节点上，由一个task进行join操作。
val rdd3 = rdd1.join(rdd2)

// Broadcast+map的join操作，不会导致shuffle操作。
// 使用Broadcast将一个数据量较小的RDD作为广播变量。
val rdd2Data = rdd2.collect()
val rdd2DataBroadcast = sc.broadcast(rdd2Data)

// 在rdd1.map算子中，可以从rdd2DataBroadcast中，获取rdd2的所有数据。
// 然后进行遍历，如果发现rdd2中某条数据的key与rdd1的当前数据的key是相同的，那么就判定可以进行join。
// 此时就可以根据自己需要的方式，将rdd1当前数据与rdd2中可以连接的数据，拼接在一起（String或Tuple）。
val rdd3 = rdd1.map(rdd2DataBroadcast...)

// 注意，以上操作，建议仅仅在rdd2的数据量比较少（比如几百M，或者一两G）的情况下使用。
// 因为每个Executor的内存中，都会驻留一份rdd2的全量数据。
```



##### 原则五：使用map-side预聚合的shuffle操作

- **map-side预聚合:** 在每个节点本地对相同的key进行一次聚合操作
- 其他节点在拉取所有节点上的相同key时，就会大大**减少拉取的数据量**，从而也就减少了磁盘IO以及网络传输开销。
- 在可能的情况下，建议使用**reduceByKey**或者**aggregateByKey**算子来替代groupByKey。因为reduceByKey和aggregateByKey算子都会对每个节点本地的相同key进行预聚合。而groupByKey算子是不会进行预聚合的



##### 原则六：使用高性能的算子

- 使用**reduceByKey/aggregateByKey**替代groupByKey
- 使用**mapPartitions**替代普通map: 一次函数调用会处理一个partition所有的数据; 使用mapPartitions会出现OOM（内存溢出）的问题, 。因为单次函数调用就要处理掉一个partition所有的数据，如果内存不够，垃圾回收时是无法回收掉太多对象的
- 使用**foreachPartitions**替代foreach: 如果是普通的foreach算子，每次函数调用可能就会创建一个数据库连接; 如果用foreachPartitions算子一次性处理一个partition的数据，那么对于每个partition，只要创建一个数据库连接. 实践中发现，对于1万条左右的数据量写MySQL，性能可以提升**30%**以上。
- 使用filter之后进行**coalesce**操作: 对一个RDD执行filter算子过滤掉RDD中较多数据后, 使用coalesce算子，手动减少RDD的partition数量
- 使用repartitionAndSortWithinPartitions 替代 repartition + sort类操作, 该算子可以一边进行重分区的shuffle操作，一边进行排序。



##### 原则七：广播大变量

- 如果使用的外部变量比较大，建议使用Spark的广播功能，对该变量进行广播。
- 广播后的变量，会保证每个Executor的内存中，只驻留一份变量副本，而Executor中的task执行时共享该Executor中的那份变量副本。
- 减少网络传输的性能开销，并减少对Executor内存的占用开销，降低GC的频率。

```java
// 以下代码在算子函数中，使用了外部的变量。
// 此时没有做任何特殊操作，每个task都会有一份list1的副本。
val list1 = ...
rdd1.map(list1...)

// 以下代码将list1封装成了Broadcast类型的广播变量。
// 在算子函数中，使用广播变量时，首先会判断当前task所在Executor内存中，是否有变量副本。
// 如果有则直接使用；如果没有则从Driver或者其他Executor节点上远程拉取一份放到本地Executor内存中。
// 每个Executor内存中，就只会驻留一份广播变量副本。
val list1 = ...
val list1Broadcast = sc.broadcast(list1)
rdd1.map(list1Broadcast...)
```



##### 原则八：使用Kryo优化序列化性能

- Kryo序列化机制比Java序列化机制，*性能**高10倍**左右*。
- 场景： 使用可序列化的持久化策略时；  使用到外部变量时，该变量会被序列化后进行网络传输；所有自定义类型对象，都会进行序列化 

```java
// 创建SparkConf对象。
val conf = new SparkConf().setMaster(...).setAppName(...)
// 设置序列化器为KryoSerializer。
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
// 注册要序列化的自定义类型。
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
```



##### 原则九：优化数据结构

- 使用占用内存较少的数据结构，但是前提是要保证代码的**可维护性**
- 尽量使用字符串替代对象,  使用原始类型（比如Int、Long）替代字符串，使用数组替代集合类型
- 尽量不要使用：对象， 字符串，  集合类型比如HashMap、LinkedList













