---
layout: post
title: "论文阅读 ICLR2017 SEMI-SUPERVISED CLASSIFICATION WITH
GRAPH CONVOLUTIONAL NETWORKS"
date: 2020-06-30
excerpt: 
tags: [GCN, 论文]
feature: 
comments: false
---





### 2、图上的快速近似卷积

这篇论文提出的模型的层间传播公式是这样的
$$
H^{(l+1)}=\sigma\left(\tilde{D}^{-\frac{1}{2}} \tilde{A} \tilde{D}^{-\frac{1}{2}} H^{(l)} W^{(l)}\right)
$$
其中$H^{(l)} \in \mathbb{R}^{N \times D}
$表示第l层GCN里面图中所有节点隐含向量的堆积，$H^{(0)}=X$；$\tilde{A}=A+I_{N}$表示给图加了自环以后的邻接矩阵；$\tilde{D}_{i i}=\sum_{j} \tilde{A}_{i j}$为度矩阵；激活函数$\sigma$是$RELU(·)=\max(0,·)$。

下面开始解释这个公式的由来，首先将图上的谱卷积定义为卷积核$g_{\theta}$与输入信号$x$的乘积：
$$
\begin{equation}
g_{\theta} \star x=U g_{\theta} U^{\top} x
\end{equation}
$$
其中$g_{\theta}$可以视为一个函数$g_{\theta}(\Lambda)$，$\theta \in \mathbb{R}^{N}$是参数，函数输出是一个对角阵。这里特征向量矩阵$U$和特征值矩阵$\Lambda$来自图正则化拉普拉斯矩阵的特征分解$L=I_{N}-D^{-\frac{1}{2}}AD^{-\frac{1}{2}}=U \Lambda U^{\top}$（PS：一般来说特征分解应该是$U\Lambda U^{-1}$的形式，但由于正则化拉普拉斯矩阵是一个半正定阵，必然有n个不同的特征值即所有的特征向量相互正交，有$U^{-1}=U^{\top}$）。对于输入信号$x$有$x \in \mathbb{R}^{N}$，这是一种简化的情况，每个点的输入信号都是一个标量。

但这样计算是十分耗时的，对于一个大图来说，拉普拉斯矩阵的特征分解计算量巨大，同时计算与$U$的矩阵乘法复杂度是$O(n^{2})$。所以在这篇论文里[^1]提出了一种利用切比雪夫多项式$T_{k}(x)$的K阶截断来近似$g_{\theta}(\Lambda)$的方法：
$$
\begin{equation}
g_{\theta^{\prime}}(\Lambda) \approx \sum_{k=0}^{K} \theta_{k}^{\prime} T_{k}(\tilde{\Lambda})
\end{equation}
$$
这里$\theta^{\prime} \in  \mathbb{R}^{K}$已经变成了切比雪夫多项式的系数。并且有$\tilde{\Lambda}=\frac{2}{\lambda_{max}}\Lambda-I_{N}$，$\lambda_{max}$表示最大的特征值，这个再次正则化的操作是可能为了满足切比雪夫多项式展开的条件（第一类切比雪夫多项式有一个`arccos x`项，要求$x \in [-1, 1]$）。关于切比雪夫多项式，有$T_{k}(x)=2xT_{k-1}(x)-T_{k-2}(x)$，$T_{0}(x)=1$，$T_{1}(x)=x$。现在对于卷积核$g_{\theta^{\prime}}$和输入信号$x$，卷积变为：
$$
\begin{equation}
g_{\theta^{\prime}} \star x \approx \sum_{k=0}^{K} \theta_{k}^{\prime} T_{k}(\tilde{L}) x
\end{equation}
$$
注意这里$T_{k}(\tilde{L})$的出现，本来应该是$UT_{k}(\tilde{\Lambda})U^{\top}$，但是考虑到特征分解有$(U\Lambda U^{\top})^{k}=U\Lambda^{k}U^{\top}$（展开计算就是）这么个性质，可以把$UT_{k}(\tilde{\Lambda})U^{\top}$转化到$T_{k}(\tilde{L})$，其中$\tilde{L}=\frac{2}{\lambda_{max}}L-I_{N}$，这么转化可以避免特征分解，减少矩阵乘法的次数，这就是一个K阶近似的谱卷积。

