---
layout: post
title: "kafka"
categories : queue
---

官方是这样定义 KAFKA 的:
> Kafka is a distributed, partitioned, replicated commit log service. It provides the functionality of a messaging system, but with a unique design.

一个分布式、可分区、基于备份的日志服务，可提供消息队列服务，却和传统的JMS有着不同的设计。这就是 Apache Kafka。
<!-- more -->
####Design
![kafka_design](/images/queue/kafka_design.jpg)

- topic。 message按topic进行分组。在生产者端，producer将message发到指定的topic。而一台kafka server-broker则会将message按topic进行partition，同时每个partition都会保存数个(可配置)replication。这种分区、备份机制都按Master-slave方式工作，其中一台broker会成为leader。
- producer。 producer可以指定发消息到具体的topic和partition。所以说生产Load balancing其实是由producer客户端实现的。
- consumer。 consumer按照group进行分组，对于一个topic里的一条消息，只会被一个consumer group的一个consumer消费。并且同一时刻，一个consumer的消息只会被group里的一个consumer消费，这样保证同一个group里的所有consumer是有序消费一个partition里的所有消息。所以，consumer数量不要大于partition数量。

####Keyword
Kafka被设计成应对网站活动日志流等case。所以有几个关键词：高吞吐、实时。

不同于传统的消息队列服务，消息在kafka中被持久化是一种常态-以log形式保存在磁盘。所以对内存的压力不会随着消息的积压而升高。

所以需要担心的是磁盘IO。但是kafka保存消息log只有append操作，即不会对log有随机读写-没有寻址。官方是这样解释的：
> The key fact about disk performance is that the throughput of hard drives has been diverging from the latency of a disk seek for the last decade. As a result the performance of linear writes on a JBOD configuration with six 7200rpm SATA RAID-5 array is about 600MB/sec but the performance of random writes is only about 100k/sec—a difference of over 6000X. These linear reads and writes are the most predictable of all usage patterns, and are heavily optimized by the operating system. A modern operating system provides read-ahead and write-behind techniques that prefetch data in large block multiples and group smaller logical writes into large physical writes.

可见线性读写其实可以很快，而且这还节省了 JVM 上的对象回收 和 GC 时间。

当然还有网络IO。kafka采用轻量级的TCP协议传输message。而且还会采取 zip message、buffer等措施减轻网络IO压力。
####Different from JMS
- Message持久保存。 不同JMS，即使消息被消费，消息仍然不会被立即删除.日志文件将会根据broker中的配置要求,保留一定的时间之后再删除。
- Message消费顺序由consumer控制。kafka消费消息并不是全局严格有序，事实上，Kafka只能保证一个group下的consumer正常情况下消费一个partition里的消息是有序的。而且consumer可以设置offset按任意顺序消费message。
- consumer主动去pull message。这样consumer可以根据消费能力去消费message。也可以对fetch message作一些优化，如batch fetch。
- 轻量级ack。不同JMS向queue发送ACK，kafka只需要consumer向zookeeper提交消息的offset。这种"宽松"的设计, 确实有丢message的危险。
