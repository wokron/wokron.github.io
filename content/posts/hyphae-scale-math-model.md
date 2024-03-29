+++
title = "基于概率模型的蘑菇菌丝规模分析"
tags = ["数学"]
categories = ["数学建模"]
aliases = ["/posts/7a0960e5"]
date = "2023-03-17T16:38:01+08:00"
+++

**摘要：**
本文提出了一种将菌丝体结构与地面上蘑菇子实体的位置联系起来的概率模型，用于分析蘑菇菌丝体的规模。我们假设地形平坦，菌丝体以恒定速率扩散。同时假设地面上蘑菇子实体的位置遵循菌丝体形状决定的均匀分布。基于这些假设，我们将菌丝体定义为以初始孢子位置 (m n) 为中心，半径为 r 的圆形区域。蘑菇子实体位置的分布服从该圆形区域内的均匀分布。

对于圆心位置和半径大小，我们分别提出了三种模型。对圆心位置来说：样本点平均值模型通过直接计算所有蘑菇子实体位置的平均值来估计圆心位置；最远点平均值模型通过计算横纵坐标投影上的最远点的平均值作为对圆心的估计；中位数模型通过分别计算横纵坐标轴上的中位数进行估计。

对半径大小来说：平均值模型通过计算所有样本点距已求得的圆心距离的平均值估计半径；第二种模型中采用了极大似然法对半径进行估计；两端间隔模型则通过考虑半径取值中两端间隔的比例关系计算半径大小。

我们将这些模型结合起来求解原问题。为了选择出更优的模型，我们采用了计算机数值模拟的方法。通过多次的随机模拟试验，考察模型预测结果的平均值、相对平均值的误差和标准差，对模型进行了评价，并据此选出了更好的求解结果。
**Keywords: 概率论，数值模拟**
<!-- more -->

## 简介
### 背景
蘑菇作为一种广泛存在于自然界的真菌，为人们所熟知。但其除了裸露在地表的子实体部分以外，在地下还拥有着极其庞大的菌丝结构。为了完成对蘑菇的全面详细研究，有必要对菌丝结构这一蘑菇的重要组成部分进行分析和度量。其中尤为重要的，就是确定菌丝结构的位置和范围。但是由于菌丝处于地下且规模巨大，难以通过直接测量得出结果。

### 问题重述
蘑菇的孢子落到土壤中，不断生长出菌丝。菌丝以相同的速度向外辐射，构成庞大的菌丝菌丝。当环境条件合适时，这些菌丝会生长成裸露在地面的“蘑菇”。现在我们已知地面上属于同一菌丝结构的所有蘑菇的位置，如表 1 和图 1 所示：

| 序号 | 横坐标 | 纵坐标 | 序号 | 横坐标 | 纵坐标 |
|-----------|--------------|--------------|-------------|--------------|--------------|
| 1         | 21.12        | 29.84        | 11          | 24.81        | 17.16        |
| 2         | 20.39        | 11.50        | 12          | 41.31        | 31.77        |
| 3         | 38.52        | 13.34        | 13          | 20.24        | 15.50        |
| 4         | 33.85        | 15.47        | 14          | 34.63        | 26.43        |
| 5         | 44.24        | 21.49        | 15          | 42.67        | 20.42        |
| 6         | 24.87        | 22.42        | 16          | 18.52        | 14.49        |
| 7         | 46.32        | 26.41        | 17          | 17.09        | 25.83        |
| 8         | 20.27        | 11.62        | 18          | 32.33        | 17.14        |
| 9         | 44.56        | 19.73        | 19          | 18.72        | 17.83        |
| 10        | 47.69        | 19.25        | 20          | 28.52        | 25.85        |

![mushrooms1](mushrooms1.png)

我们的问题是，如何根据地上蘑菇子实体的位置信息，估计形成蘑菇的孢子最初落地的位置和地下菌丝结构的规模。

### 我们的工作
在本篇文章中，我们建立了一种真菌生长的模型，将地下菌丝结构与地上子实体位置进行了概率上的关联。同时提出了求解该模型的不同思路，通过计算机模拟，对比选择出了更优的求解办法。

