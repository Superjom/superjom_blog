.. highlight:: python
    :linenothreshold: 5

.. _reservoir-sampling:

蓄水池抽样
================
蓄水池抽样用于在数据比较大的场景下，等概率随机抽样。

常规数据下，都可以很容易知道数据量的大小。

数据量可知
-----------
假设数据量为 :math:`N` ， 需要抽样出 :math:`n` 个数据。

首先计算出每个样本被抽取的概率 :math:`p=\frac{n}{N}` ， 然后随机产生一个随机数 :math:`r = rand()` .

如果 :math:`r < p` ， 则选择之，否则跳过。

伪代码::
    
    P = n / N
    
    def select_or_not(idx):
        r = rand()
        return r < P

    selected_datas = []    

    for index, data in enumerate(dataset):
        if select_or_not(index):
            selected_datas.append(data)
        


数据量未知
-----------
当然，数据量未知，可以利用一次扫描得出数据量。

但是如果数据很大，额外花一轮扫描的时间代价有点高。

利用蓄水池抽样，可以一次扫描得到样本。

原理
*******
蓄水池抽样的算法::
    
    def sample(N, n):

        selected_datas = []  # size is n

        for index, data in enumerate(read_data()):
            if index < n: 
                selected_datas.append(data)

            else:
                r = rand()
                if r < n/index:  
                    # generate a random value between 0 and n
                    rcd2replace_index = rand(0,n)
                    selected_datas[rcd2replace_index] = data

        return selected_datas

首先将前 n 个记录作为抽取的样本，
在扫描到后面第 :math:`d` 个记录时，
当前记录有 :math:`n/d` 的概率被选中，
随机替换之前抽取样本中的一个记录。

证明
******
当 :math:`N \le n` 时，算法中 6,7步会将所有数据作为抽取样本，也就是所有数据有 :math:`100%` 的概率被选。

当 :math:`d > n` 时，在算法 8步之后，用如下操作：
采用概率 :math:`n/d` 随机替换之前抽取样本中的一个记录。

假如，最终第 :math:`d` 个记录被抽取成第 :math:`i` 个样本。 

那么，其概率计算为： (后续记录不被抽取 + 抽取但是没有替换第 :math:`i` 个样本。

计算这个概率：

.. math::
    
    p = \frac{n}{d} \times 
        \prod_{d'=d+1}^N \{
        (1-\frac{n}{d'}) 
            + \frac{n}{d'}\times (1-\frac{1}{n}) 
              \}

        = \frac{n}{d} \times \prod_{d'=d+1}^N \frac{d'-1}{d'}
        = \frac{n}{N}

使用场景
---------
在桶排序中，需要用到抽样。

比如，一大堆的url数据，需要进行全局排序。
数据存储在Hadoop 中的 :math:`m` 个节点。

首先利用蓄水池抽样在每个节点随机抽取 :math:`n` 个记录，
reduce阶段合并，再次抽取 :math:`m` 个记录并排序，相邻两个数据组成的区域就是桶的范围。

这个比单独采用一轮扫描得到最大url和最小url，然后等间隔分桶要好很多：

1. 节约扫描次数
2. 随机抽样能够体现出url的频率，可以单独对于抽样后的样本采用一定方式考虑长尾，比如在分桶中加入频率。 而等间隔分桶在长尾数据下容易爆桶。
