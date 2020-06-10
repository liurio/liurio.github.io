---
logout: post
title: kafka和Flink如何保证Exactly Once的?
tags: [kafka,flink]
---

### Kafka如何保证Exactly Once？

#### 幂等性发送

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/%E6%97%A0%E5%B9%82%E7%AD%89%E5%8F%91%E9%80%81.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/无幂等发送.png)

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/%E5%B9%82%E7%AD%89%E5%8F%91%E9%80%81.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/幂等发送.png)

为实现Producer的幂等性，Kafka引入了Producer ID（即PID）和Sequence Number。对于每个PID，该Producer发送消息的每个<Topic, Partition>都对应一个单调递增的Sequence Number。同样，Broker端也会为每个<PID, Topic, Partition>维护一个序号，并且每Commit一条消息时将其对应序号递增。对于接收的每条消息，如果其序号比Broker维护的序号）大一，则Broker会接受它，否则将其丢弃：

> 如果消息序号比Broker维护的序号差值比一大，说明中间有数据尚未写入，即乱序，此时Broker拒绝该消息，Producer抛出InvalidSequenceNumber

> 如果消息序号小于等于Broker维护的序号，说明该消息已被保存，即为重复消息，Broker直接丢弃该消息，Producer抛出DuplicateSequenceNumber

消息ack确认机制

```json
- ack：producer收到多少broker的答复才算真的发送成功
	> 0表示producer无需等待leader的确认(吞吐最高、数据可靠性最差)
	> 1代表需要leader确认写入它的本地log并立即确认
	> -1/all 代表所有的ISR都完成后确认(吞吐最低、数据可靠性最高)
```

#### 分布式事务

上述幂等设计只能保证单个 Producer 对于同一个 <Topic, Partition> 的 Exactly Once 语义。

Kafka 现在通过新的事务 API 支持跨分区原子写入。这将允许一个生产者发送一批到不同分区的消息，这些消息要么全部对任何一个消费者可见，要么对任何一个消费者都不可见。这个特性也允许在一个事务中处理消费数据和提交消费偏移量，从而实现端到端的精确一次语义。

为了实现这种效果，应用程序必须提供一个稳定的（重启后不变）唯一的 ID，也即Transaction ID ，Transactin ID 与 PID 可能一一对应。区别在于 Transaction ID 由用户提供，将生产者的 **transactional.id** 配置项设置为某个唯一ID。而 PID 是内部的实现对用户透明。

另外，为了保证新的 Producer 启动后，旧的具有相同 Transaction ID 的 Producer 失效，每次 Producer 通过 Transaction ID 拿到 PID 的同时，还会获取一个单调递增的 epoch。由于旧的 Producer 的 epoch 比新 Producer 的 epoch 小，Kafka 可以很容易识别出该 Producer 是老的 Producer 并拒绝其请求。

```java
Producer<String, String> producer = new KafkaProducer<String, String>(props);
// 初始化事务，包括结束该Transaction ID对应的未完成的事务（如果有）
// 保证新的事务在一个正确的状态下启动
producer.initTransactions();
// 开始事务
producer.beginTransaction();
// 消费数据
ConsumerRecords<String, String> records = consumer.poll(100);
try{
    // 发送数据
    producer.send(new ProducerRecord<String, String>("Topic", "Key", "Value"));
    // 发送消费数据的Offset，将上述数据消费与数据发送纳入同一个Transaction内
    producer.sendOffsetsToTransaction(offsets, "group1");
    // 数据发送及Offset发送均成功的情况下，提交事务
    producer.commitTransaction();
} catch (ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e) {
    // 数据发送或者Offset发送出现异常时，终止事务
    producer.abortTransaction();
} finally {
    // 关闭Producer和Consumer
    producer.close();
    consumer.close();
}
```

### Flink 如何保证Exactly Once？

flink通过checkpoint保证了flink应用内部的exactly-once语义，通过TwoPhaseCommitSinkFunction可以保证**端到端**的exactly-once语义。

要求source和sink的外部系统都必须是支持分布式事务的，能够支持回滚和提交。然而一个简单的提交和回滚，对于一个分布式的流式数据处理系统来说是远远不够的。下面我们来看看flink是如何解决这个问题的

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/exactly-once%E9%A2%84%E6%8F%90%E4%BA%A4.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/exactly-once预提交.png)

两部提交协议

- 1、预提交。预提交是所有的算子全部完成checkpoint，并JM会向所有算子发通知说这次checkpoint完成。
- 2、正式提交。flink负责向kafka写入数据的算子也会正式提交之前写操作的数据。在任务运行中的任何阶段失败，都会从上一次的状态恢复，所有没有正式提交的数据也会回滚。

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/exactly-once%E6%AD%A3%E5%BC%8F%E6%8F%90%E4%BA%A4.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/exactly-once正式提交.png)

### 参考

[Flink两阶段提交](https://blog.csdn.net/lisenyeahyeah/article/details/90288231)