---
logout: post
title: Flink Broadcast State和Spark Streaming Broadcast
tags: [flink,all]
---

### 背景

有这样一个需求：flink或者spark任务需要访问数据库，或者用到表schema信息。但此时数据库中的字段有添加或者修改时(schama发生改变的时候)，这时候任务就会失败。最直接的做法就是重启flink或spark任务，但该做法会对业务数据造成一定的影响。

方案：**将改动的schema信息放入redis中，再通过broadcast广播的方式传送给数据流。**

### flink broadcast state

Broadcast State是Flink支持的一种Operator State。使用Broadcast State，可以在Flink程序的一个Stream中输入数据记录，然后将这些数据记录广播（Broadcast）到下游的每个Task中，使得这些数据记录能够为所有的Task所共享，比如一些用于配置的数据记录。这样，每个Task在处理其所对应的Stream中记录的时候，读取这些配置，来满足实际数据处理需要。

步骤：

1. 定义一个MapStateDescriptor来描述要广播数据的地址
2. 添加数据源，并注册为广播流
3. 连接广播流和处理数据的流
4. 实现连接的处理方法

```java
private static final MapStateDescriptor<String, TableSchema> mapStateDescriptor =
            new MapStateDescriptor<>(
                    "broadcast",
                    BasicTypeInfo.STRING_TYPE_INFO,
                    TypeInformation.of(new TypeHint<TableSchema>() {
                    }));
public static void main(String[] args){
    /**
    	首先从redis中得到最新的table schema信息
    */
    TableSchema meta = getLatestMeta(..);
    
    /**
    	数据流
    	生成KeyedStream的流
    */
    KeyedStream<String,String> keyedStream = env.addSource(..)
        .map(..)
        .returns(String.class)
        .keyby(new KeySelector<String, String>() {
            @Override
            public String getKey(String value) throws Exception {
                return value;
            }
        });
    
    /**
    	广播流
    	该流主要存储了table schema的信息，数据量小，广播到各个Task
    */
    BroadcastStream<TableSchema> broadcastStream = env.addSource(..)
        .map(..)
        .returns(TableSchema.class)
        .broadcast(mapStateDescriptor);
        
    keyedStream.connect(broadcastStream)
        .process(new KeyedBroadcastProcessFunction<String, String, TableSchema, String>(){
            @Override
        public void processBroadcastElement(TableSchema value, Context ctx, Collector<String> out){
            //获取旧的值
            TableSchema old = ctx.getBroadcastState(mapStateDescriptor).get("id");
            System.out.println("old value:"+old+",new value:"+value);
            //更新新的值
            state.put("id", value);
        }
            @Override
        public void processElement(String value, ReadOnlyContext ctx,
                                   Collector<String> out) {
            //获取上述更新后的最新值
            TableSchema meta = ctx.getBroadcastState(mapStateDescriptor).get("id");
            /**
            	具体处理逻辑...
            */
        }
    });
}
```

### spark streaming broadcast

我们知道spark的广播变量允许换成一个只读的变量在每台机器上面，而不是每个任务保存一份。常见于spark在一些全局统计的场景中应用。通过广播变量，能够以一种更有效率的方式将一个大数据量输入集合的副本分配给每个节点。Spark也尝试着利用有效的广播算法去分配广播变量，以减少通信的成本 

一个广播变量可以通过调用sparkContext.broadcast(v)方法从一个初始变量v中创建。广播变量是v的一个包装变量，它的值可以通过value方法访问，例如：

```java
public static void main(String[] args){
    JavaStreamingContext sc = new JavaStreamingContext(conf);
    JavaPairInputDStream<String,String> kafka = KafkaUtils.createStream(...);
    kafka.repartition(30)
        .transform(...)
        .foreachRDD(new VoidFunction<JavaRDD<String>>() {
            public void call(JavaRDD<String> rdd) {
                // 广播变量的注册一定要放在这里，否则不会广播到各个节点的task，这种方式可以做到自动更新
                final Broadcast<String> cast = JavaSparkContext
                    	.fromSparkContext(rdd.context())
                        .broadcast("broadcast value");
                rdd.foreach(new VoidFunction<String>() {
                    public void call(String v) throws Exception {
                        System.out.println(cast.value());
                    }
                });
            }
        });
	sc.start();
}
```