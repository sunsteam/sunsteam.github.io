---
layout:     post
title:      "Atom 编辑器"
subtitle:   ""
date:       2016-01-14 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
---


## 一、概述

Atom 是 Github 专为 hacker 推出的开源的文本编辑器，支持 linux、window 等多平台，界面简洁优雅，而且支持 markdown 语法，Atom 提供非常方便进行主题二次开发，插件扩展化等非常适合定制，并且可以直接方便得修改所有相关的 config 配置，可以打造自己独有的编辑器。

![index](https://ws4.sinaimg.cn/large/006tNc79gy1foo0jwscraj30nr0atglx.jpg)

关于 atom 很多操作都在 Settings 菜单里，进入方式： File -> Settings -> Settings，总共有 7 个模块：

    Settings：全局设置
    Keybindings：快捷键配置
    Packages： 插件管理中心, 例如设置 / 禁用 / 删除插件
    Themes：主题管理中心
    Updates：查询社区包 / 插件的状态
    Install：搜索插件和主题
    Open Config Folder：配置文件集合目录，非常具有程序员风格直接修改 config

## 二、Atom

<br>

### 2.1 常用快捷键
- 文件切换

ctrl-shift-s 保存所有打开的文件
cmd-shift-o 打开目录
cmd-\ 显示或隐藏目录树
ctrl-0 焦点移到目录树
目录树下，使用 a，m，delete 来增加，修改和删除
cmd-t 或 cmd-p 查找文件
cmd-b 在打开的文件之间切换
cmd-shift-b 只搜索从上次 git commit 后修改或者新增的文件

- 导航

（等价于上下左右）
ctrl-p 前一行
ctrl-n 后一行
ctrl-f 前一个字符
ctrl-b 后一个字符

alt-B, alt-left 移动到单词开始
alt-F, alt-right 移动到单词末尾

cmd-right, ctrl-E 移动到一行结束
cmd-left, ctrl-A 移动到一行开始

cmd-up 移动到文件开始
cmd-down 移动到文件结束

ctrl-g 移动到指定行 row:column 处

cmd-r 在方法之间跳转

- 目录树操作

`cmd-\` 或者 `cmd-k` `cmd-b` 显示 (隐藏) 目录树
ctrl-0 焦点切换到目录树 (再按一次或者 Esc 退出目录树)
a 添加文件
d 将当前文件另存为 (duplicate)
i 显示 (隐藏) 版本控制忽略的文件
alt-right 和 alt-left 展开 (隐藏) 所有目录
ctrl-al-] 和 ctrl-al-[ 同上
ctrl-[和 ctrl-] 展开 (隐藏) 当前目录
ctrl-f 和 ctrl-b 同上
cmd-k h 或者 cmd-k left 在左半视图中打开文件
cmd-k j 或者 cmd-k down 在下半视图中打开文件
cmd-k k 或者 cmd-k up 在上半视图中打开文件
cmd-k l 或者 cmd-k right 在右半视图中打开文件
ctrl-shift-C 复制当前文件绝对路径

- 书签

cmd-F2 在本行增加书签
F2 跳到当前文件的下一条书签
shift-F2 跳到当前文件的上一条书签
ctrl-F2 列出当前工程所有书签

- 选取

大部分和导航一致，只不过加上 shift

ctrl-shift-P 选取至上一行
ctrl-shift-N 选取至下一样
ctrl-shift-B 选取至前一个字符
ctrl-shift-F 选取至后一个字符
alt-shift-B, alt-shift-left 选取至字符开始
alt-shift-F, alt-shift-right 选取至字符结束
ctrl-shift-E, cmd-shift-right 选取至本行结束
ctrl-shift-A, cmd-shift-left 选取至本行开始
cmd-shift-up 选取至文件开始
cmd-shift-down 选取至文件结尾
cmd-A 全选
cmd-L 选取一行，继续按回选取下一行
ctrl-shift-W 选取当前单词

#### 编辑和删除文本

- 基本操作

ctrl-T 使光标前后字符交换
cmd-J 将下一行与当前行合并
ctrl-cmd-up, ctrl-cmd-down 使当前行向上或者向下移动
cmd-shift-D 复制当前行到下一行
cmd-K, cmd-U 使当前字符大写
cmd-K, cmd-L 使当前字符小写

- 删除和剪切

ctrl-shift-K 删除当前行
cmd-backspace 删除到当前行开始
cmd-fn-backspace 删除到当前行结束
ctrl-K 剪切到当前行结束
alt-backspace 或 alt-H 删除到当前单词开始
alt-delete 或 alt-D 删除到当前单词结束

- 多光标和多处选取

cmd-click 增加新光标
cmd-shift-L 将多行选取改为多行光标
ctrl-shift-up, ctrl-shift-down 增加上（下）一行光标
cmd-D 选取文档中和当前单词相同的下一处
ctrl-cmd-G 选取文档中所有和当前光标单词相同的位置

- 括号跳转

ctrl-m 相应括号之间，html tag 之间等跳转
ctrl-cmd-m 括号 (tag) 之间文本选取
alt-cmd-. 关闭当前 XML/HTML tag

- 编码方式

ctrl-shift-U 调出切换编码选项
- 查找和替换

cmd-F 在 buffer 中查找
cmd-shift-f 在整个工程中查找
- 代码片段

alt-shift-S 查看当前可用代码片段

    在~/.atom 目录下 snippets.cson 文件中存放了你定制的 snippets

- 定制说明

自动补全

ctrl-space 提示补全信息
折叠

alt-cmd-[ 折叠
alt-cmd-] 展开
alt-cmd-shift-{ 折叠全部
alt-cmd-shift-} 展开全部
cmd-k cmd-N 指定折叠层级 N 为层级数

文件语法高亮

ctrl-shift-L 选择文本类型
使用 Atom 进行写作

ctrl-shift-M Markdown 预览

可用代码片段

    b, legal, img, l, i, code, t, table

git 操作

cmd-alt-Z checkout HEAD 版本
cmd-shift-B 弹出 untracked 和 modified 文件列表
alt-g down alt-g up 在修改处跳转
alt-G D 弹出 diff 列表
alt-G O 在 github 上打开文件
alt-G G 在 github 上打开项目地址
alt-G B 在 github 上打开文件 blame
alt-G H 在 github 上打开文件 history
alt-G I 在 github 上打开 issues
alt-G R 在 github 打开分支比较
alt-G C 拷贝当前文件在 gihub 上的网址


### 2.2 神技

Ctrl + Shift + P，这是所有快捷键中我最为喜欢的神技。“心中无剑，万物皆为剑”，即便是忘记了前面所有的快捷键，只要记得一招也足以应付自如。

例如全屏操作，则打开 Ctrl + Shift + P，再输入 Full Screen，则会弹出：

full_screen

上图可知全屏操作快捷键为 F11，或者点击即可全屏，该神技需知道操作英文名，这也是为何前一节每个操作都写上英文名的意义所在，快捷键可能不好记忆，但是英文名就相对更好记忆。另外，这个搜索并不需要输入全名，输入部分单词即可查找到所需要的功能。

再如开启或关闭拼写检查功能：

spell_check

通过 Ctrl + Shift + P，再输入 spell，就能出现 spell check 功能，点击即可完成。

总之，即便忘记功能，可以通过该技能尝试搜索命令，非常方便。

### 2.3 其他

插件：

    markdown-scroll-sync：markdown 同步滚动预览功能
    Minimap：编辑器内部的代码图

设置：

    主题：Settings -> Themes -> UI Theme/Syntax Theme
    显示 tabs 或空格：Settings -> Settings -> Show Invisibles
    tab 转换为空格来显示： Settings -> Soft Tabs (勾选)

相关资源：

    https://atom.io/
    https://atom-china.org/
    http://wiki.jikexueyuan.com/project/atom/
    http://blog.csdn.net/column/details/atom.html
