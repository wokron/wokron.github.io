---
title: Hexo个人博客搭建及主题配置教程
tags: Hexo
categories: 教程
abbrlink: 6cacc70
date: 2022-09-11 18:28:06
---

## 一、前言
本文是作者对于Hexo搭建的阶段总结。汇总了一些搭建过程中找到的资料。希望能够较为全面的记录博客搭建的全过程。给自己备忘，为他人提供帮助。

## 二、搭建阶段
### （1）Hexo简介
Hexo是一款基于Node.js的静态博客框架，依赖少易于安装使用，可以方便的生成静态网页托管在GitHub和Coding上，是搭建博客的首选框架。大家可以进入hexo官网进行详细查看，因为Hexo的创建者是台湾人，对中文的支持很友好，可以选择中文进行查看。

### （2）前期安装
#### 安装Git
Git是目前世界上最先进的分布式版本控制系统，可以有效、高速的处理从很小到非常大的项目版本管理。也就是用来管理你的hexo博客文章，上传到GitHub的工具。

windows：到git官网上下载,Download git,下载后会有一个Git Bash的命令行工具，以后就用这个工具来使用git。windows在git安装完后，就可以直接使用git bash来敲命令行了。

linux：对linux来说实在是太简单了，因为最早的git就是在linux上编写的，只需要一行代码
``` bash
sudo apt-get install git
```
安装好后，用`git --version`来查看一下版本。

#### 安装nodejs
Hexo是基于nodeJS编写的，所以需要安装一下nodeJs和里面的npm工具。

windows：nodejs选择LTS版本就行了。

linux：
``` bash
sudo apt-get install nodejs
sudo apt-get install npm
```
安装完后，打开命令行
``` bash
node -v
npm -v
```
检查一下有没有安装成功

#### 安装Hexo
前面git和nodejs安装好后，就可以安装hexo了，可以先创建一个文件夹blog，然后cd到这个文件夹下（或者在这个文件夹下直接右键git bash打开）。

输入命令
``` bash
npm install -g hexo-cli
```
用`hexo -v`查看一下版本，至此就全部安装完了。

接下来初始化一下hexo
```bash
hexo init myblog
```
这个myblog可以自己取什么名字都行，然后
``` bash
cd myblog //进入这个myblog文件夹
npm install
```
新建完成后，指定文件夹目录下有：

- node_modules: 依赖包
- public：存放生成的页面
- scaffolds：生成文章的一些模板
- source：用来存放你的文章
- themes：主题
- _config.yml: 博客的配置文件**

打开hexo的服务
``` bash
hexo g
hexo server
```
在浏览器输入localhost:4000就可以看到你生成的博客了。

### （3）Github部署
#### GitHub创建个人仓库
首先，注册Github账号。

注册完登录后，在网站中点击New repository，新建仓库

创建一个和用户名相同的仓库，后面加.github.io，只有这样，将来要部署到GitHub page的时候，才会被识别，也就是xxxx.github.io，其中xxx就是你注册GitHub的用户名。

点击create repository，创建仓库。

#### 生成SSH添加到GitHub
回到git bash中，输入
``` bash
git config --global user.name "yourname"
git config --global user.email "youremail"
```
yourname输入GitHub用户名，youremail输入GitHub的邮箱。

可以用以下两条，检查一下你有没有输对
``` bash
git config user.name
git config user.email
```
然后创建SSH，输入
``` bash
ssh-keygen -t rsa -C "youremail"
```
接着一路回车。
会显示已经生成了.ssh的文件夹。在你的电脑中找到这个文件夹。

在GitHub的setting中，找到SSH keys的设置选项，点击New SSH key
把id_rsa.pub里面的信息复制进去。

在gitbash中，查看是否成功
``` bash
ssh -T git@github.com
```

#### 将hexo部署到GitHub
这一步，我们就可以将hexo和GitHub关联起来，也就是将hexo生成的文章部署到GitHub上，打开站点配置文件 _config.yml，翻到最后，修改为
``` bash
deploy:
  type: git
  repo: https://github.com/YourgithubName/YourgithubName.github.io.git
  branch: master
```
YourgithubName就是你的GitHub账户

这个时候需要先安装deploy-git ，也就是部署的命令,这样你才能用命令部署到GitHub。
``` bash
npm install hexo-deployer-git --save
```
然后
``` bash
hexo clean
hexo generate
hexo deploy
```

其中 hexo clean清除了你之前生成的东西，也可以不加。
hexo generate 顾名思义，生成静态文章，可以用 hexo g缩写
hexo deploy 部署文章，可以用hexo d缩写

注意deploy时可能要你输入username和password。

得到下图就说明部署成功了，过一会儿就可以在http://yourname.github.io 这个网站看到你的博客了！！

