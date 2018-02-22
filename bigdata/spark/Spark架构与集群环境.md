>>>>>>># 第1章 Spark架构与集群环境

&emsp;&emsp;本章首先介绍Spark大数据处理框架的基本概念。然后介绍Spark生态系统的主要组成部分，包括：Spark SQL、Spark Streaming、MLlib和GraphX。接着简要描述了Spark的架构，便于读者对Spark产生概要的认识和把握。最后描述了Spark集群环境搭建及Spark开发环境的构建方法。

1.1 Spark概述与架构


&emsp;&emsp;随着互联网规模的爆发式增长，不断增加的数据量要求应用程序能够延伸到更大的集群中去计算。与单台机器计算不同，集群计算引发了几个关键问题，如：集群计算资源的共享，单点宕机，节点执行缓慢及程序的并行化。针对这几个集群环境的问题，许多大数据处理框架应运而生。比如：Google的MapReduce，它提出了简单、通用并具有自动容错功能的批处理计算模型。但是MapReduce对于某些类型的计算并不适合，比如交互式和流式计算。基于这种类型需求的不一致性，大量不同于MapReduce的专门数据处理模型诞生了，如GraphLab、Impala、Storm等等。 随着大量数据模型的产生，引发的后果是对于大数据处理而言，针对不同类型的计算，通常需要一系列不同的处理框架才能完成。这些不同的处理框架由于天生的差异又带来了一系列问题：重复计算，使用范围的局限性，资源分配，统一管理等等。


1.1.1  Spark概述

&emsp;&emsp;为了解决上述MapReduce及各种处理框架所带来的问题，加州大学伯克利分校推出了Spark统一大数据处理框架。Spark是一种与Hadoop MapReduce类似的开源集群大数据计算分析框架。Spark基于内存计算，它整合了内存计算的单元，所以相对于hadoop的集群处理方法，Spark在性能方面更具优势。Spark启用了弹性内存分布式数据集，除了能够提供交互式查询外，它还可以优化迭代工作负载。

&emsp;&emsp;从另一角度来看，Spark可以看做是MapReduce的一种扩展。MapReduce之所以不擅长迭代式、交互式和流式的计算工作，主要因为它缺乏在计算的各个阶段进行有效的资源共享，针对这一点，Spark创造性的引入了RDD（弹性分布式数据集）来解决这个问题，RDD的重要特性之一就是资源共享。

&emsp;&emsp;Spark基于内存计算，提高了大数据处理的实时性，同时兼具高容错性和可伸缩性。更重要的是，Spark可以部署在大量廉价的硬件之上，形成集群。

&emsp;&emsp;提到Spark的优势就不得不提到大家熟知的Hadoop。事实上，Hadoop主要解决了两件事情：

&emsp;&emsp;(1) 数据的可靠存贮。

&emsp;&emsp;(2) 数据的分析处理。

&emsp;&emsp;相应地，Hadoop也主要包括两个核心部分：

&emsp;&emsp;(1) **分布式文件系统HDFS**（Hadoop Distributed File System）

&emsp;&emsp;在集群上提供高可靠的文件存储，通过将文件块保存多个副本的办法解决服务器或硬盘故障的问题。

&emsp;&emsp;(2) **计算框架MapReduce**

&emsp;&emsp;通过简单的Mapper和Reducer的抽象提供一个编程模型，可以在一个由几十台上百台机器组成的不可靠集群上并发地，分布式地处理大量的数据集，而把并发、分布式（如机器间通信）和故障恢复等计算细节隐藏起来。

&emsp;&emsp;Spark是MapReduce的一种更优的替代方案，同时可以兼容HDFS等分布式存储层，也可以兼容现有的Hadoop生态系统，同时弥补了MapReduce的不足。

&emsp;&emsp;与Hadoop MapReduce相比，Spark的优势如下：

&emsp;&emsp;(1) 中间结果

&emsp;&emsp;基于MapReduce的计算引擎通常将中间结果输出到磁盘上以达到存储和容错的目的。由于任务管道承接的缘故，一切查询操作会产生很多串联的Stage,这些Stage输出的中间结果存储于HDFS。而对Spark而言，它将执行操作抽象为通用的有向无环图（DAG），可以将多个Stage的任务串联或者并行执行，而无须将Stage中间结果输出到HDFS中。

&emsp;&emsp;(2) 执行策略

