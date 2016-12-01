---
title: Volley源码解析(二)
date: 2016-07-02 18:05:18
tags: [volley,函数,源码,网络,线程]
categories: Android
---

上面一个章节，从volley的主要使用流程来大概分析了一下代码，这一章将详细的网络请求流程从源码角度分析一波。

前面一章讲了NetworkDispatcher.java，这个线程主要任务就是网络请求和处理，其中run()函数中执行网络请求的代码如下：

```
NetworkResponse networkResponse = mNetwork.performRequest(request);

```
performRequest函数的实现，我们看一下BasicNetwork中performRequest函数（其中BasicNetwork.java是Network的实现类），代码很多，我截取了一部分核心的代码片段：

```
Map<String, String> headers = new HashMap<String, String>();
addCacheHeaders(headers, request.getCacheEntry());
//网络请求
httpResponse = mHttpStack.performRequest(request, headers);
StatusLine statusLine = httpResponse.getStatusLine();
int statusCode = statusLine.getStatusCode();

responseHeaders = convertHeaders(httpResponse.getAllHeaders());
if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
      Entry entry = request.getCacheEntry();
      if (entry == null) {
             return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                                responseHeaders, true,
                                SystemClock.elapsedRealtime() - requestStart);
       }
       entry.responseHeaders.putAll(responseHeaders);
       return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                            entry.responseHeaders, true,
                            SystemClock.elapsedRealtime() - requestStart);
       }
       if (httpResponse.getEntity() != null) {
            responseContents = entityToBytes(httpResponse.getEntity());
       }else {
             responseContents = new byte[0];
       }
       long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
       logSlowRequests(requestLifetime, request, responseContents, statusLine);
       if (statusCode < 200 || statusCode > 299) {
            throw new IOException();
       }
return new NetworkResponse(statusCode, responseContents, responseHeaders,   false,SystemClock.elapsedRealtime() - requestStart);
```
代码比较多，前一部分代码就是做网络请求，后面一部分是对网络请求返回的response进行处理（包括head和entity）。网络请求的代码就一行：

```
httpResponse = mHttpStack.performRequest(request, headers);
```
mHttpStack就是一个接口，它有两个实现类，就是我们上一章介绍的RequestQueue，当API>=9的时候，使用HurlStack，反之，使用HttpClientStack。这两个stack就是HttpStack的具体实现类。

**1、我们先看一下HurlStack实现performRequest方法。**

1.1 首先获取request中的url。

```
String url = request.getUrl();
URL parsedUrl = new URL(url);
```

1.2 打开http连接。

```
HttpURLConnection connection = openConnection(parsedUrl, request);
```
openConnection实现方法，代码如下：

```
HttpURLConnection connection = createConnection(url);
int timeoutMs = request.getTimeoutMs();
connection.setConnectTimeout(timeoutMs);
connection.setReadTimeout(timeoutMs);
connection.setUseCaches(false);
connection.setDoInput(true);
```
如代码所示，openConnection方法使用系统的HttpURLConnection类，设置timeout，缓存等参数，然后return。

1.3 打开网络连接了，我们继续看performRequest方法。

```
setConnectionParametersForRequest(connection, request);
```
如函数名字意思一样，就是为request设置一些parameters，我们看一下这个函数。

```
                .......
                
case Method.GET:
     connection.setRequestMethod("GET");
     break;
case Method.DELETE:
     connection.setRequestMethod("DELETE");
     break;
case Method.POST:
     connection.setRequestMethod("POST");
     addBodyIfExists(connection, request);
     break;
     
               ........
```
就是设置一些请求的方法。

1.4 返回code，报错处理

```
int responseCode = connection.getResponseCode();
if (responseCode == -1) {
  throw new IOException("Could not retrieve response code from HttpUrlConnection.");
}
```
1.5 最后返回response

```
return response;
```

**2、我们再看一下HttpClientStack的实现方法**

```
HttpUriRequest httpRequest = createHttpRequest(request, additionalHeaders);
addHeaders(httpRequest, additionalHeaders);
addHeaders(httpRequest, request.getHeaders());
onPrepareRequest(httpRequest);
HttpParams httpParams = httpRequest.getParams();
int timeoutMs = request.getTimeoutMs();
HttpConnectionParams.setConnectionTimeout(httpParams, 5000);
HttpConnectionParams.setSoTimeout(httpParams, timeoutMs);
return mClient.execute(httpRequest);
```
这里创建的网络请求是使用HttpClient中HttpUriRequest包来做网络请求，下面类似于HurlStack，用来添加参数。

好啦，介绍好了两个实现类（HttpClientStack和HurlStack）中performRequest的实现方法，我们在继续BasicNetwork的performRequest方法。

```
responseHeaders = convertHeaders(httpResponse.getAllHeaders());
if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
Entry entry = request.getCacheEntry();
if (entry == null) {
   return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                                responseHeaders, true,
                                SystemClock.elapsedRealtime() - requestStart);
}
                   
entry.responseHeaders.putAll(responseHeaders);
return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                            entry.responseHeaders, true,
                            SystemClock.elapsedRealtime() - requestStart);
}

```
把request的head和Entry进行包装。

```
if (httpResponse.getEntity() != null) {
    responseContents = entityToBytes(httpResponse.getEntity());
} else {
    responseContents = new byte[0];
}
                
long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
logSlowRequests(requestLifetime, request, responseContents, statusLine);
return new NetworkResponse(statusCode, responseContents, responseHeaders,   false,SystemClock.elapsedRealtime() - requestStart);

```
最后return新的response。

现在网络请求已经完成，下面就应该开一个线程，把这个网络请求跑起来啦。我们再回到networkDispatcher.java中。

```
mDelivery.postResponse(request, response);

```
我们看一下ResponseDelivery实现类ExecutorDelivery中的postResponse方法。

```
request.markDelivered();
request.addMarker("post-response");
mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
```
这个方法，把request网络请求放到ResponseDeliveryRunnable中。

这大概就是我理解的volley网络请求的详细流程，如果有哪里不对的地方，希望大神们指点一二。