# Hadoop监控代码分析

## 基本配置方法

Hadoop监控实现了灵活的配置机制，可根据实际需求，在配置文件中指定采用什么方式（文件或者Ganglia等监控系统）收集Hadoop指标。

一个简单的配置示例如下：

![][1]

配置方法类似Log4j，遵循Java Property文件的定义格式。

例子中，*.sink.foo.class定义了监控中所有prefix（监控配置项中的prefix代表不同的Hadoop被监控进程，比如namenode、datanode等），都定义了一个取名为foo的Sink，其对应类实现为org.apache.hadoop.metrics2.sink.FileSink。于是，Hadoop各服务进程需要输出监控信息时，会调用FileSink的相应方法进行输出（实际FileSink是将监控信息输出到文件）。<br/>
之后的配置项，定义了指标存储的具体文件名称。如namenode.sink.foo.filename=/tmp/namenode-metrics.out，定义了namenode的监控指标存储的位置。

对于不同的被监控进程（prefix），可以定义不同的指标存储文件。

对于同一个被监控进程，也可以定义将其指标存储到不同的目标文件。<br/>
比如，在定义了 <br/>

```
*.sink.foo.class=org.apache.hadoop.metrics2.sink.FileSink
namenode.sink.foo.filename=/tmp/namenode-metrics.out
```

将namenode的指标输出到/tmp/namenode-metrics.out的同时，还可以做如下定义，将其监控指标输出到别的文件（比如/tmp/namenode-metrics2.out）：

```
*.sink.foo2.class=org.apache.hadoop.metrics2.sink.FileSink
namenode.sink.foo2.filename=/tmp/namenode-metrics2.out
```

除了将sink定义为FileSink，将监控指标输出到文件，当将Sink定义为org.apache.hadoop.metrics2.sink.ganglia.GangliaSink31，可以将指标输出到第三方监控系统Ganglia，将指标以图形界面方式进行展示。

## 实现原理及代码

Hadoop监控机制围绕Metrics System、Metrics Source、Metrics Sink三个主要的角色展开。

![][2]

顾名思义，Metrics System负责协调监控体系各实体的运作，Metrics Source负责在被监控进程处收集（统计）指标，Metrics Sink负责具体的将指标输送到希望的目的地（文件或者监控系统等）。

代码实现中，MetricsSystem、MetricsSource、MeticsSink都是interface或者abstract class，其中MetricsSystem的默认实现类为MetricsSystemImpl，MetricsSource以及MeticsSink根据不同的需求场景，扩展出不同的实现类。

指标监控的整个操作流程，可以归纳为如下几步：

1. MetricsSource、MetricsSink通过调用MetricsSystem的register方法，注册到MetricsSystem上。
2. MetricsSource在被监控进程中，将关键的指标存入到MetricsSource对象的内存中。
3. MetricsSystem上启动定时器，每隔一段时间，调用已经在其上注册过的MetricsSource的getMetrics方法（MetricsSource在它的getMetrics逻辑中实现将内存中的指标数据反馈给MetricsSystem的目的（通过修改函数参数中传入的对象））。之后，MetricsSystem调用已经在其上注册过的MetricsSink的putMetrics方法，MetricsSink在各自的putMetrics方法中，实现将指标实际传输到目的地的逻辑。

以上过程可以用下图做简要描述：

![][3]

## 从一个实例进行了解

NodeManager是Hadoop Yarn体系中对任务节点资源进行管理的进程，实现类为org.apache.hadoop.yarn.server.nodemanager.NodeManager。类属性中包含如下监控相关部分：

```java
public class NodeManager extends CompositeService 
  implements EventHandler<NodeManagerEvent> {
  ...
  protected final NodeManagerMetrics metrics = NodeManagerMetrics.create();
  ...
```

这里，NodeMangerMetrics即为应用于NodeManger类对象的一个具体MetricsSource管理类：

```java
@Metrics(about="Metrics for node manager", context="yarn")
public class NodeManagerMetrics {
  @Metric MutableCounterInt containersLaunched;
  @Metric MutableCounterInt containersCompleted;
  @Metric MutableCounterInt containersFailed;
  @Metric MutableCounterInt containersKilled;
  @Metric("# of initializing containers")
      MutableGaugeInt containersIniting;
  @Metric MutableGaugeInt containersRunning;
  @Metric("Current allocated memory in GB")
      MutableGaugeInt allocatedGB;
  @Metric("Current # of allocated containers")
      MutableGaugeInt allocatedContainers;
  @Metric MutableGaugeInt availableGB;
  @Metric("Current allocated Virtual Cores")
      MutableGaugeInt allocatedVCores;
  @Metric MutableGaugeInt availableVCores;
  @Metric("Container launch duration")
      MutableRate containerLaunchDuration;
```

