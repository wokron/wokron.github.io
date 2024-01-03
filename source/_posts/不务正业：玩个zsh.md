---
title: 不务正业：玩个zsh
tags:
  - shell
  - zsh
  - starship
category: 踩坑记录
abbrlink: 93ea500a
date: 2023-10-21 17:13:03
---
## 一、前言
不知道为什么，这学期看到好几个人都用 zsh。最近恰好有时间，我也是时候换一个 shell 了。

不过嘛，shell 毕竟是用来工作的，不是一件艺术品，所以这里的配置也是以实用为主了。本文也没有用到 oh-my-zsh，用一个专门的框架来管理 shell 的配置在我看来还是太过沉重了。

> 本篇文章所用的环境当然是 linux。更加具体来说是 debian 系的发行版。

<!-- more -->

## 二、zsh 的安装
这里当然不会仔细介绍 zsh，将网上许多其他文章都重复过的内容再重复一遍。zsh 只是一个 shell，只不过这个 shell 有较强的可扩展性，同时具有一些比较有用的特性，仅此而已。

安装 zsh，只需要一条命令
```sh
apt-get install zsh
```

之后可以查看 /etc/shells 文件的内容，该文件记录了系统中已有的所有 shell，此时应当也有 zsh。
```sh
cat /etc/shells
```

本人的输出结果是这样的
```sh
# /etc/shells: valid login shells
/bin/sh
/bin/bash
/usr/bin/bash
/bin/rbash
/usr/bin/rbash
/usr/bin/sh
/bin/dash
/usr/bin/dash
/bin/zsh
/usr/bin/zsh
```

之后首次运行 zsh
```sh
zsh
```

应该会出现如下内容，因为此时我们并未创建 zsh 的配置文件。正如 bash 的 `.bashrc`。这里只需要选择 `0` 即可。之后会在用户目录创建一个空的 `.zshrc`，该文件将在下一小节中详细说明。选择后重新输入命令 `zsh` 进入 zsh，此时应该可以看到输入提示符等内容。
```text
This is the Z Shell configuration function for new users,
zsh-newuser-install.
You are seeing this message because you have no zsh startup files
(the files .zshenv, .zprofile, .zshrc, .zlogin in the directory
~).  This function can help you with a few settings that should
make your use of the shell easier.

You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

(2)  Populate your ~/.zshrc with the configuration recommended
     by the system administrator and exit (you will need to edit
     the file by hand, if so desired).

--- Type one of the keys in parentheses --- 
```

我们当然不希望每次新建终端都要在 bash 中输入一次 `zsh`。因此应当将当前用户的默认终端修改为 zsh。输入如下命令即可。此命令需要输入用户密码。

```sh
chsh -s /bin/zsh
```

之后可以查看一下 /etc/passwd 文件，搜索当前用户名，查看其 shell 是否修改为了 zsh。
```sh
vim /etc/passwd
```

此时如果新建终端，会发现使用的依旧是 bash。因为应用修改需要重启系统。当然现在还是不要重启，接着使用 bash 吧，因为现在的 zsh 比 bash 还要简陋。

## 三、`.zshrc` 的配置
如果你修改过 bash 的配置，或者至少在 Linux 下安装过一些编程语言的环境，如 python 的 conda、golang 和 rust 的编译环境等等，应该知道用户目录下有一个 `.bashrc` 文件。这个文件用于对 bash 所使用的环境进行配置。在每次创建一个 bash 的时候，bash 会首先执行命令 `source ~/.bashrc`。`source` 命令将使用当前的终端依次执行文件中的每条内容，这样就完成了 bash 初始化。

`.zshrc` 也是类似的配置文件。只不过 bash 和 zsh 的一些操作可能并不兼容，导致我们并不能直接将 `.bashrc` 的内容复制到 `.zshrc` 中。当然我们的目的是快速配置一个可用且美观的终端，所以本文不会也不可能详述 zsh 具体有什么特性，有什么和其他 shell 的区别。所以接下来我就简单列举一些自己用到的，较为实用配置。

### （1）基本命令的颜色
现在你使用毫无配置的 zsh，就连一个简单的 `ls` 命令都是没有颜色的，这怎么能忍？必须首先解决这个问题。

