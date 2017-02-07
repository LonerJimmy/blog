---
title: Android自定义注解学习三步骤（转）
date: 2017-1-1 20:50:18
tags: [android,注解,annotation,反射,代理]
categories: Android
---
## java注解 ##

----------

***一、元注解***
元注解就是注解其他注解，就是我们要自定义注解的时候，需要有元注解来注解我们的自定义注解。
**1、@Target**
@Target 这个是用来指明注解修饰的目标，可能是package、类、接口、枚举、Annotation类型、类型成员（方法、构造方法、成员变量）、方法参数和本地变量。
取值（ElementType）有：

    CONSTRUCTOR: 用于描述构造器 
    FIELD: 用于描述域 
    LOCAL_VARIABLE: 用于描述局部变量 
    METHOD: 用于描述方法 
    PACKAGE: 用于描述包 
    PARAMETER: 用于描述参数 
    TYPE: 用于描述类、接口(包括注解类型) 或enum声明
举了例子就很好理解了，如下：
```
@Target(ElementType.TYPE)
public @interface AnnotationTest1 {
    public String tableName() default "className";
}
```
这就是我们自定义的一个注解，这个注解只能用来注解类、接口或者enum。
**2、@Retention**
@Retention 定义了该Annotation的生命周期。
取值有：

    SOURCE：在源文件中有效（源文件保留）
    CLASS：在class文件中有效（class中保留）
    RUNTIME：在运行时有效（运行过程中保留）

举个例子（在上面的例子中加一个元注解）：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface AnnotationTest1 {
    public String tableName() default "className";
}
```
加上这个注解咧，就表示AnnotationTest1一直保留在程序运行过程中。
**3、@Documented**
这个作用是在生成javadoc文档的时候将Annotation写入到文档中。因为android中用到的地方太少，这里就不多做介绍了。
**4、@Inherited**
我们把我们自定义好的注解放到父类上，但是它不会被子类继承，我们可以在自定义的注解上定义上一个@Inherited。

    如果父类的注解是定义在类上面，那么子类是可以继承过来的。 
    如果父类的注解定义在方法上面，那么子类也是可以继承过来。 
    如果子类重写了父类中定义了注解的方法，那么子类将无法继承该方法的注解，即子类在重写父类中被@Inherited标注的方法时，会将该方法连带它上面的注解一并覆盖掉。
举个栗子，大家可以测试一下：
```
@Retention(RetentionPolicy.RUNTIME)  
@Inherited  
public @interface InheritedTest {  
    String value();  
}  

@InheritedTest("Jadyer")  
public class Parent {  
    @InheritedTest("JavaEE")  
    public void doSomething() {  
        System.out.println("Parent do something");  
    }  
}

public class Child extends Parent {  
}

public class Test {  
public static void main(String[] args) throws SecurityException, NoSuchMethodException {  
    classTest();  
    methodTest();  
}  

/** 
* 通过反射方式测试子类是否会继承父类中定义在类上面的注解 
*/  
public static void classTest(){  
    Class<Child> c = Child.class;  

    if (c.isAnnotationPresent(InheritedTest.class)) {  
        InheritedTest inheritedTest = c.getAnnotation(InheritedTest.class);  
        String value = inheritedTest.value();  
        System.out.println(value);  
    }  
}  

/** 
* 通过反射方式测试子类是否会继承父类中定义在方法上面的注解 
*/  
public static void methodTest() throws SecurityException, NoSuchMethodException{  
    Class<Child> c = Child.class;  
    Method method = c.getMethod("doSomething", new Class[]{});  

    if (method.isAnnotationPresent(InheritedTest.class)) {  
        InheritedTest inheritedTest = method.getAnnotation(InheritedTest.class);  
        String value = inheritedTest.value();  
        System.out.println(value);  
        }  
    }  
}  

