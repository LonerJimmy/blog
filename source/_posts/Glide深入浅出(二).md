---
title: Glide深入浅出（二）——源码解析
date: 2016-09-08 19:19:18
tags: [android,glide,源码]
categories: Android
---

我们先来看看glide最基本使用方式，如下：

```
Glide.with(this)
                .asDrawable()
                .load("http://i6.topit.me/6/5d/45/1131907198420455d6o.jpg")
                .apply(fitCenterTransform(this))
                .apply(placeholderOf(R.drawable.skyblue_logo_wechatfavorite_checked))
                .into(imageView);
```
本片文章就从这个最简单的使用方法来对源码进行一波解析。

## 一、大概介绍 ##

**Glide.java 入口文件**

这里其实就是一个入口文件，所有功能都是放在后面的类中，这里所有需要的组件都在这个文件里面进行统一初始化。

**RequestManager.java**

这个类主要是用于管理和启动glide的所有请求，可以使用activity，fragment或者连接生命周期的事件去只能的停止，启动和重启请求。也可以检索或者通过实例化一个新的对象，或者使用静态的glide去利用构建在activity和fragment生命周期处理中。它的方法跟你的fragment和activity是同步的。

**RequestBuilder.java**

可以处理设置选项，并启动负载的通用资源类型。

看前面的例子代码中，asDrawable()最终调用的的RequestBuilder中的transition()函数，这个方法主要是用于加载对象从占位符(placeholder)或者缩略图(thumbnail)到真正对象加载完成的专场动画。

load()方法中，这里可以加载很多类型的数据对象，可以是string，uri，file，resourceId，byte[]这些。对应的编码方式也是不一样的。

into()方法，是真正启动加载的地方。

## 二、具体源码解析 ##

**Glide.with()函数**

先看一下glide中的with函数，如下：

```
  public static RequestManager with(Activity activity) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(activity);
  }
```
这里返回的是RequestManagerRetriever.get()，我们看一下这个函数，如下：

```
@TargetApi(Build.VERSION_CODES.HONEYCOMB)
  public RequestManager get(Activity activity) {
    if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(activity, fm, null);
    }
  }
```
（当然这个with()可以传很多类型进来，我们只看activity的情况）
源码中，如果该activity在后台的时候，我们进入get(Application)中，继续我们刚刚说的这个步骤来。
所以我们就直接看fragmentGet(activity, fm, null)函数，源码如下：

```
@TargetApi(Build.VERSION_CODES.HONEYCOMB)
  RequestManager fragmentGet(Context context, android.app.FragmentManager fm,
      android.app.Fragment parentHint) {
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          new RequestManager(glide, current.getLifecycle(), current.getRequestManagerTreeNode());
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```
这个函数作用正如它的名字一样，就是获取fragment，其实是返回的RequestManager，那这个到底是啥？看一下getRequestManagerFragment函数，源码如下：

```
  @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
  RequestManagerFragment getRequestManagerFragment(
      final android.app.FragmentManager fm, android.app.Fragment parentHint) {
    RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      current = pendingRequestManagerFragments.get(fm);
      if (current == null) {
        current = new RequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        pendingRequestManagerFragments.put(fm, current);
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }
```
（其中传进来的参数fm，是我们获取的activity的FragmentManager）
这里其实就是就是将我们传进来的activity转化为glide需要的RequestManagerFragment，RequestManagerFragment是一个自己重新以的fragment而已。

**asDrawable函数**

源码如下：
```
  public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class).transition(new DrawableTransitionOptions());
  }
```
添加一个DrawableTransitionOptions类型的动画。

**load函数**

```
public RequestBuilder<Drawable> load(@Nullable Object model) {
    return asDrawable().load(model);
  }

public RequestBuilder<TranscodeType> load(@Nullable Object model) {
    return loadGeneric(model);
  }

  private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
  }
```
将你要传递的uri，file等等传进来，放到RequestBuilder中。

