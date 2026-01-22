+++
title = "构建最简 Linux 文件系统"
tags = ["Linux", "QEMU"]
categories = ["操作系统"]
date = 2026-01-22T17:24:05+08:00
+++

[上回](../custom-linux-kernel-in-qemu)说到了我们如何用 QEMU 搭建一个包含最新内核的开发环境。但试过一段时间后，我却感觉这样并不方便。

更多时候，我只需要运行单个程序（比如，单元测试）即可，而上文中所构建的完整环境却让这个过程便复杂了。本文介绍了一个更简单的方法。只需要一个[简单的脚本](https://github.com/wokron/condy/blob/32ef34c3d39f0fb9c5940414034426d80ba9aaff/scripts/light-initrd.sh#L1)即可构建完整运行环境。

## initrd

上一篇文章中已经提到了使用 initrd 启动操作系统的方法。initrd 是一个只读文件系统镜像，内核启动时，会将镜像内容加载到内存中。程序可以像普通文件系统一样访问。

加载文件系统之后，内核会从其中寻找可用的 init 程序。因此我们只要编写合理的 init 程序，将其打包到一个 initrd 中即可。

## busybox

在这种从零开始构建文件系统的场景下，使用动态库太过繁琐。因此我们直接用宿主机上的 busybox 来提供基本的程序。

## 准备阶段

首先创建一个 `WORK_DIR` 目录。此路径下的文件会被打包成文件系统镜像。

```sh
WORK_DIR=$(mktemp -d)
trap "rm -rf $WORK_DIR" EXIT
```

之后寻找宿主机上的 `busybox`，将其复制到 `$WORK_DIR/bin/busybox`。

```sh
mkdir -p "$WORK_DIR/bin"

# Copy busybox into the work directory
BUSYBOX_PATH=$(which busybox)
if [ -z "$BUSYBOX_PATH" ]; then
    echo "Error: busybox not found in PATH."
    exit 1
fi
cp "$BUSYBOX_PATH" "$WORK_DIR/bin/busybox"
```

创建 `$WORK_DIR/init` 脚本，赋予其可执行权限。内核在启动中会识别并执行该脚本。

```sh
# Create init script
cat << 'EOF' > "$WORK_DIR/init"
#!/bin/busybox sh

# ...

EOF
chmod +x "$WORK_DIR/init"
```

## init 脚本内容

接下来查看 `$WORK_DIR/init` 脚本的内容。首先，指明该脚本由 `/bin/busybox sh` 执行。

```sh
#!/bin/busybox sh
```

之后，创建通常的文件系统路径；并挂载 `proc`、`sys` 和 `dev` 三个特殊的文件系统。

```sh
# Initialize minimal directories
busybox mkdir -p /etc /proc /root /sbin /sys /usr/bin /usr/sbin

# Mount necessary filesystems
busybox mount -t proc proc /proc
busybox mount -t sysfs sys /sys
busybox mdev -s
```

安装 `busybox` 所提供的各项工具。这一过程会创建工具名到 `busybox` 的符号链接。

```sh
# Install busybox applets
busybox --install -s
```

设定用户。root 用户，uid=0，gid=0，无需密码，home 为 `/root`，shell 为 `/bin/bash`。

```sh
# Create a minimal passwd file
echo root::0:0:root:/root:/bin/sh > /etc/passwd
```

启动本地回环

```sh
# Set up loopback interface
hostname localhost
ip link set lo up
```

减少 kernel 日志的可见性。登陆 root 用户。如果退出的话，使用 `poweroff` 关机。

```sh
# Reduce kernel printk verbosity
echo 5 > /proc/sys/kernel/printk

# Start an interactive shell
login root

# Power off
poweroff -f
```

## 构建 initrd 镜像

接下来，构建 initrd 镜像。

`$FILES` 中定义了一系列额外需要放入镜像的文件。注意，如果需要执行该程序的话，应该采用静态链接。

```sh
# Copy user-specified files into /root
mkdir -p "$WORK_DIR/root"
for FILE in $FILES; do
    BASENAME=$(basename "$FILE")
    cp "$FILE" "$WORK_DIR/$DEST_DIR/$BASENAME"
done
```

最后，通过 `cpio` 命令将 `WORK_DIR` 转换为 initrd 文件系统。所有文件的所有者都为 root 用户。

```sh
# Create the initrd image
OUTPUT_FILE_REAL=$(realpath "$OUTPUT_FILE")
cd "$WORK_DIR"
find . | cpio -o -H newc -R 0:0 | gzip -9 > "$OUTPUT_FILE_REAL"
echo "Created initrd image: $OUTPUT_FILE"
```

## 运行

使用如下命令运行

```sh
qemu-system-x86_64 \
    -m 512M \
    -kernel /path/to/bzImage \
    -initrd /path/to/initrd \
    -append "console=ttyS0" \
    -nographic
```

> 再次宣传一下自己的新项目 [Condy](https://github.com/wokron/condy) :)