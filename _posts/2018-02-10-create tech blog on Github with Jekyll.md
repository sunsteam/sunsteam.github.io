---
layout:     post
title:      "在 GitHub 上使用 Jekyll 建立个人博客"
subtitle:   ""
date:       2018-02-10 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - blog
    - github
---


>Jekyll 是一个简单的博客形态的静态站点生产机器。它有一个模版目录，其中包含原始文本格式的文档，通过一个转换器（如 Markdown）和我们的 Liquid 渲染器转化成一个完整的可发布的静态网站，你可以发布在任何你喜爱的服务器上。Jekyll 也可以运行在 GitHub Page 上，也就是说，你可以使用 GitHub 的服务来搭建你的项目页面、博客或者网站，而且是完全免费的。

和他类似的工具还有 Hexo 和 WordPress 等，都可以通过 Theme 自定义。

### 1. Jekyll 工作流程简述

1. Jekyll 运行在 Ruby 平台上，使用 gem 包管理工具安装。
<br>
2. Bundler 是一个根据配置文件自动查找和获取 Gem 包的工具，可以理解为 Grandle 之类的工具。
<br>
3. 由于部署在 GitHub 上，因此我们可以 Bundler 安装 GitHub 提供的整合包，名为 `github-pages`, 里面已经包含了 Jekyll 程序。
<br>
4. 运行 Jekyll 在本地预览并修改，满意后将生成的内容上传到一个名为 `yourAccoutID.github.io` 的 repository， 比如我的 `sunsteam.github.io`, Github 解析名字会知道这是托管项目，因此将其运行并生成静态网站。
<br>
5. 如果你需要用个人域名显示网站，那么买一个域名并使用 `CNAME` 文件将你的域名指向到 `yourAccoutID.github.io` 这个项目域名上。

---

### 2. 实际操作流程

#### 2.1 建立 `xxx.github.io` 仓库并 clone 到本地

也可以先建立本地仓库再关联远程，在 `.gitignore` 中添加 `_site`, 这个文件夹是本地预览生成的，如果上传会导致网站不自动编译。

#### 2.2 安装 Jekyll

- ##### 2.2.1 安装 ruby、bundler

先安装 bundler

    $ gem install bundler

在 macOS 下使用 gem 命令可能遇到权限问题，由于系统中预装的 ruby 是系统御用的，比较安全的方法是使用 [`homebrew`][1] 工具安装一个用户层面的 Ruby平台。

    $ brew install ruby

假如 brew 命令访问网络太慢失败，那么你需要给 git 一个翻墙代理，因为它是使用git 访问 GitHub 的仓库的，代理自己找，这里给出如何设置和取消

```
// 列出 git 全局参数
git config --global -l
// 设置 git 全局代理
git config --global http.proxy socks5://127.0.0.1:1086
git config --global https.proxy socks5://127.0.0.1:1086
// 取消 git 代理
git config --global --unset http.proxy
git config --global --unset https.proxy
```

如果遇到无法访问 gem 源的问题，可以选择以下一种解决方式

```
// 更换 Gem 源为国内镜像
$ gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
$ gem sources -l
```
如果你使用 Gemfile 和 Bundle (例如：Rails 项目)

```
// 你可以用 Bundler 的 Gem 源代码镜像命令。 这样你不用改你的 Gemfile 的 source。
$ bundle config mirror.https://rubygems.org https://gems.ruby-china.org
```


- ##### 2.2.2 在本地仓库文件夹，新建bundler 的配置文件，名为 `Gemfile`, 内容为

```
source 'https://rubygems.org'
gem 'github-pages', group: :jekyll_plugins
```

命令行进入仓库文件夹，运行下面命令安装 Jekyll

    $ bundle install


- ##### 2.2.3 导入 Jekyll 主题

一般我们都是看好了一个模版觉得喜欢才开始动手建的吧，只要把他的主题 clone 下来复制到本地仓库就可以了，完成之后使用本地预览命令

    $ bundle exec jekyll serve

如果没有找好主题，只是想试运行一下，那么可以用下面这个命令 new 一个

    $ bundle exec jekyll new .

命令默认的会有 `watch` 参数，也就是说监听本地文件变更保存后自动更新，不需要重新启动服务。然后就可以通过浏览器打开 `127.0.0.1:4000` 访问博客进行预览。

- ##### 2.2.3 在线测试

commit并push到远程仓库后，访问


#### 参考资料

1. [Mac OS X 下使用 Ruby Gem 的两个坑](https://blog.argcv.com/articles/4429.c)
2. [Setting up your GitHub Pages site locally with Jekyll](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)


[1]: https://brew.sh/index_zh-tw.html