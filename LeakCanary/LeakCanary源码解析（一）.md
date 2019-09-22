[TOC]

***基于LeakCanary1.6.3版本***

# 一、从LeakCanary使用说起

```java
public class DemoApplication extends Application {
    @Override public void onCreate() {
    super.onCreate();
    if (LeakCanary.isInAnalyzerProcess(this)) {//1
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
    }
    LeakCanary.install(this);
  }
}
```

从注释中可以看出：如果当前进程就是LeakCanary的分析进程中，则不进行应用的初始化。可以理解为LeakCanary单独使用一个进程分析堆栈信息，不运行在App的进程中

# 二、如何判断是否运行在当前进程中的

```java
public static boolean isInAnalyzerProcess(@NonNull Context context) {
  Boolean isInAnalyzerProcess = LeakCanaryInternals.isInAnalyzerProcess;
  // This only needs to be computed once per process.
  if (isInAnalyzerProcess == null) {
    isInAnalyzerProcess = isInServiceProcess(context, HeapAnalyzerService.class);
    LeakCanaryInternals.isInAnalyzerProcess = isInAnalyzerProcess;
  }
  return isInAnalyzerProcess;
}
```

主要方法为 ```isInServiceProcess``` ，可以看到利用拆箱/装箱的小技巧：选择使用Boolean来让每个进程只运行一次 ```isInServiceProcess``` 方法。

接下来看```isInServiceProcess``` 方法

```java
public static boolean isInServiceProcess(Context context, Class<? extends Service> serviceClass) {
  PackageManager packageManager = context.getPackageManager();
  PackageInfo packageInfo;
  try {
    packageInfo = packageManager.getPackageInfo(context.getPackageName(), GET_SERVICES);
  } catch (Exception e) {
    CanaryLog.d(e, "Could not get package info for %s", context.getPackageName());
    return false;
  }
  String mainProcess = packageInfo.applicationInfo.processName;

  ComponentName component = new ComponentName(context, serviceClass);
  ServiceInfo serviceInfo;
  try {
    //获取HeapAnalyzerService服务的信息
    serviceInfo = packageManager.getServiceInfo(component, PackageManager.GET_DISABLED_COMPONENTS);
  } catch (PackageManager.NameNotFoundException ignored) {
    // Service is disabled.
    return false;
  }

  if (serviceInfo.processName.equals(mainProcess)) {
    CanaryLog.d("Did not expect service %s to run in main process %s", serviceClass, mainProcess);
    // Technically we are in the service process, but we're not in the service dedicated process.
    return false;
  }

  int myPid = android.os.Process.myPid();
  ActivityManager activityManager =
      (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
  ActivityManager.RunningAppProcessInfo myProcess = null;
  List<ActivityManager.RunningAppProcessInfo> runningProcesses;
  try {
    runningProcesses = activityManager.getRunningAppProcesses();
  } catch (SecurityException exception) {
    // https://github.com/square/leakcanary/issues/948
    CanaryLog.d("Could not get running app processes %d", exception);
    return false;
  }
  if (runningProcesses != null) {
    for (ActivityManager.RunningAppProcessInfo process : runningProcesses) {
      if (process.pid == myPid) {
        //在App运行进程中找当前pid的进程信息
        myProcess = process;
        break;
      }
    }
  }
  if (myProcess == null) {
    CanaryLog.d("Could not find running process for %d", myPid);
    return false;
  }

  //只有当前进程名和HeapAnalyzerService运行的进程名相同，才认为LeakCanary运行在了Application进程中
  return myProcess.processName.equals(serviceInfo.processName);
}
```

通过以上可以看出：只有一种情况返回 ```true``` ，在当前App运行的进程中，根据 ```Application``` 进程的pid获取的进程名和 ```HeapAnalyzerService``` 服务所在进程名称相同。有一点需要注意下：

```java
ServiceInfo serviceInfo;
try {
  serviceInfo = packageManager.getServiceInfo(component, PackageManager.GET_DISABLED_COMPONENTS);
} catch (PackageManager.NameNotFoundException ignored) {
  // Service is disabled.
  return false;
}
```

这里是使用 ```PackageManager.GET_DISABLED_COMPONENTS``` ，在LeanCanary中的服务默认状态是不可用的

```xml
<service
    android:name=".internal.HeapAnalyzerService"
    android:process=":leakcanary"
    android:enabled="false"
    />
```

所以在LeakCanary中使用了```PackageManager.GET_DISABLED_COMPONENTS``` 标签。

# 三、监控内存泄漏是怎么开启的

```java
public static @NonNull RefWatcher install(@NonNull Application application) {
  return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
      .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
      .buildAndInstall();
}
```

