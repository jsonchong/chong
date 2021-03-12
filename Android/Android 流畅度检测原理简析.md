android在不同的版本都会优化“UI的流畅性”问题，但是直到在android 4.1版本中做了有效的优化，这就是Project Butter。

Project Butter加入了三个核心元素:VSYNC、Triple Buffer和Choreographer。其中，VSYNC是理解Project Buffer的核心。VSYNC是Vertical Synchronization的缩写 也就是“垂直同步”

- VSYNC：产生一个中断信号
- Triple Buffer：当双Buffer不够使用时，该系统可分配第三块Buffer
- Choreographer：这个了用来接受一个VSYNC信号来统一协调UI更新

检测应用卡顿的方案，**Android系统每隔16.6ms发出VSYNC信号**，来通知界面进行输入、动画、绘制等动作，每一次同步的周期为16.6ms，代表一帧的刷新频率，理论上来说两次回调的时间周期应该在16.6ms，如果超过了16.6ms我们则认为发生了卡顿，利用两次回调间的时间周期来判断是否发生卡顿 这个方案的原理主要是通过Choreographer类设置它的FrameCallback函数，**当每一帧被渲染时会触发回调FrameCallback， FrameCallback回调void doFrame (long frameTimeNanos)函数。一次界面渲染会回调doFrame方法**，如果两次doFrame之间的间隔大于16.6ms说明发生了卡顿。

监控应用的流畅度一般都是通过Choreographer类的postFrameCallback方法注册一个VSYNC回调事件

```java
public static void start(final Builder builder) {
        Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
            long lastFrameTimeNanos = 0;
            long currentFrameTimeNanos = 0;

            @Override
            public void doFrame(long frameTimeNanos) {
                if (lastFrameTimeNanos == 0) {
                    lastFrameTimeNanos = frameTimeNanos;
                    LogMonitor.getInstance().setFrequency(builder.frame * 17 / 2);
                    if (builder.targetPackageName != null) {
                        LogMonitor.getInstance().setTargetPackageName(builder.targetPackageName);
                    }
                    LogMonitor.getInstance().setDumpListener(builder.onDumpListener);
                }
                currentFrameTimeNanos = frameTimeNanos;
                skipFrameCount = skipFrameCount(lastFrameTimeNanos, currentFrameTimeNanos, deviceRefreshRateMs);
                LogMonitor.getInstance().setFrame(skipFrameCount);
                if (LogMonitor.getInstance().isMonitor()) {
                    LogMonitor.getInstance().removeMonitor();
                }
                LogMonitor.getInstance().startMonitor();
                lastFrameTimeNanos = currentFrameTimeNanos;
                Choreographer.getInstance().postFrameCallback(this);
            }
        });
    }

```