**into函数**

最终我们都要调用这个into了，源码如下：
```
public Target<TranscodeType> into(ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      if (requestOptions.isLocked()) {
        requestOptions = requestOptions.clone();
      }
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions.optionalCenterCrop(context);
          break;
        case CENTER_INSIDE:
          requestOptions.optionalCenterInside(context);
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions.optionalFitCenter(context);
          break;
        //$CASES-OMITTED$
        default:
          // Do nothing.
      }
    }

    return into(context.buildImageViewTarget(view, transcodeClass));
  }
```
这里针对ImageView的填充方式做了筛选并对应设置到requestOptions上。最终的是通过ImageView和转码类型（transcodeClass）创建不通过的Target（例如Bitmap对应的BitmapImageViewTarget和Drawable对应的DrawableImageViewTarget）。
到这里，我们看到的源码，并没有涉及到解码、缓存、加载等这些功能啊，是不是我们漏掉一些重要的函数？没错，我们的确漏掉一个很重要的函数，在into函数中，有一行代码如下：

```
requestManager.track(target, request);
```
这个就是最核心的方法，它才是真正触发请求、编解码、装载、缓存等这些功能的。下面我详细介绍一下。

**track()函数**

```
void track(Target<?> target, Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
  }
```
第一行，就是把target加入targets队列（WeakHashMap）中。
看一下runRequest函数，如下：

```
 public void runRequest(Request request) {
    requests.add(request);//添加到内存缓存
    if (!isPaused) {
      request.begin();//开始
    } else {
      pendingRequests.add(request);//挂起请求
    }
  }
```

然后看一下request.begin()是如何运作的，SingleRequest继承一个抽象类，定义了begin方法，所以我们看一下SingleRequest中begin方法。

```
@Override
  public void begin() {
    stateVerifier.throwIfRecycled();
    startTime = LogTime.getLogTime();
    // 如果model空的，那么是不能执行的。 这里的model就是前面提到的RequestBuilder中的model
    if (model == null) {
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        width = overrideWidth;
        height = overrideHeight;
      }
      // Only log at more verbose log levels if the user has set a fallback drawable, because
      // fallback Drawables indicate the user expects null models occasionally.
      int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
      onLoadFailed(new GlideException("Received null model"), logLevel);
      return;
    }
    status = Status.WAITING_FOR_SIZE;
    //如果当前的View尺寸获取到了，就会进入加载流程
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      onSizeReady(overrideWidth, overrideHeight);
    } else {
      target.getSize(this);
    }

//如果等待和正在执行状态，那么当前会加载占位符Drawable
    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
      target.onLoadStarted(getPlaceholderDrawable());
    }
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
  }
```
看上面注释，如果，当前view尺寸还没有获取到，我们要执行target.getSize(this)函数，我们看一下这个函数实现方法(ViewTarget实现的)。

```
void getSize(SizeReadyCallback cb) {
      int currentWidth = getViewWidthOrParam();
      int currentHeight = getViewHeightOrParam();
      if (isSizeValid(currentWidth) && isSizeValid(currentHeight)) {
        int paddingAdjustedWidth = currentWidth == WindowManager.LayoutParams.WRAP_CONTENT
            ? currentWidth
            : currentWidth - ViewCompat.getPaddingStart(view) - ViewCompat.getPaddingEnd(view);
        int paddingAdjustedHeight = currentHeight == LayoutParams.WRAP_CONTENT
            ? currentHeight
            : currentHeight - view.getPaddingTop() - view.getPaddingBottom();
        cb.onSizeReady(paddingAdjustedWidth, paddingAdjustedHeight);
      } else {
        // We want to notify callbacks in the order they were added and we only expect one or two
        // callbacks to
        // be added a time, so a List is a reasonable choice.
        if (!cbs.contains(cb)) {
          cbs.add(cb);
        }
        if (layoutListener == null) {
        //这是尺寸大小的监听器
          final ViewTreeObserver observer = view.getViewTreeObserver();          
          layoutListener = new SizeDeterminerLayoutListener(this);
          observer.addOnPreDrawListener(layoutListener);
        }
      }
    }
```
这里加了ViewTreeObserver用来监听view尺寸大小，我们可以看看SizeDeterminerLayoutListener做了什么。

