---
title: Android WebView深入浅出
date: 2016-09-05 20:27:18
tags: [android,webview,布局,dp]
categories: Android
---

**一、基本使用**
首先在layout中一个简单的空间

```
<WebView
        android:id="@+id/webView1"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:layout_marginTop="10dp" />
```
同时，因为要访问网络，所以在manifest中必须要加网络访问的权限。

```
<uses-permission android:name="android.permission.INTERNET"/>
```
在activity中即可获得webview的引用，同时load一个网址。

```
webview = (WebView) findViewById(R.id.webView1);
webview.loadUrl("http://www.baidu.com/");
```
这个时候，启动应用后，自动打开了系统内置的浏览器，所以要为这个webview设置webviewclient，并重写方法。

```
webview.setWebViewClient(new WebViewClient(){
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }
        });
```
若自己定义一个页面加载进度的progressBar，通过以下方式来获取webview内加载进度：

```
webview.setWebChromeClient(new WebChromeClient(){
            @Override
            public void onProgressChanged(WebView view, int newProgress) {
                //get the newProgress and refresh progress bar
            }
        });
```
当然每个页面都有一个标题，js会设置标题，那我们原生的native如何添加title呢？

```
webview.setWebChromeClient(new WebChromeClient(){
            @Override
            public void onReceivedTitle(WebView view, String title) {
                titleview.setText(title);//a textview
            }
        });
```

**二、通过webview下载文件**
通常webview渲染的界面中含有可以下载文件的链接，点击该链接后，应该开始执行下载的操作并保存文件到本地中。webview来下载页面中的文件通常有两种方式：

1. 自己通过一个线程写java io的代码来下载和保存文件（可控性好）
2. 调用系统download的模块（代码简单）

方法一：
先写一个download线程类。

```
public class HttpThread extends Thread {


    private String mUrl;

    public HttpThread(String mUrl) {
        this.mUrl = mUrl;
    }
    
    @Override
    public void run() {
        URL url;
        try {
            url = new URL(mUrl);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setDoInput(true);
            conn.setDoOutput(true);
            InputStream in = conn.getInputStream();
            
            File downloadFile;
            File sdFile;
            FileOutputStream out = null;
            if(Environment.getExternalStorageState().equals(Environment.MEDIA_UNMOUNTED)){
                downloadFile = Environment.getExternalStorageDirectory();
                sdFile = new File(downloadFile, "test.file");
                out = new FileOutputStream(sdFile);
            }
            
            //buffer 4k
            byte[] buffer = new byte[1024 * 4];
            int len = 0;
            while((len = in.read(buffer)) != -1){
                if(out != null)
                    out.write(buffer, 0, len);
            }
            
            //close resource
            if(out != null)
                out.close();
            
            if(in != null){
                in.close();
            }
            
            
            
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
    
}
```

然后要实现一个DownloadLisener接口，这个接口中实现OnDownloadStart方法，用户点击下载链接的时候，这个回调方法会调用传进来的URL，然后将URL放到我们上面写的HttpThread进行下载：

```
class MyDownloadListenter implements DownloadListener{

        @Override
        public void onDownloadStart(String url, String userAgent,
                String contentDisposition, String mimetype, long contentLength) {
            System.out.println("url ==== >" + url);
            new HttpThread(url).start();
        }
        
    }

//给webview加入监听
webview.setDownloadListener(new MyDownloadListenter());
```

方法二：
直接发送一个action_view的intent就可以了：

```
class MyDownloadListenter implements DownloadListener{

        @Override
        public void onDownloadStart(String url, String userAgent,
                String contentDisposition, String mimetype, long contentLength) {
            System.out.println("url ==== >" + url);
            //new HttpThread(url).start();
            
            Uri uri = Uri.parse(url);
            Intent intent = new Intent(Intent.ACTION_VIEW, uri);
            startActivity(intent);
        }
        
    }
```

**错误处理**
当我们使用浏览器的时候，通常因为加载的页面的服务器的各种原因导致各种出错的情况，最平常的比如404错误，通常情况下浏览器会提示一个错误提示页面。事实上这个错误提示页面是浏览器在加载了本地的一个页面，用来提示用户目前已经出错了。

但是当我们的app里面使用webview控件的时候遇到了诸如404这类的错误的时候，若也显示浏览器里面的那种错误提示页面就显得很丑陋了，那么这个时候我们的app就需要加载一个本地的错误提示页面，即webview如何加载一个本地的页面。

1. 首先我们需要些一个html文件，比如error_handle.html，这个文件里面就是当出错的时候需要展示给用户看的一个错误提示页面，尽量做的精美一些。然后将该文件放置到代码根目录的assets文件夹下。

2. 随后我们需要复写WebViewClient的onRecievedError方法，该方法传回了错误码，根据错误类型可以进行不同的错误分类处理

```
webview.setWebViewClient(new WebViewClient(){
       
            @Override
            public void onReceivedError(WebView view, int errorCode,
                    String description, String failingUrl) {
                switch(errorCode)
                {
                case HttpStatus.SC_NOT_FOUND:
                    view.loadUrl("file:///android_assets/error_handle.html");
                    break;
                }
            }
        });
```
其实，当出错的时候，我们可以选择隐藏掉webview，而显示native的错误处理控件，这个时候只需要在onReceivedError里面显示出错误处理的native控件同时隐藏掉webview即可。

**四、webview与js交互**
1、webview调用js

