---
layout:     post
title:      "Android App 结束运行后重启"
subtitle:   ""
date:       2019-05-24 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
---


## 退出方法

1. 杀死当前进程 `android.os.Process.killProcess(android.os.Process.myPid()) ;`
2. 退出虚拟机 `System.exit(0);`

效果相同，杀死当前进程，对私有的子进程无效。 比如 `com.example.foo` 会被杀死， `com.example.foo:remote` 不受影响

## 自动重启

Android 在任务栈仍有内容，但进程被杀死时，会认为这是异常退出，尝试重启应用。具体操作方法是

1. 退出当前 Activity，如果已经是任务栈底，那么不会重启页面。
2. 重新运行 Application
3. 从任务栈栈顶向下寻找应用入口 Activity，找到后重新启动并替换到任务栈的原位置
4. 其他在任务栈当前 Activity 到入口 Activity 之间位置的 Activity，位置不变，轮到他们显示时会重走初始化的生命周期
5. 可以通过 save 和 restoreState 的方法来恢复之前的页面状态

## 完全退出不重启的方法

思路是通过模拟栈操作，先把任务栈中尚存的 Activity 依次 finish，然后再调用退出方法，这样就可以了。

1. 建立一个存放 Activity 引用的链表
2. Application 中注册 Activity 生命周期的监听器，`onCreate` 时添加引用，`onDestory` 时移除。
3. 执行退出指令前遍历 finish。

### 这样持有引用会不会泄露 ?

1. 与 Activity 原有生命周期绑定了，所以不会泄露。
2. 注意 Application 初始化时要判断 processName，防止多次初始化

## 把这种退出后重启的特性当做真正的重启功能来利用

没错，Android 没有提供快速重启应用的方法，但由于 Android 的硬件兼容性好，可能在很多非手机的设备上，开发特定功能的应用，作为智能设备使用。
利用这种退出后重启特性，我们不需要集成为系统桌面，也能快速重启 App 了。

思路是

1. LauncherActivity 只作为一个根使用，展示 LOGO 或者广告，做自动跳转，不要 finish 掉
2. 遍历退出时 finish 到倒数第二层就停止，即 LauncherActivity 之上的那个 Activity，然后杀死进程
3. 按照之前所述的流程，系统会退出当前 Activity，重新初始化 Applicatvion，重启 LauncherActivity 并替换你的任务栈根页面，整个过程跟冷启动没有区别。
4. 需要注意一点，远程进程并不会受这个过程影响，不会重新初始化