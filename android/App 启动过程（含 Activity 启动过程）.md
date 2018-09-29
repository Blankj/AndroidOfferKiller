# App 启动过程（含 Activity 启动过程）

这道题在曾经面试「菜鸟网络」中遇到过，不过当时只问了「Activity 启动过程」，这里对整个「App 启动过程」进行完整的源码分析，希望可以帮助到大家。

1. Launcher 捕获点击事件，其过程为 `Launcher#onClick` -> `Launcher#onClickAppShortcut` -> `Launcher#startAppShortcutOrInfoActivity` -> `Launcher#startActivitySafely` -> `Activity#startActivity`，其 Launcher3 相关源码如下所示：

```java
// https://github.com/amirzaidi/Launcher3/blob/f7951c32984036eef2f2130f21abded3ddf6160a/src/com/android/launcher3/Launcher.java#L2249
public void onClick(View v) {
    ...
    Object tag = v.getTag();
    if (tag instanceof ShortcutInfo) {
        onClickAppShortcut(v);
    }
    ...
}

// https://github.com/amirzaidi/Launcher3/blob/f7951c32984036eef2f2130f21abded3ddf6160a/src/com/android/launcher3/Launcher.java#L2412
protected void onClickAppShortcut(final View v) {
    ...
    // Start activities
    startAppShortcutOrInfoActivity(v);
}

// https://github.com/amirzaidi/Launcher3/blob/f7951c32984036eef2f2130f21abded3ddf6160a/src/com/android/launcher3/Launcher.java#L2462
private void startAppShortcutOrInfoActivity(View v) {
    ItemInfo item = (ItemInfo) v.getTag();
    Intent intent;// 应用程序安装的时候根据 AndroidManifest.xml 由 PackageManagerService 解析并保存的
    if (item instanceof PromiseAppInfo) {
        PromiseAppInfo promiseAppInfo = (PromiseAppInfo) item;
        intent = promiseAppInfo.getMarketIntent();
    } else {
        intent = item.getIntent();
    }
    ...
    boolean success = startActivitySafely(v, intent, item);
    ...
}

// https://github.com/amirzaidi/Launcher3/blob/f7951c32984036eef2f2130f21abded3ddf6160a/src/com/android/launcher3/Launcher.java#L2689
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
    ...
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    ...
    startActivity(intent, optsBundle);
    ...
}
```

2. 以 API 27 源码为例，说到了 `Acitvity#startActivity`，我们点击源码可以发现调用的是 `Activity#startActivityForResult`，其中调用到了 `Instrumentation#execStartActivity` 这个方法，源码如下所示：

``` java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/Activity.java#4800
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/Activity.java#4482
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    ...
    Instrumentation.ActivityResult ar =
        mInstrumentation.execStartActivity(
            this, mMainThread.getApplicationThread(), mToken, this,
            intent, requestCode, options);
    ...
}
```

3. 在 `Instrumentation#execStartActivity` 中我们可以发现它调用了 `ActivityManager#getService()#startActivity`，其 `ActivityManager#getService()` 是采用单例，返回的是实现 `IActivityManager` 类型的 `Binder` 对象，它的具体实现是在 `ActivityManagerService` 中。

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/Instrumentation.java#1578
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    try {
        ...
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        ...
    } catch (RemoteException e) {
        ...
    }
    return null;
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityManager.java#4216
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