```
private static class SizeDeterminerLayoutListener implements ViewTreeObserver
        .OnPreDrawListener {
      private final WeakReference<SizeDeterminer> sizeDeterminerRef;

      public SizeDeterminerLayoutListener(SizeDeterminer sizeDeterminer) {
        sizeDeterminerRef = new WeakReference<>(sizeDeterminer);
      }

      @Override
      public boolean onPreDraw() {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
          Log.v(TAG, "OnGlobalLayoutListener called listener=" + this);
        }
        SizeDeterminer sizeDeterminer = sizeDeterminerRef.get();
        if (sizeDeterminer != null) {
        // 通知SizeDeterminer去重新检查尺寸，并触发后续操作。
          sizeDeterminer.checkCurrentDimens();
        }
        return true;
      }
    }
```
看完这个getsize，我们再返回到上面begin中的onSizeReady(overrideWidth, overrideHeight)函数，这个才是真正加载的函数。主要是engine发起的load操作，如下：

```
@Override
  public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
    if (status != Status.WAITING_FOR_SIZE) {
      return;
    }
    status = Status.RUNNING;

    float sizeMultiplier = requestOptions.getSizeMultiplier();
    this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
    this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
    }
    loadStatus = engine.load(
        glideContext,
        model,
        requestOptions.getSignature(),
        this.width,
        this.height,
        requestOptions.getResourceClass(),
        transcodeClass,
        priority,
        requestOptions.getDiskCacheStrategy(),
        requestOptions.getTransformations(),
        requestOptions.isTransformationRequired(),
        requestOptions.getOptions(),
        requestOptions.isMemoryCacheable(),
        requestOptions.getUseUnlimitedSourceGeneratorsPool(),
        this);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
  }
```
那么这个engine从哪里来的呢？它在glide初始化的时候，就创建了。

```
if (engine == null) {
      engine = new Engine(memoryCache, diskCacheFactory, diskCacheExecutor, sourceExecutor);
    }
```
包括参数内存缓存和磁盘缓存。然后我们看一下load具体实现，这里代码比较多，所以我就挑了几个重要的看了一下：

```
//给每次加载资源创建一个key，作为唯一的标识。
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);

//通过key load缓存资源，这是一级内存缓存，是从内存缓存中直接拿出来的
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
         cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }

//这里是二级内存缓存，使用Map<Key, WeakReference<EngineResource<?>>>保存起来的
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }

//根据key获取缓存的任务
    EngineJob<?> current = jobs.get(key);
    if (current != null) {
      current.addCallback(cb);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }

//创建任务
    EngineJob<R> engineJob = engineJobFactory.build(key, isMemoryCacheable,
        useUnlimitedSourceExecutorPool);
    DecodeJob<R> decodeJob = decodeJobFactory.build(
        glideContext,
        model,
        key,
        signature,
        width,
        height,
        resourceClass,
        transcodeClass,
        priority,
        diskCacheStrategy,
        transformations,
        isTransformationRequired,
        options,
        engineJob);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb);
    //放入线程池，执行
    engineJob.start(decodeJob);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
```
还有一个问题，这个二级缓存，是谁放入二级内存中呢，我们看一下上面一级缓存的loadFromCache方法。

```
 private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
    }
    return cached;
  }

```
这里把带有key的缓存存到activeResources中，二级缓存中loadFromActiveResources函数，也是从activeResources中拿到资源，如下：

