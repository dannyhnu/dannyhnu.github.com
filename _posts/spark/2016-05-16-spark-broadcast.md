---
layout: post
title: "spark core - Broadcast"
description: spark广播变量
categories : realtime
---
最近刚完成一个spark sql项目：大数据实时查询引擎。所有的数据模型整合成大宽表以parquet(列存，高压缩比，完美整合spark sql)格式存在hdfs上。做成大宽表的好处是，一些费时的join操作在预处理阶段就做好了，查询时只会有简单的group by等操作。但是由于业务的特点，我们的历史数据会变(坑爹啊)，而且没有时限(一两年前的说变就变)。好在这部分变化的数据不大，可以经过**spark broadcast**广播到各个executor进行rdd map join。
<!-- more -->
说回这篇文章的主题-spark broadcast。spark目前有两种广播的方法：http和p2p。二者的区别是各个executor在第一次怎么拿到广播变量。一旦executor拿到广播变量后就会立即放到本地storage。
<pre><code>// 把广播变量缓存到BlockManager，这样后续的任务就不需要再去重新fetch了
SparkEnv.get.blockManager.putSingle(blockId, value_, StorageLevel.MEMORY_AND_DISK, tellMaster = false)
</code></pre>
####HttpBroadcast
Http的广播方法比较简单，在driver写一个local file，起一个http server。所以executor在*getValue*的时候其实就是通过url访问driver上的http server拿到广播变量。
<pre><code>//！代码都精简过的
/*
 * driver 写local file
 */
private def write(id: Long, value: Any) {
    // 拿到一个本地文件
    val file = getFile(id) 
    val fileOutputStream = new FileOutputStream(file)
    ...
   // 序列化
    val ser = SparkEnv.get.serializer.newInstance()
    val serOut = ser.serializeStream(out)
    ...
    serOut.writeObject(value)
 }
 /*
 * 起http server
 */
private def createServer(conf: SparkConf) {
  broadcastDir = Utils.createTempDir(Utils.getLocalDir(conf), "broadcast")
  val broadcastPort = conf.getInt("spark.broadcast.port", 0)
  server =
    new HttpServer(conf, broadcastDir, securityManager, broadcastPort, "HTTP broadcast server")
  server.start()
  serverUri = server.uri
}
 /*
 * 读广播变量
 */
private def read[T: ClassTag](id: Long): T = {
  // 组装请求
  val url = serverUri + "/" + BroadcastBlockId(id).name
  uc = new URL(url).openConnection()
  ...
   //请求广播变量
  val inputStream = uc.getInputStream
  val in = new BufferedInputStream(inputStream, bufferSize)
  ...
  // 反序列化
  val ser = SparkEnv.get.serializer.newInstance()
  val serIn = ser.deserializeStream(in)
}
</code></pre>
####TorrentBroadcast
Torrent采用spark storage模块广播数据。首先会按照特定大小（默认4M）对数据进行分块,  然后把各个piece写到storage。第一次读取变量也是从driver或者其他的executor拿到各个piece然后保存到本地。
<pre><code>//！代码都精简过的
/*
 * 数据写入storage
 */
private def writeBlocks(value: T): Int = {
  // driver上保存一份广播数据，防止重复存储
  SparkEnv.get.blockManager.putSingle(broadcastId, value, StorageLevel.MEMORY_AND_DISK,
    tellMaster = false)
   //对数据进行分块
  val blocks =
    TorrentBroadcast.blockifyObject(value, blockSize, SparkEnv.get.serializer, compressionCodec)
    // 各个块写到storage
  blocks.zipWithIndex.foreach { case (block, i) =>
    SparkEnv.get.blockManager.putBytes(
      BroadcastBlockId(id, "piece" + i),
      block,
      StorageLevel.MEMORY_AND_DISK_SER,
      tellMaster = true)
  }
  blocks.length
}
 /*
 * 读取数据
 */
 private def readBlocks(): Array[ByteBuffer] = {
...
  //循环各个数据块
  for (pid <- Random.shuffle(Seq.range(0, numBlocks))) {
    val pieceId = BroadcastBlockId(id, "piece" + pid)
    // 从本地取
    def getLocal: Option[ByteBuffer] = bm.getLocalBytes(pieceId)
    // 本地取不到则从远程取
    def getRemote: Option[ByteBuffer] = bm.getRemoteBytes(pieceId).map { block =>
      // 取到保存到本地
      SparkEnv.get.blockManager.putBytes(
        pieceId,
        block,
        StorageLevel.MEMORY_AND_DISK_SER,
        tellMaster = true)
      block
    }
    val block: ByteBuffer = getLocal.orElse(getRemote).getOrElse(
      throw new SparkException(s"Failed to get $pieceId of $broadcastId"))
    blocks(pid) = block
  }
  blocks
}
</code></pre>
