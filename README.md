# Trans

**打造Service Mesh领域最全的中文资料库。**

本仓库接受英文翻译与中文原创文章。

通过[提交Issue](https://github.com/servicemesher/trans/issues/new)的方式提交最新资料地址，翻译完成的文档归档到本仓库中。文档翻译完成后会显示在[ServiceMesher社区门户网站](http://www.servicemesher.com)和微信公众号中。

![](https://ws2.sinaimg.cn/large/006tNc79ly1fsyunj8ujoj309k09k748.jpg)

## 如何参与

参与该项目包括两种方式：

1. 通过[提交Issue](https://github.com/servicemesher/trans/issues/new)的方式提交文章线索
2. 通过PR参与文章翻译或提交原创文章
3. 原则上所有认领的文章要在5个工作日内完成翻译或创作

## 发布

所有通过本仓库提交的译文和原创文章将署名译者或作者姓名发布到[ServiceMesher社区](http://www.servicemesher.com)和ServiceMesher微信公众号，原作者保留文章修改的权利。

### 提交 Issue

[提交Issue](https://github.com/servicemesher/trans/issues/new)，不一定需要翻译该文档，也可以是提交一个文章线索，在Issue模板中填写：

- 原文链接：原始出处URL
- 原文发布时间
- 译文链接（如果已翻译完成可以直接填写URL，若未翻译，可以提交PR创建新的文件）
- 作者：原文作者信息
- 译者：译者信息
- 类型：见Issue模板，选择文章对应的类型

**注意**：Issue标题请使用英文原文的标题，将所有字母小写

### 提交PR

[提交PR](https://github.com/servicemesher/trans/pulls) 就需要提交对应Issue的译文文档或者原创文章，使用Markdown格式，并在文章头部添加元信息，格式如下：

```yaml
---
original: 原文链接或者原创作者的GitHub账号
author: 作者姓名
translator: 译者的GitHub账号
original: 原文地址
reviewer: ["审阅者A的GitHub账","审阅者B的GitHub账号"]
title: "标题"
description: "文章摘要"
categories: "译文或原创"
tags: ["taga","tagb","tagc"]
originalPublishDate: 2018-09-28
publishDate: 2019-02-28
---
```

注：若提交原创文章可以直接在 `original` 中添加作者的GitHub ID。

- originalPublishDate：所翻译的文章原文的发布日期
- publishDate：原创文章或译文的PR 合并日期

对于译文，建议译者在文章首段加上”编者按”一段，该段落主要内容是译者对原文的理解和解读，以帮助读者领会文章中的精神。

## 周报

[weekly](weekly)目录保存Service Mesh周报。

## 文档要求

文档格式为Markdown，尽量保留原文格式，可以根据中文阅读习惯适度调整，**中英文之间、中文和数字之间不用加空格**。

**命名规则**

文件使用英文命名，单词之间使用 `-` 连接，所有字母均为小写，例如 `the-path-to-service-mesh.md`。

**目录结构**

新的文档根据提交 PR 的时间在对应的目录中创建。如2018年5月30日提交的PR需要在 `201805` 目录中创建新的文档，即所有文档是按照月来归档的。

**关于图片**

图片使用微博图床，需要有新浪微博账户即可使用，它有以下特点：

- 上传的图片不会出现在你的个人相册中
- 可以选择上传原图和缩略图
- 支持HTTPS
- 本身是开源的

可以选择安装本地应用，也可以使用Chrome插件来上传。

## 推荐工具

### 编辑器

本仓库中的所有文章都是用Markdown格式编写，虽然Markdown是纯文本格式，可以使用任何编辑器编辑，设置是Vi也可以，但是工欲善其事，必先利其器，下面给大家推荐几款好用的Markdown编辑器。

**Typora**⭑⭑⭑⭑⭑

![](https://ws4.sinaimg.cn/large/006tNc79ly1fsyuiqktybj316c13waia.jpg)

[Typora](https://typora.io)是一款十分简洁的所见即所得的Markdown编辑器，支持多种操作系统，该软件虽然小巧但十分强大，足以满足您的所有需求。

**Visual Studio Code**⭑⭑⭑⭑

如果您习惯使用VS Code的话可以选择在其中安装Markdown插件。

### 图床工具

我们推荐以下新浪微博图床，该图床免费，没有存储大小限制，甚至不需要登陆微博账号。可以上传原图（所有图片会被转换成JPG格式）、GIF动图。

下面给大家推荐几款支持新浪微博图床的软件。

**iPic**⭑⭑⭑⭑⭑

![ipic](https://farm8.staticflickr.com/7322/28018346695_f1461c7a09_o.jpg)

Mac用户可以直接到App Store中搜索**iPic**就可以免费下载和使用了，与Typora结合使用更佳。安装了iPic后可以在typora中直接上传Markdown中引用的本地图片的新浪微博图床。

**PicGo**⭑⭑⭑⭑

[下载PicGo](https://github.com/Molunerfinn/PicGo/releases)。PicGo是一款开源的微博图床工具，使用起来十分简便。可以通过设置微博的用户名密码方式登录，也可以通过设置 Cookie 来登录。

如果选择使用 Cookie 来认证，需要打开 https://weibo.com/minipublish 页面，在 Chrome 中打开调试模式，然后选择【网络】标签页，刷新页面，看到有对 minipublish 的请求，在 Headers 里找到 Cookie，复制它的值填写到 PicGo 的微博设置中即可。

因为需要设置Cookie而且Cookie每隔一段时间就会过期，需要重新设置，所以PicGo用起来比较麻烦。

**Chrome 插件**⭑⭑⭑

在 Chrome store 中搜索 “微博图床” 就可以安装了，在 Chrome 浏览器中登陆新浪微博后就可以使用该插件上传图片了。

---

© [ServiceMesher](http:/www.servicemesher.com)