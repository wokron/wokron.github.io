---
title: 关于git推送失败
tags: git
categories: 踩坑记录
abbrlink: ebe0c30f
date: 2022-09-10 14:40:57
---

## 问题背景
其实这个问题出现过许多次了，可每次的解决方法都不尽相同。因此这里做一个简单的汇总，也为之后再次出现提供一个总结的位置。

曾经出现该问题的情况：
1. VS推送失败
2. hexo推送失败

问题形式类似于`fatal: unable to access 'https://github.com/.../.git':...`

## 问题原因
貌似是代理产生的问题。根据网上资料，通过取消代理就可以解决问题。但本人也曾有过通过更换VPN线路解决的情况，不知是否仅仅是凑巧解决。

## 问题解决
查看代理：
``` bash
git config --global --get http.proxy
git config --global --get https.proxy
```
如果发现`git`使用了代理，则取消代理：
``` bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```
需要再次设置代理请使用如下命令（设置当前代理为 http://127.0.0.1:1080）：
``` bash
git config --global http.proxy 'http://127.0.0.1:1080'
git config --global https.proxy 'http://127.0.0.1:1080'
```

值得注意的是，`git`还可以对单一网站设置代理，例如只对`github`使用代理http://127.0.0.1:58591：
``` bash
git config --global http.https://github.com.proxy http://127.0.0.1:58591
```
该类代理无法通过前面的查看代理命令查看，需要明确指出网站才可查询到：
``` bash
git config --global --get http.https://github.com.proxy
```
取消对github的代理：
``` bash
git config --global --unset http.https://github.com.proxy
```
另外代理设置也可以直接在git配置文件中查看，为`C:\Users\{用户名}\.gitconfig`。也可以在其中直接修改。