```
mWebView.loadUrl("javascript:do()");
```
比如这个中webview调用的是do方法，js在html大概如下：

```
<html>
    <script language="javascript">
        /* This function is invoked by the webview*/
        function do() {
            alert("1");
        }
    </script>
    <body>
        <a onClick="window.demo.clickOnAndroid()"><div style="width:80px;
            margin:0px auto;
            padding:10px;
            text-align:center;
            border:2px solid #111111;" >
                <img id="droid" src="xx.png"/><br>
                Click me!
        </div></a>
    </body>
</html>
```
2、js调用webview
假设我们某些方法要给js调用。

```
class DemoJavaScriptInterface {

        DemoJavaScriptInterface() {
        }

        /**
         * This is not called on the UI thread. Post a runnable to invoke
         * loadUrl on the UI thread.
         */
        public void clickOnAndroid() {
            mHandler.post(new Runnable() {
                public void run() {
                    //TODO
                }
            });

        }
    }
```
首先给webview设置：

```
mWebview.setJavaScriptEnabled(true);
```
然后将本地类映射出去

```
mWebView.addJavascriptInterface(new DemoJavaScriptInterface(), "demo");
```
这样就可以在js中直接调用本地java的方法。

**五、webview和js互相调用混淆**
若webview中的js调用了本地的方法，正常情况下发布的debug包js调用的时候是没有问题的，但是通常发布商业版本的apk都是要经过混淆的步骤，这个时候会发现之前调用正常的js却无法正常调用本地方法了。

这是因为混淆的时候已经把本地的代码的引用给打乱了，导致js中的代码找不到本地的方法的地址。

解决这个问题很简单，即在proguard.cfg文件中加上一些代码，声明本地中被js调用的代码不被混淆。下面举例说明：

第四部分中被js调用的那个类DemoJavaScriptInterface的包名为com.test.webview，那么就要在proguard.cfg文件中加入：

```
-keep public class com.test.webview.DemoJavaScriptInterface{
    public <methods>;
}
```
若是内部类，如下：

```
-keep public class com.test.webview.DemoJavaScriptInterface$InnerClass{
    public <methods>;
}
```
若是android比较新，需要添加如下：

```
-keepattributes *Annotation*  
-keepattributes *JavascriptInterface*
```

**六、webview同步cookies**
cookies是服务器用来保存每个客户的常用信息的，下次客户进入一个诸如登陆的页面时服务器会检测cookie信息，如果通过则直接进入登陆后的页面。

在webview中，如果之前已经登陆过了，那么下次再进入同样的登陆界面时，若需要再次登陆的话，一定会很恼人，所以这里提供一个webview同步cookies的方法。

 

1.首先，我们假设某个网站的登陆界面需要提供两个参数，一个是name，一个是pwd，那么要是对这个页面进行登陆，那么必须给与这两个信息。我们假设服务器已经注册了name为jason，pwd为123456这个账号。

2.下面，写一个Thread用来将name和pwd自动的登入，在服务器返回的response中获得cookie信息，稍后对这个cookie进行保存，这里先给出这个Thread的代码：

```
public class HttpCookie extends Thread {

    private Handler mHandler;

    public HttpCookie(Handler mHandler) {
        this.mHandler = mHandler;
    }
    
    @Override
    public void run() {
        HttpClient client = new DefaultHttpClient();
        HttpPost post = new HttpPost("");//this place should add the login address
        
        List<NameValuePair> list = new ArrayList<NameValuePair>();
        list.add(new BasicNameValuePair("name", "jason"));
        list.add(new BasicNameValuePair("pwd", "123456"));
        
        try {
            post.setEntity(new UrlEncodedFormEntity(list));
            HttpResponse reponse = client.execute(post);
            if(reponse.getStatusLine().getStatusCode() == HttpStatus.SC_OK){
                AbstractHttpClient absClient = (AbstractHttpClient) client;
                List<Cookie> cookies = absClient.getCookieStore().getCookies();
                
                for(Cookie cookie:cookies){
                    if(cookie != null){
                        //TODO
                        //this place would get the cookies
                    }
                }
            }
            
        } catch (UnsupportedEncodingException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } catch (ClientProtocolException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
    
}
```
由于这是一个子线程，所以需要在主线程中创建并执行。

同时，因为其实子线程，那么里面必须含有一个handler的元素，用来当成功获取cookie后通知主线程进行同步和保存。初始化这个子线程的时候需要将主线程上的handler给传过来，随后在以上代码的TODO中发送消息，让主线程记录cookie，发送的这个消息需要将cookie信息包含进去：

```
if(cookie != null){
    //TODO
    //this place would get the cookies
    Message msg = new Message();
    msg.obj = cookie;
    if(mHandler != null){
        mHandler.sendMessage(msg);
        return;
    }
}
```
随后在主线程中（webview加载登陆界面前），在handler中将会获取到cookie信息，下面将对该cookie进行保存和同步：

```
 private Handler mHandler = new Handler(){
        public void handleMessage(android.os.Message msg) 
        {
        CookieSyncManager.createInstance(MainActivity.this);
            CookieManager cookieMgr = CookieManager.getInstance();
            cookieMgr.setAcceptCookie(true);
            cookieMgr.setCookie("", msg.obj.toString());// this place should add the login host address(not the login index address)
            CookieSyncManager.getInstance().sync();
            
            webview.loadUrl("");// login index address
        };
    };
```
这个时候发现webview加载的login index页面中可以自动的登陆了并显示登陆后的界面。