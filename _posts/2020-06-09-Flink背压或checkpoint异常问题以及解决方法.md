---
logout: post
title: Flink背压或checkpoint异常问题以及解决方法
tags: [flink,all]
---

## 定位问题

1. flink的checkpoint生成超时，失败：

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/checkpoint%E8%B6%85%E6%97%B6.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/checkpoint超时.png)

2. 查找原因：出现了背压的问题，缓冲区的数据处理不过来，barrier流动慢，导致checkpoint生成时间长，出现了超时的现象。下图可以看到input和output的缓冲区都占满。同时也可以在flink的UI界面看到是否背压。

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/buffer%E7%BC%93%E5%86%B2%E5%8C%BA%E6%83%85%E5%86%B5.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/buffer缓冲区情况.png)

## 背压机制原理

[Flink中的背压机制](https://liurio.github.io/2019/12/20/Flink%E4%B8%AD%E7%9A%84%E8%83%8C%E5%8E%8B%E6%9C%BA%E5%88%B6/)

## checkpoint原理

[Flink状态管理与Checkpoint机制(一)](https://liurio.github.io/2019/12/21/Flink%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%E4%B8%8ECheckpoint%E6%9C%BA%E5%88%B6(%E4%B8%80)/)

## 解决办法

1. 首先说一下flink原来的JobGraph, 如下图,  产生背压的是中间的算子

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/flink_job_graph.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/flink_job_graph.png)

2. 背压是什么？

如果看到任务的背压警告（例如High），这意味着它生成的数据比下游算子可以消耗的速度快。下游工作流程中的记录（例如从源到汇）和背压沿着相反的方向传播到流上方。

**以一个简单的Source -> Sink工作为例。如果您看到警告Source，这意味着Sink消耗数据的速度比Source生成速度慢。Sink正在向上游算子施加压力Source。**

可以得出:  第三个算子的处理数据速度比第二个算子生成数据的速度,  明显的解决方法:  提高第三个算子的并发度,  问题又出现了:  并发度要上调到多少呢?

3. 把第三个算子的并行度设置成100, 与第二个算子的并发度一致: 这样做的好处是, **flink会自动将条件合适的算子链化, 形成算子链**,**链化成算子链可以减少线程与线程间的切换和数据缓冲的开销，并在降低延迟的同时提高整体吞吐量。**

> 满足operator chain的条件：
>
> - 上下游的并行度一致
> - 下游节点的入度为1 （也就是说下游节点没有来自其他节点的输入）
> - 上下游节点都在同一个 slot group 中
> - 下游节点的 chain 策略为 ALWAYS（可以与上下游链接，map、flatmap、filter等默认是ALWAYS）
> - 上游节点的 chain 策略为 ALWAYS 或 HEAD（只能与下游链接，不能与上游链接，Source默认是HEAD）
> - 两个节点间数据分区方式是 forward
> - 用户没有禁用 chain

4. 并发是100的buffer的情况，(背压已经大大缓解)

![https://gitee.com/liurio/image_save/raw/master/flink/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/100%E5%B9%B6%E5%8F%91%E7%9A%84buffer%E6%83%85%E5%86%B5.png](https://gitee.com/liurio/image_save/raw/master/flink/常见问题/100并发的buffer情况.png)