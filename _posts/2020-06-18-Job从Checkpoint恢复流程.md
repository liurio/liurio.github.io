---
logout: post
title: Job从Checkpoint恢复流程
tags: [flink,all]
---

## **Job 从 Checkpoint 恢复流程概述**

Flink 任务从 Checkpoint 或 Savepoint 处恢复的整体流程简单概述，如下所示

1. 首先客户端提供 Checkpoint 或 Savepoint 的目录
2. JM 从给定的目录中找到 _metadata 文件（Checkpoint 的元数据文件）
3. JM 解析元数据文件，做一些校验，将信息写入到 zk 中，然后准备从这一次 Checkpoint 中恢复任务
4. JM 拿到所有算子对应的 State，给各个 subtask 分配 StateHandle（状态文件句柄）
5. TM 启动时，也就是 StreamTask 的初始化阶段会创建 KeyedStateBackend 和 OperatorStateBackend
6. 创建过程中就会根据 JM 分配给自己的 StateHandle 从 dfs 上恢复 State

由上述流程可知，Flink 任务从 Checkpoint 恢复不只是说 TM 去 dfs 拉状态文件即可，需要 JM 先给各个 TM 分配 State，由于牵扯到修改并发，所以 JM 端给各个 subtask 分配 State 的流程也是比较复杂的。

## **JM 端恢复 Checkpoint 元数据流程**

JM 端从 Checkpoint 中恢复任务的流程是从 CheckpointCoordinator 的 restoreSavepoint 方法开始的，restoreSavepoint 方法的源码如下所示：

```java
public boolean restoreSavepoint(
  String savepointPointer,
  boolean allowNonRestored,
  Map<JobVertexID, ExecutionJobVertex> tasks,
  ClassLoader userClassLoader) throws Exception {
     // 从外部存储获取 Checkpoint 元数据，这里的 checkpointStorage 是 FsCheckpointStorage
     final CompletedCheckpointStorageLocation checkpointLocation = 
        checkpointStorage.resolveCheckpoint(savepointPointer);

     // 加载 Checkpoint 元数据，并对 Checkpoint 进行校验，
      // 校验项包括 maxParallelism、allowNonRestoredState
     CompletedCheckpoint savepoint = Checkpoints.loadAndValidateCheckpoint(
       job, tasks, checkpointLocation, userClassLoader, allowNonRestored);

     // 将要恢复的 Checkpoint 信息写入到 zk 中
     completedCheckpointStore.addCheckpoint(savepoint);

     // Reset the checkpoint ID counter
     long nextCheckpointId = savepoint.getCheckpointID() + 1;
     checkpointIdCounter.setCount(nextCheckpointId);

     // 从最近一次 Checkpoint 处恢复 State
     return restoreLatestCheckpointedState(tasks, true, allowNonRestored);
}
```

主要的流程分三个步骤：获取`_metadata`文件位置、加载元数据及校验逻辑、恢复元数据。

### 加载元数据及校验逻辑

`Checkpoints.loadAndValidateCheckpoint()` 方法加载 Checkpoint 元数据，并对 Checkpoint 进行校验，校验项包括 `maxParallelism`、`allowNonRestoredState`。

首先遍历新 Job 的 ExecutionGraph 中，通过 ExecutionGraph 生成 OperatorId 与 ExecutionJobVertex 的映射保存到 operatorToJobVertexMapping 中。然后循环检查所有的 OperatorState，这里的 OperatorState 不是指 Flink 的 OperatorState，而是指算子级别的 State（Operator 指代算子）。

在新的 ExecutionGraph 中找 Checkpoint 中 OperatorId 对应的算子保存到 executionJobVertex 中。executionJobVertex == null 说明有 OperatorState，但在新的 ExecutionGraph 中找不到对应的 executionJobVertex。

### 恢复元数据

校验流程结束后，会将本次 Checkpoint 信息写入 zk，便于从 Checkpoint 中恢复。相关代码如下：

```java
// 从 zk 获取所有可以恢复的 Checkpoint 信息，并从最近一次恢复
completedCheckpointStore.recover();

// Now, we re-register all (shared) states from the checkpoint store with the new registry
for (CompletedCheckpoint completedCheckpoint : 
     completedCheckpointStore.getAllCheckpoints()) {
 completedCheckpoint.registerSharedStatesAfterRestored(sharedStateRegistry);
}

// Restore from the latest checkpoint
CompletedCheckpoint latest = completedCheckpointStore.getLatestCheckpoint(XXX);

// Checkpoint 中恢复出来的 State
final Map<OperatorID, OperatorState> operatorStates = latest.getOperatorStates();

StateAssignmentOperation stateAssignmentOperation =
  new StateAssignmentOperation(latest.getCheckpointID(), tasks, operatorStates, allowNonRestoredState);

// 重点：JM 给各个 TM 分配 State
stateAssignmentOperation.assignStates();
```

restoreLatestCheckpointedState 方法会从最近一次 Checkpoint 中恢复，因为任务刚启动， zk 中只存放了刚刚保存的这一次 Checkpoint 信息，所以这里就是从用户刚指定的 Checkpoint 目录进行恢复。恢复出来的是 CompletedCheckpoint 对象，也就是那一次 Checkpoint 完成时对应的元数据信息。

### **JM 端为 subtask 分配 StateHandle 流程**

Checkpoint 元数据恢复完成后，JM 就会拿着元数据将 StateHandle 合理的分配给各个 `subtask (TM)`。

#### 循环遍历一个个 ExecutionJobVertex

