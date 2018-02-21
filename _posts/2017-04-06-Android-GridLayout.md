---
layout:     post
title:      "Android GridLayout 动态添加子控件 + 平均分配空间"
subtitle:   ""
date:       2017-04-06 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - UI
---

有时候会遇到这样的需求：
1. 要求子控件网格布局，平均分布
2. 内容根据接口动态加载
3. 父控件充满界面剩余空间，不可滑动

看到 1 和 2 的时候 GridView 和 RecyclerView 都能胜任，但是不可滑动这个需求就需要对他们做特殊处理，并且不容易做分辨率适配，这时候用 GridLayout 动态加载做起来更简单。

## 使用GridLayout

### 导入 v7 包的 GridLayout

由于原生 GridLayout 平均分配子布局要求 sdk21+，因此先导入 v7 包下的 GridLayout

`compile 'com.android.support:gridlayout-v7:25.3.1'`

### 设置 GridLayout 布局

下面的布局需要一个自定义轮播图和 2 个 GridLayout 按比例分配屏幕空间。
使用 `app:columnCount` 和 `app:rowCount` 属性定义列数和行数。
使用 weight 属性时，使用 0dp 设置动态计算的宽度或高度来进行优化。

```xml
<LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <FrameLayout
            android:id="@+id/viewpager_container"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_marginBottom="@dimen/dp_6"
            android:layout_weight="1.2" />

        <android.support.v7.widget.GridLayout
            android:id="@+id/function_grid"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_marginBottom="@dimen/dp_6"
            android:layout_weight="1"
            app:columnCount="3"
            app:rowCount="2" />

        <android.support.v7.widget.GridLayout
            android:id="@+id/industry_grid"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="2"
            app:columnCount="3"
            app:rowCount="3" />
    </LinearLayout>
```
---

## 平均分配空间

如果你只想要子布局平均分配空间即可，那么可以在 xml 布局中如下设置。
子控件通过 `app:layout_rowWeight` 和 `app:layout_columnWeight` 声明自己在布局中占据的比重，都设置为 1 即可平均分布。
**子控件的 width 和 height 可以 `wrap_content` 也可以 `0dp`，但是不可以 `match_parent`**

```xml
<android.support.v7.widget.GridLayout
            android:id="@+id/function_grid"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_marginBottom="@dimen/dp_6"
            android:layout_weight="1"
            app:columnCount="3"
            app:rowCount="2" >

            <View
                android:layout_width="0dp"
                android:layout_height="0dp"
                app:layout_row="0"
                app:layout_column="0"
                app:layout_rowWeight="1"
                app:layout_columnWeight="1"/>

            <View
                android:layout_width="0dp"
                android:layout_height="0dp"
                app:layout_row="0"
                app:layout_column="1"
                app:layout_rowWeight="1"
                app:layout_columnWeight="1"/>

            <View
                android:layout_width="0dp"
                android:layout_height="0dp"
                app:layout_row="0"
                app:layout_column="2"
                app:layout_rowWeight="1"
                app:layout_columnWeight="1"/>

            <View
                android:layout_width="0dp"
                android:layout_height="0dp"
                app:layout_row="1"
                app:layout_column="0"
                app:layout_rowWeight="1"
                app:layout_columnWeight="1"/>

            <View
                android:layout_width="0dp"
                android:layout_height="0dp"
                app:layout_row="1"
                app:layout_column="1"
                app:layout_rowWeight="1"
                app:layout_columnWeight="1"/>


        </android.support.v7.widget.GridLayout>
```
---

## 动态添加子控件

```java
for (int i = 0, j = list.size(); i < j; i++) {
    ......
    View functionView = new View(getContext());
    functionView.setBackgroundResource(iconResId);
    ......
    // 使用 Spec 定义子控件的位置和比重
    GridLayout.Spec rowSpec = GridLayout.spec(i / 3, 1f);
    GridLayout.Spec columnSpec = GridLayout.spec(i % 3, 1f);
    // 将 Spec 传入 GridLayout.LayoutParams 并设置宽高为 0，必须设置宽高，否则视图异常
    GridLayout.LayoutParams layoutParams = new GridLayout.LayoutParams(rowSpec, columnSpec);
    layoutParams.height = 0;
    layoutParams.width = 0;
    // 还可以根据位置动态定义子控件直接的边距，下面的意思是
    // 第一行的子控件都有 2dp 的 bottomMargin，中间位置的子控件都有 2dp 的 leftMargin 和 rightMargin
    if (i / 3 == 0)
        layoutParams.bottomMargin = getResources().getDimensionPixelSize(R.dimen.dp_2);
    if (i % 3 == 1) {
        layoutParams.leftMargin = getResources().getDimensionPixelSize(R.dimen.dp_2);
        layoutParams.rightMargin = getResources().getDimensionPixelSize(R.dimen.dp_2);
    }
    functionGrid.addView(functionView, layoutParams);
}
```

# 效果

![效果图](https://ws3.sinaimg.cn/large/006tNc79gy1foo0k2a3rcj30f20qsgxi.jpg)
