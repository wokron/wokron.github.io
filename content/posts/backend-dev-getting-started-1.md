+++
title = "后端开发入门笔记之准备工作"
tags = ["WSL", "MySQL", "Docker", "VSCode"]
categories = ["开发"]
series = ["后端开发入门笔记"]
aliases = ["/posts/e8e3ed39"]
date = "2023-04-22T22:22:55+08:00"
+++
## 一、前言
当我第一次看到那份作业的时候，我绝想不到，这作业将牵扯出多少我还未曾学习过的知识；我更想不到，自己要以多久的时间涉猎完所有这些内容。当然现在这些都已经结束了，感慨不应该抒发太多，还是趁着自己没有忘记，做一下总结吧。

这一系列应该会有几篇文章。主要内容是回顾总结我的第一次较为系统的后端开发经历。在学习的过程中我参考了许多文章，其中一些我也会在文章中给出链接。这些文章或许在一些方面比本篇文章讲的更加深入，但是本篇文章综合了许多文章的不同信息，给出了系统的安装配置流程和自己的一些见解，因此我认为还是有一些价值的。
<!-- more -->

## 二、WSL，一切的起点
### （1）WSL 简介
windows 系统虽然在个人电脑上常用，但是开发起后端来还是不方便，最好还是使用 linux。但是只有一台电脑的话，装双系统极其麻烦、虚拟机又太过笨重。那么有没有一种更加方便的方法呢？有的，那就是使用 WSL（Windows Subsystem for Linux），即 Windows 的 Linux 子系统。WSL 分为 WSL1 和 WSL2，此二者的实现原理并不相同。虽然有些跑题，但是还是介绍一下，毕竟是自己费力去查的结果。

WSL1 不是虚拟机，Hyper-V 或 VMware 等虚拟机会用软件模拟硬件的行为，其中装入的操作系统是和模拟的硬件进行交互；而 WSL1 则是通过 Windows 操作系统库模拟了一个 Linux 内核，用 Windows 的系统调用来处理 Linux 的系统调用。并且由于 WSL1 不需要模拟硬件这一中间过程，因此效率会比虚拟机高。而 WSL2 则可以看做虚拟机，但是与其他虚拟机相比更加轻量，Linux 系统运行在此虚拟机上资源占用更少、运行更快。问题是 WSL2 使用了虚拟化技术，可能与其他虚拟机冲突，不能同时使用。

在这里我们使用 WSL2。主要原因是 WSL2 运行了真正的 Linux 内核，和各种 Linux 程序的兼容性更好。

