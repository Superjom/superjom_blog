Gibbs sampling in LDA
=========================
输入数据中包含词: :math:`w = \{ w_1, \cdots, w_n\}` , 
每个词 :math:`w_i` 与文档 :math:`d_i` 的归属关系用一个 word-document的矩阵来表示。 

对于每个文档，词的topic服从T(文档的话题个数)的Multinomial分布，
对应的参数是 :math:`\theta^{(d_i)}` 。

所以，对于文档 :math:`d_i` 中的一个词，话题分布是 :math:`P(z_i=j) = \theta_j^{(d_i)}` .

其中，第 :math:

