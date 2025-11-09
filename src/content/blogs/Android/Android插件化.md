---
title: Android插件化技术
date: 2025-10-14
category: 安卓
tags: [Android, 插件化]
excerpt: 这篇文章将向你介绍安卓插件化的基本概念和相关技术点
---
最近在工作中要为其他公司提供插件，就接触到了插件化技术，学习了一下并记录整理出以下的学习笔记
# 认识插件化
## 插件化是什么
Android应用插件化是一种在Android平台上实现动态加载和运行插件（插件APK）的技术。传统的Android应用开发方式是将所有的功能和代码都打包在一个APK文件中，然后将该APK文件安装到设备上。而应用插件化则是将应用的部分或全部功能打包为插件APK，通过动态加载和运行插件APK，实现应用的灵活扩展和功能模块的动态更新。
插件化是一种动态加载四大组件的技术。最早为了解决65535限制的问题而诞生，后来Google出来了multidex专门解决65535限制问题。目前市面的插件化一定程度上可以达到缩包目的，而且对项目组件化，工程职责颗粒化，模块低耦合有着莫大裨益。插件化同时能实现bug热修复，由于Davilk虚拟机存在，Java支持动态加载任意类。因为安卓系统在四大组件上做了限制，如果你尝试打开不在清单中的组件时，Android系统会让程序Crash。插件化本质上是绕过Android系统的管控，让我们的APP自由打开、使用四大组件。

## 插件化优点
插件化让 Apk 中的代码（主要是指 Android 组件）能够免安装运行，这样能够带来很多收益：
- 减少安装Apk的体积、按需下载模块
- 动态更新插件
- 宿主和插件分开编译，提升开发效率
- 解决方法数超过65535的问题
  想象一下，你的应用拥有 Native 应用一般极高的性能，又能获取诸如 Web 应用一样的收益。

## 与组件化的区别
**组件化**：是将一个App分成多个模块，每个模块都是一个组件（module），开发过程中可以让这些组件相互依赖或独立编译、调试部分组件，但是这些组件最终会合并成一个完整的Apk去发布到应用市场。
**插件化**：是将整个App拆分成很多模块，每个模块都是一个Apk（组件化的每个模块是一个lib），最终打包的时候将宿主Apk和插件Apk分开打包，只需发布宿主Apk到应用市场，插件Apk通过动态按需下发到宿主Apk。

## 插件化的技术难点
想让插件的Apk真正运行起来，首先要先能找到插件Apk的存放位置，然后我们要能解析加载Apk里面的代码。
但是光能执行Java代码是没有意义的，在Android系统中有四大组件是需要在系统中注册的，具体来说是在 Android 系统的 ActivityManagerService (AMS) 和 PackageManagerService (PMS) 中注册的，而四大组件的解析和启动都需要依赖 AMS 和 PMS，如何欺骗系统，让他承认一个未安装的 Apk 中的组件，如何让宿主动态加载执行插件Apk中 Android 组件（即 Activity、Service、BroadcastReceiver、ContentProvider、Fragment）等是插件化最大的难点。
另外，应用资源引用（特指 R 中引用的资源，如 layout、values 等）也是一大问题，想象一下你在宿主进程中使用反射加载了一个插件 Apk，代码中的 R 对应的 id 却无法引用到正确的资源，会产生什么后果。
总结一下，其实做到插件化的要点就这几个：
- 如何加载并执行插件 Apk 中的代码（ClassLoader Injection）
- 让系统能调用插件 Apk 中的组件（Runtime Container）
- 正确识别插件 Apk 中的资源（Resource Injection）

# 插件化的三层结构
![](/img/插件化/img.png)
## 代码级插件化（Dex / ClassLoader 层）
**能做什么**
- 动态加载外部 .dex / .apk 中的 Java 类并执行（new、反射、接口回调）。
- 实现 **热修复**（替换宿主中某些类实现）。
- 动态扩展业务逻辑（按需下载模块、插件化算法库等）。
- 支持每个插件独立的类空间（隔离类名冲突）。

**不能做的**
- 无法让插件的 Activity 直接由系统启动（插件未安装、未注册）。
- 无法自动访问插件的资源（R.id 等静态资源 ID 不在宿主资源表中）。
- 不会自动处理插件的 native .so（需额外处理）。

