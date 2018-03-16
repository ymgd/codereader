# 第?章 Executor解析

&emsp;&emsp;Executor是Spark分布式运行的承载，其会分布在不同的Worker节点上的线程池中运行。本章节尝试通过剖析Executor的源码，以分析实现的细节，帮助读者在实际工作中定位与调试问题。



## x.y Executor.scala源码综述

&emsp;&emsp;此源码实现了Executor类及其伴生对象。Executor类属于Spark内核中很关键的一个组成部分，是Worker节点中Spark任务运行的承载。通过外围一系列的切分、调度、分解等，形成可运行的Task，由Worker节点上的NM容器启动执行。从此源码整体大的情况来看，Executor包含了TaskReaper和TaskRunner两个子类，还有两个私有的线程池threadPool和taskReaperPool，如下图所示：

![][1]

&emsp;&emsp;其中，TaskReaper用于管理杀死或取消一个Task的过程，并且需要监控这个Task直到其结束，后面进一步的源码分析会说明，TaskReaper的设计是很精妙的，考虑了用户多次发起杀死或取消的要求；TaskRunner则是包装了需要执行的Task。

&emsp;&emsp;整体来看，这部分代码相对较长，功能较多，实现了Task相关线程的管理，包括相关线程池的建立、Task的启停、心跳信号的处理、通过ThreadMXBean获取到各种线程管理方面的信息、建立任务的各种监测指标、相关Task及其依赖的序列化或反序列化（便于发送到分布式Worker节点上执行）等等。以下尝试对其主要功能进行详细剖析：

### x.y.z   线程池threadPool及taskReaperPool详解
&emsp;&emsp;线程池threadPool用于管理具体工作线程，其通过com.google.common.util.concurrent.ThreadFactoryBuilder建立线程工厂，再通过标准的java.util.concurrent并发工具包提供的Executors.newCachedThreadPool建立线程池。详见如下源码：

```scala
[org.apache.spark.executor]
[Executor.scala]
...
import com.google.common.util.concurrent.ThreadFactoryBuilder
import java.util.concurrent._
...
以下代码为private[spark] class Executor内部：
  private val threadPool = {
    val threadFactory = new ThreadFactoryBuilder()
      .setDaemon(true)
      .setNameFormat("Executor task launch worker-%d")
      .setThreadFactory(new ThreadFactory {
        override def newThread(r: Runnable): Thread =
          // Use UninterruptibleThread to run tasks so that we can allow running codes without being
          // interrupted by `Thread.interrupt()`. Some issues, such as KAFKA-1894, HADOOP-10622,
          // will hang forever if some methods are interrupted.
          new UninterruptibleThread(r, "unused") // thread name will be set by ThreadFactoryBuilder
      })
      .build()
    Executors.newCachedThreadPool(threadFactory).asInstanceOf[ThreadPoolExecutor]
  }
...
```

&emsp;&emsp;可以看到，以上代码建立线程工厂时，设置了一些属性，包括设定了线程的名字格式"Executor task launch worker-%d"，重载newThread方法，建立一些不被Thread.interrupt()中断的线程等等，最后通过newCachedThreadPool建立线程池threadPool。注意，在Scala的函数式语义里，不可变变量threadPool所取的值是大括号对{}之间最后一条语句的返回值。
&emsp;&emsp;而另一个线程池taskReaperPool用于管理杀死或者取消Task所使用的线程，具体代码如下所示：

```scala
[org.apache.spark.executor]
[Executor.scala]
...
import org.apache.spark.util._
...
以下代码为private[spark] class Executor内部：
// Pool used for threads that supervise task killing / cancellation
  private val taskReaperPool = ThreadUtils.newDaemonCachedThreadPool("Task reaper")
  // For tasks which are in the process of being killed, this map holds the most recently created
  // TaskReaper. All accesses to this map should be synchronized on the map itself (this isn't
  // a ConcurrentHashMap because we use the synchronization for purposes other than simply guarding
  // the integrity of the map's internal state). The purpose of this map is to prevent the creation
  // of a separate TaskReaper for every killTask() of a given task. Instead, this map allows us to
  // track whether an existing TaskReaper fulfills the role of a TaskReaper that we would otherwise
  // create. The map key is a task id.
  private val taskReaperForTask: HashMap[Long, TaskReaper] = HashMap[Long, TaskReaper]()  
```
&emsp;&emsp;可以从以上代码的注释看到，为了防止每一次调用killTask都去生成一个独立的TaskReaper，需要一个称为taskReaperForTask的哈希表（哈希key是task id），我们可以根据这个哈希表去跟踪一个已有的TaskReaper是否已经完成了它的功能（杀死或取消Task），而不是去重新再新建一个TaskReaper。另外，考虑到我们需要的同步并不是简单的维护哈希表的内部状态就可以了，而是应该有更多的原子性方面的要求，所以我们没有使用并行哈希表ConcurrentHashMap，而是使用了普通的HashMap，并在访问时通过synchonrized实现同步。




[1]: resources/executor/1_executor_arch.png
[x]: resources/model/4-1Code-Layout.png
[2]: resources/model/4-2Spark-Sequence.png
[3]: resources/model/4-3Spark-Sequence2.png