.. _ranknet:

Pairwise Rank 之 RankNet
==========================
研一的时候，跟着师兄参加KDDCUP。 
当时需要一个pairwise rank的工具，没有现成的，所以实现了一个LambdaRank的方法。 

LambdaRank是RankNet的改进版，先记录一下RankNet吧，简单许多。 

原理
-----
pairwise rank 通过学习pair对中，两两元素的先后顺序来构建一个整体的排序。

比如，现在已经学习了一个pairwise rank的模型，
总共三个元素ABC。 将pair对 (A,B)输入，得到+1，也就是 A>B，
把(B，C)得到+1，那么自然可以得到 A > B > C，
多个元素的顺序通过确定了两两顺序得到。

损失函数
--------
RankNet 中为两两元素的关系（次序）定义出了大于、小于、等于三种关系。

.. math::

    P_{ij} = \frac{ e^{o_{ij}}} { 1+ e^{o_{ij}}}

其中， :math:`o_{ij} = o_i - o_j` ， 
:math:`o_i = f(x_i)` 可以认为是在某个query下，:math:`i` 的打分（符合程度）。
其中 :math:`f(x_i)` 是神经网络的输出。

类似SVM，pairwise rank 中两个元素 :math:`i,j` ，
如果两者在目标场景下的符合程度不同，
为了能够更好地区分这两个元素的前后顺序，
那么也希望两者得分的差距越大越好（最大间隔）。

分下面三种情况，要求间隔最大化：

1. A 在 B前面（A的打分应该比B高），很明显此时的打分差距 :math:`o_{ij}>0`

.. math::

    \lim_{o_{ij} \rightarrow + \infty} P_{ij} = 1

2. A 和 B 次序相同，很明显，此时打分差距应该为0 ： :math:`o_{ij}=0`

.. math::

    p_{ij} = \frac{1}{1+1} = 0.5


3. A在B的后面，此时打分差距： :math:`o_{ij} <0`

.. math::

    \lim_{o_{ij} \rightarrow - \infty} P_{ij} = 0


如此，就有下面三种P_{ij}的目标值：

1. 对于一种排序，A比B更靠前，则目标值 :math:`\overline{P}_{AB} = 1`
2. B比A靠前： :math:`\overline{P}_{AB} = 0`
3. B 和 A的位置相同 :math:`\overline{P}_{AB} = 0.5`

利用两者间的交叉熵来定义损失函数。

交叉熵定义了两种分布之间的距离：

.. math::

    H(p, q) = - \sum p \log q

最终的损失函数是：

.. math::

    C_{ij} = -P_{ij}o_{ij} + \log (1 + e^{o_{ij}})

References
-----------
`Learning to Rank之RankNet算法简介 <http://www.cnblogs.com/kemaswill/p/kemaswill.html>`_
