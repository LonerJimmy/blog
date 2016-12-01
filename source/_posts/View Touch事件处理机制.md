---
title: View Touch事件处理机制
date: 2016-07-06 15:41:18
tags: [view,touch]
categories: Android
---

**1、MotionEvent介绍**

系统有一个线程在循环收集屏幕硬件信息，当用户触摸屏幕的时候 ，这个线程就会把从设备信息收集到的信息封装成一个MotionEvent，然后把对象放到一个消息队列中。
系统另外一个线程循环读取消息队列中的MotionEvent，然后派发给当前活动的activity。
这个时间，系统收集信息是有时间间隔的，这个取决于硬件设备，目前手机应该在20毫秒左右。
MotionEvent事件我就不作具体介绍了，ACTION_DOWN、ACTION_MOVE、ACTION_UP、ACTION_CANCEL 。

**2、view touch事件处理**

我们都知道android里面view是一层层的，当有touch时间传过来的时候，也会从父层到子层传递。这里就需要onTouchEvent和onInterceptTouchEvent来控制touch事件是否往子层传，dispatchTouchEvent是用于touch事件分发，决定事件是否由onInterceptTouchEvent来拦截处理。
我们看一下ViewGroup中的dispatchTouchEvent源码（由于代码太多太繁琐，所以我就挑着看一下重要代码）：

```
if (!canceled && !intercepted) {
	for (int i = childrenCount - 1; i >= 0; i--) {
	…
	if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                     handled = true;
             }
	…
	return handled;
	}
}

```
这里是事件处理流程的关键循环，但是for循环之前有个判断条件，就是这个intercepted，然后我们看一下intercepted如何被定义的。

```
if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }

```
由此可以看出，dispatch其实就是否则touch事件分发，决定事件是否由onInterceptTouchEvent来拦截处理。返回super.dispatchTouchEvent时，由onInterceptTouchEvent来决定事件的流向。返回false时，会继续分发事件，自己内部只处理了ACTION_DOWN。返回true时，不会继续分发事件，自己内部处理了所有事件（ACTION_DOWN,ACTION_MOVE,ACTION_UP）

**3、onInterceptTouchEvent（事件拦截）**
拦截事件，用来决定事件是否传向子View
返回true时，拦截后交给自己的onTouchEvent处理
返回false时，拦截后交给子View来处理

**4、onTouchEvent（事件处理）：事件最终到达这个方法**
返回true时，内部处理所有的事件，换句话说，后续事件将继续传递给该view的onTouchEvent()处理
返回false时，事件会向上传递，由onToucEvent来接受，如果最上面View中的onTouchEvent也返回false的话，那么事件就会消失

**5、验证**
ViewA.java

```
public class ViewA extends FrameLayout {

    public ViewA(Context context) {
        super(context);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        // TODO Auto-generated method stub
        Log.d(Constant.LOGCAT, "Group1 onInterceptTouchEvent触发事件："+Constant.getActionTAG(ev.getAction()));
        return false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // TODO Auto-generated method stub
        Log.d(Constant.LOGCAT, "Group1 onTouchEvent触发事件："+Constant.getActionTAG(event.getAction()));
        return true;
    }
}
```
ViewB.java

```
public class ViewB extends FrameLayout{

    public ViewB(Context context) {
        super(context);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        // TODO Auto-generated method stub
        Log.d(Constant.LOGCAT, "Group2 onInterceptTouchEvent触发事件："+ Constant.getActionTAG(ev.getAction()));
        return false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // TODO Auto-generated method stub
        Log.d(Constant.LOGCAT, "Group2 onTouchEvent触发事件："+Constant.getActionTAG(event.getAction()));
        return true;
    }
}
```
MyTextView.java

```
public class MyTextView extends TextView{

    public MyTextView(Context context) {
        super(context);
        this.setGravity(Gravity.CENTER);
        this.setText("点击我！");
        // TODO Auto-generated constructor stub
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // TODO Auto-generated method stub
        Log.d(Constant.LOGCAT, "MyTextView onTouchEvent触发事件："+ Constant.getActionTAG(event.getAction()));
        return true;
    }
}
```
Constant.java

```
public class Constant {
    public static final String LOGCAT = "logcat";

    public static String getActionTAG(int action) {
        switch (action) {
            case 0:
                return "ACTION_DOWN";
            case 1:
                return "ACTION_UP";
            case 2:
                return "ACTION_MOVE";
            default:
                return "NULL";
        }
    }
}
```

MainActivity.java

```
public class MainActivity extends AppCompatActivity {

    ViewA viewA;
    ViewB viewB;
    MyTextView myTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        viewA = new ViewA(this);
        viewB = new ViewB(this);
        myTextView = new MyTextView(this);
        viewB.addView(myTextView, new ActionBar.LayoutParams(ActionBar.LayoutParams.FILL_PARENT,
                ActionBar.LayoutParams.FILL_PARENT));
        viewA.addView(viewB, new ActionBar.LayoutParams(ActionBar.LayoutParams.FILL_PARENT,
                ActionBar.LayoutParams.FILL_PARENT));
        setContentView(viewA);
    }
}
```

然后更改onTouch和InterceptTouch返回值就可以看到log啦，具体大家去试一下就可以啦！
