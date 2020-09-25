---
title: GNN需要解决的关键问题
date: 2020-04-16 16:24:19
tags: 科研
categories: 科研
---

# “图神经网络在线研讨会2020” 笔记

## 1. 从同质到异质

目前很多图神经网络主要是基于同质信息网络，同质信息网络只有一种类型的节点和边；

在实际应用中，会存在大量由不同节点和边构成的交互系统，例如文献数据、电影数据、以及社交网络知识图谱等，在这些网络中，不同类型的对象相互交互。不同类型的对象性质不同，交互关系的特性不同会导致很大差异的分析，所以在异质网络中，需要考虑不同类型的对象交互关系对结果的影响。

在异质信息网络中，网络模式是对一个网络的元级描述，刻画了网络中包含了不同类型的对象和不同类型的关系。例如在图 1 的网络实例中，描述了作者撰写论文，论文发表在会议上；这个网络实例就包含三类对象：作者、论文和会议，以及他们之间的相互交互关系。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icdwveTm3DV8picININawtSibDUFicuoyNh6DnxhxTVNnGFlWIwLyrpOoHg/640?wx_fmt=png)

图 1：异质信息网络实例

</center>

### 元路径

元路径是异质信息网络中另外一个很重要的概念。简而言之，元路径就是连接两个对象的一个关系序列。

如图 2 所示，连接两个 author 可以有不同的元路径。例如：author->paper->author，描述的是两个作者之间的合作关系；还可以有 author->paper->venue->paper->author，这条元路径描述的是两个作者参加同一个会议这么一个关系。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icIYG6IA8nEH1G7xTLGdJmRRzPSqtM6iclKRWWdnVnwT8ddHaDlE8fLrA/640?wx_fmt=png)

图 2：异质信息网络中的元路径

</center>

也提出了很多其他一些概念，例如元图，元结构，以及有约束的元路径等一些概念，它可以更细致的描述网络里面的属性信息。

石川指出，异质信息网络表示学习目前存在一些挑战，例如如何解决异质性、如何融合信息以及捕捉丰富的语义信息等，主要的解决方案有**经典浅层模型**和**深度模型**两个方面。

## 2. 浅层模型

在同质网络中，有一些很经典的浅层模型，例如 DeepWalk、Line 等一系列方法，这些方法的核心思想是基于随机游走产生一个节点序列，然后类比于自然语言处理单词序列的方法，通过 skip-gram 的方法来学习网络表示。

在异质网络中也是采用类似的思路，为了高效的随机游走，一般是采用元路径的随机游走方式，元路径在游走的过程中可以把节点类型信息和边类型信息固定下来。图 3 为基于元路径随机游走的范式，给以一个元路径，如果是按照给定元路径游走的话，转移概率就是指定类型节点的邻居数目分之一，不按照元路径游走的话，转移概率就是 0。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icmFl5gia2CcLtJwvxuF6K6vyXfAG45nZPKljCKX21ZrfZCiaibqq40eeMA/640?wx_fmt=png)

图 3：基于元路径的随机游走  

</center>

针对元路径的随机游走，然后采用 skim-gram 来进行目标优化，metapath2vec 和 metapath2vec++ 有一个区别如图 4 所示。在 soft-max 操作中，metapath2vec++ 在分母中是按照下一个节点类型中的所有节点求和的，而 metapath2vec 是不考虑节点类型，直接对所有节点求和。metapath2vec++ 的优点在于考虑下一个节点的节点类型可以使游走概率的值大一些，在很多情况下效果会好一些。

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icmYnOlGUjAFnQicdYvyljLbQuqfHibR8onSPvVH2en6RQc5xib2bA9Ee3g/640?wx_fmt=png)

图 4：计算概率中 metapath2vector（上）和 metapath2vec++（下）

Metapath2vec 是基于元路径随机游走和 skim-gram 的方式解决异质性，也基本上奠基了这个方向研究的基本思路。

HERec[2] 是另外一种处理异质信息网络的浅层模型，它的基本思路是通过一些对称的元路径将异质图变成同质图，然后在同质图中用 DeepWalk、Line 等方法学习到网络表示。另外一种基于游走的方法是 HIN2Vec[3]，首先 HIN2Vec 在异质网络中游走，抽取点边序列，即节点 X、Y 和它们之间对应的关系 R，在游走的过程中点边序列抽取出来就可以构成序列样本，然后就可以通过判断节点 X、Y 是否具有关系 R 把原来的问题变成分类问题，将分类问题作为优化目标学习网络表示。

