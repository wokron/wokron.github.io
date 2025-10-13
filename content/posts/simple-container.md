+++
title = "用 Namespace 手搓一个容器"
tags = ["Linux", "Namespace", "Docker", "Shell"]
categories = ["操作系统"]
date = 2025-10-13T10:24:54+08:00
draft = true
+++

容器技术由 Linux 下三项技术构成。这三项技术分别是 Namespace、Cgroups 和 Unionfs。他们分别实现了**系统逻辑资源的隔离**、**物理资源的限制**以及**容器的文件系统**。

在这之中最为关键的是 Namespace。因为 Namespace 实现了虚拟化中最重要的隔离的功能。在 Namespace 之外即使不使用 Cgroups，用其他文件系统替代 Unionfs，依然能够实现一个容器的大部分功能。

所以本文我们尝试用 Namespace 构建一个简单的容器。让我们首先想想，一个容器中的环境究竟需要与 host 隔离哪些资源。（请把容器想象成 host 之外的另一台机器。）

- **文件系统**：容器中的进程不能访问 host 的文件系统。这意味着挂载点的隔离 -- Mount Namespace
- **进程空间**：容器中的进程无法查看容器外的进程信息。这意味着进程号的隔离 -- Pid Namespace
- **网络接口**：容器中的进程拥有自己的网络接口，不使用 host 上的网络接口。这意味着网络的隔离 -- Network Namespace
- **用户**：容器中的用户和容器外的用户无关，例如容器内的 root 和容器外的 root 并不相同。这意味着用户的隔离 -- User Namespace
- **物理资源**：容器能够看到和管理的物理资源和容器外的资源不同。这意味着 Cgroups 视图的隔离 -- Cgroups Namespace
- **时间**：容器中的时间系统和容器外的时间不一定相同 -- Time Namespace
- **主机名**：容器中的主机名和容器外的主机名不一定相同 -- UTS Namespace
- **IPC**：IPC，例如 Posix 消息队列，使用类似文件名的标识符，但是又并不真正存在于文件系统中。容器中的这些标识符和容器外的相同标识无关 -- IPC Namespace

虽然种类很多，但是想要形成虚拟化的错觉只需要用到其中的一部分即可。为了构建我们的容器，我们选择只使用 Mount、Pid、User 和 UTS。

## 隔离文件系统

容器化最重要的是隔离文件系统。所谓的程序运行环境，本质上就是文件系统中的各类库和应用程序。同一主机上的不同发行版的容器都运行在相同的内核上，他们只是在库和应用程序上存在不同。

对于 shell 环境有一个很好的命令 `unshare` 来创建各种各样的 Namespace。我们的容器就完全建立在该命令的基础上。这里输入 `--mount` 启动一个新的 shell，该 shell 位于一个新的 Mount Namespace 中。进入新的 Namespace 后，我们会发现原本的命令依然能够使用。因为新的 Namespace 继承了之前的 Namespace 中的各挂载点。整个系统的文件系统在当前进程看来依然保持不变。

```console
$ df /
文件系统          1K的块      已用    可用 已用% 挂载点
/dev/nvme0n1p6 205307624 191282292 3523192   99% /
$ sudo unshare --mount
$ df /
文件系统          1K的块      已用    可用 已用% 挂载点
/dev/nvme0n1p6 205307624 191282296 3523188   99% /
```

接下来我们要替换掉当前的文件系统。这里我们用 Alpine 发行版的文件系统。我们首先下载并解压 Alpine。下面的链接可以在 Alpine 官网找到。

```console
$ sudo unshare --mount
$ wget https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-minirootfs-3.22.2-x86_64.tar.gz
$ mkdir alpine_root
$ tar -xzvf alpine-minirootfs-3.22.2-x86_64.tar.gz -C ./alpine_root/
```

之后，我们希望使用 Alpine 的 root 替换掉当前的 root 路径。即将根路径挂载到 `./alpine_root/` 的位置。

首先，我们需要创建一个挂载点

```console
$ mount --bind ./alpine_root ./alpine_root
$ df ./alpine_root
文件系统          1K的块      已用    可用 已用% 挂载点
/dev/nvme0n1p5 308520768 285484536 7291260   98% /home/wokron/path/to/alpine_root
```

之后，我们使用 `pivot_root` 命令将 `./alpine_root/` 挂载到根路径，原本的根移动到 `./alpine_root/old_root` 处。随后切换到根路径

```console
$ mkdir ./alpine_root/old_root
$ pivot_root ./alpine_root ./alpine_root/old_root
$ cd /
```