&emsp;&emsp;MapReduce在数据Shuffle之前，需要花费大量时间来排序。而Spark不需要对所有情景都进行要排序。由于采用了DAG的执行计划，每一次输出的中间结果可以缓存在内存中。

&emsp;&emsp;(3) 任务调度的开销

&emsp;&emsp;MapReduce系统是为了处理长达数小时的批量作业而设计的，在某些极端情况下，提交任务的延迟非常高。而Spark采用了事件驱动的类库AKKA来启动任务，通过线程池复用线程来避免线程启动及切换产生的开销。

&emsp;&emsp;(4) 更好的容错性

&emsp;&emsp;RDD之间维护了血缘关系（lineage），一旦某个RDD失败了，能通过父RDD自动重建，保证了容错性。

&emsp;&emsp;(5) 高速

&emsp;&emsp;基于内存的Spark计算速度是基于磁盘的Hadoop MapReduce的大约100倍。

&emsp;&emsp;(6) 易用

&emsp;&emsp;相同的应用程序代码量一般比Hadoop MapReduce少50%～80%。

&emsp;&emsp;(7) 提供了丰富的API

&emsp;&emsp;与此同时，Spark支持多语言编程，如Scala、Python及Java，便于开发者在自己熟悉的环境下工作。Spark自带了80多个算子，同时允许在Spark shell环境下进行交互式计算，开发者可以像书写单机程序一样开发分布式程序，轻松利用Spark搭建大数据内存计算平台，并利用内存计算特性，实现对海量数据的实时处理。


1.1.2  Spark生态

&emsp;&emsp;Spark大数据计算平台包含许多子模块，其中Spark为核心，这些构成了整个Spark的生态系统。

&emsp;&emsp;伯克利将整个Spark的生态系统称之为伯克利数据分析栈（BDAS），如图1-1所示。

![][2]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图1-1 Spark生态系统-伯克利数据分析栈BDAS结构

&emsp;&emsp;以下简要介绍BDAS的各个组成部分：

&emsp;&emsp;(1) Spark core

&emsp;&emsp;Spark core是整个BDAS的核心组件，是一种大数据分布式处理框架。不仅实现了MapReduce的算子map函数和reduce函数及计算模型，还提供如filter、join、groupByKey等更丰富的算子。Spark将分布式数据抽象为弹性分布式数据集（RDD），实现了应用任务调度、RPC、序列化和压缩，并为运行在其上的上层组件提供API。其底层采用Scala函数式语言书写而成，并且所提供的API深度借鉴Scala函数式的编程思想，提供与Scala类似的编程接口。

&emsp;&emsp;(2) Mesos

&emsp;&emsp;Mesos是Apache下的开源分布式资源管理框架，它被称为是分布式系统的内核。提供了类似Yarn的功能，实现了高效的资源任务调度。

&emsp;&emsp;(3) Spark Streaming

&emsp;&emsp;Spark Streaming是一种构建在Spark上的实时计算框架，它扩展了Spark处理大规模流式数据的能力。其吞吐量能够超越现有主流流处理框架Storm，并提供丰富的API用于流数据计算。

&emsp;&emsp;(4) MLlib

&emsp;&emsp;MLlib 是Spark对常用的机器学习算法的实现库，同时包括相关的测试和数据生成器。MLlib 目前支持四种常见的机器学习问题：二元分类，回归，聚类以及协同过滤，同时也包括一个底层的梯度下降优化基础算法。

&emsp;&emsp;(5) GraphX

&emsp;&emsp;GraphX是 Spark中用于图和图并行计算的API,可以认为是GraphLab和Pregel在Spark(Scala)上的重写及优化，跟其他分布式图计算框架相比，GraphX最大的贡献是，在Spark之上提供一栈式数据解决方案，可以方便且高效地完成图计算的一整套流水作业。

&emsp;&emsp;(6) Spark SQL

&emsp;&emsp;Shark是构建在Spark和Hive基础之上的数据仓库。它提供了能够查询Hive中所存储数据的一套SQL接口，兼容现有的Hive QL语法。熟悉Hive QL或者SQL的用户可以基于Shark进行快速的Ad-Hoc、Reporting等类型的SQL查询。由于其底层计算采用了Spark，性能比Mapreduce的Hive普遍快2倍以上，当数据全部load在内存时，要快10倍以上。2014年7月1日之后，Spark社区推出了Spark SQL，重新实现了SQL解析等原来Hive完成的工作，Spark SQL在功能上全面覆盖了原有的Shark，且具备更优秀的性能。

