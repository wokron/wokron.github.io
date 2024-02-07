+++
title = "2D 物理引擎基础之刚体力学与碰撞约束"
tags = ["数学", "Python"]
categories = ["游戏引擎"]
series = ["2D 物理引擎基础"]
aliases = ["/posts/704144f6"]
date = "2023-02-10T09:47:10+08:00"
+++
## 一、前言
本文是“2D物理引擎基础”的第三篇文章，主要介绍物体刚体力学的相关知识以及碰撞发生后的处理办法。后者背后的数学原理较为复杂，因此这里大多情况只是罗列公式并做简要说明。

## 二、刚体力学
### 连续的刚体力学
我们考虑二维的情况。假设有一刚体，质量为 $M$，转动惯量为 $I$，在距质心 $\vec{r}$ 处受到一力 $\vec{F}$ 的作用。

那么刚体所受相对于质心的力矩 $\vec{M}$ 为：
$$
    \vec{M} = \vec{r} \times \vec{F}
$$

注意 $\vec{M}$ 的方向垂直于平面，后面的角速度同理。

在 $\Delta{t}$ 时间内速度变化 $\Delta{\vec{v}}$ 有
$$
    \Delta{\vec{v}} = \int_{t_0}^{t0+\Delta{t}} \frac{\vec{F}(t)}{M} dt
$$

同样的，角速度变化 $\Delta{\vec{w}}$ 有
$$
    \Delta{\vec{w}} = \int_{t_0}^{t0+\Delta{t}} \frac{\vec{M(t)}}{I} dt
$$

类似的，也有位置和角度的变化
$$
    \Delta{\vec{p}} = \int_{t_0}^{t0+\Delta{t}} \vec{v}(t) dt
$$

$$
    \Delta{\vec{\theta}} = \int_{t_0}^{t0+\Delta{t}} \vec{w}(t) dt
$$

还有冲量
$$
    \vec{I} = \int_{t_0}^{t0+\Delta{t}} \vec{F}(t) dt
$$

以及角冲量（或称冲量矩）
$$
    \vec{H} = \int_{t_0}^{t0+\Delta{t}} \vec{M}(t) dt
$$

并有等式
$$
    \vec{I} = M \Delta{\vec{v}}
$$

$$
    \vec{H} = I \Delta{\vec{w}}
$$

### 离散的刚体模拟
正如第一篇文章中说过的，一般情况下计算机无法计算连续，只能通过离散的方式进行近似。物理引擎中进行的模拟会导致误差，但这对于视觉效果来说已经足够了。

#### 力的视角
同样的，我们假设有一刚体，质量为 $M$，转动惯量为 $I$，在距质心 $\vec{r}$ 处受到一力 $\vec{F}$ 的作用。在 $\Delta{t}$ 的时间间隔下，有力矩
$$
    \vec{M} = \vec{r} \times \vec{F}
$$

速度和角速度的变化为
$$
    \Delta{\vec{v}} = \frac{\vec{F}}{M} \Delta{t}
$$

$$
    \Delta{\vec{w}} = \frac{\vec{M}}{I} \Delta{t}
$$

位置和角度的变化为
$$
    \Delta{\vec{p}} =  \vec{v} \Delta{t}
$$

$$
    \Delta{\vec{\theta}} =  \vec{w} \Delta{t}
$$

#### 冲量的视角
假设物体在距质心 $\vec{r}$ 处极短的时间内受到一冲量 $\vec{I}$ 的作用，这个冲量将直接引起速度和角速度的变化。

首先，这个冲量相对于质心的冲量矩为
$$
    \vec{H} = \vec{r} \times \vec{I}
$$

冲量更新速度
$$
    \Delta{\vec{v}} = \frac{\vec{I}}{M}
$$

冲量矩更新角速度
$$
    \Delta{\vec{w}} = \frac{\vec{H}}{I}
$$

最后需要提一句，距离质点 $\vec{r}$ 处在全局坐标系的速度为
$$
    \vec{v}_p = \vec{v} + \vec{w} \times \vec{r}
$$

> 这些概念和公式并不复杂，只是有些时候难以准确记忆。罗列公式的意义也就在于此

