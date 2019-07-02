---
layout:     post
title:      "常用 ADB 命令"
subtitle:   ""
date:       2018-10-30 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Tools
    - Android
---


### 常用基础命令

```shell
# 查询当前的adb已链接的设备列表
adb devices
# adb 安装应用，可选参数 -r 允许覆盖安装已存在的应用， -t 允许安装调试版本的应用
adb install [可选参数] xxxx.apk
# adb 卸载应用
adb uninstall 包名
# 重启设备
adb reboot
# 获取root权限并重新装载文件系统
adb root
adb remount
# 推送文件到指定位置，需要root
adb push 文件路径 目标文件路径
# 拉取文件到本地
adb pull 设备内文件路径 本地路径 

```

### shell命令

```shell
# 进入adb 命令行模式
adb shell
# adb shell模式下退出到正常命令行模式
exit
# adb 截屏保存到储存卡
adb shell /system/bin/screencap -p /sdcard/screenshot.png
# adb 录制视频到储存卡
#    --time-limit 10  限制视频录制时间为10s,如果不限制,默认180s
#    --size 1280*720  设置分辨率为1280*720，需要设备高级视频编码（AVC）支持
#    --bit-rate 6000000 指定视频的比特率为6Mbps,如果不指定,默认为4Mbps
#    –verbose 显示视频录制的log
adb shell screenrecord [可选参数] /sdcard/demo.mp4
# adb 打开系统设置
adb shell am start com.android.settings -C android.intent.category.LAUNCHER
# adb 关闭系统设置系统设置
adb shell am force-stop com.android.settings
# adb 打开系统浏览器
adb shell am start com.android.browser -C android.intent.category.LAUNCHE
# adb 获取所有设备属性
adb shell getprop
# adb 获取系统版本
adb shell getprop ro.build.version.release
# adb 获取系统api版本
adb shell getprop ro.build.version.sdk
# adb 获取手机系统信息（ CPU，厂商名称等）
adb shell "cat /system/build.prop | grep "product""
# adb 获取手机相关制造商信息
adb shell getprop | grep "model\|version.sdk\|manufacturer\|hardware\|platform\|revision\|serialno\|product.name\|brand"
# adb 获取手机分辨率
adb shell "dumpsys window | grep mUnrestrictedScreen"
# adb 查看特定app包信息
adb shell dumpsys package 包名
# adb 查看特定app版本
adb shell dumpsys package 包名 | findstr version
# 查看当前正在运行的应用的任务栈
adb shell dumpsys activity
# ADB命令获取当前activity
adb shell dumpsys window w | grep \/ | grep name=
# 导出设备日志
## unix-like 系统：
## unix-like 系统上 `>` 符号也有转储作用, 但命令行就不打印了, tee会转储也不影响命令行输出
adb shell logcat -v time thread | tee ~/log.log
## win 系统：
adb shell logcat -v time thread > log.log
```

### Linux 常用命令

```shell
# adb 查看build.prop
cat /system/build.prop
# 获取网络信息
netcfg
# 获取mac地址
cat /sys/class/net/wlan0/address
# 获取内存信息
cat /proc/meminfo
# 获取cpu信息
cat /proc/cpuinfo
# 获取储存信息
df
```

### 模拟操作

```shell
# 点击设备的Backkey键
adb shell input keyevent 4
# 解锁屏幕
adb shell input keyevent  82
# 输入文字
adb shell input text abc
```

[其他操作按键对照参考:](https://blog.csdn.net/jlminghui/article/details/39268419)