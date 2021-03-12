### 正文

1. 大多数应用在屏幕上一次显示三个层：屏幕顶部的状态栏、底部或侧面的导航栏以及应用界面。有些应用会拥有更多或更少的层（例如，默认主屏幕应用有一个单独的壁纸层，而全屏游戏可能会隐藏状态栏）。每个层都可以单独更新。状态栏和导航栏由系统进程渲染，而应用层由应用渲染，两者之间不进行协调。
2. 设备显示会按一定速率刷新，在手机和平板电脑上通常为 60 fps。如果显示内容在刷新期间更新，则会出现撕裂现象；因此，请务必只在周期之间更新内容。在可以安全更新内容时，系统便会收到来自显示设备的信号。由于历史原因，我们将该信号称为 VSYNC 信号。
3. 刷新率可能会随时间而变化，例如，一些移动设备的帧率范围在 58 fps 到 62 fps 之间，具体要视当前条件而定。对于连接了 HDMI 的电视，刷新率在理论上可以下降到 24 Hz 或 48 Hz，以便与视频相匹配。由于每个刷新周期只能更新屏幕一次，因此以 200 fps 的帧率为显示设备提交缓冲区就是一种资源浪费，因为大多数帧会被舍弃掉。**SurfaceFlinger 不会在应用每次提交缓冲区时都执行操作，而是在显示设备准备好接收新的缓冲区时才会唤醒**。
4. 当 VSYNC 信号到达时，SurfaceFlinger 会遍历它的层列表，以寻找新的缓冲区。如果找到新的缓冲区，它会获取该缓冲区；否则，它会继续使用以前获取的缓冲区。SurfaceFlinger 必须始终显示内容，因此它会保留一个缓冲区。如果在某个层上没有提交缓冲区，则该层会被忽略。
5. SurfaceFlinger 在收集可见层的所有缓冲区之后，便会询问 Hardware Composer 应如何进行合成。」

下面是上述流程所对应的流程图， 简单地说， SurfaceFlinger 最主要的功能:**SurfaceFlinger 接受来自多个来源的数据缓冲区，对它们进行合成，然后发送到显示设备**

<img src="https://www.androidperformance.com/images/15816781462135.jpg" alt="img" style="zoom:50%;" />

这里以滑动列表为例 ，我们截取主线程和渲染线程**一帧**的工作流程(每一帧都会遵循这个流程，不过有的帧需要处理的事情多，有的帧需要处理的事情少) ，重点看 “UI Thread ” 和 RenderThread 这两行

<img src="https://www.androidperformance.com/images/15732904872967.jpg" alt="img" style="zoom:50%;" />



**这张图对应的工作流程如下**

1. 主线程处于 Sleep 状态，等待 Vsync 信号
2. Vsync 信号到来，主线程被唤醒，Choreographer 回调 FrameDisplayEventReceiver.onVsync 开始一帧的绘制
3. 处理 App 这一帧的 Input 事件(如果有的话)
4. 处理 App 这一帧的 Animation 事件(如果有的话)
5. 处理 App 这一帧的 Traversal 事件(如果有的话)
6. 主线程与渲染线程同步渲染数据，同步结束后，主线程结束一帧的绘制，可以继续处理下一个 Message(如果有的话，IdleHandler 如果不为空，这时候也会触发处理)，或者进入 Sleep 状态等待下一个 Vsync
7. **渲染线程首先需要从 BufferQueue 里面取一个 Buffer(dequeueBuffer) , 进行数据处理之后，调用 OpenGL 相关的函数，真正地进行渲染操作，然后将这个渲染好的 Buffer 还给 BufferQueue (queueBuffer)** , SurfaceFlinger 在 Vsync-SF 到了之后，将所有准备好的 Buffer 取出进行合成

### 主线程运行机制的本质

在讲 Choreographer 之前，我们先理一下 Android 主线程运行的本质，其实就是 Message 的处理过程，我们的各种操作，包括每一帧的渲染操作 ，都是通过 Message 的形式发给主线程的 MessageQueue ，MessageQueue 处理完消息继续等下一个消息

### 演进

引入 Vsync 之前的 Android 版本，渲染一帧相关的 Message ，中间是没有间隔的，上一帧绘制完，下一帧的 Message 紧接着就开始被处理。这样的问题就是，帧率不稳定，可能高也可能低，不稳定

<img src="https://www.androidperformance.com/images/15717420572997.jpg" alt="img" style="zoom:70%;" />

