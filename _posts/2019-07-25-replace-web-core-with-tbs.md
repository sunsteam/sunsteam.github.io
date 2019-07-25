---
layout:     post
title:      "旧项目替换 WebView 内核解决兼容性问题"
subtitle:   ""
date:       2019-07-25 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - WebView
---


### 背景

公司的某个硬件设备项目使用的是 Android 4.4 的系统, WebView 内核 chrome 33.0, 其中有个使用 WebView 浏览外部网页的模块, 随着网站的更新, 出现了不兼容的语法导致无法播放网页中的视频，因此考虑用引入新内核的方式做个修复。

## 方案比较

比较完善的浏览器内核方案有 2 种，[Crosswalk](https://github.com/crosswalk-project) 和 [TBS 腾讯浏览服务](https://x5.tencent.com/tbs/guide.html)

|              | Crosswalk |  TBS  |
| :----------: | :-------: | :---: |
| chrome 版本  |    54     |  66   |
| 最近更新时间 |   17.02   | 19.05 |
| 增加内容大小 | 最少 25mb | 250kb |
| 完整植入内核 |     Y     |   N   |
|   更新内核   |     N     |   Y   |
|   集成难度   |    低     |  中   |


普通手机项目的话 TBS 好点，因为它只增加 250k 的 jar 包和一个 8kb 的 so，内核是可以和手机里的微信或 QQ 共享的，如果没有安装过的话会自动从网上下载 23M 左右的安装包，但这个过程需要一定的时间，因此更新版本后马上打开 WebView 不能立即生效，另外的它会收集一些手机数据，需要 READ_PHONE_STATE 权限。

Crosswalk 在 17 年头就停止维护了，不过因为公司的项目是智能硬件，服务器推送更新，不在乎安装包大小，可以完整打包立即生效是个优势。集成过程很简单，但最终有兼容性问题，不能用，以后估计也用不上了，不记了。

## TBS 集成重点记录

TBS API 和 WebView 也一样，不过官方接入文档有些内容不是很完整，重点记录如下

1. jar 包放到 libs 之后，其实还需要放 so，需要从官方 sample 的项目中拷贝出来，目录要放对
2. 项目根目录的 `build.gradle` 要添加 `flatDir { dirs 'libs' }` 到 `repositories`
3. so 只保留 `armeabi` 和 `armeabi-v7a`, 使用 NDK16 + 编译 `armeabi` 会失败, 在 gradle.properties 文件中添加 `android.useDeprecatedNdk=true`
4. WebView 相关替换完成后，使用官方的 `checkqbsdk.sh` 工具做检查是否遗漏
5. `READ_PHONE_STATE` 和 `WRITE_EXTERNAL_STORAGE` 是初始化需要用到的权限，普通手机需要动态申请后初始化
6. SDK23 + 需要手动添加 Apache 的 http 库，否则网页放视频会有问题
7. 添加一个 Service，异步进行 `dex2oat` 操作
8. manifest 中的 Application 标签下，确保硬件加速开启，添加 `android:hardwareAccelerated="true"`
9. 显示 WebView 的 Activity 添加属性 `android:configChanges="orientation|screenSize|keyboardHidden"`
10. 混淆保护内容比较多，下载 [官方混淆文档][proguard]，对比一下加入
11. 初始化内核下载过程增加进度回调
12. WebView 中通过 UserAgent 检查内核版本，同时可以访问 `http://soft.imtt.qq.com/browser/tes/feedback.html` 确认内核加载结果，大于 0 表示 X5 内核加载成功
13. 使用 X5 内核后，Android 4.4 版本播放网页可以进行最大化，还原操作，但不会回调 `onShowCustomView`，`onHideCustomView` 这 2 个回调方法了，没什么大影响
14. 视频播放布局有改变，植入了分享到微信或 QQ

封装如下初始化代码

```java
    private void initTbs() {
        
        QbSdk.setDownloadWithoutWifi(true);
        QbSdk.setTbsListener(new TbsListener() {
            @Override
            public void onDownloadFinish(int i) {
                Log.i("X5", "内核下载完成 :" + i);
            }

            @Override
            public void onInstallFinish(int i) {
                Log.i("X5", "内核安装完成 :" + i);
            }

            @Override
            public void onDownloadProgress(int i) {
                Log.d("X5", "内核下载进度 :" + i);
            }
        });
        // 搜集本地 tbs 内核信息并上报服务器，服务器返回结果决定使用哪个内核。
        QbSdk.initX5Environment(getApplicationContext(), new QbSdk.PreInitCallback() {
            @Override
            public void onCoreInitFinished() {
                Log.i("X5", "内核初始化完毕");
            }

            @Override
            public void onViewInitFinished(boolean b) {
                //x5 內核初始化完成的回调，
                // true 表示 x5 内核加载成功，
                // false 表示 x5 内核加载失败，会自动切换到系统内核。

                Log.i("X5", "内核加载结果:" + b);
            }
        });
    }

```

manifest 的 application 下添加如下内容
```xml
    <uses-library
        android:name="org.apache.http.legacy"
        android:required="false" />

    <service
        android:name="com.tencent.smtt.export.external.DexClassLoaderProviderService"
        android:label="dexopt"
        android:process=":dexopt" />
```

[proguard]: http://res.imtt.qq.com/TES/proguard.zip