## 资源级插件化（Resources / AssetManager 层）
**能做什么**
- 在宿主进程中**使用插件 APK 中的资源**（布局、图片、string、style 等），从而实现 UI/主题的动态替换。
- 插件可以带自己的资源并被宿主读取与渲染。

**不能做的**
- 单靠资源级不能让插件的 Activity 自主运行（仍需组件层代理）。
- 资源 id（R.xxx）在宿主中通常不相同，直接用宿主的 id 无效（需用插件的 R 或通过 name 查找）。

## 组件级插件化（Activity / Service / Receiver / Provider）
**能做什么**
- 让插件内的四大组件看起来像正常安装的组件一样被系统启动与交互：
- 启动插件 Activity（通过 Proxy/Hook），并把生命周期回调传给插件逻辑。
- 启动/绑定插件 Service（通过 ProxyService 或 Hook AMS）。
- 注册/接收插件的 Broadcast（动态注册或 Hook 注册流程）。
- 访问插件的 ContentProvider（通常通过代理或在宿主进程注入 provider 实例）。

**不能做的**
- 在不做大量 Hook/代理的情况下，不能真正将插件当成独立安装包（系统层面仍认宿主为实际注册者）。
- 完全隔离的进程模型比较复杂（需要虚拟化/进程隔离方案）。

