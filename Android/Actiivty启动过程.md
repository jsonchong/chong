### 根Activity组件的启动过程

一般是在一个新的进程中启动的，当MainActivity组件被应用启动器Launcher组件启动之后，我们就可以通过adb shell dumpsys activity命令查看到它的信息

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201222202035440.png" alt="image-20201222202035440" style="zoom:50%;" />

MainActivity组件是由Launcher组件启动的，而Launcher组件又是通过Activity管理服务ActivityManagerService来启动MainActivity组件的

1. Launcher组件向AMS发送一个启动MainActivity组件的进程间通信请求
2. AMS首先将要启动的MainActivity组件的信息保存下来，然后再向Lanucher组件发送一个进入终止状态的进程间通信请求
3. Launcher组件进入到终止状态之后，就会向AMS发送一个已进入终止状态的进程间通信请求，以便AMS可以继续启动MainActivity组件的操作
4. AMS发现用来运行MainActivity组件的应用进程不存在，因此，它就会启动一个新的应用程序进程
5. 新的应用程序进程启动完成之后，就会向AMS发送一个启动完成的进程间通信请求，以便AMS可以继续执行启动MainActivity组件的操作
6. AMS将第二步保存下来的MainActivity组件的信息发送给第四步创建的应用程序进程，以便它可以将MainActivity组件启动起来

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201222203003672.png" alt="image-20201222203003672" style="zoom:50%;" />

​										

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201222203045297.png" alt="image-20201222203045297" style="zoom:50%;" />

当在应用程序Launcher的界面上点击一个应用程序的快捷图标时，Laucher的成员函数startActivitySafely就会被调用来启动这个应用程序的根Activity，其中要启动的根Activity的信息包含在参数intent中。

Laucher组件是如何获得这些信息呢？系统启动时，会启动一个PackageManagerService，并且通过它来安装应用程序，**PMS在安装一个应用程序的过程中，会对它的配置文件AndroidManifest.xml进行解析，从而得到它里面的组件信息。Laucher在启动过程中，会向PMS查询所有Action名称等于"Intent.ACTION_MAIN"，并且Category名称等于"Intent.CATEGORY_LAUCHER"的Activity组件**，最后为每一个Activity组件创建一个快捷图标。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20210301161845020.png" alt="image-20210301161845020" style="zoom:50%;" />

Activity类的成员变量mInstrumentation的类型为Instrumentation，它用来监控应用程序和系统之间的交互操作。因为启动Activity组件是应用程序与系统之间的一个交互操作。

Activity类的成员变量mMainThread的类型为ActivityThread,用来描述一个应用程序进程。系统每当启动一个应用程序进程时，都会在里面加载一个ActivityThread实例

```java
//ActivityThread类的main方法
public static void main(String[] args) {
   
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    Looper.loop()；
    
    // 这里抛了个异常，主线程loop异常退出。说明主线程loop不能退出，这里和前面建立Looper对象的调用方法有关
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

并且将这个ActivityThread类实例保存在每一个在该进程中启动的Activity组件的父类Activity的成员变量mMainThread中

### **ApplicationThread**

```java
/**
 * System private API for communicating with the application.  This is given to
 * the activity manager by an application  when it starts up, for the activity
 * manager to tell the application about things it needs to do.
 *
 * {@hide}
 */