Metapath2vec、HERec 和 HIN2Vec 是异质信息网络表示的三个早期的工作，给后来的工作奠定了基础。最近几年也有一些比较优秀的工作。MCRec[4] 通过刻画 user 和 item 的丰富的交互关系来学习节点表示。为了找到有代表性的负样本，HeGAN[5] 根据关系类型用 GAN 生成好的负样本。RHINE[6] 为了区分异质信息网络中不同类型的关系，借鉴知识表示的思想学习网络表示。

## 3. 深层模型

深层模型就是用神经网络进行深度建模。在推荐领域，一般主要分析 user 和 item 之间的交互矩阵来得到 user 和 item 之间的隐含特征，但是考虑到异质信息网络实际上包含了不同方面的交互信息，NeuACF[7] 尝试将不同方面的信息融合。如图 5 所示，先通过一些不同方面的元路径抽取不同维度的信息。例如，通过 UIU 和 IUI 抽取用户购买记录方面的特征，UIBIU 和 IBI 元路径可以抽取出品牌方面的信息。然后构造出 aspect-level 的相似性矩阵，然后用 MLP 学习 aspect-level 的潜在因子，最后用 attention 机制将 aspect-level 的潜在因子融合，得到损失函数。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icas3JGGoiaxiaT9CeU9hB1Nxns09uwIp5wgia0XM9ew8tibZ5HYDSyocHqQ/640?wx_fmt=png)

图 5：NeuACF 的框架示意图

</center>

### Attention 机制

Attention 机制在图神经网络中有着重要的应用，但是在异质信息网络中应用 attention 需要两方面的考虑：一个是节点级别的 attention，考虑节点与邻居之间的 attention；另外一个是语义级别的 attention，即在元路径上将节点信息通过 attention 聚合。基于此，HAN 模型将 attention 机制应用到异质信息网络中。如图 6 所示，HAN[8] 首先把节点映射到相同的特征空间，然后用一个 node 级别的 attention 机制，把这些邻居节点聚合起来，再用 semantic attention 机制将元路径信息融合。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9ic8Sr3NxfcPWEmaxow1K7MFE6HVy4MEa4PmSzKPyAL00SGx8xVCrTtbA/640?wx_fmt=png)

图 6：HAN 框架示意图

</center>

在异质图中，不同类型的节点有不同类型的属性特征。HetGNN 将节点的属性信息融合到异质信息网络中。如图 7 所示，HetGNN[9] 先考虑某一类型节点的属性信息，通过神经网络将节点不同模态的属性信息融合起来，然后将节点的一跳邻居中同一类型的节点用 BLSTM 融合起来，最后再将不同类型的节点信息通过神经网络聚合。HetGNN 可以处理异质关系和异质属性。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icePnrRnxAIkldibTXpj9zoicjpzJ3eOiaRslFTSWJmFq1urtfTWWj7LrTw/640?wx_fmt=png)

图 7：HetGNN 架构示意图

</center>

### 元路径选择

元路径选择是异质信息网络分析的基本步骤，一般来说，都是选择连接关系比较丰富、语义特性比较强的元路径。但是找到这样的元路径需要比较多的领域知识，在实际操作中也会存在一些问题。为此，石川给出了三种解决思路：

1. 把元路径提纯，不同的元路径表示不同方面的信息，再把不同方面纯化后的信息融合起来；

2. 可以舍弃元路径，元路径之所以重要是因为它能抽取高阶关系，如果不用元路径也可以通过保持网络模式的结构特性学习到高阶关系；

3. 自动找寻元路径，例如知识图谱里面有些节点之间存在内在关系，可以借鉴知识补全的思路自动生成元路径。

## 4. 异质图神经网络的应用

### 4.1 套现用户检测

套现是套取现金的简称，一般是指用违法或虚假的手段交换取得现金利益。如图８所示，判断一个用户是不是套现用户，传统方法是把套现用户检测看成一个分类问题，通过抽取出用户特征，然后用分类器来进行分类。这个过程的一个关键问题是怎么才能够抽取出足够丰富的特征。而在电商特别是互联网金融方面，用户特征大量蕴含在交互行为里面，那么怎么从这种交互行为里面抽取出用户特征，是这个问题的关键。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icicORhazwu7WUicAZicjaMJk56s3CMyCHsx2AbMPlQrg3mm1ahoQ3icNjHg/640?wx_fmt=png)

