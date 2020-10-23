### Activity的生命周期

1. onCreate: 表示Activity正在被创建，这是生命周期的第一个方法，比如调用setContentView去加载界面布局资源，初始化Activity所需要的数据
2. onRestart: 表示Activity正在重新启动，当当前Activity从不可见重新变为可见状态时，比如按Home键切换到桌面或者用户打开了一个新的Activity,当前的Activity会onPause和onStop,接着用户又回到了这个Activity就会调用onRestart.
3. onStart: 这时Activity已经可见了，但还是没有出现到前台，可以理解为Activity已经显示出来了，但是我们还看不到。
4. onResume: 表示Activity已经可见了，onStart和onResume都表示Activity已经可见，但是onStart的时候Activity还在后台，onResume的时候Activity才显示到前台.
5. onPause: 表示Activity正在停止，正常情况下，紧接着onStop就会被调用。特殊情况下，如果这个时候快速地再回到当前Activity,那么onResume会被调用，这种情况属于极端情况，用户操作很难复现这一场景。此时可以做一些存储数据，停止动画等工作，但是注意不能太耗时，会影响到新Activity的显示，onPause必须先执行完，新Activity的onResume才会执行。
6. onStop: 表示Activity即将停止，可以做一些稍微重量级的回收工作，同样不能太耗时。
7. onDestory: 表示Activity即将被销毁，这是Activity生命周期中的最后一个回调，在这里我们可以做一些回收工作和最终的资源释放。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200915173432766.png" alt="image-20200915173432766" style="zoom:30%;" />

Activity生命周期的切换过程

注：当用户打开新的Activity或者切换到桌面的时候，回调如下：onPause -> onStop,有一种特殊情况是如果新的Activity采用了透明主题，那么当前Activity不会回调onStop.

问题1：onStart和onResume、onPause和onStop有什么本质上的不同吗

onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调的。

### 异常情况下的生命周期分析

**情况1：资源相关的系统配置发生改变Activity被杀死并重新创建**

比如当前Activity处于竖屏状态，如果突然旋转屏幕，由于系统配置发生了变化，在默认情况下Activity就会被销毁并且重新创建，当然也可以阻止系统重新创建我们的Activity

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200915175000048.png" alt="image-20200915175000048" style="zoom:50%;" />

当系统配置发生改变后，Activity会被销毁，其onPause、onStop、onDestory均会被调用，同时**由于Activity是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前Activity的状态。调用时机是在onStop之前**，需要强调的是，这个方法只会出现在Activity被异常终止的情况下，正常情况下不会回调这个方法。当Activity被重新创建后，系统会调用onRestoreInstanceState,并且把Activity销毁时onSaveInstanceState方法所保存的Bundle对象作为参数同时传递给onRestoreInstanceState和onCreate方法，**onRestoreInstanceState的调用时机在onStart之后**

关于保存和恢复View层次结构，系统的工作流程是这样的：首先Activity被意外终止时，Activity会调用onSaveInstanceState去保存数据，然后Activity会委托Window去保存数据，接着Window再委托它上面的顶级容器去保存数据，顶层容器是一个ViewGroup,一般来说很可能是DecorView.最后顶层元素再去一一通知它的子元素来保存数据，典型的委托思想，上层委托下层，比如View的绘制过程，事件分发等都是采用类似的思想，拿TextView为例，分析一下它到底保存了哪些数据：

```java
   public Parcelable onSaveInstanceState() {
        Parcelable superState = super.onSaveInstanceState();

        // Save state if we are forced to
        final boolean freezesText = getFreezesText();
        boolean hasSelection = false;
        int start = -1;
        int end = -1;

        if (mText != null) {
            start = getSelectionStart();
            end = getSelectionEnd();
            if (start >= 0 || end >= 0) {
                // Or save state if there is a selection
                hasSelection = true;
            }
        }

        if (freezesText || hasSelection) {
            SavedState ss = new SavedState(superState);

            if (freezesText) {
                if (mText instanceof Spanned) {
                    final Spannable sp = new SpannableStringBuilder(mText);

                    if (mEditor != null) {
                        removeMisspelledSpans(sp);
                        sp.removeSpan(mEditor.mSuggestionRangeSpan);
                    }

                    ss.text = sp;
                } else {
                    ss.text = mText.toString();
                }
            }

            if (hasSelection) {
                // XXX Should also save the current scroll position!
                ss.selStart = start;
                ss.selEnd = end;
            }

            if (isFocused() && start >= 0 && end >= 0) {
                ss.frozenWithFocus = true;
            }

            ss.error = getError();

            if (mEditor != null) {
                ss.editorState = mEditor.saveInstanceState();
            }
            return ss;
        }

        return superState;
    }
```

TextView保存了自己的文本选中状态和文本内容



**情况2：资源内存不足导致低优先级的Activity被杀死**

这种情况不太好模拟，但是其数据存储和恢复状态过程和情况1完全一致。











