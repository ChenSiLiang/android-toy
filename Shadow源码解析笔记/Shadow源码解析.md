## 0 引言

插件化一直以来都被视为Android中一门高深莫测的学问，它需要解决一系列难题：

- 四大组件的调用
- 如果使用插件的资源
- 尽可能减少hook系统API，降低兼容难度
- 尽量避免宿主的体积增量

腾讯最近开源的Tencent Shadow分享了很多设计细节和解决思路。对比之前的插件化框架，其优势在于零反射，无入侵性且零增量。对于有些未接触过或对插件化比较陌生的同学，整个流程可能比较难以一下看懂。下面先分析整体执行流程，再研究其中的代码细节是如何解决上述难题的。

先阅读Github的文档有助于理解。而编译代码的过程已省去，按照Github上的步骤可以顺利完成。

## 1 由宿主应用开始

本地启动的顺序

- HostApplication内进行恢复上次runtime和得到插件目录。其中PluginHelper.init方法指定了pluginManagerFile、pluginZipFile并从assets拷贝到"/data/…/files/"目录下
- MainAcitiviy -> PluginLoadActivity -> PluginManager.enter 此时还是在host内 
- SplashActivity -> 其他Activity 此时是业务方app的实现

因此重点在于1、2步。

从尾到头进行回溯。

最终是调用

```java
DynamicPluginManager.enter(Context context, long fromId, Bundle bundle, EnterCallback callback)
```

## 2 Plugin Manager

DynamicPluginManager.enter方法做了三件事

- updateManagerImpl比对上次更新时间，若不相等，则以最新文件进行更新，创建SamplePluginManager（在manager包内的pluginManagerFile内构造出ManagerFactoryImpl再构造出来dynamic包内的SamplePluginManager），属于动态化的子功能。
- SamplePluginManager.enter启动activity（前一半在宿主进程内）
- 根据最新文件更新PluginManager

 重点看第2步：

- 调用callback.showLoading

- installPlugin从压缩包中解压插件，更新数据库，安装插件的runtime，loader，.so

- startPluginActivity -> BinderPluginLoader.convertActivityIntent 该过程详细看下一章

  - loadPlugin -> bindPluginProcessService

    - loadRuntime 跨进程load runtime，hack class loader
    - loadPluginLoader 跨进程传uuid，插件进程根据uuid初始化pluginLoader，将其传回客户端
    - PluginLoader.loadPlugin(partKey) 跨进程调用mDynamicPluginLoader.loadPlugin(_arg0)。先初始化PluginServiceManager，再调用LoadPluginBloc.loadPlugin
    - 在HostApplication.loadPlugin加载插件到classloader中，调用createShadowApplication初始化插件application，将所包含的信息注册保存

  - 跨进程调用mDynamicPluginLoader.callApplicationOnCreate
  - 跨进程调用mDynamicPluginLoader.convertActivityIntent 确认启动的类名在插件中存在，将类信息放到intent内，在host进程内启动对应Activity。

- 调用callback.closeLoading

## 3 FastPluginManager.convertActivityIntent

| 宿主进程                          | Binder            | 传递跨进程对象       | 插件进程             | 作用                              |
| --------------------------------- | ----------------- | -------------------- | -------------------- | --------------------------------- |
| PluginManagerThatUseDynamicLoader | PpsBinder         | << PpsController     | PluginProcessService |                                   |
| BinderUuidManager                 | UuidManagerBinder | >> UuidManagerBinder | PluginProcessService | 根据uuid和APPID获取对应的插件信息 |
| BinderPluginLoader                | PpsBinder         | << PluginLoaderImpl  | PluginProcessService | 转换intent                        |

插件加载是重点，我们从loadPlugin作为起点开始看。

### 3. 1 PluginLoader.loadPlugin

```java
// 位于宿主进程
FastPluginManager.loadPlugin(partKey); // loadPluginLoaderAndRuntime
// 位于插件进程
LoadPluginBloc.loadPlugin(){
    // 在这里初始化PluginServiceManager
    // 加载插件到ClassLoader中.
    LoadApkBloc.loadPlugin
   	// 获取宿主包信息包括activity、meta_data、service、provider和native路径等
    PackageManager.getPackageArchiveInfo
    // 解析插件apk，将插件内的信息放到pluginInfo内
    ParsePluginApkBloc.parse
    // new PluginPackageManager
    PluginPackageManager(hostPackageManager, packageInfo, allPluginPackageInfo)
    // 创建资源resource
    CreateResourceBloc.create
    // 创建shadow Application
    CreateApplicationBloc.createShadowApplication
    // 解析插件apk
    ParsePluginApkBloc.parse
    // 将插件信息放到各个map内，包括类名，Activity，services，provider
    ComponentManager.addPluginApkInfo(pluginInfo);
    // 添加<partKey, PluginParts>键值对
    pluginPartsMap[]
    // 添加<classLoader, pluginPartInfo>键值对
    PluginPartInfoManager.addPluginInfo();
 }
// 调用shadow applciation.oncreate方法
pluginLoader.callApplicationOnCreate(partKey);
DynamicPluginLoader.convertActivityIntent();
// 最后启动插件内的plashActivity
```

