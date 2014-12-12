====================================
推荐系统常用模型(2) -- SVD/SVD++
====================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

|today|

研一的课程作业，实现了SVD模型，用的 Netflix 比赛的数据。

把几种常用的模型整理一下。

SVD
-------------
矩阵分解模型将user和item映射到一个共同的潜在空间。

每个user对应一个向量 :math:`p_u` ，每个item对应一个向量 :math:`q_i` ，user和item之间的关系（偏好）就被建模为两个向量的内积 :math:`p_u^T q_i` 。

两种向量可以包含一定的含义，比如，将user的兴趣表示为4维向量，分别代表：

1. 对颜色的偏好
2. 对重量的要求   
3. 设计风格的偏好
4. 价格偏好

相应地，item对应的特点也表示为4维向量:

1. 颜色
2. 重量
3. 设计风格
4. 价格

最终，一一对应，user对item的偏好程度自然就是两个向量的乘积了。
尽管在具体的模型中，向量每个维度的意义并不能人为给定，但是，模型会自己学习包含一些意义来最小化损失。

user :math:`u` 对 item :math:`i` 的打分如下:

.. math::

    \hat{r}_{ui} = \mu + b_i + b_u + q_i^T p_u

其中， :math:`b_i` 和 :math:`b_u` 分别为 item :math:`i` 和 user :math:`u` 的baseline。

损失函数是：

.. math::

    \min_{b_*, q_*, p_*} \sum_{(u,i)\in K} ( r_ui - \mu - b_i - b_u - q^Tp_u)^2 + 
            \lambda (b_i^2 + b_u^2 + ||q_i||^2 + ||p_u||^2).


SVD++
----------

SVD++ 可以说是SVD模型的加强版，除了打分关系，SVD++还可以对隐含的回馈(implicit feedback) 进行建模。

这种隐含的回馈可以是打分动作（谁对某个商品打过分），或者是浏览记录等。
只要有类似的隐含回馈，客观上也表示了user对某个item的偏好。
毕竟，user不会无缘无故地浏览一个item，肯定有什么原因，
比如user喜欢紫色，恰恰这个item也是紫色的，那通过隐含回馈就可以对user对紫色的偏好建模出来。

现实中，隐含回馈的原因比较复杂，专门给一部分参数空间去建模，肯定对用户的建模有一些帮助。

除了在SVD中定义的向量外，每个item对应一个向量 :math:`y_i` ，来通过user隐含回馈过的item的集合来刻画用户的偏好。 

对应的打分公式：

.. math:: 

    \hat{r}_{ui} = \mu + b_i + b_u + q_i^T 
        ( p_u + |R(u)|^{-\frac{1}{2}} \sum_{j\in R(u)} y_i)

其中， :math:`R(u)` 代表user隐含回馈（打分过的）过的item的集合。

可以看到，现在user被建模为 :math:`p_u + |R(u)|^{-\frac{1}{2}} \sum_{j\in R(u)} y_i`

SVD++可以对多种隐含回馈进行建模，比如，打分记录，以及浏览记录

.. math:: 

    \hat{r}_{ui} = \mu + b_i + b_u + q_i^T 
        ( p_u + |R^{(1)}(u)|^{-\frac{1}{2}} \sum_{j\in R^{(1)}(u)} y_i
            + |R^{(2)}(u)|^{-\frac{1}{2}} \sum_{j\in R^{(2)}(u)} y_i )