可以看到这时候的瓶颈是在 dequeueBuffer, 因为屏幕是有刷新周期的, FB 消耗 Front Buffer 的速度是一定的, 所以 SF 消耗 App Buffer 的速度也是一定的, 所以 App 会卡在 dequeueBuffer 这里,这就会导致 App Buffer 获取不稳定, 很容易就会出现卡顿掉帧的情况.

所以 Android 的演进中，引入了 **Vsync + TripleBuffer + Choreographer** 的机制，**其主要目的就是提供一个稳定的帧率输出机制**，让软件层和硬件层可以以共同的频率一起工作。

### 引入 Choreographer

Choreographer 的引入，主要是配合 Vsync ，给上层 App 的渲染提供一个稳定的 Message 处理的时机，也就是 Vsync 到来的时候 ，系统通过对 Vsync 信号周期的调整，来控制每一帧绘制操作的时机. 至于为什么 Vsync 周期选择是 16.6ms (60 fps) ，是因为目前大部分手机的屏幕都是 60Hz 的刷新率，也就是 16.6ms 刷新一次，系统为了配合屏幕的刷新频率，将 Vsync 的周期也设置为 16.6 ms，每隔 16.6 ms ，Vsync 信号到来唤醒 Choreographer 来做 App 的绘制操作 ，如果每个 Vsync 周期应用都能渲染完成，那么应用的 fps 就是 60 ，给用户的感觉就是非常流畅，这就是引入 Choreographer 的主要作用

<img src="https://www.androidperformance.com/images/15722752299458.jpg" alt="img" style="zoom:67%;" />

当然目前使用 90Hz 刷新率屏幕的手机越来越多，Vsync 周期从 16.6ms 到了 11.1ms，上图中的操作要在更短的时间内完成，对性能的要求也越来越高

### Choreographer 简介

Choreographer 扮演 Android 渲染链路中承上启下的角色

1. **承上**：负责接收和处理 App 的各种更新消息和回调，等到 Vsync 到来的时候统一处理。比如集中处理 Input(主要是 Input 事件的处理) 、Animation(动画相关)、Traversal(包括 measure、layout、draw 等操作) ，判断卡顿掉帧情况，记录 CallBack 耗时等
2. **启下**：负责请求和接收 Vsync 信号。接收 Vsync 事件回调(通过 FrameDisplayEventReceiver.onVsync )；请求 Vsync(FrameDisplayEventReceiver.scheduleVsync) .

从上面可以看出来， Choreographer 担任的是一个工具人的角色，他之所以重要，是因为通过 **Choreographer + SurfaceFlinger + Vsync + TripleBuffer** 这一套从上到下的机制，保证了 Android App 可以以一个稳定的帧率运行(目前大部分是 60fps)，减少帧率波动带来的不适感.

了解 Choreographer 还可以帮助 App 开发者知道程序每一帧运行的基本原理，也可以加深对 **Message、Handler、Looper、MessageQueue、Measure、Layout、Draw** 的理解 , 很多 **APM** 工具也用到了 **Choreographer( 利用 FrameCallback + FrameInfo )** + **MessageQueue ( 利用 IdleHandler )** + **Looper ( 设置自定义 MessageLogging)** 这些组合拳，深入了解了这些之后，再去做优化，脑子里的思路会更清晰。

### 从 Systrace 的角度来看 Choreogrepher 的工作流程

下图以滑动桌面为例子，我们先看一下从左到右滑动桌面的一个完整的预览图（App 进程），可以看到 Systrace 中从左到右，每一个绿色的帧都表示一帧，表示最终我们可以手机上看到的画面

1. 图中每一个灰色的条和白色的条宽度是一个 Vsync 的时间，也就是 16.6ms
2. 每一帧处理的流程：接收到 Vsync 信号回调-> UI Thread –> RenderThread –> SurfaceFlinger(图中未显示)
3. UI Thread 和 RenderThread 就可以完成 App 一帧的渲染，渲染完的 Buffer 抛给 SurfaceFlinger 去合成，然后我们就可以在屏幕上看到这一帧了
4. 可以看到桌面滑动的每一帧耗时都很短（Ui Thread 耗时 + RenderThread 耗时），但是由于 Vsync 的存在，每一帧都会等到 Vsync 才会去做处理

<img src="https://www.androidperformance.com/images/15717420793673.jpg" alt="img" style="zoom:50%;" />

有了上面这个整体的概念，我们将 UI Thread 的每一帧放大来看，看看 Choreogrepher 的位置以及 Choreogrepher 是怎么组织每一帧的