后面我们将看到，这个NodeMangerMetrics类似一个NodeManger中metrics的hub，它将提供方法，生成MetricsSource，管理这里用annotation方式（@Metric）声明的多个指标。

### 1. 注册

#### 流程

回顾刚才NodeManger中，对于NodeMangerMetrics的调用NodeMangerMetrics.create()的具体实现：

```java
public static NodeManagerMetrics create() {
  return create(DefaultMetricsSystem.instance());
}

static NodeManagerMetrics create(MetricsSystem ms) {
  JvmMetrics.create("NodeManager", null, ms);
  return ms.register(new NodeManagerMetrics());
}
```

create()调用create(DefaultMetricsSystem.instance())，参数中的DefaultMetricsSystem.instance()会生成一个MetricsSystem的单例，其实是一个MetricsSystemImpl对象，生成方法后面再说。

create(MetricsSystem ms)方法中，第一行调用JvmMetrics.create("NodeManager", null, ms)，的具体实现：

```java
public static JvmMetrics create(String processName, String sessionId,
                                  MetricsSystem ms) {
  return ms.register(JvmMetrics.name(), JvmMetrics.description(),
                     new JvmMetrics(processName, sessionId));
}
```

实际就是将JvmMetrics register到了参数传进来的这个单例MetricsSystem中。而这个JvmMetrics其实就是MetricsSource的一个实现类：

```java
public class JvmMetrics implements MetricsSource {
  ...
```

回到NodeManagerMetrics的create(MetricsSystem ms)中的第二条调用ms.register(new NodeManagerMetrics())。

这里同样调用了MetricsSystem（ms对象）的register方法，但是，参数中的“new NodeManagerMetrics()”生成的并不是MetricsSource对象（因为NodeManagerMetrics并不是MetricsSource的实现类，见上）。看看这个register方法的实现：

在MetricsSystem代码中：

```java
public abstract <T> T register(String name, String desc, T source);
public <T> T register(T source) {
  return register(null, null, source);
}
```

而MetricsSystem的具体实现类MetricsSystemImpl当中：

```java
@Override public synchronized <T>
T register(String name, String desc, T source) {
  MetricsSourceBuilder sb = MetricsAnnotations.newSourceBuilder(source);
  final MetricsSource s = sb.build();
  MetricsInfo si = sb.info();
  String name2 = name == null ? si.name() : name;
  final String finalDesc = desc == null ? si.description() : desc;
  final String finalName = // be friendly to non-metrics tests
      DefaultMetricsSystem.sourceName(name2, !monitoring);
  allSources.put(finalName, s);
  LOG.debug(finalName +", "+ finalDesc);
  if (monitoring) {
    registerSource(finalName, finalDesc, s);
  }
  // We want to re-register the source to pick up new config when the
  // metrics system restarts.
  register(finalName, new AbstractCallback() {
    @Override public void postStart() {
      registerSource(finalName, finalDesc, s);
    }
  });
  return source;
}
```

聚焦这个方法的前两行：

```java
MetricsSourceBuilder sb = MetricsAnnotations.newSourceBuilder(source);
final MetricsSource s = sb.build();
```

很明显，经过MetricsAnnotations.newSourceBuilder，以及MetricsSourceBuilder.build两个方法的调用，将一个普通的对象，转换成了MetricsSource（在之后内容再进行详细描述）。

再然后，调用registerSource方法，将Metrics注册到MetricsSystem当中：

```java
void registerSource(String name, String desc, MetricsSource source) {
  checkNotNull(config, "config");
  MetricsConfig conf = sourceConfigs.get(name);
  MetricsSourceAdapter sa = conf != null
      ? new MetricsSourceAdapter(prefix, name, desc, source,
                                 injectedTags, period, conf)
      : new MetricsSourceAdapter(prefix, name, desc, source,
        injectedTags, period, config.subset(SOURCE_KEY));
  sources.put(name, sa);
  sa.start();
  LOG.debug("Registered source "+ name);
}
```

这里涉及到一个更细节的控制，MetricsSystemImpl用一个HashMap类型的属性sources将所有的source管理起来（当然，每个source在这里都被一个MetricsSourceAdapter进行了封装）。

#### 补充

**1. MetricsSystem单例的生成方法**

NodeManager中用DefaultMetricsSystem.instance()生成MetricsSystem实例：

```java
public enum DefaultMetricsSystem {
  INSTANCE; // 单例

  private AtomicReference<MetricsSystem> impl =
      new AtomicReference<MetricsSystem>(new MetricsSystemImpl());
  volatile boolean miniClusterMode = false;
  transient final UniqueNames mBeanNames = new UniqueNames();
  transient final UniqueNames sourceNames = new UniqueNames();

  /**
   * Convenience method to initialize the metrics system
   * @param prefix  for the metrics system configuration
   * @return the metrics system instance
   */
  public static MetricsSystem initialize(String prefix) {
    return INSTANCE.init(prefix);
  }

  MetricsSystem init(String prefix) {
    return impl.get().init(prefix);
  }
  ...
```