## 三、碰撞约束
约束是物理引擎中的重要概念，指的是在一些情况下，通过某些手段，改变物体的位置、速度等数据，实现物体满足某种限制效果的方法。

本部分专门介绍碰撞约束，研究物体与物体发生碰撞后，限制物体不相互穿透的方法。我们的输入是碰撞的物体的物理属性、接触点、碰撞法线和穿透深度；输出是对碰撞物体的速度、角速度进行的修改。一般来说，物理引擎使用冲量对物体速度进行修改。

### 一个例子
我们首先考虑一个简单的问题，两个小球在平面上发生正碰，他们的初始速度分别为 $\vec{v}_a, \vec{v}_b$，由 A 到 B 的碰撞法线为 $\vec{n}$。恢复系数为 $C_r$。他们碰撞后速度改变，这等效于对两者施加怎样的冲量？

首先，我们计算两物体的相对速度
$$
\begin{equation}
    \vec{v}_{ab} = \vec{v}_b - \vec{v}_a
\end{equation}
$$

那么反弹后的相对速度应为
$$
\begin{equation}
    \vec{u}_{ab} = -C_r \vec{v}_{ab}
\end{equation}
$$

另一方面我们要对两物体施加冲量，因为牛顿第三定理，两个作用力等大反向，因此冲量的大小也相等，设为 $\lambda_n$

则冲量的作用效果为
$$
\begin{equation}
    \vec{u}_a = \vec{v}_a - \frac{\lambda_n \vec{n}}{m_a}
\end{equation}
$$

$$
\begin{equation}
    \vec{u}_b = \vec{v}_b + \frac{\lambda_n \vec{n}}{m_b}
\end{equation}
$$

$(4)-(3)$ 得
$$
\begin{equation}
    \vec{u}_b - \vec{u}_a = \vec{v}_b - \vec{v}_a + (\frac{1}{m_a} + \frac{1}{m_b}) \lambda_n \vec{n}
\end{equation}
$$

带入 $(1), (2)$ 式得
$$
\begin{equation}
    \lambda_n = \frac{-(1+C_r)\vec{v}_{ab}}{(\frac{1}{m_a} + \frac{1}{m_b})\vec{n}}
\end{equation}
$$

等式右侧上下同点乘 $\vec{n}$ 得
$$
\begin{equation}
    \lambda_n = \frac{-(1+C_r)(\vec{v}_b - \vec{v}_a) \cdot \vec{n}}{(\frac{1}{m_a} + \frac{1}{m_b})}
\end{equation}
$$

因此我们只需对物体 A 施加 $-\lambda_n \vec{n}$，对 B 施加 $\lambda_n \vec{n}$ 大小的冲量就足以实现碰撞后分开的约束效果。

### 刚体力学的碰撞约束
上一小节的例子只考虑了一个很简化的情况。只有正碰，而没有考虑斜碰时的摩擦力和旋转等问题。更一般的公式是如下所示的：

首先我们定义切向方向，它是与碰撞法线垂直，且位于速度同侧的单位向量或零向量，即
$$
    \vec{v}_R = \vec{v}_b - \vec{v}_a
$$

$$
    \vec{t} = normalize(\vec{v}_R - (\vec{v}_R \cdot \vec{n}) \vec{n})
$$

那么法向的冲量大小为
$$
    \lambda_n = \frac{-(1+C_r)(\vec{v}_b - \vec{v}_a) \cdot \vec{n}}{\frac{1}{m_a} + \frac{1}{m_b} + \frac{(\vec{r}_a \times \vec{n})^2}{I_a} + \frac{(\vec{r}_b \times \vec{n})^2}{I_b}}
$$

切向的冲量大小为
$$
    \lambda_t = \frac{-(\vec{v}_b - \vec{v}_a) \cdot \vec{t}}{\frac{1}{m_a} + \frac{1}{m_b} + \frac{(\vec{r}_a \times \vec{t})^2}{I_a} + \frac{(\vec{r}_b \times \vec{t})^2}{I_b}}
$$

其中 $\vec{r}_a, \vec{r}_b$ 表示从物体质心到接触点的位置向量。