这时我们列出根路径下的文件，会发现 `/old_root` 就在其中。

```console
$ ls
bin       etc       lib       mnt       opt       root      sbin      sys       usr
dev       home      media     old_root  proc      run       srv       tmp       var
```

接下来我们再用 `df` 列出挂载点。`/old_root` 挂载的就是原本的根路径。

> `df` 需要读取 `/proc`，因此需要先挂载 `/proc` 文件系统。

```console
$ mount -t proc proc /proc
$ df
df
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/nvme0n1p6       205307624 191293340   3512144  98% /old_root
udev                      7.5G         0      7.5G   0% /old_root/dev
tmpfs                     7.6G    270.8M      7.4G   3% /old_root/dev/shm
tmpfs                     1.5G      2.0M      1.5G   0% /old_root/run
...
/dev/nvme0n1p5       308520768 285486492   7289304  98% /
```

最后我们再取消 `/old_root` 的挂载。这样容器进程就隔离了 host 的文件系统。

```console
$ umount -l /old_root
$ df
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/nvme0n1p5       308520768 285494116   7281680  98% /
```

……真的是这样吗？尝试列出这个路径下的文件：`/proc/1/root/home/wokron/path/to/alpine_root/..`

```console
$ ls /proc/1/root/home/wokron/path/to/alpine_root/..
alpine-minirootfs-3.22.2-x86_64.tar.gz  alpine_root
```

然后我们就会发现在容器里也可以访问到 host 的文件系统。

当然这个问题很好解决。出现这个问题的原因是我们**能够挂载 `/proc` 文件系统**，并**访问位于容器外的进程**。只需要解决这两个问题之一或全部，就可以避免对 host 文件系统的访问。User Namespace 可用于解决前者，Pid Namespace 则可解决后者。

> `/proc` 下的一切都很奇怪，`/proc/pid/root` 同样如此。如果你查看该文件的类型，你会发现它是一个符号链接
> 
> ```console
> $ sudo file /proc/1/root
> /proc/1/root: symbolic link to /
> ```
> 
> 可即使当前 Namespace 下没有挂载进程所在的文件系统，这个所谓的符号链接依然能够让你进入进程实际所在的文件系统。例如，在我们的小容器里运行一条命令，并在 host 中查看该进程的 `/proc/pid/root`。结果总是 `/`。也即是说，`cd /proc/pid/root` 和 `cd $(readlink /proc/pid/root)` 并不等价。
> 
> ```console
> # on container
> $ sleep 10000 &
> [1] 173328
> 
> # on host
> $ sudo readlink /proc/173328/root
> /
> ```

## 隔离进程空间

和文件系统一样，进程空间也是一个树状的结构。我们可以选择将某个进程和其子进程放置于一个隔离的 Pid Namespace 中。在该 Namespace 中的进程将无法感知 Namespace 之外的进程。且其 pid 也会发生变化。

为了创建一个 Pid Namespace，需要使用 `unshare` 命令的 `--pid` 选项。但这依然不够，让我们试一下。

```console
$ sudo unshare --pid
$ echo $$
123456
$ bash
$ echo $$
1
$ exit
$ bash
zsh: fork failed: 无法分配内存
```

可以看到，设置 `--pid` 之后，我们的进程 pid 还是处在原来的空间中。只有创建一个新的进程后才会真正进入到 Pid Namespace 中。但是退出该进程后，我们就无法再次创建子进程了。似乎 Pid Namespace 存在某种设计上的考量。

因此我们还需额外增加一个 `--fork` 选项。此时执行的进程将从一开始就处在 Pid Namespace 当中。

```console
$ sudo unshare --pid --fork
$ echo $$
1
```

让我们再添加上 `--mount` 选项，重新创建一遍容器。

```console
$ sudo unshare --mount --pid --fork
$ mount --bind ./alpine_root ./alpine_root
$ pivot_root ./alpine_root ./alpine_root/old_root
$ cd /
$ mount -t proc proc /proc
$ umount -l /old_root
```

> 省略了一些挂载点的创建。因为上一节中已经创建过了。

这一次再尝试访问 `/proc/1/root/home/wokron/path/to/alpine_root/..`

```console
$ ls /proc/1/root/home/wokron/path/to/alpine_root/..
ls: /proc/1/root/home/wokron/path/to/alpine_root/..: No such file or directory
```

很明显，在我们创建 Pid Namespace 之后，容器内的进程就无法访问容器外的进程的信息了。

```console
$ ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 -zsh
   32 root      0:00 ps aux
```

## 隔离用户

