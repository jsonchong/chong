### 正文

今天来讲一下为何我们讲到流畅度，要首先说 60 帧。

1. 60 fps 的意思是说，画面每秒更新60次
2. 这60次更新，是要均匀更新的，不是说一会快，一会慢，那样视觉上也会觉得不流畅
3. 每秒60次，也就是 1/60 ~= 16.67 ms 要更新一次

### 基本概念

1. 我们前面说的 60 fps，是针对软件的
2. 这里说的屏幕的刷新率，是针对硬件的，现在大部分手机屏幕的刷新率，都维持在60 HZ，**移动设备上一般使用60HZ，是因为移动设备对于功耗的要求更高，提高手机屏幕的刷新率，对于手机来说，逻辑功耗会随着频率的增加而线性增大，同时更高的刷新率，意味着更短的TFT数据写入时间，对屏幕设计来说难度更大。**
3. 屏幕刷新率 60 HZ 只能说够用，在目前的情况下是最优解，但是未来肯定是高刷新率屏幕的天下，个人觉得主要依赖下面几点的突破：
   1. 电池技术
   2. 软件技术
   3. 硬件能力

综上，目前的情况下， Android 的渲染机制是 16.67 ms 绘制一次， 60hz 的屏幕也是 16.67 ms 刷新一次，所以大家见到的 Android 手机，基本都是这个配置，目前阶段下的最优解。

### 效果提升

如果要提升，那么软件和硬件需要一起提升，光提升其中一个，是基本没有效果的，比如你屏幕刷新率是 75 hz，软件是 60 fps，每秒软件渲染60次，你刷新 75 次，是没有啥效果的，除了重复帧率费电；同样，如果你屏幕刷新率是 30 hz，软件是 60 fps，那么软件每秒绘制的60次有一半是没有显示就被抛弃了的(丢帧)

如果你想体验120hz 刷新率的屏幕，建议你试试 ipad pro ，用过之后你会觉得，60 hz 的屏幕确实有改善的空间。

下面这张图是 Android 应用在一帧内所需要完成的任务

<img src="https://www.androidperformance.com/images/media/15225938262396.jpg" alt="GPU Profile 的含义" style="zoom:50%;" />
