&emsp;&emsp;(7) Alluxio（原名Tachyon）

&emsp;&emsp;Alluxio（原名Tachyon）是一个分布式内存文件系统，可以理解为内存中的HDFS。为了提供更高的性能，将数据存储剥离Java Heap。用户可以基于Alluxio实现RDD或者文件的跨应用共享，并提供高容错机制，保证数据的可靠性。


&emsp;&emsp;(8) BlinkDB

&emsp;&emsp;BlinkDB是一个用于在海量数据上进行交互式SQL的近似查询引擎。它允许用户在查询准确性和查询响应时间之间做出权衡，执行相似查询。


1.1.3 Spark架构

&emsp;&emsp;传统的单机系统，虽然可以CPU多核共享内存，磁盘等资源，但是当计算与存储能力无法满足大规模数据处理的需要时，面对单机系统自身CPU与存储无法扩展的先天限制，单机系统就力不从心了。

1.1.2.1  分布式系统的架构

&emsp;&emsp;所谓的分布式系统，即为在网络互连的多个计算单元执行任务的软硬件系统。一般包括分布式操作系统、分布式数据库系统、分布式应用程序等等。本书介绍的Spark分布式计算框架，可以看作分布式软件系统的组成部分，基于Spark，开发者可以编写分布式计算程序。

&emsp;&emsp;直观来看，大规模分布式系统由许多计算单元构成，每个计算单元之间松耦合。同时，每个计算单元都包含自己的CPU、内存、总线及硬盘等私有计算资源。这种分布式结构的最大特点在于不共享资源，与此同时，计算节点可以无限制扩展，计算能力和存储能力也因而得到巨大增长。但是由于分布式架构在资源共享方面的先天缺陷，开发者在书写和优化程序时应引起注意。分布式系统架构如图1-2所示。

![][3]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图1-2 分布式系统架构图

&emsp;&emsp;为了减少网络I/O开销，对于分布式计算而言，一个核心原则是数据应该尽量做到本地计算。在计算过程中，每个计算单元之间需要传输信息，因此在信息传输较少时，分布式系统可以利用资源无限扩展的优势达到高效率，这也是分布式系统的优势。目前分布式系统在数据挖掘和决策支持等方面有着广泛的应用。

&emsp;&emsp;Spark正是基于这种分布式并行架构而产生，也可以利用分布式架构的优势，根据需要，对计算能力和存储能力进行扩展。足以应对处理海量数据带来的挑战。同时，Spark的快速及容错等特性，让数据处理分析显得游刃有余。


1.1.2.2  Spark架构

&emsp;&emsp;Spark架构采用了分布式计算中的Master-Slave模型。集群中运行Master进程的节点称之为Master，同样地，集群中含有Worker进程的节点为Slave。Master负责控制整个集群的运行；Worker节点相当于分布式系统中的计算节点，它接收Master节点指令并向Master回报计算进程；Executor负责任务的执行；Client是用户提交应用的客户端；Driver负责提交后分布式应用的协调。Spark架构如图1-3所示。

![][4]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图1-3 Spark架构

&emsp;&emsp;在Spark应用的执行过程中，Driver和Worker是两个对应的概念。Driver是应用逻辑执行的起点，负责Task任务的分发和调度；Worker负责管理计算节点并创建Executor来并行处理Task任务。Task执行过程中所需要的文件和包由Driver序列化后传输给对应的worker节点，Executor对相应分区的任务进行处理。

&emsp;&emsp;下面对图中所示的Spark架构中的组件逐一介绍。

&emsp;&emsp;(1) Client： 提交应用的客户端。

&emsp;&emsp;(2) Driver： 执行Application中的main函数并创建SparkContext。

&emsp;&emsp;(3) ClusterManager： 在YARN模式中为资源管理器。在Standalone模式中为Master（主节点），控制整个集群。

&emsp;&emsp;(4) Worker： 从节点，负责控制计算节点。启动Executor或Driver，在Yarn模式中为NodeManager。

&emsp;&emsp;(5) Executor： 在计算节点上执行任务的组件。

&emsp;&emsp;(6) SparkContext： 应用的上下文，控制应用的生命周期。

&emsp;&emsp;(7) RDD： 弹性分布式数据集，Spark的基本计算单元，一组RDD可形成有向无环图。

&emsp;&emsp;(8) DAG Scheduler： 根据应用构建基于Stage的DAG，并将stage提交给Task Scheduler。

&emsp;&emsp;(9) Task Scheduler： 将Task分发给Executor执行。