隔离了文件系统和进程空间后，一个进程便无法再访问 host 上的大多数资源。但许多时候这样的隔离并不是我们所需要的，我们依然需要让 host 和容器共享同一部分文件。这就引出了 docker 中 volume 的概念。既已了解了 Mount 和 Pid Namespace，那么 volume 的实现就不言自明了。下面我们将 host 上的 `./volume_dir` 挂载到容器中的 `/host_volume_dir`。（即 docker 中的 `-v ./volume_dir:/host_volume_dir`）

```console
$ mkdir volume_dir # here!
$ sudo unshare --mount --pid --fork
$ HOST_VOLUME_DIR=$(realpath ./volume_dir)
$ mount --bind ./alpine_root ./alpine_root
$ pivot_root ./alpine_root ./alpine_root/old_root
$ cd /
$ mount -t proc proc /proc
$ mkdir /host_volume_dir # here!
$ mount --bind ./old_root/${HOST_VOLUME_DIR} /host_volume_dir # here!
$ umount -l /old_root
$ df
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/nvme0n1p5       308520768 285502488   7273308  98% /
/dev/nvme0n1p5       308520768 285502488   7273308  98% /host_volume_dir
```

但是这存在一个问题。如果我们此时向 /host_volume_dir 中写入文件。文件的拥有者将会是 root。这一问题已经在[上一篇文章（Linux 下的用户和容器中的用户）](/posts/user-perm-and-container)中进行讨论了。

```console
$ echo 1234 > /host_volume_dir/a.txt
$ exit
$ ls -l ./volume_dir/a.txt
-rw-r--r-- 1 root root 5 10月 13 22:32 ./volume_dir/a.txt
```

解决这一问题的办法就是让容器中的 root 用户在容器外看来不是 root 用户。此功能需要通过 User Namespace 实现。

在 `unshare` 命令中增加 `--user` 选项可以创建一个新的 User Namespace。从下面的命令来看，创建 User Namespace 不需要 root 权限。不过结果可能和我们的预期稍有不符。我们的用户名变成了 nobody。使用 `id` 命令也可以发现 uid 变成了 id 最大值。

```console
wokron@wokron-navi$ unshare --user
nobody@wokron-navi$ id
uid=65534(nobody) gid=65534(nogroup) 组=65534(nogroup)
```

这其实是因为我们并没有设置原来的用户到 Namespace 中用户的映射关系。因为不知道用户应该映射到哪个 uid，所以就将用户设置成了最大的 nobody 占位用户。nobody 用户和其他普通用户一样，没有任何权限。

我们可以在 Namespace 中创建一个文件，并在容器外查看该文件。这时我们会发现文件的拥有者就是创建 Namespace 时的用户。

```console
nobody@wokron-navi$ echo 1234 > nobody.txt
nobody@wokron-navi$ exit
wokron@wokron-navi$ ls -l ./nobody.txt
-rw-rw-r-- 1 wokron wokron 5 10月 13 22:33 a.txt
```

如果我们在 User Namespace 中进行用户的映射，那么 Namespace 中的进程就不会有任何特权。举个例子，这次我们加上 `--mount` 选项，尝试 `mount` 命令将会报错。

```console
wokron@wokron-navi$ unshare --user --mount
nobody@wokron$ mount --bind ./alpine_root/ ./alpine_root/
mount: /home/wokron/path/to/alpine_root: 必须以超级用户身份使用 mount.
```

我们需要 User Namespace 让容器外的普通用户成为容器内的超级用户。这样这个普通用户就能在容器内拥有像 root 一样操作其他 Namespace 的权限。这需要增加 `--map-root-user` 选项。现在执行 `unshare` 后，用户名变成了 `root` 而不是 `nobody`。

```console
wokron@wokron-navi$ unshare --user --map-root-user --mount
root@wokron-navi$ mount --bind ./alpine-root/ ./alpine-root/
root@wokron-navi$ echo $?
0
```

> 当然，也可以映射到其他非 root 用户。但是那样做有什么用处吗？

需要注意，如果 User Namespace 中的 root 用户依然无法操作同级 Namespace 之外的资源。例如，如果我们去掉上面命令中的 `--mount`。在尝试 `mount` 时同样会报错。

```console
wokron@wokron-navi$ unshare --user --map-root-user
root@wokron-navi$ mount --bind ./alpine-root/ ./alpine-root/
mount: /home/wokron/Code/Experiment/namespace/test_total/alpine-root: 权限不足.
```

一个有趣的地方在于，如果我们在容器中创建用户，会发生什么？这个新的用户在 host 上又会是什么 uid？我们可以来试一下。

（稍等，马上填坑……）