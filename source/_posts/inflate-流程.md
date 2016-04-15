title: inflate 流程
date: 2016-04-15 10:24:52
tags: [源码学习]
categories: [android]
---

# 前言
我们要加载 xml 上的 View 时，大抵都会用上下面几种方法。
1. inflater.inflate(R.layout.frag_home_detail_scroll, null)
2. inflater.inflate(R.layout.frag_home_detail_scroll, null, false)
3. LayoutInflater.from(mContext).inflater()
<!--more  -->
先看看第三种，可以看到，其实也是获取一个 LayoutInflater 实例，和前面2种方法，没区别。

```java
public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
```

接下来，继续看前面2种方法的源码。

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
```

第一个方法，就是直接调用第二个方法

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```

可以看到最终是调用了  
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)   
方法。

# inflate 真正干活的方法
## 方法签名注解
方法签名的注解很重要 ，先来看看。这是一个加载 View 层的方法。为了性能，View 的 inflate 过程  
严重依赖 构建过程中预处理的 xml 文件。因此，目前在运行时，是不可以直接加载一个原生的 xml 文件  
### 方法几个入参
parser 是一个 dom 解析的 。root 是一个可选参数，如果 attachToRoot 为 true  
那么 root 就成为要加载的 xml View 的 parentView 。如果 attachToRoot 为 false ，那么  
root 仅仅是帮助 xml 上的 root view 产生 合适的 LayoutParams 。
### 出参
如果 root 不为 null ， 并且 attachToRoot == true 的话，那么 返回 root .  
其余情况，均返回 xml 上的 root view 。

```java
/**
     * Inflate a new view hierarchy from the specified XML node. Throws
     * {@link InflateException} if there is an error.
     * <p>
     * <em><strong>Important</strong></em>&nbsp;&nbsp;&nbsp;For performance
     * reasons, view inflation relies heavily on pre-processing of XML files
     * that is done at build time. Therefore, it is not currently possible to
     * use LayoutInflater with an XmlPullParser over a plain XML file at runtime.
     *
     * @param parser XML dom node containing the description of the view
     *        hierarchy.
     * @param root Optional view to be the parent of the generated hierarchy (if
     *        <em>attachToRoot</em> is true), or else simply an object that
     *        provides a set of LayoutParams values for root of the returned
     *        hierarchy (if <em>attachToRoot</em> is false.)
     * @param attachToRoot Whether the inflated hierarchy should be attached to
     *        the root parameter? If false, root is only used to create the
     *        correct subclass of LayoutParams for the root view in the XML.
     * @return The root View of the inflated hierarchy. If root was supplied and
     *         attachToRoot is true, this is root; otherwise it is the root of
     *         the inflated XML file.
     */
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

## 方法源码
先定义返回值，并且初始值为 root 。View result = root   
接着过滤了不合法的 xml 标签。再接下来，是分开了2种情况了

```java
/**
 * Recursive method used to descend down the xml hierarchy and instantiate
 * views, instantiate their children, and then call onFinishInflate().
 * <p>
 * <strong>Note:</strong> Default visibility so the BridgeInflater can
 * override it.
 */
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

### xml 标签是 merge
#### 如果 root 为空，或者 attachToRoot 为 false 的话，就抛出异常。
这是合理的，merge 标签，尤其是对比 include ，主要是省了一层布局，如果你 root 为空或者  
attachToRoot 为 false，merge下面的子 View 怎么样添加上去呢。
#### 进入到方法 rInflate(parser, root, inflaterContext, attrs, false); 中
从注解中可以了解到，这是一个递归的方法，去解析 xml 的 View 。最后就调用 onFinishInflate 方法。  
接着解析标签的时候，有2个标签很奇怪，没见过。
##### 一个是 requestFocus ，其中  requestFocus，标签主要给 parent.requestFocus() .
##### 另一个是 tag 。而 tag 标签，则给 parent.setTag(key, value).  其中 key 取自 ViewTag_id ，  
 value 取自 ViewTag_value。来源是 attrs。
##### merge 标签，则是不应存在这里的，仅仅适用于 xml 中顶级标签。
##### include
进入到方法 parseInclude(parser, context, parent, attrs);

