=================================================
推荐系统常用模型(3) -- Time-aware factor model
=================================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

|today|

研一的课程作业，实现了SVD模型，用的 Netflix 比赛的数据。

把几种常用的模型整理一下。

Time changing baseline predictors
------------------------------------
user的偏好总在随着时间而变化，比如看电影，Tom非常喜欢科幻电影，
在一段时间看了很多部大片，忽然觉得有点腻了（也许是因为情节老套等），
接下一段时间他对科幻电影就没有最初的那么大兴趣了。
又或者他开始对悬疑电影产生了兴趣，那接下来他对悬疑电影的打分应该也变高了。

推荐系统无法解释用户偏好变化的原因，但是应该尝试去捕捉和建模用户偏好随时间的变化。

在之前的章节里介绍了Baseline模型，本章节就为Baseline 模型里的参数加上时间因素。

最终的模型如下：

.. math::
    
    b_ui = \mu + b_u(t_{ui}) + b_i (t_{ui})

其中， :math:`b_u(t_{ui})` 和 :math:`b_i (t_{ui})` 分别为 user 和item随时间变化的bias。

下面介绍 :math:`b_u(t_{ui}), b_i (t_{ui})` 的几种建模方法。

Time-changing item biases
****************************
类似 Locally weighted linear regression， 要预测一个时间 t 上的商品的bias，
那么商品最近的打分影响应该大一点。 
比如，某个电影（大话西游）现在突然红起来了，那么最近的打分应该高一点。

将打分记录日期以天为单位，划分等间隔的时间段(bin)。

对于时间 :math:`t` ， 函数 :math:`Bin(t)` 会返回它对应的时间段的号码。

为 item 的每个bin准备一个参数 :math:`b_{i, Bin(t)}` ，最终item 的bias如下 

.. math::

    b_i(t) = b_i + b_{i, Bin(t)}.


User biases
******************

用户部分的建模会稍微复杂一点。 

User biases(1)
++++++++++++++++

最简单的一种方式是，用一个线性函数来捕捉user偏好的渐进变化。

对于user :math:`u` ，将他打分的日期编号平均值定义为 :math:`t_u` 。
如果 :math:`u` 在日期 :math:`t` 打分，那对应的时间偏差定义为：

.. math::

    dev_u(t) = sign(t - t_u) . |t - t_u|^\beta

这里， :math:`t` 是日期的编号， sign是signal函数， 
:math:`sign(t-t_u)` 表示打分日期在平均日期前(-1)还是后(+1)。 
:math:`|t-t_u|` 表示两者相差的天数。 :math:`\beta` 是一个参数，需要通过交叉验证得出，在Netflix数据中取值为 0.4。

加入一个参数 :math:`\alpha_u` ， 对应的user-bias是：

.. math::
    
    b_u^{(1)} (t) = b_u + \alpha_u . dev_u(t)


User biases(2)
+++++++++++++++
对于 user :math:`u` ，得到其打分的集合 :math:`R(u)` ，对于这个打分集合，
类似对 item-bias 的建模，
等间隔划分 :math:`k_u` 个时间段 :math:`-\{ t_1^u, \cdots, t_{k_u}^u\}-` ，然后添加权重。

.. math::

    b_u^{(2)} (t) = b_u + 
        \frac{ \sum_{l=1}^{k_u} e ^{-\sigma |t-t_l^u|} b_{t_l}^u}
            {\sum_{l=1}^{K_u} e ^{-\sigma |t-t_l^u|}}


其中， :math:`\sum_{l=1}^{K_u} e ^{-\sigma |t-t_l^u|}` 起到一个权重的效果，
:math:`|t-t_l^u|` 越小，:math:`b_{t_l}^u` 的权重越大。
:math:`b_{t_l}^u` 不是直接统计得出，而是学习得到的。

linear model
-------------------
组合不同的user和item建模，就可以得到不同的模型。

比如，组合 item-bias 和 user-bias(1) 就可以得到一个线性的模型：

.. math::

    b_{ui} = \mu + b_u + \alpha_u . dev_u(t_{ui}) + b_i + b_{i, Bin(t_{ui})}

对应的目标：

.. math::
    
    \min \sum_{(u,i)\in K}
        (r_{ui} - \mu - b_u - \alpha_u dev_u(t_ui) - b_i - b_{i, Bin(t_{ui})})^2
        + \lambda (b_u^2 + a_u^2 + b_i^2 + b_{i, Bin(t_ui)}^2).


spline model
---------------
对应着，将 item-bias 和 user-bias(2) 结合，就得到了spline model：

.. math::

    b_{ui} = \mu + b_u + \alpha_u 
        \frac{ \sum_{l=1}^{k_u} e ^{-\sigma |t-t_l^u|} b_{t_l}^u}
            {\sum_{l=1}^{K_u} e ^{-\sigma |t-t_l^u|}} + b_i + b_{i, Bin(t_{ui})}





References
--------------
[recommender-system] `Kantor, P. B., Rokach, L., Ricci, F., & Shapira, B. (2011). Recommender systems handbook. Springer.`

