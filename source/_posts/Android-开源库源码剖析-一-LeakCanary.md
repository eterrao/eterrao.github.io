---
title: Android 开源库源码剖析 <一> LeakCanary
date: 2021-02-25 16:46:14
tags: Android, LeakCanary, 内存泄漏
---

`基于 LeakCanary v2.6 分支分析`

## 目录

- 1. 什么是内存泄漏？
- 2. 什么是  LeakCanary?
- 3. 如何使用 LeakCanary？
- 4. 如何实现一行代码接入？
- 5. 如何判断内存泄漏？
- 6. 如何分析 hprof 文件的？
- 7. 总结

## 1. 什么是内存泄漏

> 在 Java runtime 环境中，内存泄漏错误是因为应用保留一个长时间不用的对象的引用。结果为这个对象分配的内存无法回收，最后导致 App OutOfMemoryError(OOM) 崩溃。
>
> 例如 Android Activity 实例在 onDestroy() 后就不再需要，但是静态变量会持有这个引用导致其无法被垃圾回收器回收。

##  2. 导致内存泄漏的其他情况

> 大部分导致内存泄漏的 BUG 都与对象的生命周期有关系，以下是几个 Android 会引发内存泄漏常见的场景：
>
> -  `Fragment` 实例已经退出栈了，但是 `Fragment.onDestroyView()` 执行后实例没有被销毁（详情见[StackOverFlow](https://stackoverflow.com/a/59504797/703646)）。
> - 把  `Activity` 的实例充当 `Context` ，执行了 `onConfigurationChanged()`  zheli 导致 `Activity` 重新创建。
> - 注册 Listener 和广播接收器或者 RxJava 的订阅器与 LifeCycle 绑定注册，最后在生命周期结束时忘记解绑注册。
>

## 3. 如何使用 LeakCanary

```groovy
dependencies {
  // debugImplementation because LeakCanary should only run in debug builds.
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.6'
}
```

[官方 guideline](https://square.github.io/leakcanary/getting_started/) 

> LeakCanary 能够自动监测以下对象的泄漏：
>
> - 已销毁的 Activity 实例
> - 已销毁的 Fragment 实例
> - 已销毁的 Fragment View 实例
> - 已清除的 ViewModel 实例

## 4. 如何实现一行代码接入？

在 LeakCanary 的源码中找到 `AppWatcherInstaller` 这个类，这个类继承自 `ContentProvider` ，注释说明了两点：

- 1. `ContentProvider` 是在 `Application` 类之前被创建的
- 2. `AppWatcherInstaller`  用于在 Application start 的时候加载 `leakcanary.AppWatcher` 。

借助了 **ContentProvider 在 Application 类的创建前就已被加载了** 这个特性，LeakCanary 才无需我们手动接入额外的代码。

`AppWatcherInstaller.kt`

```kotlin
/**
 * Content providers are loaded before the application class is created.     
 * [AppWatcherInstaller] is
 * used to install [leakcanary.AppWatcher] on application start.
 */
internal sealed class AppWatcherInstaller : ContentProvider() {
  
  override fun onCreate(): Boolean {
    // 在 ContentProvider 的 onCreate() 方法中执行的 manualInstall
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }
}
```

至于 ContentProvider 的加载时机源码分析我会单独在四大组件启动的篇幅进行分析介绍。

## 5. 如何判断内存泄漏？

为了找到这个问题的答案，就需要深入看看源码，于是下面开始 LeakCanary 的源码分析：

先关注 `AppWatcher` 这个类的关键实现：

```kotlin
/**
 * The [ObjectWatcher] used by AppWatcher to detect retained objects.
 * Only set when [isInstalled] is true.
 * AppWatcher 使用 ObjectWatcher 来进行残留对象的监测
 * ObjectWatcher 的构造函数需要传入一个 Executor 对象
 */
val objectWatcher = ObjectWatcher(
        clock = { SystemClock.uptimeMillis() },
        checkRetainedExecutor = {
          check(isInstalled) {
            "AppWatcher not installed"
          }
          mainHandler.postDelayed(it, retainedDelayMillis)
        },
        isEnabled = { true }
)

@JvmOverloads
fun manualInstall(
        application: Application,
        retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
        watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application),
) {
    checkMainThread()
    check(!isInstalled) {
        "AppWatcher already installed"
    }
    check(retainedDelayMillis >= 0) {
        "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
    }
    this.retainedDelayMillis = retainedDelayMillis
    if (application.isDebuggableBuild) {
        LogcatSharkLog.install()
    }
    // Requires AppWatcher.objectWatcher to be set
    LeakCanaryDelegate.loadLeakCanary(application)
  	// 这里的 watchersToInstall 在下面的方法可以看到，最终加载的是 ActivityWatcher/FragmentAndViewModelWatcher/RootViewWatcher/ServiceWatcher 这四个默认的 Watcher,并且都实现了 InstallableWatcher 这个接口
    watchersToInstall.forEach {
        it.install()
    }
}

/**
 * Creates a new list of default app [InstallableWatcher], created with the passed in
 * [reachabilityWatcher] (which defaults to [objectWatcher]). Once installed,
 * these watchers will pass in to [reachabilityWatcher] objects that they expect to become weakly reachable.
 * The passed in [reachabilityWatcher] should probably delegate to [objectWatcher] but can be used to filter out specific instances.
 */
fun appDefaultWatchers(
        application: Application,
        reachabilityWatcher: ReachabilityWatcher = objectWatcher,
): List<InstallableWatcher> {
    return listOf(
            ActivityWatcher(application, reachabilityWatcher),
            FragmentAndViewModelWatcher(application, reachabilityWatcher),
            RootViewWatcher(reachabilityWatcher),
            ServiceWatcher(reachabilityWatcher)
    )
}
```

在 `appDefaultWatchers()` 中可以看到，分别对四个类型的 Watcher 进行加载：

-  ActivityWatcher
- FragmentAndViewModelWatcher
- RootViewWatcher
- ServiceWatcher 

这四个默认的 Watcher 都是 `InstallableWatcher` 这个接口的实现类。

```kotlin
interface InstallableWatcher {
  fun install()
  fun uninstall()
}
```

![image-20210225195641276](file:///Users/rmy/work/images/image-20210225195641276.png?lastModify=1614260104)

下面分别看看四个类内部实现：

 **ActivityWatcher**

```kotlin
/**
 * Expects activities to become weakly reachable soon after they receive the [Activity.onDestroy]
 * callback.
 */
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        // 1. Activity destroy 的时候将 activity 的弱引用加入引用队列 ReferenceQueue
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
    // 2.注册 Activity 的生命周期
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    // 3.解绑 Activity 的生命周期
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}

```

#### 关键逻辑

- 注释 1 可以看到重写 `Application.onActivityDestroyed()` 可以在 `Activity.onDestroy()` 调用的时候获取 Activity 的弱引用。`reachabilityWatcher.expectWeaklyReachable(activity)` 则是把 Activity 的弱引用加入引用队列。
- 注释 2、3 进行了 Application 生命周期回调的注册和解绑。

**FragmentAndViewModelWatcher**

```kotlin
private val fragmentDestroyWatchers: List<(Activity) -> Unit> = run {
    val fragmentDestroyWatchers = mutableListOf<(Activity) -> Unit>()
  	// 1
    if (SDK_INT >= O) {
        fragmentDestroyWatchers.add(
                AndroidOFragmentDestroyWatcher(reachabilityWatcher)
        )
    }
  	// 2
    getWatcherIfAvailable(
            ANDROIDX_FRAGMENT_CLASS_NAME,
            ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
            reachabilityWatcher
    )?.let {
        fragmentDestroyWatchers.add(it)
    }
  	// 3
    getWatcherIfAvailable(
            ANDROID_SUPPORT_FRAGMENT_CLASS_NAME,
            ANDROID_SUPPORT_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
            reachabilityWatcher
    )?.let {
        fragmentDestroyWatchers.add(it)
    }
    fragmentDestroyWatchers
}
private val lifecycleCallbacks =
        object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
            override fun onActivityCreated(
                    activity: Activity,
                    savedInstanceState: Bundle?,
            ) {
              	// 4
                for (watcher in fragmentDestroyWatchers) {
                    watcher(activity)
                }
            }
        }
```

相较于 Activity 的 Watcher，Fragment 的稍许特殊，有大概 3 种情况需要检测：

- Fragment
- FragmentView
- 基于 androidx 的 ViewModel（包含 Activity 和 Fragment 的 View Models）

在 `fragmentDestroyWatchers` 的回调方法 中，我们看到有 3 种 ReachabilityWatcher：

- 注释 1 判断系统版本大于 Android O 则使用 AndroidOFragmentDestroyWatcher
- 注释 2 `androidx.fragment.app.Fragment` 使用 `leakcanary.internal.AndroidXFragmentDestroyWatcher`
- 注释 3 `android.support.v4.app.Fragment` 使用 `leakcanary.internal.AndroidSupportFragmentDestroyWatcher`

上面的 3 个类都是 `ReachabilityWatcher` 的子类。

**RootViewWatcher**

```kotlin
/**
 * Expects root views to become weakly reachable soon after they are removed from the window
 * manager.
 */
class RootViewWatcher(
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  override fun install() {
    if (Build.VERSION.SDK_INT < 19) {
      return
    }
    swapViewManagerGlobalMViews { mViews ->
      object : ArrayList<View>(mViews) {
        override fun add(element: View): Boolean {
          onRootViewAdded(element)
          return super.add(element)
        }
      }
    }
  }

  override fun uninstall() {
    if (Build.VERSION.SDK_INT < 19) {
      return
    }
    swapViewManagerGlobalMViews { mViews ->
      ArrayList(mViews)
    }
  }

  @SuppressLint("PrivateApi")
  @Suppress("FunctionName")
  private fun swapViewManagerGlobalMViews(swap: (ArrayList<View>) -> ArrayList<View>) {
    try {
      val windowManagerGlobalClass = Class.forName("android.view.WindowManagerGlobal")
      val windowManagerGlobalInstance =
        windowManagerGlobalClass.getDeclaredMethod("getInstance").invoke(null)

      val mViewsField =
        windowManagerGlobalClass.getDeclaredField("mViews").apply { isAccessible = true }

      @Suppress("UNCHECKED_CAST")
      val mViews = mViewsField[windowManagerGlobalInstance] as ArrayList<View>

      mViewsField[windowManagerGlobalInstance] = swap(mViews)
    } catch (ignored: Throwable) {
      SharkLog.d(ignored) { "Could not watch detached root views" }
    }
  }

  private fun onRootViewAdded(rootView: View) {
    rootView.addOnAttachStateChangeListener(object : OnAttachStateChangeListener {

      val watchDetachedView = Runnable {
        reachabilityWatcher.expectWeaklyReachable(
          rootView, "${rootView::class.java.name} received View#onDetachedFromWindow() callback"
        )
      }

      override fun onViewAttachedToWindow(v: View) {
        mainHandler.removeCallbacks(watchDetachedView)
      }

      override fun onViewDetachedFromWindow(v: View) {
        mainHandler.post(watchDetachedView)
      }
    })
  }
}
```

**ServiceWatcher**

```kotlin
/**
 * Expects services to become weakly reachable soon after they receive the [Service.onDestroy]
 * callback.
 */
@SuppressLint("PrivateApi")
class ServiceWatcher(private val reachabilityWatcher: ReachabilityWatcher) : InstallableWatcher {

  private val servicesToBeDestroyed = WeakHashMap<IBinder, WeakReference<Service>>()

  private val activityThreadClass by lazy { Class.forName("android.app.ActivityThread") }

  private val activityThreadInstance by lazy {
    activityThreadClass.getDeclaredMethod("currentActivityThread").invoke(null)!!
  }

  private val activityThreadServices by lazy {
    val mServicesField =
      activityThreadClass.getDeclaredField("mServices").apply { isAccessible = true }

    @Suppress("UNCHECKED_CAST")
    mServicesField[activityThreadInstance] as Map<IBinder, Service>
  }

  private var uninstallActivityThreadHandlerCallback: (() -> Unit)? = null
  private var uninstallActivityManager: (() -> Unit)? = null

  override fun install() {
    checkMainThread()
    check(uninstallActivityThreadHandlerCallback == null) {
      "ServiceWatcher already installed"
    }
    check(uninstallActivityManager == null) {
      "ServiceWatcher already installed"
    }
    try {
      swapActivityThreadHandlerCallback { mCallback ->
        uninstallActivityThreadHandlerCallback = {
          swapActivityThreadHandlerCallback {
            mCallback
          }
        }
        Handler.Callback { msg ->
          if (msg.what == STOP_SERVICE) {
            val key = msg.obj as IBinder
            activityThreadServices[key]?.let {
              onServicePreDestroy(key, it)
            }
          }
          mCallback?.handleMessage(msg) ?: false
        }
      }
      swapActivityManager { activityManagerInterface, activityManagerInstance ->
        uninstallActivityManager = {
          swapActivityManager { _, _ ->
            activityManagerInstance
          }
        }
        Proxy.newProxyInstance(
          activityManagerInterface.classLoader, arrayOf(activityManagerInterface)
        ) { _, method, args ->
          if (METHOD_SERVICE_DONE_EXECUTING == method.name) {
            val token = args!![0] as IBinder
            if (servicesToBeDestroyed.containsKey(token)) {
              onServiceDestroyed(token)
            }
          }
          try {
            if (args == null) {
              method.invoke(activityManagerInstance)
            } else {
              method.invoke(activityManagerInstance, *args)
            }
          } catch (invocationException: InvocationTargetException) {
            throw invocationException.targetException
          }
        }
      }
    } catch (ignored: Throwable) {
      SharkLog.d(ignored) { "Could not watch destroyed services" }
    }
  }

  override fun uninstall() {
    checkMainThread()
    uninstallActivityManager?.invoke()
    uninstallActivityThreadHandlerCallback?.invoke()
    uninstallActivityManager = null
    uninstallActivityThreadHandlerCallback = null
  }

  private fun onServicePreDestroy(
    token: IBinder,
    service: Service
  ) {
    servicesToBeDestroyed[token] = WeakReference(service)
  }

  private fun onServiceDestroyed(token: IBinder) {
    servicesToBeDestroyed.remove(token)?.also { serviceWeakReference ->
      serviceWeakReference.get()?.let { service ->
        reachabilityWatcher.expectWeaklyReachable(
          service, "${service::class.java.name} received Service#onDestroy() callback"
        )
      }
    }
  }

  private fun swapActivityThreadHandlerCallback(swap: (Handler.Callback?) -> Handler.Callback?) {
    val mHField =
      activityThreadClass.getDeclaredField("mH").apply { isAccessible = true }
    val mH = mHField[activityThreadInstance] as Handler

    val mCallbackField =
      Handler::class.java.getDeclaredField("mCallback").apply { isAccessible = true }
    val mCallback = mCallbackField[mH] as Handler.Callback?
    mCallbackField[mH] = swap(mCallback)
  }

  @SuppressLint("PrivateApi")
  private fun swapActivityManager(swap: (Class<*>, Any) -> Any) {
    val singletonClass = Class.forName("android.util.Singleton")
    val mInstanceField =
      singletonClass.getDeclaredField("mInstance").apply { isAccessible = true }

    val singletonGetMethod = singletonClass.getDeclaredMethod("get")

    val (className, fieldName) = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      "android.app.ActivityManager" to "IActivityManagerSingleton"
    } else {
      "android.app.ActivityManagerNative" to "gDefault"
    }

    val activityManagerClass = Class.forName(className)
    val activityManagerSingletonField =
      activityManagerClass.getDeclaredField(fieldName).apply { isAccessible = true }
    val activityManagerSingletonInstance = activityManagerSingletonField[activityManagerClass]

    // Calling get() instead of reading from the field directly to ensure the singleton is
    // created.
    val activityManagerInstance = singletonGetMethod.invoke(activityManagerSingletonInstance)

    val iActivityManagerInterface = Class.forName("android.app.IActivityManager")
    mInstanceField[activityManagerSingletonInstance] =
      swap(iActivityManagerInterface, activityManagerInstance!!)
  }

  companion object {
    private const val STOP_SERVICE = 116

    private const val METHOD_SERVICE_DONE_EXECUTING = "serviceDoneExecuting"
  }
}
```

四个会出现内存泄漏的类细节都了解了以后，来看看另一个关键类：

**ObjectWatcher** 

```kotlin
/**
 * [ObjectWatcher] can be passed objects to [watch]. It will create [KeyedWeakReference] instances
 * that reference watches objects, and check if those references have been cleared as expected on
 * the [checkRetainedExecutor] executor. If not, these objects are considered retained and
 * [ObjectWatcher] will then notify registered [OnObjectRetainedListener]s on that executor thread.
 *
 * [checkRetainedExecutor] is expected to run its tasks on a background thread, with a significant
 * delay to give the GC the opportunity to identify weakly reachable objects.
 *
 * [ObjectWatcher] is thread safe.
 */
// Thread safe by locking on all methods, which is reasonably efficient given how often
// these methods are accessed.
class ObjectWatcher constructor(
  private val clock: Clock,
  private val checkRetainedExecutor: Executor,
  /**
   * Calls to [watch] will be ignored when [isEnabled] returns false
   */
  private val isEnabled: () -> Boolean = { true }
) : ReachabilityWatcher {

  private val onObjectRetainedListeners = mutableSetOf<OnObjectRetainedListener>()

  /**
   * References passed to [watch].
   */
  private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()

  private val queue = ReferenceQueue<Any>()

  /**
   * Returns true if there are watched objects that aren't weakly reachable, and
   * have been watched for long enough to be considered retained.
   */
  val hasRetainedObjects: Boolean
    @Synchronized get() {
      removeWeaklyReachableObjects()
      return watchedObjects.any { it.value.retainedUptimeMillis != -1L }
    }

  /**
   * Returns the number of retained objects, ie the number of watched objects that aren't weakly
   * reachable, and have been watched for long enough to be considered retained.
   */
  val retainedObjectCount: Int
    @Synchronized get() {
      removeWeaklyReachableObjects()
      return watchedObjects.count { it.value.retainedUptimeMillis != -1L }
    }

  /**
   * Returns true if there are watched objects that aren't weakly reachable, even
   * if they haven't been watched for long enough to be considered retained.
   */
  val hasWatchedObjects: Boolean
    @Synchronized get() {
      removeWeaklyReachableObjects()
      return watchedObjects.isNotEmpty()
    }

  /**
   * Returns the objects that are currently considered retained. Useful for logging purposes.
   * Be careful with those objects and release them ASAP as you may creating longer lived leaks
   * then the one that are already there.
   */
  val retainedObjects: List<Any>
    @Synchronized get() {
      removeWeaklyReachableObjects()
      val instances = mutableListOf<Any>()
      for (weakReference in watchedObjects.values) {
        if (weakReference.retainedUptimeMillis != -1L) {
          val instance = weakReference.get()
          if (instance != null) {
            instances.add(instance)
          }
        }
      }
      return instances
    }

  @Synchronized fun addOnObjectRetainedListener(listener: OnObjectRetainedListener) {
    onObjectRetainedListeners.add(listener)
  }

  @Synchronized fun removeOnObjectRetainedListener(listener: OnObjectRetainedListener) {
    onObjectRetainedListeners.remove(listener)
  }

  /**
   * Identical to [watch] with an empty string reference name.
   */
  @Deprecated(
    "Add description parameter explaining why an object is watched to help understand leak traces.",
    replaceWith = ReplaceWith(
      "expectWeaklyReachable(watchedObject, \"Explain why this object should be garbage collected soon\")"
    )
  )
  fun watch(watchedObject: Any) {
    expectWeaklyReachable(watchedObject, "")
  }

  @Deprecated(
    "Method renamed expectWeaklyReachable() to clarify usage.",
    replaceWith = ReplaceWith(
      "expectWeaklyReachable(watchedObject, description)"
    )
  )
  fun watch(
    watchedObject: Any,
    description: String
  ) {
    expectWeaklyReachable(watchedObject, description)
  }


  @Synchronized override fun expectWeaklyReachable(
    watchedObject: Any,
    description: String
  ) {
    if (!isEnabled()) {
      return
    }
    // 移除弱引用对象
    removeWeaklyReachableObjects()
    val key = UUID.randomUUID()
      .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
        (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
        (if (description.isNotEmpty()) " ($description)" else "") +
        " with key $key"
    }

    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }

  /**
   * Clears all [KeyedWeakReference] that were created before [heapDumpUptimeMillis] (based on
   * [clock] [Clock.uptimeMillis])
   */
  @Synchronized fun clearObjectsWatchedBefore(heapDumpUptimeMillis: Long) {
    val weakRefsToRemove =
      watchedObjects.filter { it.value.watchUptimeMillis <= heapDumpUptimeMillis }
    weakRefsToRemove.values.forEach { it.clear() }
    watchedObjects.keys.removeAll(weakRefsToRemove.keys)
  }

  /**
   * Clears all [KeyedWeakReference]
   */
  @Synchronized fun clearWatchedObjects() {
    watchedObjects.values.forEach { it.clear() }
    watchedObjects.clear()
  }

  @Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
  }

  private fun removeWeaklyReachableObjects() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    do {
      ref = queue.poll() as KeyedWeakReference?
      if (ref != null) {
        watchedObjects.remove(ref.key)
      }
    } while (ref != null)
  }
}

```

这个类是最关键的类之一，以下是判断内存泄漏的步骤：

- 当有对象通过 `watch()` 方法传递后，会调用 `expectWeaklyReachable()`

```kotlin
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  	// isEnabled 为 false 就不走下面的逻辑了
    if (!isEnabled()) {
      return
    }
    // 移除队列中的弱引用对象
    removeWeaklyReachableObjects()
    // 生成一个值为随机的 UUID 的 key，作为 watchedObjects 这个映射表存放 KeyedWeakReference 的 key
    val key = UUID.randomUUID()
    .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    // 将 KeyedWeakReference 存放在 watchedObjects，键为上面的 key（随机的UUID）
    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
}
```

- `expectWeaklyReachable()` 通过 checkRetainedExecutor 调用 moveToRetained()

- moveToRetained() 实现：

```kotlin
@Synchronized private fun moveToRetained(key: String) {
    // 1. 移除弱引用对象
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      // 2. 如果上面的步骤执行完后还有 残留的引用，则调用监听器的 onObjectRetained() 方法，通知  
      // 监听对象有引用残留
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
}
```

最终可以看到 onObjectRetained() 的调用是在 InternalLeakCanary 类中实现，当调用该方法时会触发 HeapDumpTrigger 进行对象引用的检测。

```kotlin
override fun onObjectRetained() = scheduleRetainedObjectCheck()
fun scheduleRetainedObjectCheck() {
    // 当被回调有引用残留时，通过 HeadDumpTrigger 检查是否有内存泄漏
    if (this::heapDumpTrigger.isInitialized) {
        heapDumpTrigger.scheduleRetainedObjectCheck()
    }
}
```

HeapDumpTrigger.scheduleRetainedObjectCheck() 内部实现：

```kotlin
fun scheduleRetainedObjectCheck(
        delayMillis: Long = 0L,
) {
    val checkCurrentlyScheduledAt = checkScheduledAt
    if (checkCurrentlyScheduledAt > 0) {
        return
    }
    // 系统运行时间（从开机开始计算，不包括深度睡眠时间）
    checkScheduledAt = SystemClock.uptimeMillis() + delayMillis
  	// backgroundHandler 与 HandlerThread(LEAK_CANARY_THREAD_NAME) 共用一个 Looper
    backgroundHandler.postDelayed({
      checkScheduledAt = 0
      checkRetainedObjects()
    }, delayMillis)
}
```

checkRetainedObjects() 内部实现：

```kotlin
private fun checkRetainedObjects() {
        val iCanHasHeap = HeapDumpControl.iCanHasHeap()

        val config = configProvider()
        if (iCanHasHeap is Nope) {
            if (iCanHasHeap is NotifyingNope) {
                // Before notifying that we can't dump heap, let's check if we still have retained object.
                var retainedReferenceCount = objectWatcher.retainedObjectCount
                // 残留对象数量大于 0 时手动触发 GC，这里的 gc 有个要注意的点
                // 调用的是 Runtime.getRuntime().gc()，而不是 System.gc()，因为前者更可能立即执行 gc，后者
                // 要等待系统调用，可能不立即执行 ，之后再调用 System.runFinalization()
                if (retainedReferenceCount > 0) {
                    gcTrigger.runGc()
                    retainedReferenceCount = objectWatcher.retainedObjectCount
                }
                // 当 gc 和 finalize() 都调用了，就再次检查 残留的对象引用数是否 > 0
                val wouldDump = !checkRetainedCount(
                        retainedReferenceCount, config.retainedVisibleThreshold, nopeReason
                )
            }
            return
        }

        var retainedReferenceCount = objectWatcher.retainedObjectCount
  			// 残留对象数量大于 0 时手动触发 GC，这里的 gc 有个要注意的点
  			// 调用的是 Runtime.getRuntime().gc()，而不是 System.gc()，因为前者更可能立即执行 gc，后者
		    // 要等待系统调用，可能不立即执行 ，之后再调用 System.runFinalization()
        if (retainedReferenceCount > 0) {
            gcTrigger.runGc()
            retainedReferenceCount = objectWatcher.retainedObjectCount
        }
        dumpHeap(
                retainedReferenceCount = retainedReferenceCount,
                retry = true,
                reason = "$retainedReferenceCount retained objects, app is $visibility"
        )
    }
```

紧接着调用 `dumpHeap()` 

```kotlin
private fun dumpHeap(
        retainedReferenceCount: Int,
        retry: Boolean,
        reason: String,
) {
    saveResourceIdNamesToMemory()
    KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
    when (val heapDumpResult = heapDumper.dumpHeap()) {
      is NoHeapDump -> {
        //…… 省略
      }
      is HeapDump -> {
        lastDisplayedRetainedObjectCount = 0
        lastHeapDumpUptimeMillis = SystemClock.uptimeMillis()
        objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
        // 最终开启了 HeapAnalyzerService 这个 Service 来进行内存泄漏文件的分析
        HeapAnalyzerService.runAnalysis(
                context = application,
                heapDumpFile = heapDumpResult.file,
                heapDumpDurationMillis = heapDumpResult.durationMillis,
                heapDumpReason = reason
        )
      }
    }
}
```

上面的长篇大论总结一下就是：

Activity / Fragment / Service / RootView 在要被销毁的时候，将它们的弱引用伴随随机生成值为 UUID 的 key 存放在一个 LinkedHashMap 中。再对弱引用进行移除等操作，把 map 中还剩余的没有被移除的 引用对象 生成一个 DumpHeap 文件，最终开启了 HeapAnalyzerService 这个 Service 来进行内存泄漏文件的分析。这个 HeapAnalyzerService 是一个 IntentService。

**注**：关于 `KeyedWeakReference` 有个特性，之前看到[这篇博文](https://www.jianshu.com/p/eb17a6bfdd9f)中有提到

> 弱引用和引用队列搭配使用，如果弱引用持有的对象被回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。也就是说如果 `KeyedWeakReference` 持有的 `Activity` 对象被回收，该`KeyedWeakReference` 就会加入到引用队列 queue 中。