1. 第一步仍然是对 State 进行检查。检查 Checkpoint 中恢复的所有 State 是否可以映射到新 Job 的 ExecutionGraph 上，并结合 allowNonRestoredState 参数进行校验。
2. 第二步从最新的 ExecutionGraph 中遍历一个个 ExecutionJobVertex ，一个个 ExecutionJobVertex 去进行 assign。可能存在 OperatorChain，因此 ExecutionJobVertex 中可能保存多个 Operator，所以这里遍历 ExecutionJobVertex 中对应的所有 Operator 算子，看一个个 Operator 算子是否有状态的。用变量 statelessTask 标识当前 ExecutionJobVertex 是否是有状态的，如果 statelessTask 为 true 表示当前 ExecutionJobVertex 上所有 Operator 都是无状态的，直接 continue，即：不分配 State。否则当前 ExecutionJobVertex 上存在有状态的 Operator，需要走分配 State 的逻辑。
3. `assignAttemptState(task.getValue(), operatorStates) `方法是真正分配 StateHandle 的逻辑，

#### 为 ExecutionJobVertex 分配 StateHandle

assignAttemptState 主要包含以下五个工作：

- 并行度检验：对并行度和 MaxParallelism 进行检验，并设置合理的 MaxParallelism
- 为当前 ExecutionJobVertex 的所有 subtask 生成对应的 KeyGroupRange
- 给当前 ExecutionJobVertex 的所有 subtask 重新分配 OperatorState
- 给当前 ExecutionJobVertex 的所有 subtask 重新分配 KeyedState
- 将生成的四种 StateHandle 封装到 ExecutionJobVertex 中

##### 并行度校验

校验并行度相关是否符合规则：

1. Job 的并行度是否超过 Checkpoint 状态中保存的 最大并行度，如果超过，直接抛出异常，无法恢复
2. 判断新旧 Job 的 MaxParallelism 是否改变：没有改变，则直接跳过。如果改变了，则判断这个改变到底是用户主动设定的，还是 Flink 引擎生成导致的改变。
   - 如果用户主动配置了 MaxParallelism 则任务不能恢复
   - 如果 MaxParallelism 改变是因为框架自动改变的，则将 JobVertex 的 MaxParallelism 设置为 State 的 MaxParallelism

##### **给 subtask 生成 KeyGroupRange**

在 KeyGroup 中讲到了，只要算子的并行度和 MaxParallelism 确定了，那么当前算子的每个 subtask 负责哪些 KeyGroup 也就确定了，即：每个 subtask 对应的 KeyGroupRange 就会确定。

##### **给所有 subtask 重新分配 KeyedState**

这里`KeyGroupsStateHandle`模式和` IncrementalRemoteKeyedStateHandle` 模式的分配过程是不同的。

例如：假设旧任务并发为 2：

- subtask a 负责 KeyGroupRange(0,9)
- subtask b 负责 KeyGroupRange(10,19)

新任务并发为 3：

- subtask A 负责 KeyGroupRange(0,6)
- subtask B 负责 KeyGroupRange(7,13)
- subtask C 负责 KeyGroupRange(14,19)

**KeyGroupsStateHandle 模式的分配结果：通过句柄和存储的offset值可以直接拿数据**

- subtask B 会拿到 subtask a 的 KeyGroupsStateHandle，只需要 KeyGroup 7、8、9 的 offset
- subtask B 还会拿到  subtask b 的 KeyGroupsStateHandle，只需要 KeyGroup 10、11、12、13 的 offset
- subtask A、C 省略

**RocksDB Incremental 模式的分配结果：**

- 对于新的 subtask A 虽然只负责 KeyGroup 0~6 的部分，但必须将 KeyGroup 0~9 的数据全拉取到 TM 本地，基于这些数据建立出一个 RocksDB 实例，读出自己想要的数据。
- 新的 subtask B 负责 KeyGroupRange(7,13)，恢复时数据来源于旧 subtask a 负责 KeyGroupRange(0,9) 和旧 subtask b 负责 KeyGroupRange(10,19)，所以需要将两个 subtask 对应的数据全部拉取到本地，建立两个 RocksDB 实例，读取自己想要的数据。
- subtask A、C 省略

至此，后续再分析 TM 拿到这些 StateHandle 后，如何在 TM 端恢复具体的 KeyedStateHandle 。

##### **给所有 subtask 重新分配 OperatorState**

并行度不变的情况，先找出所有 Union 类型的 State 保存到 unionStates 中，unionStates 为空，表示都是 SPLIT_DISTRIBUTE 模式，直接按照老的 StateHandle 进行分配。Union 类型的 State 需要将所有 subtask 的 State，添加到所有的 map 中（分发到所有机器上）。`OperatorStateHandle`可以读取到对应的状态文件，并拿到算子中定义的所有 State 在状态文件中的 offset 和 Mode。有了这些元数据，TM 端就可以正常恢复了.

至此，后续再分析 TM 拿到这些 StateHandle 后，如何在 TM 端恢复具体的 OperatorStateHandle。

##### **将生成的四种 StateHandle 封装到 ExecutionJobVertex 中**

OperatorState 和 KeyedState 的 StateHandle 分配完成后，存放到了四个 Map 集合中，然后 assignTaskStateToExecutionJobVertices 方法会通过两层循环遍历，将分配好的 StateHandle 封装成 OperatorSubtaskState，最后封装成 JobManagerTaskRestore 对象 set 到 ExecutionJobVertex 中。



