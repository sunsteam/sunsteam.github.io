---
layout:     post
title:      "自定义分享界面 动态创建模糊背景"
subtitle:   ""
date:       2016-11-07 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - Render
---


分享页面要求做一个背景半透明模糊效果, 记录一下过程, 另外分享窗口是需要部分自定义的, 直接改ShareSDK的页面代码


## 效果

![](http://ofaeieqjq.bkt.clouddn.com/CustomLayout/blurBackground.gif)

## 添加gradle支持

在android下的defaultConfig代码块进行如下设置

```
defaultConfig {
    ......
    renderscriptTargetApi 18
    renderscriptSupportModeEnabled true
}

```

## 获取当前页面根视图的截屏

```java
    /**
     * 快照
     *
     * @param view         需要快照的view
     * @param bitmapWidth  照片宽度
     * @param bitmapHeight 照片高度
     * @param offset_Y     镜头偏移量
     */
    public static Bitmap convertViewToBitmap(View view, int bitmapWidth, int bitmapHeight,
                                             int offset_Y) {
        Bitmap bitmap = Bitmap.createBitmap(bitmapWidth, bitmapHeight, Config.ARGB_4444);
        Canvas canvas = new Canvas(bitmap);
        if (offset_Y != 0)
            canvas.translate(0, -offset_Y);
        view.draw(canvas);

        return bitmap;
    }
```

## 使用RenderScript模糊图片

```java
    /**
     * 对图片添加模糊效果
     *
     * @param image 资源
     *
     * @return 模糊后的bitmap
     */
    public static Bitmap blurBitmap(Bitmap image) {
        //图片缩放比例
        final float BITMAP_SCALE = 0.4f;
        //最大模糊度(在0.0到25.0之间)
        float BLUR_RADIUS = 3f;

        // 计算图片缩小后的长宽
        int width = Math.round(image.getWidth() * BITMAP_SCALE);
        int height = Math.round(image.getHeight() * BITMAP_SCALE);

        // 将缩小后的图片做为预渲染的图片。
        Bitmap inputBitmap = Bitmap.createScaledBitmap(image, width, height, false);
        // 创建一张渲染后的输出图片。
        Bitmap outputBitmap = Bitmap.createBitmap(inputBitmap);

        // 创建RenderScript内核对象
        RenderScript rs = RenderScript.create(BaseApp.getContext());
        // 创建一个模糊效果的RenderScript的工具对象
        ScriptIntrinsicBlur blurScript = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));

        // 由于RenderScript并没有使用VM来分配内存,所以需要使用Allocation类来创建和分配内存空间。
        // 创建Allocation对象的时候其实内存是空的,需要使用copyTo()将数据填充进去。
        Allocation tmpIn = Allocation.createFromBitmap(rs, inputBitmap);
        Allocation tmpOut = Allocation.createFromBitmap(rs, outputBitmap);

        // 设置渲染的模糊程度, 25f是最大模糊度
        blurScript.setRadius(BLUR_RADIUS);
        // 设置blurScript对象的输入内存
        blurScript.setInput(tmpIn);
        // 将输出数据保存到输出内存中
        blurScript.forEach(tmpOut);

        // 将数据填充到Allocation中
        tmpOut.copyTo(outputBitmap);

        tmpIn.destroy();
        tmpOut.destroy();
        blurScript.destroy();
        rs.destroy();
		// 如果用来模糊的图片还需要使用, 就注掉recycle这行代码
        inputBitmap.recycle();

        return outputBitmap;
    }
```

## 在页面上覆盖模糊后的图片

在根布局外套一个FrameLayout并加一个ImageView在上层, 用来放模糊后的图片。 然后通过修改当前窗体的透明度实现半透明效果。

```java
    private void getBlurBackground() {
        View view = getView();
        Bitmap bitmap = UIUtils.convertViewToBitmap(view, view.getWidth(), view.getHeight(), 0);
        BitmapDrawable bitmapDrawable = new BitmapDrawable(getResources(), UIUtils.blurBitmap(bitmap));
        blurBg.setBackgroundDrawable(bitmapDrawable);

        WindowManager.LayoutParams attributes = getActivity().getWindow().getAttributes();
        attributes.alpha = 0.5f;
        getActivity().getWindow().setAttributes(attributes);
    }
```

分享窗口消失回调页面的OnResume生命周期， 在这里置空模糊图片并恢复透明度为1。

```java
    @Override
    public void onResume() {
        super.onResume();
        if (blurBg != null && blurBg.getBackground() != null) {
            blurBg.setBackgroundDrawable(null);
            WindowManager.LayoutParams attributes = getActivity().getWindow().getAttributes();
            attributes.alpha = 1f;
            getActivity().getWindow().setAttributes(attributes);
        }
    }
```

* 参考文章

[http://blog.csdn.net/wl9739/article/details/51955598](http://blog.csdn.net/wl9739/article/details/51955598)