### （2）WSL 安装
安装 WSL 很简单。这里的安装流程参考 [使用 WSL 在 Windows 上安装 Linux](https://learn.microsoft.com/zh-cn/windows/wsl/install)。插一句，微软的技术文档真的很有用，如果对搜索得到的关于微软的技术问题的解决方案有不确定的地方，参考微软的技术文档有时会有很大帮助。就比如 WSL 的安装，搜到的经常已经是过时的教程了。

安装 WSL，只需要在 PowerShell 中以管理员模式运行如下命令即可。
```shell
wsl --install
```

注意网上的教程中，需要使用 `dism.exe` 的为旧版的手动安装步骤，详见 [旧版 WSL 的手动安装步骤](https://learn.microsoft.com/zh-cn/windows/wsl/install-manual)。

这样 WSL2 就安装完成了。因为我们不会去使用 WSL1，因此关于 WSL1 的安装过程就略过不提了。

## 三、Windows 中安装 Ubuntu
### （1）Ubuntu 获取
WSL2 只提供了 Linux 内核而非发行版。我们还需要自行安装发行版。我们选择安装 Ubuntu，一个简便的方法是直接打开 Microsoft Store，从中下载 Ubuntu。

我们在搜索栏里搜索 Ubuntu，在编写本文时最好下载 Ubuntu22.04LTS，而不是那个不包含版本号的。

点击**获取**按钮以现在 Ubuntu 镜像。之后**获取**按钮会变为**安装**。如果你不想自己的 c 盘爆满，此时一定不要直接点击**安装**。如果不实现迁移，安装后 Ubuntu 会在 c 盘中占据大约 10 GB 的空间。

### （2）符号链接——C盘搬迁的好方法
我们首先找到如下的文件夹 C:\Users\yourusername\AppData\Local\Packages
\CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc，该路径就是 Ubuntu 将会安装到的位置。如果你查看一下文件夹现在的大小，应该只有 KB 级别。现在我们就要将该文件夹迁移到别的盘。

你可能会想，既然我们要迁移，那是不是应该在哪里设置一下，告知系统新的安装位置？可惜并没有这样的功能。Ubuntu 一定会安装到 c 盘的该路径。除非我们找到 Ubuntu 的安装包自行安装，可是安装包位于 C:\Program Files\WindowsApps\
CanonicalGroupLimited.Ubuntu22.04LTS_2204.1.23.0_x64__79rhkp1fndgsc 路径下，而 C:\Program Files\WindowsApps 路径下保存了 Windows 操作系统的文件，访问其中内容是危险的操作，所以我们不采用这样的方法。

我们采用的方法是符号链接。了解过 Linux 的应该熟悉 “软链接”、“硬链接” 这样的名词。Windows 下的符号链接实际上就与 Linux 中的 “软链接” 类似。他们都能将不同的路径映射到相同的文件上。这样你应该理解我们应该要做什么了。我们要做的就是将 Ubuntu 的安装目录下的内容移动到别的盘，再创建一个原路径到新路径的符号链接。这样 Ubuntu 安装时依旧使用原路径，但实际上却安装到了别的盘。同时 c 盘的原路径依旧可以访问到安装后的内容。

首先，我们将 C:\Users\yourusername\AppData\Local\Packages\
CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc 文件夹**复制**到别的盘的路径下，如 E:\Linux。

接着，我们将 C:\Users\yourusername\AppData\Local\Packages\
CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc 重命名，如在文件夹名末尾加上 bak。这样一方面避免了内容的丢失，另一方面则是创建符号链接时不能出现同名文件夹。

最后我们打开 CMD 或 Powershell，运行命令
```shell
mklink \d "C:\Users\{yourusername}\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc" "E:\Linux\CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc"
```

如果成功，应该显示创建了两个路径间的链接。这时就可以将原有的文件夹删除了。进入 C:\Users\yourusername\AppData\Local\Packages，应该可以找到名为 CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc 的 “文件夹”，只不过该 “文件夹” 的图标左下角出现了和 “快捷方式” 相同的箭头。但需要注意的是，虽然文件夹可以创建快捷方式，图标也和符号链接相同，但是两者是不同的。双击符号链接进入文件夹后，还位于原有路径中；但双击快捷方式进入文件夹后，路径便会发生变化。

### （3）Ubuntu 安装
这下我们就可以点击 Microsoft Store 中的**安装**按钮了。等待自动安装，安装完成后会弹出一个类似于 CMD 的黑底命令行窗口，提示你设置用户名及密码，成功后即进入到 Ubuntu 系统中。

顺便提一句，此处设置的密码不是 root 的密码。输入如下命令设置 root 的密码。
```sh
sudo passwd root
```

### （4）配置 apt 镜像源
现在 Ubuntu 已经可以运行了。想要使用不同的软件只需要下载。为了便于下载，我们需要为 apt 换源。apt 的源信息保存在 /etc/apt/sources.list 文件中。

想要修改，最好首先备份原本的文件
```sh
cd /etc/apt
cp sources.list sources.list.bak
```

清空文件并进行编辑
```sh
>sources.list
vim sources.list
```

将新的源地址拷贝到其中，保存并退出即可。
```sh
# 清华源
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse

# 中科大源
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
```

> 不知道如何拷贝？请右击终端的上边框，将属性-选项-编辑选项-将Ctrl+Shift+C/V 用作复制粘贴的快捷键选项勾选上即可。
>
> 另外顺便一提，有时你会发现点击 CMD 窗口后程序执行就会被卡住。这是因为终端默认勾选了 “快速编辑模式”，取消即可。

源地址参考了 [Ubuntu 22.04换国内源 清华源 阿里源 中科大源 163源](https://blog.csdn.net/xiangxianghehe/article/details/122856771)

最后执行如下命令更新 apt 的源
```sh
sudo apt-get update
```

注意对 sources.list 进行修改可能需要 root 权限。开启 root 使用 `su` 命令；退出 root 使用 `exit` 命令，此命令也是退出终端的命令

### （5）Anaconda 安装及配置源
后续会使用 flask 进行开发，因此需要使用 python。这里使用 Anaconda 管理 python 环境。

首先我们需要下载 Anaconda 的安装脚本。这里使用 wget 进行下载
```sh
wget https://repo.anaconda.com/archive/Anaconda3-2023.03-Linux-x86_64.sh
```

下载的地址在 [Anaconda 的下载页](https://www.anaconda.com/download)的最下方。选择 
64-Bit (x86) Installer (860 MB) 即可。

接着我们运行 shell 脚本进行安装
```sh
bash Anaconda3-2023.03-Linux-x86_64.sh
```

安装过程中一路 yes 即可。安装结束后可能还无法使用 `conda` 命令，这是因为 Anaconda 在安装时将环境变量的设置添加到了 `~/.bashrc` 文件中。该文件只在你登入 shell 时执行一次以设置环境变量。为了让 `conda` 命令可以直接使用，可以执行如下命令以启用对环境变量的更改。
```sh
source ~/.bashrc
```

另外 Anaconda 默认在每次登陆后都会自动进入 base 环境。我们使用 Ubuntu 并不是为了编写 python 程序这单一的目的，因此可以禁止该行为。这只需要输入如下命令。
```sh
conda config --set auto_activate_base false
```

最后同样配置镜像，这需要修改 ~/.condarc 文件，为该文件添加清华镜像源。根据[清华镜像站的说明](https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/)，需要添加如下内容
```yaml
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch-lts: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

提一句，如果你先设置了 `conda config --set auto_activate_base false` 再打开 ~/.condarc 文件，就会发现 ~/.condarc 中已经有了如下内容
```yaml
auto_activate_base: false
```

## 四、VSCode 远程连接
### （1）WSL 远程连接
好了，现在我们已经具备使用 WSL 进行后端开发的条件了。可是仅仅通过 WSL 终端，使用 vim 等命令行文本编辑器进行开发的话，效率还是太低了。我们可以使用本机的 VSCode 连接 WSL，用更加现代化的编辑器进行开发。

使用 VScode 连接 WSL 的方式很简单。这一部分的内容同样参考微软技术文档，[开始通过适用于 Linux 的 Windows 子系统使用 Visual Studio Code](https://learn.microsoft.com/zh-cn/windows/wsl/tutorials/wsl-vscode)

创建链接有两种方法，分别是从 WSL 中和从本机 VSCode 中。但是这两种方式都要在 VSCode 中添加[远程开发扩展包](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)。该扩展包中包含了用于不同连接的插件，如用于 WSL 的和用于 SSH 的。但我们可以不考虑这些，只需要安装就可以了。

从 WSL 中建立连接，只需要在 WSL 终端中，要打开的文件夹处输入如下命令
```sh
code ./
```

> `code` 是 VSCode 在命令行中的名字，说实话感觉这个名字对于一个编辑器来说有点不符实了。

然后主机上就会自动运行 VSCode，创建一个新窗口。就是这么简单。

但是，更多时候你并不想再打开 WSL 终端，这就需要直接通过 VSCode 建立连接。你需要：
- 通过 Ctrl + Shift + P 打开 VSCode 的命令面板
- 输入 wsl 进行查询，选择 “远程资源管理器：焦点在 WSL Targes 视图上” 选项（什么鬼翻译），这会打开远程资源管理器的侧边栏。
- 最后在你希望连接的 WSL 子系统上右键，选择 Connect to WSL 即可连接到对应的 WSL 子系统。

这样同样会新建一个 VSCode 窗口，连接到 WSL。

最后提一下，VSCode 左下角的蓝色方块用于指示当前的远程连接。当连接 WSL Ubuntu 成功后会变为如下内容：“WSL:Ubuntu-22.04”

### （2）Windows 和 WSL 的文件共享
这一部分参考了[跨 Windows 和 Linux 文件系统工作](https://learn.microsoft.com/zh-cn/windows/wsl/filesystems)。

Windows 和 WSL 间的文件共享实际上是一个很值得关注的问题。之前使用 VMware 时，共享文件需要一系列的配置，有些麻烦。而在 WSL 中，微软则已经为我们设置好了共享文件的方法。

WSL 中，主机的盘被挂载到了 /mnt 目录下。/mnt 目录下的各目录表示着不同的外部设备。现在进入到 /mnt 文件，你就可以看到 c d e f 盘的目录。
```sh
wokron@DESKTOP-T1ANVDS:/mnt$ cd /mnt
wokron@DESKTOP-T1ANVDS:/mnt$ ls
c  d  e  f  wsl  wslg
```

在主机中情况类似，所有的 WSL 子系统的文件都放在 \\wsl$ 路径下。在资源管理器中输入该路径即可跳转。

这样的我们就可以像一般文件一样通过路径访问并操纵另一个系统中的文件了。

## 五、Docker 的安装
### （1）Docker 简介
项目需要有环境来运行。当项目在不同设备上运行时，一般需要重复搭建相同的环境。但是搭建环境的这一过程可能是难以重复的，因为我们并不一定知道搭建同一个环境的方式；甚至是不可能的，因为一些机器上并不一定提供对环境的支持。

这样的难题时常困扰着项目开发。项目开发者便希望找到这样一种方法，能够在不同的设备上保持环境的一致性。

一种方法便是虚拟机，通过复制虚拟机，就能保持环境的一致性。但虚拟机的弊端在于太过底层。虚拟机从硬件开始模拟，包含了许多项目运行所并不需要的程序。尽管我们只想要运行单一的项目，却需要同时维护整套操作系统，这势必造成资源的浪费。那么有没有一种手段，在虚拟化的时候只包含项目运行所必须的环境，而将其他内容都去掉呢？

这就是容器技术（container，当然也可以翻译为集装箱，甚至集装箱更能表现该技术的本质。从Docker的吉祥物（Moby Dock，很明显是赫尔曼·梅尔维尔《白鲸记》的neta）来看，翻译为集装箱似乎也是一个不错的选择）。容器技术使得在不同容器内运行的程序 1. 彼此隔离；2. 但又共享同一个操作系统。这样就在保持环境一致性的同时，解决了不同虚拟机运行各自的操作系统造成资源浪费的问题。

Docker 是一个用于创建和使用容器的开源项目，可以让我们方便的创建和使用容器。只需要将项目和其环境依赖打包到 Docker 的容器中，那么该容器只要能在某一设备上运行，便可以在所有设备上运行。实现了 “一次构建，到处运行”。
```text
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
```

### （2）Windows 中安装 Docker Desktop
在 Windows 下安装 Docker 很容易，只需要进入 [Docker 的官网](https://www.docker.com/)，下载 Docker Desktop 即可。

运行 Docker Desktop Installer.exe，无需设置，下载器会全自动将 Docker 下载到 c 盘 `:)`。

这时你的 c 盘会增加几个 GB 的占用。同时当你构建新的镜像（镜像是静态的容器，是容器的声明；镜像和容器的关系类似于程序与进程）时，你的 c 盘还会越来越满……所以我们请出之前用过的技术，符号链接。

Docker 在 c 盘中占用空间最大的便是C:\Users\yourusername\AppData\Local \Docker\wsl\data\ext4.vhdx 这个文件。该文件是硬盘映像文件，其来源需要解释一下 Docker 在 Windows 下的运行原理。Docker 实际上在 WSL2 中创建了新的子系统作为承载所有容器的操作系统（这也是我们先行安装 WSL2 的原因，如果先安装 Docker，则在运行 Docker 时便会提醒你安装 WSL2），而 WSL 就通过 .vhdx 硬盘映像文件在 Windows 下存储 Linux 磁盘格式的持久化数据。那么随着镜像的增加，就需要更多的空间存储镜像数据，ext4.vhdx 文件自然就增大了。

我们只需要移动 ext4.vhdx 文件，为了方便，我们将包含该文件的 data 文件夹复制到其他位置，如 D:\Docker。之后重命名源路径的 data 文件。

接着再次使用 `mklink` 命令创建符号链接
```sh
mklink \d "C:\Users\yourusername\AppData\Local \Docker\wsl\data" "D:\Docker\data"
```

这样 Docker 的大小再增长也不会影响 c 盘了。

> Docker Desktop 的 setting-resources-advanced 中有设置 Disk image location 的地方，但是我并没有采取这一手段。因为根据最近提出的一个 [issue](https://github.com/docker/for-win/issues/13345)，这一方法似乎有 bug。为了避免产生更大的问题，还是采用 `mklink` 比较保守。

### （3）WSL 中 Docker 使能
如果我是在 WSL 中编写的项目，又要如何使用 Docker 呢？难道要再安装 Linux 版的 Docker？其实并不需要，只需要下载 Docker 一次，就可以在主机和 WSL 子系统中同时使用。

只需要在 Docker Desktop 的界面中选择 setting-resources-WSL integration，选择使能你希望使用 Docker 的 WSL 子系统即可。

在使能之前，输入 `docker --version` 的结果如下
```sh
wokron@DESKTOP-T1ANVDS:~$ docker --version
Command 'docker' not found, but can be installed with:
sudo snap install docker         # version 20.10.17, or
sudo apt  install docker.io      # version 20.10.21-0ubuntu1~22.04.2
sudo apt  install podman-docker  # version 3.4.4+ds1-1ubuntu1
See 'snap info docker' for additional versions.
```

而使能之后，同样的输入结果如下
```sh
wokron@DESKTOP-T1ANVDS:~$ docker --version
Docker version 20.10.23, build 7155243
```

## 六、使用 Docker 运行 MySQL
### （1）Docker Desktop 的使用
我们下载好了 Docker，现在来利用一下。正好后端需要使用数据库，那就用 Docker 来运行 MySQL 吧。

为了方便，这里我们就先使用 Docker Desktop，而不使用 Docker 命令。想要获取 MySQL 镜像，只需要使用最上面的搜索栏搜索一下 mysql。

选择第一个，点击 pull 将镜像从 DockerHub 拉取到本地。没错，Docker 和 Git 类似，也有自己的 Hub，并且 Docker 的一些命令也很有 Git 的味道。

现在我们回到主界面，选择查看镜像，就可以看到 mysql 镜像了。

### （2）MySQL 容器的创建
我们有了镜像，现在需要通过镜像运行容器。这一过程类似于让程序变为运行的进程。

我们需要点击 mysql 项右侧的运行键，之后修改 optional settings，填写容器名、将容器内的 3306 号端口映射到主机的端口上，另外还需要设置一个环境变量 `MYSQL_ROOT_PASSWORD=password`，这样我们就为 mysql 设置了 root 的初始密码。之后点击 run 即可运行容器。

之后会进入新创建的容器的详细信息页，我们可以从 logs 中看到容器的运行情况，其中显示的就是程序在标准输出上的结果。此时 MySQL 应该还在初始化，因此还不能连接上数据库。等到出现了类似 Plugin ready for connections 之后，才可正常连接数据库。

我们打开一个终端，可以是主机的 CMD、PowerShell 或是 WSL 中的其他 shell，输入命令以连接数据库，注意此时需要输入 `MYSQL_ROOT_PASSWORD` 对应的密码。
```sh
mysql -h localhost -P 3306 -u root -p
```

之后就成功连接到数据库了。

### （3）MySQL Server 与 Client
在输入上一小节中的命令的时候，可能出现类似 command not found 的提示。因此我们还需要在本地安装 MySQL，但是并不是安装 MySQL 的服务器，而只需要安装 MySQL 的客户端，通过客户端连接容器中的 MySQL 服务器。

这里以 Linux 为例，使用 apt 安装 MySQL 客户端
```sh
sudo apt-get mysql-client
```

之后 mysql 命令就可以使用了。
```sh
wokron@DESKTOP-T1ANVDS:~$ mysql --version
mysql  Ver 8.0.32-0ubuntu0.22.04.2 for Linux on x86_64 ((Ubuntu))
```

另外我们还希望能够可视化地使用数据库，这就需要用可视化的客户端连接服务器。我们以 Navicat for MySQL 为例。只需要点击左上角**连接**，输入连接需要的信息即可创建连接。但是也可能出现 1251--Client does not support authentication protocol requested by server 的问题。这是客户端和服务器的版本不一致导致的。在 Docker 容器中我们运行的是最新版本的 MySQL，采用了新的加密算法，因此会出现认证错误。

我们要对 MySQL 服务器的配置进行修改。首先通过 `docker exec` 命令进入终端
```sh
docker exec -it <container name> bash
```

进入 mysql
```sh
mysql -u root -p
```

设置 root 的加密规则和密码，并更新设置
```sh
alter user 'root'@'%' identified with mysql_native_password by 'yourpassword'; 
flush privileges;
```

之后， Navicat 就可以正常连接了。