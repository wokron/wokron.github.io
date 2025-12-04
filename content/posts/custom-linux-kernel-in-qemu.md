+++
title = "在 QEMU 上运行最新内核"
tags = ["Linux", "QEMU"]
categories = ["操作系统"]
date = 2025-11-29T23:03:46+08:00
+++

最近，我的业余时间都花在了一个[和系统编程有关的项目](https://github.com/wokron/condy)上。不久之后我遇到了一个问题：想要使用最新版本内核的特性并不是一件十分容易的事。发行版大多滞后于最新的内核版本，而我又不是那样激进的人，想要冒险在自己唯一的这台 Linux 电脑上升级内核（使用 `installkernel` 命令）。所以我需要一个办法，在保证我的系统安全的情况下，搭建一个具有新版本内核的开发环境。

这种情况容器帮不了我，虚拟机则是一个好选择。

我曾经写过一个[简单的 QEMU 介绍](../qemu-introduction)。但实际上我对 QEMU 的了解也仅限于那篇文章的内容了。探索本文的内容花了我一些时间。在这个过程中，我也发现网上的相关内容要么包含了未充分说明的脚本，要么存在许多冗余选项。我觉得在此处以一个简单、充分解释的方式记录这一过程似乎是有意义的。

## 当然，从编译内核开始

如果你只需要一个能在虚拟机里运行的内核，不需要深度定制的话，那么编译内核其实很简单。这里有一个[简单的教程](https://docs.kernel.org/admin-guide/quickly-build-trimmed-linux.html)，不过我们接下来要做的还要更简单。

首先，你要下载内核源码。如果不需要修改源码的话，从 [kernel.org](https://kernel.org/) 下载是最方便的。这里下载了编写本文时的 stable 版本（6.17.9）。

```sh
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.17.9.tar.xz
tar -xf linux-6.17.9.tar.xz
cd ./linux-6.17.9
```

之后，配置编译选项。这将创建编译配置文件 `.config`

```sh
make defconfig
make kvm_guest.config
```

前一条命令选择使用默认编译配置。第二条命令在默认配置基础上设置编译选项，使编译后的镜像可以作为 kvm 上的虚拟机运行。如果有其他需求，可以在此基础上继续配置。

之后直接编译即可。由于只包含了作为虚拟机运行的最小配置，不需要编译驱动程序，这一过程应该很快。在我的笔记本上只用了一分钟。

```sh
make -j $(nproc)
```

编译后的内核镜像位于 `arch/x86/boot/bzImage`。当然，是 x86 架构。

```sh
file arch/x86/boot/bzImage
# arch/x86/boot/bzImage: Linux kernel x86 boot executable bzImage, version 6.17.9 (wokron@wokron-navi) #2 SMP PREEMPT_DYNAMIC Sun Nov 30 12:34:56 CST 2025, RO-rootFS, swap_dev 0XD, Normal VGA
```

## 准备根文件系统

### 简易根文件系统

这一点和我之前[有关容器的文章](../simple-container)中的内容类似。如果**不考虑实际可用性**，我们现在就可以运行一个内核了。

> 下面的内容只是用一个简单方法展示原理。想看实际步骤可以直接跳到[此处](#完整根文件系统)。

> 插一嘴，如果不使用本节的方法，而是使用你现在运行的 Linux 自身的 initrd 而不是某个发行版的文件系统的话，运行内核会更加简单。可以试试下面的命令：
> ```sh
> qemu-system-x86_64 \
>     -kernel /path/to/your/bzImage \
>     -m 2G \
>     -append "console=ttyS0" \
>     -initrd /boot/initrd.img \
>     -nographic
> ```

我们只需要下载一个发行版的根文件系统，之后将其制作成一个文件系统镜像。下面以 ubuntu 22.04 为例。

```sh
truncate -s 512M ubuntu-22.04.img
mkfs.ext4 -F ubuntu-22.04.img

wget https://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04-base-amd64.tar.gz
mkdir ubuntu_root
sudo mount -o loop ./ubuntu-22.04.img ./ubuntu_root/
sudo tar -xzvf ubuntu-base-22.04-base-amd64.tar.gz -C ./ubuntu_root/
sudo umount ./ubuntu_root/
rmdir ./ubuntu_root/
```

这样就制作好了一个镜像。

```sh
file ubuntu-22.04.img
# ubuntu-22.04.img: Linux rev 1.0 ext4 filesystem data, UUID=40550120-faa4-46a8-ab59-67c9cdf887e3 (extents) (64bit) (large files) (huge files)
```

此处的关键是，我们使用 `mkfs.ext4 -F` 将一个普通文件 `ubuntu-22.04.img` 格式化为一个 ext4 文件系统。之后再把 ubuntu 根文件系统内容复制到这个文件当中。

然后使用 QEMU 运行如下命令，即可使用我们刚刚编译的内核运行虚拟机。

```sh
qemu-system-x86_64 \
    -kernel /path/to/your/bzImage \
    -m 2G \
    -append "console=ttyS0 root=/dev/sda" \
    -drive format=raw,file=./ubuntu-22.04.img \
    -nographic
```

`-kernel` 选项指定了要运行的内核的镜像，`-m` 提供了 2G 的内存大小，`-append` 选项设置了内核的启动参数：使用串口通信、根文件系统位于设备 `/dev/sda` 上。`-drive` 提供了我们刚刚构建的根文件系统镜像，即设备 `/dev/sda`。`-nographic` 禁用图像输出，这样 QEMU 只会将内容以字符形式输出。

等待日志滚动一段时间，之后你应该能够看到 shell 提示符。你已经成功了，可以试试使用各种命令。

```console
...
# ls
bin   dev  home  lib32	libx32	    media  opt	 root  sbin  sys  usr
boot  etc  lib	 lib64	lost+found  mnt    proc  run   srv   tmp  var
```

> 退出 QEMU 使用 `ctrl + a` 再键入 `c`。

但是这样的环境根本**没法使用**。这个虚拟机中缺失了许多工具和配置，我们也无法访问网络来安装需要的包。

我们需要一个更加完整的环境。让我们从头开始准备根文件系统。

### 完整根文件系统

> 下面的操作参考自 [syzkaller 的 create-image.sh 脚本](https://github.com/google/syzkaller/blob/master/tools/create-image.sh)，但是精简了许多内容。毕竟我们的目标不是模糊测试。

这里需要使用一个工具 `debootstrap`。这个命令可以用于在文件系统中构建一个完整的 debian 文件系统（ubuntu 也可以）。这个命令只是一个脚本，没有任何依赖项。

```sh
sudo apt-get install debootstrap
```

之后，我们使用该命令构建一个 ubuntu 的完整文件系统。使用 `--include` 指定要安装的包，我们实际上只需要安装 `systemd`。不过这里为了方便还安装了 `openssh-server`。该命令执行后将在 `$ROOT_DIR` 处构建一个发行版本为 `$RELEASE` 的根文件系统。可以在最后指定镜像源 `$MIRROR`，留空表示使用默认源。

```sh
MIRROR=...
RELEASE=noble
ROOT_DIR=noble
sudo debootstrap --include systemd,openssh-server $RELEASE $ROOT_DIR $MIRROR
```

安装完成后，我们修改 `root` 用户的密码。因为只是开发环境，所以这里直接不使用密码。（即删除 `/etc/passwd` 中密码项的 `x`）

```sh
set -u
sudo sed -i '/^root/ { s/:x:/::/ }' $ROOT_DIR/etc/passwd
```

> 小心！运行命令前确保 `ROOT_DIR` 已设置。此处通过 `set -u` 进行了检查。

之后我们进行网络的配置。运行下面命令，创建 `/etc/netplan/01-netcfg.yaml` 文件。配置中我们让 `eth0` 设备使用 DHCP 协议自动获取 IP 地址。

```sh
cat << EOF | sudo tee -a $ROOT_DIR/etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: yes
EOF
```

> ubuntu 使用 netplan 进行网络配置，debian 的配置方法则不同。可以参考 [syzkaller 的脚本](https://github.com/google/syzkaller/blob/01c07bfe113aa2369bbff34c8f845108a2273e1f/tools/create-image.sh#L168)。

然后我们修改 `/etc/fstab` 文件。将 `/dev/root` 挂载为根文件系统。此处应当是为了控制后续挂载的行为，如果不增加此项，后面将启动失败。

```sh
echo '/dev/root / ext4 defaults 0 0' | sudo tee -a $ROOT_DIR/etc/fstab
```

既然我们安装了 ssh，这里可以创建一个 ssh key。方便后面通过 ssh 连接。

```sh
ssh-keygen -f $RELEASE.id_rsa -t rsa -N ''
sudo mkdir -p $ROOT_DIR/root/.ssh
cat $RELEASE.id_rsa.pub | sudo tee -a $ROOT_DIR/root/.ssh/authorized_keys
```

最后，我们按照之前的方式创建根文件系统镜像。

```sh
MOUNT_DIR=${RELEASE}_rootfs
truncate -s 4G $RELEASE.img
mkfs.ext4 -F $RELEASE.img
mkdir $MOUNT_DIR
sudo mount -o loop $RELEASE.img $MOUNT_DIR
sudo cp -a $ROOT_DIR/. $MOUNT_DIR/
sudo umount $MOUNT_DIR
rmdir $MOUNT_DIR
```

## 运行虚拟机

最后，使用 QEMU 运行我们的内核。

```sh
qemu-system-x86_64 \
    -enable-kvm \
    -m 8G \
    -smp 4 \
    -kernel /path/to/your/bzImage \
    -append "console=ttyS0 root=/dev/vda net.ifnames=0" \
    -drive file=/path/to/your.img,format=raw,if=virtio \
    -device virtio-net-pci,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp:127.0.0.1:10022-:22 \
    -nographic
```

其中：
- `-enable-kvm` 启用 KVM 加速；
- `-m` 设定 8G 的内存；
- `-smp` 设置 4 核 CPU；
- `-kernel` 指定内核镜像；
- `-append` 设置内核选项，其中 `console=ttyS0` 设定使用串口通信，`root=/dev/vda` 设定使用 `/dev/vda` 设备作为根文件系统，`net.ifnames=0` 设定禁用网卡重命名，以便我们之前设置的 `eth0` 配置可以正常生效。
- `-drive` 设定根文件系统镜像，其中 `if=virtio` 指定接口类型为 `virtio`（虚拟化接口），对应的设备为 `/dev/vda`。
- `-device` 设定一个网卡设备，类型为 `virtio-net-pci`（虚拟网卡），名为 `eth0`（和配置对应）。
- `-netdev user,id=net0` 设置 `net0` 上宿主机与虚拟机间的网络使用 “用户网络”（User Networking），此时虚拟机的 IP 地址由 QEMU 的 DHCP 服务提供。`hostfwd=tcp:127.0.0.1:10022-:22` 将虚拟机上的 22（ssh）端口转发到宿主机上的 10022 端口。这时宿主机通过访问自身的 10022 端口即可 ssh 登陆虚拟机。
- `-nographic` 禁用图像，使用串口通信。

运行上述命令，启动虚拟机。一小段时间后出现登陆提示，输入 `root` 用户名后，直接登入系统（因为没有密码）。

```console
Ubuntu 24.04 LTS wokron-navi ttyS0

wokron-navi login: root
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.17.9 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
root@wokron-navi:~# 
```

查看内核版本，应当和我们之前编译的版本相同。

```sh
uname -r
# 6.17.9
```

查看网络设备信息，可以看到我们设置的网卡 `eth0`。其地址 `10.0.2.15` 是虚拟机通过 DHCP 协议自动从 QEMU 获取的。

```sh
ip addr
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
#     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
#     inet 127.0.0.1/8 scope host lo
#        valid_lft forever preferred_lft forever
#     inet6 ::1/128 scope host noprefixroute 
#        valid_lft forever preferred_lft forever
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 100
#     link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
#     altname enp0s3
#     inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic eth0
#        valid_lft 86244sec preferred_lft 86244sec
#     inet6 fec0::5054:ff:fe12:3456/64 scope site dynamic mngtmpaddr noprefixroute 
#        valid_lft 86247sec preferred_lft 14247sec
#     inet6 fe80::5054:ff:fe12:3456/64 scope link 
#        valid_lft forever preferred_lft forever
# 3: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
#     link/sit 0.0.0.0 brd 0.0.0.0
```

此时虚拟机应当能够访问互联网。可以通过 `apt` 安装所需包。

```sh
apt-get install vim
```

宿主机上，使用之前生成的 ssh 私钥，可以登陆虚拟机

```sh
ssh -i /your/path/to.id_rsa -p 10022 root@localhost
```
