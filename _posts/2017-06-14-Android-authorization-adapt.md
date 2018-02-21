---
layout:     post
title:      "Android 跳转应用权限设置页面 适配小米系统"
subtitle:   ""
date:       2017-06-14 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - adapt
---


跳转应用设置页面方便用户修改已拒绝的权限，是经常遇到的需求，但是MIUI 8 系统上测试发现有坑，写一篇文章记录一下。

## 通常的跳转应用设置页面方法

```java
Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
Uri uri = Uri.fromParts("package", activity.getPackageName(), null);
intent.setData(uri);
activity.startActivityForResult(intent, CODE_REQUEST);
```

就是通过系统封装的隐式意图直接打开设置页面的详情而已，intent 中提供了包名告诉系统需要调取哪个应用的信息。别忘了通过 `startActivityForResult` 方式打开，毕竟从设置页面返回之后你还要重走一遍权限检查和后续方法是吧。

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode == CODE_REQUEST) {
        //TODO something;
    }
}
```

Settings 这个类中封装了很多有用的系统属性，看一下 `Settings.ACTION_APPLICATION_DETAILS_SETTINGS` 这一条的注释:

```java
/**
     * Activity Action: Show screen of details about a particular application.
     * <p>
     * In some cases, a matching Activity may not exist, so ensure you
     * safeguard against this.
     * <p>
     * Input: The Intent's data URI specifies the application package name
     * to be shown, with the "package" scheme.  That is "package:com.my.app".
     * <p>
     * Output: Nothing.
     */
    @SdkConstant(SdkConstantType.ACTIVITY_INTENT_ACTION)
    public static final String ACTION_APPLICATION_DETAILS_SETTINGS =
            "android.settings.APPLICATION_DETAILS_SETTINGS";
```
---

## MIUI 8 系统上遇到的问题

场景是这样的，我需要请求的是打开照相机权限，模拟检测到用户禁用该权限后提示去系统设置页面打开权限的情况。

在系统权限设置页授予权限后回调可以检测到权限已获取，但是小米 MIX 的测试机打开照相机后是黑屏，并提示我权限被禁用，需要在安全中心开启。

原来小米系统在安全中心另外做了一套权限管理页面，并且直接无视了 Android 本身的权限设置页面的操作结果。明目张胆的架空啊。但是用户是不管的，所以我们在 MIUI 系统上就只能提示用户去安全中心的权限管理页面进行修改了。

MIUI 上一些特殊适配的问题可以看这篇帖子: http://www.miui.com/thread-2442999-1-1.html

上代码：

```java
public static void settingPermissionActivity(Activity activity) {
    //判断是否为小米系统
    if (TextUtils.equals(BrandUtils.getSystemInfo().getOs(), BrandUtils.SYS_MIUI)) {
        Intent miuiIntent = new Intent("miui.intent.action.APP_PERM_EDITOR");
        miuiIntent.putExtra("extra_pkgname", activity.getPackageName());
        //检测是否有能接受该Intent的Activity存在
        List<ResolveInfo> resolveInfos = activity.getPackageManager().queryIntentActivities(miuiIntent, PackageManager.MATCH_DEFAULT_ONLY);
        if (resolveInfos.size() > 0) {
            activity.startActivityForResult(miuiIntent, CODE_REQUEST_CAMERA_PERMISSIONS);
            return;
        }
    }
    //如果不是小米系统 则打开Android系统的应用设置页
    Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
    Uri uri = Uri.fromParts("package", activity.getPackageName(), null);
    intent.setData(uri);
    activity.startActivityForResult(intent, CODE_REQUEST_CAMERA_PERMISSIONS);
}
```

判断系统的代码如下:

```java

/**
 * Created by Yomii on 2017/4/12.
 */

public class BrandUtils {

    public static final String SYS_EMUI = "sys_emui";
    public static final String SYS_MIUI = "sys_miui";
    public static final String SYS_FLYME = "sys_flyme";
    private static final String KEY_MIUI_VERSION_CODE = "ro.miui.ui.version.code";
    private static final String KEY_MIUI_VERSION_NAME = "ro.miui.ui.version.name";
    private static final String KEY_MIUI_INTERNAL_STORAGE = "ro.miui.internal.storage";
    private static final String KEY_EMUI_API_LEVEL = "ro.build.hw_emui_api_level";
    private static final String KEY_EMUI_VERSION = "ro.build.version.emui";
    private static final String KEY_EMUI_CONFIG_HW_SYS_VERSION = "ro.confg.hw_systemversion";

    private static SystemInfo systemInfoInstance;

    public static SystemInfo getSystemInfo() {
        if (systemInfoInstance == null) {
            synchronized (BrandUtils.class) {
                if (systemInfoInstance == null) {
                    systemInfoInstance = new SystemInfo();
                    getSystem(systemInfoInstance);
                }
            }
        }
        return systemInfoInstance;
    }

    private static void getSystem(SystemInfo info) {
        try {
            Properties prop = new Properties();
            prop.load(new FileInputStream(new File(Environment.getRootDirectory(), "build.prop")));
            if (prop.getProperty(KEY_MIUI_VERSION_CODE, null) != null
                    || prop.getProperty(KEY_MIUI_VERSION_NAME, null) != null
                    || prop.getProperty(KEY_MIUI_INTERNAL_STORAGE, null) != null) {
                info.os = SYS_MIUI;//小米
                info.versionCode = Integer.valueOf(prop.getProperty(KEY_MIUI_VERSION_CODE, "0"));
                info.versionName = prop.getProperty(KEY_MIUI_VERSION_NAME, "V0");
            } else if (prop.getProperty(KEY_EMUI_API_LEVEL, null) != null
                    || prop.getProperty(KEY_EMUI_VERSION, null) != null
                    || prop.getProperty(KEY_EMUI_CONFIG_HW_SYS_VERSION, null) != null) {
                info.os = SYS_EMUI;//华为
                info.versionCode = Integer.valueOf(prop.getProperty(KEY_EMUI_API_LEVEL, "0"));
                info.versionName = prop.getProperty(KEY_EMUI_VERSION, "unknown");
            } else if (getMeizuFlymeOSFlag().toLowerCase().contains("flyme")) {
                info.os = SYS_FLYME;//魅族
                info.versionCode = 0;
                info.versionName = "unknown";
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static String getMeizuFlymeOSFlag() {
        return getSystemProperty("ro.build.display.id", "");
    }

    private static String getSystemProperty(String key, String defaultValue) {
        try {
            Class<?> clz = Class.forName("android.os.SystemProperties");
            Method get = clz.getMethod("get", String.class, String.class);
            return (String) get.invoke(clz, key, defaultValue);
        } catch (Exception e) {
        }
        return defaultValue;
    }

    public static class SystemInfo {
        private String os = "android";
        private String versionName = Build.VERSION.RELEASE;
        private int versionCode = Build.VERSION.SDK_INT;

        public String getOs() {
            return os;
        }

        public String getVersionName() {
            return versionName;
        }

        public int getVersionCode() {
            return versionCode;
        }

        @Override
        public String toString() {
            return "SystemInfo{" +
                    "os='" + os + '\'' +
                    ", versionName='" + versionName + '\'' +
                    ", versionCode=" + versionCode +
                    '}';
        }
    }
}

```
