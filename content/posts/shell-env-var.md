+++
title = "讨论一下环境变量"
tags = ["Shell", "Linux"]
categories = ["编程语言"]
date = 2024-11-02T15:01:54+08:00
+++

这里所说的环境变量并不仅仅指 Shell 中的变量，而是每一个进程各自拥有的，能够通过系统 API 获取的变量。当然，Shell 通常会提供环境变量的操作方法，我们也经常通过在 Shell 中管理环境变量。但是 Shell 中的变量和环境变量实际上并不完全相同，在使用 Shell 时可能会混淆这两种概念，这里便稍微分辨一下。

## 一、环境变量

从程序的角度来看，环境变量很简单。环境变量是每个进程各自拥有的键值对集合。进程可以从其环境变量中读取已有变量，修改已有变量或创建新的变量。当进程创建其子进程时，子进程会继承父进程的环境变量，但子进程的环境变量的修改并不会影响父进程的环境变量。

为了方便，这里以 python 为例。

进程可以读取已有环境变量：

```py
# inherit.py
import os
print(os.environ["HOME"])
```

也可以写入或创建环境变量：

```py
os.environ["MY_VAR"] = "my_value"
```

子进程会继承父进程的环境变量：

```py
# env.py
import os
import sys

assert len(sys.argv) == 2
father_proc = bool(int(sys.argv[1]))

print_my_var = lambda: print(
    f"process={'father' if father_proc else 'child'}, "
    f"MY_VAR={os.environ.get('MY_VAR', None)}"
)

print_my_var()

if father_proc:
    os.environ["MY_VAR"] = "1"
    print_my_var()
    os.system(f"{sys.executable} {sys.argv[0]} 0")
    print_my_var()
else:
    os.environ["MY_VAR"] = "2"
    print_my_var()
```

```console
$ python inherit.py 1
process=father, MY_VAR=None
process=father, MY_VAR=1
process=child, MY_VAR=1
process=child, MY_VAR=2
process=father, MY_VAR=1
```

## 二、Shell 的环境变量

在 Shell 中，问题相较于编程语言会更加复杂。

以 bash 为例，Shell 中可以设置变量：

```console
$ MY_VAR=1
```

但是这一变量的取值不能传递给子进程。

```console
$ MY_VAR=1
$ bash -c 'echo $MY_VAR'

```

> 此处第二条命令的输出为空。注意此处必须使用单括号。

因此此处的变量不是环境变量，而只是该 Shell 进程内的局部变量。

为了设置环境变量，需要使用 `export` 命令。

```console
$ export MY_VAR=1
$ bash -c 'echo $MY_VAR'
1
```

但如果在当前环境中已经定义了环境变量呢？我们会发现修改环境变量的值不再需要使用 `export`。

```console
$ export MY_VAR=1
$ MY_VAR=2
$ bash -c 'echo $MY_VAR'
2
```

由于这一点，我们可以做到一些比较出人意料的事。我们将之前的代码写到下面的脚本中：

```bash
# penetrate.sh
MY_VAR=1
bash -c 'echo $MY_VAR'
```

按照之前在命令行中的情况，该脚本的输出应当为空，确实如此。

```console
$ bash penetrate.sh

```

但如果我们在执行脚本前将 `MY_VAR` 定义为环境变量呢？

```console
$ export MY_VAR=1234
$ bash penetrate.sh
1
```

可以看到，脚本中的 `MY_VAR=1` “穿透” 到了子 bash 进程中，很有意思。

有时想要设置的环境变量太多，我们就希望将其写到一个脚本中。我们将带有 `export` 的代码写到脚本中。但这样做对吗？

```console
$ cat envs.sh
export MY_VAR=1
$ bash envs.sh
$ echo $MY_VAR

```

正如前面提到的，子进程的修改不会影响父进程的环境变量。`bash envs.sh` 会运行一个新的 bash 进程，所以无法修改子进程。原本在命令行中有效的操作现在又失效了。

bash 中提供了一个内建命令 `source`，用于将脚本中的命令在当前脚本中执行。让我们用 `source` 替换掉 `bash`。

```console
$ source envs.sh
$ echo $MY_VAR
1
$ bash -c 'echo $MY_VAR'
1
```

现在我们成功将 `MY_VAR` 设置为了环境变量。而如果在脚本中不加 `export` 命令，则会将 `MY_VAR` 设置成局部变量。

```console
$ cat envs.sh
MY_VAR=1
$ source envs.sh
$ echo $MY_VAR
1
$ bash -c 'echo $MY_VAR'

```

最后还有另一种定义环境变量的方式。在要执行的命令之前添加环境变量。此时只有子进程中存在环境变量的取值，而父进程则不受影响。

```console
$ MY_VAR=1 bash -c 'echo $MY_VAR'
1
$ echo $MY_VAR

```

这里我们想到了环境变量的穿透问题。那么如果我们此时使用 source 的话会发生什么？

```console
$ cat envs.sh
MY_VAR=1
$ MY_VAR=2 source envs.sh
$ echo $MY_VAR

```

结果竟然是 `MY_VAR` 未定义？而如果我们再定义一个 `MY_VAR2` 呢？

```console
$ cat envs.sh
MY_VAR=1
MY_VAR2=1
$ MY_VAR=2 source envs.sh
$ echo $MY_VAR

$ echo $MY_VAR2
1
```

竟然只有 `MY_VAR` 未定义，很有意思。

## 三、判断是否是环境变量

由于 Shell 中的变量都通过相同的文法获取，所以我们并不能直接判断哪些是局部变量，哪些是环境变量。但环境变量却会传递到子进程中，因此一些情况下判断变量是否为环境变量就很重要了。

这里的方法是使用 `printenv` 命令。`printenv` 命令直接使用可以获取当前 Shell 进程中的所有环境变量。如果为该命令指定参数，则可以获取参数名所对应的环境变量取值。并可以根据返回值判断环境变量是否存在。

```console
$ MY_VAR=1
$ printenv MY_VAR
$ echo $?
1
$ export MY_VAR=1
$ printenv MY_VAR
1
$ echo $?
0
```