图8：套现用户检测的关键问题和传统解决方案  

</center>

对此，[10] 提出把用户、商家、设备等信息的交互关系构建为一个**异质网络**，网络要学出用户的特征表示。进一步提出的模型首先考虑用户的自然属性信息，以及用户基于不同元路径的邻居。更进一步把用户的不同 Feature 和其邻居特征通过 Feature attention 机制融合起来，最后利用 Path 相关的 Attention，把基于不同元路径的特征融合起来。模型的相关结构说明如图 8 所示 [MOU1] 。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icOeSy2cIYD8hEXLZy3UMVXUWJYVGtWKt7xl4LrBJwCDtJDJkcCRTW0A/640?wx_fmt=png)

图 9：用户套现用户检测模型的相关说明

</center>

### 4.2 意图推荐

意图推荐就是分析用户的潜在意图。

可以设计一个异质网络来解决这个问题 [11]。如图 9 所示，网络中刻划了用户、物品和查询词三者之间的交互关系，模型来学习 user、item、query 之间的表示，然后看看针对一个用户来推荐什么样的 query。我们同样是基于元路径来聚合邻居信息，在这个过程中可以利用不同的元路径来聚合不同的邻居信息。实际问题当中，user、item、query 的量级是相当大的，在亿级的规模。但是用于组成它们的 word 的数量实际上是不大的，大概是十万级别的。那么仅仅学习 word 的特征表示，然后 query 和 item 的特征由一些 word 拼接而成的，这样就减少了计算量。通过和工业界的算法对比，提出的方法利用增加的元路径信息得到了性能的稳步提升。

<center>

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9ictxic6OoMbEm7xcBIicl2360xxbB7Z7oIedpysRkexicFPMadibWHSJib0Zg/640?wx_fmt=png)

图 10：用于意图推荐的异质图神经网络

</center>

### 4.3 用户聚类

用户聚类是利用用户的特征信息，以及用户的社交连接关系对用户做一个类别划分，对广告推荐是很有帮助的。目前深度学习已经广泛应用于推荐、聚类任务当中。

聚类主要是分析用户的特征信息，特征实际上也包含有结构信息，把结构和特征两方面联合起来做聚类：

[12] 首先用深度神经网络学得用户的隐含特征表示，这是深度聚类里面常用的做法。 **另一方面，根据用户的社交关系，构造 KNN 图得到用户的关联图，学得表示过程中，把 DNN 里面学的每一层的特征和 GCN 里面的节点特征表示拼合起来再做聚合。**  如图 10 所示，在这个过程当中，由于这里的 GCN 需要有监督信息，我们把在 DNN 里面做的聚类做一个软划分，得到 Q 分布，进而取平方得到 P 分布，这里的 P 分布当作一个伪标签来指导 GNN 学习，取得了比较明显的效果。

