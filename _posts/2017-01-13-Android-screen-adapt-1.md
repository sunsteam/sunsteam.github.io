---
layout:     post
title:      "android 适配相关知识 (一) -- density dpi px dp dip  sp 解释"
subtitle:   ""
date:       2017-06-12 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - adapt
---


## 简介
整理一下之前看到的几篇不错的 android 适配相关内容，糅合在一起内容很多看起来不舒服，所以分为三个部分，加上一些自己的验证和理解。

（1） 系统名词解释， 基础知识
（2） 图片加载相关内容
（3） android 分辨率适配方案

## 长度和大小单位

 - px(pixel)

表示屏幕实际的像素。例如，1200×1920 的屏幕在横向有 1200 个像素，在纵向有 1920 个像素。

- dp = dip

也叫 dip(density independent pixel) 直译为密度无关的像素。我们猜测如果使用了这个单位，我们在不同屏幕密度的设备上显示的长度就会是相同的。

dip 是旧的适配单位，现在已被 dp 取代。

- sp

sp，即 scale-independent pixels，与 dp 类似，但是可以在设置里面调节字号的时候，文字会随之改变。当安卓系统字号设为 “普通” 时，sp 与 px 的尺寸换算和 dp 与 px 是一样的。

- 屏幕尺寸 （inch）

设备的物理屏幕尺寸，指 **屏幕的对角线的长度** ，单位是英寸，1 inch = 2.54 cm。比如某某手机为 “5 寸大屏手机”，就是指对角线的尺寸，5 寸 ×2.54 厘米 / 寸 = 12.7 厘米。


## 精细度单位

- 屏幕分辨率

也叫显示分辨率，是屏幕图像的精密度，是指屏幕或者显示器所能显示的像素点有多少。一般以横向像素 × 纵向像素表示分辨率，如 1200×1920 表示此屏幕在宽度方向有 1200 个像素，在高度方向有 1920 个像素。

- 屏幕密度 dpi (dot per inch)

dpi 是指（对角线）每英寸上的像素点数。可以称作 **屏幕密度**。

顾名思义，数值越高当然显示越细腻。屏幕密度与屏幕尺寸和屏幕分辨率有关。例如在屏幕尺寸一定的条件下，屏幕分辨率越高屏幕密度越大，反之越小。同理在屏幕分辨率一定的条件下，屏幕尺寸越小屏幕密度越大，反之越小。

其实在安卓中，将屏幕密度为 160dpi 的中密度设备屏幕作为基准屏幕，在这个屏幕中，1dp=1px。其他屏幕密度的设备按照比例换算，具体如下表：

![](https://ws3.sinaimg.cn/large/006tNc79gy1foo0k31t0ij30kn05k0sw.jpg)

由上表不难计算 1dp 在 hdpi 设备下等于 1.5px，同样的在 xxhdpi 设备下 1dp=3px。这里我们从 dp 到 px 解释了 Android 中不同屏幕密度之间的像素比例关系。

下面换一个角度，从 px 到 dp 的变化来说明这种比例关系。
就拿为 App 设计 icon 来说，为了让 App 的 icon 在不同的屏幕密度设备显示相同（这里我们选择在以 mipmap 开头的目录中都要设计一个 icon），就必须要 icon 在屏幕中占据相同的 dp。那么对于不同的屏幕密度（MDPI、HDPI、XHDPI、XXHDPI 和 XXXHDPI）应按照 2:3:4:6:8 的比例进行缩放。比如说一个 icon 的尺寸为 48x48dp，这表示在 MDPI 的屏幕上其实际尺寸应为 48x48px，在 HDPI 的屏幕上其实际大小是 MDPI 的 1.5 倍 (72x72 px)，在 XDPI 的屏幕上其实际大小是 MDPI 的 2 倍 (96x96 px)，依此类推。

- dpi 的计算

不同于屏幕尺寸和屏幕分辨率，这两个值是可以直接得到的。dpi 需要我们计算得到。例如我的手机的分辨率是 1200×1920，屏幕尺寸是 5 寸的。根据屏幕尺寸、屏幕分辨率和屏幕密度定义不难看出他们之间的关系如下图：

![](https://ws2.sinaimg.cn/large/006tNc79gy1foo0k3h2bpj309x0evq2s.jpg)

这样根据勾股定理，我们得出对角线的像素数大约是 2264，那么用 2264 除以 7(屏幕尺寸) 就是此屏幕的 dpi 了，计算结果是 323。

- 逻辑密度 (density)

前文已经说了，android 将 160dpi 的中密度设备屏幕作为基准屏幕。这是由于 android 系统开发时的手机制造技术所决定的。而 density 就是将 160 看做 1 所得来的转换单位，称作 **逻辑密度**，用于在计算比例的时候更方便。

## 实际应用

**android 系统中，有关密度适配的要素被保存在 android.util.DisplayMetrics 类，阅读源码可以对以上概念有更深理解。**

下面我们继承 Application 类，这个类抽象了 app 的运行环境，会在 app 启动的时候最先初始化，生命周期与 app 相同。可以保存一些全局信息而不用担心内存泄露（但也不要都堆在里面）。继承之后需要在 manifest 文件中申明才能拿到上下文环境。

```java
public class BaseApp extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        DisplayMetrics sMetrics = getResources().getDisplayMetrics();

        LogUtils.w("逻辑密度:" + sMetrics.density);
        LogUtils.w("屏幕密度:" + sMetrics.densityDpi);
        LogUtils.w("屏幕宽度:" + sMetrics.widthPixels);
        LogUtils.w("屏幕高度:" + sMetrics.heightPixels);
        LogUtils.w("缩放密度:" + sMetrics.scaledDensity);
    }
}
```

```xml
<application
        android:name=".BaseApp"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">


```

![](https://ws2.sinaimg.cn/large/006tNc79gy1foo0k4j90wj30hb04xgm0.jpg)

因为我是使用的正式项目打印的信息，所以定位显示的行数不一样， 不过这不是重点。
缩放密度代表的是用户改变系统设置 - 显示 - 字体大小后，系统调整 sp 属性的比例。

打印的测试机是华为 P9，分辨率 1080 x 1920，可以看到其他参数和预计的一样。 然而高度打印出来是 1812，这是由于 P9 的屏幕底部有虚拟按键区域， 也就是 NAVIGATION 部分占据了一定高度，因此显示区域变小了。我们可以通过另外的方法验证。

```java
        Point outSize = new Point();
        getWindowManager().getDefaultDisplay().getSize(outSize);
        LogUtils.w("outsize_宽度" + outSize.x);
        LogUtils.w("outsize_高度" + outSize.y);
        if (Build.VERSION.SDK_INT>= 17){
            DisplayMetrics outMetrics = new DisplayMetrics();
            getWindowManager().getDefaultDisplay().getRealMetrics(outMetrics);
            LogUtils.w("outMetrics_宽度" + outMetrics.widthPixels);
            LogUtils.w("outMetrics_高度" + outMetrics.heightPixels);
        }
```

在 MainActivity 的 onCreate 方法中加入上面这段代码，第一个打印的就是普通的取屏幕区域大小，getRealMetrics 表示获取真实可显示区域大小，（不过人为改变显示参数时可能小于物理屏幕大小，具体请阅读源码注释）。

![](https://ws1.sinaimg.cn/large/006tNc79gy1foo0k5ghkrj30k103874i.jpg)

在全局环境获取到 DisplayMetrics 即可。
