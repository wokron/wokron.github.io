+++
title = "机器学习笔记之逻辑回归"
tags = ["数学", "Python"]
categories = ["人工智能"]
series = ["机器学习笔记"]
aliases = ["/posts/ba0d6023"]
date = "2022-10-05T16:06:34+08:00"
+++
## 一、前言——逻辑分类
机器学习研究的另一种问题为分类问题。给出一些信息，判断是或不是某种物体、属性。比如说给出病人的各项检测指标，判断其是否患病。

对于是、否的判断，这里用 1、0 表示。则训练样本就与之前的线性与多项式回归样本类似。

我们可以试着用一元线性回归来拟合分类问题的样本。

比如对于如下的样本数据
![scatter1](scatter1.png)

进行一元线性回归后得到
![linear1](linear1.png)

我们将数值大于 0.5 的部分看做预测为真，小于 0.5 的部分看做预测为假。可以看到，此时一元线性回归就已经可以使所有样本点符合判断结果了。

但如果我们在 x 值更大的区域增加更多的样本点，再进行回归
![linear2](linear2.png)

可以看到，有一部分应为 1 的点被分在了 y < 0.5 的部分，被预测为 0。这说明只凭线性回归无法解决分类问题。我们需要另一种回归方法。

## 二、逻辑回归
线性回归失效的原因在于，拟合的目标是对所有点的方差最小，在分类边界上的样本可能因为其他样本的影响而被分到另一类。而为什么会受到其他样本的影响呢？因为线性回归得到的表达式值域趋向于无穷；而样本的结果却只有 0、1。所以对于一个样本，只要它的特征数值足够大或足够小，就足以产生极大的损失。（为什么可能出现足够远的样本？因为分类问题需要划分出一个边界，在边界两侧可能存在足够大的范围。）

我们要对线性回归进行修改，需要将回归得到的表达式的值域缩小到 0 至 1 的范围。

这里引入 Sigmoid 函数
$$
    S(z) = \frac{1}{1 + e^{-z}}
$$

它的图像如下
![sigmoid](sigmoid.png)

该函数定义域为 $(-\infty, +\infty)$，值域为 $(0, 1)$

假设原本线性回归的函数为
$$
    f_{\vec{w}, b}(\vec{x}) = \vec{w} \cdot \vec{x} + b
$$

则现在令
$$
    f_{\vec{w}, b}(\vec{x}) = S(\vec{w} \cdot \vec{x} + b)
$$

即
$$
    f_{\vec{w}, b}(\vec{x}) = \frac{1}{1 + e^{\vec{w} \cdot \vec{x} + b}}
$$

为新的函数。其中 $S(z)$ 即为 Sigmoid 函数。这样，表达式的值域就缩小为了 $(0, 1)$，从而消除了值域对损失的影响。

## 三、损失函数
如果我们依旧采用线性回归时的损失函数计算方式，就会发现函数非凸，那么梯度下降法将不能得到很好的结果。因此我们重新定义损失函数。

设单样本损失函数
$$
    loss(f_{\vec{w},b}(\vec{x}^{(i)}), y^{(i)}) = \begin{cases}
        -log(f_{\vec{w},b}(\vec{x}^{(i)}))  &  \text{if } y^{(i)} = 1 \\
        -log(1 - f_{\vec{w},b}(\vec{x}^{(i)})) & \text{if } y^{(i)} = 0
    \end{cases}
$$
这个损失函数保证了在样本真实结果为真或为假的条件下都能以同样的标准衡量预测的误差。

另外，因为 $y^{(i)}$ 的取值只有 0、1。所以可以将上式化简为单个公式的形式
$$
    loss(f_{\vec{w},b}(\vec{x}^{(i)}), y^{(i)}) = 
    -y^{(i)}log(f_{\vec{w},b}(\vec{x}^{(i)})) - (1 - y^{(i)})log(1 - f_{\vec{w},b}(\vec{x}^{(i)}))
$$ 

则全样本代价函数
$$
    J(\vec{w}, b) = \frac{1}{m} \sum_{i = 1}^m loss(f_{\vec{w},b}(\vec{x}^{(i)}), y^{(i)})
$$

## 四、梯度下降
我们对损失函数求导，得
$$
    \frac{\partial{J(\vec{w}, b)}}{\partial{w_j}}
    = \frac{1}{m}\sum_{i=1}^m (f_{\vec{w},b}(\vec{x}^{(i)}) - y^{(i)}) x^{(i)}_j \\
    \frac{\partial{J(\vec{w}, b)}}{\partial{b}}
    = \frac{1}{m}\sum_{i=1}^m (f_{\vec{w},b}(\vec{x}^{(i)}) - y^{(i)})
$$

可以发现求导后的公式形式与线性回归时的相同，这是有意为之的。但是要注意的是， $f_{\vec{w},b}(\vec{x}^{(i)})$ 在线性回归与逻辑回归时并不相同。逻辑回归时的表达式是线性回归时的加上 Sigmoid 函数。

按照与线性回归相同的方法，以确定的步长 $\alpha$ 进行梯度下降，不断迭代
$$
    w_j = w_j - \alpha \frac{\partial{J(\vec{w}, b)}}{\partial{w_j}} \\
    b = b - \alpha \frac{\partial{J(\vec{w}, b)}}{\partial{b}}
$$
最终便可找到使损失函数最小的 $\vec{w}, b$。

## 五、代码实现
在线性回归的基础上，修改损失函数的计算
```python
def sigmoid(x):
    return 1 / (1 + np.exp(-x))


def get_loss(actual_y, y):
    return -y * math.log(actual_y) - (1 - y) * math.log(1 - actual_y)

def get_cost(w, b, X, y):
    m = X.shape[0]

    sum_cost = 0
    for i in range(m):
        z = np.dot(X[i], w) + b
        actual_y = sigmoid(z)
        sum_cost += get_loss(actual_y, y[i])
    total_cost = sum_cost / m
    return total_cost
```

修改梯度的计算
```python
def get_gradient(w, b, X, y):
    m, n = X.shape

    sum_w = np.zeros((n, ))
    sum_b = 0

    for i in range(m):
        err = sigmoid(np.dot(w, X[i]) + b) - y[i]
        sum_b += err
        for j in range(n):
            sum_w[j] += X[i,j] * err

    gradients_w = sum_w / m
    gradient_b = sum_b / m
    return gradients_w, gradient_b
```

拟合结果
![logical](logical.png)

损失函数
![cost](cost.png)

对样本的预测
![predict](predict.png)