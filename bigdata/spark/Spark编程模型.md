# Spark 编程模型

&emsp;&emsp;与Hadoop相比，Spark最初为提升性能而诞生。Spark是Hadoop Mapreduce的演化和改进，并兼容并包了一些数据库的基本思想，因此可以说，Spark一开始就站在Hadoop与数据库这两个巨人的肩膀上。与此同时，Spark依靠Scala强大的函数式编程，Actor通信模式，闭包，容器，泛型，并借助统一资源调度框架，使得Spark成为一个简洁、高效、强大的分布式大数据处理框架。

&emsp;&emsp;Spark在运算期间，将输入数据与中间计算结果保存在内存中，直接在内存中计算。另外，用户也可以将重复利用的数据缓存在内存中，缩短数据读写时间，以提高下次计算的效率。显而易见，Spark基于内存计算的特性使其擅长于迭代式与交互式任务，但也不难发现，Spark需要大量内存来完成计算任务。集群规模与Spark性能之间呈正比关系，随着集群中机器数量的增长，Spark的性能也呈线性增长。接下来将对Spark编程模型进行详细地介绍。

## 2.1 RDD弹性分布式数据集

&emsp;&emsp;通常来讲，针对数据处理有几种常见模型，包括：Iterative Algorithms，Relational Queries，MapReduce，Stream Processing。例如：Hadoop MapReduce采用了MapReduce模型，Storm则采用了Stream Processing模型。

&emsp;&emsp;与许多其他大数据处理平台不同，Spark建立在统一抽象的RDD之上，而RDD混合了上述这四种模型，因此使得Spark能以基本一致的方式应对不同的大数据处理场景，包括MapReduce，Streaming，SQL，Machine Learning以及Graph等。这契合了Matei Zaharia提出的原则：“设计一个通用的编程抽象(Unified Programming Abstraction)”，这也正是Spark的魅力所在，因此要理解Spark，先要理解RDD的概念。

### 2.1.1 RDD简介

&emsp;&emsp;RDD全称为Resilient Distributed Datasets，即弹性分布式数据集。RDD是一个容错的、并行的数据结构，可以让用户显式地将数据存储到磁盘或内存中，并能控制数据的分区。同时，RDD还提供了一组丰富的操作来操作这些数据，在这些操作中，诸如map、flatMap、filter等转换操作实现了monad模式，很好地契合了Scala的集合操作。除此之外，RDD还提供了诸如join、groupBy、reduceByKey等更为方便的操作以支持常见的数据运算。

&emsp;&emsp;RDD是Spark的核心数据结构，通过RDD的依赖关系形成Spark的调度顺序。所谓Spark应用程序，本质是一组对RDD的操作。

&emsp;&emsp;下面介绍RDD的创建方式及操作算子类型。

&emsp;&emsp;(1) RDD的两种创建方式：

&emsp;&emsp;&emsp;&emsp;(a) 从文件系统输入(如HDFS)创建。

&emsp;&emsp;&emsp;&emsp;(b) 从已存在的RDD转换得到新的RDD。

&emsp;&emsp;(2) RDD的两种操作算子：

&emsp;&emsp;&emsp;&emsp;(a) Transformation(变换)

&emsp;&emsp;&emsp;&emsp;Transformation类型的算子不是立刻执行,而是延迟执行。也就是说从一个RDD变换为另一个RDD的操作需要等到Action操作触发时，才会真正执行。

&emsp;&emsp;&emsp;&emsp;(b) Action(行动)

&emsp;&emsp;&emsp;&emsp;Action类型的算子会触发Spark提交作业，并将数据输出到Spark系统。

### 2.1.2 深入RDD细节

&emsp;&emsp;RDD从直观上可以看作一个数组，本质上是逻辑分区记录的集合。在集群中，一个RDD可以包含多个分布在不同的节点上的分区，每个分区是一个dataset片段，如图2-1所示。

![][1]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-1 RDD 分区

&emsp;&emsp;图2-1中，RDD-1含有3个分区(p1、p2和p3)，分布存储在2个节点上：node1与node2。RDD-2只有一个分区P4，存储在node3节点上。RDD-3含有2个分区P5和P6，存储在node4节点上。

&emsp;&emsp;(1) RDD依赖

