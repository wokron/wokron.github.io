+++
title = "用 Pytorch 搭建神经网络"
tags = ["Pytorch"]
categories = ["人工智能"]
aliases = ["/posts/2c9af791"]
date = "2023-01-02T15:52:55+08:00"
+++
## 一、前言
本文作为自己学习 pytorch 的记录。以搭建一个神经网络为例，介绍 pytorch 的基本使用。

本文不会讲 conda、python、pycharm 等的配置和使用，也不会讲各神经层的原理及使用。只是按照自己之前学习的理解，总结神经网络训练的基本流程。

本文所使用的例子是自己写的第一个神经网路，如下：
```py
import os.path
import torch.utils.data
from torch.utils.data import DataLoader
from torch import nn
from torch.utils.tensorboard import SummaryWriter
from torchvision import transforms, datasets

device = torch.device("cpu")
if torch.cuda.is_available():
    print("cuda available, use cuda to train module")
    device = torch.device("cuda")

train_set = datasets.CIFAR10("./CIFAR10", transform=transforms.ToTensor(), download=True)
test_set = datasets.CIFAR10("./CIFAR10", transform=transforms.ToTensor(), train=False, download=True)

train_set_size = len(train_set)
test_set_size = len(test_set)

print(f"train set size: {train_set_size}")
print(f"test set size: {test_set_size}")

train_dataloader = DataLoader(train_set, 64, shuffle=True, drop_last=True)
test_dataloader = DataLoader(test_set, 64)


class MyModule(nn.Module):
    def __init__(self):
        super(MyModule, self).__init__()
        self.module1 = nn.Sequential(
            nn.Conv2d(3, 32, 5, padding=2),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 32, 5, padding=2),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, 5, padding=2),
            nn.MaxPool2d(2),
            nn.Flatten(),
            nn.Linear(1024, 64),
            nn.Linear(64, 10),
        )

    def forward(self, x):
        return self.module1(x)


# module
my_module = MyModule().to(device)
if os.path.exists("./my_module/my_module_10.pt"):
    my_module.load_state_dict(torch.load("./my_module/my_module_10.pt"))

# loss function
loss_func = nn.CrossEntropyLoss().to(device)

# learning rete
learning_rate = 1e-2

# optimizer
optim = torch.optim.SGD(my_module.parameters(), lr=learning_rate)

epoch = 10

writer = SummaryWriter("./logs/my_module_log")

total_train_step = 0
total_test_step = 0
for i in range(epoch):
    print(f"----- start training epoch {i + 1} -----")

    my_module.train()
    for imgs, targets in train_dataloader:
        imgs = imgs.to(device)
        targets = targets.to(device)

        outputs = my_module.forward(imgs)

        loss = loss_func(outputs, targets)

        optim.zero_grad()
        loss.backward()
        optim.step()

        total_train_step = total_train_step + 1
        writer.add_scalar("train loss", loss.item(), total_train_step)
        print("#", end="")
        if total_train_step % 100 == 0:
            print()
            print(f"train step:{total_train_step}, loss:{loss.item()}")

    my_module.eval()
    total_test_loss = 0
    total_accuracy = 0
    with torch.no_grad():
        for imgs, targets in test_dataloader:
            imgs = imgs.to(device)
            targets = targets.to(device)

            outputs = my_module.forward(imgs)

            loss = loss_func(outputs, targets)
            total_test_loss = total_test_loss + loss.item()

            step_accuracy = (outputs.argmax(1) == targets).sum()
            total_accuracy = total_accuracy + step_accuracy

    total_test_step = total_test_step + 1
    print(f"total loss in test set: {total_test_loss}")
    print(f"accuracy in test set: {total_accuracy / test_set_size}")
    writer.add_scalar("test loss", total_test_loss, total_test_step)
    writer.add_scalar("test accuracy", total_accuracy / test_set_size, total_test_step)

    torch.save(my_module.state_dict(), f"./my_module/my_module_{i+1}.pt")
    print("save module success")

writer.close()
```

## 二、数据的获取
数据是深度学习十分重要的一点。模型训练的过程离不开大量数据的不断获取。pytorch 提供了数据获取的方便接口，能够简洁的实现对数据的各项操作。
### （1）Dataset
`Dataset` 类是对数据总体的抽象。对于本地的数据，我们可以继承该类来实现数据的读取。`Dataset` 类需要实现特殊方法 `__getitem__` 和 `__len__`，以实现通过下标进行数据获取和通过 `len()` 获得数据总量。

