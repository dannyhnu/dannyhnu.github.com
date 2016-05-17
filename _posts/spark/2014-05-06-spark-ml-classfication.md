---
layout: post
title: "spark ML源码 - CLASSIFICATION"
description: spark分类 源码分析
categories : spark
---
###SUMMARY
Spark classification在当前的release版本(0.9)上只实现了Naive Bayes、LogisticRegression、SVM。而在[开发分支](https://github.com/apache/spark/tree/master/mllib/src/main/scala/org/apache/spark/mllib/classification)上也没有看到有其他算法的计划。这篇文章的代码都是截取自最新的开发分支(branch 1.0)。
<!-- more -->
继承scala简洁的风格，废话不多说，直接放码。 spark分类模型接口类：
<pre><code>trait ClassificationModel extends Serializable {
    def predict(testData: RDD[Array[Double]]): RDD[Double]
    def predict(testData: Array[Double]): Double
}</code></pre>

###Naive Bayes
先回顾下贝叶斯分类，还是用文本分类来举例吧。

由贝叶斯公式，文章*a*被分到类*Ci*中的概率是
<pre><code>p(Ci|a)=p(a|Ci)*p(Ci)/p(a) ~ p(w1,w2...wn|Ci)*p(Ci)</code></pre>
朴素贝叶斯认为
<pre><code>p(w1,w2...wn|Ci) ~ p(w1|Ci) * p(w2|Ci) ...</code></pre>
所以求 *p(Ci|a)* 即求 *p(wi|Ci)* 和 *p(Ci)* 了。而这里的 *p(wi|Ci)* 为单词 *wi* 在类别 *Ci* 的里概率，即类别 *Ci* 里 *wi* 的出现次数除以 *Ci* 的单词总数。而 *p(Ci)* 即为类别 *Ci* 的概率，即类别 *Ci* 的样本数除以总样本数。

下面是 *spark bayes* 的训练代码
<pre><code> def run(data: RDD[LabeledPoint]) = {
    // Aggregates term frequencies per label.
    // TODO: Calling combineByKey and collect creates two stages, we can implement something
    // TODO: similar to reduceByKeyLocally to save one stage.
    val aggregated = data.map(p => (p.label, p.features)).combineByKey[(Long, BDV[Double])](
      createCombiner = (v: Vector) => (1L, v.toBreeze.toDenseVector),
      mergeValue = (c: (Long, BDV[Double]), v: Vector) => (c._1 + 1L, c._2 += v.toBreeze),
      mergeCombiners = (c1: (Long, BDV[Double]), c2: (Long, BDV[Double])) =>
        (c1._1 + c2._1, c1._2 += c2._2)
    ).collect()
    val numLabels = aggregated.length
    var numDocuments = 0L
    aggregated.foreach { case (_, (n, _)) =>
      numDocuments += n
    }
    val numFeatures = aggregated.head match { case (_, (_, v)) => v.size }
    val labels = new Array[Double](numLabels)
    val pi = new Array[Double](numLabels)
    val theta = Array.fill(numLabels)(new Array[Double](numFeatures))
    val piLogDenom = math.log(numDocuments + numLabels * lambda)
    var i = 0
    aggregated.foreach { case (label, (n, sumTermFreqs)) =>
      labels(i) = label
      val thetaLogDenom = math.log(brzSum(sumTermFreqs) + numFeatures * lambda)
      pi(i) = math.log(n + lambda) - piLogDenom
      var j = 0
      while (j < numFeatures) {
        theta(i)(j) = math.log(sumTermFreqs(j) + lambda) - thetaLogDenom
        j += 1
      }
      i += 1
    }

    new NaiveBayesModel(labels, pi, theta)
  }
}
</code></pre>
没错，就这么多代码。最后返回的是一个带有 *labels*， *pi*， *theta* 三个变量的模型。 先解释几个关键的变量：

- RDD[LabeledPoint]。*RDD[LabeledPoint]* 是训练数据的格式， *LabeledPoint* 是一个形如 *(label, array of features)* 格式的数据。所以输入应该是类似
<pre><code>C1 w11,w12,w13...
C2 w21,w22,w23...</code></pre>
的带lable的矩阵。

- aggregated。*aggregated* 是形如 *label -> (count, featuresSum)* 格式的数据。它是将所有训练样本分类别进行聚合。其中lable是类别标签，count是该类别的训练样本数 featuresSum是该类别所有dimension相加的和。

- numLabels,numDocuments，numFeatures。numLabels是aggregated的size，即类别的数量。numDocuments是count的和，即所有训练样本的总数。numFeatures是dimensions的长度，即样本空间有多少维。