&emsp;&emsp;RDD可以相互依赖，如果RDD的每个分区最多只能被一个Child RDD的一个分区使用，则称之为narrow dependency；若多个Child RDD分区都可以依赖，则称之为wide dependency。不同的操作依据其特性，可能会产生不同的依赖，例如：map操作会产生narrow dependency，而join操作则产生wide dependency，如图2-2所示：

![][2]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-2 RDD Dependency


&emsp;&emsp;(2) RDD支持容错性

&emsp;&emsp;支持容错通常采用两种方式：日志记录或者数据复制。对于以数据为中心的系统而言，这两种方式都非常昂贵，因为它需要跨集群网络拷贝大量数据。

&emsp;&emsp;RDD天生是支持容错的。首先，它自身是一个不变的(immutable)数据集，其次，RDD之间通过lineage产生依赖关系(在下章来继续探讨这个话题)，因此RDD能够记住构建它的操作图，因此当执行任务的Worker失败时，完全可以通过操作图获得之前执行的操作，进行重新计算。因此无需采用replication方式支持容错，很好地降低了跨网络的数据传输成本。


&emsp;&emsp;(3) RDD如何做到高效率？

&emsp;&emsp;RDD提供了两方面的特性persistence(持久化)和partitioning(分区)，用户可以通过persist与partitionBy函数来控制这两个特性。RDD的分区特性与并行计算能力(RDD定义了parallerize函数)，使得Spark可以更好地利用可伸缩的硬件资源。如果将分区与持久化二者结合起来，就能更加高效地处理海量数据。

&emsp;&emsp;另外，RDD本质上是一个内存数据集，在访问RDD时，指针只会指向与操作相关的部分。例如：存在一个面向列的数据结构，其中一个实现为Int的数组，另一个实现为Float的数组。如果只需要访问Int字段，RDD的指针可以只访问Int数组，避免了对整个数据结构的扫描。

&emsp;&emsp;再者，如前文所述，RDD将操作分为两类：Transformation与Action。无论执行了多少次Transformation操作，RDD都不会真正执行运算，只有当Action操作被执行时，运算才会触发。而在RDD的内部实现机制中，底层接口则是基于迭代器的，从而使得数据访问变得更高效，也避免了大量中间结果对内存的消耗。

&emsp;&emsp;在实现时，RDD针对Transformation操作，都提供了对应的继承自RDD的类型，例如：map操作会返回MappedRDD，而flatMap则返回FlatMappedRDD。当我们执行map或flatMap操作时，不过是将当前RDD对象传递给对应的RDD对象而已。

### 2.1.3 RDD特性总结

&emsp;&emsp;RDD是Spark的核心，也是整个Spark的架构基础。它的特性可以总结如下：

&emsp;&emsp;(1) RDD是不变的(immutable)数据结构存储。

&emsp;&emsp;(2) RDD将数据存储在内存中，从而提供了低延迟性。

&emsp;&emsp;(3) RDD是支持跨集群的分布式数据结构。

&emsp;&emsp;(4) RDD可以根据记录的Key对结构分区。

&emsp;&emsp;(5) RDD提供了粗粒度的操作，并且都支持分区。


## 2.2 Spark程序模型

&emsp;&emsp;下面给出一个经典的统计日志中ERROR的例子，以便读者对Spark程序模型产生直观的理解。

&emsp;&emsp;(1) SparkContext中的textFile函数从存储系统(如HDFS)中读取日志文件，生成file变量。

&emsp;&emsp;`scala> var file = sc.textFile("hdfs：//...")`

&emsp;&emsp;(2) 统计日志文件中，所有含ERROR的行。

&emsp;&emsp;`scala> var errors = file.filer(line=>line.contains("ERROR"))`

&emsp;&emsp;(3) 返回包含ERROR的行数： `errors.count()`。

&emsp;&emsp;操作RDD与Scala集合非常类似，这是Spark努力追求的目标：像编写单机程序一样编写分布式应用。但二者的数据和运行模型却有很大不同。如图2-3所示：

![][3]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-3 Spark程序模型

&emsp;&emsp;在图2-3中，每一次对RDD的操作都造成了RDD的变换。其中RDD的每个逻辑分区Partition都对应于Block Manager(物理存储管理器)中的物理数据块Block(保存在内存或硬盘上)。前文已强调，RDD是应用程序中核心的元数据结构，其中保存了逻辑分区与物理数据块之间的映射关系，另外，还保存了父辈RDD的依赖转换关系。



## 2.3 Spark算子

&emsp;&emsp;本节介绍Spark算子的分类及其功能。

