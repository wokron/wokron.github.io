+++
title = "Linux 下的用户和容器中的用户"
tags = ["Linux", "Docker"]
categories = ["操作系统"]
date = 2025-10-12T16:11:53+08:00
+++

多用户操作系统的时代早就结束了。现在的人们一般不会通过多个用户登陆到同一个操作系统的方式共享计算资源，我们有更好的虚拟化技术。

不过用户依然在发挥作用，其中最主要的作用就是权限隔离。这一点似乎有许多内容可以讲，不过这里我们只会列出最关键的内容。容器中的用户会出现更多特殊情况，我们也会进行讨论。

## 用户与权限

### 用户和用户组

操作系统通过用户控制权限，特定的用户才能执行特定的操作。

操作系统中有一组配置好的**用户**。其信息被保存在 `/etc/passwd` 文件中。每个用户都属于一个或多个**用户组**。用户和用户组分别有 `uid` 和 `gid`。使用 `id` 命令可以查看当前用户的对应 ID。

```console
$ id
uid=1000(wokron) gid=1000(wokron) groups=1000(wokron) 1001(xxxxx) 1002(yyyyy)
```

其中 `gid` 指向的表示该用户的**主组**。可以通过 `newgrp` 命令切换主组。这会启动一个新的 shell。

```console
$ newgrp xxxxx
$ id
uid=1000(wokron) gid=1001(xxxxx) groups=1000(wokron) 1001(xxxxx) 1002(yyyyy)
```

> 下面的内容忽略了许多细节。不过忽略的内容相对很少遇到。

当选择一个用户登陆系统时，这次登陆所创建的 shell 会将用户和其主组的 ID 附加到进程中。这两个 ID 称为该进程的**进程凭证**。

子进程创建时会继承父进程的的进程凭证。进程凭证表明了该进程代表哪一用户进行操作。

### 超级用户

进程的用户和用户组决定了其是否有权限执行某一操作。经典的 Unix 权限系统只区分了两种用户：**超级用户**和**普通用户**。超级用户能够执行操作系统所提供的所有操作，而普通用户则受到了限制。普通用户的操作不能够影响操作系统的状态。

超级用户的 uid 为 0，通常名为 root。其用户组通常只有 root（gid=0）。除此以外的用户都是普通用户，他们之间没有权限上的差异。

利用 `sudo` 命令可以临时提升当前用户的权限，使其以 root 用户的身份运行进程。（即进程和其子进程的进程凭证为 root）。

```console
$ sleep 1 & ps -o user,group,uid,gid -p $! & wait
[1] 171301
[2] 171302
USER     GROUP      UID   GID
wokron   wokron    1000  1000
$ sudo sleep 1 & ps -o user,group,uid,gid -p $! & wait
[1] 171399
[2] 171400
USER     GROUP      UID   GID
root     root         0     0
```

> 当然这只是 `sudo` 命令的默认用法，也可以用于以其他用户的身份执行命令。这时 `sudo` 可以解释为 switch user do……。对于下面的 `su` 命令同理。

只有特定的用户才能够使用 `sudo` 命令以 root 用户身份执行命令。毕竟 `sudo` 命令只需要输入当前用户的密码。一个用户只要获得了 `sudo` 权限，就获得了整个操作系统的权限。在 Debian/Ubuntu 系下，这需要将该用户加入到 `sudo` 用户组。

> 具有 `sudo` 权限的用户拥有整个系统的权限，`sudo` 的作用仅仅是保证 “只在需要的时候使用权限”。

使用 `su` 命令可以以 root 用户身份运行 shell。不过这需要使用 root 用户的密码。

如果不知道 root 的密码，可以使用 `sudo` 命令来以 root 身份运行 shell。

```bash
su
# sudo su
# sudo -i
```

在经典的 Unix 权限模型之外，Linux 还实现了**能力**权限模型。这个模型将 root 用户的权限拆分为一个个独立的 “能力”。在运行进程时只为其赋予特定的能力而非整个 root 用户的权限。从而实现更加精细化的权限管理。不过本文并不详述这部分内容。

### 文件权限

前面我们提到除了超级用户，其他用户之间是平等的关系。但这并不意味着多用户在现在的系统中没有存在的必要。多用户的一个重要作用是设置文件访问权限。

每一个文件都有**拥有者**、**用户组**以及**权限**信息。当一个进程创建一个文件时，会根据该进程的进程凭证（用户和主用户组）指定该文件的拥有者和用户组信息。

> 实际上用户组信息的设定还有其他可能，这里并不详述。

> 下面的例子给出了切换主组的情况下文件的用户组的变化
> 
> ```console
> $ id
> uid=1000(wokron) gid=1000(wokron) groups=1000(wokron) 1001(xxxxx) 1002(yyyyy)
> $ echo 1234 > test1.txt
> $ newgrp xxxxx
> $ echo 1234 > test2.txt
> $ ls -l test*.txt
> -rw-rw-r-- 1 wokron wokron 5 2月 30 12:34 test1.txt
> -rw-rw-r-- 1 wokron xxxxx  5 2月 30 12:34 test2.txt
> ```

文件的权限信息由三部分组成。分别表示 “拥有者/用户组内用户/其他用户” 对该文件是否有 “读/写/执行” 的权限。（这部分内容可能说烂了……）