public interface IApplicationThread extends IInterface {
    void schedulePauseActivity(IBinder token, boolean finished, boolean userLeaving,
            int configChanges, boolean dontReport) throws RemoteException;
    void scheduleStopActivity(IBinder token, boolean showWindow,
            int configChanges) throws RemoteException;
    void scheduleWindowVisibility(IBinder token, boolean showWindow) throws RemoteException;
    void scheduleSleeping(IBinder token, boolean sleeping) throws RemoteException;
    void scheduleResumeActivity(IBinder token, int procState, boolean isForward, Bundle resumeArgs)
            throws RemoteException;
    void scheduleSendResult(IBinder token, List<ResultInfo> results) throws RemoteException;
    void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException;
    void scheduleRelaunchActivity(IBinder token, List<ResultInfo> pendingResults,
            List<ReferrerIntent> pendingNewIntents, int configChanges, boolean notResumed,
            Configuration config, Configuration overrideConfig) throws RemoteException;
    void scheduleNewIntent(List<ReferrerIntent> intent, IBinder token) throws RemoteException;
    void scheduleDestroyActivity(IBinder token, boolean finished,
            int configChanges) throws RemoteException;
    void scheduleReceiver(Intent intent, ActivityInfo info, CompatibilityInfo compatInfo,
            int resultCode, String data, Bundle extras, boolean sync,
            int sendingUser, int processState) throws RemoteException;
    static final int BACKUP_MODE_INCREMENTAL = 0;
    static final int BACKUP_MODE_FULL = 1;
    static final int BACKUP_MODE_RESTORE = 2;
    static final int BACKUP_MODE_RESTORE_FULL = 3;
    void scheduleCreateBackupAgent(ApplicationInfo app, CompatibilityInfo compatInfo,
            int backupMode) throws RemoteException;
    void scheduleDestroyBackupAgent(ApplicationInfo app, CompatibilityInfo compatInfo)
            throws RemoteException;
    void scheduleCreateService(IBinder token, ServiceInfo info,
            CompatibilityInfo compatInfo, int processState) throws RemoteException;
    void scheduleBindService(IBinder token,
            Intent intent, boolean rebind, int processState) throws RemoteException;
    void scheduleUnbindService(IBinder token,
            Intent intent) throws RemoteException;
    void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
            int flags, Intent args) throws RemoteException;
    void scheduleStopService(IBinder token) throws RemoteException;
    static final int DEBUG_OFF = 0;
    static final int DEBUG_ON = 1;
    static final int DEBUG_WAIT = 2;
    void bindApplication(String packageName, ApplicationInfo info, List<ProviderInfo> providers,
            ComponentName testName, ProfilerInfo profilerInfo, Bundle testArguments,
            IInstrumentationWatcher testWatcher, IUiAutomationConnection uiAutomationConnection,
            int debugMode, boolean openGlTrace, boolean restrictedBackupMode, boolean persistent,
            Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
            Bundle coreSettings) throws RemoteException;
    void scheduleExit() throws RemoteException;
    void scheduleSuicide() throws RemoteException;
    void scheduleConfigurationChanged(Configuration config) throws RemoteException;
    void updateTimeZone() throws RemoteException;
    void clearDnsCache() throws RemoteException;
    void setHttpProxy(String proxy, String port, String exclList,
            Uri pacFileUrl) throws RemoteException;
    void processInBackground() throws RemoteException;
    void dumpService(FileDescriptor fd, IBinder servicetoken, String[] args)
            throws RemoteException;
    void dumpProvider(FileDescriptor fd, IBinder servicetoken, String[] args)
            throws RemoteException;
    void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
            int resultCode, String data, Bundle extras, boolean ordered,
            boolean sticky, int sendingUser, int processState) throws RemoteException;
    void scheduleLowMemory() throws RemoteException;
    void scheduleActivityConfigurationChanged(IBinder token, Configuration overrideConfig)
            throws RemoteException;
    void profilerControl(boolean start, ProfilerInfo profilerInfo, int profileType)
            throws RemoteException;
    void dumpHeap(boolean managed, String path, ParcelFileDescriptor fd)
            throws RemoteException;
    void setSchedulingGroup(int group) throws RemoteException;
    static final int PACKAGE_REMOVED = 0;
    static final int EXTERNAL_STORAGE_UNAVAILABLE = 1;
    void dispatchPackageBroadcast(int cmd, String[] packages) throws RemoteException;
    void scheduleCrash(String msg) throws RemoteException;
    void dumpActivity(FileDescriptor fd, IBinder servicetoken, String prefix, String[] args)
            throws RemoteException;
    void setCoreSettings(Bundle coreSettings) throws RemoteException;
    void scheduleTrimMemory(int level) throws RemoteException;
    // ...
    String descriptor = "android.app.IApplicationThread";
    int SCHEDULE_PAUSE_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION;
    int SCHEDULE_STOP_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+2;
    int SCHEDULE_WINDOW_VISIBILITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+3;
    int SCHEDULE_RESUME_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+4;
    int SCHEDULE_SEND_RESULT_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+5;
    int SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+6;
    int SCHEDULE_NEW_INTENT_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+7;
    int SCHEDULE_FINISH_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+8;
    int SCHEDULE_RECEIVER_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+9;
    int SCHEDULE_CREATE_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+10;
    int SCHEDULE_STOP_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+11;
    int BIND_APPLICATION_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+12;
    int SCHEDULE_EXIT_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+13;
}
```

#### 	IApplicationThread定义了一系列的scheduleXXX方法，从方法名可以看出，基本都是和Activity生命周期相关的方法，除了Activity外，还可以看到Service和Broadcast相关处理方法。

它还继承自**IInterface**，说明它是使用到了**Binder**机制，有客户端和服务端进行通信，来完成上面的定义的接口。

```java
/**
 * Base class for Binder interfaces.  When defining a new interface,
 * you must derive it from IInterface.
 */