### 2.3.1 算子简介

&emsp;&emsp;Spark应用程序的本质，无非是把需要处理的数据转换为RDD，然后将RDD通过一系列的变换(Transformation)和操作(Action)而得到结果，简要来说，这些变换和操作即为算子。

&emsp;&emsp;Spark支持的主要算子如图2-4所示：

![][4]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-4 Spark支持的算子


&emsp;&emsp;根据算子所处理的数据类型及处理阶段的不同，算子大致可以分为如下三类：
 
&emsp;&emsp;(1) 处理Value数据类型的Transformation算子，这种变换并不触发提交作业，针对处理的数据项是Value型的数据。
 
&emsp;&emsp;(2) 处理Key-Value数据类型的Transfromation算子，这种变换并不触发提交作业，针对处理的数据项是Key-Value型的数据对。
 
&emsp;&emsp;(3) Action算子，这类算子触发SparkContext提交作业。


### 2.3.2 Value型Transmation算子


&emsp;&emsp;对于处理Value类型数据的Transformation算子，依据RDD的输入分区与输出分区的对应关系，可以将该类算子分为5类，如表2-1所示：

![][5]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表2-1 Value型算子分类

&emsp;&emsp;如上表2-1所示，value型的Transformation算子分类具体如下：

&emsp;&emsp;(1) 输入分区与输出分区1对1型。

&emsp;&emsp;(2) 输入分区与输出分区多对1型。

&emsp;&emsp;(3) 输入分区与输出分区多对多型。

&emsp;&emsp;(4) 输出分区为输入分区子集。

&emsp;&emsp;(5) Cache型，对RDD的分区缓存。

&emsp;&emsp;下面对这五种分类进行详细介绍：

&emsp;&emsp;1.输入分区与输出分区1对1型

&emsp;&emsp;(1) map算子: map是对RDD中的每个元素都执行一个指定的函数来产生一个新的RDD。任何原RDD中的元素在新RDD中都有且只有一个元素与之对应。如图2-5所示：

![][6]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-5 Map

&emsp;&emsp;图中RDD-1中的元素V1经过函数映射后，变为新的元素V'1，最终构成新的RDD-2。输入输出分区1对1没有变化。注意，事实上，只有Action算子被触发后，这些操作才会真正被执行。

&emsp;&emsp;(2) flatMap： 与map类似，将原RDD中的每个元素通过函数f转换为新的元素后，并将这些元素放入一个集合，构成新的RDD。如图2-6所示：

![][7]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-6 flatMap

&emsp;&emsp;&emsp;&emsp;图中外面大的矩形表示分区，小的矩形表示元素集合。如元素A1，A2在RDD-1中属于一个集合，B1，B2，B3属于另一个集合。RDD-1经过flatMap变换之后，在新的RDD-2中，A'与B'处于同一集合中。


&emsp;&emsp;(3) mapPartitions：mapPartitions是map的一个变种。map的输入函数是应用于RDD中每个元素，而mapPartitions的输入函数是应用于每个分区，也就是把每个分区中的内容作为整体来处理的。 

&emsp;&emsp;它的函数定义为：

```scala
def mapPartitions[U: ClassTag](f: Iterator[T] => Iterator[U], preservesPartitioning: Boolean = false): RDD[U]
```

&emsp;&emsp;f 即为输入函数，它处理每个分区里面的内容。每个分区中的内容将以Iterator[T]传递给输入函数f，f的输出结果是Iterator[U]。最终的RDD由所有分区经过输入函数处理后的结果合并起来的。如图2-7所示：

![][8]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-7 mapPartitions

&emsp;&emsp;在图中，用户通过f(iter)=>iter.filter(_>0)对元素过滤，保留大于0的元素。其中方框为分区，虽然过滤了元素，但原有分区保持不变。

&emsp;&emsp;(4) glom：将每个分区内的元素组成一个数组，分区不变。如图2-8所示：

![][9]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-8 glom

&emsp;&emsp;图中方框代表分区，glom算子将每个分区内的元素组成一个数组。


&emsp;&emsp;2.输入分区与输出分区多对1型

&emsp;&emsp;(1) union: 合并同一数据类型元素，但不去重。合并后返回同类型的数据元素。如图2-9所示：

![][10]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-9 union

&emsp;&emsp;&emsp;&emsp;图中大方框代表RDD，内部小方框代表RDD分区，合并后同一类型元素位于同一分区中。