### （4）Hexo基本配置
#### Hexo框架配置
文件根目录下的_config.yml，就是整个hexo框架的配置文件了。可以在里面修改大部分的配置。

这里列出几个比较主要的配置参数：

| 参数 | 描述 | 例子 |
| -- | -- | -- | 
| title | 网站标题 | xxx的博客 |
| subtitle | 网站副标题 | 技术记录 |
| description | 网站描述 | 一个技术宅的博客 |
| author | 您的名字 | author |
| language | 网站使用的语言 | zh-CN |
| timezone | 网站时区,默认使用电脑时区 | Asia/Shanghai |

#### 文件配置
输入命令
``` bash
hexo new newpaper
```
可以创建一篇新的文章。newpaper为文章的名字，格式为markdown文件。

在文件最上方以 --- 分隔的区域，可用于指定个别文件的变量，举例来说：
``` markdown
---
title: Hexo个人博客搭建及主题配置教程
date: 2022-09-11 18:28:06
tags: Hexo
categories: 教程
---
```
以下是预先定义的参数，可在模板中使用这些参数值并加以利用。

|参数|描述|
|--------|--------|
| layout|布局|
| title|标题|
| date|建立日期|
| updated|更新日期|
| comments|开启文章的评论功能|
| tags|标签（不适用于分页）|
| categories|分类（不适用于分页）|
| permalink|覆盖文章网址|
其中，分类和标签需要区别一下，分类具有顺序性和层次性，也就是说 Foo, Bar 不等于 Bar, Foo；而标签没有顺序和层次。

## 三、更换主题
本文将以NexT主题为例
### （1）安装并启用 NexT 主题
#### 安装主题
打开博客根目录文件夹，右键Git Bash，输入如下代码将next主题下载到目录`./themes`：
``` bash
git clone https://github.com/theme-next/hexo-theme-next themes/next
```
打开主题文件夹下的package.json文件可以查看NexT主题版本。本文使用的版本为
``` json
"version": "7.8.0",
```

#### 启用主题
打开根目录下的_config.yml(称为站点配置文件)，修改主题（注意冒号后都要有空格）。搜索theme参数，修改为next
``` yaml
theme: next
```
另外next主题有四种样式，分别为Muse、Mist、Pisces、Gemini，这里设置为Gemini，在`./themes/next/`下的_config.yml（称为主题配置文件）中搜索参数Schemes，将要选的主题取消注释：
``` yaml
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini # 选择该主题
```
回到根目录打开Git Bash，输入如下三条命令：
``` bash
hexo clean
hexo g
hexo d
```
将主题修改应用到网站上。

### （2）功能设置
#### 开启tags/categories
开启该功能，博客会将写过的文章按照tags或categories进行归类。

进入博客根目录下，输入命令
``` bash
hexo new page tags
hexo new page categories
```
会在`./source`下出现tags和categories文件夹。博客中将出现关于tags和categories的新页面。

打开文件夹中的文件，可以修改相关信息，类似于修改文章配置
``` markdown
---
title: 标签
date: 2022-09-10 10:33:40
type: "tags"
---
```

#### 修改侧边栏信息
在主题配置文件`./theme/next/_config.yml`中搜索avatar
``` yaml
# Sidebar Avatar
avatar:
  # Replace the default image and set the url here.
  url: /images/avatar.jpg
  # If true, the avatar will be dispalyed in circle.
  rounded: true
  # If true, the avatar will be rotated with the cursor.
  rotated: true
```
在这里可以设置侧边栏的头像。包括头像图片，头像是否原型，鼠标放置是否会转动。

同样的文件搜索social
``` yaml
social:
  GitHub: https://github.com/wokron || fab fa-github
  E-Mail: stringcatwok@gmail.com || fa fa-envelope
  #Weibo: https://weibo.com/yourname || fab fa-weibo
  #Google: https://plus.google.com/yourname || fab fa-google
  #Twitter: https://twitter.com/yourname || fab fa-twitter
  #FB Page: https://www.facebook.com/yourname || fab fa-facebook
  #StackOverflow: https://stackoverflow.com/yourname || fab fa-stack-overflow
  #YouTube: https://youtube.com/yourname || fab fa-youtube
  #Instagram: https://instagram.com/yourname || fab fa-instagram
  #Skype: skype:yourname?call|chat || fab fa-skype
```
可以设置相关的网络账户信息。

#### 修改代码框
搜索codeblock
``` yaml
codeblock:
  # Code Highlight theme
  # Available values: normal | night | night eighties | night blue | night bright | solarized | solarized dark | galactic
  # See: https://github.com/chriskempson/tomorrow-theme
  highlight_theme: night
  # Add copy button on codeblock
  copy_button:
    enable: true
    # Show text copy result.
    show_result: default
    # Available values: default | flat | mac
    style: mac
```
可以设置代码框相关的参数。包括代码主题、复制键是否开启、是否显示复制效果、代码框的样式（`style: mac`就可以设置成很美观的mac panel样式，不需要网上那些复杂的步骤）。