<img src="https://www.androidperformance.com/images/15717420863795.jpg" alt="img" style="zoom:50%;" />

### Choreographer 的工作流程

Choreographer 初始化

1. 初始化 **FrameHandler ，绑定 Looper**
2. 初始化 **FrameDisplayEventReceiver ，与 SurfaceFlinger 建立通信用于接收和请求 Vsync**
3. 初始化 CallBackQueues

SurfaceFlinger 的 appEventThread 唤醒发送 Vsync ，Choreographer 回调 FrameDisplayEventReceiver.onVsync , 进入 Choreographer 的主处理函数 doFrame

Choreographer.doFrame 计算掉帧逻辑

Choreographer.doFrame 处理 Choreographer 的第一个 callback ： input

Choreographer.doFrame 处理 Choreographer 的第二个 callback ： animation

Choreographer.doFrame 处理 Choreographer 的第三个 callback ： insets animation

Choreographer.doFrame 处理 Choreographer 的第四个 callback ： traversal

1. traversal-draw 中 UIThread 与 RenderThread 同步数据

Choreographer.doFrame 处理 Choreographer 的第五个 callback ： commit ?

RenderThread 处理绘制数据，真正进行渲染

将渲染好的 Buffer swap 给 SurfaceFlinger 进行合成

**第一步初始化完成后，后续就会在步骤 2-9 之间循环**

<img src="https://www.androidperformance.com/images/15717420948412.jpg" alt="img" style="zoom:50%;" />

### 源码解析

下面从源码的角度来简单看一下，源码只摘抄了部分重要的逻辑，其他的逻辑则被剔除，另外 Native 部分与 SurfaceFlinger 交互的部分也没有列入

### Choreographer 的初始化

```java
// Thread local storage for the choreographer.
private static final ThreadLocal<Choreographer> sThreadInstance =
        new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        // 获取当前线程的 Looper
        Looper looper = Looper.myLooper();
        ......
        // 构造 Choreographer 对象
        Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
        if (looper == Looper.getMainLooper()) {
            mMainInstance = choreographer;
        }
        return choreographer;
    }
};
```

### Choreographer 的构造函数

```java
private Choreographer(Looper looper, int vsyncSource) {
    mLooper = looper;
    // 1. 初始化 FrameHandler
    mHandler = new FrameHandler(looper);
    // 2. 初始化 DisplayEventReceiver
    mDisplayEventReceiver = USE_VSYNC
            ? new FrameDisplayEventReceiver(looper, vsyncSource)
            : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
    //3. 初始化 CallbacksQueues
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
    ......
}
```

### FrameHandler

```java
private final class FrameHandler extends Handler {
    ......
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME://开始渲染下一帧的操作
                doFrame(System.nanoTime(), 0);
                break;
            case MSG_DO_SCHEDULE_VSYNC://请求 Vsync 
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK://处理 Callback
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}
```

### Choreographer 初始化链

在 Activity 启动过程，执行完 onResume 后，会调用 Activity.makeVisible()，然后再调用到 addView()， 层层调用会进入如下方法

```java
ActivityThread.handleResumeActivity(IBinder, boolean, boolean, String) (android.app) 
-->WindowManagerImpl.addView(View, LayoutParams) (android.view) 
  -->WindowManagerGlobal.addView(View, LayoutParams, Display, Window) (android.view) 
    -->ViewRootImpl.ViewRootImpl(Context, Display) (android.view) 
    public ViewRootImpl(Context context, Display display) {
        ......
        mChoreographer = Choreographer.getInstance();
        ......
    }
```

### FrameDisplayEventReceiver 简介

Vsync 的注册、申请、接收都是通过 FrameDisplayEventReceiver 这个类，所以可以先简单介绍一下。 FrameDisplayEventReceiver 继承 DisplayEventReceiver ， 有三个比较重要的方法

1. onVsync – Vsync 信号回调
2. run – 执行 doFrame
3. scheduleVsync – 请求 Vsync 信号

```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
    ......
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
        ......
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }
    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
    
    public void scheduleVsync() {
        ......  
        nativeScheduleVsync(mReceiverPtr);
        ......
    }
}
```

### Choreographer 中 Vsync 的注册

从下面的函数调用栈可以看到，Choreographer 的内部类 FrameDisplayEventReceiver.onVsync 负责接收 Vsync 回调，通知 UIThread 进行数据处理。

那么 FrameDisplayEventReceiver 是通过什么方式在 **Vsync 信号到来的时候回调 onVsync 呢**？答案是 FrameDisplayEventReceiver 的初始化的时候，**最终通过监听文件句柄的形式**，其对应的初始化流程如下