public interface IInterface
{
    /**
     * Retrieve the Binder object associated with this interface.
     * You must use this instead of a plain cast, so that proxy objects
     * can return the correct result.
     */
    public IBinder asBinder();
}
```

```java
public abstract class ApplicationThreadNative extends Binder implements IApplicationThread {
    /**
     * Cast a Binder object into an application thread interface, generating
     * a proxy if needed.
     */
    static public IApplicationThread asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IApplicationThread in =
            (IApplicationThread)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        return new ApplicationThreadProxy(obj);
    }
    
    public ApplicationThreadNative() {
        attachInterface(this, descriptor);
    }
    
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            boolean finished = data.readInt() != 0;
            boolean userLeaving = data.readInt() != 0;
            int configChanges = data.readInt();
            boolean dontReport = data.readInt() != 0;
            schedulePauseActivity(b, finished, userLeaving, configChanges, dontReport);
            return true;
        }
        case SCHEDULE_STOP_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            boolean show = data.readInt() != 0;
            int configChanges = data.readInt();
            scheduleStopActivity(b, show, configChanges);
            return true;
        }
        
        case SCHEDULE_WINDOW_VISIBILITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            boolean show = data.readInt() != 0;
            scheduleWindowVisibility(b, show);
            return true;
        }
        case SCHEDULE_RESUME_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            int procState = data.readInt();
            boolean isForward = data.readInt() != 0;
            Bundle resumeArgs = data.readBundle();
            scheduleResumeActivity(b, procState, isForward, resumeArgs);
            return true;
        }
        case SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            Intent intent = Intent.CREATOR.createFromParcel(data);
            IBinder b = data.readStrongBinder();
            int ident = data.readInt();
            ActivityInfo info = ActivityInfo.CREATOR.createFromParcel(data);
            Configuration curConfig = Configuration.CREATOR.createFromParcel(data);
            Configuration overrideConfig = null;
            if (data.readInt() != 0) {
                overrideConfig = Configuration.CREATOR.createFromParcel(data);
            }
            // 上层App中调用，-> ApplicationThread#scheduleLaunchActivity
            scheduleLaunchActivity(intent, b, ident, info, curConfig, overrideConfig, compatInfo,
                    referrer, voiceInteractor, procState, state, persistentState, ri, pi,
                    notResumed, isForward, profilerInfo);
            return true;
        }
        // ...
    }
    public IBinder asBinder()
    {
        return this;
    }
}
```

**ApplicationThreadNative**是一个抽象类，它有一个子类**ApplicationThread**定义在**ActivityThread**中，**ActivityThread**运行在上层App进程中，所以它也是运行在上层App的进程当中。

上面**onTransact**中，调用到了**scheduleLaunchActivity**，它在**ActivityThread$ApplicationThread**中有具体实现。

```java
private class ApplicationThread extends ApplicationThreadNative {
    private static final String DB_INFO_FORMAT = "  %8s %8s %14s %14s  %s";
    private int mLastProcessState = -1;
    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        sendMessage(finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                token, (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                configChanges);
    }
    public final void scheduleStopActivity(IBinder token, boolean showWindow,
            int configChanges) {
       sendMessage(showWindow ? H.STOP_ACTIVITY_SHOW : H.STOP_ACTIVITY_HIDE,
            token, 0, configChanges);
    }
    public final void scheduleWindowVisibility(IBinder token, boolean showWindow) {
        sendMessage(showWindow ? H.SHOW_WINDOW : H.HIDE_WINDOW,
            token);
    }
    public final void scheduleResumeActivity(IBinder token, int processState,
            boolean isForward, Bundle resumeArgs) {
        updateProcessState(processState, false);
        sendMessage(H.RESUME_ACTIVITY, token, isForward ? 1 : 0);
    }
    public final void scheduleSendResult(IBinder token, List<ResultInfo> results) {
        ResultData res = new ResultData();
        res.token = token;
        res.results = results;
        sendMessage(H.SEND_RESULT, res);
    }
    // we use token to identify this activity without having to send the
    // activity itself back to the activity manager. (matters more with ipc)
    @Override
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
        updateProcessState(procState, false);
        ActivityClientRecord r = new ActivityClientRecord();
        r.token = token;
        r.ident = ident;
        r.intent = intent;
        r.referrer = referrer;
        r.voiceInteractor = voiceInteractor;
        r.activityInfo = info;
        r.compatInfo = compatInfo;
        r.state = state;
        r.persistentState = persistentState;
        r.pendingResults = pendingResults;
        r.pendingIntents = pendingNewIntents;
        r.startsNotResumed = notResumed;
        r.isForward = isForward;
        r.profilerInfo = profilerInfo;
        r.overrideConfig = overrideConfig;
        updatePendingConfiguration(curConfig);
        // 发送消息，上层应用启动Activity
        sendMessage(H.LAUNCH_ACTIVITY, r);
    }
    // ...
}
```

**ApplicationThread**继承自**ApplicationThreadNative**，也是一个**Binder**，它在接收到客户端的请求之后，会根据不同的消息，执行相应的方法。

**ApplicationThread**中**scheduleLaunchActivity**通过Handler发送消息**H.LAUNCH_ACTIVITY**，该消息对应了方法**handleLaunchActivity**，继而调用**performLaunchActivity**，开始启动Activity，到此，上层的调用流程基本清晰了。

##### ApplicationThreadProxy

**IApplicationThread**还有一个子类**ApplicationThreadProxy**，该类实现了**IApplicationThread**的所有方法。该类在AMS系统服务层调用，在**AMS**处理好请求之后，往**Binder**服务端发送消息

需要注意的是，此时**SystemProcess**是作为客户端的角色，往上层App服务端发送消息。

在`ActivityStackSupervisor#realStartActivityLocked`中，通过调用`app.thread.scheduleLaunchActivity`发送消息给上层App，此时`app.thread`即是**ApplicationThreadProxy**实例对象，该实例对象是在创建应用程序进程的时候生成的。

