---
layout: post
title: Android SystemService 的注册与获取
categories: android
tags: android service
date: 2023-07-01
---
Android 有许多 Service，如 ActivityManagerService、ActivityTaskManagerService、PackageManagerService、WindowsManagerService 等。

本次目标：
1. 服务在是何时注册

2. 服务在哪个进程注册

3. 服务是如何获取

4. 服务存在的形式是什么，单例还是多例？单例会产生线程安全问题吗？

<!--more-->
## 1.服务在何时注册
Android 系统启动时，会 forkSystemServer ，创建 system_server 进程，用来创建及维护一些系统服务，所以这里可以回答第二个问题了，是在 `system_server` 进程中注册。

代码追踪从 `SystemServer.java` 开始。
```java
public final class SystemServer implements Dumpable {
    public static void main(String[] args) {
        new SystemServer().run();
    }

    private void run() {
        ......
        // Start services.
        //这里开始注册系统服务
        try {
            t.traceBegin("StartServices");
            startBootstrapServices(t);
            startCoreServices(t);
            startOtherServices(t);
            startApexServices(t);
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            t.traceEnd(); // StartServices
        }
        ......
    }
}
```

所有服务的注册都在这里了，首先进入 `startBootstrapServices(t)` ， `ActivityTaskManagerService` 、`ActivityManagerService` 及 `PackageManagerService` 在这里注册

```java
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    //ActivityTaskManagerService
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    
    //ActivityManagerService        
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);

}
```

再来看看具体是怎么注册的，`mSystemServiceManager` 是 `SystemServiceManager` 的一个实例对象，所以代码来到 `SystemServiceManager`

```java
public final class SystemServiceManager implements Dumpable {

    private List<SystemService> mServices;
    private Set<String> mServiceClassnames;

    public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {
            final String name = serviceClass.getName();

            // Create the service.
            //所有注册的服务要是 SystemService 的子类
            if (!SystemService.class.isAssignableFrom(serviceClass)) {
                throw new RuntimeException("Failed to create " + name
                        + ": service must extend " + SystemService.class.getName());
            }
            final T service;
            try {
                //反射，创建实例对象
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext);
            }
            .......
            startService(service);
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }

    public void startService(@NonNull final SystemService service) {
        // Check if already started
        String className = service.getClass().getName();
        if (mServiceClassnames.contains(className)) {
            Slog.i(TAG, "Not starting an already started service " + className);
            return;
        }
        mServiceClassnames.add(className);

        // Register it.保存注册信息
        mServices.add(service);

        // Start it.
        try {
            //这里执行具体的注册
            service.onStart();
        }
    }
}
```

从上面的代码可以得知，服务的注册先是存放到字段 `mServices` 中，之后再执行具体的注册 `SystemService->onStart()` ，来 `ActivityTaskManagerService.Lifecycle.java` 中看看具体是如何注册的

```java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {

    final ActivityTaskManagerInternal mInternal;

    public ActivityTaskManagerService(Context context) {
        mInternal = new LocalService();
    }

    private void start() {
        LocalServices.addService(ActivityTaskManagerInternal.class, mInternal);
    }

    public static final class Lifecycle extends SystemService {
        private final ActivityTaskManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityTaskManagerService(context);
        }

        @Override
        public void onStart() {
            //publishBinderService 是 SystemService 的方法
            publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
            mService.start();
        }

        public ActivityTaskManagerService getService() {
            return mService;
        }
    }

    final class LocalService extends ActivityTaskManagerInternal {
        ......
        @Override
        public int startActivityAsUser(IApplicationThread caller, String callerPackage,
                @Nullable String callerFeatureId, Intent intent, @Nullable IBinder resultTo,
                int startFlags, Bundle options, int userId) {
            return ActivityTaskManagerService.this.startActivityAsUser(
                    caller, callerPackage, callerFeatureId, intent,
                    intent.resolveTypeIfNeeded(mContext.getContentResolver()),
                    resultTo, null, 0, startFlags, null, options, userId,
                    false /*validateIncomingUser*/);
        }
        ......
    }
}
```

