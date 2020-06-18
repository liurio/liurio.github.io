---
logout: post
title: 清理过期checkpont引发的问题
tags: [flink,all]
---

### Flink Checkpoint目录清除策略

```java
CheckpointConfig config = env.getCheckpointConfig();
config.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
config.enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION);
```

`env.getCheckpointConfig().enableExternalizedCheckpoints()`标识当Flink任务取消时，是否保留外部保存的checkpoint信息。该参数由两种枚举：`DELETE_ON_CANCELLATION` 和`DELETE_ON_CANCELLATION`。

- DELETE_ON_CANCELLATION：仅当作业失败时，作业的 Checkpoint 才会被保留用于任务恢复。当作业取消时，Checkpoint 状态信息会被删除，因此取消任务后，不能从 Checkpoint 位置进行恢复任务。
- RETAIN_ON_CANCELLATION：当作业手动取消时，将会保留作业的 Checkpoint 状态信息。注意，这种情况下，**需要手动清除该作业保留的 Checkpoint 状态信息，否则这些状态信息将永远保留在外部的持久化存储中**。

无论是选择上述哪种方式，后面都提示了一句：如果 Flink 任务失败了，Checkpoint 的状态信息将被保留。简言之，Flink 任务重启或者失败后，Checkpoint 信息会保存在 hdfs，如果不手动定期清理，那么一年以后 Checkpoint 信息还会保存在 hdfs 上。请问一年后这个 Checkpoint 信息还有用吗？肯定没用了，完全是浪费存储资源。

### 哪些状态信息应该被删除掉？

我们的判断逻辑：**不会再使用的状态信息就是那些应该被清理掉的状态信息。**

#### 清理判断的依据

如下图所示，Flink 中配置了 Checkpoint 目录为：`/user/flink/checkpoints`，子目录名为 jobId 名。hdfs 的 api 可以拿到 jobId 目录最后修改的时间。对于一个正在运行的 Flink 任务，每次 Checkpoint 时 jobId 的 Checkpoint 目录最后修改时间会更新为当前时间。可以认为如果 jobId 对应的最后修改时间是 10 天之前，意味着这个 job 已经十天没有运行了。

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/%E6%B8%85%E7%90%86%E5%88%A4%E6%96%AD%E4%BE%9D%E6%8D%AE.jpg](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/清理判断依据.jpg)

因此，清理策略如下：定时每天调度一次，将那些最后修改时间是 10 天之前的 jobId 对应的 Checkpoint 目录清理掉。

#### 出现问题

出现了任务报错，不能从checkpoint恢复。报错如下：

```shel
任务从 /user/flink/checkpoints/0165a1467c65a95a010f9c0134c9e576/chk-17052 目录恢复时，报出以下错误：
Caused by: java.io.FileNotFoundException: File does not exist: /user/flink/checkpoints/37c37bd26da17b2e6ac433866d51201d/shared/bdc46a09-fc2a-4290-af5c-02bd39e9a1ca
```

#### 原因分析

Flink基于RocksDB的增量checkpoint备份。增量的本质是`上一次的_metadata + 本次sst文件`。该任务依赖了之前任务的chk文件，由于最新目录下只存储了之前任务的`_metadata`信息，而这个文件又被清理掉了。

### RocksDB增量CheckPoint实现原理

RocksDB 是一个基于 LSM 实现的 KV 数据库。LSM 全称 Log Structured Merge Trees，LSM 树本质是将大量的磁盘随机写操作转换成磁盘的批量写操作来极大地提升磁盘数据写入效率。一般 LSM Tree 实现上都会有一个基于内存的 MemTable 介质，所有的增删改操作都是写入到 MemTable 中，当 MemTable 足够大以后，将 MemTable 中的数据 flush 到磁盘中生成不可变且内部有序的 **ssTable（Sorted String Table）**文件，全量数据保存在磁盘的多个 ssTable 文件中。HBase 也是基于 LSM Tree 实现的，HBase 磁盘上的 HFile 就相当于这里的 ssTable 文件，每次生成的 HFile 都是不可变的而且内部有序的文件。基于 ssTable 不可变的特性，才实现了增量 Checkpoint，具体流程如下所示：

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/rocksdb%E5%A2%9E%E9%87%8Fchk%E5%8E%9F%E7%90%86.jpg](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/rocksdb增量chk原理.jpg)

1. 第一次 Checkpoint 时生成的状态快照信息包含了两个 sstable 文件：sstable1 和 sstable2 及 Checkpoint1 的元数据文件 MANIFEST-chk1，并上传到外部持久化存储中。
2. 第二次 Checkpoint 时生成的快照信息为 sstable1、sstable2、sstable3 及元数据文件 MANIFEST-chk2，由于 sstable 文件的不可变特性，所以状态快照信息的 sstable1、sstable2 这两个文件并没有发生变化，sstable1、sstable2 这两个文件不需要重复上传到外部持久化存储中，因此第二次 Checkpoint 时，只需要将 sstable3 和 MANIFEST-chk2 文件上传到外部持久化存储中即可。这里只将新增的文件上传到外部持久化存储，也就是所谓的增量 Checkpoint。

