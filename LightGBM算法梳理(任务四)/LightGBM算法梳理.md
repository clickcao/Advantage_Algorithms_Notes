# 概念
LigthGBM是boosting集合模型中的新进成员，它和xgboost一样是对GBDT的高效实现，很多方面会比xgboost表现的更为优秀。原理上它和GBDT及xgboot类似，都采用损失函数的负梯度作为当前决策树的残差近似值，去拟合新的决策树。

# lightGBM的优点
1、采用直方图算法
2、树的生长策略优化
3、相对于xgboost和GBDT,LightGBM提出了两个新方法，使得LightGBM的效率要显著要高于GBDT和xgboost。这两种新方法是：Gradient-based One-Side Sampling (GOSS：基于梯度的one-side采样) 和Exclusive Feature Bundling (EFB：互斥的特征捆绑）

# 直方图算法（Histogram）
直方图算法是先把连续的浮点特征值离散化成k个整数，同时构造一个宽度为k的直方图。遍历数据时，根据离散化后的值作为索引在直方图中累积统计量，当遍历一次数据后，直方图累积了需要的统计量，然后根据直方图的离散值，遍历寻找最优的分割点。

它的优点如下：

直方图只需对直方图统计量计算信息增益，相比较于预排序算法每次都遍历所有的值，信息增益的计算量要小很多
通过利用叶节点的父节点和相邻节点的直方图的相减来获得该叶节点的直方图，从而减少构建直方图次数，提升效率
存储直方图统计量所使用的内存远小于预排序算法

# 树的生长策略优化
LightGBM 通过 leaf-wise (best-first)策略来生长树。它将选取具有最大信息增益最大的叶节点来生长。 当生长相同的叶子时，leaf-wise 算法可以比 level-wise 算法减少更多的损失。
当 数据较小的时候，leaf-wise 可能会造成过拟合。 所以，LightGBM 可以利用额外的参数 max_depth 来限制树的深度并避免过拟合（树的生长仍然通过 leaf-wise 策略）。

# Gradient-based One-Side Sampling
GOSS是通过区分不同梯度的实例，保留较大梯度实例同时对较小梯度随机采样的方式减少计算量，从而达到提升效率的目的。

这里有一个问题，为什么只对梯度小的样本进行采样呢？

因为在提升树训练过程中目标函数学习的就是负梯度（近似残差），梯度小说明训练误差已经很小了，对这部分数据的进一步学习的效果不如对梯度大的样本进行学习的效果好或者说对梯度小的样本进行进一步学习对改善结果精度帮助其实并不大。

GOSS的计算步骤如下：

根据样本的梯度将样本降序排序。
保留前n个数据样本，作为数据子集z1。
对于剩下的数据的样本，随机采样获得大小为m的数据子集Z2。
计算信息增益时对采样的Z2样本的梯度数据乘以（1-n）/m（目的是不改变原数据的分布）

# Exclusive Feature Bundling
EFB是通过特征捆绑的方式减少特征维度（其实是降维技术）的方式，来提升计算效率。通常被捆绑的特征都是互斥的（一个特征值为零一个特征值不为零），这样两个特征捆绑起来才不会丢失信息。如果两个特征并不是完全互斥（部分情况下两个特征都是非零值），可以用一个指标对特征不互斥程度进行衡量，称之为冲突比率，当这个值较小时，我们可以选择把不完全互斥的两个特征捆绑，而不影响最后的精度。

EBF的算法步骤如下：

将特征按照非零值的个数进行排序
计算不同特征之间的冲突比率
遍历每个特征并尝试合并特征，使冲突比率最小化