#### 3.1.1 loadPluginLoaderAndRuntime

- bind PPS拿到插件进程PluginProcessService的接口PpsController，不赘述
- 跨进程loadRuntime
  - 在插件进程的PPS内，使用mUuidManager（即从宿主进程传递来的binder）反过来调用宿主进程的PluginManagerThatUseDynamicLoader.getRuntime，构造一个InstalledApk返回到PPS内
  - 使用InstalledApk内的path来调用DynamicRuntime.loadRuntime
  - 关键是DynamicRuntime.hackParentToRuntime，将RuntimeClassLoader（这个就是下面说的DexClassLoader）插入到BootClassLoader与之间。
  - 保存结果，可以用作runtime恢复

更具体的原理可以参考作者的文章[Shadow的全动态设计原理解析](<https://juejin.im/post/5d1b466f6fb9a07ed524b995>)，对应container动态化。

#### 3.1.2 loadPluginLoader

- 跟上一步骤类似取到InstalledApk，调用new LoaderImplLoader().load

  - 使用ApkClassLoader.getInterface获取LoaderFactory，ApkClassLoader主要有两个功能：
    - 实现不正常的双亲委派逻辑，既能和parent隔离类加载，也能通过白名单复用一些类
    - 封装了加载对象强转成接口类型的方法
  - LoaderFactory.build返回PluginLoaderImpl。LoaderFactory不是为了返回多种Loader实现，只是为了让构造参数能在编译期进行检查，而不是通过反射方法构造的，在参数不对应时编译期检查不出来。

更具体的原理可以参考作者的文章[Shadow的全动态设计原理解析](<https://juejin.im/post/5d1b466f6fb9a07ed524b995>)，对应关于动态加载接口实现的实践经验。

####  3.1.3 LoadApkBloc.loadPlugin

- 先通过Logger.classLoader.parent取到宿主的hostParentClassLoader。
- 如果dependson为null，则new PluginClassLoader，这个就是下面说的DexClassLoader
- loadClass方法判断如果hostParentClassLoader!=null || 是白名单 || other 走双亲委派模型，否则先尝试自己加载，再走双亲委派模型。

> 所有的插件框架中，Activity的加载都是这样的，new一个DexClassLoader加载插件apk。然后从插件ClassLoader中load指定的插件Activity名字，newInstance之后强转为Activity类型使用。实际上Android系统自身在启动Activity时也是这样做的。所以这就是插件机制能动态更新Activity的基本原理。
>
> ...
>
> 因此，我们可以通过修改ClassLoader的parent，为ClassLoader新增一个parent。将原本的`BootClassLoader <- PathClassLoader`结构变为`BootClassLoader <- DexClassLoader <- PathClassLoader`，插入的DexClassLoader加载了ContainerActivity就可以使得系统在向PathClassLoader查找ContainerActivity时能够正确找到实现。

更具体的原理可以参考作者的文章[Shadow的全动态设计原理解析](<https://juejin.im/post/5d1b466f6fb9a07ed524b995>)，对应loader动态化。

#### 3.1.4 CreateApplicationBloc.createShadowApplication

集合上述加载的部件

- 使用工厂模式生产的ShadowPluginLoader
- 宿主的资源Resource
- 插件的包信息
- 宿主的包信息
- 添加ComponentManager，主要功能是管理组件和宿主中注册的壳子之间的配对关系，这个在第一步生成ShadowPluginLoader会new ComponentManager

初始化

- 注册静态自定义广播接收器的类名，该类名是loader包里的广播类
- 设置业务名pluginInfo.businessName和partKey
- 设置ShadowRemoteViewCreatorImp

#### 3.1.5 ComponentManager.addPluginApkInfo(pluginInfo）

Activity和Service需要更新packageNameMap，pluginInfoMap，pluginComponentInfoMap，其中packageNameMap是<类名，包名>，pluginInfoMap<ComponentName, PluginInfo>，pluginComponentInfoMap<ComponentName， PluginComponentInfo>。

另外

- 插件内的Activity，更新componentMap<componentName, new ComponentName(context, DEFAULT_ACTIVITY)>
- 插件内的Providers，通过mPluginContentProviderManager.addContentProviderInfo更新providerAuthorityMap<plugin.authority, host.authority>

#### 3.1.6 callApplicationOnCreate

- 手动调用ShadowApplication.attachBaseContext(mHostAppContext)
- 调用mPluginContentProviderManager.createContentProviderAndCallOnCreate，调用contentProvider.attachInfo
- 手动调用ShadowApplication.onCreate

#### 3.1.7 convertActivityIntent(pluginIntent)

如果请求的是插件内的组件，则写入下列信息

- 类名、包名
- 参数，业务名
- partkey，pluginloader，版本号，进程ID

最后启动Activity。

### 3.2 FastPluginManager.installPlugin

回过头来看loadPlugin之前的installPlugin。

```java
// 解压插件
installPluginFromZip
// odex优化
oDexPluginLoaderOrRunTime
//插件apk的so解压
extractSo
// odex优化
 oDexPlugin
```

#### 3.2.1 installPluginFromZip

![0](/Users/doushabao/information/学习资料/笔记/shadow/pic/0.png)

- 将插件zip解压到名为zip的MD5的文件夹内，例如/data/user/0/包名/files/ShadowPluginManager/UnpackedPlugin/test-dynamic-manager/c35f2f6aa2c138b28e59e9a0d947cea0/plugin-debug.zip/"压缩包内容名"，其中从ShadowPluginManager开始为shadow新增的文件夹名，从UnpackedPlugin后为某个特定插件的名字，例如这里是test-dynamic-manager。

- 读取配置文件config.json，里面包括版本信息，pluginLoader信息，plugins信息

  ```json
  {"compact_version":[1,2,3],"pluginLoader":{"apkName":"sample-loader-debug.apk","hash":"D1CBB814545373A66537465BAD8F37B5"},"plugins":[{"partKey":"sample-plugin-app","apkName":"sample-plugin-app-debug.apk","businessName":"sample-plugin-app","hostWhiteList":["com.tencent.shadow.sample.host.lib"],"hash":"504495C8C3016B51B6E65EA9816C7955"}],"runtime":{"apkName":"sample-runtime-debug.apk","hash":"AE00182BE7F8D3D459B1BD4F0AE06E47"},"UUID":"684E32CC-4F8C-4FA2-A60F-66EAB532EDC1","version":4,"UUID_NickName":"1.1.5"}
  ```

- 将config插入数据库，包括loader.apk, runtime.apk, plugin-app.apk

#### 3.2.2 oDexPluginLoaderOrRunTime

在低于API 26时，使用DexClassLoader时需要指定生成odex的optimizedDirectory，odex是经过优化后的dex文件。

odex文件有两个优点：

- 体积比dex更小，它是应用启动前已优化的部分程序。
- 更安全，令攻击者更难读取应用内容

指定odex目录后创建DexclassLoader，其构造方法会最终调用DexPathList.makeDexElements加载dex文件，最终将odex路径赋值给RuntimeClassLoader，ApkClassLoader。

#### 3.2.3 extractSo

将插件依赖的so文件复制到 "lib/" 目录下，更新数据库关于so的path。

#### 3.2.4 oDexPlugin

最后插件下的目录为:

- md5: 存zip和解压文件
- lib：存.so
- oDex：存从apk解压出来的dex和odex文件

至此，启动插件内Activity的整体流程结束。回头看一下作者文章[Shadow解决Activity等组件生命周期的方法解析](<https://juejin.im/post/5d15d9a06fb9a07ef71087ca>) 里讲的：

> 我们的选择是在宿主PathClassLoader上给它加一个parent ClassLoader。因为PathClassLoader也是一个有正常“双亲委派”逻辑的ClassLoader，它加载什么类都会先问自己parent ClassLoader先加载。

也就对应了3.1.5节内的类加载器的实现。

再看一下壳子Activity复用的实现，作者对比了RePlugin框架需要在宿主中注册大量Activity，而Shadow用代理Activity通过Intent的参数确定构造哪个Activity，令壳子能被复用。这也是runtime包的动态实现功能之一。

打开宿主AndroidManifest文件，注册了三个launchMode分别为standard/singleInstance/singleTask的Activity，都继承于PluginContainerActivity。

主要实现的是这两点

- 插件Activity不继承Activity，通过AOP替换父类为ShadowActivity，继承了PluginActivity
- 插件Activity控制壳子Activity生命周期的调用
  - 在壳子PluginContainerActivity构造时，持有了被委托的接口delegate，可以调用插件的伪生命周期方法。接着调用了delegate.setDelegator(this)，为ShadowActivityDelegate赋值了壳子Activity的引用
  - 在ShadowActivityDelegate.onCreate方法内，通过PluginClassLoader.loadClass创建插件PluginActivity，再为插件PluginActivity赋值了壳子Activity的引用。ShadowActivityDelegate作为一个转调或者信使的功能。
- 由插件Activity直接指挥壳子Activity什么时候调用`super.onCreate()`

因此整个sample里的Activity启动流程为

MainActivity(Host) -> (上面一大堆加载) -> PluginDefaultProxyActivity(Host) -> PluginContainerActivity(host) ->  ShadowActivityDelegate(Host) -> SplashActivity(:plugin)

## 4 同名View的方法解析

> Android系统中存在一个错误的设计，导致LayoutInflater在inflate的时候使用的View类并不是简单的从ClassLoader中读取的，而是自行做了一层缓存，以View的类名作为Key保存了Class对象。这个缓存存储在一个静态域上，因此在同一个进程中，不能存在两个同名的View都使用LayoutInflater构造。这个问题常见于插件框架对于support包中的RecycleView支持上。

具体查看作者原文[Shadow解决插件和宿主有同名View的方法解析](<https://juejin.im/post/5d1c2150e51d4556dc29369a>)。

对应LayerInflater.createViewFromtag方法：

- 先用factory创建View
- 若没有，则走自己的createView方法

而上述问题出在第二步。翻查Android源码，最新版本已经修复了这个问题，通过verifyClassLoader来判断是否需要移除并置空缓存。但我们仍旧需要兼容旧版本，所以可以通过设置factory来进行自定义加载。

在Shadow内的实现，在壳子Activity内重写getLayerInflater方法，返回设置了factory2的自定义LayerInflater，具体请查看ShadowLayoutInflater，其中ShadowFactory2.onCreateView也是类似官方的修复一样来解决这个问题。

再看一下同名资源的处理，插件与宿主的是资源隔离的。因此插件只能访问到自己的资源，而宿主无从更新自身代码，也无需访问插件资源。

## 5 PackageManager

最终调用的方法顺序是 

CodeConverterExtension.redirectMethodCallToStaticMethodCall 

->PackageManagerTransform. setupPackageManagerTransform 

->PackageManagerInvokeRedirect.getPluginPackageManager

前两部是查找getPackageManager，生成静态的方法调用PackageManagerInvokeRedirect，内部的实现是通过classloader找到对应的PluginPartInfo，再取到相应的资源。

具体查看作者原文[Shadow对PackageManager的处理方法](<https://juejin.im/post/5d37dcf1e51d4510664d17d4>)。

## 6 三大组件的调用

其他三大组件并没有Activity复杂的生命周期，比较简单。

### 6.1 Service

启动流程为ShadowContext.startService -> ComponentManager.startService -> PluginServiceManager.stopPluginService，由3.1.5可知，插件Service已经在ComponentManager中注册了，先调用newServiceInstance通过pluginclassloader新建出Service，设置例如宿主Context、插件资源和loader之类，调用其onCreate方法。

其他方法类似。

### 6.2 ContentProvider

通过宿主PluginContainerContentProvider来转发方法。

### 6.3 BroadcastReveiver

 对于动态广播直接注册使用就可以了。

对于静态广播，手动将静态广播放到ShadowAcpplication内，在onCreate被调用时进行注册，具体看3.1.4小节。

## 7 总结

本文分析了插件化框架Shadow如何解决一系列难题，特别是从插件加载到启动Activity的整个流程。宿主仅在AndroidManifest注册了壳子Activity，真正位于dynamic包内。壳子Activity双向持有插件Activity的引用，来实现插件Activity伪生命周期的调用（插件Activity并不真正继承Activity类），仅通过插入一层classloader而不需要过分地hack系统API实现。其中，修改LayerInflater实现同名View和通过AOP修改PackageManager的设计思路十分巧妙，值得学习。

## 8 Reference

- Shadow源码 <https://github.com/Tencent/Shadow>
- 作者shifujun设计细节分享 <https://juejin.im/user/5d12faee6fb9a07ed8425178/posts>
- Android官网 bounde service <https://developer.android.com/guide/components/bound-services>
- Android source code <https://android.googlesource.com/platform/frameworks/base/+/master>
- Android内核—binder架构分析 <https://www.cnblogs.com/a284628487/p/3187320.html>
- Android LayoutInflater Factory 源码解析 <https://juejin.im/post/5b52ee765188251b176a666d>
- 【Android 修炼手册】常用技术篇 -- Android 插件化解析<https://juejin.im/post/5d235f8b51882554c007af06#heading-22>
- 关于odex，配置ART <https://source.android.com/devices/tech/dalvik/configure>

## 9 转载或引用

转载或引用请注明出处。