```

----------


***二、java内建注解***
Java提供了三种内建注解（就是我们常见的java自带的注解） 

    @Override——当我们想要复写父类中的方法时，我们需要使用该注解去告知编译器我们想要复写这个方法。这样一来当父类中的方法移除或者发生更改时编译器将提示错误信息。 

    @Deprecated——当我们希望编译器知道某一方法不建议使用时，我们应该使用这个注解。Java在javadoc 中推荐使用该注解，我们应该提供为什么该方法不推荐使用以及替代的方法。 

    @SuppressWarnings——这个仅仅是告诉编译器忽略特定的警告信息，例如在泛型中使用原生数据类型。它的保留策略是SOURCE并且被编译器丢弃。


----------


***三、自定义注解***
使用@interface自定义注解时，自动继承了java.lang.annotation.Annotation接口，由编译程序自动完成其他东西。
那么自定义格式是什么样子咧？
先看看注解参数可以是哪些类型：

    1、基本类型
    2、String类型
    3、Class类型
    4、enum
    5、Annotation
    6、以上所有类型的数组
注解里面的参数应该怎么设定呢：
第一,只能用public或默认(default)这两个访问权修饰.例如,String value();这里把方法设为defaul默认类型； 
第二,参数成员只能用基本类型byte,short,char,int,long,float,double,boolean八种基本数据类型和 String,Enum,Class,annotations等数据类型,以及这一些类型的数组.例如,String value();这里的参数成员就为String;　　 
第三,如果只有一个参数成员,最好把参数名称设为”value”,后加小括号 
第四,可以在使用default为每个参数设置一个默认值。注解元素必须有确定的值，要么在定义注解的默认值中指定，要么在使用注解时指定，非基本类型的注解元素的值不可为null。因此, 使用空字符串或0作为默认值是一种常用的做法。
举个栗子：
```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FruitName {
    String value() default "";
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FruitColor {
    public enum Color{ BULE,RED,GREEN};
    Color fruitColor() default Color.GREEN;
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FruitProvider {
    public int id() default -1;
    public String name() default "";
    public String address() default "";
}

```

----------

**四、使用**
针对上面的自定义注解，我们应该怎么使用呢？
```
public class Apple {
    @FruitName("Apple")
    private String appleName;

    @FruitColor(fruitColor=Color.RED)
    private String appleColor;

}
```

----------

**五、处理**
自定义看完了，那我们改怎么获取到有注解的内容呢？
AnnotateElement接口，主要有这几个实现类：

    Class：类定义 
    Constructor：构造器定义 
    Field：累的成员变量定义 
    Method：类的方法定义 
    Package：类的包定义 
所以，AnnotatedElement 接口是所有程序元素（Class、Method和Constructor）的父接口，程序通过反射获取了某个类的AnnotatedElement对象之后，程序就可以调用该对象的如下四个个方法来访问Annotation信息：

    //返回改程序元素上存在的、指定类型的注解，如果该类型注解不存在，则返回null。
    <T extends Annotation> T getAnnotation(Class<T> annotationClass)

    //返回该程序元素上存在的所有注解。
    Annotation[] getAnnotations()

    //判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false.
    boolean is AnnotationPresent(Class<?extends Annotation> annotationClass)

    //返回直接存在于此元素上的所有注释。与此接口中的其他方法不同，该方法将忽略继承的注释。（如果没有注释直接存在于此元素上，则返回长度为零的一个数组。）该方法的调用者可以随意修改返回的数组；这不会对其他调用者返回的数组产生任何影响。
    Annotation[] getDeclaredAnnotations()

----------


## java反射机制 ##

反射机制可以java对象属性、方法、构造等。
获取class类型对象有几种方式：

    1、Class.forName
    2、类名.class
    3、实例对象.getClass()
获取class类型，就可以做一系列操作，api如下：
https://developer.android.com/reference/java/lang/Class.html


----------


## android代理模式 ##

***1、静态代理***
代理类在程序前已经存在的代理方式成为静态代理。其实就是我经常使用的接口以及实现，每个代理类只能为一个接口服务，这样我们在开发中会有很多代理类。

***2、动态代理***
代理类在程序前不存在、运行时由程序动态生成的代理方式。
步骤：
**1）新建接口和委托类；**

```
public interface Agent {  

public void method1();  

public void method2();  

public void method3();  
}  

public class AgentImpl implements Agent{
    @Override
    public void method1(){
        Log.e(“Loner”,”Invoke method1”);
    }
    @Override
    public void method2(){
        Log.e(“Loner”,”Invoke method2”);
    }
    @Override
    public void method3(){
        Log.e(“Loner”,”Invoke method3”);
    }
}

```
**2）实现InvocationHandler接口，这是负责连接代理类和委托类的中间类必须实现的接口；**

```
public class TimingInvocationHandler implements InvocationHandler {  

    private Object target;  
    public TimingInvocationHandler() {}  
    public TimingInvocationHandler(Object target) {  
        this.target = target;  
    }  
    @Override  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        long start = System.currentTimeMillis();  
        Object obj = method.invoke(target, args);  
        System.out.println(method.getName() + " cost time is:" + (System.currentTimeMillis() - start));  
        return obj;  
    }  
}  

```
**3）通过Proxy类新建代理类对象。**

```
TimingInvocationHandler timingInvocationHandler = new TimingInvocationHandler(new OperateImpl());  
Agent operate = (Agent)(Proxy.newProxyInstance(Agent.class.getClassLoader(), new Class[] {Agent.class},  
timingInvocationHandler));  

// call method of proxy instance  
operate.method1();  
System.out.println();  
operate.method2();  
System.out.println();  
operate.method3(); 

```

***3、总结***


    // InvocationHandlerImpl 实现了 InvocationHandler 接口，并能实现方法调用从代理类到委托类的分派转发  
    // 其内部通常包含指向委托类实例的引用，用于真正执行分派转发过来的方法调用  
    InvocationHandler handler = new InvocationHandlerImpl(..);   

    // 通过 Proxy 为包括 Interface 接口在内的一组接口动态创建代理类的类对象  
    Class clazz = Proxy.getProxyClass(classLoader, new Class[] { Interface.class, ... });   

    // 通过反射从生成的类对象获得构造函数对象  
    Constructor constructor = clazz.getConstructor(new Class[] { InvocationHandler.class });   

    // 通过构造函数对象创建动态代理类实例  
    Interface Proxy = (Interface)constructor.newInstance(new Object[] { handler }); 

    Proxy静态方法newProxyInstance为我们封装了2-4过程，所以简化后如下：

    // InvocationHandlerImpl 实现了 InvocationHandler 接口，并能实现方法调用从代理类到委托类的分派转发  
    InvocationHandler handler = new InvocationHandlerImpl(..);   

    // 通过 Proxy 直接创建动态代理类实例  
    Interface proxy = (Interface)Proxy.newProxyInstance( classLoader,new Class[] { Interface.class },handler );  

----------

## android自定义注解 ##
了解了上面三个基础，我们就可以去做一个demo了，可以参考一下下面链接（类似于butternife的实现方式）
http://blog.csdn.net/hp910315/article/details/51199748

下面仿照retrofit的例子
http://blog.csdn.net/sbsujjbcy/article/details/50554116
