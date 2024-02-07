+++
title = "机器学习之 K 均值聚类"
tags = ["数学"]
categories = ["人工智能"]
series = ["机器学习笔记"]
aliases = ["/posts/af468c0b"]
date = "2022-10-23T08:53:53+08:00"
+++
## 一、引言——非监督学习
之前介绍的算法都是监督学习算法。需要样本中包含各特征信息以及结果标签来让模型学习特征与结果的对应关系。但是，如果我们此时只有特征信息，而不指明具体结果是什么，又需要什么样的算法和模型才能得知哪些样本会有同样的标签呢？

这种从原始数据中探索样本间联系的学习方式称为非监督学习。其中非监督意指，模型不需要人来告知他 “什么是什么”，而能自己从数据中挖掘信息。

## 二、聚类算法
非监督学习中的一种算法为聚类算法。它的目的是将数据按照一定特征进行分组。

最广泛使用的聚类算法称为 K 均值（K-means）算法。

## 三、K均值算法
假设我们要将所有数据分成 k 组，并称在一组中的数据为一个簇。那么对于每一簇内的数据，我们就希望他们之间的距离相对较小，而他们与其他簇中的数据的距离较大。直观上讲，就是同簇的数据点形成一个较为密集的区域，不同簇之间则有较大的空隙（这或许就是聚类的含义）。

我们用簇内的数据的平均作为对簇的描述，相当于指示了簇在特征空间中的位置。这一平均后得到的点称为簇质心。假设我们现在已经得到了一种分组，那么这种分组现在是否有优化的可能呢？要回答这个问题就要查看每一数据点相对于各个簇质心的距离，如果某个数据点相对于其他簇质心的距离要小于其相对于当前所在的簇的质心的距离，就说明该点被错分类了。我们就要将其重新划分为与其距离最小的簇质心所对应的簇内。

但是，这种重新划分又会使得簇质心的位置发生变化，这又导致一些之前是距离该质心位置最小的数据点不再是距离该质心位置最小，又需要再次调整分类。就这样，不断地调整，直到最终所有的点距离当前所在簇的质心距离都为最小，就代表了分类完成。

经过了上面的分析，我们现在给出 K 均值算法的流程：
1. 初始化 k 个簇质心位置，分别对应 k 个簇
2. 对每个数据点，选择与其距离最近的簇质心对应的簇作为其当前分组
3. 对所有同一簇的数据点，计算其平均值，作为新的当前簇的簇质心
4. 不断重复 2、3，直到所有簇的簇质心不再发生变化，即代表分组完成

可以看出，该算法具有很简单的流程，且没有复杂的数据结构。

> 算法中还需要明确两个点：
> - 如何初始化簇质心：
>   一般是随机选取点作为质心，或者在样本数据中随机选取 k 个点
> - 怎么计算数据点到簇质心的距离
>   该算法使用二范数，或者说欧氏距离作为距离

## 四、K均值算法的损失函数
这里我们要更理论化地解释为什么这样的算法可以将数据成功分类

假设总共有 m 个样本数据 $x^{(1)}, x^{(2)}, ..., x^{(m)}$，需要将其分为 k 类（k < m）。我们用 $c^{(i)}$ 表示第 i 个数据点选择第几个簇。用 $\mu_{j}$ 表示第 j 个簇的簇质心。

要衡量当前分组，按照之前所说的 “同簇的数据点形成一个较为密集的区域，不同簇之间则有较大的空隙”，很容易采用簇中数据点相对于质心的距离的平均作为标准。这里直接给出损失函数
$$
    J(c^{(1)}, ..., c^{(m)}, \mu_{1}, ..., \mu_{k}) =
    \frac{1}{m}\sum_{i=1}^m \Vert x^{(i)} - \mu_{c^{(i)}} \Vert^2
$$

该损失函数在这里又叫做失真函数。很明显，失真函数是所有数据点到其对应的簇质心的距离的平方的平均。距离取平方是因为这样并不影响损失函数的衡量效果，而又方便计算。

根据失真函数，我们可以证明上一节算法的正确性。
- 步骤 2 时，对第 i 个样本，我们重新选择 $c^{(i)}$ 使得 $\min\Vert x^{(i)} - \mu_{c^{(i)}} \Vert$ 从而使得 $\min J$。
- 步骤 3 时，我们使 $\mu_j$ 取所有簇内数据点的平均。可以数学证明，此时依旧使得 $\min \Vert x^{(i)} - \mu_{c^{(i)}} \Vert$。
- 这样我们就对失真函数中的每一个参数进行了调整，相当于进行了一次梯度下降。通过不断重复这一过程，最终便可达到使失真函数最小的位置。