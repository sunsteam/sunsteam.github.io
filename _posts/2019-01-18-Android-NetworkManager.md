---
layout:     post
title:      "Android NetworkManager"
subtitle:   ""
date:       2019-01-18 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
---

# Android NetworkManager

对网络相关Api进行整理

## 需要权限

@RequiresPermission(android.Manifest.permission.ACCESS_NETWORK_STATE)

## 获取网络

* 当前网络  `manager.getActiveNetwork()`
* 动态网络回调  `manager.registerNetworkCallback`


## 网络的不同侧面

新的Api中网络的不同关注面被放到的不同的对象中

### 网络状态信息 `manager.getNetworkInfo(network)`

包括是否连接、连接状态(连接中、已连接、挂起、断开等)，与其他网络设备的交互状态(DetailState, 如扫描、授权、分配地址等)

### 网络连接信息 `manager.getLinkProperties(network)`

网络连接信息包括IP、DNS、域名、路由等信息

如果需要获取动态的网络连接信息改变，可以注册回调，并使用这个Api

### 网络性能信息 `manager.getNetworkCapabilities(network)`

包括两方面
- 是否能访问该类网络，关注能与不能
- 当前该类网络是否能连通，关注目前有或没有该能力

另外该类还能预估当前网络的上行和下行带宽

### 打印信息

```
NetworkInfo : [
    type: WIFI[], 
    state: CONNECTED/CONNECTED, 
    reason: (unspecified), 
    extra: "Liking-Dev", 
    roaming: false, 
    failover: false, 
    isAvailable: true
]

LinkProperties : {
    InterfaceName: wlan0 
    LinkAddresses: [
        fe80::e6db:6dff:fefa:f720/64,
        172.16.100.105/24,
        ]  
    Routes: [
        fe80::/64 -> :: wlan0,
        172.16.100.0/24 -> 0.0.0.0 wlan0,
        0.0.0.0/0 -> 172.16.100.1 wlan0,
        ] 
    DnsAddresses: [
        114.114.114.114,
        223.5.5.5,
        ] 
    Domains: null 
    MTU: 0 
    TcpBufferSizes: 524288,1048576,2097152,262144,524288,1048576
}

NetworkCapabilities : [ 
    Transports: WIFI 
    Capabilities: INTERNET&NOT_RESTRICTED&TRUSTED&NOT_VPN&VALIDATED LinkUpBandwidth>=1048576Kbps 
    LinkDnBandwidth>=1048576Kbps 
    SignalStrength: -54
]

```


