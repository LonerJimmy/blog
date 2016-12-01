---
title: 深入了解View（一）——LayoutInflater原理分析
date: 2016-09-13 17:50:18
tags: [布局,layout,inflater]
categories: Android
---

一直想要深入了解一下view的工作原理，现在有时间空出来了，所以就着手准备了解一下，首先先看一下LayoutInflater的原理。
相信大家对LayoutInflater一定不陌生，我们在加载布局的时候通常都会用到这个，一开始对LayoutInflater也不是很了解，因为我们平时用的都是setContentView方法，今天查了一些资料才知道，原来setContentView方法内部也是用LayoutInflater来实现的。
先看看LayoutInflater的使用方法吧，首先需要获取LayoutInflater的实例：

```
LayoutInflater layoutInflater = LayoutInflater.from(context);  
```
得到LayoutInflater实例之后，我们就可以使用其中的inflate方法来加载布局了，如下：

```
layoutInflater.inflate(resourceId, root); 
```
inflate方法中有两个参数，第一个就是需要加载布局的id，第二个就是需要给该布局外部再嵌套一层父布局，如果不需要的话，我们就传一个null就可以了。

用法就介绍到这里，我们看一下inflate方法到底是如何实现的。
最终跳转到的inflate方法源码如下：

```
 public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (Exception e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                                + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);

            return result;
        }
    }
```
**从源码中我们可以看出，LayoutInflate其实就是利用android pull解析方法来解析布局文件的。**
这个代码很多，但是我们抓住几行重要的函数看一下就可以大概了解工作机制了，其中调用了createViewFromTag()方法，将节点名和参数传进去，我们看一下createViewFromTag实现的源码：

```
            View view;
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
```
这里面，又调用了createView方法，然后使用反射的方式创建出view示例，然后返回。实际上就是根据节点创建了一个view对象，但是这里只是创建一个根布局实例而已，后面又调用rInflate()方法，遍历这个跟布局的子元素：

```
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```
这里其实是一个递归方法，调用createViewFromTag创建view实例，然后rInflate方法查找这个view下面的子元素，每次递归完成后将这个view添加到父布局中。这样将整个你要添加的布局都解析完成，形成一个完整的DOM结构，然后将最顶层的根布局返回。
我们再来看看inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)中的参数的含义，可以看一下这个代码：

```
if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
```

所以得到的结论如下：
**1. 如果root为null，attachToRoot将失去作用，设置任何值都没有意义。**
**2. 如果root不为null，attachToRoot设为true，则会给加载的布局文件的指定一个父布局，即root。**
**3. 如果root不为null，attachToRoot设为false，则会将布局文件最外层的所有layout属性进行设置，当该view被添加到父view当中时，这些layout属性会自动生效。**

介绍到这里，我们对layoutInflate应该有大致的了解了吧！除此之外，通过看一些资料，我发现了一些以前没有注意过的东西。当我们在使用layoutInflater的时候，如果将一个只有button的布局加到viewgroup中，无论我们怎么设置button的layout_width和layout_height，添加到viewgroup中的button的大小都没改变，这是为啥咧？
那是因为button控件不存在任何布局中，如果你在button哪个xml中外面再包一层relativeLayout，就可以改变button的大小了。
补充：其实平时我们使用的setContentView方法中，我们自定义的布局为什么设置layout_width和layout_height是ok的呢，原来setContentView中在我们布局外面又嵌套了一层Framelayout，哈哈！

好了，今天就聊到这里，要继续苦逼做业务需求了！