4. 我们再到 `ActivityManagerService#startActivity` 查看其源码，发现其调用了 `ActivityManagerService#startActivityAsUser`，该方法又调用了 `ActivityStarter#startActivityMayWait`，源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java#4516
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
            userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    // TODO: Switch to user app stacks here.
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, "startActivityAsUser");
}
```

5. 我们查找到 `ActivityStarter#startActivityMayWait`，其间调用了 `ActivityStarter#startActivityLocked`，接着是 `ActivityStarter#startActivity`，然后是 `ActivityStarter#startActivityUnchecked`，其调用了 `ActivityStackSupervisor#resumeFocusedStackTopActivityLocked`，源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java#673
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,
        TaskRecord inTask, String reason) {
    ...
    int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor,
            resultTo, resultWho, requestCode, callingPid,
            callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, outRecord, inTask,
            reason);
    ...
}
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java#263
int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, TaskRecord inTask, String reason) {
    ...
    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
            inTask);
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java#294
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, TaskRecord inTask) {
    ...
    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
            options, inTask, outActivity);
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java#988
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    ...
    result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
            startFlags, doResume, options, inTask, outActivity);
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java#1015
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    ...
    mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
            mOptions);
    ...
}
```

6. 到 `ActivityStackSupervisor#resumeFocusedStackTopActivityLocked` 中查看发现其调用了 `ActivityStack#resumeTopActivityUncheckedLocked`，然后是 `ActivityStack#resumeTopActivityInnerLocked`，接着变又回到 `ActivityStackSupervisor.java`，调用了 `ActivityStackSupervisor#startSpecificActivityLocked`，这个方法中会判断要启动 App 的进程是否存在，存在则通知进程启动 Activity，否则就先将进程创建出来，其源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java#2085
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    ...
    return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java#2245
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    result = resumeTopActivityInnerLocked(prev, options);
    ...
}

http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStack.java#2286
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    mStackSupervisor.startSpecificActivityLocked(next, true, true);
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java#1560
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    ...
    if (app != null && app.thread != null) {
        ...
        // 如果进程已存在，则通知进程启动组件
        realStartActivityLocked(r, app, andResume, checkConfig);
        return;
        ...
    }
    // 否则先将进程创建出来
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
    ...
}
```

7. 我们分析进程尚未存在的情况，因为我们后续还会再次遇到 `ActivityStackSupervisor#realStartActivityLocked`，`ActivityStackSupervisor#startSpecificActivityLocked` 中创建进程使用到的 `mService` 为 `ActivityManagerService`，我们查看 `ActivityManagerService#startProcessLocked` 的源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java#3777
private final void startProcessLocked(ProcessRecord app, String hostingType,
        String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    ...
    if (entryPoint == null) entryPoint = "android.app.ActivityThread";
    startResult = Process.start(entryPoint,
            app.processName, uid, uid, gids, debugFlags, mountExternal,
            app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
            app.info.dataDir, invokeWith, entryPointArgs);
    ...
}
```

8. 发现最终调用的事 `Process#start` 来启动进程，进程的入口就是在 `android.app.ActivityThread.java` 类中的 `main()` 函数，因此接下来我们从 `ActivityThread#main` 来分析，其调用了 `ActivityThread#attach`，其中 `ActivityManager.getService()` 之前提到过，返回的是一个是实现 `IActivityManager` 类型的 `Binder` 对象，它的具体实现是在 `ActivityManagerService` 中，相关源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#6459
public static void main(String[] args) {
    ...
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    ...
    Looper.loop();
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#6315
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ...
        final IActivityManager mgr = ActivityManager.getService();
        try {
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            ...
        }
        ...
    }
    ...
}
```

9. 我们又回到了 `ActivityManagerService` 中，查看其 `attachApplication` 函数，发现调用了 `thread#bindApplication` 和 `mStackSupervisor#attachApplicationLocked` 我们依次讲解这两个方法要做的事情，源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java#7215
public final void attachApplication(IApplicationThread thread) {
    ...
    attachApplicationLocked(thread, callingPid);
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java#6911
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid) {
    ....
    thread.bindApplication(processName, appInfo, providers,
            app.instr.mClass,
            profilerInfo, app.instr.mArguments,
            app.instr.mWatcher,
            app.instr.mUiAutomationConnection, testMode,
            mBinderTransactionTrackingEnabled, enableTrackAllocation,
            isRestrictedBackupMode || !normalMode, app.persistent,
            new Configuration(getGlobalConfiguration()), app.compat,
            getCommonServicesLocked(app.isolated),
            mCoreSettingsObserver.getCoreSettingsLocked(),
            buildSerial);
    ...
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            ...
        }
    }
    ...
}
```

10. 上面说到的`thread#bindApplication` 中的这个 `thread` 是来自于 `ActivityThread#mAppThread`，其类型是 `ApplicationThread`，是 `ActivityThread` 的一个内部类，继承自 `IApplicationThread.Stub`，我们来查看 `ApplicationThread#bindApplication`，发现最后调用了 `ActivityThread#sendMessage` 方法，它内部调用了 `mH.sendMessage` 来发送消息，`mH` 是 `ActivityThread` 的内部类 `H` 的一个实例，查看 `H#handleMessage` 来查看它是怎么处理发送过来的消息，其最终走到了 `ActivityThread#handleBindApplication`。

