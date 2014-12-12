主成分分析(PCA)
=================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

|today|

我们通常会为解决一个机器学习问题而准备众多的feature。 
但是，feature增多之后，会使得模型的复杂度增加。 PCA一般就是用来降维的工具。



原理
------
PCA能够对基本的数据进行变换，将原始数据通过线性变换到一个新的空间。

PCA通过方差来衡量变换后信息量的保留度，认为方差越大，保留的信息越好，映射的效果也越好。

PCA经常用来减少输入数据的维数，同时保持其中对方差贡献最大的feature，因此常常用于数据降维。


主成分的一般定义
-------------------
设有随机变量 :math:`X_1, X_2, \cdots, X_p` ， 样本标准差为 :math:`S_1, S_2, \cdots, S_p` 。

首先进行标准化变换：

.. math::

    C_j = \sum_{t=q}^p a_{jt} x_t







References
------------
_`主成分分析 <http://blog.csdn.net/xiaoyu714543065/article/details/7832132>`
