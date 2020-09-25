---
title: GCN的原理
date: 2020-02-08 14:38:14
tags: 科研
categories: 科研
mathjax: true
---

### GCN的由来和最初的原理
* Graph上的拉普拉斯矩阵L：
$$L = D - A$$ 其中D是度矩阵，A是邻接矩阵。
对L进行对称归一化：
$$L = I - D^{-1/2} A D^{-1/2} $$ 拉普拉斯矩阵具有良好的性质，它是**对称半正定**的，特征分解可以写成：
$$L = U \Lambda U^T$$ 其中$U$为正交阵，即$UU^T = I$。
* GCN源于传统的傅里叶变换：
$$F(w) = \mathcal{F}[f(t)] = \int f(t)e^{-jwt}dt$$ 其中$e^{-jwt}$是傅里叶变换的基函数，它是拉普拉斯算子$\Delta$的特征函数，其中w与特征值有关。
拉普拉斯算子$\Delta$与$e^{-jwt}$满足特征方程：
$$\Delta e^{-jwt} = \frac{\partial^2 e^{-jwt}}{\partial t^2} = -w^2 e^{-jwt}$$
* 对应到Graph上的傅里叶变换：
$$F(\lambda_l) = \hat{f}(\lambda_l) = \sum\limits_{i=1}^{n} f(i)u_l(i)$$ 这里$u_l(i)$对应传统傅里叶变换中的基函数，$u_l(i)$在这为拉普拉斯矩阵的特征向量矩阵$U$的各个分量，具体为第$l$个特征向量的第$i$个分量。写成矩阵形式：

$$\left[ \begin{matrix} \hat{f}(\lambda_1)\\ \hat{f}(\lambda_2)\\ \vdots \\ \hat{f}(\lambda_n) \end{matrix} \right] = \left[ \begin{matrix} u_1(1) & u_1(2) & \cdots & u_1(n)\\ u_2(1) & u_2(2) & \cdots & u_2(n)\\ \vdots \\ u_n(1) & u_n(2) & \cdots & u_n(n) \end{matrix} \right] = \left[ \begin{matrix} f(\lambda_1)\\ f(\lambda_2)\\ \vdots \\ f(\lambda_n) \end{matrix} \right]$$

改写为矩阵形式，$U^T$即为拉普拉斯矩阵分解后的特征向量矩阵的转置：
$$\hat{f} = U^T f$$ 对应的逆变换：
$$f = U \hat{f}$$
* **图卷积**
* 传统卷积: (卷积的傅里叶变换等于傅里叶变换的乘积)
$$\mathcal{F}[f*h] = \hat{f} \cdot \hat{h}$$

变换形式：

