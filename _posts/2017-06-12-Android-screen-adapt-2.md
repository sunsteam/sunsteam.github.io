---
layout:     post
title:      "android 适配相关知识(二) -- 自动生成swNdp适配方案"
subtitle:   ""
date:       2017-06-12 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - adapt
---



## Android 官方屏幕适配方式

在 android3.2 以前，所有的资源文件都有相应的 xhdpi,hdpi,mdpi,ldpi 四种文件来对应。
android3.2 以后，为了提供更精准的对布局文件的控制，可以通过为资源文件（res 目录下文件）增加后缀来指定该文件夹里的 xml 布局文件或 color.xml，string.xml 是为哪种大小的屏幕使用。

1. `sw<N>dp`, 如 `layout-sw600dp`, `values-sw600dp`
这里的 sw 代表 smallwidth 的意思，当你所有屏幕的最小宽度都大于 600dp 时, 屏幕就会自动到带 sw600dp 后缀的资源文件里去寻找相关资源文件，这里的最小宽度是指屏幕宽高的较小值，每个屏幕都是固定的，不会随着屏幕横向纵向改变而改变。

2. `w<N>dp` 如 `layout-w600dp`, `values-w600dp`
带这样后缀的资源文件的资源文件制定了屏幕宽度的大于 Ndp 的情况下使用该资源文件，但它和 sw<N>dp 不同的是，当屏幕横向纵向切换时，屏幕的宽度是变化的，以变化后的宽度来与 N 相比，看是否使用此资源文件下的资源。

3. `h<N>dp` 如 `layout-h600dp`, `values-h600dp`
这个后缀的使用方式和 w<N>dp 一样，随着屏幕横纵向的变化，屏幕高度也会变化，根据变化后的高度值来判断是否使用 h<N>dp ，但这种方式很少使用，因为屏幕在纵向上通常能够滚动导致长度变化，不像宽度那样基本固定，因为这个方法灵活性不是很好，google 官方文档建议尽量少使用这种方式。

---

这里的 N 代表了屏幕宽度或高度的 dpi 值，计算方式如下

```java

DisplayMetrics metrics = getResources().getDisplayMetrics();
int widthDpi = (int) (metrics.widthPixels / metrics.density);
int heightDpi = (int) (metrics.heightPixels / metrics.density);

```

在不使用代码获取 density 值的情况下，知道屏幕尺寸也是可以计算的。

density = screenDpi / 160;

 160dpi 是 1dp=1px 的基准屏幕

screenDpi = $\frac{\sqrt{widthPixels^{2}+heightPixels^{2}}}{a}$

a 是屏幕对角线长度，也就是俗话说的屏幕尺寸。其实就是勾股定理算了下对角线上的像素点数量，然后除以长度，得到了每长度单位上的像素点数量，就是 density。

widthDpi 就是用宽度的像素点数量除以 density，得到了在屏幕宽度上有多少个这样的长度单位（dpi），很好理解吧。

很多公司的UI设计图以IOS为准,比如以 iphone 6/7 为例，
设备宽高为 750x1334，ppi=dpi=326，那么 widthDpi = 750 / (326 / 160) = 368

## 适配原理

既然 dp 在不同屏幕上差异的原因是由于设计的 widthDpi 和实际 widthDpi 的比值差异导致的，那么在一定的差异范围内我们把 dimens.xml 中每个值乘以这个比值就能修正为设计时的 widthDpi。

注意点：

1. `sw<N>dp` 适配是以屏幕的最短边为基准适配的，不受横屏竖屏影响。但是横屏时必须要用 `layout-land` 文件夹做适配。`w<N>dp` 方案可以随着横纵屏变化同步适配， 比如竖屏状态下 widthDpi 为 360， 横屏状态下 widthDpi 为 600， 那么横屏时会认为这是一个宽度为 1280，高度为 ( 1280 / 720 ) * 1280 = 2275 的设备。控件就会变得很高，字体变大，但如果不介意这个的话就不需要做 `layout-land` 适配了，各有千秋。我选择使用的是 `sw<N>dp` 方案。

2. 假如生成了 360、400 的文件夹，但实际 widthDpi 为 380，那么它会选择 360 文件夹下的 dimens.xml。一定程度上缩小差异。另外需要注意一点，假如生成了一个比基准参数小的适配文件夹，比如参数为 `360 320 400`，那么基准参数文件夹也要生成，否则会适配到 320 去。应改为 `360 320 360 400`。