```

  private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> active = null;
    //从activeResources中拿到key的cache
    WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
    if (activeRef != null) {
      active = activeRef.get();
      if (active != null) {
        active.acquire();
      } else {
        activeResources.remove(key);
      }
    }

    return active;
  }
```
除此之外呢?
1、一级内存缓存，glide中默认的就是我们常用的LruResourceCache。
2、我们说glide是加载imageview，我们从哪里知道的呢，还记得使用的时候into()函数中我们传入的是imageview，而我们源码中的target就是我们传进来的imageview。在track函数中

```
 public void track(Target<?> target) {
    targets.add(target);
  }
```
我们将target也就是我们的imageview加入到缓存中去了。
3、为何要两级内存缓存（loadFromActiveResources）。从资料上看猜测可能是一级缓存采用LRU算法进行缓存，不一定每个都能缓存到，添加二级缓存 可以保证每个都能缓存到；
4、EngineJob和DecodeJob各自职责：EngineJob充当了管理和调度者，主要负责加载和各类回调通知；DecodeJob是真正干活的劳动者，这个类实现了Runnable接口。


那我们来看一下真正线程(DecodeJob)里面是怎么执行的？
我怕大家忘了engine.load的代码，所以我先把使用EngineJob和DecodeJob的代码贴在下面：
```
//创建任务
    EngineJob<R> engineJob = engineJobFactory.build(key, isMemoryCacheable,
        useUnlimitedSourceExecutorPool);
    DecodeJob<R> decodeJob = decodeJobFactory.build(
        glideContext,
        model,
        key,
        signature,
        width,
        height,
        resourceClass,
        transcodeClass,
        priority,
        diskCacheStrategy,
        transformations,
        isTransformationRequired,
        options,
        engineJob);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb);
    //放入线程池，执行
    engineJob.start(decodeJob);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
```

然后我们在继续看一下DecodeJob是如何执行的：

```
@Override
  public void run() {
    // This should be much more fine grained, but since Java's thread pool implementation silently
    // swallows all otherwise fatal exceptions, this will at least make it obvious to developers
    // that something is failing.
    try {
      if (isCancelled) {
        notifyFailed();
        return;
      }
      runWrapped();
    } catch (RuntimeException e) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "DecodeJob threw unexpectedly"
            + ", isCancelled: " + isCancelled
            + ", stage: " + stage, e);
      }
      // When we're encoding we've already notified our callback and it isn't safe to do so again.
      if (stage != Stage.ENCODE) {
        notifyFailed();
      }
      if (!isCancelled) {
        throw e;
      }
    }
  }
```

然后在看里面的runWrapped函数，如下

```

  private void runWrapped() {
     switch (runReason) {
      case INITIALIZE:
      //初始化，获取下一个阶段的状态
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        //运行
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }
```

先看getNextStage方法：

```
// 这里的阶段策略首先是从resource中寻找，然后再是data，，再是source
private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
      // 根据定义的缓存策略来回去下一个状态
	  // 缓存策略来之于RequestBuilder的requestOptions域
	  // 如果你有自定义的策略，可以调用RequestBuilder.apply方法即可
	  // 详细的可用缓存策略请参看DiskCacheStrategy.java
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        return Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }

```

再看getNextGenerator函数，如下

```
// 根据Stage找到数据抓取生成器。
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
       // 产生含有降低采样/转换资源数据缓存文件的DataFetcher。
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
      // 产生包含原始未修改的源数据缓存文件的DataFetcher。
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
      // 生成使用注册的ModelLoader和加载时提供的Model获取源数据规定的DataFetcher。
      // 根据不同的磁盘缓存策略，源数据可首先被写入到磁盘，然后从缓存文件中加载，而不是直接返回。
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }
```

经过上面的流程，最后就是发起实际请求的地方了，SourceGenerator.startNext()方法。

```
@Override
  public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
          || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
  }
```
这里的Model必须是实现了GlideModule接口的，fetcher是实现了DataFetcher接口。有兴趣同学可以继续看一下integration中的okhttp和volley工程。Glide主要采用了这两种网络libray来下载图片。

