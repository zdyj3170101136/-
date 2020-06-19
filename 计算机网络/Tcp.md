## Tcp

Transmission control protocol(运输控制协议)

为了比特差错信道，引入了检验和，重传，重复分组。

![img](https://img-blog.csdn.net/20150428144449522?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlzaGVuZ2xvbmc2NjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 格式

16位源端口号 ｜ 16位目的端口号

32位序号

32位确认号

4位首部长度 ｜ 6比特标志字段 ｜16位接收窗口

16位检验和 ｜ 紧急数据指针（无用）

数据



#### MSS

mss（最大报文段长度1460）：tcp报文段的数据大小（被封装在ip和tcp里头）（40字节首部）能通过最大链路层帧长度（1500）（以太网链路层）

#### 序列号和确认号

序列号：首个数据的字节流编号（初始随机，减少已经终止的连接的报文段被误认为是之后的连接的可能性（也许碰巧使用相同ip和端口号））

确认号：期望接收到的<u>下一字节</u>的编号



累积确认机制：只确认到该流第一个丢失的字节

#### Sync_retries = 6

在发送syn报文后没有接受到确认报文的最大重试时间：127s

2^0 + 2^1 + 2^2 + 2^3 + 2^4 + 2^5 + 2^6

https://www.dazhuanlan.com/2019/10/20/5dab43fbaadb1/

#### 超时重传

当报文段大于一定时间还没有收到，就会重传。

超时间隔不能太小（大于EstimatedRTT），避免不必要重传,因此要有一定余量。

不能太大，不然长时间不能重传，数据时延大。



tcp每过一段时间就会计算SamleRTT（报文段发出到确认被收到的时间），作为接下来的报文段的RTT。（不会为已经被重传的计算）

EstimatedRTT = sampleRTT * a + (1 - a) * EstimatedRTT

a = 7 / 8；指数加权，平缓波动。



DevRTT = （1 - b） * DevRTT + b *|sampleRTT - EstimatedRTT|。

反映SampleRTT波动程度，b = 0.25.



当SampleRTT波动大时，余量也大；波动小时，余量也小；

TimeoutiNterval = EstimatedRtt + 4 * DEVRTT

初始为1，超时后加倍；

只要收到ACK或者上层应用数据就会重新计算。



#### tcp_retries

Tcp_retries1: 3,超过这个次数，就会更新路由缓存，然后mtu探测（底层链路层可能使用小于MTU）

tcp_retries2:15,超过这个次数就放弃重传。



然而，真正限制的是重传最大尝试时间。

以RTO_BASE = 200ms，计算尝试boundary次的的最大重传时间。

当大于这个时间，次数没到retries2也会放弃。



因此真正重传次数与RTT相关，当RTT比较小时，次数也会比较多。

```
  linear_backoff_thresh = ilog2(TCP_RTO_MAX/rto_base);

            if (boundary <= linear_backoff_thresh)
                    timeout = ((2 << boundary) - 1) * rto_base;
            else
                    timeout = ((2 << linear_backoff_thresh) - 1) * rto_base +
                            (boundary - linear_backoff_thresh) * TCP_RTO_MAX;
```

http://perthcharles.github.io/2015/09/07/wiki-tcp-retries/



#### 比特差错的信道

由于实际信道会有比特差错。引入重传，检验和，ACK, 重复分组

因此

- 差错检测，TCP首部+数据+假首部（包括目的IP地址，源ip地址，tcp协议号，tcp数据段长度等内容）以16bit为单位求和，然后取反得到检验和。接收端把计算所有的和加上检验和后为1111111（16bit）

- 接守方反馈，接收方用肯定确认（ACK）来确认接受。

- 重传，发送方重传

  如果ACK受损

  一，如果询问ACK的意思，则询问的内容也会受损。

  二，如果增加检验和，让差错恢复。

  三，收到模糊不清的ack，直接重传当前数据。产生重复分组。



实际：使用方法三。

采用序号解决重复分组。



当接受到受损的分组，对上一次收到的正确分组重传一次ack。发送方接受到同一分组的两个ack就知道接收方没有正确接受之后的分组。



最开始采用停等协议（发送下个分组直到接收到确认后），一次只发送一个，因此序号只需要0， 1。

#### 丢包信道

通过定时器应对分组丢失。，定时器。

发送方未收到确认，不知道是因为ack丢失，还是发送的数据丢失。

因此：重传。

对每一个发送数据开启一个定时器，超时则重传。

#### 流水线传输

发送多个分组无需等待确认。

- 增大序号范围，采用32bit的序号
- 缓存分组，缓存已经发送但未确认的分组
- 累积确认，确认号表示所有已经接受的。只重传一个，当超时发生时，只会重传超时的报文段。

接收方对于重复的报文段重复确认，避免发送窗口无法移动。

#### 流量控制

为了防止接受缓存溢出。使用接收窗口。

接收窗口：发送端维护最后一个发送的字节流 - 最后一个接收到的ack小于rwnd。

rwnd动态更新。

rwnd = 接收方应用最后一个读取的字节 - 缓存里头的字节。

#### 拥塞控制

TIMOUOUTINTERVAL * 2次倍增，为了防止路由器泛洪。

#### 为什么需要三次握手

发送方发送SYN了， 初始序列号， 进入SYNC_SEND；接受方发送SYN， ACK， 进入SYNC_ECVD， 自己的初始序列号，发送方序列号 + ！；接收方发送ACK进入ESTABLISH,发送方发送ACK也进入ESTABLISH（并且携带数据）。

1) A --> B SYN my sequence number is X 2) A <-- B ACK your sequence number is X 3) A <-- B SYN my sequence number is Y 4) A --> B ACK your sequence number is Y