android/view/Choreographer.java

```java
private Choreographer(Looper looper, int vsyncSource) {
    mLooper = looper;
    mDisplayEventReceiver = USE_VSYNC
            ? new FrameDisplayEventReceiver(looper, vsyncSource)
            : null;
    ......
}
```

android/view/Choreographer.java

```java
public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
    super(looper, vsyncSource);
}
```

android/view/DisplayEventReceiver.java

```java
public DisplayEventReceiver(Looper looper, int vsyncSource) {
    ......
    mMessageQueue = looper.getQueue();
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
            vsyncSource);
}
```

简单来说，FrameDisplayEventReceiver 的初始化过程中，通过 BitTube(本质是一个 socket pair)，来传递和请求 Vsync 事件，当 SurfaceFlinger 收到 Vsync 事件之后，通过 appEventThread 将这个事件通过 BitTube 传给 DisplayEventDispatcher ，DisplayEventDispatcher 通过 BitTube 的接收端监听到 Vsync 事件之后，回调 Choreographer.FrameDisplayEventReceiver.onVsync ，触发开始一帧的绘制

### Choreographer 处理一帧的逻辑

Choreographer 处理绘制的逻辑核心在 Choreographer.doFrame 函数中，从下图可以看到，FrameDisplayEventReceiver.onVsync post 了自己，其 run 方法直接调用了 doFrame 开始一帧的逻辑处理

doFrame 函数主要做下面几件事

1. 计算掉帧逻辑
2. 记录帧绘制信息
3. 执行 CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_INSETS_ANIMATION、CALLBACK_TRAVERSAL、CALLBACK_COMMIT

### 计算掉帧逻辑

```java
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        ......
        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
        }
        ......
    }
    ......
}
```

Choreographer.doFrame 的掉帧检测比较简单，从下图可以看到，Vsync 信号到来的时候会标记一个 start_time ，执行 doFrame 的时候标记一个 end_time ，这两个时间差就是 Vsync 处理时延，也就是掉帧

<img src="https://www.androidperformance.com/images/15717421364722.jpg" alt="img" style="zoom:50%;" />

<img src="https://www.androidperformance.com/images/15717421441350.jpg" alt="img" style="zoom:50%;" />

这里需要注意的是，这种方法计算的掉帧，是前一帧的掉帧情况，而不是这一帧的掉帧情况，这个计算方法是有缺陷的，会导致有的掉帧没有被计算到

### 记录帧绘制信息

Choreographer 中 FrameInfo 来负责记录帧的绘制信息，doFrame 执行的时候，会把每一个关键节点的绘制时间记录下来，我们使用 dumpsys gfxinfo 就可以看到。

doFrame 函数记录从 Vsync time 到 markPerformTraversalsStart 的时间

```java
void doFrame(long frameTimeNanos, int frame) {
    ......
    mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
    // 处理 CALLBACK_INPUT Callbacks 
    mFrameInfo.markInputHandlingStart();
    // 处理 CALLBACK_ANIMATION Callbacks
    mFrameInfo.markAnimationsStart();
    // 处理 CALLBACK_INSETS_ANIMATION Callbacks
    // 处理 CALLBACK_TRAVERSAL Callbacks
    mFrameInfo.markPerformTraversalsStart();
    // 处理 CALLBACK_COMMIT Callbacks
    ......
}
```

### 执行 Callbacks

```java
void doFrame(long frameTimeNanos, int frame) {
    ......
    // 处理 CALLBACK_INPUT Callbacks 
    doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
    // 处理 CALLBACK_ANIMATION Callbacks
    doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
    // 处理 CALLBACK_INSETS_ANIMATION Callbacks
    doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
    // 处理 CALLBACK_TRAVERSAL Callbacks
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    // 处理 CALLBACK_COMMIT Callbacks
    doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    ......
}
```

### **Input 回调调用栈**

**input callback** 一般是执行 ViewRootImpl.ConsumeBatchedInputRunnable

android/view/ViewRootImpl.java

```java
final class ConsumeBatchedInputRunnable implements Runnable {
    @Override
    public void run() {
        doConsumeBatchedInput(mChoreographer.getFrameTimeNanos());
    }
}
void doConsumeBatchedInput(long frameTimeNanos) {
    if (mConsumeBatchedInputScheduled) {
        mConsumeBatchedInputScheduled = false;
        if (mInputEventReceiver != null) {
            if (mInputEventReceiver.consumeBatchedInputEvents(frameTimeNanos)
                    && frameTimeNanos != -1) {
                scheduleConsumeBatchedInput();
            }
        }
        doProcessInputEvents();
    }
}
```

