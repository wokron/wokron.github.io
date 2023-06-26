---
title: 用pytorch实现简单循环神经网络
tags:
  - AI
  - 机器学习
  - pytorch
  - rnn
categories: 学习笔记
abbrlink: e7ad966d
date: 2023-01-17 21:02:34
---
## 一、歌词生成项目
想要在 nlp 方面深入，于是选择训练生成一个 RNN 网络，主要目标是自动生成歌词。在这里受到了 [最浅显易懂的 PyTorch 深度学习入门](https://www.bilibili.com/video/BV1oq4y1E7Vd) 的启发，并利用 up 主提供的 [源码](https://github.com/rossning92/ai-lyrics-writing) 中的数据集。

相关代码的编写也有参考该 up 主的部分，但均为在理解内容的基础上自行编写的。另外也有对该 up 主代码中的疏漏进行修改的地方。

## 二、数据的获取
### （1）编写数据集
原作者用爬虫获取的歌词数据被保存在 lyrics.txt 文件中。我们要将数据按可供训练的模型加载。具体来说
- 我们希望每一次获取数据，都能得到输入和目标输出（对本项目来说就是两段有一个文字偏差的序列）
- 并且将文字数字化，即 nlp 的 tokenize
- 为了实现批量训练，需要每次获取定长的序列

为了实现第一点，我们要继承 dataset；实现第二点，需要根据数据建立字符表；实现第三点，需要定长截取歌词句子的一部分。
>另外，为了减少每次加载数据所用的时间，还需要将数据集的信息持久化。

我们的 LyricsDataset 具体实现如下。首先，我们在构造函数中通过传入的路径加载数据，判断是否已经存在处理过后的数据，如存在则加载；如不存在则读入原始数据并处理

```py
class LyricsDataset(Dataset):
    def __init__(self, root_path, seq_size):
        self.seq_size = seq_size

        processed_name = "/processed/lyrics.pth"
        raw_name = "/raw/lyrics.txt"

        if os.path.exists(root_path + processed_name):
            print("find processed data")
            self.__load_processed_data(root_path + processed_name)
        else:
            print("processed data not found, will process raw data")
            self.__load_raw_data(root_path + raw_name)
            self.__save_processed_data(root_path + processed_name)

    def __load_raw_data(self, file_path):
        with open(file_path, encoding="utf-8") as f:
            sentences = f.readlines()

        self.vocab = {"<sos>": 0, "<eos>": 1}
        self.data = []

        for sentence in sentences:
            sentence = sentence.strip()
            self.data.append(self.vocab["<sos>"])
            for char in sentence:
                if char not in self.vocab:
                    self.vocab[char] = len(self.vocab)
                self.data.append(self.vocab[char])
            self.data.append(self.vocab["<eos>"])

        self.index2word = {u: v for v, u in self.vocab.items()}

    def __save_processed_data(self, file_path):
        torch.save({"data": self.data, "vocab": self.vocab}, file_path)

    def __load_processed_data(self, file_path):
        data_dict = torch.load(file_path)
        self.data = data_dict["data"]
        self.vocab = data_dict["vocab"]
        self.index2word = {u: v for v, u in self.vocab.items()}
```
我们将所有的字转化为数值并按顺序添加到 `data` 属性中，这将有助于后续操作。另外我们还提供了由数值转化回字的字典，这是翻转（nlp 意义下的）字典的键值得到的。

在保存和加载数据的时候，我们使用了 `torch` 的 `save()` 和 `load()` 方法，这些方法提供了简单的序列化和反序列化功能，并非只能保存模型的参数。另外在保存模型的时候，也可以通过字典同时保存一些附加信息。这将在之后有所体现。

> 这里还可以补充两个 python 中的知识。
> 1. python 中也有类方法的访问修饰，在方法名前有两个下划线的是 `private`，一个下划线的是 `protected`，没有下划线的是 `public`
> 2. for 可用于容器中，如 `arr2 = [i+1 for i in arr]`，可用于对容器中所有元素做某种处理并保存在新容器中。代码中有使用 `index2word = {u: v for v, u in self.vocab.items()}`。注意对于字典需要调用 `items()` 以表示对键值对的操作，否则默认为对键集合操作。

接着一如往常，我们要实现 `__getitem__` 和 `__len__`。

```py
    def __getitem__(self, item):
        start = item * self.seq_size + 1
        end = start + self.seq_size

        input_seq = self.data[start:end]
        output_seq = self.data[start + 1:end + 1]

        return torch.tensor(input_seq), torch.tensor(output_seq)

    def __len__(self):
        return int((len(self.data) - 1) / self.seq_size)
```

在初始化的时候，我们传入了 `seq_size` 参数用于指定一条文字转化而成的序列的长度。在通过下标读取数据的过程中，我们也以序列作为基本的数据单元。另外请注意，我们为了方便后续操作，将序列转化为了一维的张量（`[seq_size]`）。

在编写获取长度的函数时，原作者的实现似乎除了些问题。这里指示的长度应该是总共能获取的序列的个数，肯定不能仅仅 `len(self.data)` 就可以了。应该是字符的总数除以序列的长度向下取整。`len(self.data) - 1` 考虑了输入和目标一个字符的偏差。

### （2）划分数据集
我们现在能获取数据集了，可是这个数据集包含了所有数据，我们还希望划分出训练集和测试集。`torch` 提供一个随机划分的方法 `random_split`，但这无法在程序多次执行时保持不变。不过我们可以选择将划分后的数据集保存。这里我们定义一个函数

```py
def load_or_split_dataset(dataset_root, dataset, lengths: list):
    processed_path = dataset_root + "/processed"
    train_set_path = processed_path + "/train.pth"
    test_set_path = processed_path + "/test.pth"

    if not os.path.exists(train_set_path) or not os.path.exists(test_set_path):
        print("train and test datasets not found, start random split")
        train_set, test_set = random_split(dataset, lengths)
        torch.save(train_set, train_set_path)
        torch.save(test_set, test_set_path)
    else:
        print("find train and test datasets")
        train_set = torch.load(train_set_path)
        test_set = torch.load(test_set_path)
    return train_set, test_set
```
就可以同时实现划分或加载了
```py
train_set, test_set = load_or_split_dataset(
    dataset_root
    lyrics_dataset,
    [train_set_size, test_set_size])
```

### （3）加载数据
```py
train_loader = DataLoader(train_set, batch_size, drop_last=True)
test_loader = DataLoader(test_set, batch_size, drop_last=True)
```
按照常规创建 `DataLoader`。注意需要加上 `drop_last=True`，否则 RNN 网络可能会因为无法批处理而报错，类似
```text
RuntimeError: Expected hidden[0] size (2, 20, 256), got (2, 50, 256)
```

此时每次从 DataLoader 中获取的数据为两个二维张量（`[batch_size, seq_size]`）

## 三、网络的搭建
先给出总体的模型结构，这个模型只使用了 LSTM 循环神经网络加上线性层作为输出，结构较为简单，与原作者的模型相比变化不大。
```py
class LyricsModule(nn.Module):
    def __init__(self, vocab_size, embed_size, hidden_size, num_rnn_layers):
        super().__init__()

        self.embed = nn.Embedding(vocab_size, embed_size)
        self.rnn = nn.LSTM(embed_size, hidden_size, num_rnn_layers, batch_first=True)
        self.linear = nn.Sequential(
            nn.Linear(hidden_size, hidden_size),
            nn.Linear(hidden_size, vocab_size)
        )

    def forward(self, input, hidden=None):
        word_vec = self.embed(input)
        x, hidden_next = self.rnn(word_vec, hidden)
        predict = self.linear(x)
        return predict, hidden_next
```

接下来要分层对该模型进行介绍。

### （1）Embedding 层的使用
在输入的一开始，我们使用了一层 `Embedding` 层。
```py
        self.embed = nn.Embedding(vocab_size, embed_size)
```
创建 `Embedding` 层需要两个必须的参数，`num_embeddings` 和 `embedding_dim`。这一层的作用，本质上是创建一个数值到向量的映射，`num_embeddings` 即为映射最多的数量；效果上，一是给传入的张量升高一个维度，维度的大小为 `embedding_dim`，二是给每个词指定一个词向量，这个词向量还可以在反向传播的过程中不断调整。

对于传入模型的二维张量，经过 `Embedding` 层后变为三维张量（`[batch_size, seq_size, embedding_dim]`）

### （2）LSTM 层的使用
```py
        self.rnn = nn.LSTM(embed_size, hidden_size, num_rnn_layers, batch_first=True)
```
LSTM 的数学表示在上一篇文章中就有说明，这里只看 pytorch 中 LSTM 的使用。`LSTM` 层输入一个三维张量，输出一个同等大小的三维张量，以及隐藏层。`LSTM` 有两个必须的参数 `input_size` 和 `hidden_size` ，分别指输入的特征数（第三维大小）和隐藏层的维度大小。在本模型使用时还使用了一个可选参数 `num_layers` 表示 `LSTM` 的层数。

有几点需要注意的地方。
- 如果不添加 `batch_first=True`，则 `LSTM` 层默认第一位为 seq_size，第二维才是 batch_size。添加后才为 `[batch_size, seq_size, embedding_dim]`。
- `LSTM` 层以一个序列为一个输入单元，输出时就已经进行过 seq_size 次循环了。因此在训练的时候，不需要保存每一步后的隐藏状态并传递给下一步。
- 每个隐藏状态是一个三维张量（`[batch_size, seq_size, hidden_size]`）表示对这一批中的不同序列，在经过第几次循环之后的隐藏状态。另外因为 LSTM 有两个隐藏状态 h_t 和 c_t，返回的隐藏状态实际上是一个二元组 `(h_t, c_t)`。

### （3）模型的输出
我们的目标是歌词的生成，也就是根据前一个字符判断下一个字符应该生成哪一个。那么这就需要对每一个字符的输入，对应一个字典字符数目维数的输出，选择最大的数值所对应的字符。也正是因为训练之前我们就需要确定确定的维数，所以要事先确定字典。

对于张量来说，我们就是需要一个三维张量，其中第三维的维数为字典的字符总数。即 `[batch_size, seq_size, vocab_size]`

我们增加的线性层就将输出如此规范：
```py
        self.linear = nn.Sequential(
            nn.Linear(hidden_size, hidden_size),
            nn.Linear(hidden_size, vocab_size)
        )
```
> `LSTM` 层的输出为二元组，因此不能用 `Sequential` 直接包含

## 四、更完善的训练结构
在这一部分，我们希望优化代码，尽量减少每次训练时所需要的对代码的修改。首先，这需要及时对训练模型进行保存；其次我们还需要设置全局的 epoch。

### （1）设置检查点
我们要用 `torch.save()` 保存模型和优化器的参数，这些保存的信息称为检查点（checkpoint）。为了方便确定检查点保存模型的先后顺序，我们可以将日期加到文件的命名中。

我们定义如下函数用来保存检查点
```py
def save_checkpoint(root_path, module: Module, optimizer: Optimizer):
    checkpoint_name = "checkpoint-{}.pth".format(datetime.now().strftime("%y%m%d-%H%M%S"))
    torch.save(
        {
            "module_state_dict": module.state_dict(),
            "optim_state_dict": optimizer.state_dict()
        },
        root_path + "/" + checkpoint_name
    )
    print(f"save checkpoint file \"{checkpoint_name}\" success")
```
注意我们在确定检查点名字时获取了当前时间 `datetime.now().strftime("%y%m%d-%H%M%S")`

> 这里 `torch.save()` 保存的是一个字典

接着我们定义函数以加载检查点数据
```py
def load_checkpoint(checkpoint_path):
    checkpoint_info = torch.load(checkpoint_path)
    return checkpoint_info
```

通过检查点名称，我们可以找到最近一次训练后的数据，定义加载最近的检查点函数
```py
def load_latest_checkpoint(root_path):
    checkpoints = glob.glob(root_path + "/checkpoint-*.pth")
    return load_checkpoint(checkpoints[-1])
```

这样我们每次训练时就可以加载之前最近一次训练的数据，继续训练了。

```py
lyrics_module = LyricsModule(vocab_size, embed_size, hidden_size, num_layers).to(device)

optimizer = torch.optim.Adam(lyrics_module.parameters(), learning_rate)

if checkpoint_exist(checkpoint_root):
    print("find exist checkpoint, load checkpoint information")
    checkpoint_info = load_latest_checkpoint(checkpoint_root)
    lyrics_module.load_state_dict(checkpoint_info["module_state_dict"])
    optimizer.load_state_dict(checkpoint_info["optim_state_dict"])
```

### （2）连续的训练过程
我们通常使用 tensorboard 查看损失函数和准确度等参数的变化趋势。但是如果每次训练时的步数都从头开始，tensorboard 的图像就会交织在一起，无法分辨。因此我们需要确定全局的步数。

我们在检查点中添加 `total_epoch`  参数，表示该检查点所保存的是几轮训练后的数据。在加载检查点是，`total_epoch` 也会一并加载。
```py
    torch.save(
        {
            "total_epoch": total_epoch, # here!
            "module_state_dict": module.state_dict(),
            "optim_state_dict": optimizer.state_dict()
        },
        root_path + "/" + checkpoint_name
    )
```

这样我们就能在训练中确定当前的全局轮数
```py
for i in range(epoch_size):
    epoch = i + 1
    now_total_epoch = total_epoch + epoch
    print(f"--- start epoch {epoch}, total epoch {now_total_epoch}")
```
在训练中确定全局的训练步数
```py
        global_train_step = (now_total_epoch - 1) * total_train_step + train_step
        writer.add_scalar("train loss", loss.item(), global_train_step)
        writer.add_scalar("train accuracy", accuracy, global_train_step)
```

同样的在测试中
```py
    writer.add_scalar("test loss", avg_loss, now_total_epoch)
    writer.add_scalar("test accuracy", avg_accuracy, now_total_epoch)
```