数据下载完成后缓存处理SourceGenerator.onDataReady

```
 @Override
  public void onDataReady(Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      // We might be being called back on someone else's thread. Before doing anything, we should
      // reschedule to get back onto Glide's thread.
      cb.reschedule();
    } else {
      cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
          loadData.fetcher.getDataSource(), originalKey);
    }
  }
```

这里将下载的data存到磁盘cache中，但是咋就一句dataToCache = data，其实是在cb.reschedule()，cb就是decodeJob。

```
public void reschedule() {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
  }
```

这里又有一个Callback，继续追踪，这里的Callback接口是定义在DecodeJob内的，而实现是在外部的Engine中（这里会用线程池重新启动当前job，那为什么要这样做呢？源码中的解释是为了不同线程的切换，因为下载都是借用第三方网络库，而实际的编解码是在Glide自定义的线程池中进行的）

```
public void reschedule(DecodeJob<?> job) {
  if (isCancelled) {
    MAIN_THREAD_HANDLER.obtainMessage(MSG_CANCELLED, this).sendToTarget();
  } else {
    sourceExecutor.execute(job);
  }
}
```

接下来继续DecodeJob.runWrapped()方法。这个时候的runReason是SWITCH_TO_SOURCE_SERVICE，因此直接执行runGenerators()，这里继续执行SourceGenerator.startNext()方法，值得注意的dataToCache域，因为上一次执行的时候是下载，因此再次执行的时候内存缓存已经存在，因此直接缓存数据cacheData(data)：

```
  private void cacheData(Object dataToCache) {
    long startTime = LogTime.getLogTime();
    try {
      Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
      DataCacheWriter<Object> writer =
          new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
      originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
      helper.getDiskCache().put(originalKey, writer);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Finished encoding source to cache"
            + ", key: " + originalKey
            + ", data: " + dataToCache
            + ", encoder: " + encoder
            + ", duration: " + LogTime.getElapsedMillis(startTime));
      }
    } finally {
      loadData.fetcher.cleanup();
    }

    sourceCacheGenerator =
        new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
  }
```

我们在会到SourceGenerator.startNext()方法，这个时候已经有了sourceCacheGenerator，那么直接执行DataCacheGenerator.startNext()方法：

```
 @Override
  public boolean startNext() {
    while (modelLoaders == null || !hasNextModelLoader()) {
      sourceIdIndex++;
      if (sourceIdIndex >= cacheKeys.size()) {
        return false;
      }

      Key sourceId = cacheKeys.get(sourceIdIndex);
      Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
      cacheFile = helper.getDiskCache().get(originalKey);
      if (cacheFile != null) {
        this.sourceKey = sourceId;
        modelLoaders = helper.getModelLoaders(cacheFile);
        modelLoaderIndex = 0;
      }
    }

    loadData = null;
    boolean started = false;
    // 这里会通过model寻找注册过的ModelLoader
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      loadData =
          modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(),
              helper.getOptions());
              // 通过FileLoader继续加载数据
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
  }
```

这里的ModelLoader跟之前提到过的Register的模块加载器（ModelLoader）对应是modelLoaderRegistry域，具体执行的操作是Registry.getModelLoaders(…)方法如下：

```
public <Model> List<ModelLoader<Model, ?>> getModelLoaders(Model model) {
   List<ModelLoader<Model, ?>> result = modelLoaderRegistry.getModelLoaders(model);
   if (result.isEmpty()) {
     throw new NoModelLoaderAvailableException(model);
   }
   return result;
 }
```

继续回到DataCacheGenerator.startNext()方法，找到了ModelLoader，然后跟踪到的是FileLoader类(FileFetcher.loadData(…)方法)：

