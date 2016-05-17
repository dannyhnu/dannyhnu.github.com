---
layout: post
title: "spark  - 那些坑"
description: spark watchouts
categories : spark
---
####JobProgressListener
######问题
Spark Application跑一段时间总会被yarn kill掉，报的错是
<pre><code>Container ... is running beyond physical memory limits. Current usage: xxx GB of xxx GB physical memory used; xx GB of xx GB virtual memory used. Killing container.</code></pre>
<!-- more -->
第一反应怀疑哪里有内存泄露，所以dump内存看了下，发现**org.apache.spark.ui.jobs.JobProgressListener**几乎占掉了一半的内存。于是看代码，发现*stageIdToData*和*jobIdToData*两个变量保存了所有的job信息和stage信息。好在有*trimStagesIfNecessary*和*trimJobsIfNecessary*两个方法，当job数或者stage数超过配置的数目，JobProgressListener会每次remove掉10%的job或者stage。
######解决办法
<pre><code>spark.ui.retainedJobs
spark.ui.retainedStages</code></pre>
####Broadcast
######问题
使用broadcast的时候发现task耗费大量时间在*Task Deserialization Time*上，且当广播的变量越大task反序列化的时间也越大。于是看到[这篇文章](http://dongguo.me/blog/2014/12/30/Spark-Usage-Share/)中提到的*正确地使用广播变量(broadcast variables)*时才发现自己在真正产生action的操作前就读取broadcast的变量，导致每次广播都会在driver收集变量然后分发给各个executor。这是问题代码（java）：
<pre><code>Function<Row, Row> mapFunc = new Function<Row, Row>() {
    private Map<String, Double> joinMap = broadcastVal.getValue();
    @Override
    public Row call(Row row) throws Exception {
        joinMap.get....
    }
}</code></pre>
######解决办法
在action方法内部调用*getValue()*
<pre><code>Function<Row, Row> mapFunc = new Function<Row, Row>() {
    @Override
    public Row call(Row row) throws Exception {
        Map<String, Double> joinMap = broadcastVal.getValue();
        joinMap.get....
    }
}</code></pre>