很庆幸的是这里 zsh 完全和 bash 兼容，默认的 `.bashrc` 中应该有如下的内容，只要将如下内容复制到 `.zshrc` 中即可。
```sh
# enable color support of ls and also add handy aliases
if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
    alias ls='ls --color=auto'
    #alias dir='dir --color=auto'
    #alias vdir='vdir --color=auto'

    alias grep='grep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias egrep='egrep --color=auto'
fi
```

### （2）启用命令历史
zsh 竟然默认不启用命令历史，这导致每次新建一个终端，都没办法获取其他终端的命令历史。`.bashrc` 中可以找到和命令历史相关的配置，但可以这回直接复制并不会起作用
```sh
# don't put duplicate lines or lines starting with space in the history.
# See bash(1) for more options
HISTCONTROL=ignoreboth

# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)
HISTSIZE=1000
HISTFILESIZE=2000
```

在 zsh 中，还需要设置记录历史的文件 `HISTFILE` 以及保存的历史数量 `SAVEHIST`。这样之前的命令历史才会记录下来。注意这里具体的数值可以自己设置。
```sh
# config history
HISTFILE=~/.zsh_history
SAVEHIST=1000
HISTSIZE=1000
HISTFILESIZE=2000
```

### （3）删除单个单词
用过 vscode 应该会觉得其中的 shell 操作很舒服。比如说在默认设置下 `Ctrl + Backspace` 可以直接删除单个单词。其实 shell 中也已经内置了这一操作，只不过快捷键是 `Ctrl + W`，这就很不直观了。zsh 中可以将某一按键绑定到某一操作。这样就可以使用 `Ctrl + Backspace` 删除单个单词了。只需要如下一行指令即可。

```sh
# enable kill word using ctrl + backspace
bindkey '^H' backward-kill-word
```

### （4）补全功能选择菜单
稍微使用一下我们尚不完善的 zsh，你应该会发现补全操作似乎和 bash 中不一样。bash 中会在双击 tab 后补全或列出补全选项，而 zsh 则单击 tab 便会显示补全。这样做不只是为了禁止你在 zsh 中使用制表符（哈哈），而是因为双击 tab 还有另外的功能，只不过现在没有启用。

在 `.zshrc` 中添加如下一行，就可以在补全时启用选择菜单。在双击 tab 后会将下方列出的补全选项变为菜单，可以通过方向键选择具体的补全选项，按回车补全选定的内容。当然也可以按 `Backspace` 退出选择。
```sh
# enable menu
zstyle ':completion:*' menu select
```

### （5）不要忘记其他配置
一些程序在安装时会在 `.bashrc` 中添加内容，在搬家到 zsh 时不要把它们落下。这些程序比如说 rust、golang 的编译环境、nvm（nodejs version manager）、conda 等等。conda 似乎是其中的特例。你应该（在 bash 中）调用 `conda init zsh` 而非将 `.bashrc` 中的内容复制到 `.zshrc` 中。

## 四、zsh 插件的安装
添加了这些内容后，zsh 开始像个回事了。zsh 已经完全具有了 bash 的基本内容，还拥有了一些特殊的功能，除了外观以外已经可以使用了。但是 zsh 还可以通过插件进一步提高其操作手感。这里和许多提及 zsh 的文章一样，推荐两个插件 `zsh-autosuggestions` 和 `zsh-syntax-highlighting`。

第一个插件会在输入命令时根据你的历史输入在已输入的内容后使用浅灰色指出你可能想要的输入，如果其确实是你所需要的，只需要按右方向键即可补全。

第二个插件会在输入命令的时候对命令自动染色，尤其对于不存在的指令会显示为红色，这对命令的检查很有帮助。

很好的是 `apt` 中已经有了这两个插件的包，只需要直接下载即可。
```sh
apt-get install zsh-autosuggestions
apt-get install zsh-syntax-highlighting
```

> 你问我怎么知道的？这里提一下 apt 的搜索功能，似乎比较少有人提及。只需要执行如下命令即可搜索 zsh 相关的所有包
> ```sh
> apt-cache search zsh
> ```
> 使用如下命令还能得知软件源中的具体包版本
> ```sh
> apt-cache madison zsh-syntax-highlighting
> ```