这篇论文在这个近似的基础上取$K=1$，并进一步假设近似$\lambda_{max}=2$<font color="red">（似乎可以通过瑞利熵来确定特征值的范围[^2]，算出来最大特征值为2）</font>，然后得到了下面的公式：
$$
\begin{equation}
g_{\theta^{\prime}} \star x \approx \theta_{0}^{\prime} x+\theta_{1}^{\prime}\left(L-I_{N}\right) x=\theta_{0}^{\prime} x-\theta_{1}^{\prime} D^{-\frac{1}{2}} A D^{-\frac{1}{2}} x
\end{equation}
$$
进一步取$\theta=\theta^{\prime}_{0}=-\theta^{\prime}_{1}$，可以简化为下面这样：
$$
\begin{equation}
g_{\theta} \star x \approx \theta\left(I_{N}+D^{-\frac{1}{2}} A D^{-\frac{1}{2}}\right) x
\end{equation}
$$
（作者在这里提了一句说$I_{N}+D^{-\frac{1}{2}} A D^{-\frac{1}{2}}$的特征值在$[0,2]$的范围内，可以通过$L$与$\Lambda$相似而推导出）多次使用这个式子导致了数值不稳定、梯度消失/爆炸的情况，所以这里用了一个<font color="red">renormalization trick（当然也没有解释为什么）</font>：$I_{N}+D^{-\frac{1}{2}} A D^{-\frac{1}{2}} \rightarrow \tilde{D}^{-\frac{1}{2}} \tilde{A} \tilde{D}^{-\frac{1}{2}}$，其中有$\tilde{A}=A+I_{N}$以及$\tilde{D}_{ii}=\Sigma_{j}\tilde{A}_{ij}$（一篇文章给出了添加自环以后的拉普拉斯矩阵特征值范围的证明[^3]，并说明其可以缩小底层图的谱）。进一步将输入信号$x$中单个点的信号推广到$C$维，即现在输入信号$X \in \mathbb{R}^{N \times C}$；同样地推广卷积核参数$\theta$，假设使用$F$个不同的卷积核，则有$\Theta \in \mathbb{R}^{C \times F}$，现在卷积的输出可以表现为：
$$
\begin{equation}
Z=\tilde{D}^{-\frac{1}{2}} \tilde{A} \tilde{D}^{-\frac{1}{2}} X \Theta
\end{equation}
$$
卷积后的结果$Z \in \mathbb{R}^{N \times F}$就表示了所有节点上输入信号的卷积结果的堆叠。在这个基础上加入激活函数$\sigma$，把$\Theta$换成$W$，把$Z$和$X$换成对应的$H$，就得到了最开始的层间传播公式。

#### 关于图上的谱卷积

涉及到图上的信号处理已经有一篇综述[^4]总结了，感觉不是非常详细，知乎上有一个回答的图卷积说得很不错[^5]，在这里整合一下这两个。

首先是傅里叶变换，传统的傅里叶变换表示成：
$$
\begin{equation}
F(\omega)=\mathcal{F}[f(t)]=\int f(t) e^{-i \omega t} d t
\end{equation}
$$
可以视作是$f$在以为拉普拉斯算子的特征函数$e^{-i \omega t}$上的拓展：对于广义的特征方程$AV = \lambda V$，$A$表示一个变换，$V$表示特征向量或者特征函数，$\lambda$表示特征值，有$\Delta e^{-i \omega t}=\frac{\partial^{2}}{\partial t^{2}} e^{-i \omega t}=-\omega^{2} e^{-i \omega t}$，由此可以认为$e^{-i \omega t}$是拉普拉斯算子的特征函数，$\omega$与特征值相关。[^5]

根据傅里叶变换开始联想，图的拉普拉斯矩阵就是图上的拉普拉斯算子(类比离散拉普拉斯算子)，则基于它的特征向量可以在图上进行一个类似傅里叶的变换：
$$
\begin{equation}
F\left(\lambda_{l}\right)=\hat{f}\left(\lambda_{l}\right)=\sum_{i=1}^{N} f(i) u_{l}^{*}(i)
\end{equation}
$$
表示为图上的向量$f$和拉普拉斯算子的第$l$个特征向量$u_l$的共轭(<font color=red>说是因为复空间的内积，所以用了共轭</font>)的内积(连续域上的积分对应到离散域上就是内积)，这就是图上的傅里叶变换，推广到矩阵形式：
$$
\begin{equation}
\left(\begin{array}{c}
\hat{f}\left(\lambda_{1}\right) \\
\hat{f}\left(\lambda_{2}\right) \\
\vdots \\
\hat{f}\left(\lambda_{N}\right)
\end{array}\right)=\left(\begin{array}{cccc}
u_{1}(1) & u_{1}(2) & \ldots & u_{1}(N) \\
u_{2}(1) & u_{2}(2) & \ldots & u_{2}(N) \\
\vdots & \vdots & \ddots & \vdots \\
u_{N}(1) & u_{N}(2) & \ldots & u_{N}(N)
\end{array}\right)\left(\begin{array}{c}
f(1) \\
f(2) \\
\vdots \\
f(N)
\end{array}\right)
\end{equation}
$$
写作$\hat{f}=U^{\top}f$，$U$的定义跟上面一样。