比如说我们有一个图片数据集需要训练分类，每个不同标签的图片放在对应标签名的文件夹下。在需要将所有图片作为数据总体读取时，若直接操作文件将十分繁琐不美观。因此可以继承 `Dataset` 类实现如下：

```py
class MyDataset(Dataset):
    def __init__(self, root_dir, label_dir):
        self.root_dir = root_dir
        self.label_dir = label_dir
        self.total_path = os.path.join(self.root_dir, self.label_dir)
        self.img_names = os.listdir(self.total_path)

    def __getitem__(self, index):
        img_name = self.img_names[index]
        img_path = os.path.join(self.total_path, img_name)
        img = Image.open(img_path)
        label = self.label_dir
        return img, label

    def __len__(self):
        return len(self.img_names)
```

并利用 `Dataset` 对 + 运算符的重载实现数据集的合并。
```py
cat_train_dataset = MyDataset("./train", "cat")
dog_train_dataset = MyDataset("./train", "dog")
train_dataset = cat_train_dataset + dog_train_dataset
```

pytorch 也自带了许多数据集，可以通过一行代码实现下载和使用。如示例代码中：
```py
# from torchvision import transforms, datasets
train_set = datasets.CIFAR10("./CIFAR10", transform=transforms.ToTensor(), download=True)
test_set = datasets.CIFAR10("./CIFAR10", transform=transforms.ToTensor(), train=False, download=True)
```
便获取了 torchvision 自带的数据集 CIFAR10。将其保存在本地。同时获取了对该数据集进行访问的 `Dataset` 实例 `train_set` 和 `test_set`。

### （2）Dataloader
`Dataset` 对应了数据集，而 `Dataloader` 则对应了如何在训练中获取数据。如示例代码中：
```py
train_dataloader = DataLoader(train_set, 64, True, drop_last=True)
test_dataloader = DataLoader(test_set, 64)
```
`Dataloader` 确定了对于数据集，要每次取 64 个作为一个批次（batch）；并且对于训练数据集，每一次训练打乱数据顺序，对于最后一个个数小于 64 的批次，将其丢弃。

通过合理的设置 `Dataloader` 的参数，可以避免全量数据梯度下降训练过慢的缺点和单条数据随机梯度下降造成的无法找到最优解的问题。

### （3）transformer
`transformer` 并不用于获取数据，而是处理数据，将数据进行各种变换。以 `torchvision` 中的 `transformer` 为例（其他方向也有各自的 `transformer`）。比如说 `ToTensor`可以将 `PIL.Image` 类型的数据转换为 `Tensor` 类型。

> 示例代码在获取 CIFAR10 数据集时，也使用了 `transformer` 将数据统一转化为 `Tensor` 类型。

但是其他的一些 `transformer` 也可以用于对数据进行修改，以从原有数据中创造出新的数据。比如说通过 `Resize` 修改图像的尺寸。

另外需要介绍一个特殊的 `transformer`，`Compose`。`Compose` 可以将一系列不同的 `transformer` 组合成一个单一的变换。例如：
```py
# from torchvision import transforms
compose_trans = transforms.Compose([
    transforms.Resize((200, 200)),
    transforms.ToTensor()
])
```
就创建了一个 `transformer`，先对调用者执行 `Resize` 操作，再执行 `ToTenser` 操作。

## 三、神经网络的构建
`torch.nn` 提供了构建神经网络的组件，包括神经网络类 `Module`、各种神经网络层等等内容。

> nn 是 neural network 的缩写。这一点似乎很少有人提及？

示例代码中我们继承 `nn.Module`，创建了自己的神经网络类：
```py
# from torch import nn
class MyModule(nn.Module):
```

在其中，我们需要构造神经网络的结构。这在该类的构造函数中实现。
```py
    def __init__(self):
        super(MyModule, self).__init__()
        self.module1 = nn.Sequential(
            nn.Conv2d(3, 32, 5, padding=2),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 32, 5, padding=2),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, 5, padding=2),
            nn.MaxPool2d(2),
            nn.Flatten(),
            nn.Linear(1024, 64),
            nn.Linear(64, 10),
        )
```
> 注意这里我们首先需要调用父类的构造函数 `super(MyModule, self).__init__()`。