之后只需要将两个插件启用即可。只需要在 `.zshrc` 中添加如下两行。其中的路径是 `apt` 安装时的路径。
```sh
# enable autosuggestions and syntax highlighting
source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

## 五、Starship 定制外观
如果你使用 zsh 是为一些炫酷的主题，那抱歉这一小节似乎无法提供给你，或许 oh-my-zsh 会更加适合。但是如果你只是想要一个实用且美观的 zsh 外观，那么这部分应该还是可以起到帮助的。

这一小节会通过 [Starship](https://starship.rs/)，按官网所述，一个”轻量、迅速、可无限定制的高颜值终端“，来定制 zsh 的外观。值得注意的是 Starship 是跨 shell 的，这意味着你可以在自己系统的所有 shell 中都应用自己的配置。这是 oh-my-zsh 无法做到的。

### （1）安装 Nerd Font 字体（非必须）
Starship 会使用 [Nerd Font](https://www.nerdfonts.com/) 字体中的一些符号，因此需要下载一种 Nerd Font 类型的字体。可以从 Nerd Font 的官网上选择自己喜欢的字体。但之后并不一定需要直接在网站上下载压缩包。可以从 `apt` 或发行版本身的软件管理器上下载对应字体。如你想要使用 `fira` 字体，可以在 `apt` 上搜索相关内容，可以搜到相应的结果。
```sh
$ apt-cache search fira
fonts-firacode - Monospaced font with programming ligatures
```

> 当然，字体其实也并非必须的，你可以选择使用全 ascii 的配置，同样有很好的效果，这一点会在之后说明。

之后还需要配置终端的字体。不同的模拟终端软件应该有着不同的配置方法，这里肯定不能一一列举，只提一下 GNOME Terminal（默认终端） 和 vscode 终端的配置。

对于 GNOME Terminal，只需要右键终端，选择配置首选项，之后选择文本，使用自定义字体，选择新安装的字体即可。而对于 vscode，需要打开配置，搜索 `terminal.integrated.fontFamily` 将其值设置为新安装的字体名即可。

### （2）安装 Starship
执行如下命令即可安装。
```sh
curl -sS https://starship.rs/install.sh | sh
```

之后在 `.zshrc` 中添加如下内容，以启用 Starship
```sh
eval "$(starship init zsh)"
```

在当前 zsh 中执行 `source .zshrc` 或创建一个新的 shell，现在你的 zsh 外观应该已经大变样了！

### （3）配置 Starship
Starship 使用一个配置文件 starship.toml 来配置 shell 外观。默认应该位于 ~/.config/starship.toml。Starship 官网上已经预设了一些[外观主题](https://starship.rs/presets)，可以直接按照官网指导进行主题切换。只需要一条指令就可完成配置，比如下面的指令。
```sh
starship preset pastel-powerline -o ~/.config/starship.toml
```

不过实际上我还是比较喜欢纯 ascii 的样子，这种主题也不需要使用 Nerd Font
```sh
starship preset plain-text-symbols -o ~/.config/starship.toml
```

Starship 也提供了极其繁多的配置，可以在[文档](https://starship.rs/config/)中查找自己感兴趣的内容自行配置。不过如果只是简单的配置一下的话，并不需要去把全部内容都看一遍。只需要了解如下几个内容即可。

在配置最开始使用 format 定义你的提示符格式。默认的配置是这样的
```ini
format = '$all'
```

实际等价于
```ini
format = """
$username\
$hostname\
$localip\
$git_branch\

omit...

$cmd_duration\
$line_break\
$jobs\
$battery\
$time\
$status\
$os\
$container\
$shell\
$character"""
```

其中每一个 `$` 开头的变量都是用于表示对应内容在终端显示上的位置的占位符，当然其中很大部分都是不同语言的提示，也有很多都是未启用的功能。

如果想要自定义配置，其实并不需要按照和如上的格式一样将所有可能都列举出来。而只需要将想要移动位置的字符从 `$all` 中提出来即可。比如如果想要把用户名放在输入提示符的前面位置，只需要使用如下的 format 即可
```ini
format = '$all $username $character'
```

Starship 默认启用的功能就已经能够满足日常使用了。除此之外还可以启用一些功能，如用户名、时间等等，这就有待各位的探索了。

最后，展示一下我自己的 zsh 外观，各位也一定能配置出自己满意的外观。
{% asset_img zsh.png my-zsh %}