## 问题假设
- 假设一：当地地形平坦，菌丝以相同的速度向外辐射，形成圆形
- 假设二：地上子实体的位置服从地下菌丝形状的均匀分布

## 符号说明

| 符号               | 定义        |
|------------------|-----------|
| $(m, n)$         | 孢子下落的最初位置 |
| $r$              | 菌丝区域的半径   |
| $\{(x_i, y_i)\}$ | 子实体位置样本集合 |

## 模型的建立与分析
### 问题分析
由于我们假设地形平坦，菌丝以相同的速度向外辐射，那么在蘑菇生长的任意时刻，菌丝结构在水平面上的投影都是正圆形。我们设孢子下落的位置为 $(m, n)$，菌丝距包子位置的最远距离为 $r$，那么菌丝结构可以视为二维平面上的圆形区域 $C: (x-m)^2 + (y-n)^2 \le r$。

又因为我们假设地上子实体的位置服从地下菌丝形状的均匀分布，那么对于任意子实体的位置 $(x_i, y_i)$，服从均匀分布，其概率密度函数为
$$
    f(x, y)=\left\{
    \begin{aligned}
    \frac{1}{\pi r^2} & \quad (x, y) \in C \\
    0 & \quad (x, y) \notin C\\
    \end{aligned}
    \right.
$$

这样我们就将问题转化为：已知有一组样本 $(x_1, x_2), ... (x_n, y_n)$ 服从区域 $C: (x-m)^2 + (y-n)^2 \le r$ 上的均匀分布，$m, n, r$ 未知。求 $m, n, r$。

这一问题包含两个子问题，（1）求解 $m, n$，（2）求解 $r$。接下来我们将分别建立模型。

### 模型选择与分析
#### 样本点平均值估计圆心
对样本 $(x_1, x_2), ... (x_n, y_n)$，记 $\bar{x} = \frac{1}{n}\sum_{i=1}^n x_i$，$\bar{y} = \frac{1}{n}\sum_{i=1}^n y_i$。总体的平均值 
$$
    \bar{X} = \int_{-\infty}^{+\infty}\int_{-\infty}^{+\infty} xf(x, y)dxdy = m \\
    \bar{Y} = \int_{-\infty}^{+\infty}\int_{-\infty}^{+\infty} yf(x, y)dxdy = n
$$

用 $(\bar{x}, \bar{y})$ 作为 $(\bar{X}, \bar{Y})$ 的估计值，得到
$$
    m = \bar{x}, n = \bar{y}
$$

#### 最远点平均值估计圆心
我们设
$$
    x_{min} = \min{x_i} \\
    x_{max} = \max{x_i} \\
    y_{min} = \min{y_i} \\
    y_{max} = \max{y_i}
$$

利用最远点的平均值估计圆心，即
$$
    m = \frac{x_{max} - x_{min}}{2} \\
    n = \frac{y_{max} - y_{min}}{2}
$$

#### 横纵坐标中位数估计圆心
我们记 $\{x_i\}$ 的中位数为 $\widetilde{x}$，$\{y_i\}$ 的中位数为 $\widetilde{y}$。总体的中位数为 $\widetilde{X}$ 和 $\widetilde{Y}$。用 $(\widetilde{x}, \widetilde{y})$ 作为  $(\widetilde{X}, \widetilde{Y})$ 的估计值。由于 $\widetilde{X} = \bar{X}, \widetilde{Y} = \bar{Y}$，则
$$
    m = \widetilde{x} \\
    n = \widetilde{y}
$$

#### 平均值估计半径
假设此时我们已经确定了圆心 $(m, n)$ 的位置，那么我们可以计算出样本点相对于 $(m, n)$ 的距离
$$
    r_i = \sqrt{(x_i - m)^2 + (y_i - n)^2}
$$

我们设点到圆心的距离的平均值为 $\bar{r} = \frac{1}{n}\sum_{i=1}^n r_i$，总体到圆心的平均距离为 $\bar{R}$。用 $\bar{r}$ 作为 $\bar{R}$ 的估计值。

为此我们需要求得 $\bar{R}$ 的值。首先设随机变量 $Z$ 为点到圆心的距离，圆形半径为 $r$，则有概率分布函数
$$
    F(z) = P\{Z \le z\} = \frac{\pi z^2}{\pi r^2} = \frac{z^2}{r^2}
$$

对应的概率密度函数
$$
    f(z) = \frac{2z}{r^2}
$$

因此 $Z$ 的数学期望为
$$
    E(Z) = \int_{0}^{r} z f(z) dz = \frac{2}{3}r
$$

因此有 $\bar{R} = \frac{2}{3}r$。所以
$$
    r = \frac{3}{2} \bar{r}
$$

#### 极大似然法估计半径
我们已知距圆心距离的概率密度函数为 $f(z) = \frac{2z}{r^2}$。使用极大似然法估计参数 $r$。

构造似然函数
$$
    L(x; r) = \prod_{i=1}^n f(r_i; r) = \prod_{i=1}^n \frac{2r_i}{r^2}
$$

为方便计算，对等式两边同取对数
$$
    \ln L(x; r) = \ln \prod_{i=1}^n \frac{2r_i}{r^2} = \sum_{i=1}^n \ln 2r_i - n \ln r^2
$$

要求
$$
    r = \underset{r}{\text{argmax}} \ln L(x; r)
$$

对 $\ln L(x; r)$ 求导，即
$$
    \frac{d \ln L(x; r)}{r} = \frac{2n}{r} \gt 0
$$

又因参数 $r$ 可能的取值范围为 $[\max{r_i}, \infty)$。因此根据极大似然法得到的半径估计为
$$
    r = \max{r_i}
$$

#### 两端间隔模型估计半径
根据概率密度函数 $f(z) = \frac{2z}{r^2}$ 可以得知，从样本点距圆心距离来看，越靠近圆心的点应该更加稀疏，越远离圆心的点应该更加稠密。我们按从小到大的顺序排列 $r_1 \le r_2 \le ...\le r_n$。那么 $r_1$ 的大小概率上应该大于 $r - r_n$。我们可以假设有这样的关系
$$
    \frac{r_1 - 0}{r - r_n} = \frac{f(r)}{f(r_1)} = \frac{r}{r_1}
$$

因此有
$$
    r = \frac{r_n + \sqrt{r_n^2 + 4r_1^2}}{2}
$$

## 模型求解
### 模型求解
我们将上面估计圆心位置的三种模型与估计半径的三种模型两两组合，形成九种求解方案。分别求得对应的 $m, n, r$。

| 圆心估计模型  | 半径估计模型 | 横坐标m  | 纵坐标n  | 半径r   |
|---------|--------|-------|-------|-------|
| 样本点平均值  | 半径平均值  | 34.83 | 20.92 | 11.55 |
| 样本点平均值  | 极大似然法  | 34.83 | 20.92 | 14.42 |
| 样本点平均值  | 两端间隔模型 | 34.83 | 20.92 | 14.53 |
| 最远点平均值  | 半径平均值  | 32.62 | 19.16 | 12.29 |
| 最远点平均值  | 极大似然法  | 32.62 | 19.16 | 13.87 |
| 最远点平均值  | 两端间隔模型 | 32.62 | 19.16 | 14.17 |
| 横纵坐标中位数 | 半径平均值  | 34.98 | 22.20 | 11.55 |
| 横纵坐标中位数 | 极大似然法  | 34.98 | 22.20 | 14.26 |
| 横纵坐标中位数 | 两端间隔模型 | 34.98 | 22.20 | 14.37 |

### 模型结果与分析
从当前已经获得的结果可知，横坐标大致在 32 ∼ 34，纵坐标大致在 19 ∼ 23，半径大致在
11 ∼ 15 范围内。但是不同的方案预测的结果差异较大，我们还需要从中选择出最佳的结果。为此
我们需要对模型进行评价。

## 模型的评价
### 数值模拟
因为我们并不知道真正的结果是什么，所以无法通过模型求得的结果判断那种模型更加精确。为了进行模型评价，我们可以采用计算机模拟的方法。

首先，我们需要确定生成圆形区域均匀分布的算法。对于角度，我们可以直接将 $[0, 1)$ 随机数线性映射到 $[0, 2\pi)$。但对于半径的取值，我们必须依照概率密度函数 $f(z) = \frac{2z}{r^2}$ 生成随机数。

我们采用的方法是反函数法。具体来说，对概率分布函数 $F(z) = \frac{z^2}{r^2}$。我们取得其反函数
$$
    F^{-1}(p) = r\sqrt{p}
$$
将 $[0, 1)$ 随机数经过该反函数的映射后即得到概率密度为 $f(z)$ 的随机数。

随后将生成的随机数输入到模型中进行计算，得出数值结果。

### 结果评价
我们设定 $m = 50, n = 50, r = 20$，从总体中随机取 $20$ 个点作为样本。对每个样本分别使用模型估计 $m, n, r$（其中估计 $r$ 时，给定真实的 $m, n$）。如此取 $200$ 个样本，计算 $200$ 个样本估计的 $m, n, r$ 平均值、平均值与真值的误差和标准差。结果如下：

| 模型 | x的平均值 | x平均值的误差 | x的标准差 | y的平均值 | y平均值的误差 | y的标准差 |
|-----------|----------------|------------------|----------------|----------------|------------------|----------------|
| 样本点平均值    | 50.2           | 0.2              | 2.47           | 50.3           | 0.3              | 2.14           |
| 最远点平均值    | 50.04          | 0.04             | 1.77           | 50.02          | 0.02             | 1.66           |
| 横纵坐标中位数   | 50.26          | 0.26             | 3.63           | 50.31          | 0.31             | 3.1            |

| 模型 | r的平均值 | r平均值的误差 | r的标准差 |
|-----------|----------------|------------------|----------------|
| 半径平均值     | 20.11          | 0.11             | 1.67           |
| 极大似然法     | 19.5           | -0.5             | 0.48           |
| 两端间隔模型    | 20.47          | 0.47             | 1.01           |

我们也根据结果绘制了频率直方图，如下所示。其中模型序号与“模型选择与分析”一节中顺序相同。
![c1](c1.png)
![c2](c2.png)
![c3](c3.png)
![r1](r1.png)
![r2](r2.png)
![r3](r3.png)

根据模拟的结果可以得出以下的初步结论：
- 对于圆心位置的估计来说，最远点平均值模型优于样本点平均值和中位数模型。不仅误差较小，标准差也最小。这说明在小样本条件下，平均值和中位数未必是最优的模型。
- 对于半径的估计来说，平均值模型拥有较小的误差，但标准差较大，预测结果不稳定；极大似然估计的结果标准差较小，但结果几乎总是小于真实值；两端间隔模型标准差较小，但误差却较大。因此这部分模型的选择需要考虑实际应用情况。

总体来说，对圆心位置的估计，最远点平均值模型最优；对半径的估计，如果不看重多次估计的集中程度，则平均值模型更优；反之，则可以选择两端间隔模型。因此，最远点平均值模型+平均值模型和最远点平均值模型+两端间隔模型是更合适的选择。

## 总结
本篇文章中，我们从估计菌丝结构的问题出发，分析了“通过圆形区域内均匀分布的样本点，估计圆形位置和范围”的解决方法。利用概率论等相关知识，建立了一系列符合理论的数学模型，并通过数值模拟对模型进行了评价分析，选择出更加适合当前问题的模型。

## Appendices
### 附录 A 圆形区域均匀分布随机数生成算法
```py
import math
from random import Random

import numpy as np


def generate_circle_point(r):
    rand = Random()
    angle = rand.random() * (2 * np.pi)
    r_len = r * math.sqrt(rand.random())

    return np.array([r_len * math.sin(angle), r_len * math.cos(angle)])


def generate_circle(m, n, r, point_num):
    points = []
    for i in range(point_num):
        points.append(generate_circle_point(r))

    points = np.array(points)

    points += np.array([m, n])

    return points
```

### 附录 B 各类数学模型的代码实现
```py
import math

import numpy as np


def circle_center1(points):
    return points.mean(axis=0)

def circle_center2(points):
    x_min, y_min = points.min(axis=0)
    x_max, y_max = points.max(axis=0)
    return np.array([(x_min + x_max) / 2, (y_min + y_max) / 2])


def circle_center3(points):
    return np.median(points, axis=0)


def circle_radius1(radius_list):
    r_mean = radius_list.mean()
    return (3 / 2) * r_mean


def circle_radius2(radius_list):
    return max(radius_list)


def circle_radius3(radius_list):
    r_min = min(radius_list)
    r_max = max(radius_list)
    return (r_max + math.sqrt(r_max ** 2 + 4 * r_min ** 2)) / 2
```

### 附录 C 模型求解方法
```py
from math_model2 import *

mushrooms = []

with open("mushrooms.txt", "r") as f:
    for i, line in enumerate(f):
        if i == 0:
            continue
        grid = line.split("\t")
        mushrooms.append([float(grid[1]), float(grid[2])])

mushrooms = np.array(mushrooms)

calc_center_funcs = [circle_center1, circle_center2, circle_center3]
calc_radius_funcs = [circle_radius1, circle_radius2, circle_radius3]

with open("ans.txt", "w") as f:
    f.write(f"圆心估计模型\t半径估计模型\t横坐标\t纵坐标\t半径\n")
    for i, center_func in enumerate(calc_center_funcs):
        for j, radius_func in enumerate(calc_radius_funcs):
            m, n = center = center_func(mushrooms)

            r_list = np.linalg.norm(mushrooms - center, axis=1)

            r = radius_func(r_list)

            f.write(f"center{i+1}\tradius{j+1}\t{m:.2f}\t{n:.2f}\t{r:.2f}\n")
```

### 附录 D 数值模拟方法
```py
import matplotlib.pyplot as plt

from math_model2 import *
from random_generator import generate_circle

num = 20
m = 50
n = 50
r = 20

calc_center_funcs = [circle_center1, circle_center2, circle_center3]
calc_radius_funcs = [circle_radius1, circle_radius2, circle_radius3]

center = [[], [], []]
radius = [[], [], []]

for step in range(200):
    mushrooms = generate_circle(m, n, r, num)
    radius_list = np.linalg.norm(mushrooms - np.array([m, n]), axis=1)

    for i in range(3):
        center_func = calc_center_funcs[i]
        center_ans = center_func(mushrooms)

        center[i].append(center_ans)

    for i in range(3):
        radius_func = calc_radius_funcs[i]
        radius_ans = radius_func(radius_list)

        radius[i].append(radius_ans)

table_val = np.zeros((3, 6))
row_label = ["model 1", "model 2", "model 3"]
col_label = ["x avg", "x error", "x std", "y avg", "y error", "y std"]
for idx, one_center in enumerate(center):
    x, y = zip(*one_center)
    x = np.array(x)
    y = np.array(y)

    table_val[idx][0] = x.mean()
    table_val[idx][1] = x.mean() - m
    table_val[idx][2] = x.std()
    table_val[idx][3] = y.mean()
    table_val[idx][4] = y.mean() - n
    table_val[idx][5] = y.std()

    plt.suptitle(f"Centroid Estimation Model {idx+1}")
    plt.subplot(1, 2, 1)
    plt.hist(x)
    plt.title("Estimation for X Axis")

    plt.subplot(1, 2, 2)
    plt.hist(y)
    plt.title("Estimation for Y Axis")
    plt.show()

table_val = np.around(table_val, decimals=2)
plt.suptitle(f"Numerical Simulation for Centroid Estimation Models")
plt.table(
    cellText=table_val,
    rowLabels=row_label,
    colLabels=col_label,
    loc="center",
    cellLoc="center",
    rowLoc="center",
)
plt.axis("off")
plt.show()

table_val = np.zeros((3, 3))
row_label = ["model 1", "model 2", "model 3"]
col_label = ["r avg", "r error", "r std"]
for idx, one_radius in enumerate(radius):
    one_radius = np.array(one_radius)

    table_val[idx][0] = one_radius.mean()
    table_val[idx][1] = one_radius.mean() - r
    table_val[idx][2] = one_radius.std()
    
    plt.hist(one_radius)
    plt.suptitle(f"Radius Estimation Model {idx+1}")
    plt.show()

table_val = np.around(table_val, decimals=2)
plt.suptitle("Numerical Simulation for Radius Estimation Models")
plt.table(
    cellText=table_val,
    rowLabels=row_label,
    colLabels=col_label,
    loc="center",
    cellLoc="center",
    rowLoc="center",
)
plt.axis("off")
plt.show()
```