* ```DisplayLeakService.class``` 看名字也知道是分析内存泄露用的，先不管里面逻辑

* ```buildAndInstall``` 这里面会有```Activity/Fragment``` 内存泄漏的监控代码

  ```java
  public @NonNull RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
      if (watchActivities) {
        ActivityRefWatcher.install(context, refWatcher);
      }
      if (watchFragments) {
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
  }
  ```

```Activity``` 监控，```ActivityRefWatcher.install(context, refWatcher);```  

```java
public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
  Application application = (Application) context.getApplicationContext();
  ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);

  application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
}

private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
    new ActivityLifecycleCallbacksAdapter() {
      @Override public void onActivityDestroyed(Activity activity) {
        refWatcher.watch(activity);
      }
    };
```

其实原理很简单，就是注册一个 ```ActivityLifecycleCallbacks``` 监听，在 ```onActivityDestroyed``` 回调中查看下 ```Activity``` 是否被回收，具体逻辑在 ```refWatcher.watch(activity);``` 中，我们之后详细介绍过程

下面再看下 ```Fragment``` 监控， ```FragmentRefWatcher.Helper.install(context, refWatcher);``` 

```java
final class Helper {

  private static final String SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME =
      "com.squareup.leakcanary.internal.SupportFragmentRefWatcher";

  public static void install(Context context, RefWatcher refWatcher) {
    List<FragmentRefWatcher> fragmentRefWatchers = new ArrayList<>();

    if (SDK_INT >= O) {
      //Android O 适配，逻辑和之前版本相同
      fragmentRefWatchers.add(new AndroidOFragmentRefWatcher(refWatcher));
    }

    try {
      Class<?> fragmentRefWatcherClass = Class.forName(SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME);
      Constructor<?> constructor =
          fragmentRefWatcherClass.getDeclaredConstructor(RefWatcher.class);
      FragmentRefWatcher supportFragmentRefWatcher =
          (FragmentRefWatcher) constructor.newInstance(refWatcher);
      fragmentRefWatchers.add(supportFragmentRefWatcher);
    } catch (Exception ignored) {
    }

    if (fragmentRefWatchers.size() == 0) {
      return;
    }

    Helper helper = new Helper(fragmentRefWatchers);

    Application application = (Application) context.getApplicationContext();
    application.registerActivityLifecycleCallbacks(helper.activityLifecycleCallbacks);
  }

  private final Application.ActivityLifecycleCallbacks activityLifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
          for (FragmentRefWatcher watcher : fragmentRefWatchers) {
            //这里是重点，注册helper.activityLifecycleCallbacks，在onActivityCreated中检测
            //fragment的生命周期，具体逻辑在watchFragments中
            watcher.watchFragments(activity);
          }
        }
      };

  private final List<FragmentRefWatcher> fragmentRefWatchers;

  private Helper(List<FragmentRefWatcher> fragmentRefWatchers) {
    this.fragmentRefWatchers = fragmentRefWatchers;
  }
}
```

再看下 ```watchFragments``` 方法，我们以 ```AndroidOFragmentRefWatcher``` 为例：

```java
class AndroidOFragmentRefWatcher implements FragmentRefWatcher {

  private final RefWatcher refWatcher;

  AndroidOFragmentRefWatcher(RefWatcher refWatcher) {
    this.refWatcher = refWatcher;
  }

  private final FragmentManager.FragmentLifecycleCallbacks fragmentLifecycleCallbacks =
      new FragmentManager.FragmentLifecycleCallbacks() {

        @Override public void onFragmentViewDestroyed(FragmentManager fm, Fragment fragment) {
          View view = fragment.getView();
          if (view != null) {
            refWatcher.watch(view);
          }
        }

        @Override
        public void onFragmentDestroyed(FragmentManager fm, Fragment fragment) {
          refWatcher.watch(fragment);
        }
      };

  @Override public void watchFragments(Activity activity) {
    FragmentManager fragmentManager = activity.getFragmentManager();
    fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);
  }
}
```

会通过当前Activity对象获取FragmentManager对象，注册回调监听 ```fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);```

总结下：

* LeakCanary会单独开启一个进程运行，防止影响我们应用吧
* LeakCanary通过监听 ```onActivityDestroyed（Activity）```，```onFragmentViewDestroyed/onFragmentDestroyed（Fragment）``` 来实现内存泄漏检测 
* LeakCanary其实就是通过Activity/Fragment在销毁时，利用一些机制来检测Activity/Fragment内存是否被回收，如果没有被回收（当前中间还会再手动触发一次GC），则认为内存泄漏