&emsp;&emsp;(10) SparkEnv： 线程级别的上下文，存贮运行时重要组件的应用，具体如下:

&emsp;&emsp;&emsp;&emsp;(a) SparkConf： 存贮配置信息。

&emsp;&emsp;&emsp;&emsp;(b) BroadcastManager： 负责广播变量的控制及元信息的存贮。

&emsp;&emsp;&emsp;&emsp;(c) BlockManager： 负责Block的管理，创建和查找。

&emsp;&emsp;&emsp;&emsp;(d) MetricsSystem： 监控运行时的性能指标。

&emsp;&emsp;&emsp;&emsp;(e) MapOutputTracker： 负责shuffle元信息的存储。


&emsp;&emsp;Spark架构揭示了Spark的具体流程如下：

&emsp;&emsp;(1) 用户在Client提交了应用

&emsp;&emsp;(2) Master找到Worker启动Driver

&emsp;&emsp;(3) Driver向资源管理器（YARN模式）或者Master（Standalone模式）申请资源，并将应用转化为RDD Graph。

&emsp;&emsp;(4) DAGScheduler将RDD Graph转化为Stage的有向无环图提交给Task Scheduler。

&emsp;&emsp;(5) Task Scheduler提交任务给Exector执行。


1.1.2.3  Spark运行逻辑

&emsp;&emsp;我们举例来说明Spark的运行逻辑，如图1-4所示，在Action算子被触发之后，所有累积的算子会形成一个有向无环图DAG。Spark会根据RDD之间不同的依赖关系形成Stage,每个Stage包含了一系列函数执行流水线。图中A，B，C，D，E，F为不同的RDD，RDD内的方框为RDD的分区。

![][5]

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;图1-4 Spark执行RDD Graph

&emsp;&emsp;图中的运行逻辑如下：

&emsp;&emsp;(1) 数据从HDFS输入Spark。

&emsp;&emsp;(2) RDD A, RDD C 经过flatMap与Map操作后，分别转换为RDD B 和 RDD D。

&emsp;&emsp;(3) RDD D经过reduceByKey操作转换为RDD E。

&emsp;&emsp;(4) RDD B 与 RDD E 进行join操作转换为RDD F。

&emsp;&emsp;(5) RDD F 通过函数saveAsSequenceFile输出保存到HDFS中。




1.2 在Linux集群上部署Spark