![](https://mmbiz.qpic.cn/mmbiz_png/ZkgfUziaPIO27tLwdPiaqjuwQqgjSG0x9icmPwSfGAlNcj9tQg97ErEfZvgcc9Mm8x7iaGhUng41oyDs5HESZtchFw/640?wx_fmt=png)

图 11：用户聚类模型中 GNN 的监督学习

除了上面的介绍之外，还有一些其他的应用。如共享推荐中一个用户是否把新闻或者商品推荐给他的朋友；基于朋友关系的推荐中根据朋友的点赞信息决定是否给用户推荐。对于这些含有丰富交互信息的应用场景都可以采用异质网络建模和采用图神经网络进行分析。

## 结语

关于异质信息网络特别是异质图网络的未来研究方向：

*   异质图神经网络内在学习机理
    
*   动态网络
    
*   多模态数据处理
    

异质信息网络的更多资料可以访问网站：www.shichuan.org.

**Q&A：**

**Q1: 为什么知识图谱表示学习方法很少用到异质网络表示学习中？**

A1: 我认为知识图谱可以看成是一种异质网络，它是一种模式丰富的异质网络，那么在这种网络里面，它有很多不同类型的结点，有很多不同类型的关系。这个的话实际上是传统的异质网络很少分析的，也是很难分析的。网络表示学习和知识图谱的表示学习，是两种不同的角度来做不同的事情，其实我觉得二者是可以相互借鉴、相互结合的，但是目前还没有什么太多太好的工作。

**Q2: 网络表示学习的结果如何与结点的属性特征、描述类的文本特征进行融合，难点在哪里？以及如何自动发现元路径？**

A2: 一般的话还是要根据领域知识来选择一些语意，选择语意明确、结构丰富的这样一些元路径，这样选出的路径对于实际问题一般来说效果都还不错。

**Q3: 广度学习主要是通过定义元路径来实现的吗？可以谈一谈广度学习和图神经网络的关系以及二者今后如何发展情况吗？**

A3: 广度学习是近些年 Philip S Yu 教授倡导的一个研究方向，里面的主要一种技术方法也是用异质信息网络，异质信息网络可以很自然的把不同方面的信息关联起来，起到一个信息融合的作用。在这里面我们可以用元路径，那么就是说可以融合不同方面的信息，抽取不同方面的这种子结构。所以这是里面一个很重要的方法。

**Q4: 异质信息网络下一步的发展方向是什么？**

A3: 一个是我前面说到的，元路径选择的困境，实际上这是长期困扰这个领域的一个事情。因为这个领域的分析是严重依赖于元路径，怎么能够不依赖于元路径，或者设计一些更好的方法，能够探索语意信息，这是一个方向。在异质图神经网络里面如何更好的做聚合，实际上是研究也才刚刚开始。然后还有像这种动态网络的，在异质图里面怎么考虑这种动态性等等，都值得深入研究。

**参考文献**

[1] YuXiao Dong,Nitesh V. Chawla,Ananthram Swami. Metapath2vec: Scalable Representation Learning for Heterogeneous Networks. KDD， 2017.

[2] Chuan Shi, Binbin Hu, Wayne XinZhao, Philip S. Yu. Heterogeneous Information Network Embedding for Recommendation.TKDE,2018.

[3] Tao-yang Fu, Wang-Chien Lee, ZhenLei. HIN2Vec: Explore Meta-paths in Heterogeneous Information Networks for Representation Learning. CIKM,2017.

[4] Binbin Hu, Chuan Shi, Wayne XinZhao, Philip S. Yu. Leveraging Meta-path based Context for Top-N Recommendation with A Neural Co-Attention Model. KDD,2018.

[5] Binbin Hu, Yuan Fang, Chuan Shi.Adversarial Learning on Heterogeneous Information Networks. KDD,2019.

[6] Chuan Shi, Yuanfu Lu, Linmei Hu,Zhiyuan Liu et al. RHINE: Relation Structure-Aware Heterogeneous Information Network Embedding. TKDE, 2020

[7] Xiaotian Han, Chuan Shi, SenzhangWang, Philip S. Yu, Li Song.Aspect-Level Deep Collaborative Filtering viaHeterogeneous Information Networks. IJCAI,2018.

[8] Wang, Xiao, Houye Ji, Chuan Shi,Bai Wang, Yanfang Ye, Peng Cui, and Philip S. Yu. Heterogeneous Graph Attention Network. WWW,2019.

[9] Chuxu Zhang, Dongjin Song, Chao Huang, Ananthram Swami, Nitesh V. Chawla. Heterogeneous Graph Neural Network.KDD,2019.

[10] Binbin Hu, Zhiqiang Zhang, Chuan Shi, Jun Zhou, XiaoLong Li, Yuan Qi Cash-out User Detection based on Attributed HeterogeneousInformation Network with a Hierarchical Attention Mechanism. AAAI,2019.

[11] Shaohua Fan, Junxiong Zhu, Xiaotian Han, Chuan Shi,Linmei Hu, Biyu Ma, Yongliang Li Metapath-guided Heterogeneous Graph NeuralNetwork for Intent Recommendation. KDD,2019.

[12] Deyu Bo, Xiao Wang, Chuan Shi, Meiqi Zhu, Emiao Lu, PengCui. Structural Deep Clustering Network. WWW,2020. 

## Reference

[北邮教授石川：图神经网络需要解决的几个关键问题](https://mp.weixin.qq.com/s?__biz=MzU5ODg0MTAwMw==&mid=2247486047&idx=1&sn=2052f3f96135cbe48ddfa96841bc404d&chksm=febf499bc9c8c08d99bbf2cc6fee48414b4d920bdaa01ec9c68c9fba74fd634331274af10f09&mpshare=1&scene=23&srcid=&sharer_sharetime=1586774527616&sharer_shareid=fc06814c3a0ce2f45e4998813d787a8f#rd)