-  labels。 labels数组存储的是所有的类别。
-  pi。 pi也是长度为numLabels的数组。存储的是<pre><code>pi(i) = math.log(n + lambda) - piLogDenom</code></pre>而<pre><code>val piLogDenom = math.log(numDocuments + numLabels * lambda)</code></pre>所以<pre><code>pi(i) = math.log((n + lambda) / (numDocuments + numLabels * lambda))</code></pre>n是类别i的样本数，numDocuments是总样本数。也就是说 *pi(i)* 就是类别 i 的概率 *p(Ci)*。 分子加lambda和分母加numLabels * lambda是为了防止0概率事件，即拉普拉斯平滑。
-  theta。theta是 n X numFeatures 的矩阵。<pre><code>theta(i)(j) = math.log(sumTermFreqs(j) + lambda) - thetaLogDenom
	            = math.log(sumTermFreqs(j) + lambda) - math.log(brzSum(sumTermFreqs) + numFeatures * lambda)
		    = math.log((sumTermFreqs(j) + lambda) / (brzSum(sumTermFreqs) + numFeatures * lambda))
</code></pre>sumTermFreqs(j)是词 *wj* 在类别 *i* 的频率，sumTermFreqs是类别 *i* 的单词总数。所以 *theta(i)(j)* 存储的是单词 j 在类别 *i* 的概率 *p(wj|Ci)*。分子分母加 *lambda* 同样是为了防止0概率事件的发生。

而 *NaiveBayesModel* 的预测代码其实就一行
<pre><code>override def predict(testData: Vector): Double = {
    labels(brzArgmax(brzPi + brzTheta * testData.toBreeze))
}</code></pre>
预测样本和矩阵 *brzTheta* 相乘得到的就是样本在各个类别里的<pre><code>p(w1|Ci) * p(w2|Ci) * p(w3|Ci)..</code></pre>最后加上 *brzPi* 即为乘以 *P(Ci)*(因为所有的概率都用math.log处理过)。最后 *brzArgmax* 即返回可能性最大的类别。

###Generalized Linear Algorithm
LogisticRegression、SVM都是继承于广义线性模型(GeneralizedLinearModel)。二者都只能处理二分类问题。

稍微解释下*广义线性模型*<pre><code>f(Y)= β0 + β1X1 + β2X2 + β3X3 + ... + intercept</code></pre>因变量 *Y* 的分布可能多种多样，但是通过连接函数 *f* 的转换就可以用 *Xi* 线性表出。常见的有：
<table style="margin: 10px; border:1px solid black">
    <tr style="border:1px solid black; background-color:grey">
        <td>Y的分布</td><td>连接函数</td>
    </tr>
	<tr>
        <td>正态分布(normal)</td><td>Identity:  f(Y)=Y</td>
    </tr>
	<tr>
        <td>二项分布(binomial)</td><td>Logit:  f(Y)=Logit(Y)</td>
    </tr>
	<tr>
        <td>Poisson分布</td><td>Log:  f(Y)=log(Y)</td>
    </tr>
	<tr>
        <td>γ 分布(gamma)</td><td>inverse:  f(Y)=1/(Y^-1)</td>
    </tr>
	<tr>
        <td>负二项分布(negative binomial)</td><td>Log:  f(Y)=log(Y)</td>
    </tr>
</table>

回到spark， 下面是模型接口GeneralizedLinearAlgorithm的核心代码
<pre><code>abstract class GeneralizedLinearAlgorithm[M <: GeneralizedLinearModel]
  extends Logging with Serializable {
  def optimizer: Optimizer
  protected def createModel(weights: Vector, intercept: Double): M
  def run(input: RDD[LabeledPoint], initialWeights: Vector): M = {
  val weightsWithIntercept = optimizer.optimize(data, initialWeightsWithIntercept)
  val intercept = if (addIntercept) weightsWithIntercept(0) else 0.0
  val weights =
      if (addIntercept) {
        Vectors.dense(weightsWithIntercept.toArray.slice(1, weightsWithIntercept.size))
      } else {
        weightsWithIntercept
      }
    createModel(weights, intercept)
</code></pre>
核心方法 *run* 会通过子类的 *optimizer* 训练出 *weights* 和 *intercept*。最后返回一个带 *weights* 和 *intercept* 的model。解释下这几个变量：

- optimizer 训练数据的优化方法， 如最小二乘， 梯度下降
- weights 权重矩阵， 即方程里的 *βi*
- intercept 残差

而GeneralizedLinearAlgorithm的子类， 即各种线性算法的区别就是优化方法和连接函数的不同。
#####Logistic Regression
逻辑回归用随机梯度下降训练样本。连接函数代码<pre><code>override protected def predictPoint(dataMatrix: Vector, weightMatrix: Vector,
      intercept: Double) = {
    val margin = weightMatrix.toBreeze.dot(dataMatrix.toBreeze) + intercept
    val score = 1.0/ (1.0 + math.exp(-margin)) // 这就是logic函数
    threshold match {
      case Some(t) => if (score < t) 0.0 else 1.0
      case None => score
    }
  }</code></pre>
#####SVM
SVM同样用随机梯度下降训练样本。连接函数代码<pre><code>override protected def predictPoint(
      dataMatrix: Vector,
      weightMatrix: Vector,
      intercept: Double) = {
    val margin = weightMatrix.toBreeze.dot(dataMatrix.toBreeze) + intercept
    threshold match {
      case Some(t) => if (margin < 0) 0.0 else 1.0
      case None => margin
    }
  }</code></pre>