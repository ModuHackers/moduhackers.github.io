---
title: Android热修复技术
author: author3
date: 2016-03-31 15:18:07
tags:
- Android
- 热修复
categories:
- Android
---

热修复技术可以让应用在不重新发布的情况之下进行一定程度的更新，对于修复紧急的线上bug非常有用。

## 可选方案
目前Github上主流的开源项目有以下3个
### - 淘宝Dexposed
项目地址:[https://github.com/alibaba/dexposed](https://github.com/alibaba/dexposed)
### - 支付宝AndFix
项目地址:[https://github.com/alibaba/AndFix](https://github.com/alibaba/AndFix)
### - 女娲
项目地址:[https://github.com/jasonross/Nuwa](https://github.com/jasonross/Nuwa)
<!-- more -->

## 方案筛选
Dexposed目前对ART支持不好，第一个排除掉。  
AndFix采取了更底层的实现方式，涉及较多C++代码，深入学习成本较高，另外这套方案中最关键的差分包生成工具并没有开源，无疑也是一个隐患，所以也排除掉。  
女娲其实现分为Java和Groovy两部分，维护相对简单。对掌握Gradle插件编写有很大的帮助，这点对后续改造官方的dex分包方案很重要。  
综合考虑下来女娲是最优方案。
## 基本原理
QQ空间团队的[这篇文章](http://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=0#wechat_redirect)应该是国内最早公开介绍热修复技术原理的了，具体的细节可以直接阅读这篇文章，下面介绍一些要点。  
首先，Google有个dex分包方案，支持把多个dex文件塞入到app的classloader之中，classloader加载类时会遍历这多个dex的队列。  
官方方案中，不同的dex中是没有重复类的，如果不同的dex中有重复的类class，classloader在遍历dex队列时会加载第一个被找到的class，其它dex中的class不再被加载。  
所以将想要修复的的类打成dex，并且插入到classloader dex加载队列的最前，应该就能替换这些类，从而达到修复的目的。
Google对apk安装过程做了一些性能方面的优化，将dex转化为odex。在这个过程中会对一些类做处理，如果一个class直接引用到的所有其它类都和当前class在同一个dex中，class就会被打上CLASS_ISPREVERIFIED的标志，不能再从其它的dex中引用类了，也就无法用插入新dex的方式热替换class直接引用的类了。 
总结一下这里的要点就是：
- 1.防止类被打上CLASS_ISPREVERIFIED的标志。
- 2.将包含补丁class文件的dex插入到应用运行时ClassLoader的dexElements加载序列最前面。

## 整体流程
女娲，下文称为nuwa，就是以QQ空间的这篇介绍为基础，使用Gradle将大量工作变成了自动化方式完成。  
其整体流程可分为3个部分，分别是：
- 1.class文件的处理
- 2.补丁包的生成
- 3.补丁包的加载

### class文件的处理
对class文件的处理就是为了防止类被打上CLASS_ISPREVERIFIED的标志，这个步骤在客户端正常打包和生成补丁包两个过程中都会进行。     
前面说过，要想类能被插入新dex的方式替换，就要阻止直接引用它的类被打上CLASS_ISPREVERIFIED标记。为此，nuwa提供了一个Gradle插件(下文称为nuwa_plugin)。  
nuwa_plugin在客户端的编译期，所有class文件生成之后，使用[asm](http://asm.ow2.org/)修改了所有可能需要被替换类的class文件，向它们的构造方法中注入了一段代码，引用了一个空实现的类(下文称之为Hack.class)，Hack.class会单独存在一个dex中，这样就避免了引用Hack.class的类被打上CLASS_ISPREVERIFIED的标志，而这些类中所引用的其它类就可以从其它dex中被替换了。  
注意这里不是所有类都能被nuwa这套方案所替换的，具体是哪些类，会在下文的加载流程中说明。 
这里需要将一个包含Hack.class未签名Hack.apk放在应用编译时nuwa_plugin方便找到的路径下。nuwa_plugin在应用编译时将Hack.apk拷贝至应用的assets文件夹下，并使用当前客户端打包的key对其进行签名，使用相同的key签名是因为补丁加载时会校验签名，而Hack.apk本质也是一个补丁。
### 补丁包的生成 
补丁包的生成也是在应用编译打包时由nuwa_plugin来完成，原理就是比对前后两次同名class文件的hash值，将两次不一致的class提取出来打包成apk，并用与应用打包时相同的key签名。  
所以这一步需要将待修复应用编译生成的class映射文件保存下来，在生成补丁时将映射文件的路径传入nuwa_plugin。
### 补丁包的加载
补丁包的加载可分为以下几个部分  
#### 1. 加载Hack.class   
前面class的处理中提到，nuwa会往所有可能被替换类的class文件中插入一段对Hack.class引用的代码，所以在这些被修改的类加载之前，必须先加载Hack.class，而包含Hack.class的apk已经在编译期被nuwa_plugin放入了应用的assets文件夹。  
nuwa首先从assets中将Hack.apk拷贝至应用的私有存储中，也就是设备的/data/data目录下，这个操作只会在应用第一次启动时进行，然后将其中包含Hack.class的dex插入到ClassLoader加载序列的最前列。  
在上面的步骤完成前就加载的类，在被nuwa_plugin编译处理时是不能被注入对Hack.class引用的，因为它们加载时，Hack.class还不存在于ClassLoader的加载路径中，一旦注入了对Hack.class引用，加载后就会因为找不到Hack.class而异常，所以它们是不能被nuwa替换的。 
所以这个步骤越早进行越好。一般选择在Application或者其子类的attachBaseContext方法中进行，这同时也说明了Application或者其子类是无法被热修复的。  
#### 2. 加载补丁包  
这个过程与Hack.class的加载大致相同，区别就是补丁包位于sd卡上，不需要拷贝的操作。 
关于上述两个过程详细的源码分析，可以参考[这篇文章](http://blog.csdn.net/sbsujjbcy/article/details/50812674)。

### 我们的扩展
nuwa只是提供了一个基本的方案，如果要集成到实际的应用当中，还需要做一些其它的工作。  
### 补丁包的下载
这块是整个集成中花费时间较多的，整个nuwa方案的核心是补丁的加载，但是在实际应用中下载也是同等重要的，需要客户端通过pull或者push的触发方式去下载补丁包。  
首先简单介绍目前app的一些背景:  
- 1.app中有些内容是动态的，由服务器端控制这些变化，对应这些动态的内容，在服务器端有一个json格式的配置文件。  
- 2.app启动时在loading界面会去服务器读取配置文件的md5，判定配置是否发生了变化，如果有变化就会去请求配置文件的内容

不同的app在补丁下载这里的逻辑都会因各自业务逻辑的不同而不同，核心是找到合适的触发点，将下载逻辑挂载在这些触发点上，下面仅结合笔者团队app现有逻辑给出一种解决方案供参考：  
- 1.首先是服务器端，在应用主配置文件中加入一个新的字段hotFixMD5。  
- 2.触发下载补丁的点有两处，一是应用的主配置文件发生了变化，客户端会去检测hotFixMD5这个字段；二是主配置文件没有变化，应用会去检测是否有下载失败的补丁。  
- 3.当上述的触发点触发，客户端会判定是否需要请求补丁配置文件，如果需要下载，会保存hotFixMD5至SharedPreferences供下次触发补丁下载逻辑时判断用。  
- 4.成功请求到补丁配置文件后，会在其中查找合适当前设备和应用版本的补丁信息，如果有匹配的，并且其它前置条件(如sd卡正常加载)满足，就会读取匹配项中的md5和downloadUrl，进行下载。  
- 5.下载成功后，会将补丁包的md5保存至SharedPreferences中，供补丁包加载时校验。  
- 6.若下载失败，会保存失败状态，供下次应用启动时检测。

### 补丁包的加载
补丁包的加载同样需要在原有的基础上做一些改进，这里主要考虑的是安全性。    
nuwa现有方案中对补丁包的签名做了校验，这样可以防止替换补丁包去加载一些恶意代码。  
除此之外，客户端会不定期的更新，如果加载了之前版本的补丁，由于代码的变更，极有可能引起客户端的crash，为了防止这种情况出现，在加载补丁时增加了md5的校验。
md5校验核心的流程如下：  
- 1.前面提到过，补丁下载成功时，保存补丁的md5至SharedPreferences，key中包含客户端版本信息。 
- 2.补丁加载时，以包含当前版本信息的key去SharedPreferences取补丁的md5，与设备上的补丁包md5做比对，如果相同才加载。

### 补丁包的下架
如果一个补丁包因为某些原因需要下架，这个时候只需要在服务器端将应用配置文件中的hotFixMD5值置为空，当触发下载逻辑时，发现hotFixMD5的值由非空变为空，就会主动删除已经下载的补丁包。
### 开发环境隔离
在集成nuwa的开发过程中，避免不了对整个流程的测试，常常造成其它同事调试中的客户端加载了测试用的补丁包引发crash，影响他们的开发。
为此，主要通过两点来避免：  
- 1.在单独的服务器地址配置补丁包。  
- 2.添加加载补丁的开关，默认开启，其它功能的开发者可关闭补丁加载功能。

## 实际使用
### nuwa的使用
#### 1. 在直接依赖nuwa的module的build.gradle中 
这里是在module_nuwa的build.gradle中
```
dependencies {
    ....
    compile parent.ext.libNuwa
    ....
}
```
其中parent.ext.libNuwa在工程的总build.gradle定义，这里nuwa是被发布在私有maven仓库中，所以还要加入私有maven仓库的配置
```
buildscript {
    repositories {
        maven {
            url 'http://host/nexus/content/groups/public/'
        }
        ....
    }
}
    
allprojects {
    repositories {
        maven {
            url 'http://host/nexus/content/groups/public/'
        }
    }
    ....
    ext {
        ....
        libNuwa = 'com.eastmoney.nuwa:lib_nuwa:1.0.27'
        ....
    }
    ....
}
```
#### 2. 在应用的入口类 
也就是Application或者其子类，这里是在EmApplication类中
``` java
@Override
protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    ....
    //init nuwa
    Nuwa.initial(this);
    //load hotpatch if exist
    Nuwa.loadPatch(this, HotPatchUtils.getHotPatchDir(this), true);
    ....
}   
```
#### 3. 在需要触发下载逻辑的地方 
这里是在AppConfigManager类的saveAndUpdateCurrentConfig方法中，因为每次应用的配置发生变化后都会调用它
``` java
public void saveAndUpdateCurrentConfig(String currentConfig) {
    ....
    HotPatchManager manager = new HotPatchManager(ContextUtil.getContext());
    // currentConfig is the json String that contains 'hotFixMD5'
    manager.checkHotPatch(currentConfig);
    ....
｝
```
#### 4. 在需要检测是否有补丁下载失败的地方
这里是在AppConfigDownloadListener类的completed方法中，每次应用的配置无变化都会触发这里的逻辑
```java
@Override
public void completed(ResponseInterface resp) {
    ....
    HotPatchManager manager = new HotPatchManager(ContextUtil.getContext());
    manager.resumeFailedIfExist();
    ....
}
```
### nuwa_plugin的使用
#### 1. 在应用工程的总build.gradle中 
这里是将nuwaplugin发布到了私有maven库中，当前版本为1.0.14
```
buildscript {
    ....
    dependencies {
        ....
        classpath 'com.eastmoney.nuwaplugin:lib_nuwaplugin:1.0.14'
        ....
    }
    ....
}
```
#### 2. 在app的build.gradle中
```
apply plugin: 'lib_nuwaplugin'
nuwa {
    debugOn = true
    sdkModuleName = 'module_nuwa'
    unsignedHackApkPath = '/module_nuwa/hack/hack.apk'
    excludeClass = ['com.eastmoney.android.berlin.EMApplication']
}
```
debugOn表示打包debug的apk时是否开启nuwa  
unsignedHackApkPath表示上文提到的未签名的hack.apk的存放路径   
sdkModuleName表示签名后的hack.apk存放在哪个module的assets下  
excludeClass表示不要在编译时使用asm注入Hack.class引用的类  
#### 3.生成补丁包
编译需要打补丁的应用代码后，从*/应用根目录/app/build/outputs/*路径下拷贝出nuwa文件夹，这里假定拷贝至*/Users/nuwa*  
修复应用的缺陷之后，执行*gradle :app:assembleRelease -P NuwaDir=/Users/nuwa*，这里如果将assembleRelease替换为assembleDebug就是打出针对debug包的补丁  
如果编译正常，会在*/应用根目录/app/build/outputs/nuwa/渠道名/release/patch/*下生成补丁包

### 服务器端的配置
#### 1. 上传补丁包
将补丁包*patch.apk*上传至服务器，生成下载地址*downloadUrl*和文件*md5*
#### 2. 生成补丁配置文件
这里会生成一个json文件，对应一个数组，数组对应的JavaBean如下:
``` java
public class HotPatch {
    //客户端版本
    private String appVersion;
    //手机品牌
    private String brand;
    //第三方ROM会设置这个值
    private String display;
    //设备Android版本
    private String sdkVersion;
    //补丁包MD5
    private String md5;
    //补丁包下载地址
    private String downloadUrl;
    ....
}
```
客户端在触发请求补丁配置文件的逻辑时，服务器就会返回代表多个HotPatch对象的json。客户端根据运行环境的appVersion、brand、display、sdkVersion这几个维度从中获取匹配的HotPatch对象，最后根据md5和downloadUr完成补丁包的下载。
