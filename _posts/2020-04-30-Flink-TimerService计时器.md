---
logout: post
title: Flink TimerService计时器
tags: [flink,all]
---

Flink ProcessFunction是一个low-level的stream处理操作，它相当于可以访问keyed state及timer的FlatMapFunction，当要使用keyed state或者timer的时候，可以使用ProcessFunction。ProcessFunction继承了AbstractRichFunction(`可以通过RuntimeContext获取keyed state`)，它定义了抽象方法processElement以及抽象类Context、OnTimerContext。ProcessFunction定义了onTimer方法，可以响应TimerService触发定时器。

可以处理一些需要延迟处理处理，或者隔断时间再处理的需求。主要的方法就是把消息存储在state中，当定时器触发的时候从state中获取消息再进行处理。

```java
	public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment environment=StreamExecutionEnvironment.getExecutionEnvironment();

        environment.enableCheckpointing(5000);
        environment.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);
        environment.setRestartStrategy(RestartStrategies.noRestart());
        environment.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");
        properties.setProperty("zookeeper.connect", "localhost:2181");
        properties.setProperty("group.id", "test");

        FlinkKafkaConsumer011<String> myConsumer = new FlinkKafkaConsumer011<String>("test", new SimpleStringSchema(),properties);
        environment.addSource(myConsumer).keyBy(new KeySelector<String, String>() {
            @Override
            public String getKey(String s) throws Exception {
                return s;
            }
        }).process(new CustomProcessFunction()).print();

        System.out.println("begin to consuming kafka...");
        environment.execute("WordCount from Kafka data");
    }

	private static class CustomProcessFunction extends ProcessFunction<String,String> {
        private ValueState<Long> state;

        @Override
        public void open(Configuration conf){
            state = getRuntimeContext().getState(new ValueStateDescriptor<Long>("myState",Long.class));
        }

        @Override
        public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws IOException {
            Long res = state.value();
            if (timestamp >= res + 5000) {
                System.out.println("定时器触发的时间"+ MethodUtil.stampToDate(String.valueOf(timestamp)) + "，消息中的时间："+MethodUtil.stampToDate(String.valueOf(res)));
                System.out.println("时间异常的消息："+res.toString());
            }
        }

        @Override
        public void processElement(String s, Context context, Collector<String> collector) throws Exception {
            state.update(Long.valueOf(s));
            context.timerService().registerProcessingTimeTimer(Long.valueOf(s)+50000);
            System.out.println("定时器启动的时间："+s.toString());
        }
    }
```

ProcessFunction主要重写了open、onTimer、processElement方法。

- open

> 可以用getRuntimeContext()获取到state中的数据

- onTimer

> 定时器，当定时器触发的时候可以自定义一些规则，可以get到state值

- processElement

> 这里更新state值，并且定义一个定时器TimerService，该类有两个方法：
>
> 1. registerProcessingTimeTimer(long timestamp)
> 2. registerEventTimeTimer(long timestamp)