Input 时间经过处理，最终会传给 DecorView 的 dispatchTouchEvent，这就到了我们熟悉的 Input 事件分发

### **Animation 回调调用栈**

一般我们接触的多的是调用 View.postOnAnimation 的时候，会使用到 CALLBACK_ANIMATION

```java
public void postOnAnimation(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.mChoreographer.postCallback(
                Choreographer.CALLBACK_ANIMATION, action, null);
    } else {
        // Postpone the runnable until we know
        // on which thread it needs to run.
        getRunQueue().post(action);
    }
}
```

那么一般是什么时候回调用到 View.postOnAnimation 呢，接触最多的应该是 startScroll，Fling 这种操作

### **Traversal 调用栈**

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //为了提高优先级，先 postSyncBarrier
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        // 真正开始执行 measure、layout、draw
        doTraversal();
    }
}
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 这里把 SyncBarrier remove
mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        // 真正开始
        performTraversals();
    }
}
private void performTraversals() {
      // measure 操作
      if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight() || contentInsetsChanged || updatedConfiguration) {
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      }
      // layout 操作
      if (didLayout) {
          performLayout(lp, mWidth, mHeight);
      }
      // draw 操作
      if (!cancelDraw && !newSurface) {
          performDraw();
      }
}
```

### 下一帧的 Vsync 请求

由于动画、滑动、Fling 这些操作的存在，我们需要一个连续的、稳定的帧率输出机制。这就涉及到了 Vsync 的请求逻辑，在连续的操作，比如动画、滑动、Fling 这些情况下，每一帧的 doFrame 的时候，都会根据情况触发下一个 Vsync 的申请，这样我们就可以获得连续的 Vsync 信号。

我们比较熟悉的 invalidate 和 requestLayout 都会触发 Vsync 信号请求

我们下面以 Animation 为例，看看 Animation 是如何驱动下一个 Vsync ，来持续更新画面的

### ObjectAnimator 动画驱动逻辑

android/animation/ValueAnimator.java

```java
private void start(boolean playBackwards) {
    ......
    addAnimationCallback(0); // 动画 start 的时候添加 Animation Callback 
    ......
}
private void addAnimationCallback(long delay) {
    ......
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}
```

android/animation/AnimationHandler.java

```java
public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    if (mAnimationCallbacks.size() == 0) {
        // post FrameCallback
        getProvider().postFrameCallback(mFrameCallback);
    }
    ......
}

// 这里的 mFrameCallback 回调 doFrame，里面 post了自己
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        doAnimationFrame(getProvider().getFrameTime());
        if (mAnimationCallbacks.size() > 0) {
            // post 自己
            getProvider().postFrameCallback(this);
        }
    }
};
```

调用 postFrameCallback 会走到 mChoreographer.postFrameCallback ，这里就会触发 Choreographer 的 Vsync 请求逻辑

android/view/Choreographer.java

```java
private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
            // 请求 Vsync scheduleFrameLocked ->scheduleVsyncLocked-> mDisplayEventReceiver.scheduleVsync ->nativeScheduleVsync
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

通过上面的 Animation.start 设置，利用了 Choreographer.FrameCallback 接口，每一帧都去请求下一个 Vsync

**Choreographer小结**

1. **Choreographer** 是线程单例的，而且必须要和一个 Looper 绑定，因为其内部有一个 Handler 需要和 Looper 绑定，一般是 App 主线程的 Looper 绑定

2. **DisplayEventReceiver** 是一个 abstract class，其 JNI 的代码部分会创建一个IDisplayEventConnection 的 Vsync 监听者对象。这样，来自 AppEventThread 的 VSYNC 中断信号就可以传递给 Choreographer 对象了。当 Vsync 信号到来时，DisplayEventReceiver 的 onVsync 函数将被调用。

3. **DisplayEventReceiver** 还有一个 scheduleVsync 函数。当应用需要绘制UI时，将首先申请一次 Vsync 中断，然后再在中断处理的 onVsync 函数去进行绘制。

4. **Choreographer** 定义了一个 **FrameCallback** **interface**，每当 Vsync 到来时，其 doFrame 函数将被调用。这个接口对 Android Animation 的实现起了很大的帮助作用。以前都是自己控制时间，现在终于有了固定的时间中断。

