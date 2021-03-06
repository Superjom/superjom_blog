巨量链接排序
===============
.. sectionauthor:: Superjom <yanchunwei@outlook.com>

*2014-7-13*

如果有一大堆的链接数据(>60T)需要排序，用hadoop，如何处理，需要考虑到长尾。

这是之前在百度实习的时候，遇到的一个问题，现在觉得分桶什么的确实比较有趣，下面记录一下做法。

分桶排序
---------
对这么大的数据排序，用hadoop，自然就是分桶排序了。 

分桶排序就是预先将原始数据切割成全局有序的分块（局部无序），然后在各个分块分别进行局部排序。
这样最终的数据就是全局有序的了。 

比如::

    数据： a,d,f,b,e,c
    分割为：(b, a) (c,d), (f,e)  # 注意分块的全局有序性
    然后把这三块数据扔到3个机器上排序，最终得到:
    (a,b) (c,d) (e,f)

长尾
-----
这么大的数据，由N个节点保存。
链接数据有一个特性，就是长尾，也就是其中有一些链接的数目非常巨大(比如一些门户网站的地址作为tourl时，重复度非常大）。

在分桶的时候，如果只按照范围来分桶，其中会有多个重复度巨大链接超过单个节点的存储空间，直接导致爆桶。

比如::

    数据： a,a,a,a,a, d, f, b, e, c
    直接按照范围分割为：(b, aaaaa), (c,d), (f,e)
    如果一个节点只能存储两个元素呢？ 第一个节点就爆掉了


利用概率的解决方法
-------------------
解决方法集中在对长尾的处理上，当时就想尝试按照一定的概率，将重复度大的url分割到多个桶里。

对于数据 a,a,a,a,a, d, f, b, e, c，能否分桶成如下::

    a,a
    a,a
    a,b
    d,c
    f,e

上面是一个最优的分配方式，也就是将重复度高的a分到三个桶里。

将相同的链接要分配到不同的桶里，就需要用概率和随机数了。

就用上面的数据，就看前三个桶，讨论a分配到前三个桶的情况::

    1th: x==a      2/5      
    2th: x==a      2/5
    3th: x==a      1/5

上面的意思是，a各有2/5的概率分配到第1个和第2个桶，有1/5的概率分配到第3个桶。

利用随机数::

    1th: x==a      if rand < 2/5      
    2th: x==a      if rand < 4/5
    3th: x==a      if rand < 1

当分配a时，给定一个0-1间的随机数 rand, 有下面三种情况：

1. 如果rand<2/5，那么第一个桶
2. 否则如果 rand<4/5，那么第二个桶
3. 否则，第三个桶

如此，解决方法就差不多清楚了。

抽样
*********
如果要检测长尾链接，对这么大的数据用hashmap计数？ 几乎不可能。

那处理的方式就是蓄水池抽样了，参看 :ref:`reservoir-sampling` 。

可以每个节点抽样n个，最后汇总排序，得到一个比较小的数据集。

切割和分桶
***********
对抽样的小数据集进行排序（现在应该单个节点能够应付）。

要分成N个桶，就切割N块，对这个小数据集进行扫描，利用如上类似的作法。

一个稍微大点的数据（抽样数据已经排序）::

    a,a
    a,a
    a,b
    b,c
    d,f

为了表示简便，首先对长尾数据进行分桶，对于中间的正常链接，按照范围分配便可。

长尾链接
+++++++++
首先是对长尾数据，最后指定到如下一个表::

    a: 
        1th         rand<=0.4
        2th         rand<=0.8
        3th         rand<=1.0 
    b:
        3th         rand<=0.5
        4th         rand<=1.0

得到链接的时候，对应查表，如果是长尾链接，就对应着取一个随机数决定分桶。

常规链接
+++++++++
然后是非长尾数据::
    
    (a,b)   4th          
    (b,c)   5th
    (d,f)   6th

中间的(a,b)表示由a到b中间的范围，如果当前链接不是长尾链接，对着范围，直接分到对应的桶便可。


map阶段分桶完毕，Reduce阶段就开始排序吧。 
