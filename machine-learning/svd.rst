====================================
推荐系统常用模型(1) -- Baseline
====================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

|today|

研一的课程作业，实现了SVD模型，用的 Netflix 比赛的数据。

把几种常用的模型整理一下。

Baseline estimate
-------------------
Baseline 是比较基础的一个模型。
对于用户(user) :math:`u` 和 商品(item) :math:`i` ，预测的打分 :math:`b_{ui}` 如下:

.. math::
    :label: baseline-score

    b_{ui} = \mu + b_u + b_i

其中， :math:`\mu` 是所有商品得分的平均值。 

:math:`b_u` 是 user 所有打分的偏好。 不同的人对商品的打分偏好不同，有的人整体评价高，
那么 :math:`b_u` 就高一点。

类似，:math:`b_i` 是 item 的偏差，如果item比较好，客观上群众们评价整体就高一点，否则就低一点。

比如，item是一台Thinkpad电脑，客观上Thinkpad的口碑不错，因此得分比电脑的平均分3.7高0.5。
然后用户 Jane 是一个公司的采购，她偏爱外形比较漂亮的本儿，对于古板造型的本儿打分就倾向于比平均分低0.3。
如此，Jane 对 Thinkpad的打分就是 :math:`3.7 - 0.3 + 0.5 = 3.9` 。


可以认为 :math:`b_u` 代表user的偏好， :math:`b_i` 代表item的好坏。


利用最小平方和的方法得到优化方向：

.. math::

    \min_{b*} \sum_{(u,i) \in K} 
        (r_{ui} - \mu - b_u - b_i)^2 + \lambda ( \sum_u b_u^2 + \sum_i b_i^2)

其中， :math:`(r_{ui} - \mu - b_u - b_i)^2` 利用 :eq:`baseline-score` 来逼近真实打分 :math:`r_{ui}` 。

后面 :math:`\lambda ( \sum_u b_u^2 + \sum_i b_i^2)` 是正则化项，防止过拟合。

item baseline的计算：

.. math::
    :label: eq-b_i

    b_i = \frac{\sum_{u \in R(i)} (r_{ui} - \mu)}
                {\lambda_1 + | R(i)|}

其中，:math:`R(i)` 表示给item :math:`i` 打分的集合， :math:`|R(i)|` 表示 :math:`R(i)` 的size。

下面的 :math:`\lambda_1` 主要是防止分母为0的正则化参数，考虑到很多商品没有被打过分或者冷启动。

类似，:math:`b_u` 的计算：

.. math::
    :label: eq-b_u

    b_u = \frac{\sum_{i \in R(u)} (r_{ui} - \mu - b_i)}
                {\lambda_3 + | R(u)|}

公式 :eq:`eq-b_i` 和 公式 :eq:`eq-b_u` 很对称。
不同点在于，为了体现用户 :math:`u` 真实的偏好， :eq:`eq-b_u` 中分子部分减去 :math:`b_i` ，
毕竟用户的偏好体现在item的某种特性上（比如上文Jane对漂亮外形的喜爱），
但是与具体的item个体应该是无关的。

比如，上文Jane的例子，假设 :math:`|R(u)|=1, \lambda_3=0` ，如果计算 :math:`b_u` 时不减去 :math:`b_i` ，
然后得到 :math:`b_u = r_{ui} - \mu = 0.2` ，很明显是错误的，按照 :eq:`eq-b_u` 计算正好符合。

