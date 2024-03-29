+++
title = "2D 物理引擎基础之碰撞检测"
tags = ["数学", "Python"]
categories = ["游戏引擎"]
series = ["2D 物理引擎基础"]
aliases = ["/posts/f58966b1"]
date = "2023-02-08T20:22:21+08:00"
+++
## 一、前言
本篇是“2D物理引擎基础”的第二篇文章，将主要讲解碰撞检测的基本原理和算法。

碰撞检测是物理引擎十分重要的组成部分。碰撞检测及后续的碰撞约束反映了物质的不可入性，是物理模拟真实性的基础。另外碰撞检测也能有效用于游戏逻辑的实现，如触发陷阱、探测障碍等等。

## 二、碰撞检测的基本概念
碰撞检测的目标就是判断两个几何体是否存在重合部分，如果存在重合部分，则得到接触点、碰撞法线和穿透深度。

接触点（Contact Point）指的是两个几何体相接触的位置。当然对于碰撞检测而言，接触的部分并非点而是区域，但是我们可以通过算法从中近似地取出接触点，一般位于其中一个几何体的顶点或边上。

碰撞法线指示了一个物体与另一个物体发生碰撞后，后者为了分离而应该采取的运动方向。这一法线大多是几何体中某一条边的法线

穿透深度是沿碰撞法线方向最终分离所需的距离，这一数值常用来修正碰撞后两几何体的位置，避免两几何体在视觉上发生重叠。

## 三、分离轴定理（SAT，Separating Axis Theorem）
分离轴定理是如下内容：若两个凸物体没有重合，则总会存在一条直线，能将两个物体分离。这样的直线称为分离轴。对于高维物体来说这同样适用，只不过分离轴将变为分离超平面。

利用分离轴定理可以实现凸几何体的碰撞检测。对于多边形，算法是这样的：
1. 对两个多边形的每一条边，做垂直于该边的直线
2. 对两个多边形的每一个顶点，分别做到步骤1中所得直线的投影
3. 如果存在一条直线，两个多边形对这条直线的投影并不重合，则两个多边形不重合
4. 如果不存在步骤3中的直线，则两个多边形重合。

完全按照算法思路得到代码如下
```py
def sat(body_a, body_b):
    return not find_separate_axis(body_a, body_a, body_b) and not find_separate_axis(body_b, body_a, body_b)


def find_separate_axis(lines_body, body_a, body_b):
    for i in range(len(lines_body) - 1):
        p1 = lines_body[i]
        p2 = lines_body[i] + get_normal(lines_body[i+1] - lines_body[i])

        a_min, a_max = body_cast(p1, p2, body_a)

        b_min, b_max = body_cast(p1, p2, body_b)

        if p1[0] != p2[0]:
            max_min = a_min if a_min[0] > b_min[0] else b_min
            min_max = a_max if a_max[0] < b_max[0] else b_max
            if max_min[0] < min_max[0]:
                continue
        else:
            max_min = a_min if a_min[1] > b_min[1] else b_min
            min_max = a_max if a_max[1] < b_max[1] else b_max
            if max_min[1] < min_max[1]:
                continue

        return True

    return False


def body_cast(a, b, body):
    body_min = body_max = point_cast(a, b, body[0])
    for p in body:
        p_cast = point_cast(a, b, p)
        if p_cast[0] < body_min[0] or (p_cast[0] == body_min[0] and p_cast[1] < body_min[1]):
            body_min = p_cast
        if p_cast[0] > body_max[0] or (p_cast[0] == body_max[0] and p_cast[1] > body_max[1]):
            body_max = p_cast
    return body_min, body_max


def point_cast(a, b, p):
    u = p - a
    v = b - a
    p_cast = a + v * (np.dot(v, u) / np.dot(v, v))
    return p_cast
```

我们还需要得到接触点、碰撞法线和穿透深度。这需要对原算法进行微小的改造。

## 四、分离轴定理的优化
如果有重合部分，我们需要做的是记录所有投影的重合部分的长度，取重合长度最短的，这个长度即穿透深度；所投影到的直线的方向即为法线的正/反方向；而接触点就是重合部分其中一个端点所对应的顶点（另一个端点所对应的一定是一条边）。

另外我们也需要对算法进行优化，因为上面的代码将大部分步骤用于判断是否重合。实际上，引出直线的边已经是到直线的一个投影了；另外我们也不需要得到实际的投影位置，而只需要得到在这条直线方向上每个多边形的最远点即可。

首先，我们要引入求点到平面距离的方法，设平面上一点为 $s$，平面法向量 $n$，要求距离的点为 $t$ 。那么有
$$
    d = t \cdot n - s \cdot n
$$

```py
def get_point_to_line_distance(pos, normal, point):
    d = np.dot(pos, normal)
    return np.dot(point, normal) - d
```

这也是 box2d 中用到的方法。求得距离的同时还可以判断点在直线的方向。我们用这求点到直线的距离的同时，还可以判断点在多边形的位置。

接着，我们依旧要对每一条边求直线方向，但这次我们取法线的反方向为正方向，求另一个多边形上各点到这条边的距离。并取距离最大的顶点和对应的边。如果对某一条边，如果距离最大的顶点依旧小于零，则说明在这个方向上存在一条分离轴。

如果距离都大于等于零，那么找到这些顶点中距离最小的，这个顶点就是接触点、这个顶点对应的距离就是穿透距离、这个顶点所对应的边的法线就是碰撞法线。

