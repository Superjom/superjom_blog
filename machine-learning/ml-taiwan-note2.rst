Machine Learning Foundations 笔记 -- Lesson 2 Learning to Answer Yes/No
==================================================================

机器学习的一个定义
-------------------

机器学习从资料开始，资料是来源于一个未知的分布 :math:`f` (真实的分布函数)

我们将资料提供给一个 `演算法` ，演算法会从所有可能的分布中(假设集 :math:`H` ) 中，选取一个最接近 :math:`f` 的分布 :math:`g` 。

现在面向一个实际的应用 -- 银行的信用卡发放。

当实际开始决策时，我们将申请者的资料喂给 :math:`g` ，然后由 :math:`g` 输出一个决策结果 :math:`y` ，来表示yes/no。

感知器
------
上面有假设集( :math:`H` )这个概念，下面接触一个具体的模型：感知器

对于一个用户的特征 :math:`X = (X_1, X_2, \cdots, X_d)` ， 计算一个加权分

.. math::

    score = \sum_{i=1}^d W_i X_i 

如果 :math:`score > threshold` ，则同意派发信用卡，否则，则拒绝。

.. note::
    
    这里X是申请者的特征，每一纬度代表一份信息，比如age、annual salary可以代表其中两个纬度等。

    而W可以认为是X每个纬度的重要性。

    如果某个特征的纬度对区分度的贡献越大，那么可能此纬度对应的W(绝对)数值越大。

这里的假设集 :math:`H` 被设定为线性公式的样子: :math:`h \in H` 

具体格式是

.. math::

    h(x) = sign \left( \left( \sum_{i=1}^d W_i X_i \right) - threshold \right)

可以对这个公式进行化简：

.. math::

    \begin{split}
    h(x) &= sign \left( \left( \sum_{i=1}^d W_i X_i \right) - threshold \right) \\
         &= sign \left( \left( \sum_{i=1}^d W_i X_i \right) + (-threshold)(+1)\right) \\
         &= sign \left( \sum_{i=0}^d W_i X_i \right) ; X_0 = -threshold
    \end{split}
        
二维空间上的具体情况 
---------------------
为了能够直观地看到假设集的情况，我们可以限定特征纬度为二，从而决策的过程可以映射到一个二维平面空间中。

.. image:: ../_static/image/ml-taiwan-note2-1.png

如图所示，『叉』和 『圆』两种点代表了两个类型： Yes / No

学习/训练的目标就是，找出一条直线能够很好地将这两种点拆分开来。

此时的假设集就是直线：

.. math:: 

    h(X) = sign( W_0 + W_1 X_1 + W_2 X_2)

其中， :math:`W` 就是具体的模型参数，学习的过程就是选择好的 :math:`W` 来控制直线最好地区分两种不同的点。

从假设集上选出 g
------------------
由上面一节，我们可以认为，对于二维特征，假设集就是二维平面上的线。

那么，我们如何定挑选 :math:`g` 的标准呢？ 

我们希望 :math:`g` 尽可能地接近于真实分布 :math:`f` ，那这样就可以要求，
至少在目前已有的资料上， :math:`g` 的预测很接近 :math:`f` 的预测（真实的标签）。

这样，我们就可以用在已有的资料上的效果来评估假设 :math:`g` 了。

但有一个问题是，假设集可能是无穷大的 -- 在二维空间上，有无穷多条线，从哪条线开始，到哪条线结束呢？ 

一种可取的方式就是，从一条线 :math:`g_0` 开始，之后不断修正这条线，不断提升其效果。

感知器学习算法
---------------
感知机的学习始于对错误点的纠正过程。

首先明确错误点：

就是在现有假想 :math:`g` 下，预测错误的点。

也就是

.. math::

    sign \left( W_t^T X_n(t) \right) \neq y_n(t)

纠正(学习)的过程就是

.. math::

   W_{t+1} \leftarrow W_t + y_n(t)X_n(t)

.. note::

    这个个人理解是，宏观上的优化目标应该是 :math:`\max L = W^TXy`
    有点间隔最大化的感觉。 

    这样，求导数， :math:`\frac{\partial L} {\partial W} = Xy`

如此，不断对错误点进行修正更新，直到不犯错误为止。

下面进行算法的具体描述：

对于一个资料 :math:`D` ， 一个初始的参数 :math:`W_0` 

循环进行如下过程：

扫描 :math:`D` 中的每个点，找到下一个预测错误的点 :math:`\left(X_n(t), y_n(t) \right)` ，此时对应的参数为 :math:`W_t`

.. math::
   :label: eq-01

    sign \left( W_t^T x_n(t) \right) \neq y_n (t)

随后对这个错误点进行修正：

.. math::
   :label: eq-02

    W_{t+1} \leftarrow W_t + y_n(t) X_n(t)

循环上面的过程，直到没有错误为止。

那么，下面有一些对于PLA的遗留问题：

