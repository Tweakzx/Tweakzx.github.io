---
title: "【博客】Hugo+Typora+PicGO+Github搭建个人博客"
author: "Tweakzx"
date: 2023-02-07T10:57:40+08:00
description: 博客搭建教程
categories: 博客
tags: 
image: https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202302071614641.png
draft: false
---

# Hugo+Typora+PicGO+Github搭建个人博客

> - 我之前使用过typecho来搭建个人博客，由于是搭建在服务器上，我需要使用宝塔linux来完成各种配置，花花绿绿的各种环境让我对搭建博客多少有些累觉不爱
>
> - 不过随着我阿里云服务器的到期，我决定趁机使用github托管我的个人网站，但是我不想麻烦区配置各种后端环境， 所以选择使用hugo。
> - 与其说我精挑细选了一种博客框架，倒不如说我的目的只是想要快速搭一个博客出来，技术的选型其实并没有花太多心思。

你现在看到的博客使用到的工具链是：Hugo+Typora+PicGO+Github，它们分别的功能是

- hugo: 编译静态网站

- typora：编写博客内容

- picgo：配合github搭建图床， 与typora联动上传博客中的图片

- github：一方面网站需要使用到github提供的pages功能， 另一方面我也使用了github作为图床

下面是搭建这个博客的一些步骤， 记录下来一方面方便他人参考，另一方面如果我要重新搭建博客也方便我自己参考。

搭建前的一些环境配置

- 系统： windows
- 包管理器：scoop （非必须）
- 版本控制：git
- markdown编辑器：Typora

## github创建项目

1. 打开github新建一个库
2. 建议项目名与github用户名保持一致，比如我的是 tweakzx，那么输入的 Repository name 就是 tweakzx.github.io

## hugo创建站点

### hugo 安装

