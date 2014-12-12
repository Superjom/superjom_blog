================
拉格朗日对偶
================
在SVM的推导里面，会用到 Lagrange duality. 
这一章专门回顾一下这个知识点。

如果有这样一个优化问题：

.. math::
    
    \begin{split}
    \min_w f(w) & \\
        s.t.    & h_i(w) = 0, i=1, \cdots, l
    \end{split}

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

这个的解释是，假如在一个空间里，:math:`f(x)` 的值会形成一些等高线，而 :math:`h_i(x)=0` 代表某一维度上的一条曲线，
那么只有这条曲线的切线与 :math:`f(x)` 的等高线相切时，才能够保持稳定的极值。

扩展到有不等式限制关系的情况：

.. math::
    
    \begin{split}
        \min_w  &   f(w) \\
        s.t     &   g_i(w) \lt 0, i=1, \cdots, k \\
                &   h_i(w) = 0, i=1, \cdots, l.
    \end{split}

组合起来，形成 Lagrange函数：

.. math::
    
    L(w, \beta) = f(w) + \sum_{i=1}^k \alpha_i g_i(w) + \sum_{i=1}^l \beta_i h_i(w)

类似地， :math:`\alpha_i` 和 :math:`\beta_i` 是Lagrange乘子。

下面讨论如何将限制条件融入到Lagrange函数中去。

通过如下这个形式：



通过如下这个形式：

.. math::
    
    \theta_p(w) = \max_{\alpha, \beta: \alpha_i \lt 0} 
        f(w) + \sum_{i=1}^k \alpha_i g_i(w) + \sum_{i=1}^l \beta_i h_i(w)

        = \infty

通过定义上面的形式，可以很好地融入限制条件。

比如： 

    1. 如果 :math:`g_i(w) > 0` ， 可以调整 :math:`\alpha_i` ， 使得 :math:`\alpha_i g_i(w) \rightarrow \infty`  
    2. 如果 :math:`h_i(w) \neq 0` ， 可以调整 :math:`\beta_i` ， 使得 :math:`\beta_i h_i(w) \rightarrow \infty`

所以，发现有如下现象：

.. math::
    :label: theta_p
    
    \theta_p(w) = 
        \begin{cases}
            \begin{split}
                f(w)    &   if \quad w\quad  satisfies\quad  primal\quad  constraints \\
                \infty  &   otherwise
            \end{split}
        \end{cases}

如此，我们取

.. math::
    :label: min-max 

    \min_w \theta_p (w) = \min_w \max_{\alpha, \beta: \alpha_i \lt 0} L(w, \alpha, \beta)

总是可以让 :math:`L(w, \alpha, \beta)` 满足限制条件，因为此时， :math:`\min_w \theta_p (w) = f(w) < \infty` 是满足限制条件下才能取得的。

下面定义一个上界.

定义这样的函数：

.. math::
    
    \theta_D(\alpha, \beta) = \min_{w} L(w, \alpha, \beta).

取得 :math:`\min \max L(w, \alpha, \beta)` 的一个对偶优化问题：

.. math::
    :label: max-min
    
    \max_{\alpha, \beta: \alpha \lt 0} \theta_D (\alpha, \beta) 
        = \max_{\alpha, \beta: \alpha \lt 0} \min_{w} L(w, \alpha, \beta)

:eq:`max-min` 和 :eq:`min-max` 的关系表示如下：

.. math::
    
    d^* = \max_{\alpha, \beta: \alpha \lt 0} \min_{w} L(w, \alpha, \beta)
        \lt 
        \min_w \max_{\alpha, \beta: \alpha_i \lt 0} L(w, \alpha, \beta)
            = p^*

对于 max-min 和 min-max 的关系，可以很形象地解释为： *巨人中的矮子总不比矮子中的巨人矮。*

所以，原始的优化目标找到了一个下界 :math:`d^*` 。

结合公式 :eq:`theta_p` 可知，:math:`d^* \lt f(w) = p^*` ，
大牛们告诉我们，在满足KKT条件时， :math:`d^* = p^*` ，也就是最优解。

KKT条件：

.. math::

    \begin{split}
    \frac{\partial} {\partial w_i} L(w^*, \alpha^*, \beta^*) & = & 0 \\
    \frac{\partial} {\partial \beta_i} L(w^*, \alpha^*, \beta^*) & = & 0 \\
    \alpha_i^* g_i(w^*)     & =     & 0 \\
    g_i(w^*)                & \lt   & 0 \\
    \alpha^*                & \gt   & 0
    \end{split}














    

[standford-ml-note] `CS229 Lecture notes: Support Vector Machines`
