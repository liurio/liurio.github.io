---
logout: post
title: flink state经验总结
tags: [flink,state,all]
---

### KeyedState和Operator的区别

| 名称          | 存储              | 实现方式                      | 数据规模 |
| ------------- | ----------------- | ----------------------------- | -------- |
| OperatorState | 只有堆内存        | 手动实现snapshot和restore方法 | 规模较小 |
| KeyedState    | 有堆内存和rocksdb | 由backend实现，对用户透明     | 较大规模 |

### 三种StateBackend的区别

| 类别         | MemoryStateBackend                  | FsStateBackend                 | RocksDBStateBackend                |
| ------------ | ----------------------------------- | ------------------------------ | ---------------------------------- |
| 方式         | chk数据直接返回给master节点，不落盘 | 数据写入文件，将路径传给master | 数据写入文件，将文件路径传给master |
| 存储         | 堆内存                              | 堆内存                         | rocksdb                            |
| 性能         | 最好(但一般不用)                    | 性能好                         | 性能不好                           |
| 缺点         | 数据量小                            | 容易OOM风险                    | 需要读写、序列化、IO等耗时         |
| 是否支持增量 | 不支持                              | 不支持                         | 支持                               |

### KeyedState使用建议

#### 谨慎使用长List

flink会把state中一个list序列化成一个Byte，但当该list过长时，会导致metadata文件过大，从而挤爆master机器。

因此一般建议不要使用较长的list，但是如果一定要存储较长数据时，可以考虑用MapState替换ListState，因此MapState时候存储在rocksdb中。

#### 如何清空当前state？

- state.clear()只能清空当前的key对应的state
- 借助KeyedStateBackend的applyToAllKeys方法 (遍历所有的key)

```java
backend.applyToAllKeys(VoidNamespace.INSTANCE,VoidNamespaceSerializer.INSTANCE,listStateDescriptor,
       new KeyedStateFunction<Integer,ListState<String>>) {
    @Override
    public void process(Integer key, ListState<String> state) throws Exception {
        state.clear();
    }
}
```

#### Checkpoint的间隔时间不要太短

- 一般5min级别足够
- Checkpoint与record处理共抢一把锁，checkpoint的同步阶段会影响record的处理

#### FsStateBackend可以考虑文件压缩

对于刷出去的文件可以考虑使用压缩来减少checkpoint的体积。

```java
ExceutionConfig excutionConfig = new ExecutionConfig();
executionConfig.setUseSnapshotCompression(true);
```

#### RocksdbStateBackend的建议

rocksDB的state存储的格式是：`key=(1,K1,Window(10,20));value=v1`，key一般由keyGroup(第几组)+key+namespace(所在的窗口)组成，key和value都是以Byte的序列化形式存储的。

每个state使用一个Column Family，每个column family使用独占的write buffer(写)，整个DB共享一个block cache(读)

- 不要创建太多的state

> 每个state是一个column family，独占write buffer，过多的state会导致占据过多的write buffer
>
> 从根本上海市RocksDB StateBackend的native内存无法直接管理

#### 设置state TTL

默认情况下，只有在下次读访问时才会触发清理那条过期的数据，但如果那条数据之后不再访问，则也不会清理。

```java
/**
	Full snapshot时候会清理snapshot的内容
*/
StateTtlConfig ttl = StateTtlConfig.newBuilder(Time.days(7)).cleanupFullSnapshot().build();
```

```java
/**
	Heap state backend 的持续清理
*/
StateTtlConfig ttl = StateTtlConfig.newBuilder(Time.days(7)).cleanupIncrementally(10,false).build();
```

```java
/**
	rocksdb state backend 的持续清理
*/
StateTtlConfig ttl = StateTtlConfig.newBuilder(Time.days(7)).cleanupInRocksdbCompactFilter().build();
```