```java
private void parseInclude(XmlPullParser parser, Context context, View parent,
        AttributeSet attrs) throws XmlPullParserException, IOException {
    int type;

    if (parent instanceof ViewGroup) {
        // Apply a theme wrapper, if requested. This is sort of a weird
        // edge case, since developers think the <include> overwrites
        // values in the AttributeSet of the included View. So, if the
        // included View has a theme attribute, we'll need to ignore it.
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        final boolean hasThemeOverride = themeResId != 0;
        if (hasThemeOverride) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();

        // If the layout is pointing to a theme attribute, we have to
        // massage the value to get a resource identifier out of it.
        int layout = attrs.getAttributeResourceValue(null, ATTR_LAYOUT, 0);
        if (layout == 0) {
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            if (value == null || value.length() <= 0) {
                throw new InflateException("You must specify a layout in the"
                        + " include tag: <include layout=\"@layout/layoutID\" />");
            }

            // Attempt to resolve the "?attr/name" string to an identifier.
            layout = context.getResources().getIdentifier(value.substring(1), null, null);
        }

        // The layout might be referencing a theme attribute.
        if (mTempValue == null) {
            mTempValue = new TypedValue();
        }
        if (layout != 0 && context.getTheme().resolveAttribute(layout, mTempValue, true)) {
            layout = mTempValue.resourceId;
        }

        if (layout == 0) {
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            throw new InflateException("You must specify a valid layout "
                    + "reference. The layout ID " + value + " is not valid.");
        } else {
            final XmlResourceParser childParser = context.getResources().getLayout(layout);

            try {
                final AttributeSet childAttrs = Xml.asAttributeSet(childParser);

                while ((type = childParser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty.
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(childParser.getPositionDescription() +
                            ": No start tag found!");
                }

                final String childName = childParser.getName();

                if (TAG_MERGE.equals(childName)) {
                    // The <merge> tag doesn't support android:theme, so
                    // nothing special to do here.
                    rInflate(childParser, parent, context, childAttrs, false);
                } else {
                    final View view = createViewFromTag(parent, childName,
                            context, childAttrs, hasThemeOverride);
                    final ViewGroup group = (ViewGroup) parent;

                    final TypedArray a = context.obtainStyledAttributes(
                            attrs, R.styleable.Include);
                    final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
                    final int visibility = a.getInt(R.styleable.Include_visibility, -1);
                    a.recycle();

                    // We try to load the layout params set in the <include /> tag.
                    // If the parent can't generate layout params (ex. missing width
                    // or height for the framework ViewGroups, though this is not
                    // necessarily true of all ViewGroups) then we expect it to throw
                    // a runtime exception.
                    // We catch this exception and set localParams accordingly: true
                    // means we successfully loaded layout params from the <include>
                    // tag, false means we need to rely on the included layout params.
                    ViewGroup.LayoutParams params = null;
                    try {
                        params = group.generateLayoutParams(attrs);
                    } catch (RuntimeException e) {
                        // Ignore, just fail over to child attrs.
                    }
                    if (params == null) {
                        params = group.generateLayoutParams(childAttrs);
                    }
                    view.setLayoutParams(params);

                    // Inflate all children.
                    rInflateChildren(childParser, view, childAttrs, true);

                    if (id != View.NO_ID) {
                        view.setId(id);
                    }

                    switch (visibility) {
                        case 0:
                            view.setVisibility(View.VISIBLE);
                            break;
                        case 1:
                            view.setVisibility(View.INVISIBLE);
                            break;
                        case 2:
                            view.setVisibility(View.GONE);
                            break;
                    }

                    group.addView(view);
                }
            } finally {
                childParser.close();
            }
        }
    } else {
        throw new InflateException("<include /> can only be used inside of a ViewGroup");
    }

    LayoutInflater.consumeChildElements(parser);
}

```

这个方法体，做了一系列的异常检查，最后检查到，如果标签是 merge 的话，那么就调用 rInflate 方法  
如果用 include ，xml 里面顶级标签是 merge 的话，的确可以省一层布局。
View view = createViewFromTag() ,通过这个方法，可以找到标签对应的 View 。 然后，尽量为 view  
设置 LayoutParams .如果 include 里面 宽高都有的话，就有 include 的。如果没有就只能用 include  
里面的 xml 的宽高了。  
调用方法  rInflateChildren(childParser, view, childAttrs, true);   
这个方法，其实也是调用方法 rInflate，继续加载 View 。  
设置 View id，以及设置 visibility ,最后就是 group.addView(view)


##### others
其他的话，其实就是 View 了。接下来做的事情是，加载出 View 来，再根据属性 attrs 来设置 view 的  
LayoutParams 了。最后就是，viewGroup.addView(view, params);  
**值得注意的是** 这里的 params 是 addView 的时候加上去的，而在 include 标签中，是直接用  
view.setLayoutParams 的，这里其实是没有本质区别的。