> 注意两个公式中 $\vec{n}$ 和 $\vec{t}$ 的不同。并且注意这里的速度 $\vec{v}_a, \vec{v}_b$ 考虑了角速度，即
> $$
>     \vec{v} = \vec{v}_{\text{linear}} + \vec{w}_{\text{rotate}} \times \vec{r}
> $$

这样只要对物体 A 施加 $-(\lambda_n \vec{n} + \lambda_t \vec{t})$，对物体 B 施加 $\lambda_n \vec{n} + \lambda_t \vec{t}$ 大小的冲量就可以实现物体的碰撞的约束。

值得注意的是，我们在上面的公式中多次看到质量的倒数和转动惯量的倒数，这一数值在物理引擎中经常使用，我们可以将其在一开始便计算并储存起来。另外，冲量等式中分母部分也较为常见，我们同样可以将其提出作为单独的参数。此参数被称为（法向/切向）有效质量。
$$
    M_n^{-1} = \frac{1}{\frac{1}{m_a} + \frac{1}{m_b} + \frac{(\vec{r}_a \times \vec{n})^2}{I_a} + \frac{(\vec{r}_b \times \vec{n})^2}{I_b}}
$$

$$
    M_t^{-1} = \frac{1}{\frac{1}{m_a} + \frac{1}{m_b} + \frac{(\vec{r}_a \times \vec{t})^2}{I_a} + \frac{(\vec{r}_b \times \vec{t})^2}{I_b}}
$$

我们修改原有的 Body 类
```py
        self.mass = density * get_area(self.points)
        self.inv_mass = 1 / self.mass # add inv_mass
        self.inertia = get_moments_of_inertia(self.points, self.mass)
        self.inv_inertia = 1 / self.inertia # add inv_inertia
```

并编写计算有效质量的函数
```py
def get_effective_mass(body_a, ra, body_b, rb, unit_vect):
    ma = body_a.inv_mass
    mb = body_b.inv_mass

    ia = (np.cross(ra, unit_vect))**2 * body_a.inv_inertia
    ib = (np.cross(rb, unit_vect))**2 * body_b.inv_inertia

    return 1 / (ma + mb + ia + ib)
```

其中 `unit_vect` 表示切向量或者法向量。

### 摩擦力的模拟
很多时候我们还需要考虑摩擦力的影响，这样我们就需要对切向的冲量进行调整。具体来说，我们假设最大静摩擦系数等于动摩擦系数，那么就需要满足
$$
    F_f \le \mu F_n
$$

> 注意 $F_f, F_n$ 是力的大小，不考虑方向

不等式两边同乘 $\Delta{t}$ 就变成冲量不等式，即
$$
    \lambda_t \le \mu \lambda_n
$$

因此对摩擦力的限制即为
$$
    \lambda'_t = clamp_{[-\mu\lambda_n, \mu\lambda_n]}(\lambda_t)
$$

> clamp 函数将 $\lambda_t$ 限制在区间 $[-\mu\lambda_n, \mu\lambda_n]$ 间。

### 速度约束
据此我们可以编写实施约束的代码。首先，我们将碰撞相关的信息打包成一个类。在物理引擎理论中，这似乎被称作 Manifold。
```py
class Manifold:
    def __init__(self, body_a, body_b, collide_result):
        self.body_a = body_a
        self.body_b = body_b
        self.normal, self.contact_points = collide_result

        self.ra = []
        self.rb = []
        for point, depth in self.contact_points:
            self.ra.append(point - body_a.position)
            self.rb.append(point - body_b.position)
```
其中传入的 `collide_result` 参数即 `sat3` 函数的输出。

