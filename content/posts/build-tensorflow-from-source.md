+++
title = "编译 Tensorflow 踩坑"
tags = ["CUDA", "Tensorflow", "Docker"]
categories = ["MLSys"]
date = 2025-01-20T21:31:16+08:00
+++

前段时间发现了 Tensorflow 里的一处小 Bug，现在有空正好提一个 PR。Bug 很快就修好了，不过之后进行本地编译时我却踩了不少坑。现在记录一下。

## 一、各种版本傻傻分不清

在开始编译之前，需要介绍一下相关的 Nvidia GPU 依赖项。 

Nvidia 有不同架构的各型显卡。为了区分硬件上的区别，Nvidia 使用**计算能力**（Compute Capability）加以区分。计算能力版本分为两部分 `x.y`。大版本号表示计算架构（如 Pascal、Volta、Ampere 等等）上的变化，之间不可兼容；小版本号则表示同一架构内部的差别，更高版本可以兼容更低版本。

**GPU 驱动**（GPU Driver）为操作系统提供硬件驱动。其版本可以通过 `nvidia-smi --query-gpu=driver_version --format=csv` 找到。同一版本的驱动支持一系列**不同计算能力、不同架构**的显卡。

**CUDA 驱动**（CUDA Driver）在 GPU 驱动之上提供了 CUDA 接口。与 GPU 驱动属于内核态设备驱动不同，CUDA 驱动是一用户态的动态链接库（DSO）。CUDA 驱动的版本一般应当随着 GPU 驱动版本的更新而更新。

**CUDA Toolkit** 提供了构建 CUDA 程序所需的编译器、运行时和库。构建后的 CUDA 应用程序依赖于 CUDA 驱动所提供的接口。又由于 CUDA 是向后兼容（Backward Compatibility）的，所以旧的 CUDA 应用程序可以运行在新的 CUDA 驱动上；换句话说，要运行某一 CUDA 程序，需要高于特定版本的 CUDA 驱动。

> 向后兼容中的 **Backward** 指的是与**时间上在前的**进行兼容。这似乎是中英文导致的思维差异。

## 二、构建配置

在构建 Tensorflow 时添加 CUDA 支持后，Tensorflow 需要满足 CUDA Toolkit 和 CUDA 驱动之间的兼容性要求。如前所述，为了更强的兼容性，我们希望 CUDA Toolkit 的版本较低。但更高的 Tensorflow 版本又会需要更高版本的 CUDA 特性。因此在编译时需要在这两方面进行权衡。