&emsp;&emsp;Spark安装部署比较简单,用户可以登录其官方网站[http://spark.apache.org/downloads.html](http://spark.apache.org/downloads.html)下载Spark最新版本或历史release版本，也可以查阅Spark相关文档。本书开始写作时，Spark刚刚发布1.5.0版，因此本章所述的环境搭建均以Spark 1.5.0版为例。

&emsp;&emsp;Spark使用了Hadoop的HDFS作为持久化存储层，因此安装Spark时，应先安装与Spark版本相兼容的Hadoop。

&emsp;&emsp;本节以阿里云linux主机为例，描述集群环境及Spark开发环境的搭建过程。

&emsp;&emsp;Spark计算框架以Scala语言开发，因此部署Spark首先需要安装Scala及JDK（Spark1.5.0需要JDK1.7.0或更高版本）。另外，Spark计算框架基于持久化层，如Hadoop HDFS，因此本章也会简述Hadoop的安装配置。

1.2.1   安装OpenJDK

&emsp;&emsp;Spark1.5.0要求OpenJDK1.7.0或更高版本。 以本机Linux X86机器为例，OpenJDK安装步骤如下：

&emsp;&emsp;(1) 查询服务器上可用的JDK版本。

&emsp;&emsp;在terminal输入如下命令：

```bash
yum list "*JDK*"
```

&emsp;&emsp;yum 会列出服务器上的JDK版本。

&emsp;&emsp;(2) JDK 安装。

&emsp;&emsp;终端输入如下命令：

```bash
yum install java-1.7.0-openjdk-devel.x86
cd /usr/lib/jvm
ln -s java-1.7.0-openjdk.x86 java-1.7
```

&emsp;&emsp;(3) JDK环境配置。

&emsp;&emsp;&emsp;&emsp;(a) 用编辑器打开/etc/profile文件，加入如下内容：
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.7
export PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
```

&emsp;&emsp;&emsp;&emsp;关闭并保存profile文件。

&emsp;&emsp;&emsp;&emsp;(b) 输入命令 __`source /etc/profile`__ 让配置生效。

1.2.2   安装Scala

&emsp;&emsp;登录Scala官网 http://www.scala-lang.org/download/ 下载最新版本： **scala-2.11.7.tgz**

&emsp;&emsp;(1) 安装：

```bash
tar zxvf scala-2.11.7.tgz -C /usr/local
cd /usr/local
ln -s scala-2.11.7 scala
```

&emsp;&emsp;(2) 配置：打开/etc/profile， 加入如下语句：

```bash
export SCALA_HOME=/usr/local/scala
export PATH=$PATH:$SCALA_HOME/bin
```

1.2.3   配置SSH免密码登录

&emsp;&emsp;在分布式系统中，如Hadoop与Spark，通常使用ssh(安全协议Secure Shell)服务来启动slave节点上的程序，当节点数量比较大时，频繁地输入密码进行身份认证是一项非常艰难的体验。为了简化这个问题，我们可以使用"公私钥"认证的方式来达到ssh免密码登录。

&emsp;&emsp;首先在master节点上创建一对公私钥（公钥文件：~/.ssh/id\_rsa.pub； 私钥文件：~/.ssh/id\_rsa），然后把公钥拷贝到worker节点上（~/.ssh/authorized_keys）。二者交互步骤如下:

&emsp;&emsp;(1) Master通过ssh连接worker时， worker生成一个随机数然后用公钥加密后，发回给Master。

&emsp;&emsp;(2) Master收到加密数后，用私钥解密，并将解密数回传给worker。
 
&emsp;&emsp;(3) Worker确认解密数正确之后，允许Master连接。

&emsp;&emsp;如果配置好SSH免密码登录之后，那么在以上交互中就无须用户输入密码了。下面详细介绍安装与配置过程：

&emsp;&emsp;(1) 安装ssh： __`yum install ssh`__

&emsp;&emsp;(2) 生成公私钥对： __`ssh-keygen -t rsa`__

&emsp;&emsp;一路回车，不需要输入。执行完成后会在~/.ssh目录下可以看到已生成id_rsa.pub与id_rsa两个密钥文件。其中id_rsa.pub为公钥。

&emsp;&emsp;(3) 拷贝公钥到worker机器： __`scp ~/.ssh/id_rsa.pub <用户名>@<worker机器ip>:~/.ssh`__

&emsp;&emsp;(4) 在worker节点上，将公钥文件重命名为authorized_keys： __`mv id_rsa.pub authorized_keys`__。 类似地，在所有worker节点上都可以配置ssh免密码登录。


1.2.4   Hadoop的安装配置

&emsp;&emsp;登录Hadoop官网下载页面：http://hadoop.apache.org/releases.html 下载Hadoop 2.6.0安装包: **hadoop-2.6.0.tar.gz**。

&emsp;&emsp;然后解压至本地指定目录：

```bash
tar zxvf hadoop-2.6.0.tar.gz -C /usr/local
ln -s hadoop-2.6.0 hadoop
```

&emsp;&emsp;下面重点讲解Hadoop的配置。

&emsp;&emsp;(1) 打开/etc/profile，末尾加入：

```bash
export HADOOP_INSTALL=/usr/local/hadoop
export PATH=$PATH:$HADOOP_INSTALL/bin
export PATH=$PATH:$HADOOP_INSTALL/sbin
export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
export HADOOP_COMMON_HOME=$HADOOP_INSTALL
export HADOOP_HDFS_HOME=$HADOOP_INSTALL
export YARN_HOME=$HADOOP_INSTALL
```

&emsp;&emsp;执行 __`source /etc/profile`__使其生效。然后进入Hadoop配置目录:/usr/local/hadoop/etc/hadoop进行Hadoop的配置。

&emsp;&emsp;(2) 配置hadoop_env.sh

```bash
export JAVA_HOME=/usr/lib/jvm/java-1.7
```

&emsp;&emsp;(3) 配置core-site.xml
```
<property>
     <name>fs.defaultFS</name>
     <value>hdfs://Master:9000</value>
</property>
<property>
     <name>hadoop.tmp.dir</name>
     <value>file:/root/bigdata/tmp</value>
</property>
<property>
     <name>io.file.buffer.size</name>
     <value>131702</value>
</property>
```

&emsp;&emsp;(4) 配置yarn-site.xml
```
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<property>
    <name>yarn.nodemanager.auxservices.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
</property>
<property>
    <name>yarn.resourcemanager.address</name>
    <value>Master:8032</value>
</property>
<property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>Master:8030</value>
</property>
<property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>Master:8031</value>
</property>
<property>
    <name>yarn.resourcemanager.admin.address</name>
    <value>Master:8033</value>
</property>
<property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>Master:8088</value>
</property>
```

&emsp;&emsp;(5) 配置mapred-site.xml
```
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>Master:10020</value>
</property>
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>Master:19888</value>
</property>
```

&emsp;&emsp;(6) 创建namenode和datanode目录，并配置路径

&emsp;&emsp;&emsp;&emsp;(a) 创建目录：

```bash
mkdir -p /hdfs/namenode
mkdir -p /hdfs/datanode
```

&emsp;&emsp;&emsp;&emsp;(b) 在hdfs-site.xml中配置路径：
```
<property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/hdfs/namenode</value>
</property>
<property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/hdfs/datanode</value>
</property>
<property>
    <name>dfs.replication</name>
    <value>3</value>
</property>
<property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>Master:9001</value>
</property>
<property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
</property>
```

&emsp;&emsp;(7) 配置slaves文件，在其中加入所有从节点主机名,如：

&emsp;&emsp;`x.x.x.x worker1`

&emsp;&emsp;`x.x.x.x worker2`

&emsp;&emsp;...

&emsp;&emsp;(8) 格式化namenode：

```bash
/usr/local/hadoop/bin/hadoop namenode -format
```

&emsp;&emsp;至此，Hadoop配置过程基本完成。


1.2.5   Spark的安装部署

&emsp;&emsp;登录Spark官网下载页面：http://spark.apache.org/downloads.html 下载Spark。这里我们选择最新的Spark 1.5.0版：**spark-1.5.0-bin-hadoop2.6.tgz（Pre-built for Hadoop2.6 and later）**。

&emsp;&emsp;然后解压spark安装包至本地指定目录:

```bash
tar zxvf spark-1.5.0-bin-hadoop2.6.tgz -C /usr/local/
ln -s spark-1.5.0-bin-hadoop2.6 spark
```

&emsp;&emsp;下面我们开始spark的配置之旅吧。

&emsp;&emsp;(1) 打开/etc/profile，末尾加入：

```bash
&emsp;&emsp;`export SPARK_HOME=/usr/local/spark`
&emsp;&emsp;`PATH=$PATH:${SPARK_HOME}/bin`
```

&emsp;&emsp;关闭并保存profile。然后命令行执行 `source /etc/profile` 使配置生效。 

&emsp;&emsp;(2) 打开/etc/hosts，加入集群中Master及各个worker节点的ip与hostname配对：

&emsp;&emsp;`x.x.x.x Master-name`

&emsp;&emsp;`x.x.x.x worker1`

&emsp;&emsp;`x.x.x.x worker2`

&emsp;&emsp;`x.x.x.x worker3`

&emsp;&emsp;...

&emsp;&emsp;(3) cd 进入/usr/local/spark/conf, 命令行执行：

```bash
cp spark-env.sh.template spark-env.sh
vi spark-env.sh
``` 

&emsp;&emsp;末尾加入：
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.7
export SCALA_HOME=/usr/local/scala
export SPARK_MASTER_IP=112.74.197.158<以本机为例>
export SPARK_WORKER_MEMORY=1g
```

&emsp;&emsp;保存并退出。执行命令：

```bash
cp slaves.template slaves
vi slaves
``` 

&emsp;&emsp;在其中加入各个worker节点的hostname。以作者环境为例，四台机器：master、worker1、worker2、worker3，那么slaves文件内容如下：
 
&emsp;&emsp;`worker1`

&emsp;&emsp;`worker2`

&emsp;&emsp;`worker3`


1.2.6   Hadoop与Spark的集群复制


&emsp;&emsp;以上我们完成了Master主机上Hadoop与Spark的搭建，现在我们将该环境及部分配置文件从Master上分发到各个worker节点上（以作者环境为例）。在集群环境中，由一台主机向多台主机间的文件传输一般使用pssh工具来完成。为此，我们在master上建立一个文件workerlist.txt，其中保存了所有worker节点的ip，每次文件的分发只需要一行命令即可完成。

&emsp;&emsp;(1) 复制jdk环境。
```bash
pssh -h workerlist -r /usr/lib/jvm/java-1.7 /
```

&emsp;&emsp;(2) 复制scala环境。
```bash
pssh -h workerlist -r /usr/local/scala /
```

&emsp;&emsp;(3) 复制Hadoop。
```bash
pssh -h workerlist -r /usr/local/hadoop /
```

&emsp;&emsp;(4) 复制Spark环境。
```bash
pssh -h workerlist -r /usr/local/spark /
```

&emsp;&emsp;(5) 复制系统配置文件。
```bash
pssh -h workerlist /etc/hosts /
pssh -h workerlist /etc/profile /
```

&emsp;&emsp;至此，Spark linux集群环境搭建完毕。



1.3 Spark 集群试运行


&emsp;&emsp; 我们下面来试运行Spark。

&emsp;&emsp;(1) 在Master主机上，分别启动Hadoop与Spark:

```bash
cd /usr/local/hadoop/sbin/
./start-all.sh
cd /usr/local/spark/sbin
./start-all.sh
```

&emsp;&emsp;(2) 检查Master与Worker进程是否在各自节点上启动:

&emsp;&emsp;在Master主机上，执行命令： jps，如图1-5所示：

&emsp;&emsp; ![][6]

&emsp;&emsp; 图1-5  在Master主机上执行jps命令

&emsp;&emsp;在Worker节点上，以Worker1为例，执行命令： jps, 如图1-6所示：

&emsp;&emsp; ![][7]

&emsp;&emsp;图1-6  在Worker节点上执行jps命令

&emsp;&emsp;在图中可以清晰看到，Master进程与Worker及相关进程在各自节点上成功运行，Hadoop与Spark运行正常。

&emsp;&emsp;(3) 通过Spark web UI查看集群状态

&emsp;&emsp;在浏览器中输入Master的ip与端口，打开Spark web UI,如图1-7所示：

&emsp;&emsp; ![][8]

&emsp;&emsp;图1-7 Spark web UI界面

&emsp;&emsp;从图中可以看到，当集群内仅有一个worker节点的时候，spark web UI显示该节点处于Alive状态，CPU cores为1，内存为1G。 此页面会列出集群中所有启动后的worker节点及应用的信息。

&emsp;&emsp;(4) 运行样例。

&emsp;&emsp;Spark自带了一些样例程序可供试运行。在Spark根目录下example/src/main文件夹里存放着Scala,Java,Python及R语言编写的样例，用户可以运行其中的某个样例程序。先cd到Spark根目录下，然后执行： bin/run-example [class] [params]即可。 例如我们可以在Master主机命令行执行：

```bash
./run-example SparkPi 10
```

&emsp;&emsp;然后可以看到该应用的输出，并且在Spark Web UI上也可以查看应用的状态及其他信息。


1.4 Intellij IDEA的安装与配置

&emsp;&emsp;Intellij IDE是目前最流行的Spark开发环境。本节主要描述了Intellij开发工具的安装与配置。Intellij不但可以用来开发Spark应用，还可以用来作为Spark源代码的阅读器。

1.4.1   Intellij安装

&emsp;&emsp;Intellij开发环境依赖JDK,Scala。

&emsp;&emsp;(1) JDK的安装

&emsp;&emsp;Intellij IDE需要安装JDK 1.7或更高版本。Open JDK1.7的安装与配置前文中已讲过，这里不再赘述。

&emsp;&emsp;(2) Scala的安装

&emsp;&emsp;Scala的安装与配置前文已讲过，此处不再赘述。

&emsp;&emsp;(3) Intellij的安装

&emsp;&emsp;登录Intellij官方下载网站：http://www.jetbrains.com/idea/ 下载最新版Intellij linux安装包**ideaIC-14.1.5.tar.gz**。然后执行如下步骤：

&emsp;&emsp;&emsp;&emsp;(a)解压： __`tar zxvf ideaIC-14.1.5.tar.gz -C /usr/`__

&emsp;&emsp;&emsp;&emsp;(b)运行： cd 到解压后的目录。执行： __`./idea.sh`__

&emsp;&emsp;&emsp;&emsp;(c)安装Scala插件：打开"File" -> "Settings" -> "Plugins" -> "Install JetBrain plugin..." 运行后弹出窗口如图1-8所示:

&emsp;&emsp; ![][9]

&emsp;&emsp;图1-8 Scala插件弹出窗口

&emsp;&emsp;点击右侧绿色按钮开始安装Scala插件。

1.4.2   Intellij配置

&emsp;&emsp;(1) 在Intellij IDEA中新建Scala项目名叫“HelloScala”。如图1-9所示：

&emsp;&emsp; ![][10]

&emsp;&emsp; 图1-9 Intellij IDEA中新建Scala项目

&emsp;&emsp;(2) 选择菜单 “File”->"Project Structure"->"Libraries", 点击"+"号，选择 “java”，定位至前面Spark根目录下的lib目录，选中**spark-assembly-1.5.0-hadoop2.6.0.jar**，点击OK按钮。

&emsp;&emsp;(3) 与上一步相同，点击"+"号，选择“scala”，然后定位至前面已安装的scala目录，则scala相关库会被自动引用。

&emsp;&emsp;(4) 选择菜单“File”->"Project Structure" -> "Platform Settings" -> "SDKs", 点击“+”号,选择JDK，定位至JDK安装目录，点击OK。

&emsp;&emsp;至此，Intellij IDEA开发环境配置完毕，用户可以用它开发自己的Spark程序了。


1.5 Eclipse IDE的安装与配置

&emsp;&emsp;现在我们接着介绍如何安装Eclipse。与Intellij IDEA类似，Eclipse环境依赖于JDK与Scala的安装。JDK与Scala前文已经详细讲述过了，此不赘述。

&emsp;&emsp;对Ecplise而言，最初我们需要为已其选择版本号完全对应的scala插件才可以新建scala项目。不过自从有了scala IDE工具，问题大大简化了。因为scala IDE中集成好了eclipse，已经替我们完成了前面的工作。用户可以直接登录官网：http://scala-ide.org/download/sdk.html 下载安装。

&emsp;&emsp;安装后，进入scala IDE根目录下的bin目录， 执行： ./eclipse 启动IDE。

&emsp;&emsp;然后选择 “File” -> "New" -> "Scala Project" 打开项目配置页。

&emsp;&emsp;输入项目名称，如HelloScala， 然后选择已经安装好的JDK版本，点击Finish。接下来就可以进行开发工作了，如图1-10所示：

&emsp;&emsp;![][11]

&emsp;&emsp; 图1-10 已经创建好的HelloScala项目

1.6 使用Spark shell开发运行Spark程序

&emsp;&emsp;Spark shell是一种学习API的简单途径，也是分析数据集交互的有力工具。

&emsp;&emsp;虽然本章还没涉及到Spark的具体技术细节，但从总体上说，Spark弹性数据集RDD有两种创建方式：

&emsp;&emsp;(1)从文件系统输入（如HDFS）

&emsp;&emsp;(2)从已存在的RDD转换得到新的RDD

&emsp;&emsp;现在我们从RDD入手，利用Spark shell简单演示如何书写并运行Spark程序。下面以word count这个经典例子来说明。

&emsp;&emsp;(a)启动spark shell:  cd 进SPARK_HOME/bin， 执行 

>__`./spark-shell`__

&emsp;&emsp;(b)进入scala命令行，执行如下：

&emsp;&emsp;`scala> val file = sc.textFile("hdfs://localhost:50040/hellosparkshell")`

&emsp;&emsp;`scala>  val count = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_+_)`  

&emsp;&emsp;`scala> count.collect()`

&emsp;&emsp;首先从本机上读取文件hellosparkshell，然后对该文件解析，最后统计单词及其数量并输出如下：

&emsp;&emsp;`15/09/29 16:11:46 INFO spark.SparkContext: Job finished: collect at <console>:17, took 1.624248037 s`

&emsp;&emsp;`res5: Array[(String, Int)] = Array((hello,12), (spark,12), (shell,12), (this,1), (is,1), (chapter,1), (three,1)`


1.7 本章小结

&emsp;&emsp;本章着重描述了Spark的生态及架构，使读者对Spark的平台体系有初步的了解。进而又
描述了如何在Linux平台上构建Spark集群，帮助读者构建自己的Spark平台。在本章末尾描述了如何搭建Spark开发环境，有助于读者对Spark的开发工具有一定了解，并能独立搭建开发环境。



[1]: resources/model/point.png

[2]: resources/model/1-1Spark-Eco-BDAS.png

[3]: resources/model/1-2Distributed.png

[4]: resources/model/1-3SparkArchitecture.png

[5]: resources/model/1-4SparkExecutingDAG.png

[6]: resources/model/1-5JpsMaster.png

[7]: resources/model/1-6JpsWorker.png

[8]: resources/model/1-7SparkWebUI.png

[9]: resources/model/1-8IntellijPlugin.png

[10]: resources/model/1-9NewScalaProject.png

[11]: resources/model/1-10ScalaIDE.png