接着，我们编写碰撞中约束速度的函数：
```py
def velocity_constrain(manifold):
    body_a = manifold.body_a
    body_b = manifold.body_b
    contact_points = manifold.contact_points
    n = manifold.normal

    for i in range(len(contact_points)):
        p, depth = contact_points[i]

        ra = manifold.ra[i]
        rb = manifold.rb[i]

        va = body_a.v + cross_product(body_a.w, ra)
        vb = body_b.v + cross_product(body_b.w, rb)

        rv = vb - va

        if np.dot(rv, n) > 0: # 如果速度方向背离了碰撞方向，则说明物体已经分离
            continue
        
        t = normalize(rv - np.dot(rv, n) * n)

        effective_mass_n = get_effective_mass(body_a, ra, body_b, rb, n)
        effective_mass_t = get_effective_mass(body_a, ra, body_b, rb, t)

        cr = min(body_a.restitution, body_b.restitution)

        lambda_n = -(1 + cr) * np.dot(rv, n) * effective_mass_n

        body_a.apply_impulse(-lambda_n * n, p)
        body_b.apply_impulse(lambda_n * n, p)

        lambda_t = - np.dot(rv, t) * effective_mass_t

        mu = np.sqrt(body_a.friction * body_b.friction)

        max_friction = abs(mu * lambda_n)

        lambda_t = clamp(lambda_t, -max_friction, max_friction)

        body_a.apply_impulse(-lambda_t * t, p)
        body_b.apply_impulse(lambda_t * t, p)


def cross_product(s, a):
    return np.array([-s * a[1], s * a[0]])


def clamp(value, smallest, largest):
    if value < smallest:
        return smallest
    elif largest < value:
        return largest
    else:
        return value
```

> **十分需要注意的是**，碰撞约束的求解是一个迭代的过程，这是因为每次碰撞检测只会考虑某两个物体；对于多对物体，我们需要先求出所有发生碰撞的物体对，将碰撞信息存入 `Manifold` 类中。接着多次迭代，每一次迭代对所有的 Manifold 求解速度约束。只有这样才能尽量求得全局的约束结果。

### 穿透问题与位置约束
容易发现，我们上面的代码并没有使用穿透深度。在一些情况下，因为在碰撞后的第一帧就进行了速度处理，此时穿透深度较小，很快就能修正，所以视觉上并无大碍。但很多时候，如施加了重力的情况下，单纯的修改速度不足以抵消重力的影响，此时穿透深度会越来越大，最终会使物体陷入地面。

因此，我们除了进行速度约束以外，还需要根据穿透深度进行位置的修正。这就是位置约束。

我们设当前穿透深度为 $x$，最大允许穿透深度为 $x_{max}$，则有
$$
    \lambda_{pos} = \beta M^{-1}_{n} max(x - x_{max}, 0) \quad (0 \lt \beta \lt 1)
$$

其中 $\beta$ 是人为穿透的解决程度（虽然不太理解这有什么意义）。

类似于冲量，最终的位置的变化量为
$$
    \Delta \vec{p}_a = -\frac{\lambda_{pos}\vec{n}}{m_a}
$$

$$
    \Delta \vec{p}_b = \frac{\lambda_{pos}\vec{n}}{m_b}
$$

> 本方法的数学原理并不清楚，但似乎 box2d 中是这样做的？

据此我们能写出代码。需要注意一点，位置变化会引起接触点的变化，我们希望得知接触点分离的时刻，这通过保留物体质心到接触点的向量实现。这也是我们在 `Manifold` 类中加入 `ra, rb` 的原因。

```py
def position_constrain(manifold):
    body_a = manifold.body_a
    body_b = manifold.body_b
    contact_points = manifold.contact_points
    n = manifold.normal

    bias_factor = 0.8
    max_depth = 0.005
    for i in range(len(contact_points)):
        p, depth = contact_points[i]

        ra = manifold.ra[i]
        rb = manifold.rb[i]

        pa = ra + body_a.position
        pb = rb + body_b.position

        c = pa - pb

        if np.dot(c, n) < 0: # 判断接触点是否分离
            continue

        bias = bias_factor * max((depth - max_depth), 0)
        lambda_p = get_effective_mass(body_a, ra, body_b, rb, n) * bias

        correction = lambda_p * n

        body_a.position = body_a.position - body_a.inv_mass * correction

        body_b.position = body_b.position + body_b.inv_mass * correction
```

> 位置约束的求解也是一个迭代过程。发生在速度约束的迭代完成后。

## 四、总结
这一篇文章内容较多，简单介绍了一下刚体力学的知识，接着主要是碰撞约束的求解方法。在一些应详细解释的地方因能力不足，未能详述，实在抱歉。但还是希望这篇文章能对自己和他人有所帮助吧。