DefaultMetricsSystem为枚举类型，其中声明了AtomicReference<MetricsSystem>类型的实例impl属性，用AtomicReference实现多线程同时申请实例时可能出现的冲突。真是的单例对象，实际由new MetricsSystemImpl()产生。

因此，默认情况下，MetricsSystem的单例对象为MetricsSystemImpl的实例。

**2. NodeManagerMetrics转换成MetricsSource的过程** 

回顾这个过程使用到的方法调用：

```java
MetricsSourceBuilder sb = MetricsAnnotations.newSourceBuilder(source);
final MetricsSource s = sb.build();
```

MetricsAnnotations类的newSourceBuilder方法实现：

```java
public static MetricsSourceBuilder newSourceBuilder(Object source) {
  return new MetricsSourceBuilder(source,
      DefaultMetricsFactory.getAnnotatedMetricsFactory());
}
```

创建了一个MetricsSourceBuilder，并将需要转换的源对象作为构造函数的第一个参数传入。

当MetricsSourceBuilder对象创建完成之后，调用该对象的build方法，将之前构造函数传入的source对象转换成MetricsSource对象。

```java
public MetricsSource build() {
  if (source instanceof MetricsSource) {
    if (hasAtMetric && !hasRegistry) {
      throw new MetricsException("Hybrid metrics: registry required.");
    }
    return (MetricsSource) source;
  }
  else if (!hasAtMetric) {
    throw new MetricsException("No valid @Metric annotation found.");
  }
  return new MetricsSource() {
    @Override
    public void getMetrics(MetricsCollector builder, boolean all) {
      registry.snapshot(builder.addRecord(registry.info()), all);
    }
  };
}
```

该build方法最终返回的就是一个新建匿名类对象，实现了MetricsSource接口的getMetrics，这里实际就是获取之前构造函数传入的那个对象中，用annotation声明的所有Metrics。

### 2. 收集及发送指标

在DefaultMetricsSystem的initialize方法中，MetricsSystem的实现类MetricsSystemImpl的init方法会被调用。init进而调用start，在start方法中，启动了一个定时器：

```java
private synchronized void startTimer() {
  if (timer != null) {
    LOG.warn(prefix +" metrics system timer already started!");
    return;
  }
  logicalTime = 0;
  long millis = period * 1000;
  timer = new Timer("Timer for '"+ prefix +"' metrics system", true);
  timer.scheduleAtFixedRate(new TimerTask() {
        public void run() {
          try {
            onTimerEvent();
          }
          catch (Exception e) {
            LOG.warn(e);
          }
        }
      }, millis, millis);
  LOG.info("Scheduled snapshot period at "+ period +" second(s).");
}

synchronized void onTimerEvent() {
  logicalTime += period;
  if (sinks.size() > 0) {
    publishMetrics(sampleMetrics(), false);
  }
}
```

定时器在定时触发的处理函数onTimerEvent中，先后触发sampleMetrics，以及publishMetrics两个方法。

在sampleMetrics实现中：

```java
synchronized MetricsBuffer sampleMetrics() {
  collector.clear();
  MetricsBufferBuilder bufferBuilder = new MetricsBufferBuilder();

  for (Entry<String, MetricsSourceAdapter> entry : sources.entrySet()) {
    if (sourceFilter == null || sourceFilter.accepts(entry.getKey())) {
      snapshotMetrics(entry.getValue(), bufferBuilder);
    }
  }
  if (publishSelfMetrics) {
    snapshotMetrics(sysSource, bufferBuilder);
  }
  MetricsBuffer buffer = bufferBuilder.get();
  return buffer;
}
```

遍历之前向MetricsSystem注册过的Source，调用其getMetrics方法，并放到buffer当中返回。

sampleMetrics返回的buffer，会作为publishMetrics调用时的第一个参数。

而publishMetrics的实现：

```java
synchronized void publishMetrics(MetricsBuffer buffer, boolean immediate) {
  int dropped = 0;
  for (MetricsSinkAdapter sa : sinks.values()) {
    long startTime = Time.now();
    boolean result;
    if (immediate) {
      result = sa.putMetricsImmediate(buffer); 
    } else {
      result = sa.putMetrics(buffer, logicalTime);
    }
    dropped += result ? 0 : 1;
    publishStat.add(Time.now() - startTime);
  }
  droppedPubAll.incr(dropped);
}
```

遍历之前注册到MetricsSystem上的Sink（注册过程跟Source类似），调用他们的putMetrics方法，将buffer中的内容发送到指标的目的地。


[1]: resources/metricsconf.png
[2]: resources/metricsarch.png
[3]: resources/metricscontrol.png
