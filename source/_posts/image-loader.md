---
title: ImageLoader源码解析
date: 2016-06-06 17:12:18
tags: [android,源码,缓存]
categories: Android
---
## 分类 ##
功能介绍
总体设计
流程图
源码解析
LoadAndDisplayImageTask实现流程
## 功能介绍 ##
***1.1介绍***
android image loader是一个强大的、可以高度定制的图片缓存，主要工作就是获取图片并且显示在相应控件上。

***1.2.1初始化***

```
options = new DisplayImageOptions.Builder()
	.showImageOnLoading(R.drawable.ic_stub)
	.showImageForEmptyUri(R.drawable.ic_empty)
	.showImageOnFail(R.drawable.ic_error)
	.cacheInMemory(true)
	.cacheOnDisk(true)
	.considerExifParams(true)
	.displayer(new CircleBitmapDisplayer(Color.WHITE, 5))
	.build();
ImageLoader.getInstance().displayImage(IMAGE_URLS[position], holder.image, options, animateFirstListener);
```

options是imageloader的配置信息，包括图片最大尺寸、线程池、缓存、下载器、解码器等

***1.2.2 Manifest配置***

```
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

添加网络权限和写外设的权限。

***1.2.3 下载显示图片***
有以下两种方法

```
imageLoader.displayImage(imageUri, imageView);
imageLoader.loadImage(imageUri, new SimpleImageLoadingListener() {
    @Override
    public void onLoadingComplete(String imageUri, View   view, Bitmap loadedImage) {
        // 图片处理
    }
});//为bitmap传递给回调接口
```

***1.3 特点***
——两级缓存，内存缓存和硬盘缓存
——多线程，异步或者同步加载图片
——可配置，支持任务线程池、下载器、解码器、内存、硬盘缓存等
——支持多种缓存算法、下载进度监听、listview图片错乱解决等

**2、总体设计**

***2.1 设计图***
UI设计图如下：
![这里写图片描述](http://img.blog.csdn.net/20160606164351384)
整个库分为：
ImageLoaderEngine,Cache和ImageDownLoader,ImageDecoder,BitmapDisplayer,BitmapProcessor五个模块，其中cache分为内存缓存(DiskCache)和硬盘缓存(MemoryCache)。
ImageLoader获取图片，然后交给ImageLoaderEngine，ImageLoaderEngine中分发任务到具体线程(LoadAndDisplayImageTask和ProcessAndDisplayImageTask)，通过Cache和ImageDownLoader获取图片(LoadAndDisplayImageTask线程)，也有可能经过BimapProcessor和ImageDecoder(ProcessAndDisplyImageTask线程)，最终转化为bitmap，BitmapDisplayer将bitmap显示在ImageAware。

***2.2 概念***
ImageLoaderEngine：
任务分发器，负责分发LoadAndDisplayImageTask和ProcessAndDisplayImageTask给具体的线程去执行。ImageLoaderEngine.java
ImageAware：显示图片对象，可以是ImageView等。ImageAware.java
ImageDownloader：图片下载器，从来源获取输入流。BaseImageDownloader.java
MemoryCache：向内存缓存中读取图片。BaseMemoryCache.java
DiskCache：向本地磁盘缓存中读取图片。BaseDiskCache.java
ImageDecoder：图片解码器，将图片输入流InputStream转化为bitmap。BaseImageDecoder.java
BitmapDisplayer：将bitmap对象显示在ImageAware上。接口文件BitmapDisplayer.java
LoadAndDisplayImageTask：加载并显示图片。LoadAndDisplayImageTask.java
ProcessAndDisplayImageTask：处理并显示图片。ProcessAndDisplayImageTask.java
DisplayBitmapTask：显示图片。DisplayBitmapTask.java

***3、流程图***

![这里写图片描述](http://img.blog.csdn.net/20160606164836859)


***4、用到文件的源码解析***
下面文件是一些imageloader中用的文件，具体代码注释可以去我的github上。

4.1 ImageLoader

4.1.1 ImageLoader.java
```public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,ImageSize targetSize, ImageLoadingListener listener, ImageLoadingProgressListener progressListener)
```

注解可以看一下我的github，参数解释如下：
uri：图片url，支持HTTP(“http”), HTTPS("https"), FILE("file"), CONTENT("content"), ASSETS("assets"), DRAWABLE("drawable"), UNKNOWN(“")
imageAware：需要加载图片的对象，width，height，scaleType，id等
options：图片显示配置项，imageOnLoading（加载中），imageOnFail（加载失败显示的图片），cacheOnDisk（图片是否需要缓存到硬盘中），cacheInMemory（图片是否需要缓存到内存中）等
listener：图片加载的时候回调接口，onLoadingStarted，onLoadingFailed，onLoadingComplete，onLoadingCancelled。
processListener：图片加载进度的接口，onProgressUpdate。

4.1.2 ImageLoaderConfiguration

4.1.3 ImageLoaderConfiguration.builder

4.1.4 ImageLoaderConfiguration中的SlowNetworkImageDownloader.java 静态内部类

4.1.5 ImageLoaderConfiguration中的NetworkDeniedImageDownloader.java静态内部类

4.2 ImageLoaderEngine.java
LoadAndDisplayImageTask和ProcessAndDisplayImageTask任务分发器，负责分发任务给具体的线程池。

4.3 DefaultConfigurationFactory.java
默认配置。

4.4 ImageAware.java

4.5 ViewAware.java
实现ImageAware接口的抽象类，利用Reference来Warp View防止内存泄露。

4.6 ImageViewAware.java

4.7 NoneViewAware.java
处理图片相关信息却没有需要显示图片的 View 的ImageAware，实现了ImageAware接口。常用于加载图片后调用回调接口而不是显示的情况。

4.8 DisplayImageOptions.java
图片显示的配置项。比如加载前、中、失败应该显示的图片，图片是否需要在磁盘中缓存，是否需要在内存中缓存等。

4.8.1 DisplayImageOptions.Builder静态内部类

4.9 SimpleImageLoadingListener.java
ImageLoader.displayImage(…)中的listener，实现回调

4.10 ImageLoadingProgressListener.java
Image加载进度的回调接口。

4.11 DisplayBitmapTask.java
显示图片的task，必须要在主线程中调用
我之前写了一个内存缓存的显示imageview的相册选择器，在下拉的时候，因为异步加载和view复用的时候，会导致图片显示混乱。DisplayBitmapTask.java中的run中，使用方法isViewWasReused来判断imageAware是否被复用就可以解决这个问题，isViewWasReused就是根据当前正在加载的图片的key和缓存中的key进行比较，如果不相等，就返回true，如果相等，就返回false，当isViewWasReused为true的时候，取消加载。即当前加载图片必须要跟缓存中的图片一致才能继续加载该图片，防止图片显示混乱。

4.12  ProcessAndDisplayImageTask.java
处理并显示图片。

4.13 LoadAndDisplayImageTask.java
加载并显示图片，task从网络、文件系统或者内存中获取图片并解析，然后调用DisplayBitmapTask在ImageAware中显示图片。
详细的LoadAndDisplayImageTask实现流程可以看第五部分的内容。

4.14 ImageLoadingInfo.java
加载和显示图片任务需要的信息。

4.15 ImageDownloader.java
图片下载接口

4.16 BaseImageDownloader.java
ImageDownloader的具体实现类。得到Scheme对应的照片InputStream。

4.17 BaseMemoryCache.java
实现memoryCache主要函数的抽象类。

4.18 WeakMemoryCache.java
以WeakReference<Bitmap>做为缓存 value 的内存缓存，实现了BaseMemoryCache。
实现了BaseMemoryCache的createReference(Bitmap value)函数，直接返回一个new WeakReference<Bitmap>(value)做为缓存 value。

4.19 LimitedMemoryCache.java
限制总体字节大小的内存缓存。

4.20 LargestLimitedMemoryCache.java
继承LimitedMemoryCache，在缓存满的时候优先删除size最大的元素，实现LimitedMemoryCache中的removeNext抽象函数。

4.21 UsingFreqLimitedMemoryCache.java
继承LimitedMemoryCache，在缓存满的时候优先删除使用次数最少的元素。

4.22 LRULimitedMemoryCache.java
优先删除最近最少使用的元素。

4.23 FIFOLimitedMemoryCache.java
删除优先进入缓存的元素。

4.24 LimitedAgeMemoryCache.java

4.25 FuzzyKeyMemoryCache.java

4.26 FileNameGenerator.java
根据uri得到文件名的接口。

4.27 HashCodeFileNameGenerator.java
以uri的hashcode作为文件名。

4.28 Md5FileNameGenerator.java
以uri的MD5值作为文件名。

4.29 DiskCache.java
图片的磁盘缓存接口。

4.30 BaseDiskCache.java
一个没有大小限制的本地图片缓存，实现了DiskCache主要函数的抽象类

4.31 LimitedAgeDiskCache.java
限制缓存周期的磁盘缓存，继承BaseDiskCache

4.32 UnlimitedDiskCache.java
无大小限制的图片缓存。

4.33 DiskLruCache.java
限制总体字节大小的内存缓存，内存满的时候优先删除最少使用的元素。

4.34 LruDiskCache.java
限制总字节大小的内存缓存，缓存满时优先删除最近最少使用的元素。

4.35 FailReason.java
图片下载及显示时的错误原因，目前包括：
IO_ERROR 网络连接或是磁盘存储错误。
DECODING_ERROR decode image 为 Bitmap 时错误。
NETWORK_DENIED 当图片不在缓存中，且设置不允许访问网络时的错误。
OUT_OF_MEMORY 内存溢出错误。
UNKNOWN 未知错误。

4.36  ImageScaleType.java
NONE不缩放。
NONE_SAFE根据需要以整数倍缩小图片，使得其尺寸不超过 Texture 可接受最大尺寸。
IN_SAMPLE_POWER_OF_2根据需要以 2 的 n 次幂缩小图片，使其尺寸不超过目标大小，比较快的缩小方式。
IN_SAMPLE_INT根据需要以整数倍缩小图片，使其尺寸不超过目标大小。
EXACTLY根据需要缩小图片到宽或高有一个与目标尺寸一致。
EXACTLY_STRETCHED根据需要缩放图片到宽或高有一个与目标尺寸一致。

4.37 ViewScaleType.java
ImageAware的 ScaleType。
将 ImageView 的 ScaleType 简化为两种FIT_INSIDE和CROP两种。FIT_INSIDE表示将图片缩放到至少宽度和高度有一个小于等于 View 的对应尺寸，CROP表示将图片缩放到宽度和高度都大于等于 View 的对应尺寸。

4.38 ImageSize.java
scaleDown(…) 等比缩小宽高。
scale(…) 等比放大宽高。

4.39 LoadedFrom.java
图片来源，包括网络、硬盘缓存、内存缓存等。

4.40 ImageDecoder.java
将图片转换为Bitmap接口，抽象函数：decode（ImageDecodingInfo imageDecodingInfo），根据ImageDecodingInfo信息得到图片并将其转换为Bitmap。

4.41 BaseImageDecoder.java
实现了ImageDecoder。调用ImageDownloader获取图片，然后根据ImageDecodingInfo或图片 Exif 信息处理图片转换为 Bitmap。

4.42 ImageDecodingInfo.java

4.43 BitmapDisplay.java

4.44 FadeInBitmapDisplayer.java
图片淡入方式显示在ImageAware中，实现BitmapDisplayer接口

4.45 RoundedBitmapDisplayer.java
为图片添加圆角显示在ImageAware中

4.46 RoundedVignetteBitmapDisplayer.java
为图片添加渐变效果的圆角显示在ImageAware中。

4.47 SimpleBitmapDisplay.java
直接将图片显示在ImageAware中。
Image Decoder需要的信息

4.48 PauseOnScrollListener.java
在View滚动过程中暂停图片加载的Listener，实现OnScrollListener接口。
实现原理：
重写onScrollStateChanged(…)函数判断不同的状态下暂停或继续图片加载。
OnScrollListener.SCROLL_STATE_IDLE表示 View 处于空闲状态，没有在滚动，这时候会加载图片。
OnScrollListener.SCROLL_STATE_TOUCH_SCROLL表示 View 处于触摸滑动状态，手指依然在屏幕上，通过pauseOnScroll变量确定是否需要暂停图片加载。这种时候大都属于慢速滚动浏览状态，所以建议继续图片加载。
OnScrollListener.SCROLL_STATE_FLING表示 View 处于甩指滚动状态，手指已离开屏幕，通过pauseOnFling变量确定是否需要暂停图片加载。这种时候大都属于快速滚动状态，所以建议暂停图片加载以节省资源。

***5 LoadAndDisplayImageTask实现流程***

流程图如下：
1）判断图片的内存缓存是否存在，若存在直接执行步骤 8；
2）判断图片的磁盘缓存是否存在，若存在直接执行步骤 5；
3）从网络上下载图片；
4）将图片缓存在磁盘上；
5）将图片 decode 成 bitmap 对象；
6）根据DisplayImageOptions配置对图片进行预处理(Pre-process Bitmap)；
7）将 bitmap 对象缓存到内存中；
8）根据DisplayImageOptions配置对图片进行后处理(Post-process Bitmap)；
9）执行DisplayBitmapTask将图片显示在相应的控件上。
![这里写图片描述](http://img.blog.csdn.net/20160606170806858)





具体代码注释，我都push到我的github上，有兴趣的同学可以点击下面的链接，去我github上fork一下项目。


[***具体代码注释看请我的github***](https://github.com/LonerJimmy/Android-Universal-Image-Loader)