5. Choreographer

    

   的主要功能是，当收到 Vsync 信号时，去调用使用者通过 postCallback 设置的回调函数。目前一共定义了五种类型的回调，它们分别是：

   1. **CALLBACK_INPUT** : 处理输入事件处理有关
   2. **CALLBACK_ANIMATION** ： 处理 Animation 的处理有关
   3. **CALLBACK_INSETS_ANIMATION** ： 处理 Insets Animation 的相关回调
   4. **CALLBACK_TRAVERSAL** : 处理和 UI 等控件绘制有关
   5. **CALLBACK_COMMIT** ： 处理 Commit 相关回调

6. **ListView** 的 Item 初始化(obtain\setup) 会在 input 里面也会在 animation 里面，这取决于

7. **CALLBACK_INPUT** 、**CALLBACK_ANIMATION** 会修改 view 的属性，所以要先与 CALLBACK_TRAVERSAL 执行

### APM 与 Choreographer

由于 Choreographer 的位置，许多性能监控的手段都是利用 Choreographer 来做的，除了自带的掉帧计算，Choreographer 提供的 FrameCallback 和 FrameInfo 都给 App 暴露了接口，让 App 开发者可以通过这些方法监控自身 App 的性能

### 利用 FrameCallback 的 doFrame 回调

```java
public interface FrameCallback {
    public void doFrame(long frameTimeNanos);
}
```

```java
Choreographer.getInstance().postFrameCallback(youOwnFrameCallback );
```

TinyDancer 就是使用了这个方法来计算 FPS (https://github.com/friendlyrobotnyc/TinyDancer)

### MessageQueue 与 Choreographer

所谓的异步消息其实就是这样的，我们可以通过 enqueueBarrier 往消息队列中插入一个 Barrier，那么队列中执行时间在这个 Barrier 以后的同步消息都会被这个 Barrier 拦截住无法执行，直到我们调用 removeBarrier 移除了这个 Barrier，而异步消息则没有影响，消息默认就是同步消息，除非我们调用了 Message 的 setAsynchronous，这个方法是隐藏的。只有在初始化 Handler 时通过参数指定往这个 Handler 发送的消息都是异步的，这样在 Handler 的 enqueueMessage 中就会调用 Message 的 setAsynchronous 设置消息是异步的，从上面 Handler.enqueueMessage 的代码中可以看到

### SyncBarrier 在 Choreographer 中使用的一个示例

scheduleTraversals 的时候 postSyncBarrier

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //为了提高优先级，先 postSyncBarrier
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}
```

doTraversal 的时候 removeSyncBarrier

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 这里把 SyncBarrier remove
mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        // 真正开始
        performTraversals();
    }
}
```

Choreographer post Message 的时候，会把这些消息设为 Asynchronous ，这样Choreographer 中的这些 Message 的优先级就会比较高

![image-20201222195307886](/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201222195307886.png)

```java
Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
msg.arg1 = callbackType;
msg.setAsynchronous(true);
mHandler.sendMessageAtTime(msg, dueTime);
```

### 厂商优化

### 后台动画优化

当一个 Android App 退到后台之后，只要他没有被杀死，那么他做什么事情大家都不要奇怪，因为这就是 Android。有的 App 退到后台之后还在持续调用 Choreographer 中的 Animation Callback，而这个 Callback 的执行完全是无意义的，而且用户还不知道，但是对 cpu 的占用是比较高的。

所以在 Choreographer 中会针对这种情况做优化，禁止不符合条件的 App 在后台继续无用的操作

![img](https://www.androidperformance.com/images/15717422623134.jpg)

### 帧绘制优化

和移动事件优化一样，由于有了 Input 事件的信息，在某些场景下我们可以通知 SurfaceFlinger 不用去等待 Vsync 直接做合成操作

### 应用启动优化

我们前面说，主线程的所有操作都是给予 Message 的 ，如果某个操作，非重要的 Message 被排列到了队列后面，那么对这个操作产生影响；而通过重新排列 MessageQueue，在应用启动的时候，把启动相关的重要的启动 Message 放到队列前面，来起到加快启动速度的作用

### 高帧率优化

90 fps 的手机上 ， Vsync 间隔从 16.6ms 变成了 11.1ms ，这带来了巨大的性能和功耗挑战，如何在一帧内完成渲染的必要操作，是手机厂商必须要思考和优化的地方：

1. 超级 App 的性能表现以及优化
2. 游戏高帧率合作
3. 90 fps 和 60 fps 相互切换的逻辑





























































































