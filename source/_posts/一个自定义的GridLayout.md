---
title: 一个自定义的GridLayout
date: 2016-09-26 11:36:18
tags: [gridview,android,viewgroup]
categories: Android
---

这段时间，因为工作上业务需求，所以自己自定义了一个类似于GridView的viewgroup，源码和例子在https://github.com/LonerJimmy/GridLayoutExample
有兴趣的同学可以来参考一下，有问题的话，希望大家多多提意见。
使用方法：

```
<loner.library.gridlayout.GridLayout
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:background="@color/colorAccent"
        android:paddingTop="20dp"
        app:horizontalSpacing="10dp"
        app:numRows="4"
        app:verticalSpacing="10dp">

        <TextView
            android:layout_width="10dp"
            android:layout_height="10dp"
            android:background="#0F0"
            android:text="1" />
        <TextView
            android:layout_width="20dp"
            android:layout_height="20dp"
            android:background="#0F0"
            android:text="2" />

        <TextView
            android:layout_width="10dp"
            android:layout_height="20dp"
            android:background="#0F0"
            android:text="3" />
        <TextView
            android:layout_width="20dp"
            android:layout_height="30dp"
            android:background="#0F0"
            android:text="4" />
        <TextView
            android:layout_width="40dp"
            android:layout_height="30dp"
            android:background="#0F0"
            android:text="5" />
        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#0F0"
            android:text="6" />
        <TextView
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:background="#0F0"
            android:text="7" />
        <TextView
            android:layout_width="30dp"
            android:layout_height="50dp"
            android:background="#0F0"
            android:text="7" />

    </loner.library.gridlayout.GridLayout>
```

在values/attrs.xml中声明下面的resource：

```
<resources>
    <declare-styleable name="GridLayoutAttrs">
        <attr name="horizontalSpacing" format="dimension" />
        <attr name="verticalSpacing" format="dimension" />
        <attr name="numColumns" format="integer" />
        <attr name="numRows" format="integer" />
    </declare-styleable>
</resources>
```
通过这几个resource来控制布局，horizontalSpacing是水平方向上每列的间距，verticalSpacing是垂直方向上每列的间距，numColumns是列数，numRows是行数，这里要注意的是：
1、numColumns和numRows可以都声明，但是你的view数目要>=numColumns*numRows，否则会抛异常。这里布局跟gridview布局一样，以行为来排列。
2、你也可以只声明numColumns和numRows其中一个，但是如果只声明其中一个的话，这里布局就会跟着列或者行来进行布局。如果我说的比较抽象，大家可以去上面我的github上的例子去试一下。

    