---
title: 一种Android换肤机制的实现
author: 冯神柱
---
最近的项目中需要添加换肤机制。研究了几个开源的项目后，决定重写，期望达到：
    - 代码修改小
    - 皮肤资源与主程序资源辨识度高
    - 效率高
    - 可扩展（自定义View及自定义属性）
#### 可选技术方案
    - 手动设置View属性
        每个View的每个属性都至少要一行代码，且与具体Activity联系紧密。代码冗长，修改麻烦。
    - Activity Theme
        需要重启Activity
    - 资源重定向
        替换View属性对应的资源id
#### 资源重定向
##### 原理：见[SethFeng/Reskin](https://github.com/SethFeng/Reskin)
##### 找到下手的时机：
根据theory分析，LayoutInflater构造View对象有4种的选择：
    1. mFactory2.onCreateView()
    2. mFactory.onCreateView()
    3. mPrivateFactory.onCreateView()，会调用到Activity的onCreateView() 
    4. LayoutInflater.createView(String name, String prefix, AttributeSet attrs)
法1和法2可轻松通过Activity.getLayoutInflater().setFactory()设置，在反射调用View的构造函数前修改其属性。
法3的mPrivateFactory可以通过Activity的onCreateView()自行初始化View。
到了法4这步，已经无能为力了...
从这里看，1、2、3没有区别，demo暂时选取法2，截断系统默认createView()的过程。
如何自己实例化出View?仿制！
```java
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

List<SkinItem> textViewTextColorList = new ArrayList<>();

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
                    textViewTextColorList.add(new SkinItem(view, id));
                }
            }
        }
    }
}
```
##### 偷天换日
遍历记录感兴趣的View及其属性的数据结构，通过重定向资源更新属性值。以替换TextView的textColor为例。
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
textView.setTextColor(getColor(resId));

// 用老的资源id获取新主题资源，需要一次华丽的转身：id -> name -> new name -> new id
private int getColor(int oldResId, String suffix) {
    String oldResName = mContext.getResources().getResourceEntryName(oldResId);
    String newResName = oldResName + suffix;
    int newResId = mContext.getResources().getIdentifier(newResName, "color", mContext
            .getPackageName());
    return mContext.getResources().getColor(newResId);
}
```
过程说明：
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
使用新resource id通过Resource获取新色值，设置到view里，完成这个TextView的textColor属性更改。
##### 优点：修改基类Activity即可实现换肤，无需侵入现有业务Activity及其布局文件和资源文件
##### 关注的资源类型：
    - color
    - drawable
    - src
##### 关注的View属性：
    - View background(color/drawable)
    - ListView divider(color/drawable)
    - AbsListView listSelector(color/drawable)
    - ImageView src(src)
    - TextView textColor textHintColor drawableLeft
    - ExpandableListView childDivider
    - CheckBox button
    - ProgressBar
    - 自定义View
    - ...

#### 参考
- http://blog.zhaiyifan.cn/2015/09/10/Android换肤技术总结/
- https://github.com/fengjundev/Android-Skin-Loader
- https://github.com/hongyangAndroid/ChangeSkin 
- https://github.com/brokge/NightModel 
- http://blog.bradcampbell.nz/layoutinflater-factories/ 
- https://github.com/chrisjenx/Calligraphy 