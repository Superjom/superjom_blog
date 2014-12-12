Mini Batch K-Means
================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-7-20*

在sklearn里面发现有mini-batch KMeans这个算法。 

之前在KDDCUP2013的时候也的确用过这个算法来聚类，速度的确比标准被KMeans要快很多。

本文就对这个算法进行一个简单的介绍。

K-Means as an EM Style Algorithm
----------------------------------
Batch KMeans 可以说Mini-batch KMeans 的前身。

在 [convergence]_ 中将 KMeans 当成一种EM算法的应用。

每个cluster的中心点的集合 :math:`\{w_k|k=1,\cdots,K\}` 可以认为是EM算法的隐变量。

KMeans求解的两个过程恰好也是与EM算法对应：

1. (开始的时候随机初始化隐变量)根据隐变量 :math:`w_k` ，求解每个cluster的点集合
2. 根据每个cluster的点集合，求解隐变量 :math:`w_k`

这两个过程中，第1个过程的变化可以得到如下两个变种：

Batch K-Means
****************
对于数据集合 :math:`{x_i| i=1, \cdots}` ，损失函数是：

.. math::
    :label: E_w

    E(w) = \sum_i L(x_i, w) = \sum_i \frac{1}{2} \min_k (x_i-w_k)^2

定义 :math:`w_{s_i(w)}` 表示 :math:`x_i` 最近的cluster中心点。

公式 :eq:`E_w` 可以变成：

.. math::

    E(w) = \sum_i \frac{1}{2} (x_i - w_{s_i(w)})^2 

利用梯度下降法求解：

.. math::
    :label: w_k 

    \begin{split}
    \triangle w_k & = - \varepsilon \frac{\partial E(w)} {\partial w} \\
    & = \begin{cases}
        \varepsilon(x_i - w_k) & \text{if } k=s_i(w)\\
        0 & \text{otherwise}.
        \end{cases}
    \end{split}

其中的 :math:`\varepsilon` 是学习参数，可以人为定义。 但是在 Batch KMeans 中，有更好的方式来决定 :math:`\varepsilon` 。

假如，设定 :math:`N_k` 是属于对应的 cluster 的点的数目。那么设定 

.. math::

    \varepsilon_k = \frac{1}{N_k}

那么上面公式 :eq:`w_k` 可以变为 Batch KMeans的最终形式：

.. math::

    \triangle w_k = \sum_i 
        \begin{cases}
            \frac{1}{N_k} (x_i - w_k) & \text{if } k=s(x_i,w) \\
            0                       & \text{otherwise.}
        \end{cases}

Online K-Means
*****************
在线学习版本的K-Means，需要有一个渐进的学习过程，对较大数据或者动态增长的数据比较合适。

假定 :math:`n_k` 表示，在此刻对应 :math:`w_k` 的cluster的点的个数，
这个数目会随着时间，动态变化。

那么将 :eq:`w_k` 转化为一个在线学习的版本：

.. math::

    \begin{split}
    \triangle n_k  & = 
        \begin{cases}
            1   &   \text{if } k = s(x_i, w) \\
            0   &   \text{otherwise.}
        \end{cases}  \\
    \triangle w_k & = 
        \begin{cases}
            \frac{1}{n_k} (x_i - w_k) & \text{if } k=s(x_i, w) \\
            0                           & \text{otherwise.}
        \end{cases}
    \end{split}

对比
*****
Batch KMeans方法效果不错，但是对于大数据的处理受处理数据的量的限制（一次训练出来，可能对内存的要求比较高）。

Online KMeans利用SGD方法训练，收敛速度比较快，但是由于SGD中间的噪音问题，会走很多弯路，最终效果可能没有Batch KMeans方法好。

Mini-Batch K-Means
----------------------
`web-scale`_ Mini-Batch KMeans方法的思路是，结合 Batch和Online KMeans两种方法的优点，通过抽取部分样本形成batch：

1. mini-batch 能够减小SGD中间更新的随机性，减小噪音影响
2. mini-batch 相对于Batch方法，提高了数据增长的容纳能力

中间，mini-batch 的获得方式，是随机从原始数据中抽取一个小的样本集，在此样本集上运行K-Means。

每轮训练都会抽取一个mini batch，进行训练。

类似Online KMeans中的学习参数：:math:`\frac{1}{n_k}`

.. math::
    
    w_k \leftarrow (1- \frac{1}{n_k}) w_k + \frac{1}{n_k} x_i 
        \text{ for } x_i \text{ in mini batch }\text{if } k=s(x_i, w) 
    

具体的算法如下::

    for i in range(n_turns):
        mini_batch = select_randomly(dataset, size)

        for x_i in mini_batch:
            k = s(x_i, ws)
            n_k = len(clusters[k])
            r = 1 / n_k
            ws[k] += (1-r) * ws[k] + r * x_i 
             
        



















References
------------
.. [convergence]  Convergence Properties of the K-Means Algorithm
.. [web-scale]  Web-Scale K-Means Clustering
