+++
title = "机器学习笔记之多项式回归"
tags = ["数学", "Python"]
categories = ["人工智能"]
series = ["机器学习笔记"]
aliases = ["/posts/dfc6ce5"]
date = "2022-10-01T21:52:38+08:00"
+++
## 一、引言——拟合多项式
考虑如图所示的样本数据：
![poly](poly.png)

如果我们用一元线性回归去拟合该数据，会发现直线与数据点拟合效果很差。
![linearTry](linearTry.png)

通过观察，我们很容易得知数据点满足二次函数，具体地，函数是 $\frac{1}{2}x^2 + 5$。既然用直线 $y = wx + b$ 不能拟合，那么换用二次函数 $y = w_1x^2 + w_2x + b$ 拟合的话效果会如何？

## 二、多项式回归的步骤
对于一元多项式函数
$$
    y = w_1x + w_2x^2 + \cdots + w_nx^n + b 
$$
令 $x^i = y_i$，则原函数变为
$$
    y = w_1y_1 + w_2y_2 + \cdots + w_ny_n + b
$$
对于自变量 $x$，只需将 $x^2, x^3, \cdots, x^n$ 看成与 $x$ 不同的特征。此时一个一元多项式函数的拟合问题就转变成了我们已知的多元线性拟合问题。同理，对于多元多项式函数，也可以将其转化为多元线性函数进行拟合。

## 三、代码实现
对于如下样本：
```python
x = np.array(range(-1, 10))
y = (x ** 2) / 2 + 5
```

将 $x^2$ 也作为一个特征
```python
X_train = np.c_[x.T, (x ** 2).T] # np.c_ 用于进行列的扩展
y_train = y
```

剩下的与多元线性回归同理
```python
alpha = 1e-3
w = np.zeros(X_train.shape[1])
b = 0

for i in range(1000):
    w, b = gradient_descent(w, b, X_train, y_train, alpha)
```

迭代信息：
![print](print.png)

拟合效果：
![result](result.png)

损失函数：
![cost](cost.png)

---
**扩展：**

对于较为复杂的表达式，如 $y = sinx$
```python
x = np.array(range(-1, 10))
y = np.sin(x)
```

只要增加多项式次数
```python
X_train = np.c_[x.T, (x**2).T, (x**3).T, (x**4).T, (x**5).T, (x**6).T, (x**7).T, (x**8).T, (x**9).T, (x**10).T, (x**11).T]
```

再加上特征缩放
```python
X_train, _, _ = featureScaling(X_train)
```

在一定的步长和迭代次数下
```python
alpha = 0.2

for i in range(50000):
    w, b = gradient_descent(w, b, X_train, y_train, alpha)
```

同样可以取得一定的拟合效果
![complex](complex.png)

![complex_cost](complex_cost.png)