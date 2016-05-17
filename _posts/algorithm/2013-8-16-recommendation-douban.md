---
layout: post
title: "豆瓣读书推荐"
description: 豆瓣读书的推荐demo
categories : algorithm
---
之前还一直跟同事抱怨说做数据挖掘，没有数据挖个毛线啊。 于是naza很happy的把豆瓣读书的数据都爬下来了 ‘恩 你挖个够吧’。好吧， 一直想看看豆瓣读书的推荐到底是怎么做的。
<!-- more -->
豆瓣读书的推荐主要有两个页面：
<br />
一个是每本书的主页， 比如梁文道的[读者](http://book.douban.com/subject/4031698/)。豆瓣会有一个推荐的section: *喜欢读"读者"的人也喜欢...*。 这句话似乎是说推荐是基于**协同过滤**的。我很奇怪为什么豆瓣会用这样的标题，这里明显不会是基于协同的推荐， 因为我用不同的用户登录， 推荐的结果是一样的。从推荐的结果来也像是**content based**， 更准确点是**tag-based**。 因为推荐的大多还是梁文道的书，还有些书tag是*随笔*、*文化*什么的。
<br />
另一个是[豆瓣猜](http://book.douban.com/recommended)。首先会列出你可能感兴趣的新书。之后是你可能感兴趣的tag和book。这里应该是混合推荐了。
<br />
当然，这都是猜想。试试看吧。

####Warm-up
先解释下数据模型，爬的数据有people，book，tag以及相互的对应关系。构建三个matrix就可以将三者联系起来了。如图:
<br />
![data model](/images/recommendation/douban_model.jpg)
<br />
再说明下， 所有的算法都是based on mahout。

####tag-based
给定一本书， 找到一本tag相似的书。可以把book-tag matrix作input，这样tag就相当于book的dimension, 可以用*PearsonCorrelationSimilarity*， 即皮尔逊系数来衡量book之间的距离。这样找相似的书， 即找最近的邻居：
<pre><code>
UserSimilarity similarity = new PearsonCorrelationSimilarity(model);
NearestNUserNeighborhood neighbors = new NearestNUserNeighborhood(getHowMany(), similarity, model);
</code></pre>
但是推荐的结果不尽人意。想到之前微博推荐的经验， 可以先对book根据book-tag matrix**聚类**，再在每个cluster里找邻居，这样可以消除一些离群点的影响。看结果：
<br />
<br />
![data model](/images/recommendation/rec_douban_tag.jpg)
<br />
<br />
可以看到推荐的结果有[噪音太多](http://book.douban.com/subject/3644791/)、[弱水三千](http://book.douban.com/subject/1831760/)...

####协同过滤
协同过滤， 说白了也是在people-book matrix里找邻居。不同的是user-based collaborative filtering是把book当成people的dimension。而item-based collaborative filtering则反过来。这里仍然用*PearsonCorrelationSimilarity*来衡量相似度：
<pre><code>
UserSimilarity similarity = new PearsonCorrelationSimilarity(model);
NearestNUserNeighborhood neighbors = new NearestNUserNeighborhood(numNeighbor, similarity, model);
Recommender recommender = new GenericUserBasedRecommender(model, neighbors, similarity);
</code></pre>
很遗憾没有爬到自己的账号。随便选个账号来测试，结果：
<br />
<br />
item-based
![data model](/images/recommendation/rec_douban_item.jpg)
<br />
<br />
user-based
![data model](/images/recommendation/rec_douban_user.jpg)
<br />
<br /> 
因为爬到的数据不多， 本来就稀疏的matrix交集更少。推荐出来的结果都是些可能看过一两本相同书的people给出的推荐。这也反映出协同过滤的cold start问题。不知道豆瓣是怎样解决的。
<br />
<br /> 
同上，我也试着对people根据people-tag matrix先聚类再做item-based collaborative filtering。推荐出来的比较像tag-based的结果。


####new book recommendation
豆瓣还有个推荐新书的功能。根据爬到的数据，我能想到的方法就是计算people的历史tag跟新书tag之间distance，选距离最近的people。但是这时的距离应该是什么呢？ EuclideanDistance？试了下， 结果比较无语，因为推荐出来的都是些过分活跃的用户-那些所有tag都有比较大的weight的用户。
<br /> 
<br /> 
其实这也可以理解， content based必然会有热门问题，比如豆瓣的tag-based推荐，像*文学*、*小说*这样的tag就比较泛滥。 所以很多文学类小说的推荐常常会被几本热门的名著占据。 按理说豆瓣应该会对tag清理过，不明白为什么还是会有这样的问题。
<br />
<br /> 
显然要换个距离， 欧式距离会计算所有tag间的距离，是否可以让距离只考虑新书的tag在people上的weight。想到了对people向量做横向归一处理，这样就消除了热门tag对people的影响，也就能找到新书的tag占最大weigh的people。 看结果:
<br />
![data model](/images/recommendation/rec_douban_new.jpg)
<br />
<br />
这样就为[哈利·波特与魔法石](http://book.douban.com/subject/1041007/)找到了用户[tuzicxy](http://book.douban.com/people/tuzicxy/collect)、[70621573](http://book.douban.com/people/70621573/collect)...

