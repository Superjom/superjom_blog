=========
各种距离
=========
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

|today|

整理下常用的几种距离函数.

欧式距离(Euclidean Distance)
-------------------------------
两个n维向量 :math:`a=(x_{11}, x_{12}, \cdots, x_{1n})` 与 :math:`b=(x_{21}, x_{22}, \cdots, x_{2n})` 间的距离

.. math::

    d_12 = \sqrt{ (a-b) (a-b)^T} = 
    \sqrt{ \sum_{k=1}^n (x_{1k} - x_{2k})^2}

曼哈顿距离(Manhattan Distance)
-------------------------------
两个n维向量间的曼哈顿距离：

.. math::

    d_{12} = \sum_{k=1}^n |x_{1k} - x_{2k}| 

切比雪夫距离(Chebyshev Distance)
-----------------------------------

二维平面上两点的距离可以认为是国际象棋里国王从 :math:`(x_1, y_1)` 走到临近格子 :math:`(x_2, y_2)` 的步数(国王可以斜着走)：

.. math::

    d_{12} = \max (|x_1 - x_2| , |y_1 - y_2|)

两个n维向量的距离：

.. math::

    d_{12} = \max_i ( 
            \left| x_{1i} - x_{2i} \right| ) 
        = \lim_{k \rightarrow \infty} 
            \left(
                \sum_{i=1}^n |x_{1i} - x_{2i}| ^k 
            \right)^{1/k}

闵可夫斯基距离(Minkowski Distance)
------------------------------------
.. math::
    
    d_{12} = \sqrt[p]{
                \sum_{i=1}^n |x_{1i} - x_{2i}| ^p }

可以看到，前面讲的三种距离就是闵氏距离的特例：

1. :math:`p=1` 时，就是曼哈顿距离
2. :math:`p=2` 时，就是欧氏距离
3. :math:`p \rightarrow \infty` ，就是切比雪夫距离

闵氏距离所代表的距离集合有如下缺点：

1. 将特征的各个维度当成了独立的单位
2. 不能考察各个维度不同的情况。

标准化欧氏距离
---------------
等于在原有的欧氏距离之前，对feature的每一个维度做了一个预处理的过程。

.. math::

    X^* = \frac{X - \mu} {s}

其中 :math:`\mu` 是此维度上的平均值， :math:`s` 是标准差。

马氏距离
------------
有M个样本向量 :math:`x_1, x_m` , 协方差矩阵记为 :math:`S` ， 均值记为 :math:\mu` 

样本向量 :math:`X` 到 :math:`\mu` 的马氏距离定义为：

.. math::

    D(X) = \sqrt{ (X - \mu)^T S^{-1} (X - \mu)}

其中向量 :math:`X_i` 与 :math:`X_j` 之间的马氏距离：

.. math::

    D(X_i, D_j) = \sqrt{(X_i - X_j) ^T S^{-1} （X_i - X_j)}

可以看到，如果协方差矩阵是单位矩阵（各个维度独立同分布）， 那么马氏距离就变成了欧氏距离：

如果协方差矩阵是对角矩阵，那么公式就变成了标准化欧氏距离。


.. math::

    D(X_i, X_j) = \sqrt{(X_i - X_j)^T (X_i - X_j)}

马氏距离对比欧氏距离：

量纲无关，排除了变量之间的相关性的干扰。


Cos相似度(Cosine)
------------------
二维空间的相似度：

.. math::

    cos(\theta) = \frac{x_1 x_2 + y_1 y_2}
            { \sqrt{x_1^2 + y_2^2 } 
                \sqrt{x_2^2 + y_2^2}}

多维空间上的相似度：

.. math::

    cos(X,Y) = \frac{ X.Y} {|X| |Y|}

汉明距离(Hamming distance)
---------------------------
两个等长字符串 :math:`s_1` 和 :math:`s_2` 的汉明距离定义为一个变成另外一个的最小哦啊替换次数， 比如字符串 1111 与 1001 之间的汉明距离为2



杰卡德相似系数(Jaccard similarity coefficient)
------------------------------------------------

杰卡德相似系数
*****************
衡量两个集合A, B的相似度：

.. math::

    J(A, B) =   \frac{A \bigcup} {A \bigcap B}

Jaccard距离
**************
.. math::

    J'(A,B) = 1 - J(A, B)

应用
*******
比如，AB均是两个n维向量，且所有维度上的取值均为0或1，那么将两个样本当作集合，其相似度可以用Jaccard 系数来衡量.

相关系数(Correlation coefficient)
-----------------------------------
.. math::

    \rho_{XY} = \frac{cov(X,Y)}{\sqrt{D(X)} \sqrt{D(Y)}}
            = \frac{E( (X-EX) (Y-EY))}
                {\sqrt{D(X)} \sqrt{D(Y)}}

相关系数是衡量随机变量X与Y相关程度的一种方法，取值 :math:`[-1, +1]` 。

相关系数的绝对值越大，那么表明 X与Y的相关度越高。

相应的+1表明正相关，-1表明负相关。


互信息（mutual information)
------------------------------
互信息是源于信息论的概念，其意义是知道了两个随机变量的一个之后，
在多大程度上可以减少另一个的不确定性。

用以描述两个随机变量间的互相依赖程度或 **相关性** 。

.. math::

    I(X, Y) = \sum_{y\in Y} \sum_{x \in X} p(x,y) 
                \log \left( 
                    \frac{p(x,y)}{p(x) p(y)}
                    \right)

如果， 当 :math:`X` 与 :math:`Y` 完全不相关，或者说是独立事件时， 有两者的互信息为0

.. math::

    \log \left(
        \frac{p(x,y)}
            {p(x) p(y)}
        \right)
    = 0
    

信息增益(Information Gain)
----------------------------
信息增益在决策树里有用到，在每次划分时，寻求信息增益最大化。 

熵可以认为是不确定性，决策树在不断分支的过程可以认为是在不断降低熵（不确定性）的过程，
其中衡量这个不确定性下降程度的标准就是信息增益。

条件熵定义为已知随机变量 :math:`X` 的条件下随机变量 :math:`Y` 的不确定性:

.. math::

    H(Y|X) = \sum_{i=1}^n p_i H(Y|X=x_i)

给一个标准点的定义： 信息增益表示得知特征 :math:`X` 的信息而使得类 :math:`Y` 的信息的不确定性减少的程度。

.. math::

    g(D,A) = H(D) - H(D | A)


信息增益比(Information Gain Ratio)
-------------------------------------
.. math::

    g_R(D,A) = \frac{g(D,A)} {H(D)}

交叉熵
------------
表明对同一数据取得的两个不同的分布 :math:`p, q` 的差距

.. math::

    H(p, q) = - \sum p \log q

RMSE(root mean square error)
------------------------------
经常用作协同过滤里面的误差函数：

.. math::

    RMSE = \sqrt{ \frac{1}{N} \sum_{(u,v)\in R} (r_{u,v} - \hat{r}_{u,v})^2}

其中， :math:`N` 是rating的数目













References
---------------
..[hang-li] 统计学习方法 李航

_`机器学习中的相似度度量 <http://www.cnblogs.com/heaad/archive/2011/03/08/1977733.html>`