#### 小结
xml 顶级标签是 merge的情况下，inflate 参数必须 root 不为空，并且 attachToRoot 必须等于 true .  
返回的 View 是 root .


### xml 标签不是 merge
#### 在这个代码块内，做的事情，按照顺序如下：
1. 加载出 View 来。

 ```java
 // Temp is the root view that was found in the xml
                     final View temp = createViewFromTag(root, name, inflaterContext, attrs);
 ```

2. 如果 root 不为空
为 ViewGroup.LayoutParams params 赋值。如果 attachToRoot 是 false 的话，就直接把  
这个 params 设置给 temp .
3. 递归加载所有子 View

 ```java
 // Inflate all children under temp against its context.
                     rInflateChildren(parser, temp, attrs, true);
 ```

4. 如果需要，把 xml 上的 topView 添加到 root 上。

```java
// We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }
```

5. 看是否需要为 result 重新赋值

```java
// Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
```


#### 小结
这里，我们需要关心的2个问题是：
1. 返回的 view ，到底是什么鬼。
如果 root 为空，或者 attachToRoot 为 false，那么返回的就是 xml 上的 TopView 。其余情况，  
其实其余情况，就剩下当 root 不为空，并且 attachToRoot 为 true 的时候了，这个时候返回的就是  
root.

2. 还有，我在 xml 上设置的属性，到底有没有生效了。
首先我们需要明白，在 xml 设置的宽高属性，android:layout_width ,这个应该理解为，相对父 View  
的期望宽高，在 View 的绘制 measure 过程中，更能体现这个意思。  
因此在 inflate 过程中，只要传入的 root 不为空，xml上设置的就有效果。那么问题来了，如果我  
inflate 的时候不传入 root，xml 上的 View 的 LayoutParams 就由 viewGroup.addView(childView,params).  
这时候决定了。恩，viewGroup.addView(childView) ，这样也是可以的，这个时候 params 的值，就  
采用了默认值。代码如下  


```java
public void addView(View child) {
        addView(child, -1);
    }
```


```java
public void addView(View child, int index) {
        if (child == null) {
            throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
        }
        LayoutParams params = child.getLayoutParams();
        if (params == null) {
            params = generateDefaultLayoutParams();
            if (params == null) {
                throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
            }
        }
        addView(child, index, params);
    }
```


```java
/**
     * Returns a set of default layout parameters. These parameters are requested
     * when the View passed to {@link #addView(View)} has no layout parameters
     * already set. If null is returned, an exception is thrown from addView.
     *
     * @return a set of default layout parameters or null
     */
    protected LayoutParams generateDefaultLayoutParams() {
        return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    }
```
## 注意事项
### inflate xml merge 标签
如果 inflate 的 xml 顶级标签是 merge 的话，inflate 参数必须 root 不为空，并且 attachToRoot  
必须等于 true .
### root 不为空，并且 attachToRoot 为 true 时
这个时候会调用 rootView.addView(temp,params) .并且返回的是 root。

# 题外话
## 关于学习方法
之前看过一些相关的文章（十分感谢），会给出一些结论，例如 root 和 attachToRoot 分别在不同情况下，返回的值以及  
xml 上的属性到底有没效果。本人大概看懂之后，还略刻意的去记住了相关的结论，可是纸上得来终觉浅，遇到问题  
或者别人问到了，都会感觉有点忘了，自己心里都是有点虚的，不敢给出肯定的答案。要么直接给别人一个可能的情况，  
再去看看源码，确定一下，万一回答错了，还得（抱歉状）跟别人说。要么，回答，让先看看代码，待会再回答。这个时候，  
给别人的感觉，这丫的，不靠谱，还不如自己看代码去好了。也许，源码，最好都撸一篇，熟练于心就好了。  
可是现实是，不会让你有那么多时间的，能把基础的源码和重要的源码，看熟就很不错了（起码对于本人而言）。  
现在的社会，感觉有点信息爆炸了，各种新的技术出来，都得去看看学习下。  
## 关于追求
今早，看【APP研发实录】 中，提到做 APP 开发的同学，到底追求应该是什么。熟练各种 API ，做出各种炫酷的  
动画交互和 APP 么。 最近越来越感觉自己基础知识不够扎实，不懂的也越来越多，而想法较多。