# 实现代码级插件化
首先跳转到[ART](https://kids1934.github.io/blogs/Android-Android_Runtime/)这篇了解一下Android的类加载模型，你会了解到我们要做插件化就要在DexClassLoader上做手脚。

## 如何加载插件中的类
要加载插件中的类，我们首先要在宿主那边创建一个自定义PluginClassLoader继承DexClassLoader，先看下 DexClassLoader 的构造函数需要哪些参数。
```Java  
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent) {
        // ...
    }
}
```
构造函数需要四个参数： **dexPath** 是需要加载的 dex / apk / jar 文件路径 **optimizedDirectory** 是 dex 优化后存放的位置，在 ART 上，会执行 oat 对 dex 进行优化，生成机器码，这里就是存放优化后的 odex 文件的位置 **librarySearchPath** 是 native 依赖的位置 **parent** 就是父类加载器，默认会先从 parent 加载对应的类。
接下来要自定义PluginClassLoader，一个典型的插件化框架做法是：
```Java  
public class PluginClassLoader extends DexClassLoader {
    private final ClassLoader hostClassLoader;

    public PluginClassLoader(String dexPath, String optimizedDirectory, 
                             String librarySearchPath, ClassLoader parent) {
        super(dexPath, optimizedDirectory, librarySearchPath, parent);
        this.hostClassLoader = parent;
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // 1️⃣ 优先加载插件自己的类
        try {
            return findClass(name);
        } catch (ClassNotFoundException e) {
            // ignore
        }

        // 2️⃣ 否则交给宿主ClassLoader加载
        return hostClassLoader.loadClass(name);
    }
}
```
![](/img/插件化/Pasted%20image%2020251109115054.png)
**加载外部插件 APK**
```Java
File dexOutputDir = context.getDir("dex", 0);
String apkPath = "/sdcard/plugin_test.apk";

ClassLoader pluginClassLoader = new PluginClassLoader(
    apkPath,
    dexOutputDir.getAbsolutePath(),
    null,
    context.getClassLoader()
);

// 加载插件类
Class<?> pluginActivity = pluginClassLoader.loadClass("com.example.plugin.MainActivity");
Object instance = pluginActivity.newInstance();
```
# 实现组件级插件化
## Activity插件化
四大组件的插件化是插件化技术的核心知识点，而Activity插件化更是重中之重，Activity插件化主要有三种实现方式，分别是反射实现、接口实现和Hook技术实现。反射实现会对性能有所影响，主流的插件化框架没有采用此方式，关于接口实现可以阅读dynamic-load-apk的源码，这里不做介绍，目前Hook技术实现是主流，因此主要介绍Hook技术实现。 Hook技术实现主要有两种解决方案 ，一种是通过Hook IActivityManager来实现，另一种是Hook Instrumentation实现。在讲到这两个解决方案前，我们需要从整体上了解Activity的启动流程。

### Activity的启动过程
Activity的启动过程主要分为两种，一种是根Activity的启动过程，一种是普通Activity的启动过程。根Activity的启动过程如下图所示。
![](/img/插件化/Pasted%20image%2020251109115158.png)
首先Launcher进程向AMS请求创建根Activity，AMS会判断根Activity所需的应用程序进程是否存在并启动，如果不存在就会请求Zygote进程创建应用程序进程。应用程序进程启动后，AMS会请求应用程序进程创建并启动根Activity。 普通Activity和根Activity的启动过程大同小异，但是没有这么复杂，因为不涉及应用程序进程的创建，跟Laucher也没关系，如下图所示。
![](/img/插件化/Pasted%20image%2020251109115203.png)
上图给出了普通Acticity的启动过程。在应用程序进程中的Activity向AMS请求创建普通Activity（步骤1），AMS会对 这个Activty的生命周期管和栈进行管理，校验Activity等等。如果Activity满足AMS的校验，AMS就会请求应用程序进程中的ActivityThread去创建并启动普通Activity（步骤2）。
### Hook IActivityManager方案实现
AMS是在SystemServer进程中，我们无法直接进行修改，只能在应用程序进程中做文章。可以采用预先占坑的方式来解决没有在AndroidManifest.xml中显示声明的问题，具体做法就是在上图步骤1之前使用一个在AndroidManifest.xml中注册的Activity来进行占坑，用来通过AMS的校验。 接着在步骤2之后用插件Activity替换占坑的Activity

#### 注册Activity进行占坑
先创建一个SubActivity用来占坑，在AndroidManifest.xml中注册SubActivity，如下所示。
```XML
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.liuwangshu.pluginactivity">
    <application>
       ...
        <activity android:name=".StubActivity"/>
    </application>
</manifest>
```
TargetActivity用来代表已经加载进来的插件Activity，因此不需要在AndroidManifest.xml进行注册。如果我们直接在MainActivity中启动TargetActivity肯定会报错

#### 使用占坑Activity通过AMS验证
Android 8.0与7.0的AMS家族有一些差别，主要是Android 8.0去掉了AMS的代理ActivityManagerProxy，代替它的是IActivityManager，直接采用AIDL来进行进程间通信。 Android7.0的Activity的启动会调用ActivityManagerNative的getDefault方法，如下所示。
```Java  
 static public IActivityManager getDefault() {
        return gDefault.get();
    }
 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```
getDefault方法返回了IActivityManager类型的对象，IActivityManager借助了Singleton类来实现单例，而且gDefault又是静态的，因此IActivityManager是一个比较好的Hook点。
Android8.0的Activity的启动会调用ActivityManager的getService方法，如下所示。
```Java  
  public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```
同样的，getService方法返回了了IActivityManager类型的对象，并且IActivityManager借助了Singleton类来实现单例，确定了无论是Android7.0还是Android8.0，IActivityManager都是比较好的Hook点。Singleton类如下所示，后面会用到。
```Java  
public abstract class Singleton<T> {
    private T mInstance;
    protected abstract T create();
    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```
由于Hook需要多次对字段进行反射操作，先写一个字段工具类FieldUtil：
```Java  
public class FieldUtil {
    public static Object getField(Class clazz, Object target, String name) throws Exception {
        Field field = clazz.getDeclaredField(name);
        field.setAccessible(true);
        return field.get(target);
    }
    public static Field getField(Class clazz, String name) throws Exception{
        Field field = clazz.getDeclaredField(name);
        field.setAccessible(true);
        return field;
    }
    public static void setField(Class clazz, Object target, String name, Object value) throws Exception {
        Field field = clazz.getDeclaredField(name);
        field.setAccessible(true);
        field.set(target, value);
    }
```
其中setField方法不会马上用到，接着定义替换IActivityManager的代理类IActivityManagerProxy，如下所示。
```Java  
public class IActivityManagerProxy implements InvocationHandler {
    private Object mActivityManager;
    private static final String TAG = "IActivityManagerProxy";
    public IActivityManagerProxy(Object activityManager) {
        this.mActivityManager = activityManager;
    }
    @Override
    public Object invoke(Object o, Method method, Object[] args) throws Throwable {
        if ("startActivity".equals(method.getName())) {//1
            Intent intent = null;
            int index = 0;
            for (int i = 0; i < args.length; i++) {
                if (args[i] instanceof Intent) {
                    index = i;
                    break;
                }
            }
            intent = (Intent) args[index];
            Intent subIntent = new Intent();//2
            String packageName = "com.example.liuwangshu.pluginactivity";
            subIntent.setClassName(packageName,packageName+".StubActivity");//3
            subIntent.putExtra(HookHelper.TARGET_INTENT, intent);//4
            args[index] = subIntent;//5
        }
        return method.invoke(mActivityManager, args);
    }
}
```
Hook点IActivityManager是一个接口，建议采用动态代理。在注释1处拦截startActivity方法，接着获取参数args中第一个Intent对象，它是原本要启动插件TargetActivity的Intent。注释2、3处新建一个subIntent用来启动的StubActivity，注释4处将这个TargetActivity的Intent保存到subIntent中，便于以后还原TargetActivity。注释5处用subIntent赋值给参数args，这样启动的目标就变为了StubActivity，用来通过AMS的校验。 最后用代理类IActivityManagerProxy来替换IActivityManager，如下所示。
```Java
public class HookHelper {
    public static final String TARGET_INTENT = "target_intent";
    public static void hookAMS() throws Exception {
        Object defaultSingleton=null;
        if (Build.VERSION.SDK_INT >= 26) {//1
            Class<?> activityManageClazz = Class.forName("android.app.ActivityManager");
            //获取activityManager中的IActivityManagerSingleton字段
            defaultSingleton=  FieldUtil.getField(activityManageClazz ,null,"IActivityManagerSingleton");
        } else {
            Class<?> activityManagerNativeClazz = Class.forName("android.app.ActivityManagerNative");
            //获取ActivityManagerNative中的gDefault字段
            defaultSingleton=  FieldUtil.getField(activityManagerNativeClazz,null,"gDefault");
        }
        Class<?> singletonClazz = Class.forName("android.util.Singleton");
        Field mInstanceField= FieldUtil.getField(singletonClazz ,"mInstance");//2
        //获取iActivityManager
        Object iActivityManager = mInstanceField.get(defaultSingleton);//3
        Class<?> iActivityManagerClazz = Class.forName("android.app.IActivityManager");
        Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class<?>[] { iActivityManagerClazz }, new IActivityManagerProxy(iActivityManager));
        mInstanceField.set(defaultSingleton, proxy);
    }
}
```
首先在注释1处对系统版本进行区分，最终获取的是 Singleton&lt;IActivityManager&gt;类型的IActivityManagerSingleton或者gDefault字段。注释2处获取Singleton类中的mInstance字段，从前面Singleton类的代码可以得知mInstance字段的类型为T，在注释3处得到IActivityManagerSingleton或者gDefault字段中的T的类型，T的类型为IActivityManager。最后动态创建代理类IActivityManagerProxy，用IActivityManagerProxy来替换IActivityManager。 自定义一个Application，在其中调用HookHelper 的hookAMS方法，如下所示。
```Java  
public class MyApplication extends Application {
    @Override
    public void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        try {
            HookHelper.hookAMS();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
在MainActivity中启动TargetActivity，如下所示。
```Java  
public class MainActivity extends Activity {
    private Button bt_hook;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bt_hook = (Button) this.findViewById(R.id.bt_hook);
        bt_hook.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(MainActivity.this, TargetActivity.class);
                startActivity(intent);
            }
        });
    }
}
```
点击Button时，启动的并不是TargetActivity而是SubActivity，同时Log中打印了"hook成功"，说明我们已经成功用SubActivity通过了AMS的校验。

#### 还原插件Activity
前面用占坑Activity通过了AMS的校验，但是我们要启动的是插件TargetActivity，还需要用插件TargetActivity来替换占坑的SubActivity，这一替换的时机就在之前Activtiy启动流程图的步骤2之后。 ActivityThread启动Activity的过程，如下图所示。
![](/img/插件化/Pasted%20image%2020251109115415.png)
### Hook Instrumentation
#### Hook技术概述
Hook技术的核心实际上是动态分析技术，动态分析是指在程序运行时对程序进行调试的技术。众所周知，Android系统的代码和回调是按照一定的顺序执行的，这里举一个简单的例子，如图所示。
![](/img/插件化/Pasted%20image%2020251109115446.png)
对象A调用类对象B，对象B处理后将数据回调给对象A。接下来看看采用Hook的调用流程，如下图：
![](/img/插件化/Pasted%20image%2020251109115451.png)
上图中的Hook可以是一个方法或者一个对象，它就像一个钩子一样，始终连着AB，在AB之间互传信息的时候，hook会在中间做一些处理，比如修改方法的参数和返回值等，就这样hook起到了欺上瞒下的作用，我们把hook的这种行为称之为劫持。同理，大家知道，系统进程和应该进程之间是相互独立的，应用进程要想直接去修改系统进程，这个是很难实现的，有了hook技术，就可以在进程之间进行行为更改了。如图所示：
![](/img/插件化/Pasted%20image%2020251109115509.png)
可见，hook将自己融入到它所劫持的对象B所在的进程中，成为系统进程的一部分，这样我们就可以通过hook来更改对象B的行为了，对象B就称为hook点。

#### Hook Instrumentation
上面讲了Hook可以劫持对象，被劫持的对象叫hook点，用代理对象来替代这个Hook点，这样我们就可以在代理上实现自己想做的操作。这里我们用Hook startActivity来举例。Activity的插件化中需要解决的一个问题就是启动一个没有在AndroidManifest中注册的Activity，如果按照正常的启动流程是会报crash的。这里先简要介绍一下Activity的启动，具体的启动方式讲解还需移步专门的文献。

#### Activity的Hook点
启动Activity时应用进程会发消息给AMS，请求AMS创建Activity，AMS在SystemServer系统进程中，其与应用进程是隔离的，AMS管理所有APP的启动，所以我们无法在系统进程下做hook操作，应该在应用进程中。为了绕过AMS的验证，我们需要添加一个在Manifest中注册过的Activity，这个Activity称为**占坑**，这样可以达到欺上瞒下的效果，当AMS验证通过后再用插件Activity替换占坑去实现相应的功能。 核心功能两点：
- 替换插件Activity为占坑Activity
- 绕过AMS验证后需要还原插件Activity
  启动Activity的时候会调用Activity的startActivity（）如下：
```Java
   @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