为了初始化sequence nummber。（因为没有全局时钟， 证明这个包是新的，而不是delay的）

为什么要对sequence number做确认：接收方不知道这个syn是这条连接的，还是上上条连接的。

https://www.zhihu.com/question/24853633



第一，第二次没有数据为了防止泛洪攻击。

#### 半连接队列

没有完全三次握手

#### SYNC COOKIE

服务器资源在接受到发送端的ACK后才会分配资源；

为了防止发送者发送一堆syn报文来泛洪攻击。

步骤：当服务器接受到一个syn后，以源和目的ip，端口号， 当前时间计算一个cookie，作为初始序号发送SYNACK分组。并且不记忆任何东西。

当服务器接受到ACK后，重新计算cookie，如果发现等于ACK - 1，则分配资源。

（懒惰启用），仅仅当服务器满载时使用。

#### 拥塞控制

代价

- 巨大排队时延，当分组到达速率太快时，经历巨大的排队时延
- 需要重传以补偿因为缓存溢出而丢弃的分组
- 大时延可能会导致不必要的重传浪费带宽
- 每个上游路由器用于转发分组的流量将会被浪费

拥塞窗口cwnd：发送方可发送字节数最大值。



感知拥塞：

超时或者三个接收方的重复ack。



一，慢启动，初始时，cwnd的值为一个MSS值。每次收到传输报文段的首次确认就增加一个MSS。因此慢启动增长快，以指数的形式增长。

何时结束指数增长：

- 接受到丢包时，设置cwnd为1，并且sstresh为cwnd的一半
- 当到达sstresh时，结束慢启动进入拥塞避免阶段
- 收到三个重复ack，执行快速重传（在定时器过期前重传报文），ssthresh等于一半的cwnd，将cwnd的值减半（然后加上3个MSS），进入快速恢复。



二，拥塞避免，

每个RTT只将cwnd增加一个MSS。

- 当出现超时时，cwnd重新设置为1，ssthresh = 一半的cwnd，进入慢启动
- 收到三个重复ack时，ssthresh等于一半的cwnd，将cwnd的值减半（然后加上3个MSS），进入快速恢复



三，快速恢复

tcp重传丢失的数据，如果只收到DUPACK，那么每个增加一个MSS。

- 收到新的ack，cwnd = ssthresh，进入拥塞避免状态。

- 当超时发生，cwnd设置为1， ssthress设置为一半的cwnd，进入慢启动

P179图

#### SACK

选择确认，由于选项部分最大是40字节，所以最多四段。

![截屏2020-06-17 下午5.47.52](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-17 下午5.47.52.png)

#### 同时打开，同时关闭

同时打开：两边同时发动SYN和SYNACK，建立了一条连接，交换了四个报文

#### 关闭

ESTABLISH主动端发送一个FIN报文，进入FIN_WAIT_1状态，当接受到ACK时进入FIN_WAIT_2状态。然后接受FIN（一直没有接受到FIN，超时后会释放资源）再发送ACK， 进入TIMEWAIT状态，等待30s后进入CLOSED状态。



被动端收到FIN后发送ACK进入CLOSE_WAIT状态，发送FIN，进入LAST_ACK状态，再接受到ACK后关闭。



FINZ_WAIT_2状态关闭发送，只能接受。半关闭。



#### TIME_WAIT：

主动发送方等待两个2 * MSL（报文段最大生存时间），保证对方收到ACK，如果没有收到，则对方重发FIN，本地将会重发ACK)

#### 为什么需要四次挥手

主动方在进入了FIN_WAIT2状态后，保留接受功能；

这样被动方没有发送的数据发送完。发送完成之后，被动方也发送一个FIN相当于告诉主动方：自己数据已经发送完毕。对面可以关闭接受功能。



