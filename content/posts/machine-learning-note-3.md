+++
title = "机器学习笔记之特征缩放"
tags = ["数学", "Python"]
categories = ["人工智能"]
series = ["机器学习笔记"]
aliases = ["/posts/9d18f94a"]
date = "2022-10-01T17:16:30+08:00"
+++
## 一、引言——参数数值对权重的影响
考虑有两个特征的房价预测。其一为房子面积，其二为卧室数量。参数的范围为
$$
    x_1 \in [300, 2000] \\
    x_2 \in [0, 5]
$$
预测直线为
$$
    y = w_1x_1 + w_2x_2 + b
$$
从感性上认知，如果 $w_1$ 的数值大于 $w_2$，那么由特征 $x_1$ 贡献的房价就会远大于特征 $x_2$。因为这样的话，对于同等数值的变化，特征 $x_1$ 的贡献变化大于 $x_2$，而 $x_1$ 的范围又更大，则其对贡献的变化也会更加得大。这是很不合常理的。因此从感性上讲，$w_1$ 的数值应该小于 $w_2$。

另外，根据损失函数的定义
$$
    J(\vec{w}, b)  = \frac{1}{2m}\sum_{i=1}^{m}(\vec{w}\cdot\vec{x}^{(i)} + b - y^{(i)})^2
$$
当损失函数值固定时，改变权重 $w_1$, 则对应的权重 $w_2$ 的变化要大于 $w_1$。从图像来说，此时损失函数形成陡峭的“山谷”，它的等高线图类似于椭圆。这时进行梯度下降，在步长较大时可能出现在“崖壁”上来回跳跃的情况。

![contours](contours.png)

为了避免这种情况，我们需要找到一种优化的方法。

## 二、特征缩放
我们要做的是避免出现不同特征的范围差距较大的情况。那么对于极差较大的特征，我们需要将其数值所在的区间范围缩小。这就是特征放缩。

在进行过特征放缩后，损失函数从图像上看将会较原来更接近正圆形，这样的话，寻找通向最低点的路径将会更加容易。

特征缩放有许多不同的方法，如：
1. 除数特征缩放
2. 均值归一化（Mean normalization）
3. Z-score标准化（Z-score normalization）

除数特征缩放指的是将该特征的所有值同除以某一个数，比如说最大值。
$$
    x_{scaled} = \frac{x}{x_{max}}
$$

均值归一化是以均值为参照对所有数值进行缩放，公式：
$$
    x_{scaled} = \frac{x - x_{mean}}{x_{max} - x_{min}}
$$

Z-score标准化将数值转换为正态分布。公式：
$$
    x_{scaled} = \frac{x - \mu}{\sigma}
$$

其中
$$
    \mu = \frac{1}{m}\sum_{i = 1}^m x^{(i)} \\
    \sigma^2 = \frac{1}{m}\sum_{i = 1}^m (x^{(i)} - \mu)^2
$$

## 三、代码实现
利用 Z-score 标准化对特征进行缩放
```python
def featureScaling(X):
    mu = np.mean(X, axis=0)
    # mean 方法求平均值，其中 axis 参数指定对哪一个维度求平均。

    sigma = np.std(X, axis=0)
    # std 方法用来求标准差

    X_scaled = (X - mu) / sigma
    # 直接利用运算符，X中的每一组样本，减去 mu， 除以 sigma。

    return X_scaled, mu, sigma
```

多元线性回归代码，在初始化 `X_train` 变量的代码后添加
```python
X_train, mu, sigma = featureScaling(X_train)
```

并将步长改为 0.01（$0.01 \gg 1 \times 10^{-7}$ ！！！）
```python
alpha = 0.01
```

因为采用的是经过缩放后的特征值，所以回归之后得到的权重也和原来的权重不同。为了检验代码的正确性，在代码末尾增加语句：
```python
for i in range(X_train.shape[0]):
    print(f"predict:{np.dot(X_train[i], w) + b}, actual:{y_train[i]}")
```

执行代码，在迭代次数较多时能明显感到执行速度变快。最终结果：
![result](result.png)
说明梯度下降后得到的权重能够比较好的拟合样本数据

损失函数值随着迭代次数增加而下降。
![cost](cost.png)