在源码中我们可以发现它先创建 `mInstrumentation` 对象，调用 `data#info#makeApplication` 来创建 `Application` 对象，其对象 `data#info` 为 `LoadedApk` 的一个实例，查看 `LoadedApk#makeApplication` 中代码可以发现，其调用了 `Instrumentation#newApplication` 方法，内部靠 `Class#newInstance()` 完成对 `Application` 实例化，然后调用 `Application#attach(context)` 来绑定 `Context`。

以上创建完 `Application` 对象后便是调用 `Instrumentation#callApplicationOnCreate` 走 `Application` 的 `onCreate` 生命周期，以上涉及到的全部源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#899
public final void bindApplication(String processName, ApplicationInfo appInfo,
        List<ProviderInfo> providers, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableBinderTracking, boolean trackAllocation,
        boolean isRestrictedBackupMode, boolean persistent, Configuration config,
        CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
        String buildSerial) {
    ...
    sendMessage(H.BIND_APPLICATION, data);
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#2593
private void sendMessage(int what, Object obj) {
    sendMessage(what, obj, 0, 0, false);
}

private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
        TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
        + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#1580
public void handleMessage(Message msg) {
    ...
    switch (msg.what) {
        ...
        case BIND_APPLICATION:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
            AppBindData data = (AppBindData)msg.obj;
            handleBindApplication(data);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
        ...
    }
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#5429
private void handleBindApplication(AppBindData data) {
    ...
    final InstrumentationInfo ii;
    ...
    // 创建 mInstrumentation 实例
    if (ii != null) {
        final ApplicationInfo instrApp = new ApplicationInfo();
        ii.copyTo(instrApp);
        instrApp.initForUser(UserHandle.myUserId());
        final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                appContext.getClassLoader(), false, true, false);
        final ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

        try {
            final ClassLoader cl = instrContext.getClassLoader();
            mInstrumentation = (Instrumentation)
                cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            ...
        }
        ...
    } else {
        mInstrumentation = new Instrumentation();
    }
    ...
    Application app;
    ...
    // 创建 Application 实例
    try {
        ...
        app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;
        ...
        try {
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            ...
        }
    } finally {
        ...
    }
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/LoadedApk.java#959
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        java.lang.ClassLoader cl = getClassLoader();
        ...
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        ...
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    if (instrumentation != null) {// 传入为 null 所以不走
        try {
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            ...
        }
    }
    ...
    return app;
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/Instrumentation.java#1084
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    return newApplication(cl.loadClass(className), context);
}

static public Application newApplication(Class<?> clazz, Context context)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    Application app = (Application)clazz.newInstance();
    app.attach(context);
    return app;
}
```

11. 说完了 9 中的 `thread#bindApplication`，下面我们继续说 `mStackSupervisor#attachApplicationLocked`，其 `mStackSupervisor` 是 `ActivityStackSupervisor` 的一个实例，我们查看 `ActivityStackSupervisor#attachApplicationLocked` 方法中发现会调用 `ActivityStackSupervisor#realStartActivityLocked`，其方法会调用 `app#thread#scheduleLaunchActivity`，源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java#956
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    final String processName = app.processName;
    ...
    if (realStartActivityLocked(activity, app,
            top == activity /* andResume */, true /* checkConfig */)) {
        ...
    }
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java#1313
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
            System.identityHashCode(r), r.info,
            // TODO: Have this take the merged configuration instead of separate global
            // and override configs.
            mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
            r.persistentState, results, newIntents, !andResume,
            mService.isNextTransitionForward(), profilerInfo);
    ...
}
```

12. 上面说到的 `app#thread#scheduleLaunchActivity` 中的 `thread` 是前面提到过的 `IApplicationThread`，它的实现类是 `ActivityThread#ApplicationThread` ，我们查看 `ActivityThread#ApplicationThread#scheduleLaunchActivity` 中的代码发现最终是发送 `LAUNCH_ACTIVITY` 消息，这步我们在第 10 步中有过分析，我们直接查看其处理消息相关代码即可，在 `H#handleMessage` 中，我们可以看到其会接收并处理很多和四大组件相关的操作，我们查看对 `LAUNCH_ACTIVITY` 的处理，发现对其处理的方法是 `ActivityThread#handleLaunchActivity`，它调用到了 `ActivityThread#performLaunchActivity` 方法，其中的实现再次涉及到了 `Instrumentation` 类，之前是在创建 `Application` 对象用到了它，如今是创建 `Activity` 对象又用到了它，其 `Instrumentation#newActivity` 也是通过 `Class.newInstance()` 来实例化 `Activity`，实例化结束后回到 `ActivityThread#performLaunchActivity` 中来让 `activity` 依附到 window 中，然后`callActivityOnCreate` 走 `Activity` 的 `onCreate` 生命周期，涉及到的源码如下所示：