1. 上面说在没有错误的时候，算法就会停止，那么算法一定会停止吗？ 
2. 是否会无限循环（总有错误）
3. 在 :math:`D` 上OK，那么在训练集外效果有保障吗?


PLA停止条件的证明
-----------------
PLA的学习过程就是不断纠正错误点的过程，最终在所有点都划分正确的时候停止。

那么，如果给一个无法线性可分的资料，那么算法永远无法停止。

现在假设一个资料是线性可分的-- 存在一条线能够完美地划分资料。

首先，我们探讨下，最终的理想的直线的特性 :math:`W_f` 就是目标直线 :math:`f` 的参数，它能够很好地拆分资料：

由于如果点正好处在直线上的情况是无意义的（现实情况中，比例极小，可以随机化分配），我们可以假设不存在任何点正好处在直线上。

这样，能够得到如下：

.. math::

    \min_n y_n W_f^T X_n > 0

.. note::

   由于是正确拆分，所以 :math:`sign(W_f^T X_n) = y_n` ，所以 :math:`y_n W_f^T X_n \ge 0` 
   再因为所有点都不存在于线上，因此 :math:`>0`

在训练过程中，进行更新 :math:`\left(X_n(t), y_n(t) \right)` 

证明 :math:`W^T_f W_{t+1}` 递增
+++++++++++++++++++++++++++++++++++++++++++

.. math::
   :label: eq-1

   W_f^T W_{t+1} = W_f^T \left(W_t + y_n(t) X_n(t) \right)   

.. note::

   :math:`X_n(t)` 中， :math:`t` 表示第 :math:`t` 轮循环/时刻
   :math:`n` 表示资料中第 :math:`n` 个点

接着化简公式 :eq:`eq-1` ：

.. math::
    :label: eq-2    

    \begin{split}
        W^T_f W_{t+1} & =   & W^T_f \left(W_t + y_n(t) X_n(t) \right) \\
                      & \ge & W_f^T W_t + \min_n y_n W_f^T X_n \\
                      & >   & W_t^T W_t + 0
    \end{split}

式子 :math:`eq-2` 表示， :math:`W^T_f W_{t+1}` 的值（内积）递增。

理想状态里，我们希望直线与目标直线越来越接近，也就是两个向量 :math:`W_f` 和 :math:`W_{t}` 间的角度越来越小。

下面计算两个向量的余弦

.. math:: 
   :label: eq-3
    
   \cos \left( W_f, W_{t+1} \right) 
        = \frac{ W^T_f W_{t+1}}
               { \left|W_f\right| \left|W_{t+1}\right| }

证明 :math:`\left| W_f \right| \left|W_{t+1}\right|` 递减
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++

PLA中，只对错误点进行更新。 所以，上一节中的更新，只发生在预测错误点上。

对于错误点上的情况，有一些性质可以挖掘。

从式子 :eq:`eq-01` 可以得到在错误点上：

.. math::
   :label: eq-4

   y_n(t) W_t^T X_n(t) \le 0

在错误点上计算

.. math::
    :label: eq-5 

    \begin{split}
        \| W_t \| ^ 2 
            &= &
                 \| W_t + y_n(t) X_n(t) \|^2 \\
            &= &
                 \| W_t \|^2
                 + 
                    \bbox[blue]{2 y_n(t) W_t^T X_n(t)} +
                 \| y_n(t) X_n(t) \|^2
    \end{split}

由式子 :eq:`eq-4` 可知，上面式子 :eq:`eq-5` 的蓝色部分是不大于0的。

接着化简式子 :eq:`eq-5`

.. math::
   :label: eq-6

   \begin{split}
     \| W_t \|^2
     + \bbox[blue]{2 y_n(t) W_t^T X_n(t)} +
     \| y_n(t) X_n(t) \|^2  
      & \le &
     \| W_t \|^2
     + \bbox[blue]{0} + \bbox[red]{\| y_n(t) X_n(t) \|^2} \\
     & \le &
     \| W_t \|^2
     + \max_n \| X_n \|^2
    \end{split}

.. note::

    这里注意， :math:`y_n = +1/-1`

综合 :eq:`eq-5` 和 :eq:`eq-6` ，可以知道

.. math::
   :label: eq-7

   \| W_{t+1} \|^2 \le \| W_t \|^2 + \max_n \| X_n \|^2


证明 :math:`\frac{W_f^T}{\|W_f\|} \frac{W_T}{\| W_T\|}` 递增
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Lecture里一个现成的结论是
    
.. math::
   :label: eq-10
    
    \frac{W_f^T}{\|W_f\|} \frac{W_T}{\| W_T\|}
    \ge \sqrt{T}  constant

下面是一些证明

