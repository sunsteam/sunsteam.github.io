---
layout:     post
title:      "Android 6.0 和 7.0 储存空间适配小结"
subtitle:   ""
date:       2017-09-28 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - adapt
---

### FileProvider 配置

FileProvider所支持的几种path类型，权且记录一下，写file_paths的时候是不会自动提示的。

从Android官方文档上可以看出FileProvider提供以下几种path类型：

```xml
<files-path path="" name="camera_photos" />
```

该方式提供在应用的内部存储区的文件/子目录的文件。它对应Context.getFilesDir返回的路径：eg:"/data/data/包名/files"。

---

```xml
<cache-path name="name" path="path" />
```

该方式提供在应用的内部存储区的缓存子目录的文件。它对应getCacheDir返回的路径：eg:“/data/data/包名/cache”；

---

```xml
<external-path name="name" path="path" />
```

该方式提供在外部存储区域根目录下的文件。它对应Environment.getExternalStorageDirectory返回的路径：eg:"/storage/emulated/0";

---

```xml
<external-files-path name="name" path="path" />
```

该方式提供在应用的外部存储区根目录的下的文件。它对应Context#getExternalFilesDir(String) Context.getExternalFilesDir(null)返回的路径。eg:"/storage/emulated/0/Android/data/包名/files"。

---

```xml
<external-cache-path name="name" path="path" />
```

该方式提供在应用的外部缓存区根目录的文件。它对应Context.getExternalCacheDir()返回的路径。eg:"/storage/emulated/0/Android/data/包名/cache"。

---

### 规避申请储存空间权限

程序包名下的 file 和 cache 目录在 SDK 19 + 是不需要储存空间权限的，并且 23 + 的动态获取权限也不需要申请。那么我们就不必将所有内容都放在公共SD卡上，避免使用储存空间时要求用户动态授权的一系列麻烦。

另外我尝试了将储存空间权限限制在 19 以下，发现这样并不可以，小米 MIUI8 上直接报无权限错误了。

```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
        android:maxSdkVersion="18"/>
```


在手机上储存内容的时候常见的是以下几种情况

1. 缓存图片，临时数据  ——> 内部存储区的缓存子目录，防止被其他应用或用户访问

2. 下载的更新安装包  ——> 外部缓存区根目录，需要交给安装程序访问进行安装

3. 程序的运行日志，持久化储存的数据，插件包  ——> 内部存储区的文件子目录，防止被其他应用访问，并且不会因为系统控件不足而被清理。日志应上传后手动清理。

4. 需要与特定程序交互的持久化数据  ——> 外部文件区子目录，这个需求也是比较少的，比如为家族产品定制的数据格式文件。

5. 用户可访问的公开数据，照片，文档等  ——> 公共储存空间， SDK 23 + 必须申请授权访问。
