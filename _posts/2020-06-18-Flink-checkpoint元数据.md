---
logout: post
title: Flink checkpoint元数据
tags: [flink,all]
---

### Checkpoint完整的元数据

`CompletedCheckpoint` 封装了一次 Checkpoint 完整的元数据信息.

```java
public class CompletedCheckpoint implements Serializable {
     private final JobID job;
     private final long checkpointID;
     private final long timestamp;
     private final long duration;

     /** 本次 Checkpoint 中每个算子的 ID 及算子对应 State 信息 */
     private final Map<OperatorID, OperatorState> operatorStates;
     private final CheckpointProperties props;
     private final Collection<MasterState> masterHookStates;
     // Checkpoint 存储路径
     private final CompletedCheckpointStorageLocation storageLocation;
     // 元数据句柄
     private final StreamStateHandle metadataHandle;
      // Checkpoint 目录地址
     private final String externalPointer;
     private transient volatile CompletedCheckpointStats.DiscardCallback discardCallback;
}
```

### 算子级别的元数据 OperatorState

```java
public class OperatorState implements CompositeStateHandle {
     private final OperatorID operatorID;

     // checkpoint 时算子的并行度
     private final int parallelism;

     // checkpoint 时算子的 maxParallelism
     private final int maxParallelism;

     // 当前 Operator 算子内，每个 subtask 持有的 State 信息，
     // 这里 map 对应的 key 为 subtaskId，value 为 subtask 对应的 State,
     // OperatorState 表示一个 算子级别的，OperatorSubtaskState 是 subtask 级别的。
     // 如果一个算子有 10 个并行度，那么 OperatorState 有 10 个 OperatorSubtaskState
     private final Map<Integer, OperatorSubtaskState> operatorSubtaskStates;
}
```

### subtask级别的元数据 OperatorSubtaskState

```java
public class OperatorSubtaskState implements CompositeStateHandle {
    // managedOperatorState 元数据维护在 OperatorStateHandle 中，
    // managedKeyedState 元数据存储维护在 KeyedStateHandle 中。
     private final StateObjectCollection<OperatorStateHandle> managedOperatorState;
     private final StateObjectCollection<OperatorStateHandle> rawOperatorState;
     private final StateObjectCollection<KeyedStateHandle> managedKeyedState;
     private final StateObjectCollection<KeyedStateHandle> rawKeyedState;
     private final long stateSize;
}
```

### **OperatorStateHandle** 句柄

`OperatorStateHandle` 是一个接口，它只有一种实现`OperatorStreamStateHandle`

```java
public class OperatorStreamStateHandle implements OperatorStateHandle {
  	 // map 中 key 是 StateName，value 是 StateMetaInfo
     // StateMetaInfo 中封装的是当前 State 在状态文件所处的 offset 和 Mode
     private final Map<String, StateMetaInfo> stateNameToPartitionOffsets;

     // OperatorState 状态文件句柄，可以读出状态数据,根据 StreamStateHandle，
     // 即当前 subtask 可以从这个文件中读取状态数据。
     private final StreamStateHandle delegateStateHandle;
}

// OperatorState 分布模式的枚举
enum Mode {
     // 对应 getListState API,表示每个 subtask 只获取一部分状态数据，即：所有 subtask 的状态加起来是一份全量的。
     SPLIT_DISTRIBUTE,
     // 对应 getUnionListState API,表示每个 subtask 获取一份全量的状态数据
     UNION,
     // 对应 BroadcastState
     BROADCAST
}

class StateMetaInfo implements Serializable {
      // 当前 State 在状态文件所处的 offset 和 Mode
     private final long[] offsets;
      // OperatorState 的分布模式
     private final Mode distributionMode;
}
```

### **KeyedStateHandle** 句柄

`KeyedStateHandle`子类较多，常用的`KeyGroupsStateHandle `和`IncrementalRemoteKeyedStateHandle `。`IncrementalLocalKeyedStateHandle` 和 `DirectoryKeyedStateHandle` 是对 RocksDB Increment 模式的优化。RocksDB 在 Increment 模式开启 `local-recovery`，可以在本地目录存放一份 State，当从 Checkpoint 处恢复时，不用去 dfs 去拉，而是直接从本地目录恢复 State。

#### KeyGroupsStateHandle 句柄

```java
public class KeyGroupsStateHandle implements StreamStateHandle, KeyedStateHandle {
     // KeyedState 状态文件句柄，可以读出状态数据
     private final StreamStateHandle stateHandle;
     // KeyGroupRangeOffsets 封装了当前负责的 KeyGroupRange
      // 及 KeyGroupRange 中每个 KeyGroup 对应的 State 在 stateHandle 的 offset 位置
     private final KeyGroupRangeOffsets groupRangeOffsets;
}
public class KeyGroupRangeOffsets implements Iterable<Tuple2<Integer, Long>> , Serializable {
     // 当前 Operator 当前 subtask 负责的 KeyGroupRange
     private final KeyGroupRange keyGroupRange;

     // 数组保存了每个 KeyGroup 对应的 offset，
      // 所以：数组的长度 == keyGroupRange 中 KeyGroup 的数量
     private final long[] offsets;
}
```

#### IncrementalRemoteKeyedStateHandle 句柄

`IncrementalRemoteKeyedStateHandle` 应用于 RocksDB 增量 Checkpoint 模式, 我们知道 Checkpoint 实际存储的是 RocksDB 数据库的 sst 文件和 RocksDB 数据库的元数据文件。

```java
public class IncrementalRemoteKeyedStateHandle implements IncrementalKeyedStateHandle {
      // 每个 RocksDB 数据库的唯一 ID
     private final UUID backendIdentifier;

      // 这个 RocksDB 数据库负责的 KeyGroupRange
     private final KeyGroupRange keyGroupRange;

     private final long checkpointId;

     // RocksDB 真正存储数据的 sst 文件
     private final Map<StateHandleID, StreamStateHandle> sharedState;

     // RocksDB 数据库的一些元数据
     private final Map<StateHandleID, StreamStateHandle> privateState;

     // 本次 Checkpoint 元数据的 StateHandle
     private final StreamStateHandle metaStateHandle;
}

// StateHandleID 只是简单地对 sst 文件名做了封装
public class StateHandleID extends StringBasedID {
     private static final long serialVersionUID = 1L;
     // keyString 为 sst 文件名
     public StateHandleID(String keyString) {
      super(keyString);
 }
}
```