基于 LSM Tree 实现的数据库为了提高查询效率，都需要定期对磁盘上多个 sstable 文件进行合并操作，合并时会将删除的、过期的以及旧版本的数据进行清理，从而降低 sstable 文件的总大小。图中可以看到第三次 Checkpoint 时生成的快照信息为sstable3、sstable4、sstable5 及元数据文件 MANIFEST-chk3， 其中新增了 sstable4 文件且 sstable1 和 sstable2 文件合并成 sstable5 文件，因此第三次 Checkpoint 时只需要向外部持久化存储上传 sstable4、sstable5 及元数据文件 MANIFEST-chk3。

基于 RocksDB 的增量 Checkpoint 从本质上来讲每次 Checkpoint 时只将本次 Checkpoint 新增的快照信息上传到外部的持久化存储中，依靠的是 LSM Tree 中 sstable 文件不可变的特性。

### 如何合理的删除checkpoint目录？

由于合并操作完全依赖 RocksDB，所以时间策略肯定不靠谱。Checkpoint 的次数也不靠谱，不能说进行了 1000次 Checkpoint 了，第一次 Checkpoint 的 sstable 文件肯定已经被废弃了，还真有可能在使用，虽然几率很小。所以我们现在需要一个主动检测的策略，需要主动去发现，到底哪些 sstable 文件在使用呢？

思路：在`../../jobid/`路径下有`_metadata`文件，这里存储着本次 Checkpoint 的元数据信息，元数据信息中存储了本次 Checkpoint 依赖哪些 sstable 文件并且 sstable 文件存储在 hdfs 的位置信息。所以只需要解析最近一次 Checkpoint 目录的元数据文件就可以知道当前的 job 依赖哪些 sstable 文件。

```java
//  读取元数据文件
File f=new File("module-flink/src/main/resources/_metadata");
//第二步，建立管道,FileInputStream文件输入流类用于读文件
FileInputStream fis=new FileInputStream(f);
BufferedInputStream bis = new BufferedInputStream(fis);
DataInputStream dis = new DataInputStream(bis);

// 通过 Flink 的 Checkpoints 类解析元数据文件
Savepoint savepoint = Checkpoints.loadCheckpointMetadata(dis,
        MetadataSerializer.class.getClassLoader());
// 打印当前的 CheckpointId
System.out.println(savepoint.getCheckpointId());

// 遍历 OperatorState，这里的每个 OperatorState 对应一个 Flink 任务的 Operator 算子
// 不要与 OperatorState  和 KeyedState 混淆，不是一个层级的概念
for(OperatorState operatorState :savepoint.getOperatorStates()) {
    System.out.println(operatorState);
    // 当前算子的状态大小为 0 ，表示算子不带状态，直接退出
    if(operatorState.getStateSize() == 0){
        continue;
    }

    // 遍历当前算子的所有 subtask
    for(OperatorSubtaskState operatorSubtaskState: operatorState.getStates()) {
        // 解析 operatorSubtaskState 的 ManagedKeyedState
        parseManagedKeyedState(operatorSubtaskState);
        // 解析 operatorSubtaskState 的 ManagedOperatorState
        parseManagedOperatorState(operatorSubtaskState);

    }
}


/**
 * 解析 operatorSubtaskState 的 ManagedKeyedState
 * @param operatorSubtaskState operatorSubtaskState
 */
private static void parseManagedKeyedState(OperatorSubtaskState operatorSubtaskState) {
    // 遍历当前 subtask 的 KeyedState
    for(KeyedStateHandle keyedStateHandle:operatorSubtaskState.getManagedKeyedState()) {
        // 本案例针对 Flink RocksDB 的增量 Checkpoint 引发的问题，
        // 因此仅处理 IncrementalRemoteKeyedStateHandle
        if(keyedStateHandle instanceof IncrementalRemoteKeyedStateHandle) {
            // 获取 RocksDB 的 sharedState
            Map<StateHandleID, StreamStateHandle> sharedState =
               ((IncrementalRemoteKeyedStateHandle) keyedStateHandle).getSharedState();
            // 遍历所有的 sst 文件，key 为 sst 文件名，value 为对应的 hdfs 文件 Handle
            for(Map.Entry<StateHandleID,StreamStateHandle> entry:sharedState.entrySet()){
                // 打印 sst 文件名
                System.out.println("sstable 文件名：" + entry.getKey());
                if(entry.getValue() instanceof FileStateHandle) {
                    Path filePath = ((FileStateHandle) entry.getValue()).getFilePath();
                    // 打印 sst 文件对应的 hdfs 文件位置
                    System.out.println("sstable文件对应的hdfs位置:" + filePath.getPath());
                }
            }
        }
    }
}


/**
 * 解析 operatorSubtaskState 的 ManagedOperatorState
 * 注：OperatorState 不支持 Flink 的 增量 Checkpoint，因此本案例可以不解析
 * @param operatorSubtaskState operatorSubtaskState
 */
private static void parseManagedOperatorState(OperatorSubtaskState operatorSubState) {
    // 遍历当前 subtask 的 OperatorState
    for(OperatorState operatorStateHandle:operatorSubState.getManagedOperatorState()) {
        StreamStateHandle delegateState = operatorStateHandle.getDelegateStateHandle();
        if(delegateState instanceof FileStateHandle) {
            Path filePath = ((FileStateHandle) delegateStateHandle).getFilePath();
            System.out.println(filePath.getPath());
        }
    }
}
```