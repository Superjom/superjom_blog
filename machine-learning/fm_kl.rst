FM 和概率拟合
==================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-10-19*

Introduction
------------------
最近看了FM和LibFM的代码，感叹作者的思想实在是巧妙，libFM的代码也比较漂亮。

有个预测的需求，里面需要对概率进行预测，可惜这个场景libFM没有支持。

本文做一点FM在分布预测的推导吧。

FM模型
--------
经典格式：

.. math::
    :label: fm

    \hat{y}(x) = \bbox[yellow]{ w_0 + \sum_{i=0}^{n} w_i x_i } + 
                    \bbox[red] {\frac{1}{2} \sum_{i=0}^{n} \sum_{j \neq i} ( x_i v_i)^T (x_j v_j)}

其中，黄色部分是一阶的拟合，红色部分为二阶的拟合。

其中：

* :math:`w_i` 为一阶 weight 
* :math:`x_i` 为输入feature的vector的元素，在libFM中为稀疏向量，计算的时候只需要计算非0的元素
* :math:`v_i` 为对应到feature元素 :math:`x_i` 的 parameter vector ，类似于矩阵分解，FM为每一个feature元素准备了一个vector

FM中比较巧妙的部分就是二阶拟合部分，为feature中任意两者间的关系都进行了建模。 

而且，这部分可以很巧妙地用如下优化方式：

.. math::

    \bbox[red] {\frac{1}{2} \sum_{i=0}^{n} \sum_{j \neq i} ( x_i v_i)^T (x_j v_j)}
    = 
    (\sum_{i=1}^n x_i v_i)^2 - \sum_{i=1}^n (x_i v_i)^2 

这样，就可以在 :math:`O(n)` 时间内计算。

概率映射
---------
FM本身会学习一个实数值，由于其值域无法严格地限定到概率分布的 :math:`(0,1)` ，因此可以利用映射的方式简单地对FM输出值就行归一。

这个可以直接用神经网络激活函数中火热的 **Sigmoid** 映射实现。

损失函数
----------
损失等定义这一块，可以直接采用RMSE便可。

libFM中采用RMSE作为损失函数，这个跟FM原始工作场景是一致的： 拟合打分：1 2 3 4 5 ... 

本文中场景是拟合概率，之所以也用RMSE，而不是交叉熵等主要是，SGD中只能逐条学习纪录，
对于单个纪录，宏观的概率分布的性质无法体现。 

每次都是贪心地拟合单个纪录，这个时候，即使模型输出的值最终会被当作概率去使用，但是客观上也无法做一个宏观的概率考量。

因此，利用交叉熵等作为损失函数是无意义的。


融合模型
---------

所以，总体上的模型就是，FM -> sigmoid -> RMSE

References
-----------
.. [fm] Rendle, Steffen. "Factorization machines." Data Mining (ICDM), 2010 IEEE 10th International Conference on. IEEE, 2010.
.. [libfm] Rendle, Steffen. "Factorization machines with libFM." ACM Transactions on Intelligent Systems and Technology (TIST) 3.3 (2012): 57.


.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="fm_kl.rst" data-title="FM 和概率拟合" data-url="http://superjom.duapp.com/machine-learning/fm_kl.html"></div>
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
