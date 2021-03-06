### 背景

面对多渠道推广场景下，如何实现App用户注册安装的精准归因

除了主流的站点、第三方 App 之外，HTML5 也成了 App 推广的一个主要方式。

如果没有做好对投放效果的追溯和评估，将直接影响到用户增长的整个过程

从数据的角度来看App 激活事件是产品使用的第一入口，激活事件是指新用户首次打开 App 的行为。在进行一轮广告投放之后，对 App 激活渠道的归因分析是定位用户来源的效果评估和推广成本核算的参考标准之一。

### 传统的 App 激活渠道归因

设备号归因、渠道号归因、IP+UA 归因等。以下分别进行简要介绍。

### 1.设备号归因

设备号归因主要应用于第三方 App 中推广，应用场景以信息流广告为主。

大多数情况下，第三方 App 都可以获取到用户移动终端的设备号，如 iOS 系统下设备的 IDFA、Android 设备的 IMEI。因此在信息流等广告中，第三方平台反馈给广告主的点击数据通常会包含用户的设备号信息。当用户下载 App 完成激活后，可以将获取到的设备号与第三方广告平台反馈的设备号进行匹配进行归因，来评估投放效果。(可以参考堆糖接入的第三方广告流平台)

存在的问题：通过其他形式推广带来的激活很可能会首先与设备号归因的方式匹配。比如用户点击了信息流广告之后并没有产生下载 App 的行为，而在之后点击了某处的 H5 广告并促使其完成了最终下载激活 App 的行为。实际应该归因在 H5 广告下(H5是最终的Last Click)。但 H5 的渠道又无法获取用户的设备号信息，所以这次行为很可能就会被归因在优先级较高的信息流形式下，导致误差的产生。

### 2.渠道号归因

**渠道号**指写入安装包的渠道标识。一般会将渠道号提前写入 APK 安装包里，然后分发给不同安装渠道，渠道号会伴随安装包的整个使用周期。用户激活 App 后，可以从安装包获取到渠道号标识信息进行匹配，也是相对准确的归因。

存在的问题：需要在打包的时候就确定需要投放的各种渠道，除了既定的一些应用市场的渠道包之外，还需要针对运营投放的各个平台做渠道打包兼容(烦)

### 3. IP+UA归因

IP+UA 是指通过将用户点击广告时的 IP、User-Agent（用户的操作系统、版本号、手机型号等信息）信息与激活时的 IP、UA 进行关联匹配实现归因分析，一般来说使用短链来收集这两个信息，好处是用户友好、方便管理、方便信息收集和设置。

因为这种方式无法直接获取客户端的设备号等精准信息，并且用户的 IP、UA 两个参数容易随环境变动和重复，比如在办公环境网络下，多个用户使用同一个 IP，或者多个激活 App 的用户使用的手机品牌和型号完全相同等情况下，很难实现精准归因，本质上是一种模糊匹配的归因，适用于比如站内导流，SEM(Search Engine Marketing)推广等

### 基于剪贴板的用户唯一标识归因分析

为了应对获取设备号失败 ，或在 H5、WAP 广告投放场景下使用 IP+UA 精准度不高的问题，可以使用一种基于**剪贴板**归因的方案，来提升渠道归因的精准度。

使用剪贴板进行归因分析的特点在于，可以通过获取唯一标识的方式(uuid)，提升在 HTML5 等无法获取设备号的广告投放场景下的归因准确性，同时减少由设备号匹配或者IP+UA匹配带来的噪音。

### 总结

综上所述，为了提高对不同渠道归因方式的精准度，降低分析误差产生的可能性非常重要。

主要流程如下：

1. 在 HTML5、WAP 等广告投放中，当用户点击广告(比如推广短链或者识别二维码打开的链接)时向剪贴板写入唯一标识；
2. 写入系统剪贴板的同时由服务器记录用户唯一标识；（即通过后端api请求将唯一标识做记录）
3. 用户下载 App 激活后，由 App 读取剪贴板符合规则的信息并上报到服务器；
4. 服务器将关联点击时记录的唯一标识和 App 激活后上报的唯一标识进行匹配，如匹配成功则将相应的投放渠道，自定义的推广信息参数等做为app激活的归因分析；

由于唯一标识可以明确用户的渠道来源，因此可以优先应用剪贴板归因，还可以再用 IP+UA 作为辅助验证手段来提高归因分析的准确性

### 产品方案

1. **后台部分：**

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200707184842431.png" alt="image-20200707184842431" style="zoom:50%;" />

注：广告类型用于区分产品对于用户的展现方式，如推广的是app则选择APP通用渠道，如推广的是网站，或者某个活动的H5可以选择网页通用渠道

点击完成或者生成链接按钮后，将后台配置的渠道信息作为参数通过后端api请求，后端解析配置参数生成对应的推广短链或返回推广二维码的图片信息。

最终会通过后台渠道推广链接管理页面来展示配置成功的数据

![image-20200707185749075](/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200707185749075.png)

**2.前端部分:**













