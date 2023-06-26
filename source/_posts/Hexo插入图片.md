---
title: Hexo插入图片
tags: Hexo
categories: 测试
abbrlink: e01ea1fb
date: 2022-09-10 15:32:12
---
## 插入图片
![](blackcat.jpg)
以上图片被保存在路径`./Hexo插入图片/`中。插入图片使用的代码为：
``` markdown
![](blackcat.jpg)
```
注意这样写在VScode中无法显示图片。Vscode中写法应为：
``` markdown
![](/Hexo插入图片/blackcat.jpg)
```

## 配置
需要使用插件hexo-renderer-marked。利用命令直接安装：
``` markdown
npm install hexo-renderer-marked
```
之后在config.yaml中更改配置如下： 
``` markdown
<!-- 用于在生成文章时生成对应的资源文件夹，请在配置文件中搜索并修改为true -->
post_asset_folder: true

<!-- 插件配置 -->
marked:
  prependRoot: true
  postAsset: true
```