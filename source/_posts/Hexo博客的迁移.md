---
title: Hexo博客迁移教程
tags: Hexo
categories: 教程
abbrlink: 33f55625
date: 2023-06-26 14:44:44
---
## 一、前言
因为用了新的笔记本，为了继续更新自己的博客，我决定把原来那台笔记本上的博客资源迁移过来。不过呢，当然不能用u盘拷贝这种比较low的方法，最好还是把资源放到 github 上，这样不仅方便现在的迁移，更能防止数据丢失。

<!-- more -->

## 二、将博客资源推送到仓库
如果你使用 hexo 搭建了自己的博客，并且把博客放到了 github 上，那么很容易注意到使用 hexo 部署时并不是将本地的所有内容推送到了 github，实际推送的只是 ./public 路径下的文件。而现在我们要做的就是将博客的所有资源推送到仓库，不仅是用于网页的部分。

我们选择就在博客网站所在的仓库存储博客资源，为了做到这一点，首先要在本地克隆一个仓库
```sh
git clone https://github.com/<username>/<username>.github.io.git
```

随后我们新建一个分支用于存储博客资源。该分支与博客网站所使用的 master 分支无关，因此最好创建成一个“孤儿”分支。
```sh
git checkout --orphan <branch_name>
```

切换到该分支后，原本随着克隆拉取到本地的文件现在依旧存在，需要将这些文件删除
```sh
git rm -rf .
```

接着将位于本地的博客资源复制到该文件夹下。
```sh
cp -r <old_blog_dir>/* .
```

这里需要注意，如果你使用了 next 等主题，并且是通过克隆仓库的方式下载的，那么此时应该把主题对应的项目路径下的 .git 文件夹删除。
```sh
# take next theme as example
rm -r ./themes/next/.git
```

以上的工作都完成后，将这些复制到仓库中的博客资源文件添加并提交
```sh
git add .
git commit -m "commit info"
```

最后将本地分支推送到远程仓库的新分支中
```sh
git push --set-upstream origin <remote_branch_name>
```

## 三、迁移博客
接下来要将博客迁移到另一台设备上。首先当然要下载 git 并配置用户名和邮箱
```sh
sudo apt install git
git config --global user.name <username>
git config --global user.email <email>
```

之后克隆仓库并切换到博客资源所在的分支
```sh
git clone https://github.com/<username>/<username>.github.io.git
cd <username>.github.io
git checkout -b <branch_name> origin/<remote_branch_name>
```

接着下载 nodejs、npm 和 hexo
```sh
sudo apt install nodejs
sudo apt install npm
sudo npm install -g hexo-cli
```

最后下载项目中使用的其他包
```sh
npm install
```

## 四、在新设备上生成网页以及部署
hexo 的命令不用多说，可以用 `hexo g` 生成网页，并使用 `hexo s` 命令在本地运行。

最后使用 `hexo d` 将网页部署到 github 上。但是这时可能会出现如下的信息：
```text
# omit...
Username for 'https://github.com': <username>
Password for 'https://wokron@github.com': <password>
remote: Support for password authentication was removed on August 13, 2021.
remote: Please see https://docs.github.com/en/get-started/getting-started-with-git/about-remote-repositories#cloning-with-https-urls for information on currently recommended modes of authentication.
# omit...
```

这时就需要增加授权。可以使用 ssh 生成密钥。输入如下命令后按三次回车。
```sh
ssh-keygen -t rsa -C <email>
```

之后查看 ~/.ssh/id_rsa.pub 中的明文密钥
```sh
cat ~/.ssh/id_rsa.pub
```

进入 github，找到 Settings - SSH and GPG keys - SSH keys - New SSH Key，将该密钥粘贴并保存。之后输入如下命令查看密钥是否设置成功。
```sh
ssh git@github.com
```

此时可能会输出如下内容，这时只要输入 yes 并回车即可。
```text
The authenticity of host '[ssh.github.com]:443 ([20.205.243.160]:443)' can't be established.
ED25519 key fingerprint is <fingerprint>.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? 
```

最终出现如下输出则表示成功
```text
PTY allocation request failed on channel 0
Hi wokron! You've successfully authenticated, but GitHub does not provide shell access.
Connection to ssh.github.com closed.
```

最后打开博客的配置文件 _config.yml，将其中部署部分改为如下形式，使用 ssh 链接进行部署
```yml
deploy:
  type: git
  repo: git@github.com:<username>/<username>.github.io.git
  branch: master
```

最后再次应用 `hexo d` 即可部署。