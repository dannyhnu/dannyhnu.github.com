---
layout: post
title: "hadoop sort问题"
description: hadoop排序问题的优化
categories : 大数据
---
项目组最近遇到一个hadoop(1.2)问题,因为map中产生的key太多,导致job花费了大量时间在map的shuffle上。准确来说时间应该都花在了IO上面，map产生的数据过多，使得shuffle阶段IO操作过多。 当然因为db设计的原因，map数据过大也是必然的事。  纵观Mapreduce的shuffle过程： sort、 IO 、merge， 似乎也只有sort还有优化的可能。
<!-- more -->
<br />
<br />
这里先跑下题 Mapreduce本是一个聚合的过程, 但是因为后台db采用了星形的架构 使得map将数据发散到一个更广的空间中。 key多的点让人觉得不可理喻。 或许这种设计本身就不适合用Mapreduce来做ETL。
<br />
<br />
言归正传，于是项目组决定对Mapreduce的排序进行优化。 下面是优化的笔记：
<br />
<br />
Mapreduce的shuffle过程包括map阶段和reduce阶段。   在map阶段， 如图， 首先将输出写到缓存当中。 当缓存大小达到“阈值”时（默认的大小是缓存的80%），  变会进行'spill'： 即sort and combiner。  sort完才会把数据写进磁盘。  坑爹的是，  根据Mapreduce的设计(hadoop1.2)， sort是个无法避免的过程。
<br />
![map](/images/hadoop/map_shuffle.jpg)

而在sort的过程中， 由于hadoop往缓存中写入的不是对象， 而是序列化以后的二进制数据。 所以在WritableComparator中， 会首先将这些二进制数据反序列化为对象， 然后再调用这些对象的compareTo方法进行比较：
<pre><code>public int compare(byte[] b1, int s1, int l1, byte[] b2, int s2, int l2) {
  try {
    buffer.reset(b1, s1, l1);                   // parse key1
    key1.readFields(buffer);
  
    buffer.reset(b2, s2, l2);                   // parse key2
    key2.readFields(buffer);
  
  } catch (IOException e) {
    throw new RuntimeException(e);
  }
  return compare(key1, key2);                   // compare them
}
</code></pre>

所以说， 每次compare的过程都会有两个反序列化的过程。 当然还好hadoop留了一条后路， Job对象上有一个方法：
<pre><code>setSortComparatorClass(RawComparator c)
</code></pre>
这样我们就能避免反序列化过程了：
<pre><code>public class SampleComparator implements RawComparator{
  private Logger logger = Logger.getLogger(SampleComparator.class);

  @Override
  public int compare(Object o1, Object o2) {
    logger.error("SampleComparator does not support method compare(Object o1, Object o2)!");
    throw new RuntimeException("unsupported method!");
  }

  @Override
  public int compare(byte[] b1, int s1, int l1, byte[] b2, int s2, int l2) {
    return WritableComparator.compareBytes(b1, s1, l1, b2, s2, l2);
  }
}
</code></pre>

当然， 进行这种修改的时候一定要确保是否序列化对排序结果没有影响。

#####附
顺便记一下最近看到的一个问题：
 为什么hadoop不适合做实时计算？ 个人的感觉主要是IO太多， 再加上启动时间、 网络传输等因素 ，就决定了hadoop在实时这个领域难有作为。 
<br />storm就不一样，数据都是在内存中， task一直在监听数据， 除非kill掉(下次系统看下storm)