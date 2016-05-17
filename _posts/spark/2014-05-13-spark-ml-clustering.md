---
layout: post
title: "spark ML - CLSUTERING "
description: spark聚类 introduction
categories : spark
---

首先我想吐槽下。 相比mahout，spark ML包组织架构有点不忍直视。

- 没有统一的数据结构。而且借用了两套三方linear algebra库: [jblas](http://mikiobraun.github.io/jblas/) 和 [Breeze](https://github.com/scalanlp/breeze/wiki/Breeze-Linear-Algebra)。 借用jblas是由于其强大的矩阵计算能力。 而Breeze我怀疑是因为spark的一些作者恰好也是Breeze的贡献者。
- 没有抽象的距离类。可能因为目前没有涉及太多向量距离的算法，所以目前只有 *KMEANS* 有一个 *fastSquaredDistance*。 更坑爹的是这个距离算法写在了 *MLUtils* 这个莫名其妙的类里。当然，等数据结构统一后，这个问题自然迎刃而解了。
<!-- more -->
####KMEANS
SPARK 目前的开发分支上也只有一个聚类算法(KMeans)。所以根本谈不上接口方法。很简单的三个类：一个model类，一个并行实现类和一个本地实现类。下面是在spark shell上跑的一个简单的例子。
######数据
spark util包里的 *generateKMeansRDD* 可以生成KMeans的数据集。
<pre><code>scala> val data=KMeansDataGenerator.generateKMeansRDD(sc, 50000, 8, 20, 0.8)
data: org.apache.spark.rdd.RDD[Array[Double]] = MappedRDD[1] at map at KMeansDataGenerator.scala:56</code></pre>
这里的data有20个维度，共有50000个point，并且呈有8个中心点的混合高斯分布。
######训练模型
<pre><code>scala> val model=KMeans.train(data, 8, 50， 10)
model: org.apache.spark.mllib.clustering.KMeansModel = org.apache.spark.mllib.clustering.KMeansModel@309a5663</code></pre>
训练的核心代码是
<pre><code>def run(data: RDD[Vector]): KMeansModel = {
    // Compute squared norms and cache them.
    val norms = data.map(v => breezeNorm(v.toBreeze, 2.0))
    norms.persist()
    ...
    val model = runBreeze(breezeData)
    norms.unpersist()
    model
  }

private def runBreeze(data: RDD[BreezeVectorWithNorm]): KMeansModel = {
	...
	// 选初始中心点
	val centers = if (initializationMode == KMeans.RANDOM) {
	      initRandom(data)
	    } else {
	      initKMeansParallel(data)
	}
	...
	// 循环所有点，划分至最近中心，并重新选中心点
	while (iteration < maxIterations && !activeRuns.isEmpty) {
		...
	}
	...
	new KMeansModel(centers(bestRun).map(c => Vectors.fromBreeze(c.vector)))
}
</code></pre>
*runBreeze* 方法会先初始 *k* 个中心点，然后循环所有point，划分至最近中心，并重新选中心点。 至所有中心点不再变化时， 返回一个带 *k* 个中心点的 *KMeansModel*。
######预测
 <pre><code>class KMeansModel(val clusterCenters: Array[Vector]) extends Serializable {
    ...
    def predict(point: Vector): Int = {
        KMeans.findClosest(clusterCentersWithNorm, new BreezeVectorWithNorm(point))._1
    }
}
</code></pre>
预测代码其实很简单，找出要预测的点和 *KMeansModel* 的 *k* 个中心最近的中心即为所属类别。