```java
final boolean realStartActivityLocked(ActivityRecord r,
        ProcessRecord app, boolean andResume, boolean checkConfig)
    // 告诉上层App开始启动Activity，app.thread即是ApplicationThreadProxy对象。
    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken, ...);
}
```

```java
class ApplicationThreadProxy implements IApplicationThread {
    private final IBinder mRemote;
    
    public ApplicationThreadProxy(IBinder remote) {
        mRemote = remote;
    }
    
    public final IBinder asBinder() {
        return mRemote;
    }
    
    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(finished ? 1 : 0);
        data.writeInt(userLeaving ? 1 :0);
        data.writeInt(configChanges);
        data.writeInt(dontReport ? 1 : 0);
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    public final void scheduleStopActivity(IBinder token, boolean showWindow,
            int configChanges) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(showWindow ? 1 : 0);
        data.writeInt(configChanges);
        mRemote.transact(SCHEDULE_STOP_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    public final void scheduleWindowVisibility(IBinder token,
            boolean showWindow) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(showWindow ? 1 : 0);
        mRemote.transact(SCHEDULE_WINDOW_VISIBILITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    public final void scheduleSleeping(IBinder token,
            boolean sleeping) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(sleeping ? 1 : 0);
        mRemote.transact(SCHEDULE_SLEEPING_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    public final void scheduleResumeActivity(IBinder token, int procState, boolean isForward,
            Bundle resumeArgs)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(procState);
        data.writeInt(isForward ? 1 : 0);
        data.writeBundle(resumeArgs);
        mRemote.transact(SCHEDULE_RESUME_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    public final void scheduleSendResult(IBinder token, List<ResultInfo> results)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeTypedList(results);
        mRemote.transact(SCHEDULE_SEND_RESULT_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        intent.writeToParcel(data, 0);
        data.writeStrongBinder(token);
        data.writeInt(ident);
        info.writeToParcel(data, 0);
        curConfig.writeToParcel(data, 0);
        if (overrideConfig != null) {
            data.writeInt(1);
            overrideConfig.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        compatInfo.writeToParcel(data, 0);
        data.writeString(referrer);
        data.writeStrongBinder(voiceInteractor != null ? voiceInteractor.asBinder() : null);
        data.writeInt(procState);
        data.writeBundle(state);
        data.writePersistableBundle(persistentState);
        data.writeTypedList(pendingResults);
        data.writeTypedList(pendingNewIntents);
        data.writeInt(notResumed ? 1 : 0);
        data.writeInt(isForward ? 1 : 0);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
}
```

**scheduleLaunchActivity**方法，调用**mRemote.transact**发送消息，该消息最终发送给**ActivityThread**，在**ActivityThread#onTransact**中，根据消息的标志`SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION`，开始启动Activity。





















