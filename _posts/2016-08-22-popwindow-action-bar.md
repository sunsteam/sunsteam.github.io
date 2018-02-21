---
layout:     post
title:      "使用 PopupWindow 模仿 ActionBar 下拉菜单效果"
subtitle:   ""
date:       2016-09-22 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - UI
---



## 效果

![](http://ofaeieqjq.bkt.clouddn.com/CustomLayout/overflowDemo.gif)

## 设置布局

#### 代码
```java
public class PopupOverFlow extends PopupWindow implements View.OnClickListener, PopupWindow.OnDismissListener {


    public PopupOverFlow(Context context) {
        // 设置SelectPicPopupWindow的View
        View contentView = View.inflate(context, R.layout.common_reading_overflow, null);
        setContentView(contentView);
        // 设置SelectPicPopupWindow弹出窗体的宽
        contentView.measure(0,0);
        setWidth(contentView.getMeasuredWidth());
        // 设置SelectPicPopupWindow弹出窗体的高
        setHeight(ViewGroup.LayoutParams.WRAP_CONTENT);
        // 设置SelectPicPopupWindow弹出窗体可点击
        setFocusable(true);
        setOutsideTouchable(true);
        // 点back键和其他地方使其消失,设置了这个才能触发OnDismisslistener ，设置其他控件变化等操作
        BitmapDrawable dw = new BitmapDrawable();
        setBackgroundDrawable(dw);
        // 设置SelectPicPopupWindow弹出窗体动画效果
        setAnimationStyle(R.style.AnimTools);
        contentView.findViewById(R.id.overflow_postil).setOnClickListener(this);
        contentView.findViewById(R.id.overflow_label).setOnClickListener(this);
        contentView.findViewById(R.id.overflow_mandatory).setOnClickListener(this);
        contentView.findViewById(R.id.overflow_share).setOnClickListener(this);

        setOnDismissListener(this);
    }


    /**
     * 显示popupWindow
     *
     * @param parent anchor
     *
     * @return true: show  ,  false: hide
     */
    public boolean showOrHideOverflow(View parent) {
        boolean showing = isShowing();
        if (!showing) {
            // 以下拉方式显示popupwindow
            this.showAsDropDown(parent,  0, 18);
        } else {
            this.dismiss();
        }
        return !showing;
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.overflow_postil:
                if (l != null)
                    l.onAddStdPostil(v);
                break;
            case R.id.overflow_label:
                if (l != null)
                    l.onLabelClick(v);
                break;
            case R.id.overflow_mandatory:
                if (l != null)
                    l.onMandatoryClick(v);
                break;
            case R.id.overflow_share:
                if (l != null)
                    l.onShareClick(v);
                break;
        }
        this.dismiss();
    }

    private MenuListener l;

    public void setOnMenuClickListener(MenuListener l) {
        this.l = l;
    }

    @Override
    public void onDismiss() {
        if (l != null)
            l.onDismiss();
    }

    public interface MenuListener {

        void onDismiss();

        void onAddStdPostil(View v);

        void onLabelClick(View v);

        void onMandatoryClick(View v);

        void onShareClick(View v);
    }
}
```



#### 动画文件

* style
```xml
<style name="AnimTools" parent="@android:style/Animation">
    <item name="android:windowEnterAnimation">@anim/push_in</item>
    <item name="android:windowExitAnimation">@anim/push_out</item>
</style>
```

* push_in
```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- 左上角扩大-->
<scale   xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@android:anim/accelerate_decelerate_interpolator"
    android:fromXScale="0.001"
    android:toXScale="1.0"
    android:fromYScale="0.001"
    android:toYScale="1.0"
    android:pivotX="90%"
    android:pivotY="0"
    android:duration="100" />
```

* push_out
```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- 左上角缩小 -->
<scale   xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@android:anim/accelerate_decelerate_interpolator"
    android:fromXScale="1.0"
    android:toXScale="0.001"
    android:fromYScale="1.0"
    android:toYScale="0.001"
    android:pivotX="90%"
    android:pivotY="0"
    android:duration="200" />

```





#### 布局文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:paddingRight="@dimen/dp_5">

    <LinearLayout
        android:layout_width="@dimen/dp_120"
        android:layout_height="wrap_content"
        android:background="@color/white"
        android:orientation="vertical">

        <TextView
            android:id="@+id/overflow_postil"
            style="@style/std_overflow_item"
            android:drawableLeft="@mipmap/icon_annotation"
            android:text="@string/overflow_add_postil" />

        <View
            android:layout_width="match_parent"
            android:layout_height="@dimen/dp_0.2"
            android:background="#000" />


        <TextView
            android:id="@+id/overflow_label"
            style="@style/std_overflow_item"
            android:drawableLeft="@mipmap/icon_tips"
            android:text="@string/overflow_add_label" />

        <View
            android:layout_width="match_parent"
            android:layout_height="@dimen/dp_0.2"
            android:background="#000" />

        <TextView
            android:id="@+id/overflow_mandatory"
            style="@style/std_overflow_item"
            android:drawableLeft="@mipmap/icon_warning"
            android:text="@string/overflow_mandatory" />

        <View
            android:layout_width="match_parent"
            android:layout_height="@dimen/dp_0.2"
            android:background="#000" />


        <TextView
            android:id="@+id/overflow_share"
            style="@style/std_overflow_item"
            android:drawableLeft="@mipmap/icon_share"
            android:text="@string/overflow_share" />


    </LinearLayout>
</FrameLayout>
```





## 设置窗体透明效果

```java
//弹出窗口时结合改变窗体透明度
if (popupOverFlow.showOrHideOverflow(v)) {
    WindowManager.LayoutParams params = getWindow().getAttributes();
    params.alpha = 0.7f;
    getWindow().setAttributes(params);
}
```

```java
	//关闭时回复窗口透明度
    private void closePopupWindow(Activity context) {
        this.dismiss();
        WindowManager.LayoutParams params = context.getWindow().getAttributes();
        params.alpha = 1f;
        context.getWindow().setAttributes(params);
    }
```
