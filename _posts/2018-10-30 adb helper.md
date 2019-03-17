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
# adb 查看特定app包信息
adb shell dumpsys package 包名
# adb 查看特定app版本
adb shell dumpsys package 包名 | findstr version
# 查看当前正在运行的应用的任务栈
adb shell dumpsys activity
# ADB命令获取当前activity
adb shell dumpsys window w | grep \/ | grep name=
```

### Linux 常用命令

```shell
# adb 查看build.prop
cat /system/build.prop
```

