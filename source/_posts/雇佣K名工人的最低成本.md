---
title: 雇佣K名工人的最低成本
tags:
  - LeetCode
  - 优先队列
  - 贪心
  - c++
categories: 算法
mathjax: true
abbrlink: '22319630'
date: 2022-09-11 09:02:32
---
## 857. 雇佣 K 名工人的最低成本
> 有 n 名工人。 给定两个数组 quality 和 wage ，其中，quality[i] 表示第 i 名工人的工作质量，其最低期望工资为 wage[i] 。
>
> 现在我们想雇佣 k 名工人组成一个工资组。在雇佣 一组 k 名工人时，我们必须按照下述规则向他们支付工资：
>
> 对工资组中的每名工人，应当按其工作质量与同组其他工人的工作质量的比例来支付工资。
> 工资组中的每名工人至少应当得到他们的最低期望工资。
> 给定整数 k ，返回 组成满足上述条件的付费群体所需的最小金额 。在实际答案的 10-5 以内的答案将被接受。。

>输入： quality = [10,20,5], wage = [70,50,30], k = 2
>输出： 105.00000
>解释： 我们向 0 号工人支付 70，向 2 号工人支付 35。

## 解答
### 问题分析
假设选择某一组工人$[h_1, h_2,...h_m]$，设总工资为$totalpay$，则对每一个工人$h_i({1}\le{i}\le{m})$，有不等式：
$$
\frac{quality[h_i]}{\sum_{i=1}^{m}{quality[h_i]}}\times{totalpay}\ge{wage[h_i]}
$$
即：
$$
    totalpay = {\sum_{i=1}^{m}{quality[h_i]}}\times{max(\frac{wage[h_i]}{quality[h_i]})}
$$
$\frac{wage[h_i]}{quality[h_i]}$即为每个工人工作的“性价比”。

由等式可知，对于一组确定的工人，除了总工作质量外，其总工资只与“性价比”最高的那个工人有关。

因此可以得到这样的算法：对于每一个工人，在“性价比”低于该工人的人中找到k-1个人，使得总工作质量最小。求得当前的总工资。最后取所有得到的总工资中的最小值。

### 程序实现
我们需要找到“性价比”在倒数k-1名之前的所有工人，不妨先按照性价比进行排序。
``` c++
int n = wage.size();
int wpq[n]; // wage per quality
for (int i = 0; i < n; i++)
    wpq[i] = i;
sort(wpq, wpq+n,
    [&](int i, int j)
    { return wage[i] * quality[j] < wage[j] * quality[i];});
```
注意这里并未对quality和wage直接排序，而是将下标进行排序。

之后，再分析接下来需要进行的操作。对于每个工人，都要找到他（性价比中的位置）之前的k-1个工人，这些工人需要总工作质量最小。具体的，这些工人就是工作质量最小的k-1个人。

保存最大/最小的k个元素，可以利用优先队列。
``` c++
priority_queue<int> pq;
int qsum = 0;
for (int i = 0; i < k-1; i++)
{
    int nowq = quality[wpq[i]];
    pq.push(nowq);
    qsum += nowq;
}
```
先将前k-1个人压入队列，那么对于前k-1个人，他们一定是工作质量最小的k-1个人。
``` c++
double ans = DBL_MAX; 
for (int i = k-1; i < n; i++)
{
    int work = wpq[i];
    int nowq = quality[work];
    pq.push(nowq);
    qsum += nowq;
    double nowPay = qsum * ((double)wage[work] / nowq);
    ans = min(ans, nowPay);
    qsum -= pq.top();
    pq.pop();
}

return ans;
```
然后从第k个工人开始，求最小的总工作质量。具体的，压入第i个工人，再从优先队列中弹出工作质量最大的那个。易证操作后队列里的k个依旧是最小的k个。

## 后记
话说这道题明明有思路，想到了“性价比”，也想到了优先队列，可是就是没做出来，实在是对不起自己了。现在想想，好像也没有什么困难的地方，或许只是自己被概念缠住了吧。有时候列一列公式还是很有必要的。