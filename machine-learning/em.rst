EM(Expection Maximization) 算法
=================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-7-13*

介绍
-----
EM 算法，或者叫期望最大化算法，主要是解决含有隐变量的模型中的(MLE)最大似然或者(MAP)最大后验中使用。

EM的应用场景描述如下：

假如有一个训练集： :math:`\{ x^{(1)}, x^{(2)}, \cdots, x^{(m)} \}` .
希望用一个模型 :math:`p(x,z)` 来拟合数据，其中 :math:`z` 是隐变量， 整体的似然估计给定如下：

.. math::
    
    \begin{split}
    l(\theta)   & = \sum_{i=1}^m \log p(x;\theta) \\
                & = \sum_{i=1}^m \log p(x,z;\theta) 
    \end{split}

直接求解参数 :math:`\theta` 可能会比较难，但是，如果隐变量 :math:`z^{(i)}` 被观测到了，那么问题解决就会比较容易。

在上述场景，EM算法对MLE会比较高效。

EM算法是一个迭代过程，中间迭代包括两步：

1. E步：不断构建 :math:`l` 的下界
2. M步：不断优化这个下界

Jensen 不等式
----------------
如果 :math:`f` 是一个凸函数，那么假设 :math:`X` 是一个随机值，那么有：

.. math::

    E[ f(X)] \ge f(EX)


推导
-----
对于每个 :math:`i` ，设定 :math:`Q_i` 作为 :math:`z` 的分布，即 :math:`\sum_z Q_i(z) = 1, Q_i(z) \gt 0` 。 

.. math::
    :label: eq-1

    \begin{split}
    l(\theta)  &= \sum_i \log p(x^{(i)};\theta)  \\
               &= \sum_i \log \sum_{z^{(i)}} p( x^{(i)}, z^{(i)}, \theta) \\
               &= \sum_i \log \sum_{z^{(i)}}  
                    \left( 
                        Q_i(z^{(i)}) \frac{p(x^{(i)}, z^{(i)}; \theta)}
                        {Q_i(z^{(i)})}
                    \right) \\
                & = \sum_i \log 
                    \left( 
                    E_{z^{(i)} \sim Q_i} 
                        \left[
                            \frac{p(x^{(i)}, z^{(i)}; \theta)}
                            {Q_i(z^{(i)})}
                        \right]
                    \right) 
    \end{split}
                            
利用Jensen不等式：

.. math::
    :label: eq-2

    \begin{split}
     \log \left( 
        E_{z^{(i)} \sim Q_i} 
            \left[
                \frac{p(x^{(i)}, z^{(i)}; \theta)}
                {Q_i(z^{(i)})}
            \right]
        \right) 
    &\ge 
        E_{z^{(i)} \sim Q_i} 
        \log \left[
            \frac{p(x^{(i)}, z^{(i)}; \theta)}
            {Q_i(z^{(i)})}
        \right] \\
    & = \sum_{z^{(i)}} Q_i(z^{(i)}) 
            \log \frac{p(x^{(i)}, z^{(i)}; \theta)} {Q_i(z^{(i)})}
    \end{split}
    
结合公式 :eq:`eq-1` :eq:`eq-2` 可以得到如下结论：

.. math::
    :label: lower-bound

    \begin{split}
    l(\theta)  &= \sum_i \log p(x^{(i)};\theta)   \\
               &\ge \sum_i \sum_{z^{(i)}} Q_i(z^{(i)}) 
                    \log \frac{p(x^{(i)}, z^{(i)}; \theta)} {Q_i(z^{(i)})}
    \end{split}

也就是，我们找到了 :math:`l(\theta)` 的一个下界。 

对于 :math:`Q_i` 的分布，设为 :math:`z^{(i)}` 的后验分布:

.. math::
    :label: Q_i

    Q_i(z^{(i)}) = p(z^{(i)} | x^{(i)}; \theta)

上面公式 :eq:`lower-bound` 得到了似然函数的下界，那么最大似然就转变成了最大化这个下界。

这个最大化的操作被分成了两步：

1. E步骤：对于每个 :math:`i` ，利用公式 :eq:`Q_i` 求解出 :math:`Q_i`
2. M步骤：利用下界 :math:`lower-bound` ， 求得最大下界的 :math:`\theta`

这两步不断迭代，直到收敛。








References
------------
[stanford-ml] `Andrew NG, the EM algorithm`

`罗维博客，EM算法 <http://luowei828.blog.163.com/blog/static/3103120420120142193960/>`_