```
接着又调用了startActivity（）
```Java
    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```
查看startActivityForResult方法
```Java  
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```
上述方法中调用mInstrumentation的execStartActivity方法来启动Activity，这个mInstrumentation是Activity的成员变量，我们就选择Instrumentation为Hook点，用代理的Instrumentation去替换原始的Instrumentation来完成Hook，如下是代理类：
```Java  
public class InstrumentationProxy extends Instrumentation {

    private Instrumentation mInstrumentation;
    private PackageManager mPackageManager;

    public InstrumentationProxy(Instrumentation instrumentation, PackageManager packageManager) {
        this.mInstrumentation = instrumentation;
        this.mPackageManager = packageManager;
    }

    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {

        List<ResolveInfo> resolveInfo = mPackageManager.queryIntentActivities(intent, PackageManager.MATCH_ALL);
        //判断启动的插件Activity是否在AndroidManifest.xml中注册过
        if (null == resolveInfo || resolveInfo.size() == 0) {
            //保存目标插件
            intent.putExtra(HookHelper.REQUEST_TARGET_INTENT_NAME, intent.getComponent().getClassName());
            //设置为占坑Activity
            intent.setClassName(who, "replugin.StubActivity");
        }

        try {
            Method execStartActivity = Instrumentation.class.getDeclaredMethod("execStartActivity",
                    Context.class, IBinder.class, IBinder.class, Activity.class,
                    Intent.class, int.class, Bundle.class);
            return (ActivityResult) execStartActivity.invoke(mInstrumentation, who, contextThread, token, target, intent, requestCode, options);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return null;
    }

    public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException,
            IllegalAccessException, ClassNotFoundException {
        String intentName = intent.getStringExtra(HookHelper.REQUEST_TARGET_INTENT_NAME);
        if (!TextUtils.isEmpty(intentName)) {
            return super.newActivity(cl, intentName, intent);
        }
        return super.newActivity(cl, className, intent);
    }

}
```
InstrumentationProxy类继承类Instrumentation，实现了类execStartActivity方法，接着通过反射去用原始Instrumentation的execStartActivity方法，这就是替换为占坑Activity的过程。Activity的创建是在ActivityThread中，里面有个performLaunchActivity方法
```Java 
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    }
    ...
    activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
    ...
}
```
这里的newActivity就是创建Activity的过程，我们同样的在代理类中去实现这个方法，这就是还原插件Activity 的过程。
接下来我们看个例子： 占位坑Activity：
```Java
public class StubActivity extends BaseActivity {
    @Override
    public int bindLayout() {
        return R.layout.activity_stub;
    }

