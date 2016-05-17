---
layout: post
title: "spark introduction"
description: spark简介
categories : spark
---
###WHY SPARK
####spark vs hadoop MR
记得之前有被问到：Map Reduce为什么这么慢？之前的理解是shuffle阶段的IO和MR的执行过程所致。最近看到一篇文章([为什么之前的MapReduce系统比较慢](http://jerryshao.me/architecture/2013/04/15/%E4%BC%A0%E7%BB%9F%E7%9A%84MapReduce%E6%A1%86%E6%9E%B6%E6%85%A2%E5%9C%A8%E5%93%AA%E9%87%8C/))。
<!-- more -->
既然hadoop MR慢，那肯定会有大神站出来。dadada! SPARK就这样出来了。同样是Map Reduce流程的实现，那SPARK快在哪？最重要的区别是，不同于hadoop MR每个job的输出结果必须保存在磁盘中，SPARK可以保存在内存中(SPARK不用把中间结果保存在HDFS上，Spark有自己的存储系统)。对于迭代式的job chain，SPARK就显得快很多了。

####spark vs storm
刚开始以为storm focus on流式计算， 而spark则擅长于迭代式计算，二者专注于不同的领域，所以没有什么可比性。但是spark推出了实时module： spark streaming，这样storm在实时领域就不那么孤单了。虽然现在spark streaming应用不多，但是潜力还是有的

- hadoop生态系统为靠山。现在大多数公司的大数据都是基于hadoop生态。而spark本身就是hadoop生态的一员，spark streaming和hadoop生态系统数据库的无缝结合会使得越来越多的公司选择spark streaming。
- 简单。个人认为读懂scala的代码比clojure容易多了。（虽然jstorm挤进了apache）

####Spark sql vs hive
两个不同的工具做了同样一件事，不同的是Spark sql基于spark，hive基于hadoop MR。当然，Spark sql 更快！

###Spark Architecture 

几个比较重要的module

- core，spark核心
- graphx，图算法模块(bagel的官方建议替代包)
- mllib，机器学习模块
- streaming，实时计算模块
- yarn，和yarn整合模块，实现了yarn的ApplicationMaster和clinet


####spark core

Spark core有以下几个主要的子模块

- deploy， SPARK application和资源调度系统
- scheduler， app内tasks的调度系统
- rdd， 分布式数据模型
- storage， Spark自己的存储系统

由于调度和存储都是分布式的，所以deploy和storage模块都采用master-slave模式。

####spark mllib

实现的算法虽然不多，也没看懂多少。但是，看完后只有一个感觉：真他妈简洁！

####spark shuffle
关于spark shuffle，[这篇文章](http://jerryshao.me/architecture/2014/01/04/spark-shuffle-detail-investigation/)已经讲的很详细了。 记得之前碰到过一个问题，因为嫌MR shuffle慢，想要把shuffle阶段的sort disable掉。后来发现这是MR的必须阶段，所以觉得有点坑。reducer拿完所有mapper的数据后，又重新对所有数据做了次外排(merge sort)。其实大多数reducer并不需要key有序，只需要相同key的value相邻就可以了。当时就想到一个hashmap就可以解决，没想到spark居然真是这么做的。虽然遇到了内存问题，个人觉得还是能适应大多数场景(哪来那么多数据总是撑爆内存)。

####spark streaming
// TODO

####spark RDD
// TODO



###参考资料
[Spark：一个高效的分布式计算系统](http://tech.uc.cn/?p=2116)

[Spark Overview](http://jerryshao.me/architecture/2013/03/29/spark-overview/)
