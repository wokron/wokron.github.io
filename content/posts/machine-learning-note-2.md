+++
title = "机器学习笔记之多元线性回归"
tags = ["数学", "Python"]
categories = ["人工智能"]
series = ["机器学习笔记"]
aliases = ["/posts/2cfef6f1"]
date = "2022-09-26T14:52:30+08:00"
+++
## 一、引言——多特征的房价预测
现实中，某一变量并不一定只与单一变量有关。还以房价来举例，除了位置以外，房价还可能与房子面积、卧室数量、层数、房龄等因素有关。

|面积|卧室数|层数|房龄|房价|
|:---:|:---:|:---:|:---:|:---:|
|2104|5|1|45|460|
|1416|3|2|40|232|
|1534|3|2|30|315|
|852|2|1|36|178|
|...|...|...|...|...|

对于多个特征的情况，我们需要对之前的一元线性回归进行推广。但在此之前，需要先约定一下使用的符号：

和前面一样，这里将用 $x^{(i)}$ 表示第 i 组样本中的特征。 $y^{(i)}$ 表示第 i 组样本对应的目标结果。不同的是，对于第 j 个**特征组**，也就是第j个特征对应的数值的序列，采用 $x_{j}$ 表示。那么很自然的， $x^{(i)}_j$ 表示第 j 个特征中的第 i 个值，或者说表示第 i 组样本中的第 j 个特征。

> 与上文中的列表相结合进行理解，相当于 $x^{(i)}$ 表示第 i 行中非房价的部分，$x_{j}$ 则表示第 j 列中的数值。

要对多元特征进行拟合，也就是找到适当的 $w_1, w_2, ..., w_n$ 使得对任意的 i，有
$$
    y^{(i)} = w_1 * x^{(i)}_1 + w_2 * x^{(i)}_2 + ... + w_n * x^{(i)}_n + b
$$
设
$$
    \vec{w} = (w_1, w_2, ... w_n)
$$

$$
    \vec{x}^{(i)} = (x^{(i)}_1, x^{(i)}_2, ..., x^{(i)}_n)
$$
则原等式可以表示为
$$
    y^{(i)} = \vec{w}\cdot\vec{x}^{(i)} + b
$$


## 二、损失函数
假设给定 $\vec{w}, b$ 。对样本信息 $\vec{x}^{(i)}$ ，有对房价的预测
$$
    \hat{y}^{(i)} = \vec{w}\cdot\vec{x}^{(i)} + b
$$
与一元线性回归时同理，损失函数
$$
    J(\vec{w}, b) = \frac{1}{2m}\sum_{i=1}^{m}(\hat{y}^{(i)} - y^{(i)})^2 = \frac{1}{2m}\sum_{i=1}^{m}(\vec{w}\cdot\vec{x}^{(i)} + b - y^{(i)})^2
$$

## 三、梯度下降
对于损失函数，求对 $w_j$ 的偏导数：
$$
    \frac{\partial{J(\vec{w}, b)}}{\partial{w_j}} = \frac{1}{m}\sum_{i=1}^{m}(\vec{w}\cdot\vec{x}^{(i)} + b - y^{(i)})x^{(i)}_j
$$

对 $b$ 的偏导数：
$$
    \frac{\partial{J(\vec{w}, b)}}{\partial{b}} = \frac{1}{m}\sum_{i=1}^{m}(\vec{w}\cdot\vec{x}^{(i)} + b - y^{(i)})
$$

则每次迭代计算
$$
    w_j = w_j -  \alpha \frac{\partial J(\mathbf{w},b)}{\partial w_j}
$$

$$
    b = b -  \alpha \frac{\partial J(\mathbf{w},b)}{\partial b}
$$

最终就可得到拟合样本数据的多元线性方程。
## 四、矩阵形式
上述公式也可以表示为矩阵形式，设
$$
    Y = \begin{bmatrix}
        y^{(1)}&y^{(2)}&\cdots&y^{(m)}
    \end{bmatrix}^T
$$
$$
    W = \begin{bmatrix}
        w_1&w_2&\cdots&w_n
    \end{bmatrix}^T