    @Override
    public void initViews() {
    }

    @Override
    public void onClick(View v) {

    }
}
```
这个Activity一定是需要在AndroidManifest中去注册。 再写一个插件Activity
```Java
public class TargetActivity extends BaseActivity {
    @Override
    public int bindLayout() {
        return R.layout.activity_target;
    }

    @Override
    public void initViews() {

    }

    @Override
    public void onClick(View v) {

    }
}
```
都是很简单的Activity，TargetActivity并没有注册，现在我们需要启动这个Activity。代理类上面代码已经贴出来了。接下来就是替换代理类，达到Hook的目的，我们在Application中做这个事情：
```Java
public class MyApplication extends Application {

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        hookActivityThreadInstrumentation();
        
    }

    private void hookActivityThreadInstrumentation() {
        try {
            Class<?> activityThreadClass=Class.forName("android.app.ActivityThread");
            Field activityThreadField=activityThreadClass.getDeclaredField("sCurrentActivityThread");
            activityThreadField.setAccessible(true);
            //获取ActivityThread对象sCurrentActivityThread
            Object activityThread=activityThreadField.get(null);

            Field instrumentationField=activityThreadClass.getDeclaredField("mInstrumentation");
            instrumentationField.setAccessible(true);
            //从sCurrentActivityThread中获取成员变量mInstrumentation
            Instrumentation instrumentation= (Instrumentation) instrumentationField.get(activityThread);
            //创建代理对象InstrumentationProxy
            InstrumentationProxy proxy=new InstrumentationProxy(instrumentation,getPackageManager());
            //将sCurrentActivityThread中成员变量mInstrumentation替换成代理类InstrumentationProxy
            instrumentationField.set(activityThread,proxy);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```
这样就把原始的Instrumentation替换为代理的了，具体的操作我们在InstrumentationProxy中去做实现。接下来我们就是从主界面跳转插件Activity了：
```Java
public class PluginActivity extends BaseActivity {
    @Override
    public int bindLayout() {
        return R.layout.activity_stub;
    }

    @Override
    public void initViews() {
        Log.d("", "initViews: ");
        findViewById(R.id.btn_start_replugin).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startActivity(new Intent(PluginActivity.this, TargetActivity.class
                ));
            }
        });
    }

    @Override
    public void onClick(View v) {

    }

    public static void startActivity(Context context) {
        Intent i = new Intent(context, PluginActivity.class);
        context.startActivity(i);
    }

}
```
# 其它插件化使用相关
1. 在shadow框架中对于"file://"资源的处理:由于file://资源是在插件内部的，但是因为宿主在查找文件的时候只会去宿主的asset中找，所以框架会先把file://协议拦截并转化成http://协议去欺骗宿主，后续再将http://转化回file://协议来找到插件内资源
2. 在插件化中，插件的Manifest文件中的配置是不会被读取的，也就是说所有的配置都必须在代码中动态指定，不能在清单文件中通过属性的方式来声明
3. 插件的包名必须与宿主的包名保持一致，不然插件的application如果作为context传递会出现包名不被系统认可的情况
4. 在插件的清单文件中,application,activity,service等最好用全类名的方式，不然可能会发生冲突