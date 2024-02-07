+++
title = "对不重复随机数的数学分析"
tags = ["数学", "C"]
categories = ["算法"]
aliases = ["/posts/f44f8530"]
date = "2022-11-13T21:03:39+08:00"
+++
## 一、引子——双色球
我们知道，双色球每注投注号码由 6 个红色球号码和 1 个蓝色球号码组成。红色球号码从 1—33 中选择；蓝色球号码从 1—16 中选择。现在要求写出一个程序，模拟双色球的抽奖过程。

我们很容易想到使用某种方法生成一定范围内的随机数。蓝色球很好解决，但对于红色球，需要的是随机生成 6 个号码不同的数，可一定范围内的随机数总可能出现相同的情况，这样要如何解决？

也就是说：对于 $1, 2, ..., m$ 这 m 个数字，随机抽取其中 $n(n \lt m)$ 个数。要采取怎样的算法？

## 二、两种思路
### 其一 : 暴力方法
的确，随机数总有可能出现相同的情况，但是我们知道，同一个数多次出现的概率很小，以至于我们可以将其忽略。因此，我们只需要不断地取范围内的随机数，遇到重复的舍弃，直到取得的数字数目达到 n 即可。
```c
void rand1(int m, int n, int rands[])
{
    char* hasSelect = calloc(m+1, sizeof(char));
    for (int i = 0; i < n; )
    {
        int r = randomInt(1, m);
        if (!hasSelect[r])
        {
            rands[i++] = r;
            hasSelect[r] = 1;
        }
    }
    free(hasSelect);
}
```

但这种算法是有缺陷的。问题在于，概率达到多小才算作可以忽略？
考虑 $m=100, n=99$ 的情况。首先，取得第一个球，一定只需要选取 1 次；但是，我们再计算一下取得最后一个球的选取次数。为了取得第 99 个数，选中的概率为 $p_{99} = \frac{1}{50}$。我们设离散型随机变量 $X$ 表示为了取得第 99 个数所需的选取次数，则 $P\{X = k\} = (1-p)^{k-1}p$ 服从几何分布。因此 $E(X) = \frac{1}{p} = 50$。从期望上看，需要整整 50 次才能取到第 99 个数！！这中间的差距说明了，在 m 一定时，随着 n 的增大，随机的效率明显降低。

### 其二 : 洗牌算法
还从双色球这一实际问题来看，我们要从 33 个球中取得 6 个球，实际上是从球的一个随机排列中取得前 6 个球。这样，我们的问题就等价于找到一种将序列打乱顺序，使每一个数在每一个位置的概率相等的算法。

这里介绍 Knuth 的洗牌算法，如下：
> 在整个数组 [0, n-1] 中（包括最后一个元素）随机选出一个元素，将它和最后那个元素 [n-1] 交换，然后再在数组 [0, n-2] 中随机选出一个元素，将它与倒数第二个元素 [n-2] 交换…一直到最后一个元素。

洗牌之后，我们再取前 n 个数，就求得了 n 个不重复随机数。

```c
void rand2(int m, int n, int rands[])
{
    int* nums = calloc(m, sizeof(int));
    for (int i = 0; i < m; i++)
        nums[i] = i+1;

    for (int i = m-1; i >= 0; i--)
    {
        int j = randomInt(0, i);
        int tmp = nums[i];
        nums[i] = nums[j];
        nums[j] = tmp;
    }

    for (int i = 0; i < n; i++)
        rands[i] = nums[i];
    free(nums);
}
```

从直觉上看，这种算法的用时是与 n 无关的。在 n 足够小时，暴力方法的用时会小于该算法。我们需要对着两种方法进一步分析，以做权衡。

## 三、数学上的分析
我们首先需要解决的问题是，对于序列 $1, 2, ..., m$ 我们随机放回抽取，求当取得 n 个不同的数时所用次数的数学期望。

我们设随机变量 $X_i$ 表示已经取到了 $i-1$ 个不同的数后，还需要多少次才能取到第 $i$ 个不同的数。并设随机变量 $T_n$ 表示取得 n 个不同的数所需要的总次数，即 $T_n = \sum_{i=1}^n X_i$。

