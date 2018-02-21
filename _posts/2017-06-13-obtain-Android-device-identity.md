---
layout:     post
title:      "获取 Android 设备的唯一标识符"
subtitle:   ""
date:       2017-06-13 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
---

最近做的一个需求，客户要求账号最多绑定三台设备。我之所以说是唯一标识符而不是获取Android设备的IMEI是因为IMEI并不是唯一的解决方案，也不一定是最优解，具体还要看需求。

## IMEI

先说一下最常用的IMEI，android系统中通常用下面这段代码获取。

```java
/**
 * 获取手机IMEI号
 *
 * 需要动态权限: android.permission.READ_PHONE_STATE
 */
public static String getIMEI(Context context) {
        TelephonyManager telephonyManager = (TelephonyManager) context.getSystemService(context.TELEPHONY_SERVICE);
        String imei = telephonyManager.getDeviceId();

        return imei;
    }

```

它有3个缺点:

1. 需要`android.permission.READ_PHONE_STATE`权限，它在6.0+系统中是需要动态申请的。如果需求要求App启动时上报设备标识符的话，那么第一会影响初始化速度，第二还有可能被用户拒绝授权。

2.  android系统碎片化严重，有的手机可能拿不到DeviceId，会返回null或者000000。

3.  这个方法是只对有电话功能的设备有效的，在pad上不起作用。 可以看下方法注释

```java
/**
     * Returns the unique device ID, for example, the IMEI for GSM and the MEID
     * or ESN for CDMA phones. Return null if device ID is not available.
     *
     * <p>Requires Permission:
     *   {@link android.Manifest.permission#READ_PHONE_STATE READ_PHONE_STATE}
     */
    public String getDeviceId() {
        try {
            ITelephony telephony = getITelephony();
            if (telephony == null)
                return null;
            return telephony.getDeviceId(mContext.getOpPackageName());
        } catch (RemoteException ex) {
            return null;
        } catch (NullPointerException ex) {
            return null;
        }
    }
```

## AndroidId

在设备首次启动时，系统会随机生成一个64位的数字，并把这个数字以16进制字符串的形式保存下来。不需要权限，平板设备通用。获取成功率也较高，缺点是设备恢复出厂设置会重置。另外就是某些厂商的低版本系统会有bug，返回的都是相同的AndroidId。获取方式如下：

```java
 String ANDROID_ID = Settings.System.getString(getContentResolver(), Settings.System.ANDROID_ID);
```

## Serial Number

Android系统2.3版本以上可以通过下面的方法得到Serial Number，且非手机设备也可以通过该接口获取。不需要权限，通用性也较高，但我测试发现红米手机返回的是 `0123456789ABCDEF` 明显是一个顺序的非随机字符串。也不一定靠谱。

```
String SerialNumber = android.os.Build.SERIAL;
```

## 其他方法

经常被提到的还有下面几个

1. Mac地址 -- 属于Android系统的保护信息，高版本系统获取的话容易失败，返回0000000；

2.  SimSerialNum -- 显而易见，只能用在插了Sim的设备上，通用性不好。而且需要`android.permission.READ_PHONE_STATE`权限

## 总结

综上述，AndroidId 和 Serial Number 的通用性都较好，并且不受权限限制，如果刷机和恢复出厂设置会导致设备标识符重置这一点可以接受的话，那么将他们组合使用时，唯一性就可以应付绝大多数设备了。

```java
String androidID = Settings.Secure.getString(context.getContentResolver(), Settings.Secure.ANDROID_ID);
String id = androidID + Build.SERIAL;
```

但还可以优化一下。直接暴露用户的设备信息并不是一个好的选择，既然我需要的只是一个唯一标识，那么将他们转化成Md5即可，格式也更整齐。

```java
/**
 * Created by Yomii on 2017/6/8.
 */

public class DeviceUtils {

    public static String getUniqueId(Context context){
        String androidID = Settings.Secure.getString(context.getContentResolver(), Settings.Secure.ANDROID_ID);
        String id = androidID + Build.SERIAL;
        try {
            return toMD5(id);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
            return id;
        }
    }


    private static String toMD5(String text) throws NoSuchAlgorithmException {
        //获取摘要器 MessageDigest
        MessageDigest messageDigest = MessageDigest.getInstance("MD5");
        //通过摘要器对字符串的二进制字节数组进行hash计算
        byte[] digest = messageDigest.digest(text.getBytes());

        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < digest.length; i++) {
            //循环每个字符 将计算结果转化为正整数;
            int digestInt = digest[i] & 0xff;
            //将10进制转化为较短的16进制
            String hexString = Integer.toHexString(digestInt);
            //转化结果如果是个位数会省略0,因此判断并补0
            if (hexString.length() < 2) {
                sb.append(0);
            }
            //将循环结果添加到缓冲区
            sb.append(hexString);
        }
        //返回整个结果
        return sb.toString();
    }
}

```
