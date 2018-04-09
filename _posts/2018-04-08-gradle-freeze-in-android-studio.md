---
layout:     post
title:      "解决 Android Studio 编辑 build.gradle 卡顿问题"
subtitle:   ""
date:       2018-04-08 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
---


这几天 Android Studio 写 gradle 的时候卡的要疯， 正好又在弄新项目，gradle 有大量修改，查了一些资料后解决。

### 原因

Android Studio 某次更新之后，每次操作 gradle 文件会联网进行一次查询，比如 dependencies 有没有更新，依赖库名有没有写对，gradle 写的是否符合 Android 规范之类的。这个访问网址是 `search.maven.org`。

然后加上国内的渣网速，连到谷歌相关的服务器内容也许还被墙一下，就造成了每操作一下就查询个几秒，卡的要死！！！

找到原因我就想明白了！Android Studio 的开发和测试人员，用着 Google 的专用网络，访问内部资源还有内部专线加速！根本不会遇到卡顿这种事情嘛，脑袋一拍就觉得这是个好功能！给人民群众都用上！

真是坑爹！

### 解决

通过修改 hosts 文件将这个域名直接指向本机地址，会直接无延迟的返回查询失败，让你再查！

Windows 系统在 system32 文件夹里，打开 hosts 文件，如下增加一行映射。

```

##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost

127.0.0.1 search.maven.org

```

[macOS 查找 hosts 看这里](https://jingyan.baidu.com/article/0eb457e50554d603f1a90514.html)
