---
title: Glide深入浅出（一）——Glide vs Picasso
date: 2016-09-07 16:36:18
tags: [glide,picasso,源码]
categories: Android
---

之前，Google推出了一个图片加载库Glide。因为我们之前使用的ImageLoader比较多，所以现在我就准备研究一下Glide这个框架。
首先，我要先说的是好，Glide这个库看起来跟Picasso这个库非常的相似，感觉glide应该是Picasso库的翻版。那么第一章，我们就PK一下glide和Picasso。

**导入库**

Picasso

```
dependencies {
    compile 'com.squareup.picasso:picasso:2.5.1'
}
```
glide

```
dependencies {
    compile 'com.github.bumptech.glide:glide:3.5.2'
    compile 'com.android.support:support-v4:22.0.0'
}
```
glide需要android support library v4，所以在你的项目中，不要忘记导入support-v4包。

**基本使用**
说他们俩很像，其实就是在使用上两者很像。

Picasso

```
Picasso.with(context)
    .load("http://inthecheesefactory.com/uploads/source/glidepicasso/cover.jpg")
    .into(ivImg);
```

glide

```
Glide.with(context)
    .load("http://inthecheesefactory.com/uploads/source/glidepicasso/cover.jpg")
    .into(ivImg);
```
虽然两者使用起来看着很像，但是glide设计的更加好，因为glide.with(context)中，可以支持activity和fragment，context会自动识别转化的。我们可以看一下源码：

```
public static RequestManager with(Context context) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(context);
  }

  public static RequestManager with(Activity activity) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(activity);
  }

   public static RequestManager with(FragmentActivity activity) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(activity);
  }
```
可以看到这里，不仅可以传context还可以传activity和fragment对象，这样我们使用起来就不用转化context了。当然在在传入context的函数中，glide还做了识别处理，源码如下：

```
 public RequestManager get(Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }

    return getApplicationManager(context);
  }
```
又把属于fragment和activity的context又识别了一波。

传入activity或fragment的好处：图片加载将会跟着activity和fragment的生命周期活动，activity或fragment生命周期onPause的时候，图片就会pause加载，onResume的时候，图片就会onResume加载。所以推荐，不要传入context到glide，尽量传activity或者fragment。

**默认bitmap格式是 RGB_565**

glide默认的bitmap格式是RGB_565，这个比Picasso只用的ARGB_8888少用一半的内存，所以glide图片展示没有Picasso清晰，下面是两个内存消耗比较：

![这里写图片描述](http://img.blog.csdn.net/20160907154723240)

但是如果你需要高质量的图片，你需要把bitmap格式转化为ARGB_8888，你需要实现一个接口GlideModule，如下：

```
public class GlideConfiguration implements GlideModule {

    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
      builder.setDecodeFormat(DecodeFormat.PREFER_ARGB_8888);
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
    }
}
```
然后在AndroidManifest定义一个meta-data，如下：

```
<meta-data android:name="com.inthecheesefactory.lab.glidepicasso.GlideConfiguration"
            android:value="GlideModule"/>
```
这样的话，glide图片看起来好多了。
变成ARGB_8888后，glide内存消耗也变大了，但是仍然比Picasso小。
这是为啥咧？
假设我们要加载一个1920x1080的图片，Picasso会将整个图片加载下来，而glide加载的是imageview大小（假设我们imageview大小是768x432）

**Disk缓存**
当我的imageview有不同的size，而且同时加载一张图片，Picasso硬盘缓存只缓存这一张图片，但是glide机制不一样，因为glide加载的imageview，所以你如果有不同size的imageview，虽然是同一张图片，但是glide还是会将这些不同size的imageview都缓存下来。
当然你可以通过下面的方法：

```
Glide.with(this)
             .load("http://nuuneoi.com/uploads/source/playstore/cover.jpg")
             .diskCacheStrategy(DiskCacheStrategy.ALL)
             .into(ivImgGlide);
```

这样的话，缓存全大小的图片，然后在resize和cache。
glide默认这种加载方式，也有好处，就是加载速度快，Picasso缓存全尺寸图片，加载到imageview还需要测量imageview的大小进行裁剪，而glide加载的图片不需要测量大小，所以glide加载速度很快。

**glide能做，Picasso不能做的**
glide可以load gif图片，而Picasso做不到。

**库大小**

![这里写图片描述](http://img.blog.csdn.net/20160907163104426)

哈哈，glide大小能顶Picasso的四倍了，方法数也多了很多。

![这里写图片描述](http://img.blog.csdn.net/20160907163206801)

**总结**
glide和Picasso pk结果：
glide优点：加载速度快，可以处理gif、video等图片流。
Picasso优点：体积小，图片质量高。
