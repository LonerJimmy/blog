---
title: Volley源码解析（三）——图片加载
date: 2016-07-04 17:30:18
tags: [图片,volley,源码,缓存,对象]
categories: Android
---

上一章详细介绍了volley网络请求的流程，这一章将会介绍一下volley图片加载部分源码分析。
Volley还可以进行图片的加载和缓存，可以利用ImageRequest对象简单、方便地进行网络图片的获取。ImageLoader用于获取或缓存图片。NetworkImageView是Volley提供的一个自定义View，可直接设置网络图片。

**1、使用ImageRequest进行网络图片获取**
使用方法如下：

```
ImageRequest request = new ImageRequest(url, new Response.Listener<Bitmap>() {    
    @Override    
    public void onResponse(Bitmap response) {  
        //给imageView设置图片      
        myImageView.setImageBitmap(response);    
    }
}, 0, 0, ImageView.ScaleType.CENTER, Bitmap.Config.RGB_565, new Response.ErrorListener() {    
    @Override    
    public void onErrorResponse(VolleyError error) { 
          //设置一张错误的图片，临时用ic_launcher代替          
	               myImageView.setImageResource(R.drawable.ic_launcher);    
   }
}); 

```
然后再将ImageRequest加入到RequestQueue队列中，进行网络请求，传入一些参数：
String ： 要获取的图片地址
Response.Listener<Bitmap> ： 获取图片成功的回调
int： maxWidth，获取图片的最大宽度，会自动进行压缩或拉伸，设置为0，即获取原图
int ：maxHeight，同上
ScaleType ：显示的类型，居中，平铺等
Config：图片类型，如：Bitmap.Config.RGB_565
Response.ErrorListener： 获取图片失败的回调

**2、使用ImageLoader缓存网络图片**
使用imageloader需要以下三步骤，我们从使用方法来看一下源码实现。

2.1 实例化ImageLoader

```
ImageLoader loader = new ImageLoader(requestQueue, new BitmapCache());

```
requestQueue是前面讲过的请求队列，BitmapCache是一个接口，需要我们自定义imageCache对象。我们可以使用LruCache作为图片缓存对象，如下：

```
public class BitmapCache implements ImageLoader.ImageCache {    
  private LruCache<String, Bitmap> lruCache ;    
  private int max = 10*1024*1024;    

   public BitmapCache(){       
     lruCache = new LruCache<String, Bitmap>(max){            
      @Override            
      protected int sizeOf(String key, Bitmap value) {                
         return value.getRowBytes()*value.getHeight();            
      }        
     };    
   }   
   @Override    
   public Bitmap getBitmap(String url) {        
      return lruCache.get(url);    
   }    
   @Override    
   public void putBitmap(String url, Bitmap bitmap) {          
      lruCache.put(url, bitmap);    
   }
}

```

2.2 设置监听器

```
ImageLoader.ImageListener listener = 
         ImageLoader.getImageListener(myImageView, R.drawable.ic_launcher, R.drawable.ic_launcher);

```
我们看一下ImageLoader.java中的ImageListener函数。

```
 public static ImageListener getImageListener(final ImageView view,
            final int defaultImageResId, final int errorImageResId) {
        return new ImageListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                if (errorImageResId != 0) {
                    view.setImageResource(errorImageResId);
                }
            }

            @Override
            public void onResponse(ImageContainer response, boolean isImmediate) {
                if (response.getBitmap() != null) {
                    view.setImageBitmap(response.getBitmap());
                } else if (defaultImageResId != 0) {
                    view.setImageResource(defaultImageResId);
                }
            }
        };
    }

```
就是返回一个ImageListener，里面定义了ImageListener接口。

2.3 获取图片

```
loader.get(url, listener);

```
我们看一下源码imageloader中的get函数

