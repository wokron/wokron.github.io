+++
title = "机器学习 NLP 基本知识"
tags = ["数学", "NLP"]
categories = ["人工智能"]
aliases = ["/posts/b2ab0f69"]
date = "2023-01-17T21:01:27+08:00"
+++
## 一、自然语言处理（nlp）简介
**一份思绪奔驰的前言：**
语言的边界就是思想的边界。如果从人类所具有的一切中挑出一个事物，让它来显示出人与其他生物的不同之处，那一定是语言。语言是我们用来思考和交流的方式，我们的一切文明都构筑在这简单的一维序列中，可我们却几乎不曾深入了解过它。

有人说，我们不曾真正的理解语言，我们所有对语言的运用和理解，不过建立在婴儿及此后对他人的声音与书本上符号的猜测上。或许正是如此，但是我们依旧要做出猜测，向着语言的神秘进发，这是我们的信念，是我们认识我们的认识的开始。

### （1）作为语言学的 nlp
早在计算机的上古时代，深度学习还未诞生的时候，自然语言处理作为语言学的一个领域就已诞生了，这个领域也被语言学家们称为计算机语言学。两个名称在一起，才表达出 nlp 的真正含义——通过机器处理语言。nlp 最早的研究方向是机器翻译，那时人们人为地总结语言的规律，对词汇进行标注，对语句进行句法分析。结果是人为的规则覆盖面不足，所设计的系统无法扩展。

### （2）作为机器学习研究领域的 nlp
随着计算机的发展，出现了基于传统统计学习模型的自然语言处理方式。这些原始的模型较之之前有所进步，但受限于计算机性能，统计方法也遭遇了瓶颈。

直到近年来算力的发展，使得深度神经网络成为可能。深度神经网络结构中潜在的学习能力，在 nlp 领域发挥了作用。通过多维数据表示语言和含义，深度学习以高效且与人类认知过程相似的方式发挥了巨大的效果。

## 二、词向量
词汇作为符号，其形象是离散的；但词汇的所指作为定义，其含义却是丰富而连续的。比如说“母亲”这个词汇，既表示了这个概念所对应的事物是在一种血缘关系中的一方（她是孩子的母亲）；又表示了这个事物是能繁殖者（母鸡）；在一定程度上，同样表示了非血缘关系，但具有类似血缘关系的行为的个体（大地是母亲）。
> 偏个题，《来自深渊》中有对生骸语的类似的描述.

因此，我们就不能再将词汇只作为离散的符号看待了，不能认为词汇之间是相互排斥的关系了。我们需要将词汇看做某些元含义在不同程度下的集合，或者从机器学习的角度，把这些元含义称作特征。那么也就是说，我们将每个词汇都看成一定维度的向量。

但是我们要如何确定特征呢？特征的数量又有多少？如果人工地确定特征为“存在、含义、物质、精神”等等，这一过程将耗费精力且永无止境。实际上按照机器学习的一般策略，我们只需要通过统计文本，自发的构建词汇向量即可。

这一方面有许多算法，如 N-gram 算法，GloVe 算法等等。另外也可以在深度学习的过程中利用反向传播自发的调整词向量，在 pytorch 中这通过 Embedding 层来实现。

## 三、循环神经网络（RNN）及其变体
### （1）朴素 RNN
考虑我们说话或写作时的基本逻辑。对于一段语言序列，在之后的词汇总是和之前的词汇有关，未表达的部分总是已表达部分的补全或补充。循环神经网络的机制类似，我们需要用一个或多个隐藏变量作为对之前语句含义的表示，在输出下一个词汇时，会让隐藏变量参与决策；同时每多说完一个词汇，这个词汇也会更新隐藏变量，以实现表达含义的更新。

具体来说，我们用 $t$ 表示时间序列，对某一时间 $t$，$x_t$ 表示输入，$y_t$ 表示输出，$h_t$ 表示隐含状态。那么朴素的 RNN 网络即：
$$
    h_t = tanh(W^{(hx)} x_t + W^{(hh)} h_{t-1}) 
$$

$$
    y_t = W^{(S)} h_t
$$

> 容易看出 RNN 和 Moore 自动机有相似之处。

其中 $W^{(hx)}, W^{(hh)}, W^{(S)} h_t$ 分别为三个不同的矩阵。RNN 的激活函数也可以选择 `ReLU`。同时可以为激活函数中的部分添加偏置（bias）$b^{(hx)}, b^{(hh)}$ 等。

### （2）GRU
但是简单的 RNN 也存在着问题，那就是其结构不能很好地控制对不同词汇的关注程度以及当前内容和上文的关系。我们希望设计一种机制，在 RNN 的基础上做到对上下文关系的更好掌握。其中一种机制就是 GRU（Gate Recurrent Unit，门循环单元），它是对 RNN 的推广。

首先我们定义运算符 $\circ$，$k \circ M$，其中 $k$ 为数字 $M$ 为矩阵，表示对 $M$ 中每个元素均乘以 $k$。

那么对某一时间 $t$，$x_t$ 表示输入，$h_t$ 表示隐含状态，我们定义更新门 $z_t$ 和重置门 $r_t$
$$
    z_t = \sigma (W^{(z)} x_t + U^{(z)} h_{t-1})