&emsp;&emsp;&emsp;&emsp;(2) cartesian：对输入RDD内的所有元素计算笛卡尔积。如图2-10所示：

![][11]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-10 cartesian


&emsp;&emsp;3.输入分区与输出分区多对多型

&emsp;&emsp;(1) groupBy：先将元素通过函数生成key,元素在转为“key-value”类型之后，将key相同的元素分为一组。如下图2-11所示：

![][12]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-11 groupBy

&emsp;&emsp;图中可以看到三个分区，经过groupBy变换后，key相同的元素被合并到一组。

&emsp;&emsp;4.输出分区为输入分区子集

&emsp;&emsp;(1) filter : 对RDD中的元素进行过滤，过滤函数返回true的元素保留，否则删除。如下图2-12所示：

![][13]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-12 filter

&emsp;&emsp;图中方框为RDD的分区。

&emsp;&emsp;(2) distinct：对RDD中的元素进行去重操作，对重复的元素只保留一份。

&emsp;&emsp;(3) substract：对集合进行差操作。即RDD1中去除RDD1与RDD2的交集。

&emsp;&emsp;(4) sample： 对RDD集合内的元素采样。

&emsp;&emsp;(5) takesample：与sample算子类似，可以设定采样个数。


&emsp;&emsp;5.Cache型（RDD持久化操作）

&emsp;&emsp;(1) cache：将RDD元素从磁盘缓存到内存。

&emsp;&emsp;(2) persist：与cache类似，但比cache功能更强大，persist函数可以指定存储级别。完整的存储级别列表如表2-2所示：

![][14]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表2-2 Storage Level

### 2.3.3 Key-Value型Transmation算子

&emsp;&emsp;处理数据类型为Key-Value的Transmation算子，大致可以分为三类：

&emsp;&emsp;1.输入输出分区1对1

&emsp;&emsp;mapValues：顾名思义就是输入函数应用于RDD中KV(Key-Value)类型元素中的Value，原RDD中的Key保持不变，与新的Value一起组成新的RDD中的元素。因此，该函数只适用于元素为Key-Value对的RDD。如图2-13所示：

![][15]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-13 mapValues

&emsp;&emsp;图中输入函数对value分别进行加10操作，形成新的RDD包含KV类型新元素。


&emsp;&emsp;2.聚集操作

&emsp;&emsp;(1) 对一个RDD聚集

&emsp;&emsp;&emsp;&emsp;(a) reduceByKey：对元素为KV对的RDD中Key相同的元素的Value进行reduce，即两个值合并为一个值。因此，Key相同的多个元素的值被reduce为一个值，然后与原RDD中的Key组成一个新的KV对。如图2-14所示：

![][16]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-14 reduceByKey

&emsp;&emsp;&emsp;&emsp;(b) combineByKey： 与reduceByKey类似，相当于将元素（int,int）KV对，变换为（int,Seq[int]）新的KV对。如图2-15所示：

![][17]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-15 combineByKey

&emsp;&emsp;&emsp;&emsp;(c) partitionBy：根据KV对的Key对RDD进行分区。如图2-16所示：

![][18]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-16 partitionBy

&emsp;&emsp;(2) 对两个RDD聚集

&emsp;&emsp;coGroup：一组强大的函数，可以对多达3个的RDD根据key进行分组。对每个Key相同的元素分别聚集为一个集合。如图2-17所示：

![][19]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-17 coGroup

&emsp;&emsp;图2-17中，大方框为RDD，内部小方框为RDD中的分区。

&emsp;&emsp;3.连接

&emsp;&emsp;(1) join：本质是对两个含有KV对元素的RDD进行cogroup算子协同划分，再通过flatMapValues将合并的数据分散。

&emsp;&emsp;(2) leftOutJoin与rightOutJoin ：相当于在join基础上判断一侧的RDD是否为空，如果为空则填充空，如果有数据，则将数据进行连接计算，然后返回结果。

### 2.3.4 Action算子

&emsp;&emsp;Action算子可以依据其输出空间将其划分为：无输出，HDFS，Scala集合和数据类型。

&emsp;&emsp;1.无输出

&emsp;&emsp;foreach：对RDD中的每个元素执行无参数的f函数，返回Unit。定义如下：

&emsp;&emsp;`def foreach(f: T => Unit)`

&emsp;&emsp;foreach功能示例如图2-18所示：

