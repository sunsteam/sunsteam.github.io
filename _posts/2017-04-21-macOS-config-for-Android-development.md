---
layout:     post
title:      "Mac 下的 Android 开发环境配置"
subtitle:   ""
date:       2017-04-21 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - macOS
---

从 windows 转到 Mac Os 进行 Android 开发，对系统不熟，软件配置上遇到很多麻烦事，只能一步一步摸索，开一贴记录历程。


## 操作环境

MacBook Pro (15-inch, 2016)
macOs version: 10.12.4

## 环境变量配置

mac 下配置环境变量使用 `.bash_profile` 配置文件，当然还有别的途径不过为了简单只说这一种。
以配置 adb 环境变量为例。

在终端中按顺序输入以下命令

`touch ~/.bash_profile`   // 不存在则会创建该文件
`open -e ~/.bash_profile` // 文本编辑器打开文件

编辑内容，注意不要乱加空格

```
export ADB_HOME=~/Library/Android/sdk/platform-tools
export PATH=${ADB_HOME}:${PATH}
```

1. 第一行定义了一个 adb 命令的发起位置
2. 第二行是在原来的 PATH 中加上 ADB_HOME 这个位置，冒号作为链接

在终端中按顺序输入以下命令，然后输入 adb 测试环境是否生效
`source ~/.bash_profile` // 保存文件的更改，并立即生效（不加的话保存文件后重启才生效）

## 开发工具

1. 先安装 Xcode，安装完成后里面会自带一些工具比如 git
2. JDK 在终端输入 java 会自动提示安装，点击好会跳转到官网下载页
3. svn 使用 Cornerstone


### Android Studio

- 导入配置： 如果导入的是 windows 平台下的 setting 文件，注意不要导入 sdk、git、svn 之类的目录配置，或者导入后手动修改。

- KeyMap：

    - 在 windows 的自定义快捷键基础上增加了 command 键的备用组合快捷键。

    - 包含 TouchBar 的 mac pro 在使用 F1-F12 功能键时不方便，必须要加按 Fn 键的。因此在 KeyMap 中增加了一套备用快捷键。

- 命令行： macOs 下使用 gradlewrapper 的命令不是 `gradlew` 而是 `./gradlew`，如果提示 permission deny, 终端中输入 `chmod 755 gradlew` 给 gradlew 增加权限。

- GitHub clone: 注意在导入时候看一眼放置的文件夹是 windows 格式的还是 mac 格式，在导入 setting 时会把 windows 下的默认设置，如果放置的位置是 `C:\StudioProjects` 这样的无法访问的位置，点击 clone 会毫无反应，连个提示都没有。当初因为粗心，这个问题纠结了 1 个小时。

- studio 在建立项目时默认会使用. android 目录下的 debug.keystore 文件进行 debug 签名, 这是写在 build.gradle 文件中的。由于不同系统下 user 目录的位置不同，读取 gradle 就会报错。我的解决方法是直接把 debug.keystore 拷贝到和 build.gradle 同一层级下，并修改 debug 签名读取位置为

```java
debug {
    storeFile file("debug.keystore")
}
```
- 关联源码的配置文件：如果显示不出Android源码，在 `/Users/xxx/Library/Preferences/AndroidStudiox.x/options/jdk.table.xml` 文件中修改对应位置

```xml
<jdk version="2">
      <name value="Android API 26 Platform" />
      <type value="Android SDK" />
      <homePath value="$USER_HOME$/Library/Android/sdk" />
      <roots>
        <annotationsPath>
          <root type="composite">
            <root type="simple" url="jar://$APPLICATION_HOME_DIR$/plugins/android/lib/androidAnnotations.jar!/" />
          </root>
        </annotationsPath>
        <classPath>
          <root type="composite">
            <root type="simple" url="jar://$USER_HOME$/Library/Android/sdk/platforms/android-26/android.jar!/" />
            <root type="simple" url="file://$USER_HOME$/Library/Android/sdk/platforms/android-26/data/res" />
          </root>
        </classPath>
        <javadocPath>
          <root type="composite">
            <root type="simple" url="file://$USER_HOME$/Library/Android/sdk/docs/reference" />
          </root>
        </javadocPath>
        <sourcePath>
          <root type="composite">
            <root type="simple" url="file://$USER_HOME$/Library/Android/sdk/sources/android-26" />
          </root>
        </sourcePath>
      </roots>
      <additional jdk="1.8" sdk="android-26" />
    </jdk>
```

#### 配置 GitHub 账户和 SSH

1. 配置全局账户名

```
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
```

去掉后面的字符串再输入一遍可以检查定义是否生效

2. 配置 SSH

按照 [GitHub 上的教程](https://help.github.com/articles/checking-for-existing-ssh-keys/#platform-mac) 做。


### Atom

Atom 作为一个 MarkDown 编辑器使用，他的优势在于可以通过各种插件灵活配置，编写时支持大多数语言的代码块高亮，界面美观，同时也可以作为别的语言编辑器使用。

#### 插件组

```
atom-beautify
pretty-json
script
sync-settings

markdown-preview-plus // 支持 MarkDown 预览
markdown-pdf          // 支持导出 MarkDown 为 Pdf 格式
markdown-scroll-sync  // 支持编辑和预览界面同步滑动
pangu                 // 在英文 / 其他语言之间插入空格，美化排版
```


Atom 目前没有支持登陆账户进行同步插件和设置的功能，每次重新配置都是件繁琐的事情，好在有个插件提供了解决方案。

#### 使用 sync-settings 同步插件和软件设置

[sync-settings 传送门](https://github.com/atom-community/sync-settings)

简述配置方法

1. 安装 sync-settings

2. 在 GitHub 网站登录并创建一个 token, 一定要包含新建 Gist 权限
3. 在 Atom 的 Package 中找到 sync-settings 并进入 settings 页面
4. 填入刚刚新建的 token 到 Personal Access Token
5. 在 GitHub 网站的 Gist 页面 (需要翻墙访问)，新建一个 Gist，file name 使用 `packages.json`，内容随便打几个字符，description 不填，选择 create secret gist
6. 把生成 Gist 的 id，也就是用户名后面那部分字符串填入插件 settings 页面的 Gist Id 中。
7. 配置好其他插件和设置后再 Packages 菜单下找到 Synchronize settings，选择 backup
8. 在其他的 atom 客户端中重复 1-6 步，然后 Synchronize settings 中选择 restore