在构造函数中，我们使用了 `torch.nn` 中的卷积层 `Conv2d`、最大池化层 `MaxPool2d`、线性层 `Linear` 等不同类型的神经网络层。

这里需要另外说明的是 `nn.Sequential`，这个神经层可以将神经层序列按顺序合并为一层，方便后面的使用。

另外 `nn.Flatten` 也需要说明，这一层会将前面传来的 `Tensor` 不管维数，统一转化为一维。

一般的神经网络构造，就在构造函数中进行。除此之外，还需要定义 `forward` 函数。用来进行神经网络的前向传播。这一部分没有需要特别说明的地方。

```py
    def forward(self, x):
        return self.module1(x)
```

## 四、训练：损失函数与优化器
接下来进入了模型训练的部分，但在训练前我们还需要定义一些变量。这些变量对训练十分重要。
### （1）损失函数
损失函数既是评价模型预测能力的指标，也是进行反向传播，调整模型的依据。pytorch 提供了许多损失函数，它们和神经网络层一样，都在 `torch.nn` 模块下。

在示例代码中，我们使用了交叉熵损失函数
```py
# from torch import nn
loss_func = nn.CrossEntropyLoss().to(device) # 暂不考虑 to() 函数。
```

### （2）优化器
优化器用于设置梯度下降的策略，在 `torch.optim` 模块中。不同的优化器参数可能有所不同，但都有步长（或学习率，learning rate）作为参数。示例代码中选用 `SGD`（随机梯度下降）作为优化器。
```py
# learning rete
learning_rate = 1e-2

# optimizer
optim = torch.optim.SGD(my_module.parameters(), lr=learning_rate)
```
> 注意创建优化器需要传入要训练的模型具有的参数。在示例代码中即 `my_module.parameters()`

优化器在模型训练进行反向传播时使用，需要调用优化器的两个函数。 `zero_grad()` 和 `step()`。如在示例代码中有：
```py
        optim.zero_grad()
        loss.backward()
        optim.step()
```

优化器必须这样成对调用两函数。

## 五、训练：训练主循环与模型评价
### （1）对模型进行迭代训练
训练的主体由一个双重循环构成。外层循环决定训练的轮数（epoch）；内层循环次数是数据的批数（batch）。pytorch 以一批为单位进行训练。
```py
epoch = 10
for i in range(epoch):
```
epoch 参数决定训练的轮数。

```py
    my_module.train()
    for imgs, targets in train_dataloader:
```
在一轮训练开始前，调用模型的 `train()` 方法。这只会影响一部分网络层的行为，但最好总是加上。

对于每一批训练，迭代调用包含训练集数据的 `Dataloader`。每一次取得的元素是一个二元组，二元组的第一个元素为输入，第二个元素为预期输出，他们的数据类型都是 `Tensor`。需要注意的是，由于 `Dataloader` 以一个 batch 的大小进行读取，因此两个 `Tensor` 的第一维度均为 batch 的大小。

之后，对每一批训练，需要进行前向传播求得预测结果；利用损失函数求得预测结果和期望结果间的损失；再反向传播，对模型进行修改。如示例代码：
```py
        outputs = my_module.forward(imgs)

        loss = loss_func(outputs, targets)

        optim.zero_grad()
        loss.backward()
        optim.step()
```
这里出现了之前提到的优化器的使用。另外也需要注意，反向传播 `backward()` 的调用是在损失函数所计算得到的损失上的。

### （2）对模型进行评价
对模型进行评价的过程与训练时类似，只不过所用的数据集变成了测试集。也需要进行前向传播和求损失函数，根据所得结果进行评估。但是不能进行反向传播。
```py
    my_module.eval()
    total_test_loss = 0
    total_accuracy = 0
    with torch.no_grad():
        for imgs, targets in test_dataloader:
            imgs = imgs.to(device)
            targets = targets.to(device)

            outputs = my_module.forward(imgs)

            loss = loss_func(outputs, targets)
            total_test_loss = total_test_loss + loss.item()

            step_accuracy = (outputs.argmax(1) == targets).sum()
            total_accuracy = total_accuracy + step_accuracy
```
注意在进行测试之前需要调用模型的 `eval()` 方法，与 `train()` 作用类似。另外，这里为了避免前向传播影响梯度，还需要调用 `torch.no_grad()`， 使用 `with` 语句在离开作用域时自动取消效果。