$$
$$
    X = \begin{bmatrix}
        x^{(1)}_1 & x^{(1)}_2 & \cdots & x^{(1)}_n \\
        x^{(2)}_1 & x^{(2)}_2 & \cdots & x^{(2)}_n \\
        \vdots&\vdots& \ddots & \vdots \\
        x^{(m)}_1 & x^{(m)}_2 & \cdots & x^{(m)}_n
    \end{bmatrix}
$$
$$
    B = \begin{bmatrix}
        b&b&\cdots&b
    \end{bmatrix}_{1\times{m}}^T
$$

则原等式可以表示为
$$
    Y = XW + B
$$
对特定的 $W, B$ 对当前样本 $X$ 的预测为
$$
    \hat{Y} = XW + B
$$
则损失函数可以表示为
$$
    J(W, B) = \frac{1}{2m}(Y - \hat{Y})^T(Y - \hat{Y})
$$
设
$$
    G = \begin{bmatrix}
        \frac{\partial{J(W, B)}}{w_1}&\frac{\partial{J(W, B)}}{w_2}&\cdots&\frac{\partial{J(W, B)}}{w_n}
    \end{bmatrix}^T
$$
则有
$$
    G = \frac{1}{m}X^T(\hat{Y} - Y)
$$
另有
$$
    \frac{\partial{J(W, B)}}{b} = \frac{1}{m}sum(\hat{Y} - Y)
$$
其中 $sum$ 表示对向量上各元素求和

> 注意公式只经过个人推导，不保证正确性

## 五、代码实现
### 常规形式
同样的，我们先导入`numpy`和`Matplotlib`
```python
import numpy as np
import matplotlib.pyplot as plt
```

之后定义一些需要用到的函数。

计算损失函数
```python
def get_cost(w, b, X, y):
    m = X.shape[0]

    sum_cost = 0
    for i in range(m):
        actual_y = np.dot(X[i], w) + b
        sum_cost += (actual_y - y[i]) ** 2
    total_cost = sum_cost / (2 * m)
    return total_cost
```

计算损失函数对应的梯度
```python
def get_gradient(w, b, X, y):
    m, n = X.shape

    sum_w = np.zeros((n, ))
    sum_b = 0

    for i in range(m):
        err = (np.dot(w, X[i]) + b - y[i])
        sum_b += err
        for j in range(n):
            sum_w[j] += X[i,j] * err

    gradients_w = sum_w / m
    gradient_b = sum_b / m
    return gradients_w, gradient_b
```

进行一次梯度下降迭代
```python
def gradient_descent(w, b, X, y, alpha):
    d_w, d_b = get_gradient(w, b, X, y)
    w = w - alpha * d_w
    b = b - alpha * d_b
    return w, b
```

接下来是主体部分

设置已知数据
```python
X_train = np.array(
    [[2104, 5, 1, 45],
    [1416, 3, 2, 40],
    [852, 2, 1, 35]])
y_train = np.array([460, 232, 178])
```
初始化 $w$, $b$ 及步长 $\alpha$
```python
w = np.zeros((X_train.shape[1], ))
b = 0
alpha = 1e-7
```

使用梯度下降迭代 1000 次。每 10 次打印当前的信息。
```python
for i in range(1000):
    w, b = gradient_descent(w, b, X_train, y_train, alpha)
    if i % 100 == 0:
        print(f"times:{i}, w:{np.around(w, 3)}, b:{b:0.2f}, cost:{get_cost(w, b, X_train, y_train):.4f}")
```
![print](print.png)

代价随着迭代次数的增加而减少
![loss](loss.png)

### 矩阵形式
可以利用矩阵形式重写损失函数计算和梯度计算。
重写损失函数
```python
def get_cost(w, b, X, y):
    m= X.shape[0]
    B = np.ones((m, )) * b
    actual_y = np.matmul(X, w.T) + B
    err = actual_y - y.T
    total_cost = np.matmul(err.T, err) / (2 * m)
    return total_cost
```
重写梯度
```python
def get_gradient(w, b, X, y):
    m, n = X.shape
    
    sum_w = np.zeros((n, ))
    sum_b = 0

    B = np.ones((m, )) * b
    y_hat = np.matmul(X, w.T) + B
    sum_w = np.matmul(X.T, y_hat - y.T)
    sum_b = np.sum(y_hat - y.T)

    gradients_w = sum_w / m
    gradient_b = sum_b / m
    return gradients_w, gradient_b
```
运行，与重写之前结果相同
![printMatrix](printMatrix.png)