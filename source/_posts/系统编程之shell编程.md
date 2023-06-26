---
title: 系统编程之shell编程
tags:
  - shell
  - 系统编程
categories: 学习笔记
abbrlink: 757349f6
date: 2022-10-08 19:30:58
---

## 一、前言
本文将简单探索shell脚本编程，介绍shell的基本语法。

## 二、shell简介
shell是一个命令解释器，可以用来启动、停止、编写程序；是用户和UNIX/Linux操作系统内核程序间的一个接口。

而shell编程则是将linux命令与shell的各种流程控制和条件判断来组合成命令与变量，形成可以进行自动处理的脚本程序。

## 三、前期准备
### 创建脚本
shell脚本是一个文本文件，可用文本编辑器如vi、vim编辑保存。创建shell脚本只需按照创建文本文件的方式创建。如
``` bash
vi c1.sh
vim c2.sh
> c3.sh
```
> shell脚本一般以`.sh`为后缀，但没有后缀依旧可以执行。

创建的shell脚本，一定要在开头第一行加上如下语句：
``` shell
#!/bin/bash
```
这一行将指明该脚本执行所需要的命令解释器。

### 执行脚本
shell脚本的执行方法有
``` bash
sh <scriptname>
bash <scriptname>
```
或者使用chmod命令修改脚本为可执行，再直接使用 `./<scriptname>` 运行。

## 四、基本语法
### 变量
shell中的变量分为环境变量、用户定义变量、内部变量。

其中环境变量是操作系统的一部分，但可以利用shell脚本进行修改；用户变量即在脚本中声明的变量；而内部变量则用来指示脚本运行中出现的一些变量。

#### 声明
只有用户变量可以声明。和其他语言一样，使用等号进行声明。但要注意的是，shell脚本是弱类型的，因此变量名前不需要加上类型名。
```shell
var=hello_world
```
> 注意shell对空格敏感，声明时等号两边不能有空格

另外可以在变量名前添加 `readonly`关键字设为只读
```shell
readonly constVar
```

shell中声明数组同样直接写出数组名称
```shell
arr[0]=1
arr[1]=5
arr[10]=20
```
未赋值的部分默认为NULL。
> 注意 ubuntu 默认使用 dash 而非 bash shell。dash 并不支持数组。要使用数组可以用bash运行脚本，即运行命令 `bash <scriptname>`

#### 赋值
shell变量有类似左值右值的区别。在用其他变量进行赋值时，需要对变量使用`${ }` 进行取值。
```shell
var1=hi
var2=${var1}
```

#### 引号
shell脚本是为了自动化处理命令而设计的。因此语法中有很大一部分关注于字符串和命令的相关操作。在变量上体现在，shell中所有变量默认以字符串形式存在。

并且，为了满足命令处理的需要，shell设计出了引号变量值。

shell中的引号包括单引号、双引号和倒引号。

**单引号**中的字符均作为普通字符出现。（可以包括空格）
```shell
var1=hello_world
# var2=hello world # 不合法
var3='hello world'
```

**双引号**中的字符大部分作为普通字符对待。除了`$\’`和双引号，这些变量依旧用于对字符串内容进行变量替换。
```shell
var4=hello
var5=friend
var6="$var4, my $var5!"
# var6 means "hello, my friend!".
```

**倒引号**将引号内的内容当做命令，会先执行倒引号内的内容，再用执行后的输出替换倒引号的内容。
> 倒引号可以和双引号组合使用，在双引号内使用倒引号
> $(command) 与 \`command\` 功能相同

```shell
var7="now pwd: `pwd`"
# var6 means "now pwd: /home/username" (depend on the position where you execute the script)
```
### 基础操作
#### 字符串操作
shell脚本自身具有一些字符串操作功能。

- 字符串长度：
  ```shell
  ${#str}
  ```
- 字符串截取：
  ```shell
  ${str:position}
  ${str:start:length}
  ```
- 从字符串开头删除：
  ```shell
  ${str#substr}
  ${str##substr}
  ``` 
- 从字符串末尾删除：
  ```shell
  ${str%substr}
  ${str%%substr}
  ```
- 字符串替换：
  ```shell
  ${str/substr/replace}
  ```

> 字符串操作中出现匹配的部分都支持正则表达式。其中一个字符的为懒惰匹配，两个字符的为贪婪匹配。

> 使用 `expr` 等命令，可以更好地进行处理。

#### 数字运算
shell默认变量为字符串，因此若要进行数字运算，需要特殊指明。

具体地，有两种方法。一种是利用 `$(())` 运算符。指明括号内的表达式进行的不是字符串操作而是数字运算。
```shell
num1=10
num2=20
num3=$(($num1+$num2)) # 30
```

另一种方法是调用 linux 的 expr 命令。
```shell
num4=3
num5=$num4
num6=`expr $num4 \* $num5` # 9
```
> 注意乘号在shell脚本中有特殊含义，因此需要加入反斜杠转义。另外 expr 命令中变量和运算符各自作为参数，因此中间需要用空格隔开.

### 流程控制
#### 条件表达式
shell 利用 `test condition` 和 `[condition]` 进行条件测试。其中“condition” 表示一个条件表达式。包括字符比较、数值比较、文件操作、逻辑操作。

这一部分原理较为简单，但shell中判断符号与其他语言有较大差别，内容较多，就不一一列举了。

> 注意使用条件表达式时，表达式与方括号间要有空格隔开

#### 分支控制
分支控制语句如下

if 分支：
```shell
if [ condition ]
then
    commands
elif
    commands
else
    commands
fi
```
> 注意末尾的 fi

case分支（类似switch）：
```shell
case variable in
var1)
    commands
    ;;
var2 | var3)
    commands
    ;;
*)
    commands
esac
```
> 注意两个连续的分号，类似于 break。进入同样分支的变量值要用 “|” 隔开。末尾要添加 esac。

#### 循环控制
循环控制语句如下

while 循环：
```shell
while [ condition ]
do
    commands
done
```

until 循环：
```shell
until [ condition ]
do
    commands
done
```
> until 循环与条件取反的 while 循环等价

for 循环如下：
```shell
for arg in args
do
    commands
done
```
> 其中 args 是一组列出的变量名或字符串

### 函数
#### 函数的定义
```shell
function func_name{
    commands
}

func_name(){
    commands
}
```
两种方式均可定义函数。

#### 参数取值
shell中的函数只有可变参数。另外，整个shell脚本本身也可以看做一个函数，它同样也只有可变参数。都具有相同的取参数的方式。

shell中通过 `$0 $1 $2 ... $9`等参数来获取函数对应位置的参数。其中 `$0` 最开始为函数名/脚本名。当参数多于 10 个时，要获取十个之外的参数，可以使用 `shift` 命令，使得除了函数名/脚本名 $0 之外的所有参数整体左移一位。
```shell
# sh <scriptname>.sh 123 456
echo $1 # 123
shift
echo $1 # 456
```
> `echo` 为输出命令

#### 函数返回值
函数通过 `return [n]` 语句来返回变量值 n。如果没有设置返回值，那么会默认返回函数最后一条命令执行后的返回值。

在函数调用之后，可以使用 `$?` 获取函数的返回值。

> 注意 return 的值必须只能是 0~255间的整数。要返回字符串或者更大的数字，可以直接使用 `echo` 输出内容，再通过倒引号获取输出的内容。

#### 函数实例
编写一个递归计算阶乘的函数
```shell
frac()
{
    if [ $(($1)) -ge 1 ]
    then
        echo $((`frac $(($1-1))`*$1))
    else
        echo 1
    fi
}

echo `frac 5` # 120
```