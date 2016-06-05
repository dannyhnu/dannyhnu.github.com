---
layout: post
title: "spark  - 那些坑"
description: spark watchouts
categories : spark
---
###JobProgressListener问题
#####问题
Spark Application跑一段时间总会被yarn kill掉，报的错是
<pre><code>Container ... is running beyond physical memory limits. Current usage: xxx GB of xxx GB physical memory used; xx GB of xx GB virtual memory used. Killing container.</code></pre>
<!-- more -->
第一反应怀疑哪里有内存泄露，所以dump内存看了下，发现**org.apache.spark.ui.jobs.JobProgressListener**几乎占掉了一半的内存。于是看代码，发现*stageIdToData*和*jobIdToData*两个变量保存了所有的job信息和stage信息。好在有*trimStagesIfNecessary*和*trimJobsIfNecessary*两个方法，当job数或者stage数超过配置的数目，JobProgressListener会每次remove掉10%的job或者stage。
#####解决办法
<pre><code>spark.ui.retainedJobs
spark.ui.retainedStages</code></pre>
###Broadcast问题
#####问题
使用broadcast的时候发现task耗费大量时间在*Task Deserialization Time*上，且当广播的变量越大task反序列化的时间也越大。于是看到[这篇文章](http://dongguo.me/blog/2014/12/30/Spark-Usage-Share/)中提到的*正确地使用广播变量(broadcast variables)*时才发现自己在真正产生action的操作前就读取broadcast的变量，导致每次广播都会在driver收集变量然后分发给各个executor。这是问题代码（java）：
<pre><code>Function<Row, Row> mapFunc = new Function<Row, Row>() {
    private Map<String, Double> joinMap = broadcastVal.getValue();
    @Override
    public Row call(Row row) throws Exception {
        joinMap.get....
    }
}</code></pre>
#####解决办法
在action方法内部调用*getValue()*
<pre><code>Function<Row, Row> mapFunc = new Function<Row, Row>() {
    @Override
    public Row call(Row row) throws Exception {
        Map<String, Double> joinMap = broadcastVal.getValue();
        joinMap.get....
    }
}</code></pre>
###spark sql聚合函数问题
#####问题
先说下背景。为了最大限度的减少每次查询cover的数据，我们的数据模型不仅按月分表，而且单月的数据都会做行转列的操作。简单点说就是把指标项转成31列，随着时间的发展去填充列而不是增加数据行数。这样做的好处就是每个月第一天和到最后一天的数据量都会差不多，而这样做的代价就是更加负责的sql和每次查询都要做列转行的反转。
举个栗子，查询20160301到20160325分年龄的收入，一般的sql是
<pre><code>SELECT age, SUM(income) FROM  table_201603 WHERE date BETWEEN 20160301 AND 20160325 GROUP BY age</code></pre>
但是我们得这样处理
<pre><code>SELECT age, SUM(income_1)  AS inome_20160301, SUM(income_2)  AS inome_20160302... , SUM(income_25)  AS inome_20160325 FROM  table_201603 GROUP BY age</code></pre>
然后再flatten查询结果。
因为一般的查询结果集比较小，所以flatten结果集的代价也比较小。但是这样做在spark sql上碰到一个坑，当一条sql中**SUM**的数量超过24个时候会严重影响sql的执行速度(5-10倍的性能差)。
//TODO 原因待查
#####解决办法
目前的spark 1.6 还是存在这个问题，所以只好做和谐处理。当查询中**SUM**超过24个，将sql拆成多条sql处理。
