+++
title = "在 Conda 环境中安装 CUDA"
tags = ["CUDA", "Conda"]
categories = ["并行计算"]
date = 2024-10-14T19:18:09+08:00
+++

最近想学一下 CUDA。这里记录一下配置环境的过程。

1. 创建环境。这里只是用 Conda 实现环境隔离，不需要 python。

```console
$ conda create -n cuda-dev
$ conda activate cuda-dev
```

2. 使用 `nvidia-smi` 查看 CUDA 版本。注意这里的版本为驱动所支持的最高 CUDA 版本。而非后面安装的 CUDA 运行时版本。在安装时应当保证 CUDA 运行时版本小于等于驱动版本。

```console
$ nvidia-smi
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.183.01             Driver Version: 535.183.01   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
```

3. 安装特定版本 CUDA。

```console
$ conda install cuda -c nvidia/label/cuda-11.8.0
```

4. 查看 `nvcc` 版本。保证驱动支持当前 CUDA 版本。

```console
$ nvcc -V
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2022 NVIDIA Corporation
Built on Wed_Sep_21_10:33:58_PDT_2022
Cuda compilation tools, release 11.8, V11.8.89
Build cuda_11.8.r11.8/compiler.31833905_0
```

5. 上面安装的 CUDA 并不包含开发所需的头文件等内容。为此还需安装如下包。

```console
conda install cuda-cudart-dev -c nvidia
```

> cuda-cudart-dev 的安装基于[此处](https://github.com/conda-forge/cuda-feedstock/blob/main/recipe/doc/end_user_compile_guide.md)的说明。
