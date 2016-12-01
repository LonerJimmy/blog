---
title: android自定义手势解锁
date: 2016-05-26 19:16:18
tags: [view,支付宝,touch,自定义]
categories: Android
---

## 简介 ##
一个手势解锁的view(仿照支付宝手势解锁的样式)，可以改变手势解锁的颜色、样式（目前只有两种样式）、大小等等。 

1、需要在整个项目的build.gradle中添加如下依赖项（jcenter暂时没有传上去）

```
allprojects {
    repositories {
        maven {
            url  "http://dl.bintray.com/loner/maven"
        }
    }
}
```
2、在自己app的build.gradle中添加如下依赖项

```
compile 'loner.library.gesture:gesture:1.0.1'
```
## 使用方法 ##
**1、首先在layout文件中显示view**

```
 <loner.widget.lockpattern.view.LockPatternSmallView
	    android:id="@+id/mLocusPassWordViewFirstRegisterSmall"
        android:layout_gravity="center"
        android:layout_width="50dp"
        android:layout_height="50dp" >
  </loner.widget.lockpattern.view.LockPatternSmallView>
```

```
  <loner.widget.lockpattern.view.LockPatternView
        android:id="@+id/mLocusPassWordViewFirstRegister"
        android:layout_gravity="center"
        android:layout_width="350dp"
        android:layout_height="350dp" >
  </loner.widget.lockpattern.view.LockPatternView>
```
LockPatternSmallView是小数字键盘，LockPatternView是大数字键盘。
这里有几个属性简单介绍一下(使用方法就是在view下面添加app:pressColor=" ")：
-  pressColor表示按下是按钮的颜色。 
-  initColor表示初始时按钮的颜色。
-  errorColor表示按下是按钮的颜色。 
-  style表示手势解锁样式，目前只有0和1两种样式，大家可以试试
-  radius表示手势解锁键盘的圆圈半径

**2、在java文件中声明两个view变量，一个是大键盘，一个是小键盘，如下：**

```
private LockPatternView lpv;
private LockPatternSmallView lpvs;
lpv=(LockPatternView)findViewById(R.id.mLocusPassWordViewFirstRegister);
lpvs=(LockPatternSmallView)findViewById(R.id.mLocusPassWordViewFirstRegisterSmall);
```

**3、更多使用方法**

（1）通过一个回调方法来获取键盘数字密码，写法如下：

```
lpv.setOnCompleteListener(new LockPatternView.OnCompleteListener() {
    @Override
    public void onComplete(String mPassword) {
            ...
    }
}
```

onComplete函数就是当键盘输入结束后触发的函数。
onComplete函数中的参数mPassword就是键盘输入结束后的密码，然后在onComplete函数中进行操作。
九宫格标记如下

1          2         3
4          5         6
7          8         9

密码记录的是每个格子的标记，中间用”,”隔开。
mPassword=””的时候，及手势密码位数小于5。
```
if (mPassword.equals("")) {//手势密码位数不正确
    txtTitle.setText("手势密码不合规范"); 
    txtTitle.setTextColor(Color.RED);            
}
```
（2）设置键盘不可touch
      
```
lpv.disableTouch();
```

（3）清空键盘

```
lpv.clearPassword();
```

（4）设置错误

```
lpv.error();
```

（5）保存密码

```
lpv.setPassWord();
```

（6）设置小键盘显示

```
lpvs.setOndraw(FirstPassword);
```

FirstPassword就是String类型，String格式就是mPassword格式。

（7）密码保存存放到sharedpreference，获取密码如下：

```
SharedPreferences settings = this.getSharedPreferences("Gesture_Lock",MODE_PRIVATE);
txtPassword.setText(settings.getString("password", ""));
```
（8）设置密码长度

```
lpv.setPasswordMinLength(3);
```

[demo以及源码下载](https://github.com/LonerJimmy/LockPatternDemo)