当一个进程尝试读/写/执行某个文件时，系统会判断该进程属于何种类型：
1. 拥有者：进程的用户等于文件的拥有者
2. 用户组内用户：文件的用户组属于进程的多个用户组之一
3. 其他用户：不属于上述两种情况

根据用户的类型，将进一步判断该进程是否有该文件的读/写/执行权限。

这一点很有用处。因为通过合理配置用户和文件权限，我们能够避免因为疏忽或恶意造成的文件访问。

例如，mysql 等程序需要维护一系列文件。他们要保证这组文件内容的一致性和安全性，因此不能让其他人访问；同时为了保证最小权限，这些程序不需要以 root 身份运行。合理的做法是为每个程序创建一个单独的用户。这些程序以该用户的身份运行，并使得创建的文件具有有别于其他用户的拥有者和用户组。这时只需要为所创建文件设置权限，避免其他用户访问即可。

> 当然，root 用户依然拥有这些文件的访问权限；如果成功登陆该用户或者加入到该用户的用户组，同样能够访问这些文件。

这里的一个例子是 docker 用户组。`docker` 命令行程序通过 `/run/docker.sock` 与 dockerd 进行通信。通常来说我们需要 `sudo` 来执行 `docker` 命令。因为 `/run/docker.sock` 的拥有者是 root。

```console
$ ls -l /run/docker.sock
srw-rw---- 1 root docker 0 2月 30 12:34 /run/docker.sock=
```

但我们也可以发现，`/run/docker.sock` 的用户组是 docker，且用户组的权限和拥有者一样，是可读可写。这意味着我们只需要把当前用户加入到 docker 用户组中，即可不使用 `sudo` 命令执行 `docker` 命令。这也就是[此处](https://docs.docker.com/engine/install/linux-postinstall/)提到的方法。

## 容器中的用户

众所周知，容器不是虚拟机。这意味着容器中的资源一定存在于 host 环境中。对于用户资源来说也是如此。

我们运行一个容器。在容器中可以看到我们的用户是 root。我们写入一个文件，该文件的拥有者也是 root。

```console
wokron@host$ mkdir test_dir
wokron@host$ docker run -it --rm -v ./test_dir:/test_dir ubuntu:jammy bash
root@container$ whoami
root
root@container$ echo 1234 > /test_dir/a.txt
root@container$ ls -l /test_dir
-rw-r--r-- 1 root root 5 Feb 30 12:34 b.txt
root@container$ exit
```

这个文件写到了一个卷上。这时我们从 host 上查看这个文件，就会发现在 host 上这个文件的所有者依然是 root。

```console
wokron@host$ ls -l ./test_dir
-rw-r--r-- 1 root root 5 Feb 30 12:34 b.txt
```

这意味着一件事，**容器中的 root 就是 host 中的 root**。这实在是很危险的事，尤其是容器总会让人想到隔离和安全，这增加了大意的可能。

如果你不想以 root 身份运行，在运行容器时可以使用 `--user` 选项。这个选项可以指定运行命令所使用的 uid 和 gid。然后你就会发现容器中的用户名变成了惊人的 `I have no name!`。但是 uid 和 gid 依然被正常地设置了。这里就可以得出一个结论：**用户和用户组并不重要，uid 和 gid 才重要**。作为拥有 root 权限的进程，你可以设置任何的 uid 和 gid，即使其对应的用户并不存在。

```console
wokron@host$ docker run -it --rm --user $(id -u):$(id -g) -v ./test_dir:/test_dir ubuntu:jammy bash
I have no name!@container$ id
uid=1000 gid=1000 groups=1000
```

我们依然写入一个文件。这次我们发现原本显示为用户名和用户组名的地方，现在改为了显示 uid 和 gid。这同样可以佐证上面的结论。

```console
I have no name!@container$ echo 1234 > /test_dir/a.txt
I have no name!@container$ ls -l /test_dir
-rw-r--r-- 1 1000 1000 5 Feb 30 12:34 a.txt
```

退出容器，我们可以做出一个预测。由于我们的用户的 uid 和 gid 就是 1000，所以原本在容器里由 uid=1000 的用户创建的文件，在容器外应该显示为由当前用户创建。这里的结果和我们预期的相同。

```console
I have no name!@container$ exit
wokron@host$ ls -l ./test_dir
-rw-r--r-- 1 wokron wokron 5 2月 30 12:34 a.txt
```

前面的没有用户名的问题（`I have no name!`）很好解决。需要在构建镜像的时候预先创建 uid 为 1000 的用户。Dockerfile 中甚至还提供了一个 `USER` 指令，可以设置后续指令以及容器中使用的用户。

```dockerfile
FROM ubuntu:jammy

RUN useradd -u 1000 -m wokron
# USER wokron
```

```console
wokron@host$ docker build -t test:v1 .
wokron@host$ docker run -it --rm --user wokron -v ./test_dir:/test_dir test:v1 bash
wokron@container$ whoami
wokron
```

最后，其实还有一种方法，能够让容器中的 root 用户的行为在容器外表现为普通用户的行为。那就是使用 User Namespace。这种方法可以使得访问容器外文件系统时更加安全。有关 Namespace 的内容将在另外的文章（[Here!](/posts/simple-container/#隔离用户)）中进行介绍。