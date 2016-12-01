---
title: Volley源码解析(一)
date: 2016-06-27 19:30:18
tags: [volley,源码,内存,设计,框架]
categories: Android
---

这个章节，我先介绍volley功能介绍、总体设计、类设计、主要流程的源码解析，其他方法，例如图片加载、HurlStack、HttpClientStack、performRequest等，我会在后面进行介绍。

**1、功能介绍**
1.1 volley是异步网络请求框架和图片加载器。
1.2 优点
       1、请求队列和请求优先级。
       2、请求Cache和内存管理。
       3、扩展性强，大多是基于接口的设计，可配置型强。
       4、可以取消请求。
       5、提供简单的图片加载工具。
       
**2、总体设计**
2.1 总体设计图如下：
![这里写图片描述](http://img.blog.csdn.net/20160627191325478)
通过Dispatch Thread不断从RequestQueue中获取请求，根据是否已缓存调用Cache或者Network这两类数据获取接口其中之一，从内存缓存或者是服务器中获取数据，然后交给ResponseDelivery去做结果分发以及回调处理。

2.2 核心功能点概念
volley使用，通过newRequestQueue()函数新建并启动一个请求队列RequestQueue，然后不断向这个队列中添加Request就可以了。
Volley：对外暴露的API，通过newRequestQueue()函数新建并启动一个请求队列RequestQueue。
Request：表示一个请求的抽象类。StringRequest、JsonRequest、ImageRequest都是它的子类，表示某种类型的请求。
RequestQueue：请求队列，里面包含一个CacheDispatcher（缓存请求的线程）、多个NetworkDispatcher（处理网络请求的线程），start函数启动CacheDispatcher和NetworkDispatcher。
CacheDispatcher：用于处理缓存请求。不断从缓存请求队列中获取请求处理，队列为空的时候则等待，请求处理结束后将结果传递给ResponseDelivery去执行后续处理。当结果没有被缓存过、缓存失效等情况下，这个请求都要重新进入NetworkDispatcher去调度处理。
NetworkDispatcher：用于处理网络请求。不断从网络请求队列中获取请求处理，队列为空的时候则等待，请求结果传递给ResponseDelivery去执行后续处理，并判断结果是否要进行缓存。
ResponseDelivery：返回结果分发接口。
HttpStack：处理Http请求，返回请求结果。目前Volley中有基于HttpURLConnection的HurlStack和基于Apache HttpClient的HttpClientStack。
Network：调用HttpStack处理请求，将结果转化为可以被ResponseDelivery处理的NetworkResponse。
Cache：缓存请求结果，默认使用sdcard的DiskBasedCache。NetworkDispatcher得到结果后判断是否需要存在Cache，CacheDispatcher会从Cache中获取缓存结果。
2.3 流程图
![这里写图片描述](http://img.blog.csdn.net/20160627191431652)

3、类设计图
![这里写图片描述](http://img.blog.csdn.net/20160627191513387)
红色部分，围绕RequestQueue类，将各个功能点以组合
形式结合在一起。每个功能点(Request,CacheDispatcher,NetworkdDispatcher)，都是以接口或者抽象类的形式提供。红色外面的部分，为功能点提供具体实现。
多用组合，少用继承；针对接口编程，不针对具体实现编程。

**4、主要流程源码解析**
4.1.1
使用volley的时候，需要创建一个请求队列RequestQueue，最好在AppApplication中声明一个全局的RequestQueue，使用方法如下：

```
mRequestQueue = Volley.newRequestQueue(getApplicationContext());
```

我们看一下源码中Volley中的newRequestQueue方法，就是新建一个RequestQueue类对象，如下：

```
RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
queue.start();

```
这段代码，将缓存对象和网络对象加入到RequestQueue队列中（其中DiskBasedCache是Cache的具体实现类，BasicNetWork是NetWork的具体实现类），RequestQueue中第二个参数network实现如下：

```
if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
               stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
     }
Network network = new BasicNetwork(stack);

```
代码中可以看出来，当API>=9的时候，使用HurlStack，反之，使用HttpClientStack。关于HurlStack和HttpClientStack基于什么来实现的，我会在下一章中做详细介绍。

4.1.2
添加request，使用方法如下：

```
mRequestQueue.add(request);

```
我们看一下源码中RequestQueue中的add方法。

```
if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;}

```
首先判断当前请求是否可以缓存，如果不能直接添加到网络请求队列，如果可以缓存，就加入缓存队列。
然后判断等待请求mWaitingRequests和添加的request是否有相同的cachekey，如果有的话，将这个request加入具有相同cachekey的等待队列，如下：

```
stagedRequests.add(request);//  stagedRequests是具有cachekey的队列
mWaitingRequests.put(cacheKey, stagedRequests);//mWaitingRequests是            
                                              //map<String,Queue>类型

```
如果没有的话，就为cachekey添加一个为null的队列，加入缓存队列mCacheQueue，相当于在内存中重新添加这个request，代码如下：

```
mWaitingRequests.put(cacheKey, null);
mCacheQueue.add(request);

```
其中mCacheQueue和mNetworkQueue分别代表缓存请求队列和网络请求队列，mWaitingRequests是一个等待请求的集合，如果一个请求可以被缓存，并且有相同的cachekey，就可以全部进入此等待队列。

4.2 创建JSON请求
volley自带JsonObjectRequest和JsonArrayRequest来处理JSON对象请求和JSON数据请求。创建JsonObjectRequest对象，写好response回调接口，把这个请求放到请求队列中就可以了，使用方法如下(JsonArrayRequest使用方法也类似)：

```
JsonObjectRequest jsonObjReq = new JsonObjectRequest(Method.GET,url, null,
            new Response.Listener<JSONObject>() {
      	@Override
       	public void onResponse(JSONObject response) {
                Log.d(TAG, response.toString());
             }
        }, new Response.ErrorListener() {
       	@Override
        public void onErrorResponse(VolleyError error) {
               VolleyLog.d(TAG, "Error: " + error.getMessage());
             }
        });
mRequestQueue.add(jsonObjReq, tag_json_obj);

```
我们从源码分析JsonObjectRequest.java文件，JsonObjectRequest继承Response.java，就是一个结构体，里面包含了url，请求方式，onResponse和errorResponse回调。

4.3 创建String请求
volley中有StringRequest类，用来请求string类型的数据，使用方法如下：

```
StringRequest strReq = new StringRequest(Method.GET,
            url, new Response.Listener<String>() {
                @Override
                public void onResponse(String response) {
                    Log.d(TAG, response.toString());
                    pDialog.hide();
                }
            }, new Response.ErrorListener() {
                @Override
                public void onErrorResponse(VolleyError error) {
                    VolleyLog.d(TAG, "Error: " + error.getMessage());
                    pDialog.hide();
                }
            });
mRequestQueue.add(strReq, tag_json_obj);

```
StringRequest.java跟JsonRequest结构一样。

4.4 创建POST请求
上面都是GET请求，下面是POST请求，需要将请求类型改为POST类型，并且override Request中的getParams方法就可以了，使用方法如下：

```
JsonObjectRequest jsonObjReq = new JsonObjectRequest(Method.POST,
            url, null,
            new Response.Listener<JSONObject>() {
                @Override
                public void onResponse(JSONObject response) {
                    Log.d(TAG, response.toString());
                    pDialog.hide();
                }
            }, new Response.ErrorListener() {
                @Override
                public void onErrorResponse(VolleyError error) {
                    VolleyLog.d(TAG, "Error: " + error.getMessage());
                    pDialog.hide();
                }
            }) {

        @Override
        protected Map<String, String> getParams() {
            Map<String, String> params = new HashMap<String, String>();
            params.put("name", "Androidhive");
            params.put("email", "abc@androidhive.info");
            params.put("password", "password123");
            return params;
        }
    };

```
这里通过重写Request.java中的getParams方法，将POST中的参数传入到request中，类型是Map<String, String>。

4.5 添加头部信息
override Request中的getHeaders方法就可以了。

```
JsonObjectRequest jsonObjReq = new JsonObjectRequest(Method.POST,url, null,new Response.Listener<JSONObject>() {
    @Override
    public void onResponse(JSONObject response) {
        Log.d(TAG, response.toString());
        pDialog.hide();
    }
}, new Response.ErrorListener() {
    @Override
    public void onErrorResponse(VolleyError error) {
        VolleyLog.d(TAG, "Error: " + error.getMessage());
        pDialog.hide();
    }
}) {

@Override
public Map<String, String> getHeaders() throws AuthFailureError {
    HashMap<String, String> headers = new HashMap<String, String>();
    headers.put("Content-Type", "application/json");
    headers.put("apiKey", "xxxxxxxxxxxxxxx");
    return headers;
}
};

```
这里通过重写Request.java中的getHeaders方法，将POST中的参数传入到request中，类型是Map<String, String>。

4.7 Volley中的Cache机制
1）从请求cache中加载请求

```
Cache=mRequestQueue.getCache();
Entry entry = cache.get(url);
if(entry != null){
    try {
        String data = new String(entry.data, "UTF-8");
    } catch (UnsupportedEncodingException e) {      
        e.printStackTrace();
        }
    }
}else{
    // Cached response doesn't exists. Make network call here
}

```
2）请求缓存失效

```
mRequestQueue.getCache().invalidate(url, true);

```
3）关闭cache

```
StringRequest stringReq = new StringRequest(....);
stringReq.setShouldCache(false);//true就是打开cache

```
4）将URL的cache删除

```
mRequestQueue.getCache().remove(url);

```
5）删除所有cache

```
mRequestQueue.getCache().clear();

```
4.8 请求优先级
优先级分为：Normal, Low, Immediate, High

```
private Priority priority = Priority.HIGH;
StringRequest strReq = new StringRequest(Method.GET,
            Const.URL_STRING_REQ, new Response.Listener<String>() {
                @Override
                public void onResponse(String response) {
                    Log.d(TAG, response.toString());
                    msgResponse.setText(response.toString());
                    hideProgressDialog();

                }
            }, new Response.ErrorListener() {
                @Override
                public void onErrorResponse(VolleyError error) {
                    VolleyLog.d(TAG, "Error: " + error.getMessage());
                    hideProgressDialog();
                }
            }) {
        @Override
        public Priority getPriority() {
            return priority;
        }
    };

```
4.9 取消请求
addToRequestQueue(request, tag)方法还接受一个tag参数，这个tag就是用来标记某一类请求的，这样就可以取消这个tag的所有请求了。

```
mRequestQueue.cancelAll("feed_request");

```
4.10 RequestQueue的start方法
源码如下：

```
public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }

```
CacheDispatcher是缓存分发器，只有一个，NetworkDispatcher是网络分发器，有多个，然后我再去看一下CacheDispatcher分发器中的start方法，CacheDispatcher继承的Thread，所以直接看run方法，代码如下（注释我写在代码上面）：

```
public void run() {
        if (DEBUG) VolleyLog.v("start new dispatcher");
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // Make a blocking call to initialize the cache.
        mCache.initialize();

        while (true) {
            try {               
                final Request<?> request = mCacheQueue.take();
                request.addMarker("cache-queue-take");
                
                if (request.isCanceled()) {
                    request.finish("cache-discard-canceled");
                    continue;
                }

                // Attempt to retrieve this item from cache.
                //尝试从缓存中获取响应结果
                Cache.Entry entry = mCache.get(request.getCacheKey());
                //如果缓存中获取结果为空的话,就把这个请求放到网络请求队列中
                if (entry == null) {
                    request.addMarker("cache-miss");
                    // Cache miss; send off to the network dispatcher.
                    mNetworkQueue.put(request);
                    continue;
                }
                
                //如果缓冲结果不为空的话,判断缓存是否过期
                //如果过期的话,将这个请求放到网络请求队列中.
                if (entry.isExpired()) {
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }
                //直接使用缓存中的数据
                request.addMarker("cache-hit");
                //parseNetworkResponse对数据进行解析
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");

                //将解析出来的数据进行回调.
                if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);
                    response.intermediate = true;
                    
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {                
                if (mQuit) {
                    return;
                }
                continue;
            }
        }

```
其中cache结构体中的内容，如上面介绍的cache机制的方法向cache中添加信息和内容。
同理，networkDispatcher中run方法源码如下：

```
 public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            Request<?> request;
            try {
                request = mQueue.take();
            } catch (InterruptedException e) {
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }
                addTrafficStatsTag(request);
                //发送网络请求
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker(“network-http-complete");

                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

            // 在子线程中解析，在Request子类中parseNetworkResponse方法中实现解析
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker(“network-parse-complete");

               if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }
                // response回调.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }

```
代码比较多，但是真正网络请求的核心代码应该是

```
NetworkResponse networkResponse = mNetwork.performRequest(request);

```
最核心的是performRequest方法，具体代码分析我放到下一章节。