在模型评价部分，我们得到了 `total_test_loss` 和 `total_accuracy`，可以作为评价模型的标准。

## 六、tensorboard的数据可视化
我们在训练和测试过程中得到了许多评价模型的数值。这些数值随着训练的进行发生变化。为了方便观察变化的趋势，来更好的评价训练过程，可以以训练步数为自变量，损失或准确率为因变量作图。tensorboard 就提供了方便的作图方法。

事先创建 `SummaryWriter` 实例
```py
# from torch.utils.tensorboard import SummaryWriter
writer = SummaryWriter("./logs/my_module_log")
```

当求得损失或准确度时，调用 `add_scalar()` 方法将点添加到坐标图上。
```py
# 在训练中
        total_train_step = total_train_step + 1
        writer.add_scalar("train loss", loss.item(), total_train_step)
        print("#", end="")
        if total_train_step % 100 == 0:
            print()
            print(f"train step:{total_train_step}, loss:{loss.item()}")

# 在测试中
    total_test_step = total_test_step + 1
    print(f"total loss in test set: {total_test_loss}")
    print(f"accuracy in test set: {total_accuracy / test_set_size}")
    writer.add_scalar("test loss", total_test_loss, total_test_step)
    writer.add_scalar("test accuracy", total_accuracy / test_set_size, total_test_step)
```
> 并且我们还将数值打印了出来。通过打印训练和测试信息，可以使人知道训练进行到了何种程度，并在训练出问题时及时终止。

最后不要忘记关闭 `SummaryWriter`。
```py
writer.close()
```

接着，我们需要在控制台执行命令
```shell
tensorboard --logdir=logs/my_module_log
```
然后便可在网页中查看 tensorboard 所绘制的图表。

## 七、模型的保存与加载
对训练好的模型，我们需要将其保存到磁盘上，而不是随着程序结束而被内存释放。保存和加载模型有两种方式。

**第一种**如下所示
```py
torch.save(my_module, "./my_module.pt")

torch.load("./my_module.pt")
```
这将同时保存模型的结构和参数，但是需要注意，在加载使用模型时，依旧需要有模型类的定义。

**第二种**如下所示
```py
torch.save(my_module.state_dict(), "./my_module.pt")

my_module = MyModule().to(device)
my_module.load_state_dict(torch.load("./my_module.pt"))
```
这样的方式将只保存模型的参数，但不保存模型的结构。可以减小一部分文件大小。示例代码中使用的便是这种方法。

## 八、gpu加速
gpu 可以加速模型训练。pytorch 可以使用 NVIDIA 显卡进行加速。默认的情况，训练模型使用 cpu。我们需要将其切换到 cuda。切换到使用 cuda 有两种方法。

我们可以对模型、损失函数、张量使用 cuda。在**第一种**方法中，我们要调用 `cuda()` 方法。
```py
# 模型
my_module = MyModule()
if torch.cuda.is_available():
    my_module = my_module.cuda()

# 损失函数
loss_func = nn.CrossEntropyLoss()
if torch.cuda.is_available():
    loss_func = loss_func.cuda()

# 在训练中
        for imgs, targets in test_dataloader:
            if torch.cuda.is_available():
                imgs = imgs.cuda()
                targets = targets.cuda()
```
为了避免 cuda 不可用，要在使用 `cuda()` 前调用 `torch.cuda.is_available()` 做判断。

**第二种**方法较为简单，需要调用 `to()` 方法。
```py
# 声明训练时使用的设备
device = torch.device("cpu")
if torch.cuda.is_available():
    device = torch.device("cuda")

# 模型
my_module = MyModule().to(device)

# 损失函数
loss_func = nn.CrossEntropyLoss().to(device)

# 在训练中
    for imgs, targets in train_dataloader:
        imgs = imgs.to(device)
        targets = targets.to(device)
```
在第二种方法中，我们事先声明了所用的设备，然后只需要对模型、损失函数、张量调用 `to()` 方法即可。避免了重复的判断。

## 九、总结
以上便是 pytorch 的基本使用。我们可以从代码中总结出模型训练的一般流程：
1. 加载数据
2. 构建神经网络
3. 设置参数，比如损失函数、学习率等
4. 训练模型
5. 评价模型
6. 保存模型