$$f*h = \mathcal{F}^{-1} [\hat{f} \cdot \hat{h}]$$
* 传统卷积推广到Graph上：
设$f$为待卷积函数，$h$为卷积核，即滤波器，
$$f*h = U \left[ \begin{matrix} \hat{h}_1\\ & \hat{h}_2 \\ & & \ddots\\ & & & \hat{h}_n \end{matrix} \right] U^T f$$ 
**这个滤波器的傅里叶变换 $\hat{h}_i$ 也就是我们要设计的部分！**
把上面矩阵写成符号表示：
$$f * h = U [(U^T h) \odot (U^T f)]$$
**把$\hat{h}_i$看成卷积上的滤波器，即卷积核，我们希望卷积核能捕捉“局部特征”，所以定义$\hat{h}_i$为拉普拉斯矩阵的函数 $h(L)$。**
注意$L$和$\Lambda$是有关联的，所以我们把 $\hat{h}_i$ 进一步定义成 $\Lambda$ 的函数(为什么这么定义后边能看出来)：
$$\hat{h} = g_{\theta}(\Lambda)$$
，其中$\theta$代表参数。然后改写卷积公式：
$$g_{\theta}*x = U \cdot g_{\theta}(\Lambda) \cdot U^T x$$
，由于特征分解的计算复杂度是相当高的，所以我们引入**Chebyshev多项式**对$g_{\theta}(\Lambda)$进行展开。
切比雪夫多项式的定义：
$$T_0(x) = 1; T_1(x) = x; T_{n+1}(x) = 2xT_{n}(x) - T_{n-1}(x)$$，进一步，n次多项式按切比雪夫多项式的展开式：
$$p(x) = \sum\limits_{k=0}^{K} a_n T_{k}(x)$$
然后，把$g_{\theta}(\Lambda)$按chebyshev多项式展开：
$$g_{\theta}(\Lambda) \approx \sum\limits_{k=0}^{K} \theta_{k} T_k(\tilde{\Lambda})$$
,其中，**$\tilde{\Lambda} = \frac{2}{\lambda_{max}} \Lambda - I$,放缩到$[-1,1]$之间，保证每阶chebyshev多项式的收敛性**。
又因为，
$$L^K = (U \Lambda U^T) ^ K = U \Lambda^K U^T$$，把$U \cdot g_{\theta}(\Lambda) \cdot U^T$对应成$\tilde{L}$的函数。对$\tilde{\Lambda}$为自变量，其切比雪夫多项式，有：
$$T_0(\tilde{\Lambda}) = I, T_1(\tilde{\Lambda}) = \tilde{\Lambda}, T_2(\tilde{\Lambda}) = 2\tilde{\Lambda}^2 - I$$，$U \cdot T_i(\tilde \Lambda) \cdot U^T$则对应成：
$$T_0(\tilde{L}) = I, T_1(\tilde{L}) = \tilde{L}, T_2(\tilde{L}) = 2\tilde{L}^2 - I$$，继续改写卷积公式：
$$g_{\theta'} * x = \sum\limits_{k=0}^{K} \theta'_{k} T_k(\tilde{L}) x \tag{1}$$, 其中，$\tilde{L} = \frac{2}{\lambda_{max}}L - I$。这时候，根据chebyshev多项式，可以把$T_k(\tilde{L})$看成是$\tilde{L}$的幂级数。
现在首先以拉普拉斯矩阵$L$为例，分析一下他的谱性质：对于归一化的$L$矩阵：
$$L = I - D^{-1/2} A D^{-1/2}$$,先分析右半边：其特征方程可以写成：
$$D^{-1/2} A D^{-1/2} \vec{p} = \lambda \vec{p}$$,左乘$D^{-1/2}$,
$$D^{-1} A D^{-1/2} \vec{p} = \lambda D^{-1/2} \vec{p}$$,特征值不变，特征向量变成$D^{-1/2} \vec{p}$。所以对称归一化的A矩阵与随机游走归一化的A矩阵特征值是相同的。
很容易可以得到，$\|D^{-1}A\|_1 = 1$，其实他的1-范数和$\infty$-范数都是1。由范数的性质，一个矩阵的所有的特征值的绝对值都小于等于该矩阵的任意范数。
$$|\lambda| \leq \|D^{-1}A\|_1 = 1$$
,所以特征值范围是$[-1,1]$。所以L的特征值范围是$[0,2]$。
现在做一个近似，$\lambda_{max} \approx 2$, 所以：
$$\tilde{L} = \frac{2}{\lambda_{max}}L - I = L - I$$。 现在回到卷积的公式(1)，我们手动让$K = 1$，代表1-order Chebyshe Filter，即一阶切比雪夫滤波器。
$$g_{\theta'} * x = (\theta'_0 I + \theta'_1 \tilde{L}) x = (\theta'_0 I + \theta'_1 (L - I))) x$$，令$\theta'_0 = -\theta'_1 = \theta$,继续改写：
$$g_{\theta'} * x = (\theta'_0 I + \theta'_1 \tilde{L}) x = \theta (I + D^{-1/2} A D^{-1/2}) x \tag{2}$$
,**现在得到的矩阵就是1-order Chebyshe Filter**。切比雪夫多项式里的$K$就代表了近邻的阶数（层数）。举个例子，以$K=2$为例，我们计算一下$L$和$L^2$来对比一下。

![](./LandL2.png)
<center>Simple-GCN原理图</center>

可以看出，**K每增加1，高阶近邻位置上产生权值，即与多一层的近邻产生联系。
不过，高阶的近邻与中心点的关联逐渐减小**，这也正与**滤波器的"局部性"** 相契合。以$K=2$为例，分析一下chebyshe展开：
$$\theta_0 I + \theta_1 \tilde{L} + \theta_2 \tilde{L}^2 = (\theta_0 - \theta_1 + \theta_2) I + (\theta_1 - 2\theta_2) L + \theta_2 L^2$$
，自行代入，我们可以发现：所有的$\theta_i$都是一个常数，即**对同阶近邻来说，不论它属于谁的邻域，都共享同一个权值**$\theta_i$，这样有优点也有缺点。
**优点：** 对大规模图来说，参数只有$K+1$个，参数量小。
**缺点：** 不能在不同的邻域内分配不同的权值。

#### GCN的发展
后来，**1-order Chebyshev滤波器被改进,采用了Renormalization Trick**：
$$S = \tilde{D}^{-1/2}\tilde{A}\tilde{D}^{-1/2}$$,A和D分别是加了自环(self-loops)后的邻接矩阵和度矩阵。
GCN中初始的first-order Chebyshev filter是：$S_{1-order} = I + D^{-1/2}AD^{-1/2}$，归一化的拉普拉斯矩阵：$\Delta_{sym} = I - D^{-1/2}AD^{-1/2}$，所以一阶切比雪夫滤波器变成：$S_{1-order} = 2I - \Delta_{sym}$。然后对于$S^K_{1-order}$，滤波系数是$g_i = (2 - \lambda_i)^K$,当$\lambda < 1$时随着K增加系数爆炸式增长，不好！
然后采用了【再归一化】，$S = \tilde{D}^{-1/2}\tilde{A}\tilde{D}^{-1/2}$,然后现在的拉普拉斯矩阵就变成了$\tilde{D}_{sym} = I - \tilde{D}^{-1/2}\tilde{A}\tilde{D}^{-1/2}$，滤波器系数变为$g_i = (1 - \tilde{\lambda}_i)^K$,性能变好了！

**从此，目前的GCN框架变成了：**

![](./GCN-schematic.png)
<center>GCN原理图</center>

给定一个图，$G = (V, A)$，V是顶点集，A是邻接矩阵（对称阵）。用D表示图G的度矩阵，$D=diag(d_1,\cdots,d_n)$。用$H^{(i)}$表示图卷积中的结点特征。
输入的结点特征矩阵：
$$X = \{x_1^T,x_2^T,\cdots,x_n^T\}^T \in \mathbb{R}^{n \times d}$$
初始特征:$$H^{(0)} = X$$
图卷积包括三个步骤：特征传播、线性变换、非线性激活。
##### 特征传播：
给初始的邻接矩阵 A 添加自环(self-loops):
$$\tilde{A} = A + I$$ 用 $\tilde{D}$ 表示 $\tilde{A}$ 的度矩阵。
定义归一化的邻接矩阵：
$$S = \tilde{D}^{-\frac{1}{2}} \tilde{A} \tilde{D}^{-\frac{1}{2}}$$ 这个S在所有层都是一样的，直接通过$\tilde{A}$即可求得。
第k层的特征传播：
$$\tilde{H}^{(k)} \leftarrow SH^{(k-1)}$$
分开写：
$$\tilde{h}^{k}_{i} \leftarrow \frac{1}{d_i+1}h_i^{(k-1)} + \sum\limits_{j=1}^{n} \frac{a_{ij}}{\sqrt{(d_i+1)(d_j+1)}} h_j^{(k-1)}$$
##### 特征变换 + 非线性激活：
每一层有个权值矩阵$\theta^{(k)}$
$$H^{(k)} \leftarrow ReLU(\tilde{H^{(k)}} \theta^{(k)})$$
##### 分类器：
$$Y_{GCN} = softmax(SH^{(k-1)} \theta^{(k)})$$
##### 参数细节：
**图卷积的层数控制了图卷积的感受野，即随着层数K加深，与更高阶的近邻产生关系**。

### GCN与LLE和线性变换的关系 
现在的主流GCN框架：
$$H^{(k+1)} = ReLU\left(S H^{(k)} W^{(k)}\right)$$,其中$H^{(0)} = X$。以$K = 0$为例，分析第一层网络：
$$H^{(1)} = ReLU\left(S H^{(0)} W^{(0)}\right)$$,**左半部分$Y=S X$,本质就是LLE**：
LLE在每个高维点的局部拟合超平面，用k-近邻来线性表示被拟合点，所以在高维空间中，这个高维拟合（高维重建）过程可以用公式表示为:
$$\hat{X} = X W = (W^T X^T)^T$$,$X \in \mathbb{R}^{D\times n}$是按列排布的高维数据点，每列是一个高维样本点；$W \in \mathbb{R}^{n \times n}$是对应每个被重建点的权值向量$\vec{w_i}$，对应$W$矩阵的第 $i$ 列，这个向量的第j个位置上的值即为重建第i个点所需的第j个权值(对应第j个点)，非近邻点权值就是0。
**LLE与GCN的左半部分对比，是同样的，LLE里的$W^T$和$X^T$分别对应GCN中的$S$和$H^{(0)}$。** 其实，从$A$矩阵的归一化上来看，GCN中的$S$与LLE里的$W$也是相通的，因为LLE的W也有一个$\sum_j w_{ij} = 1$的约束。
**右边就是一个纯粹的线性变换$W$，可以把它看成是"特征增强"。** 同样可以类比线性降维的通式：
$$Y = W^T X = (X^T W)^T$$, **这里的$X^T$和$W$分别对应GCN里的$S H^{(0)}$和$W^{(0)}$**。
**所以，一层GCN就相当于LLE和线性变换的集成。对于GCN，左边的$SH$我们把它称为特征按近邻进行聚合，再乘$W$叫做线性变换（特征增强）。** 
所以，$S$是我们可以优化的点，$S \odot M$对$S$做hadamard积，就相当于加权了，这个思想在ST-GCN和很多论文中已经被提出了。


### 新论文 Simple Graph Convolution
它认为图卷积GCN和多层感知机MLP类似，只不过每一层当中对特征按照其近邻进行了平均化。这篇文章认为图卷积受目前普通卷积神经网络的影响，一开始就加入了非线性变换，还把网络层数搞得很深，于是他们把图卷积进行了简化，去掉了非线性激活，把图卷积网络改成了一个逻辑回归，在某些数据上表现不错。
![](./Simple-GCN-schematic.png)
<center>Simple-GCN原理图</center>

##### 图卷积的线性化：
$$Y = softmax(S\dots SSX \theta^{(1)}\dots \theta^{(k)})$$ 所有的S都是一样的，可以用$S^K$表示。然后后面的所有变换矩阵乘起来变成了一个矩阵$\theta = \theta^{(1)}\dots \theta^{(k)}$。
然后就变成了一个多分类的逻辑回归：
$$Y = softmax(S^K X \theta)$$
$S^K$可以在预处理阶段就可完成，因为需要用到的东西都是已知的。这样，
$$\tilde{X} = S^K X$$ 然后$softmax(\tilde{X} \theta)$就变成了单纯的多分类逻辑回归。优化可以直接利用逻辑回归的优化方法，如随机梯度下降（SGD）等。

### GCN优化
#### DeepGCNs 系列
GCN初始版本，kpif&welling那个，在core数据集上只用到了2-hop近邻。其实GCN深度增加会降低模型效果，因为过度平滑问题，所有最初的GCN层数不能很深。

后来有人讨论了GCN的模型深度问题，用了ResGCN，DenseGCN，加深了网络层数，并提高了performance。


