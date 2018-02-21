---
layout:     post
title:      "获取网络图片的 ImageSpan"
subtitle:   ""
date:       2016-10-04 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Android
    - Text
---


TextView做图文混排时可能用到Html下的ImageGetter工具或者ImageSpan。

ImageGetter获取网络图片的例子很多， 但是如果文本中不包含图片的宽高信息，那么考虑预留多少空间是挺麻烦的事。

本例使用ImageSpan+Glide获取网络图片， 适用于文本内容较短的场景（因为获取到图片后会刷新控件，内容多会卡顿）。图片加载库换成类似的也行， 原理都一样

## 效果

![](http://ofaeieqjq.bkt.clouddn.com/Demo/urlImageSpanDemo.gif)

## 代码

```java
/**
 * 获取网络图片的ImageSpan
 * Created by Yomii on 2016/10/13.
 */
public class UrlImageSpan extends ImageSpan {

    private String url;
    private TextView tv;
    private boolean picShowed;

    public UrlImageSpan(Context context, String url, TextView tv) {
        super(context, R.mipmap.pic_holder);
        this.url = url;
        this.tv = tv;
    }

    @Override
    public Drawable getDrawable() {
        if (!picShowed) {
            Glide.with(tv.getContext()).load(url).asBitmap().into(new SimpleTarget<Bitmap>() {
                @Override
                public void onResourceReady(Bitmap resource, GlideAnimation<? super Bitmap> glideAnimation) {
                    Resources resources = tv.getContext().getResources();
                    int targetWidth = (int) (resources.getDisplayMetrics().widthPixels * 0.8);
                    Bitmap zoom = zoom(resource, targetWidth);
                    BitmapDrawable b = new BitmapDrawable(resources, zoom);

                    b.setBounds(0, 0, b.getIntrinsicWidth(), b.getIntrinsicHeight());
                    Field mDrawable;
                    Field mDrawableRef;
                    try {
                        mDrawable = ImageSpan.class.getDeclaredField("mDrawable");
                        mDrawable.setAccessible(true);
                        mDrawable.set(UrlImageSpan.this, b);

                        mDrawableRef = DynamicDrawableSpan.class.getDeclaredField("mDrawableRef");
                        mDrawableRef.setAccessible(true);
                        mDrawableRef.set(UrlImageSpan.this, null);

                        picShowed = true;
                        tv.setText(tv.getText());
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    } catch (NoSuchFieldException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        return super.getDrawable();
    }

    /**
     * 按宽度缩放图片
     *
     * @param bmp  需要缩放的图片源
     * @param newW 需要缩放成的图片宽度
     *
     * @return 缩放后的图片
     */
    public static Bitmap zoom(@NonNull Bitmap bmp, int newW) {

        // 获得图片的宽高
        int width = bmp.getWidth();
        int height = bmp.getHeight();

        // 计算缩放比例
        float scale = ((float) newW) / width;

        // 取得想要缩放的matrix参数
        Matrix matrix = new Matrix();
        matrix.postScale(scale, scale);

        // 得到新的图片
        Bitmap newbm = Bitmap.createBitmap(bmp, 0, 0, width, height, matrix, true);

        return newbm;
    }
}

```

## 实现方式

先通过Uri或者ResId的构造传入一个占位图。

在getDrawable这个方法中, 如果图片未加载, 那么异步获取图片, 否则直接返回父类。

成功获取到图片的回调中， 通过反射把占位图替换为获取到的图片， 然后刷新文本控件就可以。缩放部分不是核心代码，获取到图片怎么改都行。

### 原理

首先看下父类ImageSpan， 只有一个核心方法getDrawable， 其他都是通过不同方式获取图片的构造。


```java
    @Override
    public Drawable getDrawable() {
        Drawable drawable = null;

        if (mDrawable != null) {
            drawable = mDrawable;
        } else  if (mContentUri != null) {
            Bitmap bitmap = null;
            try {
                InputStream is = mContext.getContentResolver().openInputStream(
                        mContentUri);
                bitmap = BitmapFactory.decodeStream(is);
                drawable = new BitmapDrawable(mContext.getResources(), bitmap);
                drawable.setBounds(0, 0, drawable.getIntrinsicWidth(),
                        drawable.getIntrinsicHeight());
                is.close();
            } catch (Exception e) {
                Log.e("sms", "Failed to loaded content " + mContentUri, e);
            }
        } else {
            try {
                drawable = mContext.getDrawable(mResourceId);
                drawable.setBounds(0, 0, drawable.getIntrinsicWidth(),
                        drawable.getIntrinsicHeight());
            } catch (Exception e) {
                Log.e("sms", "Unable to find resource: " + mResourceId);
            }                
        }

        return drawable;
    }
```


这个方法中对应不同的构造获取一个Drawable并返回, 并且首先判断mDrawable属性, 也就是说我们把获取到的图片改成它就行了。

但是光改它还不行， 依然没有到核心方法draw（）， 看一下它在父类中是如何被调用的。

在DynamicDrawableSpan的draw方法中，绘制的bitmap是从getCachedDrawable()方法中获取的。

可以看到在getCachedDrawable()里并不是优先调用getDrawable而是调用一个缓存弱引用mDrawableRef。

由于获取图片是异步的，因此这个mDrawableRef必定存在，它就是我们的占位图。于是只要在回调时把它置空并刷新控件，就能成功调用getDrawable，返回之前替换进去的目标图片。

```java
    @Override
    public void draw(Canvas canvas, CharSequence text,
                     int start, int end, float x,
                     int top, int y, int bottom, Paint paint) {
        Drawable b = getCachedDrawable();
        canvas.save();

        int transY = bottom - b.getBounds().bottom;
        if (mVerticalAlignment == ALIGN_BASELINE) {
            transY -= paint.getFontMetricsInt().descent;
        }

        canvas.translate(x, transY);
        b.draw(canvas);
        canvas.restore();
    }

    private Drawable getCachedDrawable() {
        WeakReference<Drawable> wr = mDrawableRef;
        Drawable d = null;

        if (wr != null)
            d = wr.get();

        if (d == null) {
            d = getDrawable();
            mDrawableRef = new WeakReference<Drawable>(d);
        }

        return d;
    }

```
