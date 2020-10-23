### 数据逻辑

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200803142102171.png" alt="image-20200803142102171" style="zoom:50%;" />



<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200803151651965.png" alt="image-20200803151651965" style="zoom:50%;" />

- 触发dislike事件：
  1.  发送dislike请求
  2.  将dislike后的user数据缓存到StackContainer容器中，即压到容器最上面
  3.  推荐次数-1
- 触发like事件：
  1. · 发送like请求
  2.   判断StackContainer容器中的数据是否为空，如果不为空，则清空历史dislike数据
  3.   推荐次数-1
- 触发反悔事件：
  1. 按照需求显示反悔按钮说明本地缓存(StackContainer)存在的dislike数据大于0，则从本地缓存中pop(弹出)一个容器顶上的dislike数据
  2. 由于后端针对like或者dislike的请求流程是覆盖逻辑，所以暂时不存在操作回退的过程，如用户在触发like或者dislike逻辑同上
  3. 注意：从缓存容器中弹出的历史dislike数据会有专门的字段如isHistory来标识是否是缓存的卡片数据，作用是当前触发反悔操作时候，不建议触发本地的推荐次数+1(会增加数据同步的问题)，所以当用户重新触发dislike或者like的逻辑的时候需要判断当前数据的isHistory是否为true，如果是历史反悔数据，则重新like或者dislike不会触发本地推荐次数-1
- 触发AppEnd事件：
  1. AppEnd事件本质上只能在App启动的时候检测到，也就是说在每次app重新启动的时候先清空本地缓存的StackContainer容器里的数据
- 后端推荐数据达到上限：
  1. 是否需要隐藏反悔功能
- 缓存的dislike数据达到上限
  1. 是否需要删除旧数据，删除策略是？
- 账号切换事件：
  1. 清空本地缓存的StackContainer容器里的数据
  2. 如果不清空缓存需要将缓存的容器和当前登录用户id关联



### 交互逻辑

1. 可呈现的最大卡片数量，一定要做限制，否则内存会受影响
2. 如果达到可呈现的最大卡片数量UI该如何处理
3. 回退效果，不太建议按照原交互路径走逆过程，处理起来比较复杂，原因是卡片展现的旋转角度是随机的，建议以一种简单顺畅的回退动画来表达反悔效果，以避免反悔和正常滑动卡片在交互边界问题上带来的视觉误差