```java
// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#756
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    ...
    sendMessage(H.LAUNCH_ACTIVITY, r);
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#1580
public void handleMessage(Message msg) {
    ...
    switch (msg.what) {
        ...
        case LAUNCH_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

            r.packageInfo = getPackageInfoNoCheck(
                    r.activityInfo.applicationInfo, r.compatInfo);
            handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        } break;
        ...
    }
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#2833
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
    Activity a = performLaunchActivity(r, customIntent);
    ...
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/ActivityThread.java#2644
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        ...
    } catch (Exception e) {
        ...
    }

    try {
        // 返回之前创建过的 application 对象
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        if (activity != null) {
            ...
            // attach 到 window 上
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);
            ...
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
        }
    } catch (Exception e) {
        ...
    }
    return activity;
}

// http://androidxref.com/8.1.0_r33/xref/frameworks/base/core/java/android/app/Instrumentation.java#1143
public Activity newActivity(ClassLoader cl, String className,
        Intent intent)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    return (Activity)cl.loadClass(className).newInstance();
}

public Activity newActivity(Class<?> clazz, Context context,
        IBinder token, Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        Object lastNonConfigurationInstance) throws InstantiationException,
        IllegalAccessException {
    Activity activity = (Activity)clazz.newInstance();
    ActivityThread aThread = null;
    activity.attach(context, aThread, this, token, 0 /* ident */, application, intent,
            info, title, parent, id,
            (Activity.NonConfigurationInstances)lastNonConfigurationInstance,
            new Configuration(), null /* referrer */, null /* voiceInteractor */,
            null /* window */, null /* activityConfigCallback */);
    return activity;
}
```

到此为止，一个 App 的启动过程已分析结束，最后献上启动涉及到的类的流程图：

![App 的启动流程图](http://ww1.sinaimg.cn/large/b75b8776gy1fulx2ikj15j20ia0o6mxs.jpg)


## 结语

我正在打造一个帮助 Android 开发者们拿到更好 offer 的面试库————**[安卓 offer 收割基](https://github.com/Blankj/AndroidOfferKiller)**，欢迎 star，觉得不错的可以持续关注，有兴趣的可以一起加入进来和我一同打造。