.. note::
    
    上面有出现 :math:`\max_n{X_n}` ，表示离目前直线 :math:`g_t` 最远的点。

    假设每次更新，我们都用离线最远的来进行更新，理论上纠正的力度会更大。 

    如果PLA最终会收敛，那么注定，以这种方式也能够收敛。

    为了书写方便，设定 :math:`X = \max_n{X_n}`

    下面的计算用到前面两个式子的结论 :eq:`eq-02` 和 :eq:`eq-7`

    这里设置 :math:`W_0 = \{0\}`

    .. math::
       :label: eq-8

        \begin{split}
        \frac{W_f^T}{\|W_f\|} \frac{W_T}{\| W_T\|}
             & \ge &
                 \frac{\left( T y X \right) W_f}
                      { \|W_f\| \sqrt{ T\|X\|^2}} \\
             & \ge & 
                 \frac{\left(  T y X \right) W_f}
                      { \|W_f\| \left(  \sqrt{T}\|X\| \right) }  \\
             & = &
                 \frac{\left( T y X \right) W_f}
                    { \|TyX\| \|W_f\|}
                 \frac{ \|TyX\|}
                    { \sqrt{T} \|X\|} \\
             & = &
                 \frac{\left( y X \right) W_f}
                    { \|yX\| \|W_f\|}
                 \frac{ \|TyX\|}
                    { \sqrt{T} \|X\|}
        \end{split}

    上面式子 :eq:`eq-8` 最后计算得到两个部分，其中，前一部分是一个常数:

    .. math::
       :label: eq-18

       constant = \frac{\left( y X \right) W_f}
                    { \|yX\| \|W_f\|}

    constant 实际上就是跟 :math:`X` 和 :math:`W_f` 之间斜率的cos有关。 

    这里 :math:`X` 可能有一定的范围要求，比如最大或者最小等，这里不加论证。

    下面计算后一部分:

    .. math::

        \lim_{T \rightarrow \infty} 
            \frac{ \|TyX\|}
                { \sqrt{T} \|X\|}
            = \sqrt{T}

    合并起来，证明完毕。

这一节就证明了，:math:`W_T` 会不断接近于理想模型 :math:`W_f`

每次接近一点，那么最终PLA算法就会停下来。 

确定 PLA 学习轮数的上限定
+++++++++++++++++++++++++
设定 

.. math:: 
    
   \begin{split}
       R^2      & = &   \max_n \| X_n \|^2 \\
       \rho     & = &   \min_n y_n \frac{W_f^T}{\|W_f\|} X_n
   \end{split}

那么算法会在不大于 :math:`R^2/ \rho^2` 次循环后停止。

.. note::

   结合 :eq:`eq-18` 和 :eq:`eq-10` 可以得到，

   .. math::
    
    \begin{split}
      1 \ge \frac{W_f^T}{\|W_f\|} \frac{W_T}{\| W_T\|}
        \ge \sqrt{T} constant
    \end{split}
       

不可分的数据
-------------
上面章节，我们证明了，如果满足两个条件， PLA能够停下来

1. 资料线性可分
2. :math:`W_t` 在错误点上不断被纠正

尽管是二维空间，但是PLA在多维数据上的行为是类似的。 

那么，如果资料不可线性分开呢？ 

上一节里所有的结论都是依赖于资料可分的前提。

所以，如果你发现，在一个资料上PLA跑了很久还没有停止，那么有两个可能的原因：

1. 算法还没有训练充足
2. 此资料无法线性分开

针对资料无法线性分开（比如，存在一定的噪音）的情况，PLA的一个简单的变种可以解决

具体算法如下：

* 初始化模型参数

1. 在目前的模型下，随机选取一个错误点 :math:`\left( X_n(t), y_n(t)\right)`
2. **尝试** 更新一下参数

.. math::

   W_{t+1} \leftarrow W_t + y_n(t) X_n(t)

3. 尝试用新的参数构成的模型对整个资料进行预测，如果正确率提高，则采纳当前参数 :math:`W_{t+1}` ， 否则，还原到原始的参数。

* 重复1,2,3，最终在足够多次的循环后，返回 :math:`W` 的模型 :math:`g`

使用这种方法，并不需要强求资料完全线性可分。 但是，效率非常低，主要是需要不断尝试预测。


References
-----------
Coursera 机器学习基石 第二课 https://www.coursera.org/course/ntumlone

.. raw:: html

    <script>window._bd_share_config={"common":{"bdSnsKey":{},"bdText":"","bdMini":"2","bdMiniList":["qzone","tsina","weixin","renren","tqq","sqq","hi","youdao"],"bdPic":"","bdStyle":"0","bdSize":"16"},"slide":{"type":"slide","bdImg":"5","bdPos":"left","bdTop":"159"}};with(document)0[(getElementsByTagName('head')[0]||body).appendChild(createElement('script')).src='http://bdimg.share.baidu.com/static/api/js/share.js?v=89860593.js?cdnversion='+~(-new Date()/36e5)];</script>


.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="gradient-check.rst" data-title="Machine Learning Foundations -- Lesson 2 Learning to Answer Yes/No" data-url="http://superjom.duapp.com/machine-learning/ml-taiwan-note2.html"></div>
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