```py
def sat2(body_a, body_b):
    e1, s1, d1 = get_separate_axis(body_a, body_b)
    e2, s2, d2 = get_separate_axis(body_b, body_a)

    if d1 < 0 or d2 < 0:
        return None

    if d1 < d2:
        return body_b[s1], get_normal(body_a[e1+1] - body_a[e1]), d1
    else:
        return body_a[s2], -get_normal(body_b[e2+1] - body_b[e2]), d2


def get_separate_axis(body_a, body_b):
    best_dist = np.Infinity
    best_edge_idx = -1
    best_support_point_idx = -1
    for i in range(body_a.shape[0] - 1):
        n = get_normal(body_a[i + 1] - body_a[i])

        support_idx, dist = get_support_point_idx(body_a[i], -n, body_b)

        if dist < best_dist:
            best_dist = dist
            best_edge_idx = i
            best_support_point_idx = support_idx

    return best_edge_idx, best_support_point_idx, best_dist


def get_support_point_idx(pos, normal, body):
    best_dist = -np.Infinity
    support_point_idx = -1
    for i in range(body.shape[0]-1):
        dist = get_point_to_line_distance(pos, normal, body[i])
        if dist > best_dist:
            best_dist = dist
            support_point_idx = i
    return support_point_idx, best_dist
```

## 五、最近内部顶点法
在现实当中，很多时候一个物体并非只有一个点与其他物体接触，而是一个平面都与其他物体接触。我们也可以认为这等同于有两个接触点（可以证明最多只有两个接触点）。因此我们还需要在原碰撞检测算法上进一步做改进。我们使用的方法可以是最近内部顶点法（这并非一个正式的名字）。

算法是在进行完分离轴定理，获得了第一个接触点等信息后进行的。具体步骤如下：
1. 在第一个接触点所在的多边形顶点中找出在碰撞法线方向上，距离对应边第二大的顶点（第一大的已为第一个接触点）
2. 判断该顶点是否在另一个多边形内部，如果是，则这个顶点也是一个碰撞点，算法结束
3. 否则，取碰撞法线对应边上两点，判断其是否在另一个多边形内部，如果在则加入接触点

关键的问题是如何判断点在多边形内部。我们假定多边形均为凸多边形，这样问题就有一个 $O(n)$ 的解法，不会对碰撞检测主体 $O(n^2)$ 的时间复杂度产生较大影响。具体来说，我们需要用到叉积的性质。

算法如下：我们遍历每一条边，设边为 $(e_i, e_{i+1})$，要判断的点为 $p$ 那么取边起点到终点的向量 $e_{i+1} - e_i$ 和起点到要判断的点的向量 $p - e_i$，将前者叉乘后者。如果点在多边形内，则所有叉积的结果符号应相同；反之若点不在多边形内，则叉积的结果符号应有不同。特别的，当多边形由逆时针的顶点序列所表示，所有的叉积结果大于等于零时点在多边形内。

我们可以简单地编程实现：

```py
def check_point_on_convex_polygon(point, poly_points):
    for i in range(poly_points.shape[0] - 1):
        right = poly_points[i + 1] - poly_points[i]
        left = point - poly_points[i]

        if np.cross(right, left) < 0:
            return False

    return True
```

这样我们就能继续实现“最近内部顶点法”了。具体代码如下：

```py
def sat3(body_a, body_b):
    e1, s1, d1 = get_separate_axis(body_a, body_b)
    e2, s2, d2 = get_separate_axis(body_b, body_a)

    if d1 < 0 or d2 < 0:
        return None

    if d1 < d2:
        edge_body = body_a
        s_point_body = body_b
        e, s, d = e1, s1, d1
        dir = 1
    else:
        edge_body = body_b
        s_point_body = body_a
        e, s, d = e2, s2, d2
        dir = -1

    contact_points = [(s_point_body[s], d)]

    second_point = closest_internal_vertices_method(edge_body, s_point_body, e, s)

    if second_point is not None:
        contact_points.append(second_point)

    return dir * get_normal(edge_body[e + 1] - edge_body[e]), contact_points


def closest_internal_vertices_method(edge_body, s_point_body, e_idx, s_idx):
    n = get_normal(edge_body[e_idx + 1] - edge_body[e_idx])
    s_point_num = s_point_body.shape[0] - 1
    dist1 = get_point_to_line_distance(edge_body[e_idx], -n, s_point_body[(s_idx - 1) % s_point_num])
    dist2 = get_point_to_line_distance(edge_body[e_idx], -n, s_point_body[s_idx + 1])

    if dist1 > dist2 and check_point_on_convex_polygon(s_point_body[(s_idx - 1) % s_point_num], edge_body):
        return s_point_body[(s_idx - 1) % s_point_num], dist1
    elif dist2 > dist1 and check_point_on_convex_polygon(s_point_body[s_idx + 1], edge_body):
        return s_point_body[s_idx + 1], dist2

    if check_point_on_convex_polygon(edge_body[e_idx], s_point_body):
        return edge_body[e_idx], 0

    if check_point_on_convex_polygon(edge_body[e_idx + 1], s_point_body):
        return edge_body[e_idx + 1], 0
```

> 需要注意的是这一方法无法求得每一个接触点的穿透深度。

## 六、总结
本篇文章介绍了一个碰撞检测所用的算法，但是这只是物理引擎使用的碰撞检测算法中很小的一部分，因本人的精力、能力以及篇幅所限，还有许多更加精妙有效的算法并未涉及。就实现一个简单的2d物理引擎来说，本文所讲的算法已经足够了。