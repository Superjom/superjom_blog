word2vec 代码实现(1) -- Skip gram
===================================
看了一下gensim的代码，python非常简明易懂。

下面将一些核心的算法记录一下。

.. note::

    python代码只为表示算法原理，与源代码有一些差别，不能直接运行。

Skip gram
--------------

Negative sampling
*******************
初始化数据：

.. code-block:: python
    :linenos:

    # 设置label 列表
    # 第一个为正例，其余为负例
    labels = zeros(k + 1)
    labels[0] = 1.0

    # 模型隐层
    self.syn1neg = zeros( (len(self.vocab), self.layer1_size), dtype=REAL)

    # 初始化 word embedding matrix
    for i in xrange(len(self.vocab)):
        self.syn0[i] = (random.rand(self.layer1_size) - 0.5) / self.layer1_size

开始训练：

.. code-block:: python
    :linenos:
    :emphasize-lines: 21-25

    for pos, word in enumerate(sentence):
        # 产生一个随机的窗口长度差值
        reduced_window = random.randint(model.window)
        # end 表示随机window长度的终点
        end = pos + model.window + 1 - reduced_window
        for pos2, word2 in enumerate(sentence[start: end]):
            # 目标词的 vector
            l1 = model.syn0[word2.index]
            neu1e = zeros(l1.shape)

            word_indices = [word.index]
            while len(word_indices) < k+1:
                # 随机抽取噪音词
                w = model.table[random.randint(model.table.shape[0])]
                if w != word.index:
                    # 最终 word_indices 中会包含 1个正例， k个负例
                    word_indices.append(w)
            # 所有样本的hidden vector
            l2b = model.syn1neg[word_indices]
            # 计算神经网络的active value，映射到(0,1) 概率
            fb = 1. / (1. + exp(-dot(l1, l2b.T)))
            gb = (labels - fb) * alpha
            model.syn1neg[word_indices] += outer(gb, l1)
            neu1e += dot(gb, 12b)
            model.syn0[word2.index] += neu1e


上面代码的意思是，在句子里随机选取一定间隔内的两个word :math:`w_1, w_2` 。
其中，用 :math:`w_1` 来预测 :math:`w_2` 的出现。 

首先，模型中有两个矩阵 :math:`V, H` ， 其中 :math:`V` 是词向量构成的矩阵， 
:math:`H` 是hidden vector构成的矩阵，两者的长度均为  :math:`|V| \times l` ，
其中 :math:`|V|` 代表词库长度， :math:`l` 代表layer长度。

每个词都会在 :math:`H, V` 中包含一列，分别对应着词的 hidden vector 和词向量。

在预测是，挑选 :math:`w_1` 为正例 :math:`(label=+1)` ，
同时，在词库中随机挑选 :math:`k` 个负例 :math:`(w^{(1)}, w^{(2)}, \cdots, w^{(k)})` ， 
负例的 label 均为 0.

将label理解为联合分布的概率，那么label=1 表示两个词共同出现，而label=0表示两个词不共同出现。

将激活概率设为 :math:`\sigma(x)=\frac{1}{1+e^{-x}}` ，正好可以将 :math:`x` 映射到 :math:`(0,1)` 的概率。 对于正例， 希望 :math:`\sigma` 的值接近于1.

输入抽取的样本和label，得到Log似然函数：

.. math::
    :label: loss

    Loss = - \log \sigma(h_{w_1}^T.v_{w_2})
        - \sum_{i=1}^k \log \sigma(-h_{w^{(i)}}^T.v_{w_2})

这里， :math:`h_{w_*}` 表示 :math:`w_*` 的 hidden vector， :math:`v_{w_*}` 表示 :math:`w_*` 的词向量. 

注意 :math:`1-\sigma(x) = \sigma(-x)` , 因此，在 :eq:`loss` 中， 
负例的概率不出现的概率被表示为: :math:`\sigma(-h_{w^{(i)}}^T.v_{w_2})`

最终的目标是最小化 Loss，采用随机梯度下降。

设定正例 :math:`w_1 = w^{(0)}`

正例的梯度如下：

.. math::

    \begin{split}
    \frac{\partial Loss} 
        {\partial h_{w_1}}
    = & - \frac{v_{w_2}} {1+ e^{h_{w_1}^T.v_{w_2}}} \\
    = & - (1 - \sigma(h_{w_1}^T.v_{w_2}))v_{w_2} \\
    = & - (label_{w_1} - \sigma(h_{w_1}^T.v_{w_2}))v_{w_2}
    \end{split}

类似的，负例的梯度如下：

.. math::

    \begin{split}
    \frac{\partial Loss} 
        {\partial h_{w^{(.)}}}
    = & - (1 - \sigma(-h_{w^{(.)}}^T.v_{w_2}))v_{w_2} \\
    = & - (label_{w^{(.)}} - \sigma(h_{w^{(.)}}^T.v_{w_2}))v_{w_2}
    \end{split}


