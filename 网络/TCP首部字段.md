<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703115723586.png" alt="image-20200703115723586" style="zoom:50%;" />

我们用一次访问百度网页抓包的例子来开始。

```shell
curl -v www.baidu.com
```

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703115815063.png" alt="image-20200703115815063" style="zoom:50%;" />

### 源端口号、目标端口号

在第一个包的详情中，首先看到的高亮部分的源端口号（Src Port）和目标端口号（Dst Port)，这个例子中本地源端口号为 61024，百度目标端口号是 80。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703115848365.png" alt="image-20200703115848365" style="zoom:50%;" />

TCP 报文头部里没有源 ip 和目标 ip 地址，只有源端口号和目标端口号

这也是初学 wireshark 抓包时很多人会有的一个疑问：过滤 ip 地址为 172.19.214.24 包的条件为什么不是 "tcp.addr == 172.19.214.24"，而是 "ip.addr == 172.19.214.24"

### 序列号（Sequence number）

TCP 是面向字节流的协议，通过 TCP 传输的字节流的每个字节都分配了序列号，序列号（Sequence number）指的是本报文段第一个字节的序列号。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703120055280.png" alt="image-20200703120055280" style="zoom:50%;" />

序列号加上报文的长度，就可以确定传输的是哪一段数据。序列号是一个 32 位的无符号整数，达到 2^32-1 后循环到 0。

在 SYN 报文中，序列号用于交换彼此的初始序列号，在其它报文中，序列号用于保证包的顺序。

### 初始序列号（Initial Sequence Number, ISN）

在建立连接之初，通信双方都会各自选择一个序列号，称之为初始序列号。在建立连接时，通信双方通过 SYN 报文交换彼此的 ISN，如下图所示

在建立连接之初，通信双方都会各自选择一个序列号，称之为初始序列号。在建立连接时，通信双方通过 SYN 报文交换彼此的 ISN

初始建立连接的过程中 SYN 报文交换过程如下图所示

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703121611637.png" alt="image-20200703121611637" style="zoom:50%;" />

其中第 2 步和第 3 步可以合并一起，这就是三次握手的过程

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703121638848.png" alt="image-20200703121638848" style="zoom:50%;" />

### 初始序列号是如何生成的

```c
__u32 secure_tcp_sequence_number(__be32 saddr, __be32 daddr,
				 __be16 sport, __be16 dport)
{
	u32 hash[MD5_DIGEST_WORDS];

	net_secret_init();
	hash[0] = (__force u32)saddr;
	hash[1] = (__force u32)daddr;
	hash[2] = ((__force u16)sport << 16) + (__force u16)dport;
	hash[3] = net_secret[15];
	
	md5_transform(hash, net_secret);

	return seq_scale(hash[0]);
}

static u32 seq_scale(u32 seq)
{
	return seq + (ktime_to_ns(ktime_get_real()) >> 6);
}
```

代码中的 net_secret 是一个长度为 16 的 int 数组，只有在第一次调用 net_secret_init 的时时候会将将这个数组的值初始化为随机值。在系统重启前保持不变。

可以看到初始序列号的计算函数 secure_tcp_sequence_number() 的逻辑是通过源地址、目标地址、源端口、目标端口和随机因子通过 MD5 进行进行计算。如果仅有这几个因子，对于四元组相同的请求，计算出的初始序列号总是相同，这必然有很大的安全风险，所以函数的最后将计算出的序列号通过 seq_scale 函数再次计算。

seq_scale 函数加入了时间因子，对于四元组相同的连接，序列号也不会重复了。

### 确认号

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703122045093.png" alt="image-20200703122045093" style="zoom:50%;" />



TCP 使用确认号（Acknowledgment number, ACK）来告知对方下一个期望接收的序列号，小于此确认号的所有字节都已经收到。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703122100954.png" alt="image-20200703122100954" style="zoom:50%;" />

### TCP Flags

TCP 有很多种标记，有些用来发起连接同步初始序列号，有些用来确认数据包，还有些用来结束连接。TCP 定义了一个 8 位的字段用来表示 flags，大部分都只用到了后 6 个，如下图所示

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703122150438.png" alt="image-20200703122150438" style="zoom:50%;" />

我们通常所说的 SYN、ACK、FIN、RST 其实只是把 flags 对应的 bit 位置为 1 而已，这些标记可以组合使用，比如 SYN+ACK，FIN+ACK 等

### 窗口大小

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200703122302859.png" alt="image-20200703122302859" style="zoom:50%;" />

可以看到用于表示窗口大小的"Window Size" 只有 16 位，可能 TCP 协议设计者们认为 16 位的窗口大小已经够用了，也就是最大窗口大小是 65535 字节（64KB）。就像网传盖茨曾经说过：“640K内存对于任何人来说都足够了”一样。

自己挖的坑当然要自己填，因此TCP 协议引入了「TCP 窗口缩放」选项 作为窗口缩放的比例因子，比例因子值的范围是 0 ~ 14，其中最小值 0 表示不缩放，最大值 14。比例因子可以将窗口扩大到原来的 2 的 n 次方，比如窗口大小缩放前为 1050，缩放因子为 7，则真正的窗口大小为 1050 * 128 = 134400