主动关闭方如果不进入TIME_WAIT，被动关闭方将会重传FIN，这个时候关闭方无法识别，会回复RST。



- 服务器关闭大量连接，出现大量TIME_WAIT状态，持续消耗资
- 客户端会消耗掉多有的端口，导致无连接可用

可以设置tcp_max_tw_buckets控制最大timewait数量

#### 同时关闭

ESTABLISH发送一个FIN，同时接受到FIN，进入CLOSING状态-》接受到ACK-〉TIMEWAIT

#### 状态图

P168

CLOSED-》发送syn进入SYN_SENT状态，接受SYN和ACK发送ACK后，进入ESTABLISH状态。

#### KEEPALIVE

即使连接双方无数据发送，也会每隔2h发送一个数据包。

确保对方存活。



#### RST

在任何状态下，收到RST，则进入CLOSED状态。

发送端丢弃缓冲全部数据，接受端收到后也没有ACK

发生情况：

- 发送到不存在的端口
- 关闭一个非空的socket

#### SOCKET

- 监听套接字，处理网络上来的连接
- 传输套接字：向外传输tcp数据
- 待处理套接字：三次握手还没有完成

socket发送只是把数据拷贝到缓冲区，拷贝完饭回。

#### CLOSE_WAIT

服务器端如果积攒大量的COLSE_WAIT状态的socket，有可能将服务器资源减少，套筒无法提供服务：一个进程打开一个socket，然后此进程再派生子进程的时候，此socket的sockfd会被继承，socket是系统级的对象，因此socket的引用计数会变成2，调用close（sockfd）时，内核检查此fd对应的socket上的引用计数。如果引用计数大于1，那么将这个引用计数减1，然后返回，如果引用计数等于1，那么内核会真正通过发FIN来关闭TCP连接，调用shutdown（sockfd，SHUT_RDWR）时，内核不会检查此fd对应的socket上的引用计数，直接通过发FIN来关闭TCP连接

https://blog.csdn.net/realmeh/article/details/18239667

#### TCP沾包

Nagle算法，将多次间隔较小，数据量较小的数据，合并发送。

- 同一个连接，多个应用层的数据混在一起，一个包会混杂好几种应用层数据
- 发送当沾包，用于TCP为了优化，会把数据合在一起再传送
- 接受方沾包，是因为接收方没有及时取走数据。导致多个包的数据都在缓冲区，而接收方按固定的大小读取数据。

先写长度，把消息的长度发送出去。

https://blog.csdn.net/tiandijun/article/details/41961785

#### 糊涂窗口综合症

接收方缓慢消耗，发送方缓慢产生数据都会导致这个问题。

由于服务器无法处理很多数据，会不断减小窗口大小。

最后可能不满一个报文，在几十到几百字节抖动。·

- 延迟向发送方修改窗口大小，除非可以增加一个报文段大小。

- 发送方只有当可以发送一个满的报文，或者至少是接收方一半大小的报文菜发送数据

#### CUBIC

由于在拥塞避免状态，每隔一个RTT窗口大小才加1，这样太慢了。



采用一种三次函数的方法。

发生丢包时的窗口大小为Wmax，而被乘以0.2后减到了W2岁后进入拥塞避免阶段，则最大窗口应该在Wmax和Wmin的中间。

![img](https://allen-kevin.github.io/pictures/paper%20read/20171223095005.png)

在拥塞避免阶段，使用时间t（距离上一次丢包的时间间隔）来计算窗口大小。

当大于Wmax，则依照凸的曲线探测。



- 开始时很快

- 接近Wmax时增长慢

- b = 0.2 《 0.5是为了让收敛更慢，协议更为稳定

- 相比较BIC算法的二分查找，以及每RTT查找一次，导致RTT小的更容易到达顶点

  https://blog.csdn.net/dog250/article/details/53013410

  #### 最大tcp连接

  四元组来唯一标识一个TCP连接：{local ip，本地端口，远程ip，远程端口}，最大tcp连接为客户端ip数×客户端端口数，对IPV4，不考虑ip地址分类等因素，最大tcp连接数大约2 ^ 32（ip数）* 2 ^ 16（端口数）

#### Nigle

对于大于MSS的数据直接发送。

如果仍然有未确认的数据，将当前数据塞进缓冲区。

（最多一个未被确认）

https://blog.csdn.net/sinat_35261315/article/details/79392116?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase

#### 延迟确认

一个报文到达，不会立马回复ack；

而是会延迟回复，希望顺便能将数据和ack一起回复。

（只能延迟单个，如果有两个延续的报文则还是得回答）

https://www.cnblogs.com/zhangkele/p/10080845.html

如果nagle和延迟确认混在一起会很麻烦。