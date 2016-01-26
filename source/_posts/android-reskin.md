---
title: 一种Android换肤机制的实现
author: 冯神柱
date: 2016-01-22 15:18:07
tags:
- Android
- 换肤
categories:
- Android开发
---
最近的项目中需要添加换肤机制。期望达到的效果：
1. 代码修改小
2. 皮肤资源与主程序资源辨识度高
3. 效率高
4. 可扩展（自定义View及自定义属性）

#### 可选技术方案
<!-- more -->
研究了一下目前的开源换肤方案的实现，大概是通过以下几种方式来实现：

1. 手动设置View属性。这种方案需要每个View的每个属性都至少要一行代码，且与具体Activity联系紧密。代码冗长，修改麻烦。   
2. 使用Activity Theme。问题是需要重启Activity。
3. 资源重定向。替换View属性对应的资源id。
想要达到预期效果，在这三种方案中用资源重定向的方式是最理想的。自己通过资源重定向写了一个换肤库：
[eastmoneyandroid/Reskin](https://github.com/eastmoneyandroid/Reskin)

下面说下整个机制的原理和过程。

#### 资源重定向换肤方案的实现
##### 资源重定向
所谓资源重定向，就是把系统默认的资源替换成我们想要更换的对应资源。这里就涉及到两部分，默认资源和对应资源。
举个例子，给TextView设置文字颜色android:textColor="@color/white"，默认资源是R.color.white，重定向后，期望通过某种机制将R.color.white映射到R.color.black上，通过setTextColor(R.color.black)可修改TextView的文字颜色。
这个过程中，应该注意的对象有View(TextView)、属性(textColor)、属性值(@color/white)。换肤要实现的就是改变属性值。

##### 记录兴趣View及其属性
这一步为换肤做准备，记下所有感兴趣的View及其属性和属性值。
我们从布局xml出发，找到一个记住View的时机。
Android布局xml中View的实例化过程可以参看我另一篇[博文](http://eastmoneyandroid.github.io/2016/01/21/android-view-instance/)。
LayoutInflater完成了从layoutResID到View创建的过程。接下来祈祷能找到某个地方安插代码吧，最好不用自行解析xml和遍历View。
LayoutInflater在递归inflate时，我们没机会侵入代码。直到LayoutInflater.createViewFromTag()。
##### LayoutInflater构造View对象的4种选择
 1. mFactory2.onCreateView()
 2. mFactory.onCreateView()
 3. mPrivateFactory.onCreateView()，由于`Activtiy implements LayoutInflater.Factory2`，会调用到Activity的onCreateView() 
 4. LayoutInflater.createView(String name, String prefix, AttributeSet attrs)

第4种选择是系统默认实例化View的方式。到了这里，生米已成熟饭，记录感兴趣的View及其属性就没有希望了。
1和2可通过`LayoutInflater.setFactory2()`和`LayoutInflater.setFactory()`，实现自己的LayoutInflater Factory插入代码。法3的mPrivateFactory可以通过Activity的onCreateView()自行初始化View。
从这里看，1、2、3没有区别，[demo](https://github.com/eastmoneyandroid/Reskin)里选取法2，截断系统默认createView()的过程。

##### 如何自己实例化出View?仿制！
前面也说了，我们通过`Activity.getLayoutInflater().setFactory()`来更改原有factory，首先实现我们的factory，并实现`onCreateView`方法:

```java
private static final String[] sClassPrefixList = {
    "android.widget.",
    "android.webkit.",
    "android.app.",
    "android.view."
};

public class SkinLayoutInflaterFactory implements LayoutInflater.Factory {
    @Override
    public View onCreateView(String name, Context context, AttributeSet attrs) {
        // 实例化View。自行预处理前缀，调用LayoutInflater.createView()实例化
        for (String prefix : sClassPrefixList) {
            try {
                View view = mLayoutInflater.createView(name, prefix, attrs);
                if (view != null) {
                    // 记录需要改变属性的View及其属性
                    addSkinViewIfNecessary(view, attrs);
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }

        return mLayoutInflater.createView(name, null, attrs);
    }
}

List<TextViewTextColorItem> textViewTextColorList = new ArrayList<>();

private void addSkinViewIfNecessary(View view, AttributeSet attrs) {
    if (view instanceof TextView) {
        int n = attrs.getAttributeCount();
        for (int i = 0; i < n; i++) {
            String attrName = attrs.getAttributeName(i);         
            if (attrName.equals("textColor")) {
                int id = 0;
                String attrValue = attrs.getAttributeValue(i);
                if (attrValue.startsWith("@")) { // 如"@2131427389"
                    id = Integer.parseInt(attrValue.substring(1));
                    textViewTextColorList.add(new TextViewTextColorItem(view, id));
                }
            }
        }
    }
}
```
对这里的onCreateView()实现有疑问，请继续深入[博文](http://eastmoneyandroid.github.io/2016/01/21/android-view-instance/)中LayoutInflater.createViewFromTag()部分。

##### 重定向--偷天换日
遍历记录感兴趣的View及其属性的数据结构，通过重定向资源更新属性值。为了减小项目修改，默认主题外的皮肤资源文件用命名后缀标识。
以替换TextView的textColor为例。
布局：
```xml
<TextView
    android:id="@+id/change_text_color"
    android:textColor="@color/textColor"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Hello World!"/>
```
更换主题：
```java
public void reSkin(String suffix) {
    for (TextViewTextColorItem skinItem : textViewTextColorList) {
            skinItem.view.setTextColor(getColor(skinItem.id, suffix));
    }
}

// 用老的资源id获取新主题资源，需要一次华丽的转身：id -> name -> new name -> new id
private int getColor(int oldResId, String suffix) {
    String oldResName = mContext.getResources().getResourceEntryName(oldResId);
    String newResName = oldResName + suffix;
    int newResId = mContext.getResources().getIdentifier(newResName, "color", mContext
            .getPackageName());
    return mContext.getResources().getColor(newResId);
}
```
示例说明：
生成的R.java中textColor resource id：
```java
public static final class color {
    ...
    public static final int textColor=0x7f0b003d;
    public static final int textColor_night=0x7f0b003e; 
}
```
oldResId = 2131427389，转16进制为0x7f0b003d，即R.color.textColor 
oldResName = textColor
newResName = textColor_night
newResId 2131427390，转16进制为0x7f0b003e，即R.color.textColor_night 
使用新resource id通过Resource获取新色值，设置到view里，完成这个TextView的textColor属性值更改。
##### 方案优点：修改基类Activity即可实现换肤，无需侵入现有业务Activity及其布局文件和资源文件
对，这就是我们想要的！

以上理论分析中我们只添加了一种关注的View和属性，接下来在换肤库中，我们将添加以下View和属性的支持，请持续关注项目[eastmoneyandroid/Reskin](https://github.com/eastmoneyandroid/Reskin)。
##### 资源类型
    - color
    - drawable
    - src
##### View及其属性
    - View background(color/drawable)
    - ListView divider(color/drawable)
    - AbsListView listSelector(color/drawable)
    - ImageView src(src)
    - TextView textColor textHintColor drawableLeft
    - ExpandableListView childDivider
    - CheckBox button
    - ProgressBar
    - 自定义View

#### 参考
- http://blog.zhaiyifan.cn/2015/09/10/Android换肤技术总结/
- https://github.com/fengjundev/Android-Skin-Loader
- https://github.com/hongyangAndroid/ChangeSkin 
- https://github.com/brokge/NightModel 
- http://blog.bradcampbell.nz/layoutinflater-factories/ 
- https://github.com/chrisjenx/Calligraphy