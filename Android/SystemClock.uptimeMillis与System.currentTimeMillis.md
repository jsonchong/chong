### 时间间隔： SystemClock.uptimeMillis与System.currentTimeMillis

SystemClock.uptimeMillis 和 System.currentTimeMillis

这两种方法有何区别呢？

1. SystemClock.uptimeMillis()  // 从开机到现在的毫秒数（手机睡眠的时间不包括在内）；
2.  System.currentTimeMillis() // 从1970年1月1日 UTC到现在的毫秒数；

但是，第2个时间，是可以通过System.setCurrentTimeMillis修改的，那么，在某些情况下，一但被修改，时间间隔就不准了。

特别说明点：AnimationUtils 中明确说了：

```
  /**
     * Returns the current animation time in milliseconds. 
     * This time should be used when invoking
     * {@link Animation#setStartTime(long)}. Refer to 
     * {@link android.os.SystemClock} for more
     * information about the different available clocks. 
     * The clock used by this method is
     * <em>not</em> the "wall" clock (it is not 
     * {@link System#currentTimeMillis}).
     *
     * @return the current animation time in milliseconds
     *
     * @see android.os.SystemClock

```

想想看，假如用第2个时间来做动画，万一被其它应用在动画过程中修改了时间，呃，这个是无法想像的