### （3）增加扩展
#### 添加Live2D
首先在博客目录下执行：
``` bash
npm install -save hexo-helper-live2d
```
然后在站点配置文件中加入：
``` yaml
live2d:
  enable: true
  scriptFrom: local
  pluginRootPath: live2dw/
  pluginJsPath: lib/
  pluginModelPath: assets/
  tagMode: false
  log: false
  model:
    use: live2d-widget-model-hijiki #选择哪种模型
  display:
    position: right
    width: 150
    height: 300
  mobile:
    show: true
```
接着下载对应模型
``` bash
npm install live2d-widget-model-hijiki
```

#### 文章图片支持
这里使用相对引用插入图片

安装插件`hexo-renderer-marked`：
``` bash
npm install hexo-renderer-marked
```
修改站点配置文件
``` yaml
post_asset_folder: true
```
同时在其中添加：
``` yaml
marked:
  prependRoot: true
  postAsset: true
```
就可以进行图片的引用了。

#### 公式支持
需要更换 Hexo 的 markdown 渲染引擎，hexo-renderer-kramed 引擎是在默认的渲染引擎 hexo-renderer-marked 的基础上修改了一些 bug ，两者比较接近，也比较轻量级。
``` bash
npm uninstall hexo-renderer-marked --save
npm install hexo-renderer-kramed --save
```
执行上面的命令，先卸载原来的渲染引擎，再安装新的。

接下来到博客根目录下，找到`node_modules\kramed\lib\rules\inline.js`，修改 escape以及em变量的值：
``` js
//escape: /^\\([\\`*{}\[\]()#$+\-.!_>])/,
escape: /^\\([`*\[\]()#$+\-.!_>])/,

//em: /^\b_((?:__|[\s\S])+?)_\b|^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
em: /^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
```

然后在主题中开启MathJax。打开主题配置文件，搜索mathjax，改为
``` yaml
# hexo-renderer-pandoc (or hexo-renderer-kramed) required for full MathJax support.
  mathjax:
    enable: true
    # See: https://mhchem.github.io/MathJax-mhchem/
    mhchem: false
```

最后在写文章时，在开头开启mathjax
``` markdown
---
title: 雇佣K名工人的最低成本
date: 2022-09-11 09:02:32
tags: [LeetCode,优先队列,贪心,c++]
categories: 算法
mathjax: true 
---
```
即可在文章中使用公式。

### （4）主题美化
#### 添加背景
把想设置的背景放入./themes/next/source/images中，命名为background.jpg。在根目录的source文件夹下新建文件夹_data与style文件source/_data/styles.styl，输入以下代码
``` css
body {
    background:url(/images/background.jpg);
    background-repeat: no-repeat;
    background-attachment:fixed;
    background-position:100% 100%;
}
```

再在主题_config.yml文件中找到对应的custom_file_path，去掉注释即可。
``` yaml
custom_file_path:
  #head: source/_data/head.swig
  #header: source/_data/header.swig
  #sidebar: source/_data/sidebar.swig
  #postMeta: source/_data/post-meta.swig
  #postBodyEnd: source/_data/post-body-end.swig
  #footer: source/_data/footer.swig
  #bodyEnd: source/_data/body-end.swig
  #variable: source/_data/variables.styl
  #mixin: source/_data/mixins.styl
  style: source/_data/styles.styl
```
#### 博客内容透明化
在上文新建的styles.styl文件中添加如下内容即可：
``` css
//博客内容透明化
//文章内容的透明度设置
.content-wrap {
  opacity: 0.9;
}

//侧边框的透明度设置
.sidebar {
  opacity: 0.9;
}

//菜单栏的透明度设置
.header-inner {
  background: rgba(255,255,255,0.9);
}

//搜索框（local-search）的透明度设置
.popup {
  opacity: 0.9;
}
```

## 四、结语
搭建和美化博客的工作内容颇多，本人也花了许久功夫去查阅资料、不断尝试，才最终得到了一个比较好的效果。在这一过程中，给过我帮助的文章已经难以一一回忆起来，因此在最后不能给出一份引用资料，实在抱歉。
这里就列出一些我还能找得到的参考文章：
- https://blog.csdn.net/sinat_37781304/article/details/82729029
- https://blog.csdn.net/qq_34003239/article/details/100883213
- https://zhuanlan.zhihu.com/p/265077468
- https://blog.csdn.net/yexiaohhjk/article/details/82526604
- https://www.snowmoon.top/2021/02/21/next%E4%B8%BB%E9%A2%98%E7%BE%8E%E5%8C%96/