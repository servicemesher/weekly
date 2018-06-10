# Trans

Translations, trans-thinkings.

Service Mesh 最新资料汇总，通过 [提交 Issue](https://github.com/servicemesher/trans/issues/new) 的方式提交最新资料地址，翻译完成的文档归档到本仓库中。文档翻译完成后会显示在 ServiceMesh 社区的门户网站和微信公众号中。

## 如何参与

大家可以通过提交 Issue 和提交 PR 的方式参与。

### 提交 Issue

[提交 Issue](https://github.com/servicemesher/trans/issues/new)，不一定需要翻译该文档，只是提出一个好的文章来源，在 Issue 模板中填写：

- 原文链接：原始出处 URL
- 译文链接（如果已翻译完成可以直接填写 URL，若未翻译，可以提交 PR 创建新的文件）
- 作者：原文作者信息
- 译者：译者信息
- 类型：见 Issue 模板，选择文章对应的类型

**注意**：Issue 标题请使用英文原文的标题

### 提交 PR

[提交 PR](https://github.com/servicemesher/trans/pulls) 就需要提交对应 Issue 的译文文档，使用 Markdown 格式。

### 文档要求

文档格式为 Markdown，尽量保留原文格式，可以根据中文阅读习惯适度调整，中英文之间、中文和数字之间请增加一个空格。

**文件名**

文件使用英文命名，单词之间使用 `-` 连接，所有字母均为小写，例如 `the-path-to-service-mesh.md`。

**组织结构**

新的文档根据提交 PR 的时间在对应的目录中创建。如 2018 年 5 月 30 日提交的 PR 需要在 `201805` 目录中创建新的文档，即所有文档是按照月来归档的。

**关于图片**

图片使用微博图床，需要有新浪微博账户即可使用，它有以下特点：

- 上传的图片不会出现在你的个人相册中
- 可以选择上传原图和缩略图
- 支持 https
- 本身是开源的

可以选择安装本地应用，也可以使用 chrome 插件来上传。

**PicGo**

[下载 PicGo](https://github.com/Molunerfinn/PicGo/releases)。PicGo 是一款开源的微博图床工具，使用起来十分简便。可以通过设置微博的用户名密码方式登录，也可以通过设置 Cookie 来登录。

如果选择使用 Cookie 来认证，需要打开 https://weibo.com/minipublish 页面，在 Chrome 中打开调试模式，然后选择【网络】标签页，刷新页面，看到有对 minipublish 的请求，在 Headers 里找到 Cookie，复制它的值填写到 PicGo 的微博设置中即可。

**Chrome 插件**

在 Chrome store 中搜索 “微博图床” 就可以安装了，在 Chrome 浏览器中登陆新浪微博后就可以使用该插件上传图片了。

---

ServiceMesher - Serivce Mesh 爱好者社群