参看官方的安装教程[Installation | Hugo (gohugo.io)](https://gohugo.io/installation/)

我使用的系统是Windows的，我推荐以下两种安装方式

**Chocolatey (Windows)**

如果使用 Windows 并且使用 Chocolatey 作为包管理器，可以使用如下一行代码来安装 Hugo ：

```
choco install hugo -confirm
```

如果想要使用支持 Sass/SCSS 的扩展版Hugo的话，可以使用如下命令：

```
choco install hugo-extended -confirm
```

**Scoop (Windows)**

如果使用 Windows 并且使用 Scoop 作为包管理器，可以使用如下一行代码来安装 Hugo ：

```
scoop install hugo
```

如果想要使用支持 Sass/SCSS 的扩展版Hugo的话，可以使用如下命令：

```
scoop install hugo-extended
```

### hugo 生成站点

创建一个用于存放网站文件的文件夹， 进入到这个文件夹，使用命令创建网站， 自定义一个站点名称

```shell
hugo new site <BLOG NAME>
```

创建成功会看到官方的指导流程

```shell
Congratulations! Your new Hugo site is created in D:\software\Scoop\apps\hugo-extended\test.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/ or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>\<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
```

然后你会在该位置发现一个站点名称命名的文件夹，进入这个文件夹

### hugo 主题

我选择的是Stack主题，当然你可以去官网或者github选择你喜欢的主题[Complete List | Hugo Themes (gohugo.io)](https://themes.gohugo.io/)

```shell
cd <BLOG NAME>
git clone https://github.com/CaiJimmy/hugo-theme-stack.git ./themes/hugo-theme-stack

## 复制stack的exampleSite下的配置, 并删除原本的config.toml
cp -rf themes/hugo-theme-stack/exampleSite/config.yaml ./
rm ./config.toml
```

编辑config.yaml， 注意修改baseurl， 其他配置自定义修改吧

```yaml
baseurl:  https://tweakzx.github.io/
languageCode: zh-cn
theme: hugo-theme-stack
paginate: 5
title: Tweakzx

...

```

### 上传Github的方式

其实这里分为两种编译方式

- 在线编译：上传整个站点文件，在线编译生成html静态文件，

- 本地编译：本地编译，直接上传静态文件， 也就是public 文件夹

在线编译的方式需要在Github Action里找到Hugo 编译的 Action模板直接用就可以了

> 我选择了使用本地编译然后上传静态文件的方式， 主要原因是当时不太懂，瞎搞的， 于是顺其自然了， 要是能重来， 我要选在线编译。

本地编译上传静态文件也是有好处的

- 配置简单， 无需复杂操作
- 本地编译成功的东西部署之后可以立刻看到， 不用等github服务器慢吞吞的编译， 
- 如果本地成功编译预览没有问题，部署后看到的肯定也没有问题， 如果自己做了一些自定义的设置， 可能在GIthub端可能就不一样，甚至无法正常编译
- 而且自己的配置信息可以不用被别人看到， 如果有一些敏感信息也比较安全

> 注意，一定要上传到仓库的default分支才可以，default分支一般是main分支， 我创建的比较早所以是master。

## Github Action的配置

进入Github 仓库， 点击Settings->Code and automations->Pages

可以看到有个Static HTML的workflow， 点击配置

![image-20230207155747151](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202302071559176.png)

完成之后就可以在上方看到一个网址， 点击访问就可以看到网站部署成功。

![image-20230207155914140](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202302071559078.png)

## 撰写博客的流程

### 创建博文

我个人的博客创建时会用【】框住主题， 然后起名，这样方便我分类并查找。

```shell
hugo new ./content/post/【主题】博文题目.md
```

### 编辑博文

使用Typora编辑刚刚创建的md文档， 在顶端设置好配置信息， 例如作者， 头图等等。

```yaml
title: "【博客】Hugo+Typora+PicGO+Github搭建个人博客"
author: "Tweakzx"
date: 2022-08-23T10:57:40+08:00
description: 
categories: 
tags: 
image: 
draft: true
```

### 本地预览

本地编辑完成后， 或者完成过程中， 可以预览效果， 将```draft:true```改为```false```

然后在网站根目录运行

```shell
hugo server
```

单击```http://localhost:1313/```,  即可本地预览。

### 上传Github

 为了实现一键更新博客， 需要写一个bat脚本， 但是如果你想要使用**手动更新**的话， 这一步可以不用做

新建并编辑upload.bat 文件， 双击上传

```shell
cd <网站根目录路径>
hugo
cd public
git add .
git commit -m "updatde-blog-post"
git push origin master:master
```

### 个人技巧

我会在./content文件下另外创建两个文件夹， 一个todo， 一个doing。todo文件只有题目， 是打算写的文章； doing是写了一半的文章 

当我开始写文章时

- 如果只是打算写但是现在不想写， 我会创建到todo文件夹
- 如果马上就要开始写，我会创建到doing文件夹
- 开始写todo的文章时我会把该文章移入doing文件夹
- 如果大体上已经写完，可以发布，日后只需要一些简单的编辑的话，我会放入post文件夹

## picgo图床的搭建

我选择使用github作为图床， 方便简单， 就是不好访问。

可能之后会转向七牛云， 不过七牛云需要一个固定的域名， 如果没有的话就比较麻烦了。

### 创建github仓库

- 创建一个github仓库， 必须是public
- 在个人设置里Settings->Developer settings生成一个token，repo要打勾， 不过具体的权限与时间自行斟酌吧。
- 复制这个token

### 安装并配置Picgo

- 下载安装[PicGo is Here | PicGo](https://picgo.github.io/PicGo-Doc/zh/guide/)
- 配置picgo
  - 仓库名：刚刚创建的仓库名
  - 分支名：注意是main还是master
  - Token：刚刚生成的token
  - 存储路径， 如果有路径
  - 配置CDN： 使用 https://cdn.jsdelivr.net/gh/ 用户名/仓库名@main 即可

## 配置giscus评论区

- 在giscus官网，输入自定义配置，生成相关配置ID [giscus](https://giscus.app/zh-CN)
  - 生成repoID
  - 生成categoryID
  - 选择合适的主题

- 修改config.yaml即可

```
giscus:
	repo: "Tweakzx/Tweakzx.github.io"
	repoID: ""
	category: "Announcements"
	categoryID: ""
	mapping: "title"
	lightTheme: "light_tritanopia"
	emitMetadata: 0
```