`publishBinderService` 是 `SystemService` 声明的方法
```java
public abstract class SystemService {
    protected final void publishBinderService(@NonNull String name, @NonNull IBinder service) {
        publishBinderService(name, service, false);
    }

    protected final void publishBinderService(@NonNull String name, @NonNull IBinder service,
            boolean allowIsolated) {
        publishBinderService(name, service, allowIsolated, DUMP_FLAG_PRIORITY_DEFAULT);
    }

    protected final void publishBinderService(String name, IBinder service,
            boolean allowIsolated, int dumpPriority) {
        ServiceManager.addService(name, service, allowIsolated, dumpPriority);
    }
}

//来到 ServiceManager
public final class ServiceManager {

    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }

    public static void addService(String name, IBinder service, boolean allowIsolated,
            int dumpPriority) {
        try {
            getIServiceManager().addService(name, service, allowIsolated, dumpPriority);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }
}

public final class ServiceManagerNative {
    public static IServiceManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }

        // ServiceManager is never local
        return new ServiceManagerProxy(obj);
    }
}


class ServiceManagerProxy implements IServiceManager {
    private IBinder mRemote;

    private IServiceManager mServiceManager;

    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
        mServiceManager = IServiceManager.Stub.asInterface(remote);
    }

    public void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority)
            throws RemoteException {
        mServiceManager.addService(name, service, allowIsolated, dumpPriority);
    }
}
```

这里就不再往下追踪了，下面是使用 Binder 来实现注册，这里的 binder 与 ActivityTaskManagerService 绑定。

```java
public interface IServiceManager extends android.os.IInterface{
    public static abstract class Stub extends android.os.Binder implements android.os.IServiceManager{
        private static class Proxy implements android.os.IServiceManager{
            public void addService(java.lang.String name, android.os.IBinder service, boolean allowIsolated, int dumpPriority) throws android.os.RemoteException{
                android.os.Parcel _data = android.os.Parcel.obtain(asBinder());
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                //DESCRIPTOR: android.os.IServiceManager
                _data.writeInterfaceToken(DESCRIPTOR);
                //name: Context.ACTIVITY_TASK_SERVICE
                _data.writeString(name);
                //service: ActivityTaskManagerService
                _data.writeStrongBinder(service);
                _data.writeBoolean(allowIsolated);
                _data.writeInt(dumpPriority);
                boolean _status = mRemote.transact(Stub.TRANSACTION_addService, _data, _reply, 0);
                _reply.readException();
                }
                finally {
                _reply.recycle();
                _data.recycle();
                }
            }
        }   
    }
}
```

所以 `publishBinderService()` 的作用是注册 binder 驱动，将 `ActivityTaskManagerService` 注册，名为 `Context.ACTIVITY_TASK_SERVICE`

再看 `mService.start()` 也即
```java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {

    final ActivityTaskManagerInternal mInternal;

    public ActivityTaskManagerService(Context context) {
        mInternal = new LocalService();
    }

    private void start() {
        LocalServices.addService(ActivityTaskManagerInternal.class, mInternal);
    }

    public static final class Lifecycle extends SystemService {
        private final ActivityTaskManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityTaskManagerService(context);
        }

        @Override
        public void onStart() {
            //publishBinderService 是 SystemService 的方法
            publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
            mService.start();
        }

        public ActivityTaskManagerService getService() {
            return mService;
        }
    }
}
```

最终会调用 `LocalServices.addService(ActivityTaskManagerInternal.class, mInternal);` ，所以代码来到 `LocalServices.java`

```java
/**
 * This class is used in a similar way as ServiceManager, except the services registered here
 * are not Binder objects and are only available in the same process.
 *
 * Once all services are converted to the SystemService interface, this class can be absorbed
 * into SystemServiceManager.
 *
 * {@hide}
 */
public final class LocalServices {
    private static final ArrayMap<Class<?>, Object> sLocalServiceObjects =
            new ArrayMap<Class<?>, Object>();

    public static <T> void addService(Class<T> type, T service) {
        synchronized (sLocalServiceObjects) {
            if (sLocalServiceObjects.containsKey(type)) {
                throw new IllegalStateException("Overriding service registration");
            }
            sLocalServiceObjects.put(type, service);
        }
    }
}
```