## res-SDK 适配方式

做屏幕适配的时候也可能遇到 `sdk` 和 `sw<N>dp` 冲突的问题。比如我做透明状态栏效果时使用的是在布局中添加一个 view 代替 statusbar 的位置，view 的大小根据 sdk 版本配置，sdk 19 + 支持透明状态栏后，view 高度为 22dp，19 版本以下为 0dp。因此我建了一个 `values-v19` 来做这个效果。有多重适配参数时，是有优先级的屏幕尺寸 > SDK: 假如 widthDpi 为 410，我们有一个 -sw400dp 文件夹，那么 -v19 文件夹会被忽略，没有任何作用，除非我们对 -sw400dp 再适配一次 sdk，生成一个 values-sw400dp-v19 就没问题了，顺序也不能写反。

## 自动生成适配方案

jar包的源码来自于 [PhoneScreenMatch](https://github.com/mengzhinan/PhoneScreenMatch) ，先感谢一下作者。原作者的文章在这里 [Android 屏幕适配 dp、px 两套解决办法](http://blog.csdn.net/fesdgasdgasdg/article/details/52325590)

我改了一下源码，主要有 3 点：

1. 去掉 px 适配的部分，这个限制较多，不好用

2. dp 适配方案从 `w<N>dp` 改为 `sw<N>dp`，取舍原因我在上一节说明了。

3. 增加了对 `-sdk` 参数的支持。 比如尺寸参数后面跟上 ` -a -v19 -v21` 这样的话会为 -v19 和 -v21 在分别生成一遍屏幕适配的文件夹，其他种类的参数也可以，但在适配上没啥用。


你需要做的是：

1. 默认的 values 文件夹中需要一份特定的 dimens.xml 文件。项目里直接拷贝 values 文件夹下的 dimens.xml，有缺的尺寸就在使用中慢慢补吧。

2. 把 screenMatchDP.bat 和 screenMatchDP.jar 两个文件拷贝到你项目的 main 目录下，和 res 文件夹平级。

3. 修改 screenMatchDP.bat 文件中你需要适配的屏幕 dp 参数，第一个为设计的基准 widthDpi，后面为需要适配的 widthDpi。

4. 如果有带参数的文件夹需要适配，屏幕 dp 参数后面在加上 ` -a ` 后面是需要适配的参数，举个栗子：

```
java -jar %~dp0\autoGenDp.jar 360 320 360 384 392 400 411 533 590 -a -v19
pause
```

5. 进到 screenMatchDP.bat 文件所在的硬盘位置，双击执行。执行是不要在 AS 里面双击，AS 不可执行 bat 文件。


如果 bat 文件中没有跟参数或者参数少于 3 个，会使用默认值生成，默认值是下面这样的

```java
if (args == null || args.length < 3)
        widthDps = new String[]{"360", "384", "400", "411", "533", "640", "720", "768", "820"};
```

具体效果请移步 GitHub，下载 Demo 在不同的机器上测试
生成工具已提取成 java 库，包含在项目中，可以自行修改满足需求，Terminal 中运行 `gradlew buildLib` 即可在目录下生成新的 jar 包，拷贝到 demo 中测试。


<div align="center">

<img src="http://ofaeieqjq.bkt.clouddn.com/AdaptDpToScreen/Screenshot_emulator_Nexus_4.png" width = "180" height = "320" alt="768*1280" />

<img src="http://ofaeieqjq.bkt.clouddn.com/AdaptDpToScreen/Screenshot_emulator_Nexus_5X.png" width = "180" height = "320" alt="1080*1920" style="margin-left:45px"/>

<img src="http://ofaeieqjq.bkt.clouddn.com/AdaptDpToScreen/Screenshot_emulator_Nexus_6P.png" width = "180" height = "320" alt="1440*2560" style="margin-left:45px"/>

 </div>

---




**传送门：https://github.com/sunsteam/AdaptDpToScreen**

---

只需要工具的看下面直接复制或下载

dimens.xml
https://github.com/sunsteam/AdaptDpToScreen/blob/master/app/src/main/res/values/dimens.xml

autoGenDp.bat、autoGenDp.jar
https://github.com/sunsteam/AdaptDpToScreen/tree/master/app/src/main