```
public void loadData(Priority priority, DataCallback<? super Data> callback) {
	// 读取文件数据
     try {
       data = opener.open(file);
     } catch (FileNotFoundException e) {
       if (Log.isLoggable(TAG, Log.DEBUG)) {
         Log.d(TAG, "Failed to open file", e);
       }
	//失败
       callback.onLoadFailed(e);
       return;
     }
  // 成功
     callback.onDataReady(data);
   }
```

**装载流程**

代码太多，实在有点绕晕了，所以就将一位大神的博客里面装载流程粘了过来。
主要线路如下：
-->DataCacheGenerator.onDataReady
  -->SourceGenerator.onDataFetcherReady
    -->DecodeJob.onDataFetcherReady
	-->DecodeJob.decodeFromRetrievedData
	-->DecodeJob.notifyEncodeAndRelease
	-->DecodeJob.notifyComplete
	  -->EngineJob.onResourceReady
	  
需要说明的就是在EngineJob中有一个Handler叫MAIN_THREAD_HANDLER。为了实现在主UI中装载资源的作用，ok继续上边的流程：

-->EngineJob.handleResultOnMainThread
  -->SingleRequest.onResourceReady
    -->ImageViewTarget.onResourceReady
    -->ImageViewTarget.setResource
      -->ImageView.setImageDrawable/ImageView.setImageBitmap

数据的装载过程中有一个很重要的步骤就是decode，这个操作发生在DecodeJob.decodeFromRetrievedData的时候，继续看代码：

```
private void decodeFromRetrievedData() {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Retrieved data", startFetchTime,
          "data: " + currentData
          + ", cache key: " + currentSourceKey
          + ", fetcher: " + currentFetcher);
    }
    Resource<R> resource = null;
    try {
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      exceptions.add(e);
    }
    if (resource != null) {
      notifyEncodeAndRelease(resource, currentDataSource);
    } else {
      runGenerators();
    }
  }
```

这中间发生了很多转换主要流程：

-->DecodeJob.decodeFromData
-->DecodeJob.decodeFromFetcher
-->DecodeJob.runLoadPath
  -->LoadPath.load
  -->LoadPath.loadWithExceptionList
  -->LoadPath.decode
  -->LoadPath.decodeResource
  -->LoadPath.decodeResourceWithList
    -->ResourceDecoder.handles
    -->ResourceDecoder.decode

这里讲到了decode，那么encode发生在什么时候呢？直接通过Encoder接口调用发现，在数据缓存的时候才会触发编码。具体调用在DiskLruCacheWrapper和DataCacheWriter中。一些值得参考的写法例如BitmapEncoder对Bitmap的压缩处理。

## 结束语 ##

**总结，glide库很大，自己在看源码的时候，也参考了别人对glide的理解，将glide的大概流程的主要源码看了一下，其中可能会有很多理解不对或者不足的地方，希望大家能给指出来。**
下面是是一位大神看完glide之后的结束语，个人觉得还是很不错的。

1、总体来说代码写的挺漂亮的，单从使用者角度来说入手是比较容易的。
2、源码使用了大量的工厂方法来创建对象，就像String.valueof(…)方法一样，这也体现编码的优雅。
3、不过想要对这个库进行改造，可能并非易事，笔者在跟踪代码的过程中发现很多地方有Callback这样的接口，来来回回查找几次很容易就晕头转向了。。。
另外一个感觉难受的地方就是构造方法带入参数太多，就拿SingleRequest来说就是12个构造参数。
4、单例的使用感觉还是有些模糊，就比如GlideContext，有些时候通过Glide.get(context).getGlideContext()获取，而有些类中采用构造传入。个人觉得既然让Glide作为单例，那么还这样传入参数是不是有点多余？代码的编写都是可以折中考虑，不过如果整个项目拟定好了一个规则的话，我想最好还是遵循它。另外再吐槽一下单例，很多开发人员喜欢用单例，如果你是有代码洁癖的开发者，那么你肯定很讨厌这样，单例很容易造成代码的散落和结构不清晰。

