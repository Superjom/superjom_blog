梯度检验
=========
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-11*

在用梯度下降法实现一个新的模型时，我们往往需要自己推导梯度求导以及反向更新的实现。
特别像神经网络这类有很多层的模型，梯度和实现都比较麻烦，出错的几率很大。

根据最终的梯度是否下降来判定梯度求导实现错误与否是不靠谱的。

如果求错了或者实现错了，最终的损失会出现一些现象：

1. 损失反而上升
2. 损失先下降，后上升
3. 损失持续下降，最后也可能震荡

如果幸运地出现前两个情况，你就知道梯度部分实现错了。 如果出现第三种现象，那就郁闷了，损失下降，只是最终训练出来的模型是错误的，而且损失比正确的实现高很多。

单个变量的梯度检验
-------------------

.. math::

    \frac{d}{d \theta} J(\theta) 
    \approx 
    \frac{J(\theta + \varepsilon) - J(\theta - \varepsilon)} {2\varepsilon} 

其实就是 $\theta$ 处损失函数导数的近似。 其中，:math:`\varepsilon` 是一个非常小的数值，比如 :math:`10^{-4}`

最终求得的近似值与自己实现的梯度函数求得的值对比，看是否足够接近。

向量系数的梯度检验
--------------------
比如，向量 :math:`\theta = [ \theta_1, \theta_2, \cdots, \theta_n ]`

对应的近似公式是：

.. math::

    \frac{\partial } {\partial \theta_i} J(\theta) = 
    \frac{
        J([ \theta_1, \theta_2, \cdots, \theta_i+\varepsilon, \cdots, \theta_n ]) 
        - 
        J([ \theta_1, \theta_2, \cdots, \theta_i-\varepsilon, \cdots, \theta_n ]) }
    {2 \varepsilon}

其实，就是在求解某个纬度时，固定其他纬度计算。

一段示例代码：

.. code-block:: C++

    typedef vector<double> vec_t;
    const double EPSILON = 0.0001;


    vec_t approx_backp(const vec_t &v) {
        vec_t tmp_v;
        vec_t res_v = v;
        for(int i = 0; i < v.size(); i++) {
            tmp_v = res_v;
            tmp_v[i] += EPSILON;
            double a = J(tmp_v);
            tmp_v[i] -= 2 * EPSILON;
            double b = J(tmp_v);
            res_v[i] = (a - b) / (2 * EPSILON);
        }
        return std::move(res_v);
    }


实现建议
--------
梯度检验应该是每个模型实现(测试)的一部分，一个基本的流程如下：

1. 实现梯度计算函数
2. 实现梯度检验函数
3. 检测，两个函数对同样的参数计算出来的梯度是否足够相近
4. 注视或者关闭梯度检测模块，开始正式的训练

梯度检验足够简单，对于每个模型其实必不可少。 当实现的代码通过了梯度检验，那么在分析结果会更加有底气一点。

References
-----------
Andrew Ng *Machine Learning(week 5) Gradient Checking* from https://class.coursera.org/ml-005/lecture

.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="gradient-check.rst" data-title="梯度检验" data-url="http://superjom.duapp.com/machine-learning/gradient-check.html"></div>
    <!-- 多说评论框 end -->
    <!-- 多说公共JS代码 start (一个网页只需插入一次) -->
    <script type="text/javascript">
    var duoshuoQuery = {short_name:"superjom"};
    (function() {
            var ds = document.createElement('script');
                    ds.type = 'text/javascript';ds.async = true;
                            ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.unstable.js';
                                    ds.charset = 'UTF-8';
                                            (document.getElementsByTagName('head')[0] 
                                                     || document.getElementsByTagName('body')[0]).appendChild(ds);
                                                })();
    </script>
    <!-- 多说公共JS代码 end -->