类似地，根据传统的傅里叶逆变换
$$
\begin{equation}
\mathcal{F}^{-1}[F(\omega)]=\frac{1}{2 \Pi} \int F(\omega) e^{i \omega t} d \omega
\end{equation}
$$
是变换与$e^{i \omega t}$对$\omega$求积分，则图上的的傅里叶逆变换可以表示为对特征值求和：
$$
\begin{equation}
f(i)=\sum_{l=1}^{N} \hat{f}\left(\lambda_{l}\right) u_{l}(i)
\end{equation}
$$
同样推广到矩阵形式：
$$
\begin{equation}
\left(\begin{array}{c}
f(1) \\
f(2) \\
\vdots \\
f(N)
\end{array}\right)=\left(\begin{array}{cccc}
u_{1}(1) & u_{2}(1) & \ldots & u_{N}(1) \\
u_{1}(2) & u_{2}(2) & \ldots & u_{N}(2) \\
\vdots & \vdots & \ddots & \vdots \\
u_{1}(N) & u_{2}(N) & \ldots & u_{N}(N)
\end{array}\right)\left(\begin{array}{c}
\hat{f}\left(\lambda_{1}\right) \\
\hat{f}\left(\lambda_{2}\right) \\
\vdots \\
\hat{f}\left(\lambda_{N}\right)
\end{array}\right)
\end{equation}
$$
写作$f = U \hat{f}$。[^5]

前面铺垫了大量的关于傅里叶变换的内容，<font color=red>目的是</font>使用傅里叶变换的一个性质：卷积定理。卷积定理简单说起来就是**卷积的变换等于变换的乘积**，这样虽然我们不能直接定义出卷积，但是可以通过傅里叶变换来导出图上的卷积。对于图上的向量$f$和卷积核$g$，有$\hat{f}=U^{\top}f$和$\hat{g}=U^{\top}g$。对于这两个变换的乘积不能简单地使用内积，由于在连续域上两个函数相乘仍然得到一个函数，这里两个向量相乘也应该得到一个向量，因此应该使用哈达玛积(Hadamard product，即对应位置相乘)，所以图上的谱卷积公式为：
$$
\begin{equation}
(f * g)_{G}=U\left(\left(U^{\top} g\right) \odot\left(U^{\top} f\right)\right)
\end{equation}
$$
也可以把哈达玛积换一个写法，既然是对应位置相乘，把$\hat{g}$写成对角阵再进行矩阵乘法结果是一样的：
$$
\begin{equation}
\hat{g} = \left(\begin{array}{ccc}
\hat{g}\left(\lambda_{1}\right) & & \\
& \ddots & \\
& & \hat{g}\left(\lambda_{n}\right)
\end{array}\right)
\end{equation}
$$
则卷积公式可以写成：
$$
\begin{equation}
(f * g)_{G}=U\left(\begin{array}{ccc}
\hat{g}\left(\lambda_{1}\right) & & \\
& \ddots & \\
& & \hat{g}\left(\lambda_{n}\right)
\end{array}\right) U^{T} f
\end{equation}
$$
这就跟本节最开始提到的谱卷积是一个形式了。[^5]

还有一点需要注意的是，在连续域上的卷积通常可以写成$f * g = \int_{-\infty}^{\infty} f(\tau) g(t-\tau) d \tau$，但在离散域和图上是找不到$g(t-\tau)$这一项的，所以只能简单地替换成拉普拉斯算子的特征向量[^4]。

[^1]: David K. Hammond, Pierre Vandergheynst, and R´emi Gribonval. Wavelets on graphs via spectral graph theory. Applied and Computational Harmonic Analysis, 30(2):129–150, 2011.
[^2]: https://zhuanlan.zhihu.com/p/65447367
[^3]: Felix Wu and Tianyi Zhang. Simplifying Graph Convolutional Networks. arXiv:1902.07153v1 [cs.LG] 19 Feb2019
[^4]: D. I. Shuman, S. K. Narang, P. Frossard, A. Ortega and P. Vandergheynst, "The emerging field of signal processing on graphs: Extending high-dimensional data analysis to networks and other irregular domains," in *IEEE Signal Processing Magazine*, vol. 30, no. 3, pp. 83-98, May 2013, doi: 10.1109/MSP.2012.2235192.
[^5]: https://www.zhihu.com/question/54504471/answer/332657604

