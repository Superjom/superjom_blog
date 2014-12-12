================
拉格朗日对偶
================
在SVM的推导里面，会用到 Lagrange duality. 
这一章专门回顾一下这个知识点。

如果有这样一个优化问题：

.. math::
    
    \min_w f(w)

    s.t. h_i(w) = 0, i=1, \cdots, l

对于这个问题，我们定义Lagrange函数为：

.. math::
    
    L(w, \beta) = f(w) + \sum_{i=1}^l \beta_i h_i(w)

这里， :math:`\beta_i` 称为Lagrange multipliers(拉格朗日乘子）。

可以看到，Lagrange函数将优化目标和限制条件融合起来，得到了一个统一的形式。

然后，我们设定 :math:`L` 的偏导为0:

.. math::
    
    \frac{\partial L} 
        {\partial w_i} = 0;

    \frac{\partial L} 
        {\partial \beta_i} = 0;

来求得 :math:`w` 和 :math:`\beta` 。
    

[standford-ml-note] `CS229 Lecture notes: Support Vector Machines`



