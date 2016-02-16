---
title: Android xml布局中View的实例化
author: author2
date: 2016-01-21 15:18:00
tags:
- Android
- View
categories:
- Android原理
---
xml布局文件是Android界面最重要的资源。当我们在Activity里调起熟悉的`setContentView(R.layout.xxx)`，是否想知道这些这些对应xml中每个标签对应的View对象是怎么创建出来的。

#### 起航
<!-- more -->
调用Activity的setContentView时，layout布局资源文件id作为参数传入。
```java
public void setContentView(int layoutResID) {
    ...
    mLayoutInflater.inflate(layoutResID, mContentParent);
    ...
}
```
真正布局需要LayoutInflater完成。
#### LayoutInflater--专业布局二十年
LayoutInflater的功能是把xml布局文件实例化成View对象。下面开始分析其过程。
##### 解析布局xml
```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    ...
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
    ...
}
```
##### 创建布局根View，递归布局Child View
```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    ...
    final View temp = createViewFromTag(root, name, inflaterContext, attrs); // 创建用户xml根布局View
    ...
    rInflateChildren(parser, temp, attrs, true); // 创建用户xml根布局的children
    ...
}

// rInflateChildren辗转调用到
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
    ...
    final View view = createViewFromTag(parent, name, context, attrs);
    final ViewGroup viewGroup = (ViewGroup) parent;
    final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
    rInflateChildren(parser, view, attrs, true);
    viewGroup.addView(view, params);
    ...
}
```
##### Parent View和Children View都走到LayoutInflater.createViewFromTag()
```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
    ...
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
            if (-1 == name.indexOf('.')) { // 系统View，需要补全前缀，之后调用如同下面的createView()
                view = onCreateView(parent, name, attrs);
            } else {
                view = createView(name, null, attrs);
            }
        } finally {
            mConstructorArgs[0] = lastContext;
        }
    }

    return view;
    ...
}
```
作为一个纯纯的人，不设置LayoutInflater的Factory2和Factory、不重载Activity的onCreateView()或重载了不在里面把View搞出来，那就按部就班地走到了line 21。
还记得吗，曾经写xml布局的时候，标签里的"android.widget.TextView"可以偷懒写成"TextView"。自定义View写到布局时，总是要确保包名正确。
```xml
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>

<com.ugly.CustomView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
```
不是亲生的外貌都被歧视了。
###### 强插：xml布局文件中简写的系统View补全前缀
View的实例化使用了Java反射机制，系统View的类名必须补全，ClassLoader才能找到对应class。如：TextView补全为android.widget.TextView。
布局的LayoutInflater类型实际为PhoneLayoutInflater。
系统View自动添加前缀列表，在PhoneLayoutInflater.onCreateView()中有三个：
    - android.widget.
    - android.webkit.
    - android.app.
添加前缀使用反射尝试找到构造函数来构造View对象，失败了就换个前缀再次尝试下一个前缀。PhoneLayoutInflater失败了，调用LayoutInflater，添加前缀:
    - android.view.
意淫一下，写布局xml的时候，把View类名补全，是不是效率又变高了呢？

line 22 onCreateView整容后也能和line 24一样，走到LayoutInflater.createView()。
##### 实例化View
```java
public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    Class<? extends View> clazz = null;
    ...
    if (constructor == null) {
        // Class not found in the cache, see if it's real, and try to add it
        clazz = mContext.getClassLoader().loadClass(
                prefix != null ? (prefix + name) : name).asSubclass(View.class);
        ...        
        constructor = clazz.getConstructor(mConstructorSignature);
        constructor.setAccessible(true);
        sConstructorMap.put(name, constructor); // sConstructorMap缓存使用过的构造函数，减少调用ClassLoader的次数
    } else {
        ...
    }

    Object[] args = mConstructorArgs;
    args[1] = attrs;
    final View view = constructor.newInstance(args);
    ...
    return view;
    ...
}
```
看来，View的对象实例化是通过反射调用其构造函数完成的。
```java
public View(Context context, @Nullable AttributeSet attrs) {
    this(context, attrs, 0);
}
```
自此，View的Java对象才创建出来。