设定学习参数为 :math:`\alpha` ，那么统一样本的更新公式是：

.. math::
    
    \begin{split}
    h_{w^{(.)}} & := h_{w^{(.)}} - \alpha 
                    \frac{\partial Loss} 
                        {\partial h_{w^{(.)}}} \\
                & := h_{w^{(.)}} 
                    + \alpha (label_{w^{(.)}} - \sigma(h_{w^{(.)}}^T.v_{w_2}))v_{w_2}
    \end{split}

这个公式就对应着上面代码的21-23行。  


同时，对于 :math:`v_{w_2}` 的梯度如下:

.. math::
    
    \begin{split}
        \frac{\partial Loss} 
            {\partial v_{w_2}}
        = & - (1 - \sigma(h_{w_1}^T.v_{w_2}))h_{w_1}
             - \sum_{i=1}^k (1 - \sigma(-h_{w^{(i)}}^T.v_{w_2}))(h_{w^{(i)}}) \\
        = & - \sum_{i=0}^k (label_{w^{(i)}} - \sigma(-h_{w^{(i)}}^T.v_{w_2}))h_{w^{(i)}}
    \end{split}

具体的更新公式是:

.. math::

    v_{w_2} & := h_{w^{(.)}} - \alpha 
                    \frac{\partial Loss} 
                        {\partial v_{w_2}} \\
                & := v_{w_2} + \alpha
                     \sum_{i=0}^k (label_{w^{(i)}} 
                     - \sigma(-h_{w^{(i)}}^T.v_{w_2}))h_{w^{(i)}}

这个就对应着上面代码第25行。


Hierarchy Tree
*****************
首先，利用词频建立一棵Haffman树


.. code-block:: python
    :linenos:

    # 利用堆排序根据词频从大到小排列整个词库 
    heap = list(itervalues(self.vocab))
    heapq.heapify(heap)
    for i in xrange(len(self.vocab) - 1):
        min1, min2 = heapq.heappop(heap), heapq.heappop(heap)
        # index 是 hidden vectors 的索引（树的中间节点对应的hidden vector)
        heapq.heappush(heap, 
            Vocab(count=min1.count + min2.count, index=i 
                + len(self.vocab), left=min1, right=min2))

    # 建立好Haffman树之后，为每个词设定一个0,1 Haffman编码
    if heap:
        max_depth, stack = 0, [(heap[0], [], [])]
        while stack:
            node, codes, points = stack.pop()
            if node.index < len(self.vocab):
                # leaf node => store its path from the root
                node.code, node.point = codes, points
                max_depth = max(len(codes), max_depth)
            else:
                # inner node => continue recursion
                points = array(list(points) + [node.index - len(self.vocab)], dtype=uint32)
                stack.append((node.left, array(list(codes) + [0], dtype=uint8), points))
                stack.append((node.right, array(list(codes) + [1], dtype=uint8), points))

在Haffman树中，会包含两类节点： 中间节点(包含 hidden vector） 和叶子节点(词)。

和 Negative sampling 方法类似，也会包含两个矩阵， :math:`V , H` ，具体的含义相同。

训练过程：

.. code-block:: python
    :linenos:

    for pos, word in enumerate(sentence):
        reduced_window = random.randint(model.window)  
        start = max(0, pos - model.window + reduced_window)

        for pos2, word2 in enumerate(
                sentence[start : pos + model.window 
                                + 1 - reduced_window], start):
            if word2 and not (pos2 == pos):
                # 取得 word2 的词向量
                l1 = model.syn0[word2.index]
                neu1e = zeros(l1.shape)
                # 取得 word2 附近一个词 word 对应的中间节点的 hidden vectors
                l2a = deepcopy(model.syn1[word.point])  
                fa = 1.0 / (1.0 + exp(-dot(l1, l2a.T)))  
                # 这里 code 是一个Haffman编码
                # 对应着 Negative sampling 中正例负例的label（0,1）
                ga = (1 - word.code - fa) * alpha  
                model.syn1[word.point] += outer(ga, l1)  
                neu1e += dot(ga, l2a) # save error

可以看到， hierarchy tree 的方法和Negative sampling的方法是想通的： 

1. 两者均有 :math:`H,V` ，其中，Negative sampling 中为每个词对应一个 :math:`h` ，而Hierarchy tree 中为每个中间隐藏节点对应一个 :math:`h`
2. 两者均包含有01标注，在Negative sampling 中抽样，对正例标注为1，对负例标注为0；而Hierarchy tree 中，每个词对应一个binary code对应的隐藏节点的路径，向左走是0,向右走为1
3. 两者的训练都可以认为是通过多次二元分类，在Negative sampling中，最终目标是求解一个正负样本和目标词的共现性01二元分类。 而Hierarchy Tree 中，将目标词word2附近的词word对应的词向量沿着word2对应的路径往前走，最终到达word节点，中间的Haffman code就是需要预测的0,1标签。

计算也是很相似，此处就不重复了。
