---
title: Weex初学之旅
date: 2016-06-27 19:30:18
tags: [weex,JavaScript,android]
categories: Android
---

## 简介 ##
阿里新出的weex，我一个做android的也过来凑个热闹，自己做了一个简单的demo，是一个知乎新闻+日记的APP，功能比较简单，代码也是最基本的写法和用法，初学weex的童鞋们可以参考使用学习一下。
## 使用方法 ##
**1、**[**项目地址**](https://github.com/LonerJimmy/WeexTwo)

**2、配置环境**

这个大家可以从weex官网去看。[weex官网](https://weex-project.io/cn/guide/index.html)

**3、运行项目**
terminal进入到项目更目录，执行命令：

```
npm install
```

执行脚本，把we文件编译成js文件：

```
./weextwo
```
继续执行脚本，将上一步生成的js文件复制到android的项目中：

```
./weextwo android
```

用AndroidStudio或者eclipse打开playground项目，好啦，这样你就可以看到效果啦！

## 遇到的坑 ##

下面这些问题就是我在做weextwo的时候遇到的问题。

**1、运行调试**

我这里使用的方法其实就是讲生成的js文件放到android中运行的，当然weex是支持类似RN那样，pc作为服务器将生成的bundle分发到真机上，只要修改代码，真机就可以立即看到结果。
不过weex文档有点乱，所以相关东西还没开始写，大家可以去探究探究。

**2、网络请求**

网络请求返回的数据格式问题，这个主要是要会weex debug，对js加断点，调试，还要主要method和created、ready是两个范围的函数（切忌不要把created方法写到method中），method是你自己要定义的方法，created和ready是weex自带的声明周期方法。
因为weex还没有很好的开发工具，所以我暂时用的webstorm，但是webstorm无法自动化we的格式，所以我当时在写第一个界面的时候就卡住了，一直没有进行我created中的方法，后面才发现，原来自己把created写到method中去了，大家一定要注意了。

**3、list的坑**

list的每个item点击传入一个id的时候，通过var itemid = e.target.attr.newsid来获取组件中newsid参数值，这种方式遇到一个非常非常奇葩的坑。就是如果这个组件是你自己定义的话，没法获取到，必须是要原生的组件才能获取的到，具体原因还未查明。

**4、文档错误**

1）webview和imageview的时候，src前面还有一个:，怪不得我开始一直检查不到问题，还有目前webview好像还不支持显示html的功能。

2）textarea和input中需要监控change和input的时候，input的写法是
oninput=“onTextAreaChange”，并不是文档中那种写法，试过，不行。

**5、this的作用范围**

this的范围不包括回调函数，所以如果要在回调函数中改变data中的数据，我们需要在函数中声明一个局部变量，使self=this，这个self的作用范围就在回调函数中。

**6、storage问题**
因为你存储或者获取的时候都是在回调函数中获取的，这是一个异步的过程，所以你不能在执行完一个函数之后，直接在下面的过程中，使用回调得到的结果，这个时候回调的结果可能还没拿到，所以下面的过程都要放到回调函数中去。这个地方坑了我老半天。

## 需要解决的问题 ##

-  日记保存后并没有立马生效，需要点击退出之后重新进app才能生效，因为weex的生命周期没有像android onresume生命周期，所以我暂时还没修改。
-  长按删除，也是没有立马生效，同上原因。
-  界面没有适配（weex没有dp单位），不同分辨率 界面看起来也不同。

## 效果 ##

好了，最后我们看一下，我们app的运行效果吧！

![新闻列表](http://img.blog.csdn.net/20170206214702556?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemptMDUxOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![新闻详情](http://img.blog.csdn.net/20170206214738150?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemptMDUxOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![日记列表](http://img.blog.csdn.net/20170206214802174?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemptMDUxOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![日记详情](http://img.blog.csdn.net/20170206214852994?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemptMDUxOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

还有上拉和下拉刷新，我没有截图下来，大家可以去试试。
