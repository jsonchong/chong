### Handler之postSyncBarrier消息屏障

简单理解为 异步消息插队并优先执行。
场景：排队买票
先来了一个普通用户来排队，买完票走了。

后面又来了一个VIP用户A来买票 就一直站在卖窗口这里 也不走（ps：添加屏障 )

简单的来说就是优于事件回调执行，为了做一些优先级更高的操作 比如 视图刷新。当一个Handler消息来时 会优先执行同步屏障消息事件。

以便系统底层可以做一些比上层业务更加重要的消息事件 ，所以 这个方法 **被注解成hide** 也是系统给自己开了一道后门。不然的话把方法公开给应用去使用，那么很可能把系统卡成翔而导致掉帧。

```java
  @hide
  public int postSyncBarrier() {
      return postSyncBarrier(SystemClock.uptimeMillis());
  }
```

```java
  private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

