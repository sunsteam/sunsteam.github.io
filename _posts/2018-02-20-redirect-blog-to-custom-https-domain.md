---
layout:     post
title:      "将个人博客指向自定义域名并开启 Https"
subtitle:   ""
date:       2018-02-20 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - blog
---

> 域名这个事实际上半年前就搞好了，最初是 Godaddy 买的 me 域名，然而博客一直懒得弄。今天上去一看，域名只剩半年好用了，续费的话 5 年￥1000 -_-! 好吧，我这种穷人续费不起，再换个便宜的域名重新弄一遍，顺便记录下。

<br>

### 1. 买域名

如果你不是土豪并且不希望几年后给自己的博客买新域名，那么还是要把价格因素考虑进去。我最终选择的是阿里云买的 `.tech` 域名，10 年￥290, 先买 1 年只要￥9，免费的匿名注册保护服务（Godaddy 上就这个服务 1 年要价￥60）。注册信息匿名还是挺重要的，因为这些信息所有人都搜得到，隐私安全意识要有。

#### 1.1 国外老牌域名供应商

可以参考 [知乎帖子][1] , Godaddy 、 NameCheap 等，不过个人认为还是贵了点，界面也没有国内友好，但是没有国内那么多麻烦事。

#### 1.2 国内网站购买

如果在国内购买，价格会便宜点，但是都是代理向万网注册的，会有些行政因素要考虑，比如实名制认证和网站备案。

[实名制认证][2]针对域名购买者，[网站备案][3]是针对部署在国内服务器主机（不包括香港、台湾）上的使用域名的网站。

买完后先把实名认证搞了，否则不一会就会暂时封停。

### 2. 域名解析（DNS）服务与动态 Https

DNS 服务买完域名会免费提供，各家的稳定性不同，国内还会受劫持啊、污染啊各种因素影响，幸好我们还有 [cloudflare][4] 提供稳定的 DNS 解析和免费的加 https 标记服务。

我们需要配置

- DNS 解析
- HTTPS
- 域名访问规则

注册一个账号，我们正式开始：

#### 2.1 DNS 解析

点击面板右上角的 **Add Site** 按钮，输入域名创建一个服务。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fonzx3aafxj31kw0nydgz.jpg)

点击 `DNS` 选项卡，DNS 服务器通过读取一条 `DNS Record` 来决定如何解析域名, A 记录说明该域名指向一个 ip ， CNAME 记录说明该域名是另一个域名的别名，因此指向解析域名指向另一个域名，相当于重定向吧。如果 Blog 部署在 VPS 上并有一个固定 ip，那么使用 A 记录。部署在 GitHub 上的 blog，访问域名是 `xxx.github.io`，因此使用 CNAME。新建 2 条记录如下图，其中 `www` 其实是 `www.sunyx.tech` 的意思，就是这么个写法。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fonzx2tu51j31kw11o0ul.jpg)


然后下拉到 `Cloudflare Nameservers` 部分，将它的域名解析服务器地址替换掉域名注册服务商默认的解析服务器，完成后等最快 2 分钟，最慢 1 小时，域名解析服务就扩散完毕了。

#### 2.2 Https

点击 `Crypto` 选项卡，保持 Flexible 就可以了，它会使用 Cloudflare 自己的证书做动态 SSL，另外 2 个 Full 选项则是去你的服务器验证是否配置了证书，strict 模式要求服务器配置的证书是可信的机构颁发的，不允许自定义证书。

![](https://ws1.sinaimg.cn/large/006tNc79gy1fonzwxfsytj31kw0x4gmu.jpg)

#### 2.3 访问规则

点击 `Page Rules` 选项卡，照着图改吧，先加第二条记录再加第一条。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fonzwus1w0j31kw0ozjsi.jpg)

### 3. Blog 项目优化

#### 3.1 网页内引用资源改为https

做完第 2 部分后，应该能正常打开加 https 绿标的 Blog。但如果网页内调用的资源仍是 http 的，那么整个网页都被认为不安全，因此最后要把项目中的引用检查一下，能改则改。样式调用使用无协议的符号。

```html
<!-- Change this -->
<link rel="stylesheet" href="http://www.somesite.com/path/to/styles.css">

<!-- to this: -->
<link rel="stylesheet" href="//www.somesite.com/path/to/styles.css">

```

还有就是图片资源需要一个https图床，可以看[图床方案推荐](https://sspai.com/post/40499)，写的很详细了。

#### 3.2 博客项目内创建CNAME

在`github.io`项目的根目录下，创建一个名为`CNAME`的文件，内容是`www.sunyx.tech`，用于在浏览器打开这个项目子域名的时候，能指向自定义域名。

---

其他可以有的优化包括： SEO 优化，添加谷歌或百度网站统计，留言板等功能。

进阶的包括：使用 Liquid 语法更改布局，编写 less 更改样式等。 

这些优化因模版的不同而各异，就不细说了，自己研究啦。


[1]: https://www.zhihu.com/question/19551906

[2]: https://help.aliyun.com/knowledge_detail/41880.html?spm=a2c4g.11186623.6.640.saPU6z

[3]: https://help.aliyun.com/document_detail/47315.html?spm=a2c4g.11186623.6.698.aoGdzM&parentId=35473

[4]: https://www.cloudflare.com/a/overview