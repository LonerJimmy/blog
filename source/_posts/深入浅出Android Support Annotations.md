---
title: 深入浅出Android Support Annotations
date: 2016-08-25 12:03:18
tags: [android,annotations,support]
categories: Android
---

自己在项目中使用一些第三方的框架的时候，经常使用各种注解，使用起来十分方便，所以就简单了解了一下注解的使用。

首先将注解添加到我们工程中。

```
compile 'com.android.support:support-annotations:20.0.0'
```
有三种类型的注解可供我们使用：
Nullness注解；
资源类型注解；
IntDef和StringDef注解；

这几种类型的使用，我就通过一些示例代码来介绍一下。

**Nullness注解**

使用@NonNull注解修饰的参数不能为null。在下面的代码例子中，我们有一个取值为null的name变量，它被作为参数传递给sayHello函数，而该函数要求这个参数是非null的String类型：

```
public class MainActivity extends ActionBarActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        String corn = null;
        sayCorns(corn);
    }

    void sayCorns(@NonNull String s) {
        Toast.makeText(this, "Hello " + s, Toast.LENGTH_LONG).show();
    }
}
```

函数hello中的参数使用NonNull来修饰，所以参数不能是null，而我们把name设置为null，所以IDE会以警告的方式提示我们。

**资源类型注解**

我们之前有没有遇到一种比较尴尬的问题，同样是资源文件，我们有时候将错误的资源类型id传给函数，如下：

```
public class MainActivity extends ActionBarActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        sayCorns(R.style.AppTheme);
    }
    
    void sayCorns(int id) {
        Toast.makeText(this, "Hello " + getString(id), Toast.LENGTH_LONG).show();
    }
}
```
本来是想获得string资源的东西，结果你传了一个style里面的id给她，这样是不是很尴尬。不过没关系，资源类型注解帮我们解决这种问题。

```
public class MainActivity extends ActionBarActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        sayCorns(R.style.AppTheme);
    }


    void sayCorns(@StringRes int id) {
        Toast.makeText(this, "Hello " + getString(id), Toast.LENGTH_LONG).show();
    }

}
```
加了一个StringRes注解之后，当传入一个资源文件id给他的时候，IDE就会有警告提示。

**IntDef和StringDef注解**

很多时候，我们使用整型常量代替枚举类型（性能考虑），例如我们有一个IceCreamFlavourManager类，它具有三种模式的操作：VANILLA，CHOCOLATE和STRAWBERRY。我们可以定义一个名为@Flavour的新注解，并使用@IntDef指定它可以接受的值类型。

```
public class IceCreamFlavourManager {

    private int flavour;

    public static final int VANILLA = 0;
    public static final int CHOCOLATE = 1;
    public static final int STRAWBERRY = 2;

    @IntDef({VANILLA, CHOCOLATE, STRAWBERRY})
    public @interface Corns {
    }

    @Corns
    public int getFlavour() {
        return flavour;
    }

    public void setFlavour(@Corns int flavour) {
        this.flavour = flavour;
    }
}
```
如果我们在使用setFlavour的时候，比如：

```
IceCreamFlavourManager ice=new IceCreamFlavourManager();
ice.setFlavour(4);
```
因为4不属于@IntDef中任一一个类型，所以IDE会报错。

@StringDef用法和@IntDef基本差不多，只不过是针对String类型而已。