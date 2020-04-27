

![flink-savepoint-3](http://www.liaojiayi.com/assets/flink-savepoint-3.png)

## Savepoint

### 概念

Apache Flink Savepoint允许你生成一个当前流式程序的快照。这个快照记录了整个程序的状态，包括数据处理的具体位置，如kafka的offset的信息。Flink采用Chandy-Lamport的快照算法来生成具有一致性的savepoint。savepoint包括两个主要元素：

1. 一个记录了当前流式程序所有状态的二进制文件（通常很大）。
2. 一个相对较小的metadata文件，它存储在你指定的分布式文件系统或者数据存储中，包括了savepoints的所有文件指针（路径）。

savepoint的创建持有删除都是由用户来控制，手动来实现备份和恢复。可以在用户变更job graph，秀改并行度，fork代码时恢复，从原理上来说savepoint的成本会更高一些，更关注程序在移植，变更之后的操作。

Flink程序中包含两种状态数据，一种是用户自定义的状态；另一种是系统状态，他们是作为operator计算一部分的数据buffer等状态数据，比如在使用Window Function时，在Window内部缓存Streaming数据记录。为了能够在创建Savepoint过程中，唯一识别对应的Operator的状态数据，Flink提供了API来为程序中每个Operator设置ID，这样可以在后续更新/升级程序的时候，可以在Savepoint数据中基于Operator ID来与对应的状态信息进行匹配，从而实现恢复。

```java
DataStream<String> stream = env.
  // 有状态的source ID (例如：Kafka)
  .addSource(new StatefulSource()).uid("source-id").name("source-id")
```

### 创建savepoint

- 通过conf/flink-conf.yaml配置

```shell
state.savepoints.dir: hdfs://namenode01.td.com/flink-1.5.3/flink-savepoints
```

- 通过命令行

在指定的jobid下创建目录

```sh
bin/flink savepoint 40dcc6d2ba90f13930abce295de8d038 hdfs://namenode01.td.com/tmp/flink/savepoints
```

### savepoint恢复

- 通过命令行

停掉Job 40dcc6d2ba90f13930abce295de8d038，然后通过Savepoint命令来恢复Job运行，命令格式如下所示：

```sh
bin/flink run -s hdfs://namenode01.td.com/tmp/flink/savepoints/savepoint-40dcc6-a90008f0f82f flink-app-jobs.jar
```

### 什么时候用Savepoint？

虽然流式程序处理的是无止境的流数据，但实际上，可能会有消费之前的已经被消费过的数据的需求。Savepoints可以帮助你满足一下场景：

- 更新生产环境的程序，包括新功能、bug修复或者更好的机器学习模型
- 在程序中引入A/B测试，从同一个数据源的同一个时间点开始消费，测试不同版本的效果
- 扩容，使用更多的集群资源
- 使用新版本的Flink，或者升级到新版本的Flink集群。

## Checkpoint

Checkpoint是Flink实现容错机制最核心的功能，它能够根据配置周期性地基于Stream中各个Operator的状态来生成Snapshot，从而将这些状态数据定期持久化存储下来，当Flink程序一旦意外崩溃时，重新运行程序时可以有选择地从这些Snapshot进行恢复，从而修正因为故障带来的程序数据状态中断。

checkpoint的原理见上一篇。[checkpoint原理](https://liurio.github.io/2019/12/21/Flink%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%E4%B8%8ECheckpoint%E6%9C%BA%E5%88%B6(%E4%B8%80)/)

### checkpoint设置

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 设置保存点的保存路径，这里是保存在hdfs中
env.setStateBackend(new FsStateBackend("hdfs://namenode01.td.com/flink-1.5.3/flink-checkpoints"));
CheckpointConfig config = env.getCheckpointConfig();
// 任务流取消和故障应保留检查点
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
// 保存点模式：exactly_once
config.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// 触发保存点的时间间隔
config.setCheckpointInterval(60000);
```

### 保存多个checkpoint

默认情况下，如果设置了Checkpoint选项，则Flink只保留最近成功生成的1个Checkpoint，而当Flink程序失败时，可以从最近的这个Checkpoint来进行恢复。但是，如果我们希望保留多个Checkpoint，并能够根据实际需要选择其中一个进行恢复，这样会更加灵活，比如，我们发现最近4个小时数据记录处理有问题，希望将整个状态还原到4小时之前。
Flink可以支持保留多个Checkpoint，需要在Flink的配置文件conf/flink-conf.yaml中，添加如下配置，指定最多需要保存Checkpoint的个数：

```sh
state.checkpoints.num-retained: 20
```

配置了state.checkpoints.num-retained的值为20，只保留了最近的20个Checkpoint。如果希望会退到某个Checkpoint点，只需要指定对应的某个Checkpoint路径即可实现。

### 恢复checkpoint

- 命令行

如果Flink程序异常失败，或者最近一段时间内数据处理错误，我们可以将程序从某一个Checkpoint点，比如chk-860进行回放，执行如下命令：恢复后文件的编号会基于运行的编号连续生成。

```sh
bin/flink run -s hdfs://namenode01.td.com/flink-1.5.3/flink-checkpoints/582e17d2cc343e6c56255d111bae0191/chk-860/_metadata flink-app-jobs.jar
```

- 从web页面恢复

![web页面](https://img-blog.csdnimg.cn/20200216143745249.png)

## savepoint和checkpoint区别

Checkpoints和Savepoints都是Apache Flink流式处理框架中比较特别的功能。它们在实现上类似，但在以下三个地方有些不同：

1. **目标**：从概念上讲，Savepoints和Checkpoints的不同之处类似于传统数据库中备份和恢复日志的不同。Checkpoints的作用是确保程序有潜在失败可能的情况下（如网络暂时异常），可以正常恢复。相反，Savepoints的作用是让用户手动触发备份后，通过重启来恢复程序。
2. **实现**：Checkpoints和Savepoints在实现上有所不同。Checkpoints轻量并且快速，它可以利用底层状态存储的各种特性，来实现快速备份和恢复。例如，以RocksDB作为状态存储，状态将会以RocksDB的格式持久化而不是Flink原生的格式，同时利用RocksDB的特性实现了增量Checkpoints。这个特性加速了checkpointing的过程，也是Checkpointing机制中第一个更轻量的实现。相反，Savepoints更注重数据的可移植性，并且支持任何对任务的修改，同时这也让Savepoints的备份和恢复成本相对更高。
3. **生命周期**：Checkpoints本身是定时自动触发的。它们的维护、创建和删除都由Flink自身来操作，不需要任何用户的干预。相反，Savepoints的触发、删除和管理等操作都需要用户手动触发。

| 维度     | Checkpoints                 | Savepoints             |
| -------- | --------------------------- | ---------------------- |
| 目标     | 任务失败的恢复/故障转移机制 | 手动备份/重启/恢复任务 |
| 实现     | 轻量快捷                    | 注重可移植性，成本较高 |
| 生命周期 | Flink自身控制               | 用户手动控制           |



## 转自

[Flink的Checkpoint和Savepoint介绍](https://blog.csdn.net/hxcaifly/article/details/84673292?depth_1-utm_source=distribute.pc_relevant_right.none-task-blog-BlogCommendFromBaidu-2&utm_source=distribute.pc_relevant_right.none-task-blog-BlogCommendFromBaidu-2)

[Flink - Savepoint vs Checkpoint](http://www.liaojiayi.com/flink-savepoint-vs-checkpoint/)