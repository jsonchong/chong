### 通过Looper检测代码是否有卡顿

UI线程也就是主线程里有个Looper，在其loop()方法中会不断取出message，调用其绑定的Handler在主线程执行。如果在handler的dispatchMesaage方法里有耗时操作，就会发生卡顿

```java
   // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
```



只要检测 msg.target.dispatchMessage(msg) 的执行时间，就能检测到主线程是否有耗时操作。注意到这行执行代码的前后，有两个logging.println函数，如果设置了mLogging，会分别打印出”>>>>> Dispatching to “和”<<<<< Finished to “这样的日志，这样我们就可以通过两次log的时间差值，来计算dispatchMessage的执行时间，从而设置阈值判断是否发生了卡顿