![][20]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-18 foreach

&emsp;&emsp;图中，定义了println打印函数，打印RDD中所有数据项。

&emsp;&emsp;2.HDFS

&emsp;&emsp;(1) saveAsTextFile：函数将RDD保存为文本至HDFS指定目录，每次输出一行。

&emsp;&emsp;功能示例如图2-19所示：

![][21]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-19 saveAsTextFile

&emsp;&emsp;图中，通过函数将RDD中每个元素映射为（null,x.toString)，然后写入HDFS block。RDD的每个分区存储为HDFS中的数据块Block。

&emsp;&emsp;(2) saveAsObjectFile：将RDD分区中每10个元素保存为一个数组并将其序列化，映射为（null,BytesWritable(Y)）的元素，以SequenceFile的格式写入HDFS。如图2-20所示：

![][22]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-20 saveAsObjectFile


&emsp;&emsp;3.Scala集合及数据类型

&emsp;&emsp;(1) collect：将RDD分散性存储的元素转换为单机上的Scala数组并返回，类似于toArray功能。如图2-21所示：

![][23]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-21 collect


&emsp;&emsp;(2) collectAsMap：与collect类似，针对元素类型为key-value对的RDD，将其转换为Scala Map并返回，保存元素的KV结构。

&emsp;&emsp;(3) lookup：扫描RDD所有元素，选择与参数匹配的Key，并将其value以Scala sequence的形式返回。如图2-22所示：

![][24]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-22 lookup

&emsp;&emsp;(4) reduceByKeyLocally：先reduce,然后collectAsMap。

&emsp;&emsp;(5) count：返回RDD中元素个数。

&emsp;&emsp;(6) reduce：对RDD中所有元素进行reduceLeft操作。

&emsp;&emsp;例如，当用户函数定义为：`f:(A,B)=>(A._1+"@"+B._1,A._2+B._2)`时，reduce算子计算过程如图2-23所示：

![][25]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-23 reduce

&emsp;&emsp;(7) top/take：返回RDD中最大/最小的K个元素。

&emsp;&emsp;(8) fold：与reduce类似，不同的时候每次对分区内value聚集时，分区内初始化的值为zero value。

&emsp;&emsp;例如，当用户自定义函数为：`fold(("A0",0))((A,B)=>A._1+"@"+B._1， A._2 + B._2 ))`时，fold算子计算过程如图2-24所示：

![][26]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图2-24 fold



&emsp;&emsp;(9) aggregate：允许用户对RDD使用两个不同的reduce函数，第一个reduce函数对各个分区内的数据聚集，每个分区得到一个结果。第二个reduce函数对每个分区的结果进行聚集，最终得到一个总的结果。aggregate相当于对RDD内元素数据归并聚集，且这种聚集是可以并行化的。而fold与reduced的聚集是串行的。

&emsp;&emsp;(10) broadcast(广播变量)：存储在单节点内存中，不需要跨节点存储。Spark运行时将广播变量数据分发到各个节点，可以跨作业共享。

&emsp;&emsp;(11) accucate：允许全局累加操作。accumulator被广泛应用于记录应用运行参数。


[1]: resources/model/2-1RDD-Partition.png
[2]: resources/model/2-2RDD-Dependency.png
[3]: resources/model/2-3Spark-example.png
[4]: resources/model/2-4T&A.png
[5]: resources/model/b2-1value-Transformation.png
[6]: resources/model/2-5Map.png
[7]: resources/model/2-6flatMap.png
[8]: resources/model/2-7mapPartitions.png
[9]: resources/model/2-8glom.png
[10]: resources/model/2-9union.png
[11]: resources/model/2-10cartesian.png
[12]: resources/model/2-11groupBy.png
[13]: resources/model/2-12filter.png
[14]: resources/model/b2-2storageLevel.png
[15]: resources/model/2-13mapValues.png
[16]: resources/model/2-14reduceByKey.png
[17]: resources/model/2-15combineByKey.png
[18]: resources/model/2-16partitionByKey.png
[19]: resources/model/2-17coGroup.png
[20]: resources/model/2-18foreach.png
[21]: resources/model/2-19saveAsTextFile.png
[22]: resources/model/2-20saveAsObjectFile.png
[23]: resources/model/2-21collect.png
[24]: resources/model/2-22lookup.png
[25]: resources/model/2-23reduce.png
[26]: resources/model/2-24fold.png