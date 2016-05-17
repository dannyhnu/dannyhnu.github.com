---
layout: post
title: "mysql session"
description: 支付宝楼方鑫的mysql分享会
categories: [db]
---
先介绍下session的主讲。[楼方鑫](http://www.anysql.net/), 大牛。

好吧， 其实没怎么听懂啦，就是感觉很牛逼的样子，所以这里记一下大概的意思咯。而且刚从嵊泗看(dou)完(di)海(zhu)回来，身心俱疲，很有可能会胡说八道。
<!-- more -->
印象比较深的是关于限流、热点隔离、事务优化的三个topic。

####限流
限流，即控制应用对服务器的资源占用，防止应用拖垮服务器。 而限流策略一般是从三个方向去做的。

- InnoDB的 Thread Concurrency。InnoDB的工作线程控制。
- server的thread pool。thread pool有两个缺点。 一是大查询会影响所有其他请求。 二是DML操作会影响简单的查询。 所以大牛的解决办法是队列限流。根据请求类型(CRUD)把请求派发到不同的thread queue。这样就解决的thread pool的问题。
- app层限流。应用自己控制，缺点太多， 不过有控制也是好事。

####热点隔离
和上面thread pool的问题有点类似，不同应用对server的资源利用不同，这样应用之间很有可能会相互影响。解决办法也差不多，不同应用进不同的queue。

####事务优化
比较赞同大牛的一个观点就是，像电商、金融这种多交易记录的行业，一个db的强事务性是绝对必须的。事务的基本流程是

1. client创建事务
2. client提交command
3. server执行command
4. client commit

其实mysql执行一条command的时间大概在微妙级别，但是从client到server端的带宽时间却往往高出一个数量级。所以大牛的优化方法是在第三部server执行完所有操作后自行判断事务是否成功，如果成功，则server自行commit，这样就节省了一半的带宽来回。大牛自己举得例子是，这样优化可以把mysql的单位时间处理量从450万提高到5000万(忘了一些条件，只记得这两个数字，总之效果很牛逼就对了)。
