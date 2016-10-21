---
layout: post
title:  Spark编程模型
date:   2016-10-21 12:08:00 +0800
categories: Spark
tag: Scala
rank: 10 
---


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



[1]: {{ '/bigdata/spark/resources/model/2-1RDD-Partition.png' | prepend: site.baseurl  }}