这个类的只供同进程获取服务，注释有说明。

到这里可以回答第4个问题了，服务是以单例的形式存在

## 2.如何获取服务
可以在 ActivityThread 中有这样一行代码
```java
IActivityTaskManager mgr = ActivityTaskManager.getService();
```

就以这为起点来看看服务是如何获取的
```java
public class ActivityTaskManager {
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
}
```

来到 ServiceManager
```java
public final class ServiceManager {

    private static Map<String, IBinder> sCache = new ArrayMap<String, IBinder>();

    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return Binder.allowBlocking(rawGetService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

    private static IBinder rawGetService(String name) throws RemoteException {

        final IBinder binder = getIServiceManager().getService(name);

        return binder;
    }

    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }
}
```

又回到 `ServiceManagerNative` 

```java
public final class ServiceManagerNative {
    public static IServiceManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        // ServiceManager is never local
        return new ServiceManagerProxy(obj);
    }
}

class ServiceManagerProxy implements IServiceManager {
    private IServiceManager mServiceManager;

    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
        mServiceManager = IServiceManager.Stub.asInterface(remote);
    }

    public IBinder getService(String name) throws RemoteException {
        return mServiceManager.checkService(name);
    }
}

public interface IServiceManager extends android.os.IInterface{
    public static abstract class Stub extends android.os.Binder implements android.os.IServiceManager{
        private static class Proxy implements android.os.IServiceManager{
            public android.os.IBinder checkService(java.lang.String name) throws android.os.RemoteException{
                android.os.Parcel _data = android.os.Parcel.obtain(asBinder());
                android.os.Parcel _reply = android.os.Parcel.obtain();
                android.os.IBinder _result;
                try {
                _data.writeInterfaceToken(DESCRIPTOR);
                //name: Context.ACTIVITY_TASK_SERVICE
                _data.writeString(name);
                boolean _status = mRemote.transact(Stub.TRANSACTION_checkService, _data, _reply, 0);
                _reply.readException();
                _result = _reply.readStrongBinder();
                }
                finally {
                _reply.recycle();
                _data.recycle();
                }
                return _result;
            }
        }   
    }
}
```

最终以 `name = Context.ACTIVITY_TASK_SERVICE` 获取对应的服务，结合上面注册服务的代码，可以得知获取的 service 为 `ActivityTaskManagerService`。

到这里上面的目标已经解答三个半了，还有一个问题是，单例的线程安全问题是怎么解决的？我们经常使用的就是 startActivity，所以追踪一下这个方法来看一看，

```java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {

    private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {

        final SafeActivityOptions opts = SafeActivityOptions.fromBundle(bOptions);

        if (opts != null && opts.getOriginalOptions().getTransientLaunch()
                && isCallerRecents(Binder.getCallingUid())) {
            final long origId = Binder.clearCallingIdentity();
            try {
                //共享资源加锁
                synchronized (mGlobalLock) {
                    Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "startExistingRecents");
                    if (mActivityStartController.startExistingRecentsIfPossible(
                            intent, opts.getOriginalOptions())) {
                        return ActivityManager.START_TASK_TO_FRONT;
                    }
                    // Else follow the standard launch procedure.
                }
            } finally {
                Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
                Binder.restoreCallingIdentity(origId);
            }
        }

        assertPackageMatchesCallingUid(callingPackage);
        enforceNotIsolatedCaller("startActivityAsUser");
        if (Process.isSdkSandboxUid(Binder.getCallingUid())) {
            SdkSandboxManagerLocal sdkSandboxManagerLocal = LocalManagerRegistry.getManager(
                    SdkSandboxManagerLocal.class);
            if (sdkSandboxManagerLocal == null) {
                throw new IllegalStateException("SdkSandboxManagerLocal not found when starting"
                        + " an activity from an SDK sandbox uid.");
            }
            sdkSandboxManagerLocal.enforceAllowedToStartActivity(intent);
        }

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(opts)
                .setUserId(userId)
                .execute();

    }
}
```

可以看到 `ActivityTaskManagerService-->startActivityAsUser` 中只有一小个代码片段需要加锁，其它资源都是局部变量，不会产生线程安全问题，所以不需要加锁。

到这里四个问题解答完毕。