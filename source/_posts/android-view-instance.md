---
title: Android View的实例化
author: 冯神柱
---
##### 从xml到View：
Activity：
```java
public void setContentView(int layoutResID) {
    ...
    mLayoutInflater.inflate(layoutResID, mContentParent);
    ...
}
```
LayoutInflater:
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

public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    ...
    final View temp = createViewFromTag(root, name, inflaterContext, attrs); // 创建用户xml根布局View
    ...
    rInflateChildren(parser, temp, attrs, true); // 创建用户xml根布局的children
    ...
}

// rInflateChildren调用到
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
都指向了LayoutInflater.createViewFromTag()
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

##### xml布局文件中简写的系统View需要补全前缀
布局的LayoutInflater类型实际为PhoneLayoutInflater。
系统View自动添加前缀列表，PhoneLayoutInflater.onCreateView()
    - android.widget.
    - android.webkit.
    - android.app.
添加前缀使用反射尝试找到构造函数来构造View对象，失败了就换个前缀再次尝试下一个前缀。遍历完还是失败，调用super LayoutInflater的方法，添加前缀:
    - android.view.
如：LinearLayout补全为android.widget.LinearLayout。

```java
private Factory mFactory;
private Factory2 mFactory2;
private Factory2 mPrivateFactory;
```

##### LayoutInflater构造View可能的选择：
1. mFactory2.onCreateView()
2. mFactory.onCreateView()
3. mPrivateFactory.onCreateView()，由于Activtiy implements LayoutInflater.Factory2，会调用到Activity的onCreateView() 
4. LayoutInflater.createView(String name, String prefix, AttributeSet attrs)

##### 实例化View：LayoutInflater.createView()
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
        
        if (mFilter != null && clazz != null) {
            boolean allowed = mFilter.onLoadClass(clazz);
            if (!allowed) {
                failNotAllowed(name, prefix, attrs);
            }
        }
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
View的对象实例化通过反射调用其构造函数：View(Context context, @Nullable AttributeSet attrs)。
自此，View的Java对象才创建出来。
如果到这里还没有修改记录View的属性，生米已成熟饭，晚矣!