```
throwIfNotOnMainThread();

   final String cacheKey = getCacheKey(requestUrl, maxWidth, maxHeight, scaleType);
   Bitmap cachedBitmap = mCache.getBitmap(cacheKey);
   if (cachedBitmap != null) {
           ImageContainer container = new ImageContainer(cachedBitmap, requestUrl, null, null);
            imageListener.onResponse(container, true);
            return container;
        }

       ImageContainer imageContainer =
                new ImageContainer(null, requestUrl, cacheKey, imageListener);
       imageListener.onResponse(imageContainer, true);
       BatchedImageRequest request = mInFlightRequests.get(cacheKey);
        if (request != null) {
            request.addContainer(imageContainer);
            return imageContainer;
        }

        Request<Bitmap> newRequest = makeImageRequest(requestUrl, maxWidth, maxHeight, scaleType,
                cacheKey);

        mRequestQueue.add(newRequest);
        mInFlightRequests.put(cacheKey,
                new BatchedImageRequest(newRequest, imageContainer));
        return imageContainer;

```
我们将这些源码分解开来看。

2.3.1

```
 final String cacheKey = getCacheKey(requestUrl, maxWidth, maxHeight, scaleType);
   Bitmap cachedBitmap = mCache.getBitmap(cacheKey);
   if (cachedBitmap != null) {
           ImageContainer container = new ImageContainer(cachedBitmap, requestUrl, null, null);
            imageListener.onResponse(container, true);
            return container;
        }

```
这段代码，我们先看一下缓存中有没有我们自定义的getBitmap图片，就是我们上面自定义的BitmapCache，上面我使用的是LruCache。如果缓存中有我们要网络加载的图片的话，即cachedBitmap != null，我们就可以回调，显示出来啦。

2.3.2

```
 BatchedImageRequest request = mInFlightRequests.get(cacheKey);
        if (request != null) {
            request.addContainer(imageContainer);
            return imageContainer;
        }
```
判断一下请求是否在请求队列中，如果在就直接return。

2.3.3

```
Request<Bitmap> newRequest = makeImageRequest(requestUrl, maxWidth, maxHeight, scaleType,  cacheKey);

        mRequestQueue.add(newRequest);
        mInFlightRequests.put(cacheKey,
                new BatchedImageRequest(newRequest, imageContainer));
        return imageContainer;

```
如果不在队列和缓存中，就要就要request加入到网络请求中，关键函数就是makeImageRequest，我们看一下源码。

```
return new ImageRequest(requestUrl, new Listener<Bitmap>() {
            @Override
            public void onResponse(Bitmap response) {
                onGetImageSuccess(cacheKey, response);
            }
        }, maxWidth, maxHeight, scaleType, Config.RGB_565, new ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                onGetImageError(cacheKey, error);
            }
        });

```
这就是网络请求，返回数据处理，看一下onGetImageSuccess函数。

```
  // 放到缓存中
        mCache.putBitmap(cacheKey, response);

        // 从请求队列中remove掉
        BatchedImageRequest request = mInFlightRequests.remove(cacheKey);

        if (request != null) {
             request.mResponseBitmap = response;
             batchResponse(cacheKey, request);
        }

```
我们在看一下batchResponse函数。

```
mBatchedResponses.put(cacheKey, request);
   // If we don't already have a batch delivery runnable in flight, make a new one.
  // Note that this will be used to deliver responses to all callers in mBatchedResponses.
        if (mRunnable == null) {
            mRunnable = new Runnable() {
                @Override
                public void run() {
                    for (BatchedImageRequest bir : mBatchedResponses.values()) {
                        for (ImageContainer container : bir.mContainers) {
                            // If one of the callers in the batched request canceled the request
                            // after the response was received but before it was delivered,
                            // skip them.
                            if (container.mListener == null) {
                                continue;
                            }
                            if (bir.getError() == null) {
                                container.mBitmap = bir.mResponseBitmap;
                                container.mListener.onResponse(container, false);
                            } else {
                                container.mListener.onErrorResponse(bir.getError());
                            }
                        }
                    }
                    mBatchedResponses.clear();
                    mRunnable = null;
                }

            };
            // Post the runnable.
            mHandler.postDelayed(mRunnable, mBatchResponseDelayMs);
        }

```
请求队列循环，当container.mListener ！= null并且bir.getError() == null，就可以回调container.mListener.onResponse(container, false)函数，然后这个线程每隔100ms执行一次。

volley图片加载部分到这里就结束了，但我还是没有找到图片网络请求具体部分在哪里，这是我对volley网络图片加载部分的理解，有些地方的理解有问题，如果有不对的地方，希望各位大神指点。
