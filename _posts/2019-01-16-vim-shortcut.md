---
layout:     post
title:      "Vim 快捷键整理"
subtitle:   ""
date:       2019-01-16 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Tools
---


## 基本命令

```
Esc	从当前模式转换到“普通模式”。所有的键对应到命令。
i	“插入模式”用于插入文字。回归按键的本职工作。
:	“命令行模式” Vim 希望你输入类似于保存该文档命令的地方。
:q	退出 Vim，如果文件已被修改，将退出失败
:w	保存文件
:w new_name	用 new_name 作为文件名保存文件
:wq	保存文件并退出 Vim
:q!	退出 Vim，不保存文件改动
ZZ	退出 Vim，如果文件被改动过，保存改动内容
ZQ	与 :q! 相同，退出 Vim，不保存文件改动
```

## 移动命令

```
h	光标向左移动一个字符
j 或 Ctrl + J	光标向下移动一行
k 或 Ctrl + P	光标向上移动一行
l	光标向右移动一个字符
0	（数字 0）移动光标至本行开头
$	移动光标至本行末尾
^	移动光标至本行第一个非空字符处
w	向前移动一个词 （上一个字母和数字组成的词之后）
W	向前移动一个词 （以空格分隔的词）
5w	向前移动五个词
b	向后移动一个词 （下一个字母和数字组成的词之前）
B	向后移动一个词 （以空格分隔的词）
5b	向后移动五个词
G	移动至文件末尾
gg	移动至文件开头
```

## 跳转命令

```
(	跳转到上一句
)	跳转到下一句
{	跳转到上一段
}	跳转到下一段
[[	跳转到上一部分
]]	跳转到下一部分
[]	跳转到上一部分的末尾
][	跳转到上一部分的开头
```

## 选择命令

```
v	进入字符可视化模式   （移动一次选择一个字符）
V	进入行可视化模式
ctrl-V	进入块可视化模式
gv	选中前一次可视化模式时选择的文本
o	光标移动到选中文本的另一结尾
O	光标移动到选中文本的另一角落



d   ------ 剪切操作
y   -------复制操作
p   -------粘贴操作 
^  --------选中当前行，光标位置到行首（或者使用键盘的HOME键）
$  --------选中当前行，光标位置到行尾（或者使用键盘的END键）
```


### 在块模式下，可以进行多列的同时修改，修改方法是：

```
首先进入块模式 Ctrl+ v
使用按键j/k/h/l进行选中多列
按键Shift + i 进行 块模式下的插入
输入字符之后，按键ESC，完成多行的插入
```


## 插入命令

```
a	在光标后插入文本
A	在行末插入文本
i	在光标前插入文本
o	（小写字母 o）在光标下方新开一行
O	（大写字母 O）在光标上方新开一行

:r [filename]	在光标下方插入文件 [filename] 的内容
:r ![command]	执行命令 [command] ，并将输出插入至光标下方


r{text}	将光标处的字符替换成 {text}
R	进入覆写模式，输入的字符将替换原有的字符
~	切换大小写
```

## 删除命令

```
x	删除光标处字符
dw	删除一个词
d0	删至行首
d$	删至行末
d)	删至句末
dgg	删至文件开头
dG	删至文件末尾
dd	删除该行
3dd	删除三行
```

## 撤销命令

```
u	撤销最后的操作
Ctrl+r	重做最后撤销的操作
```

## 搜索命令

```
/search_text	检索文档，在文档后面的部分搜索 search_text
?search_text	检索文档，在文档前面的部分搜索 search_text
n	移动到后一个检索结果
N	移动到前一个检索结果
:%s/original/replacement	检索第一个 “original” 字符串并将其替换成 “replacement”
:%s/original/replacement/g	检索并将所有的 “original” 替换为 “replacement”
:%s/original/replacement/gc
```