$$

$$
    r_t = \sigma (W^{(r)} x_t + U^{(r)} h_{t-1})
$$

再令
$$
    \tilde{h}_t = tanh (W x_t + r_t \circ U h_{t-1})
$$

$$
    h_t = z_t \circ h_{t-1} + (1 - z_t) \circ \tilde{h}_t
$$

我们就定义了隐含状态的更新方式，求输出的方式与 RNN 相同。

所以 GRU 到底做了什么呢？GRU 实际上等同于在 RNN 上增加了两个连续的开关 $z_t$ 和 $r_t$。注意这两个变量由 Sigmoid 函数得到，他们的范围是 0~1。我们可以令 $z_t = 0, r_t = 1$，此时就有 $h_t = tanh (W x_t + U h_{t-1})$，这与 RNN 中相同。

与 RNN 相比，我们在学习过程中多训练了两个开关。它们的取值暗示了语句中的某些结构。直观上讲，重置门 $r_t$ 反映了当前词汇与上文含义之间的关系和相互作用，当 $r_t = 0$ 时，当前词汇的含义不会受到上文意思的影响。而更新门 $z_t$ 则反映了当前词汇对整体语义的影响，当 $z_t = 0$ 时，更新后的语义将只与当前词汇的意思有关。

更简单来说：
- $r_t$ 反映了上文对当前词汇的作用程度（如 “你妈" 的意思受到语境的影响）
- $z_t$ 反映了当前词汇对整体语义的影响程度（如词汇 “但是” 之后的部分更能反映整体语义）

### （3）LSTM
我们还可以对 RNN 模型做相似但不同的改动，形成 LSTM （Long Short-Term Memory，长**短期**记忆）模型，这将使模型更加复杂，但同时也使得对前文语义的记忆得以更加长久的保存。

与 GRU 类似，首先我们定义三个门，分别是输入门 $i_t$，遗忘门 $f_t$ 和输出门 $o_t$。
$$
    i_t = \sigma (W^{(i)} x_t + U^{(i)} h_{t-1})
$$

$$
    f_t = \sigma (W^{(f)} x_t + U^{(f)} h_{t-1})
$$

$$
    o_t = \sigma (W^{(o)} x_t + U^{(o)} h_{t-1})
$$

再定义记忆格（memory cell，或记忆细胞？）$c_t$，计算方式为
$$
    \tilde{c}_t = tanh(W^{(c)}  x_t + U^{(c)} h_{t-1})
$$

$$
    c_t = f_t \circ c_{t-1} + i_t \circ \tilde{c}_t
$$

最后定义隐藏状态
$$
    h_t = o_t \circ tanh(c_t)
$$

LSTM 的特点是采用了两个隐藏状态 $c_t$ 和 $h_t$。从直觉上、或哲学上讲，这反映了两种不同的意义。一是作为表层的，被表达出来的语句的语义；另一是作为深层的，作为记忆的整体的意味。当我们在诉说或理解时，我们首先所接触到的是作为连续序列的语义，这一浅表的概念为了被更深入的理解，它就进入到记忆的层次，并再次从其中返回，在记忆上留下痕迹的同时带来新的表达并言说或写作下来。因此，我们可以将各个门理解为：
- 输入门反映了新的词汇对记忆的影响
- 遗忘门反映了旧的记忆在语言环境的更新下被抛弃的程度
- 输出门反映了由记忆到能言说的语言的转变的过程。

从这一模型中似乎也能体会出流利但没有内涵的口才与深思熟虑的话语间的差别。

### （4）注意力机制
我们试图让模型拥有更长的记忆，但是不管什么模型，应用什么样的隐藏状态，能存储的信息终归是有限的。只要时间序列足够长，总归会忘记最开始的内容。

注意力机制能够解决这一问题。它的想法是，既然当下的隐藏状态中不包含对很久之前内容的记忆，那我只要选择包含之前内容的隐藏状态不就可以了吗？但是为了保持模型的可微性，我们不能仅仅离散的选择一些包含需要的内容的前面的隐藏状态，而是按照一定方法依据权重，选择在此之前的所有的隐藏状态。

> 这实际上形成了一种类似于内存访问的机制。只不过内存是离散的地址和数据，而注意力机制却是连续的“地址”和“数据”。也有依照这一对应关系进一步推广的研究，如神经图灵机，但在此不做赘述。

注意力机制本身的实现如下。首先我们要以前一个隐藏状态 $h_{t-1}$ 作为评价其他更早的隐藏状态 $\overline{h}_s$ 的基准。对每一个 $\overline{h}_s$ 实施评价函数
$$
    score(h_{t-1}, \overline{h}_s)
$$

接着运用 softmax 函数获取百分比权重
$$
    a_t(s) = \frac{e^{score(s)}}{\sum_{s'}score(s')}
$$

然后按照权重对隐藏状态进行合成即可。
$$
    c_t = \sum_s a_t(s) \overline{h}_s
$$

对于评价函数，可以有多种选择，其中比较好的一种为
$$
    score(h_{t-1}, \overline{h}_s) = h_t^T W_a \overline{h}_s
$$

$W_a$ 也可以在训练的过程中不断学习优化。