在已经取得 $i-1$ 个不同的数的条件下，取一次，不属于已经取到过的数的概率为
$$
    p_i = \frac{m-i+1}{m}
$$
因 $X_i$ 服从几何分布, $P\{X_i = k\} = (1-p_i)^{k-1}p_{i}$ 则有
$$
    EX_i = \sum_{k=1}^{\infty} k (1-p_{i})^{k-1}p_{i}
    = -p_{i}[\sum_{k=1}^{\infty} (1-p_{i})^{k}]^{'}
    = \frac{1}{p_i} = \frac{m}{m-i+1}
$$

又因随机变量序列 $\{X_n\}$ 相互独立，则
$$
    ET_n = E(\sum_{i=1}^n X_i) = \sum_{i=1}^nEX_i
    = m \sum_{i=1}^n \frac{1}{m-i+1}
    = m(\sum_{i=1}^m \frac{1}{i} - \sum_{i=1}^{m-n} \frac{1}{i})
$$
即
$$
    ET_n = m(\sum_{i=1}^m \frac{1}{i} - \sum_{i=1}^{m-n} \frac{1}{i})
    = m(H_m - H_{m-n})
$$
其中 $H_m, H_{m-n}$ 分别表示第 m，m-n 个调和数（Harmonic number）。

又因 
$$
    H_n = \ln(n) + \gamma + \frac{1}{2} + O(\frac{1}{n}) \\
    \gamma \approx 0.57721 56649
$$
则
$$
    ET_n = -m \ln({1-\frac{n}{m}}) + O(\frac{1}{m})
$$
> 由上式可知，$n \ll m$ 时，$ET_n \approx 0$。这是符合直觉的。但 $m =n$ 的情况却并不满足上式子，因为并不存在 $H_0$。 $m=n$ 时的公式应为
$$
    ET_n = mH_m = m\ln(m) + \gamma + \frac{1}{2} + O(\frac{1}{m})
$$

此暴力算法的时间复杂度为 $O(-m\log(1-\frac{n}{m}))$ （认为 n 会小于 m）。

而对于洗牌算法，其时间复杂度为 $O(m)$。

要使第一种算法用时少于第二种，则需要
$$
    -m\log(1-\frac{n}{m}) \lt m
$$

得
$$
    n \lt (1-\frac{1}{e})m \approx 0.632m
$$

可见在一些情况下，暴力算法能取得很好的效果。但在 $n \ge 0.632m$ 的时候，或许采用洗牌算法效率更高。
> 当然，由于第二种算法的系数更高，所以实践中的情况会更为复杂。

## 四、代码
```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int randomInt(int from, int to)
{
    return rand() % (to - from + 1) + from;
}

void rand1(int m, int n, int rands[]);
void rand2(int m, int n, int rands[]);

int main()
{
    srand(time(0));
    int n = 5;
    int rands[10];
    rand1(10, n, rands);
    for (int i = 0; i < n; i++)
        printf("%d ", rands[i]);
    printf("\n");
    rand1(10, n, rands);
    for (int i = 0; i < n; i++)
        printf("%d ", rands[i]);
}

void rand1(int m, int n, int rands[])
{
    char* hasSelect = calloc(m+1, sizeof(char));
    for (int i = 0; i < n; )
    {
        int r = randomInt(1, m);
        if (!hasSelect[r])
        {
            rands[i++] = r;
            hasSelect[r] = 1;
        }
    }
    free(hasSelect);
}

void rand2(int m, int n, int rands[])
{
    int* nums = calloc(m, sizeof(int));
    for (int i = 0; i < m; i++)
        nums[i] = i+1;

    for (int i = m-1; i >= 0; i--)
    {
        int j = randomInt(0, i);
        int tmp = nums[i];
        nums[i] = nums[j];
        nums[j] = tmp;
    }

    for (int i = 0; i < n; i++)
        rands[i] = nums[i];
    free(nums);
}
```