[Tensorflow 官网上](https://www.tensorflow.org/install/source#gpu)已经给出了一系列经过测试的编译配置。如果没有特殊要求的话可以按照其中所述版本进行编译。这里就选择 `tensorflow-2.18.0` 版本为例。

首先克隆仓库到本地，如果不需要修改源码的话也可以直接下载 zip 包。

```bash
git clone https://github.com/tensorflow/tensorflow.git
cd tensorflow
git checkout r2.18
```

在本地配置编译环境十分麻烦，最好使用 Tensorflow 提供的 Docker 镜像 [tensorflow/build](https://hub.docker.com/r/tensorflow/build) 环境。

```bash
docker run --name tf -w /tf/tensorflow -it -d \
  --env TF_PYTHON_VERSION=3.10 \
  -v "/path/to/tensorflow:/tf/tensorflow" \
  tensorflow/build:latest-python3.10 \
  bash
```

`/path/to/tensorflow` 为宿主机上的 Tensorflow 路径。

之后进入容器

```bash
docker exec -it tf bash
```

> 需要注意接下来的编译过程与[此处](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/tools/tf_sig_build_dockerfiles)的描述不同。后者自 v2.16 之后已经过时了，见 [#63298](https://github.com/tensorflow/tensorflow/issues/63298)。

在容器中运行配置脚本 `configure`

```bash
./configure
```

python 配置保持默认，使用镜像中的 `python3.10`。

```console
$ ./configure
You have bazel 6.5.0 installed.
Please specify the location of python. [Default is /usr/bin/python3]: 


Found possible Python library paths:
  /usr/lib/python3/dist-packages
  /usr/local/lib/python3.10/dist-packages
Please input the desired Python library path to use.  Default is [/usr/lib/python3/dist-packages]
```

不选择 ROCm 支持，选择 CUDA 支持。


```console
Do you wish to build TensorFlow with ROCm support? [y/N]: n
No ROCm support will be enabled for TensorFlow.

Do you wish to build TensorFlow with CUDA support? [y/N]: y
CUDA support will be enabled for TensorFlow.
```

接下来设置 CUDA 和 cuDNN 版本。按照官网上的编译配置，对 `v2.18` 版本，最好使用 CUDA 版本 `12.5`、cuDNN 版本 `9.3` 进行编译。不过如前所述，新版本的 CUDA 应用程序无法跑在老版本的 CUDA 驱动上，所以此处设置的 CUDA 版本应当小于要运行的机器的 CUDA 驱动版本，否则后续还需根据此处设置的 CUDA 版本更新机器的 CUDA 驱动版本。cuDNN 版本也需按照官网上编译配置根据 CUDA 版本进行同步调整。

```console
Please specify the hermetic CUDA version you want to use or leave empty to use the default version. 12.5.1


Please specify the hermetic cuDNN version you want to use or leave empty to use the default version. 9.3.0
```

> CUDA 和 cuDNN 版本应当明确具体的版本号，如 `12.5.1`，而不是 `12.5`。

随后设置计算能力。此处取值需要在[Nvidia 官网](https://developer.nvidia.com/cuda-gpus)查看自己显卡的计算能力版本，应选择**大版本号相同，小版本号小于等于设备版本**的取值。如本人笔记本显卡的计算能力为 8.9，此处选择取值为 8.6。

> 需要注意的是计算能力不应设置过低，高版本的编译器可能不再支持较低的计算能力版本。如镜像中的 `clang 18.1.8` 便不再支持默认配置中的 `3.5` 版本。

```console
Please specify a list of comma-separated CUDA compute capabilities you want to build with.
You can find the compute capability of your device at: https://developer.nvidia.com/cuda-gpus. Each capability can be specified as "x.y" or "compute_xy" to include both virtual and binary GPU code, or as "sm_xy" to only include the binary code.
Please note that each additional compute capability significantly increases your build time and binary size, and that TensorFlow only supports compute capabilities >= 3.5 [Default is: 3.5,7.0]: 8.6
```

此后选项保持默认

```console
Please specify the local CUDA path you want to use or leave empty to use the default version. 


Please specify the local CUDNN path you want to use or leave empty to use the default version. 


Please specify the local NCCL path you want to use or leave empty to use the default version.
```

对于编译器，使用推荐且在镜像中内置的 clang 即可。

```console
Do you want to use clang as CUDA compiler? [Y/n]: 
Clang will be used as CUDA compiler.

Please specify clang path that to be used as host compiler. [Default is /usr/lib/llvm-18/bin/clang]: 


You have Clang 18.1.8 installed.
```

之后同样保持默认

```console
Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -Wno-sign-compare]: 


Would you like to interactively configure ./WORKSPACE for Android builds? [y/N]: 
Not configuring the WORKSPACE for Android builds.
```

## 三、编译 Tensorflow

配置完成后使用如下选项编译：

```console
$ bazel build //tensorflow/tools/pip_package:wheel --repo_env=WHEEL_NAME=tensorflow --config=cuda --config=cuda_wheel --config=opt --copt=-Wno-gnu-offsetof-extensions
```

> 注意这里额外设置了 `--copt=-Wno-gnu-offsetof-extensions`。不设置此 flag 会导致编译报错。

编译过程会占用大量系统资源。也可以通过选项限制 bazel 的资源占用。如 `--local_cpu_resources`、`--local_ram_resources` 等。具体设置可以参考 [bazel 文档](https://bazel.build/reference/command-line-reference?hl=zh-cn)。

编译完成后 `*.whl` 文件位于 `bazel-bin/tensorflow/tools/pip_package/wheel_house/` 中。

## 四、安装

使用 `pip` 安装编译后的 wheel 包。假如路径为 `/path/to/tensorflow.whl`，则安装命令为

```bash
pip3 install '/path/to/tensorflow.whl[and-cuda]'
```

`[and-cuda]` 是必须的，否则 Tensorflow 无法使用 GPU 资源。

随后使用如下命令测试 CPU 设置

```bash
python3 -c "import tensorflow as tf; print(tf.reduce_sum(tf.random.normal([1000, 1000])))"
```

使用如下命令测试 GPU 设置

```bash
python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

> 如果出现类似下面的报错信息：
> 
> ```console
> kernel version xxx.yyy.z does not match DSO version uuu.vv.w -- cannot find working devices in this configuration
> ```
> 
> 则说明编译时的 CUDA 版本过高，导致无法与运行时较低版本的 CUDA 驱动兼容。这时可以选择降低编译时设置的 CUDA 版本，重新编译。或者升级运行设备的 GPU 驱动程序